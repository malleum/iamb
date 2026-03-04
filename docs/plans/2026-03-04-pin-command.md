# Pin Commands Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add `:pin`, `:unpin`, and `:pins` commands to iamb for managing Matrix room pinned events.

**Architecture:** Mirror the `:reactions` / `SearchResultItem` patterns. `RoomInfo` caches the pinned event list synced from `m.room.pinned_events`. `:pin`/`:unpin` call `room.send_state_event(RoomPinnedEventsEventContent)` after mutating the local cache. `:pins` opens a side panel whose items dispatch `pending_scrollback_jump` on Enter, the same mechanism used by `:search`.

**Tech Stack:** Rust, matrix-sdk 0.14.0, ratatui 0.29.0, modalkit 0.0.24, ruma

---

### Task 1: Add `pinned_messages` field to `RoomInfo`

**Files:**
- Modify: `src/base.rs:909-988`

**Step 1: Add field to struct**

In `src/base.rs`, find the `pub struct RoomInfo` definition (line 909). Add after `pub polls:` (line 934):

```rust
    /// The pinned event IDs for this room, synced from `m.room.pinned_events`.
    pub pinned_messages: Vec<OwnedEventId>,
```

**Step 2: Add `Default::default()` in `RoomInfo::default()`**

In `RoomInfo::default()` (line 965), add after `polls: Default::default(),` (line 976):

```rust
            pinned_messages: Default::default(),
```

**Step 3: Add methods to `impl RoomInfo`**

After `get_event_mut` (around line 1081), add:

```rust
    /// Pin an event by ID. No-op if already pinned.
    pub fn pin_message(&mut self, event_id: OwnedEventId) {
        if !self.pinned_messages.contains(&event_id) {
            self.pinned_messages.push(event_id);
        }
    }

    /// Unpin an event by ID. No-op if not pinned.
    pub fn unpin_message(&mut self, event_id: &EventId) {
        self.pinned_messages.retain(|id| id.as_ref() != event_id);
    }
```

**Step 4: Build to verify**

```bash
cargo build 2>&1 | head -30
```
Expected: compiles (or only unrelated errors)

**Step 5: Commit**

```bash
git add src/base.rs
git commit -m "feat: add pinned_messages field and methods to RoomInfo"
```

---

### Task 2: Add action enum variants

**Files:**
- Modify: `src/base.rs:144-191` (MessageAction), `src/base.rs:450-486` (RoomAction)

**Step 1: Add to `MessageAction` enum**

After `MessageAction::Vote(String)` (line 190), add:

```rust
    /// Pin the currently selected message.
    Pin,

    /// Unpin the currently selected message.
    Unpin,
```

**Step 2: Add to `RoomAction` enum**

After `RoomAction::Reactions(Box<CommandContext>)` (line 470), add:

```rust
    /// Open the pins window for this room.
    PinList(Box<CommandContext>),
```

**Step 3: Build**

```bash
cargo build 2>&1 | head -50
```
Expected: new unreachable pattern warnings (fine), no errors

**Step 4: Commit**

```bash
git add src/base.rs
git commit -m "feat: add Pin, Unpin, PinList action variants"
```

---

### Task 3: Add `IambId::PinList` and `IambBufferId::PinList`

**Files:**
- Modify: `src/base.rs:1865-2198`

**Step 1: Add `IambId::PinList` variant**

After `IambId::Reactions(OwnedRoomId, OwnedEventId)` (line 1897), add:

```rust
    /// The `:pins` window for a given Matrix room.
    PinList(OwnedRoomId),
```

**Step 2: Add Display arm for `IambId::PinList`**

In `impl Display for IambId` (line 1900), after the `IambId::Reactions` arm (line 1922-1924), add:

```rust
            IambId::PinList(room_id) => {
                write!(f, "iamb://pins/{room_id}")
            },
```

**Step 3: Add Deserialize arm for `IambId::PinList`**

In `impl Visitor for IambIdVisitor` (line 1952), after the `"reactions"` arm (line 2078-2096), add:

```rust
            Some("pins") => {
                let [room_id] = path_rest[..] else {
                    return Err(E::custom("Invalid pins window URL"));
                };

                let room_id = OwnedRoomId::try_from(room_id).map_err(|e| {
                    E::custom(format!("Invalid room ID in pins window URL: {e}"))
                })?;

                Ok(IambId::PinList(room_id))
            },
```

**Step 4: Add `IambBufferId::PinList` variant**

