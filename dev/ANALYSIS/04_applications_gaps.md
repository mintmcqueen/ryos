# Applications Analysis - CLAUDE.md Gaps & Recommendations
**Analysis Date**: 2025-10-30
**Comparison**: 04_applications_analysis.md vs CLAUDE.md

---

## Executive Summary

**CLAUDE.md Current Coverage**: Minimal application-specific details
- Brief mentions in component hierarchy (Level 4)
- Store listings without implementation details
- No per-app feature documentation
- No external library integration mapping

**04_applications_analysis.md Additions**: Comprehensive app-level documentation
- 18 detailed app profiles with feature catalogs
- External library integration matrix
- Inter-app communication patterns
- Multi-window support mapping
- InitialData interface documentation

**Recommendation**: Merge app-level details into CLAUDE.md Section 5 (Applications Layer)

---

## Section-by-Section Gap Analysis

### 1. System Overview (CLAUDE.md Lines 1-15)
**Current State**:
```markdown
- ✅ Complete: 18 built-in apps, 4 themes, instance-based windowing, virtual FS, AI chat/voice
```

**Gap**: No breakdown of which 18 apps exist, no categorization

**Proposed Addition**:
```markdown
### Built-in Applications (18 Total)

**Productivity** (5 apps):
- Finder - File management with multi-instance support
- TextEdit - Rich text editor with Tiptap, AI integration
- Paint - Canvas-based drawing with MacPaint aesthetics
- Terminal - Unix-like shell with 15+ commands, AI assistant
- Control Panels - System settings aggregator

**Communication** (1 app):
- Chats - AI assistant + real-time chat rooms (Pusher)

**Media** (4 apps):
- iPod - YouTube player with synced lyrics API
- Videos - YouTube playlist player (QuickTime UI)
- Synth - Virtual synthesizer (Tone.js, MIDI support)
- Soundboard - Audio recording/playback with WaveSurfer.js

**Utilities** (3 apps):
- Photo Booth - WebRTC camera capture with effects
- Applet Viewer - HTML applet renderer (multi-instance)
- Internet Explorer - Time-travel web browser (Wayback + AI)

**Gaming** (1 app):
- Minesweeper - Classic puzzle game

**Emulation** (1 app):
- Virtual PC - DOSBox emulator (js-dos)
```

---

### 2. Data Flow (CLAUDE.md Lines 19-60)
**Current State**: Generic flows for launch, focus, AI chat, filesystem

**Gap**: No app-specific data flows (lyrics sync, chat rooms, terminal commands)

**Proposed Addition** (new subsection):
```markdown
### App-Specific Data Flows

**Lyrics Synchronization (iPod)**:
```
Track starts → useLyrics(videoId)
→ GET /api/lyrics?videoId=xyz
→ Parse LRC format ([00:12.34]Line text)
→ ReactPlayer.onProgress(currentTime)
→ Find current line by timestamp
→ LyricsDisplay highlights + optional translation
```

**Real-Time Chat Rooms (Chats)**:
```
User sends message → POST /api/chat/rooms/[id]/messages
→ Server stores + Pusher.trigger("room-${id}", "message-sent")
→ All clients receive via pusher:message-sent event
→ useChatsStore.addMessageToRoom(roomId, message)
→ Optimistic update (sender sees immediately)
→ ChatMessages re-renders
```

**Terminal AI Integration**:
```
User: "ryo launch internet explorer"
→ ai.ts command handler
→ getSystemState() - Collects running apps, foreground state
→ POST /api/chat.ts with system context
→ AI responds with tool calls: { tool: "launchApp", args: {...} }
→ Terminal executes via useAppStore.launchApp
→ Output rendered with TypewriterText effect
```

**Time Travel Browsing (Internet Explorer)**:
```
User enters URL + year (e.g., apple.com, 1997)
→ fetchWaybackSnapshot(url, year)
→ Wayback CDX API query for closest snapshot
→ If found: Memento API retrieves archived HTML
→ If not found: triggerAiGeneration(url, year, language, location)
→ POST /api/internet-explorer/generate (streaming)
→ AI reconstructs/imagines page based on era
→ Render in HtmlPreview (sandboxed)
```
```

---

### 3. Component Architecture (CLAUDE.md Lines 77-147)
**Current State**: Entry points, business logic, data layer documented

**Gap**: No application layer breakdown

