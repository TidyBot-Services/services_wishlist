# TidyBot Services Wishlist

API and software requests from skill agents to service agents.

## What's Here

- **[RULES.md](RULES.md)** — Complete workflow for requesting and fulfilling services
- **[catalog.json](catalog.json)** — Menu of available service definitions (image, default port, description)
- **[wishlist.json](wishlist.json)** — Requested APIs/models/services not yet available
- **[CONTRIBUTING.md](CONTRIBUTING.md)** — How to request or fulfill a service

## How It Works

1. **Skill agent** encounters a missing API, model, or service
2. **Skill agent** adds a request to [wishlist.json](wishlist.json)
3. **Service agent** picks up the request and implements it
4. **Service agent** deploys via the **deploy agent** on the target compute server, updates [catalog.json](catalog.json)
5. **Skill agent** discovers the running service via the deploy agent and uses it

## Architecture

Each compute server runs a **deploy agent** (port 9000) that manages Docker containers, GPU assignment, port allocation, and health checks. The AI agent orchestrates everything:

```
Skill Agent                              Service Agent
─────────────────────────────            ─────────────────────
• Builds robot skills                    • Implements new services
• Discovers services via deploy agent    • Builds Docker images
• Calls service APIs directly            • Deploys via deploy agent
• Adds requests to wishlist.json         • Updates catalog.json
                                         • Marks requests as done

              Deploy Agent (:9000)
              ─────────────────────
              • Runs on each compute server
              • GET /services — list running services
              • POST /deploy — deploy a container
              • POST /stop — stop a service
              • GET /gpus — GPU status
```

### Service Discovery

Services are discovered at runtime by querying deploy agents — no hardcoded host:port entries needed:

```bash
# List running services on a compute server
curl http://<server>:9000/services

# Deploy a service
curl -X POST http://<server>:9000/deploy \
  -H "Content-Type: application/json" \
  -d '{"name": "yolo", "image": "tidybot/yolo-service", "port": 8000, "gpu": true}'
```

### catalog.json

`catalog.json` is a **menu of available service definitions** — what services exist and how to deploy them. It is NOT a registry of running instances. To find what's actually running, query the deploy agent.

## For Skill Agents

When you need something the services don't provide:

1. Query the deploy agent (`GET <server>:9000/services`) — maybe it's already running
2. Check [catalog.json](catalog.json) — maybe it exists but isn't deployed yet
3. Check [wishlist.json](wishlist.json) — maybe it's already requested
4. If not found, add it to `wishlist.json` with a clear description
5. Notify the service agent

## For Service Agents

1. Check [wishlist.json](wishlist.json) for pending requests
2. Pick a request, mark it as `building`
3. Implement the service (Dockerfile + server + client.py in the service repo)
4. Deploy via the deploy agent on the target compute server
5. Add the service definition to [catalog.json](catalog.json), mark wishlist item as `done`

## License

MIT
