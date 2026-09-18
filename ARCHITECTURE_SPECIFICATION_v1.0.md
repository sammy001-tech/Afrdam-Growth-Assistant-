# AFRIDAM GROWTH ASSISTANT — ARCHITECTURE SPECIFICATION v1.0
**Document Version:** 1.0.0  
**Status:** Frozen / Approved for Build Specification  
**System Classification:** Enterprise Growth Operations AI System for Afridam AI Internship

---

## 1. Executive Summary & Architectural Vision

The **Afridam Growth Assistant** is a unified Growth Operations AI platform designed to manage, execute, measure, and document growth initiatives, prospect pipelines, campaigns, and internship deliverables for Afridam AI.

The platform operates through two complementary interfaces connected to a single authoritative backend:
1. **Telegram Operating Interface (Mobile Speed & Instant Action):** Natural language command and conversational logging from mobile devices for on-the-go prospecting, priority checks, activity recording, and follow-up generation.
2. **Web Control Center (Visibility, Deep Analytics & AI Workspace):** Mobile-first responsive dashboard providing high-density pipeline oversight, campaign experiment trackers, deterministic KPI funnels, verifiable evidence archives, weekly report generators, and visual AI execution workflows.

```
                                USER
                                 │
                   ┌─────────────┴─────────────┐
                   ▼                           ▼
          📱 TELEGRAM INTERFACE         🌐 WEB CONTROL CENTER
          - Conversational commands      - Mobile-first dashboard
          - Quick lead logging           - Visual pipeline & dossiers
          - Real-time priority alerts    - Campaign management
          - Instant draft generation     - Verifiable evidence vault
          - Deep link: "Open Dashboard"  - AI Workspace & report builder
                   │                           │
                   └─────────────┬─────────────┘
                                 ▼
                   SHARED REST & SSE API LAYER
                   (Node.js + Express + TypeScript)
                                 │
                   ┌─────────────┴─────────────┐
                   ▼                           ▼
          🤖 AI ORCHESTRATION LAYER      🔒 CORE BUSINESS ENGINES
          - Gemini Flash via @google/genai - Deterministic KPI Engine
          - Multi-tool dispatching       - Authentication & Link Broker
          - Structured Zod validation    - Job Scheduler & Task Dispatcher
          - Dual-tier approval gates     - Evidence & Audit Verifier
                   │                           │
                   └─────────────┬─────────────┘
                                 ▼
                     POSTGRESQL SOURCE OF TRUTH
                     (Drizzle ORM + Strict Types)
         ┌───────────────────────┼───────────────────────┐
         ▼                       ▼                       ▼
   Leads & Dossiers       Campaigns & Tasks       KPIs & Activities
   Evidence Vault         Internship Milestones   Audit Logs & Sessions
```

---

## 2. Core Infrastructure & Technology Stack

| Layer | Technology | Role & Rationale |
| :--- | :--- | :--- |
| **Client Frontend** | React 19 + TypeScript + Tailwind CSS | Responsive, mobile-first control center with dense data tables and zero-clutter views. |
| **Client Animations** | Motion (`motion/react`) | Fluid route transitions and micro-interactions. |
| **Backend Runtime** | Node.js + Express (TypeScript via `tsx` & `esbuild`) | Unified API serving both the Web application and Telegram Webhook/polling pipelines. |
| **AI Intelligence** | Gemini API (`gemini-3.8-flash`) via `@google/genai` | High-reasoning structured tool calling, lead dossier synthesis, and outreach drafting. |
| **Database Engine** | PostgreSQL + Drizzle ORM | Acid-compliant relational persistence. All data (leads, activities, evidence, KPIs) resides here. |
| **Validation Layer** | Zod | Runtime type safety and schema validation for all API inputs, tool payloads, and AI outputs. |
| **Telegram API** | Telegram Bot API (Webhook + polling fallback) | Bi-directional conversational interface with rich reply markups and deep-link generation. |
| **Data Visualization**| Recharts & Lucide Icons | Accessible conversion funnels, activity trends, and status indicators. |

---