**Proposed New Section** (after Line 147):
```markdown
---

## 🎮 Application Layer - 18 Apps

### **Application Architecture Patterns**

**Standard Component Structure** (all apps follow):
```typescript
export function AppComponent({
  isWindowOpen,      // Visibility toggle
  onClose,           // Close handler
  isForeground,      // Focus state
  skipInitialSound,  // Boot sound bypass
  initialData,       // Launch-time data (typed per app)
  instanceId,        // Multi-window namespace
  onNavigateNext,    // Instance cycling
  onNavigatePrevious,
}: AppProps<AppInitialDataType>) {
  // 1. Local state (dialogs, UI state)
  const [isHelpDialogOpen, setIsHelpDialogOpen] = useState(false);

  // 2. Store integration (app-specific)
  const appState = useAppStore(state => state.data);

  // 3. Theme-aware menu bar
  const currentTheme = useThemeStore(state => state.current);
  const isXpTheme = currentTheme === "xp" || currentTheme === "win98";

  const menuBar = <AppMenuBar ... />;

  if (!isWindowOpen) return null;

  return (
    <>
      {/* Mac: Global menu bar when foreground */}
      {!isXpTheme && isForeground && menuBar}

      <WindowFrame
        title="App Title"
        onClose={onClose}
        appId="app-id"
        instanceId={instanceId}
        menuBar={isXpTheme ? menuBar : undefined} // Windows: Inside title bar
      >
        {/* App content */}
      </WindowFrame>

      {/* Standard dialogs */}
      <HelpDialog ... />
      <AboutDialog ... />
    </>
  );
}
```

**Multi-Window Support Pattern** (3 apps):
- **Finder**: Instance-based navigation (currentPath, history, viewType per window)
- **TextEdit**: Instance-based documents (independent editor state per window)
- **Applet Viewer**: Per-applet window sizes persisted

**Instance State Management**:
```typescript
// Create instance state on mount
useEffect(() => {
  if (instanceId) {
    const existingInstance = store.instances[instanceId];
    if (!existingInstance) {
      createInstance(instanceId, initialPath);
    }
  }
}, [instanceId]);

// Listen for App store cleanup events
useEffect(() => {
  const handleInstanceClose = (event: CustomEvent) => {
    if (event.detail.instanceId === instanceId && !event.detail.isOpen) {
      removeInstance(instanceId);
    }
  };
  window.addEventListener("instanceStateChange", handleInstanceClose);
  return () => window.removeEventListener("instanceStateChange", handleInstanceClose);
}, [instanceId]);
```

---

### **App-Specific Stores (12 Total)**

**Pattern**: Zustand with localStorage persistence + version migrations

**`useFinderStore`** (Instance-based navigation):
- **State**: instances[instanceId] → { currentPath, viewType, sortType, history }
- **Per-path preferences**: viewTypeByPath (persists list/grid preference per directory)
- **Methods**: createInstance, updateInstance, removeInstance, setViewTypeForPath

**`useTextEditStore`** (Instance-based documents):
- **State**: instances[instanceId] → { content, selection, undoStack, filePath }
- **Methods**: createInstance, updateContent, setSelection, save, load

**`useChatsStore`** (Messages + rooms):
- **State**: aiMessages[], rooms[], currentRoomId, authToken, username
- **Persistence**: localStorage + real-time sync via Pusher
- **Methods**: addMessage, addMessageToRoom, createRoom, deleteRoom, setAuthToken

**`useIpodStore`** (Media library):
- **State**: tracks[], playlists[], currentIndex, isPlaying, volume, lyrics
- **Methods**: addTrack, removeTrack, createPlaylist, setCurrentIndex, play, pause

**`usePaintStore`** (Canvas state):
- **State**: lastFilePath (for Save vs Save As)
- **Note**: Canvas content in-memory (not persisted), undo/redo in component

**`useSynthStore`** (Synthesizer parameters):
- **State**: presets[], oscillator1, oscillator2, envelope, effects, noteLabelType
- **Methods**: savePreset, loadPreset, deletePreset, updateParameter

**`useSoundboardStore`** (Audio boards):
- **State**: boards[], activeBoardId, playbackStates[], selectedDeviceId
- **Persistence**: IndexedDB (audio blobs), localStorage (metadata)
- **Methods**: addBoard, updateSlot, deleteSlot, setSlotPlaybackState

**`useIpodStore`** (Video player):
- **State**: playlist[], currentIndex, isPlaying, loopMode, shuffleMode
- **Methods**: addVideo, removeVideo, next, previous, togglePlay

**`useInternetExplorerStore`** (Browse history):
- **State**: history[], favorites[], currentUrl, currentYear, snapshots[]
- **Methods**: addToHistory, addFavorite, setSnapshot, clearHistory

**`useTerminalStore`** (Command state):
- **State**: commandHistory[], outputBuffer[], currentDirectory, environmentVars
- **Methods**: addCommand, addOutput, setCurrentDirectory, clearOutput

**`usePhotoBoothStore`** (Camera state):
- **State**: photos[], filters[], selectedFilter, selectedCamera
- **Persistence**: IndexedDB (photo blobs)
- **Methods**: addPhoto, deletePhoto, setFilter, setCamera

**`usePcStore`** (Emulator state):
- **State**: saveStates[], currentGame, isRunning
- **Persistence**: IndexedDB (save state blobs)
- **Methods**: saveState, loadState, setGame

---

### **External Library Integration Matrix**

| Library | Apps Using | Purpose | Integration Point |
|---------|-----------|---------|-------------------|
| **Tone.js** | Synth | Web Audio synthesis | `import * as Tone from "tone"` - Oscillators, effects, MIDI |
| **Tiptap** | TextEdit | Rich text editing | Extensions: StarterKit, Underline, TaskList, SlashCommands |
| **WaveSurfer.js** | Soundboard | Waveform visualization | Live recording + playback waveforms |
| **React Player** | iPod, Videos | YouTube playback | Handles YouTube API, fullscreen, playback state |
| **Vercel AI SDK** | Chats, Terminal | AI streaming | useChat hook, tool calling, streaming responses |
| **Pusher** | Chats | Real-time messaging | Room message broadcasts, typing indicators |
| **js-dos** | Virtual PC | DOSBox emulation | Emulator initialization, save states, input handling |
| **Framer Motion** | Multiple | Animations | Window animations, dialog transitions, list animations |

---

### **Inter-App Communication Patterns**

**CustomEvent System** (global event bus):
```typescript
// App Launch
window.dispatchEvent(new CustomEvent("launchApp", {
  detail: { appId: "finder", initialData: { path: "/Documents" } }
}));

