# Funnels
## The aim of the project
To build sales funnel charts from Turing Data Analytics database and analyze them.
## Tools used
Big Query, Google Spreadsheets.
## Data Processing
Funnel analysis of TOP 3 countries by overall number of events from raw_events table.

``` sql
-- Aggregation for Funnel analysis. Data is taken from Turing Data Analytics database, raw_events table
-- Deduplication
WITH
  dedup AS (
  SELECT
    user_pseudo_id,
    event_name,
    MIN(event_timestamp) AS first_event_timestamp
  FROM
    tc-da-1.turing_data_analytics.raw_events
  GROUP BY
    user_pseudo_id, event_name),

  dedup_table AS (
  SELECT
    whole.*
  FROM
    tc-da-1.turing_data_analytics.raw_events AS whole
  LEFT JOIN
    dedup
  ON
    dedup.user_pseudo_id = whole.user_pseudo_id AND
    dedup.event_name = whole.event_name AND
    dedup.first_event_timestamp = event_timestamp
  WHERE
    dedup.first_event_timestamp  IS NOT NULL),

-- Top 3 countries by their overall number of events
  top3_countries AS (
  SELECT
    country
  FROM
    dedup_table
  GROUP BY
    country
  ORDER BY
    COUNT(*) DESC
  LIMIT 3),

-- Events that are used in the funnel
  funnel_events AS (
  SELECT 1 AS event_order, 'add_to_cart' AS event_name UNION ALL
  SELECT 2, 'add_shipping_info' UNION ALL  
  SELECT 3, 'add_payment_info' UNION ALL
  SELECT 4, 'purchase'),

-- Sales funnel from adding to cart till purchase
  sales_funnel AS (
  SELECT
    funnel_events.event_order,
    funnel_events.event_name,
    COUNTIF((SELECT * FROM top3_countries LIMIT 1) = dedup_table.country) AS first_country,
    COUNTIF((SELECT * FROM top3_countries LIMIT 1 OFFSET 1) = dedup_table.country) AS second_country,
    COUNTIF((SELECT * FROM top3_countries LIMIT 1 OFFSET 2) = dedup_table.country) AS third_country
  FROM
    dedup_table
  LEFT JOIN
    funnel_events
  USING
    (event_name)
  WHERE
    funnel_events.event_name IS NOT NULL
  GROUP BY
    funnel_events.event_order,
    funnel_events.event_name)

-- Percentage drop off values
SELECT
  *,
  ROUND (100 * (first_country + second_country + third_country) /
  (SELECT (first_country + second_country + third_country) FROM sales_funnel WHERE event_order = 1), 2) AS full_perc,
  ROUND (100 * first_country /
  (SELECT first_country FROM sales_funnel WHERE event_order = 1), 2) AS first_country_perc,
  ROUND (100 * second_country /
  (SELECT second_country FROM sales_funnel WHERE event_order = 1), 2) AS second_country_perc,
  ROUND (100 * third_country /
  (SELECT third_country FROM sales_funnel WHERE event_order = 1), 2) AS third_country_perc
FROM
  sales_funnel
ORDER BY
  event_order;
```

## Visualizations and Insights
[Visualizations and Insights are available here](https://docs.google.com/spreadsheets/d/1S_OoYO1xWU8ElVixFgr6oKLT3xxdSOBW7H-imnCdQ0U/edit?usp=sharing)

The only one SQL query is provided above because the others are very similar, the logic of it is almost the same.
