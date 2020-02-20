# More Interesting Examples

Since you now know how to read the basics of an EXPLAIN (ANALYZE) query, let's start both using that knowledge and expand our explanation of the output.

## When your query has a WHERE clause

Let's add a WHERE clause to our query. We will ask for all the ids of all the storms that happened in the home state of the class author:

```sql92
EXPLAIN ANALYZE select event_id from se_details where state = 'NEW YORK';       
```{{execute}}

You should see output something like this:

![Filter Node](basics/explain/assets/03-with-filter.png)

So again we are doing a sequential scan and the rest of the output looks standard. What's new is the second and third lines. We now see that the planner is using a filter because we put in a WHERE clause. It tells us the exact syntax of filter (for example we see that the query planner cast our input string by using *::text* since we didn't explicitly case the string to a type ourselves). The third line tells us how many records were excluded by the filter, in this case we reduced the number of records by quite a bit, filtering out most of them and only returning a relatively small subset.

The fact that we did a sequential scan with a WHERE clause, and most of the rows were eliminated by the filter, indicates that the query planner did not have an index it could use to help make the query more efficient. 

### Adding an index and re-querying
Let's go ahead and add an index and see what happens to the query plan.

```sql92
create index se_details_state_idx on se_details(state);
```{{execute}}
and now let's run our filtered query again

 ```sql92
 EXPLAIN ANALYZE select event_id from se_details where state = 'NEW YORK';       
 ```{{execute}}

Your output should look something like this:

![Filter Node with Index](basics/explain/assets/03-with-filter-index.png)

One of the first things you should notice is that we get an significant reduction in execution time. PostgreSQL took advantage of the new index we just created and by using that index PostgreSQL was able to reduce the execution time of our query.

Another nice part of this query is that now we get to see the tree like nature of the query plan. The terminal node of the tree is in the red rectangle and the
 final node is in the yellow rectangle. The terminal node is also indented to denote it being below the top node.
 
You may be wondering why the query planner had to actually do two steps to carry out this query. Why couldn't it just use the index to look up the right rows and then return them? The query planner actually did consider using an Index Scan to find the rows and then look up each one in the table, but decided that wasn't the most efficient way to get the answer.  Instead,
the chosen plan went through the index and made a note of each matching row in a Bitmap and, once it found all the matches in the index, it then performed a scan of the table by skipping to each position in the bitmap where a matching row was found, read that page, and then returned the result.

There are a few reasons why a Bitmap Heap Scan may be more efficient than an Index Scan:

1. If a lot of the table is going to be returned anyway, a Bitmap Heap Scan will scan the table in a closer-to-sequential method, unlike the random access which would be caused by an Index Scan.
1. Though it was not the case in this query, a Bitmap Heap Scan can use multiple indexes (with a BitmapAnd or BitmapOr) to find the matching rows.

Note that while a Bitmap Scan may be able to keep track of exactly all of the entries in the index it found, if there are a lot of entries then the bitmap will become "lossy" and each entry will encompass more than just the specific rows needed.  When this happen, a "Recheck" is added to the Bitmap Heap Scan, to double-check that the rows being considered match the criteria requested by the user.

If you want to see all the "default" node types you can look in the source [file here])(https://gitlab.com/postgres/postgres/blob/master/src/include/nodes/plannodes.h). We say default because extensions and other add-ons can always bring in new node types.

## A join query 

Let's move on to seeing how the query planner would satisfy a join query. We will build on the last example by adding a WHERE clause to our join as well. We have another table that, if there are fatalities from storm, will provide details on the fatality. Let's join our storm event table against that table:

```sql
EXPLAIN ANALYZE select d.event_id, d.state, f.fatality_age, f.fatality_location
from se_details as d, se_fatalities as f where d.event_id = f.event_id AND d.state = 'NEW YORK';
```{{execute}}
and again your output should look something like this:

![Join Query](basics/explain/assets/03-join.png)

So there are actually 3 nodes in this query. The node in the yellow rectangle (the sequential scan) and the node in the red rectangle (the index scan) are both children of the node in the green rectangle (the nested loop). 
To do the join the query planner has to match the event_id between both tables. You can see that the sequential scan node returns 613 records, the total number records in the se_fatalities table. The query planner chose to do a sequential scan here because it expects to pull back nearly all of the rows in the se_fatalities table and it is a small table.

All of those records are then fed one at a time into the index scan node, first for the index to be used to find a matching row for that event_id, if one exists, and then to test the filter condition of the WHERE clause.  Note that an index across both of those columns (event_id and state) would allow PostgreSQL to perform a directed lookup for exactly the rows matching instead of having to filter rows returned.  This is why the index scan node has many loops. We get an index scan on this node because the query engine can use the index on event_id to find the matching record for the join and then filter based on 'NEW YORK' to meet the required conditions.

> Note: since we are doing 613 loops on this node, the timings we are getting back for actual time are the time **per loop**. Which means this node will take ~ .802 milliseconds to finish, in this example.

## Wrap Up

In this section we got to see filters, different node types, and multi-level trees. You have been introduced to most of the major features or explain analyze output. In the next section we will look at other ways to report explain resuts and another tool to help you visualize your explain results.