## 3. Telegram ↔ Web Identity & Dual-Channel Synchronization

### 3.1 Identity Mapping
To ensure that actions taken in Telegram instantly reflect in the Web application without data bifurcation, the database maintains a unified `users` entity:
- **`id`**: Primary UUID.
- **`telegram_user_id`**: Unique BigInt storing the user's verified Telegram ID.
- **`telegram_username`**: Telegram handle for reference.
- **`web_session_token`**: Secure authentication token for web sessions.
- **`status`**: Active / Suspended.

### 3.2 Deep Linking & Handoff Protocols
1. **Web to Telegram Deep Link:**  
   The web dashboard generates a signed one-time deep link:  
   `https://t.me/AfridamGrowthBot?start=link_token_<HMAC>`  
   When opened on mobile, the Telegram bot verifies the token and immediately pairs the account, rendering `Telegram Connected ✓` on the web interface.
2. **Telegram to Web Deep Link:**  
   Telegram inline keyboards and automated digests include:  
   `[ Open Full Dashboard ]` -> `https://<APP_URL>/?auth_ref=<ONE_TIME_TOKEN>`  
   This opens the user directly to the relevant view (e.g., specific lead dossier, campaign analytics, or weekly report).
3. **Zero-Drift Shared Memory:**  
   When the user sends *"I just contacted 5 healthcare prospects"* to Telegram:
   - Backend creates 5 activity log records associated with the user ID.
   - Lead status updates to `contacted`.
   - KPI counters for `leads_contacted` increment deterministically.
   - Any open web session displays the updated metrics on refresh or live poll.

---

## 4. PostgreSQL Relational Database Schema (Drizzle ORM)

All database entities are strictly modeled without mock placeholders or static fallbacks.

### 4.1 Schema Definitions (`src/db/schema.ts`)

