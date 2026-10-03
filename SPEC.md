# Project Spec: Urbannest Voice Agent

An AI sales agent for real estate lead follow-up, built as a text-first agent that is later upgraded to voice.

> **Status:** Spec v1, planning phase
> **Company (fictional):** Urbannest Realty
> **Primary success metric:** Slot booking rate

---

## 1. Goal

Build an outbound agent that follows up with leads who previously enquired about a specific flat. The agent:

1. Confirms it is an AI and states why it is calling.
2. Checks whether the lead is still interested in the property.
3. Answers follow-up questions using only verified property data.
4. Books a slot with a human supervisor (site visit or call), or schedules a callback.
5. Ends politely and logs the outcome.

The project emphasises **reliability, evaluation, and compliance** over a flashy demo.

---

## 2. Scope

| | Weeks 1-3 | Later |
|---|---|---|
| Text-only agent | In | |
| State machine and prompts | In | |
| RAG over property data and FAQ | In | |
| Tool calling | In | |
| PostgreSQL logging | In | |
| Guardrails | In | |
| Simulated-customer evaluation | In | |
| Voice (STT, TTS, barge-in) | | Weeks 4-5 |
| Telephony (Twilio or Exotel) | | After voice |
| Hinglish support | | Stretch goal |

All data is **synthetic**. No real leads, numbers, or recordings are used.

---

## 3. Data model

| Table | Key fields |
|---|---|
| `leads` | id, name, phone, property_id, enquiry_date, consent_status, dnd_flag, preferred_language |
| `properties` | id, name, locality, bhk, area_sqft, price, floor, possession_date, amenities, nearby_landmarks |
| `slots` | id, datetime, type (site_visit or supervisor_call), is_booked |
| `calls` | id, lead_id, started_at, outcome, final_stage |
| `turns` | call_id, turn_no, speaker, text, stage, tool_called |
| `opt_outs` | lead_id, timestamp, reason |

**Seed data:** about 30 leads, 3 properties, and a detailed FAQ document per property (price breakdown, maintenance charges, parking, loan assistance, possession timeline, and so on). The FAQ should contain a few **deliberate gaps** so the "I don't know" behaviour can be tested.

---

## 4. Call flow (state machine)

Code controls the stage. The LLM controls the wording inside each stage.

| Stage | Purpose |
|---|---|
| `GREET_DISCLOSE` | Introduce as an AI assistant from Urbannest, name the property enquired about, ask if now is a good time |
| `QUALIFY` | Confirm continued interest, then ask 2-3 questions (budget range, timeline, BHK need) |
| `ANSWER` | Handle cross-questions using only the property sheet and FAQ (RAG) |
| `CLOSE` | If interest is shown, offer a slot with the supervisor (site visit or call) |
| `BOOK_OR_CALLBACK` | Confirm a slot, or schedule a callback if the lead is busy or hesitant |
| `END` | Polite goodbye, outcome logged |

### Global interrupts (active at any stage)

- **Opt-out** ("stop calling", "remove me"): call `mark_opt_out`, apologise, end immediately.
- **Wants a human**: offer a supervisor slot or callback.
- **Wrong person or bad time**: offer a callback or end the call.

---

## 5. Tools

| Tool | Purpose |
|---|---|
| `search_property_info(query)` | RAG lookup over the property sheet and FAQ |
| `check_availability(date_range)` | Return open supervisor slots |
| `book_slot(lead_id, slot_id, type)` | Confirm a booking |
| `schedule_callback(lead_id, time)` | Log a callback request |
| `mark_opt_out(lead_id, reason)` | Add the lead to the opt-out list |
| `log_outcome(call_id, outcome)` | Save the final call result |

---

## 6. Guardrails

