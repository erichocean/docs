---
title: Common Table Expressions
summary: Common Table Expressions (CTEs) simplify the definition and use of subqueries
toc: false
---


<span class="version-tag">New in v2.0:</span>
Common Table Expressions, or CTEs, provide a shorthand name to a
possibly complex [subquery](subqueries.html) before it is used in a
larger query context. This improves readability of the SQL code.

CTEs can be used in combination with [`SELECT`
clauses](select-clause.html) and [`INSERT`](insert.html),
[`DELETE`](delete.html), [`UPDATE`](update.html) and
[`UPSERT`](upsert.html) statements.

<div id="toc"></div>

## Synopsis

<div>{% include sql/{{ page.version.version }}/diagrams/with_clause.html %}</div>

<div markdown="1"></div>

## Parameters

Parameter | Description
----------|------------
`table_alias_name` | The name to use to refer to the common table expression from the accompanying query or statement.
`name` | A name for one of the columns in the newly defined common table expression.
`preparable_stmt` | The statement or subquery to use as common table expression.

## Overview

A query or statement of the form `WITH x AS y IN z` creates the
temporary table name `x` for the results of the subquery `y`, to be
reused in the context of the query `z`.

For example:

{% include copy-clipboard.html %}
~~~ sql
> WITH o AS (SELECT * FROM orders WHERE id IN (33, 542, 112))
  SELECT *
    FROM customers AS c, o
   WHERE o.customer_id = c.id;
~~~

In this example, the `WITH` clause defines the temporary name `o` for
the subquery over `orders`, and that name becomes a valid table name
for use in any [table expression](table-expressions.html) of the
subsequent `SELECT` clause.

This query is equivalent to, but arguably simpler to read than:

{% include copy-clipboard.html %}
~~~ sql
> SELECT *
    FROM customers AS c, (SELECT * FROM orders WHERE id IN (33, 542, 112)) AS o
   WHERE o.customer_id = c.id;
~~~

It is also possible to define multiple common table expressions
simultaneously with a single `WITH` clause, separated by commas. Later
subqueries can refer to earlier subqueries by name. For example, the
following query is equivalent to the two examples above:

{% include copy-clipboard.html %}
~~~ sql
> WITH o       AS (SELECT * FROM orders WHERE id IN (33, 542, 112)),
       results AS (SELECT * FROM customers AS c, o WHERE o.customer_id = c.id)
  SELECT * FROM results;
~~~

In this example, the second CTE `results` refers to the first CTE `o`
by name. The final query refers to the CTE `results`.

## Nested `WITH` Clauses

It is possible to use a `WITH` clause in a subquery, or even a `WITH` clause within another `WITH` clause. For example:

{% include copy-clipboard.html %}
~~~ sql
> WITH a AS (SELECT * FROM (WITH b AS (SELECT * FROM c)
                            SELECT * FROM b))
  SELECT * FROM a;
~~~

When analyzing [table expressions](table-expressions.html) that
mention a CTE name, CockroachDB will choose the CTE definition that is
closest to the table expression. For example:

{% include copy-clipboard.html %}
~~~ sql
> WITH a AS (TABLE x),
       b AS (WITH a AS (TABLE y)
             SELECT * FROM a)
  SELECT * FROM b;
~~~

In this example, the inner subquery `SELECT * FROM a` will select from
table `y` (closest `WITH` clause), not from table `x`.

## Data Modifying Statements

It is possible to use a data-modifying statement (`INSERT`, `DELETE`,
etc.) as a common table expression.

For example:

{% include copy-clipboard.html %}
~~~ sql
> WITH v AS (INSERT INTO t(x) VALUES (1), (2), (3) RETURNING x)
  SELECT x+1 FROM v
~~~

However, the following restriction applies: only `WITH` sub-clauses at
the top level of a SQL statement can contain data-modifying
statements. The example above is valid, but the following is not:

{% include copy-clipboard.html %}
~~~ sql
> SELECT x+1 FROM
    (WITH v AS (INSERT INTO t(x) VALUES (1), (2), (3) RETURNING x)
     SELECT * FROM v);
~~~

This is not valid because the `WITH` clause that defines an `INSERT`
common table expression is not at the top level of the query.

{{site.data.alerts.callout_info}}
If a common table expression contains
a data-modifying statement (<code>INSERT</code>, <code>DELETE</code>,
etc.), the modifications are performed fully even if only part
of the results are used, e.g., with <a
href="limit-offset.html"><code>LIMIT</code></a>. See <a
href="subqueries.html#data-writes-in-subqueries">Data
Writes in Subqueries</a> for details.
{{site.data.alerts.end}}

<div markdown="1"></div>

## Known Limitations

{{site.data.alerts.callout_info}}
The following limitations may be lifted
in a future version of CockroachDB.
{{site.data.alerts.end}}

<div markdown="1"></div>

### Use CTEs At Most Once

It is currently not possible to refer to a CTE by name more than once.

For example, the following query is invalid because the CTE `a` is
referred to twice:

{% include copy-clipboard.html %}
~~~ sql
> WITH a AS (VALUES (1), (2), (3))
  SELECT * FROM a, a;
~~~

### Use CTEs With Data Modifying Statements At Least Once

If a CTE containing data-modifying statement is not referred to,
either directly or indirectly, by the top level query, the
data-modifying statement will not be executed at all.

For example, the following query does not insert any row, because the CTE `a` is not used:

{% include copy-clipboard.html %}
~~~ sql
> WITH a AS (INSERT INTO t(x) VALUES (1), (2), (3))
  SELECT * FROM b;
~~~

Also, the following query does not insert any row, even though the CTE `a` is used, because
the other CTE that uses `a` are themselves not used:

{% include copy-clipboard.html %}
~~~ sql
> WITH a AS (INSERT INTO t(x) VALUES (1), (2), (3)),
       b AS (SELECT * FROM a)
  SELECT * FROM c;
~~~

To determine whether a modification will effectively take place, use
[`EXPLAIN`](explain.html) and check whether the desired data
modification is part of the final plan for the overall query.

## See also

- [Subqueries](subqueries.html)
- [Selection Queries](selection-queries.html)
- [Table Expressions](table-expressions.html)
- [`EXPLAIN`](explain.html)
