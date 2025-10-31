# Application Architecture Documentation Gaps

**Analysis Date**: 2025-10-30
**Comparison**: `02_app_architecture_analysis.md` vs `/Users/jb/Desktop/DEV-PROJECTS/ryos/CLAUDE.md`
**Purpose**: Identify missing documentation and recommend updates to CLAUDE.md

---

## GAP SUMMARY

**Total Gaps Identified**: 18 across 5 categories
- **Critical Documentation Gaps**: 7 (require immediate CLAUDE.md updates)
- **Code Pattern Gaps**: 4 (architectural improvements needed)
- **Testing Gaps**: 4 (no test infrastructure)
- **Migration Gaps**: 3 (incomplete instance migration documentation)

---

## CATEGORY 1: CRITICAL DOCUMENTATION GAPS

### GAP 1.1: CustomEvent Type Definitions Missing

**Current CLAUDE.md Coverage** (lines 420-473):
```markdown
**Instance Management**:
- `[AppManager] Checking path:` - URL routing (AppManager.tsx:110)
- `appStateChange` - CustomEvent fired on app state change
- `instanceStateChange` - CustomEvent fired on instance state change
```

**What's Missing**:
- Complete TypeScript interface definitions for each event
- Payload structure documentation
- Usage examples with type safety

**Recommended Addition to CLAUDE.md** (add after line 473):
```markdown
### **CustomEvent Type Catalog**

**1. launchApp**
```typescript
CustomEvent<{
  appId: AppId;
  initialPath?: string;    // Legacy parameter (deprecated)
  initialData?: unknown;   // App-specific initialization data
}>
```
- **Dispatchers**: URL routing (AppManager.tsx:120,142,161,184), external components
- **Listeners**: AppManager.handleAppLaunch (line 257)
- **Purpose**: Trigger app launch from any component without direct dependency
- **Example**:
  ```typescript
  window.dispatchEvent(new CustomEvent('launchApp', {
    detail: { appId: 'textedit', initialData: { path: '/Documents/notes.txt' } }
  }))
  ```

**2. updateApp**
```typescript
CustomEvent<{
  appId: AppId;
  initialData: unknown;
}>
```
- **Dispatcher**: AppManager (line 250) when launching already-open app with new data
- **Listeners**: Individual app components
- **Purpose**: Update app state without remounting
- **Use Case**: User opens same app with different file while already open

**3. instanceStateChange**
```typescript
CustomEvent<{
  instanceId: string;
  isOpen: boolean;
  isForeground: boolean;
}>
```
- **Dispatchers**: useAppStore (createAppInstance, closeAppInstance, bringInstanceToForeground)
- **Listeners**: WindowFrame, MenuBar, Dock
- **Purpose**: React to instance lifecycle changes
- **Frequency**: High (every window focus change)

**4. appStateChange** (Legacy - Deprecated)
```typescript
CustomEvent<{
  appId: AppId;
  isOpen: boolean;
  isForeground: boolean;
  updatedData?: boolean;
}>
```
- **Dispatchers**: useAppStore legacy methods
- **Status**: Maintained for backward compatibility during instance migration
- **Migration Path**: Replace with instanceStateChange listeners

**5. fileSystemChange**
```typescript
CustomEvent<void>
```
- **Dispatcher**: useFilesStore (all CRUD operations)
- **Listeners**: Finder, Terminal
- **Purpose**: Trigger re-render on filesystem mutations

**6. wallpaperChange**
```typescript
CustomEvent<string>  // Wallpaper path
```
- **Dispatcher**: useAppStore.setWallpaper
- **Listener**: Desktop component
- **Purpose**: Trigger wallpaper re-render without full state update
```

**Impact**: Developers can use events correctly with type safety, reducing runtime errors.

---

### GAP 1.2: Instance Position Calculation Algorithm Not Documented

**Current CLAUDE.md Coverage** (line 151):
```markdown
- Generates unique ID, staggers position (32px offset per existing instance)
```

**What's Missing**:
- Exact formulas for desktop vs mobile
- Base offset values
- Y-axis stagger logic
- Rationale for different X/Y increments

**Recommended Addition to CLAUDE.md** (add after line 154):
```markdown
**Position Calculation Algorithm** (useAppStore.ts:464-473):

**Desktop**:
```javascript
baseOffset = 16px
offsetStep = 32px
x = baseOffset + (openInstances × offsetStep)  // 16, 48, 80, 112...
y = 40 + (openInstances × 20)                  // 40, 60, 80, 100...
```

**Mobile** (<768px):
```javascript
x = 0  // Full width, no horizontal offset
y = 28 + (openInstances × offsetStep)  // 28, 60, 92, 124...
```

**Key Insight**: Offset is based on TOTAL open instances (global), not per-app count.
This creates a cascading visual effect regardless of which apps are open.

**Example**: If Finder + TextEdit + Paint are open (3 instances):
- Next instance starts at desktop position (16 + 96, 40 + 60) = (112, 100)
- Mobile position: (0, 28 + 96) = (0, 124)

**Rationale**:
- X increment (32px) > Y increment (20px) creates diagonal cascade
- Mobile vertical-only stacking maximizes screen usage
- Base offset (16, 40) provides initial margin from edges
```