```typescript
// Users & Authentication
export const users = pgTable('users', {
  id: uuid('id').defaultRandom().primaryKey(),
  email: text('email').unique().notNull(),
  fullName: text('full_name').notNull(),
  telegramUserId: text('telegram_user_id').unique(),
  telegramUsername: text('telegram_username'),
  telegramConnectedAt: timestamp('telegram_connected_at'),
  role: text('role').default('growth_intern').notNull(), // 'growth_intern' | 'lead' | 'admin'
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').defaultNow().notNull(),
});

// Internship Master & Objective Tracker
export const internships = pgTable('internships', {
  id: uuid('id').defaultRandom().primaryKey(),
  userId: uuid('user_id').references(() => users.id).notNull(),
  title: text('title').notNull(), // e.g. "Afridam AI Growth Operations Internship"
  track: text('track').notNull(), // e.g. "B2B Growth & Lead Operations"
  startDate: timestamp('start_date').notNull(),
  endDate: timestamp('end_date').notNull(),
  targetLeadsQuota: integer('target_leads_quota').default(200).notNull(),
  targetResponseRate: numeric('target_response_rate', { precision: 5, scale: 2 }).default('25.00').notNull(),
  targetQualifiedDeals: integer('target_qualified_deals').default(10).notNull(),
  status: text('status').default('active').notNull(), // 'active' | 'completed' | 'evaluated'
  supervisorNotes: text('supervisor_notes'),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});

// Leads & Prospects
export const leads = pgTable('leads', {
  id: uuid('id').defaultRandom().primaryKey(),
  userId: uuid('user_id').references(() => users.id).notNull(),
  fullName: text('full_name').notNull(),
  jobTitle: text('job_title').notNull(),
  companyName: text('company_name').notNull(),
  industry: text('industry').notNull(), // 'healthcare', 'fintech', 'ai_enterprise', etc.
  email: text('email'),
  linkedinUrl: text('linkedin_url'),
  phone: text('phone'),
  location: text('location'),
  companySize: text('company_size'), // '1-10', '11-50', '51-200', '201-1000', '1000+'
  status: text('status').default('identified').notNull(), 
  // 'identified' | 'researched' | 'contacted' | 'replied' | 'meeting_scheduled' | 'qualified' | 'disqualified'
  sentiment: text('sentiment').default('neutral'), // 'positive' | 'neutral' | 'negative'
  leadScore: integer('lead_score').default(0).notNull(), // 0 - 100
  source: text('source').notNull(), // 'telegram_ai', 'web_research', 'csv_import', 'manual'
  dossierSummary: text('dossier_summary'),
  keyPainPoints: text('key_pain_points'),
  recommendedApproach: text('recommended_approach'),
  lastContactedAt: timestamp('last_contacted_at'),
  nextFollowUpDate: timestamp('next_follow_up_date'),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').defaultNow().notNull(),
});

// Growth Campaigns & Experiments
export const campaigns = pgTable('campaigns', {
  id: uuid('id').defaultRandom().primaryKey(),
  userId: uuid('user_id').references(() => users.id).notNull(),
  name: text('name').notNull(),
  objective: text('objective').notNull(),
  targetAudience: text('target_audience').notNull(),
  channel: text('channel').notNull(), // 'email', 'linkedin', 'multichannel', 'content'
  status: text('status').default('active').notNull(), // 'draft' | 'active' | 'paused' | 'completed'
  reachCount: integer('reach_count').default(0).notNull(),
  repliesCount: integer('replies_count').default(0).notNull(),
  positiveCount: integer('positive_count').default(0).notNull(),
  hypothesis: text('hypothesis'),
  keyLearnings: text('key_learnings'),
  startDate: timestamp('start_date').defaultNow().notNull(),
  endDate: timestamp('end_date'),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});

// Tasks & Work Engine
export const tasks = pgTable('tasks', {
  id: uuid('id').defaultRandom().primaryKey(),
  userId: uuid('user_id').references(() => users.id).notNull(),
  title: text('title').notNull(),
  description: text('description'),
  category: text('category').notNull(), // 'research', 'outreach', 'follow_up', 'analysis', 'reporting'
  priority: text('priority').default('medium').notNull(), // 'low' | 'medium' | 'high' | 'urgent'
  status: text('status').default('todo').notNull(), // 'todo' | 'in_progress' | 'completed' | 'cancelled'
  dueDate: timestamp('due_date').notNull(),
  completedAt: timestamp('completed_at'),
  linkedLeadId: uuid('linked_lead_id').references(() => leads.id),
  linkedCampaignId: uuid('linked_campaign_id').references(() => campaigns.id),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});

// Activity Stream & Telemetry
export const activities = pgTable('activities', {
  id: uuid('id').defaultRandom().primaryKey(),
  userId: uuid('user_id').references(() => users.id).notNull(),
  channel: text('channel').notNull(), // 'telegram' | 'web' | 'system'
  activityType: text('activity_type').notNull(), 
  // 'lead_sourced' | 'lead_contacted' | 'reply_received' | 'positive_sentiment' | 'meeting_booked' | 'task_completed'
  description: text('description').notNull(),
  leadId: uuid('lead_id').references(() => leads.id),
  campaignId: uuid('campaign_id').references(() => campaigns.id),
  taskId: uuid('task_id').references(() => tasks.id),
  metadata: jsonb('metadata'),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});

// Verifiable Evidence Vault
export const evidence = pgTable('evidence', {
  id: uuid('id').defaultRandom().primaryKey(),
  userId: uuid('user_id').references(() => users.id).notNull(),
  title: text('title').notNull(),
  evidenceType: text('evidence_type').notNull(), // 'screenshot', 'url', 'outreach_log', 'meeting_notes', 'kpi_report'
  url: text('url'),
  fileStoragePath: text('file_storage_path'),
  description: text('description'),
  verified: boolean('verified').default(true).notNull(),
  verificationNotes: text('verification_notes'),
  linkedActivityId: uuid('linked_activity_id').references(() => activities.id),
  linkedCampaignId: uuid('linked_campaign_id').references(() => campaigns.id),
  linkedLeadId: uuid('linked_lead_id').references(() => leads.id),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});

// Deterministic Daily / Weekly KPI Aggregations
export const kpiSnapshots = pgTable('kpi_snapshots', {
  id: uuid('id').defaultRandom().primaryKey(),
  userId: uuid('user_id').references(() => users.id).notNull(),
  snapshotDate: date('snapshot_date').notNull(),
  totalLeads: integer('total_leads').default(0).notNull(),
  leadsContacted: integer('leads_contacted').default(0).notNull(),
  repliesReceived: integer('replies_received').default(0).notNull(),
  positiveReplies: integer('positive_replies').default(0).notNull(),
  dealsQualified: integer('deals_qualified').default(0).notNull(),
  campaignReach: integer('campaign_reach').default(0).notNull(),
  tasksCompleted: integer('tasks_completed').default(0).notNull(),
  totalTasksScheduled: integer('total_tasks_scheduled').default(0).notNull(),
  conversionRate: numeric('conversion_rate', { precision: 5, scale: 2 }).default('0.00').notNull(),
  responseRate: numeric('response_rate', { precision: 5, scale: 2 }).default('0.00').notNull(),
  taskCompletionRate: numeric('task_completion_rate', { precision: 5, scale: 2 }).default('0.00').notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});

// Internship Reports
export const reports = pgTable('reports', {
  id: uuid('id').defaultRandom().primaryKey(),
  userId: uuid('user_id').references(() => users.id).notNull(),
  title: text('title').notNull(),
  reportType: text('report_type').notNull(), // 'weekly_performance' | 'monthly_overview' | 'final_evaluation'
  reportingPeriodStart: date('reporting_period_start').notNull(),
  reportingPeriodEnd: date('reporting_period_end').notNull(),
  summaryMarkdown: text('summary_markdown').notNull(),
  kpiMetricsSnapshot: jsonb('kpi_metrics_snapshot').notNull(),
  evidenceIds: jsonb('evidence_ids'), // array of UUIDs
  recommendations: text('recommendations'),
  isFinalized: boolean('is_finalized').default(false).notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});

// AI Agent Conversations & Message Audit
export const conversations = pgTable('conversations', {
  id: uuid('id').defaultRandom().primaryKey(),
  userId: uuid('user_id').references(() => users.id).notNull(),
  channel: text('channel').notNull(), // 'telegram' | 'web'
  title: text('title').default('Growth Session').notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});

export const messages = pgTable('messages', {
  id: uuid('id').defaultRandom().primaryKey(),
  conversationId: uuid('conversation_id').references(() => conversations.id).notNull(),
  role: text('role').notNull(), // 'user' | 'assistant' | 'tool'
  content: text('content').notNull(),
  toolInvocations: jsonb('tool_invocations'),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});

// AI Tool Execution & Security Audit Logs
export const aiAuditLogs = pgTable('ai_audit_logs', {
  id: uuid('id').defaultRandom().primaryKey(),
  userId: uuid('user_id').references(() => users.id).notNull(),
  channel: text('channel').notNull(), // 'telegram' | 'web'
  toolName: text('tool_name').notNull(),
  toolArguments: jsonb('tool_arguments').notNull(),
  toolResult: jsonb('tool_result'),
  status: text('status').notNull(), // 'executed' | 'rejected' | 'failed' | 'requires_approval'
  executionTimeMs: integer('execution_time_ms'),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});
```

