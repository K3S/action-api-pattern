---
title: Why Headless
nav_order: 2
permalink: /why-headless/
---

# Why Headless
{: .no_toc }

1. TOC
{:toc}

---

## The problem with green screens as the API surface

For most of IBM i's history, the green screen *was* the API. If you wanted to add a supplier, you signed on, navigated to the supplier maintenance menu, hit F6, filled in the fields, and pressed Enter. The screen was the entry point, the validation surface, the workflow controller, and the output renderer all at once.

That worked because there was only one client: a 5250 terminal (or, later, a 5250 emulator running on a PC). The terminal was simple, predictable, and — critically — the only thing that ever talked to the program. The fact that the business logic lived inside the same RPG program as the display file logic didn't matter. There was nothing else to call it from.

That assumption is no longer true. In 2026, the same supplier maintenance logic might need to be invoked by:

- A React web application built for desktop browsers
- A mobile app a buyer uses while walking a warehouse
- A REST API consumed by a customer's ERP integration
- An MCP server exposing the operation as a tool to an AI agent
- A scheduled job that runs the same action across thousands of records overnight
- Another RPG program — perhaps a batch process — that wants to perform the action without staging through a screen

Six different clients. One business action. The traditional answer was to write it six times, or to write five wrappers around the green-screen program that fake keystrokes and scrape the resulting screens. Both are bad answers. The wrappers are fragile and slow. The rewrites duplicate the validation, the database access, and the business rules — and now you have six places to keep in sync.

The right answer is to separate the two things that were never really the same thing: the *interface* (how the user invokes the action) and the *action* (what actually happens). The interface can be a 5250 screen, a web form, a mobile screen, an HTTP request, or an AI tool call. The action is the same regardless. So the action should live in a program that doesn't know or care which interface called it.

That program is what we mean by *headless* RPG. It has no head — no screen, no user interaction, no presentation logic. It accepts parameters, performs the action, returns a result. Anything can call it.

## Why this matters in 2026

Three things changed around the IBM i platform in the last few years that make the headless approach not just useful but urgent.

**The web is no longer a side channel.** Ten years ago, a web frontend on top of an IBM i was a "nice to have" alongside the green screen. In 2026, for most distributors, the web frontend *is* the primary interface — used by buyers, customer service reps, and external customers — and the green screen is the secondary one used by power users and the IT team. If your business logic lives in a green-screen program, you are constantly playing catch-up: any change to the green-screen workflow has to be replicated in the web app, and any change to the web app has to find a way to call back into the green-screen logic.

**Mobile is real.** A buyer walking a warehouse with a tablet wants to be able to scan a code, see a supplier record, and make a change. That tablet cannot run a 5250 emulator. It can hit an HTTPS endpoint. If the supplier-update logic is locked inside `SPMAINT`, the tablet's only option is a slow, fragile screen-scraping wrapper. If the logic lives in `AR_UPDSUPL`, the tablet calls it directly through your web layer.

**AI agents are a first-class client now.** This is the new one. An AI agent — an LLM-powered automation — wants to perform business actions on behalf of a user. To do that safely, it needs typed, deterministic, well-described tools. The Model Context Protocol (MCP) standardizes this: an agent talks to an MCP server, and the MCP server exposes a set of tools the agent can call. Each tool is, fundamentally, a single business action with a known input schema and a known output schema.

If you have Action APIs, you already have MCP tools. You wrap each Action in an MCP tool definition and the agent can use it. If you only have green-screen programs, you have nothing to expose; you have to build a translation layer first. And the translation layer is going to look a lot like Action APIs anyway, except now you've built them under deadline pressure instead of as part of a deliberate architecture.

## The naive answer (and why it's wrong)

When teams hit this realization, the natural reaction is *we need to rewrite this in a modern stack*. Rewrite the supplier logic in Java, or Node, or Go, or whatever the architect favors that month. Sit on top of Db2 for i via JDBC. Talk to that from React. Done.

