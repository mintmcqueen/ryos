# ryOS - Developer Documentation

**Version 0.0.0** - Web-based agentic AI desktop operating system

## 🎯 System Overview & Current Status

**Core Purpose**: Full-featured web application emulating classic desktop OS experiences (macOS System 7, Mac OS X Aqua, Windows XP, Windows 98) in browser with AI integration, multi-instance window management, and 18 built-in applications.

**Architecture**: React 18 SPA with Vite build, Zustand state management (17 stores), instance-based window system, Vercel serverless backend for AI/audio processing, IndexedDB virtual filesystem, Pusher real-time messaging, theme-driven UI transformation.

**Current Status**:
- ✅ Complete: 18 built-in apps, 4 themes, instance-based windowing, virtual FS, AI chat/voice
- 🚧 Production-ready, actively maintained
- 📋 Enhancement opportunities: testing infrastructure, E2E validation, deployment automation

---

## 🔄 Critical Data Flow

### **Primary Application Launch Flow**
```
User Action (Dock/Desktop/URL)
  → CustomEvent("launchApp")
  → AppManager.handleAppLaunch (src/apps/base/AppManager.tsx:213-261)
  → useAppStore.launchApp (src/stores/useAppStore.ts:650-678)
  → createAppInstance (src/stores/useAppStore.ts:458-535)
  → instanceOrder + instances updated
  → AppManager renders instance (src/apps/base/AppManager.tsx:283-324)
  → AppComponent mounted with initialData
```

### **Window Focus Management Flow**
```
User Click/Touch on Window
  → onMouseDown/onTouchStart (src/apps/base/AppManager.tsx:295-304)
  → bringInstanceToForeground (src/stores/useAppStore.ts:580-618)
  → instanceOrder reordered (focused → end)
  → z-index recalculated (BASE_Z_INDEX + position in order)
  → CustomEvent("instanceStateChange") dispatched
```

### **AI Chat Flow**
```
User Message → Chats App (src/apps/chats/)
  → Vercel AI SDK useChat hook
  → API: /api/chat.ts (Anthropic/OpenAI/Google)
  → Stream response → useChatsStore
  → Optional: TTS via /api/speech.ts
  → useTtsQueue → Audio playback with word highlighting
```

### **Virtual Filesystem Flow**
```
File Operation (Finder/Terminal/TextEdit)
  → useFilesStore action (src/stores/useFilesStore.ts)
  → In-memory tree structure updated
  → IndexedDB persistence (background)
  → CustomEvent("fileSystemChange") → subscribers notified
```

---

## 🚀 Version History & Key Enhancements

**v0.0.0** - Initial Production Release
- 18 built-in applications with multi-instance support
- 4 switchable themes (System 7, Aqua, XP, Win98)
- Instance-based window manager (src/stores/useAppStore.ts:33-129)
- Virtual filesystem with IndexedDB persistence
- AI integration (chat, voice, TTS, transcription)
- Real-time collaboration via Pusher
- Responsive design (mobile/tablet/desktop)
- PWA support via vite-plugin-pwa

---

## 🏗️ Component Architecture & Dependencies

### **Entry Points**

**`src/main.tsx`** (0.5 KB) - React Bootstrap
- **Purpose**: Hydrates theme from localStorage, mounts React to DOM, initializes Vercel Analytics
- **Dependencies**: React 18, @vercel/analytics
- **Key Flow**: `hydrateTheme()` → `createRoot()` → `<App />` → Analytics tracking

**`src/App.tsx`** (1.5 KB) - Root Component
- **Purpose**: App registry loader, boot screen logic, display mode management
- **Dependencies**: AppManager, useAppStore, appRegistry
- **Key Functions**:
  - `applyDisplayMode()` - CSS filter application (lines 28-29)
  - Boot screen handling (lines 27-57)
  - Toaster positioning with safe-area insets (lines 60-67)
- **State**: displayMode, isFirstBoot, bootScreenMessage

**`src/apps/base/AppManager.tsx`** (10 KB) - Central Orchestrator
- **Purpose**: Instance-based window management, app lifecycle, z-index calculation, URL routing
- **Dependencies**:
  - useAppStore (instance state)
  - useThemeStore (theme-based layout)
  - appRegistry (app configs)
  - AppContext (legacy compatibility)
- **Key Functions**:
  - `getZIndexForInstance(instanceId)` - Calculates z-index from instanceOrder position (lines 76-80)
  - `bringAppToForeground(appId)` - Legacy wrapper for instance focus (lines 83-98)
  - `handleUrlNavigation()` - Processes /internet-explorer/:code, /ipod/:videoId, /app-id paths (lines 108-210)
  - `handleAppLaunch()` - CustomEvent listener for app launches (lines 214-261)
- **State Management**:
  - `instances` - Record<instanceId, AppInstance>
  - `instanceOrder` - string[] (END = foreground/top z-index)
  - `legacyAppStates` - Aggregated for backward compatibility (lines 52-74)
- **Rendering**:
  - MenuBar (XP/Win98 always shown; Mac/System7 only when no foreground) (lines 274-279)
  - Dock (always visible) (line 281)
  - Instance windows with dynamic z-index (lines 283-324)
  - Desktop with app shortcuts (lines 326-333)

---

### **Business Logic Layer**

**`src/config/appRegistry.ts`** (10 KB) - App Configuration Registry
- **Purpose**: Centralized app metadata, window constraints, component references
- **Dependencies**: All 18 app modules, appIds, BaseApp types
- **Structure**: Record<AppId, { ...BaseApp, windowConfig: WindowConstraints }>
- **Key Exports**:
  - `appRegistry` - Main registry object
  - `getAppComponent(appId)` - Returns React component for appId
  - `getWindowConfig(appId)` - Returns window constraints (min/max/default sizes)
- **Window Configs**: Default 730×475, app-specific overrides (Finder 400×300, TextEdit 430×475, etc.)

**`src/config/appIds.ts`** (0.5 KB) - App ID Constants
- **Purpose**: Single source of truth for valid app IDs
- **Exports**: `appIds` array - ['finder', 'textedit', 'paint', ...]

---

### **Data Layer - Zustand Stores (17 Total)**

**`src/stores/useAppStore.ts`** (32 KB) - **CRITICAL** Central App State
- **Purpose**: Instance-based window management, app lifecycle, settings persistence
- **Persistence**: localStorage via zustand/persist (key: "ryos:app-store")
- **Version**: 3 (migration system for breaking changes)
- **Dependencies**: ZERO imports (excellent isolation)
- **Key State** (17 properties):
  - `instances: Record<instanceId, AppInstance>` - All window instances
  - `instanceOrder: string[]` - Z-index ordering (END = foreground)
  - `foregroundInstanceId: string | null` - Currently focused instance
  - `nextInstanceId: number` - Auto-increment for new instances
  - `windowOrder: string[]` - Legacy app-level ordering (deprecated)
  - `apps: Record<appId, AppState>` - Legacy app states (deprecated)
  - Settings: debugMode, shaderEffectEnabled, selectedShaderType, aiModel
  - Audio: terminalSoundsEnabled, uiSoundsEnabled, typingSynthEnabled, speechEnabled, speechVolume, uiVolume, chatSynthVolume, ipodVolume, masterVolume
  - TTS: ttsModel ('openai' | 'elevenlabs'), ttsVoice
  - Wallpaper: currentWallpaper, wallpaperSource
  - UI: synthPreset, displayMode, htmlPreviewSplit, isFirstBoot

- **Instance Management Methods** (6 core + 4 navigation):
  - `createAppInstance(appId, initialData?, title?)` → instanceId (lines 458-535)
    - Generates unique ID from nextInstanceId counter
    - Staggers position: baseX + (existingInstances × 32px), baseY + (existingInstances × 20px)
    - Applies window config from appRegistry
    - Sets foreground, unfocuses all others
    - Dispatches CustomEvent("instanceStateChange")
    - Mobile detection: uses full viewport width if <768px
  - `closeAppInstance(instanceId)` (lines 537-578)
    - Removes from instances + instanceOrder
    - Smart next focus: Prefer same-app instance > last overall
    - Cleans up state, dispatches event
  - `bringInstanceToForeground(instanceId)` (lines 580-618)
    - Reorders instanceOrder (focused → end = highest z-index)
    - Updates isForeground flags for all instances
    - Dispatches CustomEvent("instanceStateChange")
  - `updateInstanceWindowState(instanceId, position, size)` (lines 620-627)
    - Updates window position/size for instance
    - Called by WindowFrame drag/resize handlers
  - `getInstancesByAppId(appId)` → AppInstance[] (lines 628-629)
    - Returns all instances for given appId
  - `getForegroundInstance()` → AppInstance | null (lines 630-633)
    - Returns currently focused instance
  - `navigateToNextInstance(currentId)` (lines 634-641)
    - Cycles focus to next instance in order
  - `navigateToPreviousInstance(currentId)` (lines 642-649)
    - Cycles focus to previous instance in order
  - `launchApp(appId, initialData?, title?, multiWindow?)` → instanceId (lines 650-678)
    - Checks multi-window support (TextEdit, Finder, Applet Viewer always; others optional)
    - Reuses existing instance OR creates new
    - Updates initialData if reusing
    - Returns instanceId for tracking
  - `_debugCheckInstanceIntegrity()` (lines 680-694)
    - Validates instanceOrder consistency
    - Repairs dangling references

- **Legacy App-Level Methods** (7 - backward compatibility):
  - `bringToForeground(appId)` (lines 262-295)
  - `toggleApp(appId, initialData?)` (lines 296-330)
  - `closeApp(appId)` (lines 331-356)
  - `launchOrFocusApp(appId, initialData?)` (lines 357-386)
  - `navigateToNextApp(currentAppId)` (lines 387-395)
  - `navigateToPreviousApp(currentAppId)` (lines 396-403)
  - `clearInitialData(appId)` (lines 405-414)

- **Instance Data Methods** (2):
  - `clearInstanceInitialData(instanceId)` (lines 415-427)
  - `updateInstanceInitialData(instanceId, initialData)` (lines 429-441)

