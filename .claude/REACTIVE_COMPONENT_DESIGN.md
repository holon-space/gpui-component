# Designing Components for Reactive Backends

This guide covers how to design gpui-component UI components that integrate cleanly with reactive, event-sourced backends while remaining useful as standalone components for any use case.

## The Problem

Reactive backends follow this data flow:

```
User Action -> Operation (fire-and-forget) -> Backend -> CDC stream -> UI update
```

The UI is a *projection* of backend state, not the source of truth. But drag-and-drop, reordering, and other interactive gestures need immediate visual feedback — the user can't wait 50-200ms for a round-trip.

A naive component design forces a choice: either the component owns the data (breaking reactive architectures) or it waits for the backend (breaking UX). We need components that support both.

## Core Principle: State In, Events Out

Components should be **state projectors with event ports**:

- **State in**: The component renders whatever data its `Entity<State<T>>` contains. It does not decide where that data comes from.
- **Events out**: User interactions produce semantic events (not mutations). The caller decides what to do with them.

The component never writes to external systems. It never subscribes to streams. It never retries failed operations. It renders state and emits events.

## Pattern: Entity-Backed State

Use an `Entity<ComponentState<T>>` that the caller controls, not `&[T]` borrows or internal `Vec<T>`.

### Why Entity, not borrowed data

| Approach | Problem |
|----------|---------|
| `&[T]` borrowed | Caller can't update data between renders without rebuilding the component. CDC updates from a backend can't push new data in. |
| Internal `Vec<T>` | Component owns the data — backend can't reconcile, and the component becomes the source of truth. |
| `Entity<State<T>>` | Caller holds the entity, can mutate it from CDC handlers, async tasks, or anywhere. `cx.notify()` triggers re-render. The component reads but doesn't own. |

This is the established pattern in this codebase: `ListState<D>`, `TableState<D>`, `InputState`.

### Example

```rust
pub struct SortableState<T: Clone + 'static> {
    items: Vec<T>,
    dragging_from: Option<usize>,
    drop_target: Option<usize>,
}

impl<T: Clone + 'static> SortableState<T> {
    pub fn new(items: Vec<T>) -> Self { /* ... */ }

    /// Caller uses this to update items from CDC or any other source.
    pub fn set_items(&mut self, items: Vec<T>) {
        self.items = items;
    }

    pub fn items(&self) -> &[T] {
        &self.items
    }
}
```

The caller creates the entity and keeps a handle:

```rust
// Generic usage (no backend)
let state = cx.new(|_| SortableState::new(my_items));

// Reactive backend usage
let state = cx.new(|_| SortableState::new(initial_items));
// CDC handler updates the same entity:
cdc_stream.for_each(|changes| {
    state.update(cx, |s, _| s.set_items(apply_changes(s.items(), changes)));
});
```

## Pattern: Identity-Based Events

Callbacks should reference items by **stable identity**, not by index.

### Why not indices

In a reactive backend, the underlying data can change between the moment the user initiates a drag and the moment they complete it. Indices become stale. Other items may have been inserted or removed by a concurrent CDC update.

### What to use instead

Provide an ID extraction function and use IDs in all callbacks:

```rust
Sortable::new(state, |item| item.id.clone(), |item, ix, drag_state| {
    // render the item
})
.on_reorder(|item_id: &str, after_id: Option<&str>, window, cx| {
    // "item_id was dropped after after_id (None = first position)"
    // Caller maps this to whatever operation their backend needs.
})
```

This maps directly to backend operations like `move_block(id, parent_id, after_block_id)` without index translation.

### Dual access: IDs for callbacks, indices for internal layout

The component internally works with indices for layout calculations (drop slot geometry, insertion indicators). But all external-facing callbacks use item IDs. The translation happens inside the component.

## Pattern: Optimistic Updates Are the Caller's Responsibility

The component does NOT perform optimistic reordering on its own. Instead:

1. Component emits `on_reorder(item_id, after_id)`.
2. Caller decides whether to:
   - **Optimistically** mutate `SortableState` immediately AND dispatch the backend operation. When CDC arrives, it confirms or reverts.
   - **Wait** for the backend and let CDC update the state (simpler, slightly laggy).
   - **Do something else entirely** (local-only reorder, validation, confirmation dialog).

This keeps the component agnostic about consistency models.

### For reactive backends (optimistic)

```rust
.on_reorder(|item_id, after_id, window, cx| {
    // 1. Optimistic: reorder locally for instant feedback
    state.update(cx, |s, _| s.move_item(item_id, after_id));

    // 2. Dispatch operation to backend (fire-and-forget)
    dispatcher.execute("move_block", params![id: item_id, after: after_id]);

    // CDC will arrive later:
    // - Success: state already correct, no visible change
    // - Failure: CDC overwrites state, reverting the optimistic update
})
```

