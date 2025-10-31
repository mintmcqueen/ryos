# Layout & UI System - Comprehensive Analysis

**Analysis Date**: 2025-01-21
**Scope**: Desktop environment, window management, theme system, UI components
**Purpose**: Deep code analysis for layout components and UI infrastructure

---

## Executive Summary

The ryOS layout system implements a sophisticated multi-theme desktop environment with four distinct OS experiences (macOS System 7, Mac OS X Aqua, Windows XP, Windows 98) using conditional rendering and theme-driven UI transformation. The system handles window management, drag/resize operations, responsive layouts, and safe area insets for mobile devices.

**Key Findings**:
- 6 core layout components orchestrate the desktop experience
- 30+ shadcn/ui components built on Radix UI primitives
- Theme-aware rendering affects layout structure (top menubar vs bottom taskbar)
- Complex z-index management via instanceOrder array
- Comprehensive window positioning with mobile/desktop responsiveness
- Advanced Framer Motion animations for dock magnification

---

## 1. Core Layout Components

### 1.1 Desktop.tsx (359 lines)

**Purpose**: Desktop viewport with wallpaper rendering, icon grid, and context menus

**Component Signature**:
```typescript
interface DesktopProps {
  apps: AnyApp[];
  appStates: AppManagerState;
  toggleApp: (appId: AppId, initialData?: unknown) => void;
  onClick?: () => void;
  desktopStyles?: DesktopStyles;
}
```

**State Management**:
- `selectedAppId: string | null` - Currently selected desktop icon
- `wallpaperSource: string` - From useWallpaper hook
- `isVideoWallpaper: boolean` - Derived from wallpaperSource extension
- `sortType: SortType` - Icon sorting ("name" | "kind")
- `contextMenuPos: {x, y} | null` - Right-click menu position
- `contextMenuAppId: string | null` - App for context menu

**Hooks Used**:
- `useWallpaper()` - Wallpaper state (source, isVideo)
- `useThemeStore((s) => s.current)` - Current theme
- `useLongPress()` - Mobile long-press for context menus

**Wallpaper Rendering Flow**:
1. `useWallpaper()` provides `wallpaperSource` and `isVideoWallpaper`
2. If video: `<video>` element with autoplay, loop, muted, playsInline
3. If image: CSS `backgroundImage` with conditional sizing (tiled vs cover)
4. Video playback management:
   - `useEffect` with visibility change listener (lines 64-113)
   - Resume playback on tab focus (lines 97-105)
   - Handle video ready state (lines 116-133)
   - Auto-restart when ended (lines 73-76)

**Icon Grid Layout**:
- **XP/Win98 themes** (lines 284-297):
  - `flex-col flex-wrap` with `justify-start content-start`
  - Top-to-bottom, left-to-right flow
  - Height calculation: `calc(100% - (30px + var(--sat-safe-area-bottom) + 48px))`
  - Excludes menubar, safe area, visual buffer
- **Mac themes** (lines 286-297):
  - `flex-col flex-wrap-reverse` (bottom-to-top)
  - Right alignment: `items-end`
  - Padding: `pt-8 h-[calc(100%-2rem)]`

**Icon Rendering**:
- "Macintosh HD" / "My Computer" always shown (lines 306-322)
- Apps filtered: excludes finder, control-panels, applet-viewer (lines 195-200)
- macOS X special case: Only iPod shown on desktop (lines 213-216)
- Uses `FileIcon` component with large size, selection state

**Context Menu**:
- Desktop context: "Sort By" submenu, "Set Wallpaper" (lines 230-253)
- Icon context: "Open" action (lines 221-227)
- Mobile: Long-press triggers context menu (lines 51-62)
- Desktop: Right-click triggers context menu (lines 260-264)

**Theme-Specific Logic**:
```typescript
const isXpTheme = currentTheme === "xp" || currentTheme === "win98";
const displayedApps = currentTheme === "macosx"
  ? sortedApps.filter((app) => app.id === "ipod")
  : sortedApps;
```

---

### 1.2 MenuBar.tsx (998 lines)

**Purpose**: Top menubar (Mac) or bottom taskbar (Windows) with app menus and system tray

**Component Signature**:
```typescript
interface MenuBarProps {
  children?: React.ReactNode;  // App-specific menus
  inWindowFrame?: boolean;      // Render inside window vs globally
}
```

**Conditional Rendering** (lines 649-996):
- **XP/Win98 themes**: Bottom taskbar with Start button, running apps, system tray
- **Mac themes**: Top menubar with Apple menu, app menus, volume/clock

**State Management**:
- `time: Date` - Clock state (updates every 1 second)
- `viewportWidth: number` - Responsive clock formatting
- `isStartMenuOpen: boolean` - Windows Start menu state
- `isHelpDialogOpen: boolean` - Help dialog state
- `isAboutDialogOpen: boolean` - About dialog state
- `visibleTaskbarIds: string[]` - Visible taskbar buttons (XP overflow)
- `overflowTaskbarIds: string[]` - Overflow taskbar buttons

**Hooks Used**:
- `useAppStoreShallow()` - Instance state, foreground instance
- `useThemeStore()` - Current theme
- `useFilesStore()` - For applet icons
- `useLaunchApp()` - App launching
- `useSound()` - Sound effects

**Clock Component** (lines 67-146):
- Updates every 1 second via `setInterval`
- Responsive formatting based on viewport width:
  - `< 420px`: Time only (no AM/PM)
  - `420-768px`: Time with AM/PM
  - `> 768px`: Full date and time
- XP themes: Always show AM/PM format
- macOS: Text shadow for depth

**Default Menu Items** (lines 148-443):
- **File Menu**: New Finder Window, New Folder, Close
- **Edit Menu**: Undo, Cut, Copy, Paste, Select All (all disabled)
- **View Menu**: Icon sizes, sort options (checkmarks for active)
- **Go Menu**: Quick access to folders (Applications, Documents, Images, Music, Sites, Videos, Trash)
- **Help Menu**: Finder Help, About Finder

**Volume Control** (lines 446-518):
- Dropdown with vertical slider (lines 493-501)
- Volume icons: VolumeX, Volume1, Volume2 based on level
- Master volume from useAppStore
- Sound effect on volume change (lines 500)
- Settings button to open Control Panels (lines 502-514)

**XP/Win98 Taskbar** (lines 649-962):
- **Start Button** (lines 674-677): StartMenu component
- **Running Apps Area** (lines 680-904):
  - Dynamic button sizing: `flex: 0 1 160px`, `minWidth: 110px`
  - Overflow detection via ResizeObserver (lines 588-630)
  - Overflow menu with ChevronUp icon (lines 793-904)
  - Foreground state: Different background gradient
  - Hover/active states with inline event handlers (lines 737-764)
- **System Tray** (lines 907-959):
  - Volume control (hidden on small screens)
  - Clock with bold font
  - Gradient background for XP, flat for Win98
  - Box shadow inset effects

**Applet Icon Handling** (lines 546-574):
- Extract icon from file system (via path in initialData)
- Emoji detection: `!startsWith("/") && !startsWith("http") && length <= 10`
- Fallback to "📦" if not emoji

**Taskbar Overflow Logic** (lines 576-631):
- Minimum button width: 110px + 2px gap
- Overflow button width: 40px
- Calculates visible vs overflow buttons based on container width
- ResizeObserver for responsive updates

