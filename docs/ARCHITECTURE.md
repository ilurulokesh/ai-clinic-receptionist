# Architecture Deep-Dive

## Call flow — step by step

```
1. Patient dials clinic number
2. Tough Tongue AI answers → greets in Telugu
3. AI collects: name, phone, preferred date, preferred time
4. AI calls Tool 1 (check_and_book_appointment) → POST to n8n Workflow A webhook
5. n8n checks Google Calendar FreeBusy
   ├── Slot FREE → creates calendar event → responds {status: "booked"}
   └── Slot BUSY → finds next free slot → responds {status: "suggest_new_slot", suggested_time: "14:30"}
6. AI relays result to patient in Telugu
   ├── "booked" → "మీ అపాయింట్‌మెంట్ నిర్ధారించబడింది"
   └── "suggest_new_slot" → "ఆ సమయం అందుబాటులో లేదు. 2:30కి వీలుందా?"
7. Patient confirms
8. AI asks payment preference (UPI / pay at clinic)
9. AI calls Tool 2 (send_confirmation) → POST to n8n Workflow B webhook
10. n8n:
    ├── IF UPI → Razorpay API → creates payment link
    ├── WhatsApp Business API → sends Telugu template message (with payment link if UPI)
    └── MSG91 → sends DLT-compliant SMS
11. AI tells patient: "WhatsApp మరియు SMS పంపబడింది. ధన్యవాదాలు!"
12. Call ends
```

---

## n8n Workflow A — Node Map

```
[Webhook In]
      │
      ▼
[Parse & Normalise Input]        ← Code node: extracts patient_name, phone,
      │                              requested_date/time; builds IST ISO strings
      ▼
[Google Calendar — FreeBusy]     ← Queries calendar for busy blocks in the slot window
      │
      ▼
[IF — Slot Free?]
  TRUE ──▶ [Google Calendar — Create Event] ──▶ [Respond: Booking Confirmed]
  FALSE ──▶ [Find Next Available Slot]      ──▶ [Respond: Suggest New Slot]
                  (Code node, scans +30min
                   windows up to clinic close)
```

### Response JSON shapes

**Booking confirmed:**
```json
{
  "status": "booked",
  "confirmed_time": "11:00",
  "confirmed_date": "2026-09-15",
  "patient_name": "Ravi Kumar",
  "event_id": "gcal_event_id_here",
  "message_to_speak": "Your appointment is confirmed for September 15th at 11 AM."
}
```

**Slot busy — suggest alternative:**
```json
{
  "status": "suggest_new_slot",
  "suggested_time": "11:30",
  "suggested_date": "2026-09-15",
  "patient_name": "Ravi Kumar",
  "message_to_speak": "Sorry, that slot is taken. Doctor is free at 11:30. Shall I book that?"
}
```

**No slots today:**
```json
{
  "status": "no_slots",
  "message_to_speak": "Sorry, there are no available slots for the rest of today. Would you like to book for tomorrow?"
}
```

---

## n8n Workflow B — Node Map

```
[Webhook In]
      │
      ▼
[Parse Booking Confirmation]
      │
      ▼
[IF — UPI Payment?]
  TRUE ──▶ [Razorpay — Create Payment Link]
                │
                ▼ (payment link in json.short_url)
  FALSE ──────────────────────────────────────┐
                                              ▼
                              [WhatsApp — Send Template]
                                              │
                                              ▼
                                    [MSG91 — Send SMS]
                                              │
                                              ▼
                                  [Respond — All Done]
```

---

## Webhook payload contracts

### Tool 1 inbound (Tough Tongue AI → Workflow A)
```json
{
  "patient_name": "Ravi Kumar",
  "patient_phone": "+919876543210",
  "requested_date": "2026-09-15",
  "requested_time": "11:00"
}
```

### Tool 2 inbound (Tough Tongue AI → Workflow B)
```json
{
  "patient_name": "Ravi Kumar",
  "patient_phone": "+919876543210",
  "confirmed_date": "2026-09-15",
  "confirmed_time": "11:00",
  "payment_mode": "upi"
}
```
`payment_mode` is either `"upi"` or `"pay_at_clinic"`.

---

## Latency budget

| Step | Typical time |
|------|-------------|
| Tough Tongue AI STT (speech to text) | 150–300ms |
| n8n webhook → Google Calendar → response | 300–600ms |
| Tough Tongue AI TTS (text to speech) | 100–200ms |
| **Total perceived latency** | **~600–1,100ms** |

This is well within conversational tolerance (< 1,500ms). For comparison, US-hosted platforms (Vapi) add 400–800ms of transatlantic round-trip on top of this.

---

## Security notes

- Webhook URLs are unguessable (UUID-based) but not authenticated. For production, add a shared secret header (`X-Webhook-Secret`) in Tough Tongue AI tool config and validate it in a Code node before processing.
- Store all tokens as n8n **workflow variables** or **credentials** — never hardcoded in expressions.
- Google OAuth consent screen must be published (not "Testing") for tokens to last > 7 days.
