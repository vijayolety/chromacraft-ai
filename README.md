# ChromaCraft AI

ChromaCraft AI is a scalable, identity-preserving AI agent pipeline for automated product image-to-multi-image generation, batch processing, and video synthesis. Tailored for enterprise product catalogs and digital marketing, ChromaCraft AI offers automated background removal, style-controlled generative recoloring, 360-degree spin generation, dynamic video synthesis, and targeted lifestyle integration.

Using a hybrid architecture that couples a Next.js frontend and Node.js orchestrator with high-performance Python image processing subprocesses, ChromaCraft AI implements a closed-loop ReAct agent system. The agent validates generated image assets in real time using structural and semantic quality metrics (SSIM, CLIP, DINOv2) to provide automated self-correction before final QA delivery.

---

## 🏗️ System Architecture

ChromaCraft AI is built on a modular monorepo architecture leveraging `npm` workspaces. 

```
                    +------------------------------------+
                    |        Next.js App Router          |
                    |  (Dashboard, Job Setup, QA UI)     |
                    +-----------------+------------------+
                                      |
                             Polled API Routes
                                      |
                                      v
+------------------+        +--------------------+        +--------------------+
|  PostgreSQL DB   |<-------|    Prisma Client   |<-------|  BullMQ Job Queue  |
|  (User, Jobs,    |        |    (Shared DB)     |        |   (Redis-backed)   |
|  Assets, Events) |        +--------------------+        +--------------------+
+------------------+                                                 |
                                                                     v
                                                          +--------------------+
                                                          |  Node.js Worker    |
                                                          |  (Orchestration)   |
                                                          +----------+---------+
                                                                     |
                                                              Spawns Process
                                                                     v
                                                          +--------------------+
                                                          |   Python Scripts   |
                                                          | (Pillow, rembg,    |
                                                          |  Google GenAI SDK, |
                                                          |   Stability API)   |
                                                          +----------+---------+
                                                                     |
                                                             Stores Assets
                                                                     v
                                                          +--------------------+
                                                          |   Local Storage    |
                                                          |  (/app/storage)    |
                                                          +--------------------+
```

### Component Responsibilities
*   **Next.js Web App (`apps/web`):** Serves the UI dashboard, manages authentication via NextAuth, exposes REST API endpoints for job configuration, handles user file uploads, and provides file export download routes.
*   **Node.js Worker (`apps/worker`):** Listens to BullMQ queues, orchestrates job lifecycles, runs the ReAct feedback loop, coordinates database updates, and schedules collateral generation tasks.
*   **Python Subprocesses (`apps/worker/python`):** Owns heavy-duty tasks including API client interactions (Stability AI and Google Gemini), structural composition, zero-cost HSL color-shifting, background removal, and multimedia processing (grid creation, 360 frame generation, and video compiling).
*   **Shared DB Package (`packages/db`):** Provides a single source of truth for the database schema, Prisma client generation, and schema seeds.

---

## 🌟 Key Workflows

### 🏎️ UC1: Product Catalog Color Variant Pipeline
Generates precise color variants of a product while preserving structural shape and textures.
1.  **Upload:** User uploads an original product image (e.g., a car).
2.  **Generate:** Pipeline generates 12 color variants (*White, Black, Blue, Red, Green, Brown, Silver, Yellow, Cream, Pink, Dark Blue, Orange*).
3.  **Process:** Automatically removes backgrounds using `rembg` and normalizes resolutions.
4.  **Collate:** Arranges all passed variants into a neat, professionally branded catalog grid image with color tags and custom watermarks.
5.  **Export:** Packages raw, processed, and grid assets into a downloadable ZIP file.

### 🛵 UC2: Product Lifestyle Scene Pipeline
Integrates products into realistic context-specific marketing environments and compiles video promotions.
1.  **Upload:** User uploads a product image (e.g., a scooter).
2.  **Generate Scenes:** Google Gemini places the product naturally into 3 distinct scenes based on target audience, market, and purpose.
3.  **Preserve Structure:** Employs segmentation masks to ensure the product's design, details, and shape remain completely unaffected by the generated environment.
4.  **360 Spin:** Generates multi-angle views of the product to assemble interactive 360-degree spin frames.
5.  **Showcase Video:** Compiles either an MP4 showcase video (stitching together 360 spin frames) or a Ken-Burns style zoom animation.
6.  **Export:** Packages the final catalog, lifestyle scenes, spin frames, and MP4 video in a structured ZIP format.

---

## 🛠️ Closed-Loop ReAct Agent & Quality Gates

