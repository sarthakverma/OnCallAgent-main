🗂️ ARCHITECTURE MIND MAP
================================================

on_call_agent.py
│
├── 📦 CONFIGURATION (Lines 14-38)
│   ├── Service URLs
│   │   ├── AGENT_PORT (8088)
│   │   ├── LB_URL (Load Balancer - 9000)
│   │   └── PROM_URL (Prometheus - 9090)
│   │
│   ├── GitHub Integration
│   │   ├── GITHUB_TOKEN
│   │   ├── GITHUB_OWNER/REPO
│   │   └── GITHUB_REF (branch)
│   │
│   ├── AI Integration
│   │   └── OPENAI_API_KEY (GPT-4o)
│   │
│   ├── WhatsApp Notification (Twilio)
│   │   ├── TWILIO_ACCOUNT_SID
│   │   ├── TWILIO_AUTH_TOKEN
│   │   ├── TWILIO_WHATSAPP_NUMBER
│   │   └── USER_WHATSAPP_NUMBER
│   │
│   └── Jira Integration
│       ├── JIRA_URL
│       ├── JIRA_EMAIL
│       ├── JIRA_API_TOKEN
│       └── JIRA_PROJECT_KEY
│
├── 🧠 AI SYSTEM (Lines 45-169)
│   ├── SYSTEM_PROMPT
│   │   └── Instructs AI to diagnose incidents
│   │       ├── Parallel function calls (efficiency)
│   │       ├── Root cause analysis
│   │       └── Human-in-the-loop handling
│   │
│   └── FUNCTION_DEFINITIONS (5 Tools)
│       ├── fetch_logs (get error logs)
│       ├── compute_error_rate (Prometheus metrics)
│       ├── fetch_code_from_github (code analysis)
│       ├── send_diagnosis (final report)
│       └── create_jira_ticket (task creation)
│
├── 🔧 DATA FETCHING LAYER (Lines 172-242)
│   ├── fetch_recent_logs()
│   │   └── GET {LB_URL}/logs/tail
│   │
│   ├── prom_instant_query()
│   │   └── GET {PROM_URL}/api/v1/query
│   │
│   ├── compute_error_rate()
│   │   ├── Query: sum(rate(http_errors_total[1m]))
│   │   └── Query: sum(rate(http_requests_total[1m]))
│   │
│   └── fetch_code_from_github()
│       └── GET GitHub API with base64 decode
│
├── 🤖 AI ORCHESTRATION (Lines 244-323)
│   ├── call_openai_with_functions()
│   │   └── GPT-4o with function calling
│   │
│   ├── execute_function_call()
│   │   └── Routes to appropriate handler
│   │
│   └── execute_parallel_function_calls()
│       └── asyncio.gather for efficiency
│
├── 📱 NOTIFICATION LAYER (Lines 325-422)
│   ├── send_whatsapp_message()
│   │   └── Twilio messages.create()
│   │
│   ├── create_jira_issue()
│   │   └── POST /rest/api/3/issue
│   │
│   └── format_diagnosis_for_whatsapp()
│       └── Emoji-rich formatted message
│
├── 💾 STATE MANAGEMENT (Lines 424-434)
│   ├── conversation_storage (in-memory dict)
│   ├── store_conversation()
│   └── get_conversation()
│
└── 🌐 API ENDPOINTS (Lines 436-690)
    ├── GET /healthz
    │   └── Health check
    │
    ├── POST /alerts ⭐ MAIN ENTRY POINT
    │   ├── Receives Alertmanager webhooks
    │   └── Triggers AI diagnosis workflow
    │
    ├── POST /webhook/twilio ⭐ HUMAN INTERACTION
    │   ├── Receives WhatsApp replies
    │   └── Handles Jira creation requests
    │
    └── POST /human/decision
        └── Placeholder for future use


🔄 FLOW DIAGRAM 1: ALERT PROCESSING WORKFLOW
================================================

