# 🧰 The Building Blocks Cheat Sheet

> All the cross-cutting concepts that come up in EVERY mobile system design interview.
> This is your "last night before the interview" speed-run file.

---

## 🏛️ ARCHITECTURE PATTERNS

---

### Q: "Walk me through your architecture." What's the fastest way to sound smart?

**A:** Memorize this sentence and customize it per problem:

> *"I'd use a layered architecture with Unidirectional Data Flow. The UI layer has screens driven by ViewModels that expose immutable state. The data layer has repositories as the single source of truth, coordinating between local and remote data sources."*

That's it. That covers 80% of what interviewers want to hear. Now let's break it down:

**The Big Five patterns:**

| Pattern | Plain English | Why it matters |
|---------|--------------|----------------|
| **Separation of Concerns** | Each class does ONE thing well | Screens don't fetch data. Repositories don't draw UI. Everyone stays in their lane. |
| **Unidirectional Data Flow** | Data goes one way: Source → Repo → ViewModel → UI | No spaghetti. Easy to debug. You always know where data came from. |
| **Reactive Programming** | "Hey data, let me know when you change" | Instead of asking "is there new data?" 1000 times, you just subscribe and wait. |
| **Immutable State** | Once created, state objects can't be changed | No sneaky mutations. No race conditions. New state = new object. |
| **Single Source of Truth** | One owner per data type | The ViewModel doesn't modify messages directly. Only the MessagesRepository can. Everyone else reads. |

---

### Q: What's the difference between MVC, MVVM, and MVI?

**A:** These are all ways to organize the **UI layer**. Think of them as different seating arrangements at a restaurant — same food, different vibes:

| Pattern | How it works | Feel |
|---------|-------------|------|
| **MVC** | Controller handles everything. View and Model are separate but Controller is a god object. | Old school. Controller becomes massive. |
| **MVVM** | ViewModel holds state and logic. View observes ViewModel reactively. No direct Model→View link. | The default for modern apps. Clean, testable. |
| **MVI** | Like MVVM but stricter. State is ONE immutable object. Every user action is an "Intent" that produces new state. | Most predictable. Redux vibes. |

**Interview default:** Say MVVM. It's what Android Jetpack and SwiftUI are built around. If they ask for something more structured, upgrade to MVI.

---

### Q: When should I mention modularization?

**A:** When the app is complex enough that one team can't own everything. Mention it casually:

*"For a large team, I'd split this into feature modules — `:feed`, `:chat`, `:profile` — with shared `:core-network` and `:design-system` modules. This gives us parallel builds, independent team ownership, and the ability to test features in isolation."*

Boom. You just showed you've worked on real production apps, not just tutorials.

---

## 💾 DATA MANAGEMENT

---

### Q: Quick — what storage option do I use for what?

**A:** Mental model:

| I need to store... | Use this | Think of it as... |
|-------------------|----------|-------------------|
| User preferences, tokens, simple flags | **Key-Value Store** (SharedPrefs / UserDefaults) | A sticky note on your desk |
| Messages, posts, files metadata | **Relational Database** (Room / Core Data / SQLite) | A filing cabinet with folders |
| Downloaded photos, PDFs, videos | **File Storage** (internal app sandbox) | The drawer under your desk |
| Recently viewed items for speed | **In-Memory Cache** (LRU HashMap) | Your short-term memory |
| Passwords, API keys, encryption keys | **Secure Storage** (Keystore / Keychain) | A locked safe |

---

### Q: How do you explain caching strategy without sounding like a textbook?

**A:** Tell a pizza story 🍕:

> Imagine you order pizza every Friday. Here's how caching works:
>
> **No cache** = Call Domino's, wait 30 min, every single time.
>
> **In-memory cache** = You keep leftover pizza in the kitchen. If it's still warm → eat it! (Fast, but limited space. Pizza goes bad after a while.)
>
> **Disk cache** = You store pizza in the fridge. Lasts longer, but you need to microwave it. (Slower than kitchen counter, but way faster than ordering new.)
>
> **Two-level cache** = Check kitchen counter first → then fridge → then call Domino's. This is what real apps do.

