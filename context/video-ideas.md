# Video Ideas & Strategy

## The honest framing

The "we used TYK as a BFF and that was the wrong tool" story is strong DevRel content:
- TYK wants authentic, learn-in-public, real implementation stories
- "I misused a gateway as a BFF — here's what gateways are actually for" is more credible than fake hype
- You still have real TYK experience: rate limits, docs, customer incidents, single-tenant deploy, OSS cost constraint

Don't pitch "TYK solved our BFF problem." Pitch **when TYK is the right tool, and when it isn't.**

---

## Gateway vs BFF — 9/10

Best idea. Perfect for 3–5 min face-to-camera or screencast.

Angle:
- Problem: clients need lean filtered APIs
- Wrong instinct: "put a gateway in front and reshape in the gateway"
- Right model: BFF for application logic, gateway for edge concerns
- Where TYK shines: auth, rate limit, routing, policies, analytics, multi-tenant/API productization
- How they combine: Client → TYK → BFF (Fastify/Hono) → internal Scala

Why this works:
- Shows product judgment, not just fanboy energy
- Matches their topics: API management, real implementation stories
- Uses your actual experience without lying

### Extra idea: Rate limiting deep dive — 7.5/10

Solid, safer, more "product demo." Good if you want less architectural controversy.

Angle:
- Why naive rate limiting fails (per-IP only, bursty clients, multi-endpoint abuse)
- What you actually need in B2B SaaS (per-key, per-plan, quotas, soft vs hard limits)
- How TYK models this
- 60-sec live demo: key A gets 10 req/min, key B gets 100

Risk: more generic unless tied to a real customer incident.

