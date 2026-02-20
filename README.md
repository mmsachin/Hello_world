# DocForge — Comprehensive Build Prompt for Gemini

You are building a full-stack application called **DocForge** — an AI-powered Technical Design Document generator. The system uses Gemini CLI (headless mode) as sub-agents orchestrated by a Node.js backend, with an Angular frontend that displays real-time logs and progress.

-----

## ARCHITECTURE OVERVIEW

```
┌──────────────────────────────────────────────────────────────┐
│                    Angular Frontend (UI)                       │
│  - Multi-step form (configure → generate → download)          │
│  - Real-time log streaming via SSE (Server-Sent Events)       │
│  - Section progress cards with live status updates            │
│  - Drag-to-reorder sections, toggle on/off                    │
└──────────────────────────┬───────────────────────────────────┘
                           │ HTTP + SSE
┌──────────────────────────▼───────────────────────────────────┐
│                  Node.js Backend (Express)                     │
│  - REST API endpoints                                         │
│  - Orchestrator: manages workflow, dependency graph, state     │
│  - Spawns shell scripts as child processes (non-blocking)     │
│  - Captures stdout/stderr from each agent in real-time        │
│  - Streams logs + status to frontend via SSE                  │
└──────────────────────────┬───────────────────────────────────┘
                           │ child_process.spawn()
┌──────────────────────────▼───────────────────────────────────┐
│              Shell Scripts (Sub-Agents)                        │
│                                                               │
│  analyzer.sh ──→ Analyzes requirements, produces brief        │
│  section-writer.sh ──→ Writes one section (called N times)    │
│  reviewer.sh ──→ Reviews & improves one section               │
│  merger.sh ──→ Assembles final document                       │
│                                                               │
│  Each script calls: gemini -p "prompt" (headless mode)        │
│  Output goes to files in a job-specific working directory     │
└───────────────────────────────────────────────────────────────┘
```

-----

## CRITICAL TECHNICAL ANSWERS

### Q: Can shell scripts be called from UI to run in background?

YES. The flow is:

1. Angular UI sends HTTP POST to `/api/generate` with requirements + config
1. Backend immediately returns `{ jobId }` (HTTP 202 Accepted)
1. Backend spawns shell scripts using `child_process.spawn()` (NOT `exec`) — this is non-blocking and runs in the background
1. Frontend opens an SSE connection to `/api/status/:jobId` to receive real-time updates
1. Backend pipes stdout/stderr from each spawned process into the SSE stream

### Q: Can we display logs as the agent runs?

YES. Using `spawn()` gives you access to `process.stdout` and `process.stderr` as readable streams. You attach `data` event listeners and push each chunk to the SSE connection. The Angular frontend uses `EventSource` API or `fetch` with `ReadableStream` to receive and display logs in real-time.

-----

## PART 1: PROJECT STRUCTURE

Generate this exact project structure:

```
docforge/
├── backend/
│   ├── package.json                 # express, cors, multer, uuid
│   ├── server.js                    # Express API server with SSE
│   ├── orchestrator.js              # Workflow engine with dependency graph
│   ├── agents/
│   │   ├── analyzer.sh              # Requirements analysis agent
│   │   ├── section-writer.sh        # Section writing agent
│   │   ├── reviewer.sh              # Section review agent
│   │   └── merger.sh                # Document assembly agent
│   ├── templates/
│   │   ├── technical-design.json    # Default TDD template
│   │   ├── rfc.json                 # RFC template
│   │   └── adr.json                 # Architecture Decision Record template
│   └── output/                      # Generated documents (gitignored)
├── frontend/                        # Angular 17+ application
│   ├── src/
│   │   ├── app/
│   │   │   ├── app.component.ts
│   │   │   ├── app.component.html
│   │   │   ├── app.routes.ts
│   │   │   ├── pages/
│   │   │   │   ├── configure/
│   │   │   │   │   ├── configure.component.ts
│   │   │   │   │   ├── configure.component.html
│   │   │   │   │   └── configure.component.scss
│   │   │   │   ├── generate/
│   │   │   │   │   ├── generate.component.ts
│   │   │   │   │   ├── generate.component.html
│   │   │   │   │   └── generate.component.scss
│   │   │   │   └── result/
│   │   │   │       ├── result.component.ts
│   │   │   │       ├── result.component.html
│   │   │   │       └── result.component.scss
│   │   │   ├── services/
│   │   │   │   ├── docforge.service.ts       # API calls + SSE
│   │   │   │   └── log-stream.service.ts     # Log parsing + buffering
│   │   │   ├── models/
│   │   │   │   ├── template.model.ts
│   │   │   │   ├── job.model.ts
│   │   │   │   └── log-entry.model.ts
│   │   │   └── components/
│   │   │       ├── section-list/             # Drag-reorder section list
│   │   │       ├── log-viewer/               # Real-time log display
│   │   │       ├── progress-bar/             # Animated progress bar
│   │   │       └── section-status-card/      # Per-section status card
│   │   ├── environments/
│   │   └── styles.scss
│   ├── angular.json
│   └── package.json
└── README.md
```

