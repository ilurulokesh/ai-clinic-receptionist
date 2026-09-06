# 🏥 AI Clinic Receptionist

A production-ready Telugu-language AI voice receptionist for small clinics in Andhra Pradesh / Telangana. Patients call a number, speak naturally in Telugu, and the AI handles appointment booking, availability checking, payment collection, and WhatsApp + SMS confirmation — with zero human intervention.

---

## Architecture

```
Patient phone call
      │
      ▼
Tough Tongue AI  ─── Telugu voice (low-latency, India-hosted)
      │
      ├── Custom Tool: check_and_book_appointment
      │         │
      │         ▼
      │    n8n Workflow A
      │         │
      │         ├── Google Calendar FreeBusy API (check slot)
      │         ├── IF free → Create Calendar Event
      │         └── IF busy → Find next available slot
      │
      └── Custom Tool: send_confirmation
                │
                ▼
           n8n Workflow B
                │
                ├── Razorpay → UPI Payment Link (optional)
                ├── WhatsApp Business API → Telugu template message
                └── MSG91 → DLT-compliant SMS
```

---

## Features

- 🎤 **Telugu voice** — natural conversation in Telugu (Coastal Andhra, Telangana dialects)
- 📅 **Live calendar check** — checks real availability before confirming
- 🔄 **Smart rescheduling** — if slot is taken, suggests next available time
- 💳 **UPI payment** — Razorpay link sent via WhatsApp if patient chooses online payment
- 📱 **WhatsApp + SMS confirmation** — dual-channel confirmation in Telugu
- 🔒 **India-compliant SMS** — MSG91 with DLT-registered sender ID and template
- ⚡ **Low latency** — Tough Tongue AI is India-hosted, avoiding US round-trip delay

---

## Tech Stack

| Layer | Tool |
|-------|------|
| Voice AI | [Tough Tongue AI](https://toughtongueai.com) |
| Workflow automation | [n8n](https://n8n.io) (self-hosted or cloud) |
| Calendar | Google Calendar API |
| Payments | [Razorpay](https://razorpay.com) Payment Links |
| WhatsApp | Meta WhatsApp Business API |
| SMS | [MSG91](https://msg91.com) |

---

## Repository Structure

```
ai-clinic-receptionist/
├── n8n-workflows/
│   ├── workflow-a-check-and-book.json    # Webhook → Calendar check → Book
│   └── workflow-b-confirm-and-notify.json # Webhook → Razorpay → WhatsApp → SMS
├── tough-tongue-ai/
│   ├── scenario-script.md                # Telugu conversation flow
│   └── custom-tools-config.json          # Tool definitions for Tough Tongue AI
├── docs/
│   ├── SETUP.md                          # Full setup guide
│   ├── ARCHITECTURE.md                   # Deep-dive architecture doc
│   └── WHATSAPP_TEMPLATE.md              # Meta template submission guide
├── .env.example                          # All required environment variables
├── .gitignore
└── README.md
```

---

## Quick Start

See [docs/SETUP.md](docs/SETUP.md) for the full step-by-step setup guide.

### TL;DR

1. Import both JSON files from `n8n-workflows/` into your n8n instance
2. Configure credentials (Google Calendar, Razorpay, WhatsApp, MSG91) — see [docs/SETUP.md](docs/SETUP.md)
3. Activate both workflows → copy webhook URLs
4. Register the custom tools in Tough Tongue AI using [tough-tongue-ai/custom-tools-config.json](tough-tongue-ai/custom-tools-config.json)
5. Test with the sample payloads in [docs/SETUP.md](docs/SETUP.md)
6. Go live 🎉

---

## India-specific Compliance Checklist

- [ ] DLT sender ID registration (mandatory for transactional SMS under TRAI regulations)
- [ ] DLT template registration for your SMS content
- [ ] Meta WhatsApp template approval for Telugu template
- [ ] Google OAuth consent screen published (not in "Testing" mode)
- [ ] Razorpay account KYC completed for live payment links

---

## Why Tough Tongue AI over Vapi?

| | Vapi | Tough Tongue AI |
|--|------|-----------------|
| Telugu support | Via 3rd-party TTS | Native |
| Server location | US | India |
| Avg. response latency (India callers) | ~1,450ms | ~400–600ms |
| Price | USD | INR |
| Indian telephony defaults | Manual config | Built-in |

---

## Contributing

PRs welcome. If you've tested this with a specific Telugu dialect and have feedback on voice quality, please open an issue — that data is genuinely useful.

---

## License

MIT
