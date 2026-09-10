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

## What is a BFF?
BFF = Backend for Frontend. A BFF is a service that sits between your client and your backend services. 

The goal of a BFF is to adapt your API to the specific needs, requirements and experience of a specific client. You could not care and just use an API that returns the same data for all clients, or you could have one general purpose API that allows you to filter the data that you need on different clients. Typically, I think that BFF will result in the cleanest solution to this problem.

It is responsible for orchestrating calls to multiple backend services, filtering and reshaping data, and implementing business logic that is specific to the needs of the client.

Can you build your UI views by making single API calls? If yes, then you do not need a bff. If single UI views are requiring 2 to 3 calls, maybe to the same API maybe to multiple API's, then you can alleviate this with a bff.


## The trap of Virtual Endpoints
But I can do that in the gateway itself! I can create virtual endpoints that will call the backend and filter/reshape the data.

Yes. Gateways like Tyk even documents this as a feature. Virtual endpoints run JavaScript inside the gateway (JSVM). They can call upstreams with `TykMakeHttpRequest` / `TykBatchRequest`, mash the JSON, and return one response. Tyk's own docs say: aggregate multiple internal services and skip "starting up an aggregation service."

That sentence is the trap. It describes a **BFF job** and sells you a **gateway plugin**.

### What a virtual endpoint actually is
It is not Node. It is not an app. It is ES5 JavaScript in the gateway process.

- Enable `enable_jsvm: true`, paste a function (or a base64 blob in the API definition).
- The function **terminates** the request. It must build the HTTP response itself (`TykJsResponse`).
- Outbound calls are **synchronous**. The gateway thread waits.
- Logs are `log()` into gateway logs. No real tests, debugger, types, or npm.
- Function names must be unique across the whole API portfolio (shared VM).

That is fine for glue (rename a field, add a header, mock a 200, forward a call...). It is a bad place for product logic (filter by date, merge shipments+trials, reshape per client).

### Example 1 — it looks like it works
Backend: `GET /internal/shipments` returns a huge blob (every shipment, every field).

Client wants: `GET /shipments?from=2024-01-01&to=2024-03-01` with `{ id, date, co2 }` only.

Virtual endpoint:

1. `TykMakeHttpRequest` to `/internal/shipments`
2. `JSON.parse` the body
3. Filter by date in a `for` loop
4. Map to three fields
5. Return 200

Pilot client is happy. You didn't touch Scala. You didn't stand up a service. This is the moment you think the gateway *is* the BFF.

### Example 2 — it falls apart
Next week the same client wants shipments **joined with trials**, filtered by site, with CO2 rolled up by month. Another client wants a different shape. Timezones are wrong. The internal payload is 50MB and you are filtering it in ES5 on the gateway. A bug ships as a dashboard paste / gateway reload, with no unit test.

Now you have:

- Domain rules living in API config
- No CI for the logic that customers pay for
- Iteration speed tied to gateway deploys, not an app pipeline
- Edge concerns (keys, rate limits) mixed with app concerns (filters, merges)

You did not avoid a BFF. You implemented a BFF in the worst runtime: config + a sandbox.

### When virtual endpoints *are* a good idea
Stable glue, not a product:

- Mock or terminate a route (`GET /health` style, or "this method is gone")
- Tiny rewrite / header tweak that will not grow
- Two upstream calls whose contract almost never changes
- A policy check that stock middleware cannot express

Rule of thumb: if a product manager will ask to change the response shape next sprint, it does not belong in a virtual endpoint. Put Tyk in front for auth, rate limits, routing. Put a real BFF (Fastify, Hono, whatever) behind it for filter / merge / reshape.


## How to implement a BFF

BFF is a pattern, not a product.
Use a normal HTTP app (Fastify/Hono/Nest/Go) that talks to backends, merges/filters, and returns the client shape. Dedicated “BFF platforms” are rare; GraphQL or tRPC are optional, not required.