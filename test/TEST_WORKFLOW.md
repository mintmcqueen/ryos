# ryOS Testing Strategy & Workflow

**Purpose**: Document testing philosophy and comprehensive test execution workflows for ryOS

**Current Status**: Manual testing only - formal test infrastructure represents enhancement opportunity

---

## 🧪 Testing Philosophy

**Principles**:
1. **User-Centric**: Test from user perspective (app launch → interaction → close)
2. **Instance-Aware**: Verify multi-window scenarios (TextEdit, Finder, Applet Viewer)
3. **Theme-Agnostic**: Validate across all 4 themes (System 7, Aqua, XP, Win98)
4. **State-Persistent**: Verify localStorage/IndexedDB persistence across sessions
5. **Event-Driven**: Validate CustomEvent flows (launchApp, instanceStateChange, etc.)

---

## 📋 Current Testing Approach (Manual)

### **Build Validation**
```bash
# Development build
bun dev
# Verify: http://localhost:5173 loads without errors
# Check console for warnings

# Production build
bun run build
# Verify: No TypeScript errors, build completes successfully

# Preview production build
bun run preview
# Verify: Optimizations applied, functionality intact
```

### **Lint Validation**
```bash
bun run lint
# Target: Zero ESLint errors
# Fix auto-fixable issues: bun run lint --fix
```

### **App Launch Testing**

**Browser Console Testing**:
```javascript
// Test single app launch
window.dispatchEvent(new CustomEvent('launchApp', {
  detail: { appId: 'finder' }
}))

// Test app with initialData
window.dispatchEvent(new CustomEvent('launchApp', {
  detail: {
    appId: 'textedit',
    initialData: { path: '/Documents/test.txt' }
  }
}))

// Test multi-instance launch (TextEdit)
window.dispatchEvent(new CustomEvent('launchApp', {
  detail: { appId: 'textedit', initialData: { path: '/Documents/doc1.txt' } }
}))
window.dispatchEvent(new CustomEvent('launchApp', {
  detail: { appId: 'textedit', initialData: { path: '/Documents/doc2.txt' } }
}))
// Expected: Two TextEdit windows open simultaneously

// Verify instance state
console.log(useAppStore.getState().instances)
console.log(useAppStore.getState().instanceOrder)
```

### **Window Management Testing**

**Z-Index Validation**:
```javascript
// Launch multiple apps
['finder', 'textedit', 'paint', 'chats'].forEach(appId => {
  window.dispatchEvent(new CustomEvent('launchApp', { detail: { appId } }))
})

// Check z-index order (inspect elements)
// Expected: Last launched app has highest z-index
// Expected: instanceOrder matches visual stacking
```

**Focus Management**:
```javascript
// Get current foreground instance
const foreground = useAppStore.getState().getForegroundInstance()
console.log('Foreground:', foreground)

// Click window → verify isForeground updates
// Click different window → verify focus shifts
```

**Multi-Window Testing**:
```javascript
// Test Finder multi-window
window.dispatchEvent(new CustomEvent('launchApp', { detail: { appId: 'finder' } }))
window.dispatchEvent(new CustomEvent('launchApp', { detail: { appId: 'finder' } }))
// Expected: Two Finder windows with independent navigation

// Verify instance count
const finderInstances = useAppStore.getState().getInstancesByAppId('finder')
console.log('Finder instances:', finderInstances.length) // Should be 2
```

### **Theme Switching Testing**

```javascript
// Test all themes
const themes = ['system7', 'aqua', 'xp', 'win98']
themes.forEach(theme => {
  useThemeStore.getState().setTheme(theme)
  console.log(`Theme: ${theme}`)
  // Manually verify:
  // - MenuBar/Taskbar layout
  // - WindowFrame controls (traffic lights vs XP buttons)
  // - Font rendering
  // - Icon assets
  // - Wallpaper defaults
})
```

### **Filesystem Testing**

**Virtual FS Operations**:
```javascript
// Create file
useFilesStore.getState().createFile('/Documents', 'test.txt', 'Hello World')

// Read file
const file = useFilesStore.getState().getNodeByPath('/Documents/test.txt')
console.log('File content:', file?.content)

// Delete file
useFilesStore.getState().deleteNode('/Documents/test.txt')

// Verify persistence (refresh page)
// Expected: Changes persist via IndexedDB
```

**Backup/Restore**:
```bash
# Manual test in browser:
# 1. Create files/folders in Finder
# 2. Control Panels → Backup → Download JSON
# 3. Control Panels → Format (reset filesystem)
# 4. Control Panels → Restore → Upload JSON
# 5. Verify files/folders restored correctly
```

