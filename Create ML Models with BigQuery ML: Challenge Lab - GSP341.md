# Create ML Models with BigQuery ML: Challenge Lab - GSP341  

## Task 1 - Create the Initial Model  

```sql
CREATE OR REPLACE MODEL `ecommerce.customer_classification_model`
OPTIONS
  (
  model_type='logistic_reg',
  labels = ['will_buy_on_return_visit']
  ) AS

SELECT
  * EXCEPT(fullVisitorId)
FROM
  -- Features
  (SELECT
      fullVisitorId,
      IFNULL(totals.bounces, 0) AS bounces,
      IFNULL(totals.timeOnSite, 0) AS time_on_site
   FROM
      `data-to-insights.ecommerce.web_analytics`
   WHERE
      totals.newVisits = 1
      AND date BETWEEN '20160801' AND '20170430' -- Train on first 9 months
  ) 
  JOIN
  (SELECT
      fullvisitorid,
      IF(COUNTIF(totals.transactions > 0 AND totals.newVisits IS NULL) > 0, 1, 0) AS will_buy_on_return_visit
   FROM
      `data-to-insights.ecommerce.web_analytics`
   GROUP BY fullvisitorid
  )
  USING (fullVisitorId);
```

## Task 2 - Evaluate the Initial Model
```sql
SELECT
  roc_auc,
  CASE
    WHEN roc_auc > .9 THEN 'good'
    WHEN roc_auc > .8 THEN 'fair'
    WHEN roc_auc > .7 THEN 'decent'
    WHEN roc_auc > .6 THEN 'not great'
  ELSE 'poor' END AS model_quality
FROM
  ML.EVALUATE(MODEL ecommerce.customer_classification_model,  
  (
    SELECT
      * EXCEPT(fullVisitorId)
    FROM
      -- Features
      (SELECT
          fullVisitorId,
          IFNULL(totals.bounces, 0) AS bounces,
          IFNULL(totals.timeOnSite, 0) AS time_on_site
       FROM
          `data-to-insights.ecommerce.web_analytics`
       WHERE
          totals.newVisits = 1
          AND date BETWEEN '20160801' AND '20170430'
      ) 
      JOIN
      (SELECT
          fullvisitorid,
          IF(COUNTIF(totals.transactions > 0 AND totals.newVisits IS NULL) > 0, 1, 0) AS will_buy_on_return_visit
       FROM
          `data-to-insights.ecommerce.web_analytics`
       GROUP BY fullvisitorid
      )
      USING (fullVisitorId)
  ));
```

