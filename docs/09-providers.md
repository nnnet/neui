# Providers

A provider connects neui to one model API. neui ships five: Claude, GPT, Gemini, DeepSeek, and `generic` for any server that speaks OpenAI's Chat Completions API. For another API, implement `api.Provider`.

For all users, when you create the app:

```go
provider, err := claude.New(claude.Conf{
	Model: claude.Sonnet55,
})
...
app := doors.NewApp(page, neui.WithProvider(provider))
```

For one user, when you enable neui:

```go
err := neui.Enable(ctx, neui.OptionProvider(provider))
```

`OptionProvider` takes precedence over `WithProvider`. With neither, `Enable` returns `api.ErrNoProvider`.

A provider serves one model with one set of settings. For two models, or one model at two settings, build two providers.

Choose the tier by the model you run: see [Core concepts](01-core-concepts.md).

## Provider interface

```go
type Provider interface {
	Stream(ctx context.Context, req Request) <-chan Event
	Model() Model
}
```

`Model` returns the model the provider was built with. It returns the same value on every call.

`Stream` sends one request and returns at once. The reply arrives on the channel as events, in the order the API produced them.

- When the reply ends, send `EventOk` or `EventErr`, then close the channel.
- When `ctx` is cancelled, close the channel and send no ending.
- When the response body ends before the API said how the reply ended, close the channel and send no ending.
- When your own deadline ends the turn, send `EventErr` with `CodeNetwork`.

One provider serves every thread of the app, so `Stream` must be safe for concurrent calls.

### Model

The API does not report a model's limits, so the provider declares them:

```go
type Model struct {
	Name             string
	Context          int
	ImagePixels      int
	ImageSide        int
	ImageBytes       int
	ImagesPerRequest int
	RequestBytes     int
	Inputs           Modalities
	SimpleSchemas    bool
}
```

| Field | What it declares | Zero value |
| --- | --- | --- |
| `Name` | The model id as it goes on the wire. | |
| `Context` | The context window in tokens, input and output together. | neui does not compact by token count. |
| `ImagePixels` | The largest pixel area worth sending in one image. | neui sends the image as it is. |
| `ImageSide` | The longest image side worth sending, in pixels. | 2000. |
| `ImageBytes` | The largest image the API takes, in raw bytes before base64. | 4 MiB for an image neui resizes. No image is refused for its size. |
| `ImagesPerRequest` | How many images one request may hold. A PDF counts one per page. | Nothing is cut. |
| `RequestBytes` | The largest request body the API takes, in bytes. | The body is sent without a size check. |
| `Inputs` | What the model reads besides text: `api.ModalityImage`, `api.ModalityDocument`. | Text only. |
| `SimpleSchemas` | Sends tool schemas with references inlined, for servers that refuse anything else. | Schemas go as they are. |

neui applies the limits before it calls `Stream`. An image the model does not read reaches the provider as a text note. A document it does not read arrives as its text when it is a text type, and as a note otherwise.

The provider checks `RequestBytes` itself, on the encoded body. A body over the limit is not sent, and the turn ends with `CodeTooLarge`.

## Protocol

Every turn neui sends the whole conversation as one `Request`. The provider answers with events. neui folds the events into the blocks of one assistant message and stores it. When the message holds tool calls, neui runs them and sends the results in the next request, in a user message.

### Request

```go
type Request struct {
	Tools      []Tool
	System     string
	Messages   []Message
	ToolChoice ToolChoice
}

type Tool struct {
	Name   string
	Desc   string
	Schema any
}

type Message struct {
	Role    Role
	Content Contents
}
```

`Role` is `api.RoleUser` or `api.RoleAssistant`. The system prompt is not a message, and a tool result is a block of a user message. Place both where your API expects them.

`Schema` is the JSON Schema of the tool's input. nil means the tool takes no input.

`ToolChoice` is `api.ToolChoiceAuto`, `api.ToolChoiceNone`, `api.ToolChoiceAny`, or the name of the one tool the model must call. The agent thread leaves it empty: send no field then.

The request has no model, response cap, sampling or reasoning settings. They are the provider's own configuration.

### Content blocks

