# [SQL] E-Commerce Website Performance Analysis
## **I. INTRODUCTION**
In this project, **_business performance of an e-commerce website_** was analyzed using **_SQL_** in **_Google BigQuery_**, with some advanced SQL techniques like **_window function_** and **_aggregate function_**. The analysis covered critical business aspects such as **_website traffic, customer behavior patterns, revenue generation, sales performance,_** and **_conversion optimization_**. The **_insights_** derived from this work provided **_support to the Sales and Marketing teams_**, enabling them to make informed, data-driven decisions.
## **II. DATASET**
- The e-commerce dataset is stored in a public Google BigQuery dataset under ID: `bigquery-public-data.google_analytics_sample.ga_sessions`.
  
- Table Schema: https://support.google.com/analytics/answer/3437719?hl=en
  | Field Name|	Data Type	|Description|
  | --- | :---: | --- |
  | fullVisitorId	| STRING	| The unique visitor ID.|
  | date	| STRING	| The date of the session in YYYYMMDD format.| 
  | totals	| RECORD	| This section contains aggregate values across the session.| 
  | totals.bounces	| INTEGER	| Total bounces (for convenience). For a bounced session, the value is 1, otherwise it is null.| 
  | totals.hits	| INTEGER	| Total number of hits within the session.| 
  | totals.pageviews	| INTEGER	| Total number of pageviews within the session.| 
  | totals.visits	| INTEGER	| The number of sessions (for convenience). This value is 1 for sessions with interaction events. The value is null if there are no interaction events in the session.| 
  | totals.transactions	| INTEGER	| Total number of ecommerce transactions within the session.| 
  | trafficSource.source	| STRING	| The source of the traffic source. Could be the name of the search engine, the referring hostname, or a value of the utm_source URL parameter.| 
  | hits	| RECORD	| This row and nested fields are populated for any and all types of hits.| 
  | hits.eCommerceAction	| RECORD	| This section contains all of the ecommerce hits that occurred during the session. This is a repeated field and has an entry for each hit that was collected.| 
  | hits.eCommerceAction.action_type	| STRING	| The action type. Click through of product lists = 1, Product detail views = 2, Add product(s) to cart = 3, Remove product(s) from cart = 4, Check out = 5, Completed purchase = 6, Refund of purchase = 7, Checkout options = 8, Unknown = 0. Usually this action type applies to all the products in a hit, with the following exception: when hits.product.isImpression = TRUE, the corresponding product is a product impression that is seen while the product action is taking place (i.e., a "product in list view").| 
  | hits.product	| RECORD	| This row and nested fields will be populated for each hit that contains Enhanced Ecommerce PRODUCT data.| 
  | hits.product.productQuantity	| INTEGER	| The quantity of the product purchased.| 
  | hits.product.productRevenue	| INTEGER	| The revenue of the product, expressed as the value passed to Analytics multiplied by 10^6 (e.g., 2.40 would be given as 2400000).| 
  | hits.product.productSKU	| STRING	| Product SKU.| 
  | hits.product.v2ProductName	| STRING	| Product Name.| 
  | fullVisitorId	| STRING	| The unique visitor ID.|
## **III. KEY AREAS TO ANALYZE**
  - **_Website Traffic & User Behavior:_** Analyze traffic patterns (`total visits, pageviews`), user engagement (`bounce rate`), and how different types of users (purchasers vs. non-purchasers) behave in terms of pageviews.

  - **_Sales Performance & Revenue Analysis:_** Evaluate the `revenue generated` from different traffic sources over time (by week and month).
    
  - **_Product Performance & Conversion Optimization_**: Analyze `related products purchased by customers` (to identify product affinity and potential cross-selling opportunities) and calculate `cohort map` from product view to addtocart to purchasec (to identify conversion rates and shopping behavior).
## **IV. EXPLORE THE DATASET**
### **Query 1: Calculate total visit, pageview, transaction for Jan, Feb and March 2017**
```SQL
SELECT FORMAT_DATE('%Y%m',PARSE_DATE('%Y%m%d',date)) month
      ,SUM(totals.visits) visits
      ,SUM(totals.pageviews) pageviews
      ,SUM(totals.transactions) transactions
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
WHERE _table_suffix BETWEEN '0101' AND '0331' 
GROUP BY 1
ORDER BY 1;
```
| month	| visits	| pageviews	| transactions| 
| --- | --- | --- | --- |
|201701	|64694	|257708	|713|
| 201702	| 62192	| 233373	| 733| 
| 201703	| 69931	| 259522	| 993| 

