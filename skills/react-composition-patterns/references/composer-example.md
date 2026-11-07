# Worked example: the Slack-style composer

Reconstructed from the talk's narration. The speaker did not show his exact code in the transcript, so names and signatures here are illustrative. The structure and decisions are the speaker's.

## Contents
1. The use cases and how they differ
2. The "before": boolean monolith
3. The "after": shared parts and context
4. Each variant as a composition
5. State: one interface, different providers
6. Lifting state for UI outside the composer box
7. Why actions-as-JSX beats an actions array

---

## 1. The use cases and how they differ

The "one component" is used in: channels, DMs, DM threads, channel threads, forwarding a message, editing a message, and more.

| Variant | Differences from the channel composer |
|---|---|
| Channel | baseline: attachments (drag/drop + plus icon), full actions, icon submit button |
| Channel thread | adds an "also send to channel" checkbox bar |
| DM thread | adds "also send as direct message" instead |
| Edit message | no attachments (no plus icon, so **no drag-and-drop either**), only format-text and emoji actions, **Cancel + Save** buttons instead of the icon submit |
| Forward message | state is **local/ephemeral** (modal, warns on discard), actions like edit plus mentions, **no submit inside the box**; a Forward button row sits *outside* the composer |

State also differs: most composers sync as you type across devices; forwarding is ephemeral.

## 2. The "before": boolean monolith

How it typically grows:

```tsx
function Composer({
  onSubmit,
  isThread, channelId,
  isDMThread, dmId,
  isEditing, onCancel,
  isForwarding,
}: ComposerProps) {
  return (
    <DropZone disabled={isEditing}>
      <Frame>
        <Header />
        <Input />
        {isThread && !isDMThread && <AlsoSendToChannel channelId={channelId} />}
        {isDMThread && <AlsoSendAsDM dmId={dmId} />}
        <Footer
          isEditing={isEditing}
          isForwarding={isForwarding}
          onCancel={onCancel}
        />
      </Frame>
    </DropZone>
  );
}

function Footer({ isEditing, isForwarding, onCancel }: FooterProps) {
  return (
    <div>
      {!isEditing && <PlusMenu />}
      <Emojis />
      <TextFormat />
      {!isEditing && <Mentions />}
      {isEditing ? (
        <>
          <Button onClick={onCancel}>Cancel</Button>
          <Button type="submit">Save</Button>
        </>
      ) : isForwarding ? null : (
        <SubmitIconButton />
      )}
    </div>
  );
}
```

Failure points the speaker names along the way:
- `isThread` -> then `isDMThread` + `dmId` -> every new variant is another flag.
- Edit mode needs `DropZone` disabled **and** the footer told which actions to hide.
- Footer actions as an array grows `isMenu`, `divider`, `menuItems` fields; the loop is a disaster.
- `onCancel` is a new optional prop passed down to the footer, which loses its simple submit button.
- Forwarding needs state *out* of the form to a button outside it, pushing toward `useEffect` syncing or refs.
- A `renderSubmit` prop might patch edit mode but doesn't work for forwarding.

## 3. The "after": shared parts and context

### Context interface

```tsx
import { createContext, use } from 'react';

type ComposerState = { text: string; attachments: Attachment[] };

type ComposerActions = {
  update: (updater: (prev: ComposerState) => ComposerState) => void;
  submit: () => void;
};

type ComposerMeta = {
  inputRef: React.RefObject<HTMLTextAreaElement | null>; // lives outside state
};

type ComposerContextValue = {
  state: ComposerState;
  actions: ComposerActions;
  meta: ComposerMeta;
};

const ComposerContext = createContext<ComposerContextValue | null>(null);

export function useComposer() {
  const ctx = use(ComposerContext);
  if (!ctx) throw new Error('useComposer must be used inside a composer provider');
  return ctx;
}
```

Naming (`state` / `actions` / `meta`) is the speaker's stated preference, not a requirement.

### Parts read from context and take no variant flags

```tsx
function ComposerInput() {
  const { state, actions, meta } = useComposer();
  return (
    <textarea
      ref={meta.inputRef}
      value={state.text}
      onChange={(e) =>
        actions.update((prev) => ({ ...prev, text: e.target.value }))
      }
    />
  );
}
```

The input is agnostic to whether state is local or synced. Any other component can call `meta.inputRef.current?.focus()` with no drilling or `useImperativeHandle`.

### Export as a compound API

```tsx
export const Composer = {
  Frame, Header, Input, Footer,
  DropZone, Submit,
  PlusMenu, Emojis, TextFormat, Mentions,
  CommonActions, AlsoSendToChannel,
};
// The Provider is NOT in here: each use case brings its own (section 5).
```

### Reusable group with an escape hatch

```tsx
function CommonActions() {
  return (
    <>
      <Composer.PlusMenu />
      <Composer.Emojis />
      <Composer.TextFormat />
      <Composer.Mentions />
    </>
  );
}
```

It takes no `isEditing`-style prop and does no checks. Composers that fit use it; others render individual actions.

## 4. Each variant as a composition

### Channel composer

```tsx
function ChannelComposer({ channelId }: { channelId: string }) {
  return (
    <ChannelProvider channelId={channelId}>
      <Composer.DropZone>
        <Composer.Frame>
          <Composer.Header />
          <Composer.Input />
          <Composer.Footer>
            <Composer.CommonActions />
            <Composer.Submit />
          </Composer.Footer>
        </Composer.Frame>
      </Composer.DropZone>
    </ChannelProvider>
  );
}
```

### Thread composer: only the delta