- **Settings Methods** (20):
  - `setDebugMode(enabled)`, `setShaderEffectEnabled(enabled)`, `setSelectedShaderType(type)`
  - `setAiModel(model)`, `setTerminalSoundsEnabled(enabled)`, `setUiSoundsEnabled(enabled)`
  - `setTypingSynthEnabled(enabled)`, `setSpeechEnabled(enabled)`, `setSpeechVolume(vol)`
  - `setTtsModel(model)`, `setTtsVoice(voice)`, `setSynthPreset(preset)`
  - `setDisplayMode(mode)`, `setHtmlPreviewSplit(enabled)`, `setHasBooted()`
  - `setUiVolume(vol)`, `setChatSynthVolume(vol)`, `setIpodVolume(vol)`, `setMasterVolume(vol)`
  - `updateWindowState(appId, position, size)` - Legacy, use updateInstanceWindowState

- **Wallpaper Methods** (4):
  - `setCurrentWallpaper(path)` - Sets wallpaper reference
  - `setWallpaper(path | File)` → Promise<void> (lines 187-207) - Uploads custom wallpaper to IndexedDB
  - `loadCustomWallpapers()` → Promise<string[]> (lines 209-225) - Lists all custom wallpapers from IndexedDB
  - `getWallpaperData(reference)` → Promise<string | null> (lines 227-259) - Resolves IndexedDB reference to blob URL

- **Migration Logic**:
  - v1→v2 (lines 745-748): Added TTS fields (ttsModel, ttsVoice)
  - v2→v3 (lines 750-758): Unified instanceStackOrder + instanceWindowOrder → instanceOrder
  - Uses migrate() function (lines 732-761)

- **Rehydration Logic** (lines 762-834):
  - Cleans stale instanceOrder (filters non-existent instances)
  - Fixes nextInstanceId (ensures > max existing ID)
  - Ensures positions & sizes (applies defaults if missing)
  - Migrates old app states to instances (pre-v3 data)
  - Resets legacy app flags after migration

- **Persistence Strategy**: Explicit field list via partialize() (lines 698-731)
  - Persists: windowOrder, apps, instances (open only), instanceOrder, settings, wallpapers
  - Excludes: Closed instances, computed values, methods

- **Impact Radius**: Used by 95+ components (AppManager, WindowFrame, MenuBar, Dock, Desktop, ALL 18 apps)
  - Breaking changes require version bump + migration function
  - instanceOrder changes break z-index calculation
  - Instance structure changes break AppManager rendering

**`src/stores/useChatsStore.ts`** (62 KB estimate) - Chat Application State
- **Purpose**: Messages, rooms, user auth, AI interaction state
- **Persistence**: localStorage
- **Key State**: messages[], rooms[], currentRoomId, authToken, typing indicators
- **Integration**: Pusher for real-time, Vercel AI SDK for streaming

**`src/stores/useFilesStore.ts`** (27 KB estimate) - Virtual Filesystem
- **Purpose**: File tree structure, CRUD operations, IndexedDB persistence
- **Structure**: Hierarchical tree (folders → files)
- **Methods**: createFile, createFolder, deleteNode, moveNode, renameNode, getNodeByPath
- **Integration**: Used by Finder, Terminal, TextEdit
- **Backup/Restore**: Full tree serialization (Control Panels app)

**`src/stores/useThemeStore.ts`** - Theme Management
- **Purpose**: Current theme selection, wallpaper management
- **Persistence**: localStorage
- **State**: current theme ('system7' | 'aqua' | 'xp' | 'win98'), wallpaper path
- **Impact**: Layout changes (MenuBar vs Taskbar), font loading, icon asset switching

**Additional Stores** (14):
- `useTerminalStore` (137 lines, v1) - Command history, output buffer, file system context, Vim integration
- `useTextEditStore` (279 lines, v2) - Multi-instance pattern with per-instance documents, unsaved content, cursor positions
  - **Critical**: Direct coupling to useAppStore for getForegroundInstance()
- `useIpodStore` (845 lines, v19) - Smart shuffle algorithm (68-line history-based), playlists, lyrics with translation, library
- `usePaintStore` (37 lines, v1) - Minimal canvas state, tool selection, undo/redo stack
- `useSoundboardStore` (269 lines, v1) - Lazy initialization pattern, soundboards, recordings, waveforms, keyboard shortcuts
- `useSynthStore` (244 lines, v1) - Synth parameters, 5 default presets, MIDI state, onRehydrateStorage cleanup
- `usePhotoBoothStore` (56 lines, v1) - Photo gallery, filters, camera state (NOT usePhotosStore)
- `useVideoStore` (236 lines, v8) - Video playlists, aggressive reset migration (fixes ID-based bugs)
- `useInternetExplorerStore` - Browse history, AI-generated snapshots, favorites by year
- `useMinesweeperStore` - Game state, scores, difficulty levels
- `usePcStore` (93 lines, NO VERSION) - ⚠️ Missing version field - emulator state, save states, game library
- `useFinderStore` - Navigation history, view preferences, current path
- `useAppletStore` - Applet paths, per-applet window sizes
- `useControlPanelsStore` - Settings, backup/restore state

**Utilities**:
- `helpers.ts` (99 lines, 4 exports) - Shallow comparison utilities for Zustand selectors
  - Used by 95+ components (prevents unnecessary re-renders)
  - `useAppStoreShallow`, `useThemeStoreShallow`, etc.

---

### **Presentation Layer - Components**

**`src/components/layout/Desktop.tsx`** - Desktop Viewport
- **Purpose**: Wallpaper rendering, desktop icon grid, drag-drop app launching
- **Dependencies**: useAppStore (wallpaper), useThemeStore (theme)
- **Features**: Video/image/pattern wallpapers, custom wallpaper upload, icon positioning

**`src/components/layout/MenuBar.tsx`** - Top Menu (Mac) / Taskbar (Windows)
- **Purpose**: App-specific menus (File/Edit/View), system menus (Apple/About), taskbar items
- **Theme Switching**: Mac themes → top menubar; Windows themes → bottom taskbar
- **Dynamic Menus**: Foreground app determines menu items via helpItems prop
- **Dependencies**: useAppStore (foregroundInstance), useThemeStore

**`src/components/layout/Dock.tsx`** - Application Launcher
- **Purpose**: App icons, launch/focus, running indicators, context menus
- **Theme Variations**: Mac 3D dock, Windows flat taskbar buttons
- **Dependencies**: useAppStore (instances for running indicators), appRegistry

**`src/components/layout/WindowFrame.tsx`** - Window Wrapper
- **Purpose**: Resizable/draggable window chrome, title bar, control buttons, theme styling
- **Props**: title, isOpen, isForeground, onClose, minSize, maxSize, defaultSize, position
- **Features**:
  - Drag to move (title bar)
  - 8-direction resize handles (corners + edges)
  - Min/max size constraints
  - Theme-specific controls (traffic lights vs XP buttons)
  - Mobile-safe touch resizers
- **State Management**: Calls updateInstanceWindowState(instanceId, position, size)
- **Dependencies**: useAppStore (updateInstanceWindowState), useThemeStore

**`src/components/dialogs/`** - Modal Dialogs
- About (system info)
- BootScreen (system restore animation)
- Help (app-specific help items)
- Confirmation dialogs

**`src/components/ui/`** - shadcn/ui Components (30+)
- Built on Radix UI primitives
- Themed with Tailwind CSS
- Examples: Button, Dialog, Dropdown, Select, Slider, Tabs, Tooltip, ScrollArea

---

### **Infrastructure Layer**

**`src/hooks/`** - Custom React Hooks (19 Total)

**Window Management** (1):
- `useWindowManager({ appId, instanceId? })` (lines 1-89) - Window lifecycle, focus handling, keyboard shortcuts
  - Listens to: appStateChange (legacy), instanceStateChange events
  - Handles Cmd+W (close), Cmd+` (next window), Cmd+Shift+` (prev window)
  - Returns: isOpen, isForeground, position, size, initialData, handleClose, updatePosition, updateSize, appConfig, safeAreaInsets
  - **Theme-aware insets**: Calculates safe areas based on current theme (menubar vs taskbar)

**Audio** (7):
- `useSound(soundPath, options)` - Sound effect playback with volume control
  - Integrates with useAppStore (uiVolume, masterVolume)
  - Supports preload, loop, rate options
- `useAudioRecorder({ onRecordingComplete, selectedDeviceId?, setRecordingState? })` - MediaRecorder wrapper
  - Returns: isRecording, micPermissionGranted, startRecording, stopRecording, duration
  - Used by: Soundboard app
- `useChatSynth()` - Chat typing synth effects with Tone.js
  - Returns: synth, playTypingSound, stopTypingSound
- `useTerminalSounds()` - Terminal keystroke sounds with random pitch variation
  - Returns: playKeystrokeSound, playEnterSound, playErrorSound
- `useTtsQueue()` - Text-to-speech queue management with word highlighting
  - Returns: speak, stop, isPlaying, currentWordIndex, queue
  - Integrates with: /api/speech.ts, useAppStore (speechVolume)
- `useAudioTranscription(enabled)` - Speech-to-text via /api/audio-transcribe
  - Voice Activity Detection (VAD)
  - Returns: transcript, isListening, startListening, stopListening
- `useSoundboard()` - Zustand wrapper for soundboard CRUD
  - Returns: boards, activeBoard, playbackStates, setActiveBoardId, addNewBoard, updateBoardName, deleteCurrentBoard

**Content & Media** (2):
- `useLyrics(videoId)` - Fetches/syncs lyrics via /api/lyrics with translation
  - Returns: lyrics, currentLineIndex, translatedLyrics, error
  - Supports: LRC format, multi-language translation, offset adjustment
- `useWallpaper()` - Zustand wrapper for wallpaper selection
  - Returns: currentWallpaper, wallpaperSource, setWallpaper, isVideoWallpaper, loadCustomWallpapers
  - Advanced: Stale blob detection, video wallpaper detection

**Auth & Interaction** (3):
- `useAuth(roomId?)` - Multi-mode chat authentication (19 properties returned)
  - Modes: password, token, create user
  - Returns: user, room, isAuthenticated, login, logout, setPassword, verifyToken, switchUser
  - Data cleanup on user switch
- `useLaunchApp()` - App launch orchestration with special handling
  - Returns: launchApp(appId, options?) → instanceId
  - Special logic: Applet Viewer reuse, Finder path conversion, multi-window enforcement
- `useSwipeNavigation({ threshold?, onSwipeLeft, onSwipeRight, currentAppId, isActive })` - Touch gestures
  - Returns: handleTouchStart, handleTouchMove, handleTouchEnd, isSwiping, swipeDirection
  - Visual feedback during swipe