| Block | Where | Fields |
| --- | --- | --- |
| `ContentText` | User and assistant messages, tool results. | `Text`, `Meta` |
| `ContentImage` | User messages, tool results. | `Data`, `Mime`, `Meta` |
| `ContentDocument` | User messages, tool results. | `Data`, `Mime`, `Name`, `Meta` |
| `ContentToolUse` | Assistant messages. | `ID`, `Name`, `Input`, `Meta` |
| `ContentToolResult` | User messages. | `ID`, `Content`, `Error` |
| `ContentReasoning` | Assistant messages. | `Model`, `Summary`, `Thoughts`, `Meta` |
| `ContentOpaque` | Assistant messages. | `Model`, `Kind`, `Data`, `Deltas` |

A block your API cannot carry goes as a short text note in its place. Do not fail the request for it.

`ContentToolUse.Input` is the arguments as JSON, or nil when the model sent none. Send `{}` where your API needs an object.

`ContentToolResult.ID` is the `ID` of the call it answers. `Error` is `api.ToolOk`, `api.ToolErrRuntime` or `api.ToolErrJSON`. For any value but `ToolOk`, add your API's failure marker. The text of the failure is in `Content`.

`api.ToolErrJSON` answers a call whose arguments never formed valid JSON. The `Input` of that call equals `api.DiscardedArguments`: do not read it, send empty arguments. When the result of such a call is not `ToolErrJSON`, end the turn with `CodeInvalid`.

### Reasoning and opaque blocks

`ContentReasoning` holds the model's thinking. `Thoughts` is the thinking as the model wrote it. `Summary` is a paraphrase the API wrote. `Meta` is your own data, such as a signature or an encrypted payload.

`ContentOpaque` holds a payload the protocol has no type for, such as the output of a tool the API ran itself. `Kind` starts with your `Name` and a dot, like `claude.server_tool_use`.

neui stores both and sends them back unchanged. It does not remove the blocks of other providers, so the same thread can hold blocks of several. The provider filters:

- Write `Meta` as `{"provider":"<Name>","data":<your payload>}`.
- Send back a reasoning block only when its `Meta` names your provider.
- Send back an opaque block only when its `Kind` has your prefix.
- Skip every other reasoning and opaque block whole. Send nothing in its place.

`Model` on both blocks is the model that wrote them. Compare it with your model when the API takes reasoning back only from the same model.

`Meta` on `ContentText` and `ContentToolUse` of an assistant message works the same way. `Meta` on user blocks is the app's data: do not read it and do not send it.

### Events

```text
EventStart
EventThinking  index 0
EventMeta      index 0
EventClose     index 0
EventText      index 1
EventTool      index 2
EventUsage
EventOk
```

`EventStart` comes first. Then come the blocks. Each block is the events that share one `Index`, its position in the message. You count the index. Blocks may interleave. `EventOk` or `EventErr` comes last.

| Event | Block | Fields |
| --- | --- | --- |
| `EventStart` | | `ID` is the API's response id. `Model` is your model's `Name`. |
| `EventText` | Text | `Text` appends. |
| `EventTool` | Tool call | `Name` and `Args` append. `ID` replaces. |
| `EventThinking` | Reasoning | `Text` appends to the thought at `Part`. |
| `EventSummary` | Reasoning | `Text` appends to the summary at `Part`. |
| `EventMeta` | Any but opaque | `Data` replaces the block's `Meta`. |
| `EventOpaque` | Opaque | `Kind` and `Data` replace. `Delta` appends one entry to `Deltas`. |
| `EventClose` | Any | The block is complete. |
| `EventUsage` | | The token counts of the turn so far. |
| `EventOk` | | The reply is complete. `Code` is `api.CodeTurn`. |
| `EventErr` | | The reply is not usable. `Code` says why, `Err` is the failure. |

- The first event at an index sets the block's kind. `EventMeta` cannot open a block: open a reasoning block that has only `Meta` with an empty `EventThinking`.
- Send each event as the API delivers it. Do not buffer, and do not forward keepalives.
- Send `EventClose` only where the API marks the end of that block. Without it, the block ends with the turn.
- A tool call needs an `ID`. When the API sends none, make one from the response id and the block index.
- `Part` is the API's own counter of thoughts inside one reasoning block. Send 0 when it has none.
- `EventUsage` holds running totals. The counters do not overlap: the prompt is `Input + CacheRead + CacheWrite`, the model wrote `Output + Reasoning`.

neui checks the order. An event after the ending, before the start, or of the wrong kind for its block ends the turn with `CodeUnknown`. So does a channel that closes with no ending.

### Errors

`EventErr` ends the turn with no stored message. `Err` is always set.

