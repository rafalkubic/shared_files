Task 1. Please optimize the query below according to the given goal. Data sources for a reference - tableA.com and country_mapping.csv. Explain how it is executed and how your version is better than given?

Goal:
You were approached by stakeholder who wants to know what is the languages distribution by Platform for NordVPN users in the last 90 days. He is interested only in the users that connected at least once to the service (event_name is equal to connect and event_status is equal to success). 
He also wants to understand what language is picked in what country. Unfortunately the country data is messed up (country), you can use the country mapping file to fix that (join_key is a field to go). 
Your measure should be count of unique users. Summing up:
- Only successful connections should be taken into considerations;
- Device location to be fixed;

SELECT
	A.Platform as Platform,
	B.name as Country,
	A.Language,
	COUNT(DISTINCT A.user_id) as user_count
FROM
	tableA A
LEFT JOIN country_mapping B
ON
	A.country = B.join_key
LEFT JOIN
(
SELECT
	user_id,
	MAX(MAKE_DATE(Year, Month, Day)) as latest_date
FROM
	tableA
WHERE
	MAKE_DATE(Year, Month, Day) 
	BETWEEN DATE_ADD(CURRENT_DATE(), -91) 
		AND DATE_ADD(CURRENT_DATE(), -1)
	AND event_name = 'connect'
	AND event_Status = 'success'
GROUP BY
1
) as C
 ON
A.user_id = C.user_id
WHERE
	MAKE_DATE(Year,Month,Day) 
		BETWEEN DATE_ADD(CURRENT_DATE(), -91) 
		AND DATE_ADD(CURRENT_DATE(), -1)
	AND event_name = 'connect'
	AND event_Status = 'success'
GROUP BY
1,
2,
3,
4,
5


----SOLUTION-------------


--users that connected successfully[at least once] to a service from a particular country
select 
	A.Platform as Platform,
	B.name as Country,
	A.Language,
	COUNT(DISTINCT A.user_id) as user_count
	from tableA A LEFT JOIN country_mapping B ON A.country = B.join_key
where
DATEFROMPARTS (Year,Month,Day) BETWEEN DATEADD(day, -91,getdate()) AND DATEADD(day, -1,getdate())
--we are looking for users that connected successfully at least once from a particular country
AND event_name = 'connect'
AND event_Status = 'success'
group by
A.Platform,
B.name ,
A.Language




--users that connected sucessfully to the service[in generall] (not taking country into the consideration[for some countries they may fail to connect, but in this case, they will be counted])
with last_connection_success as
(
	select distinct
	user_id,
	MAX(DATEFROMPARTS (Year, Month, Day)) over (PARTITION BY user_id) as latest_date
	from tableA
where event_name = 'connect'
AND event_Status = 'success'
AND DATEFROMPARTS (Year,Month,Day) BETWEEN DATEADD(day, -91,getdate()) AND DATEADD(day, -1,getdate())
)
	select 
	A.Platform as Platform,
	B.name as Country,
	A.Language,
	COUNT(DISTINCT A.user_id) as user_count
	from tableA A LEFT JOIN country_mapping B ON A.country = B.join_key
	inner join last_connection_success C ON A.user_id = C.user_id  --- using INNER JOIN instead of LEFT JOIN due to the fact that we want only successfuly connected users
where
DATEFROMPARTS (Year,Month,Day) BETWEEN DATEADD(day, -91,getdate()) AND DATEADD(day, -1,getdate())
--this part is not needed due to generall connection success from the "last_connection_success" data set
--AND event_name = 'connect'
--AND event_Status = 'success'
group by
A.Platform,
B.name ,
A.Language




Task 2. You hava a table (tableB.csv) with car rental events. Provide a SQL query that would answer these questions:
 - What is the most popular season to rent a Hyundai?
 
 ------SOLUTION---------------
 --in this case we can do it by using timestamp for each calculation of the season[that can put more time on the query, but the query is more clear/contains less lines]
with hyundai_rent as(
select
	YEAR(DATEADD(s, timestamp, '1970-01-01')) as [Year],
	DATEPART(QUARTER,DATEADD(s, timestamp, '1970-01-01')) as [Quarter],
	COUNT(car_vin) rents
	from tableB
	where car_make = 'Hyundai'
group by 
	YEAR(DATEADD(s, timestamp, '1970-01-01')),
	DATEPART(QUARTER,DATEADD(s, timestamp, '1970-01-01'))
)
select  top 1
	[Year],
	[Quarter],
	rents
from hyundai_rent
order by rents desc


--or we can do it by formatting the timestamp within the firs query and than calculating date parts out of that field[may be more optimized, but takes more lines of code]
with hyundai_rent as(
select  DATEADD(s, timestamp, '1970-01-01') rent_date,
		car_vin 
	from tableB
	where car_make = 'Hyundai'
),
count_rents as(
select 
		DATEPART(YEAR,rent_date) as [Year],
		DATEPART(QUARTER,rent_date) as [Quarter],
		COUNT(car_vin) as rents
	from hyundai_rent
	group by DATEPART(YEAR,rent_date),
			 DATEPART(QUARTER,rent_date)
)
select  top 1
		[Year],
		[Quarter],
		rents
		from count_rents
order by rents desc


 - What was the distribution of car brands last rented by each customer in the red segment?
 
 
 ------SOLUTION-------------
 

--last rented car by each customer in red segment only
with last_rented_car as
(
	select  distinct
			client_id,
			FIRST_VALUE(car_make) over (PARTITION BY client_id ORDER BY timestamp desc) as car_make
			--MAX(timestamp) over (PARTITION BY client_id) as timestamp
		from tableB
		where client_segment = 'red'
)
--calculate distribution by car brand
select 
		car_make,
		count(client_id) as distrib
	from last_rented_car
	group by car_make