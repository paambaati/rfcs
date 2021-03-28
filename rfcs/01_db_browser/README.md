# RFC — Scaling a Web-Based Multi-Database Browser

The Hasura GraphQL engine connects to different kinds of databases like MySQL, PostgreSQL and MS SQL. APIs are available that will either let us send raw SQL or use auto-generated GraphQL for every table in the database selected.

The problem that this RFC tries to address is (emphasis mine) —

> How to do it in the best way so that it'll be scalable for **different databases** and will cover all of our functionalities (filter, sort), how can we have **type safety** if we don't have any information about schemas and tables in build time? 
