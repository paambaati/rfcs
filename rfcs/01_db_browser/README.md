# RFC — Scaling a Web-Based Multi-Database Browser

The Hasura GraphQL engine connects to different kinds of databases like MySQL, PostgreSQL and MS SQL. APIs are available that will either let us send raw SQL or use auto-generated GraphQL for every table in the database selected.

The problem that this RFC tries to address is (emphasis mine) —

> How to do it in the best way so that it'll be scalable for **different databases** and will cover all of our functionalities (filter, sort), how can we have **type safety** if we don't have any information about schemas and tables in build time? 

## Background

The Hasura GraphQL Engine connects to customers' databases, and with the recent Hasura 2.0 release a single Hasura GraphQL Engine might be connected to multiple different databases and these databases might be different types (PostgreSQL, MySQL, MS Server, etc.)

In the console, we allow customers to browse the data in the databases they've connected to Hasura. The Browse Rows tab (see images below) allows them to view the data from a particular table in a particular database. We have information about column names, column types and other table details only in runtime, and the data might be stored in different types of databases. There are two kind of APIs that we can use:

- Raw SQL API — we can send any SQL query to the backend that will be executed on a particular database.

- GraphQL API — generated for all the tables.

We also let users to filter their data by various operators (depending on the type of the database). Users can also sort the data by column ascending or descending.

&nbsp;             |  &nbsp;
|-----------------------------|------------------------------------|
![Hasura GraphQL Engine Data browser](hasura_graphql_engine_1.png)  |  ![Hasura GraphQL Engine Data browser](hasura_graphql_engine_2.png)

The goal is to try to scale this for newer future databases and have type-safety, while still maintaining all the querying capabilities.

## Motivation

