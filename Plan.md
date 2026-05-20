# AutoPostMediaProject — Technical Specification

> **Project:** Social Media Auto-Poster for Instagram, Facebook & Threads  
> **Version:** 1.0.0 — MVP Specification  
> **Date:** 2026-05-21  
> **Author:** Claude Code (AutoPostMediaProject Team)

---

## 1. Project Overview and Goals

### What We're Building
A marketing-focused social media automation tool that empowers users to create, schedule, and publish content across Instagram Business, Facebook Pages, and Threads from a single dashboard.

### Core Goals
1. **Unified Publishing** — Post to multiple platforms simultaneously or staggered
2. **AI-Assisted Creation** — Generate captions, hashtags, and images using OpenAI
3. **Template-Based Design** — Reusable post templates with drag-and-drop media
4. **Smart Scheduling** — Queue-based scheduling with timezone awareness
5. **UGC Collection** — Curate and republish user-generated content
6. **Basic Analytics** — Track post status and platform-level performance

### Target Users
- Small business owners managing their own social presence
- Marketing agencies handling multiple client accounts
- Content creators publishing across platforms daily

### Success Metrics (MVP)
- User can connect ≥2 social accounts
- Post scheduling with 99%+ queue reliability
- AI caption generation <5s response time
- End-to-end publish flow (draft → scheduled → published) works for all 3 platforms

---

## 2. Tech Stack with Versions

| Layer | Technology | Version | Purpose |
|-------|-----------|---------|---------|
| **Frontend** | React | 18.3.x | UI framework |
| | Vite | 5.2.x | Build tool & dev server |
| | Tailwind CSS | 3.4.x | Utility-first styling |
| | React Router | 6.23.x | Client-side routing |
| | TanStack Query | 5.36.x | Server state management |
| | Axios | 1.6.x | HTTP client |
| | date-fns | 3.6.x | Date/time manipulation |
| | react-dropzone | 14.2.x | File upload drag-and-drop |
| | zustand | 4.5.x | Client state management |
| **Backend** | Node.js | 20.12.x LTS | Runtime |
| | Express | 4.19.x | Web framework |
| | TypeScript | 5.4.x | Type safety |
| | Prisma | 5.14.x | ORM & migrations |
| | bcrypt | 5.1.x | Password hashing |
| | jsonwebtoken | 9.0.x | JWT auth tokens |
| | zod | 3.23.x | Schema validation |
| | helmet | 7.1.x | Security headers |
| | cors | 2.8.x | Cross-origin requests |
| | morgan | 1.10.x | Request logging |
| | dotenv | 16.4.x | Environment config |
| **Database** | PostgreSQL | 15.x | Primary database |
| | Redis | 7.2.x | Queue broker & caching |
| **Queue** | BullMQ | 5.7.x | Job queue & scheduling |
| **Storage** | Cloudinary | 2.x | Media upload & transformation |
| **AI** | OpenAI SDK | 4.47.x | GPT-4, DALL-E 3 |
| **Testing** | Vitest | 1.6.x | Unit & integration tests |
| | Playwright | 1.44.x | E2E testing |
| | Supertest | 7.0.x | HTTP assertion library |
| **DevOps** | Docker | 25.x | Containerization |
| | Docker Compose | 2.27.x | Local orchestration |
| | GitHub Actions | — | CI/CD pipeline |

---

## 3. Database Schema (Full SQL)

### 3.1 Users
```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  display_name VARCHAR(100),
  avatar_url TEXT,
  timezone VARCHAR(50) DEFAULT 'UTC',
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
```

### 3.2 Social Accounts (Encrypted API Keys)
```sql
CREATE TABLE social_accounts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  platform VARCHAR(20) NOT NULL CHECK (platform IN ('instagram', 'facebook', 'threads')),
  account_name VARCHAR(100) NOT NULL,
  account_id VARCHAR(100) NOT NULL,
  encrypted_app_id TEXT NOT NULL,
  encrypted_app_secret TEXT NOT NULL,
  encrypted_access_token TEXT NOT NULL,
  token_type VARCHAR(20) NOT NULL DEFAULT 'oauth_access',
  scopes TEXT[] DEFAULT '{}',
  expires_at TIMESTAMPTZ,
  is_active BOOLEAN DEFAULT true,
  last_validated_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(user_id, platform, account_id)
);

CREATE INDEX idx_social_accounts_user ON social_accounts(user_id);
CREATE INDEX idx_social_accounts_platform ON social_accounts(platform);
```

### 3.3 Posts
```sql
CREATE TABLE posts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  title VARCHAR(200),
  caption TEXT,
  ai_generated_caption BOOLEAN DEFAULT false,
  status VARCHAR(20) NOT NULL DEFAULT 'draft'
    CHECK (status IN ('draft', 'scheduled', 'pending', 'publishing', 'published', 'failed', 'cancelled')),
  scheduled_at TIMESTAMPTZ,
  published_at TIMESTAMPTZ,
  failed_reason TEXT,
  template_id UUID REFERENCES templates(id),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_posts_user ON posts(user_id);
CREATE INDEX idx_posts_status ON posts(status);
CREATE INDEX idx_posts_scheduled ON posts(scheduled_at) WHERE status = 'scheduled';
```