// File System Changes
window.dispatchEvent(new CustomEvent("fileUpdated", {
  detail: { name: "file.txt", path: "/Documents/file.txt" }
}));
window.dispatchEvent(new CustomEvent("fileRenamed", {
  detail: { oldPath: "/a.txt", newPath: "/b.txt", oldName: "a.txt", newName: "b.txt" }
}));

// Instance State Changes
window.dispatchEvent(new CustomEvent("instanceStateChange", {
  detail: { instanceId: "finder-1", isOpen: false, isForeground: false }
}));
```

**Store-Based Communication** (cross-app state access):
```typescript
// Terminal reads TextEdit state
const textEditInstances = useTextEditStore.getState().instances;
const foregroundDoc = Object.values(textEditInstances).find(inst => inst.isForeground);

// Chats controls other apps
const appStore = useAppStore.getState();
appStore.launchApp("internet-explorer", { url: "https://example.com" });
appStore.closeAppInstance(instanceId);
```

**Direct API Integration** (Chats → TextEdit editing):
```typescript
// From Chats app (Ryo AI)
const textEditStore = useTextEditStore.getState();
textEditStore.insertTextAtLine(instanceId, lineNumber, "Inserted by AI");
textEditStore.replaceTextAtLine(instanceId, lineNumber, "Replaced by AI");
textEditStore.deleteLines(instanceId, startLine, endLine);
```

---

### **App Initialization Flow**

**Simple Apps** (no initialData):
```
User launches Minesweeper
→ CustomEvent("launchApp", { appId: "minesweeper" })
→ AppManager.handleAppLaunch
→ useAppStore.launchApp("minesweeper")
→ createAppInstance("minesweeper")
→ MinesweeperAppComponent mounted
→ Local state initializes game board
```

**Apps with initialData** (Paint, iPod, Internet Explorer):
```
User double-clicks image in Finder
→ Finder.handleFileOpen(file)
→ CustomEvent("launchApp", {
    appId: "paint",
    initialData: { path: "/Images/pic.png", content: blobUrl }
  })
