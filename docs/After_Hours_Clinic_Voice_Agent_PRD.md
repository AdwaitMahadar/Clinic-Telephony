# After-Hours Clinic Voice Agent — PRD (Build-Ready, MVP)

**Goal:** A phone voice agent that answers when the clinic receptionist doesn’t pick up (or rejects the call), provides basic clinic info, and can book appointments by writing into the clinic’s **Google Calendar**. After every agent-answered call, it sends:
- a **patient WhatsApp summary** (clean summary)
- an **internal staff WhatsApp summary** (structured + extracted fields + last 20 turns)

This is **not** a full appointment management system. The receptionist continues operating as usual, using **Google Calendar** + WhatsApp summaries to review.

---

## 1) Core User Experience

### 1.1 When does the agent answer?
Patient calls the clinic’s number. If either:
- the call is **not answered for 25 seconds**, OR
- the receptionist **cuts/rejects** the call  
…then the call is routed to the agent (provider-agnostic). The handoff must feel **smooth** (minimal extra ring/wait).

### 1.2 Opening script (must include AI disclosure + language choice)
Spoken **in Hindi**, first line includes disclosure:
> “ABC Clinic mein call karne ke liye dhanyavaad. Main ek automated assistant hoon. Aap kis bhasha mein baat karna pasand karenge — Hindi, Marathi, ya English?”

After language selection, agent primarily uses that language but tolerates mixing (**flexible**).

### 1.3 End of call
Closing line is dynamic based on outcome (FAQ-only / booked / callback request).  
Always includes: “I will send a WhatsApp summary of this call.”

---

## 2) In-Scope Capabilities

### 2.1 Info answering (from Google Doc)
Agent answers only these categories:
- Hours and availability (clinic open times)
- Location (address, landmarks, map link)
- Consulting fees
- Services offered + common procedures/tests

**Source:** a structured **Google Doc** (see Section 8). Agent may paraphrase but must stay within doc content.

### 2.2 Appointment booking (Google Calendar as the system)
- Single doctor.
- Single Google Calendar: **“Clinic Schedule”**.
- Calendar contains:
  - **Clinic-Open events** (define working windows)
  - Appointment events (created by agent/humans)

Agent can:
1) Explain clinic-open hours (as needed)
2) Offer **next 3 available slots** (see scheduling rules)
3) On patient confirmation, create a Google Calendar event (30 min)
4) Send WhatsApp summaries + calendar invite to patient

### 2.3 Callback request (fallback)
If booking is not possible (calendar API error, no suitable slots, patient insists on unavailable exact time, etc.):
- agent collects a callback request (minimal fields)
- internal staff WhatsApp summary highlights callback needed

---

## 3) Out of Scope (MVP)
- Insurance/TPA
- Payments
- Medical advice / diagnosis / medication guidance
- Cancel/reschedule via agent
- Delivery failure handling (WhatsApp delivery failures are out of scope)
- Dedicated database / CRM / appointment dashboard

---

## 4) Non-Negotiable Safety Rules
- If asked for medication / diagnosis / clinical advice: **refuse** and recommend clinic visit. If emergency symptoms: advise calling **108** or nearest hospital.
- No collection of sensitive IDs (Aadhaar/PAN).
- Only store conversation record in WhatsApp internal summary (no DB).

---

## 5) Booking Rules (Scheduling Spec)

### 5.1 Slot grid
- Appointments only at **:00 or :30**.
- Clinic must ensure Clinic-Open blocks also start/end at :00/:30 (**enforced by clinic**).
- Appointment event duration = **30 minutes**.

### 5.2 Capacity
- Up to **2 appointments** per 30-minute slot.
- Capacity check is based on **overlap** within the 30-min window.
- Clinic-Open events must NOT count toward capacity.

Example: 6:00–6:30 window may have:
- 1 Clinic-Open event
- Appointment A
- Appointment B  
Total overlapping events = 3, but appointment count = 2 (slot full).

### 5.3 Search window
- Offer the **next 3 available** slots.
- Booking allowed **same-day**.
- Booking horizon: **7 days** into the future.

### 5.4 If patient asks for an exact time that’s unavailable
- First offer the next 3 available times.
- If the patient insists on the exact time → record callback request.

---

## 6) Google Calendar Event Spec