| Code | When to send it | What neui does |
| --- | --- | --- |
| `CodeRefused` | The model or the API would not answer. | Ends the round as refused. |
| `CodeCapped` | The output hit the response cap. | Ends the round with an error. |
| `CodeContextLength` | The conversation does not fit the model. | Compacts the thread and sends the round again. |
| `CodeTooLarge` | The request is over `Model.RequestBytes`, or the API refused it for its size. | Compacts the thread and sends the round again. |
| `CodeNetwork` | The connection, the stream or a deadline failed. | Ends the round with an error and logs it. |
| `CodeBusy` | Rate limit, or no capacity. | Ends the round with an error and logs it. |
| `CodeInvalid` | The API refused the request as written, or you could not encode it. | Ends the round with an error and logs it. |
| `CodeAccess` | Credentials, permission or balance. | Ends the round with an error and logs it. |
| `CodeUnknown` | Nothing you recognize. | Ends the round with an error and logs it. |

When compaction fails, the round ends with an error. neui does not retry a failed round. The user does.

A stop reason you do not recognize is `CodeUnknown`, never `EventOk`. neui sends the round again, up to two times, when `EventOk` ends a reply with no content.

## Machinery

### provider/lib

`neui.dev/provider/lib` holds what the shipped providers share. With it, a provider is an encoder of the request and a converter of the reply:

```go
func (p *Provider) Stream(ctx context.Context, req api.Request) <-chan api.Event {
	body, err := lib.MarshalBody(p.encode(req))
	...
	httpReq, err := body.Request(ctx, p.url)
	...
	return lib.Stream(ctx, p.client, httpReq, &converter{...}, p.transport)
}
```

```go
type Converter interface {
	Convert(Frame) ([]api.Event, bool)
	End(err error) []api.Event
}
```

`lib.Stream` sends the request, reads the server-sent events and delivers your events. It keeps the idle deadline, handles cancel and closes the channel.

`Convert` gets one server-sent event and returns the events it holds. Return false after the ending: nothing more is read. `Frame.Data` is reused after `Convert` returns, so keep only copies.

`End` gets every other ending: a non-200 response as `*lib.HTTPError`, `lib.ErrIdle`, `io.EOF`, `io.ErrUnexpectedEOF`, or the `ctx` error. Return the `EventErr` for it, or nothing to close the channel with no ending.

Both methods run on one goroutine, so the converter keeps its state without a lock.

| Name | What it gives you |
| --- | --- |
| `Stream`, `Converter`, `Frame` | The HTTP exchange and the event channel. |
| `Options` | The idle deadline (`Idle`, default 2 minutes) and the largest frame (`Max`, default 64 MiB). Take it in a `WithTransport` option and pass it to `Stream`. |
| `Drain` | A closed channel with the events you pass, for a request that fails before it is sent. |
| `MarshalBody`, `Body` | The request body as JSON without HTML escaping. `Bytes` gives its size for the `RequestBytes` check, `Request` builds the POST. |
| `MarshalRaw` | `json.Marshal` without HTML escaping. |
| `HTTPError` | The status, headers and body of a non-200 response. |
| `APIError` | The error to put in `EventErr.Err` for a failure the API reported. Apps read it with `errors.As`. |
| `ErrorMessage`, `RetryAfter` | Fill `APIError.Message` from a body that is not JSON, and `APIError.RetryAfter` from the header. |
| `MeasuredTooLarge` | Wraps the error of a 413 with the bytes sent and the declared limit. |
| `Meta`, `OwnMeta` | Tag the opaque data your provider returns with its name, and read back only data with your tag. Data from another provider is skipped. |
| `CallID`, `CallName` | Make a stored call id or tool name fit an API that takes only 1 to 64 letters, digits, `_` and `-`. |
| `ObjectInput` | Stored arguments as a JSON object, for an API that takes only objects. |
| `Omitted`, `Unknown` | The note for a block the API cannot carry, and for a block type you have no case for. |
| `EmptyResult` | The text for a tool result with no content. |
| `ImageMime` | Normalizes an image type and reports whether every provider takes it. |

### Tests

The test kits are in `neui.dev/internal/testkit`. Go lets only code inside the neui module import them, so they serve a provider you add to the neui repository.

`conform.Run` is the conformance suite. It runs your provider against a local server:

```go
func TestConform(t *testing.T) {
	conform.Run(t, conform.Adapter{
		Name:   providerName,
		Models: []api.Model{...},
		New:    func(base string, m api.Model, p conform.Params) (api.Provider, error) {...},
		Spec:   conform.Spec{...},
		Render: func(m api.Model, s conform.Script) (string, bool) {...},
		Errors: []conform.ErrorCase{...},
		Rules:  []conform.Rule{...},
		...
	})
}
```

