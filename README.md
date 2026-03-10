# Retell AI + GoHighLevel (GHL) Integration

A complete guide to understanding and replicating the **Retell AI x GoHighLevel** integration for AI-powered outbound/inbound phone call automation with CRM pipeline management.

---

## Table of Contents

1. [Overview](#overview)
2. [System Architecture](#system-architecture)
3. [GHL Pipeline Setup](#ghl-pipeline-setup)
4. [Retell AI Agent Configuration](#retell-ai-agent-configuration)
5. [Workflow 1 - Retell_GHL Outbound Caller](#workflow-1---retell_ghl-outbound-caller)
6. [Workflow 2 - Retell_GHL_2 Inbound Webhook Handler](#workflow-2---retell_ghl_2-inbound-webhook-handler)
7. [Call Flow - Agent Conversation](#call-flow---agent-conversation)
8. [Execution Logs and Debugging](#execution-logs-and-debugging)
9. [Key Components Reference](#key-components-reference)

---

## Overview

This integration connects **Retell AI** (an AI voice agent platform) with **GoHighLevel (GHL)** (a CRM/automation platform) to automate phone-based lead qualification and appointment booking. The system:

- **Automatically calls** leads when they enter specific CRM pipeline stages
- **Uses an AI agent ("Maria")** to qualify leads, switch languages if needed, and book consultations
- **Writes call outcomes back** to GHL as pipeline stage updates, contact fields, and calendar appointments
- **Handles inbound webhooks** from Retell to process call results in GHL

---

## System Architecture

```mermaid
flowchart TD
    A["Lead or Contact in GHL CRM"]
    B["GHL Workflow 1 - Retell_GHL"]
    C["Retell AI - Creates Outbound Call"]
    D["Live Phone Call - Maria, Immigration Finder"]
    E["Retell Webhook POST"]
    F["GHL Workflow 2 - Retell_GHL_2"]
    G["GHL Calendar - Appointment Booked"]
    H["Contact Created or Updated with Tags"]
    I["SMS Confirmation Sent"]
    J["Internal Team Notification"]

    A -->|"Pipeline Stage Changes"| B
    B -->|"HTTP API Call"| C
    C -->|"AI Voice Agent"| D
    D -->|"Call Ends"| E
    E -->|"Inbound Webhook Trigger"| F
    F -->|"Booking Confirmed"| G
    F -->|"No Booking"| H
    G --> I
    G --> J
    H --> J
```

---

## GHL Pipeline Setup

The **"AI Pipeline"** in GHL tracks leads through the outbound calling funnel. It has **4 stages**:

```mermaid
flowchart LR
    S1["Stage 1 - To Be Called"]
    S2["Stage 2 - Call in Progress"]
    S3["Stage 3 - Interested or Booked"]
    S4["Stage 4 - No Answer or Follow-up"]

    S1 -->|"Workflow triggers call"| S2
    S2 -->|"Lead books appointment"| S3
    S2 -->|"No answer or voicemail"| S4
    S4 -->|"Retry attempt"| S2
```

| Stage | Name | Purpose |
|---|---|---|
| 1 | **To Be Called** | New leads waiting to be contacted |
| 2 | **Call in Progress** | Active/pending call via Retell AI |
| 3 | **Interested / Booked** | Lead converted — appointment scheduled |
| 4 | **No Answer / Follow-up** | No answer, voicemail, or needs retry |

> All stages report to both **Funnel Chart** and **Stage Distribution** in GHL Reports.

---

## Retell AI Agent Configuration

The AI voice agent **"Maria"** is a `Healthcare Check-In` agent built in Retell AI.

```mermaid
flowchart TD
    subgraph RetellAgent["Retell AI Agent - Maria, Healthcare Check-In"]
        direction TB
        M1["Model: GPT-4.1 | Voice: Kate | Language: English with auto-switch"]
        M2["Cost: 0.115 per min | Latency: 970-1300ms | Tokens: 577-897"]
        M3["Persona: Maria from Immigration Finder - Empathetic, Concise, Proactive"]
    end

    subgraph Functions["Agent Functions"]
        F1["transfer_call - Transfer to human agent"]
        F2["end_call - End the conversation"]
        F3["book_consultation - Calendar booking tool - MANDATORY trigger"]
    end

    subgraph Settings["Configuration Panels"]
        C1["Knowledge Base"]
        C2["Speech Settings"]
        C3["Realtime Transcription Settings"]
        C4["Call Settings"]
        C5["Post-Call Data Extraction"]
        C6["Security and Fallback Settings"]
        C7["Webhook Settings"]
        C8["MCPs"]
    end

    RetellAgent --> Functions
    RetellAgent --> Settings
```

### Agent Conversation Flow (Prompt Logic)

```mermaid
flowchart TD
    A1["Call Starts - AI speaks first with dynamic welcome message"]
    A2["Introduction: Hi Im Maria from Immigration Finder. May I get your full name?"]
    A3{"Language Check: Does lead respond in Spanish or Portuguese?"}
    A4["Switch Language Immediately - Continue intake in native language"]
    A5["Qualify: Which immigration service? Family Visas, Employment Residency, or Asylum?"]
    A6["The Booking: Attorney has a strategy session opening. What date and time works best?"]
    A7{"Lead provides date and time?"}
    A8["MANDATORY: Trigger book_consultation tool immediately"]
    A9["Acknowledge and probe further - Offer alternative"]
    A10["Speak During Execution: One moment while I check availability..."]
    A11["Confirm Booking: Excellent. I have you down for date at time. Does that sound correct?"]
    A12["Finalize: Details sent to legal team. Thank you and have a wonderful day!"]
    A13["end_call triggered"]

    A1 --> A2
    A2 --> A3
    A3 -->|"Yes"| A4
    A3 -->|"No - English"| A5
    A4 --> A5
    A5 --> A6
    A6 --> A7
    A7 -->|"Yes"| A8
    A7 -->|"No or Hesitant"| A9
    A9 --> A6
    A8 --> A10
    A10 --> A11
    A11 --> A12
    A12 --> A13
```

---

## Workflow 1 - Retell_GHL Outbound Caller

**Purpose:** Watches for pipeline stage changes in GHL and fires an outbound call via Retell AI.

```mermaid
flowchart TD
    T["TRIGGER: Pipeline Stage Changed - AI Pipeline, To Be Called"]
    D["Drip Mode - Optional delay or rate-limiter before firing the call"]
    P["Action 1 - Create Phone Call: From +12137771234 To user.phone via Retell"]
    U["Create or Update Opportunity - Move lead to Call in Progress stage"]
    W["Wait - Hold until call resolves"]
    GC["Action 2 - Get Concurrency: Check current calls vs org limit via Retell API"]
    END(["END"])

    T --> D
    D --> P
    P --> U
    U --> W
    W --> GC
    GC --> END
```

### Create Phone Call Parameters

| Parameter | Value | Notes |
|---|---|---|
| **Action Name** | Create Phone Call | GHL action label |
| **From Number** | `+12137771234` | Must be a Retell-managed number in E.164 format |
| **To Number** | `{{user.phone}}` | Dynamic — pulled from GHL contact |
| **Override Agent ID** | *(optional)* | One-time override for this call only |
| **Override Agent Version** | *(optional)* | One-time version override |

> **Note:** `Get Concurrency` polls the Retell API to confirm the org has not exceeded its concurrent call limit — a guard against over-dialing.

---

## Workflow 2 - Retell_GHL_2 Inbound Webhook Handler

**Purpose:** Receives webhook POSTs from Retell after a call ends, then branches based on the call outcome.

### Full Workflow — 3-Branch Conditional

```mermaid
flowchart TD
    T2["TRIGGER: Inbound Webhook - POST from Retell after call ends"]
    CC["Create Contact - Upsert contact in GHL from webhook payload"]
    CD{"Condition - Evaluate call outcome fields"}

    T2 --> CC
    CC --> CD

    CD -->|"Tags includes booked"| B1["Branch 1: BOOKED"]
    CD -->|"request_type equals check"| B2["Branch 2: AVAILABILITY CHECK"]
    CD -->|"None of the conditions met"| B3["Branch 3: NONE or UNQUALIFIED"]

    B1 --> B1A["Update Contact Field - Mark as booked"]
    B1A --> B1B["Update Appointment Status"]
    B1B --> B1C["Custom Code #4 - Post-processing logic"]
    B1C --> B1D["Book Appointment #10 - Write to GHL Calendar"]
    B1D --> B1E["Internal Notification - Alert team of new booking"]
    B1E --> END1(["END"])

    B2 --> B2A["Custom Code #1 - Parse availability request"]
    B2A --> B2B["Get Available Slots #3 - Query calendar API"]
    B2B --> END2(["END"])

    B3 --> B3A["Add Tag - e.g. no-answer or unqualified"]
    B3A --> B3B["Create or Update Opportunity - Push to No Answer stage"]
    B3B --> B3C["Custom Code #3 - Cleanup and logging"]
    B3C --> B3D["Book Appointment #9 - Tentative callback slot"]
    B3D --> B3E["Send SMS - Follow-up text to lead"]
    B3E --> B3F["Internal Notification"]
    B3F --> END3(["END"])
```

### Branch Decision Summary

```mermaid
flowchart LR
    W["Retell Webhook Payload"]
    C{"Condition Node"}
    BR1["Booked Branch: Book appointment, update CRM, notify team"]
    BR2["Availability Check Branch: Return open calendar slots to Retell"]
    BR3["None Branch: Add tag, create opportunity, send SMS"]

    W --> C
    C -->|"Tags contains booked"| BR1
    C -->|"request_type is check"| BR2
    C -->|"No match"| BR3
```

---

## Call Flow - Agent Conversation

Representative transcript from the live simulation captured in the screenshots:

```mermaid
sequenceDiagram
    participant AI as Maria - Retell AI Agent
    participant L as Lead

    AI->>L: Hi, Im Maria from Immigration Finder. Im following up on your recent inquiry. May I start by getting your full name?
    L->>AI: Provides name
    AI->>L: Thank you. Which immigration service are you looking into? Family Visas, Employment-based residency, or Asylum?
    L->>AI: Asylum or Family Visas
    Note over AI: Language Check - if Spanish or Portuguese detected, switch immediately
    AI->>L: I understand how important this is. Our attorney has an opening for a strategy session. What date and time works best?
    L->>AI: Provides date and time e.g. Feb 20th at 3 PM
    Note over AI: TRIGGER - book_consultation tool called immediately
    AI->>L: One moment while I check the attorneys availability for that time...
    AI->>L: Excellent. I have you down for that date and time. Does that sound correct?
    L->>AI: Yes
    AI->>L: Perfect. All your details have been sent to our legal team. Thank you and have a wonderful day!
    Note over AI: end_call triggered - Retell fires webhook - GHL Workflow 2 runs
```

---

## Execution Logs and Debugging

GHL's **Execution Logs** tab shows the real-time trace of every step executed:

```mermaid
flowchart LR
    EL["Execution Log Entry"]
    E1["Contact ID and Contact Details Hash"]
    E2["Execution ID - Unique run identifier"]
    E3["Enrollment Date - Timestamp of trigger"]
    E4["Workflow Version - Outdated Version warning if stale"]
    E5["Step-by-step Visual: Exited or Action Skipped"]

    EL --> E1
    EL --> E2
    EL --> E3
    EL --> E4
    EL --> E5
```

> **Tip:** If you see `Workflow Version When Execution Started: XX (Outdated Version)` in logs, it means the workflow was **updated after** the contact enrolled. Re-publish the workflow and re-enroll the contact to use the latest version.

---

## Key Components Reference

| Component | Platform | Purpose |
|---|---|---|
| **AI Pipeline** | GoHighLevel | 4-stage CRM pipeline tracking lead status |
| **Retell_GHL** (Workflow 1) | GoHighLevel | Outbound call trigger on pipeline stage change |
| **Retell_GHL_2** (Workflow 2) | GoHighLevel | Inbound webhook processor for call outcomes |
| **Maria Agent** | Retell AI | GPT-4.1 voice agent for immigration lead intake |
| **book_consultation** | Retell AI Function | Tool that triggers calendar booking mid-call |
| **transfer_call** | Retell AI Function | Transfers to human agent if needed |
| **end_call** | Retell AI Function | Gracefully ends the AI call |
| **Get Concurrency** | Retell API | Checks current vs. max concurrent call slots |
| **Drip Mode** | GHL | Rate-limits calls to prevent over-dialing |

---

## End-to-End Data Flow Summary

```mermaid
flowchart TD
    A["New Lead Enters GHL as To Be Called"]
    B["Retell_GHL Workflow Fires Outbound Call"]
    C["Retell AI Agent Maria Conducts Voice Interview"]
    D{"Call Outcome?"}
    E["book_consultation Function Triggered"]
    F["Lead Moved to No Answer or Follow-up Stage"]
    G["Tagged and Opportunity Closed"]
    H["Retell Fires Webhook to GHL"]
    I["Retell_GHL_2 Workflow Processes Outcome"]
    J["Appointment Created, SMS Sent, Team Notified"]
    K["Calendar Slots Returned to Retell"]
    L["Tagged, SMS Sent, Opportunity Updated, Team Notified"]

    A --> B
    B --> C
    C --> D
    D -->|"Appointment Booked"| E
    D -->|"No Answer or Voicemail"| F
    D -->|"Not Interested"| G
    E --> H
    F --> H
    G --> H
    H --> I
    I -->|"Booked"| J
    I -->|"Availability Check"| K
    I -->|"No Match"| L
```

---

*Built using [Retell AI](https://retellai.com) and [GoHighLevel](https://gohighlevel.com)*