**Device & UI** (6):
- `useMediaQuery(query)` - CSS media query hook (window.matchMedia wrapper)
- `useIsMobile()` - <768px OR touch device detection
- `useIsPhone()` - <640px AND touch device detection (stricter than useIsMobile)
- `useLongPress<T>(onLongPress, { delay = 500 })` - Generic long-press handler
  - Returns: onTouchStart, onTouchEnd, onTouchMove, onTouchCancel
- `useVibration(debounceMs = 200, vibrationMs = 50)` - Debounced haptic feedback
  - iOS polyfill via `ios-vibrator-pro-max`
  - Returns: vibrate()
- `useToast()` - Re-export of `sonner` toast (legacy compatibility)

**`src/lib/`** - Libraries & Utilities

**`audioContext.ts`** - Web Audio API Context
- **Purpose**: Singleton AudioContext for all audio (synth, soundboard, TTS)
- **Exports**: `getAudioContext()` - Returns shared context (auto-created on demand)
- **Used by**: Synth app, Soundboard app, TTS playback

**`pusherClient.ts`** - Real-time Messaging
- **Purpose**: Pusher client initialization for chat rooms
- **Config**: Loads from environment (VITE_PUSHER_APP_KEY, VITE_PUSHER_CLUSTER)
- **Used by**: Chats app for real-time message delivery

**`webglFilterRunner.ts`** - WebGL Shader Pipeline
- **Purpose**: Applies shaders (CRT, Galaxy, Aurora) to desktop wallpaper
- **Dependencies**: Three.js, custom GLSL shaders (lib/shaders/)
- **Integration**: Control Panels → Appearance → Shader selection

**`shaders/`** - GLSL Shader Files
- crt.frag - CRT scanline effect
- galaxy.frag - Galaxy spiral effect
- aurora.frag - Aurora borealis effect

**`src/utils/`** - Utility Functions
- `displayMode.ts` - Apply color/grayscale/sepia CSS filters
- `bootMessage.ts` - Persist boot messages across page reloads
- `sharedUrl.ts` - Extract share codes from URLs (/internet-explorer/:code)
- `indexedDB.ts` - IndexedDB initialization for virtual FS + custom wallpapers
- `performanceCheck.ts` - Detect device capabilities (shader support)

---

## 🔧 Component Interdependency Map

### **Dependency Hierarchy** (Critical → Supporting)

**Level 1 - Core State (Foundation)**
- `useAppStore` → Instance management, settings, wallpapers
- `useThemeStore` → Theme selection, layout mode
- `useFilesStore` → Virtual filesystem tree

**Level 2 - App Orchestration**
- `AppManager` → Depends on: useAppStore, useThemeStore, appRegistry
- `appRegistry` → Depends on: All 18 app modules, appIds, BaseApp types

**Level 3 - Layout Components**
- `Desktop` → Depends on: useAppStore (wallpaper), useThemeStore
- `MenuBar` → Depends on: useAppStore (foregroundInstance), useThemeStore, appRegistry
- `Dock` → Depends on: useAppStore (instances), appRegistry
- `WindowFrame` → Depends on: useAppStore (updateInstanceWindowState), useThemeStore

**Level 4 - Applications (18 Apps)**
- Each app → Depends on: WindowFrame, app-specific store, shared components (ui/)

**Level 5 - Infrastructure**
- Hooks → Depend on: stores, utils, API endpoints
- Utils → Pure functions (minimal dependencies)
- API Endpoints → Serverless functions (independent)

---

### **Impact Radius Documentation**

**CRITICAL - Changes Require Careful Coordination**:

**`useAppStore` (src/stores/useAppStore.ts)**
- **Used by**: AppManager, WindowFrame, MenuBar, Dock, Desktop, ALL 18 apps
- **Changes affect**: Instance creation, window positioning, z-index calculation, focus management
- **Breaking changes require**:
  - Version bump in CURRENT_APP_STORE_VERSION
  - Migration function in migrate() (lines 732-761)
  - Testing across all apps
  - Update AppManager z-index logic (line 76-80)
- **Example downstream effects**: Changing instanceOrder structure breaks z-index calculation → windows stack incorrectly

**`AppManager` (src/apps/base/AppManager.tsx)**
- **Used by**: App.tsx (root)
- **Changes affect**: All app launching, window rendering, focus management
- **Breaking changes require**:
  - Update CustomEvent payloads if modifying launchApp event structure
  - Update URL routing logic if adding new path patterns
  - Verify legacy appStates aggregation (lines 52-74) if touching instances
- **Example downstream effects**: Changing instance rendering loop → all apps fail to mount

**`appRegistry` (src/config/appRegistry.ts)**
- **Used by**: AppManager, Dock, MenuBar, Desktop, getAppComponent(), getWindowConfig()
- **Changes affect**: App availability, window sizing, component loading
- **Breaking changes require**:
  - Update AppId type if adding/removing apps
  - Update all appRegistry consumers if changing structure
  - Update Dock icon rendering
- **Example downstream effects**: Changing windowConfig structure → WindowFrame crashes on mount

**`useThemeStore` (src/stores/useThemeStore.ts)**
- **Used by**: AppManager, MenuBar, Dock, WindowFrame, Desktop, all apps for theme-specific styling
- **Changes affect**: Layout mode (menubar vs taskbar), font loading, icon paths, button styles
- **Breaking changes require**:
  - Update theme enum if adding new theme
  - Update MenuBar/Taskbar conditional rendering (AppManager.tsx:274-279)
  - Update WindowFrame control button rendering
  - Add theme-specific assets (fonts, icons, wallpapers)
- **Example downstream effects**: Adding new theme → must update MenuBar layout logic, WindowFrame controls, icon manifests

**`useFilesStore` (src/stores/useFilesStore.ts)**
- **Used by**: Finder, Terminal, TextEdit, Control Panels (backup/restore)
- **Changes affect**: File operations, persistence, path resolution
- **Breaking changes require**:
  - Update all file operation call sites (createFile, deleteNode, etc.)
  - Migration for IndexedDB schema changes
  - Update Finder tree rendering
  - Update Terminal file commands (ls, cd, cat, etc.)
- **Example downstream effects**: Changing tree structure → Finder fails to render, Terminal commands break

---

### **Data Flow Dependencies**

**App Launch Chain**:
```
User Action → CustomEvent("launchApp") → AppManager.handleAppLaunch
→ useAppStore.launchApp → createAppInstance → instanceOrder update
→ AppManager re-render → WindowFrame mounted → App component initialized
```

**Window Focus Chain**:
```
User Click → onMouseDown handler → bringInstanceToForeground
→ instanceOrder reordered → z-index recalculated → WindowFrame z-index updated
→ isForeground prop changed → App component receives focus
```

**Theme Change Chain**:
```
Control Panels → setTheme → useThemeStore update → localStorage persist
→ AppManager re-render → MenuBar/Taskbar switch → WindowFrame controls update
→ Font/icon paths change → Desktop wallpaper change
```

**File Operation Chain**:
```
Finder/Terminal action → useFilesStore method → Tree structure update
→ IndexedDB persist → CustomEvent("fileSystemChange") → Finder/Terminal re-render
→ TextEdit document list update
```

---

## 🏛️ Architectural Thinking Protocol

**Before changing ANY component**:
1. **Map affected components** (direct + indirect dependencies)
   - Check "Impact Radius Documentation" section above
   - Search codebase for imports of the component
   - Identify CustomEvent listeners that may depend on structure
2. **Analyze downstream effects**: "What breaks if this changes?"
   - Will existing instances fail to load? (useAppStore changes)
   - Will apps fail to mount? (AppManager changes)
   - Will window positioning break? (WindowFrame/appRegistry changes)
   - Will theme switching fail? (useThemeStore/MenuBar changes)
3. **Update interdependency maps** in this document
   - Component Architecture section (lines 40-240)
   - Interdependency Map section (lines 260-340)
   - Impact Radius section (lines 350-410)
4. **Document impact radius**: "Changes to X affect Y, Z"
   - Add to Impact Radius Documentation with examples
5. **Verify all related documentation** is synchronized
   - Update Data Flow diagrams if routing changes
   - Update Version History with what/why/where
   - Update README.md if user-facing behavior changes

**Red Flags** (Architectural Debt Warnings):
- ⚠️ Adding direct useAppStore access in app components (use AppContext instead)
- ⚠️ Modifying instanceOrder structure without migration
- ⚠️ Changing CustomEvent payloads without updating listeners
- ⚠️ Adding new theme without updating MenuBar/WindowFrame/Desktop
- ⚠️ Bypassing appRegistry for window configs
- ⚠️ Direct localStorage access instead of using stores

---

## 🔍 Critical Debugging Points

### **Key Log Identifiers**

**Instance Management**:
- `[AppManager] Checking path:` - URL routing (AppManager.tsx:110)
- `[AppManager] Detected IE share code:` - Internet Explorer share link (AppManager.tsx:115)
- `[AppManager] Launch event received for ${appId}` - App launch (AppManager.tsx:223)
- `[AppManager] Launched instance ${instanceId} for app ${appId}` - Instance created (AppManager.tsx:236)
- `[AppStore] focus missing instance ${instanceId}` - Invalid focus attempt (useAppStore.ts:583)

**State Management**:
- `[AppStore] Migrating from ${version} to ${CURRENT_APP_STORE_VERSION}` - Version migration (useAppStore.ts:738)
- `initialization_complete` - System startup successful
- `appStateChange` - CustomEvent fired on app state change
- `instanceStateChange` - CustomEvent fired on instance state change

**Filesystem**:
- `fileSystemChange` - CustomEvent fired on file operation
- `clearAllAppStates` - localStorage cleared (Control Panels reset)

### **CustomEvent Type Catalog**

**System Events** (6 types used across 42 dispatches):

**1. `launchApp`** - App launch trigger
```typescript
CustomEvent<{
  appId: AppId;
  initialPath?: string;    // Legacy parameter (deprecated)
  initialData?: unknown;   // App-specific initialization data
}>
```
- **Dispatchers**: URL routing (AppManager.tsx:120,142,161,184), external components, CustomEvent tests
- **Listeners**: AppManager.handleAppLaunch (line 214-261)
- **Purpose**: Trigger app launch from any component without direct dependency
- **Frequency**: High (every app launch)
- **Example**:
  ```typescript
  window.dispatchEvent(new CustomEvent('launchApp', {
    detail: { appId: 'textedit', initialData: { path: '/Documents/notes.txt' } }
  }))
  ```