- **Facts only from the data.** If a price, possession date, charge, or any other figure is not in the property data, the agent says a supervisor will confirm it. It never guesses.
- **No guarantees.** No claims about appreciation, rental returns, or scarcity ("selling out soon"). No pressure tactics.
- **No negotiation.** Discounts and loan approvals are escalated to the supervisor.
- **AI disclosure in the first turn**, and an honest answer if asked later.
- **Pre-call checks:** consent status, DND flag, and calling-hours window (a configurable 9am-8pm check). Verify current rules for the region before any real-world use.
- **Immediate opt-out handling**, with the number excluded from all future calls.

---

## 7. Evaluation

### Metrics

| Metric | Definition |
|---|---|
| **Booking rate (primary)** | Calls ending in a confirmed slot / completed calls (excluding wrong-person calls and opt-outs) |
| Callback rate | Calls ending in a scheduled callback / completed calls |
| Opt-out rate | Opt-outs / completed calls |
| Hallucination rate | Share of agent statements about the property that contradict or are absent from the data (LLM-as-judge, validated by hand) |
| Pushiness score | Judge rating from 1 to 5 of how pressuring the agent was |
| Turns to outcome | Average conversation length |

**Counting rule:** a booking only counts if the lead explicitly agreed to a specific slot. The pushiness score exists to catch an agent that inflates the booking rate through pressure.

### Simulated customer personas

1. **Ready buyer:** interested, asks about price and visits.
2. **Curious researcher:** asks many questions, books slowly.
3. **Budget-limited:** the flat is above their range.
4. **Busy:** wants a callback.
5. **Hostile:** annoyed at the call, or asks to stop.
6. **Tricky:** asks things not in the data (for example, "will the price rise next year?") to test hallucination.

### Method

- Each persona is played by an LLM that converses with the agent automatically.
- Run at least 20 conversations per persona for each agent version.
- Change one variable at a time (prompt wording, state logic, retrieval settings) and compare results.
- Validate the hallucination judge against about 20 hand-checked transcripts before trusting its scores.

---

## 8. Architecture (planned)

```
Lead DB (PostgreSQL)
        |
   Pre-call checks (consent, DND, hours)
        |
        v
 +--------------------------------------+
 |  Agent core                          |
 |  State machine + LLM (tool calling)  |
 |      |                  |            |
 |  RAG (pgvector)     Tools (slots,    |
 |  property + FAQ      callback,       |
 |                      opt-out)        |
 +--------------------------------------+
        |
        v
 Call logs + outcomes (PostgreSQL)
        |
        v
 Evaluation harness (simulated customers + judge + metrics)
```

In the voice phase, STT and TTS are added on either side of the agent core.

---

## 9. Tech stack (planned)

- **Language:** Python
- **LLM:** streaming API with function calling
- **Database:** PostgreSQL with pgvector
- **Orchestration (voice phase):** Pipecat or LiveKit Agents
- **Evaluation:** custom harness with LLM-as-judge
- **Synthetic data:** Faker

---

## 10. Definition of done (weeks 1-3)

- [ ] Text agent runs full calls through all stages
- [ ] RAG answers property questions and declines when data is missing
- [ ] All six tools implemented and logged
- [ ] Opt-out and AI-disclosure guardrails verified
- [ ] At least 100 simulated calls run automatically
- [ ] Results table comparing two or more agent versions
- [ ] Hallucination judge validated on about 20 hand-checked transcripts
- [ ] README with architecture diagram and results table

---

## 11. Roadmap

- [ ] **Phase 0:** Spec and synthetic data
- [ ] **Phase 1:** Text agent (state machine, tool calling)
- [ ] **Phase 2:** RAG and database logging
- [ ] **Phase 3:** Simulated-customer evaluation
- [ ] **Phase 4:** Browser voice (STT, TTS, VAD, barge-in) with latency measurements
- [ ] **Phase 5:** Telephony integration
- [ ] **Phase 6:** Compliance layer and transcript PII redaction

---

## 12. Out of scope and ethics

- This project is a demonstration. It does not place calls to real leads.
- Any real deployment would require verified consent, DND compliance, recording consent, and review of local telemarketing and AI-voice regulations.
- The agent must not mislead callers about being human or make financial promises.