### **Instance Integrity Testing**

```javascript
// Run integrity check
useAppStore.getState()._debugCheckInstanceIntegrity()

// Verify no dangling instances
const instances = useAppStore.getState().instances
const instanceOrder = useAppStore.getState().instanceOrder

// All instances in order should exist and be open
instanceOrder.forEach(id => {
  const instance = instances[id]
  console.assert(instance, `Instance ${id} in order but not in instances`)
  console.assert(instance.isOpen, `Instance ${id} in order but not open`)
})

// All open instances should be in order
Object.values(instances).forEach(instance => {
  if (instance.isOpen) {
    console.assert(
      instanceOrder.includes(instance.instanceId),
      `Instance ${instance.instanceId} open but not in order`
    )
  }
})
```

### **URL Routing Testing**

```javascript
// Test Internet Explorer share URL
window.location.href = '/internet-explorer/abc123'
// Expected: IE launches with shareCode 'abc123'

// Test iPod share URL
window.location.href = '/ipod/dQw4w9WgXcQ'
// Expected: iPod launches with videoId 'dQw4w9WgXcQ'

// Test Videos share URL
window.location.href = '/videos/dQw4w9WgXcQ'
// Expected: Videos launches with videoId

// Test direct app launch
window.location.href = '/finder'
// Expected: Finder launches

// Verify URL cleaned after launch
// Expected: URL returns to '/'
```

### **AI Integration Testing**

**Chat Flow** (requires API keys in .env):
```javascript
// 1. Launch Chats app
// 2. Send message: "Hello Ryo"
// 3. Verify: Streaming response appears
// 4. Verify: Message persisted in useChatsStore
// 5. Enable TTS (Control Panels → Sounds)
// 6. Send message with speech enabled
// 7. Verify: Audio plays with word highlighting
```

**Voice Transcription**:
```javascript
// 1. Launch Chats app
// 2. Enable microphone permission
// 3. Press and hold push-to-talk button
// 4. Speak: "Test transcription"
// 5. Release button
// 6. Verify: Text appears in chat input
```

### **Audio System Testing**

**Sound Effects**:
```javascript
// Test UI sounds
useAppStore.getState().setUiSoundsEnabled(true)
// Click buttons, open apps → verify sounds play

// Test Terminal sounds
useAppStore.getState().setTerminalSoundsEnabled(true)
// Launch Terminal → type commands → verify keystroke sounds

// Test volume controls
useAppStore.getState().setUiVolume(0.5)
useAppStore.getState().setMasterVolume(0.8)
// Verify: Sound levels adjust correctly
```

**Synth/Soundboard**:
```javascript
// Launch Synth → play virtual keyboard → verify audio output
// Launch Soundboard → record audio → verify waveform visualization
// Test audio context sharing (singleton pattern)
```

---

## 🚀 Recommended Testing Strategy (Future Enhancement)

### **Unit Tests** (Jest + @testing-library/react)

**Priority Test Suites**:

**1. Zustand Store Actions**:
```javascript
// test/unit/stores/useAppStore.test.ts
describe('useAppStore', () => {
  test('createAppInstance generates unique ID', () => {
    const store = useAppStore.getState()
    const id1 = store.createAppInstance('finder')
    const id2 = store.createAppInstance('finder')
    expect(id1).not.toBe(id2)
  })

  test('launchApp reuses instance for single-window apps', () => {
    const store = useAppStore.getState()
    const id1 = store.launchApp('paint')
    const id2 = store.launchApp('paint')
    expect(id1).toBe(id2)
  })

  test('instanceOrder maintains foreground at end', () => {
    const store = useAppStore.getState()
    const id1 = store.createAppInstance('finder')
    const id2 = store.createAppInstance('textedit')
    store.bringInstanceToForeground(id1)
    expect(store.instanceOrder[store.instanceOrder.length - 1]).toBe(id1)
  })
})
```

**2. Utility Functions**:
```javascript
// test/unit/utils/displayMode.test.ts
describe('displayMode', () => {
  test('applyDisplayMode applies correct CSS filter', () => {
    applyDisplayMode('grayscale')
    expect(document.body.style.filter).toContain('grayscale')
  })
})
```

**3. Custom Hooks**:
```javascript
// test/unit/hooks/useWindowManager.test.ts
describe('useWindowManager', () => {
  test('returns correct window state for appId', () => {
    const { result } = renderHook(() => useWindowManager('finder', jest.fn()))
    expect(result.current).toHaveProperty('isOpen')
    expect(result.current).toHaveProperty('isForeground')
  })
})
```

---

### **Integration Tests** (React Testing Library)

**Priority Test Scenarios**:

