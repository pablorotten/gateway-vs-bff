# Video Ideas & Strategy

## The honest framing

The "we used TYK as a BFF and that was the wrong tool" story is strong DevRel content:
- TYK wants authentic, learn-in-public, real implementation stories
- "I misused a gateway as a BFF — here's what gateways are actually for" is more credible than fake hype
- You still have real TYK experience: rate limits, docs, customer incidents, single-tenant deploy, OSS cost constraint

Don't pitch "TYK solved our BFF problem." Pitch **when TYK is the right tool, and when it isn't.**

---

## Your ideas

### 1. Gateway vs BFF — 9/10

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

### 2. Rate limiting deep dive — 7.5/10

Solid, safer, more "product demo." Good if you want less architectural controversy.

Angle:
- Why naive rate limiting fails (per-IP only, bursty clients, multi-endpoint abuse)
- What you actually need in B2B SaaS (per-key, per-plan, quotas, soft vs hard limits)
- How TYK models this
- 60-sec live demo: key A gets 10 req/min, key B gets 100

Risk: more generic unless tied to a real customer incident.

---

## Additional ideas

### A. OSS gateway + single-tenant cost — 8/10

Your real constraint was 1 AWS env per client = no per-instance license tax.

Angle:
- Multi-tenant: license once
- Single-tenant: license multiplies with every customer
- Why OSS gateway choice is a business decision, not just tech
- TYK as the pick under that constraint
- What you still needed beside the gateway (BFF/service layer)

Great for business + architecture audience.

### B. Rate limits that don't suck for B2B APIs — 7.5/10

If you want a cleaner product story:
- Per-API-key limits
- Different plans
- Protect expensive endpoints (exports/optimizations)
- Show TYK config + one abuse scenario

Pair with a short meme cut for LinkedIn.

### C. From internal blob endpoints to a public API contract — 7/10

Not "TYK did filtering," but:
- Internal APIs optimize for product internals
- Public APIs optimize for developer experience
- Gateway helps expose/version/protect the public surface
- Docs + support load drop when contract is stable

You can mention writing internal + external docs and reducing support noise.

### D. Meme-style short (30–45s) as teaser — 6/10

Examples:
- "API Gateway" vs "BFF" dating-app style mismatch
- "When your gateway starts writing business logic"
- "Single-tenant SaaS + per-host license = financial horror"

Use this to drive people to the 3–5 min video.

---

## What to skip

- "Why TYK is the best BFF" — dead on arrival
- Pure feature tour with no story — forgettable
- "I am junior-curious and hungry" personality video with no tech — weak for this company
- Anything that needs K8s/OTel depth you don't have yet

---

## Scorecard

| Idea | Score | Use as |
|------|------:|--------|
| Gateway vs BFF (honest) | 9/10 | Main application video |
| OSS gateway + single-tenant cost | 8/10 | Alt main / blog companion |
| Rate limiting in B2B | 7.5/10 | Second video or safer option |
| Public vs internal API contract | 7/10 | Strong follow-up |
| Meme short only | 6/10 | Teaser, not the application |

---

## Recommended application package

1. Main video: Gateway vs BFF, 3–5 min, face + diagrams
2. Optional short: meme teaser linking to it
3. Cover letter one-liner: "I ran TYK in production in single-tenant SaaS, wrote the docs, supported clients on it, and I want to teach the ecosystem when gateways are the right tool."
4. Don't hide seniority: "Junior title is fine. I'm pivoting into DevRel with production API-management scars."
