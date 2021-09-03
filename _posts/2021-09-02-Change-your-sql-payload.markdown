---
layout: post
title:  "Change your Blind-Error Based Sql-Injection payload now!"
date:   2021-09-02 15:00:00 +0100
categories: research
---

### TL;DR;

Because of PostgreSQL [Planning and Optimization](https://www.postgresql.org/docs/9.5/planner-optimizer.html), Blind-Error Based Sql-Injection payloads like this:

`SELECT CASE WHEN (YOUR-CONDITION-HERE) THEN cast(1/0 as text) ELSE NULL END`

won't work (They will always trigger an error, not depending on the condition). Use something like this instead:

`SELECT CASE WHEN (YOUR-CONDITION-HERE) THEN cast(Version() as numeric) ELSE NULL END`


### Introduction
In my last write up I said I had problems exploiting the Sql Injection I found. Referring to these queries:
```
--This gives error:
'; SELECT CASE WHEN (1=1) THEN cast(1/0 as text) ELSE 'a' END FROM users -- 
--This doesn't:
'; SELECT CASE WHEN (1=0) THEN cast(1/0 as text) ELSE 'a' END FROM users -- 
```
I said:
>Now I had a problem that I still haven’t solved. The above queries work only if the conditions inside the parenthesis are constant, like 1=1 or 1=0. Using something like username = 'administrator' to test if the user administrator exists gives an error. The problem is that using username = 'nonexistent' gives an error too, although  the nonexistent user doesn’t exist. It almost seems like the value of the TRUE condition (cast(1/0 ad text)) is evaluated in any case if the CONDITION contains non-constant values

Initially, I thought I was missing something in my payload, but as it turns out, I found another explanation to this problem.

### What are we talking about here?
The queries above are inspired directly from PortSwigger's [Sql-Injection Cheat Sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet). (Conditional Error Sections). Their payload is the following:
```
SELECT CASE WHEN (YOUR-CONDITION-HERE) THEN cast(1/0 as text) ELSE NULL END
```
And it suffers from the same problem I'm about to discuss. 

First, I want to clarify what kind of SQL injection I'm talking about. In an `Error-Based` sql injection, Error messages are reflected to the user, and can be used to leak data. But in this case of `Blind-Error-Based` our objective is slightly different. Error messages are not reflected directly to the user, but a different page is shown based on if the query triggered an error or not. This is as powerful as a not-blind Sql injection, as we can enumerate the database asking Yes or No questions.

### My Testings
For my testings, I used an [online postgreSQL playground](https://extendsclass.com/postgresql-online.html). Right now it's running `PostgreSQL 11.10`.
As I said in the Introduction, what I noticed is that the following query:

`SELECT CASE WHEN (1=1) THEN cast(1/0 as text) ELSE NULL END`

triggers an error(as expected), while the follwing doesn't (also expected):

`SELECT CASE WHEN (1=0) THEN cast(1/0 as text) ELSE NULL END`

Here comes the problem. When using a real condition we need to test instead of a constant one line `1=1` in this case, the query always throws an error. In my testings I had only one table called `scientist` structured this way:
```
 -------------------------------- 
|id  |	firstname  |	lastname |
 --------------------------------
|1   |	albert	   |	einstein |
|2   |	isaac	   |	newton   |
|3   |	marie	   |	curie    |
 --------------------------------
```
Considering this simple database, both of the following queries trigger a `division by zero` error:

```
SELECT CASE WHEN (firstname='albert') THEN cast(1/0 as text) ELSE NULL END FROM scientist

SELECT CASE WHEN (firstname='idontexist') THEN cast(1/0 as text) ELSE NULL END FROM scientist
```
Notice how in the first query this is expected, because the condition `firstname='albert'` is true at some point (first row). The point is that in the second case the condition is never true, because there is no `idontexist` firstname but the True subexpression `cast(1/0 as text)` is evaluated anyway and thus throws an exception. 

### My findings
It seemed like a constant expression is treated very differently from one using variables. With this in mind I looked into the official documentation and in fact I found exactly what I was looking for.
Section 9.18 is about [Conditional Expressions](https://www.postgresql.org/docs/current/functions-conditional.html) . At the end of section 9.18.1 it says:
>A CASE expression does not evaluate any subexpressions that are not needed to determine the result. For example, this is a possible way of avoiding a division-by-zero failure:
`SELECT ... WHERE CASE WHEN x <> 0 THEN y/x > 1.5 ELSE false END;`

So it seems like we are experiencing some weird behavior, but immediately after there's a note:
>As described in Section 4.2.14, there are various situations in which subexpressions of an expression are evaluated at different times, so that the principle that “CASE evaluates only necessary subexpressions” is not ironclad. For example a constant 1/0 subexpression will usually result in a division-by-zero failure at planning time, even if it's within a CASE arm that would never be entered at run time.

Exactly! So there is this concept of planning time. For optimization reasons, sql queries are planned before being executed and this can completely change the expected behavior we initially wrote. A lot of details are in this [section](https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-EXPRESS-EVAL) of the documentation. Particularly the last part(`4.2.14`)

Now how can we work around this problem and stil get our Blind-Error injection to work? After some testing I found out that using a variable expression instead of `1/0` to trigger an error will work.
For example:
```
SELECT CASE WHEN (YOR-CONDITION-HERE) THEN cast(id/0 as text) ELSE NULL END FROM scientist
```
Will Trigger a `division by zero` error correctly based on if `YOUR-CONDITION-HERE` is true or false. But you might notice a problem here. To use this payload we need to know a numeric type column of the table. In the initial phase of testing for an injection, we often don't have this information. 

I think there are many other ways to trigger an error using an expression that isn't evaluated during planning time. I came up with the following:

```
SELECT CASE WHEN (YOUR-CONDITION-HERE) THEN cast(Version() as numeric) ELSE NULL END
```
This will work, and in case `YOUR-CONDITION-HERE` is true, an `invalid input syntax` error is thrown (because a `text` can't be converted into `numeric`).
