# Core concepts

Neui adds a companion agent to a Doors web page. The agent works inside the page the user has open. It sees and uses only what you expose there, and calls the tools you give it. It also has a workspace for files, runs scripts that call your tools, and builds small apps for the user in the islands you place on the page.

## Turn the agent on

The agent is on or off for a whole browser session: every tab the user has open, and every tab they open later. `neui.Enable` turns it on and `neui.Disable` turns it off.

The usual place is your sign-in and sign-out handlers:

```go
func login(ctx context.Context, r doors.RequestForm[LoginData]) bool {
	...
	err := neui.Enable(ctx, neui.OptionMessageStorage(threads.User(userID)))
	...
}

func logout(ctx context.Context, r doors.RequestPointer) bool {
	...
	neui.Disable(ctx)
	...
}
```

Or the layout, where it binds the auth state:

```gox
elem (a App) Main() {
	~(a.auth.Bind(elem(s Session) {
		~~
		if s.Authorized {
			err := neui.Enable(ctx, neui.OptionMessageStorage(a.Threads.User(s.UserID)))
			...
		} else {
			neui.Disable(ctx)
		}
		~~
		...
	}))
}
```

Repeated calls do nothing: `Enable` on an enabled session, `Disable` on a disabled one. So the layout can call them on every render and in every tab. The layout also covers a user who arrives already signed in and never passes your sign-in handler.

The options you pass to `Enable` set the agent up for this user: see [Configuration](07-configuration.md).

## What you give the agent

- **A model.** A provider connects neui to a model API. You set one for the app or per user. See [Providers](09-providers.md).
- **A view of your page.** The agent doesn't see your HTML. You mark regions of the page as spaces and put into them the data, the inputs and the tools the agent may use. A tool is a Go function with typed input and output. A tool rendered on the page exists only where and for whom you rendered it, like a button. See [Spaces](02-spaces.md).
- **Tools that don't depend on the page.** They come in modules. The agent has them on every page. See [Modules](03-modules.md).
- **Places for its apps.** An island is a region of the page where the agent builds an app from the components you registered. See [Islands](04-islands.md).
- **A chat.** The user talks to the agent there. Use the included one or build your own. See [Chat](06-chat.md).
- **Storage.** Conversations and uploaded files are lost with the session unless you give neui a place to keep them. See [Storage](08-storage.md).

## One thread per tab

A thread is one conversation of the user with the agent. Each tab has one thread open, and the agent of that thread works with that tab's page. The user can start a new thread or go back to an earlier one.

Your module functions run once for each thread that opens, so every thread gets its own modules.

The agent answers a message in rounds. A round is one request to the model, its answer and the results of the tools it called. After a round with tool calls comes the next round, until the agent stops.

## What the agent has of its own

**A workspace.** Each thread has a directory on your server where the agent keeps the files it works on, its results and its scripts. Your tools can read and write there too. It is temporary and is removed when the thread is closed.

**Scripts.** The agent writes JavaScript and neui runs it on your server in a sandbox. A script calls the tools the agent has, and reaches files, the web and your data only through them. The agent uses scripts to do in one run what would take many tool calls, such as the same change to many records. An island app is a script too, limited to what you put into the island's library.

## Choose the tier by your model

The tier sets how much of this the agent does. Set it to what your model can handle:

| Tier | What the agent does | The model needs |
|---|---|---|
| `TierAssistant` | Works in your UI like a user and calls your tools, one call at a time. Writes no code. | Reliable tool calling. A small model is enough. At least 32k context, 64k recommended. |
| `TierAgent` | Also writes scripts. | To plan work of many steps and write code against a tool's documented output without seeing it first. At least 100k context. |
| `TierDeveloper` | Also builds apps in islands. The default. | To write an interactive program: lay out a screen and react to what the user does in it. 200k context recommended. |

You set the tier for the app or per user: see [Configuration](07-configuration.md).
