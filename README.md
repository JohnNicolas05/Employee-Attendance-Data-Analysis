**Created by: John Nicolas**

**Created on: August 23, 2023**

## Employee Attendance Data Analysis
**Problem**

The HR department in a medical company provided a data that includes 10-15 parameters per year (arrival/departure time, vacations, sick days, time off, etc.) on 1000_ employees of the company. The HR department are asking to provide answers to the following questions, which the CEO formulated:
1.	Identify the most disciplined and undisciplined employees and divisions.
2.	Create a visualization with the analysis of weekdays and months when the most employees were late/absent (either for vacation or sick leave)
3.	Answer the following questions. Which heads of departments tend to forgive employees for lack of discipline? Are there any favorites for any heads of departments (perhaps some employees are always forgiven for being late, given time off, etc.)?

**Methodology**

&nbsp; **1.	Data Importation, Cleaning, and Transformation in PostgreSQL**

**1.A. Create a table in SQL using the query below. Import the file schedules.csv to the created table.**

SQL Syntax:

```sh
CREATE TABLE IF NOT EXISTS raw_schedules (
	type VARCHAR,
	dates VARCHAR,
	time_start TIME,
	timezone VARCHAR,
	time_planned INT,
	break_time INT,
	leave_type VARCHAR,
	user_id VARCHAR
);
```
**1.B. Import the file schedules.csv to the created table**

*Notice that we used VARCHAR data type to both ***‘dates’*** and ***‘user_id’*** because these columns contain array values and can only be inserted as a String data type.*

**1.C. Creating a new table, splitting columns into rows, changing ‘dates’ data type into date, removing duplicates.**

SQL Syntax:

```sh
CREATE TABLE clean_schedules AS
SELECT DISTINCT ON (type, user_id, dates, leave_type)
	type,
	to_date(REPLACE(REPLACE(dates_split, '[', ''), ']', ''), 'YYYY-MM-DD') AS dates,
	time_start,
	time_end,
	timezone,
	time_planned,
	break_time,
	leave_type,
	CAST(REPLACE(REPLACE(user_id_split, '{', ''), '}', '')AS VARCHAR) AS user_id
FROM raw_schedules
CROSS JOIN LATERAL unnest(STRING_TO_ARRAY(dates, ',')) AS dates_split
CROSS JOIN LATERAL unnest(STRING_TO_ARRAY(user_id, ',')) AS user_id_split
ORDER BY type, user_id, dates;
```

**1.D. Removing rows with ‘fake’ and ‘free’ from clean_schedules.**

SQL Syntax:

```sh
DELETE FROM clean_schedules
WHERE type IN ('fake', 'free');
```

**1.E. Removing rows in the analysis with ‘day_off’ from clean_schedules.**

SQL Syntax:

```sh
DELETE FROM clean_schedules
WHERE leave_type = 'day_off';
```

*We removed rows with ***‘fake’***, ***‘free’***, and ***‘day_off’*** since the focus of the analysis is employee attendance.*

**1.F. Replacing NULL values for ‘time_start’ column which ‘type’ column value is ‘leave’.**

SQL Syntax:

```sh
UPDATE clean_schedules
SET time_start = '03:00:00'
WHERE time_start IS NULL AND type = 'leave';
```

**1.G. Replacing NULL values for ‘time_end’ column which ‘type’ column value is ‘leave’.**

SQL Syntax:

```sh
UPDATE clean_schedules
SET time_start = '03:00:00'
WHERE time_end IS NULL AND type = 'leave';
```

*We added additional values for ***‘time_start’*** and ***‘time_end’*** for rows with leave values to avoid issues in filtering columns in our analysis.*

**1.H. Creating table and importing attendance.csv to PostgreSQL.**

SQL Syntax:

```sh
CREATE TABLE attendance (
	user_id VARCHAR,
	first_name VARCHAR,
	last_name VARCHAR,
	location VARCHAR,
	date DATE,
	time TIME,
	timezone VARCHAR,
	attendance_case VARCHAR,
	source VARCHAR
);
```

**1.I. Adding a new column and inserting values for attendance table.**

SQL syntax:

```sh
ALTER TABLE attendance
ADD COLUMN type VARCHAR;

UPDATE attendance
SET type = 'work';
```

*This column will help us later in merging schedules table and attendance table.*

**1.J. Creating new table ‘clean_attendance’ and removed duplicated rows.**

SQL Syntax:

```sh
CREATE TABLE clean_attendance AS
SELECT DISTINCT ON (user_id, date) * FROM attendance;
```

**1.K. Creating a new schedules table where we will include the check-in time of employees.**

SQL Syntax:

```sh
CREATE TABLE SchedulesIN AS
SELECT 
	cs.user_id, 
	cs.type, 
	cs.dates, 
	cs.time_start, 
	ca.time AS time_in, 
	ca.attendance_case, 
	cs.time_end, 
	cs.timezone, 
	cs.time_planned, 
	cs.break_time, 
	cs.leave_type
FROM clean_schedules cs
LEFT JOIN attendance ca
	ON cs.user_id = ca.user_id
	AND cs.dates = ca.date;
```

**1.L. Deleting rows with ‘OUT’ value to remove check-in time duplicates.**

SQL Syntax:

```sh
DELETE FROM SchedulesIn
WHERE type = 'work' AND attendance_case = 'OUT';
```

**1.M. Creating a new schedules table where we will include the check-out time of employees. This will now make a table where check in and check out of employees are included.**

SQL Syntax:

```sh
CREATE TABLE schedules_in_out AS
SELECT
	si.user_id, 
	si.type, 
	si.dates, 
	si.time_start, 
	si.attendance_case, 
	si.time_end, 
	ca.time AS time_out,
	ca.attendance_case AS case_attendance, 
	si.timezone, si.leave_type
FROM SchedulesIn si
LEFT JOIN clean_attendance ca
	ON si.user_id = ca.user_id
	AND si.dates = ca.date;
```

**1.N. Deleting unnecessary rows and duplicated values and creating a new table ‘final_schedules’.**

SQL Syntax:

```sh
DELETE FROM schedules_in_out
WHERE type = 'work' AND case_attendance = 'IN';

DELETE FROM schedules_in_out
WHERE type = 'work' AND time_in IS NULL;

CREATE TABLE final_schedules AS
SELECT DISTINCT ON (user_id, type, dates) * FROM schedules_in_out;
```

**1.O. Dealing with NULL values.**

*To deal with our computation later, we will change ***‘time_in’*** and ***‘time_out’*** NULL values on ***‘type’*** column with ***‘leave’*** rows with identical values that are unlikely to hinder with ‘type’ with ‘work’ value rows. The idea is to get a difference for ***‘time_start’*** and ***‘time_in’*** and for ***‘time_end’*** and ***‘time_out’*** of 0.*

SQL Syntax:

```sh
UPDATE final_schedules
SET time_in = '03:00:00'
WHERE type = 'leave';

UPDATE final_schedules
SET time_out = '03:00:00'
WHERE type = 'leave';

UPDATE final_schedules
SET attendance_case = 'LEAVE'
WHERE type = 'leave';

UPDATE final_schedules
SET case_attendance = 'LEAVE'
WHERE type = 'leave';

UPDATE final_schedules
SET timezone = '+08:00'
WHERE timezone IS NULL;
```

**1.P. Creating a table and importing ‘users.csv’**

SQL Syntax:

```sh
CREATE TABLE users(
	user_id VARCHAR,
	first_name VARCHAR,
	last_name VARCHAR,
	gender VARCHAR,
	date_birth DATE,
	date_hire DATE,
	date_leave DATE,
	employment VARCHAR.
	position VARCHAR,
	location VARCHAR,
	department VARCHAR,
	created_at TIMESTAMP
);
```

**1.Q. Deleting unnecessary columns, creating a new table, dealing with NULL values, and translating values into English.**

SQL Syntax:

```sh
CREATE TABLE clean_users AS
SELECT
	user_id,
	position, 
	location, 
	department
FROM users;

UPDATE clean_users
SET position = 'No Data'
WHERE position IS NULL;

UPDATE clean_users
SET department = 'No Data'
WHERE department IS NULL;

UPDATE clean_users
SET position = (
CASE
	WHEN position = 'Dokter' then 'Doctor'
	WHEN position = 'Perawat' then 'Nurse'
	WHEN position = 'Bidan' then 'Midwife'
	WHEN position = 'Asisten Apoteker' then 'Pharmacist Assistant'
	WHEN position = 'Direktur Utama' then 'President Director'
	WHEN position = 'Direktur Pengembangan' then 'Development Director'
	WHEN position = 'Apoteker' then 'Pharmacist'
	WHEN position = 'Admin Bisnis' then 'Business Admin'
	ELSE position
END
);
```

**1.R. Creating a new table for schedules table, merging, and adding columns.**

SQL Syntax:

```sh
CREATE TABLE schedules_attendance_users AS
SELECT 
	fs.user_id,
	cu.position, 
	cu.department, 
	fs.type, 
	fs.dates, 
	fs.time_start, 
	fs.time_in, 
	fs.attendance_case, 
	fs.time_end, 
	fs.time_out, 
	fs.case_attendance, 
	fs.timezone, 
	fs.leave_type
FROM final_schedules fs
LEFT JOIN clean_users cu
	ON fs.user_id = cu.user_id;
	
ALTER TABLE schedules_attendance_users
ADD COLUMN time_in_diff_minutes NUMERIC(10, 2);

UPDATE schedules_attendance_users
SET time_in_diff_minutes = ROUND(EXTRACT(EPOCH FROM (time_start - time_in)) / 60, 2);

ALTER TABLE schedules_attendance_users
ADD COLUMN time_out_diff_minutes NUMERIC(10, 2);

UPDATE schedules_attendance_users
SET time_out_diff_minutes = ROUND(EXTRACT(EPOCH FROM (time_end - time_out)) / 60, 2);
```

**1.S. Adding new columns to the table to identify employees that are late or on time**

SQL syntax:

```sh
ALTER TABLE schedules_attendance_users
ADD COLUMN time_in_status VARCHAR;

ALTER TABLE schedules_attendance_users
ADD COLUMN time_out_status VARCHAR;
```

**1.T. Creating a new table that contains filtered values excluding time in and time out difference of more than and less than 2 hours.**

SQL Syntax:

```sh
CREATE TABLE time_diff_schedules AS
SELECT
	user_id, 
	position,
	department,
	type,
	leave_type,
	dates,
	time_start,
	time_in,
	time_in_diff_minutes,
	time_in_status,
	time_end,
	time_out,
	time_out_diff_minutes,
	time_out_status
FROM schedules_attendance_users
WHERE (time_in_diff_minutes > -120 AND time_in_diff_minutes < 120)
AND (time_out_diff_minutes > -120 AND time_out_diff_minutes < 120);
```

**1.U. Identifying employees who came on time and late and those who checked out early and on time.**

SQL Syntax:

```sh
UPDATE time_diff_schedules
SET time_in_status = (
CASE
	WHEN time_in_diff_minutes < -10 THEN 'Late'
	WHEN time_in_diff_minutes > -10 THEN 'On Time'
	ELSE time_in_status
END
);

UPDATE time_diff_schedules
SET time_out_status = (
CASE
	WHEN time_out_diff_minutes > 0 THEN 'Early Out'
	WHEN time_out_diff_minutes <= 0 THEN 'On Time'
END
);
```

**1.V. Adding a new column that extracts day of the week.**

SQL Syntax:

```sh
ALTER TABLE time_diff_schedules
ADD COLUMN dayoftheweek VARCHAR;

UPDATE time_diff_schedules
SET dayoftheweek = TO_CHAR(dates::DATE, 'Day');
```

&nbsp; **2.	Data Analysis and Visualization Using Power BI**

**2.A. Export the cleaned data from PostgreSQL to Power BI**

**a. Renaming the time_diff_schedules into SchedulesClean and importing to Power BI**

![Alt text](https://github.com/JohnNicolas05/Employee-Attendance-Data-Analysis/blob/main/Employee-Attendance-Data-Analysis/assets/renaming-time-diff-schedules-into-schedulesclean.png)
![Alt text](https://github.com/JohnNicolas05/Employee-Attendance-Data-Analysis/blob/main/Employee-Attendance-Data-Analysis/assets/renaming-time-diff-schedules-into-schedulesclean2.png)

**b. Importing attendance.csv and cleaning the data set using Power BI Power Query tool.**

- **Removing insignificant columns:**
  
![Alt text](https://github.com/JohnNicolas05/Employee-Attendance-Data-Analysis/blob/main/Employee-Attendance-Data-Analysis/assets/removing-insignificant-columns.png)

- **Replacing NULL values with ‘No Data’**
  
![Alt text](https://github.com/JohnNicolas05/Employee-Attendance-Data-Analysis/blob/main/Employee-Attendance-Data-Analysis/assets/replacing%20null%20with%20nodata1.png)

- **Removing duplicate rows**
  
![Alt text](https://github.com/JohnNicolas05/Employee-Attendance-Data-Analysis/blob/main/Employee-Attendance-Data-Analysis/assets/removing-dup-rows1.png)

- **Changing user_data type from numeric to string**
  
![Alt text](https://github.com/JohnNicolas05/Employee-Attendance-Data-Analysis/blob/main/Employee-Attendance-Data-Analysis/assets/changing-user-data-type-from-numeric-to-string.png)

**c. Exporting users_clean and renaming into UsersClean, and importing leave_requests.csv, payroll.csv, and UsersClean to Power BI.**

![Alt text](https://github.com/JohnNicolas05/Employee-Attendance-Data-Analysis/blob/main/Employee-Attendance-Data-Analysis/assets/c..png)

**2.B. Getting Total Working Days.**

![Alt text](https://github.com/JohnNicolas05/Employee-Attendance-Data-Analysis/blob/main/Employee-Attendance-Data-Analysis/assets/2.b..png)

Using Card visual we used TotalWorkDays calculated measurement to visualize the total number of working days.

DAX Formula:

```sh
TotalWorkDays = COUNT(SchedulesClean[dates])
```

**2.C. Getting Attendance Rate.**

![Alt text](https://github.com/JohnNicolas05/Employee-Attendance-Data-Analysis/blob/main/Employee-Attendance-Data-Analysis/assets/2.c..png)

Creating a Calculations table to store all the calculated measurement and creating a new measure to calculate the attendance rate using DAX.

DAX Formula:

```sh
AttendanceRate = ([TotalWorkDays] - [TotalLeaveDays]) / [TotalWorkDays]
TotalWorkDays = COUNT(SchedulesClean[dates])
TotalLeaveDays = COUNTROWS(FILTER(SchedulesClean,'SchedulesClean'[type] = "leave"))
```

**2.D. Getting Late Rate**

![Alt text](https://github.com/JohnNicolas05/Employee-Attendance-Data-Analysis/blob/main/Employee-Attendance-Data-Analysis/assets/2.d..png)

Using a Card visual, we used the LateRate calculated measurement to get the late rate.

DAX Formula:

```sh
LateRate = [TotalNumberOfLates] / [TotalWorkDays]
TotalNumberOfLates = COUNTROWS(FILTER(SchedulesClean, SchedulesClean[type] = "work" && SchedulesClean[time_in_status] = "Late"))
TotalLeaveDays = COUNTROWS(FILTER(SchedulesClean,'SchedulesClean'[type] = "leave"))
```

**2.E. Getting Total Number of Lates and Early Outs per Day of the Week.**

![Alt text](https://github.com/JohnNicolas05/Employee-Attendance-Data-Analysis/blob/main/Employee-Attendance-Data-Analysis/assets/2.e..png)

Using a Stacked Column Chart, we use the dayoftheweek column from SchedulesClean as the X-axis and TotalNumberOfLates and TotalNumberOfEarlyOut calculations for the Y-axis.

DAX Formula:

```sh
TotalNumberOfLates = COUNTROWS(FILTER(SchedulesClean, SchedulesClean[type] = "work" && SchedulesClean[time_in_status] = "Late"))
TotalNumberOfEarlyOut = COUNTROWS(FILTER(SchedulesClean,'SchedulesClean'[type] = "work" && SchedulesClean[time_out_status] = "Early Out"))
```