---

## 5. AI Tool Inventory & Deterministic Orchestration

All AI capabilities are mediated through strictly typed functions called by Gemini. The agent does **not** improvise database writes or state changes; it invokes dedicated deterministic tools.

```
AI Model (Gemini 3.8 Flash)
       │
       ▼
[ Function Calling Dispatcher ]
       │
       ├─► Input Validation (Zod Schema)
       ├─► Approval Gate Check (Autonomous vs User Confirmed)
       ├─► Database Transaction (PostgreSQL via Drizzle)
       ├─► Deterministic KPI Recalculation
       └─► Structured JSON Response
```

### 5.1 Tool Catalog

#### 1. `search_and_add_prospects`
- **Purpose:** Identifies target leads based on ICP criteria (e.g. Healthcare, B2B AI) and populates the database.
- **Inputs:**
  - `industry`: string (e.g., "healthcare", "logistics")
  - `roles`: string[] (e.g., ["Head of Operations", "Chief Information Officer"])
  - `location`: string (e.g., "Africa", "Global Remote")
  - `count`: number (1-25)
  - `campaignId`: optional UUID
- **Database Effect:** Inserts `leads` with status `identified`, creates `activities` ("lead_sourced").
- **Approval Gate:** Autonomous.

#### 2. `generate_lead_dossier`
- **Purpose:** Researches and analyzes a prospect’s company to synthesize value propositions and pain points.
- **Inputs:**
  - `leadId`: UUID
  - `deepAnalysis`: boolean
