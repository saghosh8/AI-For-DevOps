# Day 5 — Provider-Agnostic Integration Basics

---

## 1. Common Interface Across Providers
### 🧪 Practical

A **common interface** means every provider (Claude, GPT, Gemini) is called through the exact same function signature in your code — `ask(prompt)` — even though each provider's real SDK looks completely different underneath.

```mermaid
flowchart TD
    A["Your App Code"] --> B["Common Interface:\nask(prompt)"]
    B --> C["Provider A adapter\n(Claude)"]
    B --> D["Provider B adapter\n(GPT)"]
    B --> E["Provider C adapter\n(Gemini)"]
    C --> F["Same response shape\nreturned every time"]
    D --> F
    E --> F

    classDef blue fill:#DBEAFE,stroke:#3B82F6,color:#1F2937
    classDef purple fill:#EDE9FE,stroke:#8B5CF6,color:#1F2937
    classDef orange fill:#FFE8CC,stroke:#F97316,color:#1F2937
    classDef green fill:#D1FAE5,stroke:#10B981,color:#1F2937

    class A blue
    class B purple
    class C orange
    class D orange
    class E orange
    class F green
```

**DevOps example:** A Slack bot that explains errors shouldn't need a `if provider == "claude"` check scattered across every file that calls the model — the rest of your app just calls `ask(prompt)` and never needs to know or care which provider answered.

**🧪 Try it yourself:**
```python
from abc import ABC, abstractmethod

class LLMProvider(ABC):
    @abstractmethod
    def ask(self, prompt: str) -> str:
        """Every provider adapter must implement this exact method."""
        pass

class ClaudeProvider(LLMProvider):
    def ask(self, prompt: str) -> str:
        # a real call to the Anthropic API would go here
        return f"[Claude] Answer to: {prompt}"

class GeminiProvider(LLMProvider):
    def ask(self, prompt: str) -> str:
        # a real call to the Gemini API would go here
        return f"[Gemini] Answer to: {prompt}"

def run(provider: LLMProvider, prompt: str):
    print(provider.ask(prompt))

run(ClaudeProvider(), "Explain kubectl rollout status")
run(GeminiProvider(), "Explain kubectl rollout status")
```
```
1. Run this and notice both calls use the exact same
   run(provider, prompt) function.
2. The rest of your app only ever talks to LLMProvider -
   it has zero idea which real provider is underneath.
```

---

## 2. Abstracting SDK Differences
### 🧪 Practical

Every provider's real SDK returns data in a **different shape**. An adapter function's whole job is to hide that mess — take whatever weird structure the SDK returns, and translate it into your app's own simple format.

```mermaid
flowchart LR
    A["Provider A SDK call:\nclient.messages.create(...)"] --> C["Adapter function\ntranslates to common shape"]
    B["Provider B SDK call:\nclient.models.generate_content(...)"] --> C
    C --> D["Your app only ever\nsees ONE consistent\nfunction signature"]

    classDef orange fill:#FFE8CC,stroke:#F97316,color:#1F2937
    classDef purple fill:#EDE9FE,stroke:#8B5CF6,color:#1F2937
    classDef green fill:#D1FAE5,stroke:#10B981,color:#1F2937

    class A orange
    class B orange
    class C purple
    class D green
```

**DevOps example:** Anthropic's SDK nests the reply under `content[0].text` and the token count under `usage.output_tokens`, while Gemini's SDK exposes the reply directly as `response.text` and the token count under `usage_metadata.candidates_token_count` — without an adapter, every part of your app that reads the answer needs to know which provider it's talking to. With one, that messy detail lives in exactly one place.

**🧪 Try it yourself:**
```python
def call_claude_sdk(prompt):
    # pretend this is a real Anthropic-style SDK response
    return {"content": [{"text": f"Claude says: {prompt}"}],
            "usage": {"output_tokens": 12}}

def call_gemini_sdk(prompt):
    # pretend this is a real Gemini-style SDK response
    return {"text": f"Gemini says: {prompt}",
            "usage_metadata": {"candidates_token_count": 15}}

def claude_adapter(prompt):
    raw = call_claude_sdk(prompt)
    return {"text": raw["content"][0]["text"],
            "tokens": raw["usage"]["output_tokens"]}

def gemini_adapter(prompt):
    raw = call_gemini_sdk(prompt)
    return {"text": raw["text"],
            "tokens": raw["usage_metadata"]["candidates_token_count"]}

print(claude_adapter("status check"))
print(gemini_adapter("status check"))
```
```
1. Run this and compare the two print statements.
2. Even though call_claude_sdk() and call_gemini_sdk() return
   totally different shapes, both adapters output the SAME
   {text, tokens} structure - that's the abstraction at work.
```

---

## 3. Config-Driven Provider Switching
### 🧪 Practical

Instead of hardcoding which provider your app uses, you read it from a **config file**. Switching providers becomes a one-line config change — no code edits, no redeploy of app logic, just a different value.