-----

## PART 2: BACKEND IMPLEMENTATION

### 2A. Shell Agent Scripts

Each agent script follows this pattern:

- Accepts file paths as arguments
- Reads context from those files
- Constructs a detailed prompt
- Calls Gemini CLI in headless mode
- Writes output to a file
- Prints progress/status to stdout (this gets streamed to UI)

**IMPORTANT:** Use `gemini -p "prompt text"` for headless Gemini CLI invocation. Adjust flags based on the actual Gemini CLI version installed. The scripts must:

- Use `set -euo pipefail` for error handling
- Echo structured log lines to stdout in format: `[AGENT:agent_name] [STATUS:status] message`
- Write output content to the specified output file path
- Exit 0 on success, non-zero on failure

#### analyzer.sh

```
Usage: ./analyzer.sh <requirements_file> <template_file> <output_file>
```

- Reads requirements and template JSON
- Sends a prompt asking Gemini to:
1. Identify key technical components, decisions, and constraints
1. Flag ambiguities or missing info in requirements
1. For each template section, produce 3-5 key points to cover
1. Estimate technical depth needed per section
- Output: JSON with `analysis` and `section_briefs` fields
- Echoes: `[AGENT:analyzer] [STATUS:running] Analyzing requirements...` and `[AGENT:analyzer] [STATUS:complete] Analysis done`

#### section-writer.sh

```
Usage: ./section-writer.sh <section_id> <context_file> <analysis_file> <deps_dir> <output_file>
```

- `context_file`: JSON with section config + original requirements
- `analysis_file`: output from analyzer
- `deps_dir`: directory containing .md files of completed dependency sections (for cross-referencing)
- Constructs prompt that includes all context + tells Gemini to write comprehensive markdown for this specific section
- Prompt instructs: use proper ## and ### headers, include tables, include ASCII diagrams where helpful, be specific not generic, reference prior sections for consistency, include “Open Questions” if any, aim for 500-1500 words
- Echoes: `[AGENT:writer] [STATUS:running] [SECTION:section_id] Writing section...`

#### reviewer.sh

```
Usage: ./reviewer.sh <section_file> <requirements_file> <output_file>
```

- Takes a written section + original requirements
- Asks Gemini to review for: technical accuracy, completeness, missing edge cases, clarity, actionability
- Outputs the IMPROVED version (not review comments — the actual improved text)
- Echoes: `[AGENT:reviewer] [STATUS:running] [SECTION:section_id] Reviewing...`

#### merger.sh

```
Usage: ./merger.sh <sections_dir> <template_file> <metadata_file> <output_file>
```

- Concatenates all reviewed sections
- Asks Gemini to: arrange in template order, add title page, add TOC, write executive summary, fix terminology inconsistencies, add cross-references, add glossary
- Outputs the COMPLETE final markdown document

### 2B. Orchestrator (orchestrator.js)

This is the core workflow engine. Implement as a class extending EventEmitter.

```javascript
class DocOrchestrator extends EventEmitter {
  constructor(workDir, agentsDir) {}

  async generate(requirements, templatePath, metadata) {
    // Returns a Promise that resolves when complete
    // Emits events throughout:
    //   'log'    → { timestamp, agent, section, level, message }
    //   'status' → { phase, progress, message, sections: { id: status } }
    //   'complete' → { jobId, outputPath }
    //   'error'  → { message, agent, section }
  }
}
```

**Key implementation details:**

1. **Spawn, not exec**: Use `child_process.spawn('bash', [scriptPath, ...args])` so you get streaming stdout/stderr. Do NOT use `exec` (it buffers everything).
1. **Stream capture pattern**:

```javascript
const proc = spawn('bash', [scriptPath, ...args], { cwd: jobDir });

proc.stdout.on('data', (chunk) => {
  const line = chunk.toString().trim();
  this.emit('log', { timestamp: Date.now(), agent, message: line, level: 'info' });
});

proc.stderr.on('data', (chunk) => {
  this.emit('log', { timestamp: Date.now(), agent, message: chunk.toString().trim(), level: 'warn' });
});

await new Promise((resolve, reject) => {
  proc.on('close', (code) => code === 0 ? resolve() : reject(new Error(`Agent ${agent} exited with code ${code}`)));
});
```

1. **Dependency graph execution**: Build a topological sort from template `depends_on` fields. Process in rounds — each round contains sections whose dependencies are all satisfied. Run sections within the same round in parallel using `Promise.all()`.
1. **Working directory per job**: Create `output/<jobId>/` with subdirectories: `sections/`, `reviewed/`, `deps/`. Each agent reads/writes from these paths.
1. **Timeout handling**: Set a 5-minute timeout per agent call. If exceeded, kill the process and emit an error.

### 2C. Server (server.js)

Implement these endpoints:

```
POST   /api/generate          — Start generation job
  Body: { title, author, requirements_text, template_choice }
  Or: multipart form with requirements_file upload
  Response: { jobId } (HTTP 202)

GET    /api/status/:jobId     — SSE stream for real-time logs + progress
  Response: text/event-stream with events:
    event: status
    data: { phase, progress, message, sections }

    event: log
    data: { timestamp, agent, section, level, message }

    event: complete
    data: { jobId, downloadUrl }

    event: error
    data: { message }

GET    /api/download/:jobId   — Download generated document
  Query params: ?format=md or ?format=docx
  For docx: use pandoc to convert markdown → docx on the fly

GET    /api/templates         — List available templates
  Response: [{ id, name, description, sectionCount, sections: [...] }]

POST   /api/templates         — Save a custom template (optional)
```

**SSE implementation pattern:**

```javascript
app.get('/api/status/:jobId', (req, res) => {
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
    'X-Accel-Buffering': 'no'  // important for nginx proxying
  });

  // Send keepalive every 15s
  const keepalive = setInterval(() => res.write(': keepalive\n\n'), 15000);

  const onLog = (entry) => res.write(`event: log\ndata: ${JSON.stringify(entry)}\n\n`);
  const onStatus = (status) => res.write(`event: status\ndata: ${JSON.stringify(status)}\n\n`);
  const onComplete = (data) => {
    res.write(`event: complete\ndata: ${JSON.stringify(data)}\n\n`);
    cleanup();
  };

  orchestrator.on('log', onLog);
  orchestrator.on('status', onStatus);
  orchestrator.on('complete', onComplete);

  const cleanup = () => {
    clearInterval(keepalive);
    orchestrator.removeListener('log', onLog);
    orchestrator.removeListener('status', onStatus);
    orchestrator.removeListener('complete', onComplete);
    res.end();
  };

  req.on('close', cleanup);
});
```

-----

## PART 3: ANGULAR FRONTEND

### 3A. Technology Choices

- Angular 17+ with standalone components
- Angular Material for UI components (MatStepper, MatCard, MatChips, MatProgressBar, MatIcon, MatButton, MatFormField, MatInput, MatSelect)
- Angular CDK DragDrop for section reordering
- RxJS for SSE stream handling
- SCSS for styling
- Use Angular signals where appropriate for reactive state

### 3B. Services

#### DocForgeService (`docforge.service.ts`)

```typescript
@Injectable({ providedIn: 'root' })
export class DocForgeService {
  private apiUrl = environment.apiUrl; // http://localhost:3001/api

  // Start generation - returns Observable<{ jobId: string }>
  startGeneration(config: GenerationConfig): Observable<{ jobId: string }>;

  // Connect to SSE stream - returns Observable that emits log/status/complete events
  connectToJobStream(jobId: string): Observable<JobEvent>;

  // Get available templates
  getTemplates(): Observable<Template[]>;

  // Download document
  getDownloadUrl(jobId: string, format: 'md' | 'docx'): string;
}
```

**SSE in Angular pattern (important — EventSource doesn’t support POST, so use it for GET):**

```typescript
connectToJobStream(jobId: string): Observable<JobEvent> {
  return new Observable(observer => {
    const eventSource = new EventSource(`${this.apiUrl}/status/${jobId}`);

    eventSource.addEventListener('log', (e: MessageEvent) => {
      observer.next({ type: 'log', data: JSON.parse(e.data) });
    });

    eventSource.addEventListener('status', (e: MessageEvent) => {
      observer.next({ type: 'status', data: JSON.parse(e.data) });
    });

    eventSource.addEventListener('complete', (e: MessageEvent) => {
      observer.next({ type: 'complete', data: JSON.parse(e.data) });
      eventSource.close();
      observer.complete();
    });

    eventSource.addEventListener('error', (e: MessageEvent) => {
      observer.next({ type: 'error', data: JSON.parse(e.data) });
      eventSource.close();
      observer.complete();
    });

    // Cleanup on unsubscribe
    return () => eventSource.close();
  });
}
```

#### LogStreamService (`log-stream.service.ts`)

```typescript
@Injectable({ providedIn: 'root' })
export class LogStreamService {
  // Parses raw log lines into structured LogEntry objects
  // Maintains a buffer of recent logs (last 500 lines)
  // Provides filtered views: by agent, by section, by level
  // Supports auto-scroll behavior

  private logs = signal<LogEntry[]>([]);
  readonly allLogs = this.logs.asReadonly();

  addLog(entry: LogEntry): void;
  getLogsByAgent(agent: string): Signal<LogEntry[]>;
  getLogsBySection(sectionId: string): Signal<LogEntry[]>;
  clear(): void;
}
```

### 3C. Pages

#### Configure Page (`/configure`)

Layout:

```
┌─────────────────────────────────────────────────────┐
│  DocForge — AI Design Doc Generator     [Template ▼] │
├─────────────────────────────────────────────────────┤
│                                                       │
│  ┌─── Document Info ──────┐  ┌─── Sections ────────┐ │
│  │ Title: [________]      │  │ ⠿ 🎯 Overview    [✓]│ │
│  │ Author: [________]     │  │ ⠿ 📋 Requirements [✓]│ │
│  │                        │  │ ⠿ 🏗️ Architecture [✓]│ │
│  ├─── Requirements ───────┤  │ ⠿ 🗄️ Data Model  [✓]│ │
│  │                        │  │ ⠿ 🔌 API Design  [✓]│ │
│  │  [Paste or upload      │  │ ⠿ 🔒 Security    [✓]│ │
│  │   your requirements    │  │ ⠿ 📈 Scalability [✓]│ │
│  │   here...]             │  │ ⠿ 🧪 Testing     [✓]│ │
│  │                        │  │ ⠿ 🚀 Deployment  [✓]│ │
│  │                        │  │ ⠿ ⚠️ Risks       [✓]│ │
│  │  📎 Upload file        │  │ ⠿ 📅 Timeline    [✓]│ │
│  └────────────────────────┘  └─────────────────────┘ │
│                                                       │
│           [ Generate Document → ]                     │
│           ~22-44 min estimated                        │
└─────────────────────────────────────────────────────┘
```

Features:

- Mat-stepper or custom stepper showing Configure → Generate → Download
- Template dropdown that loads section list from backend
- CDK DragDrop on section list for reordering
- Mat-slide-toggle for enabling/disabling sections
- Textarea with file upload option for requirements
- Character count / word count on requirements
- Estimated time based on enabled section count
- Form validation: requirements min 50 chars

#### Generate Page (`/generate/:jobId`)

Layout:

