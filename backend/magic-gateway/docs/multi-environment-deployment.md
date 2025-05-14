# API网关的多环境部署指南

本文档详细说明如何使用Docker容器技术为API网关实现测试(test)、预发布(pre)和生产(production)三套独立环境的部署方案。

## 方案概述

通过Docker容器技术，我们可以为每个环境创建完全隔离的API网关实例，每个实例具有自己的：
- 配置文件
- 环境变量
- 网络端口
- 存储卷

该方案具有以下优势：
- 环境隔离：确保不同环境之间互不干扰
- 配置分离：每个环境使用独立的配置文件
- 部署一致：保证各环境使用相同的代码版本
- 可扩展性：轻松添加更多环境（如开发环境、QA环境等）
- 资源控制：可为不同环境分配不同的资源

## 前置条件

- Docker 20.10.x 或更高版本
- Docker Compose 2.x 或更高版本
- Git（用于克隆仓库）
- 基本的命令行操作知识

## 实现步骤

### 1. 项目结构调整

首先，调整项目结构以支持多环境配置：

```bash
mkdir -p config/{test,pre,prod}
```

创建以下目录结构：

```
magic-gateway/
├── config/                  # 多环境配置目录
│   ├── test/                # 测试环境配置
│   │   └── .env            # 测试环境变量
│   ├── pre/                 # 预发布环境配置
│   │   └── .env            # 预发布环境变量  
│   └── prod/                # 生产环境配置
│       └── .env            # 生产环境变量
├── Dockerfile              # 容器构建文件
├── docker-compose.yml      # 容器编排配置
└── ...                     # 其他项目文件
```

### 2. 创建多环境配置文件

为每个环境创建独立的环境变量配置文件。

#### 测试环境配置文件 (config/test/.env)

```ini
# 测试环境配置
JWT_SECRET=test-jwt-secret-key-change-me
API_GATEWAY_VERSION=1.0.0-test
MAGIC_GATEWAY_DEBUG=true
PORT=8001
MAGIC_GATEWAY_API_KEY=test-gateway-api-key

# OpenAI 服务测试环境配置
OPENAI_API_KEY=sk-test-xxxx
OPENAI_API_BASE_URL=https://api.openai.com/v1
OPENAI_MODEL=gpt-3.5-turbo

# 其他服务测试环境配置
MAGIC_API_KEY=test-xxx
MAGIC_API_BASE_URL=https://api.magic-test.com/v1
MAGIC_MODEL=gpt-4-test
```

#### 预发布环境配置文件 (config/pre/.env)

```ini
# 预发布环境配置
JWT_SECRET=pre-jwt-secret-key-change-me
API_GATEWAY_VERSION=1.0.0-pre
MAGIC_GATEWAY_DEBUG=true
PORT=8002
MAGIC_GATEWAY_API_KEY=pre-gateway-api-key

# OpenAI 服务预发布环境配置
OPENAI_API_KEY=sk-pre-xxxx
OPENAI_API_BASE_URL=https://api.openai.com/v1
OPENAI_MODEL=gpt-4

# 其他服务预发布环境配置
MAGIC_API_KEY=pre-xxx
MAGIC_API_BASE_URL=https://api.magic-pre.com/v1
MAGIC_MODEL=gpt-4-pre
```

#### 生产环境配置文件 (config/prod/.env)

```ini
# 生产环境配置
JWT_SECRET=prod-jwt-secret-key-change-me
API_GATEWAY_VERSION=1.0.0
MAGIC_GATEWAY_DEBUG=false
PORT=8003
MAGIC_GATEWAY_API_KEY=prod-gateway-api-key

# OpenAI 服务生产环境配置
OPENAI_API_KEY=sk-prod-xxxx
OPENAI_API_BASE_URL=https://api.openai.com/v1
OPENAI_MODEL=gpt-4

# 其他服务生产环境配置
MAGIC_API_KEY=prod-xxx
MAGIC_API_BASE_URL=https://api.magic.com/v1
MAGIC_MODEL=gpt-4o-global
```

