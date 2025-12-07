# Native iOS Chat Keyboard Solution

## Problem Statement

When building a chat interface in React Native, achieving native-feeling keyboard behavior is challenging:

1. **Content doesn't shift up** - When keyboard opens, messages should move up with the keyboard
2. **Animation mismatch** - JS bridge causes delays, making content lag behind keyboard
3. **Gap inconsistency** - Space between last message and composer varies between keyboard states

## Native iOS Patterns

### Pattern 1: UIScrollView with Keyboard Notifications (Recommended)

This is the standard iOS pattern used by most chat apps. It adjusts `contentInset.bottom` when the keyboard appears.

**How it works:**
1. Listen to `keyboardWillShowNotification` and `keyboardWillHideNotification`
2. Extract keyboard height and animation parameters from notification
3. Animate `contentInset.bottom` using the same animation curve as the keyboard
4. If user was near bottom, scroll to bottom after adjusting inset

**Key insight:** iOS keyboard notifications include the exact animation duration and curve used by the system keyboard. Using these values ensures your content animates in perfect sync.

### Pattern 2: inputAccessoryView (How iMessage Does It)

The `inputAccessoryView` pattern makes the keyboard and input "one unit":

1. Make your `UIViewController` return `true` for `canBecomeFirstResponder`
2. Override `inputAccessoryView` to return your composer view
3. The system automatically handles positioning

**Downside:** Requires significant architecture changes and is hard to integrate with React Native.

---

## Implementation Plan

### Phase 1: Native Chat View Module

Create an Expo module that provides a native `UITableView` (or `UIScrollView`) with built-in keyboard handling.

#### File: `LaunchChatView.swift`

```swift
import ExpoModulesCore
import UIKit

class LaunchChatView: ExpoView, UITableViewDataSource, UITableViewDelegate {
    private let tableView = UITableView()
    
    // Props from JS
    var extraBottomInset: CGFloat = 0 {
        didSet { updateContentInset(animated: false) }
    }
    
    // Internal state
    private var keyboardHeight: CGFloat = 0
    private var wasAtBottom = true
    
    // Event dispatchers
    let onScrollToBottom = EventDispatcher()
    
    required init(appContext: AppContext? = nil) {
        super.init(appContext: appContext)
        setupTableView()
        setupKeyboardObservers()
    }
    
    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
    
    deinit {
        NotificationCenter.default.removeObserver(self)
    }
    
    // MARK: - Setup
    
    private func setupTableView() {
        tableView.dataSource = self
        tableView.delegate = self
        tableView.separatorStyle = .none
        tableView.allowsSelection = false
        tableView.keyboardDismissMode = .interactive
        tableView.contentInsetAdjustmentBehavior = .never
        tableView.showsVerticalScrollIndicator = true
        tableView.alwaysBounceVertical = true
        
        // Register a generic cell
        tableView.register(UITableViewCell.self, forCellReuseIdentifier: "Cell")
        
        addSubview(tableView)
    }
    
    private func setupKeyboardObservers() {
        NotificationCenter.default.addObserver(
            self,
            selector: #selector(keyboardWillChangeFrame(_:)),
            name: UIResponder.keyboardWillChangeFrameNotification,
            object: nil
        )
    }
    
    // MARK: - Keyboard Handling
    
    @objc private func keyboardWillChangeFrame(_ notification: Notification) {
        guard let userInfo = notification.userInfo,
              let endFrame = userInfo[UIResponder.keyboardFrameEndUserInfoKey] as? CGRect,
              let duration = userInfo[UIResponder.keyboardAnimationDurationUserInfoKey] as? Double,
              let curveValue = userInfo[UIResponder.keyboardAnimationCurveUserInfoKey] as? Int
        else { return }
        
        // Check if user was at bottom BEFORE adjusting
        wasAtBottom = isNearBottom()
        
        // Calculate keyboard height (0 if keyboard is off-screen)
        let screenHeight = UIScreen.main.bounds.height
        let newKeyboardHeight = max(0, screenHeight - endFrame.origin.y)
        keyboardHeight = newKeyboardHeight
        
        // Animate with same curve as keyboard
        let curve = UIView.AnimationCurve(rawValue: curveValue) ?? .easeInOut
        let animator = UIViewPropertyAnimator(duration: duration, curve: curve) {
            self.updateContentInset(animated: false)
            
            // If was at bottom, stay at bottom
            if self.wasAtBottom {
                self.scrollToBottom(animated: false)
            }
        }
        animator.startAnimation()
    }
    
    private func isNearBottom() -> Bool {
        let contentHeight = tableView.contentSize.height
        let tableHeight = tableView.bounds.height
        let bottomInset = tableView.contentInset.bottom
        let currentOffset = tableView.contentOffset.y
        let maxOffset = max(0, contentHeight - tableHeight + bottomInset)
        
        // Consider "at bottom" if within 100pt
        return (maxOffset - currentOffset) < 100
    }
    
    private func updateContentInset(animated: Bool) {
        let bottom = keyboardHeight + extraBottomInset
        
        if animated {
            UIView.animate(withDuration: 0.25) {
                self.tableView.contentInset.bottom = bottom
                self.tableView.verticalScrollIndicatorInsets.bottom = bottom
            }
        } else {
            tableView.contentInset.bottom = bottom
            tableView.verticalScrollIndicatorInsets.bottom = bottom
        }
    }
    
    // MARK: - Public Methods
    
    func scrollToBottom(animated: Bool) {
        let contentHeight = tableView.contentSize.height
        let tableHeight = tableView.bounds.height
        let bottomInset = tableView.contentInset.bottom
        let maxOffset = max(0, contentHeight - tableHeight + bottomInset)
        
        tableView.setContentOffset(CGPoint(x: 0, y: maxOffset), animated: animated)
    }
    
    // MARK: - Layout
    
    override func layoutSubviews() {
        super.layoutSubviews()
        tableView.frame = bounds
    }
    
    // MARK: - React Native Subview Management
    
    // For a simple implementation, we won't use UITableView cells from RN
    // Instead, React children go into a container view inside the table
    
    private var contentContainer: UIView?
    
    override func insertReactSubview(_ subview: UIView!, at atIndex: Int) {
        if contentContainer == nil {
            contentContainer = UIView()
            // Add as table header or use a single cell
            tableView.tableHeaderView = contentContainer
        }
        contentContainer?.insertSubview(subview, at: atIndex)
        updateTableHeaderSize()
    }
    
    override func removeReactSubview(_ subview: UIView!) {
        subview.removeFromSuperview()
        updateTableHeaderSize()
    }
    
    override func reactSubviews() -> [UIView]! {
        return contentContainer?.subviews ?? []
    }
    
    private func updateTableHeaderSize() {
        guard let container = contentContainer else { return }
        
        // Calculate size from subviews
        var height: CGFloat = 0
        for subview in container.subviews {
            height = max(height, subview.frame.maxY)
        }
        
        container.frame = CGRect(x: 0, y: 0, width: bounds.width, height: height)
        tableView.tableHeaderView = container
    }
    
    // MARK: - UITableViewDataSource (unused but required)
    
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return 0 // We use tableHeaderView instead
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        return tableView.dequeueReusableCell(withIdentifier: "Cell", for: indexPath)
    }
}
```