[Engineers are building new databases everyday](https://www.infoworld.com/article/3563548/do-we-need-so-many-databases.html), thanks to newer high-performance languages, cheaper data storage costs, more domain-specific storage & retrieval charateristics and an increased proliferation of [NIH syndrome](https://en.wikipedia.org/wiki/Not_invented_here).

For Hasura's offerings to benefit more users and capture an even bigger market, it is imperative to support more databases.

As no two databases are alike, building a data querying experience that works equally well for all databases becomes a challenge. The below proposals attempt to offer some longer-term solutions.

## Proposals

### 1. Free-form query editor

The current experience of 3 input fields (column, operation and value) for executing queries would not scale well in a few cases —

1. When Hasura starts to support more databases; especially graph, KV or columnar databases.
2. When the user wishes to query spatial fields (like [GIS](https://postgis.net)) or composite fields (like the [`json` type in PostgreSQL](https://www.postgresql.org/docs/10/datatype-json.html)).

Other solutions that let users query databases and view results instantly, like [Redash](https://redash.io/) and [Metabase](https://www.metabase.com/), default to a free-form query editor so that it gives more power & flexibility to users and is easier to support multiple databases.

Redash             |  Metabase
|-----------------------------|------------------------------------|
![Redash query editor](redash_query_editor.png)  |  ![Metabase query editor](metabase_query_editor.png)

#### **Implementation**

This implementation offers an even more powerful query editing experience than the examples linked above.

We use [Microsoft's Monaco Editor](https://microsoft.github.io/monaco-editor/) as the centerpiece of the query editing experience. It is the core text editing engine powering the very popular VSCode, and offers rich code syntax highlighting, hints, and autocomplete.

The Monaco editor can be provided with rich context-aware autocomplete via a custom [Language Server Protocol](https://microsoft.github.io/language-server-protocol/) (LSP) service. This LSP service will connect to the database the user has selected, pick up query-specific schema information and serve it to the frontend editor.

For example, say the user has selected the table `books` from the `library` database, and say the user begins to type this query —

```sql
SELECT 
```

The LSP service will query the schema for the `books` table and offer all columns from that table as options, which can be used by Monaco as suggestions.

The steps involved in implementing such an architecture are —

1. Build an implementation of the LSP for the databases that Hasura supports.

   Prior art — [`sqls`](https://github.com/lighttiger2505/sqls), a Go-based implementation of the LSP that supports PostgreSQL, MySQL and SQLite3.

2. Build a "connector" for Monaco on the frontend that can talk to the LSP service via a suitable protocol like WebSockets.

    Prior art — [Monaco language client](https://github.com/TypeFox/monaco-languageclient#monaco-language-client), a TypeScript project that makes it easy to connect a Monaco instance to a backend LSP service.

![Monaco + Hasura LSP architecture](monaco_lsp_architecture.png)
<!-- https://swimlanes.io/d/i_Bcaxe-2 -->

It might also be worth exploring the possibility of building the LSP into a [WASM](https://webassembly.org/) binary so that it can be made to work directly on the browser, and using the existing Hasura DB APIs to fetch catalog/schema information. This has the added advantages of —

 1. Not needing the LSP as an additionally running service.
 2. Making it possible to build an Electron-based desktop app, if we have to in the future.

#### **Type safety**

Type safety in this proposal is provided by the LSP. For example, if the user has entered —

```sql
SELECT * FROM books WHERE isbn = true;
```

`isbn` is a string type, while the value given is a boolean. In this cause, LSP can [validate the query and provide an error](https://code.visualstudio.com/api/language-extensions/language-server-extension-guide#adding-a-simple-validation) to Monaco to underline the token "true" with a red squiggle underneath it, and provide a useful hint that the type isn't matching.

#### **Pagination & sorting**

In this proposal, limiting the number of rows is left up to the user — they can use a `LIMIT` clause if they'd like. Otherwise, the table will display the total number of pages in the "Pages" field, using which the user can navigate to the page of their choice, with the backend API handling pagination in addition to the raw query.

Sorting is done client-side and per-page.

#### **Pros & cons**

| **Pros** | **Cons** |
|-|-|
| Much richer querying experience for users. | Building and maintaining an LSP implementation is arguably more technical work, and might be more challenging for the backend engineering team. |
| Removes almost all frontend work in introducing support for a new database (even new types like KV stores or graph DBs).  | More moving parts; now there's 1 extra service (or possibly more if we decide to build one for each DB) to deploy, monitor, troubleshoot and scale **unless** the LSP can be WASM-ified and delivered on the frontend. |
| If we decide to build or contribute to an LSP implementation in the open, we stand to gain contributions from the community. | _Maybe_ a regression in user-experience for users that prefer the existing simpler 3-input-fields querying experience. |


### 2. Scaling the current experience

This proposal focuses on building a solid base for adding newer databases in the future while looking at ways to improve runtime type-safety, also keeping the existing experience of querying & filtering data with easy-to-use inputs.

#### **Implementation**

We start by identifying "primitive" data types across databases that can be mapped directly to user input in a traditional `<input>` HTML field. This can be included in the frontend as a static object; let's call this `fieldTypesMap`.

```javascript
// fieldTypesMap
{
	"mysql": {
		"text": ["CHAR", "VARCHAR", "TEXT"],
		"number": ["INTEGER", "INT", "SMALLINT", "TINYINT", "MEDIUMINT", "BIGINT", "DECIMAL", "FLOAT", "DOUBLE"],
		"date": ["DATE", "DATETIME", "TIMESTAMP"]
	},
	"postgresql": {
		"text": ["CHAR", "VARCHAR", "TEXT"],
		"number": ["INTEGER", "INT", "SMALLINT", "BIGINT", "DECIMAL", "FLOAT", "REAL"],
		"radio": ["BOOLEAN"]
	},
	"mssql": {
	  // ...
	}
}
```

When the user selects a particular table to browse, we query the API for that table's catalog/schema to populate the `column` field in the page. With the schema, we find out the data type of each column, and look up the corresponding input type from `fieldTypesMap`. With that, we render a type-safe and type-appropriate input component.

For example, say the user selects the `books` table from the `library` database to browse, and say the table's schema looks like this —

| Column | Type |
|-|-|
| `id` | `INT` |
| `isbn` | `VARCHAR` |
| `title` | `VARCHAR` |
| `added` | `TIMESTAMP` |

When the user selects the `added` field in the "Column" dropdown, `fieldTypesMap` is read and the other 2 fields - "Operation" and "Value", both offer input only relevant to a `TIMESTAMP` field.

For this to work, both the "Operation" and "Value" components have to be aware of the "Column" field's value.

##### **`<OperationDropdown>`**

The operation dropdown component will take the field type as a `prop` and render only relevant options.

```typescript
type FieldTypes = 'text' | 'number' | 'date'

cont fieldTypesMap = {
    'text': ['_like', '_nlike', '_in', '_notin'],
    'number': ['_eq', '_neq', '_gt', '_lt', '_gte', '_lte'],
    'date': ['_eq', '_neq', '_gt', '_lt', '_gte', '_lte'],
} as Record<FieldTypes, string[]>

type OperationDropdownProps = DropdownProps & {
    fieldType: FieldTypes
}

const OperationDropdown = (props: OperationDropdownProps) => {
    const { fieldType, ...rest } = props
    const options = fieldTypesMap[fieldType]
    return (
        <Dropdown options={options} {...rest} />
    );
}
```

##### **`<ValueInput>`**

Similar to `<OperationDropdown>`, the `<ValueInput>` component will take the field type as a `prop` and render an appropriate component — for example, a date input component will be rendered for field type `date`.

#### **Building new input components for more data types**

One possible extension to this proposal is to build new components for supporting more data types — for example, a Map component could be built for a few specific types of GIS queries.

However, this could soon become very unwieldly as there far too many data types that are wildly different from one another.

#### **Type safety**

Type safety in this proposal is provided by relying on the static `fieldTypesMap`. This ensures only the type-appropriate components (and their validations) are rendered for every column.

#### **Pros & cons**

| **Pros** | **Cons** |
|-|-|
| Smaller delta of effort (at least as compared to proposal 1). | Does not support advanced fields — spatial (GIS) fields and composite fields like JSON. |
| Fairly easy to onboard a new supported database — just add new type mapping to `fieldTypesMap`.  | |

## Performance considerations

One of the original goals of the problem statement is to build for scalability and performance, making sure the query experience works well for very large tables as well. Performance here speaks about both —

1. The backend API's performance — i.e. how quickly can the backend compile, optimize & execute the query and return the results

2. The frontend's performance — i.e. how smoothly can users expect pagination & sorting to work, and how easily many rows are displayed without loss in responsiveness.

### Pagination on the backend

The classic problem of paginating large volumes of data in a table is usually solved in 2 ways —

1. Offset-based pagination

    Most databases offer support for the `LIMIT` and `OFFSET` (sometimes called `SKIP`) clauses. Using these 2 clauses, one can query for the second page and for 10 rows with a query like —

    ```sql
    SELECT * FROM books LIMIT 10 OFFSET 10;
    ```

    This works well for tables that are small and ones that don't see a lot of `INSERT`s happening. This performs poorly with very large tables.

2. Cursor-based pagination

    For tables that often have lots of `INSERT`s or ones that have a large number of rows, cursor-based pagination is more performant. A cursor is a pointer to a specific row, and paginating with a cursor assumes that the column is unique and sequential. Assuming there's an auto-incrementing `id` column in the `books` table, one can query for the second page and for 10 rows with a query like —

    ```sql
    SELECT * FROM books WHERE id > 10 LIMIT 10;
    ```

### Pagination on the frontend

Sorting, in most cases, can be done client-side, while pagination will require calls to the backend API.

While most libraries and implementations of tabular data in React land are quite performant, there's a few things to keep in mind, especially when displaying thousands of rows.

1. Keeping the number of DOM nodes down

    As the number of DOM nodes increases, scroll performance and overall UI responsiveness goes down. To work around this, libraries like [`react-virtualized`](http://bvaughn.github.io/react-virtualized) work by loading only elements in the visible area of the browser. This can help display a table with thousands of rows without any perceived loss in performance.

    Most popular table libraries (like [`react-table`](https://react-table.tanstack.com/docs/examples/virtualized-rows)) directly support this virtualization/windowing technique. See [example](https://react-table.tanstack.com/docs/examples/virtualized-rows).

2. Using the HTML5 `<canvas>` element if needed

    A table element built with a `<canvas>` element can be highly performant, but will usually need more lower-level manipulation of all styling and event handling.

    See [`glide-data-grid`](https://grid.glideapps.com/) for one such implementation.

3. Use as many small components as possible

    React re-renders components every time their `props` change. In tables, a lot of things change — cells, rows, columns, header & footer page components, etc. Every time a prop changes, it would be expensive to re-render the table, so it makes sense to instead split every table part into its own component and pass them [memoized](https://reactjs.org/docs/hooks-faq.html#how-to-memoize-calculations) `props` so they re-render only when absolutely necessary.

## Next Steps

The choice between the 2 proposed solutions boils down to a few things.

1. Available time, resources and skills in engineering.
2. User research/feedback on whether we can move from the current experience to a fully-powered raw SQL editor.
