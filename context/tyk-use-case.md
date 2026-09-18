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
## Why a BFF is a good idea

### Case 1: Agile middleware

* The main app has a very basic API. Generic endpoints. Large data chunks.
* The core pipeline is slow and rigorous. Fine for the product. Terrible for a REST API pilot.
* You need a middleware with agile, flexible development — room to experiment and play around until the contract is right.
* That middleware is a BFF. Not the gateway.
* Once you have a final, tested, mature list of endpoints clients are happy with, you go back to the real backend and build those endpoints for real (yes, through the slow process).
* Why push them down? Stop overloading the backend with many calls from the BFF. Get more customized, performant endpoints next to the data.
* Success is **not** getting rid of the BFF. Success is putting logic in the right layer.
* Promote stable domain endpoints into the backend. Keep the BFF for orchestration, client-specific shaping, and the next experiment.
* Mature product looks like: `Client → Gateway → thin BFF → richer backend`. Not "gateway straight to backend, BFF deleted."


### Case 1: it looks like it works, but it is a trap
Backend: `GET /internal/orders` returns a huge blob (every order, every field).

Client wants: `GET /orders?from=2024-01-01&to=2024-03-01` with `{ id, date, total }` only.

Virtual endpoint:

1. `TykMakeHttpRequest` to `/internal/orders`
2. `JSON.parse` the body
3. Filter by date in a `for` loop
4. Map to three fields
5. Return 200

Pilot client is happy. You didn't touch Scala. You didn't stand up a service. This is the moment you think the gateway *is* the BFF.

Next week the same client wants orders **joined with customers**, filtered by region, with revenue rolled up by month. Another client wants a different shape. Timezones are wrong. The internal payload is 50MB and you are filtering it in ES5 on the gateway.

Now you have:

- Domain rules living in API config
- No CI for the logic that customers pay for within the gateway (need to implement with external Postman/Newman tests)
- Iteration speed tied to gateway deploys, not an app pipeline
- Edge concerns (keys, rate limits) mixed with app concerns (filters, merges)

You did not avoid a BFF. You implemented a BFF in the worst runtime: config + a sandbox.

You *can* still put Postman/Newman in CI after the gateway. That *is* CI for the customer API. The problem is you cannot test **inside the tool that holds the logic**. Tests live in a second tool. Two deploys to check one function.

### Case 3: Deploy and test fake vs real BFF

You implmenet a change in **the middleware itself**, e.g. you tweak one aggregation function for performance. Scala and Tyk routing stay put.

## Fake vs Real BFF

### Fake BFF (logic in the gateway)

**Where the tests live:** Postman/Newman; a second tool, outside the gateway. You cannot `npm test` the aggregation next to the code.

**What you deploy:** gateway config + reload Tyk (whole gateway), then roll that to each tenant.

```
edit JS / plugin in the gateway
  → tests: Postman after the gateway is up (need Tyk + Scala)
  → deploy: gateway reload
  → blast radius: the gateway
```

### Real BFF (standalone app)

**Where the tests live:** next to the code. `npm test` on `aggregate()`, mocked Scala, on the PR. Postman can still be extra e2e — it should not be the only way.

**What you deploy:** the BFF container only. Tyk and Scala untouched.

```
edit aggregate() in the BFF app
  → tests: npm test in CI (no Tyk, no Scala)
  → deploy: BFF container
  → blast radius: one service
```

### When scripting in Gateway *is* a good idea
Stable glue, not a product:

- Mock or terminate a route (`GET /health` style, or "this method is gone")
- Tiny rewrite / header tweak that will not grow
- Two upstream calls whose contract almost never changes
- A policy check that stock middleware cannot express

Rule of thumb: if a product manager will ask to change the response shape next sprint, it does not belong in a virtual endpoint. Put Tyk in front for auth, rate limits, routing. Put a real BFF (Fastify, Hono, whatever) behind it for filter / merge / reshape.

