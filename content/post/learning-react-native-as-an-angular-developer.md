---
title: "Learning React Native as an Angular developer"
date: 2026-09-12T17:00:00+02:00
categories:
- Angular
tags:
- Angular
- React
- React Native
- Mobile Development
draft: true
---

If you have been working with Angular for years, React Native can at first look like a completely different world. New framework, new rendering model, new component primitives and suddenly you also have to care about iOS and Android.

The good news is that most of the important concepts are already familiar. Components, inputs, local state, derived state, routing, state management, dependency boundaries, testing and architecture are still there. The names and mechanics are just different.

In this post, I am going to map the React Native concepts to what an Angular developer already knows and focus on the parts where the mental model is actually different. The goal is not to teach every React API. It is to get an Angular developer productive in a React Native codebase without trying to write Angular in React.

![From Angular to React Native](/img/posts/learning-react-native-as-angular-developer/angular-to-react-native.svg)

## React Native is not Angular running on a phone

The first thing to understand is that React Native is not a web application inside a WebView. Your TypeScript and React components are rendered to native platform UI.

A simplified flow looks like this:

```text
TypeScript
   ↓
React components + hooks
   ↓
React Native renderer
   ↓
<View>, <Text>, <Pressable>, <TextInput> ...
   ↓
Native iOS / Android UI
```

This means there is normally no DOM and no `div`, `span` or `button`. You work with React Native primitives such as `View`, `Text`, `Pressable`, `TextInput`, `Image` and `FlatList` instead.

For an Angular developer, I think the easiest way to look at it is: Angular gives you a web platform abstraction while React Native gives you a cross-platform abstraction over native UI.

## Angular to React Native translation

Here is the mapping I keep in my head:

| Angular | React / React Native |
| --- | --- |
| Component class | Function component |
| Angular template | JSX / TSX |
| `@Input()` | props |
| `@Output()` | callback prop |
| `signal()` | `useState()` for local component state |
| `computed()` | derived value during render, sometimes `useMemo()` |
| `effect()` | `useEffect()` in some cases, but not as a direct 1:1 mapping |
| Service | module, custom hook, context or store |
| Dependency injection | imports + composition + Context when needed |
| `@if` / `*ngIf` | normal JavaScript conditionals |
| `@for` / `*ngFor` | `.map()` or usually `FlatList` for large native lists |
| Angular Router | React Navigation / Expo Router |
| `HttpClient` | `fetch`, Axios or an API client |
| NgRx / SignalStore | Redux Toolkit, Zustand, Context, TanStack Query, etc. |
| CSS / SCSS | React Native style objects / `StyleSheet` |
| `(click)` | `onPress` |
| lifecycle hooks | React hooks + mobile app lifecycle APIs |

The important part is not memorizing the table. It is realizing that React is deliberately less framework-driven than Angular. You will often have fewer framework constructs around a feature and more plain TypeScript.

## Function components instead of component classes

A normal React Native component today is just a function:

```tsx
type Props = {
  title: string;
  onSave: () => void;
};

export function EditAssetScreen({ title, onSave }: Props) {
  const [name, setName] = useState(title);

  return (
    <View>
      <TextInput value={name} onChangeText={setName} />

      <Pressable onPress={onSave}>
        <Text>Save</Text>
      </Pressable>
    </View>
  );
}
```

If you come from Angular, you can translate this to:

- `Props` is basically the public API of the component, similar to inputs and outputs.
- `useState` is local component state.
- JSX is the template, except it is TypeScript syntax and therefore you use JavaScript expressions directly.
- `onSave` is just a callback passed down from the parent rather than an `EventEmitter`.

The React rule that matters most here is that rendering should be pure. Given the same props and state, the component should describe the same UI. Side effects belong elsewhere.

## `computed()` vs React derived state

This was one of the React concepts that clicked for me immediately when comparing it with Angular Signals.

In Angular we might write:

```ts
const firstName = signal('Christian');
const lastName = signal('Lüdemann');

const fullName = computed(
  () => `${firstName()} ${lastName()}`
);
```

Angular tracks which signals the computed value reads. When one of those dependencies changes, Angular knows the computed value is stale and consumers update.

The React version is often simpler:

```tsx
const [firstName, setFirstName] = useState('Christian');
const [lastName, setLastName] = useState('Lüdemann');

const fullName = `${firstName} ${lastName}`;

return <Text>{fullName}</Text>;
```