### For simple local usage

```rust
.on_reorder(|item_id, after_id, window, cx| {
    state.update(cx, |s, _| s.move_item(item_id, after_id));
})
```

Same component, no backend, works fine.

## Pattern: Separate Tight and Loose Concerns

Every interactive component has concerns that must be tightly integrated (part of the same render/layout pass) and concerns that can be loosely coupled (provided by the caller or composed externally).

### Tight: Must live inside the component

- **Geometry-dependent interactions**: Drop slot calculation, hit testing, insertion indicators — these need access to item bounds during the same layout pass.
- **Drag state visualization**: Which item is being dragged, where the drop indicator shows. This must be in sync with the layout.
- **Internal state machine**: Drag start/move/end tracking, hover detection over drop zones.

### Loose: Caller-provided or externally composed

- **Item rendering**: Always a closure or trait. The component should never know what the item looks like.
- **Drag preview rendering**: What appears under the cursor during a drag. Optional closure, with a sensible default.
- **Headers, footers, empty states**: Slots for surrounding content.
- **Data source / persistence**: The component is agnostic.
- **Cross-component coordination**: When items move between two instances of the same component (e.g., between two sortable lists), the caller coordinates — not the component.

### Rule of thumb

If a concern requires access to layout bounds, paint order, or the element tree during rendering → tight (inside the component). Everything else → loose (caller-provided).

## Checklist for New Components

When designing a new interactive component, verify these properties:

1. **State is an Entity**: Use `Entity<ComponentState<T>>` so external systems can update it.
2. **External mutation path**: `ComponentState` has public methods (`set_items`, `insert`, `remove`) that external code (CDC handlers, async tasks) can call to update state.
3. **Callbacks use stable IDs**: All external-facing events reference items by identity, not position.
4. **No built-in persistence**: The component does not save/load its own state. Serialization, if needed, is a separate concern (see `DockAreaState` for the pattern).
5. **No network/async in the component**: Operations, API calls, and stream subscriptions live in the caller.
6. **Optimistic updates are opt-in**: The component emits events; the caller decides whether to apply changes immediately or wait for confirmation.
7. **Render closures for content**: Item appearance is always caller-defined. Use `Fn(&T, ...) -> AnyElement` or a delegate trait.
8. **Axis/orientation is configurable**: Where possible, don't hardcode horizontal or vertical. A single `axis: Axis` parameter makes the component usable in more contexts.
9. **Sensible without callbacks**: The component should render and be interactive even if no callbacks are attached (e.g., drag reordering works visually but doesn't persist).

## Anti-Patterns

### Component subscribes to a data stream

```rust
// DON'T: component pulls its own data
impl MyComponent {
    fn new(stream: impl Stream<Item = Vec<T>>) -> Self {
        // subscribes internally, manages its own refresh cycle
    }
}
```

This couples the component to a specific data delivery mechanism. Instead, the caller subscribes and calls `state.update()`.

### Callback mutates component state directly

```rust
// DON'T: callback bypasses the state entity
.on_click(|item, window, cx| {
    self.items.remove(item);  // component owns and mutates the data
})
```

The component should emit an event. The caller (or the caller's CDC handler) decides whether and how the state changes.

### Index-based external API

```rust
// DON'T: expose indices in public callbacks
.on_reorder(|from_index: usize, to_index: usize| { ... })
```

Indices are internal. They're invalid the moment the underlying data changes, which in a reactive system can happen at any time.

### Component implements undo/redo

```rust
// DON'T: component maintains its own undo stack
impl SortableState {
    fn undo(&mut self) { self.items = self.history.pop(); }
}
```

Undo/redo belongs to the application layer. The backend already has an operation log and inverse operations. Component-level undo creates conflicting histories.

## Relationship to Existing Patterns

| Existing Component | Pattern Used | Notes |
|---|---|---|
| `Table` + `TableDelegate` | Delegate trait, state entity | Delegate provides data; `TableState` is the entity. Caller implements the trait. |
| `List` + `ListDelegate` | Delegate trait, state entity | Same pattern. Events like `Select`, `Confirm` are enums, not closures. |
| `Input` + `InputState` | State entity, events via `EventEmitter` | `InputState` is externally created. Changes emit events. |
| `Dock` / `Tiles` | State entity, drag infrastructure | `DragMoving` type for GPUI's `on_drag`. State serialization is separate (`DockAreaState`). |

New interactive components should follow these patterns. The delegate trait approach (`TableDelegate`) works well when the component needs many customization points. The closure approach works well for simpler components with 2-3 callbacks.