```mermaid
flowchart TD
    A["config.yaml:\nprovider: 'claude'"] --> B["App reads config\nat startup"]
    B --> C{"Which provider?"}
    C --> D["Load Claude adapter"]
    C --> E["Load GPT adapter"]
    C --> F["Load Gemini adapter"]
    D --> G["App runs unchanged -\nonly config changed"]
    E --> G
    F --> G

    classDef blue fill:#DBEAFE,stroke:#3B82F6,color:#1F2937
    classDef purple fill:#EDE9FE,stroke:#8B5CF6,color:#1F2937
    classDef orange fill:#FFE8CC,stroke:#F97316,color:#1F2937
    classDef green fill:#D1FAE5,stroke:#10B981,color:#1F2937

    class A blue
    class B purple
    class C orange
    class D orange
    class E orange
    class F orange
    class G green
```

**DevOps example:** This is the exact same pattern as switching a Terraform backend or a Kubernetes context — you change one config value (a `provider:` field, an env var, a `--context` flag) and the same automation now targets something different, with zero code changes.

**🧪 Try it yourself:**
```python
import yaml

config_text = "provider: claude"
config = yaml.safe_load(config_text)

def ask_claude(prompt):
    return f"[Claude] {prompt}"

def ask_gemini(prompt):
    return f"[Gemini] {prompt}"

PROVIDERS = {"claude": ask_claude, "gemini": ask_gemini}

def ask_llm(prompt):
    fn = PROVIDERS[config["provider"]]
    return fn(prompt)

print(ask_llm("Summarize this deploy log"))
```
```
1. Run this - it should print the Claude version.
2. Change config_text to "provider: gemini" and run again.
3. Notice ask_llm() itself never changed - only the config
   value did. That's config-driven switching.
```

---

## 4. Normalizing Request/Response Formats
### 🧪 Practical

**Normalizing** means every provider's raw response gets converted into one standard internal shape — usually something like `{text, tokens_used, finish_reason}` — so the rest of your app only ever needs to understand ONE format, no matter which provider actually answered.

```mermaid
flowchart LR
    A["Provider A response:\ndifferent JSON shape"] --> C["Normalizer function"]
    B["Provider B response:\ndifferent JSON shape"] --> C
    C --> D["Standard internal shape:\ntext, tokens_used,\nfinish_reason"]
    D --> E["Rest of your app only\nknows this ONE shape"]

    classDef orange fill:#FFE8CC,stroke:#F97316,color:#1F2937
    classDef purple fill:#EDE9FE,stroke:#8B5CF6,color:#1F2937
    classDef green fill:#D1FAE5,stroke:#10B981,color:#1F2937
    classDef blue fill:#DBEAFE,stroke:#3B82F6,color:#1F2937

    class A orange
    class B orange
    class C purple
    class D green
    class E blue
```

**DevOps example:** A logging/cost-tracking pipeline that records `tokens_used` for every request only needs to work against ONE field name — not `output_tokens` for Claude and `candidates_token_count` for Gemini — because normalization already flattened that difference out.

**🧪 Try it yourself:**
```python
def normalize_claude_response(raw):
    return {
        "text": raw["content"][0]["text"],
        "tokens_used": raw["usage"]["output_tokens"],
        "finish_reason": raw.get("stop_reason", "unknown"),
    }

def normalize_gemini_response(raw):
    return {
        "text": raw["text"],
        "tokens_used": raw["usage_metadata"]["candidates_token_count"],
        "finish_reason": raw.get("finish_reason", "unknown"),
    }

claude_raw = {"content": [{"text": "pod restarted"}],
              "usage": {"output_tokens": 3}, "stop_reason": "end_turn"}
gemini_raw = {"text": "pod restarted",
              "usage_metadata": {"candidates_token_count": 3}, "finish_reason": "STOP"}

print(normalize_claude_response(claude_raw))
print(normalize_gemini_response(gemini_raw))
```
```
1. Run this and compare the two printed dictionaries.
2. Both have the exact same keys (text, tokens_used,
   finish_reason) even though the raw inputs looked
   nothing alike.
```

---

## 5. Fallback Between Providers
### 🧪 Practical

**Fallback** means if your primary provider fails (outage, rate limit, timeout), your app automatically retries the same request against a backup provider — instead of the whole feature just breaking.

```mermaid
flowchart TD
    A["Call Primary Provider"] --> B{"Success?"}
    B --> C["Return the answer"]
    B --> D["Call Backup Provider"]
    D --> E{"Success?"}
    E --> C
    E --> F["Return a clear\nerror to the caller"]

    classDef blue fill:#DBEAFE,stroke:#3B82F6,color:#1F2937
    classDef purple fill:#EDE9FE,stroke:#8B5CF6,color:#1F2937
    classDef green fill:#D1FAE5,stroke:#10B981,color:#1F2937
    classDef red fill:#FEE2E2,stroke:#EF4444,color:#1F2937

    class A blue
    class B purple
    class C green
    class D blue
    class E purple
    class F red
```

