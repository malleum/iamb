# Poll Support Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add Matrix poll support to iamb — viewing polls with bar chart visualization, voting via `:vote`, and creating polls via `:poll`.

**Architecture:** Polls use MSC3381 unstable event types (the only ones current room versions support). Poll start events become a new `MessageEvent::Poll` variant in the timeline. Poll state (votes, ended status) is stored in `RoomInfo.polls`. Rendering happens in `show_with_preview()` by checking poll state for the current event.

**Tech Stack:** Rust, ruma-events 0.31.0 poll types, matrix-sdk 0.14.0, ratatui for TUI rendering.

---

### Task 1: Add PollState and RoomInfo storage

**Files:**
- Modify: `src/base.rs:142-185` (MessageAction enum)
- Modify: `src/base.rs:886-926` (RoomInfo struct)
- Modify: `src/base.rs:1076-1095` (near insert_reaction)

**Step 1: Add poll imports to `src/base.rs`**

At the top of `src/base.rs`, find the existing ruma imports and add poll-related types. Locate the `use matrix_sdk::ruma::` block and add:

```rust
use matrix_sdk::ruma::events::poll::unstable_start::UnstablePollStartContentBlock;
use matrix_sdk::ruma::events::poll::start::PollKind;
```

**Step 2: Add PollState struct**

Add this struct near line 694 (near `MessageReactions` type alias):

```rust
/// State of a poll, including its question, answers, and accumulated votes.
#[derive(Clone, Debug)]
pub struct PollState {
    pub question: String,
    pub answers: Vec<(String, String)>,
    pub votes: HashMap<OwnedUserId, Vec<String>>,
    pub kind: PollKind,
    pub ended: bool,
}
```

**Step 3: Add `MessageAction::Vote` variant**

In the `MessageAction` enum (line 142), add after the `Unreact` variant:

```rust
    /// Vote on a poll with the selected option.
    ///
    /// The [String] is either a 1-indexed option number or a partial option name.
    Vote(String),
```

**Step 4: Add `SendAction::Poll` variant**

Find `pub enum SendAction` in base.rs and add:

```rust
    /// Send a new poll to the room.
    ///
    /// Arguments: (question, Vec<answer_text>)
    Poll(String, Vec<String>),
```

**Step 5: Add polls field to RoomInfo**

In the `RoomInfo` struct (line 886), add after the `reactions` field (line 908):

```rust
    /// A map of poll start event IDs to their poll state.
    pub polls: HashMap<OwnedEventId, PollState>,
```

**Step 6: Add poll methods to RoomInfo impl**

Add these methods to the `impl RoomInfo` block, after `insert_reaction` (around line 1095):

```rust
    /// Insert a new poll from an unstable poll start event.
    pub fn insert_poll(&mut self, event_id: OwnedEventId, poll_start: &UnstablePollStartContentBlock) {
        let question = poll_start.question.text.clone();
        let answers = poll_start
            .answers
            .iter()
            .map(|a| (a.id.clone(), a.text.clone()))
            .collect();
        let state = PollState {
            question,
            answers,
            votes: HashMap::new(),
            kind: poll_start.kind.clone(),
            ended: false,
        };
        self.polls.insert(event_id, state);
    }

    /// Record a vote on a poll.
    pub fn insert_poll_response(
        &mut self,
        poll_id: &EventId,
        sender: OwnedUserId,
        selections: Vec<String>,
    ) {
        if let Some(poll) = self.polls.get_mut(poll_id) {
            if !poll.ended {
                poll.votes.insert(sender, selections);
            }
        }
    }

    /// Mark a poll as ended.
    pub fn end_poll(&mut self, poll_id: &EventId) {
        if let Some(poll) = self.polls.get_mut(poll_id) {
            poll.ended = true;
        }
    }

    /// Get the poll state for an event ID.
    pub fn get_poll(&self, event_id: &EventId) -> Option<&PollState> {
        self.polls.get(event_id)
    }
```

**Step 7: Initialize polls in RoomInfo Default**