### 3. 更新 Dockerfile

优化Dockerfile以支持多环境部署：

```dockerfile
FROM golang:1.20-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o api-gateway main.go

FROM alpine:latest
WORKDIR /app
COPY --from=builder /app/api-gateway .
# 创建配置目录
RUN mkdir -p /app/config
# 设置环境变量
ENV PORT=8000
ENV MAGIC_GATEWAY_ENV=dev
# 暴露端口
EXPOSE 8000
# 启动命令
CMD ["./api-gateway"]
```

### 4. 创建多环境 Docker Compose 配置

创建一个支持多环境的`docker-compose.yml`文件：

```yaml
version: '3'

services:
  magic-gateway:
    build:
      context: .
      dockerfile: Dockerfile
    restart: always
    environment:
      - PORT=${PORT:-8000}
      - MAGIC_GATEWAY_ENV=${ENV:-dev}
      - JWT_SECRET=${JWT_SECRET:-default-secret-key}
      - MAGIC_GATEWAY_API_KEY=${API_KEY:-default-api-key}
      - MAGIC_GATEWAY_DEBUG=${DEBUG:-false}
    ports:
      - "${EXTERNAL_PORT:-8000}:${PORT:-8000}"
    volumes:
      - ./config/${ENV:-dev}/.env:/app/.env
    networks:
      - gateway-network

networks:
  gateway-network:
    driver: bridge
```

### 5. 创建快捷启动脚本

创建一个`deploy.sh`脚本，简化多环境部署流程：

```bash
#!/bin/bash

# 定义颜色
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
RED='\033[0;31m'
NC='\033[0m' # 恢复默认颜色

# 帮助信息
function show_help {
  echo -e "${YELLOW}API网关多环境部署脚本${NC}"
  echo "用法: $0 [环境] [操作]"
  echo ""
  echo "环境选项:"
  echo "  test       - 测试环境 (端口 8001)"
  echo "  pre        - 预发布环境 (端口 8002)"
  echo "  prod       - 生产环境 (端口 8003)"
  echo ""
  echo "操作选项:"
  echo "  start      - 启动指定环境"
  echo "  stop       - 停止指定环境"
  echo "  restart    - 重启指定环境"
  echo "  logs       - 查看指定环境日志"
  echo "  status     - 查看指定环境状态"
  echo "  all        - 操作所有环境"
  echo ""
  echo "示例:"
  echo "  $0 test start    - 启动测试环境"
  echo "  $0 all start     - 启动所有环境"
  echo "  $0 prod logs     - 查看生产环境日志"
}

# 环境参数检查
if [ "$1" != "test" ] && [ "$1" != "pre" ] && [ "$1" != "prod" ] && [ "$1" != "all" ]; then
  show_help
  exit 1
fi

# 操作参数检查
if [ "$2" != "start" ] && [ "$2" != "stop" ] && [ "$2" != "restart" ] && [ "$2" != "logs" ] && [ "$2" != "status" ]; then
  show_help
  exit 1
fi

# 启动指定环境
function start_env {
  local env=$1
  local port
  local api_key
  local jwt_secret
  local debug
  
  echo -e "${GREEN}正在启动 $env 环境...${NC}"
  
  case $env in
    test)
      port=8001
      api_key="test-gateway-api-key"
      jwt_secret="test-jwt-secret-key"
      debug="true"
      ;;
    pre)
      port=8002
      api_key="pre-gateway-api-key"
      jwt_secret="pre-jwt-secret-key"
      debug="true"
      ;;
    prod)
      port=8003
      api_key="prod-gateway-api-key"
      jwt_secret="prod-jwt-secret-key"
      debug="false"
      ;;
  esac
  
  ENV=$env PORT=$port EXTERNAL_PORT=$port API_KEY=$api_key JWT_SECRET=$jwt_secret DEBUG=$debug \
    docker-compose -p magic-gateway-$env up -d
  
  echo -e "${GREEN}$env 环境已启动，访问地址: http://localhost:$port${NC}"
}

# 停止指定环境
function stop_env {
  local env=$1
  echo -e "${YELLOW}正在停止 $env 环境...${NC}"
  docker-compose -p magic-gateway-$env down
  echo -e "${YELLOW}$env 环境已停止${NC}"
}

# 查看指定环境日志
function view_logs {
  local env=$1
  echo -e "${GREEN}正在查看 $env 环境日志...${NC}"
  docker-compose -p magic-gateway-$env logs -f
}

# 查看指定环境状态
function check_status {
  local env=$1
  echo -e "${GREEN}$env 环境状态:${NC}"
  docker-compose -p magic-gateway-$env ps
}

# 处理操作
function process_operation {
  local env=$1
  local operation=$2
  
  case $operation in
    start)
      start_env $env
      ;;
    stop)
      stop_env $env
      ;;
    restart)
      stop_env $env
      start_env $env
      ;;
    logs)
      view_logs $env
      ;;
    status)
      check_status $env
      ;;
  esac
}

# 处理所有环境
function process_all_envs {
  local operation=$1
  
  for env in test pre prod; do
    process_operation $env $operation
  done
}

# 主流程
if [ "$1" == "all" ]; then
  process_all_envs $2
else
  process_operation $1 $2
fi
```

