# Week 5 Progress

**Date:** 20 - 26 April 2026
**Track:** CKBuilder — Community Keeps Building (Nervos)

---

## Overview

This week was focused on the community outreach that came out of Retric's feedback last week — gather real developer pain points before locking in the API surface for the capstone. Rather than guessing what CKB builders need from an indexer service, the priority was to put the question to the people actually building.

---

## What I worked on

### Developer survey
- Wrote a survey targeting CKB developers to capture real pain points around on-chain data access
- Questions cover what data they query most often, how they currently get it (self-hosted node, Mercury, raw RPC, etc.), what they wish existed, and what would make them switch
- Shared the survey with the community to start collecting responses

### Forum post
- Wrote a post on the CKB forum introducing the project and the survey
- Framed it as a request for input rather than a launch announcement — the goal is signal, not hype
- Aim is to reach builders who aren't active in chat but read the forum

### GitHub issue
- Raised an issue on the relevant repo to surface the discussion in a place developers already track
- Links back to the survey and forum thread so feedback can flow into one place

---

## Key learnings
- Writing the survey forced me to be specific about the assumptions I'm making about the API — several questions exist precisely because I realised I didn't actually know the answer
- Splitting outreach across survey + forum + issue covers different developer habits — some respond to forms, some to long-form discussion, some only see things in their GitHub feed

---

## Blockers
- None on the work itself, but responses will take time to come in — Week 2 of the capstone roadmap (REST API) is paused until there's enough signal to inform the endpoint design

---

## Plan for next week
- Collect and review survey responses, forum replies, and issue comments
- Synthesise the feedback into a concrete list of must-have endpoints and query patterns
- Begin Week 2 of the capstone roadmap (REST API) once the input is informing the design rather than guessing