**2. `updateApp`** - Update app with new data without remounting
```typescript
CustomEvent<{
  appId: AppId;
  initialData: unknown;
}>
```
- **Dispatcher**: AppManager (line 250) when launching already-open app with new data
- **Listeners**: Individual app components (opt-in)
- **Purpose**: Update app state without full remount (performance optimization)
- **Use Case**: User opens same app with different file while already open
- **Frequency**: Medium (multi-window scenarios)

**3. `instanceStateChange`** - Instance lifecycle notification
```typescript
CustomEvent<{
  instanceId: string;
  isOpen: boolean;
  isForeground: boolean;
}>
```
- **Dispatchers**: useAppStore (createAppInstance line 524, closeAppInstance line 568, bringInstanceToForeground line 604)
- **Listeners**: WindowFrame, MenuBar, Dock (for active indicators)
- **Purpose**: React to instance lifecycle changes (open, close, focus)
- **Frequency**: Very high (every window interaction)

**4. `appStateChange`** - Legacy app-level state (Deprecated)
```typescript
CustomEvent<{
  appId: AppId;
  isOpen: boolean;
  isForeground: boolean;
  updatedData?: boolean;
}>
```
- **Status**: Maintained for backward compatibility during instance migration
- **Dispatchers**: useAppStore legacy methods (bringToForeground, toggleApp, closeApp)
- **Migration Path**: Replace with instanceStateChange listeners
- **Deprecation Timeline**: Phase out after v4.0

**5. `fileSystemChange`** - File system operation notification
```typescript
CustomEvent<void>
```
- **Dispatchers**: useFilesStore (all CRUD operations: createFile, createFolder, deleteNode, moveNode, renameNode)
- **Listeners**: Finder (tree refresh), Terminal (file list update), TextEdit (document list)
- **Purpose**: Notify components to refresh file-based views
- **Frequency**: Medium (file operations)

**6. `wallpaperChange`** - Wallpaper update notification
```typescript
CustomEvent<{ detail: string }>  // wallpaper path
```
- **Dispatcher**: useAppStore.setWallpaper (line 204)
- **Listeners**: Desktop component (background re-render)
- **Purpose**: Trigger wallpaper re-render when changed
- **Frequency**: Low (user changes wallpaper)

---

### **Common Issues & Solutions**

**Issue**: Window fails to appear after launch
**Diagnosis**:
- Check console for `[AppManager] Launched instance` log
- Verify instanceId in useAppStore.instances
- Check instanceOrder includes instanceId
- Verify appRegistry has valid component for appId
**Fix**:
- Ensure launchApp returns valid instanceId
- Check appRegistry.ts for missing app entry
- Verify WindowFrame is rendering (check z-index calculation)

**Issue**: Windows stack incorrectly (z-index wrong)
**Diagnosis**:
- Check instanceOrder array order (END should be foreground)
- Verify getZIndexForInstance calculation (AppManager.tsx:76-80)
- Check isForeground flags in instances
**Fix**:
- Call bringInstanceToForeground to fix order
- Verify BASE_Z_INDEX + index calculation
- Check for duplicate instanceIds in instanceOrder

**Issue**: App state lost after refresh
**Diagnosis**:
- Check localStorage for "ryos:app-store" key
- Verify persistence config in useAppStore (line 695-735)
- Check migration logic executed (console logs)
**Fix**:
- Ensure partialize includes necessary state (lines 698-731)
- Check onRehydrateStorage logic (lines 762-834)
- Bump CURRENT_APP_STORE_VERSION if schema changed

**Issue**: Theme change doesn't update layout
**Diagnosis**:
- Check useThemeStore.current value
- Verify MenuBar conditional (AppManager.tsx:274-279)
- Check WindowFrame theme-specific styling
**Fix**:
- Ensure setTheme persists to localStorage
- Verify isXpTheme calculation (line 43)
- Check theme-specific component imports

**Issue**: CustomEvent not received
**Diagnosis**:
- Check event name spelling (launchApp, appStateChange, instanceStateChange)
- Verify addEventListener registered before dispatchEvent
- Check event.detail structure matches listener expectations
**Fix**:
- Add listener in useEffect with cleanup
- Use setTimeout(0) to defer event dispatch
- Log event.detail in both dispatch and listener

---

## 🚀 Development Environment

### **Local Development Setup**

```bash
# Prerequisites: Bun 1.2.19+ (package manager)
# Clone repository
cd /Users/jb/Desktop/DEV-PROJECTS/ryos

# Install dependencies
bun install

# Start development server (default port 5173 or $PORT)
bun dev

# Development server available at http://localhost:5173
# Hot module replacement (HMR) enabled via Vite
```

### **Environment Variables**

Create `.env` file (not committed):
```bash
# AI Models (Vercel AI SDK)
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
GOOGLE_GENERATIVE_AI_API_KEY=...

# Real-time Messaging (Pusher)
VITE_PUSHER_APP_KEY=...
VITE_PUSHER_CLUSTER=us2
VITE_PUSHER_APP_ID=...
PUSHER_SECRET=...

# Server-side Cache (Upstash Redis)
UPSTASH_REDIS_REST_URL=https://...
UPSTASH_REDIS_REST_TOKEN=...

# Text-to-Speech (Optional)
ELEVEN_LABS_API_KEY=...

# Rate Limiting
RATE_LIMIT_MAX_REQUESTS=100
RATE_LIMIT_WINDOW_MS=60000
```

### **Dependencies**

**Runtime**:
- **Frontend**: React 18.3.1, React DOM 18.3.1
- **Build**: Vite 6.3.5, TypeScript 5.6.3, Bun 1.2.19
- **State**: Zustand 5.0.5, React Context
- **Styling**: Tailwind CSS 4.1.10, xp.css 0.2.6, Framer Motion 12.18.1
- **UI**: Radix UI primitives (@radix-ui/react-*), Lucide React 0.475.0
- **AI**: @ai-sdk/react 2.0, @ai-sdk/anthropic 2.0, @ai-sdk/openai 2.0, @ai-sdk/google 2.0, OpenAI 4.104.0
- **Audio**: Tone.js 15.1.22, WaveSurfer.js 7.9.5, Audio Buffer Utils 5.1.2
- **Rich Text**: Tiptap 2.14.0 (core, react, starter-kit, extensions)
- **Video**: React Player 2.16.0
- **Real-time**: Pusher 5.2.0, Pusher-js 8.4.0
- **Storage**: @upstash/redis 1.35.0, IndexedDB (native)
- **Utilities**: UUID 11.1.0, Clsx 2.1.1, Zod 4, DOMPurify (types 3.2.0)
- **Internationalization**: BudouX 0.7.0, OpenCC-js 1.0.5, Hangul Romanization 1.0.1
- **3D/Shaders**: Three.js 0.176.0
- **Text Filtering**: Leo Profanity 1.8.0
- **Analytics**: @vercel/analytics 1.5.0, @vercel/functions 2.2.0

**Development**:
- **Linting**: ESLint 9.29.0, @eslint/js, typescript-eslint 8.34.0
- **Build Plugins**: @vitejs/plugin-react 4.5.2, vite-plugin-pwa 0.21.2, vite-plugin-vercel 9.0.7, @tailwindcss/vite 4.1.10
- **Styling Tools**: Autoprefixer 10.4.21, PostCSS 8.5.5, @tailwindcss/typography 0.5.16
- **Type Definitions**: @types/react, @types/react-dom, @types/node, @types/uuid, @types/three

### **Testing & Validation**

**Current State**: No formal test infrastructure (opportunity for enhancement)

**Manual Testing Checklist**:
```bash
# Build validation
bun run build
bun run preview

# Lint check
bun run lint

# App launch testing (in browser console)
window.dispatchEvent(new CustomEvent('launchApp', {
  detail: { appId: 'finder' }
}))

# Instance integrity check
useAppStore.getState()._debugCheckInstanceIntegrity()

# Theme switching validation
useThemeStore.getState().setTheme('xp')
useThemeStore.getState().setTheme('aqua')

# Virtual FS backup
// Control Panels → Backup → Download JSON
// Control Panels → Restore → Upload JSON
```

**Recommended Testing Strategy** (Future Enhancement):
- **Unit Tests**: Zustand store actions (Jest + @testing-library/react)
- **Integration Tests**: App launch flows, window management, theme switching
- **E2E Tests**: Full user workflows (Playwright/Cypress)
  - Launch app → interact → close
  - Multi-window scenarios
  - File operations (Finder → TextEdit)
  - AI chat flows
  - Theme switching
- **Visual Regression**: Screenshot comparisons for theme consistency

### **Build & Deployment**

```bash
# Production build
bun run build
# Output: dist/ directory
# Optimizations: minification, code splitting (react, ui, audio chunks)

# Preview production build
bun run preview

# Generate asset manifests (if adding icons/wallpapers)
bun run generate:icons
bun run generate:wallpapers

# Kiosk mode launch (macOS)
bun run launch-kiosk
```

**Deployment**: Vercel (automatic via GitHub integration)
- **Build Command**: `bun run build`
- **Output Directory**: `dist`
- **Framework**: Vite
- **Serverless Functions**: `/api/*.ts` → Vercel Functions
- **Environment Variables**: Set in Vercel dashboard
- **Domain**: Custom domain or vercel.app subdomain
- **Analytics**: Automatic via @vercel/analytics

**Build Configuration** (`vite.config.ts`):
- Manual chunk splitting: react, ui, audio
- Minification enabled
- Source maps disabled (production)
- PWA support (vite-plugin-pwa)
- Vercel integration (vite-plugin-vercel)

---

## 🌐 API Endpoints (Vercel Serverless Functions)

**Location**: `/api/*.ts` - Edge runtime for streaming responses
**Total**: 13 endpoints (9 primary + 4 utilities)

### **Primary Endpoints**

**1. `/api/chat.ts`** - AI Chat Inference
- **Method**: POST
- **Runtime**: Edge (streaming support)
- **Purpose**: Stream AI responses from Anthropic/OpenAI/Google
- **Request Body**:
  ```typescript
  {
    messages: Message[];          // Chat history
    model?: AIModel;             // 'claude-3-5-sonnet' | 'gpt-4' | 'gemini-pro'
    systemPrompt?: string;       // System instructions
    temperature?: number;        // 0-2 (default 1)
  }
  ```
