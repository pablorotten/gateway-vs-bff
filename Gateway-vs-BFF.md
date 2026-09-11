# Gateway vs BFF
You need to build and expose a restful API service for your existing backend services. The first thing it comes to your mind is a **Gateway**. 

But... are you sure is the **only** thing you need? Aren't you missing something before?

* You may have a mature product with a secure but slow pipeline.
* Your product have some generic and simple endpoints that return large data chunks.
* Your clients want to filter and reshape the data
* Writting new endpoints in your backend is slow and painful. Not ideal for a pilot or a proof of concept.
* So you think: "I will put a Gateway in front of my backend and I will be able to filter and reshape the data for my clients".
* MEEC!!! Wrong answer! For sure, a Gateway is a good idea. It will help you with authentication, rate limiting, routing, versioning, policies and analytics. But it is not enough. 
* You need a **BFF** (Backend for Frontend)! And this should be the first step in your API strategy. 

💀 I learned this the hard way. I will share my experience with you, so you can avoid the same mistakes I did.

## What is a BFF?
BFF = Backend for Frontend. A BFF is a service that sits between your client and your backend services. 

The goal of a BFF is to adapt your API to the specific needs, requirements and experience of a specific client. You could not care and just use an API that returns the same data for all clients, or you could have one general purpose API that allows you to filter the data that you need on different clients. Typically, I think that BFF will result in the cleanest solution to this problem.

It is responsible for orchestrating calls to multiple backend services, filtering and reshaping data, and implementing business logic that is specific to the needs of the client.

Can you build your UI views by making single API calls? If yes, then you do not need a bff. If single UI views are requiring 2 to 3 calls, maybe to the same API maybe to multiple API's, then you can alleviate this with a bff.


## The trap of fake BFFs
But I can do that in the gateway itself! 

💀 This was exactly my mistake!

Yes. Gateways have some in-gate scripting capabilities that allow you to do some filtering and reshaping of the data. Some examples of this are:
* Kong functions
* Apigee Service Callouts
* Tyk Virtual Endpoints 💀

But those are not meant to be used to implement a BFF. This is not an "App". It's just Gateway configuration.

That is fine for glue (rename a field, add a header, mock a 200, forward a call...). It is a bad place for product logic (filter by date, merge shipments+trials, reshape per client).

💀 TYK virtual endpoints was my trap. It's such a powerful tool — too good, even [HERE TALK ABOUT TYK VIRTUAL ENDPOINTS, SOMETHING QUICK BUT SHOWCASE ITS POWER]. So good you'll want to use it for everything, and you'll end up using it for something it was never meant for. Don't let this great tool fool you. Be clear about what it's for and what it isn't. If you misuse it and it goes wrong, that's on you.

### Example 1: it looks like it works, but it is a trap
Backend: `GET /internal/shipments` returns a huge blob (every shipment, every field).

Client wants: `GET /shipments?from=2024-01-01&to=2024-03-01` with `{ id, date, co2 }` only.

Virtual endpoint:

1. `TykMakeHttpRequest` to `/internal/shipments`
2. `JSON.parse` the body
3. Filter by date in a `for` loop
4. Map to three fields
5. Return 200

Pilot client is happy. You didn't touch Scala. You didn't stand up a service. This is the moment you think the gateway *is* the BFF.

Next week the same client wants shipments **joined with trials**, filtered by site, with CO2 rolled up by month. Another client wants a different shape. Timezones are wrong. The internal payload is 50MB and you are filtering it in ES5 on the gateway.

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
We got lucky because all our clients wanted more or less the same shape, so we could write all those endpoints in one single sprint and we were done.
We still kepy TYK as Gatway using it right for what it is: auth, rate limits, routing, versioning, policies and analytics.

## Gateway 💖 BFF

As you can see, a Gateway and a BFF are complementary. You need both to have a complete API strategy. The Gateway is the edge, the BFF is the app logic. 

## Concepts

**Upstream** — the service behind the gateway (your Scala API). A virtual endpoint **calls upstreams** when its JS does `TykMakeHttpRequest` to those backends, then builds the client response. Client hits Tyk; Tyk’s JS hits Scala; Tyk answers the client.

**Glue** — tiny wiring, no real rules. Rename a field, add a header, mock a 200, forward a call. It barely changes. Fine in the gateway.

**Not glue / product logic** — filter by date, merge shipments+trials, reshape per client. That belongs in a BFF.

**Orchestration** — one client call becomes several backend calls, then you combine the result. Example: client hits `GET /dashboard`. BFF calls shipments, trials, and CO2, merges them, returns one JSON. Gateway routing is “send this path there.” Orchestration is “call N services, wait, stitch, reply.”

**Virtual endpoints** — Tyk’s in-process JS (JSVM) that can terminate a request, call upstreams, and return a custom body. Other gateways have the same escape hatch under other names: Kong Lua/JS plugins, Apigee JS + ServiceCallout, Azure APIM `send-request`, NGINX njs. AWS Lambda behind API Gateway is closer to a real BFF (separate runtime).

**Experience API** — Salesforce / Apigee name for a client-facing facade. People often call the gateway proxy a BFF. Best practice even there: keep the proxy light; put complex logic in a real service (Cloud Run, etc.). Same split, muddier words.