### 6.1 Identifying Clinic-Open events
Clinic-Open events are identified by BOTH:
- Title prefix (e.g., `CLINIC OPEN - ...`)
- Reserved Calendar colorId (agreed with clinic)

### 6.2 Appointment event title + fields
- **Title:** `Appt - <PatientName>`
- **Description includes:**
  - Patient phone
  - Reason for visit (collected only after booking confirmed)
  - Tag: `Booked by Voice Agent`
  - Optional notes (free text, if collected)
- Appointment event duration: **30 min**

### 6.3 No email invites
No patient email invite flow. Patient notification is via WhatsApp only.

---

## 7) WhatsApp Messaging Spec

### 7.1 Message recipients (always both)
After every agent-answered call:
1) **Patient message:** sent to caller’s number by default
2) **Internal staff feed:** a dedicated internal WhatsApp 1:1 chat/inbox (API-friendly)

### 7.2 Message sender spec
1) **Patient WhatsApp requirement:** All patient communication must be sent from the clinic’s official WhatsApp Business number to maintain a continuous conversation thread that receptionists can access and continue
2) **Internal summary requirement:** Internal call logs must be delivered as a WhatsApp message from the clinic’s WhatsApp Business number to a dedicated internal staff WhatsApp number (1:1). Messaging the clinic number to itself is not relied upon.

### 7.3 Patient number handling
During the call, agent says:
- “Summary will be sent on WhatsApp to this number. Would you like it sent to a different number instead?”

Defaults:
- If ignored/hung up → send to **current call number**
- Allow override with basic validation (10 digits; India number format rules in implementation)

### 7.4 Patient message content (clean summary only)
Patient message includes:
- Outcome (Info provided / Appointment booked / Callback requested)
- If booked: date/time, clinic address, cancellation instruction (“please call clinic”), and calendar invite link/file
- Key links (maps link, etc.) if relevant

No transcript snippet to patient.

### 7.5 Internal staff message content (structured + last 20 turns)
Internal message includes:
- **Extracted fields**
  - Caller number
  - Patient name (if captured)
  - Preferred WhatsApp number (if different)
  - Language chosen
  - Intent(s) (FAQ / booking / callback)
  - If booked: start/end time, event link (calendar URL), reason
  - If callback: preferred time window + reason
- **Call summary**
- **Transcript snippet:** last **20 turns** (agent+patient)

This internal message is the primary “record” (no DB).

---

## 8) Clinic Information Document (Google Doc)

### 8.1 Structure (template-based headings + Q/A)
Doc must follow a predictable structure:

- `## Clinic Name`
- `## Location`
- `## Hours`
- `## Fees`
- `## Services & Procedures`
- `## FAQs`
  - `Q: ...`
  - `A: ...`

Agent may paraphrase but must stay within doc content.

### 8.2 Refresh and caching
- Convert Google Doc → Markdown in backend
- Cache duration: **15 minutes**
- Fetch latest when cache expires

---

## 9) Conversation Flow (High-Level)

### 9.1 Common flow
1) Agent answers (AI disclosure + language choice)
2) Identify intent:
   - FAQ → answer from doc
   - Booking → slot offer
3) End call with dynamic closing
4) Send WhatsApp messages (patient + staff)

### 9.2 Booking flow (happy path)
1) Offer next 3 available slots
2) Patient picks one
3) Create Google Calendar appointment event
4) Ask reason for visit (only now)
5) Update event description with phone + reason + tag
6) Send WhatsApp messages + calendar invite

### 9.3 Calendar API failure / no slots
- Apologize
- Take callback request (minimal)
- Send internal staff summary flagged as CALLBACK NEEDED
- Send patient summary (callback requested)

---

## 10) Functional Requirements (MVP)

### FR-1 Call Answering & Routing
- Agent must answer when no-answer >= 25s OR call rejected.
- Handoff must not add noticeable delay beyond routing.

### FR-2 Language Selection
- Opening line spoken in Hindi, offers Hindi/Marathi/English.
- After selection, agent primarily uses that language but tolerates mixing.

### FR-3 Info Retrieval
- Load info doc (Google Doc → Markdown) with 15-min cache.
- Answer only within doc content.

### FR-4 Slot Discovery
- Only consider times inside Clinic-Open events.
- Only consider :00/:30 start times.
- For each 30-min window, allow up to 2 overlapping appointment events (excluding Clinic-Open).
- Return next 3 available slots within 7 days (including same-day).

