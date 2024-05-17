# Virtual Kitchen Hours Comparison

This project compares business hours between Uber Eats and Grubhub for virtual kitchens. The goal is to determine if the business hours on both platforms are in range, out of range, or out of range with a 5-minute difference.

## Table of Contents
- [Platform and Database](#Platform-and-Database)
- [Project Structure](#Project-Structure)
- [Functions](#Functions)
- [Queries](#Queries)
- [Results](#Results)

## Platform and Database

This project uses Google BigQuery for data processing and analysis. The specific dataset used for this project is:

- **Dataset Name**: `arboreal-vision-339901.take_home_v2`
- **Tables**:
  - `virtual_kitchen_ubereats_hours`
  - `virtual_kitchen_grubhub_hours`

You can access Google BigQuery [here](https://cloud.google.com/bigquery).


## Project Structure

- **Functions**: Contains custom SQL functions to parse and extract data.
- **Queries**: Contains SQL queries to extract and process data.

## Functions

### jsonObjectKeys

Extracts keys from a JSON object.

```
CREATE TEMP FUNCTION jsonObjectKeys(input STRING)
RETURNS ARRAY<STRING>
LANGUAGE js AS """
  return Object.keys(JSON.parse(input));
""";
```
###extractJsonValue

Extracts a value from a JSON object using a key.

```
CREATE TEMP FUNCTION extractJsonValue(input STRING, key ARRAY<STRING>)
RETURNS STRING
LANGUAGE js AS """
  var obj = JSON.parse(input);
  if (key in obj) {
    return JSON.stringify(obj[key]);
  } else {
    return null;
  }
""";
```

###parseRegularHours

Parses regular hours from Uber Eats data.

```
CREATE TEMP FUNCTION parseRegularHours(input STRING)
RETURNS ARRAY<STRUCT<day STRING, start_time STRING, end_time STRING>>
LANGUAGE js AS """
  var hours = JSON.parse(input);
  var days = ['Sunday', 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday'];
  var result = [];
  for (var i = 0; i < hours.length; i++) {
    for (var j = 0; j < hours[i].daysBitArray.length; j++) {
      if (hours[i].daysBitArray[j]) {
        result.push({
          day: days[j],
          start_time: hours[i].startTime,
          end_time: hours[i].endTime
        });
      }
    }
  }
  return result;
""";
```

###parseGrubhubBusinessHours

Parses business hours from Grubhub data.

```
CREATE TEMP FUNCTION parseGrubhubBusinessHours(input STRING)
RETURNS ARRAY<STRUCT<day STRING, start_time STRING, end_time STRING>>
LANGUAGE js AS """
  var hours = JSON.parse(input);
  var result = [];
  if (hours) {
    for (var i = 0; i < hours.length; i++) {
      var daysOfWeek = hours[i].days_of_week;
      var startTime = hours[i].from;
      var endTime = hours[i].to;
      if (daysOfWeek) {
        for (var j = 0; j < daysOfWeek.length; j++) {
          result.push({
            day: daysOfWeek[j],
            start_time: startTime.substring(0, 5),
            end_time: endTime.substring(0, 5)
          });
        }
      }
    }
  }
  return result;
""";

```
##Queries

###Extract Menus

Extracts menus JSON from Uber Eats responses.
```
WITH extracted_menus AS (
  SELECT 
    JSON_EXTRACT(response, '$.data.menus') AS menus_json,
    response,
    vb_name
  FROM 
    `arboreal-vision-339901.take_home_v2.virtual_kitchen_ubereats_hours`
  WHERE 
    JSON_EXTRACT(response, '$.data.menus') IS NOT NULL
),
```
###Get First Menu Key

Gets the first menu key from the JSON .
```
first_menu_key AS (
  SELECT 
    vb_name,
    jsonObjectKeys(TO_JSON_STRING(menus_json)) AS first_menu_key,
    menus_json
  FROM 
    extracted_menus
),
```
###Extract Menu Data

Extracts the menu data using the first menu key .

```
menu_data AS (
  SELECT
    em.response,
    fk.first_menu_key,
    em.vb_name,
    extractJsonValue(TO_JSON_STRING(em.menus_json), fk.first_menu_key) AS menu_data_json
  FROM 
    extracted_menus em
  JOIN 
    first_menu_key fk
  ON 
    TO_JSON_STRING(em.menus_json) = TO_JSON_STRING(fk.menus_json)
),
```
Extract Regular 
HoursExtracts regular hours from the menu data.

```
regular_hours AS (
  SELECT
    vb_name,
    menu_data_json,
    JSON_EXTRACT(menu_data_json, '$.sections[0].regularHours') AS regular_hours_json
  FROM
    menu_data
  WHERE
    JSON_EXTRACT(menu_data_json, '$.sections[0].regularHours') IS NOT NULL
),
```

###Uber Eats Hours

Unnests and parses the regular hours for Uber Eats.
```
uber_eats_hours AS (
  SELECT
    ue.slug AS uber_eats_slug,
    ue.vb_name,
    ue.b_name,
    business_hours.*
  FROM
    regular_hours,
    UNNEST(parseRegularHours(CAST(regular_hours_json AS STRING))) AS business_hours,
    `arboreal-vision-339901.take_home_v2.virtual_kitchen_ubereats_hours` as ue
),
```
###Grubhub Hours
Extracts and parses the business hours for Grubhub.
```
grubhub_hours AS (
  SELECT
    slug AS grubhub_slug,
    vb_name,
    b_name,
    JSON_EXTRACT(response, '$.availability_by_catalog.STANDARD_DELIVERY.schedule_rules') AS schedule_rules_json,
    parseGrubhubBusinessHours(TO_JSON_STRING(JSON_EXTRACT(response, '$.availability_by_catalog.STANDARD_DELIVERY.schedule_rules'))),
    business_hours.*
  FROM
    `arboreal-vision-339901.take_home_v2.virtual_kitchen_grubhub_hours`,
    UNNEST(parseGrubhubBusinessHours(TO_JSON_STRING(JSON_EXTRACT(response, '$.availability_by_catalog.STANDARD_DELIVERY.schedule_rules')))) AS business_hours
)
```
###Join and Compare Hours
Joins Uber Eats and Grubhub hours and determines if they are in range or out of range.
```
SELECT
  gh.grubhub_slug,
  ue.uber_eats_slug,
  gh.day,
  gh.start_time AS Grubhub_start_time,
  gh.end_time AS Grubhub_end_time,
  ue.start_time AS UberEats_start_time,
  ue.end_time AS UberEats_end_time,
  CASE
    WHEN PARSE_TIME('%H:%M', gh.start_time) &gt;= PARSE_TIME('%H:%M', ue.start_time)
         AND PARSE_TIME('%H:%M', gh.end_time) &lt;= PARSE_TIME('%H:%M', ue.end_time)
         THEN 'In Range'
    WHEN ABS(TIME_DIFF(PARSE_TIME('%H:%M', gh.start_time), PARSE_TIME('%H:%M', ue.start_time), MINUTE)) &lt;= 5
         OR ABS(TIME_DIFF(PARSE_TIME('%H:%M', gh.end_time), PARSE_TIME('%H:%M', ue.end_time), MINUTE)) &lt;= 5
         THEN 'Out of Range with 5 mins difference'
    ELSE 'Out of Range'
  END AS is_out_of_range
FROM
  grubhub_hours gh
JOIN
  uber_eats_hours ue
ON
  ue.vb_name = gh.vb_name AND ue.b_name = gh.b_name;
```
##Results

The final query compares business hours between Uber Eats and Grubhub for virtual kitchens and classifies them as either "In Range," "Out of Range with 5 mins difference," or "Out of Range."
