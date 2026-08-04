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

**A tool-calling model.** The notebook uses `langchain-openai` and reads
`OPENAI_API_KEY` (plus `OPENAI_BASE_URL` if you route through a gateway) from the
environment. Any tool-calling chat model works — swap the one line in the setup
cell.

## Setup

```bash
uv sync
cp .env.example .env     # fill in your resource coordinates
az login
```

`.env` holds resource coordinates only — it is gitignored, and no credential
belongs in it. Auth is `DefaultAzureCredential`, which shells out to the Azure
CLI. If you keep multiple CLI profiles, select one **before** launching Jupyter:

```bash
export AZURE_CONFIG_DIR=~/.config/azure/<profile>
az account show --query "{sub:name, user:user.name}" -o tsv
```

Then:

```bash
export OPENAI_API_KEY=...
uv run jupyter lab aca_deepagents.ipynb
```

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