- **Database Effect:** Updates `leads` with `dossier_summary`, `key_pain_points`, `recommended_approach`, updates status to `researched`.
- **Approval Gate:** Autonomous.

#### 3. `draft_personalized_outreach`
- **Purpose:** Generates customized outreach messaging tailored to the lead dossier and Afridam AI’s value proposition.
- **Inputs:**
  - `leadId`: UUID
  - `channel`: 'email' | 'linkedin' | 'follow_up'
  - `tone`: 'direct' | 'consultative' | 'relationship'
- **Database Effect:** Returns formatted draft text, saves draft to lead notes.
- **Approval Gate:** Autonomous.

#### 4. `log_outreach_activity`
- **Purpose:** Records that a contact attempt was made, updating the pipeline and KPIs.
- **Inputs:**
  - `leadId`: UUID
  - `channelUsed`: 'email' | 'linkedin' | 'call' | 'other'
  - `notes`: string
  - `nextFollowUpDays`: optional integer
- **Database Effect:** Updates `leads.status` to `contacted`, updates `last_contacted_at`, calculates `next_follow_up_date`, logs in `activities`, updates KPI snapshot.
- **Approval Gate:** Autonomous.

#### 5. `record_prospect_response`
- **Purpose:** Logs responses from prospects, categorizing sentiment and triggering follow-ups.
- **Inputs:**
  - `leadId`: UUID
  - `responseContent`: string
  - `sentiment`: 'positive' | 'neutral' | 'negative'
  - `bookingRequested`: boolean
- **Database Effect:** Updates `leads.status` to `replied` (or `qualified`), increments `replies_count` and `positive_count` in linked campaign, creates task if follow-up required.
- **Approval Gate:** Autonomous.

#### 6. `create_growth_task`
- **Purpose:** Schedules actionable items for today, this week, or a campaign.
- **Inputs:**
  - `title`: string
  - `category`: 'research' | 'outreach' | 'follow_up' | 'analysis' | 'reporting'
  - `priority`: 'low' | 'medium' | 'high' | 'urgent'
  - `dueDate`: ISO date string
  - `linkedLeadId`: optional UUID
  - `linkedCampaignId`: optional UUID
- **Database Effect:** Inserts record into `tasks`.
- **Approval Gate:** Autonomous.

#### 7. `attach_evidence_record`
- **Purpose:** Ingests URLs, screenshots, or document logs proving growth activities.
- **Inputs:**
  - `title`: string
  - `evidenceType`: 'screenshot' | 'url' | 'outreach_log' | 'meeting_notes' | 'kpi_report'
  - `urlOrPath`: string
  - `description`: string
  - `linkedActivityId`: optional UUID
  - `linkedCampaignId`: optional UUID
- **Database Effect:** Inserts record into `evidence` with verification metadata.
- **Approval Gate:** Autonomous.

#### 8. `generate_internship_report`
- **Purpose:** Synthesizes deterministic KPIs, completed tasks, and evidence records into a comprehensive markdown report for internship supervisors.
- **Inputs:**
  - `reportType`: 'weekly_performance' | 'monthly_overview' | 'final_evaluation'
  - `periodStartDate`: ISO date
  - `periodEndDate`: ISO date
- **Database Effect:** Pulls real aggregated stats, builds markdown body, inserts into `reports`.
- **Approval Gate:** Requires user review before finalization.