### Phase 2: Module Definition

```swift
// In LaunchComposerModule.swift, add:

View(LaunchChatView.self) {
    Prop("extraBottomInset") { (view: LaunchChatView, value: CGFloat) in
        view.extraBottomInset = value
    }
    
    AsyncFunction("scrollToBottom") { (view: LaunchChatView, animated: Bool) in
        DispatchQueue.main.async {
            view.scrollToBottom(animated: animated)
        }
    }
    
    Events("onScrollToBottom")
}
```

### Phase 3: TypeScript Wrapper

```typescript
import { requireNativeView } from "expo";
import { forwardRef, useImperativeHandle, useRef, type ReactNode } from "react";
import type { ViewStyle, StyleProp } from "react-native";

const NativeView = requireNativeView("LaunchComposer_LaunchChatView");

export interface LaunchChatViewProps {
  children?: ReactNode;
  style?: StyleProp<ViewStyle>;
  extraBottomInset?: number;
}

export interface LaunchChatViewRef {
  scrollToBottom: (animated?: boolean) => void;
}

export const LaunchChatView = forwardRef<LaunchChatViewRef, LaunchChatViewProps>(
  ({ children, style, extraBottomInset = 0 }, ref) => {
    const nativeRef = useRef<any>(null);

    useImperativeHandle(ref, () => ({
      scrollToBottom: (animated = true) => {
        nativeRef.current?.scrollToBottom?.(animated);
      },
    }));

    return (
      <NativeView
        ref={nativeRef}
        style={style}
        extraBottomInset={extraBottomInset}
      >
        {children}
      </NativeView>
    );
  }
);
```

### Phase 4: Usage in Chat Screen

