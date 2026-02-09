# Implementation Plan
## After-Hours Clinic Voice Agent

**Version:** 1.0  
**Status:** Ready for Execution  
**References:** [PRD.md](./PRD.md), [TDD.md](./TDD.md)

---

## Overview

This document provides a step-by-step implementation plan for building the clinic voice agent. Each step should be completed sequentially, with clear success criteria before moving to the next step.

**Total Estimated Time:** 4 weeks  
**Steps:** 20 tasks organized in 6 phases

---

## Phase 1: Project Foundation (Days 1-2)

### Step 1: Initialize Project Structure
**Objective:** Set up Node.js TypeScript project with dependencies

**Tasks:**
- [ ] Create project directory structure (see TDD Appendix B)
- [ ] Initialize `package.json` with dependencies:
  - `fastify`, `ws`, `ioredis`, `googleapis`, `axios`, `date-fns-tz`, `zod`, `nanoid`
  - Dev dependencies: `typescript`, `tsx`, `@types/node`, `@types/ws`
- [ ] Create `tsconfig.json` with strict mode
- [ ] Create `.env.example` with all required environment variables (see TDD Section 9)
- [ ] Create `.gitignore` (node_modules, .env, dist, etc.)
- [ ] Initialize Git repository

**Success Criteria:**
- [ ] `npm install` runs without errors
- [ ] `tsc --noEmit` compiles successfully
- [ ] Project structure matches TDD Appendix B

**Files to Create:**
```
package.json
tsconfig.json
.env.example
.gitignore
src/index.ts (empty entry point)
```

---

### Step 2: Environment Configuration & Validation
**Objective:** Load and validate environment variables using Zod

**Tasks:**
- [ ] Create `src/config/env.ts`
- [ ] Define Zod schema for all environment variables:
  - Server config (PORT, BASE_URL, NODE_ENV)
  - Twilio (ACCOUNT_SID, AUTH_TOKEN, PHONE_NUMBER)
  - OpenAI (API_KEY, MODEL)
  - Redis (REDIS_URL)
  - Google (SERVICE_ACCOUNT_EMAIL, PRIVATE_KEY, CALENDAR_ID, DOC_ID)
  - WhatsApp (PHONE_NUMBER_ID, ACCESS_TOKEN, BUSINESS_PHONE, STAFF_NUMBER)
  - Clinic (NAME, OPEN_COLOR_ID, TIMEZONE)
- [ ] Export validated config object

**Success Criteria:**
- [ ] Config loads from `.env` file
- [ ] Invalid environment variables throw clear errors
- [ ] TypeScript types auto-generated from Zod schema

**Files to Create:**
```
src/config/env.ts
```

---

### Step 3: Logging Setup
**Objective:** Configure structured logging with pino

**Tasks:**
- [ ] Install `pino` and `pino-pretty`
- [ ] Create `src/config/logger.ts`
- [ ] Configure log levels based on NODE_ENV
- [ ] Add helper methods: `logger.info()`, `logger.error()`, etc.
- [ ] Include context fields: `callSessionId`, `timestamp`

**Success Criteria:**
- [ ] Logger can be imported and used
- [ ] Logs output as JSON in production
- [ ] Logs are pretty-printed in development

**Files to Create:**
```
src/config/logger.ts
```

---

### Step 4: Redis Connection
**Objective:** Set up Redis client with connection handling

**Tasks:**
- [ ] Create `src/services/redis.service.ts`
- [ ] Initialize ioredis client with REDIS_URL
- [ ] Add connection event handlers (connect, error, close)
- [ ] Export singleton instance
- [ ] Add helper methods: `get()`, `set()`, `incr()`, `expire()`

**Success Criteria:**
- [ ] Redis connects successfully (test with local Redis or Upstash)
- [ ] Connection errors are logged
- [ ] Can set/get a test key

**Files to Create:**
```
src/services/redis.service.ts
```

**Test:**
```typescript
await redis.set('test', 'value', 10); // 10s TTL
const val = await redis.get('test');
console.log(val === 'value'); // true
```

---

## Phase 2: Core Services (Days 3-5)

### Step 5: Call Session Model
**Objective:** Define TypeScript interfaces and in-memory session management