`New` builds your provider with the given settings. `Spec` names the API's OpenAPI file and the schemas of its request, stream event and error. `Render` writes a scripted reply in your API's stream format. `Errors` lists error responses and the code each must map to. `Rules` lists what each setting puts in the body, and which settings are refused.

| Check | What it verifies |
| --- | --- |
| `Spec` | The provider sends the API version the spec file describes. |
| `Sources` | Every rule and error case names a docs URL. |
| `Limits` | The declared models match `testdata/limits/<name>.json` of the `conform` package. |
| `Encode` | Every combination of settings gives a body that fits the API's schema, or is refused where a rule says so. |
| `Corpus` | A fixed set of conversations encodes to valid bodies. The bodies are stored as golden files. |
| `RequestBytes` | A body at the limit is sent. A body one byte over ends with `CodeTooLarge` and is not sent. A 413 ends with `CodeTooLarge`. |
| `Inputs` | Images and PDFs the model does not read go as notes. |
| `Decode` | Scripted replies give the right blocks, a valid event order, the usage, and the right ending. |
| `RoundTrip` | Reasoning and opaque blocks go back byte for byte in the next request. |
| `Errors` | Every error case ends with its code. |
| `Streams` | Streams recorded from the real API, in `testdata/streams`, decode with a valid event order and an ending. |
| `Cross` | Threads written by the other providers and by another model encode to a valid body. |

`conform.RunLive` and `conform.RunLiveCross` run against the real API. They need the `live` build tag and the API key in the environment. `go test -tags live -run Live -record` saves the streams for the `Streams` check.

Run with `-update` to write the golden files, then review the diff.

`conform.CheckGrammar` checks the event order of one stream. `response.NewBuilder` in `neui.dev/internal/response` is the fold the thread uses: `On` takes each event and `Flush` returns the blocks.

`testkit.NewProvider` is a scripted provider for tests of everything above the provider. Each turn is a list of events:

```go
provider := testkit.NewProvider(api.Model{}, testkit.Say("hi"), testkit.Call("shipping__quote", args))
```

## Shipped providers

Each package has the same shape: `New(Conf, ...Option)`.

```go
provider, err := gpt.New(gpt.Conf{
	Model: gpt.GPT56Luna,
}, gpt.WithEffort(gpt.EffortHigh))
```

`Conf.Model` takes one of the package's models, or your own `api.Model` for a model it does not list. For a listed model `New` returns an error for settings the API refuses. For your own model it checks nothing, and a refused setting fails each turn with `CodeInvalid`.

The key is not checked at `New`. A missing or wrong key ends the turn with `CodeAccess`.

Options every provider has:

| Option | What it does |
| --- | --- |
| `WithAPIKey(key)` | Sets the key in place of the environment variable. An empty key sends no auth header. |
| `WithBaseURL(url)` | Points the provider at a gateway or proxy that speaks the same API. `generic` takes `Conf.BaseURL` instead. |
| `WithHTTPClient(client)` | Sets the HTTP client. Leave its `Timeout` at zero: it cuts long turns. |
| `WithTransport(lib.Options)` | Sets the idle deadline and the largest frame. |
| `WithMaxTokens(n)` | Caps the response. A turn that hits it ends with `CodeCapped`. |

### Claude

`neui.dev/provider/claude` speaks Anthropic's Messages API. The key comes from `ANTHROPIC_API_KEY`. Amazon Bedrock and Google Cloud Vertex AI are not supported.

| Model | Name |
| --- | --- |
| `claude.Fable52` | `claude-fable-5-2` |
| `claude.Fable51` | `claude-fable-5-1` |
| `claude.Opus55` | `claude-opus-5-5` |
| `claude.Sonnet55` | `claude-sonnet-5-5` |
| `claude.Haiku55` | `claude-haiku-5-5` |
| `claude.Sonnet5` | `claude-sonnet-5` |

Each has a 1,000,000-token context, reads images and documents, and returns at most 128,000 tokens.

Caching is on. Each request marks the end of the system prompt and the end of the last message. Thinking is always adaptive, so a model without adaptive thinking does not work.