When `setFirstName()` is called, React schedules a re-render. The component function runs again, `fullName` is recalculated and React reconciles the resulting UI.

![Angular computed compared with React re-rendering](/img/posts/learning-react-native-as-angular-developer/computed-vs-rerender.svg)

The result in the view is similar, but the mechanics are different:

```text
Angular Signals

signal changes
    ↓
dependency tracking
    ↓
computed becomes stale
    ↓
consumer updates

React

state changes
    ↓
component function runs again
    ↓
derived values are recalculated
    ↓
React reconciles the UI
```

This also explains an important React best practice: do not put derived values into state unless they genuinely need an independent lifecycle.

Bad:

```tsx
const [fullName, setFullName] = useState('');

useEffect(() => {
  setFullName(`${firstName} ${lastName}`);
}, [firstName, lastName]);
```

Better:

```tsx
const fullName = `${firstName} ${lastName}`;
```

For an expensive calculation you can use `useMemo`, but IMO this is often overused. Start with the simple derived value and optimize when there is a reason.

## Do not treat `useEffect` as `ngOnInit`

This is probably one of the biggest traps for Angular developers learning React.

It is tempting to see:

```tsx
useEffect(() => {
  // something
}, []);
```

and think `ngOnInit`.

That comparison works sometimes, but it is a dangerous mental model. React describes Effects as an escape hatch for synchronizing with something outside React. If you are just transforming state into other state, you probably do not need an Effect.

Good examples for an Effect are subscriptions and integrations with external systems:

```tsx
useEffect(() => {
  const subscription = AppState.addEventListener(
    'change',
    handleAppStateChange
  );

  return () => subscription.remove();
}, []);
```

The returned function is cleanup, so there is a resemblance to `ngOnDestroy`, but the setup and teardown are colocated.

A simple rule I like is:

```text
Can I calculate it while rendering?
→ calculate it while rendering

Did the user perform an action?
→ event handler

Am I synchronizing with something outside React?
→ useEffect may be the right tool
```

That keeps a lot of unnecessary state and lifecycle complexity out of the component.

## Think about state by responsibility

Just like I would not put every piece of Angular state into NgRx, I would not put every piece of React Native state into Redux.

A useful split is:

| State | Example | Typical solution |
| --- | --- | --- |
| Local UI state | modal open, selected tab | `useState` |
| Derived state | filtered assets | calculate during render |
| Complex local state | multi-step workflow | `useReducer` or a custom hook |
| Server state | turbines from an API | TanStack Query or the project's equivalent |
| Global client state | tenant, app settings | Context, Zustand, Redux Toolkit, etc. |
| Secure persisted state | credentials/tokens | secure OS storage |
| Offline business data | inspections, queued operations | local persistence/database + sync layer |

One thing I particularly like about the modern React ecosystem is the distinction between client state and server state. Data that came from the backend has caching, loading, retry, invalidation and refetch semantics that are different from a local UI toggle.

That is why tools such as TanStack Query can be a better fit for server state than just copying API responses into a global store.

That said, when joining an existing project, follow the existing architecture first. Do not introduce a new state library because it happens to be your favorite one.

## Native UI primitives and styling

Instead of HTML elements, React Native gives us native primitives:

```tsx
<View style={styles.container}>
  <Text style={styles.title}>Wind farms</Text>

  <Pressable onPress={openAsset}>
    <Text>Open asset</Text>
  </Pressable>
</View>
```

A common styling setup looks like this:

```tsx
const styles = StyleSheet.create({
  container: {
    flex: 1,
    padding: 16,
    gap: 12,
  },
  row: {
    flexDirection: 'row',
    alignItems: 'center',
  },
  title: {
    fontSize: 20,
    fontWeight: '600',
  },
});
```

The concepts are still largely Flexbox, but do not assume browser CSS behavior is identical. One difference Angular developers immediately notice is that React Native defaults `flexDirection` to `column`.

Also remember that a phone has notches, camera islands, home indicators and platform UI around your app. Safe areas and edge-to-edge layouts are part of normal mobile development rather than an edge case.

## Use `FlatList` for real lists

This is the native equivalent of thinking about virtualization instead of dumping hundreds of elements into the DOM.

Do not do this for a large data set:

