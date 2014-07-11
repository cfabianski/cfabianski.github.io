---
layout: post
title:  "How to represent a world tree structure"
date:   2014-07-10
categories: postgresql, rails
metas: [awesome_nested_set, ltree_hierarchy, ltree, rails, postgresql]
---

In our main product ([BRIDGE][bridge]), a Store Locator, the SEO has to be really good.
As such, we had to store a world tree structure in the database.

## Use Case

A point of interest is attached to a `world_node_id` which has to be a leaf.

In a perfect world, every country would share the same structure, but that's definitely not the case.

Just to take a few, in France, they have 2 sub-levels:
  - the region (`Rhône-Alpes`)
  - the department (`Savoie`) :metal:

In USA, they have 1 sub-level (`New-York`).
The United Kingdom, well... it's complicated :smile:

So what if we need to efficiently retrieve the country name for all the points of interest in a single query?

***Bonus: What if we need to count the number of points of interest per country or attached node/level?***

## The MySQL way :trollface:

Let's take as an example how it would look like with `awesome_nested_set`.

### Schema

This is how the structure would look like:

{% highlight sql %}
                                         Table "public.world_nodes"
      Column       |            Type             |                        Modifiers
-------------------+-----------------------------+----------------------------------------------------------
 id                | integer                     | not null default nextval('world_nodes_id_seq'::regclass)
 parent_id         | integer                     |
 name              | character varying(255)      |
 created_at        | timestamp without time zone |
 updated_at        | timestamp without time zone |
 lft               | integer                     |
 rgt               | integer                     |
 depth             | integer                     |
{% endhighlight %}

  - `parent_id` retrieves the closest parent.
  - `depth` identifies how deep we are in the tree.
  - `rgt` and `lft` are there to query the relationship over the nodes.

`rgt` and `lft` are used in a lot of queries like for `ancestors` [here][ancestors], `descendants` [here][descendants] or even `leaf` [here][leaf].

### `JOIN`s vs `CTE`s

Back to our use case.

As a node is only aware about its closest parent, we need some `JOIN`s and for the case of France it would look like this:

{% highlight sql %}
  SELECT countries.name
  FROM point_of_interests
  JOIN world_nodes AS departments ON departments.id = point_of_interests.world_node_id
  JOIN world_nodes AS regions ON regions.id = departments.parent_id
  JOIN world_nodes AS countries ON countries.id = regions.parent_id;
{% endhighlight %}

But as the number of levels is different over the countries, we can't do that.
This is where a `RECURSIVE CTE` would be helpful.

{% highlight sql %}
  WITH RECURSIVE country_ids (point_of_interest_id, country_id, depth) AS (
    SELECT id, world_node_id, 0
    FROM point_of_interests
    WHERE id IN (?, ?, ?, ?, ?)

    UNION ALL

    SELECT country_ids.point_of_interests_id, wn.parent_id, country_ids.depth + 1
    FROM world_nodes wn
    JOIN country_ids ON wn.id = country_ids.country_id
    WHERE wn.parent_id IS NOT NULL
  )

  SELECT point_of_interests.name, countries.name
  FROM
    point_of_interests,
    world_nodes AS countries,
    (
      SELECT DISTINCT ON (point_of_sale_id) point_of_sale_id, country_id
      FROM country_ids
      ORDER BY point_of_sale_id, depth DESC
    ) pos_countries
  WHERE world_nodes.id = pos_countries.country_id
  AND point_of_interests.id = pos_countries.point_of_interest_id;
{% endhighlight %}

The `WITH RECURSIVE` part will create a temporary table with the `point_of_interest_id` and the `country_id` as an output.
We will then be able to use that table as any other table into another query.

That works, but we have to admit that is going to be painful at some point :smile:

### Slow is slow

As seen before, `awesome_nested_set` maintains a `rgt` and `lft` for **every** node.

This has a pretty negative impact when it comes to add more nodes to the tree.
This because we have to rebuild **all** this stuff to make it usable.

## The PostgreSQL way

What could be better than a `ltree` datatype to represent a tree? :trollface:

The gem `ltree_hierarchy` (again wrote by the awesome [Rob Worley][rob-worley] :heart:) has been built to answer that question.

It again shares the same API with `awesome_nested_set`.

Let's see how it works and how it changes the thing up.

### Schema

The structure would now look like this:

{% highlight sql %}
                                         Table "public.world_nodes"
      Column       |            Type             |                        Modifiers
-------------------+-----------------------------+----------------------------------------------------------
 id                | integer                     | not null default nextval('world_nodes_id_seq'::regclass)
 parent_id         | integer                     |
 name              | character varying(255)      |
 tree_path         | ltree                       |
{% endhighlight %}

  - `tree_path`, as its name suggests, will store the whole path of the current node.
  - `parent_id` will only store the closest parent.

`tree_path` changes everything.

### `JOIN`s the return

Back to the use case, here is a query that retrieves all the country names for every point of interest:

{% highlight sql %}
  SELECT countries.name
  FROM point_of_interests
  JOIN world_nodes AS leafs ON leafs.id = point_of_interest.world_node_id
  JOIN world_nodes AS countries ON subpath(leafs.tree_path, 0, 1) = countries.tree_path;
{% endhighlight %}

This is almost equivalent to the query that we would do for France in the previous case, but this time it's compatible with everything.

### Fast is fast

As we don't have to compute and maintain the `rgt`/`lft` columns for every node when we update a single node, it's really faster to populate the tree.

## Benchmarks

Benchs from PR.

## Conclusion

We recently made the switch from `awesome_nested_set` to `ltree_hierarchy` and so far so good.
We were massively caching the tree structure to answer faster to some queries. Every time we were performing an update in the tree we had to rebuild the cache.

The main problem is that we were not able to easily retrieve the informations we needed.

SQL query + Cache together aren't good friends, isn't it?

We came back to the basics and we just got rid of the cache. We were also able to provide some JSON from out of the database to reduce the impact of not relying on a caching layer anymore.

Thanks for reading.

[bridge]: http://www.leadformance.com
[ancestors]: https://github.com/collectiveidea/awesome_nested_set/blob/08d522ad02ad6c0fff922fef9e96ae7a210a1b56/lib/awesome_nested_set/model/relatable.rb#L13-L17
[descendants]: https://github.com/collectiveidea/awesome_nested_set/blob/08d522ad02ad6c0fff922fef9e96ae7a210a1b56/lib/awesome_nested_set/model/relatable.rb#L48-L51
[leaf]: https://github.com/collectiveidea/awesome_nested_set/blob/08d522ad02ad6c0fff922fef9e96ae7a210a1b56/lib/awesome_nested_set/...
[rob-worley]: https://github.com/robworley