### 3.4 Post Media
```sql
CREATE TABLE post_media (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  post_id UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  media_type VARCHAR(10) NOT NULL CHECK (media_type IN ('image', 'video', 'carousel')),
  url TEXT NOT NULL,
  thumbnail_url TEXT,
  cloudinary_public_id VARCHAR(255),
  file_size_bytes INTEGER,
  width INTEGER,
  height INTEGER,
  sort_order INTEGER DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_post_media_post ON post_media(post_id);
```

### 3.5 Post Platforms (Per-Platform Status)
```sql
CREATE TABLE post_platforms (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  post_id UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  social_account_id UUID NOT NULL REFERENCES social_accounts(id) ON DELETE CASCADE,
  platform VARCHAR(20) NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'pending'
    CHECK (status IN ('pending', 'publishing', 'published', 'failed', 'cancelled')),
  external_post_id VARCHAR(255),
  external_url TEXT,
  published_at TIMESTAMPTZ,
  failed_reason TEXT,
  retry_count INTEGER DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(post_id, social_account_id)
);

CREATE INDEX idx_post_platforms_post ON post_platforms(post_id);
CREATE INDEX idx_post_platforms_account ON post_platforms(social_account_id);
CREATE INDEX idx_post_platforms_status ON post_platforms(status);
```

### 3.6 Templates
```sql
CREATE TABLE templates (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  name VARCHAR(100) NOT NULL,
  description TEXT,
  category VARCHAR(50),
  background_color VARCHAR(7) DEFAULT '#ffffff',
  text_color VARCHAR(7) DEFAULT '#000000',
  font_family VARCHAR(50) DEFAULT 'Inter',
  font_size INTEGER DEFAULT 24,
  overlay_text TEXT,
  overlay_position VARCHAR(20) DEFAULT 'center',
  overlay_opacity DECIMAL(3,2) DEFAULT 0.8,
  border_radius INTEGER DEFAULT 8,
  shadow_enabled BOOLEAN DEFAULT true,
  is_public BOOLEAN DEFAULT false,
  usage_count INTEGER DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_templates_user ON templates(user_id);
CREATE INDEX idx_templates_category ON templates(category);
```

### 3.7 Scheduled Jobs (Queue Metadata)
```sql
CREATE TABLE scheduled_jobs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  post_id UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  bullmq_job_id VARCHAR(100) UNIQUE,
  queue_name VARCHAR(50) NOT NULL DEFAULT 'social-publish',
  status VARCHAR(20) NOT NULL DEFAULT 'queued'
    CHECK (status IN ('queued', 'active', 'completed', 'failed', 'delayed', 'cancelled')),
  scheduled_for TIMESTAMPTZ NOT NULL,
  processed_at TIMESTAMPTZ,
  error_message TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_scheduled_jobs_post ON scheduled_jobs(post_id);
CREATE INDEX idx_scheduled_jobs_status ON scheduled_jobs(status);
CREATE INDEX idx_scheduled_jobs_queue ON scheduled_jobs(queue_name, status);
```

### 3.8 UGC Submissions
```sql
CREATE TABLE ugc_submissions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  source_platform VARCHAR(20),
  source_url TEXT,
  caption TEXT,
  media_url TEXT NOT NULL,
  media_type VARCHAR(10) DEFAULT 'image',
  contributor_name VARCHAR(100),
  contributor_handle VARCHAR(100),
  contributor_email VARCHAR(255),
  status VARCHAR(20) DEFAULT 'pending'
    CHECK (status IN ('pending', 'approved', 'rejected', 'published')),
  post_id UUID REFERENCES posts(id),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_ugc_user ON ugc_submissions(user_id);
CREATE INDEX idx_ugc_status ON ugc_submissions(status);
```

---

## 4. API Endpoints List

### 4.1 Authentication (`/api/auth`)
| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| POST | `/api/auth/register` | Create new account | Public |
| POST | `/api/auth/login` | Login, returns JWT | Public |
| POST | `/api/auth/logout` | Invalidate token | Bearer |
| GET | `/api/auth/me` | Get current user | Bearer |
| POST | `/api/auth/refresh` | Refresh access token | Bearer |

### 4.2 Social Accounts (`/api/accounts`)
| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| GET | `/api/accounts` | List connected accounts | Bearer |
| POST | `/api/accounts` | Add new account (encrypted) | Bearer |
| GET | `/api/accounts/:id` | Get account details | Bearer |
| PUT | `/api/accounts/:id` | Update account | Bearer |
| DELETE | `/api/accounts/:id` | Remove account & purge keys | Bearer |
| POST | `/api/accounts/:id/validate` | Test token validity | Bearer |
| POST | `/api/accounts/:id/refresh` | Refresh OAuth token | Bearer |

