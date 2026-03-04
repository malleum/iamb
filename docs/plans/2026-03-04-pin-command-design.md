# Design: :pin / :unpin / :pins Commands

**Date:** 2026-03-04
**Status:** Approved

## Overview

Add three new vim commands to iamb:

- `:pin` — pin the currently focused message using the Matrix `m.room.pinned_events` state event
- `:unpin` — unpin the currently focused message
- `:pins` — open a side panel listing all pinned messages in the current room; pressing Enter scrolls the room to that message

Pins are server-side (Matrix protocol), visible to all room members. Writing pins requires room admin/mod permission.

## Data Layer (`src/base.rs`)

Add to `RoomInfo`:

```rust
pub pinned_messages: Vec<OwnedEventId>,
```

Methods on `RoomInfo`:
- `pin_message(event_id: OwnedEventId)` — appends if not already present
- `unpin_message(event_id: &EventId)` — removes if present
- `get_pinned_items(&self) -> Vec<PinItem>` — returns pinned messages with preview text and sender for panel display

`m.room.pinned_events` state events (already detected in `src/message/state.rs`) update this field directly instead of only emitting a system message.

## Actions (`src/base.rs`)

```rust
// MessageAction — operates on currently focused message
MessageAction::Pin
MessageAction::Unpin

// RoomAction
RoomAction::PinList(Box<CommandContext>)   // opens :pins panel
RoomAction::JumpToMessage(OwnedEventId)    // scrolls room scrollback to message
```

## Commands (`src/commands.rs`)

| Command | Args | Action dispatched |
|---------|------|-------------------|
| `:pin` | none | `MessageAction::Pin` |
| `:unpin` | none | `MessageAction::Unpin` |
| `:pins` | none | `RoomAction::PinList` |

## Worker (`src/worker.rs`)

`MessageAction::Pin/Unpin` handler:
1. Fetch current pinned list from `RoomInfo.pinned_messages`
2. Add or remove the event ID
3. Call `room.set_pinned_events(updated_list)` via Matrix SDK
4. Display status message on success/failure

`m.room.pinned_events` sync events update `RoomInfo.pinned_messages`.

## UI / Window Layer (`src/windows/`)

New types:

```rust
pub struct PinItem {
    pub event_id: OwnedEventId,
    pub room_id: OwnedRoomId,
    pub sender: String,
    pub preview: String,   // first ~80 chars of message body
}

type PinListState = ListState<PinItem, IambInfo>;
```

New enum variants:

```rust
// IambId (src/base.rs)
IambId::PinList(OwnedRoomId)

// IambWindow (src/windows/mod.rs)
IambWindow::PinList(PinListState, OwnedRoomId)
```

Panel opens as a side split (same as `:reactions`). Each row displays `sender: preview`. Pressing Enter dispatches `RoomAction::JumpToMessage(event_id)` to the room window.

## Jump-to (`src/windows/room/mod.rs`)

Handle `RoomAction::JumpToMessage(event_id)`: walk `RoomInfo.messages` (a `BTreeMap<MessageKey, Message>`) to find the entry matching the event ID, then set the scrollback cursor to that position.

## Patterns to Follow

The `:reactions` command implementation (commit 9aaa764) is the primary reference:
- `ReactionListState` / `ReactionItem` → `PinListState` / `PinItem`
- `IambId::Reactions` / `IambWindow::Reactions` → `IambId::PinList` / `IambWindow::PinList`
- `RoomAction::Reactions` → `RoomAction::PinList`
