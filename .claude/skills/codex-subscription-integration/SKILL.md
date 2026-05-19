---
name: codex-subscription-integration
description: 让 OpenHands 复用本机 Codex CLI 的 ChatGPT 订阅 token 调用 gpt-5 / gpt-5-codex 系列，不消耗 OpenAI Platform API 余额。覆盖 token 借用、本地 OAuth 代理、Docker 多容器 env 注入、模型选型与避坑。**步骤幂等，可重复执行用于恢复重启后失效的环境。**
version: 1.1.0
source: codex-openhands-bring-up-2026-05-19
---

# Codex 订阅接入 OpenHands

## 何时使用

- 已 `codex login`，有 `~/.codex/auth.json`
- 想让 OpenHands 用 ChatGPT 订阅算账，而不是 API key
- 自己单机 / 自用，**不要**多机共享或商用（违反 ToS）

## 何时不要用

- 团队共享同一份 token → 触发 ChatGPT ToS、并发 refresh 会互踩
- 面向客户的 SaaS → 违反 ToS"powering third-party services"

## 验收清单（5 条全 ✅ 才算装通）

> 重启后想知道是否还活着，**只跑这一段**就够。所有命令幂等。

```bash
# [1] auth token 已同步、未过期
test -s ~/.config/litellm/chatgpt/auth.json && \
python3 -c "import json,time,base64;d=json.load(open('$HOME/.config/litellm/chatgpt/auth.json'));e=d['expires_at'];print('expires in',int((e-time.time())/86400),'days');assert e>time.time(),'expired'" || echo "❌ FAIL [1] run sync-codex-to-litellm.sh"

# [2] 本地 OAuth proxy 在监听 10531
curl -sf -m 3 http://127.0.0.1:10531/v1/models >/dev/null && echo "✅ [2] proxy alive" || echo "❌ FAIL [2] run Step B"

# [3] proxy 真的能拿到模型回复（消耗 ChatGPT 订阅配额，约 5 token）
curl -sS -m 30 -X POST http://127.0.0.1:10531/v1/chat/completions \
  -H 'Content-Type: application/json' -H 'Authorization: Bearer dummy' \
  -d '{"model":"gpt-5.3-codex","messages":[{"role":"user","content":"reply: pong"}]}' \
  | python3 -c "import sys,json;r=json.load(sys.stdin);print('reply:',r['choices'][0]['message']['content'])"

# [4] OpenHands app 容器健康
curl -sf -m 3 http://localhost:3000/ >/dev/null && echo "✅ [4] openhands-app alive" || echo "❌ FAIL [4] run Step C"

# [5] sandbox 内能解析 host.docker.internal 并通代理
docker exec $(docker ps --filter name=oh-agent-server- -q | head -1) \
    sh -c 'env | grep -E "^HTTPS?_PROXY="' 2>/dev/null && echo "✅ [5] sandbox proxy env propagated"
```

UI 端验收：浏览器开 http://localhost:3000，新建对话发一句话，**收到具体回复而不是「Your last response did not include...」死循环**即通过。

---

## 部署步骤（幂等，可任意次重跑）

### Step A. 同步 Codex token 到 LiteLLM 平铺格式

```bash
mkdir -p ~/bin && cat > ~/bin/sync-codex-to-litellm.sh <<'BASH'
#!/usr/bin/env bash
set -euo pipefail
SRC="${CODEX_HOME:-$HOME/.codex}/auth.json"
DST_DIR="$HOME/.config/litellm/chatgpt"; DST="$DST_DIR/auth.json"
mkdir -p "$DST_DIR"
python3 - "$SRC" "$DST" <<'PY'
import sys,json,os,base64,time
src,dst=sys.argv[1],sys.argv[2]
c=json.load(open(src)); t=c["tokens"]
def jexp(j):
    p=j.split(".")[1]; p+="="*(-len(p)%4)
    return int(json.loads(base64.urlsafe_b64decode(p))["exp"])
out={"access_token":t["access_token"],"refresh_token":t["refresh_token"],
     "id_token":t["id_token"],"account_id":t["account_id"],
     "expires_at":jexp(t["access_token"])}
json.dump(out,open(dst,"w")); os.chmod(dst,0o600)
print(f"OK {out['account_id']} valid {(out['expires_at']-time.time())/86400:.1f} days")
PY
BASH
chmod +x ~/bin/sync-codex-to-litellm.sh
~/bin/sync-codex-to-litellm.sh
```

> Token 寿命约 10 天，Codex CLI 在 8 天 / 401 时自动 refresh `~/.codex/auth.json`，**但 LiteLLM 那份不会自动跟进**。每次出问题先重跑这条脚本。

### Step B. 起 OAuth 代理

Node 22 global fetch 默认不读 `http_proxy`，必须用 undici 注入 ProxyAgent：

```bash
mkdir -p ~/.local/log/oh-proxy && cd ~/.local/log/oh-proxy
[ -f package.json ] || npm init -y >/dev/null
[ -d node_modules/undici ] || npm install --silent undici
cat > bootstrap.cjs <<'JS'
const { setGlobalDispatcher, ProxyAgent } = require('undici');
const url = process.env.HTTPS_PROXY || process.env.https_proxy
         || process.env.HTTP_PROXY  || process.env.http_proxy;
if (url) { setGlobalDispatcher(new ProxyAgent(url));
  console.error(`[bootstrap] fetch -> ${url}`); }
JS

# kill any old instance
pkill -f openai-oauth 2>/dev/null; sleep 1

export HTTP_PROXY=${HTTP_PROXY:-http://127.0.0.1:7890}
export HTTPS_PROXY=${HTTPS_PROXY:-http://127.0.0.1:7890}
export NODE_OPTIONS="-r $HOME/.local/log/oh-proxy/bootstrap.cjs"
export NODE_PATH=$HOME/.local/log/oh-proxy/node_modules
nohup npx -y openai-oauth@latest \
    --host 0.0.0.0 --port 10531 \
    --oauth-file ~/.codex/auth.json \
    --codex-version "$(codex --version | awk '{print $2}')" \
    --models "gpt-5.4,gpt-5.3-codex,gpt-5.2,gpt-5.1-codex-max,gpt-5.1-codex-mini" \
    > ~/.local/log/openai-oauth.log 2>&1 &
disown
```

