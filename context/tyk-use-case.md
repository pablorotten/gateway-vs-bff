# TYK Use Case (as explained by Pablo)

## The use case

We had a B2B SaaS product — **Supply App** — a web application for supply chain / clinical trial optimization (upload data, run optimizations and simulations, view/export results).

It started as a normal UI-only web app. But clients kept asking for **API integration**: they wanted to connect their own systems to ours automatically — update workflows, run optimizations and simulations, and retrieve results **without any human interaction**.

The trigger was a **key client** who requested the API integration and was ready to pay for it. It started as a Proof of Concept for that one client.

The new API needed to expose **lean, filtered endpoints** (e.g. filtered by date ranges) — while the existing system only had generic internal endpoints that returned large data chunks (client data, trials, shipments, CO2 emissions, etc.).

Constraints:

- Legacy backend in **Scala + Akka**
- The person building the new API didn't know Scala/Akka
- The core product release pipeline was slow and rigorous
- **Single-tenant SaaS**: one isolated AWS instance per client, Docker containers. So the API solution had to be **100% free / open-source per instance** — no per-client license fees
- Goal: rapidly experiment with the pilot client, then productize to all customers

## The solution we found with TYK

- Placed **TYK gateway inside each client's single-tenant Docker environment**, in front of the Scala app
- TYK was chosen after an **expert assessment** — mainly because it was **open source** (no per-client license, which was a hard constraint in single-tenant)
- I put in place the whole API infrastructure on TYK and led the API product from **conception to production**
- Implemented/adapted endpoints, consolidated middleware logic, fixed performance issues
- Wrote internal docs (how the platform works, how to add endpoints, how to deploy/test) and external docs (how to use the API)
- Supported customers through questions and incidents

Results:

- **~20 endpoints** created to satisfy different needs
- Started as PoC → productized → **adopted by all existing accounts**, **5 new customers integrated**
- API usage extended to internal teams for bulk data extraction (much faster than manual UI) — then identified as a **sellable product**: customers accessing their own historic supply chain data (shipments, CO2 emissions, trial duration) directly via API
- **Launched a new profitable business line** for the company
- OpenAPI documentation updates dropped support tickets significantly

## The mistake (TYK was not the wrong tool — we used it for the wrong job)

TYK is a **Gateway** — the edge. Its job: auth, API keys, rate limiting, routing, versioning, policies, analytics. It does **not** own business logic. Having TYK is fine.

What we also needed was a **BFF (Backend for Frontend)** — an application layer that sits *behind* the gateway and handles **orchestration, filtering, merging, and reshaping** of data. The problems we hit:

- **Client needs didn't match backend endpoints.** Backends returned generic, large data chunks. Clients wanted specific filtered/reshaped responses (date ranges, subsets of fields). That's **domain logic**.
- **TYK is weak at multi-call orchestration + merge, complex domain filtering, and rapid iteration on business logic.** Those are BFF responsibilities.
- **Config is fine for glue, bad for domain logic.** TYK's middleware/plugins can fake some filtering, but it becomes unmaintainable the moment the logic gets complex.
- **Every client iteration meant touching TYK config or the slow Scala release pipeline** — the exact bottleneck we wanted to avoid.

The mistake in one line: **we implemented BFF logic inside TYK instead of adding a BFF behind it.**

What we should have done:

- **Pilot / research:** start with a **BFF alone** — one consumer, auth in the BFF, no keys/rate limits/analytics needed yet. Fast iteration on filtered endpoints without the Scala release pipeline.
- **Productize:** put **TYK in front** — keys, rate limits, routing, policies. Then: `Client → TYK (edge) → BFF (app logic) → internal services`. Gateway for edge concerns, BFF owning the domain logic.

```
Client
  ↓
TYK Gateway (auth, rate limit, routing, versioning)
  ↓
BFF / API Service (filter, merge, reshape, domain logic)
  ↓
Scala/Akka backend (internal endpoints)
```

### When TYK (a gateway) is the right tool

- You only need keys, rate limits, route to an existing endpoint, maybe strip fields
- Backends already return the right shape (or almost)
- No real orchestration or business filtering
- API productization / multi-tenant API management

### When you need a BFF

- 1 client request → N internal calls → merge
- Filter/aggregate because the backend can't yet
- Public API design that's separate from internal APIs
- Rapid experimentation without backend releases