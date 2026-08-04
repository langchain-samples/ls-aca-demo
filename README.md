# Azure Container Apps as a sandbox for Deep Agents

A teaching notebook for running agent-authored code on Azure Container Apps,
using [`langchain-azure-container-apps`](https://github.com/langchain-ai/langchain-azure/tree/main/libs/azure-container-apps).

[`aca_deepagents.ipynb`](aca_deepagents.ipynb) covers both ACA compute products,
in order:

1. **Dynamic sessions** — `SessionsPythonREPLTool`, `SessionsBashTool`, and the
   `SessionsBashBackend` Deep Agents backend.
2. **Sandboxes** — `ACASandbox`, the stateful Deep Agents backend: stop/resume,
   `pip install` that survives, snapshots, and teardown.
3. **Choosing between them** — they are different products, not tiers.

The committed notebook has real outputs from a live run against Azure.

## Prerequisites

**Azure resources** — a **Python**-typed session pool, a **Shell**-typed session
pool, and a **sandbox group**.

**Role assignments** for the identity you `az login` with:

| Resource | Role |
|---|---|
| Session pools | `Azure ContainerApps Session Executor` |
| Sandbox group | `Container Apps SandboxGroup Data Owner` |

**A tool-calling model.** The notebook uses `langchain-openai`. Any tool-calling
chat model works — swap the one line in the model cell.

## Setup

```bash
uv sync
cp .env.example .env     # then fill it in
az login
uv run jupyter lab aca_deepagents.ipynb
```

### Put everything in `.env` — including the model key

**Exporting variables in a terminal is not enough.** VS Code, and any editor
launched from Finder or the Dock, spawns the notebook kernel itself and inherits
none of your shell configuration — no `~/.zshrc`, no fish `conf.d`. The notebook
calls `load_dotenv()`, which reads `.env` and nothing else.

`.env` is gitignored and never leaves your machine; the committed history has
been checked for this.

Two entries are easy to miss, and each fails somewhere different:

| Variable | Missing it breaks |
|---|---|
| `OPENAI_API_KEY` | the model cell, immediately |
| `AZURE_CONFIG_DIR` | **every Azure cell**, with an opaque `ClientAuthenticationError` |

`AZURE_CONFIG_DIR` selects which `az login` profile `DefaultAzureCredential`
uses. Omit it and you get whichever profile was default — often a different
tenant, which fails to get a token at all rather than reporting a wrong account.
It must be an **absolute path**; `dotenv` does not expand `~`. Verify with:

```bash
AZURE_CONFIG_DIR=/Users/you/.config/azure/<profile> \
  az account show --query "{sub:name, user:user.name}" -o tsv
```

### If Azure cells fail with `ClientAuthenticationError`

Three unrelated problems produce that same error. The notebook's preflight cell
distinguishes them:

1. **`az` is not on the kernel's `PATH`.** A GUI-launched editor gets the system
   `PATH`, which excludes Homebrew — so `/opt/homebrew/bin/az` is invisible.
   Launch from a terminal (`code .`) or let the preflight cell patch `PATH`.
2. **Wrong `az` profile** — set `AZURE_CONFIG_DIR`.
3. **Not logged in** — `az login`.

## Cost

Part 2 **provisions a real Azure sandbox that bills until it is deleted.** The
final cell deletes the sandbox and its snapshot, and reports any orphan it
created. It only ever deletes what the notebook made — anything already in the
group is recorded up front and left alone.

If a run dies partway, sweep the group by hand:

```python
for sandbox in group.list_sandboxes():
    print(sandbox.id, sandbox.state, sandbox.created_at)
```

Dynamic sessions need no cleanup: sessions are pool-managed and expire on their
own.

## The unreleased dependency

`langchain-azure-container-apps` is not on PyPI yet. Both Deep Agents backends
and the entire sandboxes integration exist only on the
`feat/aca-06-deprecate-dynamic-sessions` branch, so `pyproject.toml` pins a git
source:

```toml
[tool.uv.sources]
langchain-azure-container-apps = { git = "https://github.com/langchain-ai/langchain-azure.git", branch = "feat/aca-06-deprecate-dynamic-sessions", subdirectory = "libs/azure-container-apps" }
```

Once the package ships, delete that block — the `[project].dependencies` entry
already reads as a normal PyPI requirement.

The published `langchain-azure-dynamic-sessions` is the older package. It is
deprecated in favour of this one, has no Deep Agents backends, and still carries
the dynamic-sessions bugs this one fixes.
