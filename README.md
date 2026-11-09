# 🔐 AD Attack Path Analyzer

**Automated Active Directory attack path analysis, powered by graph theory and LLMs.**

*Cybersecurity capstone project.*

AD Attack Path Analyzer takes a **SharpHound** export (the collector used by [BloodHound](https://github.com/SpecterOps/BloodHound)), analyzes it **fully offline**, models Active Directory as a directed graph, finds the shortest attack paths from a compromised account to high-value targets, and then asks an **LLM** to explain each path in plain language and propose **Blue Team remediations**.

> ⚠️ **Legal & ethical scope**
> Only use this tool against Active Directory environments you are **explicitly authorized to audit** (a personal lab, a contracted pentest engagement, a CTF). SharpHound itself is an intrusive collection tool, and using it outside an authorized scope is illegal in most jurisdictions.
> This notebook only analyzes data that has **already been collected**. It performs no offensive action of its own.

---

## ✨ Features

- 📦 **In-memory parsing**: the SharpHound `.zip` is read entirely in memory, with zero disk extraction.
- 🐘 **Streaming JSON parsing**: uses [`ijson`](https://github.com/ICRAR/ijson) (optional) to handle exports of several hundred MB without loading them fully into memory, and falls back to `json.load` otherwise.
- 🧯 **Resilient pipeline**: a corrupted file or a malformed object is skipped with a warning and never crashes the whole run.
- 🕸️ **Graph modeling** with [NetworkX](https://networkx.org/) (`MultiDiGraph`), including self-loop and isolated-node cleanup.
- 🎯 **High-value target detection** via the BloodHound `highvalue` flag, with a fallback on well-known RIDs (`-512`, `-519`, `-516`, `-498`, `-544`).
- 🗣️ **Natural-language translation** of graph paths into readable paragraphs.
- 🤖 **LLM-based Red/Blue Team analysis** with a strict JSON schema: technique, MITRE ATT&CK ID, attack mechanics, offensive tools, remediation, SOC detection ideas and risk level.
- 🔌 **Multi-provider support**: Groq, Anthropic Claude, OpenAI, an offline `mock` mode, and an optional LangChain wrapper.
- 🧮 **Token-budget management**: several short paths are batched into a single LLM call, with automatic fallback to per-path calls if a batch response is invalid.
- 📊 **Export & visualization**: CSV/JSON results and a PNG subgraph of the detected attack paths.

---

## 🧭 Pipeline

| # | Step | Description |
|---|------|-------------|
| 1 | **Configuration** | Imports, API keys, global parameters |
| 2 | **Extraction & Parsing** | In-memory reading of the `.zip`, extraction of objects and critical relationships |
| 3 | **Graph modeling** | Building and cleaning a directed graph, finding shortest paths to high-value targets |
| 4 | **Natural-language translation** | Turning each graph path into a readable paragraph |
| 5 | **LLM analysis (Red/Blue Team)** | Structured explanation of each attack and remediation advice |
| 6 | **Export & visualization** | CSV/JSON export and subgraph plot |

### Relationships extracted

| Relationship | JSON source | Offensive meaning |
|---|---|---|
| `GenericAll` / `WriteDacl` / `ForceChangePassword` | `Aces[]` of the target object | ACL abuse: full control, permission changes, password reset |
| `AdminTo` | `LocalAdmins.Results[]` of computers | Local administrator rights on a machine |
| `HasSession` | `Sessions.Results[]` of computers | Active user session on a machine (potential credential theft) |
| `AllowedToDelegate` | `AllowedToDelegate[]` of users/computers | Kerberos constrained delegation |
| `MemberOf` | `Members[]` of groups | Group membership (privilege inheritance chain) |

The monitored ACE rights are controlled by the `RELEVANT_ACE_RIGHTS` set and can easily be extended (e.g. `GenericWrite`, `Owns`, `WriteOwner`, `AddMember`, `AllowedToAct`).

---

## 🚀 Getting Started

### Prerequisites

- Python 3.9+
- Jupyter / JupyterLab, or Google Colab
- A **real SharpHound export** (`.zip`) collected from an environment you are authorized to audit
- *(Optional)* an API key for an LLM provider

### Installation

```bash
git clone https://github.com/<your-username>/<your-repo>.git
cd <your-repo>
pip install networkx pandas ijson matplotlib openai anthropic
```

> The `openai` package covers both OpenAI and Groq (OpenAI-compatible endpoint). `ijson` is optional but strongly recommended for large exports.

Optional LangChain integration:

```bash
pip install langchain langchain-anthropic langchain-openai langchain-groq
```

### Configuration

Edit the `CONFIG` dictionary in the notebook:

```python
CONFIG = {
    "SHARPHOUND_ZIP_PATH": "sharphound_export.zip",       # path to your SharpHound export (required)
    "COMPROMISED_PRINCIPAL_HINT": "your_owned_account",   # name (substring) or SID of the "owned" account
    "LLM_PROVIDER": "groq",                               # "auto" | "anthropic" | "openai" | "groq" | "mock"
    "ANTHROPIC_MODEL": "claude-sonnet-5-5",
    "OPENAI_MODEL": "gpt-4o-mini",
    "GROQ_MODEL": "llama-3.3-70b-versatile",
    "MAX_TOKENS_PER_LLM_CALL": 3000,                      # token budget per LLM call (batching)
    "MAX_PATH_LENGTH": 8,                                 # maximum number of hops per attack path
}
```

> 💡 SharpHound does **not** store which accounts are "owned" (that is a BloodHound GUI concept). You must provide the starting point manually through `COMPROMISED_PRINCIPAL_HINT`.

### API keys

Keys are read from environment variables. **Never hard-code them** in the notebook or commit them to Git.

```bash
# Linux / macOS
export GROQ_API_KEY="your_key"
export ANTHROPIC_API_KEY="your_key"
export OPENAI_API_KEY="your_key"

# Windows
setx GROQ_API_KEY "your_key"
```

On **Google Colab**, add a secret named `GROQ_API_KEY` in the 🔑 *Secrets* tab and grant the notebook access to it.

> ⚠️ If a key has ever been pasted in plain text anywhere (chat, screenshot, shared code), treat it as compromised and regenerate it.

### Run

1. Place your SharpHound export at the path defined in `CONFIG["SHARPHOUND_ZIP_PATH"]`.
2. Set `COMPROMISED_PRINCIPAL_HINT` to your scenario's compromised account.
3. Export your API key (or use `LLM_PROVIDER = "mock"` to test without any key).
4. Open the notebook and choose **Restart & Run All**.

If the export is missing, the notebook raises a clear `FileNotFoundError`. There is intentionally **no built-in sample data**.

---

## 🤖 LLM Providers

| Provider | Value | Requirement |
|---|---|---|
| Groq (free tier, very low latency) | `groq` | `GROQ_API_KEY`; automatically falls back to `mock` if initialization fails |
| Anthropic Claude | `anthropic` | `ANTHROPIC_API_KEY` |
| OpenAI | `openai` | `OPENAI_API_KEY` |
| Mock (offline, simulated responses) | `mock` | None, for pipeline testing only |
| Auto-detect | `auto` | Picks the first available key (Anthropic → OpenAI → Groq), else `mock` |

Each path is analyzed with a strict system prompt that forces a JSON response:

```json
{
  "primary_technique": "...",
  "mitre_attack": "...",
  "attack_mechanics": "...",
  "offensive_tools": ["..."],
  "remediation": ["..."],
  "soc_detection": ["..."],
  "risk_level": "Critical | High | Medium | Low"
}
```

---

## 📁 Output

Results are written to the `outputs/` folder:

| File | Content |
|---|---|
| `attack_path_analysis.json` | Full LLM analysis of each attack path |
| `attack_path_analysis.csv` | Same data in tabular form |
| `attack_path_subgraph.png` | Visualization of the subgraph formed by the detected attack paths |

---

## ⚠️ Known Limitations

- **Objects not modeled**: `PrimaryGroupSID`, OUs / GPOs / Containers (GPO inheritance paths). `AllowedToAct` (RBCD) is supported by the translation dictionary but not extracted by default.
- **High-value target detection** depends on the collector's `highvalue` flag (absent from very old SharpHound versions) and a RID fallback list. Always double-check detected targets.
- **"Owned" is not native to SharpHound**: the starting account must be supplied manually.
- **LLM reliability**: outputs (tools, techniques, remediations) **must be reviewed by a human** before any action. LLMs can be wrong or hallucinate technical details.
- **Rate limiting**: error handling is intentionally simple (one retry with light backoff). For heavy use, implement exponential backoff and respect your provider's limits (especially Groq's free tier).
- **No sample dataset** is bundled: a real SharpHound export is required.

---

## 🛠️ Tech Stack

[Python](https://www.python.org/) · [NetworkX](https://networkx.org/) · [pandas](https://pandas.pydata.org/) · [ijson](https://github.com/ICRAR/ijson) · [Matplotlib](https://matplotlib.org/) · [Groq](https://console.groq.com/) / [Anthropic](https://www.anthropic.com/) / [OpenAI](https://openai.com/) · [LangChain](https://www.langchain.com/) *(optional)*

---

## 📜 Disclaimer

This project is provided for **educational and authorized security assessment purposes only**. The author assumes no liability for misuse. Always obtain written authorization before auditing any Active Directory environment.

## 📄 License

Amine DIOURI
