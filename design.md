# Design Document: Velocity Platform

## 1. High-Level Architecture Overview

Velocity is a cloud-native, multi-platform development ecosystem built on AWS infrastructure that eliminates hardware barriers and productivity fragmentation through intelligent orchestration of specialized AI agents. The platform operates as a distributed system with three client interfaces (Web, Desktop, Mobile) backed by a unified cloud architecture.

### Core Architectural Principles

1. **Cloud-First Design**: All compute-intensive operations offloaded to AWS, enabling thin clients
2. **Model Agnosticism**: No vendor lock-in; users select AI models via Amazon Bedrock
3. **Shared Context Architecture**: Single source of truth maintained across all agents and users
4. **Privacy by Design**: Behavioral profiling in secure enclaves, never exposing raw code
5. **Horizontal Scalability**: Stateless services with managed AWS infrastructure
6. **Multi-Tenancy**: Logical isolation with shared infrastructure for cost efficiency

### Architecture Layers

```
┌─────────────────────────────────────────────────────────────────┐
│                     CLIENT LAYER                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ Velocity Web │  │   Velocity   │  │   Velocity   │          │
│  │   (Browser)  │  │   Desktop    │  │    Mobile    │          │
│  │              │  │  (Electron)  │  │ (iOS/Android)│          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     API GATEWAY LAYER                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Amazon API Gateway + AWS AppSync (GraphQL)              │   │
│  │  - Authentication (Cognito)                              │   │
│  │  - Rate Limiting                                         │   │
│  │  - Request Routing                                       │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   APPLICATION LAYER                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Streaming  │  │  Multi-Agent │  │ Collaboration│          │
│  │   Service    │  │ Orchestrator │  │   Service    │          │
│  │ (AppStream)  │  │(Step Func.)  │  │ (Chime SDK)  │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Context    │  │ Skill Wallet │  │  Learning    │          │
│  │   Janitor    │  │   Service    │  │   Engine     │          │
│  │  (Lambda)    │  │(Nitro Encl.) │  │  (Lambda)    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      AI LAYER                                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              Amazon Bedrock                              │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐              │   │
│  │  │ Architect│  │  Logic   │  │    UI    │              │   │
│  │  │  Agent   │  │  Agent   │  │  Agent   │              │   │
│  │  │(GPT-4o)  │  │(Claude)  │  │(Gemini)  │              │   │
│  │  └──────────┘  └──────────┘  └──────────┘              │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      DATA LAYER                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Aurora     │  │  ElastiCache │  │      S3      │          │
│  │  (Postgres)  │  │    (Redis)   │  │  (Storage)   │          │
│  │ Vector Search│  │Session State │  │Code/Artifacts│          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
```


## 2. System Components Breakdown

### 2.1 Client Layer Components

#### Velocity Web (Thin Client)
- **Technology**: Progressive Web App (PWA) + AWS AppStream 2.0
- **Purpose**: Zero-install IDE for budget hardware users
- **Key Features**:
  - Streams full IDE environment from AWS
  - Minimal local processing (rendering only)
  - Persistent session state in cloud
  - Optimized for 2 Mbps connections

#### Velocity Desktop (Native Client)
- **Technology**: Electron + React + Monaco Editor
- **Purpose**: High-performance native IDE for professionals
- **Key Features**:
  - Local file system access
  - Native terminal integration
  - Offline coding capability with sync
  - Multi-cursor collaboration

#### Velocity Mobile (Companion App)
- **Technology**: React Native (iOS/Android)
- **Purpose**: Command center and career engine
- **Key Features**:
  - Save-Gate approval interface
  - Verified Skill Wallet viewer
  - Push notifications for team events
  - Voice-to-note capture

### 2.2 API Gateway Layer

#### Amazon API Gateway
- **REST APIs**: User management, project CRUD, settings
- **WebSocket APIs**: Real-time collaboration events
- **Authentication**: AWS Cognito integration
- **Rate Limiting**: Token bucket algorithm (100 req/min per user)

#### AWS AppSync (GraphQL)
- **Real-Time Subscriptions**: Multi-cursor positions, file changes
- **Optimistic UI Updates**: Client-side mutations with server reconciliation
- **Offline Sync**: Conflict resolution for mobile clients
- **Schema**: Strongly-typed GraphQL schema for type safety

### 2.3 Application Services

#### Streaming Service (AWS AppStream 2.0)
- **Fleet Configuration**:
  - Instance Type: stream.standard.medium (4 vCPU, 8 GB RAM)
  - Base Image: Ubuntu 22.04 with VS Code, Node.js, Python, Docker
  - Auto-scaling: 10-1000 instances based on demand
- **Session Management**:
  - Max session duration: 8 hours
  - Idle timeout: 30 minutes
  - Session persistence via S3-backed home directories

#### Multi-Agent Orchestrator (AWS Step Functions)
- **State Machine**: Coordinates parallel agent execution
- **Workflow**:
  1. User request → Architect Agent (task decomposition)
  2. Parallel execution → Logic Agent + UI Agent
  3. Context Janitor → Validate consistency
  4. Merge results → Return to user
- **Error Handling**: Exponential backoff with 3 retries
- **Timeout**: 300 seconds per agent invocation

#### Collaboration Service (Amazon Chime SDK)
- **Voice/Video**: WebRTC-based low-latency communication
- **Screen Sharing**: H.264 encoding at 1080p/30fps
- **Recording**: Optional session recording to S3
- **Bandwidth Optimization**: Adaptive bitrate (256 kbps - 2 Mbps)

#### Context Janitor (AWS Lambda)
- **Trigger**: EventBridge schedule (every 10 seconds) + file save events
- **Function**:
  - Prune outdated context (>1 hour old)
  - Update vector embeddings in Aurora
  - Detect potential merge conflicts
  - Broadcast updates via AppSync
- **Memory**: 1024 MB
- **Timeout**: 30 seconds

#### Skill Wallet Service (AWS Nitro Enclaves)
- **Enclave Configuration**:
  - Parent Instance: m5.xlarge
  - Enclave Memory: 4 GB
  - Enclave vCPUs: 2
- **Processing**:
  - Behavioral data ingestion (encrypted)
  - Skill hash generation (SHA-256)
  - AI narration generation (via Bedrock inside enclave)
  - Cryptographic signing (AWS KMS)
- **Output**: Signed skill hashes + playback metadata (no raw code)

#### Learning Engine (AWS Lambda)
- **Roadmap Generation**:
  - Input: User interests, skill gaps, market data
  - Processing: Bedrock Claude for personalized path creation
  - Output: Structured JSON roadmap with skill gates
- **Progress Tracking**:
  - Code execution verification
  - Time-on-task measurement
  - Skill gate unlock logic

### 2.4 AI Layer (Amazon Bedrock)

#### Architect Agent (GPT-4o / o1)
- **Model**: anthropic.claude-3-5-sonnet-20241022-v2:0 (fallback to GPT-4o)
- **Role**: High-level planning and task decomposition
- **Input**: User natural language request + project context
- **Output**: Structured task manifest (JSON)
- **Context Window**: 200K tokens
- **Temperature**: 0.3 (deterministic planning)

#### Logic Agent (Claude 3.5 Sonnet)
- **Model**: anthropic.claude-3-5-sonnet-20241022-v2:0
- **Role**: Backend logic, API routes, database queries, unit tests
- **Input**: Task manifest + shared context from Aurora
- **Output**: Backend code with inline documentation
- **Context Window**: 200K tokens
- **Temperature**: 0.2 (precise code generation)

#### UI Agent (Gemini 1.5 Pro)
- **Model**: google.gemini-1.5-pro-v1
- **Role**: Frontend components, styling, responsive design
- **Input**: Task manifest + backend API schema + design mockups
- **Output**: React/Vue components with Tailwind CSS
- **Context Window**: 1M tokens (multimodal support)
- **Temperature**: 0.4 (creative UI generation)

### 2.5 Data Layer

#### Amazon Aurora PostgreSQL (Shared Memory)
- **Configuration**:
  - Engine: Aurora PostgreSQL 15.4
  - Instance Class: db.r6g.xlarge (4 vCPU, 32 GB RAM)
  - Storage: Auto-scaling (10 GB - 128 TB)
  - Read Replicas: 2 (for read scaling)
- **Extensions**:
  - pgvector: Vector similarity search for context embeddings
  - pg_cron: Scheduled cleanup jobs
- **Schema**:
  - `users`: User profiles and authentication
  - `projects`: Project metadata and ownership
  - `context_memory`: Vector embeddings of code context
  - `skill_hashes`: Verified skill credentials
  - `collaboration_sessions`: Active collaboration state

#### Amazon ElastiCache (Redis)
- **Configuration**:
  - Engine: Redis 7.0
  - Node Type: cache.r6g.large (2 vCPU, 13.07 GB RAM)
  - Cluster Mode: Enabled (3 shards, 1 replica per shard)
- **Use Cases**:
  - Session state (user authentication tokens)
  - Real-time cursor positions (TTL: 5 minutes)
  - Rate limiting counters
  - Temporary agent state during orchestration

#### Amazon S3
- **Buckets**:
  - `velocity-user-code`: User repositories and files
  - `velocity-artifacts`: Build outputs and deployments
  - `velocity-playbacks`: Skill wallet video recordings
  - `velocity-backups`: Database backups
- **Lifecycle Policies**:
  - Transition to S3-IA after 30 days
  - Transition to Glacier after 90 days
  - Delete after 365 days (except playbacks)
- **Encryption**: SSE-KMS with customer-managed keys


## 3. Detailed AWS Service Mapping

### Service Responsibility Matrix

| AWS Service | Component | Purpose | Scaling Strategy |
|------------|-----------|---------|------------------|
| AppStream 2.0 | Velocity Web | Stream IDE to browser | Auto-scaling fleet (10-1000) |
| Bedrock | AI Agents | Model inference | Cross-region routing |
| Step Functions | Orchestrator | Agent coordination | Concurrent executions (1000+) |
| Lambda | Context Janitor | Memory management | Provisioned concurrency (50) |
| Lambda | Learning Engine | Roadmap generation | On-demand scaling |
| Nitro Enclaves | Skill Wallet | Secure profiling | Parent instance scaling |
| Chime SDK | Collaboration | Voice/video/screen | Managed service (AWS-scaled) |
| Aurora PostgreSQL | Shared Memory | Context storage | Read replicas + auto-scaling |
| ElastiCache Redis | Session State | Real-time data | Cluster mode (3 shards) |
| S3 | File Storage | Code and artifacts | Unlimited scaling |
| API Gateway | REST APIs | HTTP endpoints | Throttling (10K req/sec) |
| AppSync | GraphQL | Real-time sync | Auto-scaling |
| Cognito | Authentication | User management | Managed service |
| CloudFront | CDN | Static asset delivery | Global edge locations |
| Route 53 | DNS | Domain routing | Health checks + failover |
| CloudWatch | Monitoring | Logs and metrics | Managed service |
| X-Ray | Tracing | Distributed tracing | Managed service |
| KMS | Encryption | Key management | Managed service |
| Secrets Manager | Secrets | API keys and tokens | Managed service |
| EventBridge | Events | Async event routing | Managed service |

### Cross-Region Architecture

**Primary Region**: ap-south-1 (Mumbai) - Target market proximity  
**Secondary Region**: us-east-1 (N. Virginia) - Bedrock model availability  
**Tertiary Region**: eu-west-1 (Ireland) - GDPR compliance

**Replication Strategy**:
- Aurora Global Database: <1 second replication lag
- S3 Cross-Region Replication: Async replication for user code
- ElastiCache Global Datastore: Sub-second replication for session state

**Failover Logic**:
- Route 53 health checks every 30 seconds
- Automatic DNS failover on primary region failure
- RTO: 5 minutes, RPO: 1 second


## 4. Data Flow Diagrams

### 4.1 User Request to Multi-Agent Response Flow

```
1. User submits request: "Build a secure login with React UI"
   ↓
2. API Gateway validates auth token (Cognito)
   ↓
3. Request routed to Step Functions State Machine
   ↓
4. State: "Architect Analysis"
   - Invoke Bedrock (GPT-4o) with user request + project context from Aurora
   - Output: Task manifest JSON
     {
       "tasks": [
         {"agent": "logic", "task": "Create Express auth middleware with JWT"},
         {"agent": "ui", "task": "Build React login form with validation"}
       ],
       "dependencies": ["logic.auth_routes → ui.api_endpoint"]
     }
   ↓
5. State: "Parallel Agent Execution" (Map state)
   ├─→ Logic Agent (Claude)
   │   - Reads shared context from Aurora (existing API structure)
   │   - Generates: auth.js, middleware/jwt.js, routes/auth.js
   │   - Writes context update: "POST /api/auth/login endpoint created"
   │   ↓
   │   Context Janitor (Lambda) triggered
   │   - Updates Aurora vector embeddings
   │   - Broadcasts update via AppSync subscription
   │
   └─→ UI Agent (Gemini)
       - Waits for Logic Agent context update (via AppSync)
       - Reads updated context: "POST /api/auth/login available"
       - Generates: LoginForm.jsx, useAuth.js hook
       - Writes context update: "Login component created"
       ↓
6. State: "Conflict Detection"
   - Context Janitor validates no conflicts
   - Checks: UI calls correct API endpoint, types match
   ↓
7. State: "Merge Results"
   - Combine Logic + UI outputs
   - Generate unified diff
   ↓
8. Response to user via AppSync
   - Real-time code updates in IDE
   - AI explanation of changes
   - Estimated execution time: 2.8 seconds
```

### 4.2 Save-Gate Collaboration Flow

```
1. Developer A (Owner) shares project with Developer B
   ↓
2. Developer B opens project in Velocity Desktop
   - Establishes WebSocket connection via AppSync
   - Subscribes to project events
   ↓
3. Developer B makes changes to file.js
   - Changes stored in Redis as "draft state"
   - Multi-cursor position broadcast via AppSync
   - Developer A sees B's cursor in real-time
   ↓
4. Developer B clicks "Request Save"
   - Draft diff generated and stored in Aurora
   - Push notification sent to Developer A (mobile)
   - AI summary generated: "Added error handling to login function"
   ↓
5. Developer A reviews on Velocity Mobile
   - Views AI-generated summary
   - Views side-by-side diff
   - Options: [Approve] [Reject] [Request Changes]
   ↓
6. Developer A clicks "Approve"
   - Draft state promoted to "committed state"
   - File.js updated in S3
   - Aurora context memory updated
   - Context Janitor triggered to update embeddings
   - Developer B receives approval notification
   ↓
7. Both developers see updated file
   - Real-time sync via AppSync
   - Version history recorded in Aurora
   - Skill Wallet updated for Developer B (contribution logged)
```

### 4.3 Skill Wallet Generation Flow

```
1. Developer codes for 45 minutes in Velocity Desktop
   - Local client logs behavioral data:
     * Keystrokes per minute
     * Time spent per file type
     * AI assistance usage patterns
     * Error resolution speed
     * Refactoring patterns
   ↓
2. Behavioral data encrypted locally (AES-256)
   ↓
3. Encrypted data sent to Nitro Enclave parent instance
   ↓
4. Inside Nitro Enclave (isolated environment):
   - Decrypt behavioral data
   - Analyze patterns:
     * "High proficiency in React hooks (95% correct usage)"
     * "Fast async debugging (avg 3 min per bug)"
     * "Consistent code style (98% linter compliance)"
   - Generate skill hash: SHA-256(behavioral_patterns + timestamp)
   - Invoke Bedrock (inside enclave) to generate AI narration
   - Create 30-second playback metadata (no raw code)
   - Sign with KMS key
   ↓
5. Output from enclave:
   - Skill hash: "a3f5b2c8d1e9..."
   - Skill tags: ["React", "Async/Await", "State Management"]
   - Proficiency scores: {react: 95, javascript: 92, debugging: 88}
   - Playback metadata: {duration: 30s, description: "Refactored complex state logic"}
   - Signature: KMS-signed proof
   ↓
6. Store in Aurora (skill_hashes table)
   - User can view in Velocity Mobile
   - User controls visibility (private/public)
   ↓
7. Recruiter searches for "React State Management Expert"
   - Query Aurora for skill_hashes with tag="React" AND proficiency>90
   - Returns anonymized profiles with verified badges
   - Recruiter can request to view playback (requires user approval)
```

