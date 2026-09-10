# Technical Context

## Company problem

- Product: B2B SaaS for supply chain optimization (clients upload data, run optimizations, view/export results)
- Architecture: Single-Tenant SaaS — one isolated AWS instance per client, Docker containers
- Pilot client needed a client-facing RESTful API
- Existing system only had generic internal endpoints returning large data chunks
- Clients wanted specific filtered endpoints (date ranges, etc.)

## Constraints

- Legacy backend in Scala with Akka
- New API developer doesn't know Scala/Akka
- Core product release pipeline is slow and rigorous
- Must be 100% free/open-source per instance (single-tenant = no per-client license fees)
- Goal: rapidly experiment with pilot client, then productize

## What we did

- Placed TYK gateway inside the client's single-tenant Docker environment, in front of the Scala app
- TYK was chosen after an expert assessment — mainly because it was open source (no per-client license)
- I helped implement endpoints, adapt Scala app endpoints, fix performance issues
- Wrote internal and external docs
- Supported customers with questions and incidents

## What I learned

- **Gateway vs BFF is the key insight**
- TYK is great at: auth, rate limiting, routing, policies, analytics, API productization
- TYK is weak at: multi-call orchestration + merge, complex domain filtering, rapid iteration on business logic
- The mistake: using the gateway to do BFF work (application logic that filters/merges/reshapes data)
- The right model: **Client → TYK (edge) → BFF (app logic) → internal services**

## Architecture diagram (text)

```
Client
  ↓
TYK Gateway (auth, rate limit, routing, versioning)
  ↓
BFF / API Service (filter, merge, reshape, domain logic)
  ↓
Scala/Akka backend (internal endpoints)
```

## When TYK is the right tool

- You only need: keys, rate limits, route to one existing endpoint, maybe strip fields
- Backends already return the right shape (or almost)
- No real orchestration or business filtering
- API productization, multi-tenant API management

## When you need a BFF

- 1 client request → N internal calls → merge
- Filter/aggregate because backend can't yet
- Public API design separate from internal APIs
- Rapid experimentation without backend releases