This is almost always a mistake, for three reasons.

First, the existing RPG program is correct. It's been running in production for ten or twenty or thirty years. It handles the edge cases the team has forgotten exist. It implements the business rules the original analyst wrote down on a whiteboard in 2003 and nobody has documented since. Throwing it away means rediscovering all of that from scratch, in a new language, under deadline pressure. The customer-facing failure modes during a rewrite of well-tested business logic are nearly always worse than the limitations of the existing system.

Second, the existing RPG program is fast. RPG running on the IBM i, talking to Db2 for i over RLA or embedded SQL, is one of the fastest data-access patterns ever devised. You will not match it from JDBC over the network. You will not match it from anywhere outside the box. The IBM i was designed as a single-level-store machine where the program and the database share an address space; that is a structural performance advantage that no rewrite recreates.

Third, the existing RPG program already exists. Engineering time is the scarcest resource in any IT organization. Spending it rewriting working code is, in nearly every case, the lowest-value way to use it. Spending it making the working code reachable from new clients is one of the highest.

The headless approach gets you the modernization win — new clients, new interfaces, new use cases — without any of the rewrite costs. The RPG stays. The screen goes away (eventually). What changes is how the RPG is *invoked*: not through DDS and a 5250 stream, but through a parameter list and a return envelope.

## What "headless" actually means

In web development, *headless* is a term of art. A headless CMS is a content management system that stores and serves content but doesn't render it; the rendering happens in a separate frontend (a Next.js site, a mobile app, whatever). A headless commerce platform is the same idea: cart and checkout logic exposed as APIs, with the storefront being a separate concern.

Headless RPG is the same pattern applied to IBM i. The RPG program holds the business logic and the database access. It exposes that logic through a parameter list and an error envelope. The "head" — the user interface — is a separate concern, possibly several separate concerns, each of them calling the same headless program through whatever transport they prefer.

The benefits compound:

- **One source of truth for business logic.** The validation rules, the database updates, the side effects — they all live in one program. Every client gets identical behavior.
- **Independent evolution of clients.** You can rebuild the React frontend without touching the RPG. You can add a mobile app without touching the RPG. You can expose new MCP tools without touching the RPG.
- **Testability.** A headless program is just a function with inputs and outputs. You can write a test program (`TS_DELSUPL`) that calls it with known parameters and asserts on the result. You can run those tests in a nightly job. You cannot do that with a green-screen program; the testing surface is the screen, and screens are notoriously hard to test.
- **Composability.** Headless programs can call other headless programs. Adding a supplier might internally call `AR_ADDLOCN` to create a default location. The composition is clean because each piece is a single-purpose program with a known contract.

## What this tutorial does

Over the next nine chapters, we'll do three things:

1. **Define the contract.** What every Action API looks like — the parameter conventions, the error envelope, the logging discipline, the test harness. This is the formal Action API Pattern. Once you know the contract, you can build any Action you want.
2. **Walk a real example.** We'll take Supplier Maintenance — a worked example loosely modeled on the K3S `SUPL` API family — and trace it from a hypothetical green-screen program to a full set of Actions: `AR_ADDSUPL`, `AR_DELSUPL`, `AR_UPDSUPL`, `AR_GETSUPL`, `AR_LSTSUPL`. You'll see the RPG, the CL wrappers, the test programs, and the calling code.
3. **Connect it to modern clients.** We'll show how the same Actions plug into a Mezzio PHP application (returning JSON to a React frontend) and into an MCP server (exposing the same operations as tools to an AI agent). The point is to see, concretely, that one well-designed RPG program serves all of them.

The pattern is small, opinionated, and battle-tested. It is also boring, in the best sense — once you internalize it, building a new Action becomes mechanical. Same parameters, same error envelope, same trio of programs, same test harness. The discipline is the value.

[Next: The Action API Pattern]({% link 02-action-api-pattern.md %}){: .btn .btn-primary }
