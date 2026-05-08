---
title: Home
layout: home
nav_order: 1
description: "A pattern for decomposing IBM i green-screen applications into single-purpose, callable APIs that work for web, mobile, and AI agents."
permalink: /
---

# Action API Pattern
{: .fs-9 }

Decomposing IBM i applications into single-purpose, callable APIs.
{: .fs-6 .fw-300 }

[Get started](#start-here){: .btn .btn-primary .fs-5 .mb-4 .mb-md-0 .mr-2 }
[View on GitHub](https://github.com/K3S/action-api-pattern){: .btn .fs-5 .mb-4 .mb-md-0 }

---

## What this is

A pattern for taking the business logic out of green-screen RPG applications and packaging each individual action — add a supplier, delete an order, change a product source — as a self-contained, callable API. The same RPG program then serves a web app, a mobile client, an AI agent, or another RPG program. No coupling to a particular UI. No coupling to a particular caller.

This is the pattern King III Solutions Inc. (K3S) has used in our supply chain replenishment software since the late 2000s, and the pattern that powers our R7 web platform, our internal tools, and our MCP server. It's the same pattern that lets a 35-year-old RPG codebase plug directly into a modern React frontend or a Claude-powered automation without rewriting the business logic.

We call the formal name **Action API Pattern**. The colloquial framing — *headless RPG* — captures what's actually happening: you take the head (the green screen) off the program, and what's left is a body (the business logic) that any client can talk to.

## Who it's for

- IBM i developers who want to expose RPG logic to web, mobile, or AI consumers without rewriting it
- Teams modernizing legacy green-screen applications without throwing them away
- Engineers building MCP servers, internal tools, or REST APIs on top of an IBM i backend
- Anyone who has been told "you need to rewrite this in `<modern stack>` to make it modern" and suspects that isn't actually true

## What you'll build

Across this tutorial we walk a single example program — Supplier Maintenance — from a monolithic green-screen program to a set of single-purpose Action APIs callable from anywhere. By the end you'll have:

- A working `AR_DELSUPL` RPG program with its `AC_DELSUPL` CL wrapper and `TS_DELSUPL` test harness
- The contract you'll apply to every other Action you build
- A Mezzio PHP handler that calls it and returns JSON
- An MCP tool definition that exposes the same program to an AI agent
- A migration plan you can apply to your own legacy programs

## Start here

Begin with [Why Headless]({% link 01-why-headless.md %}) for the motivation, or skip ahead to [The Action API Pattern]({% link 02-action-api-pattern.md %}) if you want to start with the contract.

## Related K3S tutorials

This tutorial is part of a series of open-source IBM i guides published by K3S:

- **[RPG Tutorial](https://rpgtutorial.k3s.com)** — Learning RPGLE and CL with modern tooling. Start here if you're new to the language.
- **[IBM i AI Workers](https://ibmi-ai-workers.k3s.com)** — Calling LLMs at scale from RPG batch jobs.
- **Action API Pattern** — *You are here.*

The recommended reading order across the series is RPG Tutorial → Action API Pattern → AI Workers, but each can be read independently.

## Contributing

This site is open source. Code is MIT-licensed; prose is CC-BY-SA 4.0. [Open a PR](https://github.com/K3S/action-api-pattern) or use the *Edit this page on GitHub* link at the bottom of any page.
