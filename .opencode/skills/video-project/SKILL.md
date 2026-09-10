---
name: video-project
description: TYK DevRel application video project. Use when working on the gateway vs BFF video, reviewing script drafts, or discussing video strategy. Trigger words: "video", "script", "TYK", "devrel", "application".
---

# Video Project Context

## Goal

3-5 min video for TYK Developer Relations Advocate application (Junior, Remote UK).

## Core message

"We used TYK in production. Here's when it's the right tool, and when it isn't."

Honest, learn-in-public framing. Not fake hype.

## Pablo's real experience with TYK

- Single-tenant SaaS (one AWS env per client, Docker)
- Chose TYK because: open source (no per-client license fee)
- Used it as: gateway in front of Scala/Akka backend
- Mistake: tried to use TYK for BFF work (filtering, merging, reshaping data)
- TYK is good at: auth, rate limiting, routing, policies, analytics
- TYK is bad at: multi-call orchestration, domain logic, rapid iteration
- Pablo did: endpoint implementation, docs, customer support, incident handling

## Recommended video structure

1. Cold open: mistake admission
2. The real problem
3. What people confuse (gateway vs BFF)
4. Split responsibilities table
5. Combined architecture diagram
6. Close: when to use TYK

## Files

- `context/tyk-offer.md` - Full job description
- `context/tyk-use-case.md` - Architecture and lessons learned
- `context/video-ideas.md` - All ideas with ratings

## Constraints

- Pablo writes the script himself (human, not AI-polished)
- This skill provides context only, does not write content