After `IambBufferId::Reactions(OwnedRoomId, OwnedEventId)` (line 2173), add:

```rust
    /// The `:pins` window for a room.
    PinList(OwnedRoomId),
```

**Step 5: Add arm in `IambBufferId::to_window()`**

In `to_window()` (line 2178), after the `IambBufferId::Reactions` arm (line 2191-2193), add:

```rust
            IambBufferId::PinList(room) => IambId::PinList(room.clone()),
```

**Step 6: Build**

```bash
cargo build 2>&1 | head -50
```
Expected: new match non-exhaustive warnings, no errors

**Step 7: Commit**

```bash
git add src/base.rs
git commit -m "feat: add IambId::PinList and IambBufferId::PinList variants"
```

---

### Task 4: Register `:pin`, `:unpin`, `:pins` commands

**Files:**
- Modify: `src/commands.rs:300-309` (near `iamb_reactions`), `src/commands.rs:828-837` (registration block)

**Step 1: Add command functions**

After `fn iamb_reactions` (line 300-309), add:

```rust
fn iamb_pin(desc: CommandDescription, ctx: &mut ProgContext) -> ProgResult {
    if !desc.arg.text.is_empty() {
        return Result::Err(CommandError::InvalidArgument);
    }

    let act = IambAction::Message(MessageAction::Pin);
    let step = CommandStep::Continue(act.into(), ctx.context.clone());

    return Ok(step);
}

fn iamb_unpin(desc: CommandDescription, ctx: &mut ProgContext) -> ProgResult {
    if !desc.arg.text.is_empty() {
        return Result::Err(CommandError::InvalidArgument);
    }

    let act = IambAction::Message(MessageAction::Unpin);
    let step = CommandStep::Continue(act.into(), ctx.context.clone());

    return Ok(step);
}

fn iamb_pins(desc: CommandDescription, ctx: &mut ProgContext) -> ProgResult {
    if !desc.arg.text.is_empty() {
        return Result::Err(CommandError::InvalidArgument);
    }

    let open = IambAction::Room(RoomAction::PinList(ctx.clone().into()));
    let step = CommandStep::Continue(open.into(), ctx.context.clone());

    return Ok(step);
}
```

**Step 2: Register commands in `add_iamb_commands`**

After the `reactions` registration block (lines 833-837), add:

```rust
    cmds.add_command(ProgramCommand {
        name: "pin".into(),
        aliases: vec![],
        f: iamb_pin,
    });
    cmds.add_command(ProgramCommand {
        name: "unpin".into(),
        aliases: vec![],
        f: iamb_unpin,
    });
    cmds.add_command(ProgramCommand {
        name: "pins".into(),
        aliases: vec![],
        f: iamb_pins,
    });
```

**Step 3: Build**

```bash
cargo build 2>&1 | head -50
```

**Step 4: Commit**

```bash
git add src/commands.rs
git commit -m "feat: register :pin, :unpin, :pins commands"
```

---

### Task 5: Handle `MessageAction::Pin` and `MessageAction::Unpin` in chat.rs

**Files:**
- Modify: `src/windows/room/chat.rs:18-40` (imports), `src/windows/room/chat.rs:197` (match arm)

**Step 1: Add import for `RoomPinnedEventsEventContent`**

In the `matrix_sdk::ruma::events` imports block (lines 22-38), add to the `events::room` section:

```rust
        events::room::pinned_events::RoomPinnedEventsEventContent,
```

**Step 2: Add match arms for `Pin` and `Unpin`**

At the end of the `match act` block in `message_command` (before the final `}`, after `MessageAction::Vote`), add:

```rust
            MessageAction::Pin => {
                let room = self.get_joined(&store.application.worker)?;
                let event_id = msg.event.event_id().to_owned();

                let mut pinned = room
                    .load_pinned_events()
                    .await
                    .map_err(IambError::from)?
                    .unwrap_or_default();

                if pinned.contains(&event_id) {
                    return Err(UIError::Failure("Message is already pinned".into()));
                }

                pinned.push(event_id.clone());
                room.send_state_event(RoomPinnedEventsEventContent::new(pinned.clone()))
                    .await
                    .map_err(IambError::from)?;

                let info = store.application.rooms.get_or_default(self.room_id.clone());
                info.pinned_messages = pinned;

                Ok(Some(InfoMessage::from("Message pinned")))
            },
            MessageAction::Unpin => {
                let room = self.get_joined(&store.application.worker)?;
                let event_id = msg.event.event_id().to_owned();

                let mut pinned = room
                    .load_pinned_events()
                    .await
                    .map_err(IambError::from)?
                    .unwrap_or_default();

                if !pinned.contains(&event_id) {
                    return Err(UIError::Failure("Message is not pinned".into()));
                }

                pinned.retain(|id| id != &event_id);
                room.send_state_event(RoomPinnedEventsEventContent::new(pinned.clone()))
                    .await
                    .map_err(IambError::from)?;

                let info = store.application.rooms.get_or_default(self.room_id.clone());
                info.pinned_messages = pinned;

                Ok(Some(InfoMessage::from("Message unpinned")))
            },
```

