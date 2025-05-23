# College Town Dining Analysis

### 1. Query to get the CBGs with highest youth ratio (>= 80%) among the top 5 counties we selected: 
```
SELECT  
  CountyName,  
  cbg,  
  YouthRatio  
from  
  ( 
    SELECT  
      *,  
      ROW_NUMBER() OVER ( 
      PARTITION BY CountyName  
      ORDER BY YouthRatio DESC 
      ) rn  
    FROM  
      ( 
        select  
          case when cbg like '18157%' then 'Tippecanoe County' when cbg like '17019%' then 'Champaign County' when cbg like '42027%' then 'Centre County' when cbg like '18105%' then 'Monroe County' when cbg like '28071%' then 'Lafayette  County' end as CountyName,  
          cbg,  
          ( 
           sum(`pop_m_18-19`)+ sum(`pop_f_18-19`)+ sum(`pop_m_20`)+ sum(`pop_f_20`)+ sum(`pop_m_21`)+ sum(`pop_f_21`)+ sum(`pop_m_22-24`)+ sum(`pop_f_22-24`)
          )/ sum(pop_total) as YouthRatio  
        FROM  
         `antilles-data-mgmt58200-final.safegraph.cbg_demographics` d  
          join `antilles-data-mgmt58200-final.safegraph.cbg_fips` c on LEFT(d.cbg, 2) = c.state_fips  
          and SUBSTRING(d.cbg, 3, 3) = c.county_fips  
        WHERE  
          ( 
            cbg LIKE '18157%'  
            or cbg LIKE '17019%'  
            or cbg LIKE '42027%'  
            or cbg LIKE '18105%'  
            or cbg LIKE '28071%' 
          )  
          and cbg is not null  
        group by  
          cbg  
        having  
          youthRatio >= 0.90  
        order by  
          youthratio desc 
      ) 
  )  
WHERE  
  rn = 1;
```



### 2. Query to get income band information for the cbgs with high youth ratio in the specified counties 
```
select  
  case when cbg like '18157%' then 'Tippecanoe County' when cbg like '17019%' then 'Champaign County' when cbg like '42027%' then 'Centre County' when cbg like '18105%' then 'Monroe County' when cbg like '28071%' then 'Lafayette  County' end as CountyName,  
  d.cbg,  
  sum(`inc_lt10`) Lt10,  
  sum(`inc_10-15`) Inc10To15,  
  sum(`inc_15-20`) Inc15To20,  
  sum(`inc_20-25`) Inc20To25,  
  sum(`inc_25-30`) Inc25To30,  
  sum(`inc_30-35`) Inc30To35,  
  sum(`inc_35-40`) Inc35To40,  
  sum(`inc_40-45`) Inc40To45,  
  sum(`inc_45-50`) Inc45To50,  
  sum(`inc_50-60`) Inc50To60,  
  sum(`inc_60-75`) Inc60To75,  
  sum(`inc_75-100`) Inc75To100,  
  sum(`inc_100-125`) Inc100To125,  
  sum(`inc_125-150`) Inc125To150,  
  sum(`inc_150-200`) Inc150To200  
FROM  
  `antilles-data-mgmt58200-final.safegraph.cbg_demographics` d  
  join `antilles-data-mgmt58200-final.safegraph.cbg_fips` c on LEFT(d.cbg, 2) = c.state_fips  
  and SUBSTRING(d.cbg, 3, 3) = c.county_fips  
WHERE  
  cbg in ( 
    select  
      cbg  
    from  
      ( 
       select  
          cbg,  
          ( 
            sum(`pop_m_18-19`)+ sum(`pop_f_18-19`)+ sum(`pop_m_20`)+ sum(`pop_f_20`)+ sum(`pop_m_21`)+ sum(`pop_f_21`)+ sum(`pop_m_22-24`)+ sum(`pop_f_22-24`) 
          )/ sum(pop_total) as YouthRatio  
        FROM  
          `antilles-data-mgmt58200-final.safegraph.cbg_demographics` d  
          join `antilles-data-mgmt58200-final.safegraph.cbg_fips` c on LEFT(d.cbg, 2) = c.state_fips  
          and SUBSTRING(d.cbg, 3, 3) = c.county_fips  
        WHERE  
          ( 
            cbg LIKE '18157%'  
            or cbg LIKE '17019%'  
            or cbg LIKE '42027%'  
            or cbg LIKE '18105%'  
            or cbg LIKE '28071%' 
          )  
        group by  
          cbg  
        having  
          youthRatio >= 0.80  
        order by  
          youthratio desc 
      ) 
  )  
group by cbg

```


