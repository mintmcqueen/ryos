# Infrastructure Documentation Gaps Analysis

**Comparison Date**: 2025-10-30
**Source Documents**:
- `/Users/jb/Desktop/DEV-PROJECTS/ryos/dev/ANALYSIS/05_infrastructure_analysis.md` (Comprehensive Infrastructure Analysis)
- `/Users/jb/Desktop/DEV-PROJECTS/ryos/CLAUDE.md` (Current Developer Documentation)

---

## EXECUTIVE SUMMARY

**CLAUDE.md Coverage**: ~35% of infrastructure layer documented
**Critical Gaps**: 14 hooks undocumented, API endpoint schemas missing, no build config details
**Priority**: HIGH - Infrastructure layer is foundation for all apps, critical for onboarding

---

## I. HOOKS DOCUMENTATION GAPS (14 Undocumented Hooks)

### A. Completely Missing Hooks (10)

**1. `useAudioRecorder.ts`** - NOT DOCUMENTED
- **What it does**: MediaRecorder wrapper with device selection
- **Critical params**: `onRecordingComplete`, `selectedDeviceId`, `setRecordingState`
- **Returns**: `{ isRecording, micPermissionGranted, startRecording, stopRecording }`
- **Why critical**: Used by Soundboard app, permission handling pattern
- **CLAUDE.md Status**: No mention

**2. `useSoundboard.ts`** - NOT DOCUMENTED
- **What it does**: Zustand wrapper for soundboard CRUD
- **Returns**: `{ boards, activeBoard, playbackStates, setActiveBoardId, addNewBoard, updateBoardName, deleteCurrentBoard }`
- **Why critical**: Direct interface to `useSoundboardStore`
- **CLAUDE.md Status**: No mention (store listed line 919, hook missing)

**3. `useSwipeNavigation.ts`** - NOT DOCUMENTED
- **What it does**: Swipe gesture detection with visual feedback
- **Critical params**: `threshold` (100px), `onSwipeLeft`, `onSwipeRight`, `currentAppId`, `isActive`
- **Returns**: `{ handleTouchStart, handleTouchMove, handleTouchEnd, isSwiping, swipeDirection }`
- **Why critical**: Mobile gesture navigation
- **CLAUDE.md Status**: No mention

**4. `useLongPress.ts`** - NOT DOCUMENTED
- **What it does**: Generic long-press handler for touch devices
- **Signature**: `useLongPress<T>(onLongPress, { delay = 500 })`
- **Returns**: `{ onTouchStart, onTouchEnd, onTouchMove, onTouchCancel }`
- **Why critical**: Pattern for all long-press interactions
- **CLAUDE.md Status**: No mention

**5. `useWallpaper.ts`** - NOT DOCUMENTED
- **What it does**: Zustand wrapper for wallpaper selection
- **Returns**: `{ currentWallpaper, wallpaperSource, setWallpaper, isVideoWallpaper, loadCustomWallpapers }`
- **Advanced**: Stale blob detection, video wallpaper detection
- **Why critical**: Desktop rendering depends on this
- **CLAUDE.md Status**: No mention

**6. `useLaunchApp.ts`** - NOT DOCUMENTED
- **What it does**: App launch orchestration with special handling
- **Special logic**: Applet Viewer reuse, Finder path conversion, multi-window enforcement
- **Returns**: `launchApp(appId, options?) → instanceId`
- **Why critical**: Centralized launch logic, multi-window rules
- **CLAUDE.md Status**: No mention

**7. `useVibration.ts`** - NOT DOCUMENTED
- **What it does**: Debounced vibration with iOS polyfill
- **Signature**: `useVibration(debounceMs = 200, vibrationMs = 50)`
- **Dependency**: `ios-vibrator-pro-max` npm package
- **Why critical**: Haptic feedback pattern for all touch interactions
- **CLAUDE.md Status**: No mention

**8. `useToast.ts`** - NOT DOCUMENTED (Legacy)
- **What it does**: Re-export of `sonner` toast
- **Note**: Kept for backward compatibility, direct import preferred
- **Why mention**: Developers might search for it
- **CLAUDE.md Status**: No mention