Note: `msg.event.event_id()` needs to work for all event types. Check the `MessageEvent` enum — use the same pattern as `React` (lines 390-403) to extract the event_id, rejecting `Redacted` variants.

Actually, use this pattern instead (consistent with `React`):

```rust
            MessageAction::Pin => {
                let room = self.get_joined(&store.application.worker)?;
                let event_id = match &msg.event {
                    MessageEvent::EncryptedOriginal(ev) => ev.event_id.clone(),
                    MessageEvent::EncryptedRedacted(ev) => ev.event_id.clone(),
                    MessageEvent::Original(ev) => ev.event_id.clone(),
                    MessageEvent::Local(event_id, _) => event_id.clone(),
                    MessageEvent::State(ev) => ev.event_id().to_owned(),
                    MessageEvent::Poll(event_id, _, _, _) => event_id.clone(),
                    MessageEvent::Redacted(_) => {
                        return Err(UIError::Failure("Cannot pin a redacted message".into()));
                    },
                };

                let mut pinned = room
                    .load_pinned_events()
                    .await
                    .map_err(IambError::from)?
                    .unwrap_or_default();

                if pinned.contains(&event_id) {
                    return Err(UIError::Failure("Message is already pinned".into()));
                }

                pinned.push(event_id);
                room.send_state_event(RoomPinnedEventsEventContent::new(pinned.clone()))
                    .await
                    .map_err(IambError::from)?;

                let info = store.application.rooms.get_or_default(self.room_id.clone());
                info.pinned_messages = pinned;

                Ok(Some(InfoMessage::from("Message pinned")))
            },
            MessageAction::Unpin => {
                let room = self.get_joined(&store.application.worker)?;
                let event_id = match &msg.event {
                    MessageEvent::EncryptedOriginal(ev) => ev.event_id.clone(),
                    MessageEvent::EncryptedRedacted(ev) => ev.event_id.clone(),
                    MessageEvent::Original(ev) => ev.event_id.clone(),
                    MessageEvent::Local(event_id, _) => event_id.clone(),
                    MessageEvent::State(ev) => ev.event_id().to_owned(),
                    MessageEvent::Poll(event_id, _, _, _) => event_id.clone(),
                    MessageEvent::Redacted(_) => {
                        return Err(UIError::Failure("Cannot unpin a redacted message".into()));
                    },
                };

                let mut pinned = room
                    .load_pinned_events()
                    .await
                    .map_err(IambError::from)?
                    .unwrap_or_default();

                if !pinned.contains(&event_id) {
                    return Err(UIError::Failure("Message is not pinned".into()));
                }

                pinned.retain(|id| id != &event_id);
                room.send_state_event(RoomPinnedEventsEventContent::new(pinned.clone()))
                    .await
                    .map_err(IambError::from)?;

                let info = store.application.rooms.get_or_default(self.room_id.clone());
                info.pinned_messages = pinned;

                Ok(Some(InfoMessage::from("Message unpinned")))
            },
```

**Step 3: Build**

```bash
cargo build 2>&1 | head -50
```
Expected: compiles

**Step 4: Commit**

```bash
git add src/windows/room/chat.rs
git commit -m "feat: implement MessageAction::Pin and Unpin in chat.rs"
```

---

### Task 6: Handle `RoomAction::PinList` in `windows/room/mod.rs`

**Files:**
- Modify: `src/windows/room/mod.rs:353-377` (after Reactions arm)

**Step 1: Add match arm**

After the `RoomAction::Reactions` arm (lines 353-377), add:

```rust
            RoomAction::PinList(mut cmd) => {
                let width = Count::Exact(45);
                let act =
                    cmd.default_axis(Axis::Vertical).default_relation(MoveDir1D::Next).window(
                        OpenTarget::Application(IambId::PinList(self.id().to_owned())),
                        width.into(),
                    );

                Ok(vec![(act, cmd.context.clone())])
            },
```

