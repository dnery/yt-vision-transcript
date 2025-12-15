# ADR-001: YouTube Transcript Extraction Tool Architecture

**Status:** Proposed  
**Version:** 2.0  
**Date:** 2024-12-14  
**Decision Makers:** Project Owner  
**Context:** Personal project, learning-focused, resource-constrained hosting

---

## 1. Executive Summary

A web application that extracts YouTube transcripts and enriches them with visual context (timestamps or embedded images) to produce AI-consumable text artifacts. The system must run comfortably on a Hostinger KVM 4 VPS alongside other services, handle video content efficiently through intelligent caching, and provide a polished, visually impressive user experience with accessibility as a primary concern.

**Core Philosophy:** Maximum functionality at minimum friction. Beautiful by default. Accessible to everyone, especially those of us with fat fingers.

---

## 2. System Context & Constraints

### 2.1 Infrastructure Constraints

**Target Environment: Hostinger KVM 4**
| Resource | Allocation | Reserved for This Project |
|----------|------------|---------------------------|
| vCPU | 4 cores | 1-2 cores max during processing |
| RAM | 8 GB | 2 GB ceiling (soft limit) |
| Storage | 100 GB NVMe | 10-15 GB for video cache |
| Bandwidth | 8 TB/month | Variable, optimize for low usage |

**Key Implication:** Every architectural decision must be evaluated against "can this run as a background service without starving other workloads?"

### 2.2 Functional Requirements

1. Extract title, description, and timestamped transcript from YouTube URLs
2. **The Moments Universe:** A core, beautiful experience for identifying visual moments
3. Preview video segments without downloading entire videos
4. Three-tier moment detection: Rules → Cloud LLM → Local LLM
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
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLIENT (Browser)                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Next.js App                                                         │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────────┐ │   │
│  │  │ URL Input &  │  │ Transcript   │  │ ✨ MOMENTS UNIVERSE ✨     │ │   │
│  │  │ Metadata View│  │ Viewer       │  │ (Segment Player + Picker)  │ │   │
│  │  └──────────────┘  └──────────────┘  └────────────────────────────┘ │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────────┐ │   │
│  │  │ Frame Review │  │ Export       │  │ Processing Status          │ │   │
│  │  │ Galaxy       │  │ Controls     │  │ (WebSocket)                │ │   │
│  │  └──────────────┘  └──────────────┘  └────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │ HTTP/REST + WebSocket
┌─────────────────────────────────────▼───────────────────────────────────────┐
│                           BACKEND (Python/FastAPI)                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         API Gateway Layer                            │   │
│  │  - Rate limiting    - Session/Auth     - Tailscale Admin Check      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         Service Layer                                │   │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────────────┐ │   │
│  │  │ Transcript     │  │ Video Segment  │  │ Moment Detection       │ │   │
│  │  │ Extractor      │  │ Manager        │  │ Engine (3-tier)        │ │   │
│  │  └────────────────┘  └────────────────┘  └────────────────────────┘ │   │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────────────┐ │   │
│  │  │ Frame          │  │ Image Upload   │  │ Artifact Compiler      │ │   │
│  │  │ Extractor      │  │ Service        │  │                        │ │   │
│  │  └────────────────┘  └────────────────┘  └────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      Infrastructure Layer                            │   │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────────────┐ │   │
│  │  │ Task Queue     │  │ Cache Manager  │  │ Process Limiter        │ │   │
│  │  │ (Huey/SQLite)  │  │ (LRU + Locks)  │  │ (Semaphore Pool)       │ │   │
│  │  └────────────────┘  └────────────────┘  └────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
          ┌───────────────────────────┼───────────────────────────┐
          ▼                           ▼                           ▼
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────────────┐
│ YouTube          │    │ Google Cloud     │    │ Tailscale                │
│ (via yt-dlp)     │    │ - Gemini API     │    │ - Admin identity         │
│                  │    │ - OAuth          │    │ - Network-level ACL      │
└──────────────────┘    └──────────────────┘    └──────────────────────────┘
```

---

## 4. Authentication & Identity Architecture

### 4.1 Design Philosophy

**"Progressive Identity"** — Users should get value immediately, with identity becoming relevant only when they want persistence or advanced features.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         IDENTITY PROGRESSION                                │
│                                                                             │
│   ┌─────────────┐      ┌─────────────┐      ┌─────────────┐                │
│   │  ANONYMOUS  │ ───▶ │  IDENTIFIED │ ───▶ │   ADMIN     │                │
│   │  SESSION    │      │   (OIDC)    │      │ (Tailscale) │                │
│   └─────────────┘      └─────────────┘      └─────────────┘                │
│                                                                             │
│   Features:            Features:            Features:                       │
│   • Full core UX       • All anonymous      • All identified               │
│   • Session storage    • Persistent data    • Feature flag control         │
│   • No persistence     • Cloud LLM access   • Local LLM toggle             │
│   • Rate limited       • Higher rate limits • Usage dashboards             │
│                        • History            • User management              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
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
| Action | Limit | Window |
|--------|-------|--------|
| Video extractions | 10 | per hour |
| Segment downloads | 50 | per hour |
| Frame extractions | 30 | per hour |
| Exports | 5 | per hour |

### 4.3 Identified Users (OIDC with Passkeys)

**Decision:** Use **Hanko** (open-source, self-hostable) or **Passage by 1Password** for passkey-first authentication.

**Why Passkeys?**
- Zero password friction (no "forgot password" flows)
- Phishing-resistant by design
- Modern UX that feels premium
- Offloads security to user's device/biometrics

**Why Hanko specifically?**
- Open source, self-hostable (fits our KVM 4)
- Passkey-first with fallback to email magic links
- Simple integration (drop-in web components)
- No vendor lock-in

**Authentication Flow:**
```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   User       │     │   Frontend   │     │   Hanko      │
│   Browser    │     │   (Next.js)  │     │   (self-host)│
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                    │
       │  Click "Sign In"   │                    │
       │───────────────────▶│                    │
       │                    │                    │
       │  Render Hanko UI   │                    │
       │◀───────────────────│                    │
       │                    │                    │
       │  Passkey prompt    │                    │
       │  (Touch ID/Face ID)│                    │
       │◀───────────────────┼───────────────────▶│
       │                    │                    │
       │  Biometric auth    │                    │
       │───────────────────▶│                    │
       │                    │   Verify           │
       │                    │───────────────────▶│
       │                    │                    │
       │                    │   JWT + Session    │
       │                    │◀───────────────────│
       │                    │                    │
       │  Logged in! 🎉     │                    │
       │◀───────────────────│                    │
       │                    │                    │
       │  (Total time: ~3 seconds)              │
       │                    │                    │