预期 `~/.local/log/openai-oauth.log` 末尾：
```
[bootstrap] fetch -> http://127.0.0.1:7890
OpenAI-compatible endpoint ready at http://0.0.0.0:10531/v1
```

### Step C. 启动 OpenHands 容器（代理 env 透传给 sandbox）

```bash
docker rm -f openhands-app 2>/dev/null
docker rm -f $(docker ps -aq --filter "name=oh-agent-server-") 2>/dev/null

AGENT_ENV='{"HTTP_PROXY":"http://host.docker.internal:7890","HTTPS_PROXY":"http://host.docker.internal:7890","http_proxy":"http://host.docker.internal:7890","https_proxy":"http://host.docker.internal:7890","NO_PROXY":"localhost,127.0.0.1,host.docker.internal"}'

docker run -d --rm \
  --name openhands-app \
  -p 3000:3000 \
  --add-host host.docker.internal:host-gateway \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v ~/.openhands:/.openhands \
  -e SANDBOX_RUNTIME_CONTAINER_IMAGE=ghcr.io/openhands/agent-server:1.19.1-python \
  -e OH_AGENT_SERVER_ENV="$AGENT_ENV" \
  -e LOG_ALL_EVENTS=true \
  ghcr.io/openhands/openhands:latest
```

### Step D. OpenHands UI 配置（一次性，配完保存）

Settings → LLM → **Advanced** 标签：

| 字段 | 值 |
|---|---|
| Name | `chatgpt-codex` |
| **Custom Model** | **`openai_like/gpt-5.3-codex`** ← 关键，前缀必须是 `openai_like/` |
| **Base URL** | **`http://host.docker.internal:10531/v1`** |
| API Key | `dummy`（任意字符串） |

---

## 关键技术决策

### 为什么必须用本地代理 + `openai_like/` 前缀

LiteLLM 1.85 的 chat→responses transformer 看到 `gpt-5.*` 模型名（不论 `openai/`、`chatgpt/` 前缀）就强行走 responses bridge，而该 bridge 不识别 Codex 后端返回的 `reasoning.encrypted_content` item type，会吐空 `output[]` 报 `Unknown items in responses API response: []`。OpenHands SDK 默认 `stream: False`、UI 又没法切，所以踩中即死循环。

`openai_like/<model>` 是 litellm 给「纯净 OpenAI 兼容端点」的 provider，跳过所有 schema 转换。配合本地 `openai-oauth` 代理把 Codex 后端归一化成标准 chat completions JSON，整条链路里 litellm 一次都不做改写。

### `~/.codex/auth.json` 关键字段

- 嵌套 schema：`{auth_mode, tokens: {access_token, refresh_token, id_token, account_id}, last_refresh}`
- access_token 是 RS256 JWT，audience `https://api.openai.com/v1`，约 10 天有效
- 真正调用的端点是 `https://chatgpt.com/backend-api/codex/responses`（**不是** `api.openai.com`）
- client_id 全网共用：`app_EMoamEEZ73f0CkXaXp7hrann`

### 可用模型（Team plan 实测）

✅ `openai_like/gpt-5.4` / `gpt-5.3-codex` / `gpt-5.2` / `gpt-5.1-codex-max` / `gpt-5.1-codex-mini`
❌ `gpt-5.2-codex` / `gpt-5-codex` — 后端 400

---

## 故障排查速查

| 症状 | 根因 | 修复 |
|---|---|---|
| `Incorrect API key provided` | model 前缀写成 `openai/` 或没前缀，litellm 真去 api.openai.com 验 key | 改 `openai_like/` |
| `Unknown items in responses API response: []` | litellm 强走 responses bridge transformer | 改 `openai_like/` |
| sandbox 日志弹"Sign in with ChatGPT using device code" | 代理 env 没透传给 sandbox | 检查 `OH_AGENT_SERVER_ENV` 是不是合法 JSON，重跑 Step C |
| proxy 启动直接 "fetch failed" | Node fetch 不走代理 | 用 `bootstrap.cjs` 注入 undici ProxyAgent |
| UI 一直 loading | 旧 conversation 卡在挂掉的 LLM 配置上 | `rm -rf ~/.openhands && docker rm -f openhands-app && 重跑 Step C` |
| `Unsupported parameter: temperature` | OpenHands SDK 默认带 temperature，GPT-5 不接受 | proxy 已自动 strip；若仍报错，UI All 标签把 temperature 清空 |

## 重启后恢复

按顺序跑 Step A → B → C，再过验收清单。整套约 1 分钟。

## 参考资料

- LiteLLM openai_like provider: https://docs.litellm.ai/docs/providers/openai_compatible
- openai-oauth: https://github.com/EvanZhouDev/openai-oauth
- OpenAI Codex auth.json 官方 CI/CD 用法: https://developers.openai.com/codex/auth/ci-cd-auth
- ToS（关键条款）: https://help.openai.com/en/articles/9793128-about-chatgpt-pro-plans
