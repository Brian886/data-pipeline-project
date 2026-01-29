# Name: Chang Jia Qian
# Neptun: ANQSCL 
# Title: Final Project (Agriculture Data Pipeline)

## Business Goal of the project:
The primary goal of this project is to design and implement an automated agricultural analytics pipeline that integrates historical crop data with up-to-date weather information to support data-driven decision-making in agriculture.
The system aims to help stakeholders understand the impact of climate factors on crop yields, identify high-performing crop–region combinations, and improve planning related to land use, irrigation, and crop selection.

By leveraging cloud-native AWS services, the project demonstrates how scalable data pipelines can be used to continuously ingest, process, and analyze agricultural data with minimal manual intervention.

## Stakeholder and value:
# Stakeholder
1) Farmers
2) Policy makers
3) Agriculture business analyst 
4) Researchers studying climate impact on crops

# Business value
1) Understand weather impact on crop yield
2) Support data-driven irrigation and crop selection
3) Enable early warning insights for extreme weather conditions

## KPIs:
1) Understand weather impact on crop yield
3) Find out which irrigation method is more efficient
4) Average yield on crops
5) Best soil type per crop

## Dataset:
- CSV: Agriculture dataset | Karnataka (kaggle)
- API: Open Meteo Weather API

yeild: amount of crop produced per unit area of land

# Step 1: Create S3 bucket
![alt text](<S3 bucket created .png>)

S3 bucket was created to store data 

# Step 2: Create folders within S3 bucket
![alt text](<Folders created .png>)

Folders was created to seperate raw data and data that were processed. 

# Step 3: Upload CSV file to S3 bucket
![alt text](<Data uploaded to S3.png>)


# Step 4: Create a Glue Database
![alt text](<Glue Database Created.png>)

A logical container for metadata. This will be helpful when storing data that can be used to do analysis on.

# Step 5: Create crawler and run it
![alt text](<Crawler created and ran successfully.png>)
Use it to extract the data

# Step 6: Query Data using Athena
```
SELECT *
FROM raw_historical
LIMIT 10;
```
![alt text](<Query result 1.png>)
This is to test if the crawler could work well. 

# Step 7: Create ETL Job
![alt text](<ETL job create.png>)
ETL job is created to clean and flatten the data

# Step 8: Visual creation in ETL Job
![alt text](<Visual creation ELT Job.png>)
This part shows how ETL job is cleaning the data by changing the format of the data deriving new column. Duplicates were removed too

# Step 9: Successfully Run ETL job
![alt text](<Successfully run ETL job.png>)

# Step 10: Create Lambda function
![alt text](<Create Lamda function.png>)
This is created to help retrieve data from the weather api.

# Step 11: Create EventBridge
![alt text](<EventBridge created.png>)
This EventBridge is created as it acts like a trigger to retrieve updated weekly weather data. 

# Step 12: Retrieve Data from Weather API with Lambda
```
import json
import urllib.request
import datetime
import boto3

def lambda_handler(event, context):
    s3 = boto3.client('s3')
    glue = boto3.client('glue')

    # Locations from your CSV
    locations = {
        "Mangalore": {"lat": 12.8855, "lon": 74.8388},
        "Kodagu": {"lat": 12.4183, "lon": 75.7408},
        "Bangalore": {"lat": 12.9629, "lon": 77.5775},
        "Chikmangaluru": {"lat": 13.3161, "lon": 75.7720},
        "Davangere": {"lat": 14.4644, "lon": 75.9218},
        "Gulbarga": {"lat": 17.3297, "lon": 76.8343},
        "Hassan": {"lat": 13.0033, "lon": 76.1004},
        "Kasaragodu": {"lat": 12.5064, "lon": 74.9943},
        "Madikeri": {"lat": 12.4244, "lon": 75.7382},
        "Mysuru": {"lat": 12.2958, "lon": 76.6394},
        "Raichur": {"lat": 16.2048, "lon": 77.3553},
    }

    today = datetime.date.today().isoformat()

    for location, coords in locations.items():
        weather_url = (
            "https://api.open-meteo.com/v1/forecast?"
            f"latitude={coords['lat']}&longitude={coords['lon']}"
            "&daily=temperature_2m_max,temperature_2m_min,"
            "precipitation_sum,et0_fao_evapotranspiration,sunshine_duration"
            "&timezone=Asia/Kolkata"
        )

        weather_response = urllib.request.urlopen(weather_url)
        weather_data = json.loads(weather_response.read())

        weather_key = f"Raw/weather/location={location}/weather_{today}.json"

        s3.put_object(
            Bucket="agri-analytics-anqscl",
            Key=weather_key,
            Body=json.dumps(weather_data)
        )

        solar_url = (
            "https://api.open-meteo.com/v1/forecast?"
            f"latitude={coords['lat']}&longitude={coords['lon']}"
            "&daily=shortwave_radiation_sum"
            "&timezone=Asia/Kolkata"
        )

        solar_response = urllib.request.urlopen(solar_url)
        solar_data = json.loads(solar_response.read())

        solar_data["location"] = location

        solar_key = f"Raw/solar/location={location}/solar_{today}.json"

        s3.put_object(
            Bucket="agri-analytics-anqscl",
            Key=solar_key,
            Body=json.dumps(solar_data)
        )
        
    return {
        "statusCode": 200,
        "body": "Weather and solar data saved"
    }
```
![alt text](<Successfully run Lambda.png>)
This code helps to retreive up to date data about temperature and precipitation. 