### 4.3 Posts (`/api/posts`)
| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| GET | `/api/posts` | List posts (paginated, filterable) | Bearer |
| POST | `/api/posts` | Create new post | Bearer |
| GET | `/api/posts/:id` | Get post with media & platforms | Bearer |
| PUT | `/api/posts/:id` | Update post content | Bearer |
| DELETE | `/api/posts/:id` | Delete post & cancel jobs | Bearer |
| POST | `/api/posts/:id/schedule` | Schedule for publishing | Bearer |
| POST | `/api/posts/:id/publish-now` | Immediate publish | Bearer |
| POST | `/api/posts/:id/cancel` | Cancel scheduled post | Bearer |
| POST | `/api/posts/:id/duplicate` | Clone post as draft | Bearer |

### 4.4 Media Upload (`/api/media`)
| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| POST | `/api/media/upload` | Upload image/video to Cloudinary | Bearer |
| DELETE | `/api/media/:id` | Delete media from storage | Bearer |

### 4.5 AI Generation (`/api/ai`)
| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| POST | `/api/ai/caption` | Generate caption from topic | Bearer |
| POST | `/api/ai/hashtags` | Generate hashtags from caption | Bearer |
| POST | `/api/ai/image` | Generate image with DALL-E 3 | Bearer |
| POST | `/api/ai/moderate` | Check content for policy violations | Bearer |
| POST | `/api/ai/full-post` | Caption + hashtags in one call | Bearer |

### 4.6 Templates (`/api/templates`)
| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| GET | `/api/templates` | List templates | Bearer |
| POST | `/api/templates` | Create template | Bearer |
| GET | `/api/templates/:id` | Get template | Bearer |
| PUT | `/api/templates/:id` | Update template | Bearer |
| DELETE | `/api/templates/:id` | Delete template | Bearer |
| POST | `/api/templates/:id/apply` | Apply template to a post | Bearer |

### 4.7 UGC (`/api/ugc`)
| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| GET | `/api/ugc` | List UGC submissions | Bearer |
| POST | `/api/ugc` | Submit new UGC | Bearer |
| PUT | `/api/ugc/:id/approve` | Approve for use | Bearer |
| PUT | `/api/ugc/:id/reject` | Reject submission | Bearer |
| POST | `/api/ugc/:id/convert` | Convert to draft post | Bearer |

### 4.8 Analytics (`/api/analytics`)
| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| GET | `/api/analytics/dashboard` | Summary stats | Bearer |
| GET | `/api/analytics/posts` | Post performance list | Bearer |
| GET | `/api/analytics/posts/:id` | Single post breakdown | Bearer |
| GET | `/api/analytics/platforms` | Per-platform stats | Bearer |

### 4.9 Webhooks (`/webhooks`)
| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| POST | `/webhooks/meta` | Meta real-time updates | HMAC |

---

## 5. Frontend Component Structure

