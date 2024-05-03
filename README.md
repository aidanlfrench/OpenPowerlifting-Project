# 🏋️‍♂️ OpenPowerlifting Mini Project

## What is this?

OpenPowerlifting.org provides a dataset containing competition results for dozens of federations worldwide. Find the latest dataset at https://openpowerlifting.gitlab.io/opl-csv/bulk-csv.html

The version I am using was updated as of March 30, 2024.

I am documenting my first mini-project using SQL.
This stemmed from me wondering if there is an amount to increase weight by for each attempt during a typical powerlifting competition.

***

Due to the size of the CSV file, I cannot use the Import Flat File function in SSMS. So, let's create our table and use BULK INSERT to put the information from
the CSV in it.

```sql
CREATE TABLE OpenPowerlifting.dbo.Lifter_Data (
	Name NVARCHAR(max) NOT NULL, Sex NCHAR(10) NOT NULL, Event NVARCHAR(50) NOT NULL, Equipment NVARCHAR(50) NOT NULL,
	Age FLOAT NULL, AgeClass NVARCHAR(50) NULL, BirthYearClass NVARCHAR(50) NULL, Division NVARCHAR(50) NULL,
	BodyweightKg FLOAT NULL, WeightClassKg NVARCHAR(50) NULL, Squat1Kg FLOAT NULL, Squat2Kg FLOAT NULL, Squat3Kg FLOAT NULL,
	Squat4Kg FLOAT NULL, Best3SquatKg FLOAT NULL, Bench1Kg FLOAT NULL, Bench2Kg FLOAT NULL, Bench3Kg FLOAT NULL,
	Bench4Kg FLOAT NULL, Best3BenchKg FLOAT NULL, Deadlift1Kg FLOAT NULL, Deadlift2Kg FLOAT NULL, Deadlift3Kg FLOAT NULL,
	Deadlift4Kg FLOAT NULL, Best3DeadliftKg FLOAT NULL, TotalKg FLOAT NULL, Place VARCHAR(50) NULL, Dots FLOAT NULL,
	Wilks FLOAT NULL, Glossbrenner FLOAT NULL, Goodlift FLOAT NULL, Tested VARCHAR(10) NULL, Country NVARCHAR(50) NULL,
	State NVARCHAR(50) NULL, Federation NVARCHAR(50) NULL, ParentFederation NVARCHAR(50) NULL, Date VARCHAR(10) NULL,
	MeetCountry NVARCHAR(MAX) NULL, MeetState NVARCHAR(MAX) NULL, MeetTown NVARCHAR(MAX) NULL, MeetName NVARCHAR(MAX) NULL,
	Sanctioned NVARCHAR(10) NULL
);
```

Now we can use BULK INSERT to populate our new table.

```sql
BULK INSERT OpenPowerlifting.dbo.Lifter_Data
FROM 'FileLocation\FileName.csv'
WITH (
	FORMAT='CSV',
	FIRSTROW=2,
	FIELDTERMINATOR=',',
	ROWTERMINATOR='0x0a',
	KEEPNULLS
);
```

***

### Let's quickly become familiar with the dataset.

```sql
SELECT TOP 1000 *
FROM OpenPowerlifting.dbo.Lifter_Data;
```

PUT AN IMAGE HERE

```sql
SELECT COUNT(Name) AS Number_of_Records
FROM OpenPowerlifting.dbo.Lifter_Data;
```

PUT AN IMAGE HERE

```sql
SELECT COUNT(DISTINCT(Name)) AS Number_of_Lifters
FROM OpenPowerlifting.dbo.Lifter_Data;
```

PUT AN IMAGE HERE

***

### 🔎 Narrowing down our search.


My goal is to derive insights from raw, full competitions only since that is the typical experience for new powerlifters. Full meets are denoted by
a value 'SBD' in the Event column and raw meets by the value 'Raw' in the Equipment column. I only include records in which the lifter does have
a first attempt for all three lifts. Any lifter who is disqualified is omitted.

```sql
SELECT TOP 1000 *
FROM 
	OpenPowerlifting.dbo.Lifter_Data
WHERE
	Event = 'SBD' AND
	Equipment = 'Raw' AND
	Squat1Kg IS NOT NULL AND
	Bench1Kg IS NOT NULL AND
	Deadlift1Kg IS NOT NULL AND
	Place <> 'DQ';
```

You'll notice that most records are NULL in the Squat4Kg, Bench4Kg, and Deadlift4Kg columns. These are used when a lifter is given a fourth attempt due to extenuating circumstances such as loader or spotter error.
In the sample table, I omit those columns as well as a few others that are not pertinent.

