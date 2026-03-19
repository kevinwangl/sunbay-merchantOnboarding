# SUNBAY × Merchantics — Merchant Boarding Automation

> Executive Summary | 2026.03 | Confidential

---

## The Problem

Your sales team currently onboards merchants by manually entering data in IRIS CRM, then re-entering it in CardPointe — a process that takes **2-3 days per merchant**, is error-prone, and has no real-time status visibility. This cannot scale with business growth.

## The Solution

SUNBAY will build an **automated middleware** between IRIS CRM and CardPointe that handles the entire merchant boarding lifecycle:

```
IRIS CRM ──(Subscribe Merchant Events)──▶ OnBoarding Service Middleware ──▶ Processors
                                                    │                    (TSYS / Elavon / Fiserv)
                                                    │                         │
CRM status updated ◀── Sync Processor Data ────────┘◀── MID/TID ────────────┘
```

Sales reps only work in IRIS CRM. Set status to "Ready for Boarding" — everything else is automated.

## Key Benefits

| Before | After |
|---|---|
| 2-3 days per merchant | **< 4 hours** |
| Manual data entry in 2 systems | **Zero manual work** |
| No approval visibility | **Real-time status in CRM** |
| Frequent field errors & rejections | **Auto-validated before submission** |

## Timeline & Next Steps

**8-week delivery**: Foundation (W1-3) → Integration (W4-6) → Testing & Go-live (W7-8)

**To get started, we need:**
1. CardPointe Boarding API access confirmation from Fiserv *(high priority)*
2. IRIS CRM API credentials
3. Kick-off meeting to align on scope

📎 See full **Proposal & Statement of Work** for detailed scope, deliverables, and terms.

---

> SUNBAY Technology | Contact: [sales@sunbay.com]
