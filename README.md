# ToolMate AI 🤖

An **agentic AI system** — an LLM that doesn't just answer, it *does things*.
Three containers working together, orchestrated with Docker Compose:

```
 we ──▶ agent (port 9000) ──▶ tools (private, port 7000)
              │
              └──▶ redis (private, conversation memory)
```

| Container | Role | Office analogy |
|---|---|---|
| **agent** | LLM + tool-calling loop | The manager — decides what needs doing |
| **tools** | Calculator + clock microservice | The departments — do the actual work |
| **redis** | Per-session conversation memory | The office register — remembers who said what |

---

## ⚠️ Read this first — the rules that prevent 90% of problems

**1. After editing ANY file, we must rebuild.**
```bash
docker compose up -d --build
```
The image is a *photograph*, not a *mirror*. Editing a file does not update the
photograph — we have to take a new one.

**2. `docker compose ps` hides dead containers.**
If a service is missing from the list, it exited. Use `docker compose ps -a`,
then `docker compose logs <service>` for the reason.

**3. Don't paste commands with trailing `#` comments into zsh.**
If our terminal shows `quote>`, press **Ctrl+C** and re-run without the comment.

**4. Edited `.env`? Recreate, don't just restart.**
```bash
docker compose up -d --force-recreate
```
Environment variables are read once, at container start.

---

## 🆕 What's new versus the LLM project

| New concept | Why it appears here |
|---|---|
| **Three services, two custom builds** | `agent/` and `tools/` each have their own code, requirements, and Dockerfile — microservices made physical. |
| **A service with NO published port** | `tools` has no `ports:` section. The outside world cannot reach it *at all* — only the agent can, over the private network. |
| **The agent loop** | The LLM is shown a menu of tools and may say "run calculator with 23*47" — we execute it and feed the result back, in a loop. |
| **Stateful memory in Redis** | Conversation history per `session_id`, surviving container restarts via a volume. |

---

## 📁 What's in this folder

| File | Purpose |
|---|---|
| `agent/app.py` | The agent: loads memory from Redis, runs the think→act→observe loop, calls tools over HTTP |
| `agent/requirements.txt` | fastapi, uvicorn, grog, redis, httpx |
| `agent/Dockerfile` | Standard 6-liner, serves on port 9000 |
| `tools/app.py` | Plain FastAPI microservice: `POST /calculator`, `GET /datetime` |
| `tools/requirements.txt` | Just fastapi + uvicorn — tools have no AI in them |
| `tools/Dockerfile` | Standard 6-liner, serves on port 7000 |
| `docker-compose.yml` | All three services, the private network, the volume |
| `.dockerignore` / `.gitignore` | Keep secrets and junk out of images and git |

---

## ✅ Prerequisites

1. **Docker Desktop** running (whale 🐳 icon steady in the menu bar).
2. **An OpenAI API key** with credit — <https://platform.openai.com/api-keys>.
   The demo uses `openai/gpt-oss-120b`; a full run costs a fraction of a cent.

we do **not** need Python installed locally.

---

## 🚀 Step-by-step

### Step 1 — Create our `.env` file
```bash
touch .env
```
Open `.env` add entry for real key:
```
GROQ_API_KEY=gsk-proj-our-actual-key-here
```
No quotes, no spaces around the `=`. Verify: `ls -la .env`


### Step 2 — Confirm Docker Desktop is running
Whale 🐳 icon steady in the menu bar.

### Step 3 — Start all three containers
```bash
docker compose up -d --build
```
First run takes 2–4 minutes: it builds TWO images (agent and tools) and pulls
Redis from Docker Hub. Watch the output — we'll see both builds happen.

### Step 4 — Verify all three are up
```bash
docker compose ps -a
```
Expect `toolmate-agent`, `toolmate-tools`, and `toolmate-redis` — all **Up**.
If any says **Exited**, run `docker compose logs <service>` and check the
Troubleshooting table below.

### Step 5 — Health check
```bash
curl http://localhost:9000/
```
```json
{"status": "ToolMate Agent is live 🤖", "tools_url": "http://tools:7000", "redis_host": "redis"}
```

### Step 6 — Prove the tools service is private 🔒
This is a feature, not a bug — try to reach tools from our Mac:
```bash
curl http://localhost:7000/
```
**Connection refused — on purpose.** The `tools` service has no `ports:` section
in the compose file, so it has no street entrance. Only the agent, living inside
the same private Docker network, can reach it at `http://tools:7000`.

To prove it works *from the inside*, step into the agent's container and call it:
```bash
docker compose exec agent python -c "import httpx; print(httpx.get('http://tools:7000/').json())"
```
```json
{'status': 'ToolMate Tools is live...'}
```
Same URL that failed from our Mac, works from a neighbor. *Internal ports for
neighbors, published ports for visitors.*

