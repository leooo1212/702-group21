-- select all tables
SELECT name FROM sqlite_master WHERE type='table' ORDER BY name;

--rename .csv name as table
--ALTER TABLE "stg_pm25_long" RENAME TO stg_pm25;
--ALTER TABLE "GDPpercapita" RENAME TO stg_gdp;
--ALTER TABLE "LifeExpectancyData" RENAME TO stg_life;
--ALTER TABLE "country_codes" RENAME TO stg_country; -- 新加
--ALTER TABLE "lifeexpectancy" RENAME TO stg_life2; -- 如果是重复或另一份数据

--DROP TABLE IF EXISTS stg_pm25;
--DROP TABLE IF EXISTS "Life Expectancy Data";

PRAGMA foreign_keys = ON;

-- 时间维度 (Single)
DROP TABLE IF EXISTS Dim_Year;
CREATE TABLE Dim_Year (
year_id INTEGER PRIMARY KEY, -- 2010, 2011, ...
year INTEGER NOT NULL
);

SELECT _ from Dim_Year
select _ from Dim_Location
--INSERT INTO Dim_Year (year_id, year) VALUES
--(1, 2010),
--(2, 2011),
--(3, 2012),
--(4, 2013),
--(5, 2014),
--(6, 2015)

-- 地理维度 (Dimension)
DROP TABLE IF EXISTS Dim_Location;
CREATE TABLE Dim_Location (
location_id INTEGER PRIMARY KEY,
country_name TEXT NOT NULL,
continent_name TEXT,
country_type TEXT,
iso_code TEXT
);

--INSERT INTO Dim_Location (country_name, continent_name, iso_code)
--SELECT DISTINCT
-- c1, -- Country Name
-- c3, -- Region
-- c2 -- Country Code (ISO)
--FROM stg_life2
--WHERE c1 IS NOT NULL AND TRIM(c1) <> '';

--UPDATE Dim_Location
--SET country_type = (
-- SELECT t.c3
-- FROM stg_life t
-- WHERE t.c1 = Dim_Location.country_name
--);

--select \* from Dim_Location

-- 指标维度 (Dimension)
DROP TABLE IF EXISTS Dim_Indicator;
CREATE TABLE Dim_Indicator (
indicator_id INTEGER PRIMARY KEY,
category TEXT NOT NULL, -- Environment / Economy / Health
indicator_name TEXT NOT NULL, -- PM2.5 / GDP per capita / Life Expectancy
unit TEXT NOT NULL
);

DELETE FROM Dim_Indicator;
INSERT INTO Dim_Indicator (indicator_id, category, indicator_name, unit) VALUES
(1,'Environment','PM2.5','µg/m³'),
(2,'Environment','CO2_emissions','kiloton'),
(3,'Economy','GDP_per_capita','USD'),
(4,'Health','Life_expectancy','years');

-- PM2.5 分档维度（Dimension）
CREATE TABLE dim_PM25Band (
band_id INTEGER PRIMARY KEY AUTOINCREMENT,
category TEXT NOT NULL,  
 severity TEXT,  
 min_value REAL NOT NULL,  
 max_value REAL NOT NULL,  
 color_code TEXT NOT NULL  
);
select \* from dim_PM25Band
INSERT INTO dim_PM25Band (category, severity, min_value, max_value, color_code) VALUES
('Good', NULL, 0, 75, '#00FF00'),
('Polluted', 'Light', 75, 115, '#FFFF00'),
('Polluted', 'Moderate', 115, 150, '#FFA500'),
('Polluted', 'Heavy', 150, 250, '#FF4500'),
('Polluted', 'Severe', 250, 500, '#FF0000');

select \* from dim_PM25Band
DROP TABLE IF EXISTS Fact_Metrics;
CREATE TABLE Fact_Metrics (
fact_id INTEGER PRIMARY KEY,
location_id INTEGER NOT NULL,
year_id INTEGER NOT NULL,
indicator_id INTEGER NOT NULL,
value REAL,
band_id INTEGER,
FOREIGN KEY(location_id) REFERENCES Dim_Location(location_id),
FOREIGN KEY(year_id) REFERENCES Dim_Year(year_id),
FOREIGN KEY(indicator_id) REFERENCES Dim_Indicator(indicator_id),
FOREIGN KEY(band_id) REFERENCES Dim_PM25Band(band_id)
);

select \* from Fact_Metrics

-- pm 2.5
DELETE FROM Fact_Metrics;

INSERT INTO Fact_Metrics (location_id, year_id, indicator_id, value, band_id)
SELECT
dL.location_id,
dY.year_id,
dI.indicator_id,
ROUND(p.c4, 2) AS value,  
 b.band_id  
FROM stg_pm25 p
JOIN Dim_Location dL
ON p.c2 = dL.iso_code
OR UPPER(TRIM(p.c1)) = UPPER(TRIM(dL.country_name))
JOIN Dim_Year dY
ON dY.year = p.c3
JOIN Dim_Indicator dI
ON dI.indicator_name = 'PM2.5'
LEFT JOIN Dim_PM25Band b
ON p.c4 >= b.min_value AND p.c4 < b.max_value
WHERE p.c3 BETWEEN 2010 AND 2015
AND p.c4 IS NOT NULL;

-- CO2
INSERT INTO Fact_Metrics (location_id, year_id, indicator_id, value, band_id)
SELECT
dl.location_id,
dy.year_id,
2,
round(p.c8,2) as value,
NULL
FROM stg_life2 p
JOIN Dim_Location dl
ON TRIM(UPPER(p.c1)) = TRIM(UPPER(dl.country_name))
JOIN Dim_Year dy
ON dy.year = p.c5
WHERE p.c8 IS NOT NULL;

