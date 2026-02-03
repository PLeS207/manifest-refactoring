# RenderBox Jobber Logging -- DevOps Task
 
## Goal
 
Implement centralized logging for **RenderBox Jobber (Windows EC2)**
machines using **Amazon CloudWatch Logs**, enabling: - Live tail in
internal UI - Search by `gameId`, `period` - Filtering by `ERROR` /
`WARN` - No AWS credentials exposed to UI or jobber processes
 
------------------------------------------------------------------------
 
## Scope
 
-   Jobbers are **Windows EC2 instances**
-   C++ processes write logs to local files
-   Node.js services may also write logs
-   Solution must be production-ready and low-maintenance
-   On‑prem support is **out of scope** for now (2--3 years)
 
------------------------------------------------------------------------
 
## Chosen Architecture
 
### High‑level flow
 
    C++ / Node.js
       ↓ (log files on disk)
    Amazon CloudWatch Agent (Windows)
       ↓ HTTPS :443
    Amazon CloudWatch Logs
       ↓
    Backend API (Node.js)
       ↓
    Internal UI (live tail + search)
 
------------------------------------------------------------------------
 
## Network Requirements
 
### Outbound from Jobber EC2
 
-   **TCP 443 (HTTPS)**
-   Destination:
    -   `logs.<AWS_REGION>.amazonaws.com`
 
### Inbound to Jobber
 
-   **None required**
 
### Notes
 
-   If instance is in **private subnet**:
    -   Either NAT Gateway **OR**
    -   VPC Interface Endpoint:\
        `com.amazonaws.<region>.logs`
 
------------------------------------------------------------------------
 
## IAM Requirements
 
### EC2 Instance Role
 
Attach an IAM Role with permissions to write CloudWatch Logs.
 
Minimum required actions: - `logs:CreateLogGroup` -
`logs:CreateLogStream` - `logs:PutLogEvents` - `logs:DescribeLogStreams`
 
Optional: - `logs:PutRetentionPolicy`
 
Recommended: - Use AWS managed policy:\
**CloudWatchAgentServerPolicy**
 
------------------------------------------------------------------------
 
## CloudWatch Logs Structure
 
### Log Group
 
    /renderbox/jobbers
 
### Log Streams
 
    <instanceId>/cpp-main
<instanceId>/cpp-worker
<instanceId>/node-service
 
------------------------------------------------------------------------
 
## Log Format (Critical Requirement)
 
All application logs **must be JSON, one object per line**.
 
### Example
 
``` json
{
  "ts": "2026-01-16T18:12:03.123Z",
  "level": "warn",
  "gameId": "123456",
  "period": 2,
  "component": "cpp-worker",
  "message": "GPU load high"
}
```
 
### Required fields
 
-   `ts` -- ISO timestamp
-   `level` -- info \| warn \| error
-   `gameId`
-   `period`
-   `message`
 
This enables: - Efficient search - CloudWatch Logs Insights queries -
Clean UI filtering
 
------------------------------------------------------------------------
 
## CloudWatch Agent -- Responsibilities
 
-   Tail log files from configured folders
-   Handle log rotation and truncation
-   Push logs to CloudWatch Logs over HTTPS
-   Run as a **Windows Service**
 
------------------------------------------------------------------------
 
## Installation on Windows EC2
 
1.  Install **Amazon CloudWatch Agent for Windows**
 
2.  Create agent config file:
 
        C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json
 
3.  Configure only the `logs` section (metrics optional)
 
4.  Start agent as Windows service
 
------------------------------------------------------------------------
 
## Backend Responsibilities (Not DevOps-owned, but dependent)
 
-   Read logs from CloudWatch Logs using:
    -   Live Tail API (WebSocket)
    -   Logs Insights queries
-   Expose APIs:
    -   `/api/logs/live`
    -   `/api/logs/search`
-   Enforce RBAC and rate limits
-   UI **must never** access AWS APIs directly
 
------------------------------------------------------------------------
 
## What Is Explicitly Out of Scope
 
-   Custom log shipping agent
-   WebSocket delivery from jobbers
-   OpenSearch / Loki
-   On‑prem jobbers
-   AWS credentials in browser
 
------------------------------------------------------------------------
 
## Validation Checklist
 
-   [ ] CloudWatch Agent running on jobber
-   [ ] Logs appear in `/renderbox/jobbers`
-   [ ] Rotation handled correctly
-   [ ] Live tail works with \<10s delay
-   [ ] Search by `gameId` and `period` works
-   [ ] ERROR / WARN filtering works
-   [ ] No inbound ports opened on EC2
-   [ ] No AWS keys stored on disk manually
 
------------------------------------------------------------------------
 
## Future Considerations (Not Now)
 
-   On‑prem migration
-   Custom log shipper
-   Multi-destination log fan‑out
 
------------------------------------------------------------------------
 
---
 
## Environment-Specific Details
 
### Development Environment
 
#### Jobber
- **Private IP:** `172.31.70.4`
 
#### Backend Server
- **Public IP:** `18.217.134.180`
 
---
 
### Log Locations (Windows)
 
#### C++ Processes
- **Path:**  
  `C:\Injecto\Log`
- **Notes:**  
  All C++ processes write logs into the same directory, using different log file names.
 
#### Node.js (RenderBox)
- **Path:**  
  `C:\Injecto\RenderBox\logs`
 
---
 
### Production Environment
- Production configuration details (IPs, subnets, log paths if different)  
  **will be provided separately via email.**
 
 
## Deliverables
 
-   IAM role attached to jobbers
-   Network rules validated
-   CloudWatch Agent installed and running
-   Documentation updated
 
------------------------------------------------------------------------
 
**Owner:** DevOps\
**Consumers:** Backend, UI\
**Status:** Approved architecture
