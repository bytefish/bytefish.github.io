title: Visualizing PostgreSQL Query Plans
date: 2026-05-09 09:20
tags: postgres, angular
category: postgres
slug: visualizing_postgresql_query_plans
author: Philipp Wagner
summary: An application for visualizing PostgreSQL Query Plans.

I am sure, there are great tools to visualize PostgreSQL Query Plans, so why not 
add another one to the list? Let me introduce you to pgplan-viz, which is a web 
app to visualize PostgreSQL `EXPLAIN` query plans and export them as PNG.

<div style="display:flex; align-items:center; justify-content:center;margin-bottom:15px;">
    <a href="/static/images/notes/pg-planner.jpg">
        <img src="/static/images/notes/pg-planner.jpg" alt="CSV Explorer">
    </a>
</div>

You can find it here:

* [https://www.bytefish.de/static/apps/pgplan-viz/](https://www.bytefish.de/static/apps/pgplan-viz/) (English)

The code is in a Git Repository at:

* [https://github.com/bytefish/misc/tree/main/pgplan-viz](https://github.com/bytefish/misc/tree/main/pgplan-viz)

## Creating the EXPLAIN as JSON ##

The following PL/SQL statement creates a JSON-formatted Query Plan for you to analyze:

```
 EXPLAIN (ANALYZE, COSTS, VERBOSE, BUFFERS, FORMAT JSON) <QUERY>
```