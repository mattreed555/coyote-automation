# Vision

Draft date: March 1, 2026

This is a working document. It is intentionally conceptual and provisional.

## Working Vision

Build a personal automation application for the local machine that helps users turn repeatable intentions into trustworthy background behavior.

The product is not primarily a chatbot, and it is not primarily a general-purpose "AI agent."

It is a personal operations layer: a place where useful work can be scheduled, remembered, inspected, constrained, and improved over time.

The core promise is simple:

- the user decides what should happen,
- the system makes that work legible, repeatable, and stateful,
- and the computer does more useful work without becoming opaque or reckless.

## What We Want It To Be

We want to build a tool that makes a personal computer feel more capable, more obedient, and more useful over time.

It should help users:

- automate recurring tasks,
- run useful work on a schedule,
- carry forward lightweight memory across runs when needed,
- understand what will happen before it happens,
- see what happened after it runs,
- and gradually build a system that reflects how they actually work.

The product should feel less like "talking to an assistant" and more like gaining a durable layer of operational capability.

## Product Shape

Conceptually, the product can serve two roles:

### 1. A direct user-facing application

The app gives people a first-party place to:

- create and manage scheduled, stateful automations,
- review what is allowed,
- review what can change external systems and what only reads,
- rely on app-owned capabilities like model access, state, memory, and safe HTML retrieval,
- inspect history and results,
- and keep control over a growing library of useful routines.

### 2. A trusted automation substrate and tool surface for other agents

The app can also act as the secure local system behind other AI tools.

In that model, users can bring the agent of their choice to:

- propose schedules,
- draft automations,
- help configure workflows,
- inspect existing automations,
- query status, history, and results,
- and request bounded changes,

while the app remains the trusted layer that owns:

- boundaries,
- core capabilities,
- memory,
- visibility,
- approval,
- scheduling,
- and execution policy.

This keeps the product focused on leverage and trust rather than trying to win by being the user's only assistant.

A key part of that trust model is that scripts and outside assistants should use app-owned capabilities, not raw service credentials, whenever possible. The app should mediate things like model access, state and memory access, and risky remote content retrieval so policy, logging, and sanitization remain centralized.

## Who It Is For

The first users are likely people who want more leverage from their computers than mainstream software usually provides.

They are often:

- technically oriented,
- curious about AI,
- motivated by workflow improvement,
- willing to invest some setup effort,
- and dissatisfied with repetitive computer work.

They do not necessarily want maximum autonomy.

They want dependable leverage: a system that saves time, compounds usefulness, and remains understandable.

## Why This Matters

There is a growing gap between what computers could do for a person and what most software safely lets them do.

Many current AI products are optimized for interaction and novelty. Fewer are optimized for dependable, scheduled, stateful, local usefulness.

This product aims to close that gap by making automation:

- more personal,
- more inspectable,
- more schedule-aware,
- more stateful,
- and more trustworthy.

If it works, the user should feel that their computer is no longer just a device they operate in real time. It becomes a system that can carry forward intentions on their behalf.

## Philosophy / Principles

These are draft principles and should be revised aggressively.

### 1. Trust before magic

The product should feel safe, legible, and controllable before it feels impressive.

### 2. Leverage over novelty

We should optimize for recurring real-world usefulness, not demo-friendly cleverness or abstract autonomy.

### 3. Time, memory, and action must be explicit

Schedules, retained memory, and side effects are central to the product and should never be hidden behind vague abstractions.

### 4. The user remains the author

Even when AI helps generate or refine automations, the user should still feel ownership, comprehension, and control.

### 5. Make invisible work visible

Automations, schedules, memory, approvals, side effects, and results should be legible enough that the user can build confidence over time.

### 6. Start narrow, deepen over time

A sharp first win is more valuable than broad but vague capability. Power should expand through use, not overwhelm at the start.

### 7. AI should clarify, not obscure

AI is valuable when it helps users express intent, understand tradeoffs, and improve workflows. It is less valuable when it replaces clear systems with ambiguity.

### 8. Extensions may extend capabilities, but they may not bypass the host

Plugins and extensions can make the system more powerful, but they should be mediated through the app's capability and policy layer rather than creating side doors around it.

### 9. Respect the computer as a personal environment

This product should treat the local machine as a place of agency, privacy, and craft, not just a host for another cloud service.

## Non-Goals (For Now)

This document does not assume that we are building:

- a general social collaboration platform,
- a full replacement for all existing scripting or automation tools,
- an always-on autonomous agent with open-ended authority,
- or a product whose main value comes from external messaging and communications channels.

Those directions may intersect later, but they should not define the core product early.

## Success, Conceptually

We are succeeding if the product starts to feel like a trusted personal utility:

- users rely on it for real recurring work,
- they trust it with recurring work that may accumulate context across runs,
- they understand what it is doing,
- they are comfortable letting it run in the background,
- and it becomes part of how they structure their computing life.

The long-term ambition is not just to help users automate isolated tasks.

It is to help them build a more capable relationship with their computers.