### 4.4 Context Janitor Operation Flow

```
Trigger: File save event OR 10-second EventBridge schedule

1. Lambda function invoked
   ↓
2. Fetch recent changes from Aurora
   - Query: SELECT * FROM context_memory WHERE updated_at > NOW() - INTERVAL '10 seconds'
   ↓
3. For each change:
   - Extract code snippets
   - Generate embeddings via Bedrock (Titan Embeddings)
   - Store in pgvector column
   ↓
4. Conflict detection:
   - Compare new embeddings with existing context
   - Check for:
     * Duplicate function definitions
     * Incompatible type signatures
     * Missing dependencies
   - If conflict detected:
     * Flag in Aurora (conflict_status = 'detected')
     * Send alert via AppSync to all collaborators
   ↓
5. Context pruning:
   - Delete embeddings older than 1 hour
   - Archive to S3 for historical analysis
   ↓
6. Broadcast update via AppSync:
   - Subscription: onContextUpdate(projectId)
   - Payload: {updatedFiles: [...], conflicts: [...]}
   ↓
7. All connected clients receive update
   - AI agents refresh their context
   - Developers see conflict warnings in IDE
```


## 5. Multi-Agent Orchestration Logic

### 5.1 Step Functions State Machine Definition

```json
{
  "Comment": "Velocity Multi-Agent Orchestration",
  "StartAt": "ValidateRequest",
  "States": {
    "ValidateRequest": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT:function:velocity-validate-request",
      "Next": "ArchitectAnalysis",
      "Catch": [{
        "ErrorEquals": ["ValidationError"],
        "Next": "HandleError"
      }]
    },
    "ArchitectAnalysis": {
      "Type": "Task",
      "Resource": "arn:aws:states:::bedrock:invokeModel",
      "Parameters": {
        "ModelId": "anthropic.claude-3-5-sonnet-20241022-v2:0",
        "Body": {
          "anthropic_version": "bedrock-2023-05-31",
          "max_tokens": 4096,
          "temperature": 0.3,
          "messages": [{
            "role": "user",
            "content": "Analyze this request and create a task manifest: $.userRequest"
          }],
          "system": "You are an architect agent. Decompose requests into parallel tasks for Logic and UI agents."
        }
      },
      "ResultPath": "$.taskManifest",
      "Next": "CheckDependencies"
    },
    "CheckDependencies": {
      "Type": "Choice",
      "Choices": [{
        "Variable": "$.taskManifest.hasDependencies",
        "BooleanEquals": true,
        "Next": "SequentialExecution"
      }],
      "Default": "ParallelExecution"
    },
    "ParallelExecution": {
      "Type": "Parallel",
      "Branches": [
        {
          "StartAt": "LogicAgent",
          "States": {
            "LogicAgent": {
              "Type": "Task",
              "Resource": "arn:aws:states:::bedrock:invokeModel",
              "Parameters": {
                "ModelId": "anthropic.claude-3-5-sonnet-20241022-v2:0",
                "Body": {
                  "anthropic_version": "bedrock-2023-05-31",
                  "max_tokens": 8192,
                  "temperature": 0.2,
                  "messages": [{
                    "role": "user",
                    "content": "$.taskManifest.logicTask"
                  }],
                  "system": "You are a backend logic specialist. Generate production-ready code."
                }
              },
              "End": true
            }
          }
        },
        {
          "StartAt": "UIAgent",
          "States": {
            "UIAgent": {
              "Type": "Task",
              "Resource": "arn:aws:states:::bedrock:invokeModel",
              "Parameters": {
                "ModelId": "google.gemini-1.5-pro-v1",
                "Body": {
                  "contents": [{
                    "role": "user",
                    "parts": [{"text": "$.taskManifest.uiTask"}]
                  }],
                  "generationConfig": {
                    "temperature": 0.4,
                    "maxOutputTokens": 8192
                  }
                }
              },
              "End": true
            }
          }
        }
      ],
      "ResultPath": "$.agentResults",
      "Next": "TriggerContextJanitor"
    },
    "SequentialExecution": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT:function:velocity-sequential-executor",
      "Comment": "Execute agents in order when dependencies exist",
      "ResultPath": "$.agentResults",
      "Next": "TriggerContextJanitor"
    },
    "TriggerContextJanitor": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT:function:velocity-context-janitor",
      "Parameters": {
        "projectId.$": "$.projectId",
        "agentResults.$": "$.agentResults"
      },
      "ResultPath": "$.janitorResult",
      "Next": "CheckConflicts"
    },
    "CheckConflicts": {
      "Type": "Choice",
      "Choices": [{
        "Variable": "$.janitorResult.hasConflicts",
        "BooleanEquals": true,
        "Next": "ResolveConflicts"
      }],
      "Default": "MergeResults"
    },
    "ResolveConflicts": {
      "Type": "Task",
      "Resource": "arn:aws:states:::bedrock:invokeModel",
      "Parameters": {
        "ModelId": "anthropic.claude-3-5-sonnet-20241022-v2:0",
        "Body": {
          "messages": [{
            "role": "user",
            "content": "Resolve these conflicts: $.janitorResult.conflicts"
          }],
          "system": "You are a conflict resolution specialist. Merge conflicting code intelligently."
        }
      },
      "ResultPath": "$.resolvedCode",
      "Next": "MergeResults"
    },
    "MergeResults": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT:function:velocity-merge-results",
      "Parameters": {
        "agentResults.$": "$.agentResults",
        "resolvedCode.$": "$.resolvedCode"
      },
      "ResultPath": "$.finalOutput",
      "Next": "PublishToAppSync"
    },
    "PublishToAppSync": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT:function:velocity-publish-results",
      "Parameters": {
        "projectId.$": "$.projectId",
        "userId.$": "$.userId",
        "output.$": "$.finalOutput"
      },
      "End": true
    },
    "HandleError": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT:function:velocity-error-handler",
      "End": true
    }
  }
}
```

### 5.2 Agent Selection Logic

**Decision Tree for Model Selection**:

```
User Request Analysis:
├─ Contains "UI", "frontend", "component", "styling"?
│  └─ YES → Primary: Gemini 1.5 Pro (multimodal, large context)
│  └─ NO → Continue
│
├─ Contains "backend", "API", "database", "logic"?
│  └─ YES → Primary: Claude 3.5 Sonnet (precise code generation)
│  └─ NO → Continue
│
├─ Contains "architecture", "design", "plan", "strategy"?
│  └─ YES → Primary: GPT-4o (reasoning and planning)
│  └─ NO → Default to Claude 3.5 Sonnet
│
Fallback Strategy:
├─ Primary model unavailable (429 rate limit)?
│  └─ Route to secondary region (us-east-1)
│
├─ Secondary region unavailable?
│  └─ Use fallback model (Claude → GPT-4o → Gemini)
│
└─ All models unavailable?
   └─ Queue request in SQS, retry with exponential backoff
```

### 5.3 Context Handover Protocol

**Shared Memory Structure in Aurora**:

```sql
CREATE TABLE context_memory (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL,
    context_type VARCHAR(50) NOT NULL, -- 'file_structure', 'api_schema', 'dependencies'
    content JSONB NOT NULL,
    embedding vector(1536), -- pgvector for similarity search
    created_by VARCHAR(50), -- 'architect', 'logic', 'ui', 'user'
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP DEFAULT NOW() + INTERVAL '1 hour',
    INDEX idx_project_embedding USING ivfflat (embedding vector_cosine_ops)
);
```

**Handover Sequence**:

1. **Architect Agent** writes task manifest:
```json
{
  "context_type": "task_manifest",
  "content": {
    "logic_task": "Create POST /api/auth/login endpoint",
    "ui_task": "Build login form",
    "dependencies": {
      "ui_depends_on": ["logic.auth_endpoint"]
    }
  }
}
```

2. **Logic Agent** reads manifest, writes API schema:
```json
{
  "context_type": "api_schema",
  "content": {
    "endpoint": "/api/auth/login",
    "method": "POST",
    "request_body": {"email": "string", "password": "string"},
    "response": {"token": "string", "user": "object"}
  }
}
```

3. **Context Janitor** generates embedding and broadcasts via AppSync

4. **UI Agent** queries Aurora for relevant context:
```sql
SELECT content FROM context_memory
WHERE project_id = $1
  AND context_type = 'api_schema'
  AND embedding <=> $2 < 0.3 -- cosine similarity threshold
ORDER BY created_at DESC
LIMIT 5;
```

5. **UI Agent** uses retrieved context to generate form with correct API call


## 6. Save-Gate Version Authority Mechanism

### 6.1 Architecture

**Core Principle**: Owner maintains final authority while enabling real-time collaboration.

**State Transitions**:
```
[Draft State] → [Review Pending] → [Approved/Rejected] → [Committed State]
     ↓              ↓                    ↓                      ↓
   Redis         Aurora              Lambda                   S3
  (temp)       (metadata)          (workflow)              (persistent)
```

### 6.2 Implementation Details

#### Redis Data Structure for Draft State

```redis
# Multi-cursor positions (TTL: 5 minutes)
HSET project:{projectId}:cursors user:{userId} '{"file": "app.js", "line": 42, "col": 15}'

# Draft changes (TTL: 24 hours)
HSET project:{projectId}:drafts user:{userId}:file:{fileId} '{
  "original_hash": "a3f5b2c8",
  "modified_content": "...",
  "timestamp": 1704067200,
  "change_summary": "Added error handling"
}'

# Active collaborators (TTL: 1 hour)
SADD project:{projectId}:active_users user:{userId}
EXPIRE project:{projectId}:active_users 3600
```

#### Aurora Schema for Save-Gate

```sql
CREATE TABLE save_gate_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL,
    requester_id UUID NOT NULL,
    file_path VARCHAR(500) NOT NULL,
    diff_s3_key VARCHAR(500) NOT NULL, -- S3 location of diff
    ai_summary TEXT,
    status VARCHAR(20) DEFAULT 'pending', -- pending, approved, rejected
    reviewed_by UUID,
    reviewed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (project_id) REFERENCES projects(id),
    FOREIGN KEY (requester_id) REFERENCES users(id),
    FOREIGN KEY (reviewed_by) REFERENCES users(id)
);

CREATE TABLE project_permissions (
    project_id UUID NOT NULL,
    user_id UUID NOT NULL,
    role VARCHAR(20) NOT NULL, -- owner, editor, viewer
    can_approve_saves BOOLEAN DEFAULT FALSE,
    PRIMARY KEY (project_id, user_id)
);
```

#### Lambda Function: Save-Gate Workflow

```python
# velocity-save-gate-handler
import boto3
import json
from datetime import datetime

s3 = boto3.client('s3')
aurora = boto3.client('rds-data')
appsync = boto3.client('appsync')
sns = boto3.client('sns')

def lambda_handler(event, context):
    action = event['action']
    
    if action == 'request_save':
        return handle_save_request(event)
    elif action == 'approve':
        return handle_approval(event)
    elif action == 'reject':
        return handle_rejection(event)

def handle_save_request(event):
    project_id = event['projectId']
    user_id = event['userId']
    file_path = event['filePath']
    draft_content = event['draftContent']
    
    # 1. Generate diff
    original = get_file_from_s3(project_id, file_path)
    diff = generate_diff(original, draft_content)
    
    # 2. Store diff in S3
    diff_key = f"diffs/{project_id}/{user_id}/{datetime.now().isoformat()}.diff"
    s3.put_object(
        Bucket='velocity-diffs',
        Key=diff_key,
        Body=diff,
        ServerSideEncryption='aws:kms'
    )
    
    # 3. Generate AI summary via Bedrock
    summary = generate_ai_summary(diff)
    
    # 4. Create save-gate request in Aurora
    request_id = create_save_gate_request(
        project_id, user_id, file_path, diff_key, summary
    )
    
    # 5. Notify owner via SNS (triggers mobile push)
    owner_id = get_project_owner(project_id)
    sns.publish(
        TopicArn=f'arn:aws:sns:region:account:velocity-save-gate-{owner_id}',
        Message=json.dumps({
            'type': 'save_gate_request',
            'requestId': request_id,
            'summary': summary,
            'requester': get_user_name(user_id)
        })
    )
    
    # 6. Broadcast via AppSync
    appsync.post_to_connection(
        ConnectionId=get_owner_connection_id(owner_id),
        Data=json.dumps({
            'type': 'SAVE_GATE_REQUEST',
            'requestId': request_id
        })
    )
    
    return {'statusCode': 200, 'requestId': request_id}

def handle_approval(event):
    request_id = event['requestId']
    reviewer_id = event['reviewerId']
    
    # 1. Verify reviewer has permission
    if not can_approve_saves(reviewer_id, get_project_id(request_id)):
        return {'statusCode': 403, 'error': 'Unauthorized'}
    
    # 2. Get diff from S3
    diff_key = get_diff_key(request_id)
    diff = s3.get_object(Bucket='velocity-diffs', Key=diff_key)['Body'].read()
    
    # 3. Apply diff to file in S3
    apply_diff_to_s3(request_id, diff)
    
    # 4. Update Aurora
    update_save_gate_status(request_id, 'approved', reviewer_id)
    
    # 5. Trigger Context Janitor
    invoke_context_janitor(get_project_id(request_id))
    
    # 6. Update Skill Wallet for requester
    update_skill_wallet(get_requester_id(request_id), {
        'contribution_type': 'code_change',
        'lines_changed': count_lines_in_diff(diff),
        'approved_by': reviewer_id
    })
    
    # 7. Notify requester
    notify_user(get_requester_id(request_id), 'save_approved')
    
    return {'statusCode': 200, 'message': 'Approved'}
```

### 6.3 Conflict Resolution Strategy

**Scenario**: Two developers edit the same file simultaneously.

**Detection**:
```python
def detect_conflict(project_id, file_path):
    # Get all pending drafts for this file
    drafts = redis.hgetall(f"project:{project_id}:drafts:*:file:{file_path}")
    
    if len(drafts) > 1:
        # Multiple drafts exist
        # Check if they modify overlapping lines
        conflicts = []
        for draft1, draft2 in combinations(drafts, 2):
            overlap = check_line_overlap(draft1, draft2)
            if overlap:
                conflicts.append({
                    'users': [draft1['user'], draft2['user']],
                    'lines': overlap
                })
        return conflicts
    return None
```

**Resolution Options**:
1. **First-Come-First-Served**: First approved draft wins, second must rebase
2. **Owner Decision**: Owner reviews both diffs and chooses or merges manually
3. **AI-Assisted Merge**: Bedrock Claude analyzes both changes and proposes merge


## 7. Shared Memory & Context Janitor Design

### 7.1 Context Memory Architecture

**Purpose**: Maintain a real-time "source of truth" that all AI agents and human developers can query to understand the current state of the codebase.

**Storage Strategy**:
- **Hot Context** (last 1 hour): Aurora PostgreSQL with pgvector
- **Warm Context** (1 hour - 7 days): Aurora with standard indexing
- **Cold Context** (7+ days): S3 with Athena for historical queries

