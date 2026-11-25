# Chat

Every page has an instance, `neui.Instance(ctx)`, with everything a chat needs: the open thread, the user's other threads and a way to render them. Build your own chat on it, or use `nekit.Chat`, the chat neui ships.

## Instance

`neui.Instance(ctx)` returns the neui side of the page instance, the browser tab. Call it with any `ctx` of that page, in a render or in a handler:

```go
inst := neui.Instance(ctx)
```

```go
type Instance interface {
	State() doors.Beam[InstanceState]
	Chats() doors.Beam[map[ThreadID]InstanceChat]
	ThreadOpen(ctx context.Context, id ThreadID, meta any) error
	ThreadNew(ctx context.Context) error
	ThreadList(ctx context.Context, from, to time.Time, asc bool, limit int, name string) ([]Thread, error)
	Render(factory ChatFactory) gox.Elem
	...
}
```

- `State` is the state of neui in this tab, like `api.InstanceDisabled`.
- `Chats` holds the threads open in the user's tabs, this one included.
- `ThreadOpen` opens a thread in this tab. `meta` reaches the chat that opens as `ChatState.Meta`; pass nil when there is nothing to hand over.
- `ThreadNew` starts a new thread in this tab.
- `ThreadList` lists the user's threads between `from` and `to`, oldest first when `asc` is set, at most `limit` of them. A non-empty `name` keeps the threads whose name contains it, ignoring case.
- `Render` renders the chat with your factory: see [Render](#render).
- `Emit` sends an event of your app into the open thread: see [From your code](05-from-your-code.md).

It works before `neui.Enable`: `State` reports `api.InstanceDisabled`, and every method that returns an error returns `api.ErrDisabled`.

## Build your own chat

### Render

Render the chat once per page and pass your factory:

```gox
~(neui.Instance(ctx).Render(factory))
```

```go
type ChatFactory interface {
	Chat(thread ChatThread, state ChatState) Chat
	Placeholder(enabled doors.Beam[bool]) gox.Elem
}
```

`Placeholder` is shown while there is no thread. `enabled` is false while neui is not enabled for the session, and true while the thread is loading.

`Chat` is called each time the tab gets a thread: on the first render, when the user switches threads, and on `thread.Reload`. It returns your view of that thread:

```go
type Chat interface {
	Update(ctx context.Context, e ChatEvent)
	gox.Comp
}
```

### State

Build the whole view from `state`. It is a snapshot of the thread:

```go
type ChatState struct {
	ID    ThreadID
	Name  string
	Input ChatInput
	Round *Round
	Ask   *ChatAsk
	Tasks []Task
	Usage EventUsage
	...
}
```

`Input` is the user's draft. `Round` is the round in progress, or nil when the agent is idle. Earlier rounds are not in the state. Read them with `thread.RoundBefore`, page by page, as the user scrolls up:

```go
rounds, err := thread.RoundBefore(ctx, api.NoRoundID, 20)
```

### Events

Every later change reaches the view through `Update`, one event at a time, in order:

```go
func (c *chat) Update(ctx context.Context, e api.ChatEvent) {
	if api.Final(e) {
		...
	}
	switch e := e.(type) {
	case api.ChatEventRoundStart:
		...
	case api.ChatEventIndexed:
		...
	case api.ChatEventToolResults:
		...
	case api.ChatEventRoundEnd:
		...
	case api.ChatEventAsk:
		...
	case api.ChatEventAskDone:
		...
	case api.ChatEventCriticalError:
		...
	}
}
```

- `ChatEventRoundStart` adds a round. When `InputTaken` is set, clear the message box. When `Truncate` names a round, first drop that round and every round after it.
- `ChatEventIndexed` carries one piece of the streaming response. Apply `e.Event` to the block at `e.Event.At()`.
- `ChatEventToolResults` delivers the results of the round's tool calls.
- `ChatEventRoundEnd` ends the round. `State` says how: `api.RoundOk`, `api.RoundAbort`, `api.RoundRefuse` or `api.RoundError`.
- `api.Final(e)` is true when the agent stops. Switch the controls back to idle there: a round with tool calls is followed by another round.

The other events update the thread's name, the pending input, the tasks, the usage and the compaction state.

`ctx` is not batched. Open a Doors batch when one event makes several page updates.

### Commands

The user acts through `thread`:

```go
type ChatThread interface {
	Input(ctx context.Context, input Input) error
	Send(ctx context.Context, input Input) error
	Stop(ctx context.Context) error
	Unsend(ctx context.Context) error
	Retry(ctx context.Context) error
	Rewind(ctx context.Context, round RoundID, input Input) error
	Compact(ctx context.Context) error
	Reload(ctx context.Context)
	...
}
```

Call a command with the `ctx` of the handler that triggered it:

```gox
~>doors.AClick{
	On: func(ctx context.Context, _ doors.RequestPointer) bool {
		err := thread.Send(ctx, api.Input{
			Text: text,
		})
		...
	},
} <button>Send</button>
```

Don't change the view in the handler. The command's event reaches `Update` before the handler's request ends, so change the view there. A command the thread can't take now returns `api.ErrWrongState` and changes nothing.

`Send` on a busy thread queues the input, and `Unsend` takes it back. `Input` saves the draft, so it survives a reload. `Rewind` is edit and resend: it deletes the round and every round after it, and starts a new one with the input.

### Attachments

Upload the user's file into the thread's filesystem, and send its name with the input:

```go
entry, err := thread.FS().UserUpload(ctx, name, r)
...
err = thread.Send(ctx, api.Input{
	Text:        text,
	Attachments: []string{entry.Name},
})
```

### Asks

An ask arrives as `ChatState.Ask` or as `ChatEventAsk`. Show its `Form` in place of the message box and its `Button` in place of the send button, until `ChatEventAskDone`. Call `Cancel` from your own dismiss control. `Error` is a message to show next to the question.

### Critical errors

After `ChatEventCriticalError` the view gets no more events, and every `thread` command returns `api.ErrUnlinked`. Tell the user, then call `thread.Reload(ctx)`: the factory builds a new view.

### Thread list

The list of chats is built on the instance: `ThreadList` and `RoundSearch` read, `ThreadOpen` and `ThreadNew` switch the tab, `ThreadRename`, `ThreadClone` and `ThreadDelete` edit.

A new thread is stored on its first message. Until then `ThreadList` does not list it.

A thread is open in one tab only. When a tab opens a thread that another tab has open, the other tab moves to a new thread. `Chats` tells you which threads are open and in which tab, so you can mark them in the list.

> Start from the shipped chat: configure it, and fork it when the options are not enough.

## Default chat

`nekit.Chat` is a complete chat you configure with plain fields. Render it once on the page:

```gox
~nekit.Chat{
	Layout: nekit.ChatLayout{
		Start: nekit.ChatDock,
	},
	ThemeLight: chatTheme,
	Disable:    nekit.ChatSimple,
}
```

The zero value is a complete chat.

### Options

| Field | What it does |
| --- | --- |
| `MaxAttachmentSize` | The largest file a user may attach, in bytes. Default 12 MiB. |
| `MaxAttachments` | How many files one message may hold. Default 10. |
| `AttachmentTimeout` | The time limit of one file upload. Default 10 minutes. |
| `AsideRight` | Puts the chat list on the right side of the chat instead of the left. |
| `Framed` | Draws a rounded border around the chat. |
| `ThemeDark`, `ThemeLight` | Recolour the chat, see [Themes](#themes). |
| `Layout` | Where the chat sits, its sizes and how it opens. |
| `GreetingTitle` | The heading of a new chat before its first message: a string or a GoX element. nil shows "How can I help?", `""` leaves it out. |
| `GreetingBody` | What shows under the heading: a string or a GoX element. nil shows "Ask a question or describe what you need.", `""` leaves it out. |
| `Disable` | The features to turn off. |

#### Layout

| Field | What it does |
| --- | --- |
| `Side` | The page edge the chat sits at: `nekit.ChatRight` (default) or `nekit.ChatLeft`. |
| `Width` | The width of the floating window. Default `25rem`. |
| `Height` | The height of the floating window. Default `40rem`. |
| `DockWidth` | The width of the docked panel. The page makes room for it. Default `28rem`. |
| `Start` | How the chat opens on a first visit: `nekit.ChatFloat` (default) as a floating window, `nekit.ChatDock` docked to the page edge. |

The sizes are CSS lengths in `px`, `rem`, `em`, `vw`, `vh` or `%`, like `"30rem"`. An invalid one keeps the default.

On a first visit the chat is collapsed, a button in the corner. For the rest of the session it stays the way the user left it. On screens narrower than `40rem` the open chat fills the screen.

#### Disable

`Disable` takes a list of features:

```gox
~nekit.Chat{
	Disable: []nekit.ChatFeature{
		nekit.ChatAttachments,
		nekit.ChatDock,
	},
}
```

| Feature | What turning it off does |
| --- | --- |
| `ChatTokens` | Hides the context gauge, the token counts and the "Compact now" button. |
| `ChatThinking` | Hides the reasoning and summary blocks. |
| `ChatTools` | Hides the tool call blocks. |
| `ChatTasks` | Hides the background tasks chip and its list. |
| `ChatEdit` | Hides the edit and resend button on user messages. |
| `ChatRetry` | Hides the retry button on failed rounds. |
| `ChatAttachments` | Hides the attach button and the attached files. A large paste stays in the message as text. |
| `ChatSidebar` | Hides the chat list and its menu button. |
| `ChatThreadEdit` | Hides rename, clone and delete in the chat list. |
| `ChatMessageSearch` | Limits the search to chat names. Message contents are not searched. |
| `ChatGreeting` | Hides the greeting of a new chat. |
| `ChatFloat` | Keeps the chat docked and hides the button that switches the mode. |
| `ChatDock` | Keeps the chat floating and hides the button that switches the mode. |

With both `ChatFloat` and `ChatDock` off the chat floats.

`nekit.ChatSimple` is a ready list for non-technical users: `ChatTokens`, `ChatThinking`, `ChatTools` and `ChatTasks`.

### Themes

The chat has a light and a dark palette. It follows the CSS `color-scheme` of your page: dark where the page is dark, light where the page is light or sets nothing. To follow the user's system, set `color-scheme: light dark` on your page's root.

Recolour each palette with a `ChatTheme`:

```gox
~nekit.Chat{
	ThemeLight: nekit.ChatTheme{
		AppBg:  "#ffffff",
		AppInk: "var(--my-ink)",
	},
	ThemeDark: nekit.ChatTheme{
		AppBg: "#18181b",
	},
}
```

Each field takes any CSS colour value. An empty field keeps the default. A value that contains `{`, `}`, `;` or `<` is ignored.

| Field | What it colours |
| --- | --- |
| `AppBg` | The chat background. |
| `AppBorder` | Borders and dividers. Hover backgrounds derive from it. |
| `AppInk` | The main text. |
| `AppInkQuiet` | Secondary text, form accents and focus rings. |
| `PanelBg` | The background of the message box, popovers, form fields and message bubbles. |
| `PanelBorderFocus` | The border of the message box, input fields, checkboxes and radio buttons, and of hovered, pressed and focused controls. |
| `ButtonBorder` | The border of buttons and the background of pressed buttons. |
| `ButtonInk` | The text and icons of buttons. |
| `ButtonInkOff` | The text and icons of disabled buttons. |
| `SecondaryBg` | The background of secondary buttons, the context gauge and attached files. |
| `DangerInk` | The text of destructive actions and errors. |
| `DangerBgHover` | The fill of a hovered destructive button. |
| `DangerBgActive` | The fill of a pressed destructive button. |
| `DangerInkOnFill` | The text on a danger fill. |
| `LauncherBg` | The fill of the round button that opens the collapsed chat. Defaults to `AppInk`. |
| `LauncherInk` | The icon of that button. Defaults to `AppBg`. |
