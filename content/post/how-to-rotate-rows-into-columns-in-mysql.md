---
title: 'How to rotate rows into columns in MySQL'
date: Sun, 18 Jan 2015 19:00:38 +0000
draft: false
tags: [crosstab, data, mysql, pivot, rows-to-columns, SQL, SQL queries, transform]
excerpt: "How to PIVOT in MySQL"
---

I have answered a lot of [MySQL pivot questions over on Stack Overflow](http://stackoverflow.com/search?tab=newest&q=user%3a426671%20%5bpivot%5d%20is%3aanswer%20%5bmysql%5d) and a few over on [Database Administrators](http://dba.stackexchange.com/search?q=user%3A9003+%5Bpivot%5D+is%3Aanswer+%5Bmysql%5D) and have learned some things about how to transform data in MySQL.

Unfortunately, MySQL does not have `PIVOT` function, so in order to rotate data from rows into columns you will have to use a `CASE` expression along with an aggregate function.

Let's set up some sample data.

{{< highlight sql>}}
create table products
(
  prod_id int not null,
  prod_name varchar(50) not null,
  primary key (prod_id)
);  

insert into products (prod_id, prod\name)
values (1, 'Shoes'), (2, 'Pants'), (3, 'Shirt');

create table reps
(
  rep_id int not null,
  rep_name varchar(50) not null,
  primary key (rep_id)
);

insert into reps (rep_id, rep_name)
values (1, 'John'), (2, 'Sally'), (3, 'Joe'), (4, 'Bob');

create table sales
(
  prod_id INT NOT NULL,
  rep_id INT NOT NULL,
  sale_date datetime not null,
  quantity int not null,
  PRIMARY KEY (prod_id, rep_id, sale_date),
  FOREIGN KEY (prod_id) REFERENCES products(prod_id),
  FOREIGN KEY (rep_id) REFERENCES reps(rep_id)
);

insert into sales (prod_id, rep_id, sale_date, quantity)
values 
  (1, 1, '2013-05-16', 20),
  (1, 1, '2013-06-19', 2),
  (2, 1, '2013-07-03', 5),
  (3, 1, '2013-08-22', 27),
  (3, 2, '2013-06-27', 500),
  (3, 2, '2013-01-07', 150),
  (1, 2, '2013-05-01', 89),
  (2, 2, '2013-02-14', 23),
  (1, 3, '2013-01-29', 19),
  (3, 3, '2013-03-06', 13),
  (2, 3, '2013-04-18', 1),
  (2, 3, '2013-08-03', 78),
  (2, 3, '2013-07-22', 69);
{{< / highlight >}}

We can easily query the rep, sales, and product data by joining the tables:

{{< highlight sql>}}
select 
  r.rep_name,
  p.prod_name,
  s.sale_date,
  s.quantity
from reps r
inner join sales s
  on r.rep_id = s.rep_id
inner join products p
  on s.prod_id = p.prod_id
{{< / highlight >}}

This will give us the data in the format:

| REP_NAME | PROD_NAME |                       SALE_DATE | QUANTITY |
|----------|-----------|---------------------------------|----------|
|     John |     Shoes |      May, 16 2013 00:00:00+0000 |       20 |
|     John |     Shoes |     June, 19 2013 00:00:00+0000 |        2 |
|     John |     Pants |     July, 03 2013 00:00:00+0000 |        5 |
|     John |     Shirt |   August, 22 2013 00:00:00+0000 |       27 |

But what if we want to see the reps in separate rows with the total number of products sold in each column. This is where we need to implement the missing `PIVOT` function, so we'll use the aggregate function `SUM` with conditional logic instead.

{{< highlight sql>}}
select 
  r.rep_name,
  sum(case when p.prod_name = 'Shoes' then s.quantity else 0 end) as Shoes,
  sum(case when p.prod_name = 'Pants' then s.quantity else 0 end) as Pants,
  sum(case when p.prod_name = 'Shirt' then s.quantity else 0 end) as Shirt
from reps r
inner join sales s
  on r.rep_id = s.rep_id
inner join products p
  on s.prod_id = p.prod_id
group by r.rep_name;
{{< / highlight >}}

The conditional logic of the `CASE` expression works hand in hand with the aggregate function to only get a total of the `prod_name` that you want in each column. Since we have 3 products, then you'd write 3 `sum(case...` expressions for each column. [Here is a demo on SQL Fiddle.](http://sqlfiddle.com/#!9/cbad7/12/0) This query will give a result of:

| REP_NAME | SHOES | PANTS | SHIRT |
|----------|-------|-------|-------|
|      Joe |    19 |   148 |    13 |
|     John |    22 |     5 |    27 |
|    Sally |    89 |    23 |   650 |

This could easily be rewritten to show the reps in each column and the products in the rows.

{{< highlight sql>}}
select 
  p.prod_name,
  sum(case when r.rep_name = 'John' then s.quantity else 0 end) as John,
  sum(case when r.rep_name = 'Sally' then s.quantity else 0 end) as Sally,
  sum(case when r.rep_name = 'Joe' then s.quantity else 0 end) as Joe,
  sum(case when r.rep_name = 'Bob' then s.quantity else 0 end) as Bob
from products p
inner join sales s
  on p.prod_id = s.prod_id
inner join reps r
  on s.rep_id = r.rep_id
group by p.prod_name;
{{< / highlight >}}

And now the data is [reversed](http://sqlfiddle.com/#!9/cbad7/16/0):

| PROD_NAME | JOHN | SALLY | JOE | BOB |
|-----------|------|-------|-----|-----|
|     Pants |    5 |    23 | 148 |   0 |
|     Shirt |   27 |   650 |  13 |   0 |
|     Shoes |   22 |    89 |  19 |   0 |

As you can see this is a fairly straightforward and easy way to convert rows into columns when you have a limited number of values. We only had 3 products and 4 reps, so we didn't have a lot of code to write. Things get a bit more complicated when we have an unknown number of columns to transform. If you aren't going to know the values ahead of time, then you will need to look at using a [prepared statement](http://dev.mysql.com/doc/refman/5.0/en/sql-syntax-prepared-statements.html) along with dynamic SQL.

When using a prepared statement, you will write a sql string that will then be executed by the server. I always recommend writing a hard-coded version of a query before attempting to write anything dynamically. This will allow you to get the logic correct before doing it with dynamic SQL.

Let's set up a dynamic query using the data above. You need to report the total quantity of items sold by each rep for each month/year combination. Again, this is easy if you only had a few dates, you would write the query:

{{< highlight sql>}}
select 
  r.rep_name,
  sum(case when Date_format(s.sale_date, '%Y-%M')= '2013-January' then s.quantity else 0 end) as `2013-January`,
  sum(case when Date_format(s.sale_date, '%Y-%M')= '2013-February' then s.quantity else 0 end) as `2013-February`,
  sum(case when Date_format(s.sale_date, '%Y-%M')= '2013-March' then s.quantity else 0 end) as `2013-March`,
  sum(case when Date_format(s.sale_date, '%Y-%M')= '2013-April' then s.quantity else 0 end) as `2013-April`,
  sum(case when Date_format(s.sale_date, '%Y-%M')= '2013-May' then s.quantity else 0 end) as `2013-May`
from reps r
inner join sales s
  on r.rep_id = s.rep_id
inner join products p
  on s.prod_id = p.prod_id
group by r.rep_name;
{{< / highlight >}}

You'd get the [result](http://sqlfiddle.com/#!9/cbad7/20/0):

| REP_NAME | 2013-JANUARY | 2013-FEBRUARY | 2013-MARCH | 2013-APRIL | 2013-MAY |
|----------|--------------|---------------|------------|------------|----------|
|      Joe |           19 |             0 |         13 |          1 |        0 |
|     John |            0 |             0 |          0 |          0 |       20 |
|    Sally |          150 |            23 |          0 |          0 |       89 |

But what happens if you don't know the dates ahead of time, or you want to pass in certain parameters to filter the dates and make the report flexible. This is where dynamic SQL is needed. In order to create the sql string to execute, you'll first need to get a full list of the dates from your `sales` table. This list will be created by using [`GROUP_CONCAT()`](http://dev.mysql.com/doc/refman/5.0/en/group-by-functions.html#function_group-concat) and [`CONCAT()`](http://dev.mysql.com/doc/refman/5.0/en/string-functions.html#function_concat).

{{< highlight sql>}}
SET @sql = NULL;
SELECT
  GROUP_CONCAT(DISTINCT
    CONCAT(
      'sum(case when Date_format(s.sale_date, ''%Y-%M'') = ''',
      dt,
      ''' then s.quantity else 0 end) AS `',
      dt, '`'
    )
  ) INTO @sql
from
(
  select Date_format(s.sale_date, '%Y-%M') as dt
  from sales s
  order by s.sale_date
) d;

select @sql;
{{< / highlight >}}

This code creates a full list of all the dates inside of the `CASE` expression and the aggregate function.

{{< highlight sql>}}
sum(case when Date_format(s.sale_date, '%Y-%M') = '2013-January' then s.quantity else 0 end) AS `2013-January`,
sum(case when Date_format(s.sale_date, '%Y-%M') = '2013-February' then s.quantity else 0 end) AS `2013-February`,
sum(case when Date_format(s.sale_date, '%Y-%M') = '2013-March' then s.quantity else 0 end) AS `2013-March`,
...
{{< / highlight >}}

Now, your full code using the prepared statement will be:

{{< highlight sql>}}
SET @sql = NULL;
SELECT
  GROUP_CONCAT(DISTINCT
    CONCAT(
      'sum(case when Date_format(s.sale_date, ''%Y-%M'') = ''',
      dt,
      ''' then s.quantity else 0 end) AS `',
      dt, '`'
    )
  ) INTO @sql
from
(
  select Date_format(s.sale_date, '%Y-%M') as dt
  from sales s
  order by s.sale_date
) d;

SET @sql 
  = CONCAT('SELECT r.rep_name, ', @sql, ' 
            from reps r
            inner join sales s
              on r.rep_id = s.rep_id
            inner join products p
              on s.prod_id = p.prod_id
            group by r.rep_name;');

PREPARE stmt FROM @sql;
EXECUTE stmt;
DEALLOCATE PREPARE stmt;
{{< / highlight >}}

Which gives the [final result](http://sqlfiddle.com/#!2/40aea/20/4):

| REP_NAME | 2013-JANUARY | 2013-FEBRUARY | 2013-MARCH | 2013-APRIL | 2013-MAY | 2013-JUNE | 2013-JULY | 2013-AUGUST |
|----------|--------------|---------------|------------|------------|----------|-----------|-----------|-------------|
|      Joe |           19 |             0 |         13 |          1 |        0 |         0 |        69 |          78 |
|     John |            0 |             0 |          0 |          0 |       20 |         2 |         5 |          27 |
|    Sally |          150 |            23 |          0 |          0 |       89 |       500 |         0 |           0 |

In a few lines of code you've got a flexible solution that returns any number of columns.

**One note** about `GROUP_CONCAT()`, MySQL has a default length on a system variable used for concatenation. The system variable is called [`group_concat_max_len`](http://dev.mysql.com/doc/refman/5.0/en/server-system-variables.html#sysvar_group_concat_max_len) and the default value is 1024, which means if you have a string that will be longer that 1024, then you will need to alter this variable to allow for a longer string.

These are just a few ways that you can convert rows into columns using MySQL.