**9. `useAuth.ts`** - PARTIALLY DOCUMENTED
- **CLAUDE.md**: Listed as "Chat authentication, room joining" (line 271)
- **Missing Details**:
  - Full return type (19 properties)
  - Authentication modes (password, token, create user)
  - Password setting flow
  - Token verification process
  - Logout confirmation dialog
  - Data cleanup on user switch

**10. `useIsPhone.ts`** - NOT DOCUMENTED
- **What it does**: Stricter mobile detection (touch AND small screen)
- **Logic**: `(ontouchstart in window || maxTouchPoints > 0) && width < 640px`
- **Difference from `useIsMobile`**: AND vs OR logic
- **Why critical**: Different breakpoint, different behavior
- **CLAUDE.md Status**: No mention

### B. Underdocumented Hooks (4)

**1. `useWindowManager.ts`** - BASIC DOCUMENTED
- **CLAUDE.md**: "Window lifecycle, focus handling, keyboard shortcuts" (line 256-259)
- **Missing Details**:
  - Full signature: `useWindowManager({ appId, instanceId? })`
  - Returns 10 properties (only 5 mentioned)
  - Theme-aware inset calculation (critical for safe areas)
  - Mobile constraints (full-width, vertical-only drag)
  - 8-direction resize handles (`ne`, `nw`, `se`, `sw` directions)
  - Sound effects (looping move/resize sounds)
  - `maximizeWindowHeight(maxHeightConstraint?)` function
  - `getSafeAreaBottomInset()` iOS detection
  - **Advanced insets logic**:
    ```typescript
    {
      menuBarHeight: system7 ? 30 : macosx ? 25 : 0,
      taskbarHeight: xp/win98 ? 30 : 0,
      dockHeight: macosx ? 56 : 0,
      topInset: menuBarHeight,
      bottomInset: taskbarHeight + dockHeight + safeAreaBottom
    }
    ```

**2. `useSound.ts`** - BASIC DOCUMENTED
- **CLAUDE.md**: "Sound effect playback with volume control" (line 262)
- **Missing Details**:
  - Shared AudioContext pattern
  - Global cache (`Map<string, AudioBuffer>`)
  - Active source tracking (`Set<AudioBufferSourceNode>`)
  - Concurrency limit (max 32 sources)
  - Cache invalidation (detects new AudioContext)
  - Lazy preloading (first interaction)
  - **17 predefined sounds** (only hinted at)
  - Returns: `{ play, stop, fadeOut, fadeIn }`

**3. `useChatSynth.ts`** - BASIC DOCUMENTED
- **CLAUDE.md**: "Chat typing synth effects" (line 264)
- **Missing Details**:
  - Preset system (5 presets: classic, ethereal, digital, retro, off)
  - Global singleton pattern (HMR persistence via `globalSynthRef`)
  - Effect chain: Synth → Reverb → Tremolo → Filter → Destination
  - Voice limiting (16 max polyphony, min 90ms between notes)
  - Note pool: Pentatonic scale `["C4", "D4", "F4", "G4", "A4", "C5", "D5"]`
  - iOS handling (closed AudioContext recreation)
  - Volume management (reactive to `chatSynthVolume` + `masterVolume`)
  - Returns: `{ playNote, currentPreset, changePreset, isAudioReady }`

**4. `useTerminalSounds.ts`** - BASIC DOCUMENTED
- **CLAUDE.md**: "Terminal keystroke sounds" (line 265)
- **Missing Details**:
  - Elevator music system (Brian Eno-inspired ambient)
  - **Time Modes**: "past", "now" (ambient), "future" (space warp)
  - **Generative layers** (melody sequencer, pad synths, bell synth for past/now)
  - **Simplified future architecture** (lead synth, single pad, 6-12s between events)
  - Randomized timing (800ms-3.3s notes, 4-12s pads)
  - **Complete architecture**:
    - Past/Now: Reverb (10s decay) → PingPongDelay (0.7s, 50% feedback) → Filter (2kHz)
    - Future: Reverb (5s decay) → PingPongDelay (16n, 40% feedback) → Filter (2.5kHz)
  - Note pools (Db major pentatonic for past, C major for future)