**Tasks:**
- [ ] Create `src/models/session.ts`
- [ ] Define `CallSession` interface (see TDD Section 3.1)
- [ ] Create `SessionManager` class:
  - `createSession(callSid, callerPhone)` → returns CallSession
  - `getSession(callSessionId)` → retrieves from memory
  - `updateSession(callSessionId, updates)` → updates and syncs to Redis
  - `deleteSession(callSessionId)` → cleanup
- [ ] Implement Redis mirroring (save to `session:{callSessionId}`)

**Success Criteria:**
- [ ] Can create, retrieve, update, delete sessions
- [ ] Sessions persist to Redis with 3600s TTL
- [ ] TypeScript types are correct

**Files to Create:**
```
src/models/session.ts
```

---

### Step 6: Google Docs Service
**Objective:** Fetch clinic information from Google Doc (fresh on each call)

**Tasks:**
- [ ] Install `googleapis` package
- [ ] Create `src/services/docs.service.ts`
- [ ] Authenticate with service account (JWT)
- [ ] Implement `fetchFromGoogleDocs()`:
  - Use `docs.documents.get()` API
  - Parse `body.content` for headings (## Hours, ## Location, etc.)
  - Extract text under each section
- [ ] Implement `getClinicInfo(category?)`:
  - Fetch from Google Docs API (fresh on each call)
  - Return full doc or specific section
- [ ] Handle API errors gracefully

**Success Criteria:**
- [ ] Can fetch real Google Doc (create test doc with template structure)
- [ ] Parsed sections match expected structure
- [ ] Fresh fetch works correctly on each call

**Files to Create:**
```
src/services/docs.service.ts
```

**Test Doc Structure:**
```
## Clinic Name
ABC Clinic

## Hours
Monday-Saturday: 6 PM - 9 PM

## Location
123 Main Street, Pune

## Fees
Consultation: ₹500

## Services & Procedures
General checkup, Blood tests
```

---

### Step 7: Google Calendar Service - Part 1 (Slot Discovery)
**Objective:** Implement slot finding algorithm

**Tasks:**
- [ ] Create `src/services/calendar.service.ts`
- [ ] Authenticate with Google Calendar API (same service account)
- [ ] Implement `isClinicOpenEvent(event)`:
  - Check `summary.startsWith("CLINIC OPEN")`
  - Check `colorId === env.CLINIC_OPEN_COLOR_ID`
- [ ] Implement `findAvailableSlots(count = 3)`:
  - Fetch events for next 7 days
  - Identify Clinic-Open windows
  - Generate 30-min slots at :00/:30 within windows
  - For each slot, count overlapping appointments (exclude Clinic-Open)
  - If count < 2, mark as available
  - Return next 3 available with formatted time strings
- [ ] Use `date-fns-tz` for Asia/Kolkata timezone

**Success Criteria:**
- [ ] Create test calendar with Clinic-Open events
- [ ] Can fetch and identify Clinic-Open correctly
- [ ] Returns correct available slots
- [ ] Respects capacity=2 per slot

**Files to Create:**
```
src/services/calendar.service.ts
src/utils/datetime.ts (timezone helpers)
```

**Test Calendar Setup:**
```
Event 1: "CLINIC OPEN - Evening" (colorId=10)
  Feb 5, 6:00 PM - 9:00 PM

Event 2: "Appt - John" 
  Feb 5, 6:00 PM - 6:30 PM

Event 3: "Appt - Jane"
  Feb 5, 6:00 PM - 6:30 PM

Expected: 6:00-6:30 is FULL (2 appointments)
          6:30-7:00 is AVAILABLE (0 appointments)
```

---

### Step 8: Google Calendar Service - Part 2 (Booking)
**Objective:** Implement appointment creation with concurrency control

**Tasks:**
- [ ] Add `bookAppointment()` method to CalendarService:
  - Generate slot lock key: `slot-lock:{YYYY-MM-DD}:{HH:MM}`
  - Atomic Redis INCR on lock key
  - If count > 2, DECR and throw error
  - Set TTL 60s on lock key
  - Create Calendar event:
    - summary: `Appt - {patientName}`
    - description: phone + reason + "Booked by Voice Agent"
    - extendedProperties.private.callSessionId (idempotency)
    - 30-min duration
  - Return eventId and calendar link
- [ ] Handle duplicate booking attempts (check extendedProperties)

**Success Criteria:**
- [ ] Can create appointment in test calendar
- [ ] Concurrent bookings are prevented (test with parallel requests)
- [ ] Idempotency works (same callSessionId doesn't create duplicate)
- [ ] Event contains all required fields

**Files to Update:**
```
src/services/calendar.service.ts
```

---

### Step 9: WhatsApp Service
**Objective:** Send patient and staff summaries via WhatsApp Business API

**Tasks:**
- [ ] Create `src/services/whatsapp.service.ts`
- [ ] Implement `sendMessage(to, body)`:
  - POST to Meta WhatsApp Cloud API
  - Handle phone format (remove + prefix)
  - Include auth token in header
- [ ] Implement `formatPatientMessage(session)`:
  - Generate clean summary based on session.intent
  - If BOOKED: include appointment details + calendar link
  - If CALLBACK: confirmation message
  - If FAQ: topics covered
- [ ] Implement `formatStaffMessage(session)`:
  - Structured fields (caller, language, outcome)
  - If BOOKED: event link
  - If CALLBACK: flag with ⚠️
  - Last 20 turns from transcript
- [ ] Implement `sendPatientSummary(toPhone, session)`
- [ ] Implement `sendStaffSummary(staffPhone, session)`

**Success Criteria:**
- [ ] Messages send successfully to test WhatsApp numbers
- [ ] Patient message format matches TDD examples
- [ ] Staff message format matches TDD examples
- [ ] Links render with preview

**Files to Create:**
```
src/services/whatsapp.service.ts
src/utils/phone.ts (E.164 validation/formatting)
```

---

## Phase 3: HTTP Server & Twilio Integration (Days 6-8)

### Step 10: Fastify Server Setup
**Objective:** Create HTTP server with health check and Twilio webhook

**Tasks:**
- [ ] Update `src/index.ts`:
  - Initialize Fastify server
  - Add CORS if needed
  - Register routes
  - Start server on PORT
- [ ] Create `GET /health` endpoint:
  - Return `{ status: "ok", uptime: process.uptime() }`
- [ ] Create `POST /twilio/voice` endpoint:
  - Parse Twilio webhook params (CallSid, From, To)
  - Generate TwiML response (see TDD Section 4.1)
  - Create initial CallSession
  - Return XML response

**Success Criteria:**
- [ ] Server starts without errors
- [ ] `GET /health` returns 200
- [ ] `POST /twilio/voice` returns valid TwiML XML
- [ ] Can test with curl/Postman

**Files to Update:**
```
src/index.ts
```

**Files to Create:**
```
src/routes/health.ts
src/routes/twilio-voice.ts
```

---

### Step 11: Twilio WebSocket Handler - Basic Setup
**Objective:** Handle Twilio Media Stream WebSocket connection

**Tasks:**
- [ ] Create `src/websocket/twilio-handler.ts`
- [ ] Register WebSocket upgrade handler for `/twilio/stream`
- [ ] Handle WebSocket connection:
  - On `start` event: log stream started, extract streamSid and callSid
  - On `media` event: log audio received (don't process yet)
  - On `stop` event: log stream stopped, cleanup session
- [ ] Parse incoming JSON messages from Twilio
- [ ] Implement `sendAudioToTwilio(streamSid, base64Audio)` helper
- [ ] Implement `sendClearToTwilio(streamSid)` helper

**Success Criteria:**
- [ ] WebSocket connection established
- [ ] Can log Twilio events (start, media, stop)
- [ ] No crashes on connection/disconnection

**Files to Create:**
```
src/websocket/twilio-handler.ts
```

**Test:** Use Twilio CLI to simulate Media Stream events

---

### Step 12: OpenAI Realtime Client - Basic Setup
**Objective:** Create OpenAI Realtime WebSocket client

**Tasks:**
- [ ] Create `src/websocket/openai-client.ts`
- [ ] Implement `OpenAIRealtimeClient` class:
  - `connect()`: WebSocket to OpenAI Realtime API
  - `sendSessionUpdate(config)`: Configure session (see TDD Section 5.2)
  - `sendAudio(audioChunk)`: Send input_audio_buffer.append
  - `onAudioDelta(callback)`: Handle response.audio.delta
  - `onFunctionCall(callback)`: Handle function call requests
  - `sendFunctionResult(callId, result)`: Return function output
  - `close()`: Graceful shutdown
- [ ] Handle connection errors and reconnection logic

**Success Criteria:**
- [ ] Can connect to OpenAI Realtime API
- [ ] Can send session configuration
- [ ] Can receive connection confirmation
- [ ] Handles errors gracefully

**Files to Create:**
```
src/websocket/openai-client.ts
```

---

### Step 13: Bridge Twilio ↔ OpenAI Audio
**Objective:** Forward audio bidirectionally between Twilio and OpenAI

**Tasks:**
- [ ] Update `src/websocket/twilio-handler.ts`:
  - On `start`: Create OpenAIRealtimeClient instance
  - On `media`: Decode base64, forward to OpenAI via `sendAudio()`
  - On OpenAI `audio.delta`: Encode to base64, forward to Twilio
  - On `stop`: Close OpenAI connection
- [ ] Handle audio format (µ-law, no conversion needed)
- [ ] Store OpenAI client reference in session
- [ ] Add error handling for disconnections

**Success Criteria:**
- [ ] Audio flows from Twilio → OpenAI
- [ ] Audio flows from OpenAI → Twilio
- [ ] Can conduct basic voice conversation
- [ ] Session cleanup works on call end

**Files to Update:**
```
src/websocket/twilio-handler.ts
src/websocket/openai-client.ts
```

---

### Step 14: System Prompt & First Message
**Objective:** Configure OpenAI session with system prompt and trigger greeting

**Tasks:**
- [ ] Create `src/config/system-prompt.ts`:
  - Export system prompt template (see TDD Appendix A)
  - Inject clinic name from env
- [ ] Update session configuration to include system prompt
- [ ] Implement first message trigger:
  - After session.update, send response.create with instructions to greet
- [ ] Test greeting is spoken in Hindi with AI disclosure

**Success Criteria:**
- [ ] First message is spoken automatically when call connects
- [ ] Greeting matches PRD opening script (Hindi, AI disclosure, language choice)
- [ ] Agent waits for user response

**Files to Create:**
```
src/config/system-prompt.ts
```

---

## Phase 4: Tool Implementation (Days 9-12)

### Step 15: Tool Definitions
**Objective:** Define OpenAI Realtime tool schemas

**Tasks:**
- [ ] Create `src/tools/definitions.ts`
- [ ] Define tool schemas (see TDD Section 5.3):
  - `get_clinic_info`
  - `find_available_slots`
  - `book_appointment`
  - `record_callback_request`
- [ ] Export as array for session configuration

**Success Criteria:**
- [ ] Tool definitions match OpenAI Realtime format
- [ ] Can be imported and used in session config

**Files to Create:**
```
src/tools/definitions.ts
```

---

### Step 16: Tool Handlers - get_clinic_info
**Objective:** Implement clinic info retrieval tool

**Tasks:**
- [ ] Create `src/tools/handlers.ts`
- [ ] Implement `handleGetClinicInfo(params, session)`:
  - Call `DocsService.getClinicInfo(params.category)`
  - Return formatted result
  - Update session transcript if needed
- [ ] Handle errors (return error message to agent)

**Success Criteria:**
- [ ] Tool returns clinic info from Google Doc
- [ ] Fresh fetch occurs on each tool call
- [ ] Agent can answer "What are your hours?" type questions

**Files to Create:**
```
src/tools/handlers.ts
```

---

### Step 17: Tool Handlers - find_available_slots
**Objective:** Implement slot discovery tool

**Tasks:**
- [ ] Implement `handleFindAvailableSlots(params, session)`:
  - Call `CalendarService.findAvailableSlots(3)`
  - Store slots in session for later booking
  - Format slots with human-readable times (use date-fns-tz)
  - Return as JSON string for agent
- [ ] Handle no slots available (return appropriate message)

**Success Criteria:**
- [ ] Returns next 3 available slots
- [ ] Slot times are formatted correctly (day, date, time)
- [ ] Slots are stored in session state

**Files to Update:**
```
src/tools/handlers.ts
```

---

### Step 18: Tool Handlers - book_appointment
**Objective:** Implement appointment booking tool

**Tasks:**
- [ ] Implement `handleBookAppointment(params, session)`:
  - Retrieve selected slot from session by index
  - Validate all required fields (name, phone, reason)
  - Call `CalendarService.bookAppointment()`
  - Store booking result in session.bookingContext
  - Return success message with confirmation details
- [ ] Handle booking failures (capacity exceeded, API error)
- [ ] Update session intent to "BOOKED"

**Success Criteria:**
- [ ] Creates calendar event successfully
- [ ] Booking details stored in session
- [ ] Handles concurrent booking attempts correctly
- [ ] Returns clear confirmation to agent

**Files to Update:**
```
src/tools/handlers.ts
```

---

### Step 19: Tool Handlers - record_callback_request
**Objective:** Implement callback request capture

**Tasks:**
- [ ] Implement `handleRecordCallbackRequest(params, session)`:
  - Store callback details in session.callbackContext
  - Update session intent to "CALLBACK"
  - Return acknowledgment message
- [ ] No external API calls needed (just state update)

**Success Criteria:**
- [ ] Callback details stored in session
- [ ] Intent updated correctly

**Files to Update:**
```
src/tools/handlers.ts
```

---

### Step 20: Connect Tools to OpenAI Client
**Objective:** Wire tool execution into conversation flow

**Tasks:**
- [ ] Update `src/websocket/openai-client.ts`:
  - On `response.function_call_arguments.done` event:
    - Parse function name and arguments
    - Call corresponding handler from handlers.ts
    - Send function result back to OpenAI
    - Trigger response.create
- [ ] Add tool execution logging
- [ ] Handle tool execution errors

**Success Criteria:**
- [ ] Agent can call tools during conversation
- [ ] Tool results flow back to agent
- [ ] Agent responds appropriately based on tool results
- [ ] All 4 tools work end-to-end

**Files to Update:**
```
src/websocket/openai-client.ts
src/websocket/twilio-handler.ts
```

---

## Phase 5: End-to-End Flow (Days 13-15)

### Step 21: WhatsApp Summary Trigger
**Objective:** Send WhatsApp summaries on call completion

**Tasks:**
- [ ] Update Twilio `stop` event handler:
  - Retrieve final session state
  - Determine target WhatsApp number (session.whatsappTargetPhone or callerPhone)
  - Call `WhatsAppService.sendPatientSummary()`
  - Call `WhatsAppService.sendStaffSummary(STAFF_NUMBER)`
  - Handle send failures (log but don't block)
- [ ] Ensure transcript has last 20 turns populated
- [ ] Clean up session after summaries sent

**Success Criteria:**
- [ ] Patient receives summary WhatsApp message
- [ ] Staff receives detailed summary with transcript
- [ ] Summaries sent even if call ends abruptly
- [ ] Session cleaned up properly

**Files to Update:**
```
src/websocket/twilio-handler.ts
```

---

### Step 22: Transcript Tracking
**Objective:** Capture and store conversation transcript

**Tasks:**
- [ ] Update `OpenAIRealtimeClient` to capture transcripts:
  - On `conversation.item.input_audio_transcription.completed`: Save patient text
  - On `response.text.done`: Save agent text
  - Append to session.transcript array
- [ ] Implement ring buffer (keep last 20 turns only)
- [ ] Store in Redis for backup (`transcript:{callSessionId}`)

**Success Criteria:**
- [ ] Transcript captures both sides of conversation
- [ ] Ring buffer maintains last 20 turns
- [ ] Transcript available for WhatsApp staff summary

**Files to Update:**
```
src/websocket/openai-client.ts
src/models/session.ts
```

---

### Step 23: Language Selection Handling
**Objective:** Capture and respect language choice

**Tasks:**
- [ ] Parse first user response for language choice
- [ ] Store in `session.language` ("hi" | "mr" | "en")
- [ ] System prompt already handles language switching
- [ ] Log language selection

**Success Criteria:**
- [ ] Language captured from first user response
- [ ] Agent continues in selected language
- [ ] Language appears in staff WhatsApp summary

**Files to Update:**
```
src/websocket/twilio-handler.ts (or create language detection helper)
```

---

### Step 24: Barge-In Implementation
**Objective:** Handle user interruptions gracefully

**Tasks:**
- [ ] Detect when user speaks during agent speech
- [ ] Send `clear` event to Twilio (flush audio queue)
- [ ] Send `response.cancel` to OpenAI
- [ ] Log barge-in events

**Success Criteria:**
- [ ] User can interrupt agent mid-sentence
- [ ] Agent stops speaking and listens
- [ ] Conversation continues naturally

**Files to Update:**
```
src/websocket/twilio-handler.ts
src/websocket/openai-client.ts
```

---

## Phase 6: Testing & Deployment (Days 16-20)

### Step 25: Unit Tests
**Objective:** Test core business logic

**Tasks:**
- [ ] Install `vitest`
- [ ] Create tests for:
  - `CalendarService.isClinicOpenEvent()`
  - `CalendarService.findAvailableSlots()` with mock data
  - Slot capacity counting algorithm
  - Phone number validation/formatting
  - Transcript ring buffer logic
- [ ] Mock external APIs (Google, WhatsApp)

**Success Criteria:**
- [ ] All unit tests pass
- [ ] Code coverage > 70% for services

**Files to Create:**
```
tests/unit/calendar.test.ts
tests/unit/docs.test.ts
tests/unit/phone.test.ts
```

---

### Step 26: Integration Testing
**Objective:** Test service integrations

**Tasks:**
- [ ] Test with real Google Calendar sandbox
- [ ] Test with real Google Doc (test doc)
- [ ] Test with WhatsApp API sandbox/test number
- [ ] Test Twilio Media Stream with test call
- [ ] Test OpenAI Realtime with sample audio

**Success Criteria:**
- [ ] Can create real calendar events
- [ ] Can fetch real Google Doc
- [ ] Can send real WhatsApp messages
- [ ] Audio flows through full pipeline

---

### Step 27: End-to-End Testing
**Objective:** Test complete call flows

**Tasks:**
- [ ] Test FAQ-only flow:
  - Call → Ask about hours → Hang up → Verify WhatsApp summaries
- [ ] Test booking flow (happy path):
  - Call → Request appointment → Select slot → Book → Verify calendar + WhatsApp
- [ ] Test callback flow:
  - Call → Request unavailable time → Callback request → Verify staff summary flagged
- [ ] Test concurrent bookings (prevent >2 in same slot)
- [ ] Test edge cases:
  - Hang up during greeting
  - Ask for medical advice (verify refusal)
  - Alternative WhatsApp number
  - Calendar API failure

**Success Criteria:**
- [ ] All PRD acceptance criteria met (Section 14)
- [ ] Edge cases handled gracefully
- [ ] No crashes or data loss

---

### Step 28: Error Handling & Logging Review
**Objective:** Ensure robust error handling

**Tasks:**
- [ ] Review all try-catch blocks
- [ ] Add appropriate error logging
- [ ] Test error scenarios:
  - Redis disconnection
  - Google API timeout
  - OpenAI disconnection
  - WhatsApp send failure
- [ ] Ensure errors don't crash server
- [ ] Verify user-facing error messages are clear

**Success Criteria:**
- [ ] All errors logged with context (callSessionId)
- [ ] Server remains stable under error conditions
- [ ] User gets appropriate feedback

---

### Step 29: Production Configuration
**Objective:** Prepare for deployment

**Tasks:**
- [ ] Create production `.env` with real credentials
- [ ] Set up production Redis (Upstash or Railway Redis)
- [ ] Configure real Twilio phone number
- [ ] Configure clinic's real Google Calendar (share with service account)
- [ ] Configure clinic's real WhatsApp Business API
- [ ] Set up error monitoring (optional: Sentry)
- [ ] Configure logging level for production

**Success Criteria:**
- [ ] All production credentials configured
- [ ] Service accounts have correct permissions
- [ ] Calendar and Doc shared properly

---

### Step 30: Deployment
**Objective:** Deploy to production environment

**Tasks:**
- [ ] Choose hosting (Railway, Render, or DigitalOcean)
- [ ] Set up deployment pipeline (GitHub → Railway/Render)
- [ ] Configure environment variables in hosting dashboard
- [ ] Deploy and verify:
  - Health check endpoint accessible
  - WebSocket connections work (WSS)
  - Can receive test call
- [ ] Configure Twilio webhook URL to production endpoint
- [ ] Set up call forwarding from clinic number to Twilio number:
  - Forward-on-no-answer: 25 seconds
  - Forward-on-reject: immediate
- [ ] Create runbook for common issues

**Success Criteria:**
- [ ] Application deployed and running
- [ ] Can receive real calls through Twilio
- [ ] All integrations work in production
- [ ] Clinic staff can test and confirm

**Deployment Checklist:**
```
[ ] Railway project created
[ ] GitHub repo connected
[ ] Environment variables set
[ ] Redis add-on provisioned
[ ] First deployment successful
[ ] Health check returns 200
[ ] Twilio webhook configured
[ ] Test call completes successfully
[ ] WhatsApp summaries delivered
[ ] Calendar event created
```

---

## Phase 7: Handoff & Documentation (Day 21)

### Step 31: Create Operations Guide
**Objective:** Document how to operate and troubleshoot the system

**Tasks:**
- [ ] Write `OPERATIONS.md`:
  - How to check if system is running
  - How to view logs
  - How to update clinic info doc
  - How to add Clinic-Open events
  - Common issues and fixes
  - Emergency contacts
- [ ] Write `CLINIC-SETUP.md`:
  - Calendar setup instructions
  - WhatsApp Business setup
  - Call forwarding configuration
  - How to review call summaries

**Files to Create:**
```
docs/OPERATIONS.md
docs/CLINIC-SETUP.md
```

---

### Step 32: Final Review & Launch
**Objective:** Verify system ready for production use

**Tasks:**
- [ ] Conduct final end-to-end test with clinic staff
- [ ] Train receptionist on WhatsApp summaries
- [ ] Verify call forwarding configuration
- [ ] Monitor first 10 real calls closely
- [ ] Collect feedback and document issues
- [ ] Schedule post-launch review meeting

**Success Criteria:**
- [ ] Clinic staff can use the system
- [ ] Real patient calls are handled successfully
- [ ] No critical issues in first 24 hours

---

## Appendix: Quick Reference

### Dependencies Summary
```json
{
  "dependencies": {
    "fastify": "^4.25.0",
    "ws": "^8.16.0",
    "ioredis": "^5.3.0",
    "googleapis": "^129.0.0",
    "axios": "^1.6.0",
    "date-fns-tz": "^2.0.0",
    "zod": "^3.22.0",
    "nanoid": "^5.0.0",
    "pino": "^8.16.0",
    "pino-pretty": "^10.2.0"
  },
  "devDependencies": {
    "typescript": "^5.3.0",
    "tsx": "^4.7.0",
    "@types/node": "^20.10.0",
    "@types/ws": "^8.5.0",
    "vitest": "^1.0.0"
  }
}
```

### Testing Checklist
- [ ] Unit tests pass
- [ ] Integration tests pass
- [ ] All PRD acceptance criteria met
- [ ] FAQ flow works
- [ ] Booking flow works
- [ ] Callback flow works
- [ ] Medical advice refused correctly
- [ ] WhatsApp summaries sent
- [ ] Calendar events created
- [ ] Concurrent bookings prevented
- [ ] Edge cases handled

### Pre-Launch Checklist
- [ ] Production environment configured
- [ ] All credentials valid
- [ ] Calendar shared with service account
- [ ] Doc shared with service account
- [ ] WhatsApp Business API approved
- [ ] Twilio webhook configured
- [ ] Call forwarding tested
- [ ] Staff trained
- [ ] Monitoring enabled
- [ ] Runbook created

---

**Document Version:** 1.0  
**Last Updated:** 2026-01-31  
**Next Action:** Begin Step 1 - Initialize Project Structure