```
frontend/
├── public/
│   └── favicon.ico
├── src/
│   ├── main.tsx                    # Entry point
│   ├── App.tsx                     # Root router
│   ├── index.css                   # Tailwind directives
│   │
│   ├── api/                        # Axios instances & interceptors
│   │   ├── client.ts               # Base axios with auth header
│   │   ├── auth.ts                 # Auth endpoints
│   │   ├── posts.ts                # Post endpoints
│   │   ├── accounts.ts             # Social account endpoints
│   │   ├── media.ts                # Upload endpoints
│   │   ├── ai.ts                   # AI generation endpoints
│   │   └── templates.ts            # Template endpoints
│   │
│   ├── components/                 # Reusable UI components
│   │   ├── ui/                     # Primitive components (Button, Input, Modal)
│   │   ├── layout/                 # Layout shells (Sidebar, TopBar, PageWrapper)
│   │   ├── media/                  # MediaUploader, MediaPreview, ImageCropper
│   │   └── forms/                  # FormField, FormError, FormLabel
│   │
│   ├── features/                   # Domain-specific feature modules
│   │   ├── auth/
│   │   │   ├── LoginForm.tsx
│   │   │   ├── RegisterForm.tsx
│   │   │   └── useAuth.ts          # Auth state & actions
│   │   │
│   │   ├── dashboard/
│   │   │   ├── DashboardStats.tsx
│   │   │   ├── RecentPosts.tsx
│   │   │   └── UpcomingSchedule.tsx
│   │   │
│   │   ├── posts/
│   │   │   ├── PostEditor.tsx      # Main post creation/editing
│   │   │   ├── PostList.tsx
│   │   │   ├── PostCard.tsx
│   │   │   ├── PostScheduler.tsx   # Date/time picker
│   │   │   ├── PostStatusBadge.tsx
│   │   │   └── PlatformSelector.tsx
│   │   │
│   │   ├── accounts/
│   │   │   ├── AccountList.tsx
│   │   │   ├── AccountCard.tsx
│   │   │   ├── AddAccountModal.tsx
│   │   │   └── AccountStatusBadge.tsx
│   │   │
│   │   ├── ai-assistant/
│   │   │   ├── AICaptionGenerator.tsx
│   │   │   ├── AIHashtagGenerator.tsx
│   │   │   ├── AIImageGenerator.tsx
│   │   │   └── AIModerationAlert.tsx
│   │   │
│   │   ├── templates/
│   │   │   ├── TemplateList.tsx
│   │   │   ├── TemplateEditor.tsx
│   │   │   └── TemplatePreview.tsx
│   │   │
│   │   ├── ugc/
│   │   │   ├── UGCList.tsx
│   │   │   ├── UGCCard.tsx
│   │   │   └── UGCApprovalActions.tsx
│   │   │
│   │   └── analytics/
│   │       ├── AnalyticsDashboard.tsx
│   │       ├── PostPerformanceChart.tsx
│   │       └── PlatformBreakdown.tsx
│   │
│   ├── hooks/                      # Shared custom hooks
│   │   ├── usePosts.ts
│   │   ├── useAccounts.ts
│   │   ├── useTemplates.ts
│   │   ├── useUpload.ts
│   │   └── useAI.ts
│   │
│   ├── stores/                     # Zustand global state
│   │   ├── authStore.ts
│   │   ├── postStore.ts
│   │   └── uiStore.ts            # Modal, toast, sidebar state
│   │
│   ├── types/                      # TypeScript interfaces
│   │   ├── auth.ts
│   │   ├── post.ts
│   │   ├── account.ts
│   │   ├── template.ts
│   │   └── api.ts
│   │
│   ├── utils/                      # Helper functions
│   │   ├── date.ts                 # Timezone-aware formatting
│   │   ├── crypto.ts               # Frontend-safe token masking
│   │   └── validators.ts           # Zod schemas shared with backend
│   │
│   └── pages/                      # Route-level page components
│       ├── LoginPage.tsx
│       ├── RegisterPage.tsx
│       ├── DashboardPage.tsx
│       ├── PostCreatePage.tsx
│       ├── PostEditPage.tsx
│       ├── PostDetailPage.tsx
│       ├── PostsPage.tsx
│       ├── AccountsPage.tsx
│       ├── TemplatesPage.tsx
│       ├── UGCPage.tsx
│       └── AnalyticsPage.tsx
│
├── index.html
├── vite.config.ts
├── tailwind.config.js
├── tsconfig.json
└── package.json
```

### Key Frontend Decisions
- **Feature-based colocation:** Each feature has its own components, hooks, and API calls
- **Server state vs client state:** TanStack Query for server state, Zustand for UI state
- **Drag-and-drop:** `react-dropzone` for media uploads, `react-beautiful-dnd` (or `@dnd-kit`) for template builder
- **Date handling:** `date-fns` with timezone support for scheduling
- **Image editing:** Canvas API for template text overlay; Cloudinary for server-side transforms

---

## 6. Backend Service Architecture

```
backend/
├── src/
│   ├── index.ts                    # Entry point — starts HTTP server
│   ├── app.ts                      # Express app setup (middleware, routes)
│   ├── config.ts                   # Environment validation & config object
│   │
│   ├── routes/                     # Route definitions
│   │   ├── auth.routes.ts
│   │   ├── accounts.routes.ts
│   │   ├── posts.routes.ts
│   │   ├── media.routes.ts
│   │   ├── ai.routes.ts
│   │   ├── templates.routes.ts
│   │   ├── ugc.routes.ts
│   │   ├── analytics.routes.ts
│   │   └── webhooks.routes.ts
│   │
│   ├── controllers/                # Request handlers
│   │   ├── auth.controller.ts
│   │   ├── accounts.controller.ts
│   │   ├── posts.controller.ts
│   │   ├── media.controller.ts
│   │   ├── ai.controller.ts
│   │   ├── templates.controller.ts
│   │   ├── ugc.controller.ts
│   │   └── analytics.controller.ts
│   │
│   ├── services/                   # Business logic layer
│   │   ├── auth.service.ts         # Registration, login, JWT
│   │   ├── accounts.service.ts     # CRUD + encryption for social accounts
│   │   ├── posts.service.ts        # Post lifecycle management
│   │   ├── publish.service.ts      # Orchestrates platform publishing
│   │   ├── media.service.ts        # Cloudinary upload/delete
│   │   ├── ai.service.ts           # OpenAI caption/image generation
│   │   ├── templates.service.ts
│   │   ├── ugc.service.ts
│   │   ├── analytics.service.ts
│   │   └── encryption.service.ts   # AES-256-GCM wrapper
│   │
│   ├── workers/                    # BullMQ worker definitions
│   │   ├── publish.worker.ts       # Main publishing worker
│   │   ├── token-refresh.worker.ts  # OAuth token refresh cron
│   │   └── cleanup.worker.ts       # Purge old failed jobs & revoked tokens
│   │
│   ├── queues/                     # BullMQ queue setup
│   │   └── publish.queue.ts
│   │
│   ├── integrations/               # Third-party API clients
│   │   ├── meta/
│   │   │   ├── instagram.client.ts # Instagram Graph API
│   │   │   ├── facebook.client.ts  # Facebook Graph API
│   │   │   ├── threads.client.ts   # Threads API
│   │   │   └── meta-base.client.ts # Shared fetch logic, rate limiting
│   │   ├── openai.client.ts
│   │   └── cloudinary.client.ts
│   │
│   ├── middleware/                 # Express middleware
│   │   ├── auth.middleware.ts      # JWT verification
│   │   ├── error.middleware.ts     # Global error handler
│   │   ├── validate.middleware.ts  # Zod schema validation
│   │   └── rate-limit.middleware.ts
│   │
│   ├── prisma/
│   │   ├── schema.prisma           # Database schema
│   │   └── migrations/             # Generated migrations
│   │
│   ├── types/                      # Shared TypeScript types
│   │   ├── express.d.ts            # Extended Request type
│   │   └── api.ts
│   │
│   └── utils/
│       ├── logger.ts               # Pino or Winston logger
│       ├── errors.ts               # Custom error classes
│       └── validators.ts           # Zod schemas
│
├── Dockerfile
├── .dockerignore
├── tsconfig.json
├── package.json
└── prisma/
    └── schema.prisma               # Alternative: Prisma schema at root
```