### FR-5 Booking Creation
- Create a 30-min event on Google Calendar.
- Title: `Appt - <Name>`.
- Description includes phone, reason (post-confirm), agent tag, optional notes.

### FR-6 WhatsApp Summaries
- Always send patient summary and staff summary after agent-answered call.
- Allow override WhatsApp number with basic validation.

### FR-7 Callback Requests
- When booking fails or patient insists on unavailable time, capture callback request and highlight in staff summary.

### FR-8 Medical Advice Refusal
- If asked for meds/diagnosis: refuse; give clinic guidance; for emergencies: 108.

---

## 11) Non-Functional Requirements

- **Latency:** Slot offering should feel quick; calendar checks should complete in a few seconds.
- **Reliability:** Calendar writes must be idempotent (avoid duplicate events on retries).
- **Privacy:** No DB; no storing transcripts outside WhatsApp summaries.
- **Auditability:** Staff summary must contain enough detail to act without replaying the call.

---

## 12) Technical Architecture (Provider-Agnostic)

### Components
1) **Telephony Webhook Service**
   - Receives inbound call events + streams transcript (provider dependent)
2) **Agent Orchestrator**
   - Maintains conversation state per call
   - Calls: info retrieval + scheduling + calendar create + WhatsApp send
3) **Google Doc Fetcher**
   - Fetches doc, converts to markdown, caches 15 min
4) **Calendar Adapter**
   - Reads events, identifies Clinic-Open, counts overlaps, creates appointment
5) **WhatsApp Adapter**
   - Sends patient + staff messages

### No DB note
MVP stores no structured records except:
- Google Calendar appointment event description
- WhatsApp message bodies (patient + staff)

---

## 13) Edge Cases (Must Handle)

- Patient hangs up immediately after agent greeting → still send WhatsApp summaries.
- Patient chooses a different WhatsApp number because current number isn’t on WhatsApp.
- Calendar contains Clinic-Open event but also has appointment overlaps → capacity logic must ignore open-block.
- Two calls try to book same slot simultaneously → avoid creating >2 overlapping appointment events for that window.
- Calendar API error → callback request.
- Patient speaks mixed language despite choice → tolerate.

---

## 14) Acceptance Criteria (MVP Done)

1) For a missed/rejected call, agent answers within expected behavior and performs greeting with AI disclosure + language choice.
2) Agent answers info questions using Google Doc content without hallucinating unsupported details.
3) Agent offers only :00/:30 slots inside Clinic-Open windows.
4) Agent enforces capacity=2 appointments per 30-min window (excluding Clinic-Open).
5) Agent can create a valid Google Calendar event and include required fields.
6) After every agent-answered call, WhatsApp patient summary + internal staff summary are sent.
7) Internal summary includes extracted fields + last 20 turns.
8) If booking fails, callback request is captured and clearly flagged to staff.
9) If asked for medicine/diagnosis, agent refuses and provides safe guidance + 108 for emergencies.

---

## 15) Implementation Notes (so you don’t get burned)

- **Concurrency:** capacity=2 requires careful overlap counting + write-time recheck. Recheck just before creating the event.
- **Clinic-Open tagging:** enforce title prefix + colorId convention and document it for the clinic.
- **Idempotency:** telephony + WhatsApp providers may retry webhooks. Prevent duplicate sends and duplicate calendar events by using a per-call idempotency key.
- **Time zone:** Asia/Kolkata only.

---

## Appendix A — Suggested WhatsApp Templates

### Patient (booked)
- ✅ Appointment booked
- Name: <Name>
- Time: <Day, Date> <HH:MM>
- Clinic: <Address>
- Note: For changes/cancel, please call the clinic.
- Calendar: <ICS link or attachment>

### Patient (FAQ-only)
- ✅ Summary of your call with ABC Clinic
- Topics covered: <hours/location/fees/services>
- If you want to book an appointment, call again during clinic hours or request a callback.

### Staff (structured + transcript)
- **CALL SUMMARY (Agent)**
- Caller: +91...
- Language: ...
- Outcome: BOOKED / FAQ / CALLBACK
- Patient: ...
- Slot: ...
- Reason: ...
- Calendar event: <link>
- Notes: ...
- **Last 20 turns:**
  - P: ...
  - A: ...
