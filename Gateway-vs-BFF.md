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

### 💀 Our case

We relied too much on the gateway scripting. Specifically the TYK virtual endpoints. That small JS machine had to deal with GB of data, complex filtering, and orchestration. It was a nightmare to maintain and iterate on. 

And the testing was another nightmare. We had to test the logic in Postman, which was outside the gateway. So we had to deploy the gateway to test the logic. And every time we wanted to change something, we had to deploy the whole gateway. 

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