### Service Layer Responsibilities

| Service | Responsibility |
|---------|---------------|
| `auth.service` | Hash passwords with bcrypt, sign/verify JWTs, refresh tokens |
| `encryption.service` | AES-256-GCM encrypt/decrypt using Node crypto; master key from `process.env.MASTER_KEY` |
| `accounts.service` | CRUD social accounts; every write encrypts tokens; every read decrypts for API calls |
| `posts.service` | Draft creation, scheduling, status transitions, cancellation |
| `publish.service` | Orchestrator: for each platform in a post, enqueue a BullMQ job or publish immediately |
| `publish.worker` | BullMQ worker: pick up job, call correct platform client, update `post_platforms` status |
| `meta-base.client` | Shared HTTP client for Meta APIs with built-in rate limit tracking and exponential backoff |
| `ai.service` | Wrap OpenAI SDK; route to correct model (gpt-4o-mini vs gpt-4o); track token usage |

---

## 7. Queue/Worker System Design

### 7.1 Queue Architecture

```typescript
// queues/publish.queue.ts
import { Queue } from 'bullmq';
import IORedis from 'ioredis';

const redis = new IORedis(process.env.REDIS_URL!, {
  maxRetriesPerRequest: null,
});

export const publishQueue = new Queue('social-publish', {
  connection: redis,
  defaultJobOptions: {
    attempts: 5,
    backoff: { type: 'exponential', delay: 60_000 },
    removeOnComplete: { count: 100 },
    removeOnFail: { count: 50 },
  },
});
```

### 7.2 Publishing Worker

```typescript
// workers/publish.worker.ts
import { Worker } from 'bullmq';
import { publishToPlatform } from '../services/publish.service';

export const publishWorker = new Worker(
  'social-publish',
  async (job) => {
    const { postId, platform, socialAccountId, content, mediaUrls } = job.data;

    // Update platform status to publishing
    await updatePlatformStatus(postId, platform, 'publishing');

    try {
      const result = await publishToPlatform({
        platform,
        socialAccountId,
        content,
        mediaUrls,
      });

      await updatePlatformStatus(postId, platform, 'published', {
        externalPostId: result.id,
        externalUrl: result.permalink,
      });
    } catch (error: any) {
      await updatePlatformStatus(postId, platform, 'failed', {
        failedReason: error.message,
      });

      // Re-throw so BullMQ handles retry
      throw error;
    }
  },
  {
    connection: redis,
    concurrency: 5,
    limiter: {
      max: 10,        // Max 10 jobs per duration
      duration: 1000, // Per second (platform-specific rate limits enforced in client)
    },
  }
);
```

### 7.3 Job Scheduling Flow

```
User clicks "Schedule"
  → POST /api/posts/:id/schedule
    → posts.service.schedulePost(postId, scheduledAt)
      → Update post.status = 'scheduled'
      → For each selected platform:
        → publishQueue.add('publish-post', {
            postId, platform, socialAccountId, content, mediaUrls
          }, {
            jobId: `${postId}:${platform}`,
            delay: scheduledAt.getTime() - Date.now(),
          })
        → Insert into scheduled_jobs table
      → Return 202 Accepted

At scheduled time:
  → BullMQ moves job to active
  → Worker picks up job
  → Calls Meta/Threads API via platform client
  → Updates post_platforms row
  → On success: post.status → 'published' (if all platforms done)
  → On failure: retry up to 5× exponential backoff
```

### 7.4 Rate Limit Strategy per Platform

| Platform | Queue Strategy |
|----------|---------------|
| **Instagram** | 50 posts/24h. Use BullMQ `limiter` + track daily counter in Redis. If limit hit, delay to next window. |
| **Facebook** | 200 calls/hour. Burst-friendly; standard limiter sufficient. |
| **Threads** | 250 posts/24h. Same counter pattern as Instagram. |