**Mac Menubar** (lines 965-996):
- Apple menu (AppleMenu component)
- Dynamic app menus (children prop from foreground app)
- Volume control (hidden on small screens)
- Clock (always visible)
- Semi-transparent background with blur (macOS X)
- Pinstripe pattern background image

**Responsive Breakpoints**:
- Volume control: Hidden on `< sm` screens
- Clock formatting: Changes at 420px, 768px
- Taskbar buttons: Overflow based on container width

---

### 1.3 Dock.tsx (636 lines)

**Purpose**: Application launcher with running indicators and magnification effects

**Component Signature**: None (exported as `Dock()`)

**Rendering Condition**:
```typescript
if (currentTheme !== "macosx") return null;
```
Only renders for macOS X theme.

**State Management** (MacDock component):
- `seenIdsRef: Set<string>` - Track which icons appeared (for animations)
- `hasMounted: boolean` - Initial mount flag
- `magnifyEnabled: boolean` - Touch vs pointer detection
- `mouseX: MotionValue<number>` - Mouse position for magnification

**Hooks Used**:
- `useAppStoreShallow()` - Instances, instanceOrder, bringInstanceToForeground
- `useLaunchApp()` - App launching
- `useFilesStore()` - Trash icon, applet file info
- `useFinderStore()` - Finder instances for path detection
- `useIsPhone()` - Phone detection for overflow behavior

**Magnification System** (lines 222-284):
- **Mouse Position**: `mouseX` MotionValue tracks cursor X position
- **Distance Calculation**: `useTransform` computes distance from icon center
- **Size Transform**: `[-DISTANCE, 0, DISTANCE] → [baseSize, maxSize, baseSize]`
- **Spring Animation**: `mass: 0.15, stiffness: 160, damping: 18`
- **MAX_SCALE**: 2.3x at cursor center
- **DISTANCE**: 140px range
- **Disable on Touch**: Detects `(pointer: coarse)` or `(hover: none)` media queries

**Icon Organization**:
- **Pinned Left** (lines 80-83): `['finder', 'chats', 'internet-explorer']`
- **Open Apps** (lines 86-130): Dynamically added based on instances
- **Divider** (line 593): Visual separator before Applications/Trash
- **Applications** (lines 596-607): Quick access to /Applications folder
- **Trash** (lines 610-622): Always on right side

**Applet Instance Handling** (lines 46-77):
- Extract path from initialData
- Get file from filesystem
- Check if icon is emoji (same logic as MenuBar)
- Return icon and label for display

**Open Items Logic** (lines 86-130):
- Group instances by appId
- If applet-viewer: Add each instance separately (line 106-114)
- If regular app: Add single entry for all instances (line 116-122)
- Sort by creation time (line 126)
- Filter out pinned apps (line 129)

**IconButton Component** (lines 312-464):
- **Props**: label, onClick, icon, idKey, showIndicator, isEmoji
- **Animation**: Scale 0→1 for new icons, layout animations
- **Magnification**: Width/height driven by `sizeSpring` motion value
- **Emoji Scaling**: `useTransform` scales emoji relative to container
- **Indicator**: Black triangle below icon if app running (lines 444-458)

**Layout Animations** (Framer Motion):
- `<LayoutGroup>` wrapper for shared layout animations
- `<AnimatePresence mode="popLayout">` for smooth add/remove
- Individual icons: `layout`, `layoutId`, `initial`, `animate`, `exit`
- Smooth reordering when apps launch/close

**Overflow Handling** (isPhone):
- Enable horizontal scroll: `overflowX: "auto"`
- Touch scrolling: `WebkitOverflowScrolling: "touch"`
- Prevent overscroll bounce: `overscrollBehaviorX: "contain"`

**Focus/Launch Logic**:
- **focusOrLaunchApp** (lines 153-168): Focus existing OR launch new
- **focusOrLaunchFinder** (lines 171-187): Finder-specific logic
- **focusFinderAtPathOrLaunch** (lines 190-219): Path-based Finder focus
- **focusMostRecentInstanceOfApp** (lines 140-151): Walk instanceOrder backwards

---

### 1.4 WindowFrame.tsx (1223 lines)

**Purpose**: Resizable/draggable window wrapper with theme-specific controls

**Component Signature**:
```typescript
interface WindowFrameProps {
  children: React.ReactNode;
  title: string;
  onClose?: () => void;
  isForeground?: boolean;
  appId: AppId;
  isShaking?: boolean;
  transparentBackground?: boolean;
  skipInitialSound?: boolean;
  windowConstraints?: {
    minWidth?: number;
    minHeight?: number;
    maxWidth?: number | string;
    maxHeight?: number | string;
  };
  instanceId?: string;
  onNavigateNext?: () => void;
  onNavigatePrevious?: () => void;
  interceptClose?: boolean;
  menuBar?: React.ReactNode;  // For XP/Win98 in-window menubars
}
```

**State Management**:
- `isOpen: boolean` - Window open state (for close animation)
- `isVisible: boolean` - Visibility after close animation
- `isInitialMount: boolean` - First 200ms animation flag
- `isFullHeight: boolean` - Full height maximized state
- `isMaximized: boolean` - Full width+height maximized state
- `previousSizeRef: Ref<{width, height}>` - Size before maximizing
- `lastTapTimeRef: Ref<number>` - Double-tap detection
- `lastToggleTimeRef: Ref<number>` - Maximize cooldown (300ms)

**Hooks Used**:
- `useWindowManager({ appId, instanceId })` - Window position, size, drag/resize handlers
- `useAppStoreShallow()` - bringInstanceToForeground, debugMode, update functions
- `useThemeStore()` - Current theme
- `useSound()` - 4 window sounds (open, close, expand, collapse)
- `useVibration()` - Haptic feedback
- `useSwipeNavigation()` - Phone-only swipe between instances
- `useIsMobile()` - Mobile detection
- `useIsPhone()` - Phone detection

**Window Positioning** (lines 574-588):
- Desktop: Absolute positioning with `left`/`top` from windowPosition
- Mobile: Full width, dynamic height
- Min constraints: From windowConstraints prop or appRegistry
- Max constraints: Viewport-aware
- Transition: Disabled during drag/resize

**Resize Handles** (lines 592-729):
- **8 handles**: Top, bottom, left, right, 4 corners
- **Touch-friendly**: Larger hit areas on mobile (lines 606-632)
- **Active state**: Expands to 200px during resize (lines 604-611)
- **Z-index**: Theme-aware (`resizerZIndexClass`)
  - macOS X: `z-[60]` (above titlebar)
  - XP/Win98: `z-40` (below titlebar controls)
  - Default: `z-50`
- **Double-click**: Height-only maximize on top/bottom handles (line 620, 640)
- **Debug mode**: Red overlay to visualize hit areas

**Title Bar Variants**:

**XP/Win98 Theme** (lines 750-845):
- Class: `title-bar` from xp.css
- Inactive state: Different background color, grayscale icon
- Structure:
  - Icon + title text (lines 806-814)
  - Control buttons (minimize, maximize, close) (lines 816-844)
  - All controls in `data-titlebar-controls` div
- Event handlers:
  - `onMouseDown`: Start drag
  - `onDoubleClick`: Full maximize (if not from controls)
  - `onTouchStart`: Double-tap detect + drag start + swipe start

