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