**Step 2: Build**

```bash
cargo build 2>&1 | head -50
```

**Step 3: Commit**

```bash
git add src/windows/room/mod.rs
git commit -m "feat: handle RoomAction::PinList to open pins side panel"
```

---

### Task 7: Add `PinItem`, `PinListState` to `windows/mod.rs`

**Files:**
- Modify: `src/windows/mod.rs:1965` (after `ReactionListState` type alias)

**Step 1: Add `PinItem` struct and impls**

After `pub type ReactionListState = ListState<ReactionItem, IambInfo>;` (line 1965), add the full block:

```rust
#[derive(Clone)]
pub struct PinItem {
    event_id: OwnedEventId,
    room_id: OwnedRoomId,
    key: Option<MessageKey>,
    sender: String,
    preview: String,
}

impl Display for PinItem {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{}: {}", self.sender, self.preview)
    }
}

impl ListItem<IambInfo> for PinItem {
    fn show(
        &self,
        selected: bool,
        _: &ViewportContext<ListCursor>,
        store: &mut ProgramStore,
    ) -> Text<'_> {
        let style = if selected {
            Style::default().add_modifier(StyleModifier::REVERSED)
        } else {
            Style::default()
        };

        let sender_span = Span::styled(self.sender.clone(), style);
        let colon_span = Span::styled(": ", style);
        let preview_span = Span::styled(self.preview.clone(), style);

        Line::from(vec![sender_span, colon_span, preview_span]).into()
    }

    fn get_word(&self) -> Option<String> {
        self.event_id.to_string().into()
    }
}

impl Promptable<ProgramContext, ProgramStore, IambInfo> for PinItem {
    fn prompt(
        &mut self,
        act: &PromptAction,
        _: &ProgramContext,
        store: &mut ProgramStore,
    ) -> EditResult<Vec<(ProgramAction, ProgramContext)>, IambInfo> {
        match act {
            PromptAction::Submit => {
                if let Some(key) = &self.key {
                    store.application.pending_scrollback_jump =
                        Some((self.room_id.clone(), key.clone()));
                }
                Ok(vec![])
            },
            PromptAction::Abort(_) => {
                let msg = "Cannot abort entry inside a list";
                let err = EditError::Failure(msg.into());
                Err(err)
            },
            PromptAction::Recall(..) => {
                let msg = "Cannot recall history inside a list";
                let err = EditError::Failure(msg.into());
                Err(err)
            },
        }
    }
}

pub type PinListState = ListState<PinItem, IambInfo>;
```

**Note on `key: Option<MessageKey>`:** The `MessageKey` is a `(MessageTimeStamp, OwnedEventId)` tuple. When building `PinItem` in the draw step (Task 8), we'll look up the key via `info.get_message_key(&event_id)`. If the message isn't in local history yet, key is `None` and jumping is a no-op.

**Step 2: Build**

```bash
cargo build 2>&1 | head -50
```
Expected: compiles (or only unused import warnings)

**Step 3: Commit**

```bash
git add src/windows/mod.rs
git commit -m "feat: add PinItem and PinListState types"
```

---

### Task 8: Add `IambWindow::PinList` variant

**Files:**
- Modify: `src/windows/mod.rs` (multiple locations)

**Step 1: Add to `window_id!` macro**

The macro at line ~325 matches on `IambWindow` variants. After `IambWindow::Reactions($id, _, _) => $e,` (line 329), add:

```rust
            IambWindow::PinList($id, _) => $e,
```

**Step 2: Add to `IambWindow` enum**

After `IambWindow::Reactions(ReactionListState, OwnedRoomId, OwnedEventId)` (line 344), add:

```rust
    PinList(PinListState, OwnedRoomId),
```

**Step 3: Add draw arm**

In `Widget for IambWindow` draw/render (around line 693), after the `IambWindow::Reactions` arm (lines 693-717), add:

```rust
            IambWindow::PinList(state, room_id) => {
                let info = store.application.rooms.get_or_default(room_id.clone());
                let mut items = Vec::new();

                for event_id in &info.pinned_messages.clone() {
                    let key = info.get_message_key(event_id).cloned();
                    let (sender, preview) = if let Some(msg) = info.get_event(event_id) {
                        let sender = msg.sender.to_string();
                        let body = msg.event.body();
                        let preview = if body.len() > 80 {
                            format!("{}…", &body[..80])
                        } else {
                            body.into_owned()
                        };
                        (sender, preview)
                    } else {
                        (event_id.to_string(), "(message not loaded)".into())
                    };

                    items.push(PinItem {
                        event_id: event_id.clone(),
                        room_id: room_id.clone(),
                        key,
                        sender,
                        preview,
                    });
                }
                state.set(items);

                List::new(store)
                    .empty_message("No pinned messages in this room")
                    .empty_alignment(Alignment::Center)
                    .focus(focused)
                    .render(area, buf, state);
            },
```