### 7.2 Context Types and Schema

```sql
-- Enum for context types
CREATE TYPE context_type_enum AS ENUM (
    'file_structure',      -- Directory tree and file organization
    'api_schema',          -- REST/GraphQL API definitions
    'database_schema',     -- Table structures and relationships
    'dependencies',        -- Package.json, requirements.txt, etc.
    'function_signature',  -- Function names, params, return types
    'type_definition',     -- TypeScript interfaces, Python classes
    'environment_config',  -- .env variables, config files
    'test_coverage',       -- Which functions have tests
    'error_log',           -- Recent errors and resolutions
    'user_intent'          -- Natural language description of changes
);

-- Main context table
CREATE TABLE context_memory (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL,
    context_type context_type_enum NOT NULL,
    file_path VARCHAR(500), -- NULL for project-wide context
    content JSONB NOT NULL,
    embedding vector(1536), -- OpenAI ada-002 or Titan embeddings
    metadata JSONB, -- Additional searchable metadata
    created_by VARCHAR(50) NOT NULL, -- 'architect', 'logic', 'ui', 'user:{id}'
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP DEFAULT NOW() + INTERVAL '1 hour',
    version INTEGER DEFAULT 1,
    parent_id UUID, -- For tracking context evolution
    FOREIGN KEY (project_id) REFERENCES projects(id),
    FOREIGN KEY (parent_id) REFERENCES context_memory(id)
);

-- Indexes for fast retrieval
CREATE INDEX idx_context_project_type ON context_memory(project_id, context_type);
CREATE INDEX idx_context_expires ON context_memory(expires_at) WHERE expires_at IS NOT NULL;
CREATE INDEX idx_context_embedding ON context_memory USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
CREATE INDEX idx_context_metadata ON context_memory USING gin(metadata);
```

### 7.3 Context Janitor Implementation

#### Lambda Function Configuration

```yaml
FunctionName: velocity-context-janitor
Runtime: python3.11
MemorySize: 1024
Timeout: 30
ReservedConcurrentExecutions: 50
Environment:
  AURORA_CLUSTER_ARN: arn:aws:rds:region:account:cluster:velocity-aurora
  AURORA_SECRET_ARN: arn:aws:secretsmanager:region:account:secret:aurora-creds
  BEDROCK_REGION: us-east-1
  APPSYNC_ENDPOINT: https://xyz.appsync-api.region.amazonaws.com/graphql
Layers:
  - arn:aws:lambda:region:account:layer:pgvector-python:1
  - arn:aws:lambda:region:account:layer:boto3-bedrock:1
```

#### Core Logic

```python
import boto3
import json
from datetime import datetime, timedelta
from pgvector.psycopg2 import register_vector

# AWS clients
rds_data = boto3.client('rds-data')
bedrock_runtime = boto3.client('bedrock-runtime', region_name='us-east-1')
appsync = boto3.client('appsync')
eventbridge = boto3.client('events')

def lambda_handler(event, context):
    """
    Triggered by:
    1. EventBridge schedule (every 10 seconds)
    2. File save events from AppSync
    3. Agent completion events from Step Functions
    """
    
    trigger_type = event.get('source', 'schedule')
    
    if trigger_type == 'schedule':
        # Periodic cleanup and update
        projects = get_active_projects()
        for project_id in projects:
            process_project(project_id)
    
    elif trigger_type == 'file_save':
        # Immediate update for file changes
        project_id = event['projectId']
        file_path = event['filePath']
        update_file_context(project_id, file_path)
    
    elif trigger_type == 'agent_completion':
        # Update after agent generates code
        project_id = event['projectId']
        agent_output = event['agentOutput']
        process_agent_output(project_id, agent_output)
    
    return {'statusCode': 200}

def process_project(project_id):
    """Main processing loop for a project"""
    
    # 1. Fetch recent changes
    recent_changes = get_recent_changes(project_id)
    
    # 2. Generate embeddings for new content
    for change in recent_changes:
        if not change.get('embedding'):
            embedding = generate_embedding(change['content'])
            update_embedding(change['id'], embedding)
    
    # 3. Detect conflicts
    conflicts = detect_conflicts(project_id)
    if conflicts:
        notify_conflicts(project_id, conflicts)
    
    # 4. Prune expired context
    prune_expired_context(project_id)
    
    # 5. Archive old context to S3
    archive_old_context(project_id)
    
    # 6. Broadcast updates
    broadcast_context_update(project_id)

def generate_embedding(content):
    """Generate vector embedding using Bedrock Titan"""
    
    # Truncate content if too long (max 8192 tokens for Titan)
    truncated = content[:30000]  # Rough token estimate
    
    response = bedrock_runtime.invoke_model(
        modelId='amazon.titan-embed-text-v1',
        body=json.dumps({
            'inputText': truncated
        })
    )
    
    result = json.loads(response['body'].read())
    return result['embedding']  # 1536-dimensional vector

def detect_conflicts(project_id):
    """Detect potential conflicts in shared context"""
    
    # Query for overlapping context updates
    query = """
    WITH recent_updates AS (
        SELECT 
            id,
            file_path,
            content,
            created_by,
            created_at,
            embedding
        FROM context_memory
        WHERE project_id = :project_id
          AND created_at > NOW() - INTERVAL '1 minute'
          AND context_type IN ('function_signature', 'api_schema')
    )
    SELECT 
        a.id as id1,
        b.id as id2,
        a.file_path,
        a.created_by as creator1,
        b.created_by as creator2,
        1 - (a.embedding <=> b.embedding) as similarity
    FROM recent_updates a
    CROSS JOIN recent_updates b
    WHERE a.id < b.id
      AND a.file_path = b.file_path
      AND a.created_by != b.created_by
      AND 1 - (a.embedding <=> b.embedding) > 0.95  -- High similarity = potential conflict
    """
    
    result = execute_aurora_query(query, {'project_id': project_id})
    
    conflicts = []
    for row in result:
        # Analyze if this is a true conflict
        conflict_analysis = analyze_conflict(row)
        if conflict_analysis['is_conflict']:
            conflicts.append(conflict_analysis)
    
    return conflicts

def analyze_conflict(similarity_row):
    """Use Bedrock to determine if similar changes are conflicting"""
    
    context1 = get_context_by_id(similarity_row['id1'])
    context2 = get_context_by_id(similarity_row['id2'])
    
    prompt = f"""
    Analyze if these two code changes conflict:
    
    Change 1 (by {similarity_row['creator1']}):
    {context1['content']}
    
    Change 2 (by {similarity_row['creator2']}):
    {context2['content']}
    
    Respond in JSON:
    {{
        "is_conflict": true/false,
        "reason": "explanation",
        "resolution_suggestion": "how to resolve"
    }}
    """
    
    response = bedrock_runtime.invoke_model(
        modelId='anthropic.claude-3-5-sonnet-20241022-v2:0',
        body=json.dumps({
            'anthropic_version': 'bedrock-2023-05-31',
            'max_tokens': 1024,
            'messages': [{'role': 'user', 'content': prompt}]
        })
    )
    
    result = json.loads(response['body'].read())
    return json.loads(result['content'][0]['text'])

def prune_expired_context(project_id):
    """Remove context that has exceeded its TTL"""
    
    # Move to S3 before deleting
    expired = execute_aurora_query("""
        SELECT * FROM context_memory
        WHERE project_id = :project_id
          AND expires_at < NOW()
    """, {'project_id': project_id})
    
    if expired:
        # Archive to S3
        s3_key = f"context-archive/{project_id}/{datetime.now().isoformat()}.json"
        boto3.client('s3').put_object(
            Bucket='velocity-context-archive',
            Key=s3_key,
            Body=json.dumps(expired, default=str),
            ServerSideEncryption='aws:kms'
        )
        
        # Delete from Aurora
        execute_aurora_query("""
            DELETE FROM context_memory
            WHERE project_id = :project_id
              AND expires_at < NOW()
        """, {'project_id': project_id})

def broadcast_context_update(project_id):
    """Notify all connected clients via AppSync"""
    
    mutation = """
    mutation PublishContextUpdate($projectId: ID!, $timestamp: AWSDateTime!) {
        publishContextUpdate(projectId: $projectId, timestamp: $timestamp) {
            success
        }
    }
    """
    
    # This triggers AppSync subscription for all clients
    appsync.post_to_connection(
        ConnectionId='broadcast',  # Special broadcast connection
        Data=json.dumps({
            'query': mutation,
            'variables': {
                'projectId': project_id,
                'timestamp': datetime.now().isoformat()
            }
        })
    )
```

### 7.4 Context Query Optimization

**Agent Context Retrieval**:

```python
def get_relevant_context_for_agent(project_id, agent_type, user_query):
    """
    Retrieve most relevant context for an agent's task
    Uses hybrid search: vector similarity + metadata filtering
    """
    
    # Generate embedding for user query
    query_embedding = generate_embedding(user_query)
    
    # Determine relevant context types based on agent
    context_types = {
        'architect': ['file_structure', 'api_schema', 'dependencies', 'user_intent'],
        'logic': ['api_schema', 'database_schema', 'function_signature', 'type_definition'],
        'ui': ['api_schema', 'type_definition', 'function_signature']
    }
    
    types = context_types.get(agent_type, ['file_structure'])
    
    # Hybrid search query
    query = """
    SELECT 
        id,
        context_type,
        file_path,
        content,
        1 - (embedding <=> :query_embedding::vector) as similarity,
        created_at
    FROM context_memory
    WHERE project_id = :project_id
      AND context_type = ANY(:context_types)
      AND (expires_at IS NULL OR expires_at > NOW())
    ORDER BY similarity DESC, created_at DESC
    LIMIT 10
    """
    
    results = execute_aurora_query(query, {
        'project_id': project_id,
        'query_embedding': query_embedding,
        'context_types': types
    })
    
    # Format for agent consumption
    context_string = format_context_for_agent(results)
    
    return context_string
```


## 8. Skill Wallet & Privacy Ledger Design

### 8.1 AWS Nitro Enclaves Architecture

**Purpose**: Process behavioral data in a cryptographically isolated environment where even AWS operators cannot access the data.

**Enclave Configuration**:
```yaml
ParentInstance: m5.xlarge
EnclaveConfiguration:
  Memory: 4096 MB
  CPUs: 2
  DebugMode: false  # Production must be false for attestation
```

**Attestation Document**: Cryptographic proof that code is running in genuine Nitro Enclave.

### 8.2 Data Flow into Enclave

```
1. Velocity Desktop Client (Local)
   ↓
   Collects behavioral data:
   - Keystrokes per minute (aggregated, not individual keys)
   - Time spent per file type (.js, .py, .css)
   - AI assistance usage (frequency, context)
   - Error resolution patterns (time to fix, approach)
   - Code refactoring patterns (before/after complexity)
   ↓
2. Local Encryption (AES-256-GCM)
   - Encrypted with ephemeral key
   - Key encrypted with Nitro Enclave public key
   ↓
3. Send to Parent EC2 Instance
   - HTTPS POST to /api/skill-wallet/process
   - Parent instance cannot decrypt (only enclave has private key)
   ↓
4. Parent forwards to Enclave via vsock
   - Virtual socket communication (isolated from network)
   ↓
5. Inside Nitro Enclave:
   - Decrypt with enclave private key
   - Process behavioral data
   - Generate skill hashes
   - Invoke Bedrock (via parent proxy, encrypted channel)
   - Sign results with KMS key (inside enclave)
   ↓
6. Return signed skill hashes (no raw data)
   ↓
7. Store in Aurora (only hashes and metadata)
```

### 8.3 Enclave Application Code

```python
# enclave_app.py - Runs inside Nitro Enclave
import json
import hashlib
import boto3
from datetime import datetime
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.asymmetric import padding

# Enclave-specific imports
from nsm_util import get_attestation_doc
from vsock_proxy import VsockProxy

def main():
    # Initialize vsock listener (port 5000)
    proxy = VsockProxy(port=5000)
    
    # Get attestation document to prove we're in genuine enclave
    attestation = get_attestation_doc()
    
    while True:
        # Receive encrypted behavioral data from parent
        encrypted_data = proxy.receive()
        
        # Decrypt with enclave private key
        behavioral_data = decrypt_data(encrypted_data)
        
        # Process data (never logs or stores raw data)
        skill_profile = analyze_behavioral_patterns(behavioral_data)
        
        # Generate skill hash
        skill_hash = generate_skill_hash(skill_profile)
        
        # Generate AI narration (via Bedrock proxy)
        narration = generate_narration(skill_profile)
        
        # Sign with KMS (via parent proxy, but key usage is attested)
        signature = sign_with_kms(skill_hash, attestation)
        
        # Return only processed results
        result = {
            'skill_hash': skill_hash,
            'skill_tags': skill_profile['tags'],
            'proficiency_scores': skill_profile['scores'],
            'narration_metadata': narration,
            'signature': signature,
            'attestation_doc': attestation  # Proves this came from enclave
        }
        
        proxy.send(result)

def analyze_behavioral_patterns(data):
    """
    Analyze coding behavior without storing raw code
    """
    patterns = {
        'typing_speed': calculate_typing_metrics(data['keystrokes']),
        'file_type_distribution': analyze_file_types(data['file_times']),
        'ai_usage_pattern': analyze_ai_usage(data['ai_interactions']),
        'error_resolution': analyze_error_patterns(data['errors']),
        'code_quality': analyze_quality_metrics(data['linter_results'])
    }
    
    # Derive skill tags
    tags = derive_skill_tags(patterns)
    
    # Calculate proficiency scores (0-100)
    scores = calculate_proficiency_scores(patterns)
    
    return {
        'tags': tags,
        'scores': scores,
        'patterns': patterns  # Aggregated only, no raw code
    }

def generate_skill_hash(skill_profile):
    """
    Generate deterministic hash of skill profile
    """
    # Sort for determinism
    profile_str = json.dumps(skill_profile, sort_keys=True)
    
    # Add timestamp and salt
    timestamp = datetime.utcnow().isoformat()
    data = f"{profile_str}:{timestamp}".encode()
    
    # SHA-256 hash
    return hashlib.sha256(data).hexdigest()

def generate_narration(skill_profile):
    """
    Generate AI narration of skills via Bedrock
    Bedrock call happens inside enclave via parent proxy
    """
    prompt = f"""
    Generate a 30-second narration script for a developer's skill profile:
    
    Skills: {', '.join(skill_profile['tags'])}
    Proficiency: {skill_profile['scores']}
    
    Focus on:
    - Problem-solving approach
    - Technical strengths
    - Growth areas
    
    Keep it professional and evidence-based.
    """
    
    # Call Bedrock via parent proxy (encrypted channel)
    response = call_bedrock_via_proxy(
        model='anthropic.claude-3-5-sonnet-20241022-v2:0',
        prompt=prompt
    )
    
    return {
        'script': response['narration'],
        'duration_seconds': 30,
        'generated_at': datetime.utcnow().isoformat()
    }

def sign_with_kms(data, attestation):
    """
    Sign data with KMS key, including attestation proof
    """
    # KMS call via parent proxy
    # KMS policy requires valid attestation document
    signature = call_kms_via_proxy(
        key_id='alias/velocity-skill-wallet',
        message=data.encode(),
        attestation=attestation
    )
    
    return signature
```

