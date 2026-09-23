# Gateway vs BFF

## You need a Gateway ✔️
* You've got an amazing web app for selling... potatos
* The app is designed for humans, but you want to reach beyond the browser
* Then the ideas start coming in.
  * Ship a mobile app
  * Integrate with a 3rd party services
  * Provide access to bots and agents
* Either way you need a REST API
* Exposing endpoints can be dangerous: scrapers, brute force, noisy neighbors.
* First thing that comes to mind? A Gateway.
* A software layer between your client and your backend services where you can set up rate limits, auth, quotas.

## But don't you need something else?

- Perhaps your backend is really several microservices.
- Maybe you need to filter and reshape the data before the client sees it.
- Or say you want to protoype with endpoints skipping the release pipelien

* In any case, you need a **logic layer** between the client and the backend.
* And we already have one, right? Just write it in the gateway
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
  * Test: The BFF is a small, isolated app. Much easier to test than a gateway script

## The trap of powerful Gateways

Yes. Some Gateways have some scripting capabilities that allow you to do some filtering and reshaping of the data
* Kong functions
* Apigee Service Callouts
* Tyk Virtual Endpoints 💀

But those are not meant to implement an "App". It's just Gateway configuration.

💀 This was exactly my mistake!

💀 TYK Virtual Endpoints was my trap. It's such a powerful feature that we misused

💀 We forced it's small JS engine to to handle GBs of data, complex filtering, and orchestration.

This was hard to maintain, hard to iterate and hard to test.

But it was totally our fault. We never added a BFF.

## Gateway 💖 BFF

As you can see, a Gateway and a BFF are complementary. You need both to have a complete API strategy. The Gateway is the edge, the BFF is the app logic. 