## Concepts

**Upstream** — the service behind the gateway (your Scala API). A virtual endpoint **calls upstreams** when its JS does `TykMakeHttpRequest` to those backends, then builds the client response. Client hits Tyk; Tyk’s JS hits Scala; Tyk answers the client.

**Glue** — tiny wiring, no real rules. Rename a field, add a header, mock a 200, forward a call. It barely changes. Fine in the gateway.

**Not glue / product logic** — filter by date, merge orders+customers, reshape per client. That belongs in a BFF.

**Orchestration** — one client call becomes several backend calls, then you combine the result. Example: client hits `GET /dashboard`. BFF calls orders, customers, and invoices, merges them, returns one JSON. Gateway routing is “send this path there.” Orchestration is “call N services, wait, stitch, reply.”

**Virtual endpoints** — Tyk’s in-process JS (JSVM) that can terminate a request, call upstreams, and return a custom body. Other gateways have the same escape hatch under other names: Kong Lua/JS plugins, Apigee JS + ServiceCallout, Azure APIM `send-request`, NGINX njs. AWS Lambda behind API Gateway is closer to a real BFF (separate runtime).

**Experience API** — Salesforce / Apigee name for a client-facing facade. People often call the gateway proxy a BFF. Best practice even there: keep the proxy light; put complex logic in a real service (Cloud Run, etc.). Same split, muddier words.

## Popular API Gateway Providers

When choosing an API gateway, you have several strong options beyond TYK. Here's a quick overview of the most popular alternatives:

### Open Source

**Kong** — Built on NGINX/OpenResty. Very popular, huge plugin ecosystem. Lua-based plugins for customization. Strong community edition. Good if you want Lua scripting or need NGINX performance.

**Apache APISIX** — Built on NGINX/OpenResty like Kong, but newer. Lua plugins, dynamic routing, real-time analytics. Growing fast in the CNCF ecosystem.

**Traefik** — Cloud-native, auto-discovers services (Docker, Kubernetes). Great for microservices. Less traditional gateway features but excellent for dynamic environments.

**NGINX** — Not a full API gateway by default, but extremely capable with NGINX Plus or custom Lua scripting. The foundation many gateways are built on.

### Enterprise / Cloud-Managed

**Apigee (Google Cloud)** — Enterprise-grade API management. Service callouts for in-gate logic. Strong analytics, monetization, developer portal. Paid product, serious enterprise tooling.

**AWS API Gateway** — Fully managed, serverless-first. Deep integration with Lambda. Pay-per-request pricing. Good if you're already in AWS ecosystem.

**Azure API Management** — Microsoft's offering. Strong for Azure/.NET shops. `send-request` policy for in-gate logic. Enterprise features with Azure integration.

**MuleSoft (Salesforce)** — Enterprise integration platform, not just a gateway. ESB-style with API management. Heavy, but powerful for complex enterprise architectures.

### What They Have in Common

All these gateways share the same core edge concerns:
- Authentication & API keys
- Rate limiting & quotas
- Routing & versioning
- Analytics & monitoring
- Plugin/middleware systems for customization

The trap is the same across all of them: **they are edge tools, not application layers**. Every gateway has some scripting escape hatch (Kong Lua, Apigee Service Callouts, TYK Virtual Endpoints, Azure `send-request`). Those are for glue, not product logic. The pattern holds: put the gateway at the edge, put a BFF behind it when you need application logic.

## 💀 How we solved the problem

If you're still curious about how it ended. 
We figured out what endpoints our clients needed, so we got rid of the virtual endpoints / fake BFF and directly implemented the logic in the final app.
Start with a BFF. Put TYK in front when you need the edge. We did it backwards.
We still kepy TYK as Gatway using it right for what it is: auth, rate limits, routing, versioning, policies and analytics.
TYK is excellent at the edge. Don’t force it to be the app.