**Mac OS X Theme** (lines 846-1103):
- Traffic light buttons (lines 902-1079):
  - Close (red): Gradient background, shine/glow effects
  - Minimize (yellow): Disabled, grayscale
  - Maximize (green): Functional, shine/glow effects
  - Each: 13px diameter, complex box-shadow layers
  - Inactive state: Grayscale with different box-shadow
- Title: Centered, truncated with ellipsis (lines 1082-1099)
- Spacer: Right side to balance traffic lights (lines 1101-1102)
- Background: Pinstripe pattern, semi-transparent
- Border bottom: 1px solid with theme color

**System 7 Theme** (lines 1105-1177):
- Close box: Left side, 4×4px with hover/active states
- Title: Centered with bg-os-button-face background
- Spacer: Right side
- Pattern background: `bg-os-titlebar-pattern`
- Border: 1.5px bottom border

**Maximize Logic**:

**Height-Only** (lines 302-359):
- Triggered by: Double-click on top/bottom resize handles
- If already full height: Restore to default height
- If not full height: Set to viewport max (minus insets)
- Sound: Expand on maximize, collapse on restore

**Full Maximize** (lines 362-479):
- Triggered by: Double-click on titlebar, maximize button click
- Toggle state with cooldown (300ms to prevent rapid toggling)
- If maximizing:
  - Save current size to `previousSizeRef`
  - Calculate max size (viewport minus insets)
  - Center position on desktop, full width on mobile
  - Sound: Expand
- If restoring:
  - Restore to default size from appRegistry
  - Center position on desktop
  - Sound: Collapse

**Double-Tap Detection** (lines 482-527):
- Touch events only (lines 483-527)
- Time threshold: 300ms between taps
- Processing flag to prevent multiple triggers
- Cooldown: 300ms after maximize/restore
- Timeout to reset tap state if no second tap

**Swipe Navigation** (Phone Only, lines 124-144):
- Left swipe: Navigate to next instance (onNavigateNext)
- Right swipe: Navigate to previous instance (onNavigatePrevious)
- Threshold: 100px swipe distance
- Visual feedback: Slight translateX during swipe (lines 541-552)
- Sound + vibration on successful swipe

**Close Handling**:
- **Normal close** (lines 173-183):
  - Set isOpen false → opacity 0 animation
  - Play close sound + vibration
  - Wait for transitionEnd → call onClose
- **Intercepted close** (lines 174-177):
  - Call onClose immediately (for confirmation dialogs)
  - Parent dispatches CustomEvent to trigger actual close
  - Listen for `closeWindow-${instanceId}` event (lines 196-224)

**XP/Win98 MenuBar Integration** (lines 1179-1190):
- If menuBar prop provided: Render inside window below titlebar
- Background: `var(--button-face)`
- Border bottom: 1px solid shadow color

**Window Content** (lines 1192-1217):
- Flex container: `flex-1 min-h-0`
- XP theme: `window-body` class from xp.css
- macOS X: Pinstripe background pattern
- Transparent background option for special windows

**Insets Calculation** (lines 238-255):
- **computeInsets()**: Centralized inset logic
  - menuBarHeight: System 7 (30px), macOS X (25px), XP/Win98 (0px)
  - taskbarHeight: XP/Win98 (30px), others (0px)
  - dockHeight: macOS X (56px), others (0px)
  - safeAreaBottom: From CSS env(safe-area-inset-bottom)
  - topInset: menuBarHeight
  - bottomInset: taskbarHeight + dockHeight + safeAreaBottom

---

### 1.5 StartMenu.tsx (248 lines)

**Purpose**: Windows-style Start menu with app launcher

**Component Signature**:
```typescript
interface StartMenuProps {
  apps: AnyApp[];
}
```

**Rendering**: Only for XP/Win98 themes (in MenuBar.tsx)

**State Management**:
- `isStartMenuOpen: boolean` - Menu open state
- `aboutFinderOpen: boolean` - About dialog state

**Visual Structure**:
- **Left Panel** (lines 130-160):
  - Width: 32px
  - Gradient background (XP: blue gradient, Win98: darker blue)
  - Rotated text: "ryOS Professional" / "ryOS 98"
  - Transform origin: left bottom
- **Right Panel** (lines 163-236):
  - White background
  - Max height: 80vh with overflow scroll
  - "About This Computer" at top (lines 176-192)
  - Separator (lines 195-198)
  - All apps with icons (lines 200-234)

**Start Button Styling**:
- **XP theme** (lines 43-46):
  - Width: 100px, rounded right corners
  - Green gradient background
  - Active state: Lighter green gradient
- **Win98 theme** (lines 47-58):
  - Flat gray background
  - Box-shadow for 3D effect
  - Active state: Inset shadow

**Menu Item Styling**:
- Height: 32px (8h)
- Hover: Blue background with white text
- Font: MS Sans Serif, 11px
- Icon: 24×24px (6×6 in Tailwind)

**Icon Rendering** (lines 213-231):
- If string starting with "/icons/": Use ThemedIcon
- If emoji string: Render in 24×24 container
- If object: Use icon.src with ThemedIcon

---

### 1.6 AppleMenu.tsx (95 lines)

**Purpose**: macOS-style Apple menu with app launcher

**Component Signature**:
```typescript
interface AppleMenuProps {
  apps: AnyApp[];
}
```

**Rendering**: Only for Mac themes (in MenuBar.tsx)

**State Management**:
- `aboutFinderOpen: boolean` - About dialog state

**Trigger Button**:
- macOS X theme: Apple icon image (30×30px) (lines 47-50)
- System 7 theme: Apple character "\uf8ff" (lines 53)

**Menu Structure**:
- "About This Computer" at top (lines 58-63)
- Separator (line 64)
- All apps with icons (lines 65-84)

**App Launching Logic** (lines 28-31):
- Simply calls `launchApp(appId)`
- Instance system handles focus vs new window

**Icon Rendering** (lines 71-80):
- If string: Render in 16×16 container
- If object: Use ThemedIcon with icon.src

---

## 2. Theme System

### 2.1 Theme Types (types.ts)

**OsThemeId**: `"system7" | "macosx" | "xp" | "win98"`

**OsTheme Interface**:
```typescript
interface OsTheme {
  id: OsThemeId;
  name: string;
  fonts: {
    ui: string;
    mono?: string;
  };
  colors: {
    windowBg: string;
    menubarBg: string;
    menubarBorder: string;
    windowBorder: string;
    windowBorderInactive?: string;
    titleBar: {
      activeBg: string;
      inactiveBg: string;
      text: string;
      inactiveText: string;
      border?: string;
      borderInactive?: string;
      borderBottom?: string;
      pattern?: string;
    };
    button: {
      face: string;
      highlight: string;
      shadow: string;
      activeFace?: string;
    };
    trafficLights?: {
      close: string;
      closeHover?: string;
      minimize: string;
      minimizeHover?: string;
      maximize: string;
      maximizeHover?: string;
    };
    selection: {
      bg: string;
      text: string;
    };
    text: {
      primary: string;
      secondary: string;
      disabled: string;
    };
  };
  metrics: {
    borderWidth: string;
    radius: string;
    titleBarHeight: string;
    titleBarRadius?: string;
    windowShadow: string;
  };
  assets?: {
    closeButton?: string;
    maximizeButton?: string;
    minimizeButton?: string;
  };
  wallpaperDefaults?: {
    photo?: string;
    tile?: string;
    video?: string;
  };
}
```

### 2.2 macOS X Theme (macosx.ts)