---

## 6. Telegram Operating Workflow

Telegram acts as the rapid operating interface for the user working from mobile.

### 6.1 Command Matrix
- `/start`: Welcomes user, establishes connection status with web account, provides deep links.
- `/priorities`: Returns today's high-priority tasks, scheduled follow-ups, and pending actions.
- `/prospects [query]`: Searches and logs new target leads into the database.
- `/draft [lead_name]`: Drafts an outreach or follow-up message using the prospect's stored dossier.
- `/log [lead_name] [action]`: Fast activity logger (e.g., `/log Dr. Mensah Sent LinkedIn invite`).
- `/status`: Instant snapshot of today's progress (Leads contacted, Replies, Positive responses, Conversion rate).
- `/report`: Generates this week's internship progress summary ready for submission.

### 6.2 Conversational Natural Language Handling
The AI agent handles natural conversational inputs and automatically maps them to tool calls:
- *"I just sent messages to 4 clinic directors in Nairobi."*  
  -> Maps to `log_outreach_activity` x4, updates today's KPI metrics, confirms with summary.
- *"Who should I follow up with today?"*  
  -> Queries `leads` where `next_follow_up_date <= today`, returns formatted list with one-tap action buttons.
- *"Add a screenshot of my LinkedIn outreach for proof."*  
  -> Generates an upload prompt or accepts media attachment, maps to `attach_evidence_record`.

---

## 7. Mobile-First Web Control Center Views

The web dashboard provides complete operational oversight, formatted with high contrast, responsive layouts, and zero clutter.

```
┌─────────────────────────────────────────────────────────────┐
│ ✦ AFRIDAM AI  Growth Operations System       [TG Connected] │
├─────────────────────────────────────────────────────────────┤
│  TODAY'S KPI SNAPSHOT                                       │
│  ┌──────────────┬──────────────┬──────────────┬───────────┐ │
│  │ 47 Leads     │ 11 Replies   │ 6 Positive   │ 2 Deals   │ │
│  │ +5 today     │ 39.3% rate   │ 54.5% ratio  │ 7.1% conv │ │
│  └──────────────┴──────────────┴──────────────┴───────────┘ │
├─────────────────────────────────────────────────────────────┤
│  NAV: [Dashboard] [Leads] [Campaigns] [Tasks] [Evidence]    │
│       [Analytics] [Reports] [AI Workspace]                  │
├─────────────────────────────────────────────────────────────┤
│  VIEW CONTAINER (Fluid responsive, dense data hierarchy)    │
│  - Active Pipeline Kanban / Table                           │
│  - Lead Dossier Inspector                                   │
│  - Deterministic Funnel & Activity Stream                   │
│  - Verifiable Evidence Gallery                              │
│  - AI Workspace with direct tool execution                  │
└─────────────────────────────────────────────────────────────┘
```

