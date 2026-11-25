# Configuration

You configure neui in two places. App options, the `neui.With*` functions, go to `doors.NewApp` and apply to every session:

```go
app := doors.NewApp(page,
	neui.WithProvider(provider),
	neui.WithSystemPrompt("You help the staff of a coffee roaster run their store."),
	neui.WithModules(shipping),
)
```

Session options, the `neui.Option*` functions, go to `neui.Enable` and apply to one user's session:

```go
err := neui.Enable(ctx,
	neui.OptionProvider(largerModel),
	neui.OptionSystemPrompt("The user is a store manager."),
	neui.OptionModules(adminTools),
)
```

Session options apply on top of the app options, in the order you pass them.

| What | App option | Session option | When both are set |
| --- | --- | --- | --- |
| Provider | `WithProvider` | `OptionProvider` | The session's wins. |
| Tier | `WithTier` | `OptionTier` | The session's wins. |
| System prompt | `WithSystemPrompt` | `OptionSystemPrompt` | The session's is added after the app's. |
| Modules | `WithModules` | `OptionModules` | The session's are added after the app's. |
| Thread storage | | `OptionMessageStorage` | |
| Upload storage | `WithUploadStorage` | `OptionUploadStorage` | The session's wins. |
| Files and workspace limits | `WithFilesConfig` | `OptionFilesConfig` | The session's replaces the app's whole config. |
| Thread limits | `WithThreadConfig` | | |
| Island limits | `WithIslandConfig` | | |
| JS runtime | `WithJSConfig` | | |

## Enable again

Once a session is enabled, later `neui.Enable` calls do nothing. Pass `neui.OptionForce()` to configure it again:

```go
err := neui.Enable(ctx, neui.OptionForce(), neui.OptionTier(neui.TierAgent))
```

The new configuration starts from the app options. Options of the earlier `Enable` are not kept, so pass them all again.

## Provider

`WithProvider` and `OptionProvider` take an `api.Provider`. A provider serves one model, so you switch the model by switching the provider.

When neither is set, `Enable` returns `api.ErrNoProvider`.

Each provider package has its own options for `New`: the API key, the base URL, the reasoning effort, the output token limit and so on. See the package's godoc:

```go
provider, err := deepseek.New(deepseek.Conf{
	Model: deepseek.Flash,
}, deepseek.WithAPIKey(key))
```

## Tier

`WithTier` and `OptionTier` set the tier, and the session's wins. What each tier does: see [Core concepts](01-core-concepts.md).

## System prompt

`WithSystemPrompt` and `OptionSystemPrompt` take one or more strings. The prompt is built in this order, with a blank line between the parts:

1. neui's own notes
2. the app's prompt
3. the session's prompt
4. the descriptions of the modules

## Modules

`WithModules` and `OptionModules` take module factories. A thread gets the default modules first, then the app's, then the session's. A module whose name is already taken is skipped, and an error is logged.

See [Modules](03-modules.md).

## Storage

| Option | What it stores | Default |
| --- | --- | --- |
| `OptionMessageStorage` | The session's threads. Takes an `api.MessageStorage`. | In memory, lost when the session ends. |
| `WithUploadStorage`, `OptionUploadStorage` | The files the user and the agent upload. Takes an `api.UploadStorage`. | A temporary directory of the session, removed when the session ends. |

`nekit` has ready storages: `NewSQLiteMessages` for threads, `NewDiskUploads` and `NewS3Uploads` for uploads.

The agent's workspace is always in the session's temporary directory. `FilesConfig.TempDir` sets where that directory is created.

## Files and workspace

`WithFilesConfig` and `OptionFilesConfig` take an `api.FilesConfig`:

```go
neui.WithFilesConfig(api.FilesConfig{
	MaxBytes: 256 << 20,
})
```

A zero field takes its default. A negative field turns that limit off.

`OptionFilesConfig` replaces the app's config: a field you leave zero takes neui's default.

