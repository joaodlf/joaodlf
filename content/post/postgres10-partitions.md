---
title: "PostgreSQL 10: Partitions of partitions!"
date: 2018-01-20
keywords:
- postgres
- postgresql
- sql
- postgres partitioning
---

PostgreSQL version 10 brings a much anticipated feature: **(Native) Table Partitioning**.

(Emphasis on the **NATIVE**, PostgreSQL supported partitioning on previous versions by
other means.)

There are a few gotchas you have to keep in mind when using this new feature: No PK allowed;
No ON CONFLICT clauses; etc... In my experience, these are all just slight annoyances,
hardly a reason to not benefit from this amazing feature.

You are also limited to two types of partitioning: **RANGE** and **LIST**.

A typical **RANGE** declaration would look something like this:

```sql
CREATE TABLE dt_totals (
    dt_total date NOT NULL,
    geo varchar(2) not null,
    impressions integer DEFAULT 0 NOT NULL,
    sales integer DEFAULT 0 NOT NULL
)
PARTITION BY RANGE (dt_total);
```

We would then create a table to act as a partition of the above:

```sql
CREATE TABLE dt_totals_201801
PARTITION OF dt_totals
FOR VALUES FROM ('2018-01-01') TO ('2018-01-31');
```

Any data for January would be stored in this partition. When it comes to performance
for large tables, this is a fantastic new feature!

But what if this isn't enough? What if we could also benefit from a **LIST**
partition on my geo field? There is no way to partition by both RANGE and LIST... Not in
a single declaration, that is!

### Above and Beyond: Partition it all!

I was surprised when I found out that you can actually create partitions of partitions!

>... it is possible to add a regular or partitioned table containing data as a partition of a partitioned table
([postgresql.org/docs](https://www.postgresql.org/docs/10/static/ddl-partitioning.html))

First, let's DROP our previous attempt: ```DROP TABLE dt_totals_201801;```

Now we get to the cool part:

```sql
CREATE TABLE dt_totals_201801
PARTITION OF dt_totals
FOR VALUES FROM ('2018-01-01') TO ('2018-01-31')
PARTITION BY LIST (geo);
```

*That's right!* We just created a RANGE partition which we can now also partition by LIST!

I can now create a LIST partition for each of my geos: UK, US and AU:

```sql
CREATE TABLE dt_totals_UK_201801 PARTITION OF dt_totals_201801 FOR VALUES IN ('UK');
CREATE TABLE dt_totals_US_201801 PARTITION OF dt_totals_201801 FOR VALUES IN ('US');
CREATE TABLE dt_totals_AU_201801 PARTITION OF dt_totals_201801 FOR VALUES IN ('AU');
```

### Conclusions

Native Partitioning was a highly anticipated PostgreSQL feature. I truly believe this to
be a game changer when it comes to hard decisions on database solutions.
Do you really need a new, complicated, NOSQL solution to handle your growing data?
Partitioning in PostgreSQL 10 might just be what you need (and save you a lot of headches!).

It's important to note that other RDBMSs (namely, MySQL and derivatives)
have had the ability to perform basic, declarative partitioning before PostgreSQL.
While this is the case, PostgreSQL has had the ability to partition data far before most
other systems (including MySQL) - Just not in a declarative, native format.
More information on partitioning before version 10 is available [here](https://www.postgresql.org/docs/9.6/static/ddl-partitioning.html).