### C. Missing Hook Usage Patterns

**Not documented in CLAUDE.md**:
1. **Hook lifecycle patterns** (mount → setup → cleanup)
2. **Event listener cleanup** (all hooks clean up on unmount)
3. **Global window event registration** (8 hooks register global listeners)
4. **Shared AudioContext pattern** (5 hooks use singleton context)
5. **Store integration patterns** (11 hooks consume Zustand stores)

---

## II. LIBRARY DOCUMENTATION GAPS (3 Libraries)

### A. Completely Missing Libraries (2)

**1. `lib/webglFilterRunner.ts`** - PARTIALLY DOCUMENTED
- **CLAUDE.md**: "WebGL shader pipeline" mentioned (line 290-298) but only for desktop wallpaper
- **Missing Details**:
  - **Two exports**: `mapCssFilterStringToUniforms()`, `runFilter()`
  - **16 supported effects**: brightness, contrast, hue, saturate, grayscale, sepia, invert, bulge, pinch, twist, fisheye, stretch, squeeze, tunnel, kaleidoscope, ripple, glitch
  - **CSS filter mapping**: `"brightness(1.2)" → { brightness: 1.2 }`
  - **WebGL pipeline**: Off-screen canvas → shader compilation → texture upload → uniform setting → quad drawing
  - **Performance**: ~60fps on modern hardware (vs ~10fps CPU-based)
  - **Usage**: Not just wallpaper, but Photo Booth filters too

**2. `lib/shaders/basicFilter.frag`** - NOT DOCUMENTED
- **What it is**: GLSL fragment shader for all filter effects
- **23 uniforms**: 7 color filters, 9 distortions, 7 new effects
- **Key functions**: `rgb2hsl()`, `hsl2rgb()`, `bulge()`, `pinch()`, `twist()`, `fisheye()`, `kaleidoscope()`, `ripple()`, `glitch()`
- **Optimization**: Conditional application (only if `u_effect != 0.0`)
- **CLAUDE.md Status**: Shader files listed (line 295-298) but only CRT/Galaxy/Aurora shaders mentioned (not basicFilter.frag)

### B. Underdocumented Libraries (1)

**1. `lib/audioContext.ts`** - BASIC DOCUMENTED
- **CLAUDE.md**: "Singleton AudioContext for all audio" (line 281-283)
- **Missing Details**:
  - **iOS quirks handled**: Closed state, interrupted state, auto-resume
  - **Global listeners**: `visibilitychange`, `focus` events
  - **Two exports**: `getAudioContext()`, `resumeAudioContext()`
  - **Recreation logic**: If closed, creates new context
  - **Usage pattern**: `const ctx = getAudioContext(); await resumeAudioContext();`

---

## III. UTILITIES DOCUMENTATION GAPS (13 Utilities)

### A. Documented Utilities (2)

**1. `utils/displayMode.ts`** - DOCUMENTED (line 301)
**2. `utils/bootMessage.ts`** - DOCUMENTED (line 302)

### B. Partially Documented Utilities (3)

**1. `utils/indexedDB.ts`** - BASIC DOCUMENTED (line 304)
- **CLAUDE.md**: "IndexedDB initialization for virtual FS + custom wallpapers"
- **Missing Details**:
  - **Database**: `ryOS` (version 7)
  - **5 object stores**: DOCUMENTS, IMAGES, TRASH, CUSTOM_WALLPAPERS, APPLETS
  - **Migration logic** (v5): Recreates stores without keyPath (use UUID keys directly)
  - **Export**: `ensureIndexedDBInitialized(): Promise<IDBDatabase>`

**2. `utils/performanceCheck.ts`** - BASIC DOCUMENTED (line 305)
- **CLAUDE.md**: "Detect device capabilities (shader support)"
- **Missing Details**: Actual detection logic, return type, usage pattern

