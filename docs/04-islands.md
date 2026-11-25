# Islands

Islands are special spaces for generative apps. You declare the building blocks, and the agent writes and runs a sandboxed app for the user's request.

- **The model is called only for decisions.** The app handles user actions itself and calls the agent only when a decision is needed. The UI stays responsive, no tokens are spent on routine steps, and the agent is still in the loop.
- **Data stays out of the context.** The agent works with data through the app's script, without loading it into its context. Data costs no tokens, and text inside it can't steer the agent.
- **App changes on the fly.** The user can ask for an extra column, another step or a different check, and the agent updates the running app without losing their work.
- **Runs with the user's permissions.** The app can do only what the signed-in user can, through the components and functions you declared, with no endpoints to build.
- **A rich widget library out of the box.** `nekit` ships widgets for most business flows, from data entry to review and reporting: see [The included library](#the-included-library).

## Building blocks

There are three types of building blocks. You register each in the island's library under a name and a description:

```go
func library(l neui.Library) error {
	l.Layout("card", "a titled card around its items", card)
	l.View("receipt", "a printable receipt of an order", receipt)
	l.Control("rating", "asks the user for a rating from 1 to 5", rating)
	l.Add(neui.Tool[StockInput, Product]{
		Name:    "set_stock",
		Desc:    "Set the stock of a product.",
		Handler: setStock,
	})
	return nil
}
```

### Views

A view shows information. The script creates it with params and changes it later with a patch.

A view is a function that takes the params and returns an `api.Widget`:

```go
type Widget[P any] interface {
	Update(ctx context.Context, patch P)
	gox.Comp
}
```

```go
func receipt(ctx context.Context, p ReceiptParams) (api.Widget[ReceiptPatch], error) {
	...
}
```

Register it in the island's library:

```go
l.View("receipt", "a printable receipt of an order", receipt)
```

### Controls

A control collects input from the user and hands it to the script. It is a view whose function also gets `submit`. What you pass to `submit` is what the script gets as the user's action:

```go
func rating(ctx context.Context, p RatingParams, submit func(ctx context.Context, stars int) bool) (api.Widget[struct{}], error) {
	...
}
```

Register it in the island's library:

```go
l.Control("rating", "asks the user for a rating from 1 to 5", rating)
```

### Layouts

A layout arranges other blocks. It is a function that takes the params and the rendered children:

```go
func card(p CardParams, children []gox.Comp) gox.Comp {
	...
}
```

Register it in the island's library:

```go
l.Layout("card", "a titled card around its items", card)
```

### Tools

Tools wire in your backend functions: the APIs the app can call. Register them in the island's library: `l.Add` takes the same `neui.Tool` struct you render on the page, `l.Tool` is the short form:

```go
l.Add(neui.Tool[StockInput, Product]{
	Name:    "set_stock",
	Desc:    "Set the stock of a product.",
	Handler: setStock,
})
l.Tool("top_customers", "customers ranked by what they paid", topCustomers)
```

The script calls them as `app.set_stock({...})`. The agent can call them too, as tools of the island's `app` space.

> An island has no access to the page's dynamic tools, so provide everything the app needs with `l.Tool` or `l.Add`.

Blocks and tools get their schemas from your Go types, the same way as module tools, so add schema tags.

## Render an island

```go
type Island struct {
	Name        string
	Desc        string
	Placeholder gox.Elem
	Template    func(slot gox.Comp) gox.Elem
	Success     func(close func(ctx context.Context)) gox.Elem
	Failure     func(close func(ctx context.Context), err error) gox.Elem
	Library     func(l Library) error
}
```

When you render an island, you register its blocks and tools in `Library`.

```gox
~neui.Island{
	Name:        "report",
	Desc:        "Reports on the shop's sales: its apps read orders, customers and products, rank customers by what they paid, chart sales and set product stock.",
	Placeholder: <p>Ask the assistant for a report.</p>,
	Template: func(slot gox.Comp) gox.Elem {
		return <div class="card">~(slot)</div>
	},
	Success: reportDone,
	Failure: reportFailed,
	Library: library,
}
```

`Template` wraps the running app: `slot` is where the app goes. It is required, like `Name`, `Desc` and `Library`.

The rest is optional:

- `Placeholder` shows while no app runs. The agent builds apps only at `TierDeveloper`. At the other tiers the island is a regular space with the placeholder as its content.
- `Success` replaces the app when it ends without an error.
- `Failure` replaces the app when it ends with an error.

`Success` and `Failure` get a `close` function as an argument: calling it brings the placeholder back. Without them the placeholder shows right after the app ends.

## Two approaches

### A scoped island

A scoped island has one place and one job, like a custom dashboard. The static dashboard is the placeholder, and a generated one replaces it:

```gox
~neui.Island{
	Name:        "dashboard",
	Desc:        "Dashboard. Shows the static dashboard by default: this month's numbers, recent orders and products running low. Build a dashboard here when the user wants other numbers or charts than the standard ones. Its apps read orders, customers and products.",
	Placeholder: dashboardBody(...),
	Template: func(slot gox.Comp) gox.Elem {
		return <div class="dashboard">~(slot)</div>
	},
	Library: dashboardLibrary,
}
```

Its library is small: layouts, stat tiles, charts and the tools that read the data.

> Below `TierDeveloper` the agent sees the island as a regular space: its name, its description and the placeholder with its spaces, inputs and data. It just can't build an app there.

### A workbench

A workbench is usually declared in the root scope, in your app shell outside any space, so it is on every page. It has a wide API and the complete set of components:

```gox
~neui.Island{
	Name:        "workbench",
	Desc:        "Apps over the shop's data, in a panel over the page. Build one here when the user wants something the pages don't offer: a chart, visualization, report or dashboard, reviewing and changing many records, comparisons or calculators. ...",
	Placeholder: <></>,
	Template: func(slot gox.Comp) gox.Elem {
		return workbenchFrame(slot)
	},
	Library: workbenchLibrary,
}
```

```go
func workbenchLibrary(l neui.Library) error {
	nekit.IslandRegisterAll(l)
	for _, t := range operations {
		l.Add(t)
	}
	return nil
}
```

With an empty placeholder, the panel is there only while an app runs.

## The included library

Out of the box neui ships a rich library in `nekit`:

- Layouts: `column`, `row` and `grid`.
- `markdown`, a view for any text: titles, messages, reports, cards.
- `chart`, a view: line, area, bar, pie, funnel and scatter charts.
- `stat`, a view: one key number with its change against a reference.
- `form`, a control: the user fills it and submits.
- `table`, a control: rows with search, sorting, pages, selection and row actions.
- `button`, a control.

You can register them all:

```go
nekit.IslandRegisterAll(l)
```

Or some:

```go
nekit.IslandRegisterLayout(l)
nekit.IslandRegisterStat(l)
nekit.IslandRegisterChart(l)
```

`IslandRegisterLayout` adds the three layouts and `IslandRegisterWidgets` all views and controls. Each view and control also has its own function: `IslandRegisterMarkdown`, `IslandRegisterChart`, `IslandRegisterStat`, `IslandRegisterForm`, `IslandRegisterTable` and `IslandRegisterButton`.

### Styling

The widgets take the font and text colour of the island. Borders and neutral fills are tints of the text colour. Like the chat, they follow the CSS `color-scheme` of your page.

Set CSS variables on the island's template, or on any element around it, to change the rest:

```css
.workbench {
	--neisland-primary: #7c3aed;
	--neisland-primary-hover: #6d28d9;
	--neisland-primary-active: #5b21b6;
	--neisland-radius: 0.5rem;
	--neisland-radius-lg: 0.75rem;
}
```

| Variable | What it sets | Default |
| --- | --- | --- |
| `--neisland-primary`, `-hover`, `-active` | Primary buttons, checked boxes, links and accents. | `#2563eb`, `#1d4ed8`, `#1e40af` |
| `--neisland-on-primary` | Text on primary buttons. | `#fff` |
| `--neisland-danger`, `-hover`, `-active` | Destructive buttons and errors. | `#dc2626`, `#b91c1c`, `#991b1b` |
| `--neisland-on-danger` | Text on destructive buttons. | `#fff` |
| `--neisland-success`, `-hover`, `-active` | Success buttons and good changes. | `#15803d`, `#166534`, `#14532d` |
| `--neisland-on-success` | Text on success buttons. | `#fff` |
| `--neisland-info`, `--neisland-warning` | Info and warning cards. | `#0369a1`, `#b45309` |
| `--neisland-focus` | The focus outline. | `#2563eb`, `#60a5fa` in dark |
| `--neisland-bg` | The island's background, under sticky table headers and tinted fields. Set it when the island is not on the page background. | `Canvas` |
| `--neisland-radius` | Corners of buttons and fields. | `0.25rem` |
| `--neisland-radius-lg` | Corners of cards, tables and charts. | `0.375rem` |
| `--neisland-gap-sm`, `-md`, `-lg` | Gaps of the layouts. | `0.375rem`, `0.75rem`, `1.25rem` |
| `--neisland-item-sm`, `-md`, `-lg` | The smallest width of a grid cell. | `9rem`, `12rem`, `18rem` |
| `--neisland-chart-1` to `-8` | Chart series colours. | A light and a dark palette. |
| `--neisland-chart-height` | The height of a chart. | `18rem` |
| `--neisland-table-max-h` | The height of a table before it scrolls. | `28rem` |

With Tailwind on the page, the colour defaults come from its palette: `--color-blue-600` for primary, `--color-red-600` for danger, and so on.

## Develop your own widgets

### Layouts

A layout is one function. It renders the children in the order the script passed them:

```gox
type CardParams struct {
	Title string `json:"title" required:"true" description:"title shown above the items"`
}

func card(p CardParams, children []gox.Comp) gox.Comp {
	return <section class="card">
		<h3>~(p.Title)</h3>
		~(children)
	</section>
}
```

### Views

A view keeps its state and renders from it. `Update` gets a patch that already passed the schema, and changes the state:

```gox
func receipt(ctx context.Context, p ReceiptParams) (api.Widget[ReceiptPatch], error) {
	return receiptView{
		order: doors.NewSource(p.Order),
	}, nil
}

func (r receiptView) Update(ctx context.Context, patch ReceiptPatch) {
	r.order.Update(ctx, patch.Order)
}

elem (r receiptView) Main() {
	~(r.order.Bind(elem(id int) {
		...
	}))
}
```

An error from `receipt` is thrown in the script, by the call that creates the view.

### Controls

A control keeps `submit` and calls it from an event handler, with that handler's `ctx`:

```gox
func rating(ctx context.Context, p RatingParams, submit func(ctx context.Context, stars int) bool) (api.Widget[struct{}], error) {
	return ratingControl{
		question: p.Question,
		submit:   submit,
	}, nil
}

func (r ratingControl) Update(ctx context.Context, patch struct{}) {}

elem (r ratingControl) Main() {
	...
	~>doors.AClick{
		Scope: neui.IslandScope(ctx),
		On: func(ctx context.Context, _ doors.RequestPointer) bool {
			r.submit(ctx, stars)
			return false
		},
	} <button type="button">~(stars)</button>
	...
}
```

Submitting is not always available: `submit` returns false when the submit was not applied, so the control can keep the user's input.

> Put `neui.IslandScope(ctx)` on the handlers that submit. It is one blocking scope for all widgets of the app: a new submit is rejected while an earlier one is still handled. Leave it off the handlers that only change the widget itself, like the sorting of a table.

### What a widget can use

| Function | Gives the widget |
| --- | --- |
| `neui.IslandStore(ctx)` | A store shared by all widgets of the app, fresh for each run. |
| `neui.IslandFS(ctx)` | The thread's workspace: the files the agent works with. |
| `neui.IslandDir(ctx)` | The app's own directory in the workspace, emptied when the app starts. |
| `neui.IslandID(ctx)` | The island's id. `[data-neui-island="<id>"]` selects inside it when a page has several islands. |
| `neui.IslandReport(ctx, err)` | Reports a runtime error of the widget to the agent. The widget keeps running. |

Call them with a `ctx` of the widget: its render, `Update` or event handlers. Outside an island they return nothing.

### Widgets for the agent

It is recommended to make widgets usable and viewable by the agent, with `neui.Tool`, `neui.Data` and `neui.Use` inside the widget. Each widget gets its own space automatically, so you don't add one:

```gox
elem (r receiptView) Main() {
	~(r.order.Bind(elem(id int) {
		~neui.Data{
			Text:   "the order on the receipt",
			Object: loadOrder(id),
		}
		<article class="receipt">...</article>
	}))
}
```

```gox
~>neui.Use{
	Desc: fmt.Sprint("rate ", stars, " of 5"),
} ~>doors.AClick{
	...
} <button type="button">~(stars)</button>
```

What the agent gets in the page tree while the app runs:

```json
{
  "name": "report",
  ...,
  "island": true,
  "children": [
    { "name": "app", ..., "tools": ["set_stock"] },
    {
      "name": "screen",
      ...,
      "children": [
        { "name": "rating_2", "desc": "", "inputs": 5 },
        { "name": "receipt_1", "desc": "", "data": 1 }
      ]
    }
  ]
}
```

The included widgets already do this.