```tsx
import { LaunchChatView, LaunchComposer } from "../modules/launch-composer";

export default function ChatScreen() {
  const chatRef = useRef<LaunchChatViewRef>(null);
  const [composerHeight, setComposerHeight] = useState(48);
  const insets = useSafeAreaInsets();
  
  // Extra inset = composer + gap + safe area (keyboard added natively)
  const extraBottomInset = composerHeight + 8 + insets.bottom;

  return (
    <View style={{ flex: 1 }}>
      <LaunchChatView
        ref={chatRef}
        style={{ flex: 1 }}
        extraBottomInset={extraBottomInset}
      >
        <View style={{ padding: 16 }}>
          {messages.map(msg => (
            <MessageBubble key={msg.id} message={msg} />
          ))}
        </View>
      </LaunchChatView>

      <KeyboardStickyView>
        <LaunchComposer
          onSend={handleSend}
          onHeightChange={setComposerHeight}
        />
      </KeyboardStickyView>
    </View>
  );
}
```

---

## Key Implementation Details

### 1. Animation Synchronization

The keyboard notification includes animation parameters:
- `UIResponder.keyboardAnimationDurationUserInfoKey` - animation duration
- `UIResponder.keyboardAnimationCurveUserInfoKey` - animation curve (7 = keyboard curve)

Using `UIViewPropertyAnimator` with these values ensures perfect sync:

```swift
let animator = UIViewPropertyAnimator(duration: duration, curve: curve) {
    self.updateContentInset()
    if self.wasAtBottom {
        self.scrollToBottom(animated: false)
    }
}
animator.startAnimation()
```

### 2. "Was at Bottom" Check

Before adjusting the content inset, check if user was scrolled to bottom:

```swift
private func isNearBottom() -> Bool {
    let maxOffset = contentHeight - tableHeight + bottomInset
    return (maxOffset - currentOffset) < 100 // 100pt threshold
}
```

Only auto-scroll to bottom if user was already there. This prevents hijacking scroll position if user was reading earlier messages.

### 3. Interactive Keyboard Dismiss

Set `keyboardDismissMode = .interactive` on the scroll view to allow drag-to-dismiss gesture.

### 4. Content Inset vs Padding

Use `contentInset.bottom` instead of padding because:
- Native iOS property, no layout recalculation
- Works with scroll indicators
- Animates smoothly
- Doesn't affect content size calculation

---

## Alternative: Simpler JS-Only Solution

If native module complexity is too high, here's a simpler JS approach that's "good enough":

```tsx
import { LegendList } from "@legendapp/list";
import { KeyboardStickyView, useReanimatedKeyboardAnimation } from "react-native-keyboard-controller";
import Animated, { useAnimatedStyle } from "react-native-reanimated";

function ChatScreen() {
  const { height: keyboardHeight, progress } = useReanimatedKeyboardAnimation();
  const listRef = useRef(null);
  
  // Animated bottom padding that grows with keyboard
  const animatedContentStyle = useAnimatedStyle(() => ({
    paddingBottom: composerHeight + GAP + insets.bottom + keyboardHeight.value,
  }));

  // Scroll to bottom when keyboard opens (if at bottom)
  useEffect(() => {
    // Listen to keyboard changes and scroll if needed
  }, []);

  return (
    <View style={{ flex: 1 }}>
      <LegendList
        ref={listRef}
        data={messages}
        renderItem={renderMessage}
        ListFooterComponent={<Animated.View style={animatedContentStyle} />}
      />
      
      <KeyboardStickyView>
        <Composer />
      </KeyboardStickyView>
    </View>
  );
}
```

**Pros:** No native code, works today
**Cons:** Animation may not be perfectly synced, relies on Reanimated

---

## Complexity Comparison

| Approach | Lines of Code | Animation Quality | Effort |
|----------|---------------|-------------------|--------|
| JS with padding | ~50 | Good (slight lag) | Low |
| Native module (simple) | ~300 | Great | Medium |
| Native module (full) | ~1000+ | Perfect (v0 level) | High |
| inputAccessoryView | ~500 | Perfect | Very High |

---

## Recommendation

For your template product, I recommend **Phase 1-3** of the native solution above. It provides:

1. ✅ Content shifts up with keyboard
2. ✅ Animation synced with keyboard curve
3. ✅ Interactive keyboard dismiss
4. ✅ Auto-scroll only if at bottom
5. ✅ Works with any React Native content

The implementation is ~300 lines of Swift and handles the majority of cases. Edge cases (like multiple rapid keyboard shows, background/foreground transitions) can be added later.

---

## Next Steps

1. Create `LaunchChatView.swift` with the code above
2. Add to module definition
3. Create TypeScript wrapper
4. Update `ai-chat.tsx` to use `LaunchChatView`
5. Test on device
6. Iterate on edge cases

Would you like me to implement this?