**3. `utils/sharedUrl.ts`** - BASIC DOCUMENTED (line 303)
- **CLAUDE.md**: "Extract share codes from URLs"
- **Missing Details**: Parsing logic, URL patterns, return type

### C. Undocumented Utilities (8)

**1. `utils/audio.ts`** - NOT DOCUMENTED
- **5 exports**:
  1. `createWaveform(container, base64Data): Promise<WaveSurfer>` - Lazy WaveSurfer load
  2. `createAudioFromBase64(base64Data): HTMLAudioElement` - Data URL conversion
  3. `getSupportedMimeType(): string` - Safari: audio/mp4, Others: audio/webm
  4. `base64FromBlob(blob): Promise<string>` - Async blob → base64
  5. `bufferToBase64(buffer): string` - Sync buffer → base64
- **Why critical**: Used by Soundboard, Audio Recorder, Audio Transcription

**2-8. Other Missing Utilities**:
- `utils/chat.ts` - Message parsing/formatting (assumed)
- `utils/device.ts` - User-agent detection (assumed)
- `utils/icons.ts` - Icon/emoji helpers (assumed)
- `utils/lrcParser.ts` - Lyrics timestamp parsing (assumed)
- `utils/markdown/index.ts` - Markdown → HTML conversion (assumed)
- `utils/markdown/saveUtils.ts` - File export helpers (assumed)
- `utils/wallpapers.ts` - Wallpaper fetching/caching (assumed)

---

## IV. TYPE DEFINITIONS DOCUMENTATION GAPS (4 Files)

### A. Documented Types (1)

**1. `types/types.ts`** - BASIC DOCUMENTED (lines 940-945 mention SoundSlot, Soundboard, WindowPosition)
- **Missing**: `DialogState`, `ResizeType`, `ResizeStart` not mentioned

### B. Undocumented Types (3)

**1. `types/chat.ts`** - NOT DOCUMENTED
- **4 types**: `MessageMetadata`, `AIChatMessage`, `ChatMessage`, `ChatRoom`, `User`
- **Critical**: `ChatMessage.clientId` for optimistic rendering
- **Critical**: `ChatRoom.type` ("public" | "private"), `members` array
- **Why critical**: Pusher integration, chat room access control

**2. `types/lyrics.ts`** - NOT DOCUMENTED (assumed `LyricLine` interface)

**3. `types/aiModels.ts`** - NOT DOCUMENTED (assumed AI model config types)

---

## V. API ENDPOINT DOCUMENTATION GAPS (10 Endpoints)

### A. Documented Endpoints (3)

**1. `/api/chat.ts`** - DOCUMENTED (lines 956-958)
**2. `/api/audio-transcribe.ts`** - DOCUMENTED (line 958)
**3. `/api/lyrics.ts`** - DOCUMENTED (line 959)

### B. Partially Documented Endpoints (2)

**1. `/api/speech.ts`** - BASIC DOCUMENTED (line 960)
- **CLAUDE.md**: "Text-to-speech (OpenAI/ElevenLabs)"
- **Missing Details**:
  - **Request schema** (12 fields)
  - **Authentication**: Same token validation as `/api/chat`
  - **Rate limiting**: Burst (10/min), Daily (50/day), ryo user bypassed
  - **ElevenLabs integration** (full API call structure)
  - **Default models/voices**: ElevenLabs default, OpenAI fallback
  - **Streaming response**: ArrayBuffer → ReadableStream

**2. `/api/chat.ts`** - BASIC DOCUMENTED (line 956)
- **CLAUDE.md**: "AI chat endpoint (Anthropic/OpenAI/Google)"
- **Missing Details**:
  - **Request schema**: `{ messages, systemState?, model? }`
  - **Authentication**: Header-based multi-token support, grace period (365 days)
  - **Rate limiting**: 50 messages / 5 hours (auth + anon)
  - **System prompt structure**: Static cacheable + dynamic per-request
  - **14 tools exposed** (full list with schemas)
  - **CORS restrictions**: Only `https://os.ryo.lu` and `http://localhost:3000`
  - **Geolocation**: Inferred via Vercel Edge
  - **Streaming**: `smoothStream` for Chinese character streaming

