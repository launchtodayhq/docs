# Making the Chat Experience More Native

Research findings on building a native-feeling AI chat experience in React Native/Expo, inspired by Vercel's v0 iOS app and modern chat app patterns.

---

## Table of Contents

1. [Critical Issues Identified](#critical-issues-identified)
2. [Root Cause Analysis](#root-cause-analysis)
3. [Current Implementation Analysis](#current-implementation-analysis)
4. [Two Approaches to Native Chat](#two-approaches-to-native-chat)
5. [Key Technologies & Libraries](#key-technologies--libraries)
6. [Native Keyboard Handling](#native-keyboard-handling)
7. [Native Composer Options](#native-composer-options)
8. [Message List Optimizations](#message-list-optimizations)
9. [Native UI Components](#native-ui-components)
10. [Animation Strategies](#animation-strategies)
11. [Implementation Recommendations](#implementation-recommendations)

---

## Critical Issues Identified

### Issue 1: Gap Inconsistency

**Symptom:** When scrolled to the bottom with the keyboard open, there's a larger gap between the composer and the last message compared to when the keyboard is closed.

**Why adjusting gap values doesn't help:** The system uses TWO separate positioning mechanisms that don't perfectly sync:

1. `KeyboardStickyView` positions the composer (keyboard controller's internal animation)
2. `footerHeight` state creates scroll space (updates via `runOnJS` with latency)

### Issue 2: Animation Lag

**Symptom:** When the keyboard animates in/out, the content above moves slower than the keyboard, appearing to "lag behind."

**Root cause:** The `runOnJS` call in `useKeyboardAwareMessageList`:

```typescript
// This causes 1-2 frame delay
onMove: (e) => {
  "worklet";
  runOnJS(onKeyboardMove)(e.height); // ← Problem!
};
```

The round-trip: Native keyboard → UI thread (worklet) → JS thread → setState → Re-render → Bridge → Native

---

## Root Cause Analysis

### Why This Is Fundamentally Hard in JS

iOS keyboard animations use a special curve called `UIViewAnimationCurveKeyboard` (curve value = 7). This curve is:

- Not a standard bezier curve
- Not exposed to JavaScript
- Slightly different timing each iOS version
- Spring-based in iOS 18+

Even `react-native-keyboard-controller`'s worklet-based approach can only **approximate** this curve. The JS bridge introduces unavoidable latency.

### The Native Solution: `keyboardLayoutGuide`

Apple's `keyboardLayoutGuide` (iOS 15+) solves both issues perfectly because:

1. Constraints update **in the same animation block** as the keyboard
2. **Zero JS bridge delay** - it's pure Auto Layout
3. Uses the **exact same animation curve** automatically
4. Works with floating keyboards on iPad

```swift
// Native iOS code - perfect sync
view.keyboardLayoutGuide.followsUndockedKeyboard = true
composerView.bottomAnchor.constraint(equalTo: view.keyboardLayoutGuide.topAnchor).isActive = true
```

### Verdict for a Template Product

**For a template that needs "perfect" chat experience:**

- The JS approach will always have these subtle issues
- Users building apps from your template will inherit these problems
- The native module approach is the right investment

---

## Current Implementation Analysis

Your current `ai-chat.tsx` uses:

| Component         | Library                            | Status          |
| ----------------- | ---------------------------------- | --------------- |
| Message List      | `@legendapp/list` (LegendList)     | ✅ Same as v0   |
| Keyboard Handling | `react-native-keyboard-controller` | ✅ Good choice  |
| Sticky Composer   | `KeyboardStickyView`               | ✅ Works        |
| Glass Effects     | `@callstack/liquid-glass`          | ✅ iOS 26 ready |
| Animations        | `react-native-reanimated`          | ✅ Good choice  |

**Gaps identified:**

- No composable provider architecture
- Basic scroll behavior (no "pin user message to top")
- No staggered text animations for streaming
- Using JS TextInput instead of native input
- No native context menus or bottom sheets

---

## Two Approaches to Native Chat

### Approach A: Enhanced JS Implementation (v0 Style)

The Vercel v0 team stayed in React Native but optimized heavily:

```tsx
// Composable provider architecture
<ChatProvider key={chatId}>
  <ComposerHeightProvider>
    <MessageListProvider>
      <NewMessageAnimationProvider>
        <KeyboardStateProvider>{children}</KeyboardStateProvider>
      </NewMessageAnimationProvider>
    </MessageListProvider>
  </ComposerHeightProvider>
</ChatProvider>
```

**Pros:**

- No native code required
- Faster iteration
- Cross-platform by default
- Already using similar stack

**Cons:**

- Keyboard animation timing can drift from native
- More complex state management
- Potential performance overhead

### Approach B: Native Module (@launch/composer)

Build a native Expo module with:

- Native `UITextView` composer
- iOS `keyboardLayoutGuide` for pixel-perfect keyboard tracking
- Native `inputAccessoryView` for jump-to-bottom button

**Pros:**

- Pixel-perfect keyboard behavior
- Native text input performance
- Native accessory view above keyboard
- True iOS system feel

**Cons:**

- Requires dev client (no Expo Go)
- iOS-only initially (Android needs separate implementation)
- More complex to maintain
- Longer development time

---

## Key Technologies & Libraries

### Already Using ✅

| Library                            | Purpose             | Notes                                          |
| ---------------------------------- | ------------------- | ---------------------------------------------- |
| `@legendapp/list`                  | Virtualized list    | Used by v0, excellent for chat                 |
| `react-native-keyboard-controller` | Keyboard handling   | Provides `KeyboardStickyView`, keyboard events |
| `react-native-reanimated`          | Animations          | Shared values, worklets                        |
| `@callstack/liquid-glass`          | iOS 26 Liquid Glass | New for iOS 26                                 |

### Recommended Additions

| Library                         | Purpose                    | Why                             |
| ------------------------------- | -------------------------- | ------------------------------- |
| `zeego`                         | Native context menus       | Renders native `UIMenu` on iOS  |
| `react-native-ios-context-menu` | Context menu (under zeego) | Native iOS menu rendering       |
| `react-native-unistyles`        | Styling/theming            | No re-renders for theme changes |
| `burnt`                         | Native toasts              | Native iOS/Android toasts       |

### For Native Module Approach

| Tool                      | Purpose                     |
| ------------------------- | --------------------------- |
| Expo Modules API          | Create native Swift views   |
| `create-expo-module`      | Scaffold native module      |
| iOS `keyboardLayoutGuide` | Native keyboard constraints |
| `UITextView`              | Native text input           |
| `inputAccessoryView`      | Native keyboard accessory   |

---

## Native Keyboard Handling

### Current: react-native-keyboard-controller

Your current approach using `KeyboardStickyView` is solid:

```tsx
<KeyboardStickyView style={styles.stickyContainer}>
  <LiquidGlassContainerView>
    <GlassTextInput />
    <GlassIconButton />
  </LiquidGlassContainerView>
</KeyboardStickyView>
```

**Enhancements available:**

1. **Interactive keyboard dismiss** (swipe down to dismiss):

```tsx
import { KeyboardProvider } from 'react-native-keyboard-controller';

// In your app root
<KeyboardProvider statusBarTranslucent navigationBarTranslucent>
  <App />
</KeyboardProvider>

// Enable interactive dismiss
<KeyboardAwareScrollView
  keyboardDismissMode="interactive"
>
```

2. **Synchronized animations** using `useKeyboardAnimation`:

```tsx
import { useKeyboardAnimation } from "react-native-keyboard-controller";

const { height, progress } = useKeyboardAnimation();

// Animate composer position in sync with keyboard
const animatedStyle = useAnimatedStyle(() => ({
  transform: [{ translateY: -height.value }],
}));
```

### Native Alternative: keyboardLayoutGuide

For pixel-perfect iOS keyboard tracking, use Apple's `keyboardLayoutGuide` (iOS 15+):

```swift
// In a native UIViewController
view.keyboardLayoutGuide.followsUndockedKeyboard = true

NSLayoutConstraint.activate([
  contentView.bottomAnchor.constraint(
    equalTo: view.keyboardLayoutGuide.topAnchor
  ),
])
```

This automatically:

- Animates with exact keyboard curve/duration
- Handles floating keyboards on iPad
- Works with hardware keyboards
- Matches system apps perfectly

**To use in Expo:** Requires creating an Expo Module config plugin that wraps the React Native root view.

---

## Native Composer Options

### Option 1: Keep JS TextInput (Current)

Your `GlassTextInput` wraps a React Native `TextInput`:

```tsx
<LiquidGlassView>
  <TextInput
    multiline
    placeholder="Type a message..."
    // ...
  />
</LiquidGlassView>
```

**Improvements:**

- Add `blurOnSubmit={false}` for chat behavior
- Handle `onSubmitEditing` for send on return
- Add `enablesReturnKeyAutomatically` on iOS

### Option 2: Native UITextView Module

Create a native Expo Module with `UITextView`:

```swift
// LaunchComposerView.swift
final class LaunchComposerView: ExpoView, UITextViewDelegate {
  private let textView = UITextView()

  let onSend = EventDispatcher()
  let onHeightChange = EventDispatcher()

  func textViewDidChange(_ textView: UITextView) {
    updatePlaceholderVisibility()
    updateHeightIfNeeded()
  }

  // Native auto-growing behavior
  private func updateHeightIfNeeded() {
    let fitting = CGSize(width: textView.bounds.width, height: .greatestFiniteMagnitude)
    let measured = textView.sizeThatFits(fitting).height
    let clamped = min(measured, maxHeight)

    textView.isScrollEnabled = measured > maxHeight
    onHeightChange(["height": clamped])
  }
}
```

**Benefits:**

- Native text selection/editing
- Native autocorrect/spell check behavior
- Native paste menu
- Better IME support for international keyboards

### Option 3: InputAccessoryView

React Native has built-in `InputAccessoryView` support:

```tsx
import { InputAccessoryView, TextInput } from 'react-native';

<TextInput inputAccessoryViewID="composer" />

<InputAccessoryView nativeID="composer">
  <View style={styles.accessory}>
    <Button title="Send" onPress={handleSend} />
  </View>
</InputAccessoryView>
```

**Note:** This is different from a sticky composer - it attaches directly above the keyboard.

---

## Message List Optimizations

### LegendList Best Practices

v0 uses LegendList with specific optimizations:

```tsx
<LegendList
  ref={listRef}
  data={messages}
  keyExtractor={(item) => item.id}
  enableAverages={false} // Important for variable height items
  renderItem={({ item, index }) => (
    <MessageBubble message={item} index={index} />
  )}
  // Footer creates space for composer
  ListFooterComponent={() => <View style={{ height: footerHeight }} />}
/>
```

### "Pin User Message to Top" Behavior

When user sends a message, scroll so their message is near the top:

```tsx
function pinUserMessageNearTop(userIndex: number) {
  listRef.current?.scrollToIndex({
    index: userIndex,
    viewPosition: 0, // 0 = top of viewport
    viewOffset: 12, // Breathing room from top
    animated: true,
  });
}

async function onSend(text: string) {
  const userMsg = { id: uuid(), role: "user", text };
  const assistantMsg = {
    id: uuid(),
    role: "assistant",
    text: "",
    status: "streaming",
  };

  setMessages((prev) => {
    const next = [...prev, userMsg, assistantMsg];
    // Pin user message after state update
    requestAnimationFrame(() => pinUserMessageNearTop(next.length - 2));
    return next;
  });
}
```

### Track "Near Bottom" for Jump Button

```tsx
const [nearBottom, setNearBottom] = useState(true);

const onScroll = useCallback((e) => {
  const { contentOffset, contentSize, layoutMeasurement } = e.nativeEvent;
  const distanceFromBottom =
    contentSize.height - contentOffset.y - layoutMeasurement.height;
  setNearBottom(distanceFromBottom < 100);
}, []);

// Show jump button when not near bottom
{
  !nearBottom && <JumpToBottomButton onPress={scrollToBottom} />;
}
```

---

## Native UI Components

### Context Menus with Zeego

Zeego renders native `UIMenu` on iOS (with Liquid Glass on iOS 26):

```tsx
import * as ContextMenu from "zeego/context-menu";

<ContextMenu.Root>
  <ContextMenu.Trigger>
    <MessageBubble />
  </ContextMenu.Trigger>
  <ContextMenu.Content>
    <ContextMenu.Item key="copy" onSelect={handleCopy}>
      <ContextMenu.ItemTitle>Copy</ContextMenu.ItemTitle>
      <ContextMenu.ItemIcon ios={{ name: "doc.on.doc" }} />
    </ContextMenu.Item>
    <ContextMenu.Item key="reply" onSelect={handleReply}>
      <ContextMenu.ItemTitle>Reply</ContextMenu.ItemTitle>
      <ContextMenu.ItemIcon ios={{ name: "arrowshape.turn.up.left" }} />
    </ContextMenu.Item>
  </ContextMenu.Content>
</ContextMenu.Root>;
```

### Native Alerts

Use React Native's built-in `Alert` (it's native):

```tsx
import { Alert } from "react-native";

Alert.alert("Delete Message?", "This action cannot be undone.", [
  { text: "Cancel", style: "cancel" },
  { text: "Delete", style: "destructive", onPress: handleDelete },
]);
```

### Native Bottom Sheets

Use React Native Modal with `presentationStyle`:

```tsx
<Modal
  visible={visible}
  presentationStyle="formSheet"
  animationType="slide"
  onRequestClose={onClose}
>
  <View style={styles.sheetContent}>{/* Sheet content */}</View>
</Modal>
```

**Note:** v0 team patched React Native to fix modal dragging issues. These fixes are now in React Native 0.82+.

---

## Animation Strategies

### Staggered Text Fade (v0 Approach)

For streaming AI responses, limit concurrent animations:

```tsx
// Create a pool with max 4 concurrent animations
const useShouldTextFadePool = createUsePool(4);

function TextFadeInStaggered({ text }) {
  const { isActive, evict } = useIsAnimatedInPool();

  return isActive ? <FadeIn onFadedIn={evict}>{text}</FadeIn> : text;
}

// Split text into words, each word gets pool slot
function AnimatedStreamingText({ text }) {
  const words = text.split(" ");
  return words.map((word, i) => (
    <TextFadeInStaggered key={i} text={word + " "} />
  ));
}
```

### Message Entry Animation

```tsx
<Animated.View
  entering={FadeInDown.duration(250).withInitialValues({
    transform: [{ translateY: 10 }],
  })}
>
  <MessageBubble />
</Animated.View>
```

### Avoid Re-Animation on Remount

Track which messages have already animated:

```tsx
const animatedMessageIds = useRef(new Set<string>());

function MessageBubble({ message }) {
  const shouldAnimate = !animatedMessageIds.current.has(message.id);

  useEffect(() => {
    animatedMessageIds.current.add(message.id);
  }, []);

  return shouldAnimate ? (
    <Animated.View entering={FadeInDown}>
      <BubbleContent />
    </Animated.View>
  ) : (
    <BubbleContent />
  );
}
```

---

## Implementation Recommendations

Given the goal of building a **template for developers to build AI apps**, the keyboard issues must be solved properly. Here's the recommended approach:

### Recommended Path: Native Module (`@launch/composer`)

Since the two core issues (gap inconsistency + animation lag) are **fundamental limitations of JS-based keyboard handling**, the native module approach is necessary.

**What we need to build:**

```
modules/
└── launch-composer/
    ├── ios/
    │   ├── LaunchComposerModule.swift      # Expo Module definition
    │   ├── LaunchComposerView.swift        # Native UIView with composer
    │   └── KeyboardHostingController.swift # Wraps RN root for keyboardLayoutGuide
    ├── src/
    │   ├── index.ts                        # JS exports
    │   └── LaunchComposer.tsx              # React component wrapper
    ├── expo-module.config.json
    └── package.json
```

### Native Module Features

| Feature           | Implementation                              |
| ----------------- | ------------------------------------------- |
| Keyboard tracking | `keyboardLayoutGuide` constraints           |
| Composer          | Native `UITextView` wrapped in Liquid Glass |
| Send button       | Native `UIButton` with callback to JS       |
| Jump to bottom    | Native `inputAccessoryView` above keyboard  |
| Height updates    | Event emitter to JS for list footer sizing  |

### Phase 1: Scaffold the Module (1-2 days)

1. Create Expo Module using `create-expo-module`
2. Create basic `LaunchComposerView` with `UITextView`
3. Wire up `onChangeText` and `onSend` events to JS
4. Test that it works as a drop-in replacement

### Phase 2: Keyboard Layout Guide (2-3 days)

1. Create `KeyboardHostingController` that wraps React Native root
2. Build config plugin to inject it during prebuild
3. Connect composer's bottom anchor to `keyboardLayoutGuide.topAnchor`
4. Emit keyboard height changes to JS for list footer

### Phase 3: Polish (2-3 days)

1. Add `inputAccessoryView` for jump-to-bottom button
2. Match Liquid Glass styling from native side
3. Handle edge cases (split keyboard, external keyboard, iPad)
4. Test on multiple iOS versions

### Fallback: Optimized JS Approach

If native module development isn't feasible short-term, here are the best possible JS improvements:

```typescript
// Use shared values instead of runOnJS for footer height
const footerHeightSV = useSharedValue(COMPOSER_HEIGHT + GAP);

// In keyboard handler - NO runOnJS
useKeyboardHandler({
  onMove: (e) => {
    "worklet";
    footerHeightSV.value = COMPOSER_HEIGHT + GAP + e.height;
    // Scroll on UI thread if possible
  },
});

// Animated footer
const ListFooter = () => {
  const animatedStyle = useAnimatedStyle(() => ({
    height: footerHeightSV.value,
  }));
  return <Animated.View style={animatedStyle} />;
};
```

**Limitations of JS fallback:**

- Still won't match keyboard curve exactly
- May have 1-frame visual discrepancy
- Gap issue partially mitigated but not eliminated

### Decision Matrix (Updated)

| Approach      | Gap Issue  | Lag Issue     | Time      | Template Quality   |
| ------------- | ---------- | ------------- | --------- | ------------------ |
| Current JS    | ❌ Broken  | ❌ Noticeable | -         | Not template-ready |
| Optimized JS  | ⚠️ Better  | ⚠️ Reduced    | 2-3 days  | Acceptable         |
| Native Module | ✅ Perfect | ✅ Perfect    | 1-2 weeks | Production-ready   |

### Recommendation

**Build the native module.** For a template product where chat is the core experience:

1. Users will compare against iMessage, WhatsApp, ChatGPT
2. Keyboard feel is subconsciously noticed by everyone
3. The investment pays off across all apps built with the template
4. It demonstrates the template's quality and differentiation

---

---

## Native Module Implementation Details

### Step 1: Create the Expo Module

```bash
npx create-expo-module@latest launch-composer --local
```

### Step 2: Swift View Implementation

```swift
// modules/launch-composer/ios/LaunchComposerView.swift

import ExpoModulesCore
import UIKit

final class LaunchComposerView: ExpoView, UITextViewDelegate {

    // MARK: - Properties
    private let textView = UITextView()
    private let sendButton = UIButton(type: .system)
    private let placeholderLabel = UILabel()

    private var minHeight: CGFloat = 48
    private var maxHeight: CGFloat = 120

    // MARK: - Events (sent to JS)
    let onChangeText = EventDispatcher()
    let onSend = EventDispatcher()
    let onHeightChange = EventDispatcher()

    // MARK: - Setup
    required init(appContext: AppContext? = nil) {
        super.init(appContext: appContext)
        setupTextView()
        setupSendButton()
        setupPlaceholder()
        setupLayout()
    }

    private func setupTextView() {
        textView.delegate = self
        textView.font = .systemFont(ofSize: 16)
        textView.backgroundColor = .clear
        textView.textContainerInset = UIEdgeInsets(top: 12, left: 16, bottom: 12, right: 44)
        textView.isScrollEnabled = false
        addSubview(textView)
    }

    // MARK: - UITextViewDelegate
    func textViewDidChange(_ textView: UITextView) {
        placeholderLabel.isHidden = !textView.text.isEmpty
        onChangeText(["text": textView.text ?? ""])
        updateHeightIfNeeded()
    }

    private func updateHeightIfNeeded() {
        let fittingSize = CGSize(width: textView.bounds.width, height: .greatestFiniteMagnitude)
        let measuredHeight = textView.sizeThatFits(fittingSize).height
        let clampedHeight = min(max(measuredHeight, minHeight), maxHeight)

        textView.isScrollEnabled = measuredHeight > maxHeight
        onHeightChange(["height": clampedHeight])
    }

    @objc private func handleSend() {
        guard let text = textView.text, !text.isEmpty else { return }
        onSend(["text": text])
        textView.text = ""
        placeholderLabel.isHidden = false
        updateHeightIfNeeded()
    }
}
```

### Step 3: Config Plugin for KeyboardLayoutGuide

```typescript
// modules/launch-composer/plugin/withKeyboardHosting.ts

import { ConfigPlugin, withMainActivity } from "expo/config-plugins";

const withKeyboardHosting: ConfigPlugin = (config) => {
  return withMainActivity(config, async (config) => {
    // For iOS, we need to modify the AppDelegate or create a hosting controller
    // This wraps the React Native root view with keyboardLayoutGuide support
    return config;
  });
};

export default withKeyboardHosting;
```

### Step 4: JS Component Wrapper

```tsx
// modules/launch-composer/src/LaunchComposer.tsx

import { requireNativeView } from "expo";
import { useCallback, forwardRef } from "react";
import { StyleProp, ViewStyle } from "react-native";

interface LaunchComposerProps {
  placeholder?: string;
  onChangeText?: (text: string) => void;
  onSend?: (text: string) => void;
  onHeightChange?: (height: number) => void;
  style?: StyleProp<ViewStyle>;
}

const NativeComposer = requireNativeView("LaunchComposer");

export const LaunchComposer = forwardRef<any, LaunchComposerProps>(
  ({ onChangeText, onSend, onHeightChange, ...props }, ref) => {
    const handleChangeText = useCallback(
      (event: any) => {
        onChangeText?.(event.nativeEvent.text);
      },
      [onChangeText]
    );

    const handleSend = useCallback(
      (event: any) => {
        onSend?.(event.nativeEvent.text);
      },
      [onSend]
    );

    const handleHeightChange = useCallback(
      (event: any) => {
        onHeightChange?.(event.nativeEvent.height);
      },
      [onHeightChange]
    );

    return (
      <NativeComposer
        ref={ref}
        onChangeText={handleChangeText}
        onSend={handleSend}
        onHeightChange={handleHeightChange}
        {...props}
      />
    );
  }
);
```

### Step 5: Usage in ai-chat.tsx

```tsx
// After native module is built

import { LaunchComposer } from "@launch/composer";

export default function AiChatScreen() {
  const [composerHeight, setComposerHeight] = useState(48);

  return (
    <View style={styles.container}>
      <LegendList
        data={messages}
        ListFooterComponent={() => (
          <View style={{ height: composerHeight + GAP }} />
        )}
        // ... rest of list config
      />

      {/* Native composer - automatically tracks keyboard */}
      <LaunchComposer
        placeholder="Type a message..."
        onSend={handleSend}
        onHeightChange={setComposerHeight}
        style={styles.composer}
      />
    </View>
  );
}
```

---

## Alternative: Hybrid Approach

If building a full native module is too much initially, consider a **hybrid approach**:

1. Keep the JS composer UI (with Liquid Glass)
2. Create a minimal native module ONLY for keyboard height tracking
3. Use `keyboardLayoutGuide` to emit perfect height values to JS

```swift
// Minimal native module - just tracks keyboard
class KeyboardTracker: NSObject {
    static let shared = KeyboardTracker()

    func startTracking(view: UIView, callback: @escaping (CGFloat) -> Void) {
        // Use keyboardLayoutGuide to get pixel-perfect height
        view.keyboardLayoutGuide.followsUndockedKeyboard = true

        // Observe constraint changes and emit to JS
        // This gives us the exact keyboard height at every frame
    }
}
```

This would solve **Issue 2 (animation lag)** but not completely solve **Issue 1 (gap)** since the composer positioning would still be JS-based.

---

## Expo Modules API Deep Dive

This section documents how to create native iOS views using Expo Modules API, based on the official Expo documentation and common patterns.

### Overview

Expo Modules API allows you to create native modules in Swift (iOS) and Kotlin (Android) that integrate seamlessly with your Expo/React Native app. Key benefits:

- **No bridging code** - Expo handles the JS ↔ Native communication
- **Type-safe** - Props and events are strongly typed
- **Dev client compatible** - Works with `expo-dev-client`
- **Hot reload** - Native views can hot reload during development

### Module Structure

```
modules/
└── my-module/
    ├── ios/
    │   ├── MyModule.swift           # Module definition (functions, constants)
    │   ├── MyModuleView.swift       # Native UIView subclass
    │   └── MyModule.podspec         # CocoaPods spec (auto-generated)
    ├── android/
    │   └── src/main/java/expo/modules/mymodule/
    │       ├── MyModule.kt
    │       └── MyModuleView.kt
    ├── src/
    │   ├── index.ts                 # JS exports
    │   ├── MyModule.ts              # Typed module interface
    │   └── MyModuleView.tsx         # React component wrapper
    ├── expo-module.config.json      # Module configuration
    └── package.json
```

### expo-module.config.json

```json
{
  "platforms": ["ios", "android"],
  "ios": {
    "modules": ["MyModule"]
  },
  "android": {
    "modules": ["expo.modules.mymodule.MyModule"]
  }
}
```

### Swift Module Definition

The module definition file declares functions, props, events, and views:

```swift
// ios/MyModule.swift
import ExpoModulesCore

public class MyModule: Module {
    public func definition() -> ModuleDefinition {
        // Module name (used in JS: `requireNativeModule('MyModule')`)
        Name("MyModule")

        // Constants exported to JS
        Constants([
            "PI": Double.pi
        ])

        // Synchronous function
        Function("hello") { (name: String) -> String in
            return "Hello, \(name)!"
        }

        // Async function
        AsyncFunction("fetchData") { (url: String, promise: Promise) in
            // Async work...
            promise.resolve(result)
        }

        // Events that can be sent to JS
        Events("onKeyboardHeight", "onTextChange")

        // Native View definition
        View(MyModuleView.self) {
            // Props passed from JS to native
            Prop("placeholder") { (view: MyModuleView, placeholder: String) in
                view.placeholder = placeholder
            }

            Prop("text") { (view: MyModuleView, text: String) in
                view.text = text
            }

            Prop("maxHeight") { (view: MyModuleView, maxHeight: CGFloat) in
                view.maxHeight = maxHeight
            }

            // Events sent from native to JS
            Events("onChangeText", "onSend", "onHeightChange")
        }
    }
}
```

### Swift Native View

```swift
// ios/MyModuleView.swift
import ExpoModulesCore
import UIKit

class MyModuleView: ExpoView {
    // MARK: - Properties
    var placeholder: String = "" {
        didSet { placeholderLabel.text = placeholder }
    }

    var text: String {
        get { textView.text }
        set { textView.text = newValue }
    }

    var maxHeight: CGFloat = 120

    // MARK: - Event Dispatchers
    // These are automatically connected to JS by Expo
    let onChangeText = EventDispatcher()
    let onSend = EventDispatcher()
    let onHeightChange = EventDispatcher()

    // MARK: - UI Elements
    private let textView = UITextView()
    private let placeholderLabel = UILabel()
    private let sendButton = UIButton(type: .system)

    // MARK: - Initialization
    required init(appContext: AppContext? = nil) {
        super.init(appContext: appContext)
        setupViews()
        setupConstraints()
    }

    private func setupViews() {
        // TextView
        textView.delegate = self
        textView.font = .systemFont(ofSize: 16)
        textView.backgroundColor = .clear
        textView.isScrollEnabled = false
        addSubview(textView)

        // Placeholder
        placeholderLabel.textColor = .placeholderText
        placeholderLabel.font = .systemFont(ofSize: 16)
        addSubview(placeholderLabel)

        // Send button
        sendButton.setImage(UIImage(systemName: "arrow.up.circle.fill"), for: .normal)
        sendButton.addTarget(self, action: #selector(handleSend), for: .touchUpInside)
        addSubview(sendButton)
    }

    private func setupConstraints() {
        textView.translatesAutoresizingMaskIntoConstraints = false
        placeholderLabel.translatesAutoresizingMaskIntoConstraints = false
        sendButton.translatesAutoresizingMaskIntoConstraints = false

        NSLayoutConstraint.activate([
            textView.leadingAnchor.constraint(equalTo: leadingAnchor, constant: 16),
            textView.trailingAnchor.constraint(equalTo: sendButton.leadingAnchor, constant: -8),
            textView.topAnchor.constraint(equalTo: topAnchor, constant: 8),
            textView.bottomAnchor.constraint(equalTo: bottomAnchor, constant: -8),

            placeholderLabel.leadingAnchor.constraint(equalTo: textView.leadingAnchor, constant: 5),
            placeholderLabel.centerYAnchor.constraint(equalTo: textView.centerYAnchor),

            sendButton.trailingAnchor.constraint(equalTo: trailingAnchor, constant: -16),
            sendButton.bottomAnchor.constraint(equalTo: bottomAnchor, constant: -12),
            sendButton.widthAnchor.constraint(equalToConstant: 32),
            sendButton.heightAnchor.constraint(equalToConstant: 32),
        ])
    }

    // MARK: - Actions
    @objc private func handleSend() {
        guard !textView.text.isEmpty else { return }

        // Send event to JS with payload
        onSend([
            "text": textView.text ?? ""
        ])

        // Clear text
        textView.text = ""
        placeholderLabel.isHidden = false
        updateHeight()
    }

    private func updateHeight() {
        let fittingSize = CGSize(width: textView.bounds.width, height: .greatestFiniteMagnitude)
        let size = textView.sizeThatFits(fittingSize)
        let newHeight = min(max(size.height + 16, 48), maxHeight)

        textView.isScrollEnabled = size.height > maxHeight

        // Emit height change to JS
        onHeightChange([
            "height": newHeight
        ])
    }
}

// MARK: - UITextViewDelegate
extension MyModuleView: UITextViewDelegate {
    func textViewDidChange(_ textView: UITextView) {
        placeholderLabel.isHidden = !textView.text.isEmpty

        // Emit text change to JS
        onChangeText([
            "text": textView.text ?? ""
        ])

        updateHeight()
    }
}
```

### JavaScript/TypeScript Wrapper

```typescript
// src/index.ts
import { requireNativeModule, requireNativeView } from "expo";

// For functions and constants
export const MyModule = requireNativeModule("MyModule");

// For native views
export { MyModuleView } from "./MyModuleView";
```

```tsx
// src/MyModuleView.tsx
import { requireNativeView } from "expo";
import { forwardRef, useCallback } from "react";
import type { StyleProp, ViewStyle } from "react-native";

// Types for the native view props
export interface MyModuleViewProps {
  placeholder?: string;
  text?: string;
  maxHeight?: number;
  onChangeText?: (text: string) => void;
  onSend?: (text: string) => void;
  onHeightChange?: (height: number) => void;
  style?: StyleProp<ViewStyle>;
}

// Native event types
interface NativeEvent<T> {
  nativeEvent: T;
}

// Get the native view
const NativeView = requireNativeView<MyModuleViewProps>("MyModule");

// React wrapper component
export const MyModuleView = forwardRef<typeof NativeView, MyModuleViewProps>(
  ({ onChangeText, onSend, onHeightChange, ...props }, ref) => {
    const handleChangeText = useCallback(
      (event: NativeEvent<{ text: string }>) => {
        onChangeText?.(event.nativeEvent.text);
      },
      [onChangeText]
    );

    const handleSend = useCallback(
      (event: NativeEvent<{ text: string }>) => {
        onSend?.(event.nativeEvent.text);
      },
      [onSend]
    );

    const handleHeightChange = useCallback(
      (event: NativeEvent<{ height: number }>) => {
        onHeightChange?.(event.nativeEvent.height);
      },
      [onHeightChange]
    );

    return (
      <NativeView
        ref={ref}
        onChangeText={handleChangeText}
        onSend={handleSend}
        onHeightChange={handleHeightChange}
        {...props}
      />
    );
  }
);
```

### Config Plugin for Native Setup

Config plugins allow you to modify native project files during `expo prebuild`:

```typescript
// plugin/withMyModule.ts
import {
  ConfigPlugin,
  withAppDelegate,
  withInfoPlist,
  IOSConfig,
} from "expo/config-plugins";

const withMyModule: ConfigPlugin = (config) => {
  // Modify Info.plist
  config = withInfoPlist(config, (config) => {
    config.modResults.NSCameraUsageDescription =
      config.modResults.NSCameraUsageDescription || "Allow camera access";
    return config;
  });

  // Modify AppDelegate
  config = withAppDelegate(config, (config) => {
    const { modResults } = config;

    // Add import at the top
    if (!modResults.contents.includes('#import "MyModule-Swift.h"')) {
      modResults.contents = modResults.contents.replace(
        '#import "AppDelegate.h"',
        '#import "AppDelegate.h"\n#import "MyModule-Swift.h"'
      );
    }

    // Add code to application:didFinishLaunchingWithOptions:
    // This is where you'd set up keyboardLayoutGuide wrapping

    return config;
  });

  return config;
};

export default withMyModule;
```

### Using keyboardLayoutGuide with Config Plugin

To use `keyboardLayoutGuide` for the entire app, you need to wrap the React Native root view controller:

```swift
// ios/KeyboardHostingController.swift
import UIKit
import React

class KeyboardHostingController: UIViewController {
    private let reactViewController: UIViewController

    init(reactViewController: UIViewController) {
        self.reactViewController = reactViewController
        super.init(nibName: nil, bundle: nil)
    }

    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }

    override func viewDidLoad() {
        super.viewDidLoad()

        // Add React Native view controller as child
        addChild(reactViewController)
        view.addSubview(reactViewController.view)
        reactViewController.didMove(toParent: self)

        // Setup constraints with keyboardLayoutGuide
        reactViewController.view.translatesAutoresizingMaskIntoConstraints = false

        NSLayoutConstraint.activate([
            reactViewController.view.topAnchor.constraint(equalTo: view.topAnchor),
            reactViewController.view.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            reactViewController.view.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            // This is the magic - bottom is pinned to keyboard guide
            reactViewController.view.bottomAnchor.constraint(
                equalTo: view.keyboardLayoutGuide.topAnchor
            )
        ])

        // Follow undocked keyboards on iPad
        view.keyboardLayoutGuide.followsUndockedKeyboard = true
    }
}
```

### Installation & Usage

1. **Create the module:**

```bash
npx create-expo-module@latest launch-composer --local
```

2. **Add to your app.json:**

```json
{
  "expo": {
    "plugins": ["./modules/launch-composer/plugin/withMyModule"]
  }
}
```

3. **Prebuild:**

```bash
npx expo prebuild --clean
```

4. **Use in your app:**

```tsx
import { MyModuleView } from "./modules/launch-composer";

export default function ChatScreen() {
  return (
    <View style={{ flex: 1 }}>
      <MessageList />
      <MyModuleView
        placeholder="Type a message..."
        onSend={(text) => sendMessage(text)}
        onHeightChange={(height) => setComposerHeight(height)}
      />
    </View>
  );
}
```

### Key Expo Modules API Patterns

| Pattern                           | Usage                             |
| --------------------------------- | --------------------------------- |
| `Name("ModuleName")`              | Define module name for JS imports |
| `Function("name") { }`            | Sync function callable from JS    |
| `AsyncFunction("name") { }`       | Async function with Promise       |
| `Events("event1", "event2")`      | Events that can be emitted to JS  |
| `View(ViewClass.self) { }`        | Define a native view              |
| `Prop("name") { }`                | Define props for native views     |
| `let onEvent = EventDispatcher()` | Emit events from view to JS       |
| `requireNativeModule()`           | Import module in JS               |
| `requireNativeView()`             | Import native view in JS          |

### Event Dispatcher Pattern

```swift
// In your view class
let onMyEvent = EventDispatcher()

// Emit an event with data
onMyEvent([
    "key1": value1,
    "key2": value2
])

// In JS, receive as:
<MyView
  onMyEvent={(event) => {
    const { key1, key2 } = event.nativeEvent;
  }}
/>
```

---

---

## Complete @launch/composer Implementation Plan

Based on all research, here's the complete plan for building a native composer module that solves both keyboard issues.

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    React Native App                         │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              LegendList (Messages)                   │   │
│  │  ┌─────────────────────────────────────────────┐    │   │
│  │  │ Message 1                                    │    │   │
│  │  │ Message 2                                    │    │   │
│  │  │ ...                                          │    │   │
│  │  │ Footer (height = composerHeight + gap)       │    │   │
│  │  └─────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         LaunchComposerView (Native)                 │   │
│  │  ┌─────────────────────┐  ┌───┐                    │   │
│  │  │    UITextView       │  │ ⬆ │ Send               │   │
│  │  └─────────────────────┘  └───┘                    │   │
│  │  bottomAnchor → keyboardLayoutGuide.topAnchor       │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                     iOS Keyboard                            │
└─────────────────────────────────────────────────────────────┘
```

### Step-by-Step Implementation

#### Step 1: Create the Module

```bash
cd apps/mobile
npx create-expo-module@latest launch-composer --local
```

This creates `modules/launch-composer/` with the basic structure.

#### Step 2: Module Definition (LaunchComposerModule.swift)

```swift
// modules/launch-composer/ios/LaunchComposerModule.swift
import ExpoModulesCore

public class LaunchComposerModule: Module {
    public func definition() -> ModuleDefinition {
        Name("LaunchComposer")

        // Constants
        Constants([
            "defaultMinHeight": 48.0,
            "defaultMaxHeight": 120.0
        ])

        // View
        View(LaunchComposerView.self) {
            // Props
            Prop("placeholder") { (view: LaunchComposerView, value: String) in
                view.placeholder = value
            }

            Prop("text") { (view: LaunchComposerView, value: String) in
                view.text = value
            }

            Prop("minHeight") { (view: LaunchComposerView, value: CGFloat) in
                view.minHeight = value
            }

            Prop("maxHeight") { (view: LaunchComposerView, value: CGFloat) in
                view.maxHeight = value
            }

            Prop("sendButtonEnabled") { (view: LaunchComposerView, value: Bool) in
                view.sendButtonEnabled = value
            }

            // Events
            Events(
                "onChangeText",
                "onSend",
                "onHeightChange",
                "onKeyboardHeightChange",
                "onFocus",
                "onBlur"
            )
        }
    }
}
```

#### Step 3: Native View (LaunchComposerView.swift)

```swift
// modules/launch-composer/ios/LaunchComposerView.swift
import ExpoModulesCore
import UIKit

class LaunchComposerView: ExpoView {

    // MARK: - Props
    var placeholder: String = "Type a message..." {
        didSet { placeholderLabel.text = placeholder }
    }

    var text: String {
        get { textView.text ?? "" }
        set {
            textView.text = newValue
            placeholderLabel.isHidden = !newValue.isEmpty
            updateHeight()
        }
    }

    var minHeight: CGFloat = 48 { didSet { updateHeight() } }
    var maxHeight: CGFloat = 120 { didSet { updateHeight() } }

    var sendButtonEnabled: Bool = true {
        didSet { updateSendButtonState() }
    }

    // MARK: - Events
    let onChangeText = EventDispatcher()
    let onSend = EventDispatcher()
    let onHeightChange = EventDispatcher()
    let onKeyboardHeightChange = EventDispatcher()
    let onFocus = EventDispatcher()
    let onBlur = EventDispatcher()

    // MARK: - UI
    private let containerView = UIView()
    private let textView = UITextView()
    private let placeholderLabel = UILabel()
    private let sendButton = UIButton(type: .system)

    // MARK: - Keyboard tracking
    private var keyboardLayoutConstraint: NSLayoutConstraint?
    private var currentKeyboardHeight: CGFloat = 0

    // MARK: - Init
    required init(appContext: AppContext? = nil) {
        super.init(appContext: appContext)
        setupUI()
        setupKeyboardTracking()
    }

    deinit {
        NotificationCenter.default.removeObserver(self)
    }

    // MARK: - Setup
    private func setupUI() {
        backgroundColor = .clear

        // Container with glass effect (if available on iOS 26+)
        containerView.backgroundColor = UIColor.systemBackground.withAlphaComponent(0.8)
        containerView.layer.cornerRadius = 24
        containerView.clipsToBounds = true
        addSubview(containerView)

        // TextView
        textView.delegate = self
        textView.font = .systemFont(ofSize: 16)
        textView.backgroundColor = .clear
        textView.textContainerInset = UIEdgeInsets(top: 12, left: 12, bottom: 12, right: 44)
        textView.isScrollEnabled = false
        textView.returnKeyType = .default
        containerView.addSubview(textView)

        // Placeholder
        placeholderLabel.text = placeholder
        placeholderLabel.font = .systemFont(ofSize: 16)
        placeholderLabel.textColor = .placeholderText
        containerView.addSubview(placeholderLabel)

        // Send button
        let config = UIImage.SymbolConfiguration(pointSize: 28, weight: .medium)
        let image = UIImage(systemName: "arrow.up.circle.fill", withConfiguration: config)
        sendButton.setImage(image, for: .normal)
        sendButton.tintColor = .systemBlue
        sendButton.addTarget(self, action: #selector(handleSend), for: .touchUpInside)
        containerView.addSubview(sendButton)

        setupConstraints()
        updateSendButtonState()
    }

    private func setupConstraints() {
        containerView.translatesAutoresizingMaskIntoConstraints = false
        textView.translatesAutoresizingMaskIntoConstraints = false
        placeholderLabel.translatesAutoresizingMaskIntoConstraints = false
        sendButton.translatesAutoresizingMaskIntoConstraints = false

        NSLayoutConstraint.activate([
            // Container fills the view
            containerView.leadingAnchor.constraint(equalTo: leadingAnchor),
            containerView.trailingAnchor.constraint(equalTo: trailingAnchor),
            containerView.topAnchor.constraint(equalTo: topAnchor),
            containerView.bottomAnchor.constraint(equalTo: bottomAnchor),

            // TextView
            textView.leadingAnchor.constraint(equalTo: containerView.leadingAnchor),
            textView.trailingAnchor.constraint(equalTo: containerView.trailingAnchor),
            textView.topAnchor.constraint(equalTo: containerView.topAnchor),
            textView.bottomAnchor.constraint(equalTo: containerView.bottomAnchor),

            // Placeholder
            placeholderLabel.leadingAnchor.constraint(equalTo: textView.leadingAnchor, constant: 17),
            placeholderLabel.centerYAnchor.constraint(equalTo: containerView.centerYAnchor),

            // Send button
            sendButton.trailingAnchor.constraint(equalTo: containerView.trailingAnchor, constant: -8),
            sendButton.bottomAnchor.constraint(equalTo: containerView.bottomAnchor, constant: -8),
            sendButton.widthAnchor.constraint(equalToConstant: 32),
            sendButton.heightAnchor.constraint(equalToConstant: 32),
        ])
    }

    // MARK: - Keyboard Layout Guide
    private func setupKeyboardTracking() {
        // Use keyboardLayoutGuide when view is added to window
        // This is the key to pixel-perfect keyboard tracking
    }

    override func didMoveToWindow() {
        super.didMoveToWindow()

        guard let window = window else { return }

        // Setup keyboard layout guide constraint
        // This makes our view automatically move with the keyboard
        if #available(iOS 15.0, *) {
            keyboardLayoutConstraint?.isActive = false

            // The composer's bottom should be above the keyboard
            keyboardLayoutConstraint = bottomAnchor.constraint(
                equalTo: window.keyboardLayoutGuide.topAnchor,
                constant: -8
            )
            keyboardLayoutConstraint?.isActive = true

            // Track keyboard height changes
            window.keyboardLayoutGuide.followsUndockedKeyboard = true

            // Observe keyboard guide changes to emit to JS
            observeKeyboardGuide(in: window)
        }
    }

    @available(iOS 15.0, *)
    private func observeKeyboardGuide(in window: UIWindow) {
        // Use CADisplayLink to track keyboard position at 60fps
        let displayLink = CADisplayLink(target: self, selector: #selector(trackKeyboard))
        displayLink.add(to: .main, forMode: .common)
    }

    @objc private func trackKeyboard() {
        guard let window = window else { return }

        if #available(iOS 15.0, *) {
            // Get keyboard height from the guide
            let guideFrame = window.keyboardLayoutGuide.layoutFrame
            let keyboardHeight = window.bounds.height - guideFrame.minY

            // Only emit if changed
            if keyboardHeight != currentKeyboardHeight {
                currentKeyboardHeight = keyboardHeight
                onKeyboardHeightChange([
                    "height": keyboardHeight
                ])
            }
        }
    }

    // MARK: - Actions
    @objc private func handleSend() {
        guard !textView.text.isEmpty else { return }

        // Haptic feedback
        let generator = UIImpactFeedbackGenerator(style: .light)
        generator.impactOccurred()

        // Emit to JS
        onSend([
            "text": textView.text ?? ""
        ])

        // Clear
        textView.text = ""
        placeholderLabel.isHidden = false
        updateHeight()
        updateSendButtonState()
    }

    private func updateHeight() {
        let fittingSize = CGSize(
            width: textView.bounds.width > 0 ? textView.bounds.width : 300,
            height: .greatestFiniteMagnitude
        )
        let size = textView.sizeThatFits(fittingSize)
        let newHeight = min(max(size.height, minHeight), maxHeight)

        textView.isScrollEnabled = size.height > maxHeight

        onHeightChange([
            "height": newHeight
        ])
    }

    private func updateSendButtonState() {
        let hasText = !textView.text.isEmpty
        sendButton.isEnabled = sendButtonEnabled && hasText
        sendButton.alpha = sendButton.isEnabled ? 1.0 : 0.3
    }
}

// MARK: - UITextViewDelegate
extension LaunchComposerView: UITextViewDelegate {
    func textViewDidChange(_ textView: UITextView) {
        placeholderLabel.isHidden = !textView.text.isEmpty

        onChangeText([
            "text": textView.text ?? ""
        ])

        updateHeight()
        updateSendButtonState()
    }

    func textViewDidBeginEditing(_ textView: UITextView) {
        onFocus([:])
    }

    func textViewDidEndEditing(_ textView: UITextView) {
        onBlur([:])
    }
}
```

#### Step 4: JavaScript Component

```tsx
// modules/launch-composer/src/LaunchComposer.tsx
import { requireNativeView } from "expo";
import { forwardRef, useCallback, useImperativeHandle, useRef } from "react";
import type { StyleProp, ViewStyle, NativeSyntheticEvent } from "react-native";

export interface LaunchComposerProps {
  placeholder?: string;
  text?: string;
  minHeight?: number;
  maxHeight?: number;
  sendButtonEnabled?: boolean;
  onChangeText?: (text: string) => void;
  onSend?: (text: string) => void;
  onHeightChange?: (height: number) => void;
  onKeyboardHeightChange?: (height: number) => void;
  onFocus?: () => void;
  onBlur?: () => void;
  style?: StyleProp<ViewStyle>;
}

export interface LaunchComposerRef {
  focus: () => void;
  blur: () => void;
  clear: () => void;
}

interface TextEvent {
  text: string;
}
interface HeightEvent {
  height: number;
}

const NativeComposer = requireNativeView<LaunchComposerProps>("LaunchComposer");

export const LaunchComposer = forwardRef<
  LaunchComposerRef,
  LaunchComposerProps
>((props, ref) => {
  const {
    onChangeText,
    onSend,
    onHeightChange,
    onKeyboardHeightChange,
    onFocus,
    onBlur,
    ...rest
  } = props;

  const nativeRef = useRef(null);

  useImperativeHandle(ref, () => ({
    focus: () => {
      // Call native focus method
    },
    blur: () => {
      // Call native blur method
    },
    clear: () => {
      // Call native clear method
    },
  }));

  const handleChangeText = useCallback(
    (event: NativeSyntheticEvent<TextEvent>) => {
      onChangeText?.(event.nativeEvent.text);
    },
    [onChangeText]
  );

  const handleSend = useCallback(
    (event: NativeSyntheticEvent<TextEvent>) => {
      onSend?.(event.nativeEvent.text);
    },
    [onSend]
  );

  const handleHeightChange = useCallback(
    (event: NativeSyntheticEvent<HeightEvent>) => {
      onHeightChange?.(event.nativeEvent.height);
    },
    [onHeightChange]
  );

  const handleKeyboardHeightChange = useCallback(
    (event: NativeSyntheticEvent<HeightEvent>) => {
      onKeyboardHeightChange?.(event.nativeEvent.height);
    },
    [onKeyboardHeightChange]
  );

  const handleFocus = useCallback(() => {
    onFocus?.();
  }, [onFocus]);

  const handleBlur = useCallback(() => {
    onBlur?.();
  }, [onBlur]);

  return (
    <NativeComposer
      ref={nativeRef}
      onChangeText={handleChangeText}
      onSend={handleSend}
      onHeightChange={handleHeightChange}
      onKeyboardHeightChange={handleKeyboardHeightChange}
      onFocus={handleFocus}
      onBlur={handleBlur}
      {...rest}
    />
  );
});
```

#### Step 5: Usage in Chat Screen

```tsx
// apps/mobile/app/ai-chat.tsx
import { useState, useRef, useCallback } from "react";
import { View, StyleSheet } from "react-native";
import { LegendList, LegendListRef } from "@legendapp/list";
import { LaunchComposer } from "../modules/launch-composer";

export default function AiChatScreen() {
  const [messages, setMessages] = useState<Message[]>([]);
  const [composerHeight, setComposerHeight] = useState(48);
  const [keyboardHeight, setKeyboardHeight] = useState(0);
  const listRef = useRef<LegendListRef>(null);

  // Footer height now updates smoothly with native keyboard tracking
  const footerHeight = composerHeight + 16 + keyboardHeight;

  const handleSend = useCallback(
    (text: string) => {
      const userMessage = { id: Date.now().toString(), text, role: "user" };
      setMessages((prev) => [...prev, userMessage]);

      // Scroll to show user message at top
      setTimeout(() => {
        listRef.current?.scrollToIndex({
          index: messages.length,
          viewPosition: 0,
          viewOffset: 12,
          animated: true,
        });
      }, 50);
    },
    [messages.length]
  );

  return (
    <View style={styles.container}>
      <LegendList
        ref={listRef}
        data={messages}
        keyExtractor={(item) => item.id}
        renderItem={({ item }) => <MessageBubble message={item} />}
        ListFooterComponent={() => <View style={{ height: footerHeight }} />}
        contentContainerStyle={styles.listContent}
      />

      {/* Native composer - pixel-perfect keyboard tracking! */}
      <LaunchComposer
        placeholder="Type a message..."
        onSend={handleSend}
        onHeightChange={setComposerHeight}
        onKeyboardHeightChange={setKeyboardHeight}
        style={styles.composer}
      />
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
  },
  listContent: {
    padding: 16,
  },
  composer: {
    position: "absolute",
    left: 16,
    right: 16,
    bottom: 0, // keyboardLayoutGuide handles the rest!
  },
});
```

### Why This Solves Both Issues

**Issue 1 (Gap inconsistency):**

- The composer's `bottomAnchor` is directly constrained to `keyboardLayoutGuide.topAnchor`
- The footer height is updated via `onKeyboardHeightChange` at 60fps
- Both systems update in the same animation frame

**Issue 2 (Animation lag):**

- `keyboardLayoutGuide` uses the exact same animation curve as the keyboard
- No JS bridge latency - the constraint update is native
- CADisplayLink tracks position at 60fps for the footer height

### Timeline

| Phase     | Task                            | Duration    |
| --------- | ------------------------------- | ----------- |
| 1         | Module scaffold + basic view    | 1 day       |
| 2         | keyboardLayoutGuide integration | 2 days      |
| 3         | Event system + JS wrapper       | 1 day       |
| 4         | Testing + edge cases            | 2 days      |
| 5         | Polish + documentation          | 1 day       |
| **Total** |                                 | **~1 week** |

---

## References

- [Vercel Blog: How we built the v0 iOS app](https://vercel.com/blog/how-we-built-the-v0-ios-app)
- [Apple: keyboardLayoutGuide](https://developer.apple.com/documentation/uikit/uiview/keyboardlayoutguide)
- [Apple: inputAccessoryView](https://developer.apple.com/documentation/uikit/uiresponder/inputaccessoryview)
- [Expo Modules API](https://docs.expo.dev/modules/module-api/)
- [Expo Native Views](https://docs.expo.dev/modules/native-view/)
- [Expo Config Plugins](https://docs.expo.dev/config-plugins/introduction/)
- [react-native-keyboard-controller](https://github.com/kirillzyusko/react-native-keyboard-controller)
- [Zeego - Native Menus](https://github.com/nandorojo/zeego)
- [LegendList](https://github.com/LegendApp/legend-list)
- [Liquid Glass (Callstack)](https://github.com/callstack/react-native-liquid-glass)
