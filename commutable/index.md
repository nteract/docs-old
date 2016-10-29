# commutable

commutable is an nteract library that allows you to interact with a Jupyter
Notebook file. Using commutable, you can create a Notebook document, append
cells, update outputs, clear outputs, update the content of a cell and much
more. commutable is written in [TypeScript]() so type-checking is enforced on
all methods. Let's get started with commutable.

## Prerequisites

We'll be working in the shell and using the Node REPL (read-eval-print-loop
or a fancy way of saying "enter a command and get a response back"). So,
you'll need to be familiar with how to:

- open a Terminal
- enter basic commands in the shell
- install Node and npm on your system

## Installing libraries

### commutable

Let's start off by installing commutable. commutable is an npm package so we
can install it by using

```
$ npm install commutable
```

**Example and output**

```
$ npm install commutable
/Users/carol/code/nteract/docs
└─┬ commutable@1.1.3
  ├── immutable@3.8.1
  ├── lodash.repeat@4.1.0
  └── uuid@2.0.3

npm WARN enoent ENOENT: no such file or directory, open '/Users/carol/code/nteract/docs/package.json'
npm WARN docs No description
npm WARN docs No repository field.
npm WARN docs No README data
npm WARN docs No license field.
$
```

Note: You may see WARN messages while the project is in Alpha. You may ignore them for now.

### uuid

Each notebook cell will need a unique ID. We'll need to install the `uuid`
library for Node to help with creating unique IDs. We'll show you how to use
it in a later section "Setting a unique ID for a code cell". Here's the
installation command:

```
$ npm install uuid
```

**Example and output**

```
$ npm install uuid
/Users/carol/code/nteract/docs
└─┬ commutable@1.1.3
  └── uuid@2.0.3

npm WARN enoent ENOENT: no such file or directory, open '/Users/carol/code/nteract/docs/package.json'
npm WARN docs No description
npm WARN docs No repository field.
npm WARN docs No README data
npm WARN docs No license field.
$
```

## Importing commutable package

Once we've installed commutable, we can open up our Node REPL and start to interact
with its functions. Let's start `node`:

```
$ node
```

This will the REPL prompt `>` where you will enter commands:

```
$ node
>
```

Let's start by importing the commutable package into our
context.

```
> const commutable = require("commutable");
```

## Creating an empty notebook

Now that we have imported the commutable package, we can create our first
empty notebook.

```
> const notebook = commutable.emptyNotebook;
> notebook
Map {
	"cellOrder": List [],
	"nbformat": 4,
	"nbformat_minor": 0,
	"cellMap": Map {}
}
```

Note: Your system may display the Map without formatting, such as:

```
> notebook
Map { "cellOrder": List [], "nbformat": 4, "nbformat_minor": 0, "cellMap": Map {} }
```

As long as the elements inside the braces `Map {}` is similar, you are doing
fine with the tutorial.

## Learning about Immutable.js and the notebook architecture

An empty Notebook is an [Immutable.js]() Map object that enforces the
requirements for version 4 of the notebook format specification. This
specification outlines what keys the JSON representing a Jupyter Notebook
document should have and how the JSON is structured. In the empty notebook, we
have the following keys set up.

* **cellOrder** is a List that contains the IDs of the cell in the order that
  they appear in the notebook.
* **nbformat** and **nbformat_minor** are used to specify the version of the
  Jupyter Notebook specification that this notebook complies with.
* **cellMap** is a Map that associates a cell ID with the cell it matches
  with.

To understand `cellMap`, we'll need to understand how cells are represented in
commutable. Let's take a look.

```
> commutable.emptyCodeCell
Map {
	"cell_type": "code",
	"execution_count": null,
	"metadata": Map {
		"collapsed": false
	},
	"source": "",
	"outputs": List []
}
```

The `emptyCodeCell` object is an Immutable map that contains the data
associated with the notebook. This contains things like the `cell_type`
(`code` or `markdown`), the `execution_count`, the `source`, and the
`outputs`. Let's see if we can add this code cell to the notebook that we
created earlier. There is one difference between the way Jupyter formats
notebook and the way commutable formats notebooks. commutable requires a
unique ID to be associated with each cell for per-cell operations. This ID is
stored inside the `cellOrder` list and used as the keys in the `cellMap`.

## Setting a unique ID for a code cell

Earlier, you installed "uuid". Let's begin using it. First, require it into
our context.

```
> const uuid = require("uuid");
```

We can create a unique ID using the following syntax.

```
> const uniqueId = uuid.v4();
```

Now that we have a unique ID that we can associate with our empty code cell,
we can add our empty code cell to our notebook using commutable.

```
> const notebookWithCell = commutable.appendCell(notebook, commutable.emptyCodeCell, uniqueId);
```

Note that the `appendCell` function returns to you a new notebook with the
appended cell and does not modify the notebook that you pass it. We now have a
notebook with a single empty code cell!

## Editing the content of an empty code cell

Now, let's see if we can edit the content of the empty code cell we just added
it. With commutable, of course we can!

```
> const notebookWithCellAndContent = commutable.updateSource(notebookWithCell, uniqueId, "print('a')");
```

This function let's commutable know that we would like to update the source
inside the code cell with the ID `uniqueId` in the notebook `notebookWithCell`
to `"print('a')"`.

Let's take a look at the notebook object to make sure that this change has
been made.

```
> notebookWithCellAndContent
Map {
	"cellOrder": List [ "da15e63f-8838-482e-9de9-a4444af8acfa" ],
	"nbformat": 4,
	"nbformat_minor": 0,
	"cellMap": Map {
		"da15e63f-8838-482e-9de9-a4444af8acfa": Map {
			"cell_type": "code",
			"execution_count": null,
			"metadata": Map {
				"collapsed": false
			},
			"source": "print('a')",
			"outputs": List []
		}
	}
}
```

Nice!

## Updating the code cell output

Now let's say that we've used enchannel to send a message to a Python
kernel and receive an output for running `print("a")` on the Python REPL. How
can we update the output of the code cell?

The outputs of a cell are represented as Immutable Lists to accommodate. So
we'll need to install `immutable`.

```
npm install immutable
```

And then require the List object from the `immutable` REPL into our context.

```
> const List = require("immutable").List;
```

```
> const notebookWithOutputs = commutable.updateOutputs(notebookWithCellAndContent, uniqueId, new List(["a"]));
```

Let's take a look at what our notebook with a single code cell with source and
outputs looks like.

```
> notebookWithOutputs
Map {
	"cellOrder": List [ "da15e63f-8838-482e-9de9-a4444af8acfa" ],
	"nbformat": 4,
	"nbformat_minor": 0,
	"cellMap": Map {
		"da15e63f-8838-482e-9de9-a4444af8acfa": Map {
			"cell_type": "code",
			"execution_count": null,
			"metadata": Map {
				"collapsed": false
			},
			"source": "print('a')",
			"outputs": List [ "a" ]
		}
	}
}
```

Sweet! You can repeat this process for as many code cells as you like! In
addition to code cells, you can also create Markdown cells using
`commutable.emptyMarkdownCell`.

Happy hacking!

## Major keys

So, what are the main takeaways from this tutorial?

* commutable stores everything as an Immutable object.
* A commutable function is non-mutating.
* A commutable cell has a unique ID.
