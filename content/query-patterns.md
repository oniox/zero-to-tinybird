# Query patterns

A collection of SQL patterns. A WIP! 

These SQL examples are built with [two simple schemas](#data-schema) that describe the data sets using in the workshop. 

## Working with time

Real-time event data are based on date and time objects. When working with real-time events, the time of when the event happened is integral to vast majority of all questions asked of the data. How many times has something happened in that month, day, hour, minute, or second? How have the temporal patterns of an event evolved and formed over time?  

So, most of the queries you build in Tinybird will have time details built into them. To gain a wide prespective of how to work with timetamps in Tinybird, these guides are critical:
* https://www.tinybird.co/docs/guides/working-with-time.html
* https://www.tinybird.co/docs/guides/best-practices-for-timestamps.html

Note:
* The sooner you standardize on the UTC timezone for all things time, the better. Let the client display composer convert to local time if needed. If you are working with data generators, push the data already in UTC. For example, with Python, be sure to generate timestamps in UTC (`current_time = datetime.datetime.utcnow()`). 
* ClickHouse provides a set of time functions that make working with time data easier. Like most data environments, ClickHouse supports both Date and DateTime objects. Check out [this ClickHouse guide](https://clickhouse.com/docs/en/sql-reference/functions/date-time-functions) for the details (tere is also this ClickHouse [blog post](https://clickhouse.com/blog/working-with-time-series-data-and-functions-ClickHouse). Functions such as [toStartOfDay](https://clickhouse.com/docs/en/sql-reference/functions/date-time-functions#tostartofmonth) are fantastic for binning and processing time-series data. If you have ever needed to bin time data in SQL, you will really appreciate these functions.
* Your Tinybird will be configured to store data in UTC. 
  

### Data from the most recent hour
```sql
SELECT symbol, timestamp, price  
FROM event_stream
WHERE toDateTime(date) BETWEEN addHours(NOW(),-1) AND NOW()
```

### Data between explicit dates
```sql
SELECT symbol, timestamp, price  
FROM event_stream
WHERE toDateTime(date) BETWEEN '2023-12-07 17:22:00' AND '2023-12-07 17:23:00'
```

### Data from yesterday, midnight to midnight

This SQL uses ClickHouse `today`,`yesterday`, and `toStartofDate` functions.

```sql
SELECT symbol, timestamp, price 
FROM event_stream
WHERE
  timestamp >= toStartOfDay(toDateTime(yesterday()))
  AND timestamp < toStartOfDay(toDateTime(today()))
LIMIT 10000
```




### Selecting most recent data 

To detect timeouts, the most recent data point from each sensor is looked up. This sounds simple enough, and it is one of the most simple detection methods. Like many things, there is more than one way to look most recent data.    

The following query sorts the data in reverse chronological order, and uses the `LIMIT BY` command to look up the most recent single (LIMIT 1) data for each sensor.

```sql

SELECT * 
FROM event_stream
ORDER BY timestamp DESC
LIMIT 1 BY id

```

Here is another SQL query for looking up the most recent data, which scans over partitions and also scans in reverse chronological order.  

```sql
WITH RankedData AS (
    SELECT
        id,
        timestamp,
        value,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY timestamp DESC) AS row_num
    FROM
        event_stream
)
SELECT
    id,
    timestamp,
    value
FROM
    RankedData
WHERE
    row_num = 1
```

These queries do not narrow down the time range of interest. With anomaly detection and real-time data, we typically have a set of *time windows* of interest. In general, our time range of interest ranged from data from the most recent 10 seconds, to generating statistics over the previous thirty minutes.

### Scanning periods of interext

If you have a data source with sensors that normally report every few seconds, it's likely that going ten seconds without a new report is a sign that a sensor is off-line. 

To focus only on the last ten seconds of data the following WHERE clause can be added to each query (inside the inner RankedData query for the second example). 

```sql
WHERE timestamp > NOW() - INTERVAL 10 SECOND
```

When it comes to *timeout* anomalies, we wanted a system that would detect if a sensor stopped reporting in the past minute. Our data source emits data every few seconds for each sensor, so we started off with a system that would identify sensors that had not reported in the last minute. 

We also wanted to provide an API Endpoint for ad hoc timeout checks that supported the following query parameters:

* **seconds** - The 'timeout' duration in seconds. How many seconds since a sensor reporting gets you worried? 
* **time_window_minutes** - How many minutes to 'look back' from 'now'.
* **sensor_id** - Test on a sensor-by-sensor basis.

This [example Pipe file](https://github.com/tinybirdco/anomaly-detection/blob/main/data-project/pipes/timeout.pipe) illustrates how to build these dynamic parameters into the queries. 

### Providing flexible query parameters

Tinybird provides a *templating* syntax for building *dynamic* query parameters into your Tinybird API Endpoints. As you design your API endpoints, it is important to consider how your users will want to specify important attributes of the objects your API is serving. 

When it comes to time, users typically want to pick a time *period* of interest, defined by start and end times. Four common modes for time requests include having the following start and end times provided:

+ Both start and end times. The user has specific time period of interest, and typically uses a second resolution. 
+ No start and end times. The most recent data is usually implied and it's up to the server to enforce limits. This usually means declaring a maximum duration to server, e.g. the last day, week, or month. Some maximum number of objects to return is recommended. 
+ Only start time. The user has a specific beginning time in mind, the beginning of some event of interest. 
+ Only end time. The user has a specific end of an event in mind, and wants data leading up to it. 

Here is a example template for providing these time request conventions. Here we support the convention of serving a maximum of 30 days if the user does not specify a start or end time. 

```sql
{% if defined(start_time) and defined(end_time) %}
   AND toDateTime(timestamp) BETWEEN parseDateTimeBestEffort({{ DateTime(start_time, description="'YYYY-MM-DD HH:mm:ss'. UTC. Optional and defaults to 30 days ago. Defines the start of the period of interest. ") }}) AND parseDateTimeBestEffort({{ DateTime(end_time, description="'YYYY-MM-DD HH:mm:ss'. UTC. Optional and defaults to time of request. Defines the end of the period of interest.") }})
{% end %}

{% if not defined(start_time) and not defined(end_time) %}
   AND toDateTime(timestamp) BETWEEN addDays(now(),-30) AND now()
{% end %}

{% if defined(start_time) and not defined(end_time) %}
   AND toDateTime(timestamp) BETWEEN parseDateTimeBestEffort({{ DateTime(start_time) }}) AND addDays(toDateTime(parseDateTimeBestEffort({{DateTime(start_time)}}),30)
{% end %}

{% if not defined(start_time) and defined(end_time) %}
   AND toDateTime(timestamp) BETWEEN addDays(toDateTime(parseDateTimeBestEffort({{DateTime(end_time)}}),-30) AND parseDateTimeBestEffort({{ DateTime(end_time) }})
{% end %}
```

p.s. Always use UTC. 

## JOIN patterns

Tinybird is built to help users unify their data sources. Our users want to combine their data sources in Tinybird so they can blend it all into their analysis and output. One of the most fundamental use cases for Tinybird is *enriching real-time data* with dimensional data. This is all made possible by *joining* the data. When you *JOIN* data in SQL you are linking, mapping, and associating common data attributes from two or more sources. 

```sql
SELECT es.timestamp, ci.symbol, es.price, ci.name, ci.sector
FROM company_info ci, event_stream es
WHERE ci.symbol = sps.symbol
LIMIT 100
```

```sql
SELECT es.timestamp, ci.symbol, es.price, ci.name, ci.sector
FROM company_info ci
JOIN event_stream es
ON ci.symbol = es.symbol
LIMIT 100
```

## Calculating slope

Developing a recipe for calculating data slopes or rates of change is surprisingly complicated (at least to me). To calculate the slope of two consectutive data points depends on *window* functions. The recipe below depends on the ClickHouse `lagInFrame` function (See this discussion on [ClickHouse window functions](https://clickhouse.com/docs/en/sql-reference/window-functions#functions)), which requires the construction of time window specification, e.g. `(PARTITION BY symbol ORDER BY date ASC ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING)`.

 (PARTITION BY partition_columns_list [ORDER BY order_by_columns_list] frame_specification)


```sql
%
{% set time_window_minutes=30 %}
{% set max_slope=3 %}

SELECT symbol, 
date, 
previous_timestamp,
(amount - previous_amount) / (date - previous_date) as slope,
amount, 
previous_amount,
(amount - previous_amount) as amount_diff,
(date - previous_date) as time_diff,
{{Int16(max_slope, 3, description="Integer. Maximum slope, any higher than this are returned.")}} as max_slope,
lagInFrame(timestamp, 1) OVER 
(PARTITION BY symbol ORDER BY date ASC ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING) AS previous_date, 
lagInFrame(amount, 1) 
OVER (PARTITION BY symbol ORDER BY date ASC ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING) AS previous_amount
FROM stock_price_stream
WHERE date > NOW() - INTERVAL time_window_minutes MINUTE
  ORDER BY date DESC

```

## Anomaly detection with SQL

### Interquartile Range

Based on a time window (defaulting here to a 10-minute window), calculate the lower quartile, the medium, and the upper quartile. The IQR is then set to (uper quartile - lower quartile) * 1.5.

Based on the IQR, lower and upper bounds are set for determining data outliers:
* lower bound = lower quartile - IQR
* upper bound = upper quartile - IQR


```sql
WITH stats AS (SELECT symbol
quantileExact(0.25) (price) AS lower_quartile,
quantileExact(0.5) (price) AS mid_quartile,
quantileExact(0.75) (price) AS upper_quartile,
(upper_quartile - lower_quartile) * 1.5 AS IQR
FROM event_stream
WHERE timestamp >= toDate(NOW()) - INTERVAL 10 MINUTES
GROUP BY symbol
LIMIT 10
)
 SELECT DISTINCT timestamp, 
    symbol, 
    price, 
    ROUND((stats.lower_quartile - stats.IQR),2) AS lower_bound, 
    ROUND((stats.upper_quartile + stats.IQR),2) AS upper_bound 
 FROM event_stream
 JOIN stats ON event_stream.symbol = stats.symbol
 WHERE timestamp >= toDate(NOW()) - INTERVAL 10 MINUTES
 AND (price > (stats.upper_quartile + stats.IQR)
 OR price < (stats.lower_quartile - stats.IQR))
 ORDER BY timestamp DESC
```

### Z-Score

This implements a simple algorith based on a time-series average and standard deviation over a minute-scale window of data. Each incoming data point, x, has a Z-Score calculated in this way:  

`zscore = (x - avg) / stddev`

Currently, this Pipe is based on two time windows: 
First, the statistics are calculated across the `_stats_time_window_minutes`.
Second, anomalies are scanned for using the `_anomaly_scan_time_window_seconds` window.

The `zscore_multiplier` parameter was added recently and defaults to 2. 
These parameters, could be promoted to API Endpoint query parameters.

- [ ] This recipe depends on two queries. The first calculates the Z-score, the second show an example for comparing with a absolute Z-score to identify anomalies. 

```sql
%
{% set _stats_time_window_minutes=10 %}
{% set _anomaly_scan_time_window_seconds=30 %}

WITH stats AS (
    SELECT symbol,
        avg(price) AS average,
        stddevPop(price) AS stddev
    FROM event_stream
    WHERE date BETWEEN NOW() - INTERVAL _stats_time_window_minutes MINUTE AND NOW()
    GROUP BY symbol  
)
SELECT es.date, 
     es.symbol, 
     es.amount, 
     (es.price - stats.average)/stats.stddev AS zscore,
     stats.average,
     stats.stddev
FROM event_stream es
JOIN stats s ON s.symbol = es.symbol
WHERE date BETWEEN NOW() - INTERVAL _anomaly_scan_time_window_seconds SECOND AND NOW()
ORDER BY date DESC
```

### Comparing data with thresholds


```sql
SELECT * 
FROM event_stream
WHERE amount < 0.05 OR amount > 0.95
LIMIT 10
```

## Data schema

These queries have been constructed in reference to these two schemas:

`event_stream` - a real-time stream of stock price events generated with Mockingbird and written to the Events API. 

```
`price` Float32 `json:$.price` ,
`timestamp` DateTime `json:$.timestamp` ,
`symbol` String `json:$.symbol` ,
```
`company_info` - Mock data about ~85 fictional companies. 

```
`symbol` String,
`name` String,
`creation_date` Date,
`sector` String,
```


# Other things

## Endpoint output objects

When pulling these objects from API Endpoints, here is what these `stock price" and "comapny info" objects look like: 

```json
{
    "symbol": "TTM",
    "name": "TechTrek Manufacturing",
    "creation_date": "2006-09-18",
    "sector": "Manufacturing"
}

```

```json
{
    "amount": 0.06580348,
    "date": "2023-12-04 16:40:10",
    "stock_symbol": "TTM"
}
```
## SQL helpers 

Formating numbers, trimming to 2 digits. 
`ROUND(x,2)`

Renaming attributes.
`MAX(price) AS max_price`

Comparing strings.
`LOWER(thisCityName) = 'london'` 
`UPPER(thisCityName) = 'LONDON'` 



## Renaming objects along the pipeway

* When generated.
* In Data Source defintion.
* In SQL queries.

