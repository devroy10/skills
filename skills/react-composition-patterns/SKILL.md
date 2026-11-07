---
name: react-composition-patterns
description: Patterns for building and refactoring React components with composition instead of boolean props - compound components, shared context providers with swappable state implementations, lifting state up, and explicit JSX variants. Make sure to use this skill whenever the user is writing, reviewing, or refactoring a React or React Native component that has (or is about to get) boolean props like isEditing / isThread / isUpdate, many optional props, conditional rendering that switches whole sections of UI, config arrays of actions or buttons, render-prop escape hatches, or "one component, many variants" (create vs edit forms, composers, modals, cards, toolbars). Also use it when designing reusable UI components, structuring component state (local vs global vs synced), sharing state with UI rendered outside a component's box, or when asked how to keep a React codebase easy for humans and AI agents to extend. Trigger even if the user never says "composition".
---

# React Composition Patterns

Source: a conference talk on React composition that walks through rebuilding a Slack-style message composer. Its thesis: **when a component grows boolean props that decide which component tree gets rendered, stop adding props and compose instead.**

The talk's worked example (with reconstructed code) is in `references/composer-example.md`. Read it when you need a concrete full implementation to model new code on.

> Provenance note: the talk is described in the transcript only through narration, so all code in this skill is a *reconstruction* of what the speaker described, not a copy of his slides. Where this skill adds something the speaker did not say, it is labeled **(addition)**.

---

## The core problem

A component starts simple, gets reused, and each new use case adds a prop:

```tsx
// How the mess grows
<Composer
  onSubmit={...}
  isThread
  channelId={...}
  isDMThread
  dmId={...}
  isEditing
  isForwarding
  onCancel={...}
  hideAttachments
  hideActions
  actions={[{ type: 'menu', divider: true, items: [...] }]}
/>
```