**Fonts**:
- UI: Lucida Grande, fallback to system sans-serif
- Mono: Monaco, Menlo

**Colors**:
- Window: #ECECEC
- Menubar: Linear gradient (#FAFAFA → #D1D1D1)
- Title bar active: Linear gradient (#f6f6f6 → #dadada)
- Title bar inactive: #f6f6f6 with 0.85 opacity
- Traffic lights: Red (#FF6057), Yellow (#FFBD2E), Green (#27C93F)
- Selection: #3067da (blue)

**Metrics**:
- Border: 1px
- Radius: 0.5rem (8px)
- Title bar height: 1.375rem (22px)
- Title bar radius: 8px top corners
- Window shadow: 0 8px 25px rgba(0,0,0,0.5)

**Wallpapers**:
- Photo: /wallpapers/photos/aqua/abstract-7.jpg
- Video: /wallpapers/photos/aqua/water.jpg

### 2.3 Windows XP Theme (xp.ts)

**Fonts**:
- UI: Tahoma, Segoe UI
- Mono: Consolas, Courier New

**Colors**:
- Window: #ECE9D8 (beige)
- Menubar: Linear gradient (#245EDC → #1941A5) (blue)
- Title bar active: Linear gradient (#0058E6 → #1941A5)
- Title bar inactive: Linear gradient (#8B9DC3 → #7A8AAB)
- Button face: #ECE9D8
- Selection: #316AC5 (blue)

**Metrics**:
- Border: 3px
- Radius: 0.5rem (8px)
- Title bar height: 1.875rem (30px)
- Window shadow: 0 4px 8px rgba(0,0,0,0.25)

**Wallpapers**:
- Photo: /wallpapers/photos/landscapes/bliss.jpg
- Video: /wallpapers/videos/bliss_og.mp4

### 2.4 Theme Impact on Layout

**MenuBar Positioning**:
- macOS/System7: Top of viewport, fixed
- XP/Win98: Bottom of viewport, fixed

**Desktop Icon Layout**:
- macOS: Bottom-to-top, right-aligned
- XP/Win98: Top-to-bottom, left-aligned

**Window Controls**:
- macOS X: Traffic light buttons (left side)
- System 7: Close box (left side)
- XP/Win98: Min/Max/Close buttons (right side)

**Dock Visibility**:
- macOS X: Always visible
- Others: Hidden (null return)

---

## 3. Shared Components

### 3.1 ThemedIcon.tsx (56 lines)

**Purpose**: Theme-aware icon loading with async resolution

**Component Signature**:
```typescript
interface ThemedIconProps extends ImgHTMLAttributes<HTMLImageElement> {
  name: string;
  alt?: string;
  themeOverride?: string | null;
}
```

**Resolution Flow**:
1. If remote URL (http/https): Passthrough directly
2. `resolveIconLegacyAware(name, theme)` - Initial sync resolution
3. `useIconPath(logical, theme)` - Async theme-specific resolution
4. Use async path if available, otherwise use initial resolved

**Logical Name Derivation** (lines 34-40):
- Strip query strings
- If starts with "/icons/": Extract filename
- Remove theme directory prefix (/icons/[theme]/)
- Keep as-is if not in /icons/ path

**Cache Busting**:
- Resolved path may include query string
- Stripped before deriving logical name
- Prevents duplicate params downstream

### 3.2 GalaxyBackground.tsx (273 lines)

**Purpose**: WebGL shader background rendering (Galaxy/Aurora/Nebula)

**Component Signature**:
```typescript
interface GalaxyBackgroundProps {
  shaderType?: ShaderType; // GALAXY | AURORA | NEBULA
}
```

**Rendering Condition**:
```typescript
const shouldRender = shaderEffectEnabled; // from useAppStore
```

**State Management**:
- `clockRef: Ref<THREE.Clock>` - Time tracking for shader uniforms

**Three.js Setup** (lines 41-254):
- Scene: OrthographicCamera (-1,1,1,-1) for fullscreen shader
- Renderer: WebGLRenderer with high-performance, capped pixel ratio (1.5)
- Geometry: PlaneGeometry(2,2) for fullscreen quad
- Material: ShaderMaterial with custom vertex/fragment shaders

**Shader Uniforms**:
- `resolution: vec2` - Canvas width/height
- `time: float` - Elapsed time from clock

**Shader Selection** (lines 82-204):
- **NEBULA**: Psychedelic flowing patterns with centered offset
- **AURORA**: Aurora borealis with raymarching (44 iterations)
- **GALAXY**: 3D spiral galaxy with rotation matrix (99 iterations)

**Performance Optimizations**:
- Antialias disabled
- Pixel ratio capped at 1.5
- Dim factors: Nebula 0.8, Aurora 0.4, Galaxy 0.4
- No texture lookups (mathematical noise only)

**Resize Handling** (lines 218-227):
- Update renderer size
- Update resolution uniform
- No camera update needed (orthographic)

**Animation Loop** (lines 230-239):
- requestAnimationFrame
- Update time uniform from clock
- Render scene

**Cleanup** (lines 242-253):
- Cancel animation frame
- Remove resize listener
- Remove canvas from DOM
- Dispose geometry, material, renderer

---

## 4. UI Components (shadcn/ui)

### 4.1 Button.tsx (247 lines)

**Variants** (via class-variance-authority):
- **default**: Primary background, hover fade
- **destructive**: Red background for dangerous actions
- **outline**: Border with background on hover
- **secondary**: Secondary background
- **ghost**: Transparent, background on hover
- **link**: Text with underline on hover
- **retro**: Border-image from SVG asset
- **aqua**: macOS Aqua style with gradient (lines 69-103)
- **player**: Video player controls style
- **aqua_select**: macOS select trigger style (lines 107-172)

**Sizes**:
- **default**: h-9 px-4 py-2
- **sm**: h-8 px-3 text-xs
- **lg**: h-10 px-8
- **icon**: h-9 w-9

**Theme-Specific Rendering**:
- **macOS + default**: Aqua primary button (lines 68-77)
- **macOS + secondary**: Aqua secondary button (lines 81-89)
- **macOS + retro**: Aqua secondary button (lines 94-102)
- **macOS + aqua_select**: Custom select trigger with focus/press states (lines 107-172)
- **XP/Win98 + default**: xp.css button class (lines 177-185)
- **XP/Win98 + ghost**: Override with transparent important (lines 201-215)

**Sound Effects** (lines 53, 62-65):
- `useSound(Sounds.BUTTON_CLICK)` on all clicks
- Plays via handleClick wrapper

**Aqua Select Variant** (lines 107-172):
- Focused state: Blue glow box-shadow
- Pressed state: Darker gradient, inset shadow
- Selected state: Different gradient background
- Height: 22px (fixed)
- Min width: 60px
- Font size: 13px
- Text shadow for depth

---

### 4.2 Dialog.tsx (404 lines)

**Components**:
- Dialog (root with sound effects)
- DialogTrigger
- DialogPortal
- DialogContent
- DialogHeader
- DialogFooter
- DialogTitle
- DialogDescription
- DialogClose

**Sound Integration** (lines 9-50):
- `WINDOW_OPEN` on open
- `WINDOW_CLOSE` on close
- Vibration on close (50ms, 50% strength)
- Skip flag to prevent double-play (lines 19-29)

**Content Styling**:
- **XP/Win98** (lines 78-84):
  - `window` class from xp.css
  - Animation: zoom-in/out, fade-in/out
- **macOS X** (lines 86-92):
  - Pinstripe window background
  - Border from theme metrics
  - Force 13px font size on all buttons
- **System 7** (lines 96-100):
  - OS window background
  - Theme border/shadow

**Header Variants**:

**XP/Win98** (lines 142-157):
- `title-bar` class from xp.css
- Title text in `title-bar-text`
- Close button in `title-bar-controls`

**macOS X** (lines 159-325):
- Traffic light buttons (same as WindowFrame)
- Centered title with truncation
- Spacer for balance
- Pinstripe background
- Border bottom from theme

**System 7** (lines 329-348):
- Close box (left)
- Centered title with bg-os-button-face
- Spacer (right)

**Pointer Events Cleanup** (lines 64-75):
- Remove pointer-events style on close
- Use requestAnimationFrame for timing
- Cleanup on unmount

---

### 4.3 DropdownMenu.tsx (418 lines)

**Components**:
- DropdownMenu (root with sound effects)
- DropdownMenuTrigger
- DropdownMenuContent
- DropdownMenuSubContent
- DropdownMenuItem
- DropdownMenuCheckboxItem
- DropdownMenuRadioItem
- DropdownMenuLabel
- DropdownMenuSeparator
- DropdownMenuShortcut
- DropdownMenuGroup
- DropdownMenuSub
- DropdownMenuSubTrigger
- DropdownMenuRadioGroup

**Sound Integration** (lines 10-32):
- `MENU_OPEN` on open
- `MENU_CLOSE` on close

**macOS X Content Styling** (lines 169-181):
- No border
- No border radius
- Pinstripe window background
- 0.92 opacity
- Box shadow: 0 4px 16px rgba(0,0,0,0.4)
- Padding: 4px 0
- Min width: 180px (desktop), unset (mobile)

**macOS X Item Styling** (lines 219-225):
- No border radius
- Padding: 6px 20px 6px 16px
- Margin: 1px 0
- WebKit font smoothing: antialiased
- Text shadow for depth
- Font size: 13px

**XP/Win98 Styling** (lines 209-217):
- Font: MS Sans Serif
- Font size: 11px

**Separator Styling**:
- **macOS X** (lines 375-380):
  - Background: rgba(0,0,0,0.15)
  - No border
  - Margin: 4px 0
  - Height: 1px
- **System 7** (lines 371):
  - Dotted border
- **Others** (lines 372):
  - Solid border

**Trigger Styling** (lines 38-56):
- macOS X: Text shadow for depth

**SubTrigger Styling** (lines 73-109):
- Chevron right icon
- Theme-specific fonts/sizes

**Mobile Adjustments**:
- Min width: unset on mobile (lines 139, 179)
- Via useMediaQuery("(max-width: 768px)")

---

## 5. Window Management Hooks

### 5.1 useWindowManager.ts (427 lines)

**Purpose**: Window position, size, drag, resize handling

**Hook Signature**:
```typescript
interface UseWindowManagerProps {
  appId: AppId;
  instanceId?: string;
}
```

**Return Value**:
```typescript
{
  windowPosition: WindowPosition;
  windowSize: WindowSize;
  isDragging: boolean;
  resizeType: ResizeType;
  handleMouseDown: (e: MouseEvent | TouchEvent) => void;
  handleResizeStart: (e: MouseEvent | TouchEvent, type: ResizeType) => void;
  setWindowSize: (size: WindowSize) => void;
  setWindowPosition: (pos: WindowPosition) => void;
  maximizeWindowHeight: (maxHeight?: number | string) => void;
  getSafeAreaBottomInset: () => number;
}
```

**State Initialization** (lines 19-79):
1. Fetch persisted state from useAppStore (instance or app level)
2. Compute default state if not persisted:
   - Position: Staggered by app index (32px offset)
   - Mobile Y: Fixed 28px (account for menubar)
   - Desktop Y: 40px + 20px offset per app
   - Size: From appRegistry config
3. Adjust position if window would be off-screen

**Dragging** (lines 179-194, 224-257):
- **handleMouseDown**: Capture offset from mouse to top-left
- **handleMove**: Update position with viewport constraints
  - Mobile: Only vertical dragging, full width
  - Desktop: Both axes, constrain to viewport
  - Menu bar: Minimum Y position
  - Bottom inset: Maximum Y position
- **Sound**: Loop playback during drag (300ms interval)

**Resizing** (lines 196-222, 256-341):
- **handleResizeStart**: Capture initial size/position, resize type
- **handleMove**: Update size/position based on resize type
  - East/West: Adjust width
  - North/South: Adjust height
  - Corners: Both
  - Min/max constraints from config
  - Mobile: Disable horizontal resize
- **Sound**: Loop playback during resize (300ms interval)

**End Handlers** (lines 344-373):
- Stop dragging/resizing
- Clear sound loops
- Play stop sound
- Persist to useAppStore (instance or app level)

**Insets Computation** (lines 106-122):
- **computeInsets()**: Same logic as WindowFrame
  - menuBarHeight, taskbarHeight, dockHeight, safeAreaBottom
  - topInset, bottomInset

**Maximize Height** (lines 137-177):
- Calculate max height (viewport - insets)
- Respect maxHeight constraint
- Update size and position
- Persist to store

**Sound Management** (lines 124-127, 238-241, 336-340):
- `moveAudioRef`: Interval for drag sound loop
- `resizeAudioRef`: Interval for resize sound loop
- Clear on mouse up, play stop sound

**Dependencies** (lines 395-412):
- Cleanup listeners on unmount
- Re-run effect when drag/resize state changes
- Update on viewport changes (isMobile, isXpTheme)

---

### 5.2 useWallpaper.ts (57 lines)

**Purpose**: Wallpaper state and helpers

**Return Value**:
```typescript
{
  currentWallpaper: string;      // Path or IndexedDB key
  wallpaperSource: string;       // Resolved URL or blob URL
  setWallpaper: (path: string) => Promise<void>;
  isVideoWallpaper: boolean;     // Derived from source
  loadCustomWallpapers: () => Promise<CustomWallpaper[]>;
  getWallpaperData: (key: string) => Promise<Blob | null>;
  INDEXEDDB_PREFIX: string;      // Constant for custom wallpapers
}
```

**State Selectors** (lines 10-11):
- `currentWallpaper` from useAppStore
- `wallpaperSource` from useAppStore (resolved)

**Actions** (lines 14-16):
- `setWallpaper` from useAppStore
- `loadCustomWallpapers` from useAppStore
- `getWallpaperData` from useAppStore

**isVideoWallpaper Derivation** (lines 18-26):
- Ends with .mp4: true
- Includes "video/": true
- HTTPS URL with video extension: true

**Stale Source Refresh** (lines 28-45):
- On mount: Check if source needs refresh
- Custom wallpaper (starts with INDEXEDDB_PREFIX)
- Source looks stale (not resolved or blob URL)
- Call setWallpaper once to refresh
- Avoid infinite loops with `hasAttemptedRefresh` ref

---

## 6. Data Flow Analysis

### 6.1 Theme Change Propagation

**Flow**:
```
Control Panels → setTheme(newTheme)
  ↓
useThemeStore.setTheme → localStorage persist
  ↓
All components re-render (useThemeStore subscribers)
  ↓
AppManager → MenuBar position change (top vs bottom)
  ↓
WindowFrame → Title bar controls change (traffic lights vs XP buttons)
  ↓
Desktop → Icon layout change (bottom-up vs top-down)
  ↓
Dock → Show/hide (macOS only)
  ↓
ThemedIcon → Asset paths change (icons/[theme]/)
  ↓
Font loading → CSS variables update
```

**Affected Components**:
- AppManager (menubar positioning)
- MenuBar (structure switch)
- WindowFrame (controls, styling)
- Desktop (icon layout)
- Dock (visibility)
- ThemedIcon (all icons)
- Button (variant styling)
- Dialog (header styling)
- DropdownMenu (item styling)

### 6.2 Window Positioning Flow

**Initial Position**:
```
AppManager → launchApp(appId, initialData)
  ↓
useAppStore.launchApp → createAppInstance
  ↓
Check persisted position (instances[id] or apps[id])
  ↓
If not persisted: computeDefaultWindowState
  ↓
Stagger position (32px offset per existing instance)
  ↓
Adjust if off-screen
  ↓
Store in instances[instanceId]
  ↓
WindowFrame receives position via useWindowManager
```

**Drag Update**:
```
User drags titlebar
  ↓
WindowFrame → handleMouseDown (capture offset)
  ↓
useWindowManager → handleMouseDown
  ↓
setIsDragging(true)
  ↓
Document mousemove listener → handleMove
  ↓
Calculate new position (mouse - offset)
  ↓
Constrain to viewport (minus insets)
  ↓
setWindowPosition(newPos)
  ↓
On mouse up: updateInstanceWindowState(instanceId, pos, size)
  ↓
useAppStore persists → localStorage
```

### 6.3 Window Resize Flow

**Resize Start**:
```
User mousedown on resize handle
  ↓
WindowFrame → handleResizeStart(e, type)
  ↓
useWindowManager → handleResizeStart
  ↓
Capture initial state (x, y, width, height, left, top)
  ↓
setResizeType(type) // "n", "s", "e", "w", "ne", "nw", "se", "sw"
  ↓
Document mousemove listener → handleMove
```

**Resize Move**:
```
handleMove with resizeType
  ↓
Calculate deltaX, deltaY from initial
  ↓
If type includes "e": newWidth = initial + deltaX
If type includes "w": newWidth = initial - deltaX, adjust left
If type includes "s": newHeight = initial + deltaY
If type includes "n": newHeight = initial - deltaY, adjust top
  ↓
Apply min/max constraints
  ↓
Mobile: Force full width (no horizontal resize)
  ↓
setWindowSize(newSize)
setWindowPosition(newPos)
  ↓
On mouse up: updateInstanceWindowState(instanceId, pos, size)
```

### 6.4 Focus Management Flow

**Window Click**:
```
User clicks on window
  ↓
WindowFrame div onClick
  ↓
if (!isForeground): bringInstanceToForeground(instanceId)
  ↓
useAppStore.bringInstanceToForeground
  ↓
Filter instanceOrder (remove instanceId)
  ↓
Append instanceId to end (highest z-index)
  ↓
Update isForeground flags (only last = true)
  ↓
Dispatch CustomEvent("instanceStateChange")
  ↓
AppManager re-renders
  ↓
WindowFrame receives new isForeground prop
  ↓
Title bar styling updates (active vs inactive)
  ↓
Z-index updates (calculated from instanceOrder position)
```

### 6.5 Wallpaper Loading Flow

**Initial Load**:
```
Desktop component mounts
  ↓
useWallpaper() hook
  ↓
currentWallpaper from useAppStore (persisted path or IndexedDB key)
  ↓
wallpaperSource from useAppStore (resolved URL or blob)
  ↓
If custom wallpaper + source stale:
  setWallpaper(currentWallpaper) → re-resolve
  ↓
If video: <video> element with src={wallpaperSource}
If image: CSS backgroundImage with url(wallpaperSource)
  ↓
Video playback management (useEffect):
  - visibility change → resume playback
  - window focus → resume playback
  - ended → restart from beginning
```

**Custom Wallpaper Upload**:
```
Control Panels → Appearance → Upload image/video
  ↓
File input onChange → read as ArrayBuffer
  ↓
Save to IndexedDB (key: "custom-wallpaper-{timestamp}")
  ↓
setWallpaper("indexeddb:custom-wallpaper-{timestamp}")
  ↓
useAppStore.setWallpaper:
  - Check if IndexedDB key
  - Load blob from IndexedDB
  - Create blob URL
  - Set wallpaperSource to blob URL
  - Persist currentWallpaper key to localStorage
  ↓
Desktop re-renders with new wallpaperSource
```

---

## 7. Responsive Design

### 7.1 Breakpoints

**Tailwind Breakpoints**:
- `sm`: 640px
- `md`: 768px
- `lg`: 1024px
- `xl`: 1280px
- `2xl`: 1536px

**Custom Hooks**:
- `useIsMobile()`: `< 768px`
- `useIsPhone()`: `< 600px`

**MenuBar Responsive**:
- Volume control: Hidden on `< sm` (640px)
- Clock format:
  - `< 420px`: Time only
  - `420-768px`: Time + AM/PM
  - `> 768px`: Full date + time

### 7.2 Mobile Adaptations

**Desktop Icons**:
- Mobile: Larger hit areas for touch
- Desktop: Smaller, more compact

**Window Positioning**:
- Mobile: Full width, vertical drag only
- Desktop: Free positioning, both axes

**Window Resizing**:
- Mobile: Height resize only (width locked to 100%)
- Desktop: 8-direction resize handles

**Dock**:
- Phone: Horizontal scroll enabled
- Desktop: No scroll, magnification on hover

**Taskbar** (XP/Win98):
- Overflow: Menu for extra apps when space limited
- Dynamic button sizing based on container width

### 7.3 Safe Area Insets

**Purpose**: Handle iPhone notch, home indicator on iOS

**CSS Variable**:
```css
--sat-safe-area-bottom: env(safe-area-inset-bottom, 0px)
```

**Usage**:
- Desktop: Height calculation excludes safe area
- Taskbar: Bottom position accounts for safe area
- WindowFrame: Max height calculation subtracts safe area
- Dock: Padding bottom includes safe area

**Calculation** (useWindowManager.ts):
```typescript
const getSafeAreaBottomInset = () => {
  const safeAreaInset = parseInt(
    getComputedStyle(document.documentElement)
      .getPropertyValue("--sat-safe-area-bottom")
  );
  return !isNaN(safeAreaInset) ? safeAreaInset : (isMobile ? 20 : 0);
};
```

---

## 8. Z-Index Management

### 8.1 Z-Index Hierarchy

**Fixed Layers** (ascending):
- `-10`: Video wallpaper (Desktop)
- `-1`: Desktop background
- `1`: Desktop icons, relative content
- `40`: WindowFrame resize handles (XP/Win98, below titlebar)
- `50`: WindowFrame (instance windows)
- `50`: Resize handles (default)
- `60`: Resize handles (macOS, above titlebar)
- `50`: MenuBar (Mac top) / Taskbar (Windows bottom)

**Dynamic Window Z-Index** (AppManager.tsx:76-80):
```typescript
const BASE_Z_INDEX = 50;
const getZIndexForInstance = (instanceId: string): number => {
  const index = instanceOrder.indexOf(instanceId);
  return index === -1 ? BASE_Z_INDEX : BASE_Z_INDEX + index;
};
```

**instanceOrder**:
- Array of instanceIds
- END of array = highest z-index (foreground)
- `bringInstanceToForeground` moves instanceId to end

**Example**:
```typescript
// instanceOrder: ["finder-1", "textedit-1", "paint-1"]
// Z-indexes:      50           51             52 (foreground)

bringInstanceToForeground("finder-1");
// instanceOrder: ["textedit-1", "paint-1", "finder-1"]
// Z-indexes:      50             51         52 (foreground)
```

### 8.2 Resize Handle Z-Index

**Theme-Aware** (WindowFrame.tsx:120-121):
```typescript
const resizerZIndexClass =
  currentTheme === "macosx" ? "z-[60]" :
  isXpTheme ? "z-40" : "z-50";
```

**Reasoning**:
- macOS X: Traffic lights on left, safe to be above titlebar
- XP/Win98: Close button on right, must be below titlebar
- System 7: Close box on left, default layer

---

## 9. Animation System

### 9.1 Window Animations

**Open Animation** (WindowFrame.tsx:559):
```typescript
isInitialMount && "animate-in fade-in-0 zoom-in-95 duration-200"
```
- Fade in from 0 opacity
- Zoom in from 95% scale
- 200ms duration
- Only on initial mount

**Close Animation** (WindowFrame.tsx:585-586):
```typescript
transform: !isInitialMount && !isOpen ? "scale(0.95)" : undefined,
opacity: !isInitialMount && !isOpen ? 0 : undefined,
```
- Zoom out to 95% scale
- Fade out to 0 opacity
- Triggered by isOpen false
- Transition via CSS (duration-200)

**Transition Handling** (lines 155-171):
- Listen for transitionEnd event
- Check propertyName === "opacity"
- After transition: setIsVisible(false), call onClose
- Prevents visual jump

### 9.2 Dock Animations (Framer Motion)

**Layout Animations**:
```typescript
<LayoutGroup>
  <AnimatePresence mode="popLayout" initial={false}>
    <IconButton
      layout
      layoutId={`dock-icon-${idKey}`}
      initial={isNew ? { scale: 0, opacity: 0 } : undefined}
      animate={{ scale: 1, opacity: 1 }}
      exit={{ scale: 0, opacity: 0 }}
      transition={{
        type: "spring",
        stiffness: 300,
        damping: 30,
        mass: 0.8,
        layout: {
          type: "spring",
          stiffness: 300,
          damping: 30,
          mass: 0.8,
        },
      }}
    />
  </AnimatePresence>
</LayoutGroup>
```

**Magnification Animation**:
```typescript
// Distance from cursor
const distanceCalc = useTransform(mouseX, (val) => {
  const bounds = wrapperRef.current?.getBoundingClientRect();
  if (!bounds || !Number.isFinite(val)) return Infinity;
  return val - (bounds.left + bounds.width / 2);
});

// Size transform
const sizeTransform = useTransform(
  distanceCalc,
  [-DISTANCE, 0, DISTANCE],
  [baseSize, maxSize, baseSize]
);

// Spring smoothing
const sizeSpring = useSpring(sizeTransform, {
  mass: 0.15,
  stiffness: 160,
  damping: 18,
});

// Apply to width/height
style={{
  width: sizeSpring,
  height: sizeSpring,
}}
```

### 9.3 Dialog Animations

**Entry/Exit** (Dialog.tsx):
```typescript
className="duration-200
  data-[state=open]:animate-in
  data-[state=closed]:animate-out
  data-[state=closed]:fade-out-0
  data-[state=open]:fade-in-0
  data-[state=closed]:zoom-out-95
  data-[state=open]:zoom-in-95
  origin-center"
```

**Overlay** (Dialog.tsx:105):
```typescript
className="data-[state=open]:animate-in
  data-[state=closed]:animate-out
  data-[state=closed]:fade-out-0
  data-[state=open]:fade-in-0"
```

---

## 10. Dependency Mapping

### 10.1 Component Dependencies

**Desktop.tsx**:
- Hooks: `useWallpaper`, `useThemeStore`, `useLongPress`
- Components: `FileIcon`, `RightClickMenu`
- Utils: `getAppIconPath`
- Types: `AnyApp`, `AppManagerState`, `AppId`, `SortType`

**MenuBar.tsx**:
- Hooks: `useAppStoreShallow`, `useThemeStore`, `useFilesStore`, `useLaunchApp`, `useSound`
- Components: `AppleMenu`, `StartMenu`, `Button`, `DropdownMenu*`, `Slider`, `ThemedIcon`, `HelpDialog`, `AboutDialog`
- Icons: `Volume1`, `Volume2`, `VolumeX`, `Settings`, `ChevronUp` (Lucide)
- Utils: `getAppIconPath`, `appRegistry`

**Dock.tsx**:
- Hooks: `useAppStoreShallow`, `useLaunchApp`, `useFilesStore`, `useFinderStore`, `useIsPhone`
- Components: `ThemedIcon`
- Animation: `AnimatePresence`, `motion`, `LayoutGroup`, `useMotionValue`, `useSpring`, `useTransform`, `useIsPresent` (Framer Motion)
- Utils: `getAppIconPath`, `appRegistry`

**WindowFrame.tsx**:
- Hooks: `useWindowManager`, `useAppContext`, `useSound`, `useVibration`, `useAppStoreShallow`, `useThemeStore`, `useSwipeNavigation`, `useIsMobile`, `useIsPhone`
- Components: `ThemedIcon`, `MenuBar` (conditional)
- Utils: `cn`, `getWindowConfig`, `getAppIconPath`, `getTheme`
- Types: `ResizeType`, `AppId`

**StartMenu.tsx**:
- Hooks: `useLaunchApp`, `useThemeStore`
- Components: `DropdownMenu*`, `AboutFinderDialog`, `ThemedIcon`
- Types: `AnyApp`, `AppId`

**AppleMenu.tsx**:
- Hooks: `useLaunchApp`, `useThemeStore`
- Components: `Button`, `DropdownMenu*`, `AboutFinderDialog`, `ThemedIcon`
- Utils: `cn`
- Types: `AnyApp`, `AppId`

### 10.2 Store Dependencies

**useAppStore**:
- Used by: All layout components, WindowFrame, useWindowManager
- Provides: Instances, instanceOrder, foregroundInstanceId, settings, update functions

**useThemeStore**:
- Used by: All layout components, all UI components
- Provides: Current theme, setTheme

**useFilesStore**:
- Used by: MenuBar (applet icons), Dock (applet icons, trash icon)
- Provides: File tree, file operations, trash state

**useFinderStore**:
- Used by: Dock (path-based focus)
- Provides: Finder instance states (currentPath)

### 10.3 Radix UI Primitives

**Button.tsx**:
- `@radix-ui/react-slot` - Slot primitive for asChild pattern

**Dialog.tsx**:
- `@radix-ui/react-dialog` - Root, Trigger, Portal, Overlay, Content, Close, Title, Description

**DropdownMenu.tsx**:
- `@radix-ui/react-dropdown-menu` - Root, Trigger, Portal, Content, Item, CheckboxItem, RadioItem, Label, Separator, Group, Sub, SubTrigger, SubContent, RadioGroup

**Icons**:
- `lucide-react` - Check, ChevronRight, Circle, Volume1, Volume2, VolumeX, Settings, ChevronUp

---

## 11. Gaps vs CLAUDE.md

### 11.1 Missing from CLAUDE.md

**Layout Component Details**:
- Desktop.tsx video playback management (visibility/focus handlers)
- Desktop.tsx long-press mobile support for context menus
- MenuBar.tsx taskbar overflow detection and handling
- MenuBar.tsx applet icon/label extraction logic
- Dock.tsx magnification system (distance calculation, spring physics)
- Dock.tsx applet instance rendering logic
- WindowFrame.tsx double-tap detection for mobile maximize
- WindowFrame.tsx swipe navigation for instance switching
- WindowFrame.tsx close interception system
- WindowFrame.tsx safe area inset calculations

**Theme System Details**:
- Complete OsTheme interface specification
- macOS X traffic light button implementation details
- XP/Win98 taskbar styling specifications
- Theme-specific font loading
- Wallpaper default configurations per theme

**Animation System**:
- Dock magnification physics (Framer Motion springs)
- Window open/close animation timing
- Dialog entry/exit animations
- Layout animations for dock icons

**Responsive Design**:
- MenuBar clock formatting breakpoints
- Taskbar button overflow algorithm
- Dock horizontal scroll on phone
- Safe area inset handling across components

**UI Component Specifications**:
- Button variant details (aqua, aqua_select, retro, player)
- Dialog theme-specific header implementations
- DropdownMenu macOS X styling specifics
- ThemedIcon async resolution flow

### 11.2 Inaccuracies in CLAUDE.md

**Component Line Counts**:
- CLAUDE.md estimates, actual line counts now documented

**Z-Index Details**:
- CLAUDE.md mentions BASE_Z_INDEX calculation
- Missing theme-aware resize handle z-index logic

**Window Management**:
- CLAUDE.md describes basic drag/resize
- Missing mobile-specific constraints (height-only resize, vertical-only drag)
- Missing maximize cooldown logic (300ms)
- Missing double-tap detection details

**Theme Impact**:
- CLAUDE.md mentions layout changes
- Missing specific inset calculations per theme
- Missing dock visibility logic
- Missing desktop icon layout algorithm differences

---

## 12. Key Architectural Insights

### 12.1 Multi-Theme Support

**Strategy**: Conditional rendering + theme-specific assets

**Implementation**:
- Single codebase, multiple render paths
- `useThemeStore.current` drives all decisions
- Components check theme and render appropriate structure
- Asset paths include theme directory: `/icons/[theme]/`

**Benefits**:
- No code duplication
- Runtime theme switching
- Consistent component interfaces

**Challenges**:
- Complex conditional logic in components
- Asset management (ensure all themes have required assets)
- Testing all theme variants

### 12.2 Instance-Based Windowing

**Strategy**: Unique instanceId per window, not per app

**Implementation**:
- `instances: Record<instanceId, AppInstance>`
- `instanceOrder: string[]` for z-index
- Multi-window apps: Create new instance for each window
- Single-window apps: Reuse existing instance

**Benefits**:
- Multiple windows of same app (TextEdit, Finder)
- Independent state per window
- Clean focus/close handling

**Challenges**:
- Complexity in instance management
- Persistence strategy (instance vs app state)
- Cleanup on instance close

### 12.3 Responsive Window Management

**Strategy**: Viewport-aware constraints with mobile adaptations

**Implementation**:
- Desktop: Free positioning, 8-direction resize
- Mobile: Full width, vertical-only drag, height-only resize
- Safe area insets: Account for iOS notch/home indicator
- Breakpoints: Different UI density for phone/tablet/desktop

**Benefits**:
- Touch-friendly on mobile
- Efficient use of small screens
- Consistent experience across devices

**Challenges**:
- Complex constraint logic
- Testing across viewport sizes
- Safe area detection on Android

### 12.4 Theme-Driven Layout Transformation

**Strategy**: Layout structure changes based on theme

**Implementation**:
- XP/Win98: Bottom taskbar, top-down icon layout
- Mac: Top menubar, bottom-up icon layout
- Conditional rendering in AppManager
- Theme-specific inset calculations

**Benefits**:
- Authentic OS experiences
- Clear separation of concerns
- Easy to add new themes

**Challenges**:
- Complex AppManager logic
- Ensuring all components adapt correctly
- Testing theme switches

### 12.5 Sound Effects Integration

**Strategy**: Centralized useSound hook with volume control

**Implementation**:
- All UI components use useSound hook
- Master volume + category volumes (UI, Terminal, etc.)
- Sound effects triggered on interactions
- Loop playback for continuous actions (drag, resize)

**Benefits**:
- Consistent audio experience
- User-controllable volumes
- Easy to add new sounds

**Challenges**:
- Audio file size (preloading)
- Mobile autoplay restrictions
- Sound loop cleanup on interruption

---

## 13. Recommendations

### 13.1 Documentation Updates

**High Priority**:
1. Add layout component details to CLAUDE.md Component Architecture section
2. Document theme system fully (OsTheme interface, asset organization)
3. Add responsive design section with breakpoints and mobile adaptations
4. Document z-index hierarchy and management strategy
5. Add animation system details (Framer Motion usage, window animations)

**Medium Priority**:
6. Document safe area inset handling across components
7. Add Radix UI primitive mapping for all UI components
8. Document sound effects system and integration points
9. Add desktop icon layout algorithm details
10. Document wallpaper loading flow (custom uploads, IndexedDB)

**Low Priority**:
11. Add visual diagrams for component hierarchy
12. Document testing strategy for multi-theme support
13. Add performance considerations for animations
14. Document accessibility features (keyboard navigation, ARIA labels)

### 13.2 Code Improvements

**High Priority**:
1. Extract magic numbers to constants (z-index BASE, magnification params, animation durations)
2. Centralize inset calculation logic (currently duplicated in WindowFrame and useWindowManager)
3. Add TypeScript strict null checks for safer window operations
4. Extract theme detection logic to utility (isXpTheme repeated everywhere)

**Medium Priority**:
5. Refactor WindowFrame to smaller sub-components (too long, 1223 lines)
6. Extract traffic light button to shared component (duplicated in Dialog)
7. Add error boundaries around window content
8. Improve type safety for theme-specific props

**Low Priority**:
9. Add unit tests for z-index calculation
10. Add integration tests for theme switching
11. Add visual regression tests for all themes
12. Optimize re-renders with React.memo where appropriate

### 13.3 Feature Enhancements

**High Priority**:
1. Add keyboard shortcuts for window management (Cmd+W, Cmd+M, Cmd+`)
2. Add window snapping to edges (like Windows Aero Snap)
3. Add window minimize animation to dock/taskbar
4. Add multi-monitor support (detect external displays)

**Medium Priority**:
5. Add window shake to bring to foreground (like macOS)
6. Add mission control / expose view (show all windows)
7. Add window preview on taskbar hover (XP/Win98)
8. Add custom window sizes per app (save user preferences)

**Low Priority**:
9. Add window transparency slider (for background blur)
10. Add window always-on-top option
11. Add window opacity animation on focus change
12. Add window shadow customization per theme

---

## Conclusion

The ryOS layout and UI system is a comprehensive implementation of multi-theme desktop environments with sophisticated window management, responsive design, and rich animations. The codebase demonstrates strong architectural patterns including theme-driven UI transformation, instance-based windowing, and conditional rendering for cross-platform experiences.

Key strengths include the flexible theme system, smooth animations, and comprehensive mobile adaptations. Areas for improvement include reducing code duplication, extracting constants, and enhancing type safety.

This analysis provides a complete reference for understanding and extending the layout/UI system.