### 7.1 View Specifications
1. **Executive Dashboard (`/`):**
   - KPI Summary Grid (Total Leads, Response Rate, Positive Replies, Qualified Deals, Campaign Reach).
   - "Today's Work" Action Matrix (urgent tasks, follow-up queues, today's targets).
   - Weekly Performance Progress Bar against Internship Quotas.
   - Live Activity Feed with channel badges (Telegram vs Web).
2. **Leads Management (`/leads`):**
   - Stage filter pills: All, Identified, Researched, Contacted, Replied, Qualified.
   - Rich Lead Cards with Lead Score (0-100), Industry, Sentiment, and Next Action Date.
   - Lead Dossier Drawer: Company profile, researched pain points, outreach history, and draft generation tool.
3. **Campaigns & Growth Experiments (`/campaigns`):**
   - Active campaigns with reach, response rates, and positive conversion metrics.
   - Hypothesis and key learnings logger for internship documentation.
4. **Tasks & Operations Engine (`/tasks`):**
   - Today, This Week, Overdue, and Completed groupings.
   - Instant inline task creation and priority toggles.
5. **Analytics & Funnel (`/analytics`):**
   - Deterministic Conversion Funnel: Sourced -> Contacted -> Replied -> Positive -> Qualified.
   - Daily activity chart and channel comparison.
6. **Evidence Vault (`/evidence`):**
   - Verifiable repository of screenshots, logs, links, and documents mapped to milestones.
   - Proof integrity status with timestamps.
7. **Internship Performance Reports (`/reports`):**
   - Automated Weekly and Monthly Report Generator formatted for Afridam AI leadership.
   - Sectional breakdown: Executive Summary, Metric Performance, Activities Completed, Key Evidence, Obstacles & Next Week Priorities.
   - Export to Markdown and formatted PDF/Print.
8. **Interactive AI Workspace (`/ai-workspace`):**
   - Visual execution terminal: Select goal (e.g. "Find 15 Healthcare Tech Leads in West Africa", "Analyze Outreach Bottlenecks", "Draft Multi-Touch Follow-up Sequence").
   - Real-time execution log showing active tool calls and generated records.

---

## 8. Afridam Internship Intelligence Framework

To serve as a genuine Growth Operations system rather than a generic bot, the agent is pre-loaded with the **Afridam AI Growth Evaluation Framework**:

### 8.1 Internship Focus Areas & Competencies
1. **Target Account Identification:** Sourcing qualified enterprise, healthcare, and tech leaders aligned with Afridam AI's capabilities.
2. **Outreach Velocity & Discipline:** Maintaining consistent daily outreach targets (minimum 10 touches/day), 24-hour follow-up compliance, and structured cadence.
3. **Message Optimization:** Measuring subject line and value proposition variants to drive response rates above 25%.
4. **Verifiable Proof of Work:** Every outreach batch, positive reply, and meeting booked must be backed by an entry in the Evidence Vault.
5. **Weekly Structured Reporting:** Synthesizing learnings, conversion barriers, and tactical recommendations every Friday.

### 8.2 Operational Questions the System Answers
- *"What am I supposed to achieve?"* -> Fetches `internships` quotas and weekly goal progress.
- *"What have I done?"* -> Queries `activities` and `tasks` completed in the timeframe.
- *"What should I do next?"* -> Evaluates overdue tasks, hot leads needing 48h follow-up, and untouched prospects.
- *"What evidence do I have?"* -> Returns `evidence` vault assets linked to deliverables.
- *"What results am I getting?"* -> Queries deterministic `kpiSnapshots`.
- *"What should I improve?"* -> Analyzes drop-offs in the funnel between `contacted` and `replied`.

---

## 9. Non-Functional Requirements & Anti-Mock Mandate

1. **Zero Hardcoded / Mock Data:**  
   All metrics, leads, activities, tasks, and reports are dynamically loaded and stored via the backend database. If a list is empty upon initial startup, the system provides genuine creation workflows and an "Initialize Internship Baseline" action that commits real database records through the backend API.
2. **Deterministic KPIs:**  
   Conversion rates and ratios are computed using exact server-side arithmetic (e.g., `(positive_replies / replies_received) * 100`), never estimated or hallucinated by language models.
3. **Resilience & Offline Handling:**  
   If network disconnection occurs, state is preserved locally, and pending operations are synced upon reconnection.
4. **Security & Data Isolation:**  
   All Gemini API keys and Telegram Bot tokens remain strictly server-side in Node.js runtime. No sensitive credentials are exposed to the client browser.

---

## 10. Next Phase Execution Plan

With Architecture Specification v1.0 frozen:
1. **Database & ORM Layer:** Implement Drizzle schema, connection pooling, and migration setup.
2. **AI Tooling & Backend Engine:** Build Express server routes, Gemini tool caller with Zod validations, Telegram Bot API webhook/poller, and deterministic KPI calculator.
3. **Mobile-First React Frontend:** Build the full suite of responsive views (Dashboard, Leads & Dossiers, Campaigns, Tasks, Analytics, Evidence Vault, Reports, and AI Workspace).
4. **Dual-Channel Synchronization Testing:** Validate that commands issued via Telegram or the Web UI immediately synchronize across the PostgreSQL database.