### 3. Query to retrieve the bottom 10 restaurants by footfall, along with their details, located in areas with high youth density. These restaurants should also be in counties where the overall footfall is above the county average. Its done similarly for each county.
```
WITH RestaurantCounts AS (
  SELECT 
    'Tippecanoe County' AS county,
    cd.cbg,
    COUNT(p.safegraph_place_id) AS restaurant_count
  FROM 
    `antilles-data-mgmt58200-final.safegraph.cbg_demographics` cd
  JOIN 
    `antilles-data-mgmt58200-final.safegraph.visits` v ON v.poi_cbg = cd.cbg
  JOIN 
    `antilles-data-mgmt58200-final.safegraph.places` p ON p.safegraph_place_id = v.safegraph_place_id
  WHERE 
    p.top_category LIKE '%Restaurant%' 
    AND cd.cbg LIKE '18157%'
  GROUP BY 
    cd.cbg
),
PopulationData AS (
  SELECT 
    'Tippecanoe County' AS county,
    cd.cbg,
    SUM(cd.pop_total) AS population,
    (SUM(cd.`pop_m_18-19`) + SUM(cd.`pop_f_18-19`) +
     SUM(cd.`pop_m_20`) + SUM(cd.`pop_f_20`) +
     SUM(cd.`pop_m_21`) + SUM(cd.`pop_f_21`) +
     SUM(cd.`pop_m_22-24`) + SUM(cd.`pop_f_22-24`)) / SUM(cd.pop_total) AS youth_ratio
  FROM 
    `antilles-data-mgmt58200-final.safegraph.cbg_demographics` cd
  WHERE 
    cd.cbg LIKE '18157%'
  GROUP BY 
    cd.cbg
),
FootfallData AS (
  SELECT 
    v.poi_cbg AS cbg,
    'Tippecanoe County' AS county,
    SUM(v.raw_visit_counts) AS total_footfall
  FROM 
    `antilles-data-mgmt58200-final.safegraph.visits` v
  WHERE 
    v.poi_cbg LIKE '18157%'
  GROUP BY 
    v.poi_cbg
),
AverageFootfall AS (
  SELECT 
    'Tippecanoe County' AS county,
    AVG(total_footfall) AS avg_footfall
  FROM 
    FootfallData
),
DensityData AS (
  SELECT 
    rc.county,
    rc.cbg,
    rc.restaurant_count,
    pd.population,
    pd.youth_ratio,
    fd.total_footfall,
    af.avg_footfall,
    (rc.restaurant_count / NULLIF(pd.population, 0)) AS restaurant_density,
    ROW_NUMBER() OVER (ORDER BY (rc.restaurant_count / NULLIF(pd.population, 0)) ASC) AS density_rank
  FROM 
    RestaurantCounts rc
  LEFT JOIN 
    PopulationData pd ON rc.cbg = pd.cbg
  LEFT JOIN 
    FootfallData fd ON rc.cbg = fd.cbg
  LEFT JOIN 
    AverageFootfall af ON rc.county = af.county
  WHERE 
    rc.restaurant_count > 0
    AND pd.youth_ratio >= 0.80
    AND fd.total_footfall > af.avg_footfall
)
SELECT 
  d.county,
  v.location_name,
  v.raw_visit_counts AS footfall,
  p.naics_code,
  v.poi_cbg AS cbg,
  d.avg_footfall  -- Include the average footfall for each area
FROM 
  `antilles-data-mgmt58200-final.safegraph.visits` v
JOIN 
  `antilles-data-mgmt58200-final.safegraph.places` p ON v.safegraph_place_id = p.safegraph_place_id
JOIN 
  DensityData d ON v.poi_cbg = d.cbg
WHERE 
  p.top_category LIKE '%Restaurant%'
ORDER BY 
  footfall ASC
LIMIT 10;

```


### 4. Query to identify the popular restaurants (highest footfall)
```
WITH RankedRestaurants AS ( 
  SELECT  
    CASE  
      WHEN cd.cbg LIKE '18157%' THEN 'Tippecanoe County' 
      WHEN cd.cbg LIKE '17019%' THEN 'Champaign County' 
      WHEN cd.cbg LIKE '42027%' THEN 'Centre County' 
      WHEN cd.cbg LIKE '18105%' THEN 'Monroe County' 
      WHEN cd.cbg LIKE '28071%' THEN 'Lafayette County' 
    END AS county,  
    v.location_name,  
    SUM(v.raw_visit_counts) AS total_visits,
    ROW_NUMBER() OVER (PARTITION BY  
      CASE  
        WHEN cd.cbg LIKE '18157%' THEN 'Tippecanoe County' 
        WHEN cd.cbg LIKE '17019%' THEN 'Champaign County' 
        WHEN cd.cbg LIKE '42027%' THEN 'Centre County' 
        WHEN cd.cbg LIKE '18105%' THEN 'Monroe County' 
        WHEN cd.cbg LIKE '28071%' THEN 'Lafayette County' 
      END 
      ORDER BY SUM(v.raw_visit_counts) DESC 
    ) AS rank 

  FROM  `antilles-data-mgmt58200-final.safegraph.cbg_demographics` cd 
  JOIN  `antilles-data-mgmt58200-final.safegraph.visits` v  
    ON v.poi_cbg = cd.cbg 
  JOIN  
    `antilles-data-mgmt58200-final.safegraph.places` p  
    ON p.safegraph_place_id = v.safegraph_place_id 
  WHERE  
    (cd.cbg LIKE '18157%' OR cd.cbg LIKE '17019%' OR cd.cbg LIKE '42027%' OR  
    cd.cbg LIKE '18105%' OR cd.cbg LIKE '28071%') 
    AND p.top_category LIKE '%Restaurant%' 
    AND p.location_name NOT LIKE '%Commons%' 
    AND EXTRACT(YEAR FROM v.date_range_start) = 2020 
    AND EXTRACT(MONTH FROM v.date_range_start) = 1
    AND EXTRACT(YEAR FROM v.date_range_end) = 2020 
    AND EXTRACT(MONTH FROM v.date_range_end) = 2 
  GROUP BY  
    cd.cbg, v.postal_code, v.location_name) 

SELECT county, location_name, SUM(total_visits) AS total_visits
FROM RankedRestaurants 
WHERE rank <= 20
GROUP BY county, location_name 
ORDER BY  
  county, total_visits DESC;

```


### 5. Query to get footfall for restaurants based on hour for selected counties (before March 2020)
```
SELECT  hour,SUM(visits) AS total_visits
FROM (
  SELECT
    [STRUCT(0 AS hour, CAST(REGEXP_REPLACE(SPLIT(popularity_by_hour, ',')[OFFSET(0)], r'[^0-9]', '') AS INT64) AS visits),
     STRUCT(1 AS hour, CAST(REGEXP_REPLACE(SPLIT(popularity_by_hour, ',')[OFFSET(1)], r'[^0-9]', '') AS INT64) AS visits),
     STRUCT(2 AS hour, CAST(REGEXP_REPLACE(SPLIT(popularity_by_hour, ',')[OFFSET(2)], r'[^0-9]', '') AS INT64) AS visits),
     STRUCT(3 AS hour, CAST(REGEXP_REPLACE(SPLIT(popularity_by_hour, ',')[OFFSET(3)], r'[^0-9]', '') AS INT64) AS visits),
     STRUCT(4 AS hour, CAST(REGEXP_REPLACE(SPLIT(popularity_by_hour, ',')[OFFSET(4)], r'[^0-9]', '') AS INT64) AS visits),
     STRUCT(5 AS hour, CAST(REGEXP_REPLACE(SPLIT(popularity_by_hour, ',')[OFFSET(5)], r'[^0-9]', '') AS INT64) AS visits),
     STRUCT(6 AS hour, CAST(REGEXP_REPLACE(SPLIT(popularity_by_hour, ',')[OFFSET(6)], r'[^0-9]', '') AS INT64) AS visits),
     STRUCT(7 AS hour, CAST(REGEXP_REPLACE(SPLIT(popularity_by_hour, ',')[OFFSET(7)], r'[^0-9]', '') AS INT64) AS visits),
     STRUCT(8 AS hour, CAST(REGEXP_REPLACE(SPLIT(popularity_by_hour, ',')[OFFSET(8)], r'[^0-9]', '') AS INT64) AS visits),
     STRUCT(9 AS hour, CAST(REGEXP_REPLACE(SPLIT(popularity_by_hour, ',')[OFFSET(9)], r'[^0-9]', '') AS INT64) AS visits),
     STRUCT(10 AS hour, CAST(REGEXP_REPLACE(SPLIT(popularity_by_hour, ',')[OFFSET(10)], r'[^0-9]', '') AS INT64) AS visits),
     STRUCT(11 AS hour, CAST(REGEXP_REPLACE(SPLIT(popularity_by_hour, ',')[OFFSET(11)], r'[^0-9]', '') AS INT64) AS visits),
     STRUCT(12 AS hour, CAST(REGEXP_REPLACE(SPLIT(popularity_by_hour, ',')[OFFSET(12)], r'[^0-9]', '') AS INT64) AS visits),
     STRUCT(13 AS hour, CAST(REGEXP_REPLACE(SPLIT(popularity_by_hour, ',')[OFFSET(13)], r'[^0-9]', '') AS INT64) AS visits),
     STRUCT(14 AS hour, CAST(REGEXP_REPLACE(SPLIT(popularity_by_hour, ',')[OFFSET(14)], r'[^0-9]', '') AS INT64) AS visits),
     STRUCT(15 AS hour, CAST(REGEXP_REPLACE(SPLIT(popularity_by_hour, ',')[OFFSET(15)], r'[^0-9]', '') AS INT64) AS visits),
     STRUCT(16 AS hour, CAST(REGEXP_REPLACE(SPLIT(popularity_by_hour, ',')[OFFSET(16)], r'[^0-9]', '') AS INT64) AS visits),
     STRUCT(17 AS hour, CAST(REGEXP_REPLACE(SPLIT(popularity_by_hour, ',')[OFFSET(17)], r'[^0-9]', '') AS INT64) AS visits),
     STRUCT(18 AS hour, CAST(REGEXP_REPLACE(SPLIT(popularity_by_hour, ',')[OFFSET(18)], r'[^0-9]', '') AS INT64) AS visits),
     STRUCT(19 AS hour, CAST(REGEXP_REPLACE(SPLIT(popularity_by_hour, ',')[OFFSET(19)], r'[^0-9]', '') AS INT64) AS visits),
     STRUCT(20 AS hour, CAST(REGEXP_REPLACE(SPLIT(popularity_by_hour, ',')[OFFSET(20)], r'[^0-9]', '') AS INT64) AS visits),
     STRUCT(21 AS hour, CAST(REGEXP_REPLACE(SPLIT(popularity_by_hour, ',')[OFFSET(21)], r'[^0-9]', '') AS INT64) AS visits),
     STRUCT(22 AS hour, CAST(REGEXP_REPLACE(SPLIT(popularity_by_hour, ',')[OFFSET(22)], r'[^0-9]', '') AS INT64) AS visits),
     STRUCT(23 AS hour, CAST(REGEXP_REPLACE(SPLIT(popularity_by_hour, ',')[OFFSET(23)], r'[^0-9]', '') AS INT64) AS visits)
    ] AS hours
  FROM
    `antilles-data-mgmt58200-final.safegraph.visits` AS v
  JOIN
    `antilles-data-mgmt58200-final.safegraph.cbg_demographics` AS d
  ON
    v.poi_cbg = d.cbg
  join   `antilles-data-mgmt58200-final.safegraph.brands` b on v.safegraph_brand_ids=b.safegraph_brand_id
    WHERE   ( 

            poi_cbg LIKE '18157%'  

            or poi_cbg LIKE '17019%'  

            or poi_cbg LIKE '42027%'  

            or poi_cbg LIKE '18105%'  

            or poi_cbg LIKE '28071%' 

          )  
    and b.top_category like '%Restaurants%'
    
   AND DATE(date_range_end) <= '2020-02-29'
 
), UNNEST(hours) AS hour_visits_struct
GROUP BY
  hour_visits_struct.hour
ORDER BY
  total_visits DESC;

```


### 6. Query to get day-wise footfall for the top restaurants we have selected across the 5 counties:
```
select distinct location_name, sum(Monday_visits) MondayVisits, sum(Tuesday_visits) TuesdayVisits, sum(Wednesday_visits) WednesdayVisits, sum(Thursday_visits) ThursdayVisits, sum(Friday_visits) FridayVisits, sum(Saturday_visits) SaturdayVisits, sum(Sunday_visits) SundayVisits, 

from (
SELECT location_name, 
       CAST(JSON_EXTRACT_SCALAR(popularity_by_day, '$.Monday') AS INT64) AS Monday_visits,
       CAST(JSON_EXTRACT_SCALAR(popularity_by_day, '$.Tuesday') AS INT64) AS Tuesday_visits,
       CAST(JSON_EXTRACT_SCALAR(popularity_by_day, '$.Wednesday') AS INT64) AS Wednesday_visits,
       CAST(JSON_EXTRACT_SCALAR(popularity_by_day, '$.Thursday') AS INT64) AS Thursday_visits,
       CAST(JSON_EXTRACT_SCALAR(popularity_by_day, '$.Friday') AS INT64) AS Friday_visits,
       CAST(JSON_EXTRACT_SCALAR(popularity_by_day, '$.Saturday') AS INT64) AS Saturday_visits,
       CAST(JSON_EXTRACT_SCALAR(popularity_by_day, '$.Sunday') AS INT64) AS Sunday_visits
FROM `antilles-data-mgmt58200-final.safegraph.visits`
WHERE location_name IN ('McDonald\'s', 'Starbucks', 'Chick-fil-A', 'Olive Garden', 
                        'Cracker Barrel', 'Texas Roadhouse', 'Panera Bread', 
                        'Chili\'s Grill & Bar', 'Steak \'n Shake', 'Wendy\'s', 
                        'IHOP', 'Culver\'s', 'Cheddar\'s Scratch Kitchen')
AND  (poi_cbg LIKE '18157%' or poi_cbg LIKE '17019%' or poi_cbg LIKE '42027%' or poi_cbg LIKE '18105%' or poi_cbg LIKE '28071%')  
AND EXTRACT(MONTH FROM date_range_start) = 2

)
group by location_name

```

