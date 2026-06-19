# GuideAnts JobSearch — Handoff & Context

> Lives on the `mydocs` orphan branch (private to my fork, syncs across machines).
> Written 2026-06-19 on the MacBook Pro, to pick up on the MacMini.

## Where things stand (git setup, done today)

- This is a **fork**: `origin` = `scottkollarik/GuideAnts`, `upstream` = `Elumenotion/GuideAnts`.
- `main` is fast-forward-synced to `upstream/main` and pushed. Keep it a clean mirror — never commit to it.
- **Three worktrees**, grouped under `Worktrees/GuideAnts/` (group by repo, not branch type):
  - `main` (the main clone) — pristine mirror for `--ff-only` syncs
  - `mydocs` — **orphan** branch (no code, no shared history); private notes + this doc; pushed to `origin`, never PR'd upstream
  - `jobsearch-console` — `feature/jobsearch-console`, pushed to `origin`, empty so far
- License is **Apache 2.0** — permissive. Free to self-host, build a closed-source client, and commercialize. Only forward risk is open-core relicensing of *future* upstream versions (current snapshot stays Apache forever).

## The big-picture strategy we settled on

1. **Consumption first, no core changes.** Treat GuideAnts as a headless backend; never fork core in phase 1, so `main` keeps fast-forwarding from upstream cleanly. Propose core changes (e.g. a daily feed poller in `GuideAntsApi.BackgroundJobs`) to the proprietor only if phase 1 proves out.
2. **Two API tiers** (verified in `src/server/GuideAntsApi/Endpoints/`):
   - *Public / stable*: `PublishedGuides`, `PublishedNotebookConversations`, `Catalog`, plus an **MCP surface** for published guides (added ~2026-06-15). Governed (auth + cost limits). Build the storefront on this.
   - *Internal app API*: Projects/Notebooks/Files/Guides authoring — powerful but uncontracted; coupling risk on a fork.
   - **OpenAPI/Swagger is present** → generate a typed TS client; regenerate after upstream syncs and let the compiler flag breaks.
3. **Build our own client** (storefront/catalog of *my* published guides) against the public tier. Zero core changes, upgrade-safe, fully my UX.
4. **Self-host on Azure later** (Container Apps, scale-to-zero per cost policy; data on Azure SQL/Postgres; AI routed to Azure OpenAI or local box). Phase 1 is local Docker on the MacMini first.
5. **Use case #1 = job search** to reduce LinkedIn dependency: a guide that uses the built-in web-search + read-web tools (SearXNG-backed) + Docling to discover RSS/ATS career feeds for Atlanta employers.

## Immediate next step (on the MacMini)

Stand up **local Docker + local inference**, use the **default UI**, and explore real capabilities before building anything. Goal: see whether the pitch holds up hands-on.

## Open investigations for "the other side"