**1. App Launch Flow**:
```javascript
// test/integration/app-launch.test.tsx
test('launching app creates instance and renders WindowFrame', async () => {
  render(<App />)
  fireEvent(window, new CustomEvent('launchApp', {
    detail: { appId: 'finder' }
  }))
  await waitFor(() => {
    expect(screen.getByText(/Finder/i)).toBeInTheDocument()
  })
})
```

**2. Multi-Window Scenarios**:
```javascript
test('TextEdit supports multiple windows', async () => {
  render(<App />)

  // Launch first instance
  fireEvent(window, new CustomEvent('launchApp', {
    detail: { appId: 'textedit', initialData: { path: '/doc1.txt' } }
  }))

  // Launch second instance
  fireEvent(window, new CustomEvent('launchApp', {
    detail: { appId: 'textedit', initialData: { path: '/doc2.txt' } }
  }))

  await waitFor(() => {
    const windows = screen.getAllByText(/TextEdit/i)
    expect(windows).toHaveLength(2)
  })
})
```

**3. Theme Switching**:
```javascript
test('theme switch updates layout', async () => {
  render(<App />)

  // Initial theme (Aqua) → MenuBar at top
  expect(screen.getByRole('menubar')).toBeInTheDocument()

  // Switch to XP → Taskbar at bottom
  act(() => {
    useThemeStore.getState().setTheme('xp')
  })

  await waitFor(() => {
    expect(screen.getByRole('navigation')).toHaveClass('taskbar')
  })
})
```

**4. Filesystem Operations**:
```javascript
test('Finder creates file and TextEdit opens it', async () => {
  render(<App />)

  // Create file via Finder
  act(() => {
    useFilesStore.getState().createFile('/Documents', 'test.txt', 'Hello')
  })

  // Open file in TextEdit
  fireEvent(window, new CustomEvent('launchApp', {
    detail: { appId: 'textedit', initialData: { path: '/Documents/test.txt' } }
  }))

  await waitFor(() => {
    expect(screen.getByDisplayValue('Hello')).toBeInTheDocument()
  })
})
```

---

### **E2E Tests** (Playwright or Cypress)

**Priority User Workflows**:

**1. Complete App Workflow**:
```javascript
// test/e2e/app-workflow.spec.ts
test('user launches Finder, creates file, opens in TextEdit', async ({ page }) => {
  await page.goto('http://localhost:5173')

  // Launch Finder from Dock
  await page.click('[data-testid="dock-icon-finder"]')

  // Create new file
  await page.click('button:has-text("New File")')
  await page.fill('input[placeholder="File name"]', 'test.txt')
  await page.click('button:has-text("Create")')

  // Double-click file to open in TextEdit
  await page.dblclick('text=test.txt')

  // Verify TextEdit window opens
  await expect(page.locator('text=TextEdit')).toBeVisible()

  // Type content
  await page.fill('[contenteditable="true"]', 'Hello World')

  // Close TextEdit
  await page.click('[data-testid="window-close-textedit"]')

  // Reopen file and verify content persisted
  await page.dblclick('text=test.txt')
  await expect(page.locator('text=Hello World')).toBeVisible()
})
```

**2. Multi-Instance Scenarios**:
```javascript
test('user opens multiple TextEdit windows', async ({ page }) => {
  await page.goto('http://localhost:5173')

  // Create two files
  await createFile(page, 'doc1.txt')
  await createFile(page, 'doc2.txt')

  // Open both files
  await page.dblclick('text=doc1.txt')
  await page.dblclick('text=doc2.txt')

  // Verify both windows visible
  const windows = await page.locator('[data-testid^="window-textedit-"]').count()
  expect(windows).toBe(2)

  // Focus first window and edit
  await page.click('[data-testid="window-textedit-1"]')
  await page.fill('[contenteditable="true"]', 'Content 1')

  // Focus second window and edit
  await page.click('[data-testid="window-textedit-2"]')
  await page.fill('[contenteditable="true"]', 'Content 2')

  // Verify independent state
  await expect(page.locator('[data-testid="window-textedit-1"] >> text=Content 1')).toBeVisible()
  await expect(page.locator('[data-testid="window-textedit-2"] >> text=Content 2')).toBeVisible()
})
```