```typescript
// Pseudo-code for daily rate limit gate
async function checkDailyLimit(platform: string, accountId: string): Promise<boolean> {
  const key = `rate_limit:${platform}:${accountId}:${format(new Date(), 'yyyy-MM-dd')}`;
  const count = await redis.incr(key);
  if (count === 1) await redis.expire(key, 86_400);
  const limit = platform === 'instagram' ? 50 : platform === 'threads' ? 250 : 200;
  return count <= limit;
}
```

### 7.5 Token Refresh Cron Worker

```typescript
// workers/token-refresh.worker.ts
import { Queue, Worker } from 'bullmq';

const refreshQueue = new Queue('token-refresh', { connection: redis });

// Schedule to run every 24 hours
await refreshQueue.add('refresh-all', {}, { repeat: { cron: '0 4 * * *' } });

const refreshWorker = new Worker('token-refresh', async () => {
  const accounts = await db.socialAccount.findMany({
    where: {
      expires_at: { lt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000) }, // Expires within 7 days
    },
  });

  for (const account of accounts) {
    try {
      const newToken = await refreshOAuthToken(account);
      await db.socialAccount.update({
        where: { id: account.id },
        data: { encrypted_access_token: encrypt(newToken.access_token), expires_at: newToken.expires_at },
      });
    } catch (err) {
      logger.error({ accountId: account.id, err }, 'Token refresh failed');
    }
  }
}, { connection: redis });
```

---

## 8. Security Implementation Details

### 8.1 Encryption at Rest (AES-256-GCM)

All API credentials stored in `social_accounts` are encrypted before database insertion.

```typescript
// services/encryption.service.ts
import crypto from 'crypto';

const ALGORITHM = 'aes-256-gcm';
const IV_LENGTH = 16;
const AUTH_TAG_LENGTH = 16;
const MASTER_KEY = Buffer.from(process.env.MASTER_KEY!, 'hex');

export function encrypt(plainText: string): string {
  const iv = crypto.randomBytes(IV_LENGTH);
  const cipher = crypto.createCipheriv(ALGORITHM, MASTER_KEY, iv);
  const encrypted = Buffer.concat([cipher.update(plainText, 'utf8'), cipher.final()]);
  const authTag = cipher.getAuthTag();
  return Buffer.concat([iv, authTag, encrypted]).toString('base64');
}

export function decrypt(cipherText: string): string {
  const data = Buffer.from(cipherText, 'base64');
  const iv = data.subarray(0, IV_LENGTH);
  const authTag = data.subarray(IV_LENGTH, IV_LENGTH + AUTH_TAG_LENGTH);
  const encrypted = data.subarray(IV_LENGTH + AUTH_TAG_LENGTH);
  const decipher = crypto.createDecipheriv(ALGORITHM, MASTER_KEY, iv);
  decipher.setAuthTag(authTag);
  return decipher.update(encrypted) + decipher.final('utf8');
}
```

**Master Key Management:**
- `MASTER_KEY` is a 64-character hex string (32 bytes)
- Stored in environment variable only; never committed
- For production: use AWS KMS, HashiCorp Vault, or similar
- Rotation procedure: decrypt all tokens with old key, re-encrypt with new key

### 8.2 Password Hashing

```typescript
import bcrypt from 'bcrypt';
const SALT_ROUNDS = 12;

export async function hashPassword(password: string): Promise<string> {
  return bcrypt.hash(password, SALT_ROUNDS);
}

export async function verifyPassword(password: string, hash: string): Promise<boolean> {
  return bcrypt.compare(password, hash);
}
```

### 8.3 JWT Authentication

```typescript
import jwt from 'jsonwebtoken';

const ACCESS_TOKEN_SECRET = process.env.ACCESS_TOKEN_SECRET!;
const REFRESH_TOKEN_SECRET = process.env.REFRESH_TOKEN_SECRET!;

export function signAccessToken(userId: string): string {
  return jwt.sign({ sub: userId, type: 'access' }, ACCESS_TOKEN_SECRET, { expiresIn: '15m' });
}

export function signRefreshToken(userId: string): string {
  return jwt.sign({ sub: userId, type: 'refresh' }, REFRESH_TOKEN_SECRET, { expiresIn: '7d' });
}
```

### 8.4 API Key Masking (UI)

Never display full tokens in the UI. Show only last 4 characters.

```typescript
export function maskToken(token: string): string {
  if (token.length <= 8) return '****';
  return token.slice(0, 4) + '...' + token.slice(-4);
}
```

### 8.5 Security Checklist

| Requirement | Implementation |
|-------------|---------------|
| HTTPS only | Enforce in production; HSTS headers via Helmet |
| CORS | Whitelist production domain only |
| Rate limiting | 100 requests/min per IP; stricter for auth endpoints (5/min) |
| Input validation | Zod schemas on all request bodies |
| SQL injection | Prisma ORM (parameterized queries) |
| XSS | React auto-escapes; sanitize any HTML rendered from AI |
| Content Security Policy | Helmet CSP directives |
| Secrets in env | `dotenv` for local; Docker secrets / K8s secrets for prod |
| Audit logging | Log all credential access and token refresh events |
| Token expiry | Access tokens 15min; refresh tokens 7 days; OAuth tokens tracked per-platform |