Find the `Default` impl for `RoomInfo` (or wherever it's constructed) and ensure `polls: HashMap::new()` is included. If `RoomInfo` derives `Default`, the HashMap will auto-initialize. If not, add it to the constructor.

**Step 8: Commit**

```bash
git add src/base.rs
git commit -m "feat(polls): add PollState, MessageAction::Vote, and RoomInfo poll storage"
```

---

### Task 2: Add MessageEvent::Poll variant

**Files:**
- Modify: `src/message/mod.rs:19-49` (imports)
- Modify: `src/message/mod.rs:462-554` (MessageEvent enum and impl)
- Modify: `src/message/mod.rs:1203-1240` (From impls)

**Step 1: Add poll imports**

In the imports at the top of `src/message/mod.rs`, add:

```rust
use matrix_sdk::ruma::events::poll::unstable_start::{
    NewUnstablePollStartEventContent,
    UnstablePollStartContentBlock,
};
```

**Step 2: Add Poll variant to MessageEvent**

In the `MessageEvent` enum (line 462), add:

```rust
    Poll(OwnedEventId, OwnedUserId, MilliSecondsSinceUnixEpoch, UnstablePollStartContentBlock),
```

This stores event_id, sender, timestamp, and the poll content block directly — avoiding complex generic event wrapper types.

**Step 3: Update MessageEvent::event_id()**

In the `event_id()` method (line 473), add the match arm:

```rust
            MessageEvent::Poll(event_id, _, _, _) => event_id.as_ref(),
```

**Step 4: Update MessageEvent::content()**

In the `content()` method (line 484), add:

```rust
            MessageEvent::Poll(_, _, _, _) => None,
```

**Step 5: Update MessageEvent::body()**

In the `body()` method (line 502), add:

```rust
            MessageEvent::Poll(_, _, _, poll) => {
                Cow::Owned(format!("Poll: {}", poll.question.text))
            },
```

**Step 6: Update MessageEvent::html()**

In the `html()` method (line 513), add:

```rust
            MessageEvent::Poll(_, _, _, _) => return None,
```

**Step 7: Update MessageEvent::redact()**

In the `redact()` method (line 534), add at the start of the match:

```rust
            MessageEvent::Poll(_, _, _, _) => return,
```

**Step 8: Update MessageEvent::is_emote()**

The `is_emote()` method uses `self.content()` which already returns None for Poll, so no change needed.

**Step 9: Commit**

```bash
git add src/message/mod.rs
git commit -m "feat(polls): add MessageEvent::Poll variant with all trait impls"
```

---

### Task 3: Add poll rendering in show_with_preview

**Files:**
- Modify: `src/message/mod.rs:800-839` (near push_reactions)
- Modify: `src/message/mod.rs:1036-1091` (show_with_preview)
- Modify: `src/message/mod.rs:1104-1142` (show_msg)

**Step 1: Add push_poll method to MessageFormatter**

Add this method to `MessageFormatter` impl, after `push_reactions` (around line 839):

```rust
    fn push_poll(
        &mut self,
        poll: &crate::base::PollState,
        user_id: &OwnedUserId,
        style: Style,
        text: &mut Text<'_>,
    ) {
        let bold = style.add_modifier(StyleModifier::BOLD);
        let bar_width: usize = 10;
        let width = self.width();

        // Question header
        let mut header = printer::TextPrinter::new(width, style, false, self.settings);
        let label = if poll.ended { "Poll [Closed]: " } else { "Poll: " };
        header.push_str(label, bold);
        header.push_str(&poll.question, bold);
        self.push_text(header.finish(), style, text);

        // Calculate totals
        let mut answer_counts: Vec<usize> = vec![0; poll.answers.len()];
        let total_votes: usize;
        let user_selections: Vec<String>;

        // Get user's votes
        user_selections = poll
            .votes
            .get(user_id)
            .cloned()
            .unwrap_or_default();

        // Count votes per answer
        for selections in poll.votes.values() {
            for sel in selections {
                if let Some(idx) = poll.answers.iter().position(|(id, _)| id == sel) {
                    answer_counts[idx] += 1;
                }
            }
        }
        total_votes = poll.votes.len();

        let is_disclosed = poll.kind == PollKind::Disclosed || poll.ended;

        for (idx, (answer_id, answer_text)) in poll.answers.iter().enumerate() {
            let mut line = printer::TextPrinter::new(width, style, false, self.settings);
            let voted = user_selections.contains(answer_id);

            // Number prefix
            line.push_str(&format!(" [{}] ", idx + 1), style);

            if is_disclosed {
                let count = answer_counts[idx];
                let pct = if total_votes > 0 {
                    (count as f64 / total_votes as f64 * 100.0) as u32
                } else {
                    0
                };

                // Answer text (padded)
                let answer_display = take_width(answer_text, 16);
                line.push_str(&format!("{:<16} ", answer_display), style);

                // Bar
                let filled = if total_votes > 0 {
                    (count * bar_width) / total_votes
                } else {
                    0
                };
                let empty = bar_width.saturating_sub(filled);
                let bar: String =
                    "\u{2588}".repeat(filled) + &"\u{2591}".repeat(empty);
                line.push_str(&bar, style);

                // Count and percentage
                line.push_str(&format!(" {} ({}%)", count, pct), style);
            } else {
                // Undisclosed: just show the answer text
                line.push_str(answer_text, style);
            }

            // Checkmark if user voted for this
            if voted {
                line.push_str(" \u{2713}", bold);
            }

            self.push_text(line.finish(), style, text);
        }
    }
```

**Step 2: Update show_with_preview to render polls**

In `show_with_preview` (line 1036), after the reactions block (line 1081-1084), add:

```rust
        if settings.tunables.poll_display {
            if let MessageEvent::Poll(_, _, _, _) = &self.event {
                if let Some(poll) = info.get_poll(self.event.event_id()) {
                    let user_id = &settings.profile.user_id;
                    fmt.push_poll(poll, user_id, style, &mut text);
                }
            }
        }
```

**Step 3: Commit**

```bash
git add src/message/mod.rs
git commit -m "feat(polls): add bar chart poll rendering in message timeline"
```

---

### Task 4: Add poll event handlers in worker.rs

**Files:**
- Modify: `src/worker.rs:39-69` (imports)
- Modify: `src/worker.rs:1033-1050` (after reaction handler)

**Step 1: Add poll imports**

In the imports section of `src/worker.rs`, within the `events::` block (around line 39-68), add:

```rust
            poll::unstable_start::UnstablePollStartEventContent,
            poll::unstable_response::UnstablePollResponseEventContent,
            poll::unstable_end::UnstablePollEndEventContent,
```

**Step 2: Add UnstablePollStart event handler**

After the reaction event handler (around line 1050), add:

```rust
        let _ = self.client.add_event_handler(
            |ev: SyncMessageLikeEvent<UnstablePollStartEventContent>,
             room: MatrixRoom,
             store: Ctx<AsyncProgramStore>| {
                async move {
                    let room_id = room.room_id();

                    if let Some(original) = ev.as_original() {
                        let poll_start = original.content.poll_start();

                        let event_id = original.event_id.clone();
                        let sender = original.sender.clone();
                        let timestamp = original.origin_server_ts;

                        let mut locked = store.lock().await;

                        let _ = locked.application.presences.get_or_default(sender.clone());

                        let info = locked.application.get_room_info(room_id.to_owned());
                        update_event_receipts(info, &room, &event_id).await;

                        info.insert_poll(event_id.clone(), poll_start);

                        let poll_content = poll_start.clone();
                        let msg_event = crate::message::MessageEvent::Poll(
                            event_id.clone(),
                            sender.clone(),
                            timestamp,
                            poll_content,
                        );
                        let msg = crate::message::Message::new(
                            msg_event,
                            sender,
                            timestamp.into(),
                        );
                        let key = (timestamp.into(), event_id.clone());
                        let loc = crate::base::EventLocation::Message(None, key.clone());
                        info.keys.insert(event_id, loc);
                        info.messages_mut().insert_message(key, msg);
                    }
                }
            },
        );
```

**Step 3: Add UnstablePollResponse event handler**

```rust
        let _ = self.client.add_event_handler(
            |ev: SyncMessageLikeEvent<UnstablePollResponseEventContent>,
             room: MatrixRoom,
             store: Ctx<AsyncProgramStore>| {
                async move {
                    let room_id = room.room_id();

                    if let Some(original) = ev.as_original() {
                        let mut locked = store.lock().await;
                        let info = locked.application.get_room_info(room_id.to_owned());

                        let poll_id = &original.content.relates_to.event_id;
                        let sender = original.sender.clone();
                        let selections = original.content.poll_response.answers.clone();

                        info.insert_poll_response(poll_id, sender, selections);
                    }
                }
            },
        );
```

**Step 4: Add UnstablePollEnd event handler**

```rust
        let _ = self.client.add_event_handler(
            |ev: SyncMessageLikeEvent<UnstablePollEndEventContent>,
             room: MatrixRoom,
             store: Ctx<AsyncProgramStore>| {
                async move {
                    let room_id = room.room_id();

                    if let Some(original) = ev.as_original() {
                        let mut locked = store.lock().await;
                        let info = locked.application.get_room_info(room_id.to_owned());

                        let poll_id = &original.content.relates_to.event_id;
                        info.end_poll(poll_id);
                    }
                }
            },
        );
```

**Step 5: Add messages_mut() accessor to RoomInfo**

In `src/base.rs`, add to the `impl RoomInfo` block:

```rust
    pub fn messages_mut(&mut self) -> &mut Messages {
        &mut self.messages
    }
```

**Step 6: Commit**

```bash
git add src/worker.rs src/base.rs
git commit -m "feat(polls): add event handlers for poll start, response, and end"
```

---

### Task 5: Add :vote command and action handler

**Files:**
- Modify: `src/commands.rs:1-32` (imports)
- Modify: `src/commands.rs:735-857` (add_iamb_commands)
- Modify: `src/windows/room/chat.rs:350-450` (action handlers)

**Step 1: Add `:vote` command function in commands.rs**

Add this function before `add_iamb_commands` (around line 735):

```rust
fn iamb_vote(desc: CommandDescription, ctx: &mut ProgContext) -> ProgResult {
    let text = desc.arg.text.trim().to_string();

    if text.is_empty() {
        return Err(CommandError::Error("Usage: :vote <number or option name>".into()));
    }

    let mact = IambAction::from(MessageAction::Vote(text));
    let step = CommandStep::Continue(mact.into(), ctx.context.clone());

    return Ok(step);
}
```

**Step 2: Register the command**

In `add_iamb_commands`, add:

```rust
    cmds.add_command(ProgramCommand {
        name: "vote".into(),
        aliases: vec![],
        f: iamb_vote,
    });
```

**Step 3: Add Vote action handler in chat.rs**

In `src/windows/room/chat.rs`, find the `MessageAction` match block (around line 330+). Add a new arm after the `React` handler (around line 414):

```rust
            MessageAction::Vote(selection) => {
                // Check the message is a poll
                let event_id = msg.event.event_id().to_owned();
                let poll = info.get_poll(&event_id).ok_or_else(|| {
                    UIError::Failure("This message is not a poll".into())
                })?;

                if poll.ended {
                    return Err(UIError::Failure("This poll has ended".into()));
                }

                // Resolve selection: number or name
                let answer_id = if let Ok(n) = selection.parse::<usize>() {
                    if n == 0 || n > poll.answers.len() {
                        let msg = format!(
                            "Invalid option number. Choose 1-{}",
                            poll.answers.len()
                        );
                        return Err(UIError::Failure(msg));
                    }
                    poll.answers[n - 1].0.clone()
                } else {
                    // Partial case-insensitive match on answer text
                    let lower = selection.to_lowercase();
                    let matches: Vec<_> = poll
                        .answers
                        .iter()
                        .filter(|(_, text)| text.to_lowercase().contains(&lower))
                        .collect();
                    match matches.len() {
                        0 => {
                            return Err(UIError::Failure(format!(
                                "No option matches {:?}",
                                selection
                            )));
                        },
                        1 => matches[0].0.clone(),
                        _ => {
                            return Err(UIError::Failure(format!(
                                "Ambiguous: {:?} matches multiple options",
                                selection
                            )));
                        },
                    }
                };

                let room = self.get_joined(&store.application.worker)?;
                let response = matrix_sdk::ruma::events::poll::unstable_response::UnstablePollResponseEventContent::new(
                    vec![answer_id],
                    event_id,
                );
                let _ = room.send(response).await.map_err(IambError::from)?;

                Ok(None)
            },
```

**Step 4: Add poll imports to chat.rs**

At the top of `src/windows/room/chat.rs`, ensure the necessary crate imports are present (the `matrix_sdk::ruma::events::poll` path is used inline above, so no additional import needed).

**Step 5: Commit**

```bash
git add src/commands.rs src/windows/room/chat.rs
git commit -m "feat(polls): add :vote command with number and name-based selection"
```

---

### Task 6: Add :poll command for creating polls

**Files:**
- Modify: `src/commands.rs` (add iamb_poll function and register it)
- Modify: `src/windows/room/chat.rs` or `src/worker.rs` (handle SendAction::Poll)

**Step 1: Add `SendAction::Poll` variant**

This was done in Task 1, Step 4. Ensure it's present in `src/base.rs`.

**Step 2: Add `:poll` command function in commands.rs**

Add before `add_iamb_commands`:

```rust
fn iamb_poll(desc: CommandDescription, ctx: &mut ProgContext) -> ProgResult {
    let text = desc.arg.text.trim().to_string();

    if text.is_empty() {
        return Err(CommandError::Error(
            "Usage: :poll <question> | <option1> | <option2> [| ...]".into(),
        ));
    }

    let parts: Vec<String> = text.split('|').map(|s| s.trim().to_string()).collect();

    if parts.len() < 3 {
        return Err(CommandError::Error(
            "A poll requires a question and at least 2 options, separated by |".into(),
        ));
    }

    let question = parts[0].clone();
    let options = parts[1..].to_vec();

    let sact = IambAction::from(SendAction::Poll(question, options));
    let step = CommandStep::Continue(sact.into(), ctx.context.clone());

    return Ok(step);
}
```

**Step 3: Register the command**

In `add_iamb_commands`, add:

```rust
    cmds.add_command(ProgramCommand {
        name: "poll".into(),
        aliases: vec![],
        f: iamb_poll,
    });
```

**Step 4: Handle SendAction::Poll in the worker/chat**

Find where `SendAction` variants are handled. Look in `src/windows/room/chat.rs` or `src/windows/room/mod.rs` for `SendAction::Upload` or similar, and add handling for `SendAction::Poll` nearby.

The handler should:

```rust
SendAction::Poll(question, options) => {
    let room = self.get_joined(&store.application.worker)?;

    let answers: Vec<_> = options
        .iter()
        .enumerate()
        .map(|(i, text)| {
            matrix_sdk::ruma::events::poll::unstable_start::UnstablePollAnswer::new(
                format!("opt{}", i),
                text.clone(),
            )
        })
        .collect();

    let poll_answers = answers.try_into().map_err(|_| {
        UIError::Failure("Invalid number of poll options (1-20 allowed)".into())
    })?;

    let poll_start = matrix_sdk::ruma::events::poll::unstable_start::UnstablePollStartContentBlock::new(
        question.clone(),
        poll_answers,
    );

    let content = matrix_sdk::ruma::events::poll::unstable_start::NewUnstablePollStartEventContent::plain_text(
        format!("Poll: {}", question),
        poll_start,
    );

    let _ = room.send(content).await.map_err(IambError::from)?;

    Ok(None)
}
```

**Step 5: Commit**

```bash
git add src/commands.rs src/base.rs src/windows/room/chat.rs
git commit -m "feat(polls): add :poll command for creating new polls"
```

---

### Task 7: Add poll_display config tunable

**Files:**
- Modify: `src/config.rs:560-585` (TunableValues)
- Modify: `src/config.rs:587-615` (Tunables)
- Modify: `src/config.rs:616-670` (merge and values)

**Step 1: Add to TunableValues**

In `TunableValues` (line 560), add after `reaction_shortcode_display`:

```rust
    pub poll_display: bool,
```

**Step 2: Add to Tunables**

In `Tunables` (line 587), add after `reaction_shortcode_display`:

```rust
    pub poll_display: Option<bool>,
```

**Step 3: Add to merge()**

In `Tunables::merge()` (line 616), add:

```rust
            poll_display: self.poll_display.or(other.poll_display),
```

**Step 4: Add to values()**

In `Tunables::values()` (line 652), add after `reaction_shortcode_display`:

```rust
            poll_display: self.poll_display.unwrap_or(true),
```

**Step 5: Commit**

```bash
git add src/config.rs
git commit -m "feat(polls): add poll_display config tunable"
```

---

### Task 8: Wire everything together and fix compilation

**Files:**
- All modified files

**Step 1: Run cargo check**

```bash
cargo check 2>&1
```

**Step 2: Fix any compilation errors**

Common issues to watch for:
- Missing match arms in exhaustive matches on `MessageEvent` (search for `match.*self.event` or `MessageEvent::`)
- Missing match arms in `MessageAction` matches (search for `MessageAction::`)
- Missing match arms in `SendAction` matches
- Missing match arms in `IambAction` matches (for `is_edit_sequence`, `is_last_action`)
- The `messages` field on `RoomInfo` is private — the `messages_mut()` accessor from Task 4 handles this
- Poll type imports that need adjusting

**Step 3: Fix all compilation issues iteratively**

Run `cargo check` after each fix until clean.

**Step 4: Commit**

```bash
git add -A
git commit -m "fix(polls): resolve compilation errors and wire together all poll support"
```

---

### Task 9: Test the implementation

**Step 1: Run existing tests**

```bash
cargo test 2>&1
```

**Step 2: Fix any test failures**

Existing tests may need updates if they match exhaustively on `MessageAction` or `SendAction`.

**Step 3: Build release**

```bash
cargo build 2>&1
```

**Step 4: Commit any fixes**

```bash
git add -A
git commit -m "fix(polls): fix test failures and ensure clean build"
```