**Step 4: Add dup arm**

In `dup()` method (around line 785), after `IambWindow::Reactions` dup arm, add:

```rust
            IambWindow::PinList(w, room_id) => {
                IambWindow::PinList(w.dup(store), room_id.clone())
            },
```

**Step 5: Add `window_id()` arm**

In `window_id()` method (around line 829), after `IambWindow::Reactions` arm, add:

```rust
            IambWindow::PinList(_, room_id) => IambId::PinList(room_id.clone()),
```

**Step 6: Add title arms**

Find the two title methods (short_title and title, around lines 861-915). After each `IambWindow::Reactions` arm, add:

```rust
            IambWindow::PinList(state, _) => {
                let n = state.len();
                let v = vec![
                    bold_span("Pins "),
                    Span::styled(format!("({n})"), bold_style()),
                ];
                Line::from(v)
            },
```

**Step 7: Add `IambId::PinList` → `IambWindow::PinList` creation**

In `open()` (around line 984), after the `IambId::Reactions` arm (lines 984-988), add:

```rust
            IambId::PinList(room_id) => {
                let id = IambBufferId::PinList(room_id.clone());
                let list = PinListState::new(id, vec![]);
                Ok(IambWindow::PinList(list, room_id))
            },
```

**Step 8: Build**

```bash
cargo build 2>&1 | head -50
```
Expected: compiles

**Step 9: Commit**

```bash
git add src/windows/mod.rs
git commit -m "feat: add IambWindow::PinList with render, dup, and id support"
```

---

### Task 9: Sync `pinned_messages` from Matrix state events in `worker.rs`

**Files:**
- Modify: `src/worker.rs:54-60` (imports), `src/worker.rs:1204` (after state_event_display handler)

**Step 1: Add import**

In the `events::room` import block (lines 54-60), add:

```rust
                pinned_events::RoomPinnedEventsEventContent,
```

**Step 2: Add event handler for pinned events sync**

After the `state_event_display` handler block (lines 1205-1217), add a new unconditional handler:

```rust
        let _ = self.client.add_event_handler(
            |ev: SyncStateEvent<RoomPinnedEventsEventContent>,
             room: MatrixRoom,
             store: Ctx<AsyncProgramStore>| {
                async move {
                    let room_id = room.room_id();
                    let mut locked = store.lock().await;
                    let info = locked.application.get_room_info(room_id.to_owned());

                    if let Some(original) = ev.as_original() {
                        info.pinned_messages = original.content.pinned.clone();
                    }
                }
            },
        );
```

**Step 3: Build**

```bash
cargo build 2>&1 | head -50
```
Expected: compiles

**Step 4: Commit**

```bash
git add src/worker.rs
git commit -m "feat: sync pinned_messages from m.room.pinned_events state events"
```

---

### Task 10: Final build and smoke test

**Step 1: Full build**

```bash
cargo build 2>&1
```
Expected: 0 errors

**Step 2: Run tests**

```bash
cargo test 2>&1
```
Expected: all pass

**Step 3: Check non-exhaustive match warnings**

```bash
cargo build 2>&1 | grep "non-exhaustive"
```
If any non-exhaustive match warnings remain, fix them — they indicate a missing arm in some match block.

**Step 4: Final commit**

If you needed any fixup during testing:

```bash
git add -p
git commit -m "fix: address any remaining non-exhaustive match warnings for pin commands"
```

---

## Key Reference: Patterns to Follow

| What | Where to look |
|------|--------------|
| Side panel open | `RoomAction::Reactions` handler in `src/windows/room/mod.rs:353-377` |
| List item with jump | `SearchResultItem` in `src/windows/mod.rs:1777-1884` |
| pending_scrollback_jump | `src/windows/mod.rs:1868` (set) / `src/windows/room/scrollback.rs:1302` (consumed) |
| MessageAction async SDK call | `MessageAction::React` in `src/windows/room/chat.rs:373-416` |
| State event sync handler | `AnySyncStateEvent` handler in `src/worker.rs:1206-1217` |
| IambWindow::Reactions draw | `src/windows/mod.rs:693-717` |
