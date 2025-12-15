# Architecture Decision Record: YT Vision Transcript

**Status:** Draft  
**Date:** 2024-12-14  
**Author:** Danilo + Claude  

---

## 1. Context & Problem Statement

We want to build a web application that transforms YouTube videos into AI-consumable text artifacts. The core insight is that transcripts alone lose critical context—diagrams, code on screen, physical demonstrations, facial expressions during sarcasm—and an AI agent reading the transcript would miss or misinterpret these moments.

**The tool must:**
1. Extract titles, descriptions, and timestamped transcripts from YouTube
2. Identify "visual moments" where the video itself contains essential context
3. Allow human review/adjustment of these moments via comfortable UI
4. Extract frames from selected moments and host them as images
5. Output a single text artifact with embedded image URLs

**Constraints:**
- Must run on Hostinger KVM 4 (4 vCPU, 8GB RAM, 100GB SSD) alongside other services
- Should not be a resource hog—"just one more thing" philosophy
- Must handle concurrent users gracefully without manual babysitting

---

## 2. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         FRONTEND (Next.js + UI Library)                     │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌──────────────────┐  ┌─────────────┐  ┌───────────────┐  │
│  │ URL Input   │→ │ Transcript View  │→ │ Moment      │→ │ Export View   │  │
│  │ + Metadata  │  │ + Auto-Suggested │  │ Picker +    │  │ + Download    │  │
│  │ Preview     │  │   Moments        │  │ Mini Player │  │               │  │
│  └─────────────┘  └──────────────────┘  └─────────────┘  └───────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      │ REST + WebSocket (progress events)
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         BACKEND (FastAPI + Python)                          │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────────┐  │
│  │ Extraction      │  │ Video Cache     │  │ Task Queue                  │  │
│  │ Service         │  │ Manager         │  │ (Concurrency Control)       │  │
│  │ (yt-dlp)        │  │ (Segments)      │  │                             │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────────┘  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────────┐  │
│  │ Frame           │  │ Moment          │  │ Export                      │  │
│  │ Extractor       │  │ Suggester       │  │ Compiler                    │  │
│  │ (ffmpeg)        │  │ (LLM or Rules)  │  │                             │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              STORAGE LAYER                                  │
├──────────────────────┬─────────────────────┬────────────────────────────────┤
│  Session Store       │  Video Segment      │  Frame Cache                   │
│  (Redis or SQLite)   │  Cache (Disk)       │  (Disk + Image Host)           │
└──────────────────────┴─────────────────────┴────────────────────────────────┘
```

---

## 3. Decision Record

### 3.1 Video Caching Strategy

#### Decision: Segmented Download with Session Locking

**Context:**  
Downloading full videos (even at 360p) wastes bandwidth and disk for moments the user never scrubs to. A 20-minute video at 360p is ~50-150MB. With multiple concurrent sessions, this compounds quickly on our KVM4.

**Approach: HLS/DASH Segment Exploitation**

YouTube serves videos as segmented streams (typically 2-10 second chunks). We can exploit this:

```python
# Pseudocode for segment-aware downloading
class SegmentCache:
    def __init__(self, video_id: str, session_id: str):
        self.video_id = video_id
        self.session_id = session_id
        self.segments: dict[int, Path] = {}  # segment_index -> file path
        self.lock = asyncio.Lock()
    
    async def ensure_segment(self, timestamp_seconds: float) -> Path:
        segment_index = int(timestamp_seconds // SEGMENT_DURATION)
        
        async with self.lock:
            if segment_index not in self.segments:
                # Download just this segment via yt-dlp with --download-sections
                path = await download_segment(self.video_id, segment_index)
                self.segments[segment_index] = path
        
        return self.segments[segment_index]
```

**yt-dlp supports this natively:**
```bash
# Download only seconds 120-130
yt-dlp --download-sections "*120-130" -f "worst[height>=360]" URL
```

**Quality Decision: 360p baseline, 480p ceiling**

| Resolution | Typical Bitrate | 10-sec segment | Rationale |
|------------|-----------------|----------------|-----------|
| 360p       | ~500 kbps       | ~625 KB        | Sufficient for "what's on screen" recognition |
| 480p       | ~1000 kbps      | ~1.25 MB       | Upper bound if 360p unavailable |

360p is enough to distinguish "there's code on screen" vs "speaker is gesturing." We're not trying to *read* the code from the frame—we're creating a visual anchor for the AI.

**Cache Structure:**
```
/var/cache/yt-vision/
├── segments/
│   ├── {video_id}/
│   │   ├── manifest.json          # Segment metadata
│   │   ├── seg_000.mp4            # Segments 0-10s
│   │   ├── seg_001.mp4            # Segments 10-20s
│   │   └── ...
├── frames/
│   ├── {session_id}/
│   │   ├── frame_00_42.jpg        # Frame at 42 seconds
│   │   └── ...
└── sessions/
    └── {session_id}.json          # Session state + locks
```

#### Cache Rotation Strategy

**Problem:** Disk fills up. Need automatic cleanup that doesn't break active sessions.

**Solution: Reference-Counted LRU with Session Locks**

```python
class CacheManager:
    MAX_CACHE_SIZE_GB = 15  # Leave plenty of headroom on 100GB disk
    
    async def cleanup_if_needed(self):
        current_size = await self.get_cache_size()
        
        if current_size < self.MAX_CACHE_SIZE_GB:
            return
        
        # Get all segments sorted by last_accessed
        segments = await self.get_segments_by_lru()
        
        for segment in segments:
            if current_size < self.MAX_CACHE_SIZE_GB * 0.8:  # Cleanup to 80%
                break
            
            # Skip if ANY active session references this video
            if await self.has_active_session_lock(segment.video_id):
                continue
            
            await self.delete_segment(segment)
            current_size -= segment.size_gb
```

**Session Locking Rules:**
1. Session creation acquires a "soft lock" on the video_id
2. Lock refreshes on any user activity (scrubbing, marking moments)
3. Lock expires after 30 minutes of inactivity
4. Cache cleanup skips any video with active locks
5. If a session tries to access a cleaned segment, re-download transparently

**Failure Scenario Handling:**

| Scenario | Detection | Recovery |
|----------|-----------|----------|
| Segment deleted mid-session | 404 on segment fetch | Re-download segment, brief loading indicator |
| Server restart mid-session | Session state in SQLite survives | Reload session, segments re-download on demand |
| Disk full during download | Catch OSError | Return 503, trigger emergency cleanup, retry |
| Concurrent downloads of same segment | Lock contention | Second request waits for first download |

**User Experience During Recovery:**
```typescript
// Frontend handling
const { data: segment, isLoading, error } = useSegment(timestamp);

// Show inline loading state, not full-page error
if (isLoading) return <SegmentLoadingShimmer />;
if (error) return <SegmentRetrying attempt={retryCount} />;
```

---

### 3.2 Mini Player Architecture

#### Decision: Custom Player with Segment-Aware Buffering

**Why not just embed YouTube's player?**
- YouTube's iframe API doesn't expose raw frames
- Can't extract frames client-side due to CORS
- We need server-side frame extraction anyway

**Player Design:**

```
┌────────────────────────────────────────────────────────────────┐
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                                                          │  │
│  │                    VIDEO VIEWPORT                        │  │
│  │                      (HTML5 <video>)                     │  │
│  │                                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ advancement bar with segment availability viz            │  │
│  │ ░░░░████░░░░░░████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ │  │
│  │     ↑ cached    ↑ cached                                  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
│  [⏮] [⏪ -5s] [▶/⏸] [⏩ +5s] [⏭]     [📍 Mark Moment]        │
└────────────────────────────────────────────────────────────────┘
```

**Segment Visualization:**
- Gray: not loaded
- Blue: loaded in cache
- Green: currently buffered in player
- Gold dots: marked moments (auto-suggested + user-picked)

**Prefetching Strategy:**
```python
async def on_playback_position_change(position: float, direction: str):
    current_segment = get_segment_index(position)
    
    # Prefetch in playback direction
    if direction == "forward":
        prefetch_targets = [current_segment + 1, current_segment + 2]
    else:
        prefetch_targets = [current_segment - 1, current_segment - 2]
    
    # Also prefetch around auto-suggested moments if nearby
    nearby_suggestions = get_suggestions_within(position, window=60)
    prefetch_targets.extend([get_segment_index(s.timestamp) for s in nearby_suggestions])
    
    # Limit concurrent prefetches
    await prefetch_with_limit(prefetch_targets, max_concurrent=2)
```

---

### 3.3 Automatic Moment Suggestion

#### Decision: Hybrid Approach (Rules + Optional LLM)

**The Honest Assessment of Local LLM Feasibility:**

| Model | VRAM/RAM | Inference Time (500 tokens) | Quality | Verdict |
|-------|----------|-----------------------------|---------|---------| 
| Phi-3 Mini (3.8B) | ~4GB | ~2-4s on CPU | Good | Feasible but slow |
| Llama 3.2 3B | ~4GB | ~3-5s on CPU | Good | Similar |
| Qwen2.5 1.5B | ~2GB | ~1-2s on CPU | Decent | Most practical |
| TinyLlama 1.1B | ~1.5GB | <1s on CPU | Limited | May miss nuance |

**The Problem:** On KVM4 with 8GB RAM (shared with other services), running even a 3B model while handling HTTP requests, video processing, and other services is risky. CPU inference is slow enough that concurrent users would queue badly.

**My Recommendation: Rules-First, Cloud-LLM-Optional**

```python
class MomentSuggester:
    def __init__(self, llm_client: Optional[AsyncAnthropic] = None):
        self.llm_client = llm_client  # Claude API, optional
    
    async def suggest_moments(self, transcript: List[TranscriptSegment]) -> List[Suggestion]:
        # Phase 1: Rule-based detection (always runs, fast)
        suggestions = []
        suggestions.extend(self.detect_deictic_references(transcript))
        suggestions.extend(self.detect_visual_cue_phrases(transcript))
        suggestions.extend(self.detect_demonstration_language(transcript))
        
        # Phase 2: LLM refinement (optional, cloud-based)
        if self.llm_client and len(transcript) < 1000:  # Token limit guard
            suggestions = await self.refine_with_llm(transcript, suggestions)
        
        return suggestions
    
    def detect_deictic_references(self, transcript):
        """Catch phrases that point to visual content"""
        patterns = [
            r"\b(this|that|these|those)\s+(diagram|chart|graph|screen|code|slide|image|picture|example)\b",
            r"\b(as you can see|look at|notice|shown here|over here|right here)\b",
            r"\b(let me show|I'll demonstrate|watch this|see how)\b",
            r"\b(on the (left|right)|at the (top|bottom)|in the corner)\b",
        ]
        # ... implementation
    
    def detect_demonstration_language(self, transcript):
        """Catch physical/visual demonstrations"""
        patterns = [
            r"\b(like this|like so|this way|watch)\b",
            r"\b(I'm (clicking|typing|dragging|selecting|highlighting))\b",
            r"\b(now I|here I|let me)\s+(click|type|drag|open|close|scroll)\b",
        ]
        # ... implementation
```

**Why This Hybrid Works:**
1. Rule-based catches 70-80% of obvious visual moments
2. It's instant—no latency, no resource contention
3. Cloud LLM (Claude Haiku is cheap and fast) handles nuance when available
4. Graceful degradation: works offline, just less refined

**If You REALLY Want Local LLM:**

Use `llama.cpp` in server mode with request queuing:

```bash
# Start llama.cpp server with limited concurrency
./server -m qwen2.5-1.5b-q4_k_m.gguf \
    --ctx-size 2048 \
    --parallel 1 \          # Only 1 concurrent inference
    --threads 2 \           # Leave CPU for other tasks
    --port 8081
```

Then in Python:
```python
class LocalLLMClient:
    def __init__(self):
        self.semaphore = asyncio.Semaphore(1)  # Enforce single inference
        self.base_url = "http://localhost:8081"
    
    async def complete(self, prompt: str, timeout: float = 30.0):
        async with self.semaphore:  # Queue requests
            async with httpx.AsyncClient(timeout=timeout) as client:
                response = await client.post(
                    f"{self.base_url}/completion",
                    json={"prompt": prompt, "n_predict": 200}
                )
                return response.json()["content"]
```

---

### 3.4 Concurrency & Process Management

#### Decision: In-Process Async + Semaphores (No External Queue)

**Why not Celery/Redis?**
- Additional services to run and monitor
- Overkill for our concurrency needs
- More moving parts = more failure modes

**Our concurrency concerns are narrow:**
1. Limit concurrent yt-dlp downloads (network + disk I/O)
2. Limit concurrent ffmpeg processes (CPU)
3. Limit concurrent LLM inferences (if local)

**Solution: Named Semaphores in FastAPI**

```python
# app/concurrency.py
from contextlib import asynccontextmanager
import asyncio

class ResourcePool:
    """Manages concurrency limits for expensive operations"""
    
    def __init__(self):
        self.ytdlp_semaphore = asyncio.Semaphore(2)      # Max 2 concurrent downloads
        self.ffmpeg_semaphore = asyncio.Semaphore(3)     # Max 3 concurrent ffmpeg
        self.llm_semaphore = asyncio.Semaphore(1)        # Max 1 LLM inference
        self.upload_semaphore = asyncio.Semaphore(4)     # Max 4 concurrent uploads
    
    @asynccontextmanager
    async def ytdlp(self):
        async with self.ytdlp_semaphore:
            yield
    
    @asynccontextmanager
    async def ffmpeg(self):
        async with self.ffmpeg_semaphore:
            yield
    
    # ... etc

# Global instance
resources = ResourcePool()

# Usage in endpoint
@app.post("/api/extract-segment")
async def extract_segment(video_id: str, start: float, end: float):
    async with resources.ytdlp():
        return await download_segment(video_id, start, end)
```

**Queue Visibility for Users:**

```python
@app.get("/api/queue-status")
async def queue_status():
    return {
        "downloads": {
            "active": 2 - resources.ytdlp_semaphore._value,
            "limit": 2,
            "waiting": len(resources.ytdlp_semaphore._waiters)
        },
        # ... other resources
    }
```

Frontend can show: "2 downloads active, you're #3 in queue" with smooth UX.

---

### 3.5 UI Framework Decision

#### Decision: shadcn/ui + Framer Motion + Custom Video Components

**shadcn/ui Assessment for Our Needs:**

| Component Need | shadcn Has It? | Quality | Notes |
|----------------|----------------|---------|-------|
| Buttons, inputs, forms | ✅ | Excellent | Core strength |
| Cards, dialogs, popovers | ✅ | Excellent | Works great for transcript cards |
| Slider (for timeline) | ✅ | Good | Will need customization for segment viz |
| Tabs (for workflow steps) | ✅ | Excellent | Perfect for our multi-step flow |
| Toast notifications | ✅ | Good | For upload success, errors |
| Video player | ❌ | N/A | Must build custom |
| Timeline with markers | ❌ | N/A | Must build custom |
| Frame gallery/picker | ❌ | N/A | Must build custom |

**Verdict:** shadcn provides ~60% of what we need out of the box, and its composition model (you own the code) makes customization clean.

**Extensibility Model:**

shadcn isn't a package—it copies component source code into your project. This means:
```
src/components/
├── ui/                    # shadcn components (you own these)
│   ├── button.tsx
│   ├── slider.tsx         # We'll extend this
│   └── ...
├── video/                 # Our custom components
│   ├── segment-player.tsx
│   ├── timeline.tsx
│   └── moment-marker.tsx
```

You can directly modify shadcn components or compose them into larger custom components. This is excellent for our needs.

---

### 3.6 The Cool Factor Section

#### Let's Have Fun: Visual Polish & Delight

**Tier 1: Essential Polish (Low Effort, High Impact)**

**Framer Motion for Micro-interactions:**
```tsx
import { motion, AnimatePresence } from "framer-motion";

// Moment markers that pop satisfyingly
<motion.button
  whileHover={{ scale: 1.1 }}
  whileTap={{ scale: 0.95 }}
  initial={{ scale: 0, opacity: 0 }}
  animate={{ scale: 1, opacity: 1 }}
  exit={{ scale: 0, opacity: 0 }}
  className="moment-marker"
  onClick={() => markMoment(timestamp)}
>
  📍
</motion.button>

// Transcript segments that slide in as video plays
<motion.div
  initial={{ x: -20, opacity: 0 }}
  animate={{ x: 0, opacity: 1 }}
  transition={{ type: "spring", stiffness: 300 }}
>
  <TranscriptSegment text={segment.text} />
</motion.div>
```

**Tier 2: Distinctive UI Libraries**

**Option A: Aceternity UI** (https://ui.aceternity.com/)
- Built on shadcn primitives
- Adds effects like: spotlight cards, 3D card hover, text shimmer, aurora backgrounds
- Copy-paste components, same model as shadcn
- *Recommendation: Use sparingly for hero sections and key interactions*

```tsx
// Example: Spotlight effect following cursor on main input area
import { Spotlight } from "@/components/ui/spotlight";

<div className="relative">
  <Spotlight className="-top-40 left-0 md:left-60" fill="blue" />
  <URLInputCard />
</div>
```

**Option B: Magic UI** (https://magicui.design/)
- Similar vibe to Aceternity
- Standout: animated borders, particle effects, text animations
- Good for: Loading states, success celebrations

**Tier 3: Going Wild (High Effort, Maximum Cool)**

**Idea 1: Waveform Visualization for Audio**
```tsx
// Extract audio levels from video, visualize as waveform behind timeline
// Shows where speech is happening vs silence
import { useAudioAnalyzer } from "@/hooks/useAudioAnalyzer";

const Timeline = ({ segments }) => {
  const waveform = useAudioAnalyzer(audioBuffer);
  
  return (
    <div className="relative">
      {/* Waveform background */}
      <svg className="absolute inset-0 opacity-20">
        <path d={waveform.path} fill="currentColor" />
      </svg>
      
      {/* Segment bars on top */}
      {segments.map(seg => <SegmentBar key={seg.id} segment={seg} />)}
    </div>
  );
};
```

**Idea 2: "Scrub Preview" Thumbnails**
Like YouTube's thumbnail preview on hover, but cooler:

```tsx
// As user hovers over timeline, show frame preview with glass morphism
<motion.div
  className="absolute -top-24 backdrop-blur-md bg-white/10 rounded-lg p-2"
  style={{ left: hoverX }}
  initial={{ opacity: 0, y: 10 }}
  animate={{ opacity: 1, y: 0 }}
>
  <img src={framePreviewUrl} className="rounded" />
  <span className="text-xs">{formatTime(hoverTimestamp)}</span>
</motion.div>
```

**Idea 3: "Moment Constellation" View**
Alternative to linear timeline—show all moments as a spatial visualization:

```tsx
// Use React Flow or custom canvas
// Each moment is a node, connected by transcript flow
// User can zoom, pan, click to jump

import ReactFlow, { Background, Controls } from "reactflow";

const MomentConstellation = ({ moments }) => {
  const nodes = moments.map((m, i) => ({
    id: m.id,
    position: polarToCartesian(i, moments.length), // Arrange in circle
    data: { moment: m },
    type: "momentNode",
  }));
  
  return (
    <ReactFlow nodes={nodes} nodeTypes={nodeTypes}>
      <Background variant="dots" gap={12} size={1} />
      <Controls />
    </ReactFlow>
  );
};
```

**Idea 4: Shader Background (WebGL)**
Subtle, animated gradient background that responds to activity:

```tsx
// Using @react-three/fiber for a simple shader plane
import { Canvas } from "@react-three/fiber";

const AnimatedBackground = () => (
  <Canvas className="fixed inset-0 -z-10">
    <mesh>
      <planeGeometry args={[10, 10]} />
      <shaderMaterial
        fragmentShader={gradientShader}
        uniforms={{
          time: { value: 0 },
          activity: { value: 0.5 }, // Increases during processing
        }}
      />
    </mesh>
  </Canvas>
);
```

**Idea 5: Export Preview with "Typewriter" Effect**
When generating final output, show it being "typed" with syntax highlighting:

```tsx
import { motion } from "framer-motion";

const ExportPreview = ({ markdown }) => {
  const characters = markdown.split("");
  
  return (
    <pre className="font-mono">
      {characters.map((char, i) => (
        <motion.span
          key={i}
          initial={{ opacity: 0 }}
          animate={{ opacity: 1 }}
          transition={{ delay: i * 0.01 }} // 10ms per character
        >
          {char}
        </motion.span>
      ))}
    </pre>
  );
};
```

**Idea 6: Sound Design**
Tiny audio feedback for key actions (optional, user can disable):

```tsx
import useSound from "use-sound";

const MomentPicker = () => {
  const [playMark] = useSound("/sounds/mark.mp3", { volume: 0.3 });
  const [playUnmark] = useSound("/sounds/unmark.mp3", { volume: 0.3 });
  
  const handleMarkMoment = () => {
    markMoment(timestamp);
    playMark();
  };
};
```

---

## 4. Data Models

### 4.1 Core Entities

```typescript
// Session: One user working on one video
interface Session {
  id: string;                    // UUID
  videoId: string;               // YouTube video ID
  createdAt: Date;
  lastActivityAt: Date;          // For cache lock refresh
  status: "extracting" | "reviewing" | "exporting" | "complete";
}

// Video metadata + transcript
interface VideoData {
  videoId: string;
  title: string;
  description: string;
  duration: number;              // seconds
  transcript: TranscriptSegment[];
  suggestedMoments: Moment[];    // Auto-detected
}

interface TranscriptSegment {
  start: number;                 // seconds
  duration: number;
  text: string;
}

// A marked moment (auto or manual)
interface Moment {
  id: string;
  timestamp: number;             // seconds
  source: "auto" | "manual";
  reason?: string;               // Why it was flagged (for auto)
  frame?: Frame;                 // Populated after frame extraction
  status: "suggested" | "confirmed" | "rejected";
}

// An extracted frame
interface Frame {
  id: string;
  momentId: string;
  localPath: string;             // Server path to cached frame
  hostedUrl?: string;            // After upload to image host
  width: number;
  height: number;
}

// Cache metadata
interface CachedSegment {
  videoId: string;
  segmentIndex: number;
  startTime: number;
  endTime: number;
  path: string;
  sizeBytes: number;
  createdAt: Date;
  lastAccessedAt: Date;
  lockedBySessions: string[];    // Session IDs with active locks
}
```

### 4.2 API Endpoints

```yaml
# Session Management
POST   /api/sessions                    # Create session for video URL
GET    /api/sessions/{id}               # Get session state
DELETE /api/sessions/{id}               # Cleanup session

# Video & Transcript
GET    /api/sessions/{id}/video         # Get video metadata + transcript
GET    /api/sessions/{id}/suggestions   # Get auto-suggested moments

# Video Playback (Segment Streaming)
GET    /api/sessions/{id}/segments/{index}    # Get video segment (binary)
GET    /api/sessions/{id}/segment-map         # Get which segments are cached

# Moment Management  
POST   /api/sessions/{id}/moments       # Add manual moment
PATCH  /api/sessions/{id}/moments/{mid} # Update moment (confirm/reject)
DELETE /api/sessions/{id}/moments/{mid} # Remove moment

# Frame Extraction
POST   /api/sessions/{id}/moments/{mid}/extract   # Extract frame for moment
GET    /api/sessions/{id}/frames/{fid}            # Get frame image

# Upload & Export
POST   /api/sessions/{id}/upload        # Upload all confirmed frames to host
GET    /api/sessions/{id}/export        # Generate final markdown artifact

# System
GET    /api/health                      # Health check
GET    /api/queue-status                # Current processing queue state
```

---

## 5. Deployment Architecture

```yaml
# docker-compose.yml
version: "3.8"

services:
  app:
    build: .
    ports:
      - "3000:3000"      # Next.js
      - "8000:8000"      # FastAPI
    volumes:
      - cache_data:/var/cache/yt-vision
      - ./sessions.db:/app/sessions.db
    environment:
      - IMGBB_API_KEY=${IMGBB_API_KEY}
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}  # Optional, for LLM refinement
    deploy:
      resources:
        limits:
          memory: 4G     # Leave 4GB for system + other services
          cpus: "2"      # Leave 2 CPUs for other services