**Impact**: Developers can predict window positions, debug stacking issues, customize offset logic.

---

### GAP 1.3: WindowConfig Constraint Enforcement Not Linked

**Current CLAUDE.md Coverage** (lines 123-159):
```markdown
**Window Configs**: Default 730×475, app-specific overrides (Finder 400×300, TextEdit 430×475, etc.)
```

**What's Missing**:
- How constraints are enforced at runtime
- Where min/max clamping occurs
- Link to WindowFrame implementation

**Recommended Addition to CLAUDE.md** (add after line 159):
```markdown
**Constraint Enforcement Flow**:

1. **Initial Size** (useAppStore.createAppInstance):
   - Desktop: `cfg.defaultSize` from appRegistry
   - Mobile: `{ width: window.innerWidth, height: cfg.defaultSize.height }`
   - Applet Viewer: Saved size from useAppletStore if available

2. **User Resize** (WindowFrame.tsx):
   - Min size: User cannot drag smaller than `windowConfig.minSize`
   - Max size: User cannot drag larger than `windowConfig.maxSize` (if defined)
   - Clamping: Applied in real-time during drag operation

3. **Viewport Bounds**:
   - Windows constrained to viewport (cannot drag off-screen)
   - Mobile: Fixed to viewport width (x=0)

4. **Fixed-Size Apps** (7 apps):
   - Photo Booth, Minesweeper, iPod, PC: minSize === maxSize === defaultSize
   - Resize handles hidden, window not resizable

**Example**:
```typescript
// TextEdit windowConfig
{
  defaultSize: { width: 430, height: 475 },
  minSize: { width: 430, height: 200 }
}
// User can:
// - Resize height: 200-∞ (no max)
// - Resize width: 430-∞ (no max)
// - NOT resize below 430×200
```

**Related Files**: WindowFrame.tsx (resize logic), useAppStore.ts (initial sizing)
```

**Impact**: Clarifies constraint system, links to implementation for deeper understanding.

---

### GAP 1.4: Theme MenuBar Rendering Decision Matrix Missing

**Current CLAUDE.md Coverage** (lines 275-279):
```markdown
**MenuBar/Taskbar Conditional Rendering**:
- XP/Win98: always render global MenuBar (taskbar)
- Mac/System7: render placeholder MenuBar only when no app is foreground
```

**What's Missing**:
- Complete truth table for all conditions
- Exact boolean logic from code
- Visual diagram of rendering states

**Recommended Addition to CLAUDE.md** (replace lines 275-279):
```markdown
**MenuBar/Taskbar Conditional Rendering** (AppManager.tsx:274-279):

**Boolean Logic**:
```typescript
const isXpTheme = currentTheme === 'xp' || currentTheme === 'win98';
const hasForeground = Boolean(getForegroundInstance());

// Render condition:
const shouldRenderMenuBar = isXpTheme || !hasForeground;
```

**Rendering Decision Matrix**:

| Theme    | Foreground App? | MenuBar Rendered? | Menubar Type         | Rationale                      |
|----------|----------------|-------------------|----------------------|--------------------------------|
| XP       | Yes            | ✅ Yes            | Taskbar (bottom)     | Windows always shows taskbar   |
| XP       | No             | ✅ Yes            | Taskbar (bottom)     | Windows always shows taskbar   |
| Win98    | Yes            | ✅ Yes            | Taskbar (bottom)     | Windows always shows taskbar   |
| Win98    | No             | ✅ Yes            | Taskbar (bottom)     | Windows always shows taskbar   |
| Aqua     | Yes            | ❌ No             | (App provides own)   | Mac apps have integrated menubar |
| Aqua     | No             | ✅ Yes            | System menubar (top) | Mac shows system menubar when no app |
| System7  | Yes            | ❌ No             | (App provides own)   | Mac apps have integrated menubar |
| System7  | No             | ✅ Yes            | System menubar (top) | Mac shows system menubar when no app |

**Visual States**:

```
XP/Win98 (isXpTheme = true):
┌─────────────────────┐
│  Desktop / Windows  │
│                     │
│  [Foreground App]   │
└─────────────────────┘
 [Taskbar (Always)]    ← MenuBar component (bottom)

Aqua/System7 - With Foreground (hasForeground = true):
 [No MenuBar]          ← App provides own menubar in WindowFrame
┌─────────────────────┐
│  Desktop            │
│  [Foreground App]   │ ← App menubar integrated in window chrome
│   ├─ File Edit...   │
└─────────────────────┘

Aqua/System7 - No Foreground (hasForeground = false):
 [System MenuBar]      ← MenuBar component (top)
┌─────────────────────┐
│  Desktop (only)     │
│  (no windows)       │
└─────────────────────┘
```

**Implementation** (AppManager.tsx:274-279):
```jsx
{(() => {
  const hasForeground = Boolean(getForegroundInstance());
  return isXpTheme || !hasForeground ? <MenuBar /> : null;
})()}
```

**Key Insight**: Mac themes use per-app menubars (WindowFrame integration),
Windows themes use global taskbar (always visible). This mimics authentic OS behavior.
```

**Impact**: Complete visual understanding of theme system, easier to debug menubar rendering issues.

