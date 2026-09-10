# Gateway vs BFF — Video Project

Short video (3–5 min) for TYK Developer Relations Advocate application.

## Goal

Show real production experience with API gateways, honest reflection on misuse, and correct mental model for when to use a gateway vs a BFF.

## Video title (draft)

*"We put TYK where a BFF belonged. Here's what I learned."*

## Structure (draft)

1. Cold open: "We chose TYK for the wrong reason… and still learned the right lesson."
2. The real problem (single-tenant SaaS, slow Scala releases, JS/TS API dev, $0 license)
3. What people confuse: gateway ≠ place to write domain filter/merge logic
4. Split responsibilities: BFF for app logic, gateway for edge concerns
5. Combined architecture diagram
6. Close: "TYK is excellent at the edge. Don't force it to be your app."

## Context files

- `context/tyk-offer.md` — Full job description
- `context/technical-context.md` — What we actually built and why
- `context/video-ideas.md` — Ideas, ratings, and strategy