- **Response**: `ReadableStream` (Server-Sent Events format)
- **Features**:
  - Anthropic prompt caching (reduces cost 90% for repeated system prompts)
  - Multi-provider routing via Vercel AI SDK
  - Streaming with tool calling support
  - Redis-backed rate limiting (burst + daily)
- **Rate Limits**: 100 req/hour burst, 500 req/day per IP
- **Used By**: Chats app, Terminal (`ryo` command)
- **Cost**: ~$0.002-0.01 per request (varies by model)

**2. `/api/audio-transcribe.ts`** - Speech-to-Text
- **Method**: POST (multipart/form-data)
- **Purpose**: Convert audio to text via OpenAI Whisper
- **Request**: FormData with 'audio' field (Blob, max 25MB)
- **Response**:
  ```typescript
  {
    text: string;                // Transcribed text
    duration: number;            // Audio duration (seconds)
    language?: string;           // Detected language
  }
  ```
- **Features**: Voice Activity Detection (VAD), multi-language support
- **Rate Limits**: 50 req/hour per IP
- **Used By**: Chats app (voice messages), useAudioTranscription hook
- **Cost**: $0.006 per minute of audio

**3. `/api/speech.ts`** - Text-to-Speech
- **Method**: POST
- **Purpose**: Generate audio from text (OpenAI TTS or ElevenLabs)
- **Request**:
  ```typescript
  {
    text: string;                // Text to speak (max 4096 chars)
    model?: 'openai' | 'elevenlabs';  // Default: 'openai'
    voice?: string;              // Voice ID
    speed?: number;              // 0.25-4.0 (default 1.0)
  }
  ```
- **Response**: `audio/mpeg` stream (MP3)
- **Rate Limits**: 100 req/hour per IP
- **Used By**: Chats app, useTtsQueue hook
- **Cost**: $0.015 per 1000 chars (OpenAI), $0.30 per 1000 chars (ElevenLabs)

**4. `/api/lyrics.ts`** - Lyric Fetching
- **Method**: GET `?videoId=<youtube_id>`
- **Purpose**: Fetch time-synced lyrics from Genius/LRClib APIs
- **Response**:
  ```typescript
  {
    lyrics: string;              // LRC format with timestamps
    source: 'genius' | 'lrclib' | 'local';
    artist?: string;
    title?: string;
    syncType: 'line' | 'word' | 'none';
  }
  ```
- **Features**:
  - Fallback chain: Genius → LRClib → YouTube captions
  - LRC parsing with [mm:ss.xx] timestamps
  - Redis caching (24hr TTL)
- **Rate Limits**: 200 req/hour per IP
- **Used By**: iPod app, useLyrics hook
- **Cost**: Free (public APIs)

**5. `/api/translate-lyrics.ts`** - Lyric Translation
- **Method**: POST
- **Purpose**: Translate lyrics to target language (Google Translate API)
- **Request**:
  ```typescript
  {
    text: string;                // Lyrics to translate
    targetLang: string;          // ISO 639-1 code ('en', 'es', 'zh', etc.)
    sourceLang?: string;         // Auto-detect if omitted
  }
  ```
- **Response**:
  ```typescript
  {
    translatedText: string;
    sourceLang: string;          // Detected source language
    confidence: number;          // 0-1 detection confidence
  }
  ```
- **Rate Limits**: 100 req/hour per IP
- **Used By**: iPod app (multi-language lyric display)
- **Cost**: $20 per 1M characters

**6. `/api/ie-generate.ts`** - Internet Explorer AI Content Generation
- **Method**: POST
- **Purpose**: Generate historical/futuristic website snapshots
- **Request**:
  ```typescript
  {
    url: string;                 // Website URL
    year: number;                // Target year (1990-2030)
    mode: 'historical' | 'futuristic';
    prompt?: string;             // Custom generation instructions
  }
  ```
- **Response**:
  ```typescript
  {
    html: string;                // Generated HTML content
    description: string;         // AI description of changes
    shareCode: string;           // Unique share code for URL
  }
  ```
- **Features**: Claude 3.5 Sonnet for content generation, prompt caching
- **Rate Limits**: 20 req/hour per IP
- **Used By**: Internet Explorer app
- **Cost**: ~$0.01 per generation

**7. `/api/link-preview.ts`** - Link Metadata Extraction
- **Method**: GET `?url=<encoded_url>`
- **Purpose**: Extract Open Graph metadata from URLs
- **Response**:
  ```typescript
  {
    title: string;
    description: string;
    image?: string;              // OG image URL
    url: string;                 // Canonical URL
    favicon?: string;
  }
  ```
- **Features**: HTML meta tag parsing, OG protocol support, Redis caching
- **Rate Limits**: 100 req/hour per IP
- **Used By**: Internet Explorer app, Chats app (link previews)
- **Cost**: Free

**8. `/api/parse-title.ts`** - YouTube Video Metadata
- **Method**: GET `?videoId=<youtube_id>`
- **Purpose**: Extract video title, artist, duration from YouTube
- **Response**:
  ```typescript
  {
    title: string;               // Full video title
    artist?: string;             // Parsed from title (before "-")
    track?: string;              // Parsed from title (after "-")
    duration: number;            // Video duration (seconds)
    thumbnail: string;           // Thumbnail URL
  }
  ```
- **Rate Limits**: 200 req/hour per IP
- **Used By**: iPod app, Videos app
- **Cost**: Free (YouTube Data API)

**9. `/api/chat-rooms.js`** - Chat Room Management (Pusher)
- **Method**: POST
- **Purpose**: Create/join/leave chat rooms with presence tracking
- **Request**:
  ```typescript
  {
    action: 'create' | 'join' | 'leave' | 'list';
    roomId?: string;             // Required for join/leave
    userId: string;
    username: string;
  }
  ```
- **Response**:
  ```typescript
  {
    success: boolean;
    roomId: string;
    members: User[];             // Room participants
    channelAuth?: string;        // Pusher auth for presence
  }
  ```
- **Features**: Pusher presence channels, real-time member list
- **Rate Limits**: 50 req/hour per IP
- **Used By**: Chats app (public rooms)
- **Cost**: Free tier (100 concurrent connections)

---

### **Utility Endpoints**

**`/api/utils/rateLimiter.ts`** - Rate Limiting Middleware
- **Purpose**: Redis-backed rate limiting (Upstash)
- **Algorithm**: Sliding window with burst + daily limits
- **Features**:
  - Per-IP tracking via headers (x-forwarded-for)
  - Configurable burst window (1 hour) + daily cap
  - Grace period for key rotation (30 min)
- **Usage**: Imported by all primary endpoints
- **Storage**: Upstash Redis (persistent)

**`/api/utils/cors.ts`** - CORS Header Injection
- **Purpose**: Cross-origin request handling
- **Configuration**:
  - Methods: GET, POST, OPTIONS
  - Credentials: true (cookies/auth headers)
  - Origin: Wildcard in dev, allowlist in prod
- **Usage**: Applied to all endpoints

**`/api/utils/auth.ts`** - API Key Authentication
- **Purpose**: Multi-token authentication with rotation
- **Features**:
  - Primary + secondary token support (grace period)
  - Token validation via environment variables
  - Automatic fallback during key rotation
- **Usage**: AI endpoints (chat, transcribe, speech)

**`/api/utils/geo.ts`** - Geolocation Inference
- **Purpose**: Extract user location from request headers
- **Source**: Vercel edge network headers
- **Returns**:
  ```typescript
  {
    country: string;             // ISO country code
    region: string;              // State/province
    city: string;
    ip: string;                  // Anonymized IP
  }
  ```
- **Usage**: Rate limiting, analytics, locale detection

---

### **Common API Patterns**

**Error Handling**:
```typescript
// All endpoints return consistent error format
{
  error: string;                 // Human-readable error message
  code: string;                  // Error code (RATE_LIMIT_EXCEEDED, etc.)
  status: number;                // HTTP status code
}
```

**Rate Limit Headers** (on all responses):
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 87
X-RateLimit-Reset: 1642531200000  // Unix timestamp
```

**Streaming Format** (chat, speech):
```
data: {"type": "text", "content": "Hello"}
data: {"type": "done"}
```

**Caching Strategy**:
- Lyrics: 24hr Redis cache
- Link previews: 1hr Redis cache
- Video metadata: 6hr Redis cache
- AI responses: No cache (always fresh)

---

## 🔧 Developer Quick Start & Context Rebuilding

### **Essential Files for Understanding**

**Priority Reading Order**:
1. **This file (CLAUDE.md)** - Complete system context
2. **README.md** - User perspective, feature overview
3. **src/apps/base/AppManager.tsx** - Central orchestration logic
4. **src/stores/useAppStore.ts** - Instance management, state persistence
5. **src/config/appRegistry.ts** - App configuration registry
6. **src/App.tsx** - Root component, boot logic

### **Most Common Developer Tasks**

**Add New Application**:
1. Create app directory: `src/apps/my-app/`
2. Export BaseApp object with id, name, icon, Component, initialData type
3. Add to `src/config/appRegistry.ts` with windowConfig
4. Add app ID to `src/config/appIds.ts`
5. Create app-specific Zustand store if needed: `src/stores/useMyAppStore.ts`
6. Test launch via CustomEvent: `window.dispatchEvent(new CustomEvent('launchApp', { detail: { appId: 'my-app' } }))`
7. Update this file (CLAUDE.md):
   - Component Architecture → Business Logic Layer (add app description)
   - Data Flow Dependencies (if new patterns)
   - Version History (what was added)

**Add New Theme**:
1. Create theme definition: `src/themes/my-theme.ts`
2. Add theme fonts: `public/fonts/my-theme/`
3. Add theme icons: `public/icons/my-theme/`
4. Add theme wallpapers: `public/wallpapers/my-theme/`
5. Update `useThemeStore` with new theme ID
6. Update `MenuBar` conditional (AppManager.tsx:274-279) for layout mode
7. Update `WindowFrame` control button rendering for theme-specific styling
8. Update `Dock` for theme-specific appearance
9. Test theme switch: `useThemeStore.getState().setTheme('my-theme')`
10. Update this file (CLAUDE.md):
    - Impact Radius Documentation (theme change effects)
    - Component Architecture → useThemeStore (new theme ID)
    - Version History

**Modify Window Management**:
1. Read `useAppStore` instance methods (lines 458-678)
2. Test changes with `_debugCheckInstanceIntegrity()`
3. If changing instance structure:
   - Bump `CURRENT_APP_STORE_VERSION`
   - Add migration logic to `migrate()` (lines 732-761)
   - Test migration from previous version
4. Update `AppManager` if z-index calculation changes (lines 76-80)
5. Update this file (CLAUDE.md):
   - Data Flow Dependencies (if routing changes)
   - Impact Radius Documentation (useAppStore effects)
   - Common Issues & Solutions (new debugging patterns)
   - Version History (breaking change documentation)

**Add API Endpoint**:
1. Create `/api/my-endpoint.ts` (Vercel serverless function)
2. Add rate limiting via `/api/utils/rateLimiter.ts`
3. Add CORS headers via `/api/utils/cors.ts`
4. Test locally: `bun dev` → `http://localhost:5173/api/my-endpoint`
5. Add environment variables to `.env` (if needed)
6. Update Vercel dashboard with environment variables
7. Update this file (CLAUDE.md):
   - Critical Data Flow (if new flow pattern)
   - Development Environment → Environment Variables (if new vars)