| Field | What it limits | Default |
| --- | --- | --- |
| `TempDir` | Where each session creates its temporary directory. Read only by the session's first `Enable`. | `os.TempDir()` |
| `MaxFiles` | Files in the workspace. | 10000 |
| `MaxBytes` | Bytes in the workspace. | 1 GiB |
| `ReadLimit` | Bytes the agent gets from one file read. It continues with another read. | 50 KiB |
| `EditLimit` | Size of a file the agent may edit. | 10 MiB |
| `ImageFileLimit` | Size of an image file the agent may read. | 20 MiB |
| `DocumentLimit` | Size of a document the agent may read. | 32 MiB |
| `UnzipLimit` | Unpacked size of one archive. | 512 MiB |
| `UnzipFiles` | Entries unpacked from one archive. | 10000 |

Uploads are not counted in `MaxFiles` and `MaxBytes`.

## Thread limits

`WithThreadConfig` takes an `api.ThreadConfig`:

```go
neui.WithThreadConfig(api.ThreadConfig{
	CompactionTrigger: 0.7,
	ToolCallLimit:     16,
})
```

A zero field takes its default.

| Field | What it sets | Default |
| --- | --- | --- |
| `CompactionTrigger` | The share of the model's context a thread must fill to be compacted into a summary. Checked when a round ends. | 0.8 |
| `CompactionAllowed` | The share of the model's context a thread must fill before the user may compact it by hand. | 0.2 |
| `CompactionPreserve` | The share of the conversation kept as it is after the summary. | 0.1 |
| `ReplayItemShare` | The share of the model's context that one user input, or the text of one round's tool results, may take. At most 0.25. | 0.1 |
| `ReplayMediaShare` | The share of half of `ThreadMemoryLimit` that the images and documents of one round's tool results may take. At most 1. | 0.5 |
| `ThreadMemoryLimit` | Bytes of text, images and documents a thread sends to the model. Over it, the images and documents of the oldest rounds are replaced by notes, then the thread is compacted. Negative means no limit. | 16 MiB |
| `ToolCallLimit` | Tool calls run in one round. A call past it gets an error. Negative means no limit. | 32 |

The compaction shares need the model's context size, `api.Model.Context`. When it is zero, only `ThreadMemoryLimit` and the provider's too-long errors start compaction.

## Islands

`WithIslandConfig` takes an `api.IslandConfig`. It limits the apps the agent runs in islands:

```go
neui.WithIslandConfig(api.IslandConfig{
	TurnTime: time.Minute,
})
```

A zero or negative field takes its default.

| Field | What it limits | Default |
| --- | --- | --- |
| `MaxItems` | Slots, widgets and layouts in one island. | 512 |
| `MaxOps` | Changes a script may queue before it shows them. | 512 |
| `TurnTime` | How long a script may run between two screens. | 40s |
| `Memory` | Memory of the script's JS runtime, at least 4 MiB. | 16 MiB |
| `MaxCallDepth` | Nested JS calls. | 1000 |
| `MaxQueue` | Scripts of the agent waiting to run on one island. | 16 |
| `FnTime` | How long one `IslandCall` may run. | 5s |
| `FnMemory` | Memory of the JS runtime that `IslandCall`s run in, at least 4 MiB. | 4 MiB |

See [Islands](04-islands.md).

## JS runtime

`WithJSConfig` takes a `neui.JSConfig`. It limits the scripts of the `js` module:

```go
neui.WithJSConfig(neui.JSConfig{
	MaxWorkers: 4,
	JobTimeout: time.Minute,
})
```

A zero or negative field takes its default. The limits are shared by all sessions of the app.

| Field | What it limits | Default |
| --- | --- | --- |
| `TotalMemory` | Memory of all scripts running at once. | 512 MiB |
| `WorkerMemory` | The most memory one script may ask for. | 128 MiB |
| `DefaultMemory` | Memory of a script that asks for none, at least 4 MiB. | 16 MiB |
| `MaxWorkers` | Scripts running at once. | 16 |
| `MaxQueue` | Scripts waiting to start. One past it is rejected. | 100 |
| `JobTimeout` | How long one script may run, its tool calls included. | 30s |
| `AsyncAfter` | How long the agent waits for a script. After that the script goes on as a background task. | 10s |
| `MaxCallDepth` | Nested JS calls. | 1000 |