volumes:
  cache_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /var/cache/yt-vision
```

**Resource Budget on KVM4:**

| Resource | Total | YT-Vision | Other Services |
|----------|-------|-----------|----------------|
| RAM | 8 GB | 4 GB max | 4 GB |
| CPU | 4 vCPU | 2 vCPU avg | 2 vCPU |
| Disk | 100 GB | 15 GB cache | 85 GB |

---

## 6. Implementation Phases

### Phase 1: Core Pipeline (CLI-First)
**Goal:** End-to-end flow works via command line
- [ ] yt-dlp extraction wrapper
- [ ] Transcript parsing
- [ ] Rule-based moment suggestion
- [ ] ffmpeg frame extraction
- [ ] Markdown export generation
- **Deliverable:** `python main.py "youtube-url"` → outputs markdown file

### Phase 2: API Layer
**Goal:** Pipeline exposed as REST API
- [ ] FastAPI scaffolding
- [ ] Session management
- [ ] Segment caching with locks
- [ ] Concurrency controls
- **Deliverable:** Postman/curl can drive full flow

### Phase 3: Basic Frontend
**Goal:** Functional UI without polish
- [ ] Next.js + shadcn setup
- [ ] URL input → transcript view
- [ ] Manual moment marking
- [ ] Export download
- **Deliverable:** Usable web app, ugly but works

### Phase 4: Video Player & Polish
**Goal:** The cool stuff
- [ ] Custom segment-aware player
- [ ] Timeline with segment visualization
- [ ] Moment markers on timeline
- [ ] Frame preview on hover
- [ ] Framer Motion animations
- **Deliverable:** Feels good to use

### Phase 5: Advanced Features
**Goal:** Nice-to-haves
- [ ] Cloud LLM integration for better suggestions
- [ ] Waveform visualization
- [ ] Sound design
- [ ] Constellation view (experimental)
- **Deliverable:** Wow factor

---

## 7. Open Questions & Risks

| Question | Impact | Current Thinking |
|----------|--------|------------------|
| Will YouTube block segment downloads? | High | yt-dlp handles this well, but may need rotation strategies |
| imgbb rate limits? | Medium | Free tier is 32k uploads/day, should be fine |
| How accurate are auto-suggestions? | Medium | Start with rules, measure, iterate |
| Mobile experience? | Low | Desktop-first, responsive as bonus |

---

## 8. Appendix: Tool & Library Versions

```
# Backend
python = "^3.11"
fastapi = "^0.109"
yt-dlp = "^2024.1"
ffmpeg-python = "^0.2"
httpx = "^0.26"
sqlmodel = "^0.0.14"  # SQLite ORM

# Frontend
next = "^14.1"
react = "^18.2"
typescript = "^5.3"
tailwindcss = "^3.4"
framer-motion = "^11.0"
# shadcn/ui components installed via CLI

# Optional
llama-cpp-python = "^0.2"  # Only if local LLM
anthropic = "^0.18"        # Only if cloud LLM
```

---

*This ADR is a living document. Update as decisions evolve.*