→ useAppStore.launchApp("paint", initialData)
→ createAppInstance → instances["paint-1"] = { ..., initialData }
→ PaintAppComponent receives initialData prop
→ useEffect: handleFileOpen(initialData.path, initialData.content)
→ Canvas loads image
→ clearInitialData("paint-1") - Prevent re-loading on re-render
```

**Multi-Instance Apps** (Finder, TextEdit):
```
User: File > New Finder Window
→ Finder.handleNewWindow()
→ launchApp("finder", { path: "/" }, undefined, true) // multiWindow = true
→ Check: finder supports multi-window
→ Create new instance: "finder-2"
→ Both finder-1 and finder-2 rendered independently
→ Each has own currentPath, history in useFinderStore.instances
```

---

### **Window Configuration (appRegistry.ts)**

**Default Constraints**:
```typescript
{
  defaultWidth: 730,
  defaultHeight: 475,
  minWidth: 400,
  minHeight: 300,
}
```

**App-Specific Overrides**:
| App | defaultWidth | defaultHeight | minWidth | minHeight | Notes |
|-----|--------------|---------------|----------|-----------|-------|
| Finder | - | - | 400 | 300 | - |
| TextEdit | 430 | 475 | - | - | Narrower for text editing |
| Virtual PC | - | - | 640 | 480 | Fixed aspect ratio |
| Minesweeper | Fixed | Fixed | - | - | Non-resizable |

---

### **Per-App Feature Matrix**

| App | Multi-Window | Store | External Lib | API | File I/O | Realtime |
|-----|-------------|-------|--------------|-----|----------|----------|
| Finder | ✅ | useFinderStore | - | - | ✅ | - |
| TextEdit | ✅ | useTextEditStore | Tiptap | - | ✅ | - |
| Paint | ❌ | usePaintStore | Canvas API | - | ✅ | - |
| Chats | ❌ | useChatsStore | Vercel AI SDK | /api/chat.ts | - | ✅ Pusher |
| Terminal | ❌ | useTerminalStore | - | /api/chat.ts | ✅ | - |
| Internet Explorer | ❌ | useInternetExplorerStore | - | Wayback API | - | - |
| iPod | ❌ | useIpodStore | React Player | /api/lyrics.ts | - | - |
| Videos | ❌ | useVideoStore | React Player | - | - | - |
| Synth | ❌ | useSynthStore | Tone.js | - | - | - |
| Soundboard | ❌ | useSoundboardStore | WaveSurfer | - | - | - |
| Photo Booth | ❌ | usePhotoBoothStore | WebRTC | - | - | - |
| Minesweeper | ❌ | - | - | - | - | - |
| Virtual PC | ❌ | usePcStore | js-dos | - | - | - |
| Control Panels | ❌ | Multiple | - | - | - | - |
| Applet Viewer | ✅ | useAppletStore | - | - | ✅ | - |

---
```

---

### 4. Data Layer (CLAUDE.md Lines 139-205)
**Current State**: useAppStore, useChatsStore, useFilesStore documented

**Gap**: Missing detailed documentation for 12 app-specific stores

**Recommendation**: Expand each store entry with:
1. State shape (key fields)
2. Persistence strategy (localStorage vs IndexedDB)
3. Key methods
4. Instance-based vs singleton pattern

**Example Enhancement** (useIpodStore):
```markdown
**`src/stores/useIpodStore.ts`** - Music Library State
- **Purpose**: Track management, playback state, lyrics, playlists
- **Persistence**: localStorage (tracks, playlists), state (playback)
- **Key State**:
  - `tracks: Track[]` - YouTube video metadata + lyrics
  - `playlists: Playlist[]` - User-created playlists
  - `currentIndex: number` - Currently playing track index
  - `isPlaying: boolean` - Playback state
  - `volume: number` - Volume level
  - `loopMode: "none" | "single" | "all"` - Repeat mode
  - `shuffleMode: boolean` - Shuffle state
- **Key Methods**:
  - `addTrack(videoId, title, thumbnail)` - Add YouTube video to library
  - `removeTrack(trackId)` - Delete track from library
  - `setCurrentIndex(index)` - Select track to play
  - `togglePlay()` - Play/pause current track
  - `next()` - Skip to next track (respects shuffle/loop)
  - `previous()` - Skip to previous track
  - `createPlaylist(name)` - Create new playlist
  - `addTrackToPlaylist(playlistId, trackId)` - Add track to playlist
- **Integration**: iPod app, Videos app (shared YouTube playback)
- **External API**: `/api/lyrics.ts` - Fetches LRC lyrics for current track
```

---

### 5. Infrastructure Layer (CLAUDE.md Lines 252-306)
**Current State**: Hooks and utilities documented

**Gap**: Missing app-specific custom hooks

