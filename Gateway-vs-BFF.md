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

## The trap of Virtual Endpoints
But I can do that in the gateway itself! I can create virtual endpoints that will call the backend and filter/reshape the data.



## What is a BFF?
BFF = Backend for Frontend. A BFF is a service that sits between your client and your backend services. 

The goal of a BFF is to adapt your API to the specific needs, requirements and experience of a specific client. You could not care and just use an API that returns the same data for all clients, or you could have one general purpose API that allows you to filter the data that you need on different clients. Typically, I think that BFF will result in the cleanest solution to this problem.

It is responsible for orchestrating calls to multiple backend services, filtering and reshaping data, and implementing business logic that is specific to the needs of the client.

Can you build your UI views by making single API calls? If yes, then you do not need a bff. If single UI views are requiring 2 to 3 calls, maybe to the same API maybe to multiple API's, then you can alleviate this with a bff.




## What's the difference between a Gateway and a BFF?

## How to implement

Yes. BFF is a pattern, not a product.
Use a normal HTTP app (Fastify/Hono/Nest/Go) that talks to backends, merges/filters, and returns the client shape. Dedicated “BFF platforms” are rare; GraphQL or tRPC are optional, not required.