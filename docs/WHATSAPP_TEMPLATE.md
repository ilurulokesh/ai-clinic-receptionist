# WhatsApp Template Submission Guide (Telugu)

Meta requires pre-approved message templates for all outbound WhatsApp messages. You cannot send a free-form message to a patient — you must use an approved template.

---

## Step 1 — Access Meta Business Manager

1. Go to [business.facebook.com](https://business.facebook.com)
2. Left sidebar → **WhatsApp Manager** → **Message Templates**
3. Click **Create Template**

---

## Step 2 — Template settings

| Field | Value |
|-------|-------|
| Category | **Transactional** (appointment confirmations qualify) |
| Name | `clinic_booking_confirmation` *(exactly this — it's referenced in the n8n workflow)* |
| Language | **Telugu (te)** |

---

## Step 3 — Template body (Telugu)

Copy-paste this into the template body field:

```
నమస్కారం {{1}},

మీ అపాయింట్‌మెంట్ నిర్ధారించబడింది:
📅 తేదీ: {{2}}
⏰ సమయం: {{3}}
💳 చెల్లింపు: {{4}}

దయచేసి సమయానికి రండి.
ధన్యవాదాలు — డాక్టర్ క్లినిక్ 🏥
```

**Variable mapping:**
| Variable | Value sent by n8n |
|----------|------------------|
| `{{1}}` | Patient name |
| `{{2}}` | Appointment date (e.g. 15 September 2026) |
| `{{3}}` | Appointment time (e.g. 11:00 AM) |
| `{{4}}` | Razorpay payment link OR "క్లినిక్‌లో చెల్లించండి" (Pay at clinic) |

---

## Step 4 — Sample values (required by Meta)

Meta requires you to fill in sample variable values so reviewers can read the template:

| Variable | Sample value |
|----------|-------------|
| `{{1}}` | రవి కుమార్ |
| `{{2}}` | 15 సెప్టెంబర్ 2026 |
| `{{3}}` | 11:00 AM |
| `{{4}}` | https://rzp.io/l/sample |

---

## Step 5 — Submit and wait

- Click **Submit**
- Approval typically takes **24–48 hours**
- Status will change to **Approved** in WhatsApp Manager
- You cannot use the template in production until it is Approved

---

## Step 6 — Update n8n

Once approved, confirm the template name matches exactly in Workflow B's WhatsApp node:
```json
"template": {
  "name": "clinic_booking_confirmation",
  "language": { "code": "te" }
}
```

---

## Troubleshooting

| Issue | Fix |
|-------|-----|
| Template rejected | Usually due to promotional language — keep it purely transactional (no "offer", "discount" etc.) |
| Template in "Pending" > 48 hours | Contact Meta Business Support |
| Messages failing with error code 132000 | Template name mismatch — check spelling exactly |
| Messages failing with error code 131030 | Phone number not in approved country — ensure patient numbers are formatted `+91XXXXXXXXXX` |
