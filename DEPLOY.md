# DeerFlow 完整部署指南（WSL2 + Docker Desktop）

#把文档下面的url、key之类的提前编辑好，确定能正常调用；然后随便建立一个文件夹把本文档放进去，然后用trae选solo模式说：读取本文档，并按照文档步骤部署；真正的超简单部署；反正官方的我是闹不了。

## 环境要求
- 操作系统：Windows 10/11
- WSL2 + Ubuntu（推荐 22.04 或 24.04）
- Docker Desktop（已配置 WSL2 集成）
- Git

---

## 第一步：克隆仓库

```bash
git clone https://github.com/your-repo/deer-flow.git
cd deer-flow
```

---

## 第二步：创建配置文件

### 2.1 创建主配置文件 `config.yaml`

在项目根目录创建 `config.yaml`：

```yaml
# DeerFlow Configuration

config_version: 3

log_level: info

token_usage:
  enabled: false

models:
  - name: siliconflow-model
    display_name: SiliconFlow Qwen2.5-32B
    use: deerflow.models.patched_openai:PatchedChatOpenAI
    model: Qwen/Qwen2.5-32B-Instruct  # SiliconFlow 模型名称格式
    api_key: sk-your-api-key-here
    base_url: https://api.siliconflow.cn/v1
    max_tokens: 32768
    temperature: 0.7
    supports_thinking: true
    supports_vision: true

tool_groups:
  - name: web
  - name: file:read
  - name: file:write
  - name: bash

tools:
  - name: web_search
    group: web
    use: deerflow.community.ddg_search.tools:web_search_tool
    max_results: 5

sandbox:
  use: deerflow.sandbox.local:LocalSandboxProvider
```

**注意**：
- SiliconFlow 模型名称格式为 `Qwen/Qwen2.5-32B-Instruct`（带厂商前缀）
- 其他常见模型：`deepseek-ai/DeepSeek-R1`, `THUDM/glm-4-9b-chat`

### 2.2 创建扩展配置文件 `extensions_config.json`

在项目根目录创建 `extensions_config.json`：

```json
{
  "mcpServers": {},
  "skills": {}
}
```

### 2.3 创建前端环境文件 `frontend/.env`

```bash
touch frontend/.env
```

内容可以为空，但文件必须存在。

### 2.4 创建主环境文件 `.env`

在项目根目录创建 `.env`：

```bash
# 容器内配置文件路径（应用读取用，固定值）
DEER_FLOW_CONFIG_PATH=/app/backend/config.yaml
DEER_FLOW_EXTENSIONS_CONFIG_PATH=/app/backend/extensions_config.json

# 主机配置文件路径（Docker卷挂载用，根据实际情况修改）
# Windows 用户建议使用 /mnt/c/Users/用户名/ 路径
DEER_FLOW_HOST_CONFIG_PATH=/mnt/c/Users/YOUR_USERNAME/deer-flow-config.yaml
DEER_FLOW_HOST_EXTENSIONS_CONFIG_PATH=/mnt/c/Users/YOUR_USERNAME/deer-flow-extensions-config.json

# 其他配置
DEER_FLOW_HOME=/app/backend/.deer-flow
DEER_FLOW_REPO_ROOT=/app
DEER_FLOW_DOCKER_SOCKET=/var/run/docker.sock
PORT=2026
HOME=/root
BETTER_AUTH_SECRET=deer-flow-secret-key-change-in-production
```

**重要**：将 `YOUR_USERNAME` 替换为你的 Windows 用户名。

---

## 第三步：修改 docker-compose.yaml

**必须修改**，解决配置文件路径冲突问题。

编辑 `docker/docker-compose.yaml`：

### 3.1 修改 gateway 服务的 volumes 部分

找到 gateway 服务的 volumes 配置（约第 70-72 行），修改为：

```yaml
gateway:
  # ... 其他配置 ...
  volumes:
    - ${DEER_FLOW_HOST_CONFIG_PATH}:/app/backend/config.yaml:ro
    - ${DEER_FLOW_HOST_EXTENSIONS_CONFIG_PATH}:/app/backend/extensions_config.json:ro
    - ../skills:/app/skills:ro
    - ${DEER_FLOW_HOME}:/app/backend/.deer-flow
    # ... 其他卷 ...
```

### 3.2 修改 langgraph 服务的 volumes 部分

找到 langgraph 服务的 volumes 配置（约第 118-120 行），修改为：

```yaml
langgraph:
  # ... 其他配置 ...
  volumes:
    - ${DEER_FLOW_HOST_CONFIG_PATH}:/app/config.yaml:ro
    - ${DEER_FLOW_HOST_EXTENSIONS_CONFIG_PATH}:/app/extensions_config.json:ro
    - ${DEER_FLOW_HOME}:/app/backend/.deer-flow
    # ... 其他卷 ...
```

**修改原因**：
- 原配置使用 `DEER_FLOW_CONFIG_PATH` 同时作为主机路径和容器路径
- 应用代码读取的是容器内路径 `/app/backend/config.yaml`
- Docker 卷挂载需要的是主机路径
- 分离后：`DEER_FLOW_HOST_CONFIG_PATH` 用于挂载，`DEER_FLOW_CONFIG_PATH` 用于应用读取

---

## 第四步：复制配置文件到可访问位置

由于 Docker Desktop 的文件访问限制，需要将配置文件复制到 Windows 可访问的路径：