```sql
CREATE TABLE OpenPowerlifting.dbo.SBD_Data (
	Name NVARCHAR(max) NOT NULL, Sex NCHAR(10) NOT NULL, Event NVARCHAR(50) NOT NULL, Equipment NVARCHAR(50) NOT NULL, Age FLOAT NULL,
	AgeClass NVARCHAR(50) NULL, Division NVARCHAR(50) NULL, BodyweightKg FLOAT NULL, WeightClassKg NVARCHAR(50) NULL, Squat1Kg FLOAT NULL, Squat2Kg FLOAT NULL, Squat3Kg FLOAT NULL, Best3SquatKg FLOAT NULL,
	Bench1Kg FLOAT NULL, Bench2Kg FLOAT NULL, Bench3Kg FLOAT NULL, Best3BenchKg FLOAT NULL, Deadlift1Kg FLOAT NULL, Deadlift2Kg FLOAT NULL, Deadlift3Kg FLOAT NULL,
	Best3DeadliftKg FLOAT NULL, TotalKg FLOAT NULL, Place VARCHAR(50) NULL, Dots FLOAT NULL, Wilks FLOAT NULL, Date VARCHAR(10) NULL, Sanctioned NVARCHAR(10) NULL
);
```

Great, now we can insert data into this table for our sample.

```sql
INSERT INTO OpenPowerlifting.dbo.SBD_Data
	(Name, Sex, Event, Equipment, Age, AgeClass, Division, BodyweightKg, WeightClassKg, Squat1Kg, Squat2Kg, Squat3Kg, Best3SquatKg,
	Bench1Kg, Bench2Kg, Bench3Kg, Best3BenchKg, Deadlift1Kg, Deadlift2Kg, Deadlift3Kg, Best3DeadliftKg, TotalKg, Dots, Wilks, Date, Sanctioned)
SELECT
	Name, Sex, Event, Equipment, Age, AgeClass, Division, BodyweightKg, WeightClassKg, Squat1Kg, Squat2Kg, Squat3Kg, Best3SquatKg,
	Bench1Kg, Bench2Kg, Bench3Kg, Best3BenchKg, Deadlift1Kg, Deadlift2Kg, Deadlift3Kg, Best3DeadliftKg, TotalKg, Dots, Wilks, Date, Sanctioned
FROM
	OpenPowerlifting.dbo.Lifter_Data
WHERE
	Event = 'SBD' AND
	Equipment = 'Raw' AND
	Squat1Kg IS NOT NULL AND
	Bench1Kg IS NOT NULL AND
	Deadlift1Kg IS NOT NULL AND
	Place <> 'DQ';
```

***

### Now our sample table is set up well enough to query!

The main reason I have delved into this dataset is to determine if there is, on average, an amount to increase
the weight of each attempt to optimize for success. This may be of particular interest to lifters who are self-coached. 
From personal experience, choosing weight can create undue stress and confusion without a coach.

The query below uses a CASE statement with the BodyweightKg field to create standardized weight class bins based on USAPL standards
since the original dataset contains weight classes for many federations. I have omitted any records where BodyweightKg is NULL.

Using CASE statements, I created columns for the mean weight for each attempt for each lift. For the second and third attempts,
there are columns for either successful or failed attempts.