```
┌─────────────────────────────────────────────────────┐
│  Writing Sections                          67%       │
│  ████████████████████░░░░░░░░░░                     │
│  Currently: Writing API Design...                    │
├─────────────────────────────────────────────────────┤
│                                                       │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐               │
│  │ 🎯   │ │ 📋   │ │ 🏗️   │ │ 🗄️   │               │
│  │ ✓    │ │ ✓    │ │ ✓    │ │ ⟳    │  ...          │
│  │Overview│Requirements│Arch │ │Data  │               │
│  └──────┘ └──────┘ └──────┘ └──────┘               │
│                                                       │
│  ┌─── Pipeline ─────────────────────────────────┐   │
│  │ [✓ Analyze] → [● Write] → [○ Review] → [○ Merge] │
│  └──────────────────────────────────────────────┘   │
│                                                       │
│  ┌─── Live Logs ────────────────────────────────┐   │
│  │ [All] [Analyzer] [Writer] [Reviewer] [Merger] │   │
│  │                                                │   │
│  │ 14:32:01 [analyzer] Parsing requirements...    │   │
│  │ 14:32:05 [analyzer] Found 12 key components    │   │
│  │ 14:32:06 [analyzer] Analysis complete           │   │
│  │ 14:32:07 [writer] Starting: Overview            │   │
│  │ 14:32:08 [writer] Generating content...         │   │
│  │ 14:32:45 [writer] Overview complete (847 words) │   │
│  │ 14:32:46 [reviewer] Reviewing: Overview         │   │
│  │ 14:33:12 [reviewer] Overview improved           │   │
│  │ 14:33:13 [writer] Starting: Requirements        │   │
│  │ > _                                             │   │
│  └─────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

Features:

- Progress bar with phase-aware coloring (amber=analyzing, blue=writing, purple=reviewing, green=merging/complete)
- Section status grid: cards showing pending/writing/reviewing/complete per section
- Pipeline visualization: 4-step horizontal flow showing current phase
- **LOG VIEWER** (the key feature):
  - Dark terminal-style container with monospace font
  - Auto-scrolls to bottom as new logs arrive
  - Filter tabs: All, Analyzer, Writer, Reviewer, Merger
  - Each log line shows: timestamp, agent badge (color-coded), message
  - Log levels: info (white), warn (yellow), error (red)
  - “Pause auto-scroll” toggle when user scrolls up
  - Optional: click a section card to filter logs to that section only

**Log Viewer Component Implementation Notes:**

```typescript
@Component({
  selector: 'app-log-viewer',
  // Use OnPush change detection + manual scroll handling
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class LogViewerComponent {
  logs = input.required<LogEntry[]>();
  activeFilter = signal<string>('all');

  filteredLogs = computed(() => {
    const filter = this.activeFilter();
    if (filter === 'all') return this.logs();
    return this.logs().filter(l => l.agent === filter);
  });

  // Auto-scroll logic:
  // - Track if user has scrolled up (isAutoScroll = false)
  // - On new log, if isAutoScroll, scrollTop = scrollHeight
  // - If user scrolls to bottom, re-enable auto-scroll
}
```

**Log Viewer SCSS:**

```scss
.log-container {
  background: #0a0a0a;
  border: 1px solid #1e293b;
  border-radius: 8px;
  font-family: 'JetBrains Mono', 'Fira Code', monospace;
  font-size: 12px;
  height: 350px;
  overflow-y: auto;
  padding: 12px;

  .log-line {
    display: flex;
    gap: 8px;
    padding: 2px 0;
    line-height: 1.6;

    .timestamp { color: #475569; min-width: 70px; }
    .agent-badge {
      padding: 0 6px;
      border-radius: 3px;
      font-size: 11px;
      font-weight: 600;
      min-width: 64px;
      text-align: center;

      &.analyzer { background: rgba(245,158,11,0.15); color: #f59e0b; }
      &.writer   { background: rgba(59,130,246,0.15); color: #60a5fa; }
      &.reviewer { background: rgba(139,92,246,0.15); color: #a78bfa; }
      &.merger   { background: rgba(16,185,129,0.15); color: #34d399; }
    }
    .message { color: #e2e8f0; }
    &.warn .message { color: #fbbf24; }
    &.error .message { color: #f87171; }
  }
}
```

#### Result Page (`/result/:jobId`)

Layout:

- Large success checkmark animation
- Document title, author, date, section count
- Download buttons: Markdown (.md) and Word (.docx)
- “Generate Another” button
- Optional: preview of the generated markdown rendered as HTML

### 3D. Routing

```typescript
export const routes: Routes = [
  { path: '', redirectTo: 'configure', pathMatch: 'full' },
  { path: 'configure', component: ConfigureComponent },
  { path: 'generate/:jobId', component: GenerateComponent },
  { path: 'result/:jobId', component: ResultComponent },
];
```

### 3E. Models

```typescript
// template.model.ts
export interface Template {
  id: string;
  name: string;
  description: string;
  sections: TemplateSection[];
}

export interface TemplateSection {
  id: string;
  title: string;
  icon: string;
  agent: string;
  depends_on: string[];
  prompt_context: string;
  subsections: string[];
  enabled: boolean;  // UI-only field
}

// job.model.ts
export interface GenerationConfig {
  title: string;
  author: string;
  requirements: string;
  templateId: string;
  sections: TemplateSection[];  // ordered, with enabled flag
}

export interface JobStatus {
  phase: 'idle' | 'analyzing' | 'writing' | 'reviewing' | 'merging' | 'complete' | 'error';
  progress: number;  // 0-100
  message: string;
  sections: Record<string, 'pending' | 'writing' | 'reviewing' | 'complete' | 'error'>;
}

// log-entry.model.ts
export interface LogEntry {
  timestamp: number;
  agent: 'analyzer' | 'writer' | 'reviewer' | 'merger' | 'system';
  section?: string;
  level: 'info' | 'warn' | 'error';
  message: string;
}

export interface JobEvent {
  type: 'log' | 'status' | 'complete' | 'error';
  data: any;
}
```

### 3F. Design System

Use Angular Material with a custom dark theme:

```scss
// styles.scss — Custom Material theme
@use '@angular/material' as mat;

$dark-primary: mat.m2-define-palette(mat.$m2-indigo-palette, 400, 300, 600);
$dark-accent: mat.m2-define-palette(mat.$m2-purple-palette, A200, A100, A400);
$dark-theme: mat.m2-define-dark-theme((
  color: (primary: $dark-primary, accent: $dark-accent),
  typography: mat.m2-define-typography-config(
    $font-family: 'Google Sans, Roboto, sans-serif'
  )
));

@include mat.all-component-themes($dark-theme);

:root {
  --bg-primary: #0a0e1a;
  --bg-card: #111827;
  --bg-elevated: #1e293b;
  --border: #1e293b;
  --text-primary: #e2e8f0;
  --text-secondary: #94a3b8;
  --text-muted: #64748b;
  --accent-blue: #818cf8;
  --accent-green: #4ade80;
  --accent-amber: #f59e0b;
  --accent-purple: #a78bfa;
  --accent-red: #f87171;
}

body {
  background: var(--bg-primary);
  color: var(--text-primary);
  font-family: 'Google Sans', 'Roboto', sans-serif;
  margin: 0;
}
```

-----

## PART 4: DOCUMENT TEMPLATES

### Default Template: Technical Design Document (technical-design.json)

```json
{
  "id": "technical-design",
  "name": "Technical Design Document",
  "description": "Comprehensive technical design doc covering architecture, APIs, data model, security, and deployment.",
  "sections": [
    {
      "id": "overview",
      "title": "Overview & Problem Statement",
      "icon": "🎯",
      "agent": "section-writer",
      "depends_on": [],
      "prompt_context": "Write a clear problem statement. Include business context, user pain points, and why this work matters now. Include Goals and Non-Goals.",
      "subsections": ["Background", "Problem Statement", "Goals", "Non-Goals"]
    },
    {
      "id": "requirements",
      "title": "Requirements",
      "icon": "📋",
      "agent": "section-writer",
      "depends_on": ["overview"],
      "prompt_context": "Detail functional and non-functional requirements. Be specific, measurable, and prioritized (P0/P1/P2).",
      "subsections": ["Functional Requirements", "Non-Functional Requirements", "Constraints", "Assumptions"]
    },
    {
      "id": "architecture",
      "title": "System Architecture",
      "icon": "🏗️",
      "agent": "section-writer",
      "depends_on": ["requirements"],
      "prompt_context": "Describe high-level architecture. Include component diagram (ASCII), data flow, technology choices with justification, and alternatives considered.",
      "subsections": ["High-Level Design", "Component Diagram", "Technology Stack", "Alternatives Considered"]
    },
    {
      "id": "data_model",
      "title": "Data Model",
      "icon": "🗄️",
      "agent": "section-writer",
      "depends_on": ["architecture"],
      "prompt_context": "Define database schema, entity relationships, data flow patterns, indexing strategy. Include migration strategy if applicable.",
      "subsections": ["Entity Definitions", "Relationships", "Indexing Strategy", "Data Flow", "Migration Plan"]
    },
    {
      "id": "api_design",
      "title": "API Design",
      "icon": "🔌",
      "agent": "section-writer",
      "depends_on": ["architecture", "data_model"],
      "prompt_context": "Specify API endpoints with request/response examples, authentication, rate limiting, pagination, versioning strategy, error codes.",
      "subsections": ["Endpoints", "Authentication", "Rate Limiting", "Error Handling", "Versioning"]
    },
    {
      "id": "security",
      "title": "Security Considerations",
      "icon": "🔒",
      "agent": "section-writer",
      "depends_on": ["architecture", "api_design"],
      "prompt_context": "Address threat modeling (STRIDE), authentication/authorization flows, data encryption (at rest + in transit), input validation, compliance (SOC2/GDPR as applicable).",
      "subsections": ["Threat Model", "Auth Flow", "Data Protection", "Input Validation", "Compliance"]
    },
    {
      "id": "scalability",
      "title": "Scalability & Performance",
      "icon": "📈",
      "agent": "section-writer",
      "depends_on": ["architecture"],
      "prompt_context": "Discuss expected load (QPS, storage growth), horizontal/vertical scaling strategy, caching layers, CDN, database sharding/replication, performance targets with SLOs.",
      "subsections": ["Load Estimates", "Scaling Strategy", "Caching", "SLOs & Performance Targets"]
    },
    {
      "id": "testing",
      "title": "Testing Strategy",
      "icon": "🧪",
      "agent": "section-writer",
      "depends_on": ["architecture", "api_design"],
      "prompt_context": "Define unit, integration, e2e, performance, and chaos testing approaches. Include coverage targets, testing tools, CI integration, and test data strategy.",
      "subsections": ["Unit Tests", "Integration Tests", "E2E Tests", "Performance Tests", "Test Data"]
    },
    {
      "id": "deployment",
      "title": "Deployment & Rollout",
      "icon": "🚀",
      "agent": "section-writer",
      "depends_on": ["architecture", "testing"],
      "prompt_context": "Describe CI/CD pipeline, rollout strategy (canary/blue-green with percentages), feature flags, rollback plan, monitoring dashboards, alerting thresholds.",
      "subsections": ["CI/CD Pipeline", "Rollout Strategy", "Feature Flags", "Rollback Plan", "Monitoring & Alerting"]
    },
    {
      "id": "risks",
      "title": "Risks & Mitigations",
      "icon": "⚠️",
      "agent": "section-writer",
      "depends_on": ["architecture", "security", "scalability"],
      "prompt_context": "Identify technical risks with likelihood/impact matrix. Cover: dependency risks, performance risks, security risks, operational risks. Include concrete mitigation strategies.",
      "subsections": ["Risk Matrix", "Technical Risks", "Operational Risks", "Mitigation Strategies"]
    },
    {
      "id": "timeline",
      "title": "Timeline & Milestones",
      "icon": "📅",
      "agent": "section-writer",
      "depends_on": ["requirements", "architecture"],
      "prompt_context": "Provide phased implementation plan. Include milestones with dates, team size/allocation, dependencies between phases, definition of done for each phase.",
      "subsections": ["Phases", "Milestones", "Resource Allocation", "Dependencies"]
    }
  ]
}
```

Also create abbreviated templates for RFC and ADR formats (fewer sections, different focus).

-----

## PART 5: COMPLETE WORKFLOW SEQUENCE

Here is the exact sequence of events when a user clicks “Generate”:

```
1. User clicks "Generate Document" on Configure page
2. Angular sends POST /api/generate with config
3. Backend creates job directory: output/<jobId>/
4. Backend returns { jobId } → Angular navigates to /generate/<jobId>
5. Angular opens EventSource to GET /api/status/<jobId>

--- PHASE 1: ANALYZE (5-15% progress) ---
6. Backend spawns: bash analyzer.sh requirements.txt template.json analysis.json
7. analyzer.sh echoes: [AGENT:analyzer] [STATUS:running] Parsing requirements...
8. Backend captures stdout → emits 'log' event → SSE → Angular log viewer
9. analyzer.sh calls Gemini CLI, writes analysis.json
10. analyzer.sh echoes: [AGENT:analyzer] [STATUS:complete] Analysis done
11. Backend emits status: { phase: 'analyzing', progress: 15 }

--- PHASE 2: WRITE + REVIEW SECTIONS (15-85% progress) ---
12. Backend resolves dependency graph → determines Round 1 sections (no deps)
13. For each ready section IN PARALLEL:
    a. Backend spawns: bash section-writer.sh <id> context.json analysis.json deps/ section.md
    b. Writer echoes progress → log events stream to UI
    c. Writer calls Gemini CLI, writes sections/<id>.md
    d. Backend spawns: bash reviewer.sh sections/<id>.md requirements.txt reviewed/<id>.md
    e. Reviewer echoes progress → log events stream to UI
    f. Reviewer calls Gemini CLI, writes reviewed/<id>.md
    g. Backend marks section complete, emits status update
14. Backend proceeds to Round 2 (sections whose deps are now all complete)
15. Repeat until all sections done
16. Progress increments proportionally: 15 + (completed/total * 70)

--- PHASE 3: MERGE (85-95% progress) ---
17. Backend spawns: bash merger.sh reviewed/ template.json metadata.json final-document.md
18. Merger echoes progress → log events stream to UI
19. Merger calls Gemini CLI, writes final-document.md

--- PHASE 4: POST-PROCESS (95-100%) ---
20. Backend optionally converts MD → DOCX using pandoc
21. Backend emits: { type: 'complete', data: { jobId, downloadUrl } }
22. Angular navigates to /result/<jobId>
23. User downloads the document
```

-----

## PART 6: ERROR HANDLING & EDGE CASES

Implement these safeguards:

1. **Agent timeout**: 5 min per agent call. On timeout, kill process, mark section as error, allow user to retry that section.
1. **Gemini CLI failure**: If exit code != 0, capture stderr, emit error log, retry once with exponential backoff.
1. **Circular dependency**: Detect in orchestrator before starting. Return 400 error if template has circular deps.
1. **Empty output**: If an agent produces empty output file, retry once. If still empty, use a fallback prompt asking for a shorter version.
1. **SSE reconnection**: Angular should auto-reconnect EventSource on disconnect with exponential backoff. On reconnect, request full current state.
1. **Concurrent jobs**: Support multiple simultaneous jobs. Each job gets its own orchestrator instance and working directory.
1. **Large requirements**: If requirements > 10KB, summarize first using analyzer before passing to section writers (to stay within context limits).

-----

## PART 7: IMPLEMENTATION ORDER

Build in this exact order:

1. **Backend shell scripts** (analyzer.sh, section-writer.sh, reviewer.sh, merger.sh) — test each independently from command line first
1. **Orchestrator** — test with mock shell scripts that echo fake logs and sleep
1. **Express server with SSE** — test with curl: `curl -N http://localhost:3001/api/status/test`
1. **Angular project scaffold** — `ng new frontend --style=scss --routing`
1. **Angular services** (DocForgeService, LogStreamService)
1. **Configure page** — form + section list with drag-drop
1. **Generate page** — progress bar + section cards + log viewer
1. **Result page** — download buttons
1. **Integration testing** — end to end with real Gemini CLI calls
1. **Polish** — animations, error states, loading states, responsive design

-----

## PART 8: GEMINI CLI SPECIFICS

Adjust these based on your actual Gemini CLI installation:

```bash
# Headless invocation pattern (verify exact flags with: gemini --help)
# Option A: Pipe prompt via stdin
echo "$PROMPT" | gemini -p

# Option B: Pass prompt as argument
gemini -p "$PROMPT"

# Option C: Read from file
gemini -p "$(cat prompt.txt)"

# Common flags you may need:
# --model gemini-2.5-pro       (specify model)
# --output-format text         (plain text output, no markdown fencing)
# --temperature 0.3            (lower for more consistent technical writing)
# --max-output-tokens 8192     (increase for long sections)
```

If Gemini CLI doesn’t support direct piping, create a temporary prompt file per invocation:

```bash
PROMPT_FILE=$(mktemp)
cat > "$PROMPT_FILE" << 'PROMPT_EOF'
Your prompt here...
PROMPT_EOF
gemini -p "$(cat "$PROMPT_FILE")"
rm -f "$PROMPT_FILE"
```

-----

## SUMMARY OF WHAT TO BUILD

|Component            |Technology              |Key Feature                           |
|---------------------|------------------------|--------------------------------------|
|4 shell agent scripts|Bash + Gemini CLI       |Structured stdout logging             |
|Orchestrator         |Node.js EventEmitter    |Dependency-aware parallel execution   |
|API Server           |Express.js              |SSE streaming for real-time logs      |
|Configure Page       |Angular + Material + CDK|Drag-reorder sections, template picker|
|Generate Page        |Angular + EventSource   |Live log viewer with agent filters    |
|Result Page          |Angular                 |Download MD/DOCX                      |
|3 document templates |JSON                    |Technical Design, RFC, ADR            |

Build all of this as a production-quality application with proper error handling, TypeScript types, Angular best practices (standalone components, signals, OnPush change detection), and a polished dark-themed UI using Angular Material.

--------------------

# Hello_world
My first project
BM is a three tier application on GCP that has a data persistence layer, a business logic layer and a user experience layer. The goal is to manage the budget lifecycle across one more annual operating plans.

Annual operating plans definition - Annual operating plan (AOP) has a header and detail content. The header is represented by a name, total approved amount and has the states draft, active and EOL. There can only be one active AOP at any given time. The user should be able to set the lifecycle of an AOP. You can reset a draft AOP to active or an EOL AOP to active. When the AOP is set to active, we should ensure that the total amounts across active budgets cannot exceed the total amount in active AOP. If there is no lifecycle stated, the default lifecycle will be draft when a new AOP is created. The detail will have the AOP header reference, cost center and the amount for the cost center. When the AOP detail is updated, the total amount of AOP should be calculated and updated for the header so that the total amount is easily accessible. Once an AOP is in  the active state, no modification to the AOP amounts are permitted unless the process that is used to update the AOP is through a special process called Adjusted Plan. The specifics of Adjusted Plan will be discussed later in the doc. We will have CRUD APIs for AOP

Cost center - We will have a master list of cost centers which will be shown by cost center code and cost center name. We will have CRUD APIs for Cost Centers

Employees - Employees are users of this application. A user will be represented by LDAP, first name, last name, email, level, cost center code. We will also have an organization hierarchy that will have the manager employee relationship for the organization. The level for employees will be from 1 through 12. We will have CRUD APIs for employees and organization hierarchy

Budgets - A budget is a line item of spend for an employee. Every employee can request for budget for themselves or for an employee within their org if they have reporting structure per the organization hierarchy. For every budget line we will assign a unique identifier. All budgets are associated with an AOP. The budget can be active or inactive should the user want to remove a budget. The total amounts across active budgets in an AOP cannot exceed the total amount of all active budgets. For every budget line entry, the user will provide a project, a short description, the amount and the ldap of the employee responsible for budget. If the employee is not a manager, then the default is themselves and not be able to set budget for others. If the user is a manager, they should able to set budget to one of their employee within their organization. A users cannot set budget for employees who are not within their organization. For every budget line, we will also have purchase requisition amount, purchase order amount and receipt amount. These amounts are typically referred to as actuals. The actuals cannot exceed the budget. We will have APIs to update actuals. The actuals update will be incremental meaning, we will get a purchase request to add an amount of say $30. All amounts are in us dollars. Here is the information we will collect for actuals:
Purchase requests: purchase requisition reference, requestor ldap, budget identifier (should be valid against the budgets we created), requisition amount, date
Purchase orders: purchase order number, purchase order line number, requestor ldap, budget identifier (should be valid against the budgets we created), purchase item, amount, date
Receipts will only have receipt date, purchase order number, purchase order line number, purchase item and will be matched to ensure that receipts are made against purchase orders
We will have CRUD APIs for budgets, purchase requests, purchase orders and receipts. When any of the actuals are update, we will summarize the totals for that budget id and update the budget. When updating budget, if any of the amounts exceed budget, we will raise and error and fail the API or show the user an error if they are using the front end

Budget management features: Here are all the capabilities that I would the user to be able to do:
Copy budget: User should be able to specify a budget identifier and the target AOP to copy a budget to. When copying the budget, a new budget identifier should be generated in the destination AOP. If the destination AOP is an active AOP, it is important to ensure that the total budgets within AOP cannot exceed the amount on AOP
Remove budget: The database should never lose a budget. Removed of budget is a flag in database
Reconcile budget: At any time, the user should be able to run a reconciliation process where the amount on AOP is compared with the totals on active budgets and ensure that the budget for the AOP comply
Reduce budget: If the user wants to reduce the amount in a budget, it cannot be reduce below the amounts of active actuals

User experience: The user interface layer will be a chat interface versus a traditional forms based display. The user should be able to provide commands to the chatbot per the following:
Add user: This conversation should support adding a user. All mandatory fields should be prompted for if not provided by the user. For example, the user should be able to say add user lsamuel (ldap) and the chatbot should prompt to user to provide first name, last name, email, level and cost center code. If the user provides the command as add user with ldap lsamuel, first name Samuel, last name Larkins, then the chatbot prompt for email, level and cost center code. If there is an add user for a previously removed user, just mark the ldap active 
Remove user: No removes should be done in the database. Removed users should be marked inactive. An active user with an active budget within an active AOP cannot be removed. All other users can be removed
Similar rules apply for all transactions
Charting: User should be able to ask for bar chart of budgets (paretto top 10 etc) for an AOP
Non budget queries: If the user ask queries not related to budget, automatically process the same based on standard LLM app. For example, what is the weather in San Francisco.

Website: The UX will be a website UX. Should be accessible from open internet.

Authentication: There is no active sign-on for this application. If the user enters “IKnowYou241202”, the system should know that they are a valid user and allow for subsequent operations. If the user does not enter this code, then the error should be displayed to state the they are not an authenticated user.

Reporting: The user should be able to report data fairly easily per the following:
Show me my organization: Show the organization hierarchy for the user. Note that if a person A reports to B and B reports to C, when the user C says show me my organization, the report should show the hierarchy of A to B and B to C. If C has D and E reporting to them, that should also be shown
Show me my budget: Show the budgets totals for the user. This should only include the total budgets across C, D and E per the above example. If D and E have additional reports, then those should be automatically included as well

The application will be a 3 tier application on GCP. The total number of users is only 10. We expect to manage about 1000 budget lines in total. The cost across all GCP services should not exceed $50. The code repository should be github. Please include setup instructions for github, GCP and provide CI/CD instruction for object creation. The business logic can be in python or Java. Provide code base, deployment instructions, functional test cases for this.
