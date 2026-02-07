# OpenClaw CN-IM Docker 镜像（优化版）
# 支持飞书/钉钉/QQ/企业微信等中国 IM 平台
FROM node:22-slim AS builder

WORKDIR /build

# 安装构建依赖
RUN apt-get update \
  && apt-get install -y --no-install-recommends \
    git \
    python3 \
    ca-certificates \
  && rm -rf /var/lib/apt/lists/*

RUN npm install -g npm@latest openclaw@latest

# 切换到 node 用户安装插件
USER node
RUN mkdir -p /home/node/.openclaw/extensions

# 安装中国 IM 插件（忽略错误以防网络问题）
# RUN timeout 300 openclaw plugins install @m1heng-clawd/feishu || true
RUN timeout 300 openclaw plugins install @openclaw/feishu

RUN timeout 300 openclaw plugins install https://github.com/soimy/clawdbot-channel-dingtalk.git || true
RUN cd /tmp \
  && git clone https://github.com/justlovemaki/qqbot.git \
  && cd qqbot \
  && timeout 300 openclaw plugins install . || true
RUN timeout 300 openclaw plugins install openclaw-plugin-wecom || true

# --- 最终镜像 ---
FROM node:22-slim

# 安装运行时依赖（合并 RUN 层减少镜像体积）
RUN apt-get update \
  && apt-get install -y --no-install-recommends \
    bash \
    ca-certificates \
    chromium \
    curl \
    fonts-liberation \
    fonts-noto-cjk \
    fonts-noto-color-emoji \
    git \
    jq \
    python3 \
    tini \
  && rm -rf /var/lib/apt/lists/* \
  && npm install -g npm@latest \
    openclaw@latest \
    opencode-ai@latest \
    playwright \
    playwright-extra \
    puppeteer-extra-plugin-stealth \
    @steipete/bird \
  && npx playwright install chromium --with-deps

# 创建目录并设置权限
RUN mkdir -p /home/node/.openclaw/workspace \
  && chown -R node:node /home/node

# 从 builder 阶段复制预装插件
COPY --from=builder --chown=node:node /home/node/.openclaw/extensions /home/node/.openclaw/extensions

ENV HOME=/home/node \
    TERM=xterm-256color

EXPOSE 18789 18790

WORKDIR /home/node

USER node

ENTRYPOINT ["tini", "--"]
CMD ["openclaw", "gateway", "--verbose"]
