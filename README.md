# Voice AI System - Architectural Flow

## Overview

This is a voice call processing system that handles outbound calls, processes webhooks, and stores analytics data. The system uses a microservices architecture with event-driven components for reliable, asynchronous call processing.

---

## Table of Contents

- [System Architecture](#system-architecture)
- [System Components](#system-components)
  - [1. Call Initiation Flow](#1-call-initiation-flow)
  - [2. Call Processing Flow](#2-call-processing-flow)
  - [3. Event Processing Flow](#3-event-processing-flow)
  - [4. Business Operations Flow](#4-business-operations-flow)
- [Event Types](#event-types)
- [Technology Stack](#technology-stack)

---

## System Architecture

<img width="1768" height="1118" alt="image" src="https://github.com/user-attachments/assets/4687f0fe-2c2b-465b-9a81-97cd05dbb369" />


---

## System Components

### 1. Call Initiation Flow

#### Engage Service
- **Purpose**: External trigger/entry point for initiating voice calls
- **Output**: Triggers Voice API with call request

#### Voice API Service (`internal_api.py`)
- **Purpose**: Internal API layer that receives call requests
- **Actions**:
  - Receives trigger from Engage
  - Publishes call messages to SQS queue
- **Output**: Messages sent to SQS

#### SQS Queue #1
- **Purpose**: Message queue for decoupling call requests
- **Benefit**: Ensures reliable, asynchronous processing of call requests

---

### 2. Call Processing Flow

#### Voice Call Processor (`call_processor.py`)
- **Purpose**: Core call orchestration service
- **Actions**:
  - Consumes messages from SQS Queue #1
  - Initiates outbound voice calls via Retell
  - Sends "voice call initiated" ABT event
- **Integration**: Communicates with Retell platform

#### Retell Platform
- **Purpose**: Third-party voice call service provider
- **Actions**:
  - Executes actual voice calls
  - Generates call lifecycle events:
    - `call_started`
    - `call_ended`
    - `call_analyzed`
- **Output**: Sends events to AWS Lambda via webhooks

---

### 3. Event Processing Flow

#### AWS Lambda
- **Purpose**: Serverless event handler for Retell webhooks
- **Actions**:
  - Receives call events from Retell
  - Transforms and forwards events to SQS Queue #2
- **Events Handled**:
  - Call started
  - Call ended
  - Call analyzed

#### SQS Queue #2
- **Purpose**: Message queue for webhook events
- **Benefit**: Decouples Lambda from webhook processor

#### Retell WebhookProcessor (`webhook_processor.py`)
- **Purpose**: Business logic processor for call events
- **Actions**:
  - **On `call_started` event**:
    - Sends "call started" ABT event
  - **On `call_ended` event**:
    - No ABT event sent
    - No action taken
  - **On `call_analyzed` event**:
    - Updates database with call details
    - Sends "call completed" ABT event (on success)

---

### 4. Business Operations Flow

#### Business API (`business_api.py`)
- **Purpose**: API layer for business operations
- **Actions**:
  - FIELD_WRITEBACK = "field_writeback"
  - RESCHEDULE_CALL = "reschedule_call"
  - DELETE_SCHEDULED_TASK = "delete_scheduled_task"
  - UPDATE_TIMEZONE = "update_timezone"
  - INCOMING_CALL = "incoming_call"
  - SET_DISPOSITION = "set_disposition"
  - SEND_ABT_EVENT = "send_abt_event"
  - CALL_ANALYSED = "call_analysed"
  - NWB = "note_writeback"
- **Output**: Executes business logic and persists data

---

## Event Types

The system uses ABT (Activity-Based Triggering) events to monitor call lifecycle:

- VOICE_AI_CALL_INITIATED = "voice_ai_call_initiated"
- VOICE_AI_CALL_FAILED = "voice_ai_call_failed"
- VOICE_AI_CALL_STARTED = "voice_ai_call_started"
- VOICE_AI_CALL_LEFT_VOICEMAIL = "voice_ai_call_left_voicemail"
- VOICE_AI_CALL_COMPLETED = "voice_ai_call_completed"
- VOICE_AI_CALL_RESCHEDULED = "voice_ai_call_rescheduled"
- VOICE_AI_CALL_PARTIALLY_COMPLETED = "voice_ai_call_partially_completed"

---

## Technology Stack

- **Message Queuing**: AWS SQS
- **Serverless Computing**: AWS Lambda
- **Voice Platform**: Retell
- **Language**: Python
- **Architecture Pattern**: Event-driven microservices
