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
  - [4. Analytics Flow](#4-analytics-flow)
- [Key Integration Points](#key-integration-points)
- [Event Types](#event-types)
- [Technology Stack](#technology-stack)

---

## System Architecture

<img width="2803" height="2058" alt="image" src="https://github.com/user-attachments/assets/3717706a-2041-42e0-9b8f-a98c0a8b2fc2" />

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

### 4. Analytics Flow

#### User
- **Purpose**: End user or analytics consumer
- **Action**: Queries analytics data

#### WebAPI Service (`web_api.py`)
- **Purpose**: API layer for analytics queries
- **Actions**:
  - Receives analytics requests from users
  - Queries Snowflake data warehouse
- **Output**: Returns analytics data to user

#### Snowflake
- **Purpose**: Data warehouse
- **Function**: Stores historical call data and analytics

---

## Key Integration Points

| Source | Target | Integration Method |
|--------|--------|-------------------|
| Engage | Voice API | HTTP/Internal API call |
| Voice API | SQS #1 | AWS SQS publish |
| SQS #1 | Call Processor | SQS consumer polling |
| Call Processor | Retell | HTTP API call |
| Retell | AWS Lambda | Webhook POST |
| AWS Lambda | SQS #2 | AWS SQS publish |
| SQS #2 | WebhookProcessor | SQS consumer polling |
| User | WebAPI | HTTP API request |
| WebAPI | Snowflake | Database query |

---

## Event Types

The system uses ABT (Activity-Based Tracking) events to monitor call lifecycle:

| Event Name | Trigger Point | Sent By |
|------------|--------------|---------|
| **Voice call initiated** | When call is triggered | Voice Call Processor |
| **Call started** | When Retell confirms call connection | Retell WebhookProcessor |
| **Call completed** | After call analysis is complete (with DB update) | Retell WebhookProcessor |

---

## Technology Stack

- **Message Queuing**: AWS SQS
- **Serverless Computing**: AWS Lambda
- **Voice Platform**: Retell
- **Data Warehouse**: Snowflake
- **Language**: Python
- **Architecture Pattern**: Event-driven microservices

