# Hermes Agent 优化：解决 GLM-5.2 模型 429 (1305 overloaded)

> Hermes Agent 使用 `zai/glm-5.2` Coding Plan 后疯狂吃 429？换 key、换 endpoint、降请求长度全没用？本文记录了一次从假限流到真凶（system prompt 内容过滤 + 客户端指纹检测）的完整排查，并给出了不改磁盘、只改请求路径的 Provider-gated 方案。
>
> 作者：Yawatasensei | 分类：技术分享 | 标签：开发, AI, LLM, AGENT


> 🌐 **[在线宣传页](https://samge0.github.io/hermes-provider-api-workarounds/)** — 可视化了解功能特性与工作流程

---

<img width="798" height="920" alt="Hermes_版本信息查询" src="https://github.com/user-attachments/assets/5e29bdfd-5da1-4d8b-b1b0-2b58003c364e" />


## 初步排查：这不是 Z.ai 的限流

最近升级了 Z.ai 的 Coding Plan，也尝试把 Hermes Agent 的主模型从 `DeepSeek V4 Pro` 切到 `zai/glm-5.2`，但是发出去的第一个请求直接吃 `HTTP 429` + provider `code 1305`。第一反应是额度用完了，去 Z.AI 后台查，额度充足。换了一个新 key，一样。然后我尝试降低请求长度、砍 tools、换 model 变体、减少 prefill 的 system prompt 注入等等，全没用。但诡异的是，同一个 key 用 curl 裸调一个短 prompt，能通。

### 1) 第一反应：额度 / Key 问题

Z.AI 后台显示额度充足（5 小时和周额度都在 90% 以上），key 状态也正常。换了一个全新的 Coding Plan key，同样的 429 / `code 1305`。排除了凭证问题。

### 2) 缩小变量：同 key 不同 payload

尝试用同一个 key 发两个请求：

- **curl 裸调** `{"model": "glm-5.2", "messages": [{"role":"user","content":"hi"}]}` → **200**，正常返回
- **Hermes Agent 完整请求**（system prompt + tools + history）→ **429** / `code 1305 overloaded`

请求长度差了两个数量级，所以最初我怀疑是 payload 太大触发了某种保护。但把 Hermes 的 tools 删到只剩一个、history 清空，仍然会返回 429。甚至把 system prompt 手动缩短到几百字，还是 429。

### 3) 关键发现：GitHub #47685

多番尝试检索之后，终于在 GitHub 上翻到了一个情况完全一致的 issue：`NousResearch/hermes-agent#47685`。

帖子里的复现极其干净：**同 key、同 endpoint、同 model、同请求长度**，唯一变量是 system prompt 的具体内容。当 prompt 包含精确短语 `Hermes Agent` 时，返回 429 / `code 1305`；把它替换成 `Hermes framework`（或任何其他词），立刻 200。

所以我猜测这不是限流。这是 Z.AI Coding Plan 后端对 system prompt 做了内容检测——可能是产品名/客户端指纹检测，也可能是某种 WAF 规则误伤。它只是披着 429（code 1305 overloaded）的皮。

---

## 再次排查：这个词到底藏在哪

### 1) 全局搜索

在 `~/.hermes` 下搜 `Hermes Agent`，50 多处命中：

- `SOUL.md` ← 身份定义
- `skills/**/SKILL.md` ← mandatory guidance + skill 描述
- `agent/system_prompt.py` ← 内置 guidance 常量
- `agent/prompt_builder.py` ← DEFAULT_AGENT_IDENTITY 等
- `sessions/*.sqlite` ← 已持久化的 session system_prompt

所以这不是通过改一个 Hermes Agent 的配置文件就能解决的。Hermes Agent 这个词从四面八方注入到最终 system prompt 里。

### 2) 为什么不能全库 sed 替换

最直觉的方案是 `sed -i 's/Hermes Agent/ZCode/g'`，但这条路有三个坑：

1. **污染技能库**：SOUL.md、skills 都是积累的内容，里面可能有精心调过的各种定义及引用。一刀切替换会破坏语义。
2. **污染历史 session**：已持久化的 session system_prompt 被改后，session resume 时会出现语义漂移。
3. **被版本更新覆盖**：Hermes 更新时这些文件被官方版本覆盖，改动静默丢失，bug 复发。

### 3) 换一个思路：改请求路径，不改磁盘

整体梳理 Hermes Agent 的 system prompt 构建流程是这样的：

```
build_system_prompt_parts()
  |-> stable:   SOUL.md + 工具指导 + skills index + 平台 hints
  |-> context:  AGENTS.md / .cursorrules / cwd 下的上下文文件
  |-> volatile: memory 快照 + USER.md + 时间戳
       |
       v
  三层用 \n\n 拼成一个字符串
       |
       v
  缓存到 agent._cached_system_prompt（整个 session 复用）
       |
       v
  发送给 provider
```

**与其在数据源头逐个替换，不如在最终组装好的字符串上、发送前做一次原地替换**。磁盘上的 SOUL.md / skills / session 全不动。

---

## 解决方案：双层修复

### 第一层：System Prompt 内容替换

在 `agent/system_prompt.py` 的 `build_system_prompt()` 返回前加一段：

```python
# Z.AI Coding Plan 的 glm-5.2 后端会检测 system prompt 里的精确短语
# "Hermes Agent"，命中则返回误导性 429/code 1305。在最终组装好的
# prompt 边界做替换，覆盖所有来源层（SOUL/skills/memory/guidance），
# 不改动磁盘上的文档、技能或历史 session。
provider_lower = str(getattr(agent, "provider", "") or "").lower()
model_lower = str(getattr(agent, "model", "") or "").lower()
if provider_lower == "zai" and "glm-5.2" in model_lower:
    joined = joined.replace("Hermes Agent", "ZCode")
```

#### 为什么用 provider + model 双重 gate？

- `provider == "zai"`：只有 Z.AI 直连才触发。OpenRouter 转发的 `zai-org/glm-5.2` 不受影响。
- `"glm-5.2" in model_lower`：只针对触发过滤的模型。glm-5.1 / glm-4.7 如果以后也有类似问题，加进来就行；现在不盲目扩大范围。

#### 为什么放在 `build_system_prompt()` 尾部？

**Hermes 的 prompt caching 是 sacred**——一次 session 里 system prompt 字节级稳定，上游 provider 的 KV cache 才能复用。任何 mid-conversation 的 prompt 变化都会击穿缓存。

`build_system_prompt()` 只在 session 初始化和 context compression 重建时被调用，都是合法的缓存边界。session 内部 turn-by-turn 循环直接读 `agent._cached_system_prompt`，不走这段逻辑，所以替换不影响 session 内的 cache 稳定性。

### 第二层：客户端指纹 Headers

光改 prompt 不够。Z.AI 后端还会看 HTTP headers 判断请求来源——是不是官方 ZCode 客户端发的。官方客户端带三个指纹头：

```
User-Agent: ZCode/0.14.8
X-ZCode-App-Version: 0.14.8
X-ZCode-Agent: glm
```

Hermes 默认带的是 OpenAI Python SDK 的 `User-Agent: OpenAI/Python ...` + 一堆 `X-Stainless-*`。这些 SDK 特征头可能被 Z.AI 的网关当作非官方客户端拒绝。

Hermes 本身已经有 per-provider header 注入机制（`_apply_client_headers_for_base_url`），Kimi、Copilot、NVIDIA NIM、Qwen Portal 都有自己的分支。依葫芦画瓢加一个 Z.AI 分支。

先写一个 header 工厂函数：

```python
def build_zcode_headers() -> dict:
    """Return the ZCode client fingerprint expected by Z.AI Coding Plan."""
    return {
        "User-Agent": "ZCode/0.14.8",
        "X-ZCode-App-Version": "0.14.8",
        "X-ZCode-Agent": "glm",
    }
```

然后在主 agent 请求路径注入：

```python
elif (
    getattr(self, "provider", None) == "zai"
    or base_url_host_matches(base_url, "api.z.ai")
    or base_url_host_matches(base_url, "open.bigmodel.cn")
):
    self._client_kwargs["default_headers"] = build_zcode_headers()
```

Auxiliary 路径（title 生成、context compression、vision 等）也要覆盖，一共四处：pool 解析、credential 解析、async 转换、custom endpoint resolution。每处加同样的 `elif` 分支。

---

## 整体验证

```bash
python -m pytest tests/agent/test_system_prompt.py \
  tests/run_agent/test_provider_attribution_headers.py -q

22 passed in 24.54s
```

**覆盖：**

- Z.AI glm-5.2 会把 `Hermes Agent` 替换成 `ZCode`
- 非 Z.AI provider 不替换
- Z.AI base_url 会注入 ZCode headers

**端到端：**

重启 Hermes，切 `zai/glm-5.2`，发消息。成功返回，无 429。

---

## 经验总结

### 1) 不是所有 429 都是限流

当一个 429 出现在特定 provider + 特定 model + 特定 prompt 内容的组合下，优先考虑内容过滤而不是额度。尤其是当 curl 裸调同 key 能通、完整请求却挂的时候，假 429 的概率远高于真限流。

### 2) 改请求路径，不改数据

磁盘上的 SOUL.md、skills、session 是用户的资产。运行时的请求是可变流量。全库 sed 替换会把修 bug 变成埋雷。找到请求组装的最后一层边界，在那里做最小侵入的改动。

### 3) Provider gate 是底线

任何 provider-specific 的 workaround 必须 gated。这次是 `provider == "zai"` + `"glm-5.2" in model`。全局改动会影响不相关的 provider，制造幽灵 bug。

### 4) Prompt caching invariant 不能破

任何 system prompt 的修改必须在构建边界做，不能在 turn 循环里做。Hermes 的 cache 稳定性是设计核心，破坏它等于每轮多烧一倍的 token。

### 5) 客户端指纹检测比你想的常见

从 Kimi（要求 `User-Agent: claude-code/0.1.0`）到 Z.AI（要求 ZCode headers），越来越多 provider 在网关层做客户端识别。遇到同 payload 不同客户端行为不同的情况，先查 HTTP headers。

---

## 参考文章

- **[Z.AI GLM-5.2 Coding Plan returns 429/code 1305 when system prompt contains exact phrase Hermes Agent](https://github.com/NousResearch/hermes-agent/issues/47685)** — Z.AI glm-5.2 system prompt 内容过滤的原始报告
- ZCode 官方客户端请求头样本：`User-Agent: ZCode/0.14.8` / `X-ZCode-App-Version: 0.14.8` / `X-ZCode-Agent: glm`
- Hermes `model.default_headers` 配置文档：`hermes config edit` → `model.default_headers` 字段

---

> 原文链接：https://deeprouter.org/article/hermes-agent-optimization-fix-glm-5-2-model-429-1305-overloaded
> 作者：Yawatasensei | 许可协议：[CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)
