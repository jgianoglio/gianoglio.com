---
draft: false
date: 2020-10-20T21:59:14-04:00
title: "Updating Google Analytics Hit-Level Custom Dimensions in BigQuery"
description: "Updating hit-level custom dimension information in BigQuery is not as simple as it sounds. Here's the code to do it."
tags: [BigQuery, Google Analytics, SQL]
categories: [BigQuery]
featured_image: ""
summary: "Updating hit-level custom dimension information in BigQuery is not as simple as it sounds. Here's the code to do it."
---

I recently had a client with a problem. There was an issue with their Google Tag Manager implementation that prevented the collection of certain hit-level custom dimensions to Google Analytics.

Fortunately, they had GA360 (with the export to BigQuery already enabled), and the missing custom dimension information was able to be replicated based on the page path. In other words, I could create a lookup table table based on the page path of the hit to grab the corresponding custom dimension value (in this particular case, it was a custom dimension named Site ID).

It seemed simple enough. But it turned out to be rather complex. (If you know of a simpler soultion, please let me know! I do sometimes have a tendency to find overly complicated solutions.)

The complexity stems from the fact that these are hit-level custom dimensions, requiring me to have to `UNNEST` two levels deep (`UNNEST` the hits, then `UNNEST` the custom dimensions).

Also, I wanted to use the [BigQuery data manipulation language (DML)](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-manipulation-language) to do the updates - mainly because I haven't had a chance to really use it before and also because it seemed appropriate for the task.

With DML, you can update, insert, and delete data from your tables. I needed to update the tables, but specifically, I needed to update the `hits` field. Due to the nested nature of the Google Analytics data in BigQuery, the `hits` field is a single column (comprised of 176 other "sub-fields"!).

I thought I'd be able to update just part of the hit (specifically, the value for custom dimension index 1). However, almost every method I tried ended in the following error:

```Error: Correlated subqueries that reference other tables are not supported unless they can be de-correlated, such as by transforming them into an efficient JOIN.```