**DevOps example:** An on-call assistant that depends on a single provider becomes a single point of failure during an incident — if that provider has an outage at the exact moment you need it most, a fallback provider keeps the assistant working instead of going dark.

**🧪 Try it yourself:**
```python
def call_primary(prompt):
    raise ConnectionError("Primary provider is down")  # simulate an outage

def call_backup(prompt):
    return f"[Backup] {prompt}"

def ask_with_fallback(prompt):
    try:
        return call_primary(prompt)
    except Exception as e:
        print(f"Primary failed ({e}), falling back...")
        return call_backup(prompt)

print(ask_with_fallback("Explain ImagePullBackOff"))
```
```
1. Run this - the primary call is set up to always fail.
2. Notice the app doesn't crash - it catches the failure
   and automatically retries with the backup provider.
```

---

## 6. Vendor Lock-In Risks
### 📖 Theory

**Vendor lock-in** happens when your app is written so tightly around one provider's specific SDK, response shapes, and quirks that switching providers later means rewriting large chunks of code — not just changing a config value.

```mermaid
flowchart TD
    A["App hardcoded to\none provider's SDK"] --> B["Provider raises prices\nor has an outage"]
    B --> C["Switching providers\nmeans rewriting\nlarge parts of the app"]

    D["App built on a\ncommon interface"] --> E["Provider raises prices\nor has an outage"]
    E --> F["Switching = change\none config value"]

    classDef red fill:#FEE2E2,stroke:#EF4444,color:#1F2937
    classDef orange fill:#FFE8CC,stroke:#F97316,color:#1F2937
    classDef blue fill:#DBEAFE,stroke:#3B82F6,color:#1F2937
    classDef green fill:#D1FAE5,stroke:#10B981,color:#1F2937

    class A red
    class B orange
    class C red
    class D blue
    class E orange
    class F green
```

**DevOps example:** This is the same lock-in risk as building a pipeline entirely around one cloud provider's proprietary APIs instead of portable tools — it's not that using a single provider is wrong, it's that building *without any abstraction layer* is what makes switching later expensive and risky.

**Why it matters even if you don't pick the provider:** As we discussed for the old Day 5 topic — the business usually decides *which* provider to use, not the engineer. But *how tightly your code depends on that one provider's SDK* is entirely something you control, and it's what determines whether a future provider change is a one-line config edit or a multi-week rewrite.

---

## Quick Recap (Day 5)

```mermaid
flowchart TD
    A["Common Interface: one function\nsignature for every provider"] --> B["Abstracting SDK Differences:\nhide each SDK's messy shape"]
    B --> C["Config-Driven Switching:\nchange provider via config,\nnot code"]
    C --> D["Normalizing Formats: every\nresponse becomes one shape"]
    D --> E["Fallback Between Providers:\nbackup provider on failure"]
    E --> F["Vendor Lock-In Risks: tight\nSDK coupling = costly to switch"]

    classDef purple fill:#EDE9FE,stroke:#8B5CF6,color:#1F2937
    classDef orange fill:#FFE8CC,stroke:#F97316,color:#1F2937
    classDef blue fill:#DBEAFE,stroke:#3B82F6,color:#1F2937
    classDef green fill:#D1FAE5,stroke:#10B981,color:#1F2937
    classDef red fill:#FEE2E2,stroke:#EF4444,color:#1F2937

    class A purple
    class B orange
    class C blue
    class D orange
    class E green
    class F red
```

---

## ⭐ Support

If you found this repository useful:

<a href="https://github.com/saghosh8/AI-For-DevOps">
  <img src="https://img.shields.io/github/stars/saghosh8/AI-For-DevOps?style=for-the-badge&logo=github&logoColor=white&label=STAR%20THIS%20REPO" />
</a>
<a href="https://github.com/saghosh8/AI-For-DevOps/fork">
  <img src="https://img.shields.io/github/forks/saghosh8/AI-For-DevOps?style=for-the-badge&logo=github&label=FORK" />
</a>

---

## 💬 Have a Query

Have a question, suggestion, or idea?

<a href="https://github.com/saghosh8/AI-For-DevOps/discussions/3">
  <img src="https://img.shields.io/badge/💬%20JOIN%20DISCUSSION-6366f1?style=for-the-badge&logo=github&logoColor=white" />
</a>

---

| 📘 Next — Day 6: AI Application Architecture | <a href="https://github.com/saghosh8/AI-For-DevOps/blob/main/Week%201%20%E2%80%94%20LLM%20%26%20GenAI%20Foundations/Day%206%20%E2%80%94%20AI%20Application%20Architecture.md"><img src="https://img.shields.io/badge/NEXT%20DAY-0ea5e9?style=for-the-badge&logo=github&logoColor=white" /></a> |
| ------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
