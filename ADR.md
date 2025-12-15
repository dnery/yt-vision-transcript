# ADR-001: YouTube Transcript Extraction Tool Architecture

**Status:** Proposed  
**Version:** 3.0 (Final Draft)  
**Date:** 2024-12-14  
**Decision Makers:** Project Owner  
**Context:** Personal project, learning-focused, resource-constrained hosting

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [System Context & Constraints](#2-system-context--constraints)
3. [Architecture Overview](#3-architecture-overview)
4. [Authentication & Identity Architecture](#4-authentication--identity-architecture)
5. [Mobile-First Accessibility Design](#5-mobile-first-accessibility-design)
6. [The Moments Universe (Core Feature)](#6-the-moments-universe-core-feature)
7. [Moment Detection Engine (3-Tier System)](#7-moment-detection-engine-3-tier-system)
8. [Video Segment Strategy](#8-video-segment-strategy)
9. [Cache Architecture & Session Locking](#9-cache-architecture--session-locking)
10. [Process Management & Concurrency Control](#10-process-management--concurrency-control)
11. [Data Models](#11-data-models)
12. [API Specification](#12-api-specification)
13. [Export Artifact Format](#13-export-artifact-format)
14. [Admin Dashboard](#14-admin-dashboard)
15. [Visual Polish & Cool Factor](#15-visual-polish--cool-factor)
16. [Deployment Architecture](#16-deployment-architecture)
17. [Implementation Phases](#17-implementation-phases)
18. [Risk Assessment](#18-risk-assessment)
19. [Open Questions](#19-open-questions)
20. [Appendix: Dependencies](#20-appendix-dependencies)
21. [Revision History](#21-revision-history)

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

### 2.2 Functional Requirements

1. Extract title, description, and timestamped transcript from YouTube URLs
2. **The Moments Universe:** A core, beautiful experience for identifying visual moments
3. Preview video segments without downloading entire videos
4. Three-tier moment detection: Rules -> Cloud LLM -> Local LLM
5. Extract frames at selected timestamps
6. Upload frames to external image hosting
7. Generate final AI-consumable artifact with embedded images

### 2.3 Non-Functional Requirements

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
|  |  | Galaxy         | | Controls       | | (WebSocket)            || |
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
|  +------------------------------------------------------------------+ |
|  +------------------------------------------------------------------+ |
|  |                        Service Layer                             | |
|  |  +----------------+ +----------------+ +------------------------+| |
|  |  | Transcript     | | Video Segment  | | Moment Detection       || |
|  |  | Extractor      | | Manager        | | Engine (3-tier)        || |
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
| YouTube          |    | Google Cloud     |    | Tailscale            |
| (via yt-dlp)     |    | - Gemini API     |    | - Admin identity     |
|                  |    | - OAuth          |    | - Network-level ACL  |
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
|   - No persistence       - Cloud LLM access     - Local LLM toggle    |
|   - Rate limited         - Higher rate limits   - Usage dashboards    |
|                          - History              - User management     |
|                                                                       |
+-----------------------------------------------------------------------+
```

### 4.2 Anonymous Sessions

**Every visitor gets full core functionality without any signup.**

**Implementation:**

```typescript
// Client-side session management
interface AnonymousSession {
  id: string;           // UUID, generated on first visit
  createdAt: string;
  lastActiveAt: string;
  currentVideoId?: string;
  selectedMoments: VisualMoment[];
}

// Stored in localStorage, sent as header
// X-Session-ID: uuid-here
```

**Storage Strategy:**

- Session ID stored in `localStorage` (persists across tabs/refreshes)
- Fallback to `sessionStorage` if localStorage unavailable
- Server maintains session data for 24 hours after last activity
- Anonymous sessions can be "upgraded" to identified accounts (data migrates)

**Rate Limits (Anonymous):**

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

### 4.3 Identified Users (Hanko)

**Decision:** Use **Hanko** (open-source, self-hostable) for passkey-first authentication with optional Google social login.

**Why Hanko?**

- Open source, self-hostable (fits our KVM 4)
- Passkey-first with fallback to email magic links
- **Built-in social login support** including Google OAuth
- Beautiful, customizable drop-in UI components
- No vendor lock-in
- Active development, strong community

**Why Passkeys as Default?**

- Zero password friction (no "forgot password" flows)
- Phishing-resistant by design
- Modern UX that feels premium
- Offloads security to user's device/biometrics

**Authentication Options Flow:**

```
+-----------------------------------------------------------------------+
|                                                                       |
|   User arrives at sign-up/login                                       |
|                         |                                             |
|                         v                                             |
|   +---------------------------------------------------------------+   |
|   |                   HANKO AUTH UI                               |   |
|   |                                                               |   |
|   |   +-------------------------+  +---------------------------+  |   |
|   |   |                         |  |                           |  |   |
|   |   |   [*] Create Passkey   |  |   [G] Continue with        |  |   |
|   |   |       (Recommended)     |  |       Google              |  |   |
|   |   |                         |  |                           |  |   |
|   |   +-------------------------+  +---------------------------+  |   |
|   |                                                               |   |
|   |   +-------------------------------------------------------+   |   |
|   |   |  [...] Email magic link (fallback)                    |   |   |
|   |   +-------------------------------------------------------+   |   |
|   |                                                               |   |
|   +---------------------------------------------------------------+   |
|                         |                                             |
|          +--------------+--------------+                              |
|          |              |              |                              |
|          v              v              v                              |
|      Passkey        Google         Email                              |
|      Created        OAuth          Link Sent                          |
|          |              |              |                              |
|          +--------------+--------------+                              |
|                         |                                             |
|                         v                                             |
|                  User Authenticated                                   |
|                  (Hanko session created)                              |
|                         |                                             |
|                         v                                             |
|        +--------------------------------+                             |
|        | Wants Gemini AI features?      |                             |
|        +--------------------------------+                             |
|                    |           |                                      |
|                    v           v                                      |
|                   No          Yes                                     |
|                    |           |                                      |
|                    |           v                                      |
|                    |   +-----------------------------+                |
|                    |   | Additional Google OAuth     |                |
|                    |   | (Gemini + YouTube scopes)   |                |
|                    |   +-----------------------------+                |
|                    |           |                                      |
|                    v           v                                      |
|               Done         AI Features Unlocked                       |
|                                                                       |
+-----------------------------------------------------------------------+
```

**Key Insight:** Hanko's Google social login is for **authentication** (proving identity). The separate Google OAuth for Gemini is for **authorization** (accessing Gemini API). These serve different purposes:

1. User signs up via Hanko with Google -> We know who they are
2. User later clicks "Enable AI features" -> We get Gemini API access token

If user originally signed up with Google via Hanko, the second OAuth flow is even smoother (Google remembers the consent).

**Hanko Deployment (on KVM 4):**

```yaml
# docker-compose.yml
services:
  hanko:
    image: ghcr.io/teamhanko/hanko:latest
    environment:
      - DATABASE_URL=postgres://hanko:password@postgres:5432/hanko
      - SECRETS_KEYS=${HANKO_SECRET_KEY}
      - WEBAUTHN_RELYING_PARTY_ID=yourdomain.com
      - WEBAUTHN_RELYING_PARTY_ORIGIN=https://yourdomain.com
      - PASSCODE_EMAIL_FROM_ADDRESS=auth@yourdomain.com
      - PASSCODE_SMTP_HOST=smtp.yourprovider.com
      - PASSCODE_SMTP_PORT=587
      - PASSCODE_SMTP_USER=${SMTP_USER}
      - PASSCODE_SMTP_PASSWORD=${SMTP_PASSWORD}
      # Google social login
      - THIRD_PARTY_PROVIDERS_GOOGLE_ENABLED=true
      - THIRD_PARTY_PROVIDERS_GOOGLE_CLIENT_ID=${GOOGLE_CLIENT_ID}
      - THIRD_PARTY_PROVIDERS_GOOGLE_CLIENT_SECRET=${GOOGLE_CLIENT_SECRET}
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
const hankoApi = "https://yourdomain.com/hanko";
const hanko = new Hanko(hankoApi);

// Auth component (drop-in)
export function AuthPage() {
  return (
    <div className="auth-container">
      <hanko-auth api={hankoApi} />
    </div>
  );
}

// Profile component (manage passkeys, connected accounts)
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

**Session Upgrade (Anonymous -> Identified):**

```python
async def upgrade_session(anonymous_id: str, user_id: str):
    """Migrate anonymous session data to identified user."""
    anon_data = await get_anonymous_session(anonymous_id)
    if anon_data:
        # Migrate moments, history, preferences
        await merge_into_user_account(user_id, anon_data)
        await delete_anonymous_session(anonymous_id)
        
        # Log upgrade for analytics
        await log_event("session_upgraded", {
            "anonymous_id": anonymous_id,
            "user_id": user_id,
            "moments_migrated": len(anon_data.selected_moments)
        })
```

### 4.4 Google OAuth (For Gemini + YouTube Integration)

**Decision:** Offer optional Google OAuth for users who want Cloud LLM features. This is separate from Hanko's Google social login.

**Why Separate OAuth Flows?**

```
Hanko Google Login:
  Scopes: openid, email, profile
  Purpose: Prove identity
  We get: User info for account creation
  
Gemini Google OAuth:
  Scopes: generative-language, youtube.readonly (optional)
  Purpose: API access authorization
  We get: Access token for Gemini API calls
```

**Scopes Requested (Gemini OAuth):**

```
https://www.googleapis.com/auth/generative-language.retriever
https://www.googleapis.com/auth/youtube.readonly  (optional)
```

**OAuth Consent Screen Copy:**

```
+---------------------------------------------------------------+
|                                                               |
|   [App Logo]  YourAppName wants to access your Google Account |
|                                                               |
|   This will allow YourAppName to:                             |
|                                                               |
|   [check] Use Google AI services (Gemini)                     |
|           For smart visual moment detection                   |
|                                                               |
|   [check] View your YouTube account (optional)                |
|           To access unlisted videos you own                   |
|                                                               |
|   YourAppName will not be able to:                            |
|   - See your Google password                                  |
|   - Access your Gmail or Drive                                |
|   - Make purchases or changes to your account                 |
|                                                               |
|            [Cancel]        [Allow]                            |
|                                                               |
+---------------------------------------------------------------+
```

**UI for Enabling AI Features:**

```
+---------------------------------------------------------------+
|                                                               |
|  .:. Enhance with AI                                          |
|                                                               |
|  Get smarter moment suggestions by connecting your Google     |
|  account. We'll use Gemini to analyze transcripts.            |
|                                                               |
|  +-----------------------------------------------------------+|
|  |                                                           ||
|  |  This grants access to:                                   ||
|  |  [x] Gemini AI (for smart suggestions)                    ||
|  |  [ ] Your YouTube library (for unlisted videos) [?]       ||
|  |                                                           ||
|  |  We never store your Google password.                     ||
|  |  Disconnect anytime in settings.                          ||
|  |                                                           ||
|  |  [    Connect Google Account    ]                         ||
|  |                                                           ||
|  +-----------------------------------------------------------+|
|                                                               |
|  --------------------------or---------------------------------|
|                                                               |
|  [Continue without AI] --> Uses rule-based detection only     |
|                                                               |
+---------------------------------------------------------------+
```

**Token Storage:**

- Access tokens stored server-side, encrypted at rest
- Refresh tokens stored securely, auto-refresh on expiry
- Tokens scoped to user, never shared
- User can revoke via settings (we also call Google's revoke endpoint)

### 4.5 Admin Access via Tailscale

**Decision:** Admin functionality is only accessible from the Tailscale network. No code-level auth checks needed -- network topology *is* the auth.

**Why Tailscale?**

- Zero-trust networking without VPN complexity
- Identity tied to your Tailscale account
- Can expose specific routes only to your tailnet
- Already have it set up (your requirement!)
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
yourdomain.com {
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
# This listens on the Tailscale IP, not the public IP
http://<your-tailscale-hostname>:8443 {
    # Admin routes - full access
    handle /admin/* {
        reverse_proxy localhost:8000
    }
    
    # Also proxy regular routes for convenience when on tailnet
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
| LLM Usage Dashboard    | Gemini API calls, token counts           |
+------------------------+------------------------------------------+
```

**Admin UI Location:** `http://<tailscale-hostname>:8443/admin` (only reachable via Tailscale)

---

## 5. Mobile-First Accessibility Design

### 5.1 Design Philosophy

**"Fat fingers are not a bug, they're the primary use case."**

Every interactive element must be designed assuming the user:

- Has large fingers relative to screen size
- Is using one hand (thumb-only navigation)
- Is in motion (on transit, walking)
- Might accidentally tap adjacent elements

### 5.2 Touch Target Standards

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
// Touch-friendly button wrapper
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
    // Haptic feedback on supported devices
    onTapStart={() => navigator.vibrate?.(10)}
  >
    {children}
  </motion.button>
);
```

### 5.3 Gesture-Based Interactions

**Replace Small Buttons with Gestures:**

```
+---------------------------+---------------------------+--------------------+
| Instead of...             | Use...                    | Benefit            |
+---------------------------+---------------------------+--------------------+
| Small "x" close button    | Swipe down to dismiss     | Whole screen=target|
| Tiny edit/delete icons    | Swipe left on item        | Natural, intuitive |
| Small +/- buttons         | Pinch to zoom timeline    | Intuitive, precise |
| Checkbox for select       | Long-press to select      | Harder to mis-tap  |
| Settings gear icon        | Pull down past top        | Hidden but there   |
+---------------------------+---------------------------+--------------------+
```

**Gesture Implementation (Moments Universe):**

```tsx
// Timeline segment with gesture controls
<motion.div
  drag="x"
  dragConstraints={{ left: 0, right: 0 }}
  dragElastic={0.2}
  onDragEnd={(e, info) => {
    if (info.offset.x < -100) {
      // Swiped left: delete moment
      onDeleteMoment();
    } else if (info.offset.x > 100) {
      // Swiped right: confirm moment
      onConfirmMoment();
    }
  }}
  // Visual feedback during drag
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

### 5.4 Mobile Layout Patterns

**Bottom Sheet for Actions (Not Modals):**

```
Desktop:                          Mobile:
+---------------------+          +---------------------+
|                     |          |                     |
|  +---------------+  |          |                     |
|  |    Modal      |  |          |                     |
|  |   (centered)  |  |    vs    |                     |
|  |               |  |          |                     |
|  +---------------+  |          +---------------------+
|                     |          | =================== |<-- Drag handle
+---------------------+          |                     |
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

### 5.5 Forgiving Interactions

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
// Every destructive action is reversible
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

### 5.6 Mobile-Specific Moments Universe

**Portrait Mode Layout:**

```
+-------------------------------------+
|  <  Video Title Here...         [=] |  <-- Compact header
+-------------------------------------+
|                                     |
|  +-------------------------------+  |
|  |                               |  |
|  |      Video Preview            |  |  <-- 16:9 aspect
|  |      (Segment Player)         |  |
|  |                               |  |
|  +-------------------------------+  |
|                                     |
|  ....####......######....#......... |  <-- Timeline (expandable)
|                                     |
+-------------------------------------+
|                                     |
|  Transcript                    [Q]  |
|  -----------------------------------+
|                                     |
|  | 0:00  Welcome to today's...      |
|  |                                  |  <-- Scrollable
|  | 0:15  As you can see here... *   |  <-- * = moment marker
|  |                                  |
|  | 0:32  The diagram shows...  *    |
|  |                                  |
|                                     |
|                            ( + )    |  <-- FAB for adding moments
|                                     |
+-------------------------------------+
|                                     |
|  =================================  |  <-- Bottom sheet handle
|                                     |
|  Moments (3)             [Export]   |
|                                     |
|  +------+ +------+ +------+         |  <-- Horizontal scroll
|  | [##] | | [##] | | [##] |         |
|  | 0:15 | | 0:32 | | 1:45 |         |
|  +------+ +------+ +------+         |
|                                     |
+-------------------------------------+
```

**Landscape Mode (Immersive Player):**

```
+------------------------------------------------------------------------+
|                                                                        |
|                                                                        |
|                    Video Preview (Full Width)                          |
|                                                                        |
|                                                                        |
+------------------------------------------------------------------------+
| ....####......######....########....#..........###########.........*   |
| ^                                                                  ^   |
| Tap anywhere to seek                           Tap * to jump to moment |
|                                                                        |
| [Exit]                                        [Mark Moment] (large)    |
+------------------------------------------------------------------------+
```

---

## 6. The Moments Universe (Core Feature)

### 6.1 Concept

The **Moments Universe** is not just a feature -- it's the soul of the application. It's where users discover, select, and curate the visual moments that transform a transcript into a complete, AI-comprehensible document.

**Design Principles:**

1. **Discoverable:** Moments should feel like they're floating in a space, waiting to be found
2. **Tactile:** Every interaction should feel physical and satisfying
3. **Forgiving:** Wrong selections are trivially reversible
4. **Beautiful:** This is where we go all-in on cool factor

### 6.2 Visual Design: The Timeline Constellation

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

### 6.3 Moment States & Animations

**State Machine:**

```
                    +-----------+
                    |   EMPTY   |
                    | (no moment|
                    +-----------+
                          |
           +--------------+--------------+
           |              |              |
           v              v              v
   +---------------+ +---------+ +---------------+
   | AUTO-DETECTED | |  USER   | |  RULE-BASED   |
   |   (by LLM)    | | SELECTED| |   (pattern)   |
   +-------+-------+ +----+----+ +-------+-------+
           |              |              |
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

### 6.4 Moment Picker Mode

When user activates moment picking:

```tsx
// Full-screen takeover for moment selection
<AnimatePresence>
  {pickingMode && (
    <motion.div
      className="fixed inset-0 z-50"
      initial={{ opacity: 0 }}
      animate={{ opacity: 1 }}
      exit={{ opacity: 0 }}
    >
      {/* Darkened overlay */}
      <div className="absolute inset-0 bg-black/80" />
      
      {/* Spotlight follows cursor/touch */}
      <SpotlightCursor 
        size={200}
        color="rgba(147, 51, 234, 0.3)"  // Purple glow
      />
      
      {/* Expanded video */}
      <VideoPreview expanded />
      
      {/* Expanded timeline */}
      <motion.div
        initial={{ height: 60 }}
        animate={{ height: 160 }}
        className="fixed bottom-0 left-0 right-0"
      >
        <ExpandedTimeline 
          onMark={handleMarkMoment}
          precision="frame"  // Frame-accurate in this mode
        />
      </motion.div>
      
      {/* Instructions */}
      <div className="fixed top-4 left-1/2 -translate-x-1/2">
        <motion.p
          initial={{ y: -20, opacity: 0 }}
          animate={{ y: 0, opacity: 1 }}
          className="text-white text-lg"
        >
          Tap timeline or press SPACE to mark this moment
        </motion.p>
      </div>
      
      {/* Exit button (large, accessible) */}
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

### 6.5 Frame Preview Gallery

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

## 7. Moment Detection Engine (3-Tier System)

### 7.1 Architecture Overview

**Decision:** Three distinct detection modes, progressively more intelligent, with clear user control.

```
+-----------------------------------------------------------------------+
|                     MOMENT DETECTION ENGINE                           |
|                                                                       |
|   Mode Selection (User Controlled)                                    |
|                                                                       |
|   +--------------+   +--------------+   +--------------+              |
|   |    [=]       |   |   (cloud)    |   |    <cpu>     |              |
|   |   RULES      |   |    CLOUD     |   |    LOCAL     |              |
|   |   ONLY       |   |   (Gemini)   |   |   (Ollama)   |              |
|   |              |   |              |   |              |              |
|   |  Default     |   |  Requires    |   |  Admin       |              |
|   |  Always on   |   |  Google      |   |  granted     |              |
|   |              |   |  OAuth       |   |  only        |              |
|   +--------------+   +--------------+   +--------------+              |
|          |                 |                 |                        |
|          v                 v                 v                        |
|   +---------------------------------------------------------------+   |
|   |           Combined Results --> Deduplicated                   |   |
|   +---------------------------------------------------------------+   |
|                                                                       |
+-----------------------------------------------------------------------+
```

### 7.2 Tier 1: Rule-Based Detection (Always Active)

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
    "deictic_explicit": 0.95,    # Very high confidence
    "deictic_implicit": 0.75,    # Good confidence
    "code_demo": 0.85,           # High for tech content
    "ui_navigation": 0.80,       # High for tutorials
    "transitions": 0.70,         # Medium - might be false positive
    "physical_demo": 0.90,       # High confidence
}
```

**Contextual Boosting:**

```python
def calculate_moment_confidence(
    segment: TranscriptSegment,
    matches: List[PatternMatch],
    context: AnalysisContext
) -> float:
    """
    Adjust confidence based on context.
    """
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
    if context.pattern_frequency[matches[0].pattern] > 0.1:  # >10% of segments
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

### 7.3 Tier 2: Cloud LLM (Gemini via User's Google Account)

**Decision:** Use **Gemini 1.5 Flash** via user's own Google OAuth. No API keys, no BYOK, no cost to us.

**Why Gemini 1.5 Flash?**

- Available via Google AI Studio free tier (60 requests/minute!)
- Accessible through user's existing Google account
- Fast inference (hence "Flash")
- Good at structured extraction tasks
- Generous context window (1M tokens) for long transcripts

**Quota & Limits (Free Tier):**

```
+--------------------+------------+------------------------+
| Metric             | Limit      | Our Usage Pattern      |
+--------------------+------------+------------------------+
| Requests/minute    | 60         | ~1-3 per video         |
| Requests/day       | 1,500      | More than enough       |
| Tokens/minute      | 1,000,000  | Transcripts are small  |
+--------------------+------------+------------------------+
```

**Implementation Flow:**

```
+-----------------------------------------------------------------------+
|                                                                       |
|   User clicks "Enhance with AI"                                       |
|                           |                                           |
|                           v                                           |
|   +---------------------------------------------------------------+   |
|   |  Has valid Google OAuth token with Gemini scope?              |   |
|   +---------------------------------------------------------------+   |
|                           |                                           |
|              +------------+------------+                              |
|              |                         |                              |
|              v No                      v Yes                          |
|   +---------------------+   +-------------------------------------+   |
|   | Trigger OAuth flow  |   | Call Gemini API with user's token   |   |
|   | (via our app)       |   |                                     |   |
|   +---------------------+   +-------------------------------------+   |
|                                         |                             |
|                                         v                             |
|                           +-----------------------------------+       |
|                           | Parse response --> Visual moments |       |
|                           +-----------------------------------+       |
|                                                                       |
+-----------------------------------------------------------------------+
```

**API Call (via Google AI SDK):**

```python
import google.generativeai as genai
from google.oauth2.credentials import Credentials

async def analyze_with_gemini(
    transcript: Transcript,
    user_access_token: str,
    user_refresh_token: str
) -> List[LLMMoment]:
    """
    Call Gemini using user's OAuth token.
    """
    # Build credentials from user's tokens
    credentials = Credentials(
        token=user_access_token,
        refresh_token=user_refresh_token,
        token_uri="https://oauth2.googleapis.com/token",
        client_id=settings.GOOGLE_CLIENT_ID,
        client_secret=settings.GOOGLE_CLIENT_SECRET,
    )
    
    # Configure genai with user credentials
    genai.configure(credentials=credentials)
    
    model = genai.GenerativeModel('gemini-1.5-flash')
    
    # Chunked analysis for long transcripts
    chunks = chunk_transcript(transcript, max_tokens=8000, overlap=500)
    all_moments = []
    
    for chunk in chunks:
        response = await model.generate_content_async(
            GEMINI_PROMPT.format(transcript=chunk.text),
            generation_config={
                "response_mime_type": "application/json",
                "temperature": 0.2,  # Low for consistency
            }
        )
        
        moments = parse_gemini_response(response.text)
        all_moments.extend(moments)
    
    return deduplicate_moments(all_moments, threshold_seconds=5)
```

**Gemini Prompt:**

```python
GEMINI_PROMPT = """You are analyzing a video transcript to identify \
moments where visual context is essential for understanding.

A "visual moment" is when:
1. The speaker explicitly references something visible \
("as you can see", "look at this")
2. On-screen content (code, diagrams, text) is being discussed
3. A physical demonstration is happening
4. Visual comparison or transition occurs
5. Context would be significantly lost without seeing the video

Analyze this transcript segment and identify visual moments.

IMPORTANT:
- Be selective. Not every timestamp is a visual moment.
- Prefer moments where the visual adds essential context, not just \
"nice to have"
- If the speaker says "um, you know, like, here" that's probably \
not a strong visual moment
- Technical tutorials have more visual moments than talking-head \
discussions

Return ONLY valid JSON in this exact format:
{{
  "moments": [
    {{
      "timestamp_seconds": <number>,
      "confidence": <0.0-1.0>,
      "reason": "<brief explanation>",
      "visual_type": "<diagram|code|demonstration|ui|comparison|other>"
    }}
  ]
}}

If no visual moments exist, return: {{"moments": []}}

TRANSCRIPT:
{transcript}
"""
```

**Usage Tracking (For User Transparency):**

```python
# Store in user's session/account
@dataclass
class GeminiUsage:
    user_id: str
    date: str  # YYYY-MM-DD
    requests_count: int
    tokens_used: int
    
async def track_gemini_usage(user_id: str, tokens: int):
    today = datetime.now().strftime("%Y-%m-%d")
    usage = await get_or_create_usage(user_id, today)
    usage.requests_count += 1
    usage.tokens_used += tokens
    await save_usage(usage)
```

**UI for Usage Display:**

```
+---------------------------------------------------------------+
|  AI Enhancement                                   Connected + |
|  -------------------------------------------------------------+
|                                                               |
|  Using: Gemini 1.5 Flash (via your Google account)            |
|                                                               |
|  Today's usage: 12 / 1,500 requests                           |
|  ####................................... 0.8%                 |
|                                                               |
|  [Disconnect Google Account]                                  |
|                                                               |
+---------------------------------------------------------------+
```

### 7.4 Tier 3: Local LLM (Admin-Granted Feature)

**Decision:** Self-hosted Ollama with small model, enabled per-user via admin feature flag.

**Why Feature-Flagged?**

- Local LLM consumes server resources (RAM, CPU)
- Not everyone needs it (Cloud tier is usually sufficient)
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

**Feature Flag System:**

```python
# Database schema
class UserFeatureFlags(BaseModel):
    user_id: str
    local_llm_enabled: bool = False
    local_llm_granted_at: Optional[datetime] = None
    local_llm_granted_by: str = "admin"  # Always admin for now
    
# Admin endpoint (Tailscale-only)
@router.post("/admin/users/{user_id}/features/local-llm")
async def toggle_local_llm(
    user_id: str,
    enabled: bool,
    request: Request
):
    # No auth check needed - Tailscale network IS the auth
    await update_feature_flag(user_id, "local_llm_enabled", enabled)
    await log_admin_action(
        admin_ip=request.client.host,  # Tailscale IP for audit
        action=f"Set local_llm={enabled} for {user_id}"
    )
    return {"status": "updated"}
```

**UI Indication:**

```
+---------------------------------------------------------------+
|  Moment Detection Mode                                        |
|  -------------------------------------------------------------+
|                                                               |
|  ( ) Rules Only (fast, no AI)                                 |
|                                                               |
|  ( ) Rules + Cloud AI (requires Google account)               |
|                                                               |
|  (*) Rules + Local AI **                                      |
|      '-> Enabled by admin. Runs on our server.                |
|      '-> May be slower during high usage.                     |
|                                                               |
+---------------------------------------------------------------+
```

### 7.5 Mode Switching UI

**Decision:** Simple toggle group in settings panel, with smart defaults.

```tsx
// MomentDetectionSettings component
const MomentDetectionSettings = () => {
  const { user, features } = useAuth();
  const [mode, setMode] = useDetectionMode();
  
  return (
    <div className="space-y-4">
      <h3 className="font-medium">Moment Detection</h3>
      
      <RadioGroup value={mode} onValueChange={setMode}>
        {/* Always available */}
        <RadioItem value="rules">
          <div className="flex items-center gap-2">
            <RulerIcon className="w-4 h-4" />
            <span>Rules Only</span>
            <Badge variant="secondary">Fast</Badge>
          </div>
          <p className="text-sm text-muted-foreground">
            Pattern matching, no AI. Works offline.
          </p>
        </RadioItem>
        
        {/* Requires Google OAuth */}
        <RadioItem 
          value="cloud" 
          disabled={!user?.googleConnected}
        >
          <div className="flex items-center gap-2">
            <CloudIcon className="w-4 h-4" />
            <span>Rules + Cloud AI</span>
            <Badge variant="default">Recommended</Badge>
          </div>
          <p className="text-sm text-muted-foreground">
            Uses Gemini via your Google account.
          </p>
          {!user?.googleConnected && (
            <Button 
              size="sm" 
              variant="outline" 
              onClick={connectGoogle}
            >
              Connect Google
            </Button>
          )}
        </RadioItem>
        
        {/* Requires admin feature flag */}
        <RadioItem 
          value="local"
          disabled={!features?.localLlmEnabled}
        >
          <div className="flex items-center gap-2">
            <CpuIcon className="w-4 h-4" />
            <span>Rules + Local AI</span>
            {features?.localLlmEnabled && (
              <Badge variant="outline">** Enabled</Badge>
            )}
          </div>
          <p className="text-sm text-muted-foreground">
            Runs on our server. Admin-granted access.
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

### 7.6 Result Merging & Deduplication

When multiple tiers are active, results must be intelligently merged:

```python
async def detect_moments(
    transcript: Transcript,
    mode: DetectionMode,
    user: Optional[User]
) -> List[DetectedMoment]:
    """
    Run detection based on mode, merge results.
    """
    all_moments: List[DetectedMoment] = []
    
    # Tier 1: Rules (always)
    rule_moments = await detect_with_rules(transcript)
    all_moments.extend(rule_moments)
    
    # Tier 2 or 3: LLM (if enabled and authorized)
    if mode == DetectionMode.CLOUD and user and user.google_token:
        llm_moments = await detect_with_gemini(
            transcript, 
            user.google_token,
            user.google_refresh_token
        )
        all_moments.extend(llm_moments)
    elif mode == DetectionMode.LOCAL and user and user.features.local_llm_enabled:
        llm_moments = await detect_with_ollama(transcript)
        all_moments.extend(llm_moments)
    
    # Deduplicate: moments within 5 seconds are merged
    merged = merge_nearby_moments(all_moments, threshold_seconds=5)
    
    # Sort by timestamp
    merged.sort(key=lambda m: m.timestamp)
    
    # Combine confidences for duplicates
    for moment in merged:
        if moment.sources_count > 1:
            # Boost confidence when multiple sources agree
            moment.confidence = min(moment.confidence + 0.15, 1.0)
    
    return merged
```

---

## 8. Video Segment Strategy

### 8.1 Segment-Based Downloads

**Decision:** Download video in segments on-demand, not complete files.

**Context:**  
Full video downloads are wasteful -- a 1-hour video at 360p is ~200-400MB. Users only need to preview specific portions to pick moments. YouTube's underlying format (DASH/HLS) is already segmented.

**Implementation:**

```bash
yt-dlp --download-sections "*00:05:00-00:05:30" \
       --format "worst[height>=360]" \
       -o "segments/%(id)s_%(section_start)s.%(ext)s" \
       "https://youtube.com/watch?v=..."
```

**Segment Strategy:**

- Initial transcript fetch: metadata only, no video download
- User scrolls to timestamp: trigger 30-second segment download centered on that point
- Segments are cached and painted on the player timeline
- Adjacent segment prefetching when user hovers near segment boundaries

**Quality Selection:**

```
+------------+-----------------+------------------+-------------------+
| Resolution | Bitrate (approx)| 30s Segment Size | Use Case          |
+------------+-----------------+------------------+-------------------+
| 360p       | 500 kbps        | ~2 MB            | ** DEFAULT **     |
| 480p       | 1000 kbps       | ~4 MB            | User requests HD  |
| 240p       | 300 kbps        | ~1 MB            | Bandwidth fallback|
+------------+-----------------+------------------+-------------------+
```

**Why 360p default:** Sufficient to distinguish visual content (is there a diagram? a person? text on screen?) without excessive storage. Most "is this a visual moment?" decisions don't require HD.

---

## 9. Cache Architecture & Session Locking

### 9.1 Directory Structure

```
/var/cache/yttool/
|-- segments/                    # Video segment files
|   |-- {video_id}_{start}_{end}.mp4
|   '-- ...
|-- frames/                      # Extracted frame images
|   '-- {video_id}_{timestamp}.jpg
|-- metadata/                    # Transcript + video info
|   '-- {video_id}.json
'-- locks/                       # Session lock files
    '-- {session_id}.lock        # Contains locked segment IDs
```

### 9.2 Two-Tier Cache with Locks

**Problem:**  
With a 10GB cache ceiling, a rotation policy might evict segments that a user is actively previewing. This would cause playback failures and a broken experience.

**Solution:** LRU cache with session-aware locking to prevent active segment eviction.

```python
# Cache manager implementation

from dataclasses import dataclass
from pathlib import Path
from typing import Dict, Set, Optional
import asyncio
import logging

logger = logging.getLogger(__name__)

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
    def __init__(self, cache_dir: Path, max_size_gb: float = 10.0):
        self.cache_dir = cache_dir
        self.max_bytes = int(max_size_gb * 1024**3)
        self.segments_dir = cache_dir / "segments"
        self.frames_dir = cache_dir / "frames"
        self.metadata_dir = cache_dir / "metadata"
        
        # Session locks: session_id -> set of segment_ids
        self.locks: Dict[str, Set[str]] = {}
        
        # Ensure directories exist
        for d in [self.segments_dir, self.frames_dir, self.metadata_dir]:
            d.mkdir(parents=True, exist_ok=True)
    
    async def acquire_segment(
        self, 
        session_id: str, 
        video_id: str,
        start_time: float,
        end_time: float
    ) -> Path:
        """
        Get segment path, downloading if needed.
        Locks segment against eviction for this session.
        """
        segment_id = f"{video_id}_{int(start_time)}_{int(end_time)}"
        segment_path = self.segments_dir / f"{segment_id}.mp4"
        
        # Lock this segment to the session
        if session_id not in self.locks:
            self.locks[session_id] = set()
        self.locks[session_id].add(segment_id)
        
        # Download if not cached
        if not segment_path.exists():
            await self._download_segment(video_id, start_time, end_time, segment_path)
        
        # Update access time for LRU
        await self._touch_segment(segment_id)
        
        # Trigger eviction check (non-blocking)
        asyncio.create_task(self._maybe_evict())
        
        return segment_path
    
    def release_session(self, session_id: str):
        """
        Called on session end/timeout. Releases all locks.
        """
        if session_id in self.locks:
            released_count = len(self.locks[session_id])
            del self.locks[session_id]
            logger.info(f"Released {released_count} segment locks for session {session_id}")
            
            # Trigger eviction since we freed locks
            asyncio.create_task(self._maybe_evict())
    
    async def _maybe_evict(self):
        """
        Evict oldest unlocked segments until under max_size.
        """
        current_size = await self._calculate_current_size()
        
        if current_size <= self.max_bytes:
            return
        
        # Gather all locked segments across all sessions
        locked_segments: Set[str] = set()
        for session_locks in self.locks.values():
            locked_segments.update(session_locks)
        
        # Get segments sorted by last access (oldest first)
        segments = await self._get_segments_sorted_by_access()
        
        bytes_to_free = current_size - self.max_bytes
        bytes_freed = 0
        
        for segment in segments:
            if bytes_freed >= bytes_to_free:
                break
                
            if segment.id in locked_segments:
                # Skip locked segments
                continue
            
            # Delete this segment
            try:
                segment.path.unlink()
                bytes_freed += segment.size_bytes
                logger.info(f"Evicted segment {segment.id} ({segment.size_bytes} bytes)")
            except Exception as e:
                logger.error(f"Failed to evict segment {segment.id}: {e}")
        
        if bytes_freed < bytes_to_free:
            logger.warning(
                f"Cache full but could only free {bytes_freed} of "
                f"{bytes_to_free} bytes needed. "
                f"{len(locked_segments)} segments are locked."
            )
    
    async def _download_segment(
        self, 
        video_id: str, 
        start: float, 
        end: float, 
        output_path: Path
    ):
        """Download segment using yt-dlp."""
        import subprocess
        
        url = f"https://youtube.com/watch?v={video_id}"
        section = f"*{self._format_time(start)}-{self._format_time(end)}"
        
        cmd = [
            "yt-dlp",
            "--download-sections", section,
            "--format", "worst[height>=360]",
            "--output", str(output_path),
            "--quiet",
            url
        ]
        
        proc = await asyncio.create_subprocess_exec(
            *cmd,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE
        )
        stdout, stderr = await proc.communicate()
        
        if proc.returncode != 0:
            raise Exception(f"yt-dlp failed: {stderr.decode()}")
    
    @staticmethod
    def _format_time(seconds: float) -> str:
        """Format seconds as HH:MM:SS for yt-dlp."""
        h = int(seconds // 3600)
        m = int((seconds % 3600) // 60)
        s = int(seconds % 60)
        return f"{h:02d}:{m:02d}:{s:02d}"
    
    async def _calculate_current_size(self) -> int:
        """Sum size of all cached segments."""
        total = 0
        for path in self.segments_dir.glob("*.mp4"):
            total += path.stat().st_size
        return total
    
    async def _get_segments_sorted_by_access(self) -> list:
        """Get all segments sorted by last access time (oldest first)."""
        # Implementation would read from DB or file metadata
        # Simplified here
        pass
    
    async def _touch_segment(self, segment_id: str):
        """Update last access time for segment."""
        # Implementation would update DB or file metadata
        pass
```

**Step-by-step UX for seeking to uncached regions:**

1. User seeks to uncached region
2. Timeline shows "loading" animation for that segment
3. Video displays "Loading preview..." placeholder
4. Segment downloads (2-4 seconds for 30s @ 360p)
5. Video begins playing from requested timestamp
6. Adjacent segments prefetch in background

**Notes on gapless playback (nice-to-have):**
- Preload next segment's first few seconds into buffer
- Use MediaSource API for seamless switching
- Fall back to brief loading indicator if segment not ready

### 9.3 Session Lifecycle

```
1. Session Start
   '-> Client connects, gets session ID (UUID)
   '-> Session stored server-side with creation time

2. Segment Request  
   '-> Each viewed segment is locked to that session
   '-> Lock prevents eviction during active viewing

3. Heartbeat
   '-> Client sends ping every 30 seconds
   '-> Updates session's last_active timestamp

4. Session End
   '-> Explicit close (user navigates away)
   '-> OR 2 minutes without heartbeat --> timeout
   '-> Releases all segment locks

5. Graceful Degradation
   '-> If segment somehow evicted mid-session (edge case)
   '-> Client receives "segment unavailable" response
   '-> Client can re-request (will re-download)
   '-> User sees brief "Reloading preview..." message
```

### 9.4 Failure Scenarios & UX

```
+----------------------------------+-------------------------+--------------------+
| Scenario                         | System Behavior         | User Experience    |
+----------------------------------+-------------------------+--------------------+
| Segment evicted while locked     | Should never happen     | N/A                |
|                                  | (lock prevents this)    |                    |
+----------------------------------+-------------------------+--------------------+
| User leaves tab open for hours   | Session times out,      | Segments may need  |
|                                  | locks released          | re-download; shows |
|                                  |                         | "Refreshing..."    |
+----------------------------------+-------------------------+--------------------+
| Cache completely full, all       | Log warning, refuse     | "Server busy,      |
| segments locked                  | new sessions            | try again shortly" |
+----------------------------------+-------------------------+--------------------+
| Segment download fails (network) | Retry 2x with backoff   | "Couldn't load,    |
|                                  |                         | retrying..."       |
+----------------------------------+-------------------------+--------------------+
| Session reconnects after timeout | New session ID,         | Transparent if     |
|                                  | segments re-locked if   | cached; brief      |
|                                  | still in cache          | reload if not      |
+----------------------------------+-------------------------+--------------------+
```

---

## 10. Process Management & Concurrency Control

### 10.1 Task Queue: Huey with SQLite

**Decision:** Use `huey` with SQLite backend for task queue, plus asyncio semaphores for fine-grained resource limits.

**Why Not Celery?**  
Celery is overkill and requires Redis/RabbitMQ. Huey uses SQLite -- zero additional services.

**Why Not Just asyncio?**  
Need persistence. If the server restarts mid-task, we want to resume, not lose work.

**Setup:**

```python
from huey import SqliteHuey

# Task queue configuration
huey = SqliteHuey(
    filename='/var/lib/yttool/tasks.db',
    immediate=False,  # Use queue, don't run synchronously
)

# Example background task
@huey.task()
def extract_frames_task(video_id: str, timestamps: list[float]):
    """Extract frames at given timestamps. Runs in background."""
    for ts in timestamps:
        extract_single_frame(video_id, ts)
    return {"status": "complete", "count": len(timestamps)}

# Periodic cleanup task
@huey.periodic_task(crontab(minute='0', hour='3'))  # 3 AM daily
def cleanup_old_sessions():
    """Remove expired sessions and unlock their segments."""
    expired = get_expired_sessions(max_age_hours=24)
    for session in expired:
        cache_manager.release_session(session.id)
        delete_session(session.id)
```

### 10.2 Resource Limiters

```python
from contextlib import asynccontextmanager
import asyncio
import threading

class ResourceLimiter:
    """
    Manages concurrency limits for resource-intensive operations.
    Uses both asyncio semaphores (for async code) and threading 
    semaphores (for huey workers).
    """
    
    def __init__(self):
        # Async semaphores (for FastAPI handlers)
        self.async_llm = asyncio.Semaphore(1)
        self.async_ffmpeg = asyncio.Semaphore(2)
        self.async_download = asyncio.Semaphore(3)
        
        # Threading semaphores (for Huey workers)
        self.thread_llm = threading.Semaphore(1)
        self.thread_ffmpeg = threading.Semaphore(2)
        self.thread_download = threading.Semaphore(3)
    
    @asynccontextmanager
    async def llm_slot(self):
        """Acquire LLM processing slot (async context)."""
        async with self.async_llm:
            yield
    
    @asynccontextmanager
    async def ffmpeg_slot(self):
        """Acquire ffmpeg processing slot (async context)."""
        async with self.async_ffmpeg:
            yield
    
    @asynccontextmanager
    async def download_slot(self):
        """Acquire download slot (async context)."""
        async with self.async_download:
            yield
    
    # Sync versions for Huey tasks
    def acquire_llm_sync(self):
        return self.thread_llm
    
    def acquire_ffmpeg_sync(self):
        return self.thread_ffmpeg

# Global instance
limiter = ResourceLimiter()

# Usage in async handler
async def analyze_transcript_handler(video_id: str):
    async with limiter.llm_slot():
        result = await run_llm_analysis(video_id)
    return result

# Usage in Huey task
@huey.task()
def analyze_transcript_task(video_id: str):
    with limiter.acquire_llm_sync():
        result = run_llm_analysis_sync(video_id)
    return result
```

### 10.3 Concurrency Limits (Tuned for KVM 4)

```
+---------------------+----------------+----------------------------------+
| Resource            | Max Concurrent | Rationale                        |
+---------------------+----------------+----------------------------------+
| LLM Inference       | 1              | Memory-bound; 2GB ceiling        |
| FFmpeg (frames)     | 2              | CPU-bound; leave cores for others|
| yt-dlp downloads    | 3              | Network-bound; parallelism helps |
| Total active sessions| 5             | Prevents cache explosion         |
+---------------------+----------------+----------------------------------+
```

### 10.4 Queue Priority

Processing priority (highest to lowest):

1. **Frame extraction** -- user is actively waiting
2. **Segment download** -- user is actively waiting
3. **LLM analysis** -- background, can be slower
4. **Image upload** -- background, can be batched

---

## 11. Data Models

### 11.1 TypeScript Interfaces (Frontend)

```typescript
// Video metadata (cached after first fetch)
interface VideoMetadata {
  id: string;                    // YouTube video ID
  title: string;
  description: string;
  duration: number;              // Total seconds
  channelName: string;
  thumbnailUrl: string;
  fetchedAt: string;             // ISO timestamp
}

// Transcript structure
interface Transcript {
  videoId: string;
  language: string;
  segments: TranscriptSegment[];
}

interface TranscriptSegment {
  start: number;                 // Start time in seconds
  duration: number;
  text: string;
}

// Cached video segment
interface VideoSegment {
  id: string;                    // "{videoId}_{startSec}_{endSec}"
  videoId: string;
  startTime: number;
  endTime: number;
  quality: '240p' | '360p' | '480p';
  status: 'pending' | 'downloading' | 'ready' | 'error';
}

// User-selected moment
interface VisualMoment {
  id: string;                    // UUID
  videoId: string;
  timestamp: number;             // Seconds
  source: 'user' | 'rule' | 'llm';
  confidence: number;            // 0.0 - 1.0
  reason?: string;               // Why this was flagged
  status: 'pending' | 'confirmed' | 'dismissed' | 'extracted' | 'uploaded';
  framePath?: string;            // Local path after extraction
  uploadedUrl?: string;          // External URL after upload
}

// Session state
interface UserSession {
  id: string;                    // UUID
  type: 'anonymous' | 'identified';
  userId?: string;               // If identified
  videoId?: string;              // Currently working on
  createdAt: string;
  lastActiveAt: string;
  selectedMoments: VisualMoment[];
}

// User account (identified users)
interface User {
  id: string;
  email?: string;
  hankoId: string;               // From Hanko
  googleConnected: boolean;
  googleTokenExpiry?: string;
  features: UserFeatures;
  createdAt: string;
}

interface UserFeatures {
  localLlmEnabled: boolean;
  rateLimitOverride?: number;    // Custom rate limit multiplier
}

// Final artifact
interface ExportArtifact {
  id: string;
  videoId: string;
  title: string;
  format: 'markdown' | 'json' | 'plaintext';
  includesImages: boolean;
  momentCount: number;
  exportedAt: string;
  downloadUrl: string;
}
```

### 11.2 SQLite Schema (Backend)

```sql
-- Main database schema

-- Video metadata cache
CREATE TABLE videos (
    id TEXT PRIMARY KEY,              -- YouTube video ID
    title TEXT NOT NULL,
    description TEXT,
    duration_seconds INTEGER NOT NULL,
    channel_name TEXT,
    thumbnail_url TEXT,
    fetched_at TEXT NOT NULL          -- ISO timestamp
);

-- Transcript cache
CREATE TABLE transcripts (
    video_id TEXT PRIMARY KEY REFERENCES videos(id),
    language TEXT NOT NULL,
    segments_json TEXT NOT NULL,      -- JSON array of segments
    fetched_at TEXT NOT NULL
);

-- Cached video segments
CREATE TABLE cached_segments (
    id TEXT PRIMARY KEY,              -- "{video_id}_{start}_{end}"
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

-- Anonymous sessions
CREATE TABLE anonymous_sessions (
    id TEXT PRIMARY KEY,              -- UUID
    video_id TEXT REFERENCES videos(id),
    created_at TEXT NOT NULL,
    last_active_at TEXT NOT NULL,
    locked_segments_json TEXT DEFAULT '[]',
    selected_moments_json TEXT DEFAULT '[]'
);

CREATE INDEX idx_anon_sessions_active ON anonymous_sessions(last_active_at);

-- Identified users (synced with Hanko)
CREATE TABLE users (
    id TEXT PRIMARY KEY,              -- UUID
    hanko_id TEXT UNIQUE NOT NULL,    -- From Hanko
    email TEXT,
    google_connected INTEGER DEFAULT 0,
    google_access_token TEXT,         -- Encrypted
    google_refresh_token TEXT,        -- Encrypted
    google_token_expiry TEXT,
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
    id TEXT PRIMARY KEY,              -- UUID
    user_id TEXT NOT NULL REFERENCES users(id),
    video_id TEXT REFERENCES videos(id),
    created_at TEXT NOT NULL,
    last_active_at TEXT NOT NULL,
    locked_segments_json TEXT DEFAULT '[]',
    selected_moments_json TEXT DEFAULT '[]'
);

CREATE INDEX idx_user_sessions_user ON user_sessions(user_id);
CREATE INDEX idx_user_sessions_active ON user_sessions(last_active_at);

-- LLM auto-suggestions (cached per video)
CREATE TABLE auto_suggestions (
    id TEXT PRIMARY KEY,              -- UUID
    video_id TEXT NOT NULL REFERENCES videos(id),
    timestamp_seconds REAL NOT NULL,
    confidence REAL NOT NULL,
    reason TEXT,
    visual_type TEXT,
    source TEXT NOT NULL,             -- 'rule', 'gemini', 'ollama'
    model_version TEXT,
    suggested_at TEXT NOT NULL
);

CREATE INDEX idx_suggestions_video ON auto_suggestions(video_id);

-- Gemini API usage tracking
CREATE TABLE gemini_usage (
    user_id TEXT NOT NULL REFERENCES users(id),
    date TEXT NOT NULL,               -- YYYY-MM-DD
    requests_count INTEGER DEFAULT 0,
    tokens_used INTEGER DEFAULT 0,
    PRIMARY KEY (user_id, date)
);

-- Admin audit log
CREATE TABLE admin_audit_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp TEXT NOT NULL,
    admin_ip TEXT NOT NULL,           -- Tailscale IP
    action TEXT NOT NULL,
    target_user_id TEXT,
    details_json TEXT
);
```

---

## 12. API Specification

### 12.1 Endpoints Overview

```
+--------+----------------------------------+----------------------------------------+
| Method | Path                             | Purpose                                |
+--------+----------------------------------+----------------------------------------+
| POST   | /api/videos/extract              | Fetch metadata + transcript for URL    |
| GET    | /api/videos/{id}                 | Get cached video metadata              |
| GET    | /api/videos/{id}/transcript      | Get transcript                         |
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
| GET    | /api/auth/google/authorize       | Start Google OAuth flow                |
| GET    | /api/auth/google/callback        | OAuth callback                         |
| DELETE | /api/auth/google                 | Disconnect Google account              |
| GET    | /api/auth/gemini/usage           | Get user's Gemini usage stats          |
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

### 12.2 Key Endpoint Details

**POST /api/videos/extract**

```typescript
// Request
{ 
  url: string;  // YouTube URL
}

// Response
{
  video: VideoMetadata;
  transcript: Transcript;
  cached: boolean;  // Whether we already had this
}

// Errors
// 400: Invalid URL
// 404: Video not found or unavailable
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
  quality?: '240p' | '360p' | '480p';  // Default 360p
}

// Response
{
  segmentId: string;
  status: 'ready' | 'downloading' | 'queued';
  estimatedWaitMs?: number;  // If not ready
  streamUrl?: string;        // If ready
}
```

**POST /api/videos/{id}/analyze**

```typescript
// Request
{
  sessionId: string;
  mode: 'rules' | 'cloud' | 'local';
}

// Response
{
  taskId: string;          // For polling status
  status: 'processing' | 'complete';
  moments?: VisualMoment[];  // If complete
}

// Errors
// 401: Cloud mode requires Google connection
// 403: Local mode requires feature flag
```

**POST /api/export**

```typescript
// Request
{
  sessionId: string;
  videoId: string;
  moments: string[];           // Moment IDs to include
  format: 'markdown' | 'json' | 'plaintext';
  options: {
    includeImages: boolean;    // Embed base64 images
    includeTimestampLinks: boolean;
    imageQuality?: 'low' | 'medium' | 'high';
  };
}

// Response
{
  exportId: string;
  status: 'processing' | 'ready';
  downloadUrl?: string;  // If ready
}
```

### 12.3 WebSocket Protocol

**Connection:** `wss://yourdomain.com/ws/session/{sessionId}`

**Messages (Server -> Client):**

```typescript
// Segment download progress
{
  type: 'segment_progress';
  segmentId: string;
  progress: number;  // 0-100
  status: 'downloading' | 'ready' | 'error';
}

// Analysis progress
{
  type: 'analysis_progress';
  videoId: string;
  stage: 'rules' | 'llm';
  progress: number;
  momentsFound: number;
}

// New moment detected
{
  type: 'moment_detected';
  moment: VisualMoment;
}

// Export ready
{
  type: 'export_ready';
  exportId: string;
  downloadUrl: string;
}

// Session warning
{
  type: 'session_warning';
  message: string;  // e.g., "Session expiring in 1 minute"
}
```

**Messages (Client -> Server):**

```typescript
// Heartbeat (every 30s)
{
  type: 'heartbeat';
}

// Acknowledge moment
{
  type: 'moment_ack';
  momentId: string;
  action: 'confirm' | 'dismiss';
}
```

---

## 13. Export Artifact Format

### 13.1 Design for AI Consumption

**Decision:** Primary format is **Markdown with base64-embedded images**.

**Why Base64?**

- **Self-contained:** No external dependencies, works offline
- **LLM-compatible:** Claude, GPT-4V, and Gemini all accept base64 images in API calls
- **Portable:** Single file, easy to share/archive
- **No link rot:** External image hosts can go down; base64 is forever

### 13.2 Markdown Format Specification

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

### 13.3 JSON Format (Programmatic Use)

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
    "detection_mode": "cloud"
  }
}
```

### 13.4 Plain Text Format (Fallback)

```
VIDEO: [Title]
URL: https://youtube.com/watch?v=VIDEO_ID
DURATION: 45:00
EXTRACTED: 2024-12-14

DESCRIPTION:
[Original description]

----------------------------------------

TRANSCRIPT:

[00:00] Welcome to today's tutorial...
[00:15] As you can see here... [VISUAL: Code editor - see image 1]
[00:32] The CSS flexbox model...
[00:45] Let me show you this diagram... [VISUAL: Diagram - see image 2]
...

----------------------------------------

VISUAL MOMENTS:
1. 00:15 - Code editor showing HTML structure (code)
2. 00:45 - Flexbox diagram explaining axis (diagram)
3. 02:30 - Browser DevTools demonstration (ui)

----------------------------------------
Generated by [Tool Name]
```

---

## 14. Admin Dashboard

### 14.1 Access Method

**URL:** `http://<your-tailscale-hostname>:8443/admin` (Tailscale only)

No login required -- if you can reach it, you're authorized (Tailscale identity).

### 14.2 Dashboard Sections

**Overview Page:**

```
+-----------------------------------------------------------------------+
|  ADMIN DASHBOARD                                         [Your TS ID] |
+-----------------------------------------------------------------------+
|                                                                       |
|  System Health                                                        |
|  +------------------+  +------------------+  +------------------+     |
|  |  CPU: 23%        |  |  RAM: 1.8/8 GB   |  |  Cache: 4.2 GB   |     |
|  |  ####......      |  |  ######....      |  |  ########..      |     |
|  +------------------+  +------------------+  +------------------+     |
|                                                                       |
|  Active Sessions: 3     Queue Depth: 2      Gemini Calls Today: 47    |
|                                                                       |
+-----------------------------------------------------------------------+
|                                                                       |
|  Recent Activity                                                      |
|  -----------------------------------------------------------------    |
|  * 2 min ago   user_abc    Extracted video (tutorial, 12 moments)     |
|  * 5 min ago   anon_xyz    Exported artifact (rules only)             |
|  * 12 min ago  user_abc    Connected Google account                   |
|                                                                       |
+-----------------------------------------------------------------------+
```

**User Management Page:**

```
+-----------------------------------------------------------------------+
|  Users                                          [Search] [Export CSV] |
+-----------------------------------------------------------------------+
|                                                                       |
|  ID         | Type       | Google | Local LLM | Videos | Last Active  |
|  -----------+------------+--------+-----------+--------+--------------+
|  user_abc   | Identified | [x]    | [x]       | 23     | 2 min ago    |
|  user_def   | Identified | [x]    | [ ]       | 8      | 1 hour ago   |
|  anon_xyz   | Anonymous  | [ ]    | [ ]       | 2      | 5 min ago    |
|                                                                       |
|  [Click row for details / feature management]                         |
|                                                                       |
+-----------------------------------------------------------------------+
```

**User Detail / Feature Flags Page:**

```
+-----------------------------------------------------------------------+
|  User: user_abc                                             [<- Back] |
+-----------------------------------------------------------------------+
|                                                                       |
|  Account Details                                                      |
|  -----------------------------------------------------------------    |
|  Created: 2024-12-01                                                  |
|  Email: user@example.com (via Hanko)                                  |
|  Google: Connected (gemini scope)                                     |
|                                                                       |
|  Feature Flags                                                        |
|  -----------------------------------------------------------------    |
|                                                                       |
|  Local LLM Access     [=========] ON                                  |
|                       Granted: 2024-12-10                             |
|                                                                       |
|  Rate Limit Override  [.........] OFF                                 |
|                       Default limits apply                            |
|                                                                       |
|  [Save Changes]                                                       |
|                                                                       |
+-----------------------------------------------------------------------+
```

**Cache Management Page:**

```
+-----------------------------------------------------------------------+
|  Cache Status                                                         |
+-----------------------------------------------------------------------+
|                                                                       |
|  Total Size: 4.2 GB / 10 GB                                           |
|  ####################....................                             |
|                                                                       |
|  Segments: 234 files (3.8 GB)                                         |
|  Frames: 1,203 files (0.3 GB)                                         |
|  Metadata: 89 files (0.1 GB)                                          |
|                                                                       |
|  Currently Locked: 12 segments (3 sessions)                           |
|                                                                       |
|  [Clear Unlocked Segments]  [Clear All Frames]  [Clear Everything]    |
|                                                                       |
+-----------------------------------------------------------------------+
```

---

## 15. Visual Polish & Cool Factor

### 15.1 Framework Stack

**Building blocks:**
```
shadcn/ui (foundation)
        |
        v
Framer Motion (micro-interactions)
        |
        v
Aceternity UI / Magic UI (hero effects)
        |
        v
React Three Fiber (background shaders) -- optional
```

**Component strategy:**
```
+----------------------------------+------------------------+-----------------------------+
| Component Need                   | Solution               | Source                      |
+----------------------------------+------------------------+-----------------------------+
| Forms, buttons, dialogs, inputs  | shadcn/ui              | Standard, accessible        |
| Transcript list (virtual scroll) | shadcn + custom        | @tanstack/react-virtual     |
| Video segment player             | Custom build           | No OOTB option              |
| Timeline with segment indicators | Custom build           | Unique requirement          |
| Frame review gallery             | shadcn/ui + Framer     | Lightbox with gestures      |
| Loading states, skeletons        | shadcn/ui              | Standard                    |
| Toasts, notifications            | sonner                 | shadcn default              |
| Hero animations                  | Aceternity UI          | Pre-built wow effects       |
| Micro-interactions               | Framer Motion          | Entrance/exit animations    |
| Background effects               | Three.js or CSS        | Optional shader             |
+----------------------------------+------------------------+-----------------------------+
```

### 15.2 Key "Wow" Moments

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
| Background         | Subtle gradient shift          | CSS or Three.js  |
+--------------------+--------------------------------+------------------+
```

### 15.3 Mobile-Specific Polish

```
+--------------------+--------------------+-----------------------------+
| Interaction        | Desktop            | Mobile                      |
+--------------------+--------------------+-----------------------------+
| Moment select      | Click + hover glow | Long-press + haptic + ripple|
| Delete             | Small x button     | Swipe left + red reveal     |
| Reorder            | Drag handle        | Drag with lift shadow       |
| Settings           | Dropdown           | Bottom sheet slide-up       |
| Loading            | Spinner            | Skeleton shimmer            |
+--------------------+--------------------+-----------------------------+
```

### 15.4 Performance Budget

**Golden Rule:** One "hero" effect per view. Don't stack multiple expensive animations.

```
+----------------------+------------------+---------------------------------+
| Effect               | Performance Cost | Mitigation                      |
+----------------------+------------------+---------------------------------+
| Three.js background  | Medium (GPU)     | Simple shader, pause when hidden|
| Framer Motion        | Low              | Use layout prop wisely          |
| Aceternity effects   | Varies           | Pick lightweight ones           |
| Particle effects     | Medium-High      | Limit count, use CSS if possible|
| Lottie               | Low              | Pre-optimized vectors           |
+----------------------+------------------+---------------------------------+
```

### 15.5 `shadcn` Extensibility Assessment

**Strengths:**
- Headless architecture (you own the code, not node_modules)
- Tailwind-based, easy to modify
- Accessible by default (Radix primitives)
- Great starting point for custom components

**Gaps for This Project:**
- No video player component (expected—this is specialized)
- No timeline/range-with-markers component (we'll build it)
- No virtualized list (use @tanstack/react-virtual)

**Verdict:** shadcn is the right foundation. The gaps are specific to our unique features and would exist with any general-purpose UI library.


### 15.6 Cool Libraries
```
+--------------------+---------------------------------------------+--------------------------------+
| Library            | What It Does                                | Use Case in Our App            |
+--------------------+---------------------------------------------+--------------------------------+
| Aceternity UI      | Animated components (cards, backgrounds)    | Hero section, card hovers      |
| Magic UI           | Animated gradients, text shimmer, borders   | Text effects, borders          |
| Framer Motion      | Production animation library                | Transitions, reordering        |
| React Three Fiber  | React renderer for Three.js                 | Background shader (optional)   |
| Lottie             | After Effects animations in web             | Success/error state animations |
| GSAP               | Professional animation (complex timelines)  | Scroll-triggered sequences     |
+--------------------+---------------------------------------------+--------------------------------+
```

---

## 16. Deployment Architecture

### 16.1 Process Layout

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
|  |  (2 workers)|      |  (2 workers)|      |  (1 worker) |           |
|  +-------------+      +-------------+      +-------------+           |
|                              |                                       |
|              +---------------+---------------+                       |
|              v                               v                       |
|       +-------------+                 +-------------+                |
|       |  Huey       |                 |  Ollama     |                |
|       |  Worker     |                 |  :11434     |                |
|       |  (1 process)|                 |  (on-demand)|                |
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
|  |  /var/lib/postgresql/data                                      |  |
|  +----------------------------------------------------------------+  |
|                                                                      |
+----------------------------------------------------------------------+
```

### 16.2 systemd Service Units

**FastAPI Service:**

```ini
# /etc/systemd/system/yttool-api.service
[Unit]
Description=YouTube Tool API
After=network.target

[Service]
Type=simple
User=yttool
WorkingDirectory=/opt/yttool
Environment=PATH=/opt/yttool/venv/bin
ExecStart=/opt/yttool/venv/bin/uvicorn app.main:app \
    --host 127.0.0.1 \
    --port 8000 \
    --workers 2
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

**Huey Worker Service:**

```ini
# /etc/systemd/system/yttool-worker.service
[Unit]
Description=YouTube Tool Background Worker
After=network.target

[Service]
Type=simple
User=yttool
WorkingDirectory=/opt/yttool
Environment=PATH=/opt/yttool/venv/bin
ExecStart=/opt/yttool/venv/bin/huey_consumer app.tasks.huey \
    --workers 2 \
    --worker-type thread
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

**Next.js Service:**

```ini
# /etc/systemd/system/yttool-frontend.service
[Unit]
Description=YouTube Tool Frontend
After=network.target

[Service]
Type=simple
User=yttool
WorkingDirectory=/opt/yttool/frontend
ExecStart=/usr/bin/node server.js
Environment=PORT=3000
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

---

## 17. Implementation Phases

### Phase 1: Foundation (Week 1-2)

- [ ] FastAPI + SQLite setup
- [ ] yt-dlp transcript extraction
- [ ] Segment download with quality selection
- [ ] Basic cache manager (no locking yet)
- [ ] CLI for testing

**Deliverable:** `python cli.py "youtube-url"` outputs transcript JSON

### Phase 2: Auth Foundation (Week 2-3)

- [ ] Hanko self-hosted deployment
- [ ] Hanko frontend components integration
- [ ] Anonymous session system
- [ ] Session upgrade flow (anon -> identified)
- [ ] Tailscale admin route protection via Caddy

**Deliverable:** Users can sign up with passkey or Google, sessions persist

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
- [ ] Session locking for cache
- [ ] Manual moment selection
- [ ] Touch-friendly interactions

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
- [ ] Google OAuth for Gemini scopes
- [ ] Gemini API integration
- [ ] Usage tracking + display
- [ ] Result merging + deduplication

**Deliverable:** Auto-suggested moments from rules and/or Gemini

### Phase 7: Local LLM + Admin (Week 7-8)

- [ ] Ollama setup with Qwen2.5-1.5B
- [ ] Feature flag system in database
- [ ] Admin dashboard (Tailscale-only)
- [ ] User management UI
- [ ] Process limiter integration

**Deliverable:** Admin can grant local LLM access to specific users

### Phase 8: Export + Final Polish (Week 8-9)

- [ ] Markdown artifact generation with base64 images
- [ ] JSON and plaintext export options
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

## 18. Risk Assessment

```
+---------------------------------+------------+--------+---------------------------+
| Risk                            | Likelihood | Impact | Mitigation                |
+---------------------------------+------------+--------+---------------------------+
| YouTube blocks yt-dlp           | Medium     | High   | Monitor releases; fallback|
|                                 |            |        | to Invidious API          |
+---------------------------------+------------+--------+---------------------------+
| KVM 4 runs out of RAM           | Medium     | Medium | Strict limits; swap file  |
|                                 |            |        | as safety net             |
+---------------------------------+------------+--------+---------------------------+
| LLM quality insufficient        | Low        | Medium | Cloud API fallback; prompt|
|                                 |            |        | iteration                 |
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
| Google OAuth scope changes      | Low        | High   | Monitor API announcements;|
|                                 |            |        | graceful degradation      |
+---------------------------------+------------+--------+---------------------------+
```

---

## 19. Open Questions

1. **Branding:** What should this tool be called?
2. **Domain:** What domain will this run on?
3. **Monetization:** Future consideration for premium features?
4. **Multi-language:** Priority for non-English transcripts?
5. **Collaboration:** Multiple users working on same video?
6. **Public gallery:** Should finished artifacts be shareable publicly?

---

## 20. Appendix: Dependencies

### 20.1 Backend (Python)

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
passlib[bcrypt]>=1.7.4
google-generativeai>=0.3.0
google-auth>=2.25.0
google-auth-oauthlib>=1.2.0
python-dotenv>=1.0.0
structlog>=24.1.0
```

### 20.2 Frontend (Node)

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

### 20.3 System Packages

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

### 20.4 External Services

```
+-------------------+---------------------------+---------------------------+
| Service           | Purpose                   | Account Required          |
+-------------------+---------------------------+---------------------------+
| Google Cloud      | OAuth + Gemini API        | Yes (for OAuth setup)     |
| imgbb             | Image hosting             | Optional (API key)        |
| Tailscale         | Admin network access      | Yes (your account)        |
| SMTP Provider     | Hanko magic links         | Yes (any provider)        |
+-------------------+---------------------------+---------------------------+
```

---

## 21. Revision History

```
+---------+------------+--------------------------------------------------------+
| Version | Date       | Changes                                                |
+---------+------------+--------------------------------------------------------+
| 1.0     | 2024-12-14 | Initial draft                                          |
+---------+------------+--------------------------------------------------------+
| 2.0     | 2024-12-14 | Added: Auth architecture, mobile UX, 3-tier LLM,       |
|         |            | Tailscale admin, elevated Moments Universe             |
+---------+------------+--------------------------------------------------------+
| 3.0     | 2024-12-14 | Finalized: Locked Hanko, removed Passage, detailed     |
|         |            | Google OAuth flows, ASCII-safe diagrams, restored      |
|         |            | cache/process/data model sections from v1, added       |
|         |            | systemd configs, expanded API spec, added dependencies |
+---------+------------+--------------------------------------------------------+
| 3.1     | 2024-12-14 | Not really finalized. Opus got straight ass after the  |
|         |            | second iteration or so, added stuff that's completely  |
|         |            | irrelevant (systemd resource files) and just got lazy  |
|         |            | about others (details lost between v1 and v2).         |
|         |            |                                                        |
|         |            |                                                        |
+---------+------------+--------------------------------------------------------+
```

---

*End of Document*