-- GDP per capita
INSERT INTO Fact_Metrics (location_id, year_id, indicator_id, value, band_id)
SELECT
dl.location_id,
dy.year_id,
3, -- GDP_per_capita
round(p.c17,2) as value,
NULL
FROM stg_life p
JOIN Dim_Location dl
ON UPPER(TRIM(p.c1)) = UPPER(TRIM(dl.country_name))
JOIN Dim_Year dy
ON dy.year = p.c2
WHERE p.c17 IS NOT NULL;

-- Life expectancy
INSERT INTO Fact_Metrics (location_id, year_id, indicator_id, value, band_id)
SELECT
dl.location_id,
dy.year_id,
4, -- Life_expectancy
p.c4 as value,
NULL
FROM stg_life p
JOIN Dim_Location dl
ON UPPER(TRIM(p.c1)) = UPPER(TRIM(dl.country_name))
JOIN Dim_Year dy
ON dy.year = p.c2
WHERE p.c4 IS NOT NULL;

select \* from Fact_Metrics

-- Q1
-- 空气污染与健康
-- 在每个大洲中，PM2.5 与寿命最高和最低的国家分别是谁？
SELECT
l.continent_name,
l.country_name,
round(AVG(CASE WHEN i.indicator_name='PM2.5' THEN round(f.value,2) END),2) AS avg_pm25,
round(AVG(CASE WHEN i.indicator_name='Life_expectancy' THEN round(f.value,2) END),2) AS avg_life
FROM Fact_Metrics f
JOIN Dim_Location l ON f.location_id = l.location_id
JOIN Dim_Indicator i ON f.indicator_id = i.indicator_id
GROUP BY l.continent_name, l.country_name
ORDER BY l.continent_name, avg_life DESC;

-- Q2
-- 在各大洲内，人均 GDP 与寿命的相关性如何？
-- Q2 (加年份): 在各大洲内，人均 GDP 与 PM2.5 的关系 (按年)
SELECT
l.continent_name,
l.country_name,
y.year,
AVG(CASE WHEN i.indicator_name='GDP_per_capita' THEN f.value END) AS avg_gdp,
AVG(CASE WHEN i.indicator_name='PM2.5' THEN f.value END) AS avg_pm25
FROM Fact_Metrics f
JOIN Dim_Location l ON f.location_id = l.location_id
JOIN Dim_Indicator i ON f.indicator_id = i.indicator_id
JOIN Dim_Year y ON f.year_id = y.year_id
GROUP BY l.continent_name, l.country_name, y.year
ORDER BY l.continent_name, l.country_name, y.year;

-- Q3. PM2.5 档位 (国家分布)
-- 不同 PM2.5 档位下，各大洲的国家分布情况如何？
SELECT
l.continent_name,
l.country_name,
b.band_name,
COUNT(\*) AS n_records
FROM Fact_Metrics f
JOIN Dim_Location l ON f.location_id = l.location_id
JOIN Dim_PM25Band b ON f.band_id = b.band_id
GROUP BY l.continent_name, l.country_name, b.band_name
ORDER BY b.band_name, l.continent_name;

-- Q4. 碳排放效率 (国家效率比较)
-- 各国的 CO₂ 排放 / GDP per capita，谁的碳效率更高或更差？
WITH co2 AS (
SELECT f.location_id, f.year_id, AVG(f.value) AS avg_co2
FROM Fact_Metrics f
JOIN Dim_Indicator i ON f.indicator_id = i.indicator_id
WHERE i.indicator_name = 'CO2_emissions'
GROUP BY f.location_id, f.year_id
),
gdp AS (
SELECT f.location_id, f.year_id, AVG(f.value) AS avg_gdp
FROM Fact_Metrics f
JOIN Dim_Indicator i ON f.indicator_id = i.indicator_id
WHERE i.indicator_name = 'GDP_per_capita'
GROUP BY f.location_id, f.year_id
)
SELECT
l.continent_name,
l.country_name,
y.year,
co2.avg_co2,
gdp.avg_gdp,
round(co2.avg_co2 / NULLIF(gdp.avg_gdp,0),2) AS co2_per_gdp
FROM Dim_Location l
JOIN Dim_Year y ON 1=1
LEFT JOIN co2 ON l.location_id=co2.location_id AND y.year_id=co2.year_id
LEFT JOIN gdp ON l.location_id=gdp.location_id AND y.year_id=gdp.year_id
ORDER BY l.continent_name, l.country_name, y.year;

-- Q5
-- 综合健康-经济-环境指数 (国家层面)
-- formula: Life_expectancy / PM2.5 × log(GDP_per_capita+1) - CO₂_emissions/1000

WITH country_avgs AS (
SELECT
l.continent_name,
l.country_name,
AVG(CASE WHEN i.indicator_name='Life_expectancy' THEN f.value END) AS avg_life,
AVG(CASE WHEN i.indicator_name='PM2.5' THEN f.value END) AS avg_pm25,
AVG(CASE WHEN i.indicator_name='GDP_per_capita' THEN f.value END) AS avg_gdp,
AVG(CASE WHEN i.indicator_name='CO2_emissions' THEN f.value END) AS avg_co2
FROM Fact_Metrics f
JOIN Dim_Location l ON f.location_id = l.location_id
JOIN Dim_Indicator i ON f.indicator_id = i.indicator_id
GROUP BY l.continent_name, l.country_name
)
SELECT
continent_name,
country_name,
round((avg_life / avg_pm25) \* log(avg_gdp + 1) - (avg_co2/1000),2) AS health_index
FROM country_avgs
ORDER BY health_index DESC;