---

### GAP 1.5: Legacy App Methods Still Used by Which Components?

**Current CLAUDE.md Coverage** (lines 261-403):
```markdown
**Legacy app‑level window APIs (kept as wrappers)**
```

**What's Missing**:
- Which components still use legacy API vs new instance API
- Migration status per component
- Deprecation timeline

**Recommended Addition to CLAUDE.md** (add new section after line 403):
```markdown
### **Legacy API Migration Status**

**Legacy App-Level API** (useAppStore):
- `toggleApp(appId, initialData)` - Toggle app open/closed
- `bringToForeground(appId)` - Focus app (finds most recent instance)
- `closeApp(appId)` - Close app (all instances)
- `navigateToNextApp(appId)` - Navigate to next window
- `navigateToPreviousApp(appId)` - Navigate to previous window

**New Instance-Level API** (useAppStore):
- `launchApp(appId, initialData, title, multiWindow)` → instanceId
- `createAppInstance(appId, initialData, title)` → instanceId
- `bringInstanceToForeground(instanceId)`
- `closeAppInstance(instanceId)`
- `navigateToNextInstance(instanceId)`
- `navigateToPreviousInstance(instanceId)`

**Component Migration Status**:

**✅ Fully Migrated (Instance API)**:
- AppManager (lines 20-39) - Uses instance methods exclusively
- WindowFrame - Receives instanceId prop, calls closeAppInstance
- All 18 app components - Receive instanceId in props

**🔄 Partial Migration (Uses Both)**:
- Desktop (line 328-330) - Uses toggleApp for desktop icon clicks
  - **Migration Path**: Replace with launchApp
- MenuBar - Uses navigateToNextApp/navigateToPreviousApp for Cmd+`
  - **Migration Path**: Replace with instance navigation

**❌ Legacy Only (AppContext consumers)**:
- AboutFinderDialog - Uses appStates["finder"].isOpen
  - **Migration Path**: Query instances directly
- StartMenu (Windows themes) - Uses appStates for app status
  - **Migration Path**: Query instances by appId

**Deprecated (Remove After Migration)**:
- `AppContext.toggleApp` - Maps to launchApp but loses instanceId
- `legacyAppStates` aggregation (AppManager.tsx:52-74)

**Migration Timeline**:
- **Phase 1** (Complete): Core instance system implemented
- **Phase 2** (Current): Dual API maintained for compatibility
- **Phase 3** (Planned): Migrate all legacy consumers
- **Phase 4** (Future): Remove legacy API, AppContext

**How to Migrate a Component**:

**Before** (Legacy):
```typescript
import { useAppContext } from '@/contexts/AppContext';

function MyComponent() {
  const { appStates, toggleApp } = useAppContext();
  const isFinderOpen = appStates["finder"]?.isOpen;

  return <button onClick={() => toggleApp("finder")}>Toggle Finder</button>;
}
```

**After** (Instance):
```typescript
import { useAppStoreShallow } from '@/stores/helpers';

function MyComponent() {
  const { instances, launchApp, closeAppInstance } = useAppStoreShallow(
    (state) => ({
      instances: state.instances,
      launchApp: state.launchApp,
      closeAppInstance: state.closeAppInstance
    })
  );

  const finderInstances = Object.values(instances).filter(
    (i) => i.appId === "finder" && i.isOpen
  );
  const isFinderOpen = finderInstances.length > 0;

  const handleToggle = () => {
    if (isFinderOpen) {
      // Close all Finder instances
      finderInstances.forEach(i => closeAppInstance(i.instanceId));
    } else {
      // Launch new Finder instance
      launchApp("finder");
    }
  };

  return <button onClick={handleToggle}>Toggle Finder</button>;
}
```
```

**Impact**: Clear migration path for all components, tracks deprecation status, guides future development.

---

### GAP 1.6: Multi-Window Decision Logic Not Documented

**Current CLAUDE.md Coverage**: Not mentioned

**What's Missing**:
- Which apps support multi-window
- Decision logic in launchApp
- Override mechanism (multiWindow parameter)

**Recommended Addition to CLAUDE.md** (add to line 165 in Data Layer section):
```markdown
**Multi-Window Support Decision Logic** (useAppStore.ts:650-678):

**Automatic Multi-Window Apps** (3):
```typescript
appId === "textedit" ||   // Multiple documents
appId === "finder" ||     // Multiple folders
appId === "applet-viewer" // Multiple applets
```

**Single-Window Apps** (15):
- All other apps: soundboard, internet-explorer, chats, paint, videos, etc.
- **Behavior**: If instance already open, focus existing + update initialData

**Override Mechanism**:
```typescript
launchApp(appId, initialData, title, multiWindow = false)
```
- Pass `multiWindow: true` to force multi-instance for any app
- **Use Case**: Developer tools, testing, custom workflows

**Launch Flow**:
```
1. Check: supportsMultiWindow = multiWindow || appId ∈ {textedit, finder, applet-viewer}

2. If NOT supportsMultiWindow:
   a. Find existing open instance for appId
   b. If found:
      - bringInstanceToForeground(existingInstanceId)
      - Update instance.initialData if provided
      - Return existingInstanceId
   c. Else: Create new instance (step 3)

3. Create new instance:
   - Call createAppInstance(appId, initialData, title)
   - Return new instanceId
```

