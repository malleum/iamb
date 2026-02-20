# Poll Support Design

## Overview

Add Matrix poll support to iamb: viewing polls with bar chart visualization, voting via `:vote` command, and creating polls via `:poll` command. Uses the unstable MSC3381 poll events (the only ones supported in current room versions).

## Data Model

New `PollState` struct in `RoomInfo`:

```rust
pub struct PollState {
    pub question: String,
    pub answers: Vec<(String, String)>,  // (answer_id, answer_text)
    pub votes: HashMap<OwnedUserId, Vec<String>>,  // user -> selected answer_ids
    pub kind: PollKind,  // Disclosed or Undisclosed
    pub ended: bool,
}
```

Stored in `RoomInfo.polls: HashMap<OwnedEventId, PollState>` keyed by poll start event ID.

## Event Handling (worker.rs)

Three new event handlers registered in `add_event_handler`:

- **`SyncMessageLikeEvent<UnstablePollStartEventContent>`** - Creates `PollState`, inserts `MessageEvent::Poll(...)` into the timeline
- **`SyncMessageLikeEvent<UnstablePollResponseEventContent>`** - Updates votes in the referenced poll's `PollState`
- **`SyncMessageLikeEvent<UnstablePollEndEventContent>`** - Sets `ended = true` on the referenced poll

## MessageEvent Variant

```rust
pub enum MessageEvent {
    // ... existing variants ...
    Poll(Box<OriginalRoomMessageLikeEvent<NewUnstablePollStartEventContent>>),
}
```

The Poll variant stores the original event for event_id, sender, timestamp access.

## Rendering

In `show_msg()`, when the message is a Poll variant, render using poll state from `RoomInfo`:

```
Poll: What should we have for lunch?
 [1] Pizza     ████░░ 4 (57%)
 [2] Sushi     ██░░░░ 2 (29%)  ✓
 [3] Tacos     █░░░░░ 1 (14%)
```

- Numbered options with ASCII bar charts proportional to available width
- Vote counts and percentages
- Checkmark on the current user's selection(s)
- Undisclosed polls (before ending): show options without counts, only show user's own vote marker
- Ended polls: show "[Closed]" label after the question

## Commands

### `:vote <n>` or `:vote <name>`

- `<n>` - 1-indexed option number
- `<name>` - partial case-insensitive match on option text
- Sends `UnstablePollResponseEventContent` referencing the poll's event ID
- Errors if cursor isn't on a poll, poll is ended, or name is ambiguous

### `:poll <question> | <option1> | <option2> [| ...]`

- Pipe-separated: question first, then 2+ options
- Sends `NewUnstablePollStartEventContent` to the current room
- Errors if fewer than 2 options provided

## Configuration

New tunable in `TunableValues`:

```rust
pub poll_display: bool,  // default: true
```

Controls whether polls render with rich visualization or show the plain text fallback.

## Files Changed

| File | Change |
|------|--------|
| `src/base.rs` | Add `MessageAction::Vote(String)`, `PollState` struct, `RoomInfo.polls` field, poll insert/update methods |
| `src/message/mod.rs` | Add `MessageEvent::Poll(...)` variant, poll rendering in `show_msg()`, `body()`, `event_id()` etc. |
| `src/worker.rs` | Add event handlers for UnstablePollStart, UnstablePollResponse, UnstablePollEnd |
| `src/windows/room/chat.rs` | Handle `MessageAction::Vote` - send poll response event |
| `src/commands.rs` | Add `:vote` and `:poll` command parsing |
| `src/config.rs` | Add `poll_display` tunable |