Symptoms the talk calls out:
- A boolean that decides **which component tree the parent renders** (the speaker's rule of thumb: if you see this, "shaking my head" over your shoulder).
- The same condition (`isEditing`) checked in many places inside one component, and drilled through intermediate components (header, footer, drop zone).
- TypeScript unions "to avoid impossible states" layered on top of the flags.
- An array of action descriptors that keeps gaining fields (`isMenu`, `divider`, `items`) and whose loop becomes a disaster.
- Optional callback props (`onCancel`) passed all the way down so one variant can swap a single button.
- A `renderSubmit` / `renderFooter` prop as a patch, which stops working when the UI needs state from outside the component's box.

The talk's claim: this is what happens in many codebases as one-off components get reused in different ways, and it is hard to work in for both humans and AI.

---

## The five moves

### 1. Split the monolith into compound components

Replace one component with a giant props list by a set of small parts the consumer assembles, in the style of Radix primitives.

```tsx
<Composer.Provider>
  <Composer.Frame>
    <Composer.Header />
    <Composer.Input />
    <Composer.Footer>
      <Composer.Submit />
    </Composer.Footer>
  </Composer.Frame>
</Composer.Provider>
```

Each use case (channel, thread, edit, forward) is its own component that composes the same parts differently. Shared internals, re-assembled per use case.

### 2. Express variants by rendering or not rendering, not by flags

- Edit mode has no attachments? Don't render `<Composer.DropZone />`. No `isEditing` check, no `disableDrop` prop.
- Thread composer needs an "also send to channel" checkbox? Render `<Composer.AlsoSendToChannel />` in the thread version only.
- DM thread says "also send as direct message"? That is a different component, not another conditional inside the first.

A variant is a different composition. The absence of a part is the feature.

### 3. Use JSX, not config arrays, for UI

Replace `actions: [...]` with one component per action (each can wrap a shared button). Custom styling or a one-off action is trivially an ordinary JSX element. The speaker: he has yet to find a better abstraction than JSX for UI.

```tsx
// Instead of an array with flags and a loop
<Composer.Footer>
  <Composer.PlusMenu />
  <Composer.Emojis />
  <Composer.TextFormat />
  <Composer.Mentions />
</Composer.Footer>
```

One-off pieces (the Cancel and Save buttons in edit mode) are just inline `<Button>` elements. No `renderSubmit` prop needed.

### 4. Abstract when it helps, and keep an escape hatch

Several composers share the same bottom-left actions, so wrap them:

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

`CommonActions` is a **reusable mini-monolith built from the compound parts** and takes no `isEditing`-style prop. Composers that fit use it; composers that don't (edit, forward) just render individual items. You can always drop down a level. And per the speaker, you don't always need to abstract; sometimes shared internals simply get re-implemented per use case.

### 5. Define a context interface; let the provider own the implementation

Children read from one interface and don't know how state is stored.

```tsx
type ComposerContextValue = {
  state: ComposerState;          // what to render
  actions: ComposerActions;      // update, submit, ...
  meta: { inputRef: RefObject<TextInput | null> }; // things outside "state"
};
```

- `state` + `actions` are the common interface. The speaker also likes a `meta` property for things that live outside internal state, e.g. an **input ref**, so any component can focus the input without prop drilling or `useImperativeHandle`. (Naming is preference.)
- The **component that renders the provider** decides the implementation:
  - Forward-message composer (ephemeral, lives in a modal, discard warning on close): `useState`.
  - Channel composer (synced across devices): a global hook (`useGlobalChannel`) that sends messages and syncs to the backend.
- As long as the implementation returns something conforming to the interface, children work unchanged. Swapping state management means swapping **one provider** at the root, not adding boolean checks inside the component.
- `ComposerInput` etc. just render from context. They are agnostic to how state/actions are implemented.

---

## Lift state when UI lives outside the box

If one thing to take away: *many problems are solved by lifting state higher in the tree.*

Case: the forward-message modal has a **Forward** button row *below* the composer box, outside the frame. Bad answers: sync form state upward in `useEffect`, a shared ref with `onSubmit`, an `onFormStateDidChange` callback, or a `renderFooter` prop (it can't work because the composer has nothing to render there).

Good answer: pull the provider into its own component that takes `children`, and render the composer **and** the outside UI as children of it.

```tsx
function ForwardMessageDialog({ message }: { message: Message }) {
  return (
    <ForwardMessageProvider>
      <ForwardMessageComposer />          {/* uses Composer.* parts */}
      <MessagePreview message={message} />  {/* unrelated, renders under it */}
      <ForwardActions />                  {/* uses useComposer().actions.submit */}
    </ForwardMessageProvider>
  );
}
```

The provider wraps more than the frame, so anything inside it can reach `state` and `actions` without being inside the composer's UI. The row of buttons is "entirely isolated from its UI" yet uses its state and actions.

---

## Performance note

If frequent context re-renders worry you, the speaker's answer is the **React Compiler**. He pasted a context-consuming input into the compiler playground and showed the compiled output memoizes the component based only on the context fields it actually reads. **(addition)** Without the compiler, consider splitting state and actions into separate contexts, or memoizing the value, so components that only call actions don't re-render on every keystroke.

---

## Workflow: refactoring a boolean-prop component

1. **Inventory the variants.** List every use case and diff them against each other (like the talk's channel vs edit comparison). Note differences in: UI parts shown, actions available, submit/cancel behavior, drop/attach behavior, and where the state lives.
2. **Find the shared atoms.** Header, input, frame, footer, buttons, individual actions. These become the compound parts.
3. **Define the context interface** (`state`, `actions`, `meta`) from what the parts actually need. Keep it implementation-agnostic.
4. **Extract parts that read from context.** Each part uses the hook, takes no variant flags.
5. **Build one provider per state strategy** (local `useState`, global/synced hook, etc.), each implementing the same interface.
6. **Rebuild each variant as its own component**: provider + the parts it needs. Delete the flag props. Variant differences become "render this, don't render that."
7. **Extract shared groups** (like `CommonActions`) only where two or more variants really share them.
8. **If something outside the box needs the state**, lift the provider up and make that UI a sibling child.

## Smells to look for in review

- `is*` / `has*` / `hide*` / `disable*` props that change which subtree renders
- The same flag read in 3+ places inside one component
- A flag passed through a component that doesn't use it, only to reach a child
- A config array with growing optional fields and a loop to interpret it
- `useEffect` copying child state up to the parent
- `render*` props added to patch around structure
- A component that internally branches on where it's used (channel vs thread vs DM)

## When generating new React code

Follow the same rules so the output is easy to extend and hard to get wrong:

1. Don't add a boolean prop to select a variant. Create the variant as a new composition of existing parts.
2. Put state behind a context interface (`state`, `actions`, `meta`); put the implementation in the provider component only.
3. Children consume context; they don't take state/handlers as props unless they are truly standalone.
4. Prefer JSX children over config arrays and render props.
5. If UI outside a component needs its state, lift the provider; don't sync state upward.
6. Don't abstract preemptively; extract a shared group only after two use cases need it.

The speaker reports that feeding slides from this talk to an AI tool and prompting it to use composition for state management and rendering produced fewer bugs and code that was pleasant to edit by hand, "pair programming" with the AI. His argument that this still matters in an AI era: how you instruct the code generator depends on how clear the codebase's structure is, and a good codebase matters more, not less.

## Judgment notes (additions, not from the talk)

- Composition is a response to *variant sprawl*. A component with one or two stable variants and a single boolean is usually fine; don't convert it by reflex.
- Compound components add files and indirection. The payoff comes when variants multiply or differ structurally.
- Keep the context interface small and stable; every field you add is a contract all providers must satisfy.
- Prop drilling of one or two levels is fine. The target is drilling of *variant flags* through components that don't use them.