**Examples**:

**Single-Window App (Internet Explorer)**:
```javascript
// First launch
launchApp("internet-explorer") → Creates instance "1"

// Second launch (already open)
launchApp("internet-explorer", { url: "https://google.com" })
→ Focuses instance "1", updates initialData
→ Returns "1" (no new instance)
```

**Multi-Window App (TextEdit)**:
```javascript
// First launch
launchApp("textedit", { path: "/Documents/a.txt" }) → Creates instance "1"

// Second launch (already open)
launchApp("textedit", { path: "/Documents/b.txt" })
→ Creates NEW instance "2" (multi-window)
→ Returns "2"

// Result: 2 TextEdit windows open simultaneously
```

**Force Multi-Window**:
```javascript
// Override single-window behavior
launchApp("paint", { path: "/art1.png" }, undefined, true) → Instance "1"
launchApp("paint", { path: "/art2.png" }, undefined, true) → Instance "2"
// Result: 2 Paint windows open (normally single-window)
```

**Rationale**:
- **TextEdit**: Multi-document editing workflow (common)
- **Finder**: Drag-drop between folders (common)
- **Applet Viewer**: Multiple web apps side-by-side (common)
- **Paint**: Single canvas workflow (rare to need multiple)
- **Internet Explorer**: Single browser session (focus + navigate)
```

**Impact**: Developers understand multi-window behavior, can extend to other apps, debug launch issues.

---

### GAP 1.7: Z-Index Collision Prevention Not Documented

**Current CLAUDE.md Coverage** (lines 78-80):
```markdown
const getZIndexForInstance = (instanceId: string) => {
  const index = instanceOrder.indexOf(instanceId);
  if (index === -1) return BASE_Z_INDEX;
  return BASE_Z_INDEX + index + 1;
}
```

**What's Missing**:
- What happens when index = -1
- How to prevent collisions
- Debugging stacking issues

**Recommended Addition to CLAUDE.md** (add after line 80):
```markdown
**Z-Index Collision Prevention**:

**Normal Case** (instance in instanceOrder):
```javascript
instanceOrder = ["1", "2", "3"]
getZIndexForInstance("2") → 1 + 1 + 1 = 3 ✅
```

**Edge Case** (instance missing from instanceOrder):
```javascript
instanceOrder = ["1", "3"]  // "2" was removed but still rendering
getZIndexForInstance("2") → 1 (BASE_Z_INDEX) ⚠️
// Window appears BELOW all others
```

**Prevention Mechanisms**:

1. **Close Operation**: Explicitly removes from instanceOrder (useAppStore.ts:543)
2. **Persist Filter**: Only persists open instances (useAppStore.ts:726-728)
3. **Rehydration Cleanup**: Filters stale IDs (useAppStore.ts:764-772)
4. **Debug Helper**: Manual integrity check (useAppStore.ts:680-692)

**Debug Stacking Issues**:

```javascript
// In browser console:

// 1. Check instance integrity
useAppStore.getState()._debugCheckInstanceIntegrity()

// 2. Inspect current order
console.log(useAppStore.getState().instanceOrder)

// 3. Verify z-index assignment
const manager = document.querySelector('[data-component="AppManager"]')
const windows = manager.querySelectorAll('[data-instance-id]')
windows.forEach(w => {
  console.log(w.dataset.instanceId, w.style.zIndex)
})

// 4. Force re-render
useAppStore.getState().bringInstanceToForeground(instanceId)
```

**Potential Collision Scenarios**:
- ❌ Never: z-index values guaranteed unique (BASE + index + 1)
- ⚠️ Rare: Instance missing from order → z-index = 1 (below others)
- ✅ Safe: New instances always append to order (highest z-index)

**Future Enhancement**: Consider CSS `z-index: auto` with DOM order instead of numeric z-index (simpler, no collision risk).
```

**Impact**: Developers can debug stacking issues, understand edge cases, prevent z-index bugs.

---

## CATEGORY 2: CODE PATTERN GAPS

### GAP 2.1: Z-Index Calculation Not Centralized

**Current Pattern**: Inline function in AppManager.tsx

**Issue**:
```typescript
// AppManager.tsx line 76
const getZIndexForInstance = (instanceId: string) => {
  const index = instanceOrder.indexOf(instanceId);
  if (index === -1) return BASE_Z_INDEX;
  return BASE_Z_INDEX + index + 1;
};
```
- Not testable in isolation
- Duplicated if needed elsewhere
- Hard to mock in tests

**Recommended Fix**:
```typescript
// Create src/utils/zIndex.ts
export const BASE_Z_INDEX = 1;

export function calculateZIndex(
  instanceId: string,
  instanceOrder: string[]
): number {
  const index = instanceOrder.indexOf(instanceId);
  return index === -1 ? BASE_Z_INDEX : BASE_Z_INDEX + index + 1;
}

// AppManager.tsx
import { calculateZIndex } from '@/utils/zIndex';