### Step 7 — Talk to the agent
Give it a task that needs **two different tools**:
```bash
curl -X POST http://localhost:9000/chat -H "Content-Type: application/json" -d '{"session_id": "test", "message": "What is 20*80, and what time is it right now?"}'
```
The agent will call the calculator AND the clock, then compose one answer
{"response":"The product is **1600**.\n\nThe current date and time (UTC) is **2026‑11‑05 13:54 UTC**."}

### Step 8 — Prove it has memory 🧠
Same `session_id`, follow-up question with no numbers in it:
```bash
curl -X POST http://localhost:9000/chat -H "Content-Type: application/json" -d '{"session_id": "test", "message": "Now double that multiplication result"}'
```
{"response":"Doubling the result gives **3200**."}
history lives in Redis under `history:test`.

Now change the session and ask the same follow-up:
```bash
curl -X POST http://localhost:9000/chat -H "Content-Type: application/json" -d '{"session_id": "something-else", "message": "Now double that multiplication result"}'
```
It has no idea what we mean — separate session, separate memory. Per-user
isolation, for free. And responds as below:
{"response":"I’m not sure which multiplication result you’d like doubled. Could you please let me know the numbers you’d like multiplied (or the result you have in mind), and I’ll double it for you?"}

### Step 9 — Watch the agent think (the best demo) ⚡
Split our terminal into two panes. In one:
```bash
docker compose logs -f
```
In the other, send a chat request from Step 8 again. we'll see the request
ripple across services live: the agent receiving it, tools serving the
calculator call, the agent responding. This is how multi-service systems are
debugged in real life.

`Ctrl+C` stops watching (the containers keep running).

### Step 10 — Memory survives restarts ♻️
```bash
docker compose down
docker compose up -d
curl -X POST http://localhost:9000/chat -H "Content-Type: application/json" -d '{"session_id": "demo", "message": "What was that multiplication answer again?"}'
```
All three containers were destroyed and recreated, yet it still knows —
Redis stores its data in the `agent_memory` volume.

To actually wipe all memory:
```bash
docker compose down -v
```

### Step 11 — Clean up
```bash
docker compose down       # stop everything, keep the memory
docker compose down -v    # stop everything and wipe the memory too
```

---

## 🔍 How the agent loop works (agent/app.py)

```
loop (max 5 times):
    show the LLM:  conversation so far + the tool menu (TOOL_DEFS)
    if the LLM answers in words        -> break, we're done
    if the LLM says "run tool X(args)" -> call the tools service over HTTP,
                                          append the result to the conversation,
                                          go around again
```

- **TOOL_DEFS** is the menu card: each tool's name, description, and expected
  inputs. The LLM reads the descriptions to decide when a tool is needed.
- **Why max 5 laps?** A safety fuse. A confused model could otherwise call tools
  forever — and every lap costs we API tokens.
- **Why does an LLM need a calculator?** LLMs predict text; they are famously
  unreliable at arithmetic. Giving one a calculator is like giving an eloquent
  manager an actual accountant.
- **Why no retry loop for Redis** (unlike ScalerGPT's Chroma)? The Redis client
  connects *lazily* — only when the first command runs, which is our first
  `/chat` request, long after Redis has booted. Chroma's client connects
  immediately at startup, which is why it needed the polite-knocking loop.

---


## 🔑 Key concepts demonstrated

- **Microservice separation** — agent and tools are independent services with
  independent code, dependencies, images, and lifecycles
- **Private services** — omitting `ports:` removes the public entrance entirely;
  the strongest security line in the file is the one that isn't there
- **Service-name networking** — `http://tools:7000`, `host="redis"`; never IPs
- **The agent pattern** — think → act → observe, with a loop cap as a safety fuse
- **Stateful memory** — Redis + a volume = conversations that survive restarts
- **Lazy vs eager connections** — why Redis needs no retry loop but Chroma did

---

## The Flow is

1. FastAPI receives POST /chat
2. Load history:test from Redis
3. Add the user message
4. Send message and tool descriptions to Groq
5. Groq requests calculator and get_datetime
6. Agent calls tools:7000/calculator
7. Agent calls tools:7000/datetime
8. Add both results to the conversation
9. Ask Groq to compose the final response
10. Save the conversation to Redis
11. Return {"response": "..."}

---

## What LLM does?

1. User message
2. LLM understands the request
3. LLM compares it with tool descriptions
4. LLM returns either:
5. a normal answer, or
6. a structured tool call
---

## Important implementation notes
- The agent uses Groq for language generation, not OpenAI directly.
- Redis provides session-based memory.
- The tools container performs calculations and retrieves the current time.
- The agent is the only public service; the tools service remains private inside Docker.
- The current call_tool() expects the datetime service to return "now", which matches tools/app.py.
- The calculator tool uses Python eval in the tools service. That is acceptable only for a controlled demo and should be replaced with a safe math parser in production.