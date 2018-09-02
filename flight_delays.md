
# Flight Delays

## Instructions
Download the dataset https://www.kaggle.com/usdot/flight-delays/data (you need a Kaggle account)
Instructions: Using the dataset above answer below questions
1. Find the correlation between route distance and flight delays.
2. Which airports are part of the most delayed flights (either source or destination)?
3. Which airline should you fly on to avoid significant delays?
4. Are there any interesting observations you can make from the dataset?

## Deliverables:
Include source codes for all the questions with a ReadMe file.
You can use sql, python or any other technology you prefer.
What counts:
* Working Solution
* Simplicity/Cleanliness of solution

## Solutions
### 1. Find the correlation between route distance and flight delays 
My initial hypotheses/questions are noted below on the correlation of route distance and flight delays. They were as follows:
1. As flight distance increases, flight delays will increase
2. As flight distance increases, the length of delay will increase
3. What is considered a delayed flight?

I decided that the best way to aggregate the results of this query was to group the flight distances into 500 mile increments and perform calculations over those increment lengths. The key metric in finding the correlation was the percent of flights delayed grouped by length of flight. As you'll notice when running the results, the percentage of delays increases as the flight distance increases. This confirms hypothesis number one. In the extremely long range flights of 1,500 miles and up, flight delays decreased slightly, but still an overall increase from the shorter flights (0-1000 miles). Regarding what is considered a delayed flight, research indicated that it was any time the flight is delayed over 15 minutes.

#### Query 1
```
select x.distance
        , round(sum(delayed)) as total_delayed
        , count(x.delayed) total
        , round(sum(delayed) * 100.0 / count(x.delayed),2) as departure_delay
from (select case when (distance > 0 and distance < 500) then '1.0-500'
            when (distance >= 500 and distance < 1000) then '2.500-1000'
            when (distance >= 1000 and distance < 1500) then '3.1000-1500'
            when (distance >= 1500 and distance < 2000) then '4.1500-2000'
            when (distance >= 2000 and distance < 2500) then '5.2000-2500'
            else "6.>2500" end as distance
            , case when departure_delay > 15 then 1 else 0 end as delayed
            , departure_delay
            from flights
            order by 1) x
            group by 1
```

### 2. Which airports are part of the most delayed flights (either source or destination)?
The airports that are part of the most delayed flights can be found by creating a boolean value (0 or 1) for when a departure delay is greater than 0. This boolean value can then be summed to find the count of delays. Once the count of delays is calculated, you can then find the percentage of delays for each airport. As you'll notice in the result set Chicago O'Hare appears to have the highest count of delays as well as a very high delay percentage. This is partially due to the fact that O'Hare has a large volume of flights, but in addition to that, they still boast one of the highest departure delay percentages in the United States. I filtered the sets to airports with greater than 2,000 flights. To investigate further, I was curious as to if the weather was the major impact in causing these delays. I created a second query to understand the major cause of these delays which surprisingly had the largest delays resulting from Airline Delays.

#### Query 1
```
select *
from (
select x.origin_airport as ap
        , sum(departure_delay_count) count_dep_delays
        , count(origin_airport) total_flights
        , round(sum(departure_delay_count) * 100.0 / (count(x.origin_airport)),2) as departure_delay_pct
        
from (
select origin_airport
        , case when departure_delay > 15 then 1 else 0 end as departure_delay_count
        , case when arrival_delay > 15 then 1 else 0 end as arrival_delay_count
from flights f
) x
group by 1
order by 2 desc) y
where total_flights > 2000
```
#### Query 2
```
select case when air_system_delay > 15 then 'air_system'
                    when security_delay > 15 then 'security'
                    when airline_delay > 15 then 'airline'
                    when late_aircraft_delay > 15 then 'late_aircraft'
                    when weather_delay > 15 then 'weather'
                    else 'no_delay'
                    end as departure_delay_type
                , count(*) as count
                , avg(departure_delay) avg_delay
                , avg(distance) avg_distance
                from flights
                where origin_airport = 'ORD'
                group by 1
                order by 2
```
### 3. Which airline should you fly on to avoid significant delays?
This question was solved by creating a query in which we grouped by airline and averaged the length of delays per flight. The results noted that Alaska Airlines had the lowest avg delay time as well as the lowest percent of delays. I reduced the results to only airlines with greater than 2,000 flights.

#### Query 1
```
select *
from (select airline
        , sum(departure_delay_count) as delay_count
        , count(airline) as count_flight
        , round(100*sum(departure_delay_count)/count(airline), 2) as delay_pct
        , avg(departure_delay) as avg_dep_delay
        , avg(arrival_delay) as avg_arr_delay
from
(select airline
        , case when departure_delay > 15 then 1 else 0 end as departure_delay_count
        , departure_delay
        , arrival_delay
from flights f
) f

group by 1
order by 4 asc)
where count_flight > 2000
```

### 4. Are there any interesting observations you can make from the dataset?
There were a lot of interesting data points in this set. I have outlined the points and queries below.

#### Query 1
To further my investigation into O'Hare, I wanted to see if the time of year had an impact on weather delays. I queried by month the amount of weather related delays and noticed that the winter months saw an uptick in delays.
```
select month
        , sum(weather_delay) as weather
from(
select month
        ,case when weather_delay > 15 then 1 else 0 end as weather_delay
from flights
) group by 1
order by 1
```
#### Query 2
I was surprised to see that the majority of delays were related to 'air_system', 'late_aircraft', and 'airline', contrary to my initial theory that most delays were related to weather.
```
select *
from (select case when air_system_delay > 15 then 'air_system'
                    when security_delay > 15 then 'security'
                    when airline_delay > 15 then 'airline'
                    when late_aircraft_delay > 15 then 'late_aircraft'
                    when weather_delay > 15 then 'weather'
                    else 'no_delay'
                    end as departure_delay_type
                , count(*) as count
                , (select count(*) from flights) as count_all
                , count(*) * 100.0 / (select count(*) from flights) as pct_of_delay
                , avg(departure_delay) avg_delay
                , avg(distance) avg_distance
                from flights
                group by 1
                order by 2 desc)
                where departure_delay_type <> 'no_delay'
```
#### Query 3
I was interested in seeing that Friday was the day in which had the most delays of any day of the week.
```
select day_of_week
        , sum(departure) as departure_delays
from(
select day_of_week
        ,case when departure_delay > 15 then 1 else 0 end as departure
from flights
) group by 1
order by 1
```

#### Query 4
It appears as though mornings are the best time to fly if you'd like to avoid delays.
```
select distance
        , sum(delayed) total_delays
        , count(*) total
        , sum(delayed) * 100.0 / (count(*)) as pct_of_delay
from(select case when (distance > 0 and distance < 400) then '1.early_morning'
            when (distance >= 400 and distance < 800) then '2.mid_morning'
            when (distance >= 800 and distance < 1200) then '3.late_morning'
            when (distance >= 1200 and distance < 1600) then '4.afternoon'
            when (distance >= 1600 and distance < 2000) then '5.early_night'
            else "night" end as distance
            , case when departure_delay > 0 then 1 else 0 end as delayed
from flights
) 
group by 1
order by 1
```

## Code
```
import pandas as pd
flights = pd.read_csv('flights.csv')
import pandasql as ps
q1 = """PLACE QUERY HERE"""
print(ps.sqldf(q1, locals()))
```
