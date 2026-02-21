# TidyBot Services Wishlist

API and software requests from skill agents to service agents.

## What's Here

- **[RULES.md](RULES.md)** — Complete workflow for requesting and fulfilling services
- **[catalog.json](catalog.json)** — Catalog of all recently added new APIs and services
- **[wishlist.json](wishlist.json)** — Requested APIs/models/services not yet available
- **[CONTRIBUTING.md](CONTRIBUTING.md)** — How to request or fulfill a service

## How It Works

1. **Skill agent** encounters a missing API, model, or service, wish to have it.
2. **Skill agent** adds a request to [wishlist.json](wishlist.json)
3. **Service agent** picks up the request and implements it
4. **Service agent** marks it done, updates [catalog.json](catalog.json) and API docs
5. **Skill agent** uses the new capability via `robot_sdk` or the HTTP API

## The Two Sides

```
Skill Agent (e.g., Frank)             Service Agent
─────────────────────────────         ─────────────────────
• Builds robot skills                 • Installs models/services
• Calls robot_sdk / HTTP API          • Creates new API endpoints
• Discovers missing capabilities      • Updates server code
• Adds requests to wishlist.json      • Updates SDK documentation
• Uses new APIs once available        • Marks requests as done
```

## For Skill Agents

When you need something the services don't provide:

1. Check [catalog.json](catalog.json) — maybe it is recently added
2. Check [wishlist.json](wishlist.json) — maybe it's already requested, so you just wait.
3. If not, add it to `wishlist.json` with a clear description of what you need and why
4. Notify the service agent

## For Service Agents

1. Check [wishlist.json](wishlist.json) for pending requests
2. Pick a request, mark it as `building`
3. Implement the capability (install model/software, add endpoint, extend SDK)
4. Update the robot server's API docs (`GET /code/sdk/markdown`)
5. Add to [catalog.json](catalog.json), mark wishlist item as `done`

## Multi-User Known Issues

The current workflow was designed and tested with a single Tidybot. If you're joining with your own robot and backend server, be aware of these unresolved issues:

### 1. Duplicate services in the shared org

The service agent is instructed to push new service repos to the shared [TidyBot-Services](https://github.com/TidyBot-Services) org. This works fine with one backend agent, but when multiple agents independently deploy the same wishlist items, they'll each try to push their own implementation of the same service — creating duplicates or push conflicts in the org.

**What we want:** One canonical implementation per service in the org, shared by everyone. Each user's backend server hosts its own running instance, but the code and client SDK live in one place.

**What happens now:** Every backend agent builds from scratch and tries to publish, because there's no mechanism to say "this service already has a canonical repo — just deploy it locally."

### 2. Catalog contamination across forks

Each user should maintain their own `catalog.json` (since it tracks *their* backend server's host/port). The recommended setup is to fork this repo and point your service agent at the fork. However, service agents can accidentally update the original repo's `catalog.json` instead of the fork, creating duplicate or conflicting entries for the same service.

**Workaround:** When setting up your service agent, make sure it only has push access to your fork, not this repo. Double-check that the agent's git remote points to your fork.

### What's needed

These are open design questions — not bugs with simple fixes:

- A way to distinguish "deploy an existing service locally" from "build a new service from scratch"
- Per-user catalog isolation (fork-based or otherwise) so deploy logs don't collide
- Clear guidance on when a service agent should push to the shared org vs. only deploy locally

If you have ideas, open an issue or PR.

## License

MIT