### 8.4 KMS Key Policy for Enclave

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Enable enclave to sign skill hashes",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT:role/VelocityEnclaveRole"
      },
      "Action": "kms:Sign",
      "Resource": "*",
      "Condition": {
        "StringEqualsIgnoreCase": {
          "kms:RecipientAttestation:ImageSha384": "ENCLAVE_IMAGE_HASH"
        }
      }
    }
  ]
}
```

### 8.5 Skill Wallet Database Schema

```sql
CREATE TABLE skill_wallets (
    user_id UUID PRIMARY KEY,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    visibility VARCHAR(20) DEFAULT 'private', -- private, public, recruiters_only
    FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE TABLE skill_hashes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    skill_hash VARCHAR(64) NOT NULL, -- SHA-256 hash
    skill_tags TEXT[] NOT NULL, -- ['React', 'TypeScript', 'State Management']
    proficiency_scores JSONB NOT NULL, -- {"react": 95, "typescript": 88}
    behavioral_summary JSONB NOT NULL, -- Aggregated patterns, no raw code
    narration_metadata JSONB, -- AI-generated narration script
    signature TEXT NOT NULL, -- KMS signature
    attestation_doc TEXT NOT NULL, -- Nitro Enclave attestation
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP, -- Optional expiration for time-sensitive skills
    is_public BOOLEAN DEFAULT FALSE,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE TABLE skill_playbacks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    skill_hash_id UUID NOT NULL,
    s3_video_key VARCHAR(500), -- S3 location of narrated video
    duration_seconds INTEGER,
    thumbnail_s3_key VARCHAR(500),
    view_count INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (skill_hash_id) REFERENCES skill_hashes(id)
);

CREATE TABLE recruiter_access_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    recruiter_id UUID NOT NULL,
    skill_wallet_user_id UUID NOT NULL,
    accessed_at TIMESTAMP DEFAULT NOW(),
    access_type VARCHAR(50), -- 'view_profile', 'view_playback', 'download_report'
    ip_address INET,
    user_agent TEXT,
    FOREIGN KEY (recruiter_id) REFERENCES users(id),
    FOREIGN KEY (skill_wallet_user_id) REFERENCES users(id)
);

-- Indexes
CREATE INDEX idx_skill_tags ON skill_hashes USING gin(skill_tags);
CREATE INDEX idx_skill_proficiency ON skill_hashes USING gin(proficiency_scores);
CREATE INDEX idx_public_skills ON skill_hashes(is_public) WHERE is_public = true;
```

### 8.6 Privacy Controls

**User Privacy Dashboard**:
```typescript
interface PrivacySettings {
  skillWalletVisibility: 'private' | 'public' | 'recruiters_only';
  allowedRecruiters: string[]; // Whitelist of recruiter IDs
  publicSkills: string[]; // Which skill tags to show publicly
  hideSkills: string[]; // Which skills to never show
  allowPlaybackSharing: boolean;
  dataRetentionDays: number; // Auto-delete old skill hashes
}
```

**Recruiter Access Control**:
```python
def can_recruiter_access_profile(recruiter_id, user_id):
    """
    Check if recruiter can access user's skill wallet
    """
    # Get user's privacy settings
    settings = get_privacy_settings(user_id)
    
    if settings['visibility'] == 'private':
        return False
    
    if settings['visibility'] == 'recruiters_only':
        # Check if recruiter is in whitelist
        if recruiter_id not in settings['allowed_recruiters']:
            return False
    
    # Log access attempt
    log_recruiter_access(recruiter_id, user_id, 'attempt')
    
    return True
```


## 9. Security Architecture

### 9.1 Defense in Depth Strategy

```
Layer 1: Network Security
├─ AWS WAF: SQL injection, XSS protection
├─ CloudFront: DDoS protection, geo-blocking
├─ VPC: Private subnets for Aurora, Lambda
└─ Security Groups: Least privilege network access

Layer 2: Authentication & Authorization
├─ AWS Cognito: User authentication with MFA
├─ IAM Roles: Service-to-service authentication
├─ API Gateway: JWT validation, rate limiting
└─ AppSync: Field-level authorization

Layer 3: Data Protection
├─ Encryption at Rest: KMS for all data stores
├─ Encryption in Transit: TLS 1.3 everywhere
├─ Nitro Enclaves: Isolated behavioral processing
└─ S3 Bucket Policies: Deny public access

Layer 4: Application Security
├─ Input Validation: All user inputs sanitized
├─ Output Encoding: Prevent XSS in responses
├─ CSRF Protection: Token-based validation
└─ Secrets Management: AWS Secrets Manager

Layer 5: Monitoring & Response
├─ CloudWatch: Real-time log analysis
├─ GuardDuty: Threat detection
├─ Security Hub: Compliance monitoring
└─ CloudTrail: Audit logging
```

### 9.2 Authentication Flow

```
1. User Login Request
   ↓
2. Cognito User Pool
   - Username/password validation
   - MFA challenge (TOTP or SMS)
   ↓
3. Issue JWT Tokens
   - ID Token: User identity claims
   - Access Token: API authorization
   - Refresh Token: Long-lived session
   ↓
4. Client stores tokens (secure storage)
   - Web: HttpOnly cookies + localStorage
   - Desktop: Encrypted keychain
   - Mobile: Secure enclave (iOS) / Keystore (Android)
   ↓
5. API Request with Access Token
   ↓
6. API Gateway validates token
   - Signature verification
   - Expiration check
   - Scope validation
   ↓
7. Request forwarded to backend with user context
```

### 9.3 IAM Roles and Policies

#### Lambda Execution Role

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "rds-data:ExecuteStatement",
        "rds-data:BatchExecuteStatement"
      ],
      "Resource": "arn:aws:rds:*:ACCOUNT:cluster:velocity-aurora"
    },
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:*:ACCOUNT:secret:aurora-*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel"
      ],
      "Resource": [
        "arn:aws:bedrock:*::foundation-model/anthropic.claude-*",
        "arn:aws:bedrock:*::foundation-model/amazon.titan-*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::velocity-*/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:ACCOUNT:log-group:/aws/lambda/velocity-*"
    }
  ]
}
```

#### Step Functions Execution Role

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": ["us-east-1", "us-west-2"]
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "lambda:InvokeFunction"
      ],
      "Resource": "arn:aws:lambda:*:ACCOUNT:function:velocity-*"
    }
  ]
}
```

### 9.4 Data Encryption Strategy

#### Encryption at Rest

| Service | Encryption Method | Key Management |
|---------|------------------|----------------|
| Aurora PostgreSQL | AES-256 | AWS KMS (customer-managed) |
| ElastiCache Redis | AES-256 | AWS-managed |
| S3 | SSE-KMS | Customer-managed CMK |
| EBS (AppStream) | AES-256 | AWS-managed |
| Secrets Manager | AES-256 | AWS-managed |

#### Encryption in Transit

- **Client to CloudFront**: TLS 1.3
- **CloudFront to API Gateway**: TLS 1.3
- **API Gateway to Lambda**: AWS PrivateLink (encrypted)
- **Lambda to Aurora**: TLS 1.2+ with certificate validation
- **Lambda to Bedrock**: TLS 1.3 with SigV4 signing
- **AppStream to Client**: HTTPS with custom certificate

### 9.5 Secrets Management

```python
# Example: Retrieving database credentials
import boto3
import json

secrets_client = boto3.client('secretsmanager')

def get_db_credentials():
    """
    Retrieve Aurora credentials from Secrets Manager
    Credentials automatically rotated every 30 days
    """
    secret_name = "velocity/aurora/master"
    
    response = secrets_client.get_secret_value(SecretId=secret_name)
    secret = json.loads(response['SecretString'])
    
    return {
        'host': secret['host'],
        'port': secret['port'],
        'username': secret['username'],
        'password': secret['password'],
        'database': secret['dbname']
    }

# Bedrock API keys (if using third-party models)
def get_bedrock_api_key(model_provider):
    """
    Retrieve API keys for external model providers
    """
    secret_name = f"velocity/bedrock/{model_provider}"
    response = secrets_client.get_secret_value(SecretId=secret_name)
    return response['SecretString']
```

### 9.6 Security Monitoring

#### CloudWatch Alarms

```yaml
Alarms:
  - Name: HighFailedLoginAttempts
    Metric: Cognito.SignInAttempts
    Threshold: 10 failed attempts in 5 minutes
    Action: SNS notification to security team
  
  - Name: UnauthorizedAPIAccess
    Metric: APIGateway.4XXError
    Threshold: 100 errors in 1 minute
    Action: Trigger Lambda to block IP via WAF
  
  - Name: AnomalousDataAccess
    Metric: Aurora.SelectThroughput
    Threshold: 3 std deviations from baseline
    Action: Alert + automatic read replica throttling
  
  - Name: EnclaveAttestationFailure
    Metric: Custom.EnclaveAttestation
    Threshold: Any failure
    Action: Immediate alert + disable skill wallet processing
```

#### GuardDuty Integration

```python
# Lambda function triggered by GuardDuty findings
def handle_guardduty_finding(event, context):
    """
    Respond to security threats detected by GuardDuty
    """
    finding = event['detail']
    severity = finding['severity']
    finding_type = finding['type']
    
    if severity >= 7.0:  # High or Critical
        if 'UnauthorizedAccess' in finding_type:
            # Block the source IP
            block_ip_in_waf(finding['service']['action']['networkConnectionAction']['remoteIpDetails']['ipAddressV4'])
        
        elif 'InstanceCredentialExfiltration' in finding_type:
            # Rotate compromised credentials
            rotate_credentials(finding['resource']['instanceDetails']['instanceId'])
        
        # Always notify security team
        notify_security_team(finding)
```


## 10. Scalability & Fault Tolerance Strategy

### 10.1 Auto-Scaling Configuration

#### AppStream 2.0 Fleet Scaling

```yaml
FleetName: velocity-web-ide
FleetType: ON_DEMAND
InstanceType: stream.standard.medium
ComputeCapacity:
  DesiredInstances: 10
ScalingPolicies:
  - PolicyName: ScaleOnCapacity
    TargetTrackingConfiguration:
      TargetValue: 75.0  # Target 75% capacity utilization
      PredefinedMetricType: CapacityUtilization
    ScaleInCooldown: 300
    ScaleOutCooldown: 60
  - PolicyName: ScaleOnSchedule
    ScheduledActions:
      - ScheduleName: MorningRampUp
        StartTime: "08:00"
        MinCapacity: 50
      - ScheduleName: EveningRampDown
        StartTime: "22:00"
        MinCapacity: 10
MaxConcurrentSessions: 1000
DisconnectTimeoutInSeconds: 900
IdleDisconnectTimeoutInSeconds: 1800
```

#### Lambda Concurrency Management

```yaml
Functions:
  velocity-context-janitor:
    ReservedConcurrentExecutions: 50
    ProvisionedConcurrency: 10  # Always warm
  
  velocity-save-gate-handler:
    ReservedConcurrentExecutions: 100
    ProvisionedConcurrency: 20
  
  velocity-skill-wallet-processor:
    ReservedConcurrentExecutions: 30
    ProvisionedConcurrency: 5
```

#### Aurora Auto-Scaling

```yaml
AuroraCluster:
  Engine: aurora-postgresql
  EngineVersion: 15.4
  MinCapacity: 2 ACUs
  MaxCapacity: 64 ACUs
  AutoPause: false  # Production should not pause
  SecondsUntilAutoPause: N/A
  
  ReadReplicas:
    Count: 2
    AutoScaling:
      MinCapacity: 1
      MaxCapacity: 5
      TargetCPU: 70
      TargetConnections: 1000
```

### 10.2 High Availability Architecture

```
Region: ap-south-1 (Mumbai)
├─ Availability Zone A
│  ├─ Public Subnet: NAT Gateway, ALB
│  ├─ Private Subnet: Lambda, AppStream
│  └─ Data Subnet: Aurora Primary
│
├─ Availability Zone B
│  ├─ Public Subnet: NAT Gateway, ALB
│  ├─ Private Subnet: Lambda, AppStream
│  └─ Data Subnet: Aurora Replica 1
│
└─ Availability Zone C
   ├─ Public Subnet: NAT Gateway, ALB
   ├─ Private Subnet: Lambda, AppStream
   └─ Data Subnet: Aurora Replica 2

Cross-Region Replication:
└─ Region: us-east-1 (N. Virginia)
   └─ Aurora Global Database (Read Replica)
   └─ S3 Cross-Region Replication
   └─ Bedrock Failover Region
```

### 10.3 Fault Tolerance Mechanisms

#### Circuit Breaker Pattern

```python
from functools import wraps
import time

