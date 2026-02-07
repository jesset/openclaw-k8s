# OpenClaw K8s (CN-IM)

OpenClaw 在 Kubernetes 上的一站式部署方案，支持飞书/钉钉/QQ/企业微信等中国 IM 平台。

基于 [openclaw-kubernetes](https://github.com/feiskyer/openclaw-kubernetes) Helm Chart 和 [OpenClaw-Docker-CN-IM](https://github.com/justlovemaki/OpenClaw-Docker-CN-IM) 镜像整合而成。

## 项目结构

```
openclaw-k8s/
├── docker/                     # 优化后的 CN-IM 镜像构建文件
│   ├── Dockerfile
│   └── .dockerignore
├── chart/                      # Helm Chart
│   ├── Chart.yaml
│   ├── values.yaml             # 默认值（含中国 IM 渠道配置）
│   ├── values.schema.json
│   ├── values-minimal.yaml     # 最小化测试
│   ├── values-production.yaml  # 生产环境
│   ├── templates/
│   │   ├── _helpers.tpl
│   │   ├── NOTES.txt
│   │   ├── configmap.yaml
│   │   ├── secret.yaml
│   │   ├── statefulset.yaml
│   │   ├── service.yaml
│   │   ├── ingress.yaml
│   │   ├── serviceaccount.yaml
│   │   ├── pvc.yaml
│   │   ├── hpa.yaml
│   │   ├── poddisruptionbudget.yaml
│   │   └── validate.yaml
│   └── .helmignore
├── README.md
├── CHANGELOG.md
└── .gitignore
```

## 快速开始

### 安装

```bash
# 最小化测试（无持久化）
helm install openclaw chart/ \
  -f chart/values-minimal.yaml \
  --set secrets.openclawGatewayToken=my-secret-token

# 默认安装（带持久化）
helm install openclaw chart/ \
  --set secrets.openclawGatewayToken=my-secret-token

# 生产环境
helm install openclaw chart/ \
  -f chart/values-production.yaml \
  --set secrets.openclawGatewayToken=my-secret-token
```

### 升级

```bash
helm upgrade openclaw chart/ --reuse-values
```

### 配置持久化机制

**重要说明**：OpenClaw 会在运行时动态修改配置文件（例如通过 UI 添加渠道、修改设置）。

- **首次部署**：`values.yaml` 中的 `openclaw.config` 通过 ConfigMap 种子化到 PVC
- **后续重启**：保留 PVC 中的配置（包含所有运行时修改）
- **Helm 升级**：默认**不会**覆盖 PVC 中的配置（保护用户数据）

**如需强制从 values.yaml 重新加载配置**（会丢失运行时修改）：

```bash
# 1. 设置强制重新种子化标志
helm upgrade openclaw chart/ --reuse-values \
  --set openclaw.forceReseedConfig=true

# 2. 等待 Pod 重启完成后，恢复标志
helm upgrade openclaw chart/ --reuse-values \
  --set openclaw.forceReseedConfig=false
```

### 持久化方式选择

Chart 支持两种持久化方式：

**1. PVC（推荐用于生产环境）**

```yaml
persistence:
  enabled: true
  type: pvc
  storageClass: ""
  size: 10Gi
```

**2. HostPath（适用于单节点集群或开发环境）**

```yaml
persistence:
  enabled: true
  type: hostPath
  hostPath: /data/openclaw
```

⚠️ **HostPath 注意事项**：
- 数据存储在节点的文件系统上，Pod 调度到其他节点时无法访问数据
- 需要确保目录存在且权限正确（应由 UID 1000 拥有）
- 不适合多副本或多节点场景

```bash
# 使用 HostPath 部署示例
helm install openclaw chart/ \
  --set secrets.openclawGatewayToken=my-token \
  --set persistence.type=hostPath \
  --set persistence.hostPath=/data/openclaw
```

### 卸载

```bash
helm uninstall openclaw
```

## 模型配置

在 `values.yaml` 的 `openclaw.config.models.providers` 中配置 AI 模型：

```yaml
openclaw:
  config:
    models:
      mode: "merge"
      providers:
        default:
          baseUrl: "https://api.openai.com/v1"
          apiKey: "your-api-key"
          api: "openai-completions"
          models:
            - id: "gpt-4o"
              name: "GPT-4o"
              reasoning: false
              input: ["text", "image"]
              contextWindow: 128000
              maxTokens: 4096
    agents:
      defaults:
        model:
          primary: "default/gpt-4o"
```

## IM 渠道配置

### 飞书 (Feishu)

https://docs.openclaw.ai/channels/feishu

```yaml
secrets:
  feishuAppId: "your-app-id"
  feishuAppSecret: "your-app-secret"

openclaw:
  config:
    channels:
      feishu:
        enabled: true
        connectionMode: "websocket"
        dmPolicy: "pairing"
        groupPolicy: "allowlist"
        requireMention: true
        appId: "your-app-id"
        appSecret: "your-app-secret"
    plugins:
      entries:
        feishu:
          enabled: true
      installs:
        feishu:
          source: "npm"
          spec: "@m1heng-clawd/feishu"
          installPath: "/home/node/.openclaw/extensions/feishu"
```

### 钉钉 (DingTalk)

```yaml
secrets:
  dingtalkClientId: "dingxxxxxx"
  dingtalkClientSecret: "your-secret"
  dingtalkRobotCode: "dingxxxxxx"
  dingtalkCorpId: "dingxxxxxx"
  dingtalkAgentId: "123456789"

openclaw:
  config:
    channels:
      dingtalk:
        enabled: true
        clientId: "dingxxxxxx"
        clientSecret: "your-secret"
        robotCode: "dingxxxxxx"
        corpId: "dingxxxxxx"
        agentId: "123456789"
        dmPolicy: "open"
        groupPolicy: "open"
        messageType: "markdown"
    plugins:
      entries:
        dingtalk:
          enabled: true
      installs:
        dingtalk:
          source: "npm"
          spec: "https://github.com/soimy/clawdbot-channel-dingtalk.git"
          installPath: "/home/node/.openclaw/extensions/dingtalk"
```

### QQ 机器人 (QQ Bot)

```yaml
secrets:
  qqbotAppId: "your-app-id"
  qqbotClientSecret: "your-secret"

openclaw:
  config:
    channels:
      qqbot:
        enabled: true
        appId: "your-app-id"
        clientSecret: "your-secret"
    plugins:
      entries:
        qqbot:
          enabled: true
      installs:
        qqbot:
          source: "path"
          sourcePath: "/home/node/.openclaw/qqbot"
          installPath: "/home/node/.openclaw/extensions/qqbot"
```

### 企业微信 (WeCom)

```yaml
secrets:
  wecomToken: "your-token"
  wecomEncodingAesKey: "your-aes-key"

openclaw:
  config:
    channels:
      wecom:
        enabled: true
        token: "your-token"
        encodingAesKey: "your-aes-key"
    plugins:
      entries:
        openclaw-plugin-wecom:
          enabled: true
      installs:
        openclaw-plugin-wecom:
          source: "npm"
          spec: "https://github.com/sunnoy/openclaw-plugin-wecom.git"
          installPath: "/home/node/.openclaw/extensions/openclaw-plugin-wecom"
```

### Telegram

```yaml
secrets:
  telegramBotToken: "your-bot-token"

openclaw:
  config:
    channels:
      telegram:
        dmPolicy: "pairing"
        botToken: "your-bot-token"
        groupPolicy: "allowlist"
        streamMode: "partial"
    plugins:
      entries:
        telegram:
          enabled: true
```

## 自定义镜像构建

如需自定义镜像（例如添加额外插件），可使用 `docker/Dockerfile`：

```bash
docker build -t my-openclaw:latest docker/
```

然后在 values 中指定自定义镜像：

```yaml
image:
  repository: my-openclaw
  tag: latest
```

## 验证

```bash
# Lint 检查
helm lint chart/

# 渲染模板
helm template test chart/ --set secrets.openclawGatewayToken=test123

# 最小化配置渲染
helm template test chart/ -f chart/values-minimal.yaml --set secrets.openclawGatewayToken=test123

# 生产配置渲染
helm template test chart/ -f chart/values-production.yaml --set secrets.openclawGatewayToken=test123
```

## 架构设计

- **ConfigMap 驱动配置**：`openclaw.config` 通过 ConfigMap 挂载为 `openclaw.json`，init 容器负责种子化到 PVC
- **extensions 目录隔离**：init 容器在首次启动时从镜像复制预装插件到 PVC，后续升级时增量同步
- **Secret 统一管理**：所有 IM 渠道的敏感信息通过 K8s Secret 管理，通过环境变量注入容器
- **安全上下文**：`readOnlyRootFilesystem: false`（Chromium 等组件需要可写文件系统）

## 致谢

- [feiskyer/openclaw-kubernetes](https://github.com/feiskyer/openclaw-kubernetes) - 原始 Helm Chart
- [justlovemaki/OpenClaw-Docker-CN-IM](https://github.com/justlovemaki/OpenClaw-Docker-CN-IM) - 中国 IM 插件镜像
