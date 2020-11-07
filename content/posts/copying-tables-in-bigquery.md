---
draft: false
date: 2020-10-21T15:23:06-04:00
title: "Copying Google Analytics Tables in BigQuery"
description: "If you need to copy multiple date-sharded tables from one dataset to another in BigQuery, here's a script to automate it."
tags: [BigQuery, Google Analytics, SQL]
categories: [BigQuery]
featured_image: "../../images/copy_dataset.png"
summary: "If you need to copy multiple date-sharded tables from one dataset to another in BigQuery, here's a script to automate it."
---
If you've ever needed to copy tables from one project or dataset to another in BigQuery, the below script may be a good option for you.

There are many options for copying tables in BigQuery, ranging from completely non-technical (point and click in the BigQuery UI) to more technical (command line and/or Python scripts).

I won't go into the pros and cons of each option, or why you would want to use one over the other. For a good explanation of that, [this article from Vamsi Namburu](https://medium.com/@vmnamburu/how-to-efficiently-copy-data-between-bigquery-environments-part-1-475df9c6cac6) does an excellent job at covering those details.

Below is the code I used to copy a range of daily tables from one dataset to another (can be the same project or a different project, as long as you have the right roles/permissions in each project). If you want to copy every single table, this is **not** a good option. In that case, you can just copy the entire dataset using the BQ UI, like pictured below:

{{< figure src="../../images/copy_dataset.png" alt="Copy Dataset in the BigQuery user interface." caption="Copying every table in a dataset into another project/dataset is both quick, free, and easy in the interface. Just click on 'Copy Dataset' and specify the destination project and dataset.">}}

In my particular use case, however, I needed to copy just a specific range of tables - not the entire dataset. For that, I wrote a script using [data definition language (DDL)](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-definition-language). I've done my best to generalize the script to be as plug-and-play as possible. All you have to do is update the first few lines to specify *your* source and destination projects and datasets, as well as the start and end dates for the Google Analytics daily tables you want to copy.

{{<highlight sql>}}
--------------------------------------------------------------
-- Copy multiple date-sharded GA tables to new project/dataset
--------------------------------------------------------------

-- Update the variables below to specify your source project and dataset (where
-- you want to copy the tables from), the destination project/dataset (where you
-- want to copy the tables to), and the first and last date. The first and last
-- date specify the date range (inclusive) of tables you want to copy.
DECLARE source_project STRING DEFAULT "PROJECT_NAME";
DECLARE source_dataset STRING DEFAULT "DATASET_ID";
DECLARE destination_project STRING DEFAULT "PROJECT_NAME";
DECLARE destination_dataset STRING DEFAULT "DATASET_ID";
DECLARE start_date INT64 DEFAULT 20200625;
DECLARE end_date INT64 DEFAULT 20200627;

-- DO NOT TOUCH ANYTHING BELOW THIS LINE --
DECLARE source_ga_tables_array ARRAY<STRING>;
DECLARE destination_ga_tables_array ARRAY<STRING>;
DECLARE source_ga_table STRING;
DECLARE destination_ga_table STRING;
DECLARE i INT64 DEFAULT 0;

SET source_ga_tables_array = (
  WITH date_array AS (
    SELECT
      GENERATE_ARRAY(start_date, end_date) AS dates
  )
  SELECT
    ARRAY_AGG(
        CONCAT(source_project, '.', source_dataset, '.ga_sessions_', date)
    )
  FROM
    date_array, UNNEST(dates) AS date
);

SET destination_ga_tables_array = (
  SELECT
    ARRAY_AGG(
        REGEXP_REPLACE(source_ga_tables_array,
            CONCAT(source_project, '.', source_dataset),
            CONCAT(destination_project, '.', destination_dataset)
        )
    ) AS table_name
  FROM
    UNNEST(source_ga_tables_array) AS source_ga_tables_array
);

WHILE i < ARRAY_LENGTH(source_ga_tables_array) DO
  SET source_ga_table = (
    SELECT
      CONCAT('`', source_ga_tables_array[OFFSET(i)], '`')
  );
  SET destination_ga_table = (
    SELECT
      CONCAT('`', destination_ga_tables_array[OFFSET(i)], '`')
  );
  EXECUTE IMMEDIATE format("""
  CREATE TABLE
    %s
  AS
  SELECT
    *
  FROM
    %s
  """, destination_ga_table, source_ga_table);
  SET i = i + 1;
END WHILE;
{{</highlight>}}
