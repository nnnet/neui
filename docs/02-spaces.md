# Spaces

With spaces you design how the agent sees your app and works with it. To the agent a space is a box with a label: it reads the label and decides whether the task needs what is inside.

A catalog page with a sidebar of categories:

```gox
~>neui.Space{
	Name: "aside",
	Desc: "Browse products and categories",
} <aside>
	~(for _, c := range categories {
		~>neui.Use{
			Desc: "show " + c.Name,
		} <a (doors.ALink{...})>~(c.Name)</a>
	})
</aside>
```

What the agent gets when it asks for the page tree: each space's name, description and counts.

```json
{
  "children": [
    { "name": "aside", "desc": "Browse products and categories", "inputs": 3 },
    { "name": "products", "desc": "Products of the chosen category", "data": 1 }
  ]
}
```

When the user asks for grinders, the description tells the agent to look for the link in `aside`:

> User: Show me the grinders.
>
> Agent: *reads `aside`, clicks "show Grinders"*

The agent reads a space before it acts in it, and the read gives it all of the space: tools with schemas, inputs, data, and child spaces by name and description.

Each action answers with what changed: `diff` holds the new content of the space the agent acted in, `changes` only names the other spaces that changed. The click on "show Grinders" switches the sidebar to the grinder subcategories and changes the product list. What the agent gets back:

```json
{
  "handled": true,
  "diff": {
    "inputs": {
      "added": [
        { "type": "link", "desc": "show Hand grinders", ... },
        { "type": "link", "desc": "show Electric grinders", ... }
      ],
      "removed": [...]
    }
  },
  "changes": [{ "path": "products", "changed": ["data"] }]
}
```

## Divide the page into spaces

Your goal is the best agent experience.

Put everything into one big space, and the agent reads the whole page for every task. Any change on the page makes it read the whole page again. That is expensive and confusing.

Divide the page into pieces that are too small, and the agent has to read several spaces to perform one action.

Some good spaces from the catalog:

### A form

A form, its fields and its submit button are one space. Render the validation result into the same space, as data written for the agent:

```gox
~>neui.Space{
	Name: "product_form",
	Desc: "Form adding a product: name, category, price and units in stock",
} <form (doors.ASubmit[ProductForm]{...})>
	~(if len(errs) > 0 {
		~neui.Data{
			Text:   "The last save was refused; fix these fields (keys are field names) and save again",
			Object: errs,
		}
	})
	~>neui.Use{
		Desc: "product name",
	} <input name="name">
	~>neui.Use{
		Desc: "price in USD, like 24.90",
	} <input name="price">
	...
	~>neui.Use{
		Desc: "save the product with the values in the form",
	} <button type="submit">Add product</button>
</form>
```

The agent fills the name, writes `cheap` as the price and clicks the button. The click's answer tells it what to fix:

```json
{
  "handled": true,
  "diff": {
    "data": [
      {
        "text": "The last save was refused; fix these fields (keys are field names) and save again",
        "object": {
          "category": "choose a category without subcategories",
          "price": "enter a price above 0, like 24.90"
        }
      }
    ]
  }
}
```

Rendered in another space, the errors would reach the agent as one line in `changes`, and it would need one more read to learn why the save failed.

### Navigation

Links that move the user around the app get their own space:

```gox
~>neui.Space{
	Name: "menu",
	Desc: "Main menu",
} <nav>
	~>neui.Use{
		Desc: "go to the catalog: browse by category, search by name, filter by stock",
	} <a (doors.ALink{...})>Catalog</a>
	~>neui.Use{
		Desc: "open the form that adds a product",
	} <a (doors.ALink{...})>New product</a>
</nav>
```

The menu itself stays as the agent read it, and the click's answer names the spaces that came and went, so the agent knows where to continue:

```json
{
  "handled": true,
  "diff": {},
  "changes": [
    {
      "path": "product_form",
      "added": true,
      "desc": "Form adding a product: name, category, price and units in stock"
    },
    { "path": "products", "removed": true }
  ]
}
```

### A table

A table is a space with two child spaces: the filter, with its controls and a small data item (the filter as applied and the number of matches), and the rows:

```gox
~>neui.Space{
	Name: "products",
	Desc: "Product list: a filter and the rows that match it",
} <main>
	~>neui.Space{
		Name: "filter",
		Desc: "Filter of the product list: search by name, hide what is out of stock",
	} <div>
		~>neui.Use{
			Desc: "search products by name",
		} <input type="search" name="q">
		...
		~neui.Data{
			Text:   "the filter as applied and how many products match it",
			Object: applied,
		}
	</div>
	~>neui.Space{
		Name: "rows",
		Desc: "The products that match the filter, with price and stock",
	} <table>
		~neui.Data{
			Text:   "the rows of the table",
			Object: products,
		}
		...
	</table>
</main>
```

The agent writes `grinder` into the search box and sees that the filter applies and how many products match, without the rows:

```json
{
  "handled": true,
  "diff": {
    "data": [
      {
        "text": "the filter as applied and how many products match it",
        "object": {
          "search": "grinder",
          "matching": 3
        }
      }
    ]
  },
  "changes": [
    { "path": "products.rows", "changed": ["data"] }
  ]
}
```

It reads `products.rows` when the user asks about the products themselves. When any data item of a space changes, the diff carries the space's whole data list, so rows kept in the filter space would come back with every change of the filter.

## Fill a space

### Data