### C. Undocumented Endpoints (8)

**1. `/api/translate-lyrics.ts`** - NOT DOCUMENTED
- **Purpose**: Translate lyrics to target language
- **Input**: `{ lines: LyricLine[], targetLanguage: string }`
- **Return**: LRC string (translated)
- **Why critical**: Used by `useLyrics` hook conditionally

**2. `/api/parse-title.ts`** - NOT DOCUMENTED
- **Purpose**: Extract title/artist from YouTube video
- **Why critical**: iPod app YouTube integration

**3. `/api/ie-generate.ts`** - NOT DOCUMENTED
- **Purpose**: Generate retro HTML for Internet Explorer time-travel
- **Why critical**: Internet Explorer app core functionality

**4. `/api/iframe-check.ts`** - NOT DOCUMENTED
- **Purpose**: Check if URL is embeddable (CORS/X-Frame-Options)
- **Why critical**: Internet Explorer app safety check

**5. `/api/link-preview.ts`** - NOT DOCUMENTED
- **Purpose**: Fetch Open Graph metadata for URL previews
- **Why critical**: Internet Explorer app link previews

**6. `/api/utils/aiModels.ts`** - NOT DOCUMENTED
- **Expected exports**: `SupportedModel` type, `DEFAULT_MODEL`, `getModelInstance()`
- **Why critical**: Model registry for all AI endpoints

**7. `/api/utils/aiPrompts.ts`** - NOT DOCUMENTED
- **Expected exports**: 7 prompt templates (CORE_PRIORITY_INSTRUCTIONS, RYO_PERSONA_INSTRUCTIONS, etc.)
- **Why critical**: System prompt assembly for `/api/chat`

**8. `/api/utils/rate-limit.d.ts`** - NOT DOCUMENTED
- **Purpose**: TypeScript declarations for rate limiting
- **Implementation**: Redis-backed counters (Upstash)
- **Why critical**: Security layer for all endpoints

---

## VI. BUILD CONFIGURATION DOCUMENTATION GAPS (3 Configs)

### A. Documented Build Configs (1)

**1. `vite.config.ts`** - PARTIALLY DOCUMENTED (lines 674-679)
- **CLAUDE.md**: "Manual chunk splitting, minification, PWA support"
- **Missing Details**:
  - **Framework**: React + Vite 6
  - **Plugins**: 3 total (`@vitejs/plugin-react`, `@tailwindcss/vite`, `vite-plugin-vercel`)
  - **Path aliases**: `@/*` → `./src/*`
  - **Dev server config**: Port 5173, CORS wildcard, watch ignore `.terminals/**`
  - **Manual chunks**: `{ react, ui, audio }` (specific packages listed)
  - **Vercel config**: `defaultSupportsResponseStreaming: true`

### B. Undocumented Build Configs (2)

**1. `tailwind.config.js`** - NOT DOCUMENTED
- **Mode**: `darkMode: ["class"]`
- **Plugins**: 3 total (`tailwindcss-animate`, `@tailwindcss/typography`, custom pixelated images)
- **Theme extensions**:
  - Custom colors: OS theme variables (`--os-color-*`)
  - Custom border radius: `os: var(--os-metrics-radius)`
  - Custom heights: `os-titlebar`, `os-menubar`
  - Custom fonts: `os-ui`, `os-mono`
  - Custom shadows: `os-window`
  - Custom animations: shake, marquee, shimmer
  - Typography customization (reduced margins, list styles)
- **Why critical**: All theme-based styling depends on this config

**2. `tsconfig.json`** - NOT DOCUMENTED
- **Base config**: References `tsconfig.app.json`, `tsconfig.node.json`
- **Path aliases**: `@/*` → `./src/*`
- **JSX**: `react-jsx` (new transform)
- **Skip Lib Check**: `true` (faster compilation)
- **Why critical**: TypeScript resolution for all imports