**Proposed Addition**:
```markdown
### **App-Specific Hooks**

**Finder**:
- `useFileSystem(initialPath, { instanceId })` - File operations, navigation, trash
  - Returns: currentPath, files, selectedFile, navigateToPath, moveToTrash, etc.

**TextEdit**:
- `useTextEditState(instanceId)` - Document state for instance
- `useFileOperations(instanceId)` - Save, open, export logic
- `useDragAndDrop()` - File drop handling

**Chats**:
- `useAiChat()` - AI conversation management (Vercel AI SDK wrapper)
- `useChatRoom()` - Room management, Pusher integration
- `useRyoChat()` - @ryo mention detection in rooms

**Internet Explorer**:
- `useAiGeneration()` - Website reconstruction/imagination logic

**iPod**:
- `useLyrics(videoId)` - Lyrics fetching, parsing, synchronization
- `useLibraryUpdateChecker()` - Detects library changes for UI refresh

**Soundboard**:
- `useSoundboard()` - Board/slot management
- `useAudioRecorder()` - Recording functionality

**Virtual PC**:
- `useJsDos()` - js-dos emulator integration
```

---

### 6. Component Interdependency Map (CLAUDE.md Lines 309-399)
**Current State**: Hierarchy and impact radius documented

**Gap**: No application layer in hierarchy

**Proposed Addition** (insert at Line 329):
```markdown
**Level 4 - Application Layer (18 Apps)**
- Finder → Depends on: WindowFrame, useFinderStore, useFilesStore, useFileSystem hook
- TextEdit → Depends on: WindowFrame, useTextEditStore, Tiptap, useFileSystem hook
- Chats → Depends on: WindowFrame, useChatsStore, Vercel AI SDK, Pusher, useAiChat/useChatRoom hooks
- Terminal → Depends on: WindowFrame, useTerminalStore, useFileSystem, useAiChat, command modules
- Paint → Depends on: WindowFrame, usePaintStore, Canvas API, useFileSystem
- Internet Explorer → Depends on: WindowFrame, useInternetExplorerStore, useAiGeneration, Wayback API
- iPod → Depends on: WindowFrame, useIpodStore, React Player, useLyrics hook
- Videos → Depends on: WindowFrame, useVideoStore, React Player
- Synth → Depends on: WindowFrame, useSynthStore, Tone.js
- Soundboard → Depends on: WindowFrame, useSoundboardStore, WaveSurfer.js, useSoundboard hook
- Photo Booth → Depends on: WindowFrame, usePhotoBoothStore, WebRTC
- Minesweeper → Depends on: WindowFrame (no store)
- Virtual PC → Depends on: WindowFrame, usePcStore, js-dos, useJsDos hook
- Control Panels → Depends on: WindowFrame, multiple stores (Theme, App, Files, etc.)
- Applet Viewer → Depends on: WindowFrame, useAppletStore, HtmlPreview
```

---

### 7. Testing & Quality Assurance (Not in CLAUDE.md)
**Gap**: No testing section exists

**Proposed New Section**:
```markdown
---

## 🧪 Testing Strategy & Quality Assurance

### **Current State**
- ❌ No unit tests for app components
- ❌ No integration tests for app workflows
- ❌ No E2E tests for critical user paths
- ⚠️ Manual testing only

### **Recommended Testing Infrastructure**

**Unit Tests** (Vitest + React Testing Library):
```typescript
// Example: Finder file operations
describe("Finder - File Operations", () => {
  it("should create folder with unique name", () => {
    const { result } = renderHook(() => useFileSystem("/"));
    act(() => {
      result.current.createFolder({ path: "/untitled folder", name: "untitled folder" });
    });
    expect(useFilesStore.getState().getItem("/untitled folder")).toBeDefined();
  });
});
```

**Integration Tests** (Playwright Component Testing):
```typescript
// Example: TextEdit save flow
test("should save document to IndexedDB", async ({ mount }) => {
  const component = await mount(<TextEditAppComponent ... />);
  await component.locator("textarea").fill("Test content");
  await component.locator('button:has-text("Save")').click();
  const files = await dbOperations.getAll(STORES.DOCUMENTS);
  expect(files).toContainEqual(expect.objectContaining({ content: "Test content" }));
});
```

**E2E Tests** (Playwright):
```typescript
// Example: App launch → file open → edit → save
test("complete file editing workflow", async ({ page }) => {
  await page.goto("/");
  await page.click('[data-app="finder"]'); // Launch Finder
  await page.dblclick('[data-file="document.txt"]'); // Open file
  await page.fill('textarea', "Edited content"); // Edit in TextEdit
  await page.click('button:has-text("Save")'); // Save
  await page.reload(); // Refresh
  await page.click('[data-app="finder"]'); // Re-open Finder
  await page.dblclick('[data-file="document.txt"]'); // Re-open file
  await expect(page.locator('textarea')).toHaveValue("Edited content"); // Verify persistence
});
```

### **Priority Testing Targets**

**Critical Paths** (must have E2E tests):
1. App launch → instance creation → window rendering
2. Finder → file operations (create, rename, delete, move)
3. TextEdit → file save/load → persistence
4. Chats → AI message → streaming response → TTS playback
5. Terminal → command execution → output rendering
6. Multi-window → instance isolation → state independence

**High-Risk Areas** (unit test coverage):
1. useAppStore → createAppInstance, launchApp, bringInstanceToForeground
2. useFilesStore → file tree mutations, path resolution
3. WindowFrame → resize/drag calculations, constraint enforcement
4. All stores → persistence migrations, rehydration logic
```

