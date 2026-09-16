# Gateway vs BFF

## You need a Gateway ✔️
* You've got a web app.
* Happy users. They use it every day.
* Then the ideas start coming in.
  * Ship a mobile app
  * Integrate with a 3rd party service
  * Provide access to bots and agents
* Either way you need a REST API
* Exposing endpoints can be dangerous: scrapers, brute force, noisy neighbors.
* First thing that comes to mind? A Gateway.
* A software layer between your client and your backend services where you can set up rate limits, auth, quotas.

## But don't you need something else?

* Perhaps your backend is composed of multiple services. You have to make multiple API calls, filter the results, and merge them before sending the response to the client.
* Or maybe the internal API isn't something you'd expose as-is.
* Could be that there's a legacy backend that you can't change.
* Or you need a pilot, but shipping a new endpoint in the core app takes forever.

* In any case, you need a logic layer between your client and your backend services. 
* And we already have it right? We can write that logic in the gateway itself... MEEC!!!! 💀⚠️🚨 ERROR 👺😈👹
* DON'T DO THAT. You will regret it. 
* You will be stuck with a gateway that is hard to maintain, hard to test, and hard to iterate on.
* The gateway is not the right place for product logic. 💀 I learned this the hard way


* And I can tell. You need a Friend, a Best Friend Forever, a BFF 💖 

## What is a BFF?
BFF stands for **Backend for Frontend**. A BFF is a service that sits between your client and your backend services. 

The goal of a BFF is to adapt your API to the specific needs, requirements and experience of a specific client. You could not care and just use an API that returns the same data for all clients, or you could have one general purpose API that allows you to filter the data that you need on different clients. Typically, I think that BFF will result in the cleanest solution to this problem.

It is responsible for orchestrating calls to multiple backend services, filtering and reshaping data, and implementing business logic that is specific to the needs of the client.

Can you build your UI views by making single API calls? If yes, then you do not need a bff. If single UI views are requiring 2 to 3 calls, maybe to the same API maybe to multiple API's, then you can alleviate this with a bff.

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

## The trap of fake BFFs
But I can do that in the gateway itself! 

💀 This was exactly my mistake!

Yes. Gateways have some in-gate scripting capabilities that allow you to do some filtering and reshaping of the data. Some examples of this are:
* Kong functions
* Apigee Service Callouts
* Tyk Virtual Endpoints 💀

But those are not meant to be used to implement a BFF. This is not an "App". It's just Gateway configuration.

That is fine for glue (rename a field, add a header, mock a 200, forward a call...). It is a bad place for product logic (filter by date, merge orders+customers, reshape per client).

💀 TYK virtual endpoints was my trap. It's such a powerful tool — too good, even. 
💀 [HERE TALK ABOUT: TYK Virtual Endpoints, single-tenant Docker, chose TYK because OSS / no per-client license, ~20 endpoints, productized, still kept TYK for auth/rate limits/routin]. 
💀 So good you'll want to use it for everything, and you'll end up using it for something it was never meant for. Don't let this great tool fool you. Be clear about what it's for and what it isn't. If you misuse it and it goes wrong, that's on you.

### Example 1: it looks like it works, but it is a trap
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

### Example 3: Deploy and test fake vs real BFF

You implmenet a change in **the middleware itself**, e.g. you tweak one aggregation function for performance. Scala and Tyk routing stay put.

#### Fake BFF (logic in the gateway)

**Where the tests live:** Postman/Newman; a second tool, outside the gateway. You cannot `npm test` the aggregation next to the code.

**What you deploy:** gateway config + reload Tyk (whole gateway), then roll that to each tenant.

```
edit JS / plugin in the gateway
  → tests: Postman after the gateway is up (need Tyk + Scala)
  → deploy: gateway reload
  → blast radius: the gateway
```

#### Real BFF (standalone app)

**Where the tests live:** next to the code. `npm test` on `aggregate()`, mocked Scala, on the PR. Postman can still be extra e2e — it should not be the only way.

**What you deploy:** the BFF container only. Tyk and Scala untouched.

```
edit aggregate() in the BFF app
  → tests: npm test in CI (no Tyk, no Scala)
  → deploy: BFF container
  → blast radius: one service
```

### 💀 Our case

We relied too much on the gateway scripting. Specifically the TYK virtual endpoints. That small JS machine had to deal with GB of data, complex filtering, and orchestration. It was a nightmare to maintain and iterate on. 

And the testing was another nightmare. We had to test the logic in Postman, which was outside the gateway. So we had to deploy the gateway to test the logic. And every time we wanted to change something, we had to deploy the whole gateway. 

### When scripting in Gateway *is* a good idea
Stable glue, not a product:

- Mock or terminate a route (`GET /health` style, or "this method is gone")
- Tiny rewrite / header tweak that will not grow
- Two upstream calls whose contract almost never changes
- A policy check that stock middleware cannot express

Rule of thumb: if a product manager will ask to change the response shape next sprint, it does not belong in a virtual endpoint. Put Tyk in front for auth, rate limits, routing. Put a real BFF (Fastify, Hono, whatever) behind it for filter / merge / reshape.

## How to implement a BFF

BFF is a pattern, not a product. 
Keep it simple.
Use a normal HTTP app (Fastify/Hono/Nest/Go) that talks to backends, merges/filters, and returns the client shape.

## 💀 How we solved the problem

If you're still curious about how it ended. 
We figured out what endpoints our clients needed, so we got rid of the virtual endpoints / fake BFF and directly implemented the logic in the final app.
Start with a BFF. Put TYK in front when you need the edge. We did it backwards.
We still kepy TYK as Gatway using it right for what it is: auth, rate limits, routing, versioning, policies and analytics.
TYK is excellent at the edge. Don’t force it to be the app.

## Gateway 💖 BFF

As you can see, a Gateway and a BFF are complementary. You need both to have a complete API strategy. The Gateway is the edge, the BFF is the app logic. 

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