确保该脚本具有执行权限：

```bash
chmod +x deploy.sh
```

### 6. 部署与管理

现在可以使用`deploy.sh`脚本方便地管理不同环境：

```bash
# 启动测试环境
./deploy.sh test start

# 启动预发布环境
./deploy.sh pre start

# 启动生产环境
./deploy.sh prod start

# 启动所有环境
./deploy.sh all start

# 查看测试环境日志
./deploy.sh test logs

# 停止生产环境
./deploy.sh prod stop
```

## 环境访问方式

不同环境的API网关通过不同端口访问：

- 测试环境: http://localhost:8001
- 预发布环境: http://localhost:8002
- 生产环境: http://localhost:8003

## 环境隔离与数据独立性

- 每个环境使用独立的Docker容器，确保进程隔离
- 每个环境使用独立的环境变量文件，确保配置隔离
- 每个环境使用独立的端口，确保网络隔离
- JWT密钥在不同环境独立设置，确保令牌不互通

## 环境特性差异

1. **测试环境**：
   - 调试模式开启，输出详细日志
   - 使用测试版API密钥
   - 可以使用较低配置模型进行测试

2. **预发布环境**：
   - 调试模式开启，便于排查问题
   - 使用接近生产的配置
   - 用于功能验证和性能测试

3. **生产环境**：
   - 调试模式关闭，提高性能
   - 使用正式API密钥
   - 使用最稳定的模型配置

## 安全建议

1. 生产环境的配置文件应妥善保管，避免泄露敏感信息
2. 各环境的JWT密钥和API密钥应当不同
3. 定期轮换各环境的密钥
4. 生产环境应使用HTTPS或内部网络保护
5. 使用环境变量注入敏感信息，避免将敏感信息硬编码在配置文件中

## 故障排查

1. 查看容器日志
   ```bash
   ./deploy.sh <环境> logs
   ```

2. 检查环境配置文件
   ```bash
   cat config/<环境>/.env
   ```

3. 检查容器状态
   ```bash
   ./deploy.sh <环境> status
   ```

4. 重启服务
   ```bash
   ./deploy.sh <环境> restart
   ```

## 扩展与优化

1. **添加监控**: 集成Prometheus和Grafana监控容器状态
2. **负载均衡**: 在高流量场景下，可以通过Docker Compose横向扩展服务
3. **CI/CD集成**: 将多环境部署集成到CI/CD流程中
4. **自动化测试**: 为每个环境添加自动化测试脚本

---

本部署方案通过Docker容器技术实现了API网关的多环境部署，确保各环境之间的隔离性和配置独立性，同时保持代码的一致性，便于管理和维护。 