┌─────────────────────────────────────────────────────────────────┐
│                    POST /alerts ENDPOINT                         │
│                   (Line 440-550)                                 │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. RECEIVE ALERT FROM ALERTMANAGER                             │
│     - Extract alert data (name, severity, service)              │
│     - Build initial message for AI                              │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. INITIALIZE CONVERSATION                                     │
│     messages = [                                                 │
│       {"role": "system", "content": SYSTEM_PROMPT},             │
│       {"role": "user", "content": alert_details}                │
│     ]                                                            │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. AI ITERATION LOOP (max 10 iterations)                       │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                ┌──────────┴──────────┐
                │                     │
                ▼                     │
   ┌──────────────────────┐          │
   │ Call OpenAI GPT-4o   │          │
   │ with function tools  │          │
   └──────────┬───────────┘          │
              │                       │
              ▼                       │
   ┌──────────────────────┐          │
   │ AI Returns Response  │          │
   │ + Function Calls     │          │
   └──────────┬───────────┘          │
              │                       │
              ▼                       │
   ┌─────────────────────────────────┴───────────────────────┐
   │  PARALLEL FUNCTION EXECUTION                            │
   │  (execute_parallel_function_calls)                      │
   │                                                          │
   │  ┌────────────┐  ┌──────────────┐  ┌─────────────┐    │
   │  │fetch_logs()│  │compute_error │  │fetch_code   │    │
   │  │            │  │_rate()       │  │_from_github │    │
   │  └────────────┘  └──────────────┘  └─────────────┘    │
   │         │               │                  │            │
   │         └───────────────┴──────────────────┘            │
   │                         │                                │
   │                  asyncio.gather()                        │
   │                         │                                │
   └─────────────────────────┴──────────────────────────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │ Add results to       │
                  │ conversation history │
                  └──────────┬───────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │ Check if             │
              ┌───│ send_diagnosis()     │───┐
              │   │ was called?          │   │
              │   └──────────────────────┘   │
              │                               │
         YES  │                               │  NO
              │                               │
              ▼                               ▼
   ┌──────────────────────┐      ┌──────────────────────┐
   │ 4. DIAGNOSIS COMPLETE│      │ Continue AI loop     │
   │    (break loop)      │      │ (go back to step 3)  │
   └──────────┬───────────┘      └──────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. POST-DIAGNOSIS ACTIONS                                      │
│     ├── Generate conversation_id (UUID)                         │
│     ├── Store conversation in memory                            │
│     ├── Format diagnosis for WhatsApp                           │
│     └── Send WhatsApp notification to user                      │
└─────────────────────────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────────┐
│  6. RETURN SUCCESS RESPONSE                                     │
│     {"ok": True, "iterations": N, "final_response": ...}       │
└─────────────────────────────────────────────────────────────────┘


🔄 FLOW DIAGRAM 2: WHATSAPP INTERACTION (HUMAN-IN-THE-LOOP)
============================================================


┌─────────────────────────────────────────────────────────────────┐
│              POST /webhook/twilio ENDPOINT                       │
│                  (Line 552-677)                                  │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. RECEIVE WHATSAPP MESSAGE FROM USER                          │
│     - Extract from_number                                       │
│     - Extract message_body                                      │
│     Example: "Create Jira task and assign to Ravi"             │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. RETRIEVE CONVERSATION CONTEXT                               │
│     - Get latest conversation from storage                      │
│     - Extract diagnosis data (root cause, file, priority)       │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. BUILD CONTEXT MESSAGE FOR AI                                │
│     context_message = f"""                                      │
│       User message: "{message_body}"                            │
│       Incident context:                                         │
│       - Root Cause: {diagnosis.root_cause}                      │
│       - Affected File: {diagnosis.affected_file}                │
│       - Priority: {diagnosis.priority}                          │
│     """                                                          │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. AI ITERATION LOOP (max 5 iterations)                        │
│     - Append user message to conversation                       │
│     - Call OpenAI with function calling                         │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. AI DETECTS JIRA REQUEST                                     │
│     - AI calls create_jira_ticket()                             │
│     - Extract parameters:                                       │
│       • summary (from incident)                                 │
│       • description (from diagnosis)                            │
│       • assignee (from user message, e.g., "Ravi")             │
│       • priority (from diagnosis)                               │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  6. EXECUTE create_jira_issue()                                 │
│     ├── Build Basic Auth (email:token)                          │
│     ├── POST to JIRA_URL/rest/api/3/issue                      │
│     ├── Payload includes:                                       │
│     │   • project key                                           │
│     │   • summary                                               │
│     │   • description (ADF format)                              │
│     │   • issuetype: "Task"                                     │
│     │   • assignee (if provided)                                │
│     └── Return: issue_key, issue_url, issue_id                 │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                    ┌──────┴───────┐
                    │              │
              SUCCESS│              │FAILURE
                    │              │
                    ▼              ▼
   ┌─────────────────────┐  ┌─────────────────────┐
   │ Format Success Msg  │  │ Format Error Message│
   │ ✅ Jira Task Created│  │ ❌ Failed to create │
   │ Task: PROJ-123      │  │ Error: {reason}     │
   │ URL: jira.com/...   │  │                     │
   │ Assignee: Ravi      │  │                     │
   └──────────┬──────────┘  └──────────┬──────────┘
              │                        │
              └────────────┬───────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  7. SEND WHATSAPP REPLY TO USER                                 │