| Option | What it does |
| --- | --- |
| `WithMaxTokens(n)` | Default 16384. It caps thinking too. |
| `WithEffort(effort)` | How much the model thinks: `EffortLow`, `EffortMedium`, `EffortHigh`, `EffortXHigh`, `EffortMax`. Unset leaves the model's default. |
| `WithThinkingSummaries(on)` | Streams summaries of the thinking. Default on. |
| `WithCacheTTL(ttl)` | How long a cache entry lives: `CacheTTL5M` (default) or `CacheTTL1H`. |
| `WithoutCache()` | Sends no cache markers, for a gateway that refuses them. |
| `WithStopSequences(seqs...)` | Ends the turn at one of the sequences. |
| `WithVersion(v)` | The `anthropic-version` header. Default `2026-10-21`. |
| `WithBeta(flags...)` | Adds `anthropic-beta` flags. |

### GPT

`neui.dev/provider/gpt` speaks OpenAI's Responses API. The key comes from `OPENAI_API_KEY`.

| Model | Name |
| --- | --- |
| `gpt.GPT62Astra` | `gpt-6.2-astra` |
| `gpt.GPT62Sol` | `gpt-6.2-sol` |
| `gpt.GPT61Terra` | `gpt-6.1-terra` |
| `gpt.GPT61Sol` | `gpt-6.1-sol` |
| `gpt.GPT6Astra` | `gpt-6-astra` |
| `gpt.GPT6Sol` | `gpt-6-sol` |
| `gpt.GPT6Luna` | `gpt-6-luna` |
| `gpt.GPT56Sol` | `gpt-5.6-sol` |
| `gpt.GPT56Luna` | `gpt-5.6-luna` |

Each has a 1,050,000-token context, reads images and documents, and returns at most 128,000 tokens.

Caching is automatic. Responses are not stored on the service. Reasoning summaries are off.

| Option | What it does |
| --- | --- |
| `WithMaxTokens(n)` | Default is no cap. The smallest value is 16. |
| `WithEffort(effort)` | `EffortNone`, `EffortMinimal`, `EffortLow`, `EffortMedium`, `EffortHigh`, `EffortXHigh`, `EffortMax`. No listed model takes `EffortMinimal`. `gpt-6-astra`, `gpt-6.1-sol`, `gpt-6.2-astra` and `gpt-6.2-sol` do not take `EffortNone`. |
| `WithReasoningSummary(summary)` | Streams reasoning summaries: `SummaryAuto`, `SummaryConcise`, `SummaryDetailed`. |
| `WithReasoningMode(mode)` | `ReasoningStandard` or `ReasoningPro`. |
| `WithReasoningContext(context)` | Which earlier reasoning the model uses: `ContextAuto`, `ContextCurrentTurn`, `ContextAllTurns`. |
| `WithReasoningReplay(mode)` | `ReplayAll` (default) sends earlier reasoning back. `ReplayNever` sends none, for a gateway that refuses it. |
| `WithTemperature(v)`, `WithTopP(v)` | Sampling. The listed models take them only with `EffortNone`. |
| `WithVerbosity(verbosity)` | How long answers tend to be: `VerbosityLow`, `VerbosityMedium`, `VerbosityHigh`. |
| `WithParallelToolCalls(ok)` | Allows or forbids several tool calls in one turn. |
| `WithStrictTools(strict)` | Sets the strict flag on every tool. |
| `WithPromptCacheKey(key)` | Routes requests with the same prefix to the same cache. |
| `WithPromptCacheOptions(mode, ttl)` | `CacheImplicit` or `CacheExplicit`, `CacheTTL30M` or `CacheTTL12H`. Use `CacheImplicit`. Models before `gpt-5.6` refuse the option. |
| `WithServiceTier(tier)` | `TierAuto`, `TierDefault`, `TierFlex`, `TierScale`, `TierPriority`, `TierFast`, `TierUltrafast`. |
| `WithStore(store)` | Keeps responses on the service. Default off. |
| `WithInclude(values...)` | Replaces the `include` list. Keep `reasoning.encrypted_content` in it: without it and with store off, reasoning is not sent back. |
| `WithOrganization(id)`, `WithProject(id)` | The `OpenAI-Organization` and `OpenAI-Project` headers. |
| `WithMetadata(m)` | Sends the pairs as `metadata` on every turn. |
| `WithObfuscation(on)` | Adds the service's random padding to stream events. Default off. |

### Gemini

`neui.dev/provider/gemini` speaks Google's Interactions API. The key comes from `GEMINI_API_KEY`. Vertex AI is not supported.