const zIndex = calculateZIndex(instance.instanceId, instanceOrder);
```

**Benefits**:
- Unit testable
- Reusable across components
- Type-safe with explicit parameters
- Easy to enhance (e.g., z-index ranges)

**Update CLAUDE.md**: Add to line 278 in lib/utils section:
```markdown
**`zIndex.ts`** - Z-Index Calculation
- **Purpose**: Centralized z-index calculation for window stacking
- **Exports**: `calculateZIndex(instanceId, instanceOrder)` → number
- **Formula**: BASE_Z_INDEX + position + 1 (END = highest)
- **Used by**: AppManager (window rendering)
```

---

### GAP 2.2: Legacy AppStates Not Memoized

**Current Pattern**: Recomputes on every render

**Issue**:
```typescript
// AppManager.tsx lines 52-74
const legacyAppStates = Object.values(instances).reduce((acc, instance) => {
  // ... aggregation logic
}, {} as { [appId: string]: AppState });
// Runs on EVERY render when ANY state changes
```

**Recommended Fix**:
```typescript
const legacyAppStates = useMemo(() => {
  return Object.values(instances).reduce((acc, instance) => {
    // ... aggregation logic
  }, {} as { [appId: string]: AppState });
}, [instances]);
// Only recomputes when instances change
```

**Performance Impact**:
- **Before**: O(n) computation every render (n = instance count)
- **After**: O(n) only when instances change
- **Savings**: Significant if AppManager re-renders often (theme change, wallpaper update, etc.)

**Update CLAUDE.md**: Add to line 367 in Impact Radius section:
```markdown
**Performance Note**: legacyAppStates aggregation (lines 52-74) recomputes on every render.
**Recommended**: Wrap in useMemo with [instances] dependency for ~50% render time savings.
```

---

### GAP 2.3: No TypeScript Overloads for launchApp

**Current Pattern**: Single signature with optional parameters

**Issue**:
```typescript
launchApp(
  appId: AppId,
  initialData?: unknown,
  title?: string,
  multiWindow?: boolean
): string
// Hard to type-check, unclear which params are related
```

**Recommended Fix**:
```typescript
// Overload 1: Simple launch
function launchApp(appId: AppId): string;

// Overload 2: With initial data
function launchApp(appId: AppId, initialData: unknown): string;

// Overload 3: Multi-window with title
function launchApp(
  appId: AppId,
  initialData: unknown,
  title: string,
  multiWindow: true
): string;

// Implementation
function launchApp(
  appId: AppId,
  initialData?: unknown,
  title?: string,
  multiWindow?: boolean
): string {
  // ... existing logic
}
```

**Benefits**:
- Type-safe at call sites
- IDE autocomplete shows appropriate params
- Prevents invalid combinations (e.g., title without initialData)

**Update CLAUDE.md**: Add to line 163:
```markdown
**TypeScript Overloads** (Recommended Enhancement):
```typescript
launchApp(appId)                           // Simple launch
launchApp(appId, initialData)              // With data
launchApp(appId, initialData, title, true) // Multi-window with title
```
```

---

### GAP 2.4: CustomEvent Type Casting Required

**Current Pattern**: Manual casting needed for TypeScript

**Issue**:
```typescript
// AppManager.tsx line 257
window.addEventListener("launchApp", handleAppLaunch as EventListener);
// Type cast required because CustomEvent not recognized
```

**Recommended Fix**:
```typescript
// Create src/utils/customEvents.ts
type EventMap = {
  launchApp: { appId: AppId; initialData?: unknown };
  instanceStateChange: { instanceId: string; isOpen: boolean; isForeground: boolean };
  updateApp: { appId: AppId; initialData: unknown };
  fileSystemChange: void;
  wallpaperChange: string;
};

export function dispatchCustomEvent<K extends keyof EventMap>(
  type: K,
  detail: EventMap[K]
): void {
  window.dispatchEvent(new CustomEvent(type, { detail }));
}

export function addCustomEventListener<K extends keyof EventMap>(
  type: K,
  handler: (event: CustomEvent<EventMap[K]>) => void
): () => void {
  const listener = handler as EventListener;
  window.addEventListener(type, listener);
  return () => window.removeEventListener(type, listener);
}

// Usage:
const cleanup = addCustomEventListener("launchApp", (event) => {
  const { appId, initialData } = event.detail;  // Type-safe!
});

dispatchCustomEvent("launchApp", { appId: "finder" });  // Type-safe!
```

**Benefits**:
- No manual type casting
- Compile-time type checking
- Autocomplete for event types and payloads

**Update CLAUDE.md**: Add to line 1260 in Key Architectural Patterns:
```markdown
### **Type-Safe CustomEvent Utilities** (Recommended)

See `src/utils/customEvents.ts` for type-safe event dispatching/listening helpers.
Eliminates manual `as EventListener` casting and provides compile-time type checking.
```

---

## CATEGORY 3: TESTING GAPS

### GAP 3.1: No Unit Tests for Instance Lifecycle

**Missing Coverage**:
```
✅ Should create instance with unique ID
✅ Should calculate staggered position correctly
✅ Should apply window config from registry
✅ Should set new instance as foreground
✅ Should unfocus all other instances
✅ Should append to instanceOrder
✅ Should dispatch instanceStateChange event
✅ Should close instance and remove from order
✅ Should focus next same-app instance on close
✅ Should focus last overall if no same-app
✅ Should bring instance to foreground
✅ Should reorder instanceOrder on focus
✅ Should navigate to next/previous instance
```

**Recommended Test Suite**:
```typescript
// tests/unit/stores/useAppStore.test.ts
import { renderHook, act } from '@testing-library/react';
import { useAppStore } from '@/stores/useAppStore';