```sql
 SELECT
	CASE
		WHEN BodyweightKg < 44.0 THEN '44'
		WHEN BodyweightKg BETWEEN 44.00 AND 48.00 THEN '48'
		WHEN BodyweightKg BETWEEN 48.00 AND 52.00 THEN '52'
		WHEN BodyweightKg BETWEEN 52.00 AND 56.00 THEN '56'
		WHEN BodyweightKg BETWEEN 56.00 AND 60.00 THEN '60'
		WHEN BodyweightKg BETWEEN 60.00 AND 67.50 THEN '67.5'
		WHEN BodyweightKg BETWEEN 67.50 AND 75.00 THEN '75'
		WHEN BodyweightKg BETWEEN 75.00 AND 82.50 THEN '82.5'
		WHEN BodyweightKg BETWEEN 82.50 AND 90.00 THEN '90'
		WHEN BodyweightKg BETWEEN 90.00 AND 100.00 THEN '100'
		WHEN BodyweightKg BETWEEN 100.00 AND 110.00 THEN '110'
		WHEN BodyweightKg BETWEEN 110.00 AND 125.00 THEN '125'
		WHEN BodyweightKg BETWEEN 125.00 AND 140.00 THEN '140'
		WHEN BodyweightKg > 140 THEN '140+'
		ELSE 'OTHER'
	END AS WeightClassKgBin,
	ROUND(AVG(CASE WHEN Squat1Kg > 0 THEN Squat1Kg ELSE NULL END), 1) as Average_Squat1Kg,
	ROUND(AVG(CASE WHEN Squat2Kg > 0 THEN Squat2Kg ELSE NULL END), 1) as Average_Squat2Kg,
	ROUND(AVG(CASE WHEN Squat2Kg < 0 THEN Squat2Kg ELSE NULL END), 1) as Average_Squat2Kg_Fail,
	ROUND(AVG(CASE WHEN Squat3Kg > 0 THEN Squat3Kg ELSE NULL END), 1) as Average_Squat3Kg,
	ROUND(AVG(CASE WHEN Squat3Kg < 0 THEN Squat3Kg ELSE NULL END), 1) as Average_Squat3Kg_Fail,
	ROUND(AVG(CASE WHEN Bench1Kg > 0 THEN Bench1Kg ELSE NULL END), 1) as Average_Bench1Kg,
	ROUND(AVG(CASE WHEN Bench2Kg > 0 THEN Bench2Kg ELSE NULL END), 1) as Average_Bench2Kg,
	ROUND(AVG(CASE WHEN Bench2Kg < 0 THEN Bench2Kg ELSE NULL END), 1) as Average_Bench2Kg_Fail,
	ROUND(AVG(CASE WHEN Bench3Kg > 0 THEN Bench3Kg ELSE NULL END), 1) as Average_Bench3Kg,
	ROUND(AVG(CASE WHEN Bench3Kg < 0 THEN Bench3Kg ELSE NULL END), 1) as Average_Bench3Kg_Fail,
	ROUND(AVG(CASE WHEN Deadlift1Kg > 0 THEN Deadlift1Kg ELSE NULL END), 1) as Average_Deadlift1Kg,
	ROUND(AVG(CASE WHEN Deadlift2Kg > 0 THEN Deadlift2Kg ELSE NULL END), 1) as Average_Deadlift2Kg,
	ROUND(AVG(CASE WHEN Deadlift2Kg < 0 THEN Deadlift2Kg ELSE NULL END), 1) as Average_Deadlift2Kg_Fail,
	ROUND(AVG(CASE WHEN Deadlift3Kg > 0 THEN Deadlift3Kg ELSE NULL END), 1) as Average_Deadlift3Kg,
	ROUND(AVG(CASE WHEN Deadlift3Kg < 0 THEN Deadlift3Kg ELSE NULL END), 1) as Average_Deadlift3Kg_Fail
FROM 
	OpenPowerlifting.dbo.SBD_Data
WHERE
	BodyweightKg IS NOT NULL
GROUP BY
	CASE
		WHEN BodyweightKg < 44.0 THEN '44'
		WHEN BodyweightKg BETWEEN 44.00 AND 48.00 THEN '48'
		WHEN BodyweightKg BETWEEN 48.00 AND 52.00 THEN '52'
		WHEN BodyweightKg BETWEEN 52.00 AND 56.00 THEN '56'
		WHEN BodyweightKg BETWEEN 56.00 AND 60.00 THEN '60'
		WHEN BodyweightKg BETWEEN 60.00 AND 67.50 THEN '67.5'
		WHEN BodyweightKg BETWEEN 67.50 AND 75.00 THEN '75'
		WHEN BodyweightKg BETWEEN 75.00 AND 82.50 THEN '82.5'
		WHEN BodyweightKg BETWEEN 82.50 AND 90.00 THEN '90'
		WHEN BodyweightKg BETWEEN 90.00 AND 100.00 THEN '100'
		WHEN BodyweightKg BETWEEN 100.00 AND 110.00 THEN '110'
		WHEN BodyweightKg BETWEEN 110.00 AND 125.00 THEN '125'
		WHEN BodyweightKg BETWEEN 125.00 AND 140.00 THEN '140'
		WHEN BodyweightKg > 140 THEN '140+'
		ELSE 'OTHER'
	END
ORDER BY
	WeightClassKgBin
```
***

At this point, I moved to work in Excel since it is convenient for creating basic visualizations. Export the output of the above query to a CSV file and open in Excel.

![SquatIncrease](https://github.com/aidanlfrench/OpenPowerlifting-Project/assets/167480631/4eb462ae-7f49-43bb-a21d-6aab580444c0)

![BenchIncrease](https://github.com/aidanlfrench/OpenPowerlifting-Project/assets/167480631/fdc8fd93-0d9d-4d2e-a687-7a599cedc948)

![DeadliftIncrease](https://github.com/aidanlfrench/OpenPowerlifting-Project/assets/167480631/6d3cc2a3-887d-4d23-95f0-305f671c7870)


## 🕵️‍♂️ Observations

In general, our line charts show that lifters who fail a subsequent attempt "go up in" weight for each attempt more than lifters who succeed the corresponding attempt.
The bench press does raise an eyebrow since the 48kg through the 82.5kg weight classes exhibit the opposite. This could be for any number of reasons, one example being
that the lighter weight classes may start with a lighter first attempt to get one on the board, and then increase more (relative to those who failed their second attempt)
for their second attempt.
