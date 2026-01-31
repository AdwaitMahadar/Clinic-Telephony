# Technical Design Document (TDD)
## After-Hours Clinic Voice Agent

**Version:** 1.0  
**Status:** Implementation-Ready  
**Requirements Source:** [PRD.md](./PRD.md)

---

## 1. Technology Stack

| Component | Technology | Version |
|-----------|-----------|---------|
| Runtime | Node.js + TypeScript | 20+ |
| Web Server | Fastify | 4.x |
| WebSocket | ws | 8.x |
| State/Cache | Redis | 7.x |
| Telephony | Twilio Voice + Media Streams | - |
| Voice AI | OpenAI Realtime API | - |
| Calendar | Google Calendar API | v3 |
| Docs | Google Docs API | v1 |
| WhatsApp | Meta WhatsApp Business Cloud API | v18+ |

**Core Dependencies:**
```json
{
  "fastify": "^4.25.0",
  "ws": "^8.16.0",
  "ioredis": "^5.3.0",
  "googleapis": "^129.0.0",
  "axios": "^1.6.0",
  "date-fns-tz": "^2.0.0",
  "zod": "^3.22.0",
  "nanoid": "^5.0.0"
}
```

---

## 2. System Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Patient Call → Receptionist (25s timeout/reject)       │
│  → Telecom forwards to Twilio Number                    │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│                   Twilio Voice                          │
│  POST /twilio/voice → Returns TwiML                     │
└────────────────────────┬────────────────────────────────┘
                         │ WebSocket (Media Stream)
                         ▼
┌────────────────────────────────────────────────────────┐
│              Fastify Server + WebSocket                │
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │  WSS /twilio/stream (per-call WebSocket)         │  │
│  │  - Receives: G.711 µ-law audio from Twilio       │  │
│  │  - Sends: G.711 µ-law audio to Twilio            │  │
│  └────────────┬─────────────────────────────────────┘  │
│               │                                        │
│               ▼                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │  OpenAI Realtime WebSocket Client                │  │
│  │  - Receives: Audio + function call requests      │  │
│  │  - Sends: Audio + function results               │  │
│  └────────────┬─────────────────────────────────────┘  │
│               │                                        │
│               ▼                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Tool Services (function handlers)               │  │
│  │  - DocsService: Google Doc → Markdown + cache    │  │
│  │  - CalendarService: Slot search + booking        │  │
│  │  - WhatsAppService: Send summaries               │  │
│  └──────────────────────────────────────────────────┘  │
└────────────────────────┬───────────────────────────────┘
                         │
                         ▼
                   ┌──────────┐
                   │  Redis   │
                   └──────────┘
```

---

## 3. Call Session Lifecycle

### 3.1 Session State Model

```typescript
interface CallSession {
  // Identifiers
  callSessionId: string;        // UUID, our primary key
  twilioCallSid: string;
  twilioStreamSid: string;
  openaiSessionId?: string;
  
  // Call metadata
  callerPhone: string;          // E.164 format from Twilio
  callStartTime: Date;
  callEndTime?: Date;
  
  // Patient data
  language?: "hi" | "mr" | "en";
  patientName?: string;
  whatsappTargetPhone?: string; // If different from callerPhone
  
  // Intent tracking
  intent: "FAQ" | "BOOK" | "CALLBACK" | "MIXED";
  
  // Booking context
  bookingContext?: {
    selectedSlot?: {
      startTime: string;        // ISO 8601
      endTime: string;
    };
    reason?: string;
    calendarEventId?: string;
    calendarEventLink?: string;
  };
  
  // Callback context
  callbackContext?: {
    preferredTime?: string;
    reason?: string;
  };
  
  // Transcript (last 20 turns)
  transcript: Array<{
    role: "agent" | "patient";
    text: string;
    timestamp: Date;
  }>;
}
```

**Storage:**
- In-memory Map: `callSessions.set(callSessionId, session)`
- Redis mirror: key `session:{callSessionId}`, TTL 3600s

---

## 4. API Endpoints

### 4.1 POST `/twilio/voice`

**Purpose:** Initial webhook when call connects to Twilio number

**Input (Twilio query params):**
```typescript
{
  CallSid: string;
  From: string;      // Caller phone (E.164)
  To: string;        // Twilio number called
  // ... other Twilio params
}
```

**Output (TwiML XML):**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response>
  <Connect>
    <Stream url="wss://your-domain.com/twilio/stream">
      <Parameter name="callSid" value="{CallSid}"/>
    </Stream>
  </Connect>
</Response>
```