**Debug Instance Issue**:
1. Check console logs: `[AppManager]` and `[AppStore]` prefixes
2. Inspect state: `useAppStore.getState().instances`
3. Verify instanceOrder: `useAppStore.getState().instanceOrder`
4. Check z-index: Right-click window → Inspect → Computed styles → z-index
5. Run integrity check: `useAppStore.getState()._debugCheckInstanceIntegrity()`
6. Check CustomEvent listeners: `getEventListeners(window)` in Chrome DevTools
7. Refer to "Critical Debugging Points" section above (lines 420-490)

---

### **Implementation Status**

**✅ Complete**:
- Instance-based window management with multi-window support
- 18 fully-functional applications
- 4 theme system with dynamic layout switching
- Virtual filesystem with IndexedDB persistence
- AI integration (chat, voice, TTS, transcription via Vercel AI SDK)
- Real-time collaboration (Pusher chat rooms)
- Responsive design (mobile/tablet/desktop)
- PWA support (offline capability, installable)
- Custom wallpaper upload with IndexedDB storage
- Sound effects system (UI, Terminal, Chat, synth)
- App-specific state persistence (17 Zustand stores)
- URL-based app launching and sharing (Internet Explorer, iPod, Videos)
- Drag-drop file operations (Finder)
- Multi-document editing (TextEdit multi-window)
- Backup/restore system (Control Panels)

**🚧 In Progress / Enhancement Opportunities**:
- Formal testing infrastructure (unit, integration, E2E)
- API endpoint documentation (Swagger/OpenAPI)
- Performance monitoring (lighthouse scores, Core Web Vitals)
- Accessibility audit (WCAG 2.1 AA compliance)
- Bundle size optimization (dynamic imports for large apps)
- Service Worker caching strategy (PWA enhancement)
- Error boundary implementation (graceful error handling)
- Analytics dashboard (user behavior, popular apps)

**📋 Planned / Future Considerations**:
- Plugin system for third-party apps (applet-viewer expansion)
- Collaborative editing (real-time document co-editing)
- Cloud sync (cross-device state synchronization)
- Mobile app (React Native / Capacitor)
- Multi-user chat rooms (expanded Pusher integration)
- File sharing between users (P2P or server-mediated)
- Advanced AI features (code execution, image generation)
- Performance mode (disable shaders/animations on low-end devices)

---

## 📁 File Structure

```
ryos/
├── public/                          # Static Assets (88 MB total)
│   ├── assets/                      # Videos, sounds, media
│   │   ├── sounds/                 # UI sound effects (click, hover, etc.)
│   │   └── videos/                 # Background videos for wallpapers
│   ├── fonts/                       # Theme-specific system fonts
│   │   ├── aqua/                   # Lucida Grande, SF Pro
│   │   ├── system7/                # Chicago, Charcoal
│   │   ├── xp/                     # Tahoma, Trebuchet MS
│   │   └── win98/                  # MS Sans Serif
│   ├── icons/                       # UI icons organized by category
│   │   ├── apps/                   # Application icons (theme-specific)
│   │   ├── files/                  # File type icons
│   │   ├── controls/               # Window controls (close, min, max)
│   │   └── system/                 # System icons (trash, folder, etc.)
│   ├── patterns/                    # Pattern files for MacPaint
│   └── wallpapers/                  # Wallpaper images
│       ├── photos/                 # Photo wallpapers by theme
│       │   ├── aqua/               # Aqua wallpapers (water, space)
│       │   ├── system7/            # System 7 wallpapers
│       │   ├── xp/                 # Windows XP wallpapers (bliss)
│       │   └── win98/              # Windows 98 wallpapers
│       └── tiles/                  # Tiling pattern wallpapers
├── src/
│   ├── apps/                        # Application Modules (18 apps)
│   │   ├── applet-viewer/          # Custom web app viewer
│   │   │   ├── components/         # App-specific components
│   │   │   ├── hooks/              # App-specific hooks
│   │   │   └── index.tsx           # Main app component + BaseApp export
│   │   ├── base/                   # Base types & AppManager
│   │   │   ├── AppManager.tsx      # Central orchestrator (10 KB)
│   │   │   └── types.ts            # BaseApp, AppState, AppInstance types
│   │   ├── chats/                  # AI chat with voice & tool calling
│   │   ├── control-panels/         # System settings & backup/restore
│   │   ├── finder/                 # File manager
│   │   ├── internet-explorer/      # Time-travel browser with AI
│   │   ├── ipod/                   # Music player with lyrics
│   │   ├── minesweeper/            # Classic game
│   │   ├── paint/                  # Bitmap graphics editor
│   │   ├── pc/                     # DOS game emulator
│   │   ├── photo-booth/            # Camera app with filters
│   │   ├── soundboard/             # Audio recorder/player
│   │   ├── synth/                  # Virtual synthesizer
│   │   ├── terminal/               # Unix-like CLI
│   │   ├── textedit/               # Rich text editor
│   │   └── videos/                 # YouTube playlist player
│   │
│   ├── components/                  # Shared React Components
│   │   ├── dialogs/                # Modal dialogs
│   │   │   ├── AboutFinderDialog.tsx
│   │   │   ├── BootScreen.tsx      # System restore animation
│   │   │   ├── HelpDialog.tsx      # App-specific help
│   │   │   └── ConfirmDialog.tsx   # Confirmation dialogs
│   │   ├── layout/                 # Core layout components
│   │   │   ├── Desktop.tsx         # Desktop viewport & wallpaper (5 KB)
│   │   │   ├── Dock.tsx            # Application launcher (8 KB)
│   │   │   ├── MenuBar.tsx         # Top menu (Mac) / Taskbar (Windows) (15 KB)
│   │   │   ├── StartMenu.tsx       # Windows Start menu (3 KB)
│   │   │   ├── WindowFrame.tsx     # Resizable window wrapper (12 KB)
│   │   │   └── AppleMenu.tsx       # macOS Apple menu (2 KB)
│   │   ├── shared/                 # Shared utility components
│   │   │   ├── ThemedIcon.tsx      # Theme-aware icon loader
│   │   │   ├── GalaxyBackground.tsx # WebGL shader background
│   │   │   └── VolumeSlider.tsx    # Volume control UI
│   │   └── ui/                     # shadcn/ui components (30+)
│   │       ├── button.tsx          # Radix UI Button
│   │       ├── dialog.tsx          # Radix UI Dialog
│   │       ├── dropdown-menu.tsx   # Radix UI Dropdown
│   │       ├── select.tsx          # Radix UI Select
│   │       ├── slider.tsx          # Radix UI Slider
│   │       ├── tabs.tsx            # Radix UI Tabs
│   │       ├── tooltip.tsx         # Radix UI Tooltip
│   │       ├── scroll-area.tsx     # Radix UI ScrollArea
│   │       └── sonner.tsx          # Toast notifications
│   │
│   ├── config/                      # Configuration
│   │   ├── appIds.ts               # App ID constants (0.5 KB)
│   │   └── appRegistry.ts          # App registry with window configs (10 KB)
│   │
│   ├── contexts/                    # React Context Providers
│   │   └── AppContext.tsx          # Global app state context (legacy compatibility)
│   │
│   ├── hooks/                       # Custom React Hooks (19 total)
│   │   ├── useAuth.ts              # Chat authentication
│   │   ├── useAudioTranscription.ts # Speech-to-text
│   │   ├── useChatSynth.ts         # Chat typing synth
│   │   ├── useLyrics.ts            # Lyric fetching & sync
│   │   ├── useSound.ts             # Sound effect playback
│   │   ├── useTerminalSounds.ts    # Terminal audio
│   │   ├── useTtsQueue.ts          # Text-to-speech queue
│   │   ├── useWindowManager.ts     # Window lifecycle
│   │   ├── useMediaQuery.ts        # CSS media query
│   │   ├── useIsMobile.ts          # <768px detection
│   │   ├── useIsPhone.ts           # <600px detection
│   │   └── ...9 more hooks
│   │
│   ├── lib/                         # Libraries & Utilities
│   │   ├── audioContext.ts         # Web Audio API singleton
│   │   ├── pusherClient.ts         # Pusher initialization
│   │   ├── webglFilterRunner.ts    # WebGL shader pipeline
│   │   ├── utils.ts                # General utilities
│   │   └── shaders/                # GLSL shader files
│   │       ├── crt.frag            # CRT effect
│   │       ├── galaxy.frag         # Galaxy spiral
│   │       └── aurora.frag         # Aurora borealis
│   │
│   ├── stores/                      # Zustand State Management (17 stores)
│   │   ├── useAppStore.ts          # Instance management, settings (32 KB) **CRITICAL**
│   │   ├── useChatsStore.ts        # Chat messages & rooms (62 KB est.)
│   │   ├── useFilesStore.ts        # Virtual filesystem (27 KB est.)
│   │   ├── useThemeStore.ts        # Theme selection (5 KB est.)
│   │   ├── useTerminalStore.ts     # Terminal state (10 KB est.)
│   │   ├── useTextEditStore.ts     # Document editor state (15 KB est.)
│   │   ├── useIpodStore.ts         # Music player state (31 KB est.)
│   │   ├── usePaintStore.ts        # Canvas state (20 KB est.)
│   │   ├── useSoundboardStore.ts   # Soundboard state (15 KB est.)
│   │   ├── useSynthStore.ts        # Synth parameters (10 KB est.)
│   │   ├── usePhotosStore.ts       # Photo gallery (8 KB est.)
│   │   ├── useVideoStore.ts        # Video playlists (12 KB est.)
│   │   ├── useInternetExplorerStore.ts # Browser history (15 KB est.)
│   │   ├── useMinesweeperStore.ts  # Game state (5 KB est.)
│   │   ├── usePcStore.ts           # Emulator state (10 KB est.)
│   │   ├── useFinderStore.ts       # Finder preferences (8 KB est.)
│   │   └── useAppletStore.ts       # Applet viewer state (5 KB est.)
│   │
│   ├── styles/                      # Global CSS
│   │   ├── globals.css             # Base styles, Tailwind imports
│   │   └── theme-overrides.css     # Theme-specific overrides
│   │
│   ├── themes/                      # Theme Definitions
│   │   ├── system7.ts              # System 7 theme config
│   │   ├── aqua.ts                 # Aqua theme config
│   │   ├── xp.ts                   # Windows XP theme config
│   │   └── win98.ts                # Windows 98 theme config
│   │
│   ├── types/                       # TypeScript Type Definitions
│   │   ├── aiModels.ts             # AI model types
│   │   ├── files.ts                # File system types
│   │   └── global.d.ts             # Global type declarations
│   │
│   ├── utils/                       # Utility Functions
│   │   ├── displayMode.ts          # CSS filter application
│   │   ├── bootMessage.ts          # Boot message persistence
│   │   ├── sharedUrl.ts            # URL parsing (share codes)
│   │   ├── indexedDB.ts            # IndexedDB initialization
│   │   ├── performanceCheck.ts     # Device capability detection
│   │   └── helpers.ts              # Zustand shallow comparison
│   │
│   ├── App.tsx                      # Root component (1.5 KB)
│   ├── main.tsx                     # React bootstrap (0.5 KB)
│   └── index.css                    # Global styles entry
│
├── api/                             # Vercel Serverless Functions
│   ├── chat.ts                     # AI chat endpoint (Anthropic/OpenAI/Google)
│   ├── audio-transcribe.ts         # Speech-to-text (OpenAI Whisper)
│   ├── lyrics.ts                   # Lyric fetching (Genius/LRClib)
│   ├── speech.ts                   # Text-to-speech (OpenAI/ElevenLabs)
│   ├── translate-lyrics.ts         # Lyric translation (Google Translate)
│   ├── ie-generate.ts              # Internet Explorer AI generation
│   ├── link-preview.ts             # Link metadata extraction
│   ├── parse-title.ts              # Title parsing utility
│   ├── chat-rooms.js               # Chat room management (Pusher)
│   └── utils/                      # API utilities
│       ├── rateLimiter.ts          # Rate limiting (Upstash Redis)
│       └── cors.ts                 # CORS headers
│
├── scripts/                         # Utility Scripts
│   ├── generate-icon-manifest.ts   # Icon manifest generator (Bun)
│   ├── generate-wallpaper-manifest.ts # Wallpaper manifest generator (Bun)
│   └── launch-kiosk.sh             # Kiosk mode launcher (macOS)
│
├── .git/                            # Git Repository
│   └── hooks/                      # Git Hooks (to be created)
│       └── pre-commit              # Documentation sync validation
│
├── .cursor/                         # Cursor IDE Settings
│   └── project-spec-todos.md       # Project-specific todos (Cursor)
│
├── package.json                     # Dependencies & scripts (3.3 KB)
├── bun.lock / bun.lockb            # Bun lockfiles (380 KB + 397 KB)
├── vite.config.ts                  # Vite build configuration (6.5 KB)
├── tailwind.config.js              # Tailwind CSS configuration (6.5 KB)
├── tsconfig.json                   # TypeScript configuration
├── eslint.config.js                # ESLint configuration
├── components.json                 # shadcn/ui configuration
├── vercel.json                     # Vercel deployment config
├── index.html                      # HTML entry point (2.1 KB)
├── .gitignore                      # Git ignore rules
├── .hintrc                         # Webhint configuration
├── README.md                       # User-facing documentation (7.8 KB)
├── CLAUDE.md                       # **THIS FILE** - Developer documentation
└── LICENSE                         # AGPL-3.0 license (33 KB)
```