│     - Twilio sends formatted message                            │
└─────────────────────────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────────┐
│  8. RETURN SUCCESS RESPONSE                                     │
│     {"ok": True}                                                 │
└─────────────────────────────────────────────────────────────────┘



Data Fetching Functions
=======================

- fetch_recent_logs() 

┌─────────────────┐
│  Load Balancer  │
│  /logs/tail     │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────┐
│ GET {LB_URL}/logs/tail?     │
│   seconds=300&limit=2000    │
└────────┬────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│ Returns: List[Dict]         │
│ {                           │
│   "timestamp": "...",       │
│   "level": "ERROR",         │
│   "message": "...",         │
│   "service": "inventory-1"  │
│ }                           │
└─────────────────────────────┘



- compute_error_rate()

┌──────────────────────────────────────────┐
│ Step 1: Query error rate                 │
│ sum(rate(http_errors_total[1m]))        │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│ Step 2: Query total request rate        │
│ sum(rate(http_requests_total[1m]))      │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│ Step 3: Calculate error rate             │
│ error_rate = errors / total              │
│ Returns: float (0.0 to 1.0)              │
└──────────────────────────────────────────┘



- fetch_code_from_github()

┌────────────────────────────────────────┐
│ GitHub API Call                        │
│ GET /repos/{owner}/{repo}/contents/    │
│     {file_path}?ref={branch}           │
└────────┬───────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────┐
│ Response includes:                     │
│ - content (base64 encoded)             │
│ - size, sha, url                       │
└────────┬───────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────┐
│ Decode base64 → readable code          │
│ Return: Dict with file content         │
└────────────────────────────────────────┘



Execution Flow Functions
========================

- call_openai_with_functions() 

Input: messages (conversation history)
    ↓
OpenAI API Call:
- Model: gpt-4o
- Tools: FUNCTION_DEFINITIONS (5 tools)
- tool_choice: "auto"
- temperature: 0.1 (deterministic)
    ↓
Output: ChatCompletion response
- May contain function calls (tool_calls)
- May contain text response (content)



- execute_parallel_function_calls() 

Input: List of function_calls
    ↓
┌─────────────────────────────────────────┐
│ Create async tasks for each call:      │
│                                         │
│ tasks = [                               │
│   execute_single_call(call_1),         │
│   execute_single_call(call_2),         │
│   execute_single_call(call_3)          │
│ ]                                       │
└────────┬────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────┐
│ Execute in parallel:                    │
│ results = await asyncio.gather(*tasks)  │
│                                         │
│ ⚡ Key: This makes multiple API calls   │
│    simultaneously for speed             │
└────────┬────────────────────────────────┘
         │
         ▼
Output: List of results (one per call)




Notification Layer
==================


- send_whatsapp_message() 

Input: message text
    ↓
Check: twilio_client initialized?
    │
    ├─ NO → Print skip message, return False
    │
    └─ YES ↓
    
Twilio API Call:
twilio_client.messages.create(
    body=message,
    from_=TWILIO_WHATSAPP_NUMBER,
    to=USER_WHATSAPP_NUMBER
)
    ↓
Output: Bool (success/failure)




- create_jira_issue() 

Input: summary, description, assignee, priority
    ↓