**3. Theme Switching**:
```javascript
test('user switches themes and layout updates', async ({ page }) => {
  await page.goto('http://localhost:5173')

  // Open Control Panels
  await page.click('[data-testid="dock-icon-control-panels"]')

  // Click Appearance tab
  await page.click('text=Appearance')

  // Select XP theme
  await page.click('button:has-text("Windows XP")')

  // Verify taskbar appears at bottom
  await expect(page.locator('[data-testid="taskbar"]')).toBeVisible()

  // Verify menubar not visible
  await expect(page.locator('[data-testid="menubar"]')).not.toBeVisible()

  // Switch back to Aqua
  await page.click('button:has-text("Aqua")')

  // Verify menubar appears at top
  await expect(page.locator('[data-testid="menubar"]')).toBeVisible()

  // Verify taskbar not visible (when no foreground app)
  await page.click('[data-testid="window-close-control-panels"]')
  await expect(page.locator('[data-testid="taskbar"]')).not.toBeVisible()
})
```

**4. AI Chat Flow** (requires mock API or test environment):
```javascript
test('user chats with AI assistant', async ({ page }) => {
  await page.goto('http://localhost:5173')

  // Launch Chats
  await page.click('[data-testid="dock-icon-chats"]')

  // Type message
  await page.fill('input[placeholder="Type a message..."]', 'Hello Ryo')

  // Send message
  await page.click('button[type="submit"]')

  // Wait for AI response (streaming)
  await expect(page.locator('.message-ai')).toBeVisible({ timeout: 10000 })

  // Verify response contains text
  const response = await page.locator('.message-ai').textContent()
  expect(response.length).toBeGreaterThan(0)
})
```

**5. Persistence Testing**:
```javascript
test('user data persists across page reloads', async ({ page }) => {
  await page.goto('http://localhost:5173')

  // Create file with content
  await createFileWithContent(page, 'persistent.txt', 'Persistent Content')

  // Set custom wallpaper
  await setWallpaper(page, '/wallpapers/photos/aqua/space.jpg')

  // Switch theme
  await switchTheme(page, 'xp')

  // Reload page
  await page.reload()

  // Verify file persists
  await page.click('[data-testid="dock-icon-finder"]')
  await expect(page.locator('text=persistent.txt')).toBeVisible()

  // Verify wallpaper persists
  const bgImage = await page.locator('body').evaluate(el =>
    window.getComputedStyle(el).backgroundImage
  )
  expect(bgImage).toContain('space.jpg')

  // Verify theme persists
  await expect(page.locator('[data-testid="taskbar"]')).toBeVisible()
})
```

---

### **Visual Regression Tests** (Percy or Chromatic)

**Test Coverage**:
- All 4 themes (System 7, Aqua, XP, Win98)
- Each app window (18 apps)
- MenuBar vs Taskbar layouts
- Window controls (traffic lights vs XP buttons)
- Desktop with different wallpapers
- Responsive layouts (mobile, tablet, desktop)

**Example Configuration** (Percy):
```javascript
// test/visual/theme-comparison.spec.ts
describe('Theme Visual Regression', () => {
  const themes = ['system7', 'aqua', 'xp', 'win98']

  themes.forEach(theme => {
    test(`Desktop appearance - ${theme}`, async ({ page }) => {
      await page.goto('http://localhost:5173')
      await switchTheme(page, theme)
      await percySnapshot(page, `Desktop - ${theme}`)
    })

    test(`Finder window - ${theme}`, async ({ page }) => {
      await page.goto('http://localhost:5173')
      await switchTheme(page, theme)
      await page.click('[data-testid="dock-icon-finder"]')
      await percySnapshot(page, `Finder - ${theme}`)
    })
  })
})
```

---

## 🧹 Test Maintenance

### **Adding New Tests**

**When to Add Tests**:
- New application added → Add E2E workflow test
- New store action → Add unit test
- New component → Add integration test
- Bug fixed → Add regression test

**Test File Naming**:
```
test/
├── unit/
│   ├── stores/
│   │   └── useAppStore.test.ts
│   ├── utils/
│   │   └── displayMode.test.ts
│   └── hooks/
│       └── useWindowManager.test.ts
├── integration/
│   ├── app-launch.test.tsx
│   ├── multi-window.test.tsx
│   └── theme-switching.test.tsx
├── e2e/
│   ├── finder-workflow.spec.ts
│   ├── textedit-workflow.spec.ts
│   └── chat-workflow.spec.ts
└── visual/
    └── theme-comparison.spec.ts
```

### **Updating Tests**

**When Code Changes**:
1. Run affected tests first (unit → integration → E2E)
2. Update test assertions if behavior intentionally changed
3. Add new test cases for new functionality
4. Remove obsolete test cases for removed functionality
5. Update test fixtures when data models change

**When Tests Fail**:
1. Verify failure is legitimate (not flaky test)
2. Debug using console logs, breakpoints
3. Check for timing issues (add proper waits)
4. Verify test environment setup (API keys, mocks)
5. Update test if code behavior intentionally changed
6. Fix code if test reveals actual bug

