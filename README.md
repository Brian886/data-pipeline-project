# Name: Chang Jia Qian
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
<img width="921" height="484" alt="image" src="https://github.com/user-attachments/assets/7ff7ee4d-ee8d-419d-a3b9-f0ec48403a3c" />

S3 bucket was created to store data 


# Step 2: Create folders within S3 bucket
<img width="693" height="440" alt="image" src="https://github.com/user-attachments/assets/18740f73-e7ce-4e18-ad65-b3f4293f0fe8" />

Folders was created to seperate raw data and data that were processed. 


# Step 3: Upload CSV file to S3 bucket
<img width="1336" height="336" alt="image" src="https://github.com/user-attachments/assets/70085f0c-4f47-4ee9-a7a2-062843b9f139" />


# Step 4: Create a Glue Database
<img width="1072" height="212" alt="image" src="https://github.com/user-attachments/assets/d67e9a09-c1d8-458c-8883-144191507ca8" />

A logical container for metadata. This will be helpful when storing data that can be used to do analysis on.


# Step 5: Create crawler and run it
<img width="1056" height="318" alt="image" src="https://github.com/user-attachments/assets/1c1598db-8e96-4733-acd9-9a0e0a11cf60" />

Use it to extract the data


# Step 6: Query Data using Athena
```
SELECT *
FROM raw_historical
LIMIT 10;
```
<img width="948" height="485" alt="image" src="https://github.com/user-attachments/assets/8f0760a2-5212-48d1-9e89-8355954220d2" />

This is to test if the crawler could work well. 


# Step 7: Create ETL Job
<img width="1086" height="290" alt="image" src="https://github.com/user-attachments/assets/d6d75cb5-c310-4e7b-b85e-3d2c5c40f8e3" />

ETL job is created to clean and flatten the data


# Step 8: Visual creation in ETL Job
<img width="518" height="380" alt="image" src="https://github.com/user-attachments/assets/de68593c-b49d-4e56-b89c-d0cfdbb15cd3" />

This part shows how ETL job is cleaning the data by changing the format of the data deriving new column. Duplicates were removed too


# Step 9: Successfully Run ETL job
<img width="1035" height="510" alt="image" src="https://github.com/user-attachments/assets/a0172026-985b-4bde-a891-c076054e52b4" />


# Step 10: Create Lambda function
<img width="892" height="418" alt="image" src="https://github.com/user-attachments/assets/96a74a3b-e5bc-4e3d-8e50-205cf41193dc" />

This is created to help retrieve data from the weather api.


# Step 11: Create EventBridge
<img width="881" height="434" alt="image" src="https://github.com/user-attachments/assets/f63f1e07-9f0c-46d9-a494-95d772924d42" />

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
<img width="703" height="241" alt="image" src="https://github.com/user-attachments/assets/1f839214-14e2-4a53-9678-88762c8342dd" />

This code helps to retreive up to date data about temperature and precipitation. 


# Step 13: Create Crawler for weather API
<img width="439" height="268" alt="image" src="https://github.com/user-attachments/assets/b2471f5a-494a-491f-b372-5f69006a7772" />

To retrieve data from S3 and store it in a database


# Step 14: Table created from weather crawler
<img width="723" height="399" alt="image" src="https://github.com/user-attachments/assets/7da9c318-2727-49c7-a360-e4b2ff032466" />

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
<img width="935" height="413" alt="image" src="https://github.com/user-attachments/assets/d077398b-8234-48a3-a266-154300b81cc3" />


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
<img width="713" height="487" alt="image" src="https://github.com/user-attachments/assets/a7f70df2-08ad-461f-9918-22f67fc212ac" />

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
<img width="768" height="489" alt="image" src="https://github.com/user-attachments/assets/7452aea5-5aa1-4114-8ebc-87f0904d4d43" />

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
<img width="867" height="458" alt="image" src="https://github.com/user-attachments/assets/2b3e2b2f-49fc-43fb-9918-7072feb554ba" />

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

<img width="758" height="290" alt="image" src="https://github.com/user-attachments/assets/b7e8bd62-68ab-449c-9d6e-ed077395c1cc" />

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
<img width="819" height="491" alt="image" src="https://github.com/user-attachments/assets/44a8c1a7-b66d-4336-8ee1-687aeeb6baa3" />

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
<img width="838" height="494" alt="image" src="https://github.com/user-attachments/assets/f63d9f4e-4e91-4953-adc8-81fc6f1c854c" />

Example, to get an average of 11301.27 of Pepper per area, an average of 5.11 mm  of evaporated water are needed. 