---

## 📊 Application Details (18 Apps)

### **Core Applications**

**1. Finder** (`src/apps/finder/`)
- **Purpose**: Virtual filesystem manager with Quick Access, Storage Info, drag-drop
- **Window**: 400×300 (min 300×200)
- **Store**: useFinderStore (navigation history, view preferences)
- **Multi-window**: ✅ Yes
- **Key Features**: Tree view, breadcrumb navigation, file operations (create, delete, rename, move), storage quota display
- **Integration**: useFilesStore (filesystem), CustomEvent("launchApp") for file opening

**2. TextEdit** (`src/apps/textedit/`)
- **Purpose**: Rich text editor with Markdown, task lists, syntax highlighting
- **Window**: 430×475 (min 430×200)
- **Store**: useTextEditStore (open documents, cursor positions)
- **Multi-window**: ✅ Yes (open multiple documents simultaneously)
- **Key Features**: Tiptap editor, Markdown support, task lists, syntax highlighting (Shiki), auto-save to virtual FS
- **Integration**: useFilesStore (document persistence), initialData: { path: string } for file opening

**3. MacPaint** (`src/apps/paint/`)
- **Purpose**: Classic bitmap graphics editor with patterns, shapes, tools
- **Window**: 713×480 (min 400×400, max 713×535)
- **Store**: usePaintStore (canvas state, tool selection, undo/redo stack)
- **Multi-window**: ❌ No
- **Key Features**: Drawing tools (pencil, brush, spray, eraser), shapes (rectangle, oval, line), fill patterns, undo/redo, image import/export
- **Integration**: Canvas API, pattern assets from public/patterns/

**4. Videos** (`src/apps/videos/`)
- **Purpose**: YouTube playlist player with VCR-style UI, scrolling titles, LCD display
- **Window**: 730×475 (default)
- **Store**: useVideoStore (playlists, playback state, current video)
- **Multi-window**: ❌ No
- **Key Features**: React Player integration, playlist management, shuffle/repeat modes, share URLs (/videos/:videoId)
- **Integration**: YouTube IFrame API, localStorage persistence, URL sharing

**5. Soundboard** (`src/apps/soundboard/`)
- **Purpose**: Audio recorder/player with waveform visualization, keyboard shortcuts
- **Window**: 650×475 (min 550×375)
- **Store**: useSoundboardStore (soundboards, recordings, metadata)
- **Multi-window**: ❌ No
- **Key Features**: Microphone recording, WaveSurfer.js visualization, keyboard shortcuts (1-9), emoji customization, synth effects, import/export
- **Integration**: Web Audio API, WaveSurfer.js, audioContext.ts

**6. Synth** (`src/apps/synth/`)
- **Purpose**: Virtual synthesizer with MIDI support, effects, presets
- **Window**: 730×475 (default)
- **Store**: useSynthStore (synth parameters, presets, MIDI state)
- **Multi-window**: ❌ No
- **Key Features**: Tone.js synthesis, oscillator types (sine, square, sawtooth, triangle), effects (reverb, delay, distortion), MIDI input, preset system
- **Integration**: Tone.js, Web Audio API, MIDI API, audioContext.ts

**7. Photo Booth** (`src/apps/photo-booth/`)
- **Purpose**: Camera app with real-time filters, brightness/contrast, gallery
- **Window**: 644×510 (fixed size)
- **Store**: usePhotosStore (photo gallery, filters, camera state)
- **Multi-window**: ❌ No
- **Key Features**: Webcam access, WebGL filters, brightness/contrast adjustments, photo gallery, export to virtual FS, multi-photo sequence mode
- **Integration**: MediaDevices API, Canvas API, useFilesStore (export)

---

### **Internet & Communication**

**8. Internet Explorer** (`src/apps/internet-explorer/`)
- **Purpose**: Time-travel browser with AI-generated historical snapshots
- **Window**: 730×600 (min 400×300)
- **Store**: useInternetExplorerStore (history, snapshots, favorites)
- **Multi-window**: ❌ No
- **Key Features**: Time Machine view (snapshots across years), AI content generation (pre-1996 sites, futuristic designs), favorites by year, share URLs (/internet-explorer/:shareCode)
- **Integration**: /api/ie-generate.ts (AI), /api/link-preview.ts, localStorage favorites, URL sharing

**9. Chats** (`src/apps/chats/`)
- **Purpose**: AI-powered chat with voice, tool calling, public rooms
- **Window**: 560×360 (min 300×320)
- **Store**: useChatsStore (messages, rooms, auth, typing indicators)
- **Multi-window**: ❌ No
- **Key Features**:
  - Natural conversation with Ryo AI (Anthropic/OpenAI/Google)
  - Public chat rooms with @ryo mentions
  - Push-to-talk voice messages with transcription
  - Text-to-speech with word highlighting
  - Tool calling (app control, document editing)
  - Nudge system (👋) with context-aware responses
  - ryOS FM DJ mode (music commentary)
  - Save transcript to Markdown
- **Integration**:
  - Vercel AI SDK (streaming responses)
  - /api/chat.ts (AI backend)
  - /api/audio-transcribe.ts (speech-to-text)
  - /api/speech.ts (text-to-speech)
  - Pusher (real-time messaging)
  - useTtsQueue (speech playback)

---

### **Media & Entertainment**

**10. iPod** (`src/apps/ipod/`)
- **Purpose**: 1st-gen iPod-style music player with lyrics, translation, fullscreen mode
- **Window**: 730×475 (default)
- **Store**: useIpodStore (library, playlists, playback state, lyrics)
- **Multi-window**: ❌ No
- **Key Features**:
  - Classic click-wheel navigation
  - YouTube URL import for music library
  - Time-synced lyrics with multi-language translation
  - Interactive lyric offset adjustment (gestures)
  - Multiple alignment modes (Focus Three, Alternating, Center)
  - Chinese character variants (Traditional/Simplified)
  - Korean romanization
  - Fullscreen lyrics mode with video
  - Real-time lyric highlighting
  - Playlist management, shuffle/loop modes
  - Back-light toggle
  - Share URLs (/ipod/:videoId)