# Step 13: Create Crawler for weather API
![alt text](<Crawler weather created.png>)
To retrieve data from S3 and store it in a database

# Step 14: Table created from weather crawler
![alt text](<Weather table created.png>)

```
SELECT
    location,
    daily.temperature_2m_max[1]         AS max_temp,
    daily.temperature_2m_min[1]         AS min_temp,
    daily.et0_fao_evapotranspiration[1] AS evapotranspiration_mm,
    daily.sunshine_duration[1] / 3600   AS sunshine_hours
FROM raw_weather;
```

# flatten data
# Step 15: Create View with Athena to avoid repeating logic
```
CREATE OR REPLACE VIEW weather_flat AS
SELECT
    location,
    daily.temperature_2m_max[1]         AS max_temp,
    daily.temperature_2m_min[1]         AS min_temp,
    daily.precipitation_sum[1]          AS precipitation_mm,
    daily.et0_fao_evapotranspiration[1] AS evapotranspiration_mm,
    daily.sunshine_duration[1] / 3600   AS sunshine_hours,
    CASE WHEN daily.temperature_2m_max[1] >= 35 THEN 1 ELSE 0 END AS extreme_heat
FROM raw_weather;
```

```
SELECT *
FROM weather_flat
LIMIT 10;
```
So that unneeded data are not included. 
# Step 16: Join the data from CSV and from API

# save as a processed table
```
CREATE TABLE processed.crops_weather
WITH (
  format = 'PARQUET',
  external_location = 's3://agri-analytics-anqscl/Processed/crops_weather/'
) AS
SELECT
  c.year,
  c.location,
  c.crops,
  c.season,
  c."soil type",
  c.irrigation,
  c.area,
  c.humidity,
  c.yeilds,
  c.rainfall,
  w.max_temp,
  w.min_temp,
  w.precipitation_mm,
  w.evapotranspiration_mm,
  w.sunshine_hours,
  w.extreme_heat
FROM raw_historical c
LEFT JOIN weather_flat w
  ON c.location = w.location;
``` 
This new join table will be used to do anaylysis on.  

# Step 17: Verify the data
```
SELECT *
FROM Processed.crops_weather
LIMIT 10;
```
![alt text](<Query result 5.png>)

# Step 18:  Do analysis

# Average Yield by Crop
```
SELECT
  crops,
  AVG(yeilds) AS avg_yield
FROM processed.crops_weather
GROUP BY crops
ORDER BY avg_yield DESC;
```
![alt text](<Query result (Average Yield by Crop).png>)

The result shows the the average unit produce per area of land of each different crop. 

# Temperature Impact on Yield
```
SELECT
    crops,
    ROUND(AVG(yeilds), 2) AS avg_yield,
    ROUND(AVG((max_temp + min_temp)/2), 2) AS avg_temperature
FROM processed.crops_weather
GROUP BY crops
ORDER BY avg_yield DESC;
```
![alt text](<Query result (Temperature Impact on Yield).png>)
For example, to get an average yield for Cocoa, an average temperature of 23.59 degree celcius


# Best soil type per Crop
```
SELECT
  crops,
  "soil type",
  ROUND(AVG(yeilds), 2) AS avg_yield
FROM processed.crops_weather
GROUP BY crops, "soil type"
ORDER BY crops ASC, avg_yield DESC;
```
![alt text](<Query result (Best Soil Type per Crop).png>)
For example, if we want to have the highest yield to grow Arecanut, we can use Sandy Loam soil type to grow it. 

# Irrigation method efficiency
```
SELECT
  irrigation,
  AVG(yeilds) AS avg_yield
FROM processed.crops_weather
GROUP BY irrigation
ORDER BY avg_yield DESC;
```
![alt text](<Query result (Irrigation Method Efficiency).png>)

This result shows the average yield of crop produced based on the 3 different irrigation method produced grouped by the irrigation method. 

# Crop yeild based on sunshine
```
SELECT
    crops,
    ROUND(AVG(sunshine_hours), 2) AS avg_sunshine_hours,
    ROUND(AVG(yeilds), 2) AS avg_yield
FROM processed.crops_weather
GROUP BY crops
ORDER BY avg_sunshine_hours DESC;
```
![alt text](<Query result (sunshine coverage).png>)
Example, to get an average of 51247 cotton per area, an average of 10.41 hours of sunshine are needed per day.  

# Crops yield based on evaporation
```
SELECT
    crops,
    ROUND(AVG(evapotranspiration_mm), 2) AS avg_evaporation_mm,
    ROUND(AVG(yeilds), 2) AS avg_yield
FROM processed.crops_weather
GROUP BY crops
ORDER BY avg_evaporation_mm DESC;
```
![alt text](<Query result (evaporation).png>)

Example, to get an average of 11301.27 of Pepper per area, an average of 5.11 mm  of evaporated water are needed. 