```tsx
function ThreadComposer({ channelId, threadId }: Props) {
  return (
    <ThreadProvider channelId={channelId} threadId={threadId}>
      <Composer.DropZone>
        <Composer.Frame>
          <Composer.Header />
          <Composer.Input />
          <Composer.AlsoSendToChannel />   {/* the only difference */}
          <Composer.Footer>
            <Composer.CommonActions />
            <Composer.Submit />
          </Composer.Footer>
        </Composer.Frame>
      </Composer.DropZone>
    </ThreadProvider>
  );
}
```

"Where are the booleans and the special props from the top of the tree telling us what to render? They're nowhere to be found." A DM thread would render an `AlsoSendAsDirectMessage` part instead, as its own component.

### Edit message composer

```tsx
function EditMessageComposer({ message, onDone }: Props) {
  return (
    <EditMessageProvider message={message} onDone={onDone}>
      {/* No DropZone: editing doesn't support attachments. No boolean. We just don't render it. */}
      <Composer.Frame>
        <Composer.Header />
        <Composer.Input />
        <Composer.Footer>
          <Composer.TextFormat />
          <Composer.Emojis />
          <Button onClick={onDone}>Cancel</Button>
          <SaveButton />
        </Composer.Footer>
      </Composer.Frame>
    </EditMessageProvider>
  );
}

// One-off, only used by this composer
function SaveButton() {
  const { actions } = useComposer();
  return <Button onClick={actions.submit}>Save</Button>;
}
```

The footer needs only format and emoji, so render those directly. Cancel/Save are one-offs, so render plain buttons. No `renderSubmit`, no `onCancel` drilled through.

## 5. State: one interface, different providers

The context provider "defines the common interface... but the implementation itself is only handled by the component which renders the provider."

### Ephemeral (forward message): `useState`

```tsx
function ForwardMessageProvider({ children }: { children: React.ReactNode }) {
  const [state, setState] = useState<ComposerState>({ text: '', attachments: [] });
  const inputRef = useRef<HTMLTextAreaElement | null>(null);
  const forwardMessage = useForwardMessage(); // the only hook that hits the network

  return (
    <ComposerContext.Provider
      value={{
        state,
        actions: {
          update: setState,
          submit: () => forwardMessage(state),
        },
        meta: { inputRef },
      }}
    >
      {children}
    </ComposerContext.Provider>
  );
}
```

Closing the modal discards the text, which is the desired ephemeral behavior.

### Synced across devices (channel): a global hook

```tsx
function ChannelProvider({
  channelId,
  children,
}: { channelId: string; children: React.ReactNode }) {
  // Sends messages and syncs to backend/devices. Must return things that
  // conform to the context interface.
  const { state, update, submit } = useGlobalChannel(channelId);
  const inputRef = useRef<HTMLTextAreaElement | null>(null);

  return (
    <ComposerContext.Provider
      value={{ state, actions: { update, submit }, meta: { inputRef } }}
    >
      {children}
    </ComposerContext.Provider>
  );
}
```

"All we need to do is swap out this root provider." Every child now uses synced state. To change state management for an entire composer, change **one place**, rather than a component calling a global hook and branching on booleans.

### Re-render concern

The speaker's answer is the React Compiler: its output memoizes `ComposerInput` based only on the context fields it reads.

## 6. Lifting state for UI outside the composer box

Forward-message layout:

```
┌ modal ──────────────────────┐
│ ┌ composer frame ─────────┐ │
│ │ header / input / footer │ │
│ └─────────────────────────┘ │
│ [ message preview ]         │
│ [ Cancel ]   [ Forward ]    │  <- outside the frame, needs composer state
└─────────────────────────────┘
```

Anti-patterns: `useEffect` pushing form state up; a shared ref with `onSubmit` that reads form state; `onFormStateDidChange`; a `renderFooter` prop (nothing renders down there).

Fix: the provider is its own component with `children`, and it sits above both the composer and the outside UI.

```tsx
function ForwardMessageDialog({ message }: { message: Message }) {
  return (
    <ForwardMessageProvider>
      <ForwardMessageComposer />
      <MessagePreview message={message} />
      <ForwardActions />
    </ForwardMessageProvider>
  );
}

function ForwardMessageComposer() {
  return (
    <Composer.Frame>
      <Composer.Header />
      <Composer.Input />
      <Composer.Footer>
        <Composer.TextFormat />
        <Composer.Emojis />
        <Composer.Mentions />
      </Composer.Footer>
    </Composer.Frame>
  );
}

function ForwardActions() {
  const { actions } = useComposer(); // not inside the frame, still has access
  return (
    <div>
      <Button onClick={closeDialog}>Cancel</Button>
      <Button onClick={actions.submit}>Forward</Button>
    </div>
  );
}
```

The forward button "can just use the submit action from the context without being inside the composer because it's within the composer provider, though not necessarily within the composer frame."

Result: reuse some composer parts but not all; an unrelated preview renders under it; custom actions use composer state yet are isolated from its UI.

## 7. Why actions-as-JSX beats an actions array

Array version (what the speaker warns against):

```tsx
const actions = [
  { id: 'plus', type: 'menu', divider: true, items: [/* only some have these */] },
  { id: 'emoji', type: 'button' },
  /* ... */
];

actions
  .filter((a) => !(isEditing && a.id === 'plus'))
  .map((a) => (a.type === 'menu' ? <Menu {...a} /> : <ActionButton {...a} />));
```

JSX version:

```tsx
<Composer.PlusMenu />
<Composer.Emojis />
<Composer.TextFormat />
```

Each action component can internally use a shared button. A one-off with different styling (the speaker mentions the far-left button) is just a different element. Variants include or omit elements. No data format, no loop, nothing to interpret.