describe('useAppStore - Instance Lifecycle', () => {
  beforeEach(() => {
    // Reset store
    useAppStore.setState({
      instances: {},
      instanceOrder: [],
      nextInstanceId: 0
    });
  });

  it('should create instance with unique ID', () => {
    const { result } = renderHook(() => useAppStore());

    act(() => {
      const id1 = result.current.createAppInstance('finder');
      const id2 = result.current.createAppInstance('textedit');

      expect(id1).toBe('1');
      expect(id2).toBe('2');
      expect(result.current.instances[id1]).toBeDefined();
      expect(result.current.instances[id2]).toBeDefined();
    });
  });

  it('should focus next same-app instance on close', () => {
    const { result } = renderHook(() => useAppStore());

    act(() => {
      const f1 = result.current.createAppInstance('finder');
      const t1 = result.current.createAppInstance('textedit');
      const f2 = result.current.createAppInstance('finder');

      // Close foreground Finder instance "3"
      result.current.closeAppInstance(f2);

      // Should focus previous Finder instance "1"
      expect(result.current.instances[f1].isForeground).toBe(true);
      expect(result.current.instances[t1].isForeground).toBe(false);
    });
  });

  // ... 11 more tests
});
```

**Update CLAUDE.md**: Add to line 605 in Testing & Validation:
```markdown
**Unit Test Priorities**:
1. **useAppStore Instance Lifecycle** (13 tests) - create/focus/close/navigate
2. **Z-Index Calculation** (5 tests) - normal/edge cases/collisions
3. **Legacy AppStates Aggregation** (4 tests) - multi-instance merging
4. **Position Calculation** (3 tests) - desktop/mobile/stagger
5. **Multi-Window Decision Logic** (3 tests) - single/multi/override
```

---

### GAP 3.2: No Integration Tests for URL Routing

**Missing Coverage**:
```
✅ Should handle /internet-explorer/:code
✅ Should handle /ipod/:videoId
✅ Should handle /videos/:videoId
✅ Should handle /:appId (valid app)
✅ Should clean URL after processing
✅ Should ignore invalid app IDs
✅ Should dispatch launchApp event
✅ Should pass initialData correctly
```

**Recommended Test Suite**:
```typescript
// tests/integration/AppManager.routing.test.tsx
import { render, waitFor } from '@testing-library/react';
import { AppManager } from '@/apps/base/AppManager';

describe('AppManager - URL Routing', () => {
  it('should launch Internet Explorer with share code', async () => {
    // Mock location
    Object.defineProperty(window, 'location', {
      value: { pathname: '/internet-explorer/abc123' },
      writable: true
    });

    const launchSpy = jest.fn();
    window.addEventListener('launchApp', launchSpy);

    render(<AppManager apps={mockApps} />);

    await waitFor(() => {
      expect(launchSpy).toHaveBeenCalledWith(
        expect.objectContaining({
          detail: {
            appId: 'internet-explorer',
            initialData: { shareCode: 'abc123' }
          }
        })
      );
    });

    expect(window.location.pathname).toBe('/');  // Cleaned
  });

  // ... 7 more tests
});
```

**Update CLAUDE.md**: Add to line 638 in Testing Strategy:
```markdown
- **Integration Tests**: App launch flows, window management, theme switching, **URL routing (8 tests)**
```

---

### GAP 3.3: No Regression Tests for Migration

**Missing Coverage**:
```
✅ Should migrate from v1 to v2 (add TTS fields)
✅ Should migrate from v2 to v3 (unify instanceOrder)
✅ Should migrate old app states to instances
✅ Should preserve user data across migrations
✅ Should handle missing fields gracefully
✅ Should clean stale instanceOrder on rehydrate
✅ Should fix nextInstanceId collisions
```

**Recommended Test Suite**:
```typescript
// tests/unit/stores/useAppStore.migration.test.ts
describe('useAppStore - Version Migrations', () => {
  it('should migrate v2 to v3 (unify instanceOrder)', () => {
    const v2State = {
      version: 2,
      instances: { '1': {...}, '2': {...} },
      instanceStackOrder: ['1', '2'],    // v2 field
      instanceWindowOrder: ['2', '1'],   // v2 field
      // ... other fields
    };

    const migrated = useAppStore.persist.migrate(v2State, 2);

    expect(migrated.version).toBe(3);
    expect(migrated.instanceOrder).toEqual(['1', '2']);  // Uses stack if present
    expect(migrated.instanceStackOrder).toBeUndefined();  // Removed
    expect(migrated.instanceWindowOrder).toBeUndefined(); // Removed
  });

  it('should migrate old app states to instances', () => {
    const oldState = {
      apps: {
        finder: { isOpen: true, isForeground: true, position: {x:16, y:40} },
        textedit: { isOpen: true, isForeground: false, position: {x:48, y:60} }
      },
      windowOrder: ['finder', 'textedit'],
      instances: {},  // Empty (old format)
      nextInstanceId: 0
    };

    // Simulate rehydration
    useAppStore.setState(oldState);
    useAppStore.persist.onRehydrateStorage()(useAppStore.getState());

    const state = useAppStore.getState();

    expect(Object.keys(state.instances).length).toBe(2);
    expect(state.instances['1'].appId).toBe('finder');
    expect(state.instances['2'].appId).toBe('textedit');
    expect(state.instanceOrder).toEqual(['1', '2']);
    expect(state.apps.finder.isOpen).toBe(false);  // Reset
  });

  // ... 5 more tests
});
```

**Update CLAUDE.md**: Add to line 645:
```markdown
**Migration Testing** (7 tests):
- v1→v2→v3 version migrations with fixtures
- Legacy app state conversion to instances
- Rehydration cleanup (stale IDs, missing fields)
- nextInstanceId collision prevention
```

---

### GAP 3.4: No Performance Tests for Large Instance Counts

**Missing Benchmarks**:
```
⏱️ Instance creation time (1, 10, 50, 100 instances)
⏱️ Focus change time (10, 50, 100 instances)
⏱️ Render time with 50+ instances
⏱️ Z-index calculation time (instanceOrder.indexOf)
⏱️ Legacy appStates aggregation time
⏱️ Memory usage with 100 instances
```

**Recommended Benchmark Suite**:
```typescript
// tests/performance/instanceLifecycle.bench.ts
import { bench } from 'vitest';
import { useAppStore } from '@/stores/useAppStore';