```tsx
<ScrollView>
  {assets.map(asset => (
    <AssetCard key={asset.id} asset={asset} />
  ))}
</ScrollView>
```

Use a virtualized list:

```tsx
<FlatList
  data={assets}
  keyExtractor={asset => asset.id}
  renderItem={({ item }) => (
    <AssetCard asset={item} />
  )}
/>
```

`FlatList` only keeps the relevant window of items rendered, which matters a lot more on a resource-constrained mobile device.

My recommendation is still the same as on the web: do not randomly add memoization everywhere. Get the architecture right, profile, then optimize the bottleneck.

## Navigation is your router, but mobile navigation has more state

Most React Native apps use React Navigation or a router built around it.

Conceptually you will see something like:

```text
NavigationContainer
    │
    ├── AuthStack
    │     └── Login
    │
    └── MainStack
          ├── Home
          ├── Assets
          └── AssetDetails
```

Navigating might look like:

```tsx
navigation.navigate('AssetDetails', {
  assetId: asset.id,
});
```

As with Angular routing, keep route parameters small and stable. I would normally pass an identifier and load/read the data from the relevant state layer rather than serializing a large object through navigation.

Deep linking is also a first-class concern on mobile. A URL or push notification can open the application directly on a specific screen, so the navigation structure should be designed with that in mind.

## Mobile lifecycle is different from a browser tab

A mobile app moves between states:

```text
active
  ↓
inactive
  ↓
background
  ↓
active
```

React Native exposes this through `AppState`.

This matters for much more than UI. It affects authentication refresh, location tracking, Bluetooth, WebSockets, API refetches, background work and what happens when a user returns to an application that has been sitting in the background for hours.

A typical pattern could be:

```tsx
useEffect(() => {
  const subscription = AppState.addEventListener(
    'change',
    state => {
      if (state === 'active') {
        refreshData();
      }
    }
  );

  return () => subscription.remove();
}, []);
```

Do not blindly refetch everything every time the app becomes active, though. The right behavior depends on freshness requirements and the server-state solution used by the project.

## Offline-first becomes a real architecture concern

For many enterprise mobile apps, this is where React Native development starts being very different from normal web development.

A field user might lose connectivity completely. The application still needs to show useful data, allow work to continue and synchronize later.

A robust flow often looks something like:

```text
User changes data
      ↓
Persist locally
      ↓
Queue operation
      ↓
Network available?
  ┌───┴───┐
 no      yes
 │         │
wait      sync
           ↓
     acknowledge
```

For write operations I like using a unique operation ID:

```ts
type SyncOperation = {
  operationId: string;
  entityId: string;
  type: 'INSPECTION_UPDATED';
  payload: InspectionUpdate;
};
```

This lets the backend make retries idempotent. If a device sends the same queued operation twice because the connection dropped halfway through, the server can recognize that it has already processed it.

You also need an explicit strategy for conflicts. "Last write wins" might be fine for one type of data and completely unacceptable for another. Make that a domain decision rather than something that happens accidentally in the sync code.

## Authentication and secure storage

Coming from Angular, one important change is where credentials live.

On the web we often rely on secure cookies, browser sessions or historically token storage in browser APIs. On native iOS and Android you have OS-backed secure storage options such as Keychain and Keystore-backed solutions.

React Native's security guidance explicitly warns against using AsyncStorage for tokens and secrets because it is unencrypted. Think of AsyncStorage much more like `localStorage`: useful for non-sensitive persisted data, not for credentials.

Also remember that an `.env` file bundled into a mobile app is not a secret store. If a secret ships with the application bundle, an attacker can eventually extract it. Real secrets belong on a backend.

## The native boundary: know what is below TypeScript

Most of your day can still be TypeScript, but eventually a mobile app needs native capabilities:

- camera
- GPS
- Bluetooth
- NFC
- biometrics
- push notifications
- platform SDKs
- background services

Normally you consume an existing React Native library. If a native capability is not exposed, React Native's New Architecture lets you create typed native integrations using Turbo Native Modules and Codegen.

![How a React Native app fits together](/img/posts/learning-react-native-as-angular-developer/react-native-app-architecture.svg)

The terms worth knowing are:

**Hermes** is the JavaScript engine optimized for React Native. Hermes V1 became the default in React Native 0.84.

**Fabric** is the modern rendering system.

**TurboModules** are the modern native module system.

