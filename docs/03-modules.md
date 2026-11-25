# Modules

A module is a group of tools. Compared to space tools, module tools are always loaded into the agent's context.

You create your own modules and add your tools:

```go
func shipping(ctx context.Context) neui.AnyModule {
	m := neui.NewModule("shipping", "Carrier quotes, labels and tracking for the shop's parcels.")
	m.Tool("quote", "Quote the cheapest carrier for a parcel.", quote)
	m.Tool("label", "Make the shipping label of a parcel.", label)
	...
	return m
}
```

```go
func quote(ctx context.Context, in QuoteInput) (Quote, error) {
	...
}
```

The agent calls the tools as `shipping__quote` and `shipping__label`. The module's description goes into the system prompt.

You enable a module globally or per session. neui calls `shipping` each time a thread opens.

For all users, when you create the app:

```go
app := doors.NewApp(page, neui.WithProvider(provider), neui.WithModules(shipping))
```

For one user, when you enable neui:

```go
err := neui.Enable(ctx, neui.OptionModules(shipping))
```

> The agent calls module tools directly and from scripts. Scripts need the output schema, so it is included in the tool's description. At `TierAssistant` there are no scripts, and the agent gets only a short summary of the output.

neui prepares the schemas of the input and output types for you, with [swaggest/jsonschema-go](https://github.com/swaggest/jsonschema-go). So it is always a good idea to add schema tags:

```go
type QuoteInput struct {
	Country string  `json:"country" required:"true" description:"destination country as an ISO code, like DE"`
	Weight  float64 `json:"weight" required:"true" minimum:"0.1" description:"parcel weight in kg"`
}

type Quote struct {
	_       struct{} `description:"The cheapest carrier for the parcel."`
	Carrier string   `json:"carrier" description:"carrier name"`
	Price   float64  `json:"price" description:"price in USD"`
	Days    int      `json:"days" description:"working days until delivery"`
}
```

The `description` tag of a `_ struct{}` field describes the type itself.

The output schema is added to the tool's description. What the agent gets as the description of `shipping__quote` at `TierDeveloper` and `TierAgent`:

```text
Quote the cheapest carrier for a parcel.
Output schema: {"description":"The cheapest carrier for the parcel.","properties":{"carrier":{"description":"carrier name","type":"string"},...},"type":"object"}
```

And at `TierAssistant`:

```text
Quote the cheapest carrier for a parcel.
Result: The cheapest carrier for the parcel.
carrier: carrier name
days: working days until delivery
price: price in USD
```

Only the described top-level fields are listed there.

A direct tool call returns the output to the agent as JSON. Some types change that: a `string` or a type with a `String()` method arrives as plain text, and a type that implements `api.Container` as its native content blocks. `api.Image` and `api.Document` are ready-made containers:

```go
type Container interface {
	Contents() []UserContent
}
```

A script always gets the output as JSON, and the output schema describes that JSON.

You can check who called the tool, the agent or a script:

```go
func label(ctx context.Context, in ParcelInput) (Label, error) {
	if neui.ToolFromJS(ctx) {
		return Label{}, errors.New("label is not available from scripts")
	}
	...
}
```

`neui.ToolFromAgent(ctx)` reports a direct call.

Tools have access to the agent's workspace. `neui.ToolFS(ctx)` returns the same filesystem the agent's file tools and scripts work with, so the agent opens what you write by its path:

```go
func label(ctx context.Context, in ParcelInput) (Label, error) {
	...
	path := "labels/" + in.Parcel + ".pdf"
	f, err := neui.ToolFS(ctx).Create(path)
	...
	return Label{
		Path: path,
	}, nil
}
```

A handler's `ctx` also tells it about the thread:

| Function | Returns |
| --- | --- |
| `neui.ToolThreadID(ctx)` | The ID of the thread the call belongs to, for example to keep your own state per conversation. |
| `neui.ToolModel(ctx)` | The model the thread runs on, with its context, image and request limits. |
| `neui.ToolConfig(ctx)` | The thread limits: the app's `WithThreadConfig` with defaults filled in. |

## When to use modules

Each module increases the thread's starting context, compared to space tools, which are discovered on demand. On the other hand, it goes into the model's cache and so costs almost nothing: don't be afraid to use modules.

Modules make sense for general-purpose tools that are not tied to your app's business logic and don't depend on page state.

## Tasks

A tool can start a long-running task that the user can cancel. `neui.ToolTask` registers it, and the tool returns at once:

```go
func track(ctx context.Context, in ParcelInput) (Tracking, error) {
	work, cancel := context.WithCancelCause(neui.ToolThreadContext(ctx))
	task, err := neui.ToolTask(ctx, "tracking parcel "+in.Parcel, nil, cancel)
	if err != nil {
		...
	}
	go watch(work, task, in.Parcel)
	return Tracking{
		Deferred: true,
	}, nil
}
```

The comment is shown to the user while the task runs. `cancel` is called when the user cancels the task. Run the work on `neui.ToolThreadContext(ctx)`: the call's own `ctx` ends when the tool returns.

A script cannot start a task: `ToolTask` returns an error there.

The task delivers its result to the agent later:

```go
func watch(ctx context.Context, task neui.Task, parcel string) {
	...
	task.Ok(result)
}
```

`Ok` and `Err` end the task. The result reaches the agent as a `<task_result>` block in the conversation.

## Asks

A module can declare asks: questions shown to the user in the chat, in place of the message box. An ask handler gets the agent's input and returns an `api.ChatAsk`:

```go
type ChatAsk struct {
	Form   gox.Elem
	Button gox.Elem
	Cancel func(context.Context)
	Error  doors.Beam[string]
}
```

`Form` is the question and its inputs. `Button` is shown in place of the send button, and its handler passes the answer with `answer.Submit`. `Error` is a message shown next to the question. Leave `Cancel` empty: neui sets it.

```gox
m.Ask("pickup", "Ask the user to choose a pickup slot.", func(in PickupInput, answer neui.Answer[Pickup]) api.ChatAsk {
	return api.ChatAsk{
		Form:   <fieldset>...</fieldset>,
		Button: <button (doors.AClick{...})>Book</button>,
	}
})
```

The agent calls an ask as a tool: the input is its arguments, the answer is the tool's output. The call waits for the user and fails when they dismiss the question. neui adds a note about this to the description. What the agent gets as the description of `shipping__pickup`:

```text
Ask the user to choose a pickup slot.

This shows the user something in the chat and waits for them to answer, which can take a long time. ...
Output schema: {"properties":{"slot":{"description":"the slot the user chose","type":"string"}},"type":"object"}
```

## Default modules

Some modules are enabled by default:

- `chat` lets the agent name the chat.
- `time` gives it the server's and the browser's clock, and timers.
- `files` gives it the workspace and the user's uploads.
- `js` lets it run scripts that call your tools. It is not there at `TierAssistant`. `neui.WithJSConfig` sets its limits.
- `page` is how it works with your spaces, while the chat is on a page.

neui also ships some common optional modules:

- `neui.ModuleXLS(neui.XLSConfig{})` reads and builds Excel workbooks.
- `neui.ModuleWeb(neui.WebConfig{})` reads web pages, and searches the web when you set `WebConfig.Search` to a `neui.SearchBackend`: your search service, which returns results for a query, best first.
- `neui.ModuleMCP(neui.MCPConfig{...})` adds the tools of a remote MCP server. They follow the same rules as your own tools.
- `nekit.ModuleAsk()` adds an ask where the user picks from a list of options.