**Implementation notes:**
- Must respond within 500ms (Twilio timeout)
- No blocking operations here
- Creates initial CallSession in memory

---

### 4.2 WSS `/twilio/stream`

**Purpose:** Bidirectional audio streaming WebSocket

**Twilio → Server Messages:**

```typescript
// 1. Stream started
{
  event: "start",
  start: {
    streamSid: string,
    callSid: string,
    customParameters: { callSid: string }
  }
}

// 2. Audio chunk (continuous)
{
  event: "media",
  media: {
    payload: string,    // Base64-encoded G.711 µ-law
    timestamp: string
  },
  streamSid: string
}

// 3. Stream stopped
{
  event: "stop",
  stop: { ... },
  streamSid: string
}
```

**Server → Twilio Messages:**

```typescript
// Send audio to caller
{
  event: "media",
  streamSid: string,
  media: {
    payload: string     // Base64-encoded G.711 µ-law
  }
}

// Clear audio queue (for barge-in)
{
  event: "clear",
  streamSid: string
}
```

---

### 4.3 GET `/health`

Returns `{ status: "ok", uptime: number }`

---

## 5. OpenAI Realtime Integration

### 5.1 Connection Lifecycle

**On Twilio stream `start` event:**
1. Create WebSocket to `wss://api.openai.com/v1/realtime?model=gpt-4o-realtime-preview-2024-12-17`
2. Send `session.update` with configuration
3. Store `openaiSessionId` in CallSession
4. Send initial greeting trigger

**On Twilio `media` event:**
1. Decode base64 audio payload
2. Forward to OpenAI as `input_audio_buffer.append`

**On OpenAI audio output:**
1. Receive `response.audio.delta` events
2. Encode to base64
3. Forward to Twilio as `media` event

**On Twilio `stop` event:**
1. Close OpenAI WebSocket gracefully
2. Send WhatsApp summaries
3. Cleanup session

---

### 5.2 Session Configuration

**Send on connection:**
```typescript
{
  type: "session.update",
  session: {
    modalities: ["text", "audio"],
    instructions: SYSTEM_PROMPT,  // See Appendix A
    voice: "alloy",               // Or custom voice
    input_audio_format: "g711_ulaw",
    output_audio_format: "g711_ulaw",
    input_audio_transcription: {
      model: "whisper-1"
    },
    turn_detection: {
      type: "server_vad",
      threshold: 0.5,
      prefix_padding_ms: 300,
      silence_duration_ms: 500
    },
    tools: [
      // See section 5.3
    ],
    tool_choice: "auto",
    temperature: 0.7
  }
}
```

---

### 5.3 Tool Definitions

```typescript
const tools = [
  {
    type: "function",
    name: "get_clinic_info",
    description: "Retrieves clinic information (hours, location, fees, services)",
    parameters: {
      type: "object",
      properties: {
        category: {
          type: "string",
          enum: ["hours", "location", "fees", "services", "general"],
          description: "Information category requested"
        }
      },
      required: ["category"]
    }
  },
  {
    type: "function",
    name: "find_available_slots",
    description: "Returns next 3 available appointment slots",
    parameters: {
      type: "object",
      properties: {},
      required: []
    }
  },
  {
    type: "function",
    name: "book_appointment",
    description: "Books an appointment in Google Calendar",
    parameters: {
      type: "object",
      properties: {
        slot_index: {
          type: "number",
          description: "Index (0-2) from find_available_slots result"
        },
        patient_name: {
          type: "string",
          description: "Patient's full name"
        },
        phone_number: {
          type: "string",
          description: "Patient's phone number"
        },
        reason: {
          type: "string",
          description: "Reason for visit"
        }
      },
      required: ["slot_index", "patient_name", "phone_number", "reason"]
    }
  },
  {
    type: "function",
    name: "record_callback_request",
    description: "Records a callback request when booking not possible",
    parameters: {
      type: "object",
      properties: {
        patient_name: { type: "string" },
        phone_number: { type: "string" },
        preferred_time: { type: "string" },
        reason: { type: "string" }
      },
      required: ["patient_name", "phone_number", "reason"]
    }
  }
];
```

