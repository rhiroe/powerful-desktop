FROM mcr.microsoft.com/devcontainers/typescript-node:1-22-bookworm

# node-pty のネイティブビルド、Electron のヘッドレス実行（xvfb）に必要
RUN apt-get update && export DEBIAN_FRONTEND=noninteractive \
    && apt-get install -y --no-install-recommends \
        build-essential python3 pkg-config \
        xvfb libnss3 libatk1.0-0 libatk-bridge2.0-0 libcups2 libdrm2 \
        libgbm1 libgtk-3-0 libasound2 libxss1 libxtst6 libsecret-1-dev \
    && rm -rf /var/lib/apt/lists/*

# Claude Code の設定・認証情報・セッションを永続化する named volume のマウント先。
# 事前に node 所有で作成しておくことで、初回マウント時に volume の所有者が node になる
RUN mkdir -p /home/node/.claude && chown -R node:node /home/node/.claude