In code terms:
```
Request for data:
  1. Check in-memory cache → HIT? Return instantly ⚡
  2. Check disk cache      → HIT? Load to memory + return 🏎️
  3. Fetch from network    → Store in both caches + return 📡
```

**Eviction** = throwing out old pizza:
- **LRU**: Throw out the pizza nobody's touched the longest
- **TTL**: Throw out pizza older than 2 hours, regardless
- **Size-based**: Fridge full? Oldest pizza goes

---

### Q: What's the deal with pagination? Offset vs Cursor — when do I use which?

**A:**

**Offset** = "Give me items 21 through 40"
```
GET /v1/items?page=2&limit=20
```
- ✅ Simple. Everyone understands it.
- ❌ Breaks if items get added/deleted between requests. You'll see duplicates or miss items.
- 👍 Good for: Admin panels, search results, order history (things that don't change mid-browse)

**Cursor** = "Give me the 20 items after this bookmark"
```
GET /v1/items?cursor=eyJpZCI6MTAwfQ&limit=20
```
- ✅ Rock-solid even when data is constantly changing
- ❌ Can't jump to "page 7" directly
- 👍 Good for: Social feeds, chat history, anything real-time

**Interview tip:** 90% of the time, cursor is the better answer. Say: *"I'll use cursor-based pagination because the data is dynamic and offset would cause inconsistencies."* Then they know you actually thought about it.

---

## 🌐 NETWORKING

---

### Q: Give me the one-slide summary of all network protocols I need to know.

**A:**

```
               ┌─────── You need ───────┐
               │                         │
        One-time data?          Continuous data?
               │                         │
            REST/HTTP            ┌───────┴───────┐
               │                 │               │
          Standard CRUD    Server→Client    Bidirectional
               │                only               │
          GET/POST/PUT/      SSE (Server-     WebSocket
          DELETE             Sent Events)    (ws:// or wss://)
                                │               │
                          Feed updates,    Chat messages,
                          notifications    live prices,
                                           typing indicators
```

**Bonus:** If they mention extreme scale (Google/YouTube level), drop **gRPC** — binary protocol, faster serialization, great for server-to-server AND mobile.

---

### Q: What do I say about error handling and retry logic?

**A:** Three magic words: **Exponential Backoff with Jitter**.

When a request fails:
- 1st retry: wait 1 second
- 2nd retry: wait 2 seconds
- 3rd retry: wait 4 seconds
- 4th retry: wait 8 seconds
- (with random jitter so 10,000 clients don't all retry at the exact same moment)

```
delay = min(baseDelay * 2^attempt + random(0, 1000ms), maxDelay)
```

Also mention:
- **Idempotency keys** on POST requests (so retries don't create duplicates)
- **Circuit breaker pattern** (if the server is down, stop hammering it — try again in 30 seconds)
- **User feedback** ("Connection lost. Retrying..." vs. silent failure)

---

### Q: What about authentication? Quick rundown?

**A:**

| Method | How | When |
|--------|-----|------|
| **JWT Bearer Token** | Login → get token → send in `Authorization: Bearer <token>` header | Default for most APIs |
| **OAuth 2.0** | "Sign in with Google/Apple" | Social login |
| **Token Refresh** | Access token expires (15 min) → use refresh token to get new one → refresh expires (30 days) → re-login | Keeps sessions alive securely |
| **Certificate Pinning** | Client only trusts YOUR server's specific certificate | Banking, fintech (prevents MITM attacks) |

---

## 🚀 FEATURE DEVELOPMENT

---

### Q: What are feature flags and why should I mention them?

**A:** Feature flags are on/off switches for features — controllable from the server, no app update needed.

```json
{
  "dark_mode_v2": true,
  "new_checkout_flow": false,
  "ai_recommendations": true
}
```

Why they're gold in interviews:
- **Gradual rollout**: Enable for 5% of users → monitor → increase to 50% → 100%
- **Kill switch**: Feature is crashing? Turn it off instantly. No app store update.
- **A/B testing**: 50% of users get Feature A, 50% get Feature B. Measure which performs better.
- **Different configs per market**: Feature X enabled in US, disabled in EU (GDPR reasons)

If you mention feature flags unprompted, interviewers assume you've shipped real products.

---

### Q: What about push notifications — anything beyond "use FCM/APNs"?

**A:** Yes! Here's what makes your answer stand out:

- **Don't abuse them.** Users will disable notifications (or uninstall your app) if you spam them.
- **Rich notifications**: Images, action buttons, expandable text — not just plain text
- **Silent push**: A notification that wakes the app in the background to sync data — user never sees it. Great for chat apps to pre-load messages.
- **Notification channels** (Android): Group notifications by type (Orders, Promotions, Chat). Users can mute Promotions but keep Chat alerts.
- **Always have a server-side component**: The backend decides WHAT to push and WHEN. The app just registers its device token and listens.

---

### Q: What do I say about observability and analytics?

**A:** Two levels:

**Crash reporting** (things going wrong):
- "We'd integrate Crashlytics/Sentry to catch and report crashes with stack traces, device info, and user actions leading up to the crash."

**Analytics** (things going right... or not):
- "We'd track key events: screen views, button taps, feature usage, funnel completion rates."
- "For performance: app startup time, API latency, frame drop rate, memory usage."
- "This data feeds into feature flag decisions — if the new flow has worse conversion, we roll it back."

---

### Q: Privacy, localization, accessibility — should I mention these?

**A:** Absolutely. Even one sentence shows maturity:

- 🔒 **Privacy**: *"We'd minimize data collection, encrypt sensitive data at rest, and respect platform privacy APIs (App Tracking Transparency on iOS, privacy dashboard on Android)."*

- 🌍 **Localization**: *"The app should support RTL layouts for Arabic/Hebrew, use locale-aware date/number formatting, and externalize all strings for easy translation."*

- ♿ **Accessibility**: *"All interactive elements need semantic labels for screen readers. Tap targets should be at least 44x44pt. We'd support Dynamic Type (iOS) and font scaling (Android) for users with low vision."*

These show you think about **all users**, not just the default case.

---

## 📱 DEVICE CONSIDERATIONS

---

### Q: "What about supporting different devices?"

**A:**

**Screen sizes**: Phones, tablets, foldables
- Use responsive layouts (ConstraintLayout on Android, Auto Layout + Size Classes on iOS)
- Tablets might show a split view (list + detail side by side)
- Foldables need to handle fold/unfold transitions gracefully

**OS versions**: Not everyone is on the latest
- Set a reasonable minimum SDK (e.g., Android API 26 / iOS 15)
- Use `@available` checks on iOS, SDK version checks on Android
- Feature-degrade gracefully: *"PiP is available on Android 8+. For older devices, we fall back to background audio only."*

**Network conditions**: 5G, 4G, 3G, airplane mode
- Adaptive quality (lower image/video quality on slow connections)
- Offline-first architecture for core features
- Show connection status indicators

---

### Q: What's a good closing statement for ANY MSD interview?

**A:** After your wrap-up, say something like:

> *"With more time, I'd also explore [pick 2-3]:*
> - *Automated testing strategy (unit, integration, UI tests)*
> - *CI/CD pipeline for continuous delivery*
> - *App size optimization (ProGuard, asset compression, dynamic delivery)*
> - *A/B testing framework for measuring feature impact*
> - *Accessibility audit to ensure inclusive design*
>
> *These would make the app production-ready for a large-scale rollout."*

This shows **depth AND breadth**. You nailed the 45-minute problem AND you know what comes next.

---

## ⚛️ REACT NATIVE SPECIFICS

---

### Q: What does the state management landscape look like in React Native?

**A:** The modern answer layers three separate concerns — don't mix them:

| State type | Tool | Why |
|-----------|------|-----|
| **Server state** (API data, lists, user profiles) | **React Query (TanStack Query)** | Caching, deduplication, background refresh, optimistic updates — all built-in |
| **Global client state** (auth, theme, cart, modals) | **Zustand** or **Jotai** | Zero-boilerplate, no provider nesting, works outside React components |
| **Local UI state** (form inputs, toggles, animations) | React `useState` / `useReducer` | No need to reach for a library |
| **Navigation state** | **React Navigation** | The de facto standard. Stack, Tab, Drawer navigators |
| **Form state** | **React Hook Form** | Controlled inputs without re-rendering on every keystroke |

**The Zustand pitch:**

```typescript
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import { MMKV } from 'react-native-mmkv';

const storage = new MMKV();

// Auth store — persists across app restarts
export const useAuthStore = create(
  persist(
    (set) => ({
      user: null as User | null,
      token: null as string | null,
      login: (user: User, token: string) => set({ user, token }),
      logout: () => set({ user: null, token: null }),
    }),
    {
      name: 'auth-storage',
      storage: createJSONStorage(() => ({
        getItem: (key) => storage.getString(key) ?? null,
        setItem: (key, value) => storage.set(key, value),
        removeItem: (key) => storage.delete(key),
      })),
    }
  )
);
```

**Why not Redux for most React Native apps?** It's not wrong — but it adds actions, reducers, selectors, and middleware for problems that React Query + Zustand solve in a fraction of the code. Redux shines in very complex apps with many concurrent writers who need strict state transition guarantees and time-travel debugging.

---

### Q: What are the storage options in React Native and when do I use each?

**A:**

| Need | Library | Notes |
|------|---------|-------|
| Key-value, fast synchronous reads | **MMKV** | 10× faster than AsyncStorage, C++ backed, synchronous |
| Key-value, async (legacy) | **AsyncStorage** | Still fine for non-critical, infrequent reads |
| Large relational datasets (10k+ rows) | **WatermelonDB** | Reactive, SQLite-backed, built for offline-first |
| Medium relational data | **react-native-sqlite-storage + Drizzle ORM** | Direct SQLite with type-safe queries |
| Persisted server state cache | **React Query + MMKV persister** | Instant first load from cache, refresh in background |
| Binary files (videos, PDFs) | **react-native-fs** | App sandbox file reads/writes |
| Secrets, tokens, encryption keys | **react-native-keychain** | iOS Keychain / Android Keystore — hardware-backed, biometric unlock |

**Why MMKV over AsyncStorage:**

```typescript
// AsyncStorage — async, needs await everywhere
const token = await AsyncStorage.getItem('token'); // Can't call this at module level

// MMKV — synchronous, call anywhere
const storage = new MMKV();
const token = storage.getString('token'); // Works at module init, in middleware, anywhere
```

The synchronous read matters at app boot — you need the auth token *immediately* to decide whether to show the login screen or the home screen. With AsyncStorage, you'd always see a brief flash of the wrong screen.

**WatermelonDB for chat history:**

WatermelonDB was built for apps with large local datasets. It adds a reactive layer on top of SQLite so your message list only re-renders when *those specific messages* change:

```typescript
// Only re-renders when messages in this conversation change
const messages = useQuery(
  Message.query(Q.where('conversation_id', conversationId))
);
```

---

### Q: What is OTA (Over-the-Air) updates and why does every React Native developer mention it?

**A:** One of React Native's killer features: because your app logic is JavaScript, you can ship bug fixes and new features **without going through App Store / Play Store review** (which takes 1–3 days).

**CodePush** (Microsoft App Center) is the standard for React Native CLI projects — `code-push release-react MyApp ios` to publish a new bundle.

**How it works:**

1. You run `code-push release-react MyApp ios --target-binary-version "~1.2.0"`
2. Users open the app → CodePush checks for updates in the background
3. New JS bundle downloads silently
4. On next cold start (or immediately if you configure `InstallMode.IMMEDIATE`), the new bundle loads

**The critical limitations to mention in interviews** (shows you've actually shipped this):
- You can only update **JavaScript and assets** — native code changes (adding a new library with native modules) require a full App Store release
- Apple's review guidelines: updates must be "consistent with the app's intended purpose" — no major behavior changes via OTA
- Classic use case: *"We shipped a crash fix at 11pm to 100% of users without waiting 3 days for App Store approval"*

```typescript
import codePush, { CodePushOptions } from 'react-native-code-push';
import { AppState, AppStateStatus } from 'react-native';

const codePushOptions: CodePushOptions = {
  checkFrequency: codePush.CheckFrequency.ON_APP_RESUME,
  installMode: codePush.InstallMode.ON_NEXT_RESTART,   // Don't interrupt the user mid-session
  mandatoryInstallMode: codePush.InstallMode.IMMEDIATE, // Critical patches install now
};

// Wrap your root component
export default codePush(codePushOptions)(App);
```

CodePush also supports **staged rollouts** — release to 10% of users first, monitor crash rates, then promote to 100%. Same concept as feature flags but for JS bundles.

---

### Q: What are the React Native performance fundamentals I should know for an interview?

**A:** The core mental model: React Native runs in **two threads**. Performance problems usually come from one blocking the other.

**JS Thread** — React reconciliation, state updates, business logic
- Blocked by: heavy `JSON.parse`, complex renders, synchronous loops
- Fix: memoize with `useMemo`/`React.memo`, move computation off-thread

**UI Thread (Main Thread)** — native rendering, gestures, animations
- Blocked by: synchronous JS calls via the Bridge
- Fix: use **React Native Reanimated** (animations run entirely on UI thread via worklets)

**The Bridge / New Architecture:**
- Old Architecture: JS ↔ Native calls are serialized to JSON and sent async over a bridge — expensive for high-frequency calls
- New Architecture (JSI): native calls become direct C++ function pointers — synchronous and near-zero overhead

**Five things to say that show performance maturity:**

1. **"I'd use React Native Reanimated for animations"** — worklets run on the UI thread, so dragging a card stays smooth even when the JS thread is busy parsing API responses
2. **"I'd memoize `renderItem` with `React.memo`"** — prevents re-rendering every list item when parent state changes
3. **"I'd add `getItemLayout` to FlatList"** — lets FlatList skip layout measurement, making `scrollToIndex` instant
4. **"I'd use `InteractionManager.runAfterInteractions`"** — defer expensive setup until navigation transitions finish, so screens appear instantly
5. **"I'd enable Hermes"** — Google's JS engine optimized for RN: faster startup, lower memory, TTI (time-to-interactive) improvements

```typescript
import { InteractionManager } from 'react-native';
import { useEffect, useState } from 'react';

function HeavyScreen() {
  const [ready, setReady] = useState(false);

  useEffect(() => {
    // Don't run expensive setup until the screen's transition animation finishes
    const handle = InteractionManager.runAfterInteractions(() => {
      loadHeavyData();
      setReady(true);
    });
    return () => handle.cancel();
  }, []);

  if (!ready) return <LoadingPlaceholder />;
  return <TheActualScreen />;
}
```

---

### Q: What navigation pattern should I propose in a React Native interview?

**A:** **React Navigation** is the answer. It's the de facto standard and what interviewers expect.

```tsx
// Typical production app shell
export function AppNavigator() {
  const { user } = useAuthStore();

  return (
    <NavigationContainer>
      {user ? <MainNavigator /> : <AuthNavigator />}
    </NavigationContainer>
  );
}

function AuthNavigator() {
  return (
    <Stack.Navigator screenOptions={{ headerShown: false }}>
      <Stack.Screen name="Login" component={LoginScreen} />
      <Stack.Screen name="Register" component={RegisterScreen} />
    </Stack.Navigator>
  );
}

function MainNavigator() {
  return (
    <Tab.Navigator>
      <Tab.Screen name="Feed" component={FeedStack} />
      <Tab.Screen name="Search" component={SearchScreen} />
      <Tab.Screen name="Profile" component={ProfileScreen} />
    </Tab.Navigator>
  );
}
```

**Performance tip to mention:** Use `@react-navigation/native-stack` (not the JS stack) — it uses actual native `UINavigationController` (iOS) and `Fragment` transitions (Android), which run on the UI thread and feel identical to native apps.

React Navigation is the answer for RN CLI projects — it's what interviewers expect and what most production codebases use.

---

### Q: How do I think about testing in React Native?

**A:** Three levels, each with a clear purpose:

| Level | Tool | Tests |
|-------|------|-------|
| **Unit** | **Jest** | Custom hooks, utility functions, data transformers |
| **Component** | **React Native Testing Library (RNTL)** | Component rendering, user interactions, state changes |
| **E2E** | **Detox** or **Maestro** | Full user flows on a real device/simulator |

```typescript
// RNTL — test user interactions, not implementation
import { render, screen, fireEvent, waitFor } from '@testing-library/react-native';

test('like button updates count optimistically', async () => {
  // Arrange: render with mocked React Query
  const wrapper = createQueryClientWrapper();
  render(<PostCard post={mockPost} />, { wrapper });

  // Act: user taps the like button
  fireEvent.press(screen.getByTestId('like-button'));

  // Assert: count increments immediately (optimistic)
  expect(screen.getByTestId('like-count')).toHaveTextContent('43');

  // Assert: API was called
  await waitFor(() => {
    expect(mockApi.likePost).toHaveBeenCalledWith(mockPost.id);
  });
});
```

**The testing pyramid for React Native:**
- **Many unit tests** (fast, cheap, test business logic in custom hooks)
- **Some component tests** (test that components render correctly and respond to interaction)
- **Few E2E tests** (slow, expensive, brittle — only for the most critical paths: login, checkout, onboarding)

**Detox pitfall to mention:** E2E tests flake on timing. Use `waitFor` with explicit conditions rather than `sleep` calls.

---

## 🎯 FINAL RAPID-FIRE: 10 Things to Always Mention

1. **"I'll use cursor-based pagination because..."** → Shows you know the tradeoffs
2. **"The repository is the single source of truth"** → Shows clean architecture
3. **"I'd add an idempotency key to prevent duplicates"** → Shows you've dealt with real networks
4. **"For offline support, local DB is the SSOT"** → Shows mobile-first thinking
5. **"I'd use feature flags for gradual rollout"** → Shows production experience
6. **"Exponential backoff with jitter for retries"** → Shows you respect servers
7. **"WebSocket for real-time, REST for everything else"** → Shows protocol awareness
8. **"In-memory cache first, then disk, then network"** → Shows performance thinking
9. **"I'd track this with analytics to measure impact"** → Shows data-driven mindset
10. **"Let me present the tradeoffs..."** → The single most important habit

---

## ⚛️ REACT NATIVE RAPID-FIRE: 8 Things That Signal Senior RN Experience

1. **"I'd use React Query for server state — it eliminates manual loading/error state and gives me caching for free"** → Shows you're current
2. **"MMKV instead of AsyncStorage — synchronous reads matter at app boot"** → Shows performance awareness
3. **"I'd use Reanimated for this animation — worklets run on the UI thread so they won't drop frames when the JS thread is busy"** → Shows threading knowledge
4. **"I'd add `getItemLayout` to the FlatList since items have a fixed height"** → Shows you've hit real scroll performance walls
5. **"We can ship critical bug fixes same-day with CodePush — no App Store wait"** → Shows production shipping experience
6. **"I'd use `InteractionManager.runAfterInteractions` for the heavy setup so the screen transition animation stays smooth"** → Shows startup performance care
7. **"TypeScript interfaces for the API contracts, validated at runtime with Zod"** → Shows you've been burned by unexpected API shapes
8. **"I'd split this into feature modules — `:feed`, `:chat`, `:core-network` — for parallel builds and team ownership"** → Shows large-team experience

---

*You've made it through all 5 modules! Now go crush that interview. 🚀*