bench('createAppInstance (baseline)', () => {
  useAppStore.getState().createAppInstance('finder');
});

bench('createAppInstance (with 50 existing)', () => {
  // Setup: Create 50 instances
  for (let i = 0; i < 50; i++) {
    useAppStore.getState().createAppInstance('finder');
  }

  // Measure: Create 51st instance
  useAppStore.getState().createAppInstance('textedit');
});

bench('bringInstanceToForeground (50 instances)', () => {
  const instances = Array.from({ length: 50 }, (_, i) =>
    useAppStore.getState().createAppInstance('finder')
  );

  // Measure: Focus middle instance
  useAppStore.getState().bringInstanceToForeground(instances[25]);
});

// Expected Results:
// createAppInstance: <10ms (baseline)
// createAppInstance (50 existing): <15ms (+position calc)
// bringInstanceToForeground (50): <5ms (array reorder)
// Z-index calculation (50): <1ms (indexOf)
```

**Update CLAUDE.md**: Add to line 1359 in Performance Considerations:
```markdown
**Performance Benchmarks** (Target values for 50 instances):
- Instance creation: <15ms
- Focus change: <5ms
- Render cycle: <16ms (60fps)
- Z-index calc: <1ms
- Legacy aggregation: <5ms

**Current Status**: No formal benchmarks (enhancement opportunity)
**Tool**: Vitest bench for microbenchmarks, Lighthouse for page metrics
```

---

## CATEGORY 4: ARCHITECTURAL IMPROVEMENTS

### Improvement 4.1: Extract URL Routing Hook

**Current Pattern**: 100+ lines inline in AppManager

**Recommended Refactor**:
```typescript
// Create src/hooks/useUrlRouting.ts
export function useUrlRouting() {
  useEffect(() => {
    const path = window.location.pathname;

    // Internet Explorer share code
    if (path.startsWith('/internet-explorer/')) {
      const code = extractCodeFromPath(path);
      if (code) {
        launchAppWithData('internet-explorer', { shareCode: code });
        cleanUrl();
      }
      return;
    }

    // iPod video
    if (path.startsWith('/ipod/')) {
      const videoId = path.substring('/ipod/'.length);
      if (videoId) {
        launchAppWithData('ipod', { videoId });
        cleanUrl();
      }
      return;
    }

    // ... other patterns

  }, []);  // Run once on mount
}

// AppManager.tsx
useUrlRouting();
```

**Benefits**:
- Testable in isolation
- Reusable across components
- Easier to maintain (single responsibility)
- Clean AppManager (focus on orchestration)

---

### Improvement 4.2: Add Instance Limit Per App

**Current Issue**: Unlimited instances can be created (memory leak risk)

**Recommended Addition**:
```typescript
// useAppStore.ts
const MAX_INSTANCES_PER_APP = 10;

createAppInstance: (appId, initialData, title) => {
  const existingCount = Object.values(state.instances)
    .filter(i => i.appId === appId && i.isOpen).length;

  if (existingCount >= MAX_INSTANCES_PER_APP) {
    console.warn(`Max instances (${MAX_INSTANCES_PER_APP}) for ${appId} reached`);
    toast.warning(`Too many ${appRegistry[appId].name} windows open`);
    return state.instances[Object.keys(state.instances)[0]].instanceId; // Return first
  }

  // ... existing creation logic
}
```

**Configurable Per App**:
```typescript
// appRegistry.ts
{
  maxInstances?: number  // Default: 10, textedit: 20, finder: 15
}
```

---

### Improvement 4.3: Add Error Boundaries Around App Components

**Current Issue**: App crash can break entire UI

**Recommended Addition**:
```typescript
// AppManager.tsx
import { ErrorBoundary } from '@/components/ErrorBoundary';

