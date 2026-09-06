# Clinic Voice Booking Bot — Setup Guide

## What you're building

```
Patient call
    └─▶ Tough Tongue AI (Telugu voice)
            ├─▶ Webhook → n8n Workflow A → Google Calendar (check + book)
            └─▶ Webhook → n8n Workflow B → Razorpay + WhatsApp + SMS
```

---

## Files

| File | Purpose |
|------|---------|
| [`workflow-a-check-and-book.json`](file:///C:/Users/shams/.gemini/antigravity/scratch/clinic-booking-bot/n8n-workflows/workflow-a-check-and-book.json) | Checks Google Calendar, books slot, returns result to AI |
| [`workflow-b-confirm-and-notify.json`](file:///C:/Users/shams/.gemini/antigravity/scratch/clinic-booking-bot/n8n-workflows/workflow-b-confirm-and-notify.json) | Sends Razorpay link + WhatsApp + SMS after booking confirmed |

---

## Step-by-step setup

### 1. Import workflows into n8n

1. Open your n8n instance → **Workflows → Import from File**
2. Import `workflow-a-check-and-book.json`
3. Import `workflow-b-confirm-and-notify.json`

---

### 2. Google Calendar credential

> [!IMPORTANT]
> You need a Google Cloud project with the **Calendar API** enabled.

1. Go to [console.cloud.google.com](https://console.cloud.google.com) → New Project
2. Enable **Google Calendar API**
3. Create **OAuth 2.0 credentials** (type: Web Application)
4. Add your n8n instance URL as an Authorized Redirect URI:
   `https://YOUR-N8N-DOMAIN/rest/oauth2-credential/callback`
5. In n8n: **Settings → Credentials → New → Google Calendar OAuth2 API**
6. Paste your Client ID and Secret → connect the Google account
7. In both workflows, replace `REPLACE_WITH_YOUR_GCAL_CREDENTIAL_ID` with the credential ID n8n assigns

**Set the Calendar ID variable:**
- In Workflow A → **Variables** → set `GOOGLE_CALENDAR_ID` to the doctor's calendar ID
- Find it in Google Calendar → Settings → your calendar → "Calendar ID" (usually `xyz@gmail.com` or a long string ending in `@group.calendar.google.com`)

---

### 3. Razorpay (for UPI payments)

> [!NOTE]
> Skip this section if you're collecting payment at the clinic only.

1. Sign up at [razorpay.com](https://razorpay.com) → get API Key ID + Key Secret
2. In n8n: **Settings → Credentials → New → HTTP Basic Auth**
   - Username: your Razorpay Key ID (starts with `rzp_`)
   - Password: your Razorpay Key Secret
3. Replace `REPLACE_WITH_RAZORPAY_CREDENTIAL_ID` in Workflow B
4. Change the `amount` field (currently `50000` = ₹500 in paise) to your fee

---

### 4. WhatsApp Business API

> [!WARNING]
> Meta requires pre-approved message templates for outbound WhatsApp messages. Allow 1–2 days for approval.

1. Set up a [Meta Business Account](https://business.facebook.com) + WhatsApp Business API
2. Create a message template named `clinic_booking_confirmation` in **Telugu** with this body:
   ```
   నమస్కారం {{1}}, మీ అపాయింట్‌మెంట్ {{2}} తేదీన {{3}} గంటలకు నిర్ధారించబడింది.
   చెల్లింపు లింక్: {{4}}
   ధన్యవాదాలు — డాక్టర్ క్లినిక్
   ```
   *(Variables: 1=name, 2=date, 3=time, 4=payment link or "Pay at clinic")*
3. Submit for Meta approval
4. In n8n Workflow B → **Variables** → set `WHATSAPP_TOKEN` to your permanent System User access token
5. In the WhatsApp node, replace `YOUR_WHATSAPP_PHONE_NUMBER_ID` with your phone number ID from Meta Business Manager

---

### 5. SMS via MSG91

> [!WARNING]
> India requires DLT registration for transactional SMS. Register at your telecom operator's DLT portal before going live.

1. Sign up at [msg91.com](https://msg91.com)
2. Register your **Sender ID** (6 chars, e.g. `CLINIC`) on DLT
3. Create and register an SMS template on DLT, then add it in MSG91
4. In n8n Workflow B → **Variables** → set `MSG91_AUTHKEY` to your authkey
5. In the SMS node, set `template_id` to your MSG91 flow template ID

---

### 6. Set up Tough Tongue AI custom tools

In your Tough Tongue AI scenario, create **two custom tools**:

#### Tool 1 — `check_and_book_appointment`
| Field | Value |
|-------|-------|
| Method | POST |
| URL | *(Webhook URL from Workflow A — copy from n8n after activating)* |
| Description | "Check if a time slot is available and book the appointment" |

**Parameters to collect before calling:**
```json
{
  "patient_name":   { "type": "string", "description": "Patient's full name" },
  "patient_phone":  { "type": "string", "description": "Patient's phone number with country code" },
  "requested_date": { "type": "string", "description": "Date in YYYY-MM-DD format" },
  "requested_time": { "type": "string", "description": "Time in HH:MM 24-hour format (IST)" }
}
```

#### Tool 2 — `send_confirmation`
| Field | Value |
|-------|-------|
| Method | POST |
| URL | *(Webhook URL from Workflow B)* |
| Description | "Send booking confirmation via WhatsApp and SMS" |

**Parameters:**
```json
{
  "patient_name":   { "type": "string" },
  "patient_phone":  { "type": "string" },
  "confirmed_date": { "type": "string" },
  "confirmed_time": { "type": "string" },
  "payment_mode":   { "type": "string", "enum": ["upi", "pay_at_clinic"] }
}
```

---

### 7. Activate both workflows

In n8n, toggle both workflows to **Active**. The webhook URLs become live only after activation.

---

## Testing checklist

- [ ] **Workflow A test** — use n8n's "Test Webhook" feature, POST:
  ```json
  {
    "patient_name": "Ravi Kumar",
    "patient_phone": "+919876543210",
    "requested_date": "2026-09-15",
    "requested_time": "11:00"
  }
  ```
  Verify: Google Calendar event is created, response JSON comes back with `status: "booked"`.

- [ ] **Workflow B test** — POST with `payment_mode: "pay_at_clinic"` — verify WhatsApp and SMS land on a test number.

- [ ] **End-to-end** — call your Tough Tongue AI number, speak in Telugu, complete a full booking.

---

## Customise slot duration and clinic hours

Both settings are in Workflow A code nodes:

| Setting | Node | Line to change |
|---------|------|----------------|
| Appointment length | `Parse & Normalise Input` | `const endMinutes = m + 30` |
| Clinic closing hour | `Find Next Available Slot` | `const clinicEndHour = 18` |

---

## Future additions

| Feature | How |
|---------|-----|
| Day-before reminders | n8n Schedule trigger → Google Calendar list → WhatsApp |
| Voice cancellations | Third Tough Tongue AI tool → n8n → Calendar delete |
| Multiple doctors | Add `doctor_name` param, use separate calendar IDs |
| Patient history | Add Google Sheets / Airtable node to log past appointments |