:arrow_right: In March 2017, there was a notable **_improvement across all key metrics_** (visits, pageviews, and transactions) compared to January and February. This increase could indicate either **_improved conversion rates_** or **_influence of seasonal factors_** only.

### **Query 2: Bounce rate per traffic source in July 2017**
- Bounce_rate = num_bounce/total_visit
```SQL
SELECT trafficSource.`source` AS source
      ,SUM(totals.visits) AS total_visits
      ,COUNT(totals.bounces) AS total_no_of_bounces
      ,ROUND(COUNT(totals.bounces)/SUM(totals.visits)*100.0,3) AS bounce_rate
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`
GROUP BY 1
ORDER BY 2 DESC;
```
| source	| total_visits	| total_no_of_bounces	| bounce_rate| 
| --- | --- | --- | --- |
| google	| 38400	| 19798	| 51.557| 
| (direct)	| 19891	| 8606	| 43.266| 
| youtube.com	| 6351	| 4238	| 66.73| 
| analytics.google.com	| 1972	| 1064	| 53.955| 
| Partners	| 1788	| 936	| 52.349| 
| m.facebook.com	| 669	| 430	| 64.275| 
| google.com	| 368	| 183	| 49.728| 
| dfa	| 302	| 124	| 41.06| 
| sites.google.com	| 230	| 97	| 42.174| 
| facebook.com	| 191	| 102	| 53.403| 
| reddit.com	| 189	| 54	| 28.571| 
| qiita.com	| 146	| 72	| 49.315| 

:arrow_right: **_Google_** generates the **_highest traffic_**, yet it also experiences a **_high bounce rate_**. Both **_YouTube and Facebook_** exhibit the **_highest bounce rates_**, whereas other sources of traffic such as direct,dfa,reddit,etc. demonstrates stronger user engagement. A **_targeted marketing strategy_** should be implemented to **_reduce bounce rates_**, particularly **_from high-bounce sources_** such as Google, YouTube, and Facebook.

### **Query 3: Revenue by traffic source by week, by month in June 2017**
```SQL
WITH 
month_data AS(
      SELECT 'Month' AS time_type
            ,FORMAT_DATE('%Y%m',PARSE_DATE('%Y%m%d',date)) time
            ,trafficSource.`source` source
            ,ROUND(SUM (product.productRevenue)/1000000,4) revenue
      FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`,
            UNNEST (hits) hits,
            UNNEST (hits.product) product
      WHERE product.productRevenue IS NOT NULL
      GROUP BY 1,2,3
      ORDER BY revenue DESC
),

week_data AS(
      SELECT 'Week' AS time_type
            ,FORMAT_DATE('%Y%W',PARSE_DATE('%Y%m%d',date)) time
            ,trafficSource.`source` source
            ,ROUND(SUM (product.productRevenue)/1000000,4) revenue
      FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`,
            UNNEST (hits) hits,
            UNNEST (hits.product) product
      WHERE product.productRevenue IS NOT NULL
      GROUP BY 1,2,3
      ORDER BY revenue DESC
)

SELECT * FROM month_data
UNION ALL
SELECT * FROM week_data
ORDER BY time_type, time, revenue DESC;
```
| time_type	| time	| source	| revenue| 
| --- | --- | --- | --- |
| Month	| 201706	| (direct)	| 97333.6197| 
| Month	| 201706	| google	| 18757.1799| 
| Month	| 201706	| dfa	| 8862.23| 
| Month| 	201706|	mail.google.com| 	2563.13| 
| Month	| 201706| 	search.myway.com| 	105.94| 
| Month| 	201706| 	groups.google.com| 	101.96| 
| Month| 	201706| 	chat.google.com| 	74.03| 
| Month| 	201706| 	dealspotr.com| 	72.95| 
| Month| 	201706| 	mail.aol.com| 	64.85| 
| Month| 	201706| 	phandroid.com| 	52.95| 
| Month| 	201706| 	sites.google.com| 	39.17| 
| Month| 	201706| 	google.com| 	23.99| 
| Month| 	201706| 	yahoo| 	20.39| 
| Month| 	201706| 	youtube.com| 	16.99| 
| Month| 	201706| 	bing| 	13.98| 
| Month	| 201706| 	l.facebook.com| 	12.48| 
| Week	| 201722	| (direct)	| 6888.9| 
| Week	| 201722	| google| 	2119.39| 
| Week	| 201722	| dfa	| 1670.65| 
| Week	| 201722	| sites.google.com	| 13.98| 
| Week	| 201723	| (direct)	| 17325.6799| 
| Week| 	201723	| dfa| 	1145.28| 
| Week| 	201723	| google	| 1083.95| 
| Week	| 201723| 	search.myway.com| 	105.94| 
| Week| 	201723	| chat.google.com	| 74.03| 
| Week	| 201723	| youtube.com	| 16.99| 
| Week	| 201724	| (direct)	| 30908.9099| 
| Week	| 201724	| google| 	9217.17| 
| Week	| 201724| 	mail.google.com	| 2486.86| 
| Week	| 201724	| dfa	| 2341.56| 
| Week	| 201724	| dealspotr.com	| 72.95| 
| Week	| 201724	| bing	| 13.98| 
| Week| 	201724	| l.facebook.com	| 12.48| 
| Week	| 201725	| (direct)	| 27295.3199| 
| Week	| 201725	| google	| 1006.1| 
| Week	| 201725| 	mail.google.com	| 76.27| 
| Week	| 201725| 	mail.aol.com| 	64.85| 
| Week	| 201725	| phandroid.com	| 52.95| 
| Week	| 201725	| groups.google.com	| 38.59| 
| Week	| 201725	| sites.google.com	| 25.19| 
| Week	| 201725	| google.com| 	23.99| 
| Week	| 201726	| (direct)	| 14914.81| 
| Week	| 201726	| google	| 5330.57| 
| Week	| 201726	| dfa	|3704.74| 
| Week	| 201726	| groups.google.com	| 63.37| 
| Week	| 201726	| yahoo	| 20.39| 

:arrow_right: **_Direct traffic and Google_** consistently rank as the **_highest and second-highest sources of revenue_** over both weekly and monthly periods

### **Query 4: Average number of pageviews by purchaser type (purchasers vs non-purchasers) in June, July 2017**
```SQL
WITH 
month_data AS(
      SELECT 'Month' AS time_type
            ,FORMAT_DATE('%Y%m',PARSE_DATE('%Y%m%d',date)) time
            ,trafficSource.`source` source
            ,ROUND(SUM (product.productRevenue)/1000000,4) revenue
      FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`,
            UNNEST (hits) hits,
            UNNEST (hits.product) product
      WHERE product.productRevenue IS NOT NULL
      GROUP BY 1,2,3
      ORDER BY revenue DESC
),

week_data AS(
      SELECT 'Week' AS time_type
            ,FORMAT_DATE('%Y%W',PARSE_DATE('%Y%m%d',date)) time
            ,trafficSource.`source` source
            ,ROUND(SUM (product.productRevenue)/1000000,4) revenue
      FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`,
            UNNEST (hits) hits,
            UNNEST (hits.product) product
      WHERE product.productRevenue IS NOT NULL
      GROUP BY 1,2,3
      ORDER BY revenue DESC
)

SELECT * FROM month_data
UNION ALL
SELECT * FROM week_data
ORDER BY time_type, time, revenue DESC;
```
| month	| avg_pageviews_purchase| avg_pageviews_non_purchase
| --- | --- | --- |
| 201706	| 94.02050113895217	| 316.86558846341671| 
| 201707	| 124.23755186721992	| 334.05655979568053| 

:arrow_right: **_Non-purchasers_** display a notably **_higher average number of pageviews_** compared to purchasers. However, **_both groups_** experienced an **_increase in pageviews from June to July_**, with **_purchasers_** demonstrating a **_more pronounced relative growth_**.

### **Query 5: Average number of transactions per user that made a purchase in July 2017**
```SQL
SELECT FORMAT_DATE('%Y%m',PARSE_DATE('%Y%m%d',date)) month
      ,SUM (totals.transactions)/COUNT(DISTINCT fullVisitorId) AS avg_total_transactions_per_user
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
      UNNEST (hits) hits,
      UNNEST (hits.product) product 
WHERE totals.transactions >=1 
AND product.productRevenue IS NOT NULL 
GROUP BY 1;
```
| month	| avg_total_transactions_per_user| 
| --- | --- |
| 201707	| 4.16390041493776| 

:arrow_right: In July 2017, **_each user_** made an **_average_** of approximately **_4.16 transactions_**.

### **Query 6: Average amount of money spent per session. Only include purchaser data in July 2017**
```SQL
SELECT 
      FORMAT_DATE('%Y%m',PARSE_DATE('%Y%m%d',date)) month
      ,ROUND((SUM(product.productRevenue)/SUM(totals.visits))/1000000,2) AS avg_revenue_by_user_per_visit
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
      UNNEST (hits) hits,
      UNNEST (hits.product) product 