**Codegen** generates typed interfaces between JavaScript/TypeScript and native implementations.

**Metro** is the React Native bundler. If you are an Angular developer, think of it as filling roughly the tooling role that Vite/Webpack/esbuild does around a web app, although the details are obviously different.

React Native 0.82 was the first release that runs entirely on the New Architecture. As of this post, React Native 0.87 is current and also makes the Strict TypeScript API the default. An enterprise application can of course be on an older version, so always check the actual project before assuming what APIs and architecture decisions are available.

## iOS and Android are not one platform

React Native lets us share a lot of code, but I would avoid the "write once, behaves identically everywhere" mindset.

You can handle small differences explicitly:

```tsx
if (Platform.OS === 'ios') {
  // iOS-specific behavior
}
```

or use platform-specific files:

```text
CameraButton.ios.tsx
CameraButton.android.tsx
```

Typical differences include permissions, keyboard behavior, Android back navigation, notifications, fonts, shadows/elevation, safe areas, file systems and native SDK behavior.

The best React Native codebases share the domain logic and UI where it makes sense while being explicit about genuine platform differences.

## Accessibility still needs semantics

Do not turn every interaction into an anonymous clickable `View`.

Prefer semantic controls and expose the right accessibility information:

```tsx
<Pressable
  onPress={save}
  accessibilityRole="button"
  accessibilityLabel="Save inspection"
>
  <Text>Save</Text>
</Pressable>
```

React Native maps these semantics to technologies such as VoiceOver on iOS and TalkBack on Android.

This is the same principle we know from web accessibility: the visual UI is only one representation of the interaction model.

## Testing: test behavior, not implementation details

The testing philosophy does not need to change dramatically just because the UI is native.

I would still aim for:

```text
          E2E
       /       \
 component / integration
    /             \
unit + types + linting
```

For component tests, prefer queries that reflect how a user interacts with the application instead of asserting internal component state.

For example, prefer locating a save button by role/name instead of making every test depend on arbitrary test IDs when a semantic query is available.

For E2E, the project might use tools such as Detox, Maestro or Appium. The exact tool is less important than having a small number of high-value flows running against something close to the real application.

## What I would learn first as an Angular developer

If I had to become productive in a React Native project quickly, I would learn in this order:

1. Function components, JSX, props and `useState`.
2. The React rendering model and why derived state normally gets calculated during render.
3. `useEffect` and, just as importantly, when not to use it.
4. React Native primitives: `View`, `Text`, `Pressable`, `TextInput`, `Image` and `FlatList`.
5. Flexbox and native layout.
6. Navigation, route params and deep linking.
7. The app lifecycle and permissions.
8. The project's chosen approach for server state and global state.
9. Offline persistence, synchronization and idempotency if the app supports field/offline usage.
10. Native concepts such as Hermes, Fabric, TurboModules and the `ios/` and `android/` projects.

The main mental-model shift is this:

```text
Angular Signals

state changes
   ↓
Angular tracks affected consumers
   ↓
change detection updates the view

React

state changes
   ↓
component function runs again
   ↓
JSX describes the new UI
   ↓
React reconciles the difference
   ↓
native UI updates
```

Once that clicks, React Native becomes a lot less foreign.

## Conclusion

As an Angular developer, you are not starting over when learning React Native. The architecture problems are largely the same: keep responsibilities clear, keep state close to where it belongs, model server state properly, keep components reusable, make side effects explicit and test behavior that matters.

The biggest new layer is mobile itself: app lifecycle, offline behavior, native permissions, secure device storage, deep links, platform differences and the native boundary.

My recommendation is therefore not to spend your first days memorizing every hook or React Native API. Get the React rendering model right, learn the core native primitives and then understand how the specific application handles navigation, state, synchronization and native capabilities.

You will find that most of your Angular experience transfers very well. Same engineering problems, different mechanics.

### Further reading

- [React Native 0.87 release](https://reactnative.dev/blog/2026/08/11/react-native-0.87)
- [React Native New Architecture](https://reactnative.dev/architecture/landing-page)
- [React Native security](https://reactnative.dev/docs/security)
- [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect)
- [New Angular project? This is how I would start](https://christianlydemann.com/new-angular-project-this-is-how-i-would-start/)
- [The stages of an Angular architecture](https://christianlydemann.com/the-stages-of-an-angular-architecture-with-nx/)