## Task 3 - Improve Model Performance
### Task 3 - Part 1: Create the Improved Model
```sql
CREATE OR REPLACE MODEL `ecommerce.improved_customer_classification_model`
OPTIONS
  (model_type='logistic_reg', labels = ['will_buy_on_return_visit']) AS

WITH all_visitor_stats AS (
SELECT
  fullvisitorid,
  IF(COUNTIF(totals.transactions > 0 AND totals.newVisits IS NULL) > 0, 1, 0) AS will_buy_on_return_visit
FROM `data-to-insights.ecommerce.web_analytics`
GROUP BY fullvisitorid
)

-- Add new features
SELECT * EXCEPT(unique_session_id) FROM (
  SELECT
      CONCAT(fullvisitorid, CAST(visitId AS STRING)) AS unique_session_id,

      -- Labels
      will_buy_on_return_visit,

      -- Checkout progress
      MAX(CAST(h.eCommerceAction.action_type AS INT64)) AS latest_ecommerce_progress,

      -- User behavior
      IFNULL(totals.bounces, 0) AS bounces,
      IFNULL(totals.timeOnSite, 0) AS time_on_site,
      IFNULL(totals.pageviews, 0) AS pageviews,

      -- Traffic source
      trafficSource.source,
      trafficSource.medium,
      channelGrouping,

      -- Device type
      device.deviceCategory,

      -- Geographic location
      IFNULL(geoNetwork.country, "") AS country

  FROM `data-to-insights.ecommerce.web_analytics`,
     UNNEST(hits) AS h

  JOIN all_visitor_stats USING(fullvisitorid)

  WHERE
    totals.newVisits = 1
    AND date BETWEEN '20160801' AND '20170430' -- Train on 9 months

  GROUP BY
    unique_session_id,
    will_buy_on_return_visit,
    bounces,
    time_on_site,
    totals.pageviews,
    trafficSource.source,
    trafficSource.medium,
    channelGrouping,
    device.deviceCategory,
    country
);
```
### Task 3 - Part 2: Evaluate the Improved Model
```sql
SELECT
  roc_auc,
  CASE
    WHEN roc_auc > .9 THEN 'good'
    WHEN roc_auc > .8 THEN 'fair'
    WHEN roc_auc > .7 THEN 'decent'
    WHEN roc_auc > .6 THEN 'not great'
  ELSE 'poor' END AS model_quality
FROM
  ML.EVALUATE(MODEL ecommerce.improved_customer_classification_model,  
  (
    WITH all_visitor_stats AS (
    SELECT
      fullvisitorid,
      IF(COUNTIF(totals.transactions > 0 AND totals.newVisits IS NULL) > 0, 1, 0) AS will_buy_on_return_visit
    FROM `data-to-insights.ecommerce.web_analytics`
    GROUP BY fullvisitorid
    )

    -- Add new features
    SELECT * EXCEPT(unique_session_id) FROM (
      SELECT
          CONCAT(fullvisitorid, CAST(visitId AS STRING)) AS unique_session_id,
          will_buy_on_return_visit,
          MAX(CAST(h.eCommerceAction.action_type AS INT64)) AS latest_ecommerce_progress,
          IFNULL(totals.bounces, 0) AS bounces,
          IFNULL(totals.timeOnSite, 0) AS time_on_site,
          totals.pageviews,
          trafficSource.source,
          trafficSource.medium,
          channelGrouping,
          device.deviceCategory,
          IFNULL(geoNetwork.country, "") AS country
      FROM `data-to-insights.ecommerce.web_analytics`,
         UNNEST(hits) AS h

      JOIN all_visitor_stats USING(fullvisitorid)

      WHERE
        totals.newVisits = 1
        AND date BETWEEN '20170501' AND '20170630' -- Evaluate on 2 months

      GROUP BY
        unique_session_id,
        will_buy_on_return_visit,
        bounces,
        time_on_site,
        totals.pageviews,
        trafficSource.source,
        trafficSource.medium,
        channelGrouping,
        device.deviceCategory,
        country
    )
  ));
```
## Task 4 - Final Model and Prediction
### Task 4 - Part 1: Create the Final Model
```sql
CREATE OR REPLACE MODEL `ecommerce.finalized_classification_model`
OPTIONS
  (model_type="logistic_reg", labels = ["will_buy_on_return_visit"]) AS

WITH all_visitor_stats AS (
SELECT
  fullvisitorid,
  IF(COUNTIF(totals.transactions > 0 AND totals.newVisits IS NULL) > 0, 1, 0) AS will_buy_on_return_visit
FROM `data-to-insights.ecommerce.web_analytics`
GROUP BY fullvisitorid
)

SELECT * EXCEPT(unique_session_id) FROM (
  SELECT
      CONCAT(fullvisitorid, CAST(visitId AS STRING)) AS unique_session_id,
      will_buy_on_return_visit,
      MAX(CAST(h.eCommerceAction.action_type AS INT64)) AS latest_ecommerce_progress,
      IFNULL(totals.bounces, 0) AS bounces,
      IFNULL(totals.timeOnSite, 0) AS time_on_site,
      IFNULL(totals.pageviews, 0) AS pageviews,
      trafficSource.source,
      trafficSource.medium,
      channelGrouping,
      device.deviceCategory,
      IFNULL(geoNetwork.country, "") AS country
  FROM `data-to-insights.ecommerce.web_analytics`,
     UNNEST(hits) AS h
  JOIN all_visitor_stats USING(fullvisitorid)
  WHERE totals.newVisits = 1 AND date BETWEEN "20160801" AND "20170430"
  GROUP BY unique_session_id, will_buy_on_return_visit, bounces, time_on_site, totals.pageviews, trafficSource.source, trafficSource.medium, channelGrouping, device.deviceCategory, country
);
```

### Task 4 - Part 2: Make Predictions
```sql
SELECT
*
FROM
  ml.PREDICT(MODEL `ecommerce.finalized_classification_model`,
   (

WITH all_visitor_stats AS (
SELECT
  fullvisitorid,
  IF(COUNTIF(totals.transactions > 0 AND totals.newVisits IS NULL) > 0, 1, 0) AS will_buy_on_return_visit
  FROM `data-to-insights.ecommerce.web_analytics`
  GROUP BY fullvisitorid
)

  SELECT
      CONCAT(fullvisitorid, '-',CAST(visitId AS STRING)) AS unique_session_id,

      # labels
      will_buy_on_return_visit,

      MAX(CAST(h.eCommerceAction.action_type AS INT64)) AS latest_ecommerce_progress,

      # behavior on the site
      IFNULL(totals.bounces, 0) AS bounces,
      IFNULL(totals.timeOnSite, 0) AS time_on_site,
      totals.pageviews,

      # where the visitor came from
      trafficSource.source,
      trafficSource.medium,
      channelGrouping,

      # mobile or desktop
      device.deviceCategory,

      # geographic
      IFNULL(geoNetwork.country, "") AS country

  FROM `data-to-insights.ecommerce.web_analytics`,
     UNNEST(hits) AS h

    JOIN all_visitor_stats USING(fullvisitorid)

  WHERE
    # only predict for new visits
    totals.newVisits = 1
    AND date BETWEEN '20170331' AND '20170430' # test 1 month

  GROUP BY
  unique_session_id,
  will_buy_on_return_visit,
  bounces,
  time_on_site,
  totals.pageviews,
  trafficSource.source,
  trafficSource.medium,
  channelGrouping,
  device.deviceCategory,
  country
)

)

ORDER BY
  predicted_will_buy_on_return_visit DESC;
```
---
© Mateus Silva | 2025