---

## 9. Development Phases

### Phase 1 — MVP (Weeks 1-4)
**Goal:** Working end-to-end post creation and publishing

| Week | Deliverables |
|------|-------------|
| **Week 1** | Project scaffold (Vite + Express + Prisma + Docker), user auth (register/login/JWT), database setup with migrations |
| **Week 2** | Social account CRUD with encryption, Meta API client base, manual token entry flow |
| **Week 3** | Post editor (caption + media upload), Cloudinary integration, post scheduling UI |
| **Week 4** | BullMQ queue + worker, Instagram publishing (two-step container flow), post status tracking |

**MVP Exit Criteria:**
- [ ] User can register, login, add Instagram account
- [ ] User can create post with image + caption
- [ ] User can schedule post for future
- [ ] Worker publishes to Instagram at scheduled time
- [ ] Post status updates correctly (draft → scheduled → published)

---

### Phase 2 — Multi-Platform & AI (Weeks 5-8)
**Goal:** Add Facebook, Threads, AI generation, templates

| Week | Deliverables |
|------|-------------|
| **Week 5** | Facebook Graph API integration, Threads API integration, multi-platform selector |
| **Week 6** | OpenAI integration: caption generation, hashtag generation, content moderation |
| **Week 7** | Template builder (background, text overlay, font selection), apply template to post |
| **Week 8** | Staggered publishing (rate limit aware), UGC collection & approval flow |

**Phase 2 Exit Criteria:**
- [ ] Publish to Instagram + Facebook + Threads simultaneously
- [ ] AI generates captions and hashtags in <5s
- [ ] Templates save and apply correctly
- [ ] UGC can be submitted, approved, and converted to posts

---

### Phase 3 — Polish & Scale (Weeks 9-12)
**Goal:** Analytics, performance, production readiness

| Week | Deliverables |
|------|-------------|
| **Week 9** | Basic analytics dashboard (posts published, success rate, per-platform stats) |
| **Week 10** | Auto token refresh cron, dead letter queue monitoring, email alerts for failures |
| **Week 11** | E2E test suite (Playwright), load testing (k6), performance optimization |
| **Week 12** | Production deployment (Vercel + Railway/Railway/Render), monitoring (Sentry), docs |

**Phase 3 Exit Criteria:**
- [ ] Analytics dashboard shows meaningful metrics
- [ ] OAuth tokens auto-refresh before expiry
- [ ] E2E tests cover critical user flows
- [ ] Deployed to production with monitoring

---

## 10. Testing Strategy

### 10.1 Unit Tests (Vitest)

**Backend services:**
- `encryption.service.test.ts` — Encrypt/decrypt roundtrip, tamper detection
- `auth.service.test.ts` — Password hash/verify, JWT sign/verify
- `publish.service.test.ts` — Mock platform clients, verify status transitions
- `ai.service.test.ts` — Mock OpenAI responses, verify cost tracking

**Frontend utilities:**
- `validators.test.ts` — Zod schema validation edge cases
- `date.test.ts` — Timezone conversions, scheduling logic

### 10.2 Integration Tests (Vitest + Supertest)

- `auth.routes.test.ts` — Register → Login → Access protected route
- `posts.routes.test.ts` — Create post → Schedule → Verify queue job created
- `accounts.routes.test.ts` — Add account → Encrypt stored → Decrypt on read
- `media.routes.test.ts` — Upload file → Verify Cloudinary mock called

### 10.3 E2E Tests (Playwright)

| Flow | Steps |
|------|-------|
| **Auth** | Register → Login → Logout → Re-login |
| **Create & Schedule** | Login → New post → Upload image → AI caption → Schedule → Verify in list |
| **Multi-Platform Publish** | Login → Connect 3 accounts → Create post → Publish now → Verify statuses |
| **UGC Workflow** | Submit UGC → Approve → Convert to post → Schedule |

### 10.4 Test Environment

```yaml
# docker-compose.test.yml
services:
  postgres-test:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: autopost_test
      POSTGRES_PASSWORD: test
  redis-test:
    image: redis:7-alpine
```

### 10.5 CI/CD Pipeline (GitHub Actions)

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres: { image: postgres:15, env: { POSTGRES_PASSWORD: test } }
      redis: { image: redis:7 }
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npx prisma migrate deploy
      - run: npm run test:unit
      - run: npm run test:integration
      - run: npm run test:e2e
```

---

## 11. Deployment Plan

### 11.1 Local Development

```bash
# 1. Clone and install
git clone https://github.com/Ferdinanzk/AutoPostMediaProject.git
cd AutoPostMediaProject
cp backend/.env.example backend/.env

# 2. Start infrastructure
docker-compose up -d postgres redis

# 3. Run migrations & seed
cd backend && npx prisma migrate dev && npm run seed