- **Integration**:
  - React Player
  - /api/lyrics.ts (Genius/LRClib)
  - /api/translate-lyrics.ts
  - BudouX, OpenCC-js, Hangul Romanization
  - localStorage persistence

---

### **System Utilities**

**11. Control Panels** (`src/apps/control-panels/`)
- **Purpose**: System preferences, appearance, sounds, backup/restore, format
- **Window**: 730×475 (default)
- **Store**: useControlPanelsStore (settings, backup state)
- **Multi-window**: ❌ No
- **Key Features**:
  - Appearance: theme selection, wallpaper picker, shader selection (CRT, Galaxy, Aurora)
  - Sounds: UI/typing/Terminal sound toggles, volume controls
  - Backup: one-click full system backup to JSON
  - Restore: upload backup JSON to restore state
  - Format: reset virtual filesystem
- **Integration**:
  - useAppStore (all settings)
  - useThemeStore (theme)
  - useFilesStore (filesystem backup/restore/format)
  - All 17 stores (backup/restore)

**12. Terminal** (`src/apps/terminal/`)
- **Purpose**: Unix-like CLI with familiar commands, AI assistant, Vim integration
- **Window**: 730×475 (default)
- **Store**: useTerminalStore (command history, output buffer, FS context)
- **Multi-window**: ❌ No
- **Key Features**:
  - Commands: ls, cd, cat, touch, mkdir, rm, mv, cp, pwd, clear, help, vim, edit, ryo <prompt>
  - ↑/↓ history navigation
  - Tab auto-completion
  - "ryo <prompt>" for AI assistance
  - Open documents in TextEdit or Vim
  - Toggle Terminal sounds (View → Sounds)
- **Integration**:
  - useFilesStore (filesystem operations)
  - useTerminalSounds (audio feedback)
  - CustomEvent("launchApp") for TextEdit/Vim

---

### **Games & Entertainment**

**13. Minesweeper** (`src/apps/minesweeper/`)
- **Purpose**: Classic Minesweeper game implementation
- **Window**: Default size
- **Store**: useMinesweeperStore (game state, scores, difficulty)
- **Multi-window**: ❌ No
- **Key Features**: Classic gameplay, difficulty levels, timer, mine counter
- **Integration**: Canvas API for rendering

**14. Virtual PC** (`src/apps/pc/`)
- **Purpose**: DOS game emulator (Doom, SimCity, etc.)
- **Window**: Default size
- **Store**: usePcStore (emulator state, save states, game library)
- **Multi-window**: ❌ No
- **Key Features**: DOS environment emulation, game save states, game library management
- **Integration**: DOS emulator library (e.g., js-dos or similar)

---

### **Developer Tools**

**15. Applet Viewer** (`src/apps/applet-viewer/`)
- **Purpose**: Custom web app viewer/launcher for external web content
- **Window**: Variable (saved per applet path)
- **Store**: useAppletStore (applet paths, window sizes)
- **Multi-window**: ✅ Yes (open multiple applets simultaneously)
- **Key Features**:
  - Load external web content in iframe
  - Remember window sizes per applet
  - Custom app integration
- **Integration**:
  - initialData: { path: string } for applet URL
  - localStorage for window size persistence

---

## 🔑 Key Architectural Patterns

### **Instance-Based Window Management**

**Pattern**: Multi-window support via unique instance IDs
- **Implementation**: `useAppStore.instances` Record<instanceId, AppInstance>
- **Z-index**: Calculated from `instanceOrder` position (END = highest)
- **Focus**: `bringInstanceToForeground(instanceId)` moves to end of order
- **Multi-window apps**: TextEdit, Finder, Applet Viewer (explicit), others single-instance

**Benefits**:
- Multiple documents open simultaneously (TextEdit)
- Multiple folders open simultaneously (Finder)
- Independent window positions/sizes per instance
- Clean separation of concerns (instance vs app state)

**Example**:
```typescript
// Launch new TextEdit instance
const instanceId = useAppStore.getState().launchApp('textedit', {
  path: '/Documents/notes.txt'
})

// Later: focus that instance
useAppStore.getState().bringInstanceToForeground(instanceId)

// Close that instance
useAppStore.getState().closeAppInstance(instanceId)
```

---

### **Theme-Driven UI Transformation**

**Pattern**: Single codebase, multiple OS aesthetics
- **Implementation**: `useThemeStore.current` determines layout + styling
- **Layout Changes**: MenuBar (top) vs Taskbar (bottom), WindowFrame controls
- **Asset Changes**: Fonts, icons, wallpapers loaded dynamically
- **CSS Changes**: Tailwind classes + theme-specific overrides

**Conditional Rendering Example** (AppManager.tsx:274-279):
```typescript
const isXpTheme = currentTheme === 'xp' || currentTheme === 'win98'
const hasForeground = Boolean(getForegroundInstance())

// XP/Win98: always show taskbar
// Mac/System7: show menubar only when no app foreground
return isXpTheme || !hasForeground ? <MenuBar /> : null
```

**Benefits**:
- Authentic OS experiences without code duplication
- Easy theme switching at runtime
- Theme-specific behavior (menubar vs taskbar)

---

### **Event-Driven App Communication**

**Pattern**: CustomEvents for loose coupling
- **App Launch**: `CustomEvent("launchApp", { detail: { appId, initialData } })`
- **State Change**: `CustomEvent("instanceStateChange", { detail: { instanceId, isOpen, isForeground } })`
- **Filesystem**: `CustomEvent("fileSystemChange")`

**Benefits**:
- Decoupled components (no direct imports)
- Easy third-party app integration
- Testable via event mocking

**Example** (Finder opens file in TextEdit):
```typescript
// Finder dispatches event
window.dispatchEvent(new CustomEvent('launchApp', {
  detail: { appId: 'textedit', initialData: { path: '/Documents/notes.txt' } }
}))

// AppManager listens and handles
window.addEventListener('launchApp', (e) => {
  const { appId, initialData } = e.detail
  launchApp(appId, initialData)
})
```

---

### **Zustand Store Pattern**

**Pattern**: Modular state management with persistence
- **17 stores**: One per major feature (app state, files, chats, theme, etc.)
- **Persistence**: localStorage via zustand/persist middleware
- **Shallow comparison**: useAppStoreShallow helper prevents unnecessary re-renders
- **Migration**: Version-based migration system for breaking changes

**Store Template**:
```typescript
export const useMyStore = create<MyStoreState>()(
  persist(
    (set, get) => ({
      // State
      myData: [],

      // Actions
      addData: (item) => set((state) => ({
        myData: [...state.myData, item]
      })),

      // Computed
      getDataById: (id) => get().myData.find(d => d.id === id),
    }),
    {
      name: 'ryos:my-store',
      version: 1,
      partialize: (state) => ({ myData: state.myData }),
      migrate: (persisted, version) => {
        // Handle version migrations
        return persisted
      },
    }
  )
)
```

**Benefits**:
- Type-safe state management
- Automatic localStorage sync
- Selective rehydration (partialize)
- Version migrations for breaking changes

---

### **Virtual Filesystem Pattern**

**Pattern**: In-memory tree structure with IndexedDB persistence
- **Implementation**: useFilesStore with hierarchical node structure
- **Operations**: createFile, createFolder, deleteNode, moveNode, renameNode
- **Persistence**: Background IndexedDB writes
- **Backup/Restore**: Full tree serialization to JSON

**Node Structure**:
```typescript
type FileNode = {
  id: string
  name: string
  type: 'file' | 'folder'
  content?: string  // file content
  children?: FileNode[]  // folder children
  createdAt: number
  modifiedAt: number
}
```

**Benefits**:
- Familiar filesystem abstraction
- Persistent across sessions
- Backup/restore capability
- Integration with Finder, Terminal, TextEdit

---

## 🚦 Performance Considerations

**Current Optimizations**:
- Manual chunk splitting (react, ui, audio) reduces initial bundle
- Zustand shallow comparison prevents unnecessary re-renders
- IndexedDB for large data (files, wallpapers) keeps memory low
- Lazy shader loading (only when enabled in Control Panels)
- PWA caching for offline performance

**Known Performance Bottlenecks**:
- Large file tree (>1000 nodes) slows Finder rendering → consider virtualization
- Multiple TextEdit instances with large documents → consider Web Workers for parsing
- Shader effects on low-end devices → automatic disable via performanceCheck.ts
- Pusher connection overhead → consider reconnection backoff

**Optimization Opportunities**:
- Dynamic app imports (load app code only when launched)
- Virtual list for Finder (react-window or similar)
- Web Worker for Markdown parsing (TextEdit)
- Service Worker caching strategy (PWA enhancement)
- Image optimization (wallpapers, icons) → WebP conversion

---

## 🔐 Security Considerations

**Current Measures**:
- Rate limiting on API endpoints (Upstash Redis)
- CORS headers on API endpoints
- DOMPurify for HTML sanitization (TextEdit, Chats)
- Environment variable protection (API keys not exposed client-side)
- localStorage encryption (not implemented yet - opportunity)

**Potential Vulnerabilities**:
- XSS via user-generated content (Chats, TextEdit) → mitigated by DOMPurify
- API key exposure (client-side) → mitigated by serverless functions
- localStorage manipulation → consider encryption for sensitive data
- Pusher message injection → validate message structure

**Recommendations**:
- Implement Content Security Policy (CSP) headers
- Add input validation on all user-generated content
- Encrypt localStorage sensitive data (auth tokens)
- Implement CSRF protection for API endpoints
- Audit third-party dependencies (npm audit)

---

## 📈 Analytics & Monitoring

**Current Implementation**:
- Vercel Analytics for page views, session duration
- Console logging for debugging (prefixed: [AppManager], [AppStore])
- CustomEvent tracking for app launches, state changes

**Enhancement Opportunities**:
- Error boundary with Sentry/BugSnag integration
- Performance monitoring (Core Web Vitals)
- User behavior analytics (app usage patterns, popular apps)
- A/B testing framework for UI experiments
- Real-time monitoring dashboard (Upstash Analytics)

---

**Document Version**: 0.0.0
**Last Updated**: 2025-01-21
**Status**: Initial comprehensive documentation following /dev-process methodology
