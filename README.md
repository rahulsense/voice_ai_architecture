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

## 4. Business Operations Flow

### Business API (`business_api.py`)
- **Purpose**: Internal business logic API for executing tasks based on call outcomes
- **Authentication**: Business API key + Retell authentication
- **Endpoints**:
  
  - **`POST /api/v2/voice-ai/trigger`**
    - **Purpose**: Execute business tasks based on call data and transcripts
    - **Task Types**:
      - `field writeback`: Updates CRM/database fields with information collected during the call. Ensures call data is persisted back to source system.
      - `reschedule call`: Schedules follow-up calls based on user request during conversation. Creates new scheduled task with preferred date/time.
      - `delete scheduled task`: Cancels previously scheduled calls or tasks.
      - `update timezone`: Updates contact's timezone based on call time preferences.
      - `incoming call`: Handles routing logic for inbound calls.
      - `set disposition`: Sets final call outcome/disposition code (e.g., `screening_complete`, `call_rescheduled`).
      - `send abt event`: Manually triggers analytics/business tracking events.
      - `call analysed`: Processes comprehensive call analysis results.
      - `note writeback`: Writes call summary notes back to CRM.
        
    - **Workflow**:
      1. Receives task request with call data and transcript
      2. Processes transcript to extract structured information
      3. Validates task configuration against business rules
      4. Executes task-specific handler
      5. Updates database with task results
      6. Returns task execution status
      
- **Output**: Executes business logic and persists data to database 

---

## Event Types

The system uses ABT (Activity-Based Tracking) events to monitor call lifecycle and trigger downstream workflows:

- `voice ai call initiated`: Call attempt started, dialing in progress
- `voice ai call failed`: Call failed to connect (busy, no answer, invalid number)
- `voice ai call started`: Call successfully connected with recipient
- `voice ai call left voicemail`: Voicemail detected and message left
- `voice ai call completed`: Call successfully completed with desired outcome
- `voice ai call rescheduled`: Call resulted in rescheduling request from user
- `voice ai call partially completed`: Call completed with partial success

- **Purpose**: These events enable downstream analytics, reporting, business workflows, and real-time monitoring of call campaigns.

---

## Technology Stack

- **Message Queuing**: AWS SQS
- **Serverless Computing**: AWS Lambda
- **Voice Platform**: Retell
- **Language**: Python
- **Architecture Pattern**: Event-driven microservices