- **ARM64 / Metal build.** I worked on the docker compose ~a year ago and PR'd it to a *different* repo of the proprietor's — unclear if it landed here. Check whether `docker/` builds cleanly on Apple Silicon and uses Metal for local inference. If not, that's the first fix.
- **Local inference endpoint goal** (also in this machine's Claude memory, which does NOT travel — recorded here so it survives): host AI inference on a dedicated "special" machine, expose via **dynamic DNS** through the router/ISP device, and point GuideAnts' per-capability provider routing at it. Route cheap/bulk work (embeddings, parsing) local; frontier only where a small model demonstrably won't do. Add **TLS** and lock the endpoint to my instance only.

## Commands to run on the MacMini

```bash
# 1. Get the repo (if not already cloned). Mirror the ~/Documents/GitHub/!projects layout.
git clone https://github.com/scottkollarik/GuideAnts.git
cd GuideAnts

# 2. Add upstream (a fresh clone only has origin)
git remote add upstream https://github.com/Elumenotion/GuideAnts.git
git fetch --all

# 3. Recreate the worktrees (adjust the base path to match this machine)
git worktree add ../Worktrees/GuideAnts/mydocs mydocs
git worktree add ../Worktrees/GuideAnts/jobsearch-console feature/jobsearch-console

# 4. Read this doc + notes
#    ../Worktrees/GuideAnts/mydocs/HANDOFF.md  (this file)
#    ../Worktrees/GuideAnts/mydocs/GIT-WORKFLOW.md

# 5. Stand up local GuideAnts (macOS). See docs/setup-guide.md + docs/local-ai-setup-guide.md
./quickstart.sh        # or inspect docker/ + `docker compose` lanes first
```

## Starter prompt for the next Claude Code session (MacMini)

> I'm continuing the GuideAnts JobSearch project on my MacMini. Read
> `Worktrees/GuideAnts/mydocs/HANDOFF.md` for full context. Today's goal:
> stand up GuideAnts locally with Docker + local inference and explore
> capabilities using the default UI. First, audit `docker/` and the compose
> setup for Apple Silicon (ARM64) + Metal support — I PR'd ARM work to a
> different repo of the proprietor's ~a year ago and don't know if it landed
> here. Tell me what builds, what doesn't, and the root cause before changing
> anything. Work on the `feature/jobsearch-console` branch worktree.

## Roadmap (to flesh out — deferred)

- [ ] Phase 0: local Docker + default UI working on Apple Silicon w/ Metal
- [ ] Phase 1a: build "Atlanta Job Feed Builder" guide in a notebook; validate discovery
- [ ] Phase 1b: publish it; consume via MCP and/or HTTP invoke API from a thin client
- [ ] Phase 1c: custom storefront/catalog client (TS, generated from OpenAPI) on the public tier
- [ ] Phase 2: Azure self-host (Container Apps, scale-to-zero); local-inference box via DDNS
- [ ] Phase 2+: propose a BackgroundJobs feed poller upstream to the proprietor (if proven)

---

## UPDATE — Mac local inference RESOLVED (supersedes the ARM64/Metal investigation above)

**Root cause (settled):** Docker on macOS runs all containers inside a *Linux VM* (Apple
Virtualization.framework). Metal is a macOS-only API and there is **no GPU passthrough** to the
guest. So **no container on a Mac can use Metal — ARM64 or AMD64, ever.** It's CPU-only in Docker,
full stop. (`start_macos.sh:157-158` forces `DOCKER_DEFAULT_PLATFORM=linux/amd64`, and every
service hardcodes `platform: linux/amd64`, because the GHCR images are AMD64-only. That AMD64
emulation only slows the *supporting* services; it is NOT the thing blocking Metal.)

**The correct architecture (decided):** Route inference to wherever the GPU lives *natively*.
On Mac that means **host-native Ollama** (Metal-accelerated), with GuideAnts' containers talking to
it over the OpenAI-compatible protocol. GuideAnts is provider-routed (Anthropic/Azure/OpenAI/
OpenRouter/HuggingFace/Llama, per-capability), so this is first-class, not a hack. Same
OpenAI-compatible seam scales later to the DDNS "special machine" or cloud.

**Validated config** (`OpenAiEmbeddingService.cs:48` reads `OpenAI:Endpoint`, default
api.openai.com/v1 — overridable):
```jsonc
"OpenAI": {
  "Endpoint": "http://host.docker.internal:11434/v1",  // native Ollama (Metal)
  "ApiKey": "ollama"                                    // dummy; client wants non-empty
}
```
Then route **chat** + **embeddings** to the OpenAI provider in Settings (likely editable in the
default UI — settings-editor layer exists: ProviderConfigurationResolver, SettingsEndpoints).

**Gotchas:** (1) embeddings hit `{baseUrl}/embeddings` → need an embedding model pulled
(`ollama pull nomic-embed-text`), separate from the chat model (`ollama pull llama3.1`);
(2) `host.docker.internal` reaches the Mac host from containers (add
`extra_hosts: ["host.docker.internal:host-gateway"]` if it ever doesn't resolve);
(3) model name in GuideAnts must match the Ollama tag exactly.

**Design rule:** On Apple Silicon, **bypass GuideAnts' bundled containerized `llama-cpp` runtime**
(`localhost:8110/llama-cpp` → CPU-only on Mac) and route to host Ollama. On a Linux+NVIDIA box the
containerized runtime CAN use the GPU, so flip back to it there.

**Still to verify on the Mini:** confirm the *chat* completion service reads the same
`OpenAI:Endpoint` and hits `/v1/chat/completions` (embeddings path is confirmed).

### Updated starter prompt for the MacMini session
> Continuing GuideAnts JobSearch on the MacMini. Read
> `Worktrees/GuideAnts/mydocs/HANDOFF.md` — especially the "Mac local inference RESOLVED" section.
> Decided architecture: route inference to host-native Ollama (Metal) via GuideAnts'
> OpenAI-compatible provider (`OpenAI:Endpoint=http://host.docker.internal:11434/v1`), NOT the
> containerized llama-cpp (CPU-only on Mac). Today: (1) confirm the chat completion service honors
> `OpenAI:Endpoint`; (2) stand up the stack with `docker compose`; (3) pull `llama3.1` +
> `nomic-embed-text` in host Ollama; (4) wire chat+embeddings to the OpenAI provider in Settings and
> verify end-to-end in the default UI. Work on the `feature/jobsearch-console` worktree.
