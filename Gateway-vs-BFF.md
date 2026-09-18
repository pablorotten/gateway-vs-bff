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

- [x] Perhaps your backend is composed of multiple services. You have to make multiple API calls and merge them before sending the response to the client.
- [x] Maybe some backend deetails should not be exposed to the client. You need to filter and reshape the data.
- [x] Or say you need a pilot: the client wants new endpoints, but shipping them through the release pipeline is too slow.

* In any case, you need a logic layer between the client and the backend.
* And we already have one, right? Just write it in the gateway we already have...
* 🚨MEEC🚨 DON'T. DO. THAT.
* You'll get something:
  * hard to test
  * hard to maintain
  * slow to change.
* The gateway is the wrong place for product logic. 💀 I learned this the hard way.

* All you need is a Friend, a Best Friend Forever, a BFF 💖 

## What is a BFF?
* BFF stands for **Backend for Frontend**.
* A BFF is a small app that sits right in front of your backend. Still inside your network.
* In a BFF you can:
  * Merge: A client makes 1 request. The BFF makes several backend calls and merges the results into a single response.
  * Filter: Your Internal API might expose sensitive data, the BFF can filter it out, keep only what the client should see
  * Fast: play around fast with endpoints without touching the core product. It's safer — the main app stays intact and you skip the heavy pipeline.
  * Test: The BFF is a small, isolated app. Much easier to test than logic baked into a gateway.

## The trap of powerful Gateways

Yes. Some Gateways have some scripting capabilities that allow you to do some filtering and reshaping of the data
* Kong functions
* Apigee Service Callouts
* Tyk Virtual Endpoints 💀

But those are not meant to implement an "App". It's just Gateway configuration.

💀 This was exactly my mistake!

💀 TYK Virtual Endpoints was my trap. It's such a powerful feature — too good, even. So we abused it.

💀 That small JS machine had to handle GBs of data, complex filtering, and orchestration.

Hard to maintain, hard to iterate and hard to test.

But it was totally our fault. We never added a BFF.

## How to implement a BFF

BFF is a pattern, not a product. 
Keep it simple.
Use a normal HTTP app (Fastify/Hono/Nest/Go) that talks to backends, merges/filters, and returns the client shape.

## Gateway 💖 BFF

As you can see, a Gateway and a BFF are complementary. You need both to have a complete API strategy. The Gateway is the edge, the BFF is the app logic. 