After spending too many hours on [Stack Overflow](https://stackoverflow.com/questions/45040472/avoid-correlated-subqueries-error-in-bigquery), reading [BigQuery documentation on `ARRAY` and `STRUCT`](https://cloud.google.com/bigquery/docs/reference/standard-sql/arrays), and experimenting with different queries, I came to the realization that I'd just have to `UNNEST` the `hits` and the `hits.customDimensions`, join on the lookup table table, then recreate the the entire `hits` field for each session. Fun.

For those that want to follow along, below is the SQL to generate a couple rows to mimic Google Analytics data in BQ. This is a simpler dataset to work with, which I found helps when trying to tackle larger problems.

{{< highlight sql >}}
-- Create a table to mimic a session in GA
SELECT
  '1234' AS clientId,
  12345678 AS visitStartTime,
  STRUCT(1 AS visits,
    2 AS hits) AS totals,
  [ STRUCT(1 AS hitNumber,
    STRUCT('/careers/' AS pagePath,
      'example.com' AS hostname) AS page,
    [STRUCT(1 AS index,
      'one' AS value),
    STRUCT(2 AS index,
      'two' AS value)] AS customDimensions),
  STRUCT(2 AS hitNumber,
    STRUCT('/insights/' AS pagePath,
      'example.com' AS hostname) AS page,
    [STRUCT(1 AS index,
      '(not set)' AS value),
    STRUCT(2 AS index,
      'two-more' AS value)] AS customDimensions) ] AS hits
UNION ALL
SELECT
  '9877' AS clientId,
  23456789 AS visitStartTime,
  STRUCT(1 AS visits,
    3 AS hits) AS totals,
  [ STRUCT(1 AS hitNumber,
    STRUCT('/' AS pagePath,
      'example2.com' AS hostname) AS page,
    [STRUCT(1 AS index,
      'one' AS value),
    STRUCT(2 AS index,
      'two' AS value)] AS customDimensions),
  STRUCT(2 AS hitNumber,
    STRUCT('/blog/' AS pagePath,
      'example2.com' AS hostname) AS page,
    [STRUCT(2 AS index,
      'dos' AS value)] AS customDimensions),
  STRUCT(3 AS hitNumber,
    STRUCT('/blog/post/' AS pagePath,
      'example2.com' AS hostname) AS page,
    [STRUCT(1 AS index,
      'un' AS value),
    STRUCT(2 AS index,
      'deux' AS value)] AS customDimensions) ] AS hits
{{</highlight >}}

{{< figure src="../../images/mimic-sessions.png" alt="BigQuery table with two rows that mimic the nested nature of Google Analytics data." caption="The above SQL will ouput two rows, with nested hits and nested custom dimensions.">}}

Notice in the screenshot above how the second hit of the first row (with a `hits.page.pagePath` of */insights/*) has a value of *(not set)* for custom dimension index 1. The goal is to update that value from *(not set)* to a value based on a separate lookup table table.

The following SQL outputs our sample lookup table table:

{{<highlight sql>}}
-- Create a table to mimic a lookup table table with pagePath and site ID
SELECT 
  '/careers/' AS pagePath,
  'C-3PO' AS site_id
UNION ALL
SELECT
  '/' AS pagePath,
  'R2-D2' AS site_id
UNION ALL
SELECT
  '/insights/' AS pagePath,
  '867-5309' AS site_id
{{</highlight >}}

{{< figure src="../../images/lookup-table.png" alt="Lookup table with page path and site ID." caption="This is our lookup table table. Whenever custom dimension index 1 has a value of *(not set)*, we'll replace that with the value in this table, based on the page path of the hit.">}}

Given the sample table generated by the first bit of SQL, we see that the second hit of the first session has a value of *(not set)* for custom dimension index 1 and a `hits.page.pagePath` of */insights/*. So we'd join with the lookup table table where `hits.page.pagePath` = `pagePath`, and replace *(not set)* with *867-5309* - the value of the *site_id* field in the lookup table table.

To follow along with this example, run the queries above and save the results as tables. I named them `ga_sessions_sample` and `site_id_lookup`, which I'll reference throughout the rest of this article.

## Time to Transform

Now that we've set up some sample tables, it's time to get to work. There are basically 2 steps that need to happen:

**Step 1:** Create an "updated hits" table with `clientId`, `visitStartTime`, and the entire `hits` field (with the custom dimension value updated). This will be the table that we use to replace the `hits` field in the original `ga_sessions_*` tables.

**Step 2:** Run through the DML to update the `hits` field in the `ga_sessions_*` tables with the `hits` in our updated table.

### Step 1
This query will update every instance of custom dimension Index 1 where the value is *(not set)*, using the lookup table table to replace *(not set)* with the `site_id` value. Run this query and save the output to a table (I named this table *updated_hits*)

{{<highlight sql>}}
-- t1 UNNESTs the table and joins on the lookup table to get the correct value for Site ID
WITH t1 AS (
  SELECT
    clientId
    , visitStartTime
    , hits
    , cd.index
    , IF(cd.index = 1 AND cd.value = '(not set)', site_id, cd.value) AS value
  FROM
    `PROJECT.DATASET.ga_sessions_sample` AS sessions, UNNEST(hits) AS hits, UNNEST(hits.customDimensions) AS cd
      LEFT JOIN `PROJECT.DATASET.site_id_lookup` AS site_id_lookup
      ON hits.page.pagePath = site_id_lookup.pagePath
),

-- t2 puts the CD index and value into an array of structs, and selects all other fields
t2 AS (
  SELECT
    * EXCEPT(hits, index, value)    
    , hits.hitNumber
    , ANY_VALUE(hits.page) AS page
    , ARRAY_AGG(STRUCT(index, value)) AS customDimensions
  FROM t1
  GROUP BY clientId, visitStartTime, hitNumber
  ORDER BY clientId, visitStartTime, hitNumber ASC
)

-- forms all the hit fields into an array of structs
  SELECT
    clientId
    , visitStartTime
    , ARRAY_AGG(STRUCT(hitNumber, page, customDimensions)) AS hits
  FROM
    t2
  GROUP BY
    clientId, visitStartTime
{{</highlight>}}

{{< figure src="../../images/updated_hits.png" alt="Table with clientId, visitStartTime, and hits." caption="This is our updated hits table. It includes just the clientId, visitStartTime, and hits fields (with the updated custom dimension value).">}}

Now that you have a table with the `clientId` and `visitStartTime` (which we use to uniquely identify a session), along with the updated hits field for that session (which includes the updated custom dimension value), all that's left is a simple DML script to update the original `ga_sessions_sample` table:

{{<highlight sql>}}
UPDATE
  `PROJECT.DATASET.ga_sessions_sample` AS a
SET a.hits = b.hits
FROM
  `PROJECT.DATASET.updated_hits` AS b
WHERE
  a.clientId = b.clientId AND a.visitStartTime = b.visitStartTime
{{</highlight>}}

After running the above script, check your `ga_sessions_sample` table again. You should see something like below:

{{< figure src="../../images/updated_table.png" alt="Updated ga_sessions_sample table." caption="This is our updated ga_sessions_sample table. It includes all of the original fields, with the updated custom dimension value outlined in red).">}}


## Play Time's Over
Now that we have the proof of concept working on a sample dataset, it's time to apply the same principles (and make some slight modifications) to work with multiple days of real GA data.

Here's the SQL to update and recreate the entire hits record:

{{<highlight sql>}}
-- t1 UNNESTs the table and joins on the lookup table to get the correct value for SiteID
WITH t1 AS (
  SELECT
    clientId
    , visitStartTime
    , hits
    , cd.index
    , CASE 
        WHEN cd.index = 1 AND cd.value = '(not set)' THEN site_id
        ELSE cd.value
    END AS value
  FROM
    `PROJECT.DATASET.ga_sessions_*` AS sessions, UNNEST(hits) AS hits, UNNEST(hits.customDimensions) AS cd
      LEFT JOIN `PROJECT.DATASET.site_id_lookup` AS lookup_table
      ON lookup_table.pagePath = hits.page.pagePath
  WHERE
    _TABLE_SUFFIX BETWEEN '20201001' AND '20201007'
),

-- t2 puts the CD index and value into an array of structs, and selects all other fields
t2 AS (
  SELECT
    * EXCEPT(hits, index, value)
    , hits.* EXCEPT(page
        , customDimensions 
        , transaction
        , item
        , contentInfo
        , appInfo
        , exceptionInfo
        , eventInfo
        , product
        , promotion
        , promotionActionInfo
        , refund
        , eCommerceAction
        , experiment
        , publisher 
        , customVariables
        , customMetrics
        , social
        , latencyTracking
        , sourcePropertyInfo
        , contentGroup
        , publisher_infos)
    , ANY_VALUE(hits.page) AS page
    , ANY_VALUE(hits.transaction) AS transaction
    , ANY_VALUE(hits.item) AS item
    , ANY_VALUE(hits.contentInfo) AS contentInfo
    , ANY_VALUE(hits.appInfo) AS appInfo
    , ANY_VALUE(hits.exceptionInfo) AS exceptionInfo
    , ANY_VALUE(hits.eventInfo) AS eventInfo
    , ANY_VALUE(hits.product) AS product
    , ANY_VALUE(hits.promotion) AS promotion
    , ANY_VALUE(hits.promotionActionInfo) AS promotionActionInfo
    , ANY_VALUE(hits.refund) AS refund
    , ANY_VALUE(hits.ecommerceAction) AS ecommerceAction
    , ANY_VALUE(hits.experiment) AS experiment
    , ANY_VALUE(hits.publisher) AS publisher
    , ANY_VALUE(hits.customVariables) AS customVariables
    , ANY_VALUE(hits.customMetrics) AS customMetrics
    , ANY_VALUE(hits.social) AS social
    , ANY_VALUE(hits.latencyTracking) AS latencyTracking
    , ANY_VALUE(hits.sourcePropertyInfo) AS sourcePropertyInfo
    , ANY_VALUE(hits.contentGroup) AS contentGroup
    , ANY_VALUE(hits.publisher_infos) AS publisher_infos
    , ARRAY_AGG(STRUCT(index, value)) AS customDimensions
  FROM t1
  GROUP BY 1,2,3,4,5,6,7,8,9,10,11,12,13
  ORDER BY clientId, visitStartTime, hitNumber ASC
)

-- forms all the hit fields into an array of structs
  SELECT
    clientId
    , visitStartTime
    , ARRAY_AGG(STRUCT(
        hitNumber
        , time
        , hour
        , minute
        , isSecure
        , isInteraction
        , isEntrance
        , isExit
        , referer
        , page
        , transaction
        , item
        , contentInfo
        , appInfo
        , exceptionInfo
        , eventInfo
        , product
        , promotion
        , promotionActionInfo
        , refund
        , ecommerceAction
        , experiment
        , publisher 
        , customVariables
        , customDimensions
        , customMetrics
        , type
        , social
        , latencyTracking
        , sourcePropertyInfo
        , contentGroup
        , dataSource
        , publisher_infos)) AS hits
  FROM
    t2
  GROUP BY
    clientId, visitStartTime
{{</highlight>}}

114 lines of SQL isn't bad, considering there are 176 different fields all nested within the hits record.

Running the above script will output the "updated_hits," with the `clientID`, `visitStartTime`, and (updated) `hits` fields. This is what we did in Step 1 above. Again, you'll want to save the output of this query to a table (I named it *updated_hits*).

Unfortunately, if your date range is too long or if you have a lot of data, the above query will produce the following error:
{{< figure src="../../images/resources_exceeded_error.png" alt="Resources exceeded error message." caption="Uh oh - looks like we got a little crazy with our query and BQ can't handle it!">}}

This is the result of several computationally expensive operations, starting with the multiple `UNNEST`s in the first [common table expression (CTE)](https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax#with_clause), followed by horrendous 1-2 punch of a 13-field `GROUP BY` and 3-field `ORDER BY` - ouch!

I haven't found an elegant solution around this yet, and I had a bit of a time crunch to get this out for the client. So I'm somewhat ashamed to admit that I just brute-forced my way through this by shortening the date range (to no more than 3 days as a time) and kept running this query over and over again (for about an hour and a half). I was appending the results to a single "updated hits" table each time. I'm not proud.

Fortunately, once I suffered through the manual updates, the rest was easily automated. I just repurposed the DML statement from above and sprinkled in some variables and a `WHILE` loop. This way, it loops through each `ga_sessions_*` table and replaces the `hits` field from the updated table.

{{<highlight sql>}}
-- Update the first 7 variables to reflect your project, dataset, and table names and the
-- start and end dates of the tables you want to update.
DECLARE original_ga_sessions_project STRING DEFAULT "PROJECT";
DECLARE original_ga_sessions_dataset STRING DEFAULT "DATASET";
DECLARE updated_table_project STRING DEFAULT "PROJECT";
DECLARE updated_table_dataset STRING DEFAULT "DATASET";
DECLARE updated_table STRING DEFAULT "TABLE_NAME";
DECLARE first_date INT64 DEFAULT 20200625;
DECLARE last_date INT64 DEFAULT 20200627;

-- DO NOT TOUCH ANYTHING BELOW THIS LINE --
DECLARE original_ga_tables_array ARRAY<STRING>;
DECLARE original_ga_table STRING;
DECLARE updated_hits STRING;
DECLARE i INT64 DEFAULT 0;

SET updated_hits = (
  SELECT
    CONCAT('`', updated_table_project, '.', updated_table_dataset, '.', updated_table, '`')
);

SET original_ga_tables_array = (
  WITH date_array AS (
    SELECT
      GENERATE_ARRAY(first_date, last_date) AS dates
  )
  SELECT
    ARRAY_AGG(CONCAT(original_ga_sessions_project, '.', original_ga_sessions_dataset, '.ga_sessions_', date))
  FROM
    date_array, UNNEST(dates) AS date
);

WHILE i < ARRAY_LENGTH(original_ga_tables_array) DO
  SET original_ga_table = (
    SELECT
      CONCAT('`', original_ga_tables_array[OFFSET(i)], '`')
  );
  
  EXECUTE IMMEDIATE format("""
  UPDATE
    %s AS a
  SET a.hits = b.hits
  FROM
    %s AS b
  WHERE
    a.clientId = b.clientId AND a.visitStartTime = b.visitStartTime
  """, original_ga_table, updated_hits);
  SET i = i + 1;
END WHILE;
{{</highlight>}}

All you have to do is update the first 7 variables in the above script with your specific project, dataset, and table names, along with the start and end dates of the tables you want to update. Just be ready to wait a while for this script to run if you have a lot of tables to update. In my particular example, it took 1 hour and 3 minutes to update about 3 months' worth of tables.

{{< figure src="../../images/query_processing_time.png" alt="BigQuery query processing time details." caption="My new record for longest time to process a query - 1 hour and 3 minutes!">}}

## But Jim, what about ...
I know, I can already hear the question "I need to update a hit-level custom dimension that isn't included at all in the data - how do I do that?" That's a great question. When I worked through the client problem that inspired this post, I was fortunate (I think) that if the custom dimension didn't have a value in the dataLayer, they were falling back to manually setting the value to *(not set)*.

This makes the update fairly straight forward:

`IF(cd.index = 1 AND cd.value = '(not set)', site_id, cd.value) AS value`

But if nothing at all is being passed, it gets tricky. I haven't quite thought through that scenario, but if you're interested in a solution and willing to buy me a drink at the next in-person conference once the plague lifts, let me know and I'll figure it out for you.