---

## VII. DATA FLOW DOCUMENTATION GAPS (5 Flows)

### A. Documented Flows (4)

**1. App Launch Flow** - DOCUMENTED (lines 21-30)
**2. Window Focus Flow** - DOCUMENTED (lines 33-40)
**3. AI Chat Flow** - DOCUMENTED (lines 43-50)
**4. Virtual Filesystem Flow** - DOCUMENTED (lines 53-59)

### B. Missing Flows (5)

**1. Audio Pipeline (Shared AudioContext)** - NOT DOCUMENTED
- **Singleton pattern**: All audio hooks use same context
- **Benefits**: Resource efficiency, iOS compatibility, auto-resume
- **Drawback**: If one hook corrupts context, all audio breaks
- **5 hooks sharing**: `useSound`, `useTtsQueue`, `useAudioTranscription`, `useChatSynth`, `useTerminalSounds`

**2. WebGL Shader Rendering Pipeline** - NOT DOCUMENTED
- **Flow**: `runFilter(source, uniforms, fragmentShader)`
  1. Create offscreen canvas
  2. Compile shaders → link program
  3. Upload source as texture (Y-flip for video)
  4. Set uniforms
  5. Draw fullscreen quad → fragment shader per pixel
  6. Return canvas with rendered result
- **Performance**: GPU-bound, ~60fps for 720p video

**3. TTS Flow** - NOT DOCUMENTED
- **Flow**: `useTtsQueue.speak(text)` → queuedFetch → `/api/speech` → OpenAI/ElevenLabs
  → ArrayBuffer → decodeAudioData → BufferSource → schedule at nextStartRef (gap-free)
- **Why critical**: Zero-latency audio streaming pattern

**4. Transcription Flow** - NOT DOCUMENTED
- **Flow**: `useAudioTranscription` → silence detection → MediaRecorder.stop() → blob
  → FormData → `/api/audio-transcribe` → OpenAI Whisper → `{ text }`
  → onTranscriptionComplete(text)
- **Why critical**: Voice Activity Detection (VAD) pattern

**5. Lyrics Flow** - NOT DOCUMENTED
- **Flow**: `useLyrics` → `/api/lyrics` → lrclib.net API → parseLRC → LyricLine[]
  → (if translateTo) → `/api/translate-lyrics` → AI translation → parseLRC
  → currentTime → calculateCurrentLine → setCurrentLine
- **Why critical**: Multi-step data transformation pattern

---

## VIII. DEPENDENCY MAPPING GAPS (4 Maps)

### A. Documented Dependencies (1)

**1. Store Dependencies** - PARTIALLY DOCUMENTED (lines 312-327 list stores)
- **Missing**: Hook → Store mapping (11 hooks consume Zustand stores)

### B. Missing Dependency Maps (4)

**1. Hook Dependencies on Stores** - NOT DOCUMENTED
- **11 hooks** consume Zustand stores (table needed)
- **Examples**: `useAuth` → `useChatsStore`, `useChatSynth` → `useAppStore`

**2. Hook Dependencies on Other Hooks** - NOT DOCUMENTED
- **2 dependencies**: `useChatSynth` → `useVibration`, `useWindowManager` → `useSound` (4 instances)

**3. Utility Function Usage** - NOT DOCUMENTED
- **Most common**: `utils/audio` (3 hooks), `lib/audioContext` (2 hooks), `lib/utils` (`cn` used everywhere)

**4. API Endpoint Consumers** - NOT DOCUMENTED
- **Table needed**: Endpoint → Frontend consumers
- **Example**: `/api/chat` → AI Chat component (via `useChat` hook)

---

## IX. SPECIAL FOCUS ITEMS GAPS (2 Areas)

### A. Complete Hook Inventory

**CLAUDE.md Status**: 19 hooks listed (lines 253-276, 886-898)
**Missing Details**:
- Full signatures for all hooks
- Return types for all hooks
- Parameter descriptions for all hooks
- Side effects documentation for all hooks