`neui.Data` gives the agent context. `Object` takes any value that encodes to JSON; `Text` says what the object is, or stands alone as a note:

```gox
~neui.Data{
	Text:   "the rows of the table",
	Object: products,
}
~neui.Data{
	Text: "Prices are in USD without tax. Stock 0 means sold out.",
}
```

A read of the space returns all of its data, so everything you render there is paid for on every read. Data the agent needs only sometimes goes into a child space or behind a tool.

### Tools

Besides the static tools of your modules, you declare tools inside your layout. A rendered tool can be:

- an API call: an operation of your backend, such as setting a product's stock;
- a data fetch: data the agent loads only when a task needs it;
- the backend of a UI action: the function your form's submit handler calls, taking the whole form as one typed input. The agent does in one call what takes the user four fields and a button.

The product list offers the function behind the form as a tool:

```gox
~>neui.Space{
	Name: "products",
	Desc: "Product list: a filter and the rows that match it",
} <main>
	~neui.Tool[ProductForm, Product]{
		Name:    "add_product",
		Desc:    "Add a product to the catalog.",
		Handler: addProduct,
	}
	...
</main>
```

The agent calls a tool itself or, except at `TierAssistant`, from a script.

A tool's description and schemas reach the agent when it reads the tool's space. Some spaces hold tools only, so the agent loads them when a task needs them and not with every read of a UI space:

```gox
~>neui.Space{
	Name: "api",
	Desc: "The catalog's data and operations as tools",
} <>
	~neui.Tool[findInput, []Product]{
		Name:    "find_products",
		Desc:    "Find products by name in the whole catalog, whatever the list on screen shows.",
		Handler: findProducts,
	}
	~neui.Tool[stockInput, Product]{
		Name:    "set_stock",
		Desc:    "Set the units in stock of a product.",
		Handler: setStock,
	}
</>
```

A tool call answers with the tool's output next to the same `diff` and `changes`. `set_stock` with `{"id": 2, "stock": 20}`, while the list shows only products in stock:

```json
{
  "output": { "id": 2, "name": "Pocket hand grinder", ..., "stock": 20 },
  "diff": {},
  "changes": [
    { "path": "products.filter", "changed": ["data"] },
    { "path": "products.rows", "changed": ["data"] }
  ]
}
```

The page tree lists a space's tool names, so the agent sees them before it reads the space. Pick names that say what the tool does without its description:

```json
{
  "name": "api",
  "desc": "The catalog's data and operations as tools",
  "tools": ["find_products", "set_stock"]
}
```

### Inputs

The agent works with an element in the user's browser once you mark it with `neui.Use`. You can mark `a`, `button`, `input`, `textarea` and `select`. An input of type `submit`, `button`, `reset` or `image` counts as a button, and a radio stands for its whole group.

By default the agent may read the element, write or click it, and focus it for the user:

> User: Did I fill this in right?
>
> Agent: *reads the fields of `product_form`*

> User: *pastes a supplier's product sheet* Add this one.
>
> Agent: *writes name, category, price and stock, clicks "save the product with the values in the form"*

> User: Which field is still wrong?
>
> Agent: *focuses "price in USD, like 24.90"*

`Rights` narrows this:

| Rights | The agent may | Use it for |
|---|---|---|
| `neui.RW` (default) | read, write, click, focus | |
| `neui.R` | read, focus | a step the user confirms themselves; a value the agent reports but must not change |
| `neui.W` | write, click, focus | a value the agent fills in but must not see |

```gox
~>neui.Use{
	Desc:   "supplier account password",
	Rights: neui.W,
} <input type="password" name="supplier_password">
~>neui.Use{
	Desc:   "delete the product for good",
	Rights: neui.R,
} <button type="submit">Delete</button>
```

The description is all the agent knows about an element before it uses it. Keep it informative and short: for a link, say what the destination offers.

```gox
~>neui.Use{
	Desc: "open",
} <a (doors.ALink{...})>~(c.Name)</a>
~>neui.Use{
	Desc: "open " + c.Name + ": its products with price and stock, searchable by name",
} <a (doors.ALink{...})>~(c.Name)</a>
```

`Highlight` puts an element's description into the page tree, where the agent sees it before it reads the element's space. It suits the links of the main menu:

```gox
~>neui.Space{
	Name: "menu",
	Desc: "Main menu",
} <nav>
	~>neui.Use{
		Desc:      "go to the catalog: browse by category, search by name, filter by stock",
		Highlight: true,
	} <a (doors.ALink{...})>Catalog</a>
	~>neui.Use{
		Desc:      "open the form that adds a product",
		Highlight: true,
	} <a (doors.ALink{...})>New product</a>
</nav>
```

What the agent gets in the tree:

```json
{
  "name": "menu",
  "desc": "Main menu",
  "inputs": 2,
  "highlights": [
    "go to the catalog: browse by category, search by name, filter by stock",
    "open the form that adds a product"
  ]
}
```

The space's description stays "Main menu", and the highlighted links tell the agent upfront what the app offers. Every highlight is sent with each tree call: highlight only what helps the agent find its way.

## Check how the agent does

There are rules to follow. Give content that changes on its own, like a live gauge, a space of its own. Keep the diff informative: the form renders its validation data in the space where the agent submits it, so the click's answer carries it.

But in general, check how well the agent performs real tasks in your app, and how much context and how many tool rounds it uses. Optimize and improve the agent's interface as you do the user's.