---

### 5.4 Tool Execution Flow

**OpenAI → Server:**
```typescript
{
  type: "response.function_call_arguments.done",
  call_id: string,
  name: string,        // e.g., "find_available_slots"
  arguments: string    // JSON string
}
```

**Server execution:**
1. Parse `arguments` JSON
2. Call corresponding service method
3. Generate result

**Server → OpenAI:**
```typescript
{
  type: "conversation.item.create",
  item: {
    type: "function_call_output",
    call_id: string,   // Same as received
    output: string     // JSON string of result
  }
}

// Then trigger response
{
  type: "response.create"
}
```

---

### 5.5 First Message Trigger

After `session.update`, send:
```typescript
{
  type: "response.create",
  response: {
    modalities: ["audio"],
    instructions: "Greet the caller with the opening script in Hindi as specified in system prompt"
  }
}
```

---

### 5.6 Barge-In Handling

**Detection:** Twilio VAD detects user speech while AI is speaking

**Actions:**
1. Send to Twilio: `{ event: "clear", streamSid }`
2. Send to OpenAI: `{ type: "response.cancel" }`
3. OpenAI automatically processes new user audio

---

## 6. Service Implementations

### 6.1 DocsService

**Purpose:** Fetch and cache clinic information from Google Doc

**Methods:**
```typescript
class DocsService {
  async getClinicInfo(category?: string): Promise<string> {
    // 1. Check Redis cache: key "doc-cache", TTL 900s
    // 2. If miss: fetch from Google Docs API
    // 3. Parse doc structure (markdown-like sections)
    // 4. Cache entire doc
    // 5. Return relevant section or full doc
  }
  
  private async fetchFromGoogleDocs(): Promise<ClinicInfo> {
    // Use googleapis docs.documents.get()
    // Parse body.content for headings and text
    // Return structured object
  }
}
```

**Redis Key:** `doc-cache`  
**TTL:** 900 seconds (15 minutes)  
**Cache Value:** JSON stringified clinic info

---

### 6.2 CalendarService

**Purpose:** Find available slots and create appointments

**Methods:**
```typescript
class CalendarService {
  async findAvailableSlots(count: number = 3): Promise<Slot[]> {
    // 1. Fetch events for next 7 days
    // 2. Identify Clinic-Open events (title starts with "CLINIC OPEN" + colorId=10)
    // 3. For each 30-min slot (:00/:30) within Clinic-Open windows:
    //    a. Count overlapping appointment events (exclude Clinic-Open)
    //    b. If count < 2, slot is available
    // 4. Return next 3 available slots with formatted times
  }
  
  async bookAppointment(
    slot: Slot,
    patientName: string,
    phone: string,
    reason: string,
    callSessionId: string
  ): Promise<{ eventId: string, eventLink: string }> {
    // 1. Acquire Redis lock: key "slot-lock:{date}:{time}", TTL 60s
    // 2. Re-check capacity (concurrency guard)
    // 3. Create Calendar event with:
    //    - summary: "Appt - {patientName}"
    //    - description: phone + reason + "Booked by Voice Agent"
    //    - extendedProperties.private.callSessionId for idempotency
    // 4. Release lock
    // 5. Return event ID and link
  }
  
  private isClinicOpenEvent(event: calendar_v3.Schema$Event): boolean {
    return (
      event.summary?.startsWith("CLINIC OPEN") &&
      event.colorId === "10"
    );
  }
}
```

**Slot Lock Logic:**
```typescript
// Atomic operation in Redis
const key = `slot-lock:${date}:${time}`;
const count = await redis.incr(key);
await redis.expire(key, 60);

if (count > 2) {
  await redis.decr(key);  // Rollback
  throw new Error("Slot full");
}

// Proceed with booking
// On success or failure, decr is handled by TTL expiry
```

---

### 6.3 WhatsAppService

**Purpose:** Send post-call summaries