**Gap**: Only 5 hooks have detailed documentation (useWindowManager, useSound, useAuth, useLyrics, useMediaQuery), 14 missing details

### B. Complete API Endpoint Inventory

**CLAUDE.md Status**: 9 endpoints listed (lines 956-968)
**Missing Details**:
- Request/response schemas for all endpoints
- Authentication requirements for all endpoints
- Rate limiting details for all endpoints
- External service integrations for all endpoints

**Gap**: Only 3 endpoints have basic documentation (chat, audio-transcribe, lyrics), 8 missing entirely, 2 missing details

---

## X. CRITICAL FINDINGS GAP ANALYSIS

### A. Strengths Not Documented in CLAUDE.md

**Missing from CLAUDE.md**:
1. **Comprehensive hook coverage** - 19 well-scoped hooks (only 5 documented)
2. **Singleton pattern mastery** - AudioContext, Pusher, global synth refs (not explained)
3. **iOS Safari resilience** - Explicit closed/interrupted audio context handling (not documented)
4. **GPU acceleration** - WebGL shaders for 60fps video filtering (basic mention only)
5. **API security layers** - Multi-token auth, grace period, rate limiting details (missing)
6. **Streaming optimization** - Anthropic prompt caching, smooth Chinese streaming (not explained)

### B. Potential Issues Not Documented

**Missing from CLAUDE.md**:
1. **Global state fragmentation** - Audio hooks use both stores AND refs (consistency risk)
2. **Error boundary gaps** - Many hooks lack try-catch in async operations
3. **Rate limit bypass** - `ryo` user bypasses all limits (security concern)
4. **CORS hardcoding** - Only 2 allowed origins (deployment flexibility issue)
5. **No retry logic** - Most API calls fail silently without exponential backoff
6. **Memory leaks possible** - WaveSurfer instances not always destroyed
7. **IndexedDB v7** - Migration path assumes clean upgrade (no rollback strategy)

---

## XI. PRIORITY GAPS FOR DOCUMENTATION UPDATE

### Priority 1: CRITICAL (Blocks Developer Onboarding)

**1. Complete Hook Inventory** (14 undocumented hooks)
- Add all 19 hooks with full signatures, returns, side effects
- Estimated effort: 4-6 hours

**2. API Endpoint Schemas** (8 undocumented endpoints + 2 partial)
- Document all 13+ endpoints with request/response schemas
- Estimated effort: 3-4 hours

**3. Data Flow Diagrams** (5 missing flows)
- Audio Pipeline, WebGL Shader Pipeline, TTS Flow, Transcription Flow, Lyrics Flow
- Estimated effort: 2-3 hours

**4. Dependency Maps** (4 missing maps)
- Hook → Store, Hook → Hook, Utility Usage, API Consumers
- Estimated effort: 2 hours

### Priority 2: HIGH (Enhances Understanding)

**5. Build Configuration Details** (2 undocumented configs)
- `tailwind.config.js`, `tsconfig.json` with full theme extensions
- Estimated effort: 1-2 hours

**6. Library Details** (2 missing libraries + 1 partial)
- `lib/webglFilterRunner.ts`, `lib/shaders/basicFilter.frag`, enhance `lib/audioContext.ts`
- Estimated effort: 1-2 hours

**7. Type Definitions** (3 undocumented type files)
- `types/chat.ts`, `types/lyrics.ts`, `types/aiModels.ts`
- Estimated effort: 1 hour

**8. Utility Functions** (8 undocumented utilities)
- `utils/audio.ts` (critical), plus 7 others
- Estimated effort: 2 hours

### Priority 3: MEDIUM (Improves Completeness)

**9. Hook Usage Patterns** (5 missing patterns)
- Hook lifecycle, event listener cleanup, global event registration, shared AudioContext, store integration
- Estimated effort: 1 hour

**10. Critical Findings** (6 strengths + 7 issues)
- Document architectural strengths and potential vulnerabilities
- Estimated effort: 1 hour