# 4. Start backend
cd backend && npm run dev

# 5. Start frontend (new terminal)
cd frontend && npm run dev
```

### 11.2 Production Architecture

```
┌─────────────┐
│   Vercel    │  ← Frontend (React + Vite static build)
│  (Edge CDN) │
└──────┬──────┘
       │ HTTPS
       ▼
┌─────────────┐
│   Railway   │  ← Backend (Node.js + Express Docker container)
│   / Render  │
└──────┬──────┘
       │
   ┌───┴───┐
   ▼       ▼
┌──────┐ ┌──────┐
│ RDS  │ │Redis │  ← PostgreSQL + Redis (managed)
│(PG15)│ │(ElastiCache/Upstash)
└──────┘ └──────┘
```

### 11.3 Environment Variables

```bash
# Database
DATABASE_URL=postgresql://user:pass@host:5432/autopost

# Redis
REDIS_URL=redis://host:6379

# Encryption
MASTER_KEY=64-char-hex-string

# JWT
ACCESS_TOKEN_SECRET=random-string
REFRESH_TOKEN_SECRET=different-random-string

# Meta (for OAuth redirects if we add it later)
META_APP_ID=
META_APP_SECRET=

# OpenAI
OPENAI_API_KEY=sk-...

# Cloudinary
CLOUDINARY_CLOUD_NAME=
CLOUDINARY_API_KEY=
CLOUDINARY_API_SECRET=

# Sentry (optional)
SENTRY_DSN=
```

### 11.4 Deployment Checklist

- [ ] Set all environment variables in hosting platform
- [ ] Run `prisma migrate deploy` as part of startup
- [ ] Configure health check endpoint (`GET /health`)
- [ ] Set up log aggregation (Railway has built-in; otherwise Datadog/Logtail)
- [ ] Configure Sentry for error tracking
- [ ] Set up uptime monitoring (UptimeRobot or Pingdom)
- [ ] Enable automated backups for PostgreSQL
- [ ] Document rollback procedure (`git revert` + redeploy)

### 11.5 Cost Estimates (MVP Scale, ~1,000 users)

| Service | Provider | Monthly Cost |
|---------|----------|-------------|
| Frontend hosting | Vercel Pro | $20 |
| Backend hosting | Railway / Render | $25 |
| PostgreSQL | Railway / Supabase | $25 |
| Redis | Upstash / Railway | $20 |
| Cloudinary | Free tier (25GB) | $0 |
| OpenAI API | Pay-per-use (~50K captions/mo) | ~$50 |
| **Total** | | **~$140/mo** |

---

## Appendix A: Meta API Quick Reference

### Instagram Publishing (Two-Step Flow)

```typescript
// Step 1: Create media container
const container = await instagramClient.post(`/${userId}/media`, {
  image_url: mediaUrl,
  caption: caption,
  published: false,
  scheduled_publish_time: Math.floor(scheduledDate.getTime() / 1000),
});

// Step 2: Publish container
await instagramClient.post(`/${userId}/media_publish`, {
  creation_id: container.id,
});
```

### Rate Limits

| Platform | Limit | Header |
|----------|-------|--------|
| Instagram publishing | 50 posts / 24h | `X-Business-Use-Case-Usage` |
| Threads publishing | 250 posts / 24h | `X-Ads-Api-Limit` |
| General API | 200 calls / hour | `X-App-Usage` |

### Required Permissions

- `instagram_basic` — Read Instagram profile
- `instagram_content_publish` — Publish posts
- `pages_read_engagement` — Read Page engagement
- `threads_basic` — Read Threads profile
- `threads_content_publish` — Publish to Threads

---

## Appendix B: OpenAI Cost Optimization

| Task | Model | Cost |
|------|-------|------|
| Routine caption | gpt-4o-mini | ~$0.15 / 1M tokens |
| Hero campaign copy | gpt-4o | ~$2.50 / 1M tokens |
| Standard image (1024x1024) | dall-e-3 | $0.04 / image |
| HD image | dall-e-3 HD | $0.08 / image |
| Content moderation | text-moderation-latest | ~$0.001 / 1K chars |

---

## Appendix C: Post Lifecycle State Machine

```
                    ┌─────────────┐
                    │   DRAFT     │◄─────────┐
                    └──────┬──────┘          │
                           │ save            │ duplicate
                           ▼                 │
                    ┌─────────────┐          │
              ┌────►│  SCHEDULED  │──────────┘
              │     └──────┬──────┘ cancel
              │            │ enqueue
              │            ▼
              │     ┌─────────────┐
              └─────│   PENDING   │ (worker picked up)
                    └──────┬──────┘
                           │ process
                           ▼
                    ┌─────────────┐
                    │  PUBLISHING │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         ┌───────┐   ┌────────┐   ┌────────┐
         │FAILED │   │PUBLISHED│   │CANCELLED│
         └───┬───┘   └────────┘   └────────┘
             │ retry (max 5)
             └──────────────────────────────►
```

---

*End of Plan.md — Version 1.0.0*