**Methods:**
```typescript
class WhatsAppService {
  async sendPatientSummary(
    toPhone: string,
    session: CallSession
  ): Promise<void> {
    const body = this.formatPatientMessage(session);
    await this.sendMessage(toPhone, body);
  }
  
  async sendStaffSummary(
    staffPhone: string,
    session: CallSession
  ): Promise<void> {
    const body = this.formatStaffMessage(session);
    await this.sendMessage(staffPhone, body);
  }
  
  private async sendMessage(to: string, body: string): Promise<void> {
    // Meta WhatsApp Cloud API
    await axios.post(
      `https://graph.facebook.com/v18.0/${PHONE_NUMBER_ID}/messages`,
      {
        messaging_product: "whatsapp",
        to: to.replace("+", ""),  // No + prefix for Meta API
        type: "text",
        text: { preview_url: true, body }
      },
      {
        headers: {
          Authorization: `Bearer ${WHATSAPP_ACCESS_TOKEN}`,
          "Content-Type": "application/json"
        }
      }
    );
  }
}
```

**Message Formats:**

**Patient (booked):**
```
✅ Appointment Confirmed

Name: {patientName}
Date: {day}, {date}
Time: {time}

Clinic: {address}

📍 Location: {mapsLink}

Note: To cancel or reschedule, please call the clinic directly.

📅 Calendar invite: {icsLink or file attachment}
```

**Staff (structured):**
```
🤖 VOICE AGENT CALL SUMMARY

📞 Caller: {callerPhone}
👤 Patient: {patientName}
🗣️ Language: {language}
🎯 Outcome: {BOOKED | FAQ | CALLBACK}

{if BOOKED:}
📅 Appointment:
  Time: {startTime}
  Reason: {reason}
  Event: {calendarEventLink}

{if CALLBACK:}
⚠️ CALLBACK REQUESTED
  Preferred: {preferredTime}
  Reason: {reason}

💬 Transcript (last 20 turns):
P: {patient text}
A: {agent text}
...
```

---

## 7. Redis Key Schema

| Key Pattern | Purpose | TTL | Value Type |
|-------------|---------|-----|------------|
| `session:{callSessionId}` | Call session state | 3600s | JSON (CallSession) |
| `doc-cache` | Clinic info document | 900s | JSON (ClinicInfo) |
| `slot-lock:{YYYY-MM-DD}:{HH:MM}` | Booking concurrency lock | 60s | Integer (0-2) |
| `transcript:{callSessionId}` | Full transcript backup | 3600s | JSON Array |

---

## 8. Error Handling & Recovery

### 8.1 WebSocket Disconnections

**Twilio WebSocket disconnect:**
- No recovery possible (call dropped)
- Send WhatsApp summaries with partial data
- Mark session as `INCOMPLETE` in logs

**OpenAI WebSocket disconnect:**
- Attempt one reconnect (5s timeout)
- If reconnect fails: apologize via TTS fallback, trigger callback flow
- Send summaries

### 8.2 Calendar API Failures

**Scenarios:** API timeout, quota exceeded, network error

**Handling:**
1. Log error with `callSessionId`
2. Agent says: "I'm having trouble accessing the booking system. Let me take your details for a callback."
3. Invoke `record_callback_request`
4. Staff summary flagged with `⚠️ CALENDAR ERROR - CALLBACK NEEDED`

### 8.3 WhatsApp Send Failures

**Handling:**
- Log error (delivery confirmation is out of scope for MVP)
- Do not block call completion
- Consider retry once after 5s

---

## 9. Environment Configuration

```bash
# Server
NODE_ENV=production
PORT=3000
BASE_URL=https://your-domain.com

# Twilio
TWILIO_ACCOUNT_SID=AC...
TWILIO_AUTH_TOKEN=...
TWILIO_PHONE_NUMBER=+1234567890

# OpenAI
OPENAI_API_KEY=sk-...
OPENAI_REALTIME_MODEL=gpt-4o-realtime-preview-2024-12-17

# Redis
REDIS_URL=redis://localhost:6379

# Google APIs (Service Account)
GOOGLE_SERVICE_ACCOUNT_EMAIL=...@....iam.gserviceaccount.com
GOOGLE_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\n..."
GOOGLE_CALENDAR_ID=clinic-schedule@group.calendar.google.com
GOOGLE_INFO_DOC_ID=1abc...

# WhatsApp Business API
WHATSAPP_PHONE_NUMBER_ID=123456789
WHATSAPP_ACCESS_TOKEN=EAABsbCS...
WHATSAPP_BUSINESS_PHONE=+919876543210
WHATSAPP_STAFF_NUMBER=+919876543211