---

## XII. RECOMMENDED ADDITIONS TO CLAUDE.MD

### A. New Sections Needed

**1. "Infrastructure Layer - Complete Inventory"**
- **Hooks**: 19 hooks with full signatures (lines 253-276 expand)
- **Libraries**: 5 libraries with full APIs (lines 278-309 expand)
- **Utilities**: 15 utilities with full signatures (new section)
- **Types**: 6 type files with all interfaces (new section)
- **API Endpoints**: 13+ endpoints with full schemas (lines 956-968 expand)
- **Build Configs**: 3 configs with full options (lines 674-679 expand)

**2. "Data Flow - Infrastructure Patterns"**
- **Audio Pipeline**: Shared AudioContext singleton pattern (new)
- **WebGL Shader Pipeline**: GPU-accelerated filter rendering (new)
- **TTS Flow**: Gap-free audio streaming (new)
- **Transcription Flow**: VAD + Whisper integration (new)
- **Lyrics Flow**: Multi-step data transformation (new)

**3. "Dependency Maps - Infrastructure Layer"**
- **Hook Dependencies**: Hook → Store, Hook → Hook tables (new)
- **Utility Usage**: Most common utilities by hook (new)
- **API Consumers**: Endpoint → Frontend component map (new)
- **Type Usage**: Interface → Component usage (new)

**4. "Critical Findings - Infrastructure Analysis"**
- **Strengths**: 6 architectural achievements (new)
- **Potential Issues**: 7 areas for improvement (new)
- **Recommendations**: Immediate, long-term, security hardening (new)

### B. Enhancements to Existing Sections

**1. "Critical Data Flow" (lines 19-60)**
- **ADD**: Audio Pipeline flow (shared context)
- **ADD**: WebGL Shader Rendering flow (GPU filters)
- **ADD**: TTS Queue flow (gap-free playback)
- **ADD**: Transcription flow (VAD pattern)
- **ADD**: Lyrics flow (translation pipeline)

**2. "Infrastructure Layer" (lines 251-309)**
- **EXPAND**: Hook list from basic to full signatures
- **EXPAND**: Library details (webglFilterRunner, shaders)
- **ADD**: Complete utility inventory
- **ADD**: Type definitions section
- **ADD**: API endpoint schemas

**3. "Component Interdependency Map" (lines 309-392)**
- **ADD**: Hook dependency hierarchy
- **ADD**: Utility function usage patterns
- **ADD**: API endpoint consumption map

**4. "Critical Debugging Points" (lines 457-530)**
- **ADD**: Hook debugging patterns
- **ADD**: Audio context debugging
- **ADD**: WebGL shader debugging
- **ADD**: API endpoint debugging

---

## XIII. ESTIMATED EFFORT FOR FULL DOCUMENTATION

### Total Effort Breakdown

**Priority 1 (Critical)**: 11-15 hours
**Priority 2 (High)**: 6-8 hours
**Priority 3 (Medium)**: 2 hours
**Total Effort**: 19-25 hours

### Phased Approach

**Phase 1 (Week 1)**: Priority 1 items - Complete hook inventory, API schemas, data flows
**Phase 2 (Week 2)**: Priority 2 items - Build configs, libraries, types, utilities
**Phase 3 (Week 3)**: Priority 3 items - Usage patterns, critical findings

---

## XIV. CONCLUSION

**CLAUDE.md Infrastructure Coverage**: ~35%
**Critical Gaps**: 14 hooks, 8 API endpoints, 5 data flows, 4 dependency maps
**Impact**: HIGH - New developers lack critical infrastructure context
**Recommended Action**: Prioritize Phase 1 (11-15 hours) for immediate developer onboarding improvement

**Next Steps**:
1. Review this gap analysis
2. Approve phased documentation update plan
3. Execute Phase 1 (Priority 1 items)
4. Validate with new developer onboarding test
5. Continue with Phases 2-3 based on feedback

---

**End of Infrastructure Gaps Analysis**