---

## 📊 Test Coverage Goals

**Current Coverage**: 0% (manual testing only)

**Target Coverage** (when test infrastructure implemented):
- **Unit Tests**: 80% overall, 100% for stores
- **Integration Tests**: 70% for critical flows
- **E2E Tests**: 100% for core user workflows (app launch, file operations, theme switching)
- **Visual Tests**: 100% for all themes + all apps

**Priority Coverage**:
1. **useAppStore** - 100% (instance management is critical)
2. **useFilesStore** - 100% (data persistence is critical)
3. **AppManager** - 90% (orchestration logic)
4. **WindowFrame** - 80% (window management)
5. **App launch flows** - 100% (E2E)
6. **Theme switching** - 100% (E2E)

---

## 🚀 Test Execution Commands (Future)

**When test infrastructure is implemented**:

```bash
# Run all tests
bun test

# Run unit tests only
bun test:unit

# Run integration tests only
bun test:integration

# Run E2E tests only
bun test:e2e

# Run visual regression tests
bun test:visual

# Run tests in watch mode (development)
bun test:watch

# Run tests with coverage report
bun test:coverage

# Run single test file
bun test test/unit/stores/useAppStore.test.ts

# Run tests matching pattern
bun test --grep "multi-window"

# Update visual snapshots
bun test:visual --update-snapshots
```

---

## 🔧 Test Setup & Utilities (Future)

### **Test Setup Scripts**

**`test/setup.ts`** - Global test setup:
```typescript
import '@testing-library/jest-dom'
import { cleanup } from '@testing-library/react'

// Cleanup after each test
afterEach(() => {
  cleanup()
  localStorage.clear()
})

// Mock Web APIs
global.AudioContext = jest.fn()
global.MediaDevices = jest.fn()
```

**`test/fixtures/`** - Test Data:
```typescript
// test/fixtures/app-instances.ts
export const mockFinderInstance = {
  instanceId: '1',
  appId: 'finder',
  isOpen: true,
  isForeground: true,
  position: { x: 16, y: 40 },
  size: { width: 400, height: 300 },
  createdAt: Date.now(),
}

// test/fixtures/file-tree.ts
export const mockFileTree = {
  id: 'root',
  name: '/',
  type: 'folder',
  children: [
    {
      id: '1',
      name: 'Documents',
      type: 'folder',
      children: [
        {
          id: '2',
          name: 'test.txt',
          type: 'file',
          content: 'Hello World',
        },
      ],
    },
  ],
}
```

### **Test Helpers**

**`test/helpers/`** - Reusable test utilities:
```typescript
// test/helpers/app-helpers.ts
export const launchApp = (appId: AppId, initialData?: unknown) => {
  fireEvent(window, new CustomEvent('launchApp', {
    detail: { appId, initialData }
  }))
}

export const waitForAppToOpen = async (appId: AppId) => {
  await waitFor(() => {
    const instance = Object.values(useAppStore.getState().instances)
      .find(i => i.appId === appId && i.isOpen)
    expect(instance).toBeDefined()
  })
}

// test/helpers/theme-helpers.ts
export const switchTheme = (theme: 'system7' | 'aqua' | 'xp' | 'win98') => {
  act(() => {
    useThemeStore.getState().setTheme(theme)
  })
}

// test/helpers/filesystem-helpers.ts
export const createFile = (path: string, name: string, content: string) => {
  act(() => {
    useFilesStore.getState().createFile(path, name, content)
  })
}
```

---

## 🎯 Testing Best Practices

**DOs**:
- ✅ Test user behavior, not implementation details
- ✅ Use data-testid for stable selectors (avoid CSS classes)
- ✅ Test accessibility (screen reader support, keyboard navigation)
- ✅ Mock external APIs (AI endpoints, Pusher)
- ✅ Use realistic test data (not foo/bar/baz)
- ✅ Test error states and edge cases
- ✅ Keep tests independent (no shared state)
- ✅ Use descriptive test names (what + expected behavior)

**DON'Ts**:
- ❌ Test implementation details (Zustand store internals)
- ❌ Rely on timing (use waitFor, not setTimeout)
- ❌ Share state between tests (reset after each test)
- ❌ Mock too much (prefer integration tests)
- ❌ Write flaky tests (fix or remove)
- ❌ Skip cleanup (memory leaks in test suite)
- ❌ Test framework code (React, Zustand)
- ❌ Duplicate coverage (one test per behavior)

---

**Document Version**: 0.1.0
**Last Updated**: 2025-01-21
**Status**: Initial test strategy documentation (implementation pending)
