# LLM Observability with LangSmith & Langfuse

A hands-on guide to adding **observability, tracing, and monitoring** to LLM apps — using the [AI Code Review Agent](https://github.com/aswathh/AI-Code-Review-Agent) (Streamlit + Groq + LangChain) as the working example.

> **Why observability?** Once your app has multiple chained LLM calls (like this agent's 4 tools: bug detection, security audit, test generation, docs), it becomes a black box. Observability tools let you see every prompt, response, token count, latency, and error — nested in a trace tree — so debugging isn't guesswork.

---

## Table of Contents

- [What is LangSmith?](#what-is-langsmith)
- [What is Langfuse?](#what-is-langfuse)
- [LangSmith vs Langfuse — Quick Comparison](#langsmith-vs-langfuse--quick-comparison)
- [Setup: LangSmith](#setup-langsmith)
- [Setup: Langfuse](#setup-langfuse)
- [Sample Code: Before vs After](#sample-code-before-vs-after)
- [What You Can Do With These Tools](#what-you-can-do-with-these-tools)
- [Common Pitfalls](#common-pitfalls)
- [Security Note](#security-note)

---

## What is LangSmith?

[LangSmith](https://smith.langchain.com) is LangChain's native observability + evaluation platform. Because it's built by the same team as LangChain, tracing is often **zero-code** — just set env vars and every chain, tool, and LLM call is automatically captured.

**Core capabilities:**
- **Tracing** — nested trace trees for every chain/agent run
- **Evaluation** — run test datasets against your chains, score outputs (LLM-as-judge or custom)
- **Debugging** — inspect exact prompts/responses at each step, replay failed runs
- **Monitoring** — latency, token usage, cost dashboards over time
- **Prompt playground** — iterate on prompts directly in the UI, no redeploy needed
- ️ **Datasets** — curate examples from production traces for regression testing

---

## What is Langfuse?

[Langfuse](https://langfuse.com) is an open-source (self-hostable) observability platform, framework-agnostic — works with LangChain, raw SDKs (Groq, OpenAI, Anthropic), or anything else via decorators.

**Core capabilities:**
- **Tracing** — via `@observe()` decorator or LangChain callback handler
- **Session/user tracking** — group traces by Streamlit session or user ID
- **Cost & usage analytics** — per-model, per-user token/cost breakdown
- ‍️ **Evaluation** — scores, human annotation queues, LLM-as-judge
- ️ **Prompt management** — version and fetch prompts from Langfuse instead of hardcoding
- **Self-hostable** — run it on your own infra (Docker) if data can't leave your network
- **Open source** — full transparency into how tracing works

---

## ️ LangSmith vs Langfuse — Quick Comparison

| | LangSmith | Langfuse |
|---|---|---|
| Best fit | Pure LangChain/LangGraph apps | Any stack (LangChain, raw SDKs, mixed) |
| Hosting | Cloud only | Cloud **or** self-hosted (open source) |
| Setup effort | Lowest (auto-traces LangChain) | Low (decorator or callback) |
| Prompt management | Playground UI | Prompt versioning + fetch API |
| Pricing model | Free tier, then usage-based | Free tier (generous), self-host = free |
| Vendor lock-in | Tied to LangChain ecosystem | Framework-agnostic |

**Rule of thumb:** if you're 100% LangChain and want the path of least resistance → LangSmith. If you want portability, self-hosting, or you mix raw SDK calls with LangChain → Langfuse. Many teams run **both** during migration or for redundancy.

---

## ️ Setup: LangSmith

```bash
uv add langsmith
```

`.env`:
```env
LANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY=your_langsmith_key
LANGCHAIN_PROJECT=ai-code-review-agent
```

That's it for LangChain-based chains — tracing is automatic once these are set.

---

## ️ Setup: Langfuse

```bash
uv add langfuse
```

`.env`:
```env
LANGFUSE_PUBLIC_KEY=pk-lf-...
LANGFUSE_SECRET_KEY=sk-lf-...
LANGFUSE_HOST=https://cloud.langfuse.com
```

---

## Sample Code: Before vs After

### Before (no observability)

```python
def detect_bugs(code: str, language: str = "Python") -> str:
prompt = ChatPromptTemplate.from_messages([
("system", "You are an expert code reviewer..."),
("human", "Find bugs in this {language} code:\n```{language}\n{code}\n```")
])
return (prompt | get_llm() | StrOutputParser()).invoke(
{"language": language, "code": code}
)
```

### After (Langfuse + LangChain)

```python
from langfuse.callback import CallbackHandler
from langfuse.decorators import observe

langfuse_handler = CallbackHandler()

@observe(name="detect_bugs")
def detect_bugs(code: str, language: str = "Python") -> str:
prompt = ChatPromptTemplate.from_messages([
("system", "You are an expert code reviewer..."),
("human", "Find bugs in this {language} code:\n```{language}\n{code}\n```")
])
return (prompt | get_llm() | StrOutputParser()).invoke(
{"language": language, "code": code},
config={"callbacks": [langfuse_handler]}
)
```

### After (LangSmith)

```python
# No code change needed for LangChain calls —
# just the env vars above. Optional: name runs explicitly
from langsmith import traceable

@traceable(name="detect_bugs")
def detect_bugs(code: str, language: str = "Python") -> str:
...
```

### Wrapping the orchestrator (nests all 4 tools under one trace)

```python
@observe(name="run_agent")
def run_agent(code: str, language: str = "Python", **flags) -> dict:
results = {}
if flags.get("run_bugs"): results["bugs"] = detect_bugs(code, language)
if flags.get("run_security"): results["security"] = check_security(code, language)
if flags.get("run_tests"): results["tests"] = generate_tests(code, language)
if flags.get("run_docs"): results["documentation"] = generate_docs(code, language)
return results
```

One Streamlit button click → one parent trace → 4 nested tool spans, each with its own LLM generation span underneath.

---

## What You Can Do With These Tools

- **Debug a bad output** — click into the exact trace, see the exact prompt sent to Groq, spot a bad variable substitution
- **Catch silent failures** — a tool returns empty string? Trace shows the LLM call, response, and any parser errors
- **Track cost** — see token usage per tool, per user session, per day — useful once this agent has real traffic
- **A/B test prompts** — change the system prompt for `check_security`, compare trace outputs side by side
- **Build a regression dataset** — save traces where the agent found real bugs as a golden dataset, re-run on every prompt change
- **Latency profiling** — see which of the 4 tools is slowest, optimize or parallelize accordingly
- **Session replay** — reconstruct what a user submitted and what the agent returned, end to end

---

## ️ Common Pitfalls

- **API key ≠ instrumentation.** Setting env vars alone sends nothing — you still need `@observe`/`@traceable` or the callback handler in your actual code.
- **Forgetting the parent wrapper.** Without wrapping `run_agent`, you'll get 4 disconnected traces instead of one tree — much harder to read.
- **Recreating the LLM client per call.** `get_llm()` is called fresh in every function — fine for correctness, but consider caching (`@lru_cache`) to cut overhead.
- **Committing `.env`.** Never push API keys to GitHub — see the security note below.

---

## Security Note

Never commit `.env` files. Add to `.gitignore`:
```
.env
```
Provide a `.env.example` instead with placeholder values so contributors know what's needed:
```env
GROQ_API_KEY=your_key_here
LANGCHAIN_API_KEY=your_langsmith_key
LANGFUSE_PUBLIC_KEY=pk-lf-...
LANGFUSE_SECRET_KEY=sk-lf-...
```
If a key is ever exposed, rotate it immediately — removing it from git history doesn't undo the exposure.

---

## Resources

- [LangSmith Docs](https://docs.smith.langchain.com)
- [Langfuse Docs](https://langfuse.com/docs)
- [Langfuse GitHub (self-hosting)](https://github.com/langfuse/langfuse)