```

**Hanko Deployment (on KVM 4):**
```yaml
# docker-compose.yml snippet
services:
  hanko:
    image: ghcr.io/teamhanko/hanko:latest
    environment:
      - DATABASE_URL=postgres://...
      - SECRETS_KEYS=your-secret-key
      - WEBAUTHN_RELYING_PARTY_ID=yourdomain.com
      - WEBAUTHN_RELYING_PARTY_ORIGIN=https://yourdomain.com
    ports:
      - "127.0.0.1:8001:8000"  # Only local, proxied by Caddy
```

**Session Upgrade (Anonymous → Identified):**
```python
async def upgrade_session(anonymous_id: str, user_id: str):
    """Migrate anonymous session data to identified user."""
    anon_data = await get_anonymous_session(anonymous_id)
    if anon_data:
        # Migrate moments, history, preferences
        await merge_into_user_account(user_id, anon_data)
        await delete_anonymous_session(anonymous_id)
```

### 4.4 Google OAuth (For Gemini + YouTube Integration)

**Decision:** Offer optional Google OAuth for users who want Cloud LLM features.

**Why Google OAuth specifically?**
1. **Gemini API access** via user's own Google account (no BYOK needed)
2. **YouTube Data API** access for better metadata (user's own videos, unlisted content)
3. **Most users already have Google accounts** (lowest friction)
4. **Single OAuth grants multiple capabilities**

**Scopes Requested:**
```
openid                           # Basic identity
email                            # For account linking
https://www.googleapis.com/auth/generative-language  # Gemini API
https://www.googleapis.com/auth/youtube.readonly     # YouTube metadata (optional)
```

**User-Facing UI:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ⚡ Enhance with AI                                             │
│                                                                 │
│  Get smarter moment suggestions by connecting your Google       │
│  account. We'll use Gemini to analyze transcripts.              │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  🔗 Connect Google Account                               │   │
│  │                                                          │   │
│  │  This grants access to:                                  │   │
│  │  ✓ Gemini AI (for smart suggestions)                     │   │
│  │  ✓ Your YouTube library (optional, for unlisted videos)  │   │
│  │                                                          │   │
│  │  We never store your Google password.                    │   │
│  │  Disconnect anytime in settings.                         │   │
│  │                                                          │   │
│  │  [Continue with Google]                                  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ─────────────────── or ───────────────────                    │
│                                                                 │
│  [Continue without AI] → Uses rule-based detection only         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Token Storage:**
- Access tokens stored server-side, encrypted at rest
- Refresh tokens stored securely, auto-refresh on expiry
- Tokens scoped to user, never shared
- User can revoke via settings (we also call Google's revoke endpoint)

### 4.5 Admin Access via Tailscale

**Decision:** Admin functionality is only accessible from the Tailscale network. No code-level auth checks needed—network topology *is* the auth.

**Why Tailscale?**
- Zero-trust networking without VPN complexity
- Identity tied to your Tailscale account
- Can expose specific routes only to your tailnet
- Already have it set up (your requirement!)
- Elegant: if you can reach the admin route, you're authorized

**Implementation Architecture:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              INTERNET                                       │
│                                                                             │
│    Public Users ───────────────────┐                                       │
│                                    │                                       │
│                                    ▼                                       │
│                         ┌──────────────────┐                               │
│                         │  Caddy (Public)  │                               │
│                         │  :443            │                               │
│                         │                  │                               │
│                         │  Routes:         │                               │
│                         │  /* → Next.js    │                               │
│                         │  /api/* → FastAPI│                               │
│                         │                  │                               │
│                         │  BLOCKS:         │                               │
│                         │  /admin/* → 404  │◀── Admin routes not exposed   │
│                         └──────────────────┘    to public internet         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                            YOUR TAILNET                                     │
│                                                                             │
│    Your Device ────────────────────┐                                       │
│    (Tailscale connected)           │                                       │
│                                    ▼                                       │
│                         ┌──────────────────┐                               │
│                         │ Caddy (Tailscale)│                               │
│                         │ :8443 (TS only)  │                               │
│                         │                  │                               │
│                         │ Routes:          │                               │
│                         │ /admin/* → ✓     │◀── Full admin access          │
│                         │ /* → proxy to    │                               │
│                         │     public Caddy │                               │
│                         └──────────────────┘                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Caddy Configuration:**

```caddyfile
# Public-facing (Internet)
yourdomain.com {
    # Public routes
    handle /api/* {
        reverse_proxy localhost:8000
    }
    
    handle {
        reverse_proxy localhost:3000
    }
    
    # Block admin routes from public internet
    handle /admin/* {
        respond "Not Found" 404
    }
}

# Tailscale-only (Admin)
yourdomain.com:8443 {
    # Only bind to Tailscale interface
    bind tailscale/yourdomain
    
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

**Alternative: Tailscale Serve/Funnel**

```bash
# Expose admin only to your tailnet
tailscale serve --bg --set-path /admin http://localhost:8000/admin

# Public routes via Funnel (optional)
tailscale funnel --bg https://localhost:443
```

**Admin Capabilities:**

| Capability | Description |
|------------|-------------|
| User Management | View all users, usage stats |
| Feature Flags | Toggle local LLM per user |
| Rate Limit Override | Adjust limits for specific users |
| Cache Management | View/clear cache, see stats |
| System Health | Resource usage, queue depth |
| LLM Usage Dashboard | Gemini API calls, costs |

**Admin UI Location:** `https://yourdomain.com:8443/admin` (only reachable via Tailscale)

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

| Element Type | Minimum Size | Ideal Size | Spacing from neighbors |
|--------------|--------------|------------|------------------------|
| Primary actions | 48×48px | 56×56px | 8px |
| Secondary actions | 44×44px | 48×48px | 8px |
| List items | 48px height | 56px height | 0 (full-width) |
| Timeline markers | 44×44px | 48×48px | 12px |
| Close/dismiss | 48×48px | 56×56px | Corner, 16px padding |

**Visual vs. Touch Target:**

```
┌─────────────────────────────────────────┐
│                                         │
│    ┌─────────────────────────────┐      │
│    │  ┌───────────────────────┐  │      │
│    │  │                       │  │      │
│    │  │    Visual Button      │  │◀── What user sees (32×32)
│    │  │    (smaller)          │  │      │
│    │  │                       │  │      │
│    │  └───────────────────────┘  │      │
│    │                             │      │
│    │     Touch Target Area       │◀── Actual tappable area (48×48)
│    │     (larger, invisible)     │      │
│    │                             │      │
│    └─────────────────────────────┘      │
│                                         │
└─────────────────────────────────────────┘
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
      // Visual size can be smaller
      // Touch target extends via padding
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

| Instead of... | Use... | Benefit |
|---------------|--------|---------|
| Small "×" close button | Swipe down to dismiss | Whole screen is target |
| Tiny edit/delete icons | Swipe left on item | Natural, discoverable |
| Small +/- buttons | Pinch to zoom timeline | Intuitive, precise |
| Checkbox for select | Long-press to select | Harder to mis-tap |
| Settings gear icon | Pull down past top | Hidden but accessible |

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
┌─────────────────────┐          ┌─────────────────────┐
│                     │          │                     │
│  ┌───────────────┐  │          │                     │
│  │    Modal      │  │          │                     │
│  │   (centered)  │  │    vs    │                     │
│  │               │  │          │                     │
│  └───────────────┘  │          ├─────────────────────┤
│                     │          │ ═══════════════════ │◀─ Drag handle
└─────────────────────┘          │                     │
                                 │   Bottom Sheet      │◀─ Thumb-reachable
                                 │   (actions here)    │
                                 │                     │
                                 │  [ Big Button ]     │
                                 │                     │
                                 └─────────────────────┘
```

**Thumb Zone Optimization:**

```
┌─────────────────────────────────────┐
│                                     │
│          😰 HARD TO REACH           │   ← Avoid placing actions here
│                                     │
├─────────────────────────────────────┤
│                                     │
│          😐 OKAY TO REACH           │   ← Secondary actions okay
│                                     │
├─────────────────────────────────────┤
│                                     │
│          😊 EASY TO REACH           │   ← Primary actions HERE
│                                     │
│    [  Primary Action Button  ]      │
│                                     │
└─────────────────────────────────────┘
        ↑ Thumb naturally rests here
```

### 5.5 Forgiving Interactions

**Error Prevention:**

| Situation | Solution |
|-----------|----------|
| Accidental tap | 300ms delay before destructive actions |
| Tap near multiple elements | Enlarge closest target's hitbox |
| Shaky hands | Debounce rapid successive taps |
| Scrolling vs. tapping | Require stationary touch for tap |

**Undo Everything:**

```tsx
// Every destructive action is reversible
const handleDeleteMoment = async (momentId: string) => {
  // Optimistically remove from UI
  setMoments(prev => prev.filter(m => m.id !== momentId));
  
  // Show undo toast
  const { dismiss } = toast({
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
┌─────────────────────────────────────┐
│  ◀  Video Title Here...        ⚙️  │  ← Compact header
├─────────────────────────────────────┤
│                                     │
│  ┌─────────────────────────────┐   │
│  │                             │   │
│  │      Video Preview          │   │  ← 16:9 aspect
│  │      (Segment Player)       │   │
│  │                             │   │
│  └─────────────────────────────┘   │
│                                     │
│  ░░░████░░░░██████░░░░█░░░░░░░░░░  │  ← Timeline (expandable)
│                                     │
├─────────────────────────────────────┤
│                                     │
│  Transcript                    🔍  │
│  ─────────────────────────────────  │
│                                     │
│  │ 0:00  Welcome to today's...     │
│  │                                 │  ← Scrollable transcript
│  │ 0:15  As you can see here... ●  │  ← ● = moment marker
│  │                                 │
│  │ 0:32  The diagram shows...  ●   │
│  │                                 │
│                                     │
│                         [ + Add ]   │  ← FAB for adding moments
│                                     │
├─────────────────────────────────────┤
│                                     │
│  ═══════════════════════════════   │  ← Bottom sheet handle
│                                     │
│  Moments (3)              [Export]  │
│                                     │
│  ┌──────┐ ┌──────┐ ┌──────┐       │  ← Horizontal scroll
│  │ 🖼️   │ │ 🖼️   │ │ 🖼️   │       │
│  │ 0:15 │ │ 0:32 │ │ 1:45 │       │
│  └──────┘ └──────┘ └──────┘       │
│                                     │
└─────────────────────────────────────┘
```

**Landscape Mode (Immersive Player):**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│                                                                             │
│                                                                             │
│                         Video Preview (Full Width)                          │
│                                                                             │
│                                                                             │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  ░░░░░████████░░░░░░░░░░████░░░░░░░░░░░░░░░░████████████░░░░░░░░░░░░  ●    │
│  ▲                                                                    ▲    │
│  Tap anywhere on timeline to seek              Tap ● to jump to moment     │
│                                                                             │
│  [Exit]                                              [Mark Moment] (large) │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. The Moments Universe (Core Feature)

### 6.1 Concept

The **Moments Universe** is not just a feature—it's the soul of the application. It's where users discover, select, and curate the visual moments that transform a transcript into a complete, AI-comprehensible document.

**Design Principles:**
1. **Discoverable:** Moments should feel like they're floating in a space, waiting to be found
2. **Tactile:** Every interaction should feel physical and satisfying
3. **Forgiving:** Wrong selections are trivially reversible
4. **Beautiful:** This is where we go all-in on cool factor

### 6.2 Visual Design: The Timeline Constellation

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│                              THE TIMELINE                                   │
│                                                                             │
│     0:00                                                            45:00   │
│       │                                                               │     │
│       ▼                                                               ▼     │
│   ╔═══════════════════════════════════════════════════════════════════╗    │
│   ║                                                                   ║    │
│   ║  ░░░░░▓▓▓▓▓▓▓▓░░░░░░░░░░▓▓▓▓░░░░░░░░░░░░░░░░▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░  ║    │
│   ║       ╱    ╲              ╲                      ╲                ║    │
│   ║      ╱      ╲              ╲                      ╲               ║    │
│   ║     ╱        ╲              ╲                      ╲              ║    │
│   ║    ●          ●              ◐                      ●             ║    │
│   ║   0:15       0:45           5:20                  32:10           ║    │
│   ║  (locked)   (locked)      (pending)              (auto)          ║    │
│   ║                                                                   ║    │
│   ╚═══════════════════════════════════════════════════════════════════╝    │
│                                                                             │
│   Legend:                                                                   │
│   ░░░░ = Uncached segment (dim, clickable to load)                         │
│   ▓▓▓▓ = Cached segment (bright, playable)                                 │
│   ● = Confirmed moment (solid, pulsing glow)                               │
│   ◐ = Pending moment (half-filled, waiting confirmation)                   │
│   ○ = Auto-suggested moment (outline only, needs review)                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.3 Moment States & Animations

**State Machine:**

```
                    ┌─────────────┐
                    │   EMPTY     │
                    │  (no moment)│
                    └──────┬──────┘
                           │
            ┌──────────────┼──────────────┐
            │              │              │
            ▼              ▼              ▼
    ┌───────────────┐ ┌─────────┐ ┌───────────────┐
    │ AUTO-DETECTED │ │  USER   │ │   RULE-BASED  │
    │  (by LLM)     │ │ SELECTED│ │   (pattern)   │
    └───────┬───────┘ └────┬────┘ └───────┬───────┘
            │              │              │
            └──────────────┼──────────────┘
                           │
                           ▼
                    ┌─────────────┐
                    │   PENDING   │
                    │ (needs ack) │
                    └──────┬──────┘
                           │
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
       ┌─────────────┐          ┌─────────────┐
       │  CONFIRMED  │          │  DISMISSED  │
       │  (locked)   │          │  (removed)  │
       └──────┬──────┘          └─────────────┘
              │
              ▼
       ┌─────────────┐
       │   FRAME     │
       │  EXTRACTED  │
       └──────┬──────┘
              │
              ▼
       ┌─────────────┐
       │  UPLOADED   │
       │  (ready)    │
       └─────────────┘
```

**Animation Specifications:**

| Transition | Animation | Duration | Easing |
|------------|-----------|----------|--------|
| Appear (new moment) | Scale from 0 + fade | 300ms | spring(0.5, 0.7) |
| Pending → Confirmed | Pulse + fill | 200ms | ease-out |
| Pending → Dismissed | Shrink + fade | 200ms | ease-in |
| Hover (desktop) | Gentle float + glow | continuous | ease-in-out |
| Drag (reorder) | Lift shadow + scale 1.05 | 150ms | ease-out |
| Delete | Disintegrate particles | 400ms | ease-in |

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
        <X size={32} />
      </motion.button>
    </motion.div>
  )}
</AnimatePresence>
```

### 6.5 Frame Preview Gallery

After moments are confirmed and frames extracted:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   Your Moments (5)                                    [Upload All] [Export] │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                                                                     │  │
│   │    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐   │  │
│   │    │          │    │          │    │          │    │          │   │  │
│   │    │   🖼️     │    │   🖼️     │    │   🖼️     │    │   🖼️     │   │  │
│   │    │          │    │          │    │          │    │          │   │  │
│   │    │  0:15    │    │  0:45    │    │  5:20    │    │  32:10   │   │  │
│   │    │  ✓ Ready │    │  ✓ Ready │    │ ↻ Upload │    │  ✓ Ready │   │  │
│   │    │          │    │          │    │          │    │          │   │  │
│   │    └──────────┘    └──────────┘    └──────────┘    └──────────┘   │  │
│   │         ↕              ↕               ↕               ↕          │  │
│   │    Drag to reorder                                                │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Interactions:**
- **Tap:** Expand to full preview + transcript context
- **Long-press:** Enter edit mode (adjust timestamp ±2s)
- **Swipe left:** Delete (with undo)
- **Drag:** Reorder (affects final artifact order)
- **Pinch:** Zoom into frame details

---

## 7. Moment Detection Engine (3-Tier System)

### 7.1 Architecture Overview

**Decision:** Three distinct detection modes, progressively more intelligent, with clear user control.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        MOMENT DETECTION ENGINE                              │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                                                                     │  │
│   │   Mode Selection (User Controlled)                                  │  │
│   │                                                                     │  │
│   │   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐              │  │
│   │   │   📏        │   │   ☁️        │   │   🖥️        │              │  │
│   │   │   RULES     │   │   CLOUD     │   │   LOCAL     │              │  │
│   │   │   ONLY      │   │   (Gemini)  │   │   (Ollama)  │              │  │
│   │   │             │   │             │   │             │              │  │
│   │   │  Default    │   │  Requires   │   │  Admin      │              │  │
│   │   │  Always on  │   │  Google     │   │  granted    │              │  │
│   │   │             │   │  OAuth      │   │  only       │              │  │
│   │   └─────────────┘   └─────────────┘   └─────────────┘              │  │
│   │         │                 │                 │                       │  │
│   │         ▼                 ▼                 ▼                       │  │
│   │   ┌─────────────────────────────────────────────────────────────┐  │  │
│   │   │              Combined Results → Deduplicated                 │  │  │
│   │   └─────────────────────────────────────────────────────────────┘  │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
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
    transcript_snippet: str  # ±15 words around match
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

| Metric | Limit | Our Usage Pattern |
|--------|-------|-------------------|
| Requests/minute | 60 | ~1-3 per video |
| Requests/day | 1,500 | More than enough |
| Tokens/minute | 1,000,000 | Transcripts are small |

**Implementation Flow:**

```
┌──────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   User clicks "Enhance with AI"                                          │
│                           │                                              │
│                           ▼                                              │
│   ┌────────────────────────────────────────────────────────────────┐    │
│   │  Has valid Google OAuth token with Gemini scope?                │    │
│   └────────────────────────────────────────────────────────────────┘    │
│                           │                                              │
│              ┌────────────┴────────────┐                                │
│              │                         │                                │
│              ▼ No                      ▼ Yes                            │
│   ┌─────────────────────┐   ┌─────────────────────────────────────┐    │
│   │ Trigger OAuth flow  │   │ Call Gemini API with user's token   │    │
│   │ (passkey + Google)  │   │                                     │    │
│   └─────────────────────┘   └─────────────────────────────────────┘    │
│                                         │                                │
│                                         ▼                                │
│                           ┌─────────────────────────────────────────┐   │
│                           │ Parse response → Visual moments         │   │
│                           └─────────────────────────────────────────┘   │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

**API Call (via Google AI SDK):**

```python
import google.generativeai as genai

async def analyze_with_gemini(
    transcript: Transcript,
    user_access_token: str
) -> List[LLMMoment]:
    """
    Call Gemini using user's OAuth token.
    """
    # Configure with user's credentials
    genai.configure(credentials=user_access_token)
    
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
GEMINI_PROMPT = """You are analyzing a video transcript to identify moments where visual context is essential for understanding.

A "visual moment" is when:
1. The speaker explicitly references something visible ("as you can see", "look at this")
2. On-screen content (code, diagrams, text) is being discussed
3. A physical demonstration is happening
4. Visual comparison or transition occurs
5. Context would be significantly lost without seeing the video

Analyze this transcript segment and identify visual moments.

IMPORTANT:
- Be selective. Not every timestamp is a visual moment.
- Prefer moments where the visual adds essential context, not just "nice to have"
- If the speaker says "um, you know, like, here" that's probably not a strong visual moment
- Technical tutorials have more visual moments than talking-head discussions

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
┌─────────────────────────────────────────────────────────────────┐
│  AI Enhancement                                     Connected ✓ │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  Using: Gemini 1.5 Flash (via your Google account)             │
│                                                                 │
│  Today's usage: 12 / 1,500 requests                            │
│  ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ 0.8%               │
│                                                                 │
│  [Disconnect Google Account]                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 7.4 Tier 3: Local LLM (Admin-Granted Feature)

**Decision:** Self-hosted Ollama with small model, enabled per-user via admin feature flag.

**Why Feature-Flagged?**
- Local LLM consumes server resources (RAM, CPU)
- Not everyone needs it (Cloud tier is usually sufficient)
- Allows you to grant access to trusted users/testers
- Keeps base experience snappy for casual users

**Model Selection (Reaffirmed):**

| Model | RAM | Speed | Quality | Notes |
|-------|-----|-------|---------|-------|
| **Qwen2.5-1.5B-Instruct** | ~2GB | Fast | Good | ✅ Primary choice |
| Phi-3.5-mini | ~3GB | Medium | Better | Fallback if quality issues |
| Gemma-2-2B | ~2.5GB | Fast | Good | Alternative |

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
    admin_user: str = Depends(get_tailscale_user)  # From Tailscale headers
):
    await update_feature_flag(user_id, "local_llm_enabled", enabled)
    await log_admin_action(admin_user, f"Set local_llm={enabled} for {user_id}")
    return {"status": "updated"}
```

**UI Indication:**

```
┌─────────────────────────────────────────────────────────────────┐
│  Moment Detection Mode                                          │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  ○ Rules Only (fast, no AI)                                     │
│                                                                 │
│  ○ Rules + Cloud AI (requires Google account)                   │
│                                                                 │
│  ◉ Rules + Local AI ✨                                          │
│    ↳ Enabled by admin. Runs on our server.                      │
│    ↳ May be slower during high usage.                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
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
            <Ruler className="w-4 h-4" />
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
            <Cloud className="w-4 h-4" />
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
            <Cpu className="w-4 h-4" />
            <span>Rules + Local AI</span>
            {features?.localLlmEnabled && (
              <Badge variant="outline">✨ Enabled</Badge>
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
    
    # Tier 2: Cloud LLM (if enabled and authorized)
    if mode in [DetectionMode.CLOUD, DetectionMode.LOCAL]:
        if mode == DetectionMode.CLOUD and user?.google_token:
            llm_moments = await detect_with_gemini(transcript, user.google_token)
            all_moments.extend(llm_moments)
        elif mode == DetectionMode.LOCAL and user?.features.local_llm_enabled:
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

## 8. Export Artifact Format

### 8.1 Design for AI Consumption

**Decision:** Primary format is **Markdown with base64-embedded images**.

**Why Base64?**
- **Self-contained:** No external dependencies, works offline
- **LLM-compatible:** Claude, GPT-4V, and Gemini all accept base64 images in API calls
- **Portable:** Single file, easy to share/archive
- **No link rot:** External image hosts can go down; base64 is forever

**Format Specification:**

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

| Timestamp | Description | Type |
|-----------|-------------|------|
| 00:15 | Code editor showing HTML structure | code |
| 00:45 | Flexbox diagram explaining axis distribution | diagram |
| 02:30 | Browser DevTools demonstration | ui |

---

*Generated by [Tool Name] • AI-consumable transcript with visual context*
```

### 8.2 Alternative Formats

**JSON (For Programmatic Use):**

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
    },
    ...
  ],
  "visual_moments": [
    {
      "timestamp": 15.0,
      "description": "Code editor showing HTML structure",
      "type": "code",
      "image_base64": "/9j/4AAQSkZJRg...",
      "image_mime": "image/jpeg"
    },
    ...
  ],
  "metadata": {
    "extracted_at": "2024-12-14T10:30:00Z",
    "tool_version": "1.0.0",
    "detection_mode": "cloud"
  }
}
```

**Plain Text (Fallback):**

For contexts where images aren't needed, or as a lightweight preview:

```
VIDEO: [Title]
URL: https://youtube.com/watch?v=VIDEO_ID

TRANSCRIPT:
[00:00] Welcome to today's tutorial...
[00:15] As you can see here... [📷 VISUAL MOMENT: Code editor]
[00:32] The CSS flexbox model...
[00:45] Let me show you this diagram... [📷 VISUAL MOMENT: Diagram]
...

VISUAL MOMENTS:
- 00:15: Code editor showing HTML structure
- 00:45: Flexbox diagram
- 02:30: Browser DevTools demonstration
```

---

## 9. Video Segment Strategy

*(Unchanged from v1, included for completeness)*

### 9.1 Segment-Based Downloads

**Decision:** Download video in 30-second segments on-demand, not complete files.

**Quality Selection:**

| Resolution | Bitrate | 30s Size | Default |
|------------|---------|----------|---------|
| 360p | 500 kbps | ~2 MB | ✅ Yes |
| 480p | 1000 kbps | ~4 MB | On request |
| 240p | 300 kbps | ~1 MB | Fallback |

### 9.2 Cache Architecture with Session Locking

```
/var/cache/yttool/
├── segments/
│   └── {video_id}_{start}_{end}.mp4
├── frames/
│   └── {video_id}_{timestamp}.jpg
├── metadata/
│   └── {video_id}.json
└── locks/
    └── {session_id}.lock
```

**Lock Behavior:**
- Active sessions lock their segments against eviction
- Session timeout: 2 minutes without heartbeat
- Graceful degradation: re-download if segment somehow evicted

---

## 10. Admin Dashboard

### 10.1 Access Method

**URL:** `https://yourdomain.com:8443/admin` (Tailscale only)

### 10.2 Dashboard Sections

**Overview:**
```
┌─────────────────────────────────────────────────────────────────────────────┐
│  ADMIN DASHBOARD                                        [Your Tailscale ID] │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  System Health                                                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐             │
│  │  CPU: 23%       │  │  RAM: 1.8/8 GB  │  │  Cache: 4.2 GB  │             │
│  │  ████░░░░░░     │  │  ██████░░░░     │  │  ████████░░     │             │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘             │
│                                                                             │
│  Active Sessions: 3        Queue Depth: 2        Gemini Calls Today: 47    │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Recent Activity                                                            │
│  ─────────────────────────────────────────────────────────────────────────  │
│  • 2 min ago   user_abc    Extracted video (tutorial, 12 moments)          │
│  • 5 min ago   anon_xyz    Exported artifact (rules only)                  │
│  • 12 min ago  user_abc    Connected Google account                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**User Management:**
```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Users                                               [Search] [Export CSV]  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ID          │ Type      │ Google │ Local LLM │ Videos │ Last Active       │
│  ────────────┼───────────┼────────┼───────────┼────────┼─────────────────  │
│  user_abc    │ Identified│ ✓      │ ✓         │ 23     │ 2 min ago         │
│  user_def    │ Identified│ ✓      │ ○         │ 8      │ 1 hour ago        │
│  anon_xyz    │ Anonymous │ ○      │ ○         │ 2      │ 5 min ago         │
│                                                                             │
│  [Select user for details / feature management]                             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Feature Flag Toggle:**
```
┌─────────────────────────────────────────────────────────────────────────────┐
│  User: user_abc                                                 [← Back]    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Account Details                                                            │
│  ─────────────────────────────────────────────────────────────────────────  │
│  Created: 2024-12-01                                                        │
│  Email: user@example.com (via Hanko)                                        │
│  Google: Connected (gemini scope)                                           │
│                                                                             │
│  Feature Flags                                                              │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  Local LLM Access     [████████████] ON                                     │
│                       Granted: 2024-12-10 by admin                          │
│                                                                             │
│  Rate Limit Override  [░░░░░░░░░░░░] OFF                                    │
│                       Default limits apply                                  │
│                                                                             │
│  Beta Features        [░░░░░░░░░░░░] OFF                                    │
│                       No beta features enabled                              │
│                                                                             │
│  [Save Changes]                                                             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 11. Cool Factor: Visual Polish

*(Enhanced from v1)*

### 11.1 Framework Stack

```
shadcn/ui (foundation)
    ↓
Framer Motion (micro-interactions)
    ↓
Aceternity UI / Magic UI (hero effects)
    ↓
React Three Fiber (background shaders) — optional
```

### 11.2 Key "Wow" Moments

| Screen | Effect | Library |
|--------|--------|---------|
| Landing hero | Aurora gradient + text reveal | Aceternity |
| Transcript load | Staggered line fade-in | Framer Motion |
| Moment picker mode | Spotlight cursor + UI expansion | Custom + Framer |
| Moment confirmation | Particle burst + pulse | Framer + Custom |
| Frame gallery | Masonry + physics drag | Framer Reorder |
| Export ready | Confetti + button materialize | Custom |
| Background | Subtle gradient shift | CSS or Three.js |

### 11.3 Mobile-Specific Polish

| Interaction | Desktop | Mobile |
|-------------|---------|--------|
| Moment select | Click + hover glow | Long-press + haptic + ripple |
| Delete | Small × button | Swipe left + red reveal |
| Reorder | Drag handle | Drag with lift shadow |
| Settings | Dropdown | Bottom sheet slide-up |
| Loading | Spinner | Skeleton shimmer |

---

## 12. Implementation Phases (Revised)

### Phase 1: Foundation (Week 1-2)
- [ ] FastAPI + SQLite setup
- [ ] yt-dlp transcript extraction
- [ ] Segment download with quality selection
- [ ] Basic cache manager (no locking yet)
- [ ] CLI for testing

### Phase 2: Auth Foundation (Week 2-3)
- [ ] Hanko setup (self-hosted)
- [ ] Anonymous session system
- [ ] Session upgrade flow
- [ ] Tailscale admin route protection

### Phase 3: Frontend Foundation (Week 3-4)
- [ ] Next.js + shadcn setup
- [ ] Mobile-first responsive layout
- [ ] URL input + metadata display
- [ ] Virtualized transcript viewer

### Phase 4: Moments Universe - Core (Week 4-5)
- [ ] Custom segment player
- [ ] Timeline with segment visualization
- [ ] Manual moment selection
- [ ] Touch-friendly interactions

### Phase 5: Moments Universe - Polish (Week 5-6)
- [ ] Picker mode (full-screen, spotlight)
- [ ] Frame extraction + gallery
- [ ] Animations + micro-interactions
- [ ] Mobile gesture refinements

### Phase 6: Detection Engine (Week 6-7)
- [ ] Rule-based detection
- [ ] Google OAuth integration
- [ ] Gemini API integration
- [ ] Usage tracking UI

### Phase 7: Local LLM + Admin (Week 7-8)
- [ ] Ollama setup
- [ ] Feature flag system
- [ ] Admin dashboard (Tailscale)
- [ ] User management UI

### Phase 8: Export + Final Polish (Week 8-9)
- [ ] Markdown artifact generation
- [ ] Base64 image embedding
- [ ] Export download flow
- [ ] Cool factor effects

### Phase 9: Testing + Hardening (Week 9-10)
- [ ] Edge case handling
- [ ] Error states + recovery
- [ ] Performance optimization
- [ ] Mobile device testing

---

## 13. Open Questions (Updated)

1. ~~Authentication~~ → Resolved: Hanko + Google OAuth + Tailscale admin
2. ~~Mobile support~~ → Resolved: Core principle with fat-finger focus
3. ~~Image format~~ → Resolved: Base64 for AI compatibility
4. **Branding:** What should this tool be called?
5. **Monetization:** Future consideration for premium features?
6. **Multi-language:** Priority for non-English transcripts?
7. **Collaboration:** Multiple users working on same video?

---

## 14. Revision History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2024-12-14 | Initial draft |
| 2.0 | 2024-12-14 | Added: Auth architecture, mobile UX, 3-tier LLM, Tailscale admin, elevated Moments Universe |