WHERE totals.transactions >=1 AND product.productRevenue IS NOT NULL 
GROUP BY 1;
```
| month	| avg_revenue_by_user_per_visit| 
| --- | --- |
| 201707	| 43.86| 

:arrow_right: In July 2017, **_purchasing users_** incurred an **_average expenditure_** of **_$43.86 per session_**. This metric serves as an indicator of the typical transaction value, providing valuable insights for **_understanding customer spending patterns_** and **_refining pricing strategies_**.

### **Query 7: Other products purchased by customers who purchased product "YouTube Men's Vintage Henley" in July 2017**
```SQL
SELECT 
      product.v2ProductName AS other_purchased_products
      ,SUM(product.productQuantity) AS quantity
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
      UNNEST (hits) hits,
      UNNEST (hits.product) product 
WHERE product.productRevenue IS NOT NULL 
      AND fullVisitorId IN(
            SELECT fullVisitorId
            FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
                  UNNEST (hits) hits,
                  UNNEST (hits.product) product 
            WHERE product.v2ProductName = "YouTube Men's Vintage Henley" AND product.productRevenue IS NOT NULL
          )
      AND product.v2ProductName <> "YouTube Men's Vintage Henley"
GROUP BY 1
ORDER BY 2 DESC;
```
| other_purchased_products	| quantity| 
| --- | --- |
| Google Sunglasses	| 20| 
| Google Women's Vintage Hero Tee Black	| 7| 
| SPF-15 Slim & Slender Lip Balm	| 6| 
| Google Women's Short Sleeve Hero Tee Red Heather	| 4| 
| Google Men's Short Sleeve Badge Tee Charcoal	| 3| 
| YouTube Men's Fleece Hoodie Black	| 3| 
| YouTube Twill Cap	| 2| 
| Google Men's Short Sleeve Hero Tee Charcoal	| 2| 
| Red Shine 15 oz Mug	| 2| 
| Crunch Noise Dog Toy	| 2| 

:arrow_right: **_Customers_** who purchased the **_YouTube Men's Vintage Henley_** in July 2017 exhibited a **_preference_** for **_Google-branded products_**, particularly sunglasses. This trend suggests potential opportunities for **_cross-selling_**.

### **Query 8: Calculate cohort map from product view to addtocart to purchase in Jan, Feb and March 2017. Output calculated in product level**
- Add_to_cart_rate = number product add to cart / number product view.
- Purchase_rate = number product purchase/number product view.

```SQL
WITH total_num AS(
      SELECT 
            FORMAT_DATE('%Y%m',PARSE_DATE('%Y%m%d',date)) month
            ,COUNT(CASE WHEN hits.eCommerceAction.action_type = '2' THEN product.v2ProductName ELSE NULL END) AS num_product_view
            ,COUNT(CASE WHEN hits.eCommerceAction.action_type = '3' THEN product.v2ProductName ELSE NULL END) AS num_addtocart
            ,COUNT(CASE WHEN hits.eCommerceAction.action_type = '6' AND product.productRevenue IS NOT NULL THEN product.v2ProductName ELSE NULL END) AS num_purchase
      FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
            UNNEST (hits) hits,
            UNNEST (hits.product) product
      WHERE _table_suffix BETWEEN '0101' AND '0331'
            AND eCommerceAction.action_type IN ('2','3','6')
      GROUP BY 1
      ORDER BY 1
)
SELECT 
      *
      ,ROUND(num_addtocart/num_product_view*100.0,2) AS add_to_cart_rate
      ,ROUND(num_purchase/num_product_view*100.0,2) AS purchase_rate
FROM total_num;
```
| month	| num_product_view	| num_addtocart	| num_purchase	| add_to_cart_rate	| purchase_rate| 
| --- | --- | --- | --- | --- | --- |
| 201701	| 25787	| 7342	| 2143	| 28.47	| 8.31| 
| 201702	| 21489	| 7360	| 2060	| 34.25	| 9.59| 
| 201703	| 23549	| 8782	| 2977	| 37.29	| 12.64| 

:arrow_right: The **_product view-to-purchase conversion rates_** showed **_consistent improvement_** from January to March 2017, with March achieving the highest engagement levels. This trend suggests an **_increasing effectiveness in converting_** browsers into buyers, likely attributable to enhanced marketing efforts or an improved user experience.

## **V. CONCLUSION**
In conclusion, my analysis of the eCommerce dataset using SQL on Google BigQuery has uncovered valuable **_insights into total visits, pageviews, transactions, bounce rates, and revenue per traffic source_**, which can inform future business decisions. The **_next step_** will involve **_visualizing_** these insights and key trends using software such as Power BI or Tableau. Overall, this project demonstrates the **_effectiveness of employing SQL and big data tools_** like Google BigQuery to **_derive meaningful insights from large datasets_**.