| Model | Name | Thinking levels |
| --- | --- | --- |
| `gemini.Hydrogen4` | `gemini-4-hydrogen` | low, medium, high (default), xhigh |
| `gemini.Neon4` | `gemini-4-neon` | low, medium, high (default), xhigh |
| `gemini.Carbon4` | `gemini-4-carbon` | low, medium (default), high |
| `gemini.Argon4` | `gemini-4-argon` | low, medium (default), high |
| `gemini.Flash38` | `gemini-3.8-flash` | low, medium (default), high |
| `gemini.FlashLite35` | `gemini-3.5-flash-lite` | minimal (default), low, medium, high |

The Gemini 4 models have a 1,048,576-token context. The two Flash models take 1,048,576 input tokens and 65,536 output tokens on top. All read images and documents and return at most 65,536 tokens.

Caching is automatic. Nothing is stored on the service. Thinking cannot be turned off.

| Option | What it does |
| --- | --- |
| `WithMaxTokens(n)` | Default is no cap. |
| `WithEffort(effort)` | The thinking level: `EffortMinimal`, `EffortLow`, `EffortMedium`, `EffortHigh`, `EffortXHigh`. |
| `WithThinkingSummaries(on)` | Streams summaries of the thinking. Default on. |
| `WithStopSequences(seqs...)` | Ends the turn at one of the sequences. |
| `WithRevision(v)` | The `Api-Revision` header. Default `2026-05-20`. |

### DeepSeek

`neui.dev/provider/deepseek` speaks DeepSeek's Chat Completions API. The key comes from `DEEPSEEK_API_KEY`.

| Model | Name | Reads |
| --- | --- | --- |
| `deepseek.Flash` | `deepseek-flash` | Text and images. |
| `deepseek.V4Pro` | `deepseek-v4-pro` | Text. |

Each has a 1,048,576-token context and returns at most 393,216 tokens.

Caching is automatic. Thinking is on and streams as readable reasoning.

| Option | What it does |
| --- | --- |
| `WithMaxTokens(n)` | Default is no cap. |
| `WithEffort(effort)` | `EffortNone` turns thinking off. `EffortLow`, `EffortHigh` and `EffortMax` set its depth. Unset, the server thinks at high. |
| `WithTemperature(t)` | Sampling. Needs `EffortNone`. |
| `WithTopP(t)` | Sampling. `deepseek-flash` takes it with thinking on, `deepseek-v4-pro` with `EffortNone`. |
| `WithStopSequences(seqs...)` | Ends the turn at one of the sequences. |
| `WithUserID(id)` | Splits the cache and the concurrency allowance by user. Requests with different ids share no cache. |

### Generic

`neui.dev/provider/generic` speaks OpenAI's Chat Completions API to any server: llama.cpp, vLLM, SGLang, Ollama, LM Studio, OpenRouter.

```go
provider, err := generic.New(generic.Conf{
	Model: api.Model{
		Name:    "qwen3.8-27b",
		Context: 131_072,
	},
	BaseURL: "http://127.0.0.1:8080/v1",
}, generic.WithEffort(generic.EffortLow))
```

The package lists no models. Declare the model yourself: nothing is read from the server. Add `api.ModalityImage` to `Inputs` for a vision model, or images go as notes.

No key is read from the environment. Set one with `WithAPIKey`.

Reasoning goes back only to the model with exactly the same `Name`.

| Option | What it does |
| --- | --- |
| `WithMaxTokens(n)` | Default is no cap. |
| `WithEffort(effort)` | Sends `reasoning_effort`: `EffortNone`, `EffortMinimal`, `EffortLow`, `EffortMedium`, `EffortHigh`, `EffortXHigh`, `EffortMax`. What a level does is the server's choice. |
| `WithTemperature(t)`, `WithTopP(t)` | Sampling. |
| `WithStopSequences(seqs...)` | Ends the turn at one of the sequences. |
| `WithExtraBody(fields)` | Adds fields to the top level of every request body, such as OpenRouter's `provider` or vLLM's `chat_template_kwargs`. |
| `WithCacheMarker(fields)` | Adds the fields to the system prompt and to the last text of the conversation, for servers that cache only at markers. |

Most servers cache on their own. `WithCacheMarker` is for the ones that do not, such as Claude models on OpenRouter:

```go
generic.WithCacheMarker(map[string]any{
	"cache_control": map[string]any{
		"type": "ephemeral",
	},
})
```

Servers that refuse unknown fields refuse every request with a marker.

A server that loads the model on the first request sends nothing while it loads. Set `lib.Options.Idle` above the load time.
