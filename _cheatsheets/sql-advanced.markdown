---
title: "Advanced SQL Concepts"
layout: post
name: "sql-advanced"
tags: ["SQL"]
date: 2023-01-12 00:00:00 +0100
author: "Darren Foley"
show_sidebar: false
menubar: cheatsheet_menu
---

This cheatsheet covers SQL concepts that would be considered advanced or beyond the basics of running simple select statements. The following examples use the T-SQL variant of SQL but are more generally applicable to other SQL vendors such as Oracle, Teradata & Postgres.

<br>

## Logical Query Processing

Users write select query statements in "keyed-in order" as follows.

1. SELECT
2. FROM
3. WHERE
4. GROUP BY
5. HAVING
6. ORDER BY

However, the SQL engine executes the statement in logical processing order as follows.

1. FROM, JOIN
2. WHERE
3. GROUP BY
4. HAVING
5. SELECT
6. ORDER BY

Each phase of the operation returns a relation as output to the next step. Its important to understand logical query processing as it explains why you cannot reference an alias in the where clause for example. From a query tuning point of view, its clear that filtering the data is an extremely important part of SQL querying as it happens early on in the processing phase. The less data to process early on the faster the execution time of the query.

<br>

## Window Functions 