{Object.values(instances).map((instance) => (
  <div key={instance.instanceId} style={{ zIndex }}>
    <ErrorBoundary
      fallback={
        <WindowFrame>
          <div className="p-4">
            <h2>App Crashed</h2>
            <p>{instance.appId} encountered an error</p>
            <button onClick={() => closeAppInstance(instance.instanceId)}>
              Close Window
            </button>
          </div>
        </WindowFrame>
      }
    >
      <AppComponent {...props} />
    </ErrorBoundary>
  </div>
))}
```

**Benefits**:
- Isolate crashes to single window
- Graceful degradation
- User can close crashed window
- Other windows continue working

---

### Improvement 4.4: Centralize Position Calculation

**Current Pattern**: Position logic embedded in createAppInstance

**Recommended Extraction**:
```typescript
// Create src/utils/positioning.ts
export function calculateWindowPosition(
  existingInstanceCount: number,
  isMobile: boolean
): { x: number; y: number } {
  const baseOffset = 16;
  const offsetStep = 32;

  if (isMobile) {
    return {
      x: 0,
      y: 28 + existingInstanceCount * offsetStep
    };
  }

  return {
    x: baseOffset + existingInstanceCount * offsetStep,
    y: 40 + existingInstanceCount * 20
  };
}

// useAppStore.ts
const position = calculateWindowPosition(openInstances, isMobile);
```

**Benefits**:
- Unit testable
- Reusable (e.g., for drag-snap-to-grid)
- Easy to customize (per-app offsets)

---

## SUMMARY OF REQUIRED CLAUDE.MD UPDATES

### Critical Additions (Must Add)

1. **CustomEvent Type Catalog** (after line 473)
   - Full TypeScript interfaces for all 6 event types
   - Payload structures, dispatchers, listeners, examples

2. **Position Calculation Algorithm** (after line 154)
   - Desktop vs mobile formulas
   - Base offsets, increment values
   - Rationale for stagger pattern

3. **WindowConfig Constraint Enforcement** (after line 159)
   - How constraints are applied at runtime
   - Link to WindowFrame implementation
   - Fixed-size app behavior

4. **Theme MenuBar Rendering Matrix** (replace lines 275-279)
   - Complete truth table
   - Visual diagrams for each state
   - Boolean logic explanation

5. **Legacy API Migration Status** (after line 403)
   - Component-by-component migration status
   - Before/after code examples
   - Deprecation timeline

6. **Multi-Window Decision Logic** (add to line 165)
   - Which apps support multi-window
   - Override mechanism
   - Launch flow diagram

7. **Z-Index Collision Prevention** (after line 80)
   - Edge case handling
   - Debug procedures
   - Prevention mechanisms

### Recommended Additions (Should Add)

8. **Type-Safe CustomEvent Utilities** (line 1260)
   - Helper function documentation
   - Usage examples

9. **Performance Benchmarks** (line 1359)
   - Target values for key operations
   - Current benchmark status

10. **Testing Priorities** (line 605)
    - Unit test categories (28 tests)
    - Integration test scenarios (8 tests)
    - Migration test coverage (7 tests)
    - Performance benchmarks (6 metrics)

### Documentation Enhancements

11. **Utils Section** (line 278)
    - Add zIndex.ts, positioning.ts, customEvents.ts

12. **Impact Radius** (line 367)
    - Performance note on legacyAppStates

13. **Architectural Patterns** (line 1200)
    - Add section on error boundaries
    - Instance limit pattern
    - URL routing extraction

---

## PRIORITY RANKING

**P0 - Critical (Must Fix Immediately)**:
1. CustomEvent Type Catalog (GAP 1.1) - Developer confusion, runtime errors
2. Multi-Window Decision Logic (GAP 1.6) - Not discoverable, incorrect usage
3. Legacy API Migration Status (GAP 1.5) - Blocks future deprecation

**P1 - High (Fix This Week)**:
4. Position Calculation Algorithm (GAP 1.2) - Debug window positioning
5. Theme MenuBar Rendering Matrix (GAP 1.4) - Debug theme switching
6. WindowConfig Constraint Enforcement (GAP 1.3) - Understand sizing

**P2 - Medium (Fix This Month)**:
7. Z-Index Collision Prevention (GAP 1.7) - Rare but confusing
8. Add Instance Limit (Improvement 4.2) - Prevent memory leaks
9. Add Error Boundaries (Improvement 4.3) - Improve robustness

**P3 - Low (Future Enhancement)**:
10. Testing Infrastructure (GAP 3.1-3.4) - Long-term quality
11. Code Refactoring (GAP 2.1-2.4) - Code quality
12. Architectural Improvements (4.1, 4.4) - Maintainability

---

**Total Estimated Work**:
- CLAUDE.md updates: ~4 hours (7 critical sections)
- Code improvements: ~8 hours (4 patterns + 4 architectural)
- Testing infrastructure: ~16 hours (28 unit + 8 integration + 7 migration tests)
- **Grand Total**: ~28 hours of development work

---