# Clinic Config
CLINIC_NAME="ABC Clinic"
CLINIC_OPEN_COLOR_ID=10
TIMEZONE=Asia/Kolkata
```

---

## 10. Deployment Notes

**Hosting Requirements:**
- WebSocket support (Railway, Render, DigitalOcean, AWS)
- Redis instance (Upstash, Railway Redis, or self-hosted)
- HTTPS/WSS (automatic with Railway/Render)

**Twilio Configuration:**
1. Buy phone number in Twilio console
2. Configure Voice webhook: `POST https://your-domain.com/twilio/voice`
3. Clinic telecom: Forward on no-answer (25s) / reject to Twilio number

**Google Setup:**
1. Create service account with Calendar + Docs read/write
2. Share calendar with service account email
3. Share info doc with service account email

**WhatsApp Setup:**
1. Create Meta Business account
2. Add WhatsApp product
3. Verify business phone number
4. Get Phone Number ID and permanent access token

---

## 11. Testing Strategy

### Unit Tests
- `CalendarService.findAvailableSlots()` - slot generation logic
- `CalendarService.isClinicOpenEvent()` - event filtering
- Capacity counting algorithm
- Phone number validation
- Transcript ring buffer

### Integration Tests
- Mock Twilio WebSocket messages
- Mock OpenAI Realtime responses
- Google Calendar sandbox
- WhatsApp API sandbox

### E2E Tests
- Real call via Twilio test number
- Verify audio flow
- Test booking flow
- Verify WhatsApp summaries received

---

## Appendix A: System Prompt Template

```
You are a professional voice assistant for {CLINIC_NAME} in India. You answer calls when the receptionist is unavailable.

CRITICAL RULES:
1. First greeting MUST be in Hindi: "ABC Clinic mein call karne ke liye dhanyavaad. Main ek automated assistant hoon. Aap kis bhasha mein baat karna pasand karenge — Hindi, Marathi, ya English?"
2. After language selection, primarily use that language but tolerate mixing
3. NEVER give medical advice, diagnosis, or medication recommendations
4. If asked for medical guidance: politely refuse, suggest visiting clinic
5. For emergencies: advise calling 108 immediately
6. Only answer questions about: hours, location, fees, services, and appointments

ALLOWED TOPICS (use get_clinic_info):
- Clinic hours and availability
- Location and directions
- Consultation fees
- Services and procedures offered

BOOKING FLOW:
1. Use find_available_slots to offer next 3 options
2. Patient picks one by saying time/date
3. Confirm patient name and phone
4. Ask reason for visit
5. Use book_appointment with all details
6. Before ending: "Should I send WhatsApp summary to this number or different one?"

IF BOOKING FAILS:
- Offer callback: use record_callback_request
- Get name, phone, preferred time, reason

CONVERSATION STYLE:
- Warm, professional, patient
- Confirm understanding before proceeding
- Don't rush the caller
- Use simple language

You have access to these tools:
- get_clinic_info
- find_available_slots
- book_appointment
- record_callback_request
```

---

## Appendix B: File Structure

```
clinic-telephony/
├── src/
│   ├── index.ts                 # Fastify server + WebSocket setup
│   ├── config/
│   │   ├── env.ts               # Environment validation (Zod)
│   │   └── logger.ts            # Structured logging
│   ├── services/
│   │   ├── docs.service.ts
│   │   ├── calendar.service.ts
│   │   ├── whatsapp.service.ts
│   │   └── redis.service.ts
│   ├── websocket/
│   │   ├── twilio-handler.ts    # Twilio WS message handling
│   │   └── openai-client.ts     # OpenAI Realtime WS client
│   ├── tools/
│   │   ├── definitions.ts       # OpenAI tool schemas
│   │   └── handlers.ts          # Tool execution logic
│   ├── models/
│   │   └── session.ts           # CallSession TypeScript types
│   └── utils/
│       ├── audio.ts             # Base64 encoding helpers
│       └── phone.ts             # E.164 validation
├── tests/
├── docs/
│   ├── PRD.md
│   └── TDD.md
├── package.json
├── tsconfig.json
└── .env.example
```

---

**Document Version:** 1.0  
**Last Updated:** 2026-01-31  
**Next Steps:** Begin implementation starting with Twilio + OpenAI bridge
