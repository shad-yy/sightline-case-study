# Sightline: engineering case study

[sightlinetv.com](https://sightlinetv.com) answers one question: **where can I legally watch this, in my country, right now?** That covers a Premier League match, a Formula 1 race, a film or a series. The site sits on top of third-party data (sports fixtures, streaming availability, news) and a hand-verified table of broadcast rights.

The source is private. This page is about the problems that make a site like this hard to get right, which are the same problems any product built on other people's APIs runs into, and the strategies used to solve them.

**Stack:** Next.js 15 (App Router), React, TypeScript, Tailwind, Upstash Redis, Zod, Vitest, Playwright with axe accessibility scans, Cloudflare Workers (via OpenNext), GitHub Actions.

**Measured on 18 Nov 2026:** 1,591 Vitest tests in 133 files passing. 53 page routes and 39 API routes.

---

## The rule behind the design

**Telling someone to watch on a channel they can't actually get is worse than telling them nothing.** Most of the decisions below follow from that one sentence.

---

## Hard problem 1: "We don't know" needs somewhere to live

**The trap.** Code that carries availability as a list or a boolean has one value for three different situations: "we checked and there's nothing", "we never looked" and "the lookup failed". All three render as the same confident negative.

**Where it shows up.** A title index printed "No offers recorded" under every card for a visitor whose country hadn't been detected yet. That's an availability claim with no lookup behind it. A blackout page told visitors "the Saturday blackout does not apply in your country" when the country was unknown, and some of them would be in exactly the place where it does apply.

**The strategy.** Absence is modelled explicitly. Streaming availability has four states (`listed`, `none`, `unchecked`, `failed`) and the blackout rule has three (`applies`, `does-not-apply`, `country-unknown`). The order of the checks is part of the design: with no country there was no lookup, so nothing that came back can describe this visitor, and that check comes first. Each page renders each state differently, and tests pin every state on the pages where the mistake first appeared.

**What carries over:** whenever an empty result can mean "nothing" or "didn't ask", give "didn't ask" its own value.

## Hard problem 2: A fallback that can't admit it failed

**The trap.** `catch (err) { return null }` turns a timeout, a 500, a broken connection and malformed JSON into the same `null`. Every caller then reads `null` as "this doesn't exist".

**Where it shows up.** The fallback sports provider, the one that exists to answer when the primary can't, had exactly that catch. During an outage a race page returned 200 with a generic heading, no date and no venue. It wasn't an error page. It was a complete-looking page saying the race had no details.

**The strategy.** Faults and absences are separate return values from the client up. A 404 is a deliberate exception: that provider publishes event summaries only when an event is close, so a 404 for a later event is a real answer ("not published yet"), and treating it as a fault would break a legitimate fallback to the scheduled calendar. Pages that lose a third-party source degrade to the data the site owns and say what couldn't be checked. A provider failing never becomes a 404 for the visitor.

## Hard problem 3: Data that goes stale without telling anyone

**The trap.** Broadcast rights change hands on contract cycles, and nobody sends a notification. A table that was right in July can be wrong in September, and nothing in the code will notice.

**How it was found.** The rights rows were re-checked against the rights holders by hand, and some had already turned over. One pointed visitors in one country to a service that had stopped selling those rights weeks earlier.

**The strategy:** two mechanisms, both enforced in CI and not left to anyone's memory.
- **Rows expire on their own.** Every listing carries `validFrom` and `validUntil`, and every consumer reads `activeCompetitionRights()`, never the raw table. A listing disappears the day its cycle ends. A missing end date means "nobody has checked when this ends", not "runs for ever", and the code treats it that way.
- **A test that fails on a date.** Each competition carries the date it was last verified by hand. The build fails when a verification is older than 180 days, or when a listing is within 60 days of expiring, and the message names the row and what to check. A dashboard can be ignored. A red build can't, and it needs no extra service to keep running.

**What carries over:** if correctness depends on the calendar, make the calendar fail the build.

## Hard problem 4: Rate limits that fail in the right direction

The site has an inbound limiter for its visitors and an outbound budget for the APIs it calls, and the two fail in opposite directions on purpose. If Redis goes down, the inbound limiter fails **open**, so visitors aren't locked out. The outbound budget fails **closed** and counts in process memory, because when Redis is down the cache is gone too. Every page view becomes a cache miss, and one cold render of one page is 28 calls to the movie API. A refused call is reported as a fault, never as "no results".

That piece is public and runnable: [fail-closed-budget](https://github.com/shad-yy/fail-closed-budget), with property and mutation tests in CI.

## Hard problem 5: Being findable for a question people ask in a hundred ways

The product is only useful if it ranks for "where to watch X in Y". Each competition, country and title has its own server-rendered route with structured data, and the comparison and country pages are generated from the same verified rights table, so the copy can't drift from the data. The film and series title pages come from one source of availability data, read through a cache with time limits.

---

## How it's tested

- **1,591 unit and component tests** (Vitest), many written as regression tests named after the failure they pin, such as `espn-fault-vs-absence` and `match-detail-absence`.
- **End-to-end tests** (Playwright) on desktop Chrome and mobile Safari and Chrome, including axe scans against WCAG 2 A/AA.
- **CI on every push**, plus scheduled workflows: a dependency audit, an SEO health check and a production health probe.

*I set the architecture, write the specifications, review and test every change, and decide what ships. AI coding agents are the tools I direct for implementation.*
