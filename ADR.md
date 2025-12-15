# ADR-001: YouTube Transcript Extraction Tool Architecture

**Status:** Proposed  
**Version:** 4.0  
**Date:** 2024-12-15  
**Decision Makers:** Project Owner  
**Context:** Personal project, learning-focused, resource-constrained hosting

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [System Context & Constraints](#2-system-context--constraints)
3. [Architecture Overview](#3-architecture-overview)
4. [Authentication & Identity Architecture](#4-authentication--identity-architecture)
5. [Session Management](#5-session-management)
6. [Rate Limiting & Quotas](#6-rate-limiting--quotas)
7. [Mobile-First Accessibility Design](#7-mobile-first-accessibility-design)
8. [The Moments Universe (Core Feature)](#8-the-moments-universe-core-feature)
9. [Moment Detection Engine (2-Tier System)](#9-moment-detection-engine-2-tier-system)
10. [Video Segment Strategy](#10-video-segment-strategy)
11. [Cache Architecture](#11-cache-architecture)
12. [Process Management & Concurrency Control](#12-process-management--concurrency-control)
13. [Data Models](#13-data-models)
14. [API Specification](#14-api-specification)
15. [Export Artifact Format](#15-export-artifact-format)
16. [Admin Dashboard](#16-admin-dashboard)
17. [Visual Polish & Cool Factor](#17-visual-polish--cool-factor)
18. [Deployment Architecture](#18-deployment-architecture)
19. [Implementation Phases](#19-implementation-phases)
20. [Risk Assessment](#20-risk-assessment)
21. [Open Questions](#21-open-questions)
22. [Appendix A: Dependencies](#appendix-a-dependencies)
23. [Appendix B: Environment Variables](#appendix-b-environment-variables)
24. [Revision History](#revision-history)

---

## 1. Executive Summary

A web application that extracts YouTube transcripts and enriches them with visual context (timestamps or embedded images) to produce AI-consumable text artifacts. The system must run comfortably on a Hostinger KVM 4 VPS alongside other services, handle video content efficiently through intelligent caching, and provide a polished, visually impressive user experience with accessibility as a primary concern.

**Core Philosophy:** Maximum functionality at minimum friction. Beautiful by default. Accessible to everyone, especially those of us with fat fingers.

---

## 2. System Context & Constraints

### 2.1 Infrastructure Constraints

**Target Environment: Hostinger KVM 4**

```
+------------------+------------+---------------------------+
| Resource         | Allocation | Reserved for This Project |
+------------------+------------+---------------------------+
| vCPU             | 4 cores    | 1-2 cores max processing  |
| RAM              | 8 GB       | 2 GB ceiling (soft limit) |
| Storage          | 100 GB NVMe| 10-15 GB for video cache  |
| Bandwidth        | 8 TB/month | Variable, optimize low    |
+------------------+------------+---------------------------+
```

**Key Implication:** Every architectural decision must be evaluated against "can this run as a background service without starving other workloads?"

### 2.2 Memory Budget

The 2 GB ceiling must accommodate all project components. Here is the expected memory footprint:

```
+---------------------------+-------------+--------------------------------+
| Component                 | Memory      | Notes                          |
+---------------------------+-------------+--------------------------------+
| FastAPI (2 workers)       | ~200 MB     | ~100 MB per uvicorn worker     |
| Next.js (production)      | ~150 MB     | Single Node.js process         |
| Hanko                     | ~100 MB     | Lightweight Go binary          |
| PostgreSQL (Hanko only)   | ~200 MB     | Small dataset, shared_buffers  |
| Huey worker (1 process)   | ~100 MB     | Python process with task queue |
| Ollama (idle)             | ~50 MB      | Process overhead when idle     |
| Ollama + Qwen2.5-1.5B     | ~1.8 GB     | Only when running inference    |
+---------------------------+-------------+--------------------------------+
| TOTAL (LLM idle)          | ~800 MB     | Normal operation               |
| TOTAL (LLM active)        | ~2.5 GB     | Temporarily exceeds ceiling    |
+---------------------------+-------------+--------------------------------+
```

**Mitigation for LLM peak usage:**
- Ollama loads model on-demand and unloads after idle timeout (default 5 min)
- Configure `OLLAMA_KEEP_ALIVE=2m` to reduce idle memory hold
- Local LLM is admin-gated; most users never trigger this path
- System swap file (2 GB) provides safety margin for brief spikes

### 2.3 Functional Requirements

1. Extract title, description, and timestamped transcript from YouTube URLs
2. Support transcript language selection when multiple tracks available
3. **The Moments Universe:** A core, beautiful experience for identifying visual moments
4. Preview video segments without downloading entire videos
5. Two-tier moment detection: Rules -> Local LLM (admin-granted)
6. Extract frames at selected timestamps
7. Upload frames to external image hosting
8. Generate final AI-consumable artifact with embedded images

### 2.4 Non-Functional Requirements

- **Zero-Friction Start:** Core functionality works without signup
- **Responsiveness:** UI must remain fluid during backend processing
- **Mobile-First Accessibility:** Large touch targets, forgiving interactions
- **Resilience:** Partial cache eviction must not break active sessions
- **Resource Efficiency:** Must coexist with other services on KVM 4
- **Cool Factor:** Visual polish is a core requirement, not a nice-to-have

---

## 3. Architecture Overview

```
+-----------------------------------------------------------------------+
|                           CLIENT (Browser)                            |
|  +------------------------------------------------------------------+ |
|  |  Next.js App                                                     | |
|  |  +----------------+ +----------------+ +------------------------+| |
|  |  | URL Input &    | | Transcript     | | ** MOMENTS UNIVERSE ** || |
|  |  | Metadata View  | | Viewer         | | (Segment Player+Picker)|| |
|  |  +----------------+ +----------------+ +------------------------+| |
|  |  +----------------+ +----------------+ +------------------------+| |
|  |  | Frame Review   | | Export         | | Processing Status      || |
|  |  | Gallery        | | Controls       | | (WebSocket)            || |
|  |  +----------------+ +----------------+ +------------------------+| |
|  +------------------------------------------------------------------+ |
+-----------------------------------------------------------------------+
                                   |
                                   | HTTP/REST + WebSocket
                                   v
+-----------------------------------------------------------------------+
|                        BACKEND (Python/FastAPI)                       |
|  +------------------------------------------------------------------+ |
|  |                      API Gateway Layer                           | |
|  |  - Rate limiting   - Session/Auth    - Tailscale Admin Check     | |
|  |  - CORS headers                                                   | |
|  +------------------------------------------------------------------+ |
|  +------------------------------------------------------------------+ |
|  |                        Service Layer                             | |
|  |  +----------------+ +----------------+ +------------------------+| |
|  |  | Transcript     | | Video Segment  | | Moment Detection       || |
|  |  | Extractor      | | Manager        | | Engine (2-tier)        || |
|  |  +----------------+ +----------------+ +------------------------+| |
|  |  +----------------+ +----------------+ +------------------------+| |
|  |  | Frame          | | Image Upload   | | Artifact Compiler      || |
|  |  | Extractor      | | Service        | |                        || |
|  |  +----------------+ +----------------+ +------------------------+| |
|  +------------------------------------------------------------------+ |
|  +------------------------------------------------------------------+ |
|  |                    Infrastructure Layer                          | |
|  |  +----------------+ +----------------+ +------------------------+| |
|  |  | Task Queue     | | Cache Manager  | | Process Limiter        || |
|  |  | (Huey/SQLite)  | | (LRU + Locks)  | | (Semaphore Pool)       || |
|  |  +----------------+ +----------------+ +------------------------+| |
|  +------------------------------------------------------------------+ |
+-----------------------------------------------------------------------+
                                   |
          +------------------------+------------------------+
          v                        v                        v
+------------------+    +------------------+    +----------------------+
| YouTube          |    | Ollama           |    | Tailscale            |
| (via yt-dlp)     |    | - Local LLM      |    | - Admin identity     |
|                  |    | - Qwen2.5-1.5B   |    | - Network-level ACL  |
+------------------+    +------------------+    +----------------------+
```

---

## 4. Authentication & Identity Architecture

### 4.1 Design Philosophy

**"Progressive Identity"** -- Users should get value immediately, with identity becoming relevant only when they want persistence or advanced features.

```
+-----------------------------------------------------------------------+
|                      IDENTITY PROGRESSION                             |
|                                                                       |
|   +--------------+       +--------------+       +--------------+      |
|   |  ANONYMOUS   |  -->  |  IDENTIFIED  |  -->  |    ADMIN     |      |
|   |  SESSION     |       |   (Hanko)    |       | (Tailscale)  |      |
|   +--------------+       +--------------+       +--------------+      |
|                                                                       |
|   Features:              Features:              Features:             |
|   - Full core UX         - All anonymous        - All identified      |
|   - Session storage      - Persistent data      - Feature flag ctrl   |
|   - No persistence       - Higher rate limits   - Local LLM toggle    |
|   - Rate limited         - History              - Usage dashboards    |
|                                                 - User management     |
|                                                                       |
+-----------------------------------------------------------------------+
```

### 4.2 Identified Users (Hanko)

**Decision:** Use **Hanko** (open-source, self-hostable) for passkey-first authentication with email magic link fallback.

**Why Hanko?**

- Open source, self-hostable (fits our KVM 4)
- Passkey-first with fallback to email magic links
- Beautiful, customizable drop-in UI components
- No vendor lock-in
- Active development, strong community

**Why Passkeys as Default?**

- Zero password friction (no "forgot password" flows)
- Phishing-resistant by design
- Modern UX that feels premium
- Offloads security to user's device/biometrics

**Authentication Flow:**

```
+-----------------------------------------------------------------------+
|                                                                       |
|   User arrives at sign-up/login                                       |
|                         |                                             |
|                         v                                             |
|   +---------------------------------------------------------------+   |
|   |                   HANKO AUTH UI                               |   |
|   |                                                               |   |
|   |   +-------------------------------------------------------+   |   |
|   |   |                                                       |   |   |
|   |   |   [*] Create Passkey (Recommended)                    |   |   |
|   |   |                                                       |   |   |
|   |   +-------------------------------------------------------+   |   |
|   |                                                               |   |
|   |   +-------------------------------------------------------+   |   |
|   |   |  [...] Email magic link (fallback)                    |   |   |
|   |   +-------------------------------------------------------+   |   |
|   |                                                               |   |
|   +---------------------------------------------------------------+   |
|                         |                                             |
|          +--------------+--------------+                              |
|          |                             |                              |
|          v                             v                              |
|      Passkey                       Email                              |
|      Created                       Link Sent                          |
|          |                             |                              |
|          +--------------+--------------+                              |
|                         |                                             |
|                         v                                             |
|                  User Authenticated                                   |
|                  (Hanko session created)                              |
|                                                                       |
+-----------------------------------------------------------------------+
```

**Hanko Deployment (on KVM 4):**

```yaml
# docker-compose.yml
services:
  hanko:
    image: ghcr.io/teamhanko/hanko:latest
    environment:
      - DATABASE_URL=postgres://hanko:password@postgres:5432/hanko
      - SECRETS_KEYS=${HANKO_SECRET_KEY}
      - WEBAUTHN_RELYING_PARTY_ID=${DOMAIN}
      - WEBAUTHN_RELYING_PARTY_ORIGIN=https://${DOMAIN}
      - PASSCODE_EMAIL_FROM_ADDRESS=auth@${DOMAIN}
      - PASSCODE_SMTP_HOST=${SMTP_HOST}
      - PASSCODE_SMTP_PORT=${SMTP_PORT}
      - PASSCODE_SMTP_USER=${SMTP_USER}
      - PASSCODE_SMTP_PASSWORD=${SMTP_PASSWORD}
    ports:
      - "127.0.0.1:8001:8000"  # Only local, proxied by Caddy
    depends_on:
      - postgres

  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=hanko
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=hanko
    volumes:
      - hanko_data:/var/lib/postgresql/data

volumes:
  hanko_data:
```

**Hanko Frontend Integration:**

```tsx
// Using @teamhanko/hanko-elements
import { Hanko } from "@teamhanko/hanko-elements";

// Initialize
const hankoApi = `https://${process.env.NEXT_PUBLIC_DOMAIN}/hanko`;
const hanko = new Hanko(hankoApi);

// Auth component (drop-in)
export function AuthPage() {
  return (
    <div className="auth-container">
      <hanko-auth api={hankoApi} />
    </div>
  );
}

// Profile component (manage passkeys)
export function ProfilePage() {
  return (
    <div className="profile-container">
      <hanko-profile api={hankoApi} />
    </div>
  );
}

// Session check
const isLoggedIn = await hanko.session.isValid();
const user = await hanko.user.getCurrent();
```

### 4.3 Admin Access via Tailscale

**Decision:** Admin functionality is only accessible from the Tailscale network. No code-level auth checks needed -- network topology *is* the auth.

**Why Tailscale?**

- Zero-trust networking without VPN complexity
- Identity tied to your Tailscale account
- Can expose specific routes only to your tailnet
- Elegant: if you can reach the admin route, you're authorized

**Implementation via Caddy:**

```
+-----------------------------------------------------------------------+
|                              INTERNET                                 |
|                                                                       |
|    Public Users -----------------------+                              |
|                                        |                              |
|                                        v                              |
|                              +------------------+                     |
|                              |  Caddy (Public)  |                     |
|                              |  :443            |                     |
|                              |                  |                     |
|                              |  Routes:         |                     |
|                              |  /* --> Next.js  |                     |
|                              |  /api/* --> API  |                     |
|                              |  /hanko/* -->    |                     |
|                              |      Hanko       |                     |
|                              |                  |                     |
|                              |  BLOCKS:         |                     |
|                              |  /admin/* --> 404|<-- Not exposed      |
|                              +------------------+    to public        |
|                                                                       |
+-----------------------------------------------------------------------+

+-----------------------------------------------------------------------+
|                           YOUR TAILNET                                |
|                                                                       |
|    Your Device ------------------------+                              |
|    (Tailscale connected)               |                              |
|                                        v                              |
|                            +--------------------+                     |
|                            | Caddy (Tailscale)  |                     |
|                            | :8443 (TS IP only) |                     |
|                            |                    |                     |
|                            | Routes:            |                     |
|                            | /admin/* --> OK    |<-- Full admin       |
|                            | /* --> proxy to    |                     |
|                            |     public Caddy   |                     |
|                            +--------------------+                     |
|                                                                       |
+-----------------------------------------------------------------------+
```

**Caddyfile Configuration:**

```caddyfile
# Public-facing (Internet)
{$DOMAIN} {
    # Hanko authentication server
    handle /hanko/* {
        reverse_proxy localhost:8001
    }
    
    # API routes
    handle /api/* {
        reverse_proxy localhost:8000
    }
    
    # Frontend
    handle {
        reverse_proxy localhost:3000
    }
    
    # Block admin routes from public internet
    handle /admin/* {
        respond "Not Found" 404
    }
}

# Tailscale-only (Admin)
http://{$TAILSCALE_HOSTNAME}:8443 {
    # Admin routes - full access
    handle /admin/* {
        reverse_proxy localhost:8000
    }
    
    # Also proxy regular routes for convenience
    handle /api/* {
        reverse_proxy localhost:8000
    }
    
    handle {
        reverse_proxy localhost:3000
    }
}
```

**Admin Capabilities:**

```
+------------------------+------------------------------------------+
| Capability             | Description                              |
+------------------------+------------------------------------------+
| User Management        | View all users, usage stats              |
| Feature Flags          | Toggle local LLM per user                |
| Rate Limit Override    | Adjust limits for specific users         |
| Cache Management       | View/clear cache, see stats              |
| System Health          | Resource usage, queue depth              |
+------------------------+------------------------------------------+
```

---

## 5. Session Management

### 5.1 Session Types

The system supports two session types with different storage characteristics:

```
+------------------+------------------+------------------------------------+
| Aspect           | Anonymous        | Identified                         |
+------------------+------------------+------------------------------------+
| Creation         | Auto on first    | After Hanko authentication         |
|                  | visit            |                                    |
+------------------+------------------+------------------------------------+
| Client Storage   | localStorage     | localStorage (Hanko token)         |
|                  | (session UUID)   |                                    |
+------------------+------------------+------------------------------------+
| Server Storage   | SQLite           | SQLite (linked to user account)    |
+------------------+------------------+------------------------------------+
| Data Retention   | 24 hours after   | Indefinite (until user deletes)    |
|                  | last activity    |                                    |
+------------------+------------------+------------------------------------+
| Upgrade Path     | Can upgrade to   | N/A (already identified)           |
|                  | identified       |                                    |
+------------------+------------------+------------------------------------+
```

### 5.2 Anonymous Sessions

**Every visitor gets full core functionality without any signup.**

**Client-Side Implementation:**

```typescript
interface AnonymousSession {
  id: string;           // UUID, generated on first visit
  createdAt: string;
  lastActiveAt: string;
  currentVideoId?: string;
  selectedMoments: VisualMoment[];
}

// Stored in localStorage, sent as header: X-Session-ID: uuid-here
// Fallback to sessionStorage if localStorage unavailable
```

### 5.3 Session Lifecycle

```
1. Session Start
   '-> Client connects
   '-> If no session ID in localStorage: generate UUID, send to server
   '-> Server creates session record in SQLite
   '-> Server loads any existing cache locks for this session from DB

2. Segment Request  
   '-> Each viewed segment is locked to that session
   '-> Lock stored in both memory (for fast access) and SQLite (for persistence)
   '-> Lock prevents eviction during active viewing

3. Heartbeat
   '-> Client sends ping every 30 seconds via WebSocket or REST
   '-> Updates session's last_active_at timestamp
   '-> Failure to heartbeat for 2 minutes triggers lock release

4. Lock Release (2-minute heartbeat timeout)
   '-> All segment locks for session are released
   '-> Session DATA is NOT deleted (only locks)
   '-> Allows cache eviction of previously-locked segments
   '-> User can continue working; segments re-lock on next request

5. Session Data Expiry (24 hours for anonymous)
   '-> Anonymous session data deleted after 24 hours of inactivity
   '-> Identified user session data persists indefinitely

6. Session Upgrade (Anonymous -> Identified)
   '-> User authenticates via Hanko
   '-> Anonymous session data migrated to user account
   '-> Anonymous session record deleted
   '-> New identified session created with same working state

7. Graceful Degradation
   '-> If segment evicted while user inactive (locks released)
   '-> Client receives "segment unavailable" response
   '-> Client can re-request (will re-download and re-lock)
   '-> User sees brief "Reloading preview..." message
```

**Key Distinction:**
- **Lock timeout:** 2 minutes without heartbeat releases segment locks
- **Data retention:** 24 hours for anonymous sessions, indefinite for identified users

### 5.4 Session Upgrade Flow

```python
async def upgrade_session(anonymous_id: str, hanko_user_id: str) -> str:
    """
    Migrate anonymous session data to identified user account.
    Returns the new identified session ID.
    """
    anon_data = await get_anonymous_session(anonymous_id)
    
    if anon_data:
        # Get or create user record
        user = await get_or_create_user(hanko_user_id)
        
        # Migrate moments, history, preferences
        await merge_into_user_account(user.id, anon_data)
        
        # Transfer any active segment locks
        await transfer_locks(anonymous_id, user.id)
        
        # Delete anonymous session
        await delete_anonymous_session(anonymous_id)
        
        # Log upgrade for analytics
        await log_event("session_upgraded", {
            "anonymous_id": anonymous_id,
            "user_id": user.id,
            "moments_migrated": len(anon_data.selected_moments)
        })
    
    # Create new identified session
    return await create_identified_session(user.id)
```

### 5.5 WebSocket Connection Management

**Connection:** `wss://{domain}/ws/session/{sessionId}`

**Reconnection Strategy:**

```typescript
class WebSocketManager {
  private ws: WebSocket | null = null;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 10;
  private lastEventId: string | null = null;
  
  connect(sessionId: string) {
    const url = `wss://${domain}/ws/session/${sessionId}`;
    
    // Include last event ID for replay on reconnect
    const params = this.lastEventId 
      ? `?lastEventId=${this.lastEventId}` 
      : '';
    
    this.ws = new WebSocket(url + params);
    
    this.ws.onopen = () => {
      this.reconnectAttempts = 0;
      // Server will replay missed events if lastEventId provided
    };
    
    this.ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      this.lastEventId = data.eventId;
      this.handleEvent(data);
    };
    
    this.ws.onclose = () => {
      this.scheduleReconnect(sessionId);
    };
  }
  
  private scheduleReconnect(sessionId: string) {
    if (this.reconnectAttempts >= this.maxReconnectAttempts) {
      this.onMaxRetriesExceeded();
      return;
    }
    
    // Exponential backoff: 1s, 2s, 4s, 8s... max 30s
    const delay = Math.min(
      1000 * Math.pow(2, this.reconnectAttempts),
      30000
    );
    
    this.reconnectAttempts++;
    setTimeout(() => this.connect(sessionId), delay);
  }
}
```

**Server-Side Event Replay:**

```python
# Server maintains recent events per session (last 5 minutes)
# On reconnect with lastEventId, replay all events after that ID

async def handle_websocket_connect(
    websocket: WebSocket,
    session_id: str,
    last_event_id: Optional[str] = None
):
    await websocket.accept()
    
    # Replay missed events
    if last_event_id:
        missed_events = await get_events_after(session_id, last_event_id)
        for event in missed_events:
            await websocket.send_json(event)
    
    # Continue normal operation
    await session_event_loop(websocket, session_id)
```

---

## 6. Rate Limiting & Quotas

### 6.1 Storage

**Decision:** Rate limits stored in SQLite alongside other application data.

```sql
-- Rate limit tracking
CREATE TABLE rate_limits (
    key TEXT PRIMARY KEY,          -- "{session_or_user_id}:{action}:{window}"
    count INTEGER NOT NULL,
    window_start TEXT NOT NULL,    -- ISO timestamp
    expires_at TEXT NOT NULL       -- For cleanup
);

CREATE INDEX idx_rate_limits_expires ON rate_limits(expires_at);
```

**Implementation:**

```python
from datetime import datetime, timedelta

async def check_rate_limit(
    identifier: str,  # session_id or user_id
    action: str,
    limit: int,
    window_seconds: int
) -> tuple[bool, int]:
    """
    Check if action is allowed under rate limit.
    Returns (allowed, remaining_count).
    """
    window_start = datetime.utcnow().replace(
        second=0, microsecond=0
    ) - timedelta(seconds=datetime.utcnow().second % window_seconds)
    
    key = f"{identifier}:{action}:{window_start.isoformat()}"
    
    async with db.transaction():
        row = await db.fetchone(
            "SELECT count FROM rate_limits WHERE key = ?",
            (key,)
        )
        
        current_count = row["count"] if row else 0
        
        if current_count >= limit:
            return False, 0
        
        # Increment or insert
        expires_at = window_start + timedelta(seconds=window_seconds * 2)
        await db.execute("""
            INSERT INTO rate_limits (key, count, window_start, expires_at)
            VALUES (?, 1, ?, ?)
            ON CONFLICT(key) DO UPDATE SET count = count + 1
        """, (key, window_start.isoformat(), expires_at.isoformat()))
        
        return True, limit - current_count - 1

# Periodic cleanup (run via Huey scheduled task)
async def cleanup_expired_rate_limits():
    await db.execute(
        "DELETE FROM rate_limits WHERE expires_at < ?",
        (datetime.utcnow().isoformat(),)
    )
```

### 6.2 Limits by User Type

**Anonymous Users:**

```
+---------------------+-------+------------+
| Action              | Limit | Window     |
+---------------------+-------+------------+
| Video extractions   | 10    | per hour   |
| Segment downloads   | 50    | per hour   |
| Frame extractions   | 30    | per hour   |
| Exports             | 5     | per hour   |
+---------------------+-------+------------+
```

**Identified Users:**

```
+---------------------+-------+------------+
| Action              | Limit | Window     |
+---------------------+-------+------------+
| Video extractions   | 30    | per hour   |
| Segment downloads   | 150   | per hour   |
| Frame extractions   | 100   | per hour   |
| Exports             | 20    | per hour   |
| LLM analyses        | 10    | per hour   |
+---------------------+-------+------------+
```

**Admin Override:**

Admins can set a `rate_limit_multiplier` per user (e.g., 2.0 = double limits).

### 6.3 Response Headers

All API responses include rate limit information:

```
X-RateLimit-Limit: 30
X-RateLimit-Remaining: 27
X-RateLimit-Reset: 1702656000
```

### 6.4 Exceeded Limit Response

```json
{
  "error": "rate_limit_exceeded",
  "message": "Video extraction limit reached. Try again in 47 minutes.",
  "retry_after_seconds": 2820
}
```

HTTP Status: `429 Too Many Requests`

---

## 7. Mobile-First Accessibility Design

### 7.1 Design Philosophy

**"Fat fingers are not a bug, they're the primary use case."**

Every interactive element must be designed assuming the user:

- Has large fingers relative to screen size
- Is using one hand (thumb-only navigation)
- Is in motion (on transit, walking)
- Might accidentally tap adjacent elements

### 7.2 Touch Target Standards

**Minimum Sizes (Non-Negotiable):**

```
+-----------------------+--------------+------------+------------------+
| Element Type          | Minimum Size | Ideal Size | Spacing          |
+-----------------------+--------------+------------+------------------+
| Primary actions       | 48x48px      | 56x56px    | 8px from neighbor|
| Secondary actions     | 44x44px      | 48x48px    | 8px from neighbor|
| List items            | 48px height  | 56px height| 0 (full-width)   |
| Timeline markers      | 44x44px      | 48x48px    | 12px             |
| Close/dismiss         | 48x48px      | 56x56px    | Corner, 16px pad |
+-----------------------+--------------+------------+------------------+
```

**Visual vs. Touch Target:**

```
+-------------------------------------------+
|                                           |
|    +----------------------------------+   |
|    |    +------------------------+    |   |
|    |    |                        |    |   |
|    |    |    Visual Button       |    |<-- What user sees (32x32)
|    |    |    (smaller)           |    |   |
|    |    |                        |    |   |
|    |    +------------------------+    |   |
|    |                                  |   |
|    |    Touch Target Area             |<-- Actual tappable (48x48)
|    |    (larger, invisible)           |   |
|    |                                  |   |
|    +----------------------------------+   |
|                                           |
+-------------------------------------------+
```

**Implementation:**

```tsx
const TouchTarget = ({ children, onTap, minSize = 48 }) => (
  <motion.button
    onClick={onTap}
    style={{ 
      minWidth: minSize,
      minHeight: minSize,
      display: 'flex',
      alignItems: 'center',
      justifyContent: 'center',
    }}
    whileTap={{ scale: 0.95 }}
    onTapStart={() => navigator.vibrate?.(10)}
  >
    {children}
  </motion.button>
);
```

### 7.3 Gesture-Based Interactions

**Replace Small Buttons with Gestures:**

```
+---------------------------+---------------------------+--------------------+
| Instead of...             | Use...                    | Benefit            |
+---------------------------+---------------------------+--------------------+
| Small "x" close button    | Swipe down to dismiss     | Whole screen=target|
| Tiny edit/delete icons    | Swipe left on item        | Natural, intuitive |
| Small +/- buttons         | Pinch to zoom timeline    | Intuitive, precise |
| Checkbox for select       | Long-press to select      | Harder to mis-tap  |
+---------------------------+---------------------------+--------------------+
```

**Gesture Implementation (Moments Universe):**

```tsx
<motion.div
  drag="x"
  dragConstraints={{ left: 0, right: 0 }}
  dragElastic={0.2}
  onDragEnd={(e, info) => {
    if (info.offset.x < -100) {
      onDeleteMoment();
    } else if (info.offset.x > 100) {
      onConfirmMoment();
    }
  }}
  style={{
    x: dragX,
    backgroundColor: dragX < -50 ? 'rgba(255,0,0,0.1)' : 
                     dragX > 50 ? 'rgba(0,255,0,0.1)' : 
                     'transparent'
  }}
>
  <MomentCard />
</motion.div>
```

### 7.4 Mobile Layout Patterns

**Bottom Sheet for Actions (Not Modals):**

```
Mobile:
+---------------------+
|                     |
|                     |
|                     |
|                     |
+---------------------+
| =================== |<-- Drag handle
|                     |
|   Bottom Sheet      |<-- Thumb zone
|   (actions here)    |
|                     |
|  [ Big Button ]     |
|                     |
+---------------------+
```

**Thumb Zone Optimization:**

```
+-------------------------------------+
|                                     |
|        :-( HARD TO REACH            |   <-- Avoid primary actions
|                                     |
+-------------------------------------+
|                                     |
|        :-| OKAY TO REACH            |   <-- Secondary actions OK
|                                     |
+-------------------------------------+
|                                     |
|        :-) EASY TO REACH            |   <-- Primary actions HERE
|                                     |
|    [  Primary Action Button  ]      |
|                                     |
+-------------------------------------+
        ^ Thumb naturally rests here
```

### 7.5 Forgiving Interactions

**Error Prevention:**

```
+-------------------------------+------------------------------------------+
| Situation                     | Solution                                 |
+-------------------------------+------------------------------------------+
| Accidental tap                | 300ms delay before destructive actions   |
| Tap near multiple elements    | Enlarge closest target's hitbox          |
| Shaky hands                   | Debounce rapid successive taps           |
| Scrolling vs. tapping         | Require stationary touch for tap         |
+-------------------------------+------------------------------------------+
```

**Undo Everything:**

```tsx
const handleDeleteMoment = async (momentId: string) => {
  // Optimistically remove from UI
  setMoments(prev => prev.filter(m => m.id !== momentId));
  
  // Show undo toast
  toast({
    title: "Moment removed",
    action: (
      <ToastAction onClick={() => undoDelete(momentId)}>
        Undo
      </ToastAction>
    ),
    duration: 5000,
  });
  
  // Actually delete after toast expires
  setTimeout(() => {
    permanentlyDelete(momentId);
  }, 5500);
};
```

---

## 8. The Moments Universe (Core Feature)

### 8.1 Concept

The **Moments Universe** is not just a feature -- it's the soul of the application. It's where users discover, select, and curate the visual moments that transform a transcript into a complete, AI-comprehensible document.

**Design Principles:**

1. **Discoverable:** Moments should feel like they're floating in a space, waiting to be found
2. **Tactile:** Every interaction should feel physical and satisfying
3. **Forgiving:** Wrong selections are trivially reversible
4. **Beautiful:** This is where we go all-in on cool factor

### 8.2 Visual Design: The Timeline Constellation

```
+-----------------------------------------------------------------------+
|                                                                       |
|                           THE TIMELINE                                |
|                                                                       |
|     0:00                                                      45:00   |
|       |                                                         |     |
|       v                                                         v     |
|   +===============================================================+   |
|   |                                                               |   |
|   |  .....########.........####..............############.......  |   |
|   |       /      \          |                      |              |   |
|   |      /        \         |                      |              |   |
|   |     /          \        |                      |              |   |
|   |    @            @       o                      @              |   |
|   |   0:15         0:45    5:20                  32:10            |   |
|   |  (locked)     (locked) (pending)            (auto)            |   |
|   |                                                               |   |
|   +===============================================================+   |
|                                                                       |
|   Legend:                                                             |
|   ..... = Uncached segment (dim, clickable to load)                   |
|   ##### = Cached segment (bright, playable)                           |
|   @     = Confirmed moment (solid, pulsing glow)                      |
|   o     = Pending moment (hollow, waiting confirmation)               |
|   *     = Auto-suggested moment (asterisk, needs review)              |
|                                                                       |
+-----------------------------------------------------------------------+
```

### 8.3 Moment States & Animations

**State Machine:**

```
                    +-----------+
                    |   EMPTY   |
                    | (no moment|
                    +-----------+
                          |
           +--------------+--------------+
           |                             |
           v                             v
   +---------------+             +---------------+
   | AUTO-DETECTED |             |  USER         |
   | (by LLM/rules)|             | SELECTED      |
   +-------+-------+             +-------+-------+
           |                             |
           +--------------+--------------+
                          |
                          v
                   +------------+
                   |  PENDING   |
                   | (needs ack)|
                   +-----+------+
                         |
            +------------+------------+
            |                         |
            v                         v
      +-----------+            +-----------+
      | CONFIRMED |            | DISMISSED |
      | (locked)  |            | (removed) |
      +-----+-----+            +-----------+
            |
            v
      +-----------+
      |   FRAME   |
      | EXTRACTED |
      +-----+-----+
            |
            v
      +-----------+
      | UPLOADED  |
      | (ready)   |
      +-----------+
```

**Animation Specifications:**

```
+----------------------+----------------------------+----------+----------------+
| Transition           | Animation                  | Duration | Easing         |
+----------------------+----------------------------+----------+----------------+
| Appear (new moment)  | Scale from 0 + fade        | 300ms    | spring(0.5,0.7)|
| Pending -> Confirmed | Pulse + fill               | 200ms    | ease-out       |
| Pending -> Dismissed | Shrink + fade              | 200ms    | ease-in        |
| Hover (desktop)      | Gentle float + glow        | cont.    | ease-in-out    |
| Drag (reorder)       | Lift shadow + scale 1.05   | 150ms    | ease-out       |
| Delete               | Disintegrate particles     | 400ms    | ease-in        |
+----------------------+----------------------------+----------+----------------+
```

### 8.4 Moment Picker Mode

When user activates moment picking:

```tsx
<AnimatePresence>
  {pickingMode && (
    <motion.div
      className="fixed inset-0 z-50"
      initial={{ opacity: 0 }}
      animate={{ opacity: 1 }}
      exit={{ opacity: 0 }}
    >
      <div className="absolute inset-0 bg-black/80" />
      
      <SpotlightCursor 
        size={200}
        color="rgba(147, 51, 234, 0.3)"
      />
      
      <VideoPreview expanded />
      
      <motion.div
        initial={{ height: 60 }}
        animate={{ height: 160 }}
        className="fixed bottom-0 left-0 right-0"
      >
        <ExpandedTimeline 
          onMark={handleMarkMoment}
          precision="frame"
        />
      </motion.div>
      
      <div className="fixed top-4 left-1/2 -translate-x-1/2">
        <motion.p
          initial={{ y: -20, opacity: 0 }}
          animate={{ y: 0, opacity: 1 }}
          className="text-white text-lg"
        >
          Tap timeline or press SPACE to mark this moment
        </motion.p>
      </div>
      
      <motion.button
        className="fixed top-4 right-4 p-4"
        whileTap={{ scale: 0.9 }}
      >
        <XIcon size={32} />
      </motion.button>
    </motion.div>
  )}
</AnimatePresence>
```

### 8.5 Frame Preview Gallery

After moments are confirmed and frames extracted:

```
+------------------------------------------------------------------------+
|                                                                        |
|   Your Moments (5)                               [Upload All] [Export] |
|                                                                        |
|   +---------------------------------------------------------------+    |
|   |                                                               |    |
|   |    +--------+    +--------+    +--------+    +--------+       |    |
|   |    |        |    |        |    |        |    |        |       |    |
|   |    | [IMG]  |    | [IMG]  |    | [IMG]  |    | [IMG]  |       |    |
|   |    |        |    |        |    |        |    |        |       |    |
|   |    |  0:15  |    |  0:45  |    |  5:20  |    | 32:10  |       |    |
|   |    | Ready  |    | Ready  |    | Upload |    | Ready  |       |    |
|   |    |        |    |        |    |        |    |        |       |    |
|   |    +--------+    +--------+    +--------+    +--------+       |    |
|   |         ^             ^             ^             ^           |    |
|   |    Drag to reorder                                            |    |
|   |                                                               |    |
|   +---------------------------------------------------------------+    |
|                                                                        |
+------------------------------------------------------------------------+
```

**Interactions:**

- **Tap:** Expand to full preview + transcript context
- **Long-press:** Enter edit mode (adjust timestamp +/- 2s)
- **Swipe left:** Delete (with undo)
- **Drag:** Reorder (affects final artifact order)
- **Pinch:** Zoom into frame details

---

## 9. Moment Detection Engine (2-Tier System)

### 9.1 Architecture Overview

**Decision:** Two distinct detection modes, with clear user control.

```
+-----------------------------------------------------------------------+
|                     MOMENT DETECTION ENGINE                           |
|                                                                       |
|   Mode Selection (User Controlled)                                    |
|                                                                       |
|   +-----------------------+        +-----------------------+          |
|   |         [=]           |        |        <cpu>          |          |
|   |     RULES ONLY        |        |    RULES + LOCAL      |          |
|   |                       |        |       (Ollama)        |          |
|   |     Default           |        |                       |          |
|   |     Always on         |        |    Admin-granted      |          |
|   |                       |        |    only               |          |
|   +-----------------------+        +-----------------------+          |
|              |                              |                         |
|              v                              v                         |
|   +---------------------------------------------------------------+   |
|   |           Combined Results --> Deduplicated                   |   |
|   +---------------------------------------------------------------+   |
|                                                                       |
+-----------------------------------------------------------------------+
```

### 9.2 Tier 1: Rule-Based Detection (Always Active)

**Decision:** Sophisticated pattern matching that catches 60-70% of visual moments without any ML.

**Pattern Categories:**

```python
VISUAL_MOMENT_PATTERNS = {
    # Category 1: Direct visual references
    "deictic_explicit": [
        r"\b(as you can see|look at|take a look|see here|shown here)\b",
        r"\b(on (?:the )?screen|on display|displayed here)\b",
        r"\b(this (?:diagram|chart|graph|image|picture|screenshot|slide))\b",
        r"\b(the (?:diagram|chart|graph|image|picture|screenshot|slide) shows)\b",
    ],
    
    # Category 2: Pointing/demonstrating
    "deictic_implicit": [
        r"\b(right here|over here|down here|up here)\b",
        r"\b(this part|this section|this area|this portion)\b",
        r"\b(notice (?:how|that|the)|observe (?:how|that|the))\b",
        r"\b(watch (?:this|what happens|closely))\b",
    ],
    
    # Category 3: Code/technical demonstrations
    "code_demo": [
        r"\b(the code (?:here|shows|demonstrates|looks like))\b",
        r"\b((?:this|the) function|(?:this|the) method|(?:this|the) class)\b",
        r"\b(line \d+|lines \d+(?:\s*[-–]\s*\d+)?)\b",
        r"\b((?:this|the) output|(?:the )?result(?:s)? (?:show|are|is))\b",
        r"\b(let me (?:show|demonstrate|walk through))\b",
    ],
    
    # Category 4: UI/screen navigation
    "ui_navigation": [
        r"\b(click(?:ing)? (?:on|here)|tap(?:ping)? (?:on|here))\b",
        r"\b((?:this|the) button|(?:this|the) menu|(?:this|the) tab)\b",
        r"\b(navigate to|go to|open(?:ing)?|select(?:ing)?)\b",
        r"\b(drag(?:ging)?|scroll(?:ing)?|swipe|zoom)\b",
    ],
    
    # Category 5: Transitions and comparisons
    "transitions": [
        r"\b(before and after|side by side|comparison)\b",
        r"\b((?:let me |now |we )switch to|moving (?:on )?to)\b",
        r"\b((?:on the )?(?:left|right)(?: side)?|(?:at the )?(?:top|bottom))\b",
        r"\b(split screen|dual view)\b",
    ],
    
    # Category 6: Physical demonstrations
    "physical_demo": [
        r"\b((?:I am |I'm |we're |we are )(?:holding|showing|pointing))\b",
        r"\b((?:this|the) device|(?:this|the) hardware|(?:this|the) equipment)\b",
        r"\b(physically|in (?:my|the) hand|on (?:my|the) desk)\b",
    ],
}

# Confidence weights by category
CATEGORY_WEIGHTS = {
    "deictic_explicit": 0.95,
    "deictic_implicit": 0.75,
    "code_demo": 0.85,
    "ui_navigation": 0.80,
    "transitions": 0.70,
    "physical_demo": 0.90,
}
```

**Contextual Boosting:**

```python
def calculate_moment_confidence(
    segment: TranscriptSegment,
    matches: List[PatternMatch],
    context: AnalysisContext
) -> float:
    """Adjust confidence based on context."""
    base_confidence = max(m.category_weight for m in matches)
    
    # Boost: Multiple patterns in same segment
    if len(matches) > 1:
        base_confidence += 0.10
    
    # Boost: Technical video (detected from title/description)
    if context.video_type in ['tutorial', 'walkthrough', 'demo']:
        base_confidence += 0.05
    
    # Penalty: Very short segment (might be filler)
    if segment.duration < 2.0:
        base_confidence -= 0.15
    
    # Penalty: Pattern appears very frequently (speaker's verbal tic)
    if context.pattern_frequency[matches[0].pattern] > 0.1:
        base_confidence -= 0.20
    
    # Boost: Follows or precedes silence (intentional pause)
    if context.has_pause_nearby(segment.start, threshold=1.5):
        base_confidence += 0.10
    
    return min(max(base_confidence, 0.0), 1.0)
```

**Output:**

```python
@dataclass
class RuleBasedMoment:
    timestamp: float
    confidence: float  # 0.0 - 1.0
    matched_patterns: List[str]
    category: str
    transcript_snippet: str  # +/- 15 words around match
    reason: str  # Human-readable explanation
```

### 9.3 Tier 2: Local LLM (Admin-Granted Feature)

**Decision:** Self-hosted Ollama with small model, enabled per-user via admin feature flag.

**Why Feature-Flagged?**

- Local LLM consumes server resources (RAM, CPU)
- Not everyone needs it (rule-based tier catches most cases)
- Allows you to grant access to trusted users/testers
- Keeps base experience snappy for casual users

**Model Selection:**

```
+------------------------+------+--------+---------+------------------+
| Model                  | RAM  | Speed  | Quality | Notes            |
+------------------------+------+--------+---------+------------------+
| Qwen2.5-1.5B-Instruct  | ~2GB | Fast   | Good    | ** PRIMARY **    |
| Phi-3.5-mini           | ~3GB | Medium | Better  | Fallback option  |
| Gemma-2-2B             | ~2.5GB| Fast  | Good    | Alternative      |
+------------------------+------+--------+---------+------------------+
```

**Ollama Configuration:**

```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Pull model
ollama pull qwen2.5:1.5b-instruct-q4_K_M

# Configure memory behavior (in environment)
export OLLAMA_KEEP_ALIVE=2m  # Unload model after 2 min idle
```

**Feature Flag System:**

```python
class UserFeatureFlags(BaseModel):
    user_id: str
    local_llm_enabled: bool = False
    local_llm_granted_at: Optional[datetime] = None
    
# Admin endpoint (Tailscale-only)
@router.post("/admin/users/{user_id}/features/local-llm")
async def toggle_local_llm(
    user_id: str,
    enabled: bool,
    request: Request
):
    await update_feature_flag(user_id, "local_llm_enabled", enabled)
    await log_admin_action(
        admin_ip=request.client.host,
        action=f"Set local_llm={enabled} for {user_id}"
    )
    return {"status": "updated"}
```

**LLM Prompt:**

```python
LOCAL_LLM_PROMPT = """Analyze this transcript segment to identify visual moments.

A "visual moment" is when:
1. The speaker explicitly references something visible
2. On-screen content (code, diagrams, text) is being discussed
3. A physical demonstration is happening
4. Visual comparison or transition occurs
5. Context would be significantly lost without seeing the video

Be selective. Not every timestamp is a visual moment.

Return JSON only:
{
  "moments": [
    {
      "timestamp_seconds": <number>,
      "confidence": <0.0-1.0>,
      "reason": "<brief explanation>",
      "visual_type": "<diagram|code|demonstration|ui|comparison|other>"
    }
  ]
}

TRANSCRIPT:
{transcript}
"""
```

### 9.4 Mode Switching UI

```tsx
const MomentDetectionSettings = () => {
  const { user, features } = useAuth();
  const [mode, setMode] = useDetectionMode();
  
  return (
    <div className="space-y-4">
      <h3 className="font-medium">Moment Detection</h3>
      
      <RadioGroup value={mode} onValueChange={setMode}>
        <RadioItem value="rules">
          <div className="flex items-center gap-2">
            <RulerIcon className="w-4 h-4" />
            <span>Rules Only</span>
            <Badge variant="secondary">Fast</Badge>
          </div>
          <p className="text-sm text-muted-foreground">
            Pattern matching, no AI. Works instantly.
          </p>
        </RadioItem>
        
        <RadioItem 
          value="local"
          disabled={!features?.localLlmEnabled}
        >
          <div className="flex items-center gap-2">
            <CpuIcon className="w-4 h-4" />
            <span>Rules + Local AI</span>
            {features?.localLlmEnabled && (
              <Badge variant="outline">Enabled</Badge>
            )}
          </div>
          <p className="text-sm text-muted-foreground">
            Smarter detection. Admin-granted access.
          </p>
          {!features?.localLlmEnabled && (
            <p className="text-xs text-muted-foreground">
              Contact admin to enable this feature.
            </p>
          )}
        </RadioItem>
      </RadioGroup>
    </div>
  );
};
```

### 9.5 Result Merging & Deduplication

```python
async def detect_moments(
    transcript: Transcript,
    mode: DetectionMode,
    user: Optional[User]
) -> List[DetectedMoment]:
    """Run detection based on mode, merge results."""
    all_moments: List[DetectedMoment] = []
    
    # Tier 1: Rules (always)
    rule_moments = await detect_with_rules(transcript)
    all_moments.extend(rule_moments)
    
    # Tier 2: Local LLM (if enabled and authorized)
    if mode == DetectionMode.LOCAL and user and user.features.local_llm_enabled:
        llm_moments = await detect_with_ollama(transcript)
        all_moments.extend(llm_moments)
    
    # Deduplicate: moments within 5 seconds are merged
    merged = merge_nearby_moments(all_moments, threshold_seconds=5)
    
    # Sort by timestamp
    merged.sort(key=lambda m: m.timestamp)
    
    # Combine confidences for duplicates
    for moment in merged:
        if moment.sources_count > 1:
            moment.confidence = min(moment.confidence + 0.15, 1.0)
    
    return merged
```

---

## 10. Video Segment Strategy

### 10.1 Segment-Based Downloads

**Decision:** Download video in segments on-demand, not complete files.

**Context:**  
Full video downloads are wasteful -- a 1-hour video at 360p is ~200-400MB. Users only need to preview specific portions to pick moments. YouTube's underlying format (DASH/HLS) is already segmented.

**Implementation:**

```bash
yt-dlp --download-sections "*00:05:00-00:05:30" \
       --format "best[height<=720][height>=360]" \
       -o "segments/%(id)s_%(section_start)s.%(ext)s" \
       "https://youtube.com/watch?v=..."
```

**Segment Strategy:**

- Initial transcript fetch: metadata only, no video download
- User seeks to timestamp: trigger segment download centered on that point
- Segment boundaries snap to nearest keyframes (yt-dlp handles this)
- Segments are cached and painted on the player timeline
- Adjacent segment prefetching when user hovers near segment boundaries

**Quality Selection:**

The format string `best[height<=720][height>=360]` selects the best quality between 360p and 720p, maximizing functionality for videos that might only be available in 720p.

```
+------------+-----------------+------------------+-------------------+
| Resolution | Bitrate (approx)| 30s Segment Size | Use Case          |
+------------+-----------------+------------------+-------------------+
| 360p       | 500 kbps        | ~2 MB            | Lower bandwidth   |
| 480p       | 1000 kbps       | ~4 MB            | Balanced          |
| 720p       | 2500 kbps       | ~9 MB            | Best available    |
+------------+-----------------+------------------+-------------------+
```

**Why 720p ceiling:** Sufficient to distinguish visual content clearly. Most "is this a visual moment?" decisions don't require 1080p+, and the storage/bandwidth savings are significant.

### 10.2 yt-dlp Segment Download Precision

**Important caveat:** The `--download-sections` option aligns downloads to keyframes, so the actual downloaded segment may be slightly longer than requested (typically a few seconds on each end). This is expected behavior.

**Potential issues:**

- Some videos (live streams, DRM-protected) don't support sectioned download
- Network interruptions during download require full retry

**Fallback behavior:**

```python
async def download_segment(
    video_id: str,
    start: float,
    end: float,
    output_path: Path
) -> SegmentResult:
    """Download segment with fallback for unsupported videos."""
    try:
        result = await _download_section(video_id, start, end, output_path)
        return result
    except SectionDownloadUnsupported:
        # Fallback: download lowest quality full video, extract section with ffmpeg
        # This is slower but works for all video types
        full_path = await _download_full_video(video_id, quality="worst")
        await _extract_section_ffmpeg(full_path, start, end, output_path)
        return SegmentResult(path=output_path, fallback_used=True)
```

If fallback is triggered, UI shows: "This video requires slower processing. Please wait..."

### 10.3 Transcript Language Selection

When a video has multiple transcript tracks:

```python
async def get_transcript(video_id: str, preferred_lang: Optional[str] = None) -> Transcript:
    """
    Get transcript, prompting for language selection if multiple available.
    """
    available = await list_available_transcripts(video_id)
    
    if not available:
        raise TranscriptUnavailable(
            error_code="VIDEO_NO_TRANSCRIPT",
            message="This video doesn't have captions. Try a different video."
        )
    
    if len(available) == 1:
        return await fetch_transcript(video_id, available[0].lang_code)
    
    if preferred_lang and preferred_lang in [t.lang_code for t in available]:
        return await fetch_transcript(video_id, preferred_lang)
    
    # Multiple available, no preference: return list for user selection
    raise LanguageSelectionRequired(
        available_languages=available,
        message="Multiple transcript languages available. Please select one."
    )
```

**UI Flow:**

```
+---------------------------------------------------------------+
|                                                               |
|   Multiple languages available                                |
|                                                               |
|   +-------------------------------------------------------+   |
|   |  ( ) English (auto-generated)                         |   |
|   |  ( ) English                                          |   |
|   |  (*) Spanish                                          |   |
|   |  ( ) Portuguese (Brazil)                              |   |
|   +-------------------------------------------------------+   |
|                                                               |
|   [  Continue with Spanish  ]                                 |
|                                                               |
+---------------------------------------------------------------+
```

---

## 11. Cache Architecture

### 11.1 Directory Structure

```
/var/cache/yttool/
|-- segments/                    # Video segment files
|   |-- {video_id}_{start}_{end}.mp4
|   '-- ...
|-- frames/                      # Extracted frame images
|   '-- {video_id}_{timestamp}.jpg
'-- metadata/                    # Transcript + video info
    '-- {video_id}.json
```

### 11.2 LRU Cache with Session-Aware Locking

**Problem:**  
With a 10GB cache ceiling, a rotation policy might evict segments that a user is actively previewing. This would cause playback failures.

**Solution:** LRU cache with session-aware locking. Locks are persisted in SQLite and loaded on startup to survive server restarts.

```python
from dataclasses import dataclass
from pathlib import Path
from typing import Dict, Set
import asyncio

@dataclass
class CachedSegment:
    id: str
    video_id: str
    start_time: float
    end_time: float
    path: Path
    size_bytes: int
    cached_at: float
    last_accessed_at: float


class CacheManager:
    def __init__(self, cache_dir: Path, db: Database, max_size_gb: float = 10.0):
        self.cache_dir = cache_dir
        self.db = db
        self.max_bytes = int(max_size_gb * 1024**3)
        self.segments_dir = cache_dir / "segments"
        self.frames_dir = cache_dir / "frames"
        self.metadata_dir = cache_dir / "metadata"
        
        # In-memory lock cache (mirrors DB for fast access)
        self._locks: Dict[str, Set[str]] = {}
        
        for d in [self.segments_dir, self.frames_dir, self.metadata_dir]:
            d.mkdir(parents=True, exist_ok=True)
    
    async def initialize(self):
        """Load locks from database on startup."""
        rows = await self.db.fetchall("""
            SELECT session_id, segment_id FROM segment_locks
            WHERE expires_at > datetime('now')
        """)
        for row in rows:
            session_id = row["session_id"]
            segment_id = row["segment_id"]
            if session_id not in self._locks:
                self._locks[session_id] = set()
            self._locks[session_id].add(segment_id)
    
    async def acquire_segment(
        self, 
        session_id: str, 
        video_id: str,
        start_time: float,
        end_time: float
    ) -> Path:
        """Get segment path, downloading if needed. Locks segment to session."""
        segment_id = f"{video_id}_{int(start_time)}_{int(end_time)}"
        segment_path = self.segments_dir / f"{segment_id}.mp4"
        
        # Lock in memory
        if session_id not in self._locks:
            self._locks[session_id] = set()
        self._locks[session_id].add(segment_id)
        
        # Persist lock to DB
        await self._persist_lock(session_id, segment_id)
        
        # Download if not cached
        if not segment_path.exists():
            await self._download_segment(video_id, start_time, end_time, segment_path)
        
        await self._touch_segment(segment_id)
        asyncio.create_task(self._maybe_evict())
        
        return segment_path
    
    async def _persist_lock(self, session_id: str, segment_id: str):
        """Persist lock to database."""
        expires_at = datetime.utcnow() + timedelta(minutes=5)
        await self.db.execute("""
            INSERT INTO segment_locks (session_id, segment_id, expires_at)
            VALUES (?, ?, ?)
            ON CONFLICT(session_id, segment_id) DO UPDATE SET expires_at = ?
        """, (session_id, segment_id, expires_at, expires_at))
    
    async def release_session_locks(self, session_id: str):
        """Release all locks for a session (on timeout or disconnect)."""
        if session_id in self._locks:
            del self._locks[session_id]
        
        await self.db.execute(
            "DELETE FROM segment_locks WHERE session_id = ?",
            (session_id,)
        )
        
        asyncio.create_task(self._maybe_evict())
    
    async def _maybe_evict(self):
        """Evict oldest unlocked segments until under max_size."""
        current_size = await self._calculate_current_size()
        
        if current_size <= self.max_bytes:
            return
        
        # Gather all locked segments
        locked_segments: Set[str] = set()
        for session_locks in self._locks.values():
            locked_segments.update(session_locks)
        
        segments = await self._get_segments_sorted_by_access()
        bytes_to_free = current_size - self.max_bytes
        bytes_freed = 0
        
        for segment in segments:
            if bytes_freed >= bytes_to_free:
                break
                
            if segment.id in locked_segments:
                continue
            
            try:
                segment.path.unlink()
                bytes_freed += segment.size_bytes
            except Exception as e:
                logger.error(f"Failed to evict {segment.id}: {e}")
```

### 11.3 Failure Scenarios & UX

```
+----------------------------------+-------------------------+--------------------+
| Scenario                         | System Behavior         | User Experience    |
+----------------------------------+-------------------------+--------------------+
| Segment evicted while locked     | Should never happen     | N/A                |
|                                  | (lock prevents this)    |                    |
+----------------------------------+-------------------------+--------------------+
| User inactive > 2 min            | Locks released,         | Segments may need  |
|                                  | data retained           | re-download; shows |
|                                  |                         | "Refreshing..."    |
+----------------------------------+-------------------------+--------------------+
| Cache full, all segments locked  | Log warning, refuse     | "Server busy,      |
|                                  | new segment requests    | try again shortly" |
+----------------------------------+-------------------------+--------------------+
| Segment download fails (network) | Retry 2x with backoff   | "Couldn't load,    |
|                                  |                         | retrying..."       |
+----------------------------------+-------------------------+--------------------+
| Server restart                   | Locks loaded from DB    | Transparent        |
+----------------------------------+-------------------------+--------------------+
```

---

## 12. Process Management & Concurrency Control

### 12.1 Task Queue: Huey with SQLite

**Decision:** Use `huey` with SQLite backend for task queue, plus asyncio semaphores for resource limits.

**Why Huey?**  
Lightweight, SQLite-backed (no Redis required), good enough for our scale.

**Configuration:**

```python
from huey import SqliteHuey

# Worker count: 1 by default, configurable via environment
HUEY_WORKERS = int(os.getenv("HUEY_WORKERS", "1"))

huey = SqliteHuey(
    filename='/var/lib/yttool/tasks.db',
    immediate=False,
)

@huey.task()
def extract_frames_task(video_id: str, timestamps: list[float]):
    """Extract frames at given timestamps. Runs in background."""
    for ts in timestamps:
        extract_single_frame(video_id, ts)
    return {"status": "complete", "count": len(timestamps)}

@huey.periodic_task(crontab(minute='0', hour='3'))
def cleanup_expired_data():
    """Daily cleanup of expired sessions, rate limits, and orphaned cache."""
    cleanup_expired_sessions(max_age_hours=24)
    cleanup_expired_rate_limits()
    cleanup_orphaned_cache_entries()
```

**Running the worker:**

```bash
huey_consumer app.tasks.huey --workers ${HUEY_WORKERS:-1} --worker-type thread
```

### 12.2 Resource Limiters

```python
from contextlib import asynccontextmanager
import asyncio

class ResourceLimiter:
    """Manages concurrency limits for resource-intensive operations."""
    
    def __init__(self):
        self.llm = asyncio.Semaphore(1)      # Only 1 LLM at a time
        self.ffmpeg = asyncio.Semaphore(2)   # Max 2 ffmpeg processes
        self.download = asyncio.Semaphore(3) # Max 3 concurrent downloads
    
    @asynccontextmanager
    async def llm_slot(self):
        async with self.llm:
            yield
    
    @asynccontextmanager
    async def ffmpeg_slot(self):
        async with self.ffmpeg:
            yield
    
    @asynccontextmanager
    async def download_slot(self):
        async with self.download:
            yield

limiter = ResourceLimiter()

# Usage
async def analyze_transcript_handler(video_id: str):
    async with limiter.llm_slot():
        result = await run_llm_analysis(video_id)
    return result
```

### 12.3 Concurrency Limits (Tuned for KVM 4)

```
+---------------------+----------------+----------------------------------+
| Resource            | Max Concurrent | Rationale                        |
+---------------------+----------------+----------------------------------+
| LLM Inference       | 1              | Memory-bound; model is ~2GB      |
| FFmpeg (frames)     | 2              | CPU-bound; leave cores for others|
| yt-dlp downloads    | 3              | Network-bound; parallelism helps |
| Total active sessions| 5             | Prevents cache explosion         |
+---------------------+----------------+----------------------------------+
```

### 12.4 Queue Priority

Processing priority (highest to lowest):

1. **Frame extraction** -- user is actively waiting
2. **Segment download** -- user is actively waiting
3. **LLM analysis** -- background, can be slower
4. **Image upload** -- background, can be batched

---

## 13. Data Models

### 13.1 TypeScript Interfaces (Frontend)

```typescript
interface VideoMetadata {
  id: string;
  title: string;
  description: string;
  duration: number;
  channelName: string;
  thumbnailUrl: string;
  availableLanguages: TranscriptLanguage[];
  fetchedAt: string;
}

interface TranscriptLanguage {
  code: string;       // e.g., "en", "es", "pt-BR"
  name: string;       // e.g., "English", "Spanish"
  isAutoGenerated: boolean;
}

interface Transcript {
  videoId: string;
  language: string;
  segments: TranscriptSegment[];
}

interface TranscriptSegment {
  start: number;
  duration: number;
  text: string;
}

interface VideoSegment {
  id: string;
  videoId: string;
  startTime: number;
  endTime: number;
  quality: '360p' | '480p' | '720p';
  status: 'pending' | 'downloading' | 'ready' | 'error';
}

interface VisualMoment {
  id: string;
  videoId: string;
  timestamp: number;
  source: 'user' | 'rule' | 'llm';
  confidence: number;
  reason?: string;
  status: 'pending' | 'confirmed' | 'dismissed' | 'extracted' | 'uploaded';
  framePath?: string;
  uploadedUrl?: string;
}

interface UserSession {
  id: string;
  type: 'anonymous' | 'identified';
  userId?: string;
  videoId?: string;
  createdAt: string;
  lastActiveAt: string;
  selectedMoments: VisualMoment[];
}

interface User {
  id: string;
  email?: string;
  hankoId: string;
  features: UserFeatures;
  createdAt: string;
}

interface UserFeatures {
  localLlmEnabled: boolean;
  rateLimitMultiplier?: number;
}

interface ExportArtifact {
  id: string;
  videoId: string;
  title: string;
  format: 'markdown' | 'json';
  includesImages: boolean;
  momentCount: number;
  exportedAt: string;
  downloadUrl: string;
}
```

### 13.2 SQLite Schema (Backend)

```sql
-- Video metadata cache
CREATE TABLE videos (
    id TEXT PRIMARY KEY,
    title TEXT NOT NULL,
    description TEXT,
    duration_seconds INTEGER NOT NULL,
    channel_name TEXT,
    thumbnail_url TEXT,
    available_languages_json TEXT,  -- JSON array of language objects
    fetched_at TEXT NOT NULL
);

-- Transcript cache
CREATE TABLE transcripts (
    id TEXT PRIMARY KEY,           -- "{video_id}_{lang_code}"
    video_id TEXT NOT NULL REFERENCES videos(id),
    language_code TEXT NOT NULL,
    segments_json TEXT NOT NULL,
    fetched_at TEXT NOT NULL
);

CREATE INDEX idx_transcripts_video ON transcripts(video_id);

-- Cached video segments metadata
CREATE TABLE cached_segments (
    id TEXT PRIMARY KEY,
    video_id TEXT NOT NULL REFERENCES videos(id),
    start_time REAL NOT NULL,
    end_time REAL NOT NULL,
    quality TEXT NOT NULL,
    file_path TEXT NOT NULL,
    size_bytes INTEGER NOT NULL,
    cached_at TEXT NOT NULL,
    last_accessed_at TEXT NOT NULL
);

CREATE INDEX idx_segments_video ON cached_segments(video_id);
CREATE INDEX idx_segments_access ON cached_segments(last_accessed_at);

-- Segment locks (persisted for server restart survival)
CREATE TABLE segment_locks (
    session_id TEXT NOT NULL,
    segment_id TEXT NOT NULL,
    expires_at TEXT NOT NULL,
    PRIMARY KEY (session_id, segment_id)
);

CREATE INDEX idx_locks_expires ON segment_locks(expires_at);

-- Anonymous sessions
CREATE TABLE anonymous_sessions (
    id TEXT PRIMARY KEY,
    video_id TEXT REFERENCES videos(id),
    created_at TEXT NOT NULL,
    last_active_at TEXT NOT NULL,
    selected_moments_json TEXT DEFAULT '[]'
);

CREATE INDEX idx_anon_sessions_active ON anonymous_sessions(last_active_at);

-- Identified users (synced with Hanko)
CREATE TABLE users (
    id TEXT PRIMARY KEY,
    hanko_id TEXT UNIQUE NOT NULL,
    email TEXT,
    created_at TEXT NOT NULL,
    last_login_at TEXT
);

-- User feature flags
CREATE TABLE user_features (
    user_id TEXT PRIMARY KEY REFERENCES users(id),
    local_llm_enabled INTEGER DEFAULT 0,
    local_llm_granted_at TEXT,
    rate_limit_multiplier REAL DEFAULT 1.0
);

-- Identified user sessions
CREATE TABLE user_sessions (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL REFERENCES users(id),
    video_id TEXT REFERENCES videos(id),
    created_at TEXT NOT NULL,
    last_active_at TEXT NOT NULL,
    selected_moments_json TEXT DEFAULT '[]'
);

CREATE INDEX idx_user_sessions_user ON user_sessions(user_id);
CREATE INDEX idx_user_sessions_active ON user_sessions(last_active_at);

-- Rate limiting
CREATE TABLE rate_limits (
    key TEXT PRIMARY KEY,
    count INTEGER NOT NULL,
    window_start TEXT NOT NULL,
    expires_at TEXT NOT NULL
);

CREATE INDEX idx_rate_limits_expires ON rate_limits(expires_at);

-- LLM auto-suggestions (cached per video)
CREATE TABLE auto_suggestions (
    id TEXT PRIMARY KEY,
    video_id TEXT NOT NULL REFERENCES videos(id),
    timestamp_seconds REAL NOT NULL,
    confidence REAL NOT NULL,
    reason TEXT,
    visual_type TEXT,
    source TEXT NOT NULL,          -- 'rule' or 'ollama'
    model_version TEXT,
    suggested_at TEXT NOT NULL
);

CREATE INDEX idx_suggestions_video ON auto_suggestions(video_id);

-- WebSocket events (for replay on reconnect)
CREATE TABLE ws_events (
    id TEXT PRIMARY KEY,
    session_id TEXT NOT NULL,
    event_type TEXT NOT NULL,
    payload_json TEXT NOT NULL,
    created_at TEXT NOT NULL
);

CREATE INDEX idx_ws_events_session ON ws_events(session_id, created_at);

-- Admin audit log
CREATE TABLE admin_audit_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp TEXT NOT NULL,
    admin_ip TEXT NOT NULL,
    action TEXT NOT NULL,
    target_user_id TEXT,
    details_json TEXT
);
```

---

## 14. API Specification

### 14.1 CORS Configuration

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        f"https://{os.getenv('DOMAIN')}",
        "http://localhost:3000",  # Development
    ],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PATCH", "DELETE"],
    allow_headers=["*"],
    expose_headers=["X-RateLimit-Limit", "X-RateLimit-Remaining", "X-RateLimit-Reset"],
)
```

### 14.2 Endpoints Overview

```
+--------+----------------------------------+----------------------------------------+
| Method | Path                             | Purpose                                |
+--------+----------------------------------+----------------------------------------+
| POST   | /api/videos/extract              | Fetch metadata + transcript for URL    |
| GET    | /api/videos/{id}                 | Get cached video metadata              |
| GET    | /api/videos/{id}/transcript      | Get transcript (with language param)   |
| GET    | /api/videos/{id}/languages       | List available transcript languages    |
| POST   | /api/videos/{id}/analyze         | Trigger moment detection               |
| GET    | /api/videos/{id}/suggestions     | Get detected moments                   |
+--------+----------------------------------+----------------------------------------+
| POST   | /api/sessions                    | Create new session                     |
| POST   | /api/sessions/{id}/heartbeat     | Keep session alive                     |
| DELETE | /api/sessions/{id}               | End session, release locks             |
| POST   | /api/sessions/{id}/upgrade       | Upgrade anonymous to identified        |
+--------+----------------------------------+----------------------------------------+
| POST   | /api/segments/request            | Request segment download               |
| GET    | /api/segments/{id}/stream        | Stream segment video                   |
| GET    | /api/segments/status             | Get cache status for video             |
+--------+----------------------------------+----------------------------------------+
| POST   | /api/moments                     | Add user-selected moment               |
| PATCH  | /api/moments/{id}                | Update moment (confirm, adjust)        |
| DELETE | /api/moments/{id}                | Remove moment                          |
| POST   | /api/moments/{id}/extract-frame  | Extract frame image                    |
| POST   | /api/moments/batch-upload        | Upload frames to image host            |
+--------+----------------------------------+----------------------------------------+
| POST   | /api/export                      | Generate final artifact                |
| GET    | /api/export/{id}/download        | Download generated artifact            |
+--------+----------------------------------+----------------------------------------+
| WS     | /ws/session/{id}                 | Real-time updates (progress, etc.)     |
+--------+----------------------------------+----------------------------------------+
| GET    | /admin/dashboard                 | Admin overview (Tailscale only)        |
| GET    | /admin/users                     | List users (Tailscale only)            |
| GET    | /admin/users/{id}                | User details (Tailscale only)          |
| POST   | /admin/users/{id}/features/*     | Toggle features (Tailscale only)       |
| GET    | /admin/cache                     | Cache stats (Tailscale only)           |
| POST   | /admin/cache/clear               | Clear cache (Tailscale only)           |
+--------+----------------------------------+----------------------------------------+
```

### 14.3 Key Endpoint Details

**POST /api/videos/extract**

```typescript
// Request
{ 
  url: string;
  preferredLanguage?: string;  // Optional language code
}

// Response (success)
{
  video: VideoMetadata;
  transcript: Transcript;
  cached: boolean;
}

// Response (language selection required)
{
  error: "language_selection_required",
  video: VideoMetadata,
  availableLanguages: TranscriptLanguage[]
}

// Errors
// 400: Invalid URL
// 404: VIDEO_NOT_FOUND - Video not found or unavailable
// 404: VIDEO_NO_TRANSCRIPT - Video has no captions
// 429: Rate limit exceeded
```

**POST /api/segments/request**

```typescript
// Request
{
  sessionId: string;
  videoId: string;
  timestamp: number;    // Center point in seconds
  duration?: number;    // Default 30
}

// Response
{
  segmentId: string;
  status: 'ready' | 'downloading' | 'queued';
  estimatedWaitMs?: number;
  streamUrl?: string;
  actualStart: number;  // May differ due to keyframe snapping
  actualEnd: number;
}
```

**POST /api/videos/{id}/analyze**

```typescript
// Request
{
  sessionId: string;
  mode: 'rules' | 'local';
  transcriptLanguage: string;
}

// Response
{
  taskId: string;
  status: 'processing' | 'complete';
  moments?: VisualMoment[];
}

// Errors
// 403: Local mode requires feature flag
```

**POST /api/export**

```typescript
// Request
{
  sessionId: string;
  videoId: string;
  moments: string[];
  format: 'markdown' | 'json';
  options: {
    includeImages: boolean;
    includeTimestampLinks: boolean;
  };
}

// Response
{
  exportId: string;
  status: 'processing' | 'ready';
  downloadUrl?: string;
}
```

### 14.4 WebSocket Protocol

**Connection:** `wss://{domain}/ws/session/{sessionId}?lastEventId={optional}`

**Server -> Client Messages:**

```typescript
// All messages include eventId for replay support
interface WSMessage {
  eventId: string;
  type: string;
  // ... type-specific fields
}

// Segment download progress
{
  eventId: "evt_abc123",
  type: "segment_progress",
  segmentId: string,
  progress: number,  // 0-100
  status: "downloading" | "ready" | "error"
}

// Analysis progress
{
  eventId: "evt_abc124",
  type: "analysis_progress",
  videoId: string,
  stage: "rules" | "llm",
  progress: number,
  momentsFound: number
}

// New moment detected
{
  eventId: "evt_abc125",
  type: "moment_detected",
  moment: VisualMoment
}

// Export ready
{
  eventId: "evt_abc126",
  type: "export_ready",
  exportId: string,
  downloadUrl: string
}

// Session warning
{
  eventId: "evt_abc127",
  type: "session_warning",
  message: string  // e.g., "Session expiring in 1 minute"
}
```

**Client -> Server Messages:**

```typescript
// Heartbeat (every 30s)
{ type: "heartbeat" }

// Acknowledge moment
{
  type: "moment_ack",
  momentId: string,
  action: "confirm" | "dismiss"
}
```

---

## 15. Export Artifact Format

### 15.1 Design for AI Consumption

**Decision:** Primary format is **Markdown with base64-embedded images**.

**Why Base64?**

- **Self-contained:** No external dependencies, works offline
- **LLM-compatible:** Claude, GPT-4V, and Gemini all accept base64 images
- **Portable:** Single file, easy to share/archive
- **No link rot:** External image hosts can go down; base64 is forever

**Image Format:**

- **Format:** JPEG (good balance of quality and size)
- **Quality:** 80% (yields ~50-150KB per frame at 720p)
- **Resolution:** Native frame resolution from video segment

Note: The 80% quality setting is a starting point. Experimentation may reveal that 70% is sufficient for AI comprehension or that 90% is needed for certain diagram-heavy content. Adjust based on artifact size vs. AI accuracy tradeoffs.

### 15.2 Markdown Format Specification

```markdown
# [Video Title]

**Source:** https://youtube.com/watch?v=VIDEO_ID
**Channel:** [Channel Name]
**Duration:** [HH:MM:SS]
**Extracted:** [ISO Date]
**Visual Moments:** [Count]

---

## Description

[Original video description, cleaned up]

---

## Transcript

[00:00] Welcome to today's tutorial on building responsive layouts.

[00:15] As you can see here, we have our basic HTML structure.

![Visual context at 00:15 - Code editor showing HTML](data:image/jpeg;base64,/9j/4AAQSkZJRg...)

[00:32] The CSS flexbox model works by distributing space along a single axis.

[00:45] Let me show you this diagram that explains the concept.

![Visual context at 00:45 - Flexbox diagram](data:image/jpeg;base64,/9j/4AAQSkZJRg...)

[01:02] Now we'll implement this in our project.

...

---

## Visual Moments Index

| Timestamp | Description                                   | Type    |
|:---------:|:----------------------------------------------|:-------:|
|  00:15    | Code editor showing HTML structure            |  code   |
|  00:45    | Flexbox diagram explaining axis distribution  | diagram |
|  02:30    | Browser DevTools demonstration                |   ui    |

---

*Generated by [Tool Name] - AI-consumable transcript with visual context*
```

### 15.3 JSON Format (Programmatic Use)

```json
{
  "version": "1.0",
  "video": {
    "id": "VIDEO_ID",
    "title": "Video Title",
    "channel": "Channel Name",
    "duration_seconds": 1234,
    "url": "https://youtube.com/watch?v=VIDEO_ID"
  },
  "description": "Original description...",
  "transcript": [
    {
      "start": 0.0,
      "duration": 3.5,
      "text": "Welcome to today's tutorial"
    }
  ],
  "visual_moments": [
    {
      "timestamp": 15.0,
      "description": "Code editor showing HTML structure",
      "type": "code",
      "image_base64": "/9j/4AAQSkZJRg...",
      "image_mime": "image/jpeg"
    }
  ],
  "metadata": {
    "extracted_at": "2024-12-14T10:30:00Z",
    "tool_version": "1.0.0",
    "detection_mode": "rules"
  }
}
```

---

## 16. Admin Dashboard

### 16.1 Access Method

**URL:** `http://{tailscale-hostname}:8443/admin` (Tailscale only)

No login required -- if you can reach it, you're authorized (Tailscale identity).

### 16.2 Dashboard Sections

**Overview:** System health (CPU, RAM, cache usage), active sessions, queue depth.

**User Management:** List users, view usage stats, toggle feature flags.

**Cache Management:** View cache size breakdown, clear unlocked segments.

**Admin Capabilities:**

```
+------------------------+------------------------------------------+
| Capability             | Description                              |
+------------------------+------------------------------------------+
| User Management        | View all users, usage stats              |
| Feature Flags          | Toggle local LLM per user                |
| Rate Limit Override    | Adjust multiplier for specific users     |
| Cache Management       | View/clear cache, see stats              |
| System Health          | Resource usage, queue depth              |
+------------------------+------------------------------------------+
```

---

## 17. Visual Polish & Cool Factor

### 17.1 Framework Stack

```
shadcn/ui (foundation)
        |
        v
Framer Motion (micro-interactions)
        |
        v
Aceternity UI / Magic UI (hero effects) -- selective use
```

**Component Strategy:**

```
+----------------------------------+------------------------+
| Component Need                   | Solution               |
+----------------------------------+------------------------+
| Forms, buttons, dialogs, inputs  | shadcn/ui              |
| Transcript list (virtual scroll) | @tanstack/react-virtual|
| Video segment player             | Custom build           |
| Timeline with segment indicators | Custom build           |
| Frame review gallery             | shadcn/ui + Framer     |
| Loading states, skeletons        | shadcn/ui              |
| Toasts, notifications            | sonner                 |
| Micro-interactions               | Framer Motion          |
+----------------------------------+------------------------+
```

### 17.2 Key "Wow" Moments

```
+--------------------+--------------------------------+------------------+
| Screen             | Effect                         | Library          |
+--------------------+--------------------------------+------------------+
| Landing hero       | Aurora gradient + text reveal  | Aceternity       |
| Transcript load    | Staggered line fade-in         | Framer Motion    |
| Moment picker mode | Spotlight cursor + UI expand   | Custom + Framer  |
| Moment confirmation| Particle burst + pulse         | Framer + Custom  |
| Frame gallery      | Masonry + physics drag         | Framer Reorder   |
| Export ready       | Confetti + button materialize  | Custom           |
+--------------------+--------------------------------+------------------+
```

### 17.3 Performance Budget

**Golden Rule:** One "hero" effect per view. Don't stack multiple expensive animations.

```
+----------------------+------------------+---------------------------------+
| Effect               | Performance Cost | Mitigation                      |
+----------------------+------------------+---------------------------------+
| Framer Motion        | Low              | Use layout prop wisely          |
| Aceternity effects   | Varies           | Pick lightweight ones           |
| Particle effects     | Medium-High      | Limit count, use CSS if possible|
+----------------------+------------------+---------------------------------+
```

---

## 18. Deployment Architecture

### 18.1 Process Layout

```
+----------------------------------------------------------------------+
|                         Hostinger KVM 4                              |
|                                                                      |
|  +----------------------------------------------------------------+  |
|  |  Caddy (Reverse Proxy)                                         |  |
|  |  - TLS termination (public)                                    |  |
|  |  - Route /api/* --> FastAPI :8000                              |  |
|  |  - Route /hanko/* --> Hanko :8001                              |  |
|  |  - Route /* --> Next.js :3000                                  |  |
|  |  - Tailscale listener :8443 for /admin/*                       |  |
|  +----------------------------------------------------------------+  |
|                              |                                       |
|         +--------------------+--------------------+                  |
|         v                    v                    v                  |
|  +-------------+      +-------------+      +-------------+           |
|  |  Next.js    |      |  FastAPI    |      |  Hanko      |           |
|  |  :3000      |      |  :8000      |      |  :8001      |           |
|  +-------------+      +-------------+      +-------------+           |
|                              |                                       |
|              +---------------+---------------+                       |
|              v                               v                       |
|       +-------------+                 +-------------+                |
|       |  Huey       |                 |  Ollama     |                |
|       |  Worker     |                 |  :11434     |                |
|       |  (1 thread) |                 |  (on-demand)|                |
|       +-------------+                 +-------------+                |
|                                                                      |
|  +----------------------------------------------------------------+  |
|  |  Persistent Storage                                            |  |
|  |  /var/lib/yttool/                                              |  |
|  |  |-- data.db          (SQLite main database)                   |  |
|  |  |-- tasks.db         (Huey task queue)                        |  |
|  |  '-- cache/           (Video segments, frames)                 |  |
|  +----------------------------------------------------------------+  |
|                                                                      |
|  +----------------------------------------------------------------+  |
|  |  PostgreSQL (for Hanko only)                                   |  |
|  +----------------------------------------------------------------+  |
|                                                                      |
+----------------------------------------------------------------------+
```

---

## 19. Implementation Phases

### Phase 1: Foundation (Week 1-2)

- [ ] FastAPI + SQLite setup
- [ ] yt-dlp transcript extraction with language selection
- [ ] Segment download with quality selection
- [ ] Basic cache manager (with lock persistence)
- [ ] CLI for testing

**Deliverable:** `python cli.py "youtube-url"` outputs transcript JSON

### Phase 2: Auth Foundation (Week 2-3)

- [ ] Hanko self-hosted deployment
- [ ] Hanko frontend components integration
- [ ] Anonymous session system
- [ ] Session lifecycle (heartbeat, timeout, upgrade)
- [ ] Tailscale admin route protection via Caddy

**Deliverable:** Users can sign up with passkey or email, sessions persist

### Phase 3: Frontend Foundation (Week 3-4)

- [ ] Next.js + shadcn setup
- [ ] Mobile-first responsive layout
- [ ] Touch target standards implementation
- [ ] URL input + metadata display
- [ ] Virtualized transcript viewer

**Deliverable:** Web page that shows transcript for a YouTube URL

### Phase 4: Moments Universe - Core (Week 4-5)

- [ ] Custom segment player component
- [ ] Timeline with segment visualization
- [ ] Manual moment selection
- [ ] Touch-friendly interactions
- [ ] WebSocket for real-time updates

**Deliverable:** Working video preview with segment-based playback and moment selection

### Phase 5: Moments Universe - Polish (Week 5-6)

- [ ] Picker mode (full-screen, spotlight cursor)
- [ ] Frame extraction backend
- [ ] Frame review gallery
- [ ] Animations + micro-interactions
- [ ] Mobile gesture refinements

**Deliverable:** Beautiful, tactile moment picking experience

### Phase 6: Detection Engine (Week 6-7)

- [ ] Rule-based detection (all patterns)
- [ ] Ollama setup with Qwen2.5-1.5B
- [ ] Feature flag system
- [ ] Result merging + deduplication
- [ ] Mode switching UI

**Deliverable:** Auto-suggested moments from rules and/or local LLM

### Phase 7: Admin Dashboard (Week 7-8)

- [ ] Admin UI (Tailscale-only)
- [ ] User management
- [ ] Feature flag controls
- [ ] Cache management
- [ ] Rate limit configuration

**Deliverable:** Admin can manage users and grant local LLM access

### Phase 8: Export + Final Polish (Week 8-9)

- [ ] Markdown artifact generation with base64 images
- [ ] JSON export option
- [ ] Export download flow
- [ ] Image upload to external host (optional path)
- [ ] Cool factor effects (hero, transitions, etc.)

**Deliverable:** Complete workflow from URL to downloadable AI-ready artifact

### Phase 9: Testing + Hardening (Week 9-10)

- [ ] Edge case handling
- [ ] Error states + recovery flows
- [ ] Performance optimization
- [ ] Mobile device testing (various sizes)
- [ ] Load testing (concurrent sessions)

**Deliverable:** Production-ready application

---

## 20. Risk Assessment

```
+---------------------------------+------------+--------+---------------------------+
| Risk                            | Likelihood | Impact | Mitigation                |
+---------------------------------+------------+--------+---------------------------+
| YouTube blocks yt-dlp           | Medium     | High   | Monitor releases; fallback|
|                                 |            |        | to Invidious API          |
+---------------------------------+------------+--------+---------------------------+
| KVM 4 runs out of RAM           | Medium     | Medium | Strict limits; swap file  |
|                                 |            |        | as safety net; LLM gated  |
+---------------------------------+------------+--------+---------------------------+
| LLM quality insufficient        | Low        | Medium | Good prompts; rules as    |
|                                 |            |        | primary tier              |
+---------------------------------+------------+--------+---------------------------+
| Cache fills during high use     | Medium     | Low    | Aggressive eviction;      |
|                                 |            |        | session limits            |
+---------------------------------+------------+--------+---------------------------+
| Segment player too complex      | Medium     | Medium | Start simple; iterate;    |
|                                 |            |        | fallback to timestamp only|
+---------------------------------+------------+--------+---------------------------+
| imgbb rate limits               | Low        | Low    | Batch uploads; user API   |
|                                 |            |        | key option                |
+---------------------------------+------------+--------+---------------------------+
| Hanko upgrade breaks things     | Low        | Medium | Pin version; test upgrades|
|                                 |            |        | in staging                |
+---------------------------------+------------+--------+---------------------------+
```

---

## 21. Open Questions

1. **Branding:** What should this tool be called?
2. **Domain:** What domain will this run on?
3. **Monetization:** Future consideration for premium features?
4. **Multi-language:** Priority for non-English transcript improvements?
5. **Collaboration:** Multiple users working on same video?
6. **Public gallery:** Should finished artifacts be shareable publicly?

---

## Appendix A: Dependencies

### A.1 Backend (Python)

```
# requirements.txt
fastapi>=0.109.0
uvicorn[standard]>=0.27.0
yt-dlp>=2024.11.0
ffmpeg-python>=0.2.0
huey>=2.5.0
httpx>=0.26.0
pydantic>=2.5.0
pydantic-settings>=2.1.0
python-multipart>=0.0.6
websockets>=12.0
aiosqlite>=0.19.0
python-jose[cryptography]>=3.3.0
cryptography>=42.0.0
python-dotenv>=1.0.0
structlog>=24.1.0
```

### A.2 Frontend (Node)

```json
{
  "dependencies": {
    "next": "^14.1.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "@teamhanko/hanko-elements": "^0.8.0",
    "framer-motion": "^11.0.0",
    "@tanstack/react-virtual": "^3.0.0",
    "@radix-ui/react-dialog": "^1.0.0",
    "@radix-ui/react-dropdown-menu": "^2.0.0",
    "@radix-ui/react-toast": "^1.1.0",
    "class-variance-authority": "^0.7.0",
    "clsx": "^2.1.0",
    "tailwind-merge": "^2.2.0",
    "tailwindcss": "^3.4.0",
    "tailwindcss-animate": "^1.0.7",
    "lucide-react": "^0.312.0",
    "sonner": "^1.4.0",
    "zustand": "^4.5.0"
  },
  "devDependencies": {
    "typescript": "^5.3.0",
    "@types/react": "^18.2.0",
    "@types/node": "^20.11.0",
    "autoprefixer": "^10.4.0",
    "postcss": "^8.4.0"
  }
}
```

### A.3 System Packages

```bash
# Ubuntu 24.04
apt install -y \
    ffmpeg \
    python3.12 \
    python3.12-venv \
    nodejs \
    npm \
    postgresql-15 \
    caddy

# Ollama (separate installer)
curl -fsSL https://ollama.com/install.sh | sh
ollama pull qwen2.5:1.5b-instruct-q4_K_M
```

### A.4 External Services

```
+-------------------+---------------------------+---------------------------+
| Service           | Purpose                   | Account Required          |
+-------------------+---------------------------+---------------------------+
| imgbb             | Image hosting             | Optional (API key)        |
| Tailscale         | Admin network access      | Yes (your account)        |
| SMTP Provider     | Hanko magic links         | Yes (any provider)        |
+-------------------+---------------------------+---------------------------+
```

---

## Appendix B: Environment Variables

All environment variables required for deployment:

```bash
# =============================================================================
# DOMAIN & NETWORKING
# =============================================================================
DOMAIN=yourdomain.com                    # Public domain for the application
TAILSCALE_HOSTNAME=your-machine          # Tailscale hostname for admin access

# =============================================================================
# HANKO AUTHENTICATION
# =============================================================================
HANKO_SECRET_KEY=<generate-secure-key>   # openssl rand -hex 32
POSTGRES_PASSWORD=<generate-secure-pw>   # For Hanko's PostgreSQL

# =============================================================================
# EMAIL (for Hanko magic links)
# =============================================================================
SMTP_HOST=smtp.yourprovider.com
SMTP_PORT=587
SMTP_USER=your-smtp-username
SMTP_PASSWORD=your-smtp-password

# =============================================================================
# ENCRYPTION (for sensitive data in SQLite)
# =============================================================================
# Used by cryptography.fernet for encrypting tokens at rest
FERNET_KEY=<generate-fernet-key>         # python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"

# =============================================================================
# OPTIONAL: IMAGE HOSTING
# =============================================================================
IMGBB_API_KEY=<your-imgbb-key>           # Optional, for external image uploads

# =============================================================================
# TUNING (all have sensible defaults)
# =============================================================================
HUEY_WORKERS=1                           # Background task workers
CACHE_MAX_SIZE_GB=10                     # Video cache ceiling
OLLAMA_KEEP_ALIVE=2m                     # How long to keep model in memory
```

**Token Encryption Implementation:**

```python
from cryptography.fernet import Fernet
import os

# Initialize once at startup
_fernet = Fernet(os.environ["FERNET_KEY"].encode())

def encrypt_token(plaintext: str) -> str:
    """Encrypt sensitive data before storing in SQLite."""
    return _fernet.encrypt(plaintext.encode()).decode()

def decrypt_token(ciphertext: str) -> str:
    """Decrypt sensitive data retrieved from SQLite."""
    return _fernet.decrypt(ciphertext.encode()).decode()
```

---

## Revision History

```
+---------+------------+--------------------------------------------------------+
| Version | Date       | Changes                                                |
+---------+------------+--------------------------------------------------------+
| 1.0     | 2024-12-14 | Initial draft                                          |
+---------+------------+--------------------------------------------------------+
| 2.0     | 2024-12-14 | Added: Auth architecture, mobile UX, 3-tier LLM,       |
|         |            | Tailscale admin, elevated Moments Universe             |
+---------+------------+--------------------------------------------------------+
| 3.0     | 2024-12-14 | Finalized: Locked Hanko, detailed flows, ASCII         |
|         |            | diagrams, cache/process/data model sections            |
+---------+------------+--------------------------------------------------------+
| 4.0     | 2024-12-15 | Major revision: Removed cloud LLM (Gemini) tier due    |
|         |            | to OAuth architecture incompatibility. Consolidated    |
|         |            | session management and rate limiting into dedicated    |
|         |            | sections. Added memory budget analysis. Fixed cache    |
|         |            | lock persistence (now survives restarts). Added        |
|         |            | transcript language selection. Added WebSocket         |
|         |            | reconnection with event replay. Added CORS config.     |
|         |            | Added environment variables appendix. Removed          |
|         |            | unnecessary systemd configs and plaintext export.      |
|         |            | Clarified segment download precision and fallback.     |
+---------+------------+--------------------------------------------------------+
```

---

*End of Document*
