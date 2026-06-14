FROM sparfenyuk/mcp-proxy:latest

RUN apk add --no-cache nodejs npm \
 && npm config set registry https://registry.npmmirror.com \
 && npm install -g @agentmemory/mcp@0.9.27 \
 && npm cache clean --force

ENTRYPOINT ["catatonit", "--", "mcp-proxy"]