```bash
# 在 WSL2 中执行，将 YOUR_USERNAME 替换为你的 Windows 用户名
cp config.yaml /mnt/c/Users/YOUR_USERNAME/deer-flow-config.yaml
cp extensions_config.json /mnt/c/Users/YOUR_USERNAME/deer-flow-extensions-config.json
```

**避坑**：不要放在 WSL2 的 `/tmp` 或 `/home` 目录，Docker Desktop 无法访问这些路径。

---

## 第五步：启动服务

```bash
cd docker
docker compose -f docker-compose.yaml --env-file ../.env up -d
```

---

## 第六步：验证部署

### 6.1 查看容器状态
```bash
docker ps
```

应该看到 4 个容器都在运行：
- deer-flow-nginx
- deer-flow-gateway
- deer-flow-langgraph
- deer-flow-frontend

### 6.2 查看日志
```bash
# gateway 日志
docker logs deer-flow-gateway --tail 20

# langgraph 日志
docker logs deer-flow-langgraph --tail 20
```

正常应该看到 `Application startup complete.`

### 6.3 测试 API
```bash
curl http://localhost:2026/api/models
```

应该返回配置的模型列表。

### 6.4 访问界面
打开浏览器访问：http://localhost:2026

---

## 常用命令

```bash
# 查看所有日志
docker logs deer-flow-gateway --tail 50
docker logs deer-flow-langgraph --tail 50
docker logs deer-flow-nginx --tail 50

# 重启服务
docker compose -f docker/docker-compose.yaml --env-file .env restart

# 停止服务
docker compose -f docker/docker-compose.yaml --env-file .env down

# 进入容器调试
docker exec -it deer-flow-gateway bash
docker exec -it deer-flow-langgraph bash
```

---

## 避坑指南

### ❌ 坑 1：502 Bad Gateway 错误

**现象**：HTTP 502，nginx 无法连接到后端

**原因**：重启服务后容器 IP 变化，nginx 缓存了旧的 IP

**解决**：重启 nginx
```bash
docker compose -f docker/docker-compose.yaml --env-file .env restart nginx
```

---

### ❌ 坑 2：模型不存在错误

**现象**：`Model does not exist. Please check it carefully.`

**原因**：模型名称格式不正确

**解决**：SiliconFlow 模型名称需要带厂商前缀，如：
- ✅ `Qwen/Qwen2.5-32B-Instruct`
- ✅ `deepseek-ai/DeepSeek-R1`
- ❌ `Qwen2.5-32B-Instruct`（缺少前缀）

---

### ❌ 坑 3：配置文件找不到

**现象**：`Config file specified by environment variable DEER_FLOW_CONFIG_PATH not found`

**原因**：
1. 文件路径错误
2. Docker 无法访问该路径
3. 文件被挂载为目录（文件不存在时 Docker 会自动创建目录）

**解决**：
1. 确认文件存在：`ls -la /mnt/c/Users/用户名/deer-flow-config.yaml`
2. 确认是文件不是目录
3. 使用 Windows 可访问的路径（`/mnt/c/Users/...`）

---

### ❌ 坑 4：线程找不到错误

**现象**：`Thread with ID xxx not found`

**原因**：使用的是内存存储（InMemorySaver），重启服务后数据丢失

**解决**：新建对话即可，这是正常现象

---

### ❌ 坑 5：缺少 HOME 环境变量

**现象**：`required variable HOME is missing a value`

**解决**：在 `.env` 中添加 `HOME=/root`

---

### ❌ 坑 6：frontend/.env 文件缺失

**现象**：`env file frontend/.env not found`

**解决**：`touch frontend/.env`

---

## 文件结构总结

```
deer-flow/
├── config.yaml                      # 主配置文件（项目目录）
├── extensions_config.json           # 扩展配置文件（项目目录）
├── .env                             # 主环境变量文件
├── DEPLOY.md                        # 本部署文档
├── frontend/
│   └── .env                         # 前端环境变量文件（可为空）
├── docker/
│   └── docker-compose.yaml          # Docker Compose 配置（已修改）
└── ...
```

Windows 用户目录下：
```
C:\Users\YOUR_USERNAME\
├── deer-flow-config.yaml            # 主配置文件（Docker挂载用）
└── deer-flow-extensions-config.json # 扩展配置文件（Docker挂载用）
```

---

## 模型配置参考

### SiliconFlow 常用模型
| 模型名称 | 说明 |
|---------|------|
| `Qwen/Qwen2.5-32B-Instruct` | 通义千问 2.5 32B |
| `Qwen/Qwen2.5-72B-Instruct` | 通义千问 2.5 72B |
| `deepseek-ai/DeepSeek-R1` | DeepSeek R1 |
| `deepseek-ai/DeepSeek-V3` | DeepSeek V3 |
| `THUDM/glm-4-9b-chat` | 智谱 GLM-4 |

### 其他 API 提供商

**OpenAI 格式通用配置**：
```yaml
models:
  - name: my-model
    display_name: My Model
    use: deerflow.models.patched_openai:PatchedChatOpenAI
    model: gpt-4o  # 根据提供商文档填写
    api_key: sk-xxx
    base_url: https://api.xxx.com/v1
    max_tokens: 32768
    temperature: 0.7
    supports_thinking: true
    supports_vision: true
```

---

## 访问地址

- 主界面：http://localhost:2026

---

## 故障排查

如果部署后遇到问题：

1. **先看日志**：`docker logs deer-flow-langgraph --tail 50`
2. **检查配置**：确认 `.env` 和 `config.yaml` 路径正确
3. **检查模型名称**：确认模型名称格式正确
4. **重启服务**：`docker compose restart`
5. **重启 nginx**：解决 502 错误