class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failures = 0
        self.last_failure_time = None
        self.state = 'CLOSED'  # CLOSED, OPEN, HALF_OPEN
    
    def call(self, func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            if self.state == 'OPEN':
                if time.time() - self.last_failure_time > self.timeout:
                    self.state = 'HALF_OPEN'
                else:
                    raise Exception("Circuit breaker is OPEN")
            
            try:
                result = func(*args, **kwargs)
                if self.state == 'HALF_OPEN':
                    self.state = 'CLOSED'
                    self.failures = 0
                return result
            
            except Exception as e:
                self.failures += 1
                self.last_failure_time = time.time()
                
                if self.failures >= self.failure_threshold:
                    self.state = 'OPEN'
                
                raise e
        
        return wrapper

# Usage
bedrock_breaker = CircuitBreaker(failure_threshold=3, timeout=30)

@bedrock_breaker.call
def invoke_bedrock_model(model_id, prompt):
    return bedrock_runtime.invoke_model(
        modelId=model_id,
        body=json.dumps({'prompt': prompt})
    )
```

#### Retry Strategy with Exponential Backoff

```python
import time
import random

def exponential_backoff_retry(func, max_retries=3, base_delay=1):
    """
    Retry function with exponential backoff and jitter
    """
    for attempt in range(max_retries):
        try:
            return func()
        except Exception as e:
            if attempt == max_retries - 1:
                raise e
            
            # Calculate delay: base * 2^attempt + random jitter
            delay = base_delay * (2 ** attempt) + random.uniform(0, 1)
            time.sleep(delay)
            
            print(f"Retry attempt {attempt + 1} after {delay:.2f}s")

# Usage
def risky_operation():
    response = exponential_backoff_retry(
        lambda: bedrock_runtime.invoke_model(modelId='...', body='...'),
        max_retries=3,
        base_delay=2
    )
    return response
```

#### Dead Letter Queue (DLQ) Configuration

```yaml
StepFunctions:
  StateMachine: velocity-multi-agent-orchestrator
  ErrorHandling:
    Catch:
      - ErrorEquals: ["States.ALL"]
        ResultPath: "$.error"
        Next: SendToDLQ
    
    SendToDLQ:
      Type: Task
      Resource: arn:aws:sqs:region:account:velocity-failed-orchestrations
      Parameters:
        MessageBody.$: "$"
        MessageAttributes:
          ErrorType:
            StringValue.$: "$.error.Error"
          Timestamp:
            StringValue.$: "$$.State.EnteredTime"

Lambda:
  Functions:
    velocity-context-janitor:
      DeadLetterConfig:
        TargetArn: arn:aws:sqs:region:account:velocity-context-janitor-dlq
      OnFailure:
        Destination: arn:aws:sns:region:account:velocity-lambda-failures
```

### 10.4 Data Backup and Recovery

#### Aurora Backup Strategy

```yaml
BackupRetentionPeriod: 30 days
PreferredBackupWindow: "03:00-04:00 UTC"  # Low traffic period
BackupType: Automated + Manual Snapshots

PointInTimeRecovery:
  Enabled: true
  EarliestRestorableTime: Current time - 30 days

ManualSnapshots:
  Schedule: Daily at 02:00 UTC
  Retention: 90 days
  CrossRegionCopy:
    Enabled: true
    DestinationRegion: us-east-1
    KmsKeyId: arn:aws:kms:us-east-1:account:key/...
```

#### S3 Versioning and Lifecycle

```yaml
Bucket: velocity-user-code
Versioning: Enabled
LifecycleRules:
  - Id: TransitionToIA
    Status: Enabled
    Transitions:
      - Days: 30
        StorageClass: STANDARD_IA
      - Days: 90
        StorageClass: GLACIER_IR
  
  - Id: DeleteOldVersions
    Status: Enabled
    NoncurrentVersionExpiration:
      NoncurrentDays: 90
  
  - Id: AbortIncompleteMultipart
    Status: Enabled
    AbortIncompleteMultipartUpload:
      DaysAfterInitiation: 7

Replication:
  Role: arn:aws:iam::account:role/s3-replication
  Rules:
    - Id: ReplicateToUSEast1
      Status: Enabled
      Priority: 1
      Destination:
        Bucket: arn:aws:s3:::velocity-user-code-replica
        ReplicationTime:
          Status: Enabled
          Time:
            Minutes: 15
```

### 10.5 Disaster Recovery Plan

**RTO (Recovery Time Objective)**: 1 hour  
**RPO (Recovery Point Objective)**: 1 second (Aurora Global Database)

**Failover Procedure**:

```python
# Automated failover script
import boto3

def initiate_disaster_recovery():
    """
    Failover to secondary region (us-east-1)
    """
    # 1. Promote Aurora read replica to primary
    rds = boto3.client('rds', region_name='us-east-1')
    rds.failover_global_cluster(
        GlobalClusterIdentifier='velocity-global-cluster',
        TargetDbClusterIdentifier='velocity-aurora-us-east-1'
    )
    
    # 2. Update Route 53 to point to us-east-1
    route53 = boto3.client('route53')
    route53.change_resource_record_sets(
        HostedZoneId='Z1234567890ABC',
        ChangeBatch={
            'Changes': [{
                'Action': 'UPSERT',
                'ResourceRecordSet': {
                    'Name': 'api.velocity.dev',
                    'Type': 'A',
                    'AliasTarget': {
                        'HostedZoneId': 'Z1234567890DEF',
                        'DNSName': 'api-us-east-1.velocity.dev',
                        'EvaluateTargetHealth': True
                    }
                }
            }]
        }
    )
    
    # 3. Update AppSync endpoint
    appsync = boto3.client('appsync', region_name='us-east-1')
    # Update GraphQL API configuration
    
    # 4. Notify operations team
    sns = boto3.client('sns')
    sns.publish(
        TopicArn='arn:aws:sns:us-east-1:account:velocity-ops',
        Subject='DISASTER RECOVERY INITIATED',
        Message='Failover to us-east-1 completed. RTO: <1 hour achieved.'
    )
    
    return {'status': 'success', 'new_region': 'us-east-1'}
```


## 11. API Layer Design

### 11.1 API Architecture

**Hybrid Approach**: REST (API Gateway) + GraphQL (AppSync)

- **REST APIs**: CRUD operations, authentication, file uploads
- **GraphQL**: Real-time collaboration, subscriptions, complex queries

### 11.2 REST API Endpoints (API Gateway)

```yaml
BasePath: /api/v1

Endpoints:
  # Authentication
  POST /auth/register:
    Handler: velocity-auth-register
    Auth: None (public)
    RateLimit: 5 req/min per IP
  
  POST /auth/login:
    Handler: velocity-auth-login
    Auth: None (public)
    RateLimit: 10 req/min per IP
  
  POST /auth/refresh:
    Handler: velocity-auth-refresh
    Auth: Refresh token
    RateLimit: 60 req/hour
  
  # Projects
  GET /projects:
    Handler: velocity-projects-list
    Auth: JWT required
    RateLimit: 100 req/min
  
  POST /projects:
    Handler: velocity-projects-create
    Auth: JWT required
    RateLimit: 10 req/min
  
  GET /projects/{projectId}:
    Handler: velocity-projects-get
    Auth: JWT + project access
    RateLimit: 100 req/min
  
  PUT /projects/{projectId}:
    Handler: velocity-projects-update
    Auth: JWT + owner role
    RateLimit: 50 req/min
  
  DELETE /projects/{projectId}:
    Handler: velocity-projects-delete
    Auth: JWT + owner role
    RateLimit: 10 req/min
  
  # Files
  GET /projects/{projectId}/files:
    Handler: velocity-files-list
    Auth: JWT + project access
    RateLimit: 100 req/min
  
  GET /projects/{projectId}/files/{filePath}:
    Handler: velocity-files-get
    Auth: JWT + project access
    RateLimit: 200 req/min
  
  PUT /projects/{projectId}/files/{filePath}:
    Handler: velocity-files-update
    Auth: JWT + editor role
    RateLimit: 100 req/min
  
  # AI Agents
  POST /agents/invoke:
    Handler: velocity-agents-invoke
    Auth: JWT required
    RateLimit: 20 req/min
    Timeout: 300s
  
  GET /agents/status/{executionId}:
    Handler: velocity-agents-status
    Auth: JWT required
    RateLimit: 100 req/min
  
  # Skill Wallet
  GET /skill-wallet/{userId}:
    Handler: velocity-skill-wallet-get
    Auth: JWT + privacy check
    RateLimit: 50 req/min
  
  POST /skill-wallet/process:
    Handler: velocity-skill-wallet-process
    Auth: JWT required
    RateLimit: 10 req/hour
  
  # Learning
  POST /learning/roadmap:
    Handler: velocity-learning-roadmap
    Auth: JWT required
    RateLimit: 5 req/day
  
  GET /learning/progress:
    Handler: velocity-learning-progress
    Auth: JWT required
    RateLimit: 100 req/min
```

### 11.3 GraphQL Schema (AppSync)

```graphql
# Types
type User {
  id: ID!
  email: String!
  displayName: String!
  avatarUrl: String
  skillWallet: SkillWallet
  projects: [Project!]!
}

type Project {
  id: ID!
  name: String!
  description: String
  owner: User!
  collaborators: [Collaborator!]!
  files: [File!]!
  contextMemory: [ContextEntry!]!
  createdAt: AWSDateTime!
  updatedAt: AWSDateTime!
}

type Collaborator {
  user: User!
  role: CollaboratorRole!
  canApproveSaves: Boolean!
  joinedAt: AWSDateTime!
}

enum CollaboratorRole {
  OWNER
  EDITOR
  VIEWER
}

type File {
  id: ID!
  path: String!
  content: String!
  language: String!
  size: Int!
  lastModifiedBy: User!
  lastModifiedAt: AWSDateTime!
}

type ContextEntry {
  id: ID!
  contextType: ContextType!
  content: AWSJSON!
  createdBy: String!
  createdAt: AWSDateTime!
  expiresAt: AWSDateTime
}

enum ContextType {
  FILE_STRUCTURE
  API_SCHEMA
  DATABASE_SCHEMA
  DEPENDENCIES
  FUNCTION_SIGNATURE
  TYPE_DEFINITION
}

type CursorPosition {
  userId: ID!
  userName: String!
  filePath: String!
  line: Int!
  column: Int!
  timestamp: AWSDateTime!
}

type SaveGateRequest {
  id: ID!
  projectId: ID!
  requester: User!
  filePath: String!
  aiSummary: String!
  status: SaveGateStatus!
  reviewedBy: User
  reviewedAt: AWSDateTime
  createdAt: AWSDateTime!
}

enum SaveGateStatus {
  PENDING
  APPROVED
  REJECTED
}

type SkillWallet {
  userId: ID!
  visibility: SkillWalletVisibility!
  skillHashes: [SkillHash!]!
  updatedAt: AWSDateTime!
}

enum SkillWalletVisibility {
  PRIVATE
  PUBLIC
  RECRUITERS_ONLY
}

type SkillHash {
  id: ID!
  hash: String!
  skillTags: [String!]!
  proficiencyScores: AWSJSON!
  narrationMetadata: AWSJSON
  createdAt: AWSDateTime!
  isPublic: Boolean!
}

# Queries
type Query {
  getUser(id: ID!): User
  getProject(id: ID!): Project
  listProjects(limit: Int, nextToken: String): ProjectConnection!
  getFile(projectId: ID!, filePath: String!): File
  getContextMemory(projectId: ID!, contextType: ContextType): [ContextEntry!]!
  getSaveGateRequests(projectId: ID!, status: SaveGateStatus): [SaveGateRequest!]!
  getSkillWallet(userId: ID!): SkillWallet
}

type ProjectConnection {
  items: [Project!]!
  nextToken: String
}

# Mutations
type Mutation {
  createProject(input: CreateProjectInput!): Project!
  updateProject(id: ID!, input: UpdateProjectInput!): Project!
  deleteProject(id: ID!): Boolean!
  
  updateFile(projectId: ID!, filePath: String!, content: String!): File!
  
  addCollaborator(projectId: ID!, userId: ID!, role: CollaboratorRole!): Collaborator!
  removeCollaborator(projectId: ID!, userId: ID!): Boolean!
  
  requestSaveGate(projectId: ID!, filePath: String!, draftContent: String!): SaveGateRequest!
  approveSaveGate(requestId: ID!): SaveGateRequest!
  rejectSaveGate(requestId: ID!, reason: String): SaveGateRequest!
  
  updateCursorPosition(projectId: ID!, filePath: String!, line: Int!, column: Int!): CursorPosition!
  
  updateSkillWalletVisibility(visibility: SkillWalletVisibility!): SkillWallet!
}

# Subscriptions (Real-time)
type Subscription {
  onFileUpdated(projectId: ID!): File
    @aws_subscribe(mutations: ["updateFile"])
  
  onCursorMoved(projectId: ID!): CursorPosition
    @aws_subscribe(mutations: ["updateCursorPosition"])
  
  onContextUpdated(projectId: ID!): ContextEntry
  
  onSaveGateRequest(projectId: ID!): SaveGateRequest
    @aws_subscribe(mutations: ["requestSaveGate"])
  
  onSaveGateResolved(projectId: ID!): SaveGateRequest
    @aws_subscribe(mutations: ["approveSaveGate", "rejectSaveGate"])
}

# Inputs
input CreateProjectInput {
  name: String!
  description: String
  template: String
}

input UpdateProjectInput {
  name: String
  description: String
}
```

### 11.4 API Gateway Configuration

```yaml
RestApi:
  Name: velocity-api
  EndpointType: REGIONAL
  
  Authorizers:
    CognitoAuthorizer:
      Type: COGNITO_USER_POOLS
      ProviderARNs:
        - arn:aws:cognito-idp:region:account:userpool/velocity-users
      IdentitySource: method.request.header.Authorization
  
  UsagePlans:
    FreeTier:
      Quota:
        Limit: 1000
        Period: DAY
      Throttle:
        BurstLimit: 20
        RateLimit: 10
    
    ProTier:
      Quota:
        Limit: 10000
        Period: DAY
      Throttle:
        BurstLimit: 100
        RateLimit: 50
    
    EnterpriseTier:
      Quota:
        Limit: 100000
        Period: DAY
      Throttle:
        BurstLimit: 500
        RateLimit: 200
  
  Stages:
    - StageName: prod
      ThrottleSettings:
        BurstLimit: 5000
        RateLimit: 2000
      MethodSettings:
        - ResourcePath: "/*"
          HttpMethod: "*"
          LoggingLevel: INFO
          DataTraceEnabled: false
          MetricsEnabled: true
  
  BinaryMediaTypes:
    - "application/octet-stream"
    - "image/*"
    - "video/*"
```

### 11.5 AppSync Configuration

```yaml
GraphQLApi:
  Name: velocity-graphql
  AuthenticationType: AMAZON_COGNITO_USER_POOLS
  UserPoolConfig:
    UserPoolId: velocity-users
    AwsRegion: ap-south-1
    DefaultAction: ALLOW
  
  AdditionalAuthenticationProviders:
    - AuthenticationType: AWS_IAM  # For service-to-service
  
  LogConfig:
    CloudWatchLogsRoleArn: arn:aws:iam::account:role/appsync-logs
    FieldLogLevel: ERROR
    ExcludeVerboseContent: false
  
  XrayEnabled: true

DataSources:
  AuroraDataSource:
    Type: RELATIONAL_DATABASE
    RelationalDatabaseConfig:
      RdsHttpEndpointConfig:
        AwsRegion: ap-south-1
        DbClusterIdentifier: velocity-aurora
        DatabaseName: velocity
        Schema: public
      RelationalDatabaseSourceType: RDS_HTTP_ENDPOINT
  
  LambdaDataSource:
    Type: AWS_LAMBDA
    LambdaConfig:
      LambdaFunctionArn: arn:aws:lambda:region:account:function:velocity-graphql-resolver

Resolvers:
  Query.getProject:
    DataSource: AuroraDataSource
    RequestMappingTemplate: |
      {
        "version": "2018-05-29",
        "statements": [
          "SELECT * FROM projects WHERE id = :PROJECT_ID"
        ],
        "variableMap": {
          ":PROJECT_ID": $util.toJson($ctx.args.id)
        }
      }
    ResponseMappingTemplate: |
      $util.toJson($util.rds.toJsonObject($ctx.result)[0])
  
  Mutation.updateFile:
    DataSource: LambdaDataSource
    # Lambda handles complex logic (S3 update, context janitor trigger)
  
  Subscription.onCursorMoved:
    # Managed by AppSync, no custom resolver needed
```


## 12. Database Schema Overview

### 12.1 Aurora PostgreSQL Schema

```sql
-- Enable extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgvector";
CREATE EXTENSION IF NOT EXISTS "pg_cron";

-- Users table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    cognito_sub VARCHAR(255) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    display_name VARCHAR(255) NOT NULL,
    avatar_url VARCHAR(500),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    last_login_at TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE,
    subscription_tier VARCHAR(20) DEFAULT 'free', -- free, pro, enterprise
    INDEX idx_users_email (email),
    INDEX idx_users_cognito (cognito_sub)
);

-- Projects table
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    owner_id UUID NOT NULL,
    s3_bucket VARCHAR(255) NOT NULL,
    s3_prefix VARCHAR(500) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    is_archived BOOLEAN DEFAULT FALSE,
    FOREIGN KEY (owner_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_projects_owner (owner_id),
    INDEX idx_projects_active (is_archived) WHERE is_archived = FALSE
);

-- Project collaborators
CREATE TABLE project_collaborators (
    project_id UUID NOT NULL,
    user_id UUID NOT NULL,
    role VARCHAR(20) NOT NULL, -- owner, editor, viewer
    can_approve_saves BOOLEAN DEFAULT FALSE,
    joined_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (project_id, user_id),
    FOREIGN KEY (project_id) REFERENCES projects(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_collab_user (user_id)
);

-- Context memory (shared memory for AI agents)
CREATE TABLE context_memory (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL,
    context_type VARCHAR(50) NOT NULL,
    file_path VARCHAR(500),
    content JSONB NOT NULL,
    embedding vector(1536),
    metadata JSONB,
    created_by VARCHAR(50) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP DEFAULT NOW() + INTERVAL '1 hour',
    version INTEGER DEFAULT 1,
    parent_id UUID,
    FOREIGN KEY (project_id) REFERENCES projects(id) ON DELETE CASCADE,
    FOREIGN KEY (parent_id) REFERENCES context_memory(id) ON DELETE SET NULL,
    INDEX idx_context_project_type (project_id, context_type),
    INDEX idx_context_expires (expires_at) WHERE expires_at IS NOT NULL,
    INDEX idx_context_embedding USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100)
);

-- Save-gate requests
CREATE TABLE save_gate_requests (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL,
    requester_id UUID NOT NULL,
    file_path VARCHAR(500) NOT NULL,
    diff_s3_key VARCHAR(500) NOT NULL,
    ai_summary TEXT,
    status VARCHAR(20) DEFAULT 'pending',
    reviewed_by UUID,
    reviewed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (project_id) REFERENCES projects(id) ON DELETE CASCADE,
    FOREIGN KEY (requester_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (reviewed_by) REFERENCES users(id) ON DELETE SET NULL,
    INDEX idx_savegate_project_status (project_id, status),
    INDEX idx_savegate_requester (requester_id)
);

-- Skill wallets
CREATE TABLE skill_wallets (
    user_id UUID PRIMARY KEY,
    visibility VARCHAR(20) DEFAULT 'private',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

-- Skill hashes
CREATE TABLE skill_hashes (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL,
    skill_hash VARCHAR(64) NOT NULL,
    skill_tags TEXT[] NOT NULL,
    proficiency_scores JSONB NOT NULL,
    behavioral_summary JSONB NOT NULL,
    narration_metadata JSONB,
    signature TEXT NOT NULL,
    attestation_doc TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP,
    is_public BOOLEAN DEFAULT FALSE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_skill_user (user_id),
    INDEX idx_skill_tags USING gin(skill_tags),
    INDEX idx_skill_proficiency USING gin(proficiency_scores),
    INDEX idx_skill_public (is_public) WHERE is_public = TRUE
);

-- Skill playbacks
CREATE TABLE skill_playbacks (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    skill_hash_id UUID NOT NULL,
    s3_video_key VARCHAR(500),
    duration_seconds INTEGER,
    thumbnail_s3_key VARCHAR(500),
    view_count INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (skill_hash_id) REFERENCES skill_hashes(id) ON DELETE CASCADE,
    INDEX idx_playback_skill (skill_hash_id)
);

-- Learning roadmaps
CREATE TABLE learning_roadmaps (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    roadmap_data JSONB NOT NULL, -- Structured roadmap with modules and skill gates
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_roadmap_user (user_id)
);

-- Learning progress
CREATE TABLE learning_progress (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL,
    roadmap_id UUID NOT NULL,
    module_id VARCHAR(100) NOT NULL,
    status VARCHAR(20) DEFAULT 'not_started', -- not_started, in_progress, completed
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    time_spent_seconds INTEGER DEFAULT 0,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (roadmap_id) REFERENCES learning_roadmaps(id) ON DELETE CASCADE,
    UNIQUE (user_id, roadmap_id, module_id),
    INDEX idx_progress_user_roadmap (user_id, roadmap_id)
);

-- Agent execution logs
CREATE TABLE agent_executions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    execution_arn VARCHAR(500) NOT NULL, -- Step Functions execution ARN
    project_id UUID NOT NULL,
    user_id UUID NOT NULL,
    user_request TEXT NOT NULL,
    task_manifest JSONB,
    agent_results JSONB,
    status VARCHAR(20) DEFAULT 'running', -- running, succeeded, failed
    started_at TIMESTAMP DEFAULT NOW(),
    completed_at TIMESTAMP,
    duration_ms INTEGER,
    error_message TEXT,
    FOREIGN KEY (project_id) REFERENCES projects(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_execution_project (project_id),
    INDEX idx_execution_user (user_id),
    INDEX idx_execution_status (status)
);

-- Recruiter access logs (audit trail)
CREATE TABLE recruiter_access_logs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    recruiter_id UUID NOT NULL,
    skill_wallet_user_id UUID NOT NULL,
    accessed_at TIMESTAMP DEFAULT NOW(),
    access_type VARCHAR(50),
    ip_address INET,
    user_agent TEXT,
    FOREIGN KEY (recruiter_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (skill_wallet_user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_access_recruiter (recruiter_id),
    INDEX idx_access_wallet_user (skill_wallet_user_id),
    INDEX idx_access_time (accessed_at)
);

-- Automated cleanup job (pg_cron)
SELECT cron.schedule(
    'cleanup-expired-context',
    '*/10 * * * *', -- Every 10 minutes
    $$
    DELETE FROM context_memory 
    WHERE expires_at IS NOT NULL 
      AND expires_at < NOW()
    $$
);
```

### 12.2 ElastiCache Redis Schema

```redis
# Session tokens (TTL: 1 hour)
SET session:{sessionId} '{"userId": "...", "projectId": "...", "role": "..."}' EX 3600

# Cursor positions (TTL: 5 minutes)
HSET project:{projectId}:cursors user:{userId} '{"file": "app.js", "line": 42, "col": 15}'
EXPIRE project:{projectId}:cursors 300

# Draft changes (TTL: 24 hours)
HSET project:{projectId}:drafts user:{userId}:file:{fileId} '{...}' 
EXPIRE project:{projectId}:drafts 86400

# Active collaborators (TTL: 1 hour)
SADD project:{projectId}:active_users user:{userId}
EXPIRE project:{projectId}:active_users 3600

# Rate limiting (token bucket)
SET ratelimit:{userId}:{endpoint} {tokens_remaining} EX 60

# Agent execution cache (TTL: 5 minutes)
SET agent:execution:{executionId}:status 'running' EX 300
HSET agent:execution:{executionId}:results architect '{...}' logic '{...}' ui '{...}'
```

### 12.3 S3 Bucket Structure

```
velocity-user-code/
├── projects/
│   └── {projectId}/
│       ├── .git/
│       ├── src/
│       ├── package.json
│       └── ...

velocity-diffs/
└── {projectId}/
    └── {userId}/
        └── {timestamp}.diff

velocity-playbacks/
└── {userId}/
    └── {skillHashId}/
        ├── video.mp4
        └── thumbnail.jpg

velocity-context-archive/
└── {projectId}/
    └── {date}/
        └── context-{timestamp}.json

velocity-backups/
└── aurora/
    └── {date}/
        └── snapshot-{timestamp}.sql
```

### 12.4 Data Retention Policies

```yaml
DataRetention:
  ContextMemory:
    HotData: 1 hour (Aurora)
    WarmData: 7 days (Aurora)
    ColdData: 90 days (S3)
    Deletion: After 90 days
  
  SaveGateRequests:
    Active: 30 days (Aurora)
    Archived: 1 year (S3)
    Deletion: After 1 year
  
  SkillHashes:
    Active: Indefinite (user-controlled)
    Expired: 2 years after expiration
  
  AgentExecutions:
    Recent: 30 days (Aurora)
    Historical: 1 year (S3 + Athena)
    Deletion: After 1 year
  
  UserCode:
    Active: Indefinite (S3 Standard)
    Inactive: 90 days (S3-IA)
    Archived: 1 year (Glacier)
  
  AuditLogs:
    Recent: 90 days (CloudWatch)
    Historical: 7 years (S3 + Glacier)
```


## 13. Deployment Architecture

### 13.1 Infrastructure as Code (CloudFormation)

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Velocity Platform - Multi-Agent Development Ecosystem'

Parameters:
  Environment:
    Type: String
    Default: prod
    AllowedValues: [dev, staging, prod]
  
  DomainName:
    Type: String
    Default: velocity.dev
  
  CertificateArn:
    Type: String
    Description: ACM certificate ARN for HTTPS

Resources:
  # VPC and Networking
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Sub velocity-vpc-${Environment}
  
  PublicSubnetA:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: !Select [0, !GetAZs '']
      MapPublicIpOnLaunch: true
  
  PublicSubnetB:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.2.0/24
      AvailabilityZone: !Select [1, !GetAZs '']
      MapPublicIpOnLaunch: true
  
  PrivateSubnetA:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.10.0/24
      AvailabilityZone: !Select [0, !GetAZs '']
  
  PrivateSubnetB:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.11.0/24
      AvailabilityZone: !Select [1, !GetAZs '']
  
  # Aurora PostgreSQL Cluster
  AuroraCluster:
    Type: AWS::RDS::DBCluster
    Properties:
      Engine: aurora-postgresql
      EngineVersion: '15.4'
      DatabaseName: velocity
      MasterUsername: !Sub '{{resolve:secretsmanager:velocity/aurora/master:SecretString:username}}'
      MasterUserPassword: !Sub '{{resolve:secretsmanager:velocity/aurora/master:SecretString:password}}'
      DBSubnetGroupName: !Ref DBSubnetGroup
      VpcSecurityGroupIds:
        - !Ref AuroraSecurityGroup
      BackupRetentionPeriod: 30
      PreferredBackupWindow: '03:00-04:00'
      PreferredMaintenanceWindow: 'sun:04:00-sun:05:00'
      EnableHttpEndpoint: true
      StorageEncrypted: true
      KmsKeyId: !Ref DatabaseKMSKey
      ServerlessV2ScalingConfiguration:
        MinCapacity: 2
        MaxCapacity: 64
  
  AuroraInstance1:
    Type: AWS::RDS::DBInstance
    Properties:
      DBClusterIdentifier: !Ref AuroraCluster
      DBInstanceClass: db.serverless
      Engine: aurora-postgresql
      PubliclyAccessible: false
  
  # ElastiCache Redis Cluster
  RedisSubnetGroup:
    Type: AWS::ElastiCache::SubnetGroup
    Properties:
      Description: Subnet group for Velocity Redis
      SubnetIds:
        - !Ref PrivateSubnetA
        - !Ref PrivateSubnetB
  
  RedisCluster:
    Type: AWS::ElastiCache::ReplicationGroup
    Properties:
      ReplicationGroupDescription: Velocity session and cache store
      Engine: redis
      EngineVersion: '7.0'
      CacheNodeType: cache.r6g.large
      NumCacheClusters: 2
      AutomaticFailoverEnabled: true
      MultiAZEnabled: true
      CacheSubnetGroupName: !Ref RedisSubnetGroup
      SecurityGroupIds:
        - !Ref RedisSecurityGroup
      AtRestEncryptionEnabled: true
      TransitEncryptionEnabled: true
  
  # Cognito User Pool
  UserPool:
    Type: AWS::Cognito::UserPool
    Properties:
      UserPoolName: !Sub velocity-users-${Environment}
      AutoVerifiedAttributes:
        - email
      MfaConfiguration: OPTIONAL
      EnabledMfas:
        - SOFTWARE_TOKEN_MFA
      Schema:
        - Name: email
          Required: true
          Mutable: false
      Policies:
        PasswordPolicy:
          MinimumLength: 12
          RequireUppercase: true
          RequireLowercase: true
          RequireNumbers: true
          RequireSymbols: true
  
  UserPoolClient:
    Type: AWS::Cognito::UserPoolClient
    Properties:
      ClientName: velocity-web-client
      UserPoolId: !Ref UserPool
      GenerateSecret: false
      ExplicitAuthFlows:
        - ALLOW_USER_SRP_AUTH
        - ALLOW_REFRESH_TOKEN_AUTH
      AccessTokenValidity: 1
      IdTokenValidity: 1
      RefreshTokenValidity: 30
      TokenValidityUnits:
        AccessToken: hours
        IdToken: hours
        RefreshToken: days
  
  # API Gateway
  RestApi:
    Type: AWS::ApiGateway::RestApi
    Properties:
      Name: !Sub velocity-api-${Environment}
      Description: Velocity REST API
      EndpointConfiguration:
        Types:
          - REGIONAL
  
  # AppSync GraphQL API
  GraphQLApi:
    Type: AWS::AppSync::GraphQLApi
    Properties:
      Name: !Sub velocity-graphql-${Environment}
      AuthenticationType: AMAZON_COGNITO_USER_POOLS
      UserPoolConfig:
        UserPoolId: !Ref UserPool
        AwsRegion: !Ref AWS::Region
        DefaultAction: ALLOW
      LogConfig:
        CloudWatchLogsRoleArn: !GetAtt AppSyncLogsRole.Arn
        FieldLogLevel: ERROR
      XrayEnabled: true
  
  # Step Functions State Machine
  MultiAgentOrchestrator:
    Type: AWS::StepFunctions::StateMachine
    Properties:
      StateMachineName: !Sub velocity-orchestrator-${Environment}
      RoleArn: !GetAtt StepFunctionsRole.Arn
      DefinitionString: !Sub |
        {
          "Comment": "Velocity Multi-Agent Orchestration",
          "StartAt": "ArchitectAnalysis",
          "States": {
            "ArchitectAnalysis": {
              "Type": "Task",
              "Resource": "arn:aws:states:::bedrock:invokeModel",
              "Parameters": {
                "ModelId": "anthropic.claude-3-5-sonnet-20241022-v2:0",
                "Body": {
                  "anthropic_version": "bedrock-2023-05-31",
                  "max_tokens": 4096,
                  "messages.$": "$.messages"
                }
              },
              "Next": "ParallelExecution"
            },
            "ParallelExecution": {
              "Type": "Parallel",
              "Branches": [
                {
                  "StartAt": "LogicAgent",
                  "States": {
                    "LogicAgent": {
                      "Type": "Task",
                      "Resource": "arn:aws:states:::bedrock:invokeModel",
                      "End": true
                    }
                  }
                },
                {
                  "StartAt": "UIAgent",
                  "States": {
                    "UIAgent": {
                      "Type": "Task",
                      "Resource": "arn:aws:states:::bedrock:invokeModel",
                      "End": true
                    }
                  }
                }
              ],
              "End": true
            }
          }
        }
  
  # Lambda Functions
  ContextJanitorFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: !Sub velocity-context-janitor-${Environment}
      Runtime: python3.11
      Handler: index.lambda_handler
      Code:
        S3Bucket: !Ref LambdaCodeBucket
        S3Key: context-janitor.zip
      Role: !GetAtt LambdaExecutionRole.Arn
      MemorySize: 1024
      Timeout: 30
      ReservedConcurrentExecutions: 50
      Environment:
        Variables:
          AURORA_CLUSTER_ARN: !GetAtt AuroraCluster.DBClusterArn
          AURORA_SECRET_ARN: !Ref AuroraSecret
          BEDROCK_REGION: us-east-1
  
  # S3 Buckets
  UserCodeBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub velocity-user-code-${Environment}
      VersioningConfiguration:
        Status: Enabled
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: aws:kms
              KMSMasterKeyID: !Ref S3KMSKey
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
      LifecycleConfiguration:
        Rules:
          - Id: TransitionToIA
            Status: Enabled
            Transitions:
              - TransitionInDays: 30
                StorageClass: STANDARD_IA
              - TransitionInDays: 90
                StorageClass: GLACIER_IR
  
  # CloudFront Distribution
  CloudFrontDistribution:
    Type: AWS::CloudFront::Distribution
    Properties:
      DistributionConfig:
        Enabled: true
        Aliases:
          - !Ref DomainName
        ViewerCertificate:
          AcmCertificateArn: !Ref CertificateArn
          SslSupportMethod: sni-only
          MinimumProtocolVersion: TLSv1.2_2021
        Origins:
          - Id: ApiGatewayOrigin
            DomainName: !Sub '${RestApi}.execute-api.${AWS::Region}.amazonaws.com'
            CustomOriginConfig:
              HTTPSPort: 443
              OriginProtocolPolicy: https-only
        DefaultCacheBehavior:
          TargetOriginId: ApiGatewayOrigin
          ViewerProtocolPolicy: redirect-to-https
          AllowedMethods: [GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE]
          CachedMethods: [GET, HEAD, OPTIONS]
          ForwardedValues:
            QueryString: true
            Headers:
              - Authorization
              - Content-Type
        PriceClass: PriceClass_100

Outputs:
  VPCId:
    Value: !Ref VPC
    Export:
      Name: !Sub velocity-vpc-${Environment}
  
  AuroraClusterEndpoint:
    Value: !GetAtt AuroraCluster.Endpoint.Address
    Export:
      Name: !Sub velocity-aurora-endpoint-${Environment}
  
  RedisEndpoint:
    Value: !GetAtt RedisCluster.PrimaryEndPoint.Address
    Export:
      Name: !Sub velocity-redis-endpoint-${Environment}
  
  ApiGatewayUrl:
    Value: !Sub 'https://${RestApi}.execute-api.${AWS::Region}.amazonaws.com/prod'
    Export:
      Name: !Sub velocity-api-url-${Environment}
  
  GraphQLApiUrl:
    Value: !GetAtt GraphQLApi.GraphQLUrl
    Export:
      Name: !Sub velocity-graphql-url-${Environment}
```

### 13.2 CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy Velocity Platform

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  AWS_REGION: ap-south-1
  ENVIRONMENT: prod

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest pytest-cov
      
      - name: Run unit tests
        run: pytest tests/ --cov=src/ --cov-report=xml
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
  
  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Build Lambda packages
        run: |
          cd lambda/context-janitor
          pip install -r requirements.txt -t .
          zip -r ../../context-janitor.zip .
      
      - name: Upload artifacts
        uses: actions/upload-artifact@v3
        with:
          name: lambda-packages
          path: '*.zip'
  
  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Download artifacts
        uses: actions/download-artifact@v3
        with:
          name: lambda-packages
      
      - name: Upload Lambda code to S3
        run: |
          aws s3 cp context-janitor.zip s3://velocity-lambda-code/
      
      - name: Deploy CloudFormation stack
        run: |
          aws cloudformation deploy \
            --template-file infrastructure/cloudformation.yml \
            --stack-name velocity-platform-${ENVIRONMENT} \
            --parameter-overrides Environment=${ENVIRONMENT} \
            --capabilities CAPABILITY_IAM \
            --no-fail-on-empty-changeset
      
      - name: Update Lambda functions
        run: |
          aws lambda update-function-code \
            --function-name velocity-context-janitor-${ENVIRONMENT} \
            --s3-bucket velocity-lambda-code \
            --s3-key context-janitor.zip
      
      - name: Run smoke tests
        run: |
          python scripts/smoke_tests.py --environment ${ENVIRONMENT}
```

### 13.3 Blue-Green Deployment Strategy

```python
# scripts/blue_green_deploy.py
import boto3
import time

def blue_green_deployment():
    """
    Perform blue-green deployment for zero-downtime updates
    """
    route53 = boto3.client('route53')
    elbv2 = boto3.client('elbv2')
    
    # 1. Deploy new version (green) alongside existing (blue)
    print("Deploying green environment...")
    deploy_green_environment()
    
    # 2. Run health checks on green
    print("Running health checks...")
    if not health_check_green():
        print("Health check failed, rolling back...")
        cleanup_green_environment()
        return False
    
    # 3. Gradually shift traffic (10% → 50% → 100%)
    print("Shifting traffic to green...")
    shift_traffic(green_weight=10, blue_weight=90)
    time.sleep(300)  # Monitor for 5 minutes
    
    shift_traffic(green_weight=50, blue_weight=50)
    time.sleep(300)
    
    shift_traffic(green_weight=100, blue_weight=0)
    
    # 4. Monitor for issues
    time.sleep(600)  # Monitor for 10 minutes
    
    # 5. Cleanup blue environment
    print("Cleaning up blue environment...")
    cleanup_blue_environment()
    
    print("Deployment successful!")
    return True

def shift_traffic(green_weight, blue_weight):
    """Update Route 53 weighted routing"""
    route53 = boto3.client('route53')
    
    route53.change_resource_record_sets(
        HostedZoneId='Z1234567890ABC',
        ChangeBatch={
            'Changes': [
                {
                    'Action': 'UPSERT',
                    'ResourceRecordSet': {
                        'Name': 'api.velocity.dev',
                        'Type': 'A',
                        'SetIdentifier': 'green',
                        'Weight': green_weight,
                        'AliasTarget': {
                            'HostedZoneId': 'Z1234567890DEF',
                            'DNSName': 'green-alb.velocity.dev',
                            'EvaluateTargetHealth': True
                        }
                    }
                },
                {
                    'Action': 'UPSERT',
                    'ResourceRecordSet': {
                        'Name': 'api.velocity.dev',
                        'Type': 'A',
                        'SetIdentifier': 'blue',
                        'Weight': blue_weight,
                        'AliasTarget': {
                            'HostedZoneId': 'Z1234567890DEF',
                            'DNSName': 'blue-alb.velocity.dev',
                            'EvaluateTargetHealth': True
                        }
                    }
                }
            ]
        }
    )
```


## 14. Monitoring & Logging Strategy

### 14.1 Observability Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    OBSERVABILITY LAYER                       │
├─────────────────────────────────────────────────────────────┤
│  Metrics          Logs              Traces        Alarms    │
│  (CloudWatch)     (CloudWatch)      (X-Ray)      (SNS)      │
└─────────────────────────────────────────────────────────────┘
                              ↑
                              │
┌─────────────────────────────────────────────────────────────┐
│                    APPLICATION LAYER                         │
│  Lambda │ Step Functions │ AppSync │ Aurora │ AppStream     │
└─────────────────────────────────────────────────────────────┘
```

### 14.2 CloudWatch Metrics

#### Custom Metrics

```python
import boto3
from datetime import datetime

cloudwatch = boto3.client('cloudwatch')

def publish_custom_metrics(metric_name, value, unit='Count', dimensions=None):
    """
    Publish custom metrics to CloudWatch
    """
    metric_data = {
        'MetricName': metric_name,
        'Value': value,
        'Unit': unit,
        'Timestamp': datetime.utcnow(),
        'Dimensions': dimensions or []
    }
    
    cloudwatch.put_metric_data(
        Namespace='Velocity/Platform',
        MetricData=[metric_data]
    )

# Usage examples
def track_agent_execution(agent_type, duration_ms, success):
    """Track AI agent performance"""
    publish_custom_metrics(
        metric_name='AgentExecutionDuration',
        value=duration_ms,
        unit='Milliseconds',
        dimensions=[
            {'Name': 'AgentType', 'Value': agent_type},
            {'Name': 'Success', 'Value': str(success)}
        ]
    )

def track_context_janitor_performance(items_processed, duration_ms):
    """Track Context Janitor efficiency"""
    publish_custom_metrics(
        metric_name='ContextItemsProcessed',
        value=items_processed,
        unit='Count'
    )
    publish_custom_metrics(
        metric_name='ContextJanitorDuration',
        value=duration_ms,
        unit='Milliseconds'
    )

def track_save_gate_metrics(approval_time_seconds):
    """Track Save-Gate approval latency"""
    publish_custom_metrics(
        metric_name='SaveGateApprovalTime',
        value=approval_time_seconds,
        unit='Seconds'
    )
```

#### Key Metrics Dashboard

```yaml
CloudWatchDashboard:
  Name: velocity-platform-overview
  Widgets:
    - Type: Metric
      Title: "API Gateway Requests"
      Metrics:
        - [AWS/ApiGateway, Count, ApiName, velocity-api]
        - [., 4XXError, ., .]
        - [., 5XXError, ., .]
    
    - Type: Metric
      Title: "Lambda Invocations"
      Metrics:
        - [AWS/Lambda, Invocations, FunctionName, velocity-context-janitor]
        - [., Errors, ., .]
        - [., Duration, ., ., {stat: Average}]
    
    - Type: Metric
      Title: "Aurora Performance"
      Metrics:
        - [AWS/RDS, CPUUtilization, DBClusterIdentifier, velocity-aurora]
        - [., DatabaseConnections, ., .]
        - [., ReadLatency, ., ., {stat: Average}]
        - [., WriteLatency, ., ., {stat: Average}]
    
    - Type: Metric
      Title: "Step Functions Executions"
      Metrics:
        - [AWS/States, ExecutionsStarted, StateMachineArn, velocity-orchestrator]
        - [., ExecutionsSucceeded, ., .]
        - [., ExecutionsFailed, ., .]
        - [., ExecutionTime, ., ., {stat: Average}]
    
    - Type: Metric
      Title: "Bedrock Inference"
      Metrics:
        - [AWS/Bedrock, Invocations, ModelId, anthropic.claude-3-5-sonnet]
        - [., InvocationLatency, ., ., {stat: p95}]
        - [., InvocationClientErrors, ., .]
        - [., InvocationServerErrors, ., .]
    
    - Type: Metric
      Title: "AppStream Sessions"
      Metrics:
        - [AWS/AppStream, ActualCapacity, Fleet, velocity-web-ide]
        - [., InUseCapacity, ., .]
        - [., AvailableCapacity, ., .]
    
    - Type: Metric
      Title: "Custom: Agent Performance"
      Metrics:
        - [Velocity/Platform, AgentExecutionDuration, AgentType, architect, {stat: Average}]
        - [., ., ., logic, {stat: Average}]
        - [., ., ., ui, {stat: Average}]
```

### 14.3 Structured Logging

```python
import json
import logging
from datetime import datetime
from aws_lambda_powertools import Logger

# Use AWS Lambda Powertools for structured logging
logger = Logger(service="velocity-platform")

def log_agent_execution(execution_id, agent_type, status, duration_ms, error=None):
    """
    Structured logging for agent executions
    """
    log_entry = {
        'timestamp': datetime.utcnow().isoformat(),
        'execution_id': execution_id,
        'agent_type': agent_type,
        'status': status,
        'duration_ms': duration_ms,
        'error': error
    }
    
    if status == 'success':
        logger.info('Agent execution completed', extra=log_entry)
    else:
        logger.error('Agent execution failed', extra=log_entry)

def log_context_janitor_run(project_id, items_processed, conflicts_detected):
    """
    Log Context Janitor operations
    """
    logger.info(
        'Context Janitor completed',
        extra={
            'project_id': project_id,
            'items_processed': items_processed,
            'conflicts_detected': conflicts_detected,
            'timestamp': datetime.utcnow().isoformat()
        }
    )

def log_save_gate_event(event_type, request_id, user_id, project_id):
    """
    Log Save-Gate events for audit trail
    """
    logger.info(
        f'Save-Gate: {event_type}',
        extra={
            'event_type': event_type,
            'request_id': request_id,
            'user_id': user_id,
            'project_id': project_id,
            'timestamp': datetime.utcnow().isoformat()
        }
    )
```

### 14.4 Distributed Tracing (X-Ray)

```python
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.core import patch_all

# Patch all supported libraries
patch_all()

@xray_recorder.capture('invoke_multi_agent_orchestrator')
def invoke_orchestrator(user_request, project_id):
    """
    Trace multi-agent orchestration flow
    """
    # Add metadata to trace
    xray_recorder.put_metadata('user_request', user_request)
    xray_recorder.put_metadata('project_id', project_id)
    
    # Start subsegment for Architect Agent
    with xray_recorder.capture('architect_agent'):
        architect_result = invoke_architect_agent(user_request)
        xray_recorder.put_annotation('architect_tokens', architect_result['tokens'])
    
    # Parallel subsegments for Logic and UI agents
    with xray_recorder.capture('parallel_agents'):
        logic_result = invoke_logic_agent(architect_result)
        ui_result = invoke_ui_agent(architect_result)
    
    # Context Janitor subsegment
    with xray_recorder.capture('context_janitor'):
        janitor_result = run_context_janitor(project_id)
        xray_recorder.put_annotation('conflicts_detected', janitor_result['conflicts'])
    
    return merge_results(logic_result, ui_result)
```

### 14.5 Alerting Strategy

```yaml
CloudWatchAlarms:
  # Critical Alarms (Page on-call)
  - AlarmName: HighAPIErrorRate
    MetricName: 5XXError
    Namespace: AWS/ApiGateway
    Threshold: 10
    EvaluationPeriods: 2
    Period: 60
    Statistic: Sum
    ComparisonOperator: GreaterThanThreshold
    AlarmActions:
      - !Ref PagerDutySNSTopic
  
  - AlarmName: AuroraHighCPU
    MetricName: CPUUtilization
    Namespace: AWS/RDS
    Threshold: 90
    EvaluationPeriods: 3
    Period: 300
    Statistic: Average
    ComparisonOperator: GreaterThanThreshold
    AlarmActions:
      - !Ref PagerDutySNSTopic
  
  - AlarmName: LambdaHighErrorRate
    MetricName: Errors
    Namespace: AWS/Lambda
    Threshold: 5
    EvaluationPeriods: 2
    Period: 60
    Statistic: Sum
    ComparisonOperator: GreaterThanThreshold
    AlarmActions:
      - !Ref PagerDutySNSTopic
  
  # Warning Alarms (Slack notification)
  - AlarmName: HighAgentLatency
    MetricName: AgentExecutionDuration
    Namespace: Velocity/Platform
    Threshold: 5000  # 5 seconds
    EvaluationPeriods: 3
    Period: 300
    Statistic: Average
    ComparisonOperator: GreaterThanThreshold
    AlarmActions:
      - !Ref SlackSNSTopic
  
  - AlarmName: AppStreamLowCapacity
    MetricName: AvailableCapacity
    Namespace: AWS/AppStream
    Threshold: 5
    EvaluationPeriods: 2
    Period: 60
    Statistic: Average
    ComparisonOperator: LessThanThreshold
    AlarmActions:
      - !Ref SlackSNSTopic
  
  - AlarmName: HighSaveGateApprovalTime
    MetricName: SaveGateApprovalTime
    Namespace: Velocity/Platform
    Threshold: 300  # 5 minutes
    EvaluationPeriods: 2
    Period: 300
    Statistic: Average
    ComparisonOperator: GreaterThanThreshold
    AlarmActions:
      - !Ref SlackSNSTopic
```

### 14.6 Log Aggregation and Analysis

```python
# CloudWatch Insights Queries

# Query 1: Find slow agent executions
SLOW_AGENTS_QUERY = """
fields @timestamp, execution_id, agent_type, duration_ms
| filter agent_type in ['architect', 'logic', 'ui']
| filter duration_ms > 3000
| sort duration_ms desc
| limit 100
"""

# Query 2: Analyze Context Janitor performance
CONTEXT_JANITOR_QUERY = """
fields @timestamp, project_id, items_processed, conflicts_detected
| filter @message like /Context Janitor completed/
| stats avg(items_processed), max(items_processed), count() by bin(5m)
"""

# Query 3: Save-Gate approval patterns
SAVE_GATE_QUERY = """
fields @timestamp, event_type, request_id, user_id
| filter event_type in ['request', 'approve', 'reject']
| stats count() by event_type, bin(1h)
"""

# Query 4: Error analysis
ERROR_ANALYSIS_QUERY = """
fields @timestamp, @message, error.type, error.message
| filter @message like /ERROR/
| stats count() by error.type
| sort count desc
"""

def run_insights_query(query, start_time, end_time):
    """
    Execute CloudWatch Insights query
    """
    logs = boto3.client('logs')
    
    response = logs.start_query(
        logGroupName='/aws/lambda/velocity-*',
        startTime=int(start_time.timestamp()),
        endTime=int(end_time.timestamp()),
        queryString=query
    )
    
    query_id = response['queryId']
    
    # Poll for results
    while True:
        result = logs.get_query_results(queryId=query_id)
        if result['status'] == 'Complete':
            return result['results']
        time.sleep(1)
```

### 14.7 Performance Monitoring

```python
class PerformanceMonitor:
    """
    Track and report performance metrics
    """
    def __init__(self):
        self.cloudwatch = boto3.client('cloudwatch')
    
    def track_bedrock_latency(self, model_id, latency_ms, tokens):
        """Track Bedrock inference performance"""
        self.cloudwatch.put_metric_data(
            Namespace='Velocity/Bedrock',
            MetricData=[
                {
                    'MetricName': 'InferenceLatency',
                    'Value': latency_ms,
                    'Unit': 'Milliseconds',
                    'Dimensions': [
                        {'Name': 'ModelId', 'Value': model_id}
                    ]
                },
                {
                    'MetricName': 'TokensGenerated',
                    'Value': tokens,
                    'Unit': 'Count',
                    'Dimensions': [
                        {'Name': 'ModelId', 'Value': model_id}
                    ]
                }
            ]
        )
    
    def track_aurora_query_performance(self, query_type, duration_ms):
        """Track database query performance"""
        self.cloudwatch.put_metric_data(
            Namespace='Velocity/Database',
            MetricData=[{
                'MetricName': 'QueryDuration',
                'Value': duration_ms,
                'Unit': 'Milliseconds',
                'Dimensions': [
                    {'Name': 'QueryType', 'Value': query_type}
                ]
            }]
        )
    
    def track_user_experience(self, action, duration_ms, success):
        """Track end-user experience metrics"""
        self.cloudwatch.put_metric_data(
            Namespace='Velocity/UserExperience',
            MetricData=[{
                'MetricName': 'ActionDuration',
                'Value': duration_ms,
                'Unit': 'Milliseconds',
                'Dimensions': [
                    {'Name': 'Action', 'Value': action},
                    {'Name': 'Success', 'Value': str(success)}
                ]
            }]
        )
```


## 15. Cost Optimization Strategy

### 15.1 Cost Breakdown Estimate (Monthly)

```
Service                    Usage                           Cost (USD)
─────────────────────────────────────────────────────────────────────
Amazon Bedrock
  - Claude 3.5 Sonnet      1M input tokens/day            $3,000
  - Gemini 1.5 Pro         500K input tokens/day          $1,750
  - GPT-4o                 200K input tokens/day          $1,200
  - Titan Embeddings       10M tokens/day                 $100
                                                  Subtotal: $6,050

AWS AppStream 2.0
  - 100 concurrent users   stream.standard.medium         $7,200
  - Storage (100GB/user)   10TB total                     $230
                                                  Subtotal: $7,430

Aurora PostgreSQL
  - Serverless v2          16 ACU average                 $2,880
  - Storage                500GB                          $115
  - I/O                    10M requests                   $200
  - Backup storage         1TB                            $95
                                                  Subtotal: $3,290

ElastiCache Redis
  - cache.r6g.large        2 nodes                        $438
                                                  Subtotal: $438

Lambda
  - Context Janitor        10M invocations                $20
  - Other functions        5M invocations                 $10
  - Duration               50M GB-seconds                 $833
                                                  Subtotal: $863

Step Functions
  - State transitions      1M transitions                 $25
                                                  Subtotal: $25

S3
  - Standard storage       10TB                           $230
  - Requests               100M PUT/GET                   $50
  - Data transfer          5TB egress                     $450
                                                  Subtotal: $730

API Gateway
  - REST API calls         100M requests                  $350
                                                  Subtotal: $350

AppSync
  - GraphQL operations     50M operations                 $200
  - Real-time updates      10M minutes                    $80
                                                  Subtotal: $280

CloudFront
  - Data transfer          10TB                           $850
  - Requests               500M                           $50
                                                  Subtotal: $900

Other Services
  - Cognito                100K MAU                       $275
  - Secrets Manager        50 secrets                     $20
  - KMS                    100K requests                  $3
  - CloudWatch             100GB logs                     $50
  - X-Ray                  10M traces                     $50
                                                  Subtotal: $398

─────────────────────────────────────────────────────────────────────
TOTAL MONTHLY COST (1000 active users):                   $20,754
Cost per active user:                                     $20.75
```

### 15.2 Cost Optimization Techniques

#### 1. Bedrock Model Selection

```python
class CostOptimizedModelSelector:
    """
    Select most cost-effective model based on task complexity
    """
    MODEL_COSTS = {
        'claude-3-5-sonnet': {'input': 0.003, 'output': 0.015},  # per 1K tokens
        'claude-3-haiku': {'input': 0.00025, 'output': 0.00125},
        'gpt-4o': {'input': 0.005, 'output': 0.015},
        'gpt-4o-mini': {'input': 0.00015, 'output': 0.0006},
        'gemini-1.5-pro': {'input': 0.00125, 'output': 0.005},
        'gemini-1.5-flash': {'input': 0.000075, 'output': 0.0003}
    }
    
    def select_model(self, task_type, complexity, budget_constraint=None):
        """
        Select optimal model based on task requirements and cost
        """
        if task_type == 'architecture' and complexity == 'high':
            # Use premium model for critical planning
            return 'claude-3-5-sonnet'
        
        elif task_type == 'logic' and complexity == 'medium':
            # Balance cost and quality
            return 'claude-3-haiku'  # 10x cheaper than Sonnet
        
        elif task_type == 'ui' and complexity == 'low':
            # Use cheapest model for simple UI
            return 'gemini-1.5-flash'  # 40x cheaper than Pro
        
        elif budget_constraint == 'strict':
            # Always use cheapest capable model
            return 'gpt-4o-mini'
        
        return 'claude-3-5-sonnet'  # Default to quality
    
    def estimate_cost(self, model_id, input_tokens, output_tokens):
        """Calculate estimated cost for inference"""
        costs = self.MODEL_COSTS[model_id]
        input_cost = (input_tokens / 1000) * costs['input']
        output_cost = (output_tokens / 1000) * costs['output']
        return input_cost + output_cost
```

#### 2. AppStream Fleet Optimization

```yaml
AppStreamOptimization:
  # Use scheduled scaling for predictable patterns
  ScheduledScaling:
    - Name: BusinessHours
      Schedule: "0 8 * * MON-FRI"  # 8 AM weekdays
      MinCapacity: 50
      MaxCapacity: 200
    
    - Name: OffHours
      Schedule: "0 22 * * *"  # 10 PM daily
      MinCapacity: 10
      MaxCapacity: 50
  
  # Use smaller instance types for light users
  FleetTiers:
    - Name: velocity-web-basic
      InstanceType: stream.standard.small  # 2 vCPU, 4 GB
      UserSegment: free_tier
      CostSavings: 50%
    
    - Name: velocity-web-standard
      InstanceType: stream.standard.medium  # 4 vCPU, 8 GB
      UserSegment: pro_tier
    
    - Name: velocity-web-premium
      InstanceType: stream.standard.large  # 8 vCPU, 16 GB
      UserSegment: enterprise_tier
  
  # Implement session timeout to free resources
  SessionConfig:
    MaxSessionDuration: 8 hours
    IdleDisconnectTimeout: 30 minutes
    DisconnectTimeout: 15 minutes
```

#### 3. Aurora Serverless v2 Optimization

```python
def optimize_aurora_capacity():
    """
    Dynamically adjust Aurora capacity based on load
    """
    rds = boto3.client('rds')
    cloudwatch = boto3.client('cloudwatch')
    
    # Get current CPU utilization
    cpu_metrics = cloudwatch.get_metric_statistics(
        Namespace='AWS/RDS',
        MetricName='CPUUtilization',
        Dimensions=[
            {'Name': 'DBClusterIdentifier', 'Value': 'velocity-aurora'}
        ],
        StartTime=datetime.utcnow() - timedelta(minutes=5),
        EndTime=datetime.utcnow(),
        Period=300,
        Statistics=['Average']
    )
    
    avg_cpu = cpu_metrics['Datapoints'][0]['Average']
    
    # Adjust capacity based on utilization
    if avg_cpu < 30:
        # Scale down to minimum
        target_capacity = 2
    elif avg_cpu > 70:
        # Scale up
        target_capacity = 32
    else:
        # Maintain current
        return
    
    rds.modify_db_cluster(
        DBClusterIdentifier='velocity-aurora',
        ServerlessV2ScalingConfiguration={
            'MinCapacity': target_capacity,
            'MaxCapacity': 64
        }
    )
```

#### 4. S3 Intelligent-Tiering

```yaml
S3LifecyclePolicy:
  Rules:
    - Id: IntelligentTiering
      Status: Enabled
      Transitions:
        # Automatically move to IA after 30 days
        - Days: 30
          StorageClass: INTELLIGENT_TIERING
      
      # Archive old versions
      NoncurrentVersionTransitions:
        - NoncurrentDays: 90
          StorageClass: GLACIER_IR
      
      # Delete very old versions
      NoncurrentVersionExpiration:
        NoncurrentDays: 365
    
    # Separate policy for playback videos (high access)
    - Id: PlaybackOptimization
      Status: Enabled
      Filter:
        Prefix: playbacks/
      Transitions:
        - Days: 90
          StorageClass: STANDARD_IA  # Keep accessible longer
```

#### 5. Lambda Cost Optimization

```python
# Use Lambda Powertools for efficient cold starts
from aws_lambda_powertools import Logger, Tracer, Metrics
from aws_lambda_powertools.utilities.typing import LambdaContext

logger = Logger()
tracer = Tracer()
metrics = Metrics()

# Optimize memory allocation (cost = memory × duration)
OPTIMAL_MEMORY_CONFIGS = {
    'context-janitor': 1024,  # CPU-bound, benefits from more memory
    'save-gate-handler': 512,  # I/O-bound, less memory needed
    'skill-wallet-processor': 2048  # Memory-intensive encryption
}

# Use provisioned concurrency only for critical functions
PROVISIONED_CONCURRENCY = {
    'context-janitor': 10,  # Always warm
    'save-gate-handler': 5,
    # Other functions use on-demand (cheaper for sporadic use)
}

# Implement caching to reduce invocations
from functools import lru_cache

@lru_cache(maxsize=1000)
def get_user_permissions(user_id):
    """Cache user permissions to avoid repeated DB queries"""
    return query_aurora(f"SELECT * FROM permissions WHERE user_id = '{user_id}'")
```

#### 6. Data Transfer Optimization

```python
class DataTransferOptimizer:
    """
    Minimize cross-region and internet data transfer costs
    """
    def __init__(self):
        self.s3 = boto3.client('s3')
        self.cloudfront = boto3.client('cloudfront')
    
    def use_cloudfront_for_static_assets(self, bucket, key):
        """
        Serve via CloudFront instead of direct S3 (cheaper egress)
        """
        # CloudFront: $0.085/GB
        # S3 direct: $0.09/GB
        # Savings: 5% + caching benefits
        return f"https://cdn.velocity.dev/{key}"
    
    def use_s3_transfer_acceleration(self, large_file):
        """
        Use S3 Transfer Acceleration for large uploads
        Only when file > 1GB (cost-effective threshold)
        """
        if large_file['size'] > 1_000_000_000:  # 1GB
            return self.s3.upload_file(
                Filename=large_file['path'],
                Bucket='velocity-user-code',
                Key=large_file['key'],
                Config=boto3.s3.transfer.TransferConfig(
                    use_threads=True,
                    max_concurrency=10
                )
            )
    
    def compress_before_transfer(self, data):
        """
        Compress data before S3 upload to reduce transfer costs
        """
        import gzip
        compressed = gzip.compress(data.encode())
        # Typical compression: 70% reduction
        # Cost savings: 70% of transfer cost
        return compressed
```

### 15.3 Reserved Capacity and Savings Plans

```yaml
CostSavingsCommitments:
  # Compute Savings Plan (1-year commitment)
  ComputeSavingsPlan:
    Commitment: $5,000/month
    Discount: 17%
    Applies: Lambda, AppStream
    AnnualSavings: $10,200
  
  # Aurora Reserved Instances (1-year)
  AuroraReservedCapacity:
    Commitment: 16 ACU
    Discount: 35%
    AnnualSavings: $12,096
  
  # ElastiCache Reserved Nodes (1-year)
  ElastiCacheReserved:
    NodeType: cache.r6g.large
    Quantity: 2
    Discount: 30%
    AnnualSavings: $1,577

TotalAnnualSavings: $23,873 (19% reduction)
```

### 15.4 Cost Monitoring and Alerts

```python
def setup_cost_alerts():
    """
    Configure AWS Budgets for cost monitoring
    """
    budgets = boto3.client('budgets')
    
    # Monthly budget alert
    budgets.create_budget(
        AccountId='123456789012',
        Budget={
            'BudgetName': 'velocity-monthly-budget',
            'BudgetLimit': {
                'Amount': '25000',
                'Unit': 'USD'
            },
            'TimeUnit': 'MONTHLY',
            'BudgetType': 'COST'
        },
        NotificationsWithSubscribers=[
            {
                'Notification': {
                    'NotificationType': 'ACTUAL',
                    'ComparisonOperator': 'GREATER_THAN',
                    'Threshold': 80,  # Alert at 80% of budget
                    'ThresholdType': 'PERCENTAGE'
                },
                'Subscribers': [
                    {
                        'SubscriptionType': 'EMAIL',
                        'Address': 'ops@velocity.dev'
                    }
                ]
            }
        ]
    )
    
    # Per-service budget tracking
    services = ['Bedrock', 'AppStream', 'Aurora', 'Lambda', 'S3']
    for service in services:
        budgets.create_budget(
            AccountId='123456789012',
            Budget={
                'BudgetName': f'velocity-{service.lower()}-budget',
                'BudgetLimit': {
                    'Amount': str(get_service_budget(service)),
                    'Unit': 'USD'
                },
                'TimeUnit': 'MONTHLY',
                'BudgetType': 'COST',
                'CostFilters': {
                    'Service': [f'Amazon {service}']
                }
            }
        )
```

### 15.5 Cost Optimization Recommendations

```yaml
ImmediateActions:
  - Action: Implement Bedrock model tiering
    Impact: 30% reduction in AI costs
    Savings: $1,815/month
  
  - Action: Use AppStream scheduled scaling
    Impact: 25% reduction in streaming costs
    Savings: $1,858/month
  
  - Action: Enable S3 Intelligent-Tiering
    Impact: 40% reduction in storage costs
    Savings: $92/month
  
  - Action: Optimize Lambda memory allocation
    Impact: 20% reduction in compute costs
    Savings: $173/month

MediumTermActions:
  - Action: Purchase 1-year Savings Plans
    Impact: 19% overall reduction
    Savings: $3,943/month
  
  - Action: Implement aggressive caching
    Impact: 15% reduction in database costs
    Savings: $494/month
  
  - Action: Compress data transfers
    Impact: 50% reduction in transfer costs
    Savings: $225/month

TotalPotentialSavings: $8,600/month (41% reduction)
OptimizedMonthlyCost: $12,154
OptimizedCostPerUser: $12.15
```

## Conclusion

The Velocity platform architecture is designed for:

1. **Scalability**: Horizontal scaling across all components using AWS managed services
2. **Reliability**: Multi-AZ deployment with cross-region failover (RTO: 1 hour, RPO: 1 second)
3. **Security**: Defense-in-depth with encryption, Nitro Enclaves, and comprehensive audit logging
4. **Performance**: Sub-3-second AI inference through cross-region optimization and caching
5. **Cost Efficiency**: Optimized resource utilization with potential 41% cost reduction through best practices

The architecture leverages AWS-native services to minimize operational overhead while maintaining enterprise-grade reliability and security. The modular design allows for independent scaling and evolution of components as the platform grows.

**Key Technical Achievements**:
- Model-agnostic AI orchestration supporting any Bedrock model
- Zero-knowledge behavioral profiling using Nitro Enclaves
- Real-time collaboration with conflict-free merge authority
- Thin-client delivery enabling development on any hardware
- Comprehensive observability with metrics, logs, and traces

This design is production-ready for the AWS AI for Bharat hackathon and can scale to support millions of developers globally.