Step 1: Build Basic Auth header
    auth_string = f"{JIRA_EMAIL}:{JIRA_API_TOKEN}"
    base64_encode → Authorization header
    ↓
Step 2: Build Jira payload
    {
      "fields": {
        "project": {"key": JIRA_PROJECT_KEY},
        "summary": "...",
        "description": {...ADF format...},
        "issuetype": {"name": "Task"},
        "assignee": {"name": "ravi"}
      }
    }
    ↓
Step 3: POST to Jira
    POST {JIRA_URL}/rest/api/3/issue
    ↓
Output: {
  "success": True,
  "issue_key": "PROJ-123",
  "issue_url": "https://jira.com/browse/PROJ-123",
  "issue_id": "10001"
}



- format_diagnosis_for_whatsapp()


Input: diagnosis dict
    ↓
Format with emojis and structure:
🚨 *Incident Diagnosis*

*Root Cause:* {diagnosis.root_cause}
*File:* {diagnosis.affected_file}
*Priority:* {diagnosis.priority}

*Suggested Fix:*
{diagnosis.suggested_fix}

*Next Steps:*
- Step 1
- Step 2

Reply with your action
    ↓
Output: Formatted string



Conversation State Management
============================

conversation_storage = {
  "<uuid>": {
    "messages": [...conversation history...],
    "diagnosis": {...diagnosis data...},
    "timestamp": 1234567890
  }
}



Human-in-the-Loop Flow
======================

Alert → AI Diagnosis → WhatsApp Notification
                              ↓
                        User Replies
                              ↓
                   AI Processes Request
                              ↓
                      Creates Jira Task
                              ↓
                  Confirms via WhatsApp




📈 EXECUTION TIMELINE
=====================

TIME →
│
├─ T0: Alert fires in Alertmanager
│       │
│       └─→ POST /alerts received
│
├─ T1: AI Iteration 1 begins
│       │
│       ├─→ AI calls fetch_logs() + compute_error_rate() in parallel
│       │
│       └─→ Results returned (e.g., 500ms)
│
├─ T2: AI Iteration 2
│       │
│       ├─→ AI analyzes logs, finds NPE error
│       │
│       └─→ AI calls fetch_code_from_github("inventory_service.py")
│
├─ T3: AI Iteration 3
│       │
│       ├─→ AI analyzes code, identifies line 47
│       │
│       └─→ AI calls send_diagnosis() with full report
│
├─ T4: Diagnosis complete
│       │
│       ├─→ Store conversation (UUID)
│       │
│       ├─→ Format WhatsApp message
│       │
│       └─→ Send WhatsApp notification
│
├─ T5: Human receives WhatsApp message
│       │
│       └─→ Reads diagnosis, decides action
│
├─ T6: Human replies "Create Jira task assign to Ravi"
│       │
│       └─→ POST /webhook/twilio received
│
├─ T7: AI processes user request
│       │
│       ├─→ Extracts intent: create Jira
│       │
│       ├─→ Extracts assignee: "Ravi"
│       │
│       └─→ Calls create_jira_ticket()
│
├─ T8: Jira ticket created
│       │
│       └─→ Returns: PROJ-123
│
└─ T9: Confirmation sent via WhatsApp
        │
        └─→ "✅ Jira Task Created: PROJ-123"




💡 USAGE EXAMPLES
Scenario 1: Alert → Diagnosis → WhatsApp

1. Error rate > 20% for 1 minute
2. Alertmanager sends webhook to POST /alerts
3. AI fetches logs + metrics in parallel
4. AI finds: "NullPointerException at line 47"
5. AI fetches inventory_service.py from GitHub
6. AI analyzes code, generates diagnosis
7. WhatsApp sent: "🚨 Incident Diagnosis... Reply with action"


Scenario 2: Human Creates Jira Ticket

1. User receives WhatsApp diagnosis
2. User replies: "create jira task assign to ravi"
3. Twilio forwards to POST /webhook/twilio
4. AI extracts: action=create_jira, assignee=ravi
5. AI calls create_jira_ticket() with incident details
6. Jira API returns: PROJ-456
7. WhatsApp confirmation: "✅ Jira Task Created: PROJ-456"