● ATLAS System - Complete Data Handling Summary

  System Architecture Overview

  ┌─────────────────────────────────────────────────────────────────────┐
  │                         ATLAS Data Flow                             │
  └─────────────────────────────────────────────────────────────────────┘

  User Browser
       │
       ├──► Frontend (Next.js) :3000
       │         │
       │         ├──► WebSocket (Real-time updates)
       │         │
       │         └──► HTTP API
       │                 │
       └──────────────►  Flask Backend :5002
                         │
                         ├──► Local Storage (Filesystem)
                         ├──► SQLite Databases
                         ├──► Redis (Session/Cache) :6379
                         │
                         └──► Proxy to AI-ML Backend :8000
                                  │
                                  ├──► Enqueue Job → Redis Queue :6379
                                  │                      │
                                  │                      ▼
                                  │                  RQ Worker(s)
                                  │                      │
                                  │                      ├──► Process PCAP
                                  │                      ├──► ML Analysis
                                  │                      └──► Publish Progress
                                  │
                                  └──► Callback Results → Update Workflow

  Service Ports & Components

  | Service       | Port | Process                      | Purpose                      |
  |---------------|------|------------------------------|------------------------------|
  | Frontend      | 3000 | Next.js (next-router-worker) | React UI, WebSocket client   |
  | Flask Backend | 5002 | Python/Quart                 | Main API, PCAP upload, auth  |
  | AI-ML Backend | 8000 | Python/FastAPI               | Workflow engine, ML services |
  | Redis         | 6379 | redis-server                 | Queue, cache, pub/sub        |
  | RQ Worker     | -    | python3 -m rq.cli            | Background PCAP processing   |

  Complete PCAP Data Flow

  1. Upload Phase

  User uploads PCAP → Flask Backend :5002/api/v1/pcap/upload
                     │
                     ├──► Validate file (size, type)
                     ├──► Save temporarily to /opt/atlas/data/uploads/
                     └──► Proxy to AI-ML Backend :8000/workflow/pcap

  2. Workflow Creation Phase

  AI-ML Backend :8000
      │
      ├──► Create workflow record in atlas_workflows.db
      ├──► Generate workflow_id (UUID)
      ├──► Move PCAP to /opt/atlas/data/pcaps/{workflow_id}.pcap
      └──► Call TaskQueueManager.enqueue_pcap_processing()

  3. Queue Enqueue Phase

  TaskQueueManager
      │
      ├──► Select priority queue (high/default/low)
      ├──► Create RQ job with workflow_id as job_id
      ├──► Enqueue to Redis :6379
      │    └──► Job parameters:
      │         ├── workflow_id
      │         ├── file_path
      │         ├── task_data (analysis types)
      │         ├── timeout: 30 minutes
      │         ├── result_ttl: 24 hours
      │         └── failure_ttl: 7 days
      │
      └──► Return immediately to API (non-blocking)

  4. Worker Processing Phase

  RQ Worker (PID 276180)
      │
      ├──► Poll Redis queues (high → default → low)
      ├──► Pick up job: process_pcap_task()
      │
      ├──► Initialize PcapProcessingAgent
      │
      ├──► Processing Steps:
      │    ├── Parse PCAP (scapy/dpkt)
      │    ├── Extract packet metadata
      │    ├── Perform TCP analysis
      │    ├── Detect protocols
      │    ├── Run ML anomaly detection
      │    └── Generate statistics
      │
      ├──► Progress Updates:
      │    └──► Publish to Redis pub/sub: workflow:{workflow_id}:progress
      │         └──► WebSocket broadcasts to UI clients
      │
      └──► Save Results:
           ├── Filesystem: /srv/atlas-Final/data/analysis_results/{workflow_id}/
           │   ├── metadata.json
           │   ├── tcp_analysis.json
           │   ├── protocol_analysis.json
           │   └── anomaly_detection.json
           │
           └── Database: Update atlas_workflows.db with completion status

  5. Notification Phase

  Worker → HTTP Callback → AI-ML Backend :8000/workflow/{workflow_id}/complete
                                │
                                ├──► Update workflow status
                                ├──► Publish completion event
                                └──► WebSocket broadcast to subscribed clients

  Data Storage Locations

  Filesystem Storage (/srv/atlas-Final/data/ & /opt/atlas/data/)

  /srv/atlas-Final/data/
  ├── agents/                    # Agent configuration
  ├── agents.json               # Agent registry (3.1KB)
  ├── analysis_results/         # PCAP analysis outputs (2.8GB)
  │   └── {workflow_id}/
  │       ├── metadata.json
  │       ├── tcp_analysis.json
  │       ├── protocol_analysis.json
  │       ├── anomaly_detection.json
  │       └── [additional analysis files]
  ├── captures/                 # Live capture metadata
  ├── captures.json             # Capture index
  ├── config/                   # System configuration
  ├── insights/                 # Generated insights
  ├── logs/                     # Application logs
  │   ├── audit/               # Audit trail
  │   ├── captures/            # Capture-specific logs
  │   └── frontend/            # Frontend logs
  ├── metadata/                # System metadata
  ├── metadata.json            # Metadata index
  ├── models/                  # ML models
  ├── pcaps/                   # Uploaded PCAP files
  ├── sessions/                # User sessions
  ├── streams/                 # Live stream data
  ├── tokens.json              # API tokens (2.3KB)
  ├── uploads/                 # Temporary upload staging
  └── users.json               # User registry

  Database Storage (SQLite)

  | Database           | Location                                               | Purpose              | Size    |
  |--------------------|--------------------------------------------------------|----------------------|---------|
  | atlas.db           | /srv/atlas-Final/                                      | Legacy main database | 0 bytes |
  | atlas_config.db    | Multiple locations                                     | System configuration | Small   |
  | atlas_auth.db      | /srv/atlas-Final/ai-ml_backend/agents/                 | User authentication  | Medium  |
  | atlas_workflows.db | /srv/atlas-Final/ai-ml_backend/agents/                 | Workflow tracking    | Medium  |
  | atlas_audit.db     | /srv/atlas-Final/main_backend/atlas-ml-service/agents/ | Audit logging        | Growing |
  | auth_manager.db    | /srv/atlas-Final/main_backend/                         | Auth sessions        | Small   |

  Redis Storage (Port 6379)

  | Key Pattern                     | Purpose                    | TTL          |
  |---------------------------------|----------------------------|--------------|
  | rq:queue:high                   | High-priority job queue    | Persistent   |
  | rq:queue:default                | Default job queue          | Persistent   |
  | rq:queue:low                    | Low-priority job queue     | Persistent   |
  | rq:job:{job_id}                 | Job metadata & results     | 24 hours     |
  | rq:failed                       | Failed job tracking        | 7 days       |
  | workflow:{workflow_id}:progress | Real-time progress updates | Session      |
  | session:{token}                 | User session data          | Token expiry |
  | cache:*                         | General application cache  | Varies       |

  Data Flow by Feature

  Agent PCAP Capture

  Remote Agent
      │
      └──► POST /api/agents/{agent_id}/capture
              │
              ├──► Authenticate agent (token validation)
              ├──► Save PCAP: /opt/atlas/data/captures/{agent_id}/{timestamp}.pcap
              ├──► Save metadata: *.meta.json
              ├──► Add to pcap_manager in-memory cache
              └──► Broadcast WebSocket event: capture.new

  User Authentication

  User Login
      │
      └──► POST /api-login
              │
              ├──► Check atlas_auth.db (password hash comparison)
              ├──► Generate JWT token (HS256, 60min expiry)
              ├──► Create session in Redis
              ├──► Log to atlas_audit.db (audit trail)
              └──► Return token + user info

  Live Capture Analysis

  Agent sends packets → WebSocket /ws/agent/{agent_id}
                            │
                            ├──► Stream to /opt/atlas/data/streams/{stream_id}
                            ├──► Real-time analysis (if enabled)
                            └──► Broadcast to UI: WebSocket /ws/captures/{capture_id}

  Queue Management

  RQ Worker Configuration

  Command: python3 -m rq.cli worker default high low --url redis://localhost:6379/0

  Queues (Priority Order):
  1. high    - Critical/priority analysis
  2. default - Normal analysis
  3. low     - Background tasks

  Worker Settings:
  - Job timeout: 30 minutes
  - Result TTL: 24 hours
  - Failed job TTL: 7 days
  - Burst mode: No (continuous)

  Queue Operations

  Enqueue Job:
  job = queue.enqueue(
      'services.task_queue.process_pcap_task',
      workflow_id,
      file_path,
      task_data,
      job_id=workflow_id,
      job_timeout='30m',
      result_ttl=86400,
      failure_ttl=604800
  )

  Job Status Tracking:
  - Queued → Processing → Completed/Failed
  - Progress published via Redis pub/sub
  - Status stored in atlas_workflows.db
  - Results stored in filesystem + database

  Data Persistence & Cleanup

  Retention Policies

  | Data Type         | Location                | Retention               | Cleanup Method        |
  |-------------------|-------------------------|-------------------------|-----------------------|
  | PCAP files        | /data/pcaps/            | Manual deletion         | User-triggered        |
  | Analysis results  | /data/analysis_results/ | Manual deletion         | User-triggered        |
  | RQ job data       | Redis                   | 24h success, 7d failure | Automatic (RQ)        |
  | Session data      | Redis                   | Token expiry (60min)    | Automatic (Redis TTL) |
  | Audit logs        | atlas_audit.db          | Indefinite              | Manual pruning        |
  | Workflow records  | atlas_workflows.db      | Indefinite              | Manual pruning        |
  | Temporary uploads | /data/uploads/          | Processing completion   | Automatic             |
  | Stream buffers    | /data/streams/          | Stream closure          | Automatic             |

  Backup Considerations

  Critical Data:
  - /srv/atlas-Final/data/analysis_results/ (2.8GB - growing)
  - atlas_auth.db (user accounts)
  - atlas_workflows.db (analysis history)
  - atlas_audit.db (compliance)
  - /data/pcaps/ (original captures)

  Transient Data:
  - Redis queues (can be rebuilt)
  - /data/uploads/ (temporary staging)
  - /data/sessions/ (active sessions)

  Performance Characteristics

  Current System State

  - Analysis Results: 2.8GB stored
  - Redis Memory: Active (process running)
  - Active Workers: 1 RQ worker process
  - Processing Capacity: ~2 concurrent PCAPs (30min timeout)

  Scaling Considerations

  Horizontal Scaling:
  - Add more RQ workers: python3 -m rq.cli worker ... (multiple processes)
  - Each worker can process jobs independently
  - Redis handles job distribution

  Vertical Scaling:
  - Increase worker timeout for larger PCAPs
  - Adjust Redis memory limits
  - Expand filesystem storage for analysis results

  Security & Access Control

  Data Protection Layers

  1. Authentication: JWT tokens (HS256, 60min expiry)
  2. Authorization: Role-based access (atlas_auth.db)
  3. Audit Trail: All operations logged (atlas_audit.db)
  4. Network: Services bound to localhost (Redis)
  5. File Permissions: Restricted access to data directories

  Encryption Status

  - In Transit: HTTPS/WSS (production)
  - At Rest: Filesystem-level (OS-dependent)
  - Passwords: Hashed (bcrypt in atlas_auth.db)
  - Tokens: Signed JWT (HS256)

  ---
  Summary

  YES, the Flask backend DOES use Redis with RQ workers for asynchronous PCAP processing. The complete data flow is:

  1. User uploads PCAP → Flask backend :5002
  2. Flask proxies → AI-ML backend :8000
  3. AI-ML enqueues job → Redis queue :6379
  4. RQ worker picks job → Processes PCAP
  5. Results saved → Filesystem + Database
  6. Progress updates → Redis pub/sub → WebSocket → UI

  Key Ports:
  - Frontend: 3000
  - Flask Backend: 5002
  - AI-ML Backend: 8000
  - Redis: 6379
  - No port 5003 (doesn't exist)

  Data Storage:
  - Filesystem: 2.8GB analysis results
  - Databases: SQLite (auth, workflows, audit)
  - Redis: Queues, cache, pub/sub
  - Total: ~3GB+ with PCAPs