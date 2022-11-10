# Analysis of Sports Event for marketing

**Role:** Data Scientist for the up incoming sports marketing company **DataBallers.** The company has recently been asked to **market** the largest global **sports event** in the world: **the Olympics.**

For this analysis, we will use the Olympics dataset.

![Olympics dataset ER diagram.PNG](Analysis%20of%20Sports%20Event%20for%20marketing%204747b610fdf84bf1a755676ca3b3ef4b/Olympics_dataset_ER_diagram.png)

Let’s explore the data and perform some analysis to find the insights.

1. **Sports with most athletes**

```sql
SELECT 
		sport 
    ,COUNT (DISTINCT athlete_id) AS athletes
FROM summer_games
GROUP BY sport
-- Only include the 3 sports with the most athletes
ORDER BY athletes DESC
LIMIT 3 ;
```

1. **Number of events and athletes by sports**

```sql
SELECT 
		sport
    ,COUNT (DISTINCT event) AS events
    ,COUNT (DISTINCT athlete_id) AS athletes
FROM summer_games
GROUP BY sport;
```

![Untitled](Analysis%20of%20Sports%20Event%20for%20marketing%204747b610fdf84bf1a755676ca3b3ef4b/Untitled.png)

1. **Age of oldest athlete for each region**

```sql
SELECT 
	region, 
    MAX (age) AS age_of_oldest_athlete
FROM athletes AS a
INNER JOIN summer_games AS s
	ON a.id = s.athlete_id
INNER JOIN countries AS c
	ON s.country_id = c.id
GROUP BY region;
```

1. **Number of events by sport for both summer and winter events**

```sql
SELECT 
		sport
    ,COUNT(DISTINCT event) AS events
FROM summer_games
GROUP BY sport

UNION

SELECT 
		sport
    ,COUNT(DISTINCT event) AS events
FROM winter_games
GROUP BY sport
ORDER BY events DESC;
```

1. **Athletes with 3 or more gold medals**

```sql
SELECT 
		name AS athlete_name
    ,SUM(gold) AS gold_medals
FROM summer_games AS s
JOIN athletes AS a
    ON s.athlete_id = a.id
GROUP BY athlete_name
HAVING SUM(gold) >= 3
ORDER BY gold_medals DESC;
```

![Untitled](Analysis%20of%20Sports%20Event%20for%20marketing%204747b610fdf84bf1a755676ca3b3ef4b/Untitled%201.png)

1. **Number of events held by each country in the summer and winter seasons.**

```sql

SELECT 
		'summer' AS season  -- created new column representing summer season
    ,country 
    ,COUNT (DISTINCT event) AS events
FROM summer_games AS s
	JOIN countries AS c
ON s.country_id = c.id
GROUP BY country, season

UNION     -- Combine the queries

SELECT 
		'winter' AS season  -- created new column representing winter season
    ,country
    ,COUNT (DISTINCT event) AS events
FROM winter_games AS w
JOIN countries AS c
ON w.country_id = c.id
GROUP BY country, season
ORDER BY events DESC
LIMIT 10;
```

![Untitled](Analysis%20of%20Sports%20Event%20for%20marketing%204747b610fdf84bf1a755676ca3b3ef4b/Untitled%202.png)

**Demographics Analysis**

1. **Let’s create a segment with tall male and female athletes.**

```sql
SELECT 
	name
	,CASE WHEN gender = 'F' AND height >= 175 THEN 'Tall Female'
         WHEN gender = 'M' AND height >= 190 THEN 'Tall Male'
        ELSE 'Other' END AS segment
FROM athletes;
```

1. **How BMI differs by each summer sport ?**

$$
BMI = 100 * weight / height^2
$$

```sql

SELECT 
		sport
    -- Bucket BMI in three groups: <.25, .25-.30, and >.30	
    ,CASE WHEN (100 * weight/height^2) < 0.25 THEN '<.25'
        WHEN (100 * weight/height^2) < 0.30  THEN '.25-.30'
        WHEN (100 * weight/height^2) > 0.30  THEN '>.30' END AS bmi_bucket
    ,COUNT (DISTINCT a.name) AS athletes
FROM summer_games AS s
JOIN athletes AS a
	ON s.athlete_id = a.id
GROUP BY sport, bmi_bucket
ORDER BY sport, athletes DESC;
```

The above query gives null values in bmi_bucket. Let’s see why ?

```sql
SELECT 
		height
    ,weight
    ,weight/height^2*100 AS bmi
FROM athletes
-- Filter for NULL bmi values
WHERE (weight/height^2*100) IS NULL;
```