To guarantee client-ready product images, ChromaCraft AI implements an automated validation loop that reviews outputs before they reach the user:
```
           +-------------------------------------------+
           |           Plan Prompt & Generate          |
           +---------------------+---------------------+
                                 |
                                 v
           +---------------------+---------------------+
           |         Evaluate Quality Metrics          |
           |    (SSIM + CLIP + DINOv2 Alignment)       |
           +---------------------+---------------------+
                                 |
                        [Score >= Threshold?]
                       /                     \
                     Yes                      No (Attempt < Max)
                     /                         \
                    v                           v
     +-----------------------+     +-----------------------+
     |   Mark Color Passed   |     | Generate Critique &   |
     |   & Proceed to QA     |     | Refine Prompt (Retry) |
     +-----------------------+     +-----------------------+
```

*   **Semantic Matching (CLIP):** Confirms the generated image matches the prompt and color requirements.
*   **Structural Alignment (DINOv2 & SSIM):** Ensures the product's shape, proportions, reflections, and orientation remain identical to the original reference.
*   **Critique & Self-Correction:** If the composite score falls below the threshold (default: `0.92`), the agent writes a detailed critique, refines the prompt, and retries the generation. The worker permits up to 2 attempts before marking a variant as failed.

---

## 💻 Tech Stack

| Domain | Technology | Description |
|---|---|---|
| **Frontend Framework** | [Next.js 15](https://nextjs.org/) | React-based server and client app with App Router. |
| **Styling & UI** | [TailwindCSS](https://tailwindcss.com/) & [shadcn/ui](https://ui.shadcn.com/) | Curated dark-themed layout, smooth micro-animations. |
| **State Management** | [Zustand](https://github.com/pmndrs/zustand) | Lightweight reactive client-side state. |
| **Data Fetching** | [React Query](https://tanstack.com/query/latest) | Asynchronous client-side data cache and polling queries. |
| **Authentication** | [NextAuth.js](https://next-auth.js.org/) | Secure user session management via Credentials provider. |
| **Database ORM** | [Prisma](https://www.prisma.io/) | Schema management, type safety, and migrations. |
| **Database** | [PostgreSQL 15](https://www.postgresql.org/) | Persistent relational storage. |
| **Queue Broker** | [Redis 7](https://redis.io/) | Message broker powering BullMQ jobs. |
| **Queue Framework** | [BullMQ](https://bullmq.io/) | Robust Node.js job queue with concurrency limits. |
| **Image Processing** | [Pillow](https://python-pillow.org/) & [rembg](https://github.com/danielgatis/rembg) | Image crop, resize, grid layouts, and background removal. |
| **Generative APIs** | [Stability AI](https://stability.ai/) & [Google Gemini](https://ai.google.dev/) | High-fidelity image manipulation, ControlNet, and GenAI SDK. |

---

## 📂 Project Structure

```
chromacraft-ai/
├── apps/
│   ├── web/                    # Next.js 15 Frontend and API handlers
│   │   ├── app/                # Pages and API endpoints
│   │   │   ├── api/v1/         # JSON REST API routes
│   │   │   └── globals.css     # Styling core variables
│   │   ├── components/         # Page components (auth, dashboard, workflow)
│   │   │   └── workflow/       # Setup, Generation, QA, and Delivery panels
│   │   └── lib/                # Shared front-end helper classes
│   └── worker/                 # Node.js Background Job Consumer
│       ├── python/             # Python CLI scripts spawned by Node
│       │   ├── generate.py     # Stability & Gemini client router
│       │   ├── identity.py     # Masking and composition mechanics
│       │   ├── quality.py      # CLIP, SSIM, and DINOv2 validation
│       │   ├── grid.py         # Grid builder script
│       │   ├── multiview.py    # 360-degree rotation engine
│       │   └── video.py        # Video compiler (showcase MP4s)
│       ├── worker.ts           # BullMQ worker loops
│       └── orchestrator.ts     # ReAct orchestrator class
├── packages/
│   └── db/                     # Shared Database configuration
│       ├── prisma/             # Schema, migrations, and seeds
│       └── src/                # Shared Prisma client exports
├── storage/                    # Local filesystem persistent storage volume
├── docker-compose.yml          # Container configuration
├── package.json                # Monorepo workspaces definition
└── .env.example                # Sample environment config
```

---

## 🚀 Getting Started

### Prerequisites
Make sure you have the following installed on your system:
*   [Docker](https://www.docker.com/) and Docker Compose
*   [Node.js](https://nodejs.org/) (v20+ recommended)
*   [Python 3.10+](https://www.python.org/) (for native local execution)

---

### Environment Variables

Create a `.env` file in the root directory. You can copy the contents from `.env.example` or write them manually:

```env
# Database Settings (Prisma & Postgres)
DATABASE_URL="postgresql://postgres:postgrespw@localhost:5432/chromacraft?schema=public"

# Redis Configuration (BullMQ backend)
REDIS_URL="redis://default:redispw@localhost:6379"
REDIS_HOST="localhost"
REDIS_PORT=6379
REDIS_PASSWORD="redispw"

# NextAuth Authentication Config
NEXTAUTH_URL="http://localhost:3000"
NEXTAUTH_SECRET="supersecretkeyfortestingpurposesonly"

# AI Service Provider Credentials
# (Required for production AI pipelines. Fallback to mock logic in Dev Mode)
CHROMACRAFT_API_KEY="your-stability-api-key"
GEMINI_API_KEY="your-google-gemini-api-key"

# System Settings
STORAGE_PATH="./storage"
DEVELOPMENT_MODE=true
QUALITY_THRESHOLD=0.92
COLOR_CONCURRENCY=3
```

---

### Method 1: Docker Compose (Recommended)

To launch the database, redis cache, minio storage, web interface, and queue worker all together in containers, run:

```bash
docker compose up --build
```

The system will start, automatically run database migrations, seed default users, and become accessible at:
*   **Web App:** [http://localhost:3000](http://localhost:3000)
*   **Minio Console:** [http://localhost:9001](http://localhost:9001) (Credentials: `minioadmin` / `minioadmin123`)

---

### Method 2: Native Local Setup

For faster feedback loops during active development, you can run services directly on your host machine.

#### 1. Setup the Shared Database
Install monorepo dependencies and set up the database using Prisma:
```bash
# Install root package workspaces
npm install

# Run migration and generate client
npm run prisma:migrate
npm run prisma:generate
```

#### 2. Install Python Dependencies
Create a virtual environment inside the worker's python directory and install required libraries:
```bash
cd apps/worker/python
python -m venv venv
# On Windows:
venv\Scripts\activate
# On Linux/macOS:
source venv/bin/activate

pip install -r requirements.txt
cd ../../..
```

#### 3. Run Infrastructure Containers
You can start just PostgreSQL and Redis using Docker Compose:
```bash
docker compose up -d postgres redis
```

#### 4. Run Applications
Open two terminals to run the Web server and Worker worker process simultaneously:

**Terminal 1 (Web Interface):**
```bash
npm run dev
# Starts Next.js app on http://localhost:3000
```

**Terminal 2 (Background Worker):**
```bash
npm run dev:worker
# Starts typescript tsc compiler and BullMQ listeners
```

---

## 🔑 Default Credentials

A default administrator account is seeded automatically on database initialization. You can use it to log in:
*   **Email:** `admin@example.com`
*   **Password:** `admin123`

---

## 🔌 API Reference

| Endpoint | Method | Description |
|---|---|---|
| `/api/v1/auth/session` | `GET` | Retrieve active authentication session info. |
| `/api/v1/jobs` | `POST` | Create a new image processing job. |
| `/api/v1/jobs` | `GET` | List all historical jobs with statuses and metadata. |
| `/api/v1/jobs/[id]` | `GET` | Retrieve detailed status, assets, and event logs for a specific job. |
| `/api/v1/upload` | `POST` | Upload original product reference images. |
| `/api/v1/qa/approve` | `POST` | Manually approve a generated variant to proceed to packaging. |
| `/api/v1/qa/reject` | `POST` | Reject a variant and provide custom critique feedback for regeneration. |
| `/api/v1/export` | `POST` | Bundle a job's approved assets into a structured ZIP file. |

---

## 🛠️ Development & Debugging Tips

*   **Mock Outputs (`DEVELOPMENT_MODE=true`):** In local development, enable `DEVELOPMENT_MODE=true` in your `.env`. This skips actual API requests to Stability and Gemini (saving credits) and generates solid color canvas placeholders to quickly test the end-to-end front-end and queue lifecycle.
*   **Worker Diagnostics:** View worker activity directly via stdout logs, which output Pino-formatted JSON messages showing current queue jobs, python subprocess spawns, and database updates.
*   **Stale Job Recovery:** On startup, the worker scans the database for jobs stuck in a `PROCESSING` status for over 30 minutes, automatically failing them and logging a recovery event to prevent deadlocks.