---

## Summary of Gaps

### Missing in CLAUDE.md
1. **Application Layer Documentation** (Lines 329+)
   - Per-app feature catalogs
   - External library integration
   - InitialData interfaces
   - Multi-window support matrix

2. **App-Specific Data Flows** (Lines 19-60)
   - Lyrics sync, chat rooms, terminal AI, time travel browsing

3. **Store Implementation Details** (Lines 139-205)
   - State shapes for 12 app-specific stores
   - Persistence strategies (localStorage vs IndexedDB)
   - Instance-based vs singleton patterns

4. **Custom Hooks Documentation** (Lines 252-306)
   - App-specific hooks (useFileSystem, useAiChat, useRyoChat, etc.)

5. **Testing Strategy Section** (No current section)
   - Unit/integration/E2E test recommendations
   - Critical path identification

### Redundancies to Merge
- App listings (CLAUDE.md Line 12 + 04_applications_analysis.md Executive Summary)
- Store listings (CLAUDE.md Lines 191-205 + 04_applications_analysis.md store sections)

---

## Recommended CLAUDE.md Enhancements

### Immediate Additions
1. **Insert Application Layer Section** after Line 329
   - Use content from "Proposed New Section" above
   - Include app registry table, feature matrix, external library matrix

2. **Expand Data Flow Section** (Lines 19-60)
   - Add app-specific flows (lyrics, chat rooms, terminal AI, time travel)

3. **Enhance Store Documentation** (Lines 139-205)
   - Add state shapes, methods, persistence details for all 12 app stores

4. **Add Testing Section** (new section)
   - Unit/integration/E2E testing strategy
   - Priority targets, example tests

### Future Enhancements
1. **API Endpoint Documentation**
   - /api/chat.ts (streaming AI)
   - /api/lyrics.ts (LRC lyrics fetching)
   - /api/internet-explorer/generate (website reconstruction)
   - /api/speech.ts (TTS generation)
   - /api/transcribe.ts (speech-to-text)

2. **Deployment Documentation**
   - Vercel serverless functions
   - Environment variables
   - Database schema (Pusher, chat rooms)

3. **Performance Optimization**
   - React profiling results
   - Memoization strategies
   - Store subscription patterns (useShallow)

---

## Conclusion

**04_applications_analysis.md** provides comprehensive app-level documentation missing from CLAUDE.md. Key additions include:

- **18 detailed app profiles** with feature catalogs, stores, external libraries
- **Inter-app communication patterns** (CustomEvents, store access, direct APIs)
- **Multi-window support documentation** (3 apps: Finder, TextEdit, Applet Viewer)
- **External library integration matrix** (8 major libraries mapped to apps)
- **App initialization flows** (simple vs initialData vs multi-instance)

**Recommendation**: Merge app-level details into CLAUDE.md as new "Application Layer" section after Line 329, expanding existing store/hook sections with implementation details.

**Impact**: Provides developers complete reference for:
1. Adding new applications following established patterns
2. Understanding app dependencies and integration points
3. Debugging app-specific issues (store state, hooks, data flows)
4. Planning feature enhancements (multi-window support, external API integration)

**Maintenance**: Keep both documents in sync:
- CLAUDE.md - Architectural overview + app layer
- 04_applications_analysis.md - Detailed per-app reference (can be more verbose)