Turns out, there are missing values in the weight column. Now, let’s update the above query to represent this condition.

```sql
SELECT 
		sport
    -- Bucket BMI in three groups: <.25, .25-.30, and >.30	
    ,CASE WHEN (100 * weight/height^2) < 0.25 THEN '<.25'
        WHEN (100 * weight/height^2) < 0.30  THEN '.25-.30'
        WHEN (100 * weight/height^2) > 0.30  THEN '>.30' 
				ELSE 'no weight recorded' END AS bmi_bucket    -- for missing values
		END AS bmi_bucket
    ,COUNT (DISTINCT a.name) AS athletes
FROM summer_games AS s
JOIN athletes AS a
	ON s.athlete_id = a.id
GROUP BY sport, bmi_bucket
ORDER BY sport, athletes DESC
```

1. **Top athletes representing Nobel-Prize winning countries**
.

```sql
SELECT  --for summer games
    event
    -- Bucketing events into gender
    ,CASE WHEN event LIKE '%Women%' THEN 'female' 
            ELSE 'male' 
            END AS gender
    ,COUNT(DISTINCT athlete_id) AS athletes
FROM summer_games
-- Only include countries that won a nobel prize
WHERE country_id IN 
		(SELECT country_id 
    FROM country_stats 
    WHERE nobel_prize_winners > 0)
GROUP BY event

UNION  -- combining summer and winter games results

SELECT --for winter games
    event
    -- Add the gender field below
    ,CASE WHEN event LIKE '%Women%' THEN 'female' 
			    ELSE 'male' 
					END AS gender
    ,COUNT(DISTINCT athlete_id) AS athletes
FROM winter_games
-- Only include countries that won a nobel prize
WHERE country_id IN 
		(SELECT country_id 
    FROM country_stats 
    WHERE nobel_prize_winners > 0)
GROUP BY event
ORDER BY athletes DESC
LIMIT 10;
```

![Untitled](Analysis%20of%20Sports%20Event%20for%20marketing%204747b610fdf84bf1a755676ca3b3ef4b/Untitled%203.png)

1. **Top 25 countries with high medal rates**

```sql
SELECT 
	--Clean the country field to only show country_code
    UPPER(LEFT(REPLACE (TRIM(c.country),'.',''),3)) AS country_code
	,pop_in_millions
	,SUM(COALESCE(bronze,0) + COALESCE(silver,0) + COALESCE(gold,0)) AS medals
	,SUM(COALESCE(bronze,0) + COALESCE(silver,0) + COALESCE(gold,0)) 
	/ 
	CAST(cs.pop_in_millions AS float) AS medals_per_million
FROM summer_games AS s
JOIN countries AS c 
ON s.country_id = c.id
JOIN country_stats AS cs 
ON s.country_id = cs.country_id AND s.year = cs.year :: DATE
-- Filter out null populations
WHERE pop_in_millions IS NOT NULL
GROUP BY c.country, pop_in_millions
ORDER BY medals_per_million DESC
LIMIT 25;
```

![Untitled](Analysis%20of%20Sports%20Event%20for%20marketing%204747b610fdf84bf1a755676ca3b3ef4b/Untitled%204.png)

1. **Average total country medals by region**

```sql
SELECT 
	  region
    ,AVG(total_golds) AS avg_total_golds
FROM
  (SELECT 
      region, 
      country_id, 
      SUM(gold) AS total_golds
  FROM summer_games_clean AS s
  JOIN countries AS c
  ON s.country_id = c.id
  GROUP BY region, country_id) AS subquery
GROUP BY region
ORDER BY avg_total_golds DESC;
```

1. **Athletes with the most golds by Region**

```sql

SELECT 
	region
    ,athlete_name
    ,total_golds
FROM
    (SELECT 
        region
        ,name AS athlete_name
        ,SUM(gold) AS total_golds
        -- Assign a regional rank to each athlete
        ,ROW_NUMBER() OVER (PARTITION BY region ORDER BY SUM(gold) DESC) AS row_num
    FROM summer_games_clean AS s
    JOIN athletes AS a
        ON a.id = s.athlete_id
    JOIN countries AS c
        ON s.country_id = c.id
    GROUP BY region, athlete_name) AS subquery
-- Filter for only the top athlete per region
WHERE row_num = 1;
```

![Untitled](Analysis%20of%20Sports%20Event%20for%20marketing%204747b610fdf84bf1a755676ca3b3ef4b/Untitled%205.png)