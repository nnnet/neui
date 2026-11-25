# From your code

Your Go code can tell the agent what happened in your app with `Instance.Emit`, and call the model itself with `neui.Generate`.

## Emit

`Emit` tells the agent that something happened in your app: a payment arrived, an import finished, a colleague changed the record on screen. Use it when the agent should know or react and the user did not type anything.

```go
func (p *page) paid(ctx context.Context, invoice Invoice) {
	...
	err := neui.Instance(ctx).Emit(ctx, "payment_received", map[string]string{
		"invoice": invoice.ID,
	}, Payment{
		Amount:   invoice.Total,
		Currency: invoice.Currency,
	})
	...
}
```

The agent receives it in the conversation:

```text
<app_event invoice="INV-1042" name="payment_received">{"amount":120.5,"currency":"EUR"}</app_event>
```

- `name` says what happened. It becomes the `name` attribute.
- `attrs` are short facts about the event. They become attributes. An attr called `name` is replaced by the event name.
- `data` is the body. A string goes as text. Any other value goes as JSON, and nil goes as `null`.

The agent is told that app events come from your app and that their body is data. Put facts there. Say what to do about an event in your system prompt (`neui.WithSystemPrompt`).

**It goes to one tab.** The event goes to the thread open in the tab of that instance. For an event that comes from outside the page, like a webhook, keep the `Instance` of each open page and call `Emit` on it.

**When the agent reads it.** An idle agent starts a round at once. A busy agent gets the event with its next round. If it was about to stop, it runs one more round for the event.

**It is stored.** The event stays in the thread like a message. A new thread is saved with its first event.

**Keep it small.** A body over the round item budget, `ThreadConfig.ReplayItemShare`, is cut in the middle.

**Errors.** `Emit` returns when the event is stored. It does not wait for the agent.

- `api.ErrDisabled`: neui is not enabled for the session.
- `api.ErrUnlinked`: the tab has no thread yet, shortly after the page loads, or it switched threads before the event was stored. The event is lost.
- A storage error: the event is lost and the chat gets `ChatEventCriticalError`: see [Chat](06-chat.md).

## Generate

`neui.Generate` asks the model one question and returns the answer. Use it where your code needs a model and the user is not chatting: to summarize a record, sort a ticket, read fields out of a document. The call has no thread, no page and no tools, and nothing of it is stored.

```go
type Generate struct {
	Provider api.Provider
	System   string
	Usage    *api.EventUsage
	Attempts int
}
```

`Text` returns the answer as text, trimmed:

```go
summary, err := neui.Generate{
	System: "You summarize support tickets in one sentence.",
}.Text(ctx, "Summarize this ticket:", ticket)
```

`Object` returns the answer as a value of your type:

```go
type Invoice struct {
	_      struct{} `description:"An invoice read from a document."`
	Vendor string   `json:"vendor" required:"true" description:"company that issued the invoice"`
	Total  float64  `json:"total" required:"true" description:"total in the invoice currency"`
}

invoice, err := neui.Generate{
	System: "You extract invoices.",
}.Object[Invoice](ctx, "Extract the invoice:", scan)
```

The schema comes from the type, with the same tags as a tool's input: see [Modules](03-modules.md). A field is required only with `required:"true"`. `T` can be any type, like `[]string`.

**Parts.** Everything after `ctx` goes to the model as one user message, in order. A string or a `fmt.Stringer` goes as text. API content goes as it is, like an `api.Image`. Any other value goes as JSON.

**Provider.** With `Provider` unset, the call uses the app's `neui.WithProvider`, read from `ctx`. So `ctx` must be a Doors context, from a render or a handler. A provider the user got from `neui.OptionProvider` is not used: pass it in `Provider`. Outside Doors, as in a scheduled job, always set `Provider`. With no provider the call returns `api.ErrNoProvider`.

The model and its settings are the provider's. For a cheaper model, build a second provider and pass it: see [Providers](09-providers.md).

**Object needs tool calls.** The model returns the value by calling a tool, so it must support tool calls. When the answer does not match `T`, neui sends the error back and the model tries again. After `Attempts` answers, 3 by default, the call returns an error that wraps `api.ErrInvalidOutput`.

**Errors.** A provider error ends the call. The error wraps the code, like `api.CodeBusy`, and the cause: see [Providers](09-providers.md#errors). neui does not retry it. Retry in your code when you need to. Cancel `ctx` to stop the call.

**Usage.** Point `Usage` at an `api.EventUsage` to get the tokens the call spent, retries included. Each call overwrites it, so calls that run at the same time need one each.
