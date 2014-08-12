---
layout: post
title:  "Mondrian semi-additive measures"
summary: This article explains a workaround you can use to implement semi-additive measures in Mondrian
date:   2014-08-12
categories: Mondrian
tags: Pentaho
published: true
---

Currently **Mondrian** doesn't offer a way to define **semi-additive** measures (see as well [this Jira case](http://jira.pentaho.com/browse/MONDRIAN-962)). However, Vladislav Krakhalev proposed a workaround using `OPENINGPERIOD()` or `CLOSINGPERIOD()` **MDX** functions. Thanks a lot! 

So what are **semi-additive measures**? Let's best explain this based on an example:

Imagine you have to prepare a **subscriber base analysis** for a telecom provider. You have the obvious date dimension, tariff plan dimension, etc.

In our **OLAP** Schema the subscriber base measure currently looks like this:

```
<Measure name="Subscribers" column="subscribers" datatype="Integer" formatString="#,###" aggregator="sum" visible="false" />
``` 

As you can see, we use a use sum to aggregate this measure ... but this not quite right. So you can sum the subscriber base on a daily level, but once you roll up to weekly, monthly, quarterly or yearly level, you have to take the first or last value of this period (whatever the client is happy with really - averages usually don't do the trick). 

First make sure you have proper **time dimension** defined (like the one shown below), so that the rollup works correctly:

```
<Dimension type="TimeDimension" visible="true" highCardinality="false" name="Date">
	<Hierarchy name="Monthly Calendar" visible="true" hasAll="true" allMemberName="Total" allMemberCaption="Total" primaryKey="date_tk">
		<Table name="dim_date" schema="dma_subscriber_base"/>
		<Level name="Year" visible="true" column="year4" type="Integer" uniqueMembers="false" levelType="TimeYears" hideMemberIf="Never" />
		<Level name="Quarter" visible="true" column="year_quarter" ordinalColumn="year_quarter_int" type="String" uniqueMembers="false" levelType="TimeQuarters" hideMemberIf="Never" />
		<Level name="Month" visible="true" column="year_month"  ordinalColunn="year_month_int" type="String" uniqueMembers="false" levelType="TimeMonths" hideMemberIf="Never"/>
		<Level name="Date" visible="true" column="date_digits" ordinalColumn="date_tk" type="Date" uniqueMembers="true" levelType="TimeDays" hideMemberIf="Never"/>
	</Hierarchy>	
</Dimension>
```

Then use MDX functions [`OPENINGPERIOD()`](http://msdn.microsoft.com/en-us/library/ms145992.aspx) or [`CLOSINGPERIOD()`](http://msdn.microsoft.com/en-us/library/ms145584.aspx). Let's test these function for the day, month, quarter and year level. Note that we define first a calculated member which will return the sum of subscribers for the first day in a given period using the `OPENINGPERIOD()` function in this example:

```
WITH
MEMBER [Measures].[Subbase] AS
(
	(OPENINGPERIOD([Date.Monthly Calendar].[Date]), [Measures].[Subscribers])
) 
SELECT
[Measures].[Subbase] On Columns,
Non Empty({[Date.Monthly Calendar].[Date].Members}) On Rows
FROM [Subscriber Base]

WITH
MEMBER [Measures].[Subbase] AS
(
	(OPENINGPERIOD([Date.Monthly Calendar].[Date]), [Measures].[Subscribers])
) 
SELECT
[Measures].[Subbase] On Columns,
Non Empty({[Date.Monthly Calendar].[Month].Members}) On Rows
FROM [Subscriber Base]

WITH
MEMBER [Measures].[Subbase] AS
(
	(OPENINGPERIOD([Date.Monthly Calendar].[Date]), [Measures].[Subscribers])
) 
SELECT
[Measures].[Subbase] On Columns,
Non Empty({[Date.Monthly Calendar].[Quarter].Members}) On Rows
FROM [Subscriber Base]

WITH
MEMBER [Measures].[Subbase] AS
(
	(OPENINGPERIOD([Date.Monthly Calendar].[Date]), [Measures].[Subscribers])
) 
SELECT
[Measures].[Subbase] On Columns,
Non Empty({[Date.Monthly Calendar].[Year].Members}) On Rows
FROM [Subscriber Base]
```

Our test tells us that the logic is working as expected, so the next step is to add a `CalculatedMember` to the OLAP Schema:

```
<CalculatedMember name="Subbase" formatString="#,##0" caption="Subbase" description="Subbase at the first day of the period" formula="(OPENINGPERIOD([Date.Monthly Calendar].[Date]), [Measures].[Subscribers])" dimension="Measures" visible="true"/>
```

This looks like a very simple workaround and should help solving quite a lot of scenarios.

Ok, are you up to the next challenge? With one hiearchy only in the date dimensions things are fairly straight forward. But what happens if we add a new hierarchy for the week?

```
<Dimension name="Date" type="TimeDimension" visible="true" highCardinality="false">
	<Hierarchy name="Monthly Calendar" visible="true" hasAll="true" allMemberName="Total" allMemberCaption="Total" primaryKey="date_tk">
		<Table name="dim_date" schema="dma_subscriber_base"/>
		<Level name="Year" visible="true" column="year4" type="Integer" uniqueMembers="false" levelType="TimeYears" hideMemberIf="Never" />
		<Level name="Quarter" visible="true" column="year_quarter" ordinalColumn="year_quarter_int" type="String" uniqueMembers="false" levelType="TimeQuarters" hideMemberIf="Never" />
		<Level name="Month" visible="true" column="year_month"  ordinalColunn="year_month_int" type="String" uniqueMembers="false" levelType="TimeMonths" hideMemberIf="Never"/>
		<Level name="Date" visible="true" column="date_digits" ordinalColumn="date_tk" type="Date" uniqueMembers="true" levelType="TimeDays" hideMemberIf="Never"/>
	</Hierarchy>	
	<Hierarchy name="Weekly Calendar" visible="true" hasAll="true" allMemberName="Total" allMemberCaption="Total" primaryKey="date_tk">
		<Table name="dim_date" schema="dma_subscriber_base"/>
		<Level name="Year" visible="true" column="year_week_based" type="Integer" uniqueMembers="false" levelType="TimeYears" hideMemberIf="Never" />
		<Level name="Week" visible="true" column="year_week"  ordinalColumn="year_week_int" type="String" uniqueMembers="false" levelType="TimeWeeks" hideMemberIf="Never"/>
		<Level name="Date" visible="true" column="date_digits" ordinalColumn="date_tk" type="Date" uniqueMembers="true" levelType="TimeDays" hideMemberIf="Never"/>
	</Hierarchy>
</Dimension>
```

If we want to get the weekly stats now like shown below we will get no result:

```
SELECT
[Measures].[Subbase] On Columns,
Non Empty({[Date.Weekly Calendar].[Year].Members}) On Rows
FROM [Subscriber Base]
```

This is because we defined our calculated member to only work with the **Monthly Calendar**. We have to find a way to make our calculated measure work **across all date hierarchies**.

Let's just make sure that our `OPENINGPERIOD()` is still returning the expected results with these two date hierarchies:

```
SELECT
OPENINGPERIOD([Date.Weekly Calendar].[Date]) On Columns,
[Measures].[Subscribers] On Rows
FROM [Subscriber Base]
```

In my case this didn't return anything: no result and no date member. So let's check if we get a result with the `CLOSINGPERIOD()` function:

```
SELECT
CLOSINGPERIOD([Date.Weekly Calendar].[Date]) On Columns,
[Measures].[Subscribers] On Rows
FROM [Subscriber Base]
```

This time I got a date member returned: 2019-12 (which happened to be the last month in my date dimension), but no value. But hold on ... why do we get a month returned when we run the function on the **Weekly Calendar** hierarchy? My guess is that this happens because the **Monthly Calendar** happens to be the default hierarchy.

We can fix this by specifying the optional **Member Expression** for the `OPENINGPERIOD()` function:

```
SELECT
OPENINGPERIOD([Date.Weekly Calendar].[Date],[Date.Weekly Calendar].[2014]) On Columns,
[Measures].[Subscribers] On Rows
FROM [Subscriber Base]

SELECT
OPENINGPERIOD([Date.Monthly Calendar].[Date],[Date.Monthly Calendar].[2014]) On Columns,
[Measures].[Subscribers] On Rows
FROM [Subscriber Base]
```

In both cases we get the expected results now.

Next we have to find a way to get the calculated measures working **across all the hiearchies**. We could try to achieve this by creating a new calculated measure and using a **IIF** statement. Let's just simply check if this would work:

```
WITH
MEMBER [Measures].[Subbase Monthly Calendar] AS
(
	OPENINGPERIOD([Date.Monthly Calendar].[Date], [Date.Monthly Calendar].CurrentMember)
	, [Measures].[Subscribers]
)
MEMBER [Measures].[Subbase Weekly Calendar] AS
(
	OPENINGPERIOD([Date.Weekly Calendar].[Date], [Date.Weekly Calendar].CurrentMember)
	, [Measures].[Subscribers]
)
MEMBER [Measures].[Subbase Final] AS
(
	IIF([Date.Monthly Calendar].currentmember.level.ordinal > 0
		, [Measures].[Subbase Monthly Calendar]
		, [Measures].[Subbase Weekly Calendar]
	)
)
SELECT
[Measures].[Subbase Final] On Columns,
Non Empty({[Date.Monthly Calendar].[Month].Members}) On Rows
FROM [Subscriber Base]

WITH
MEMBER [Measures].[Subbase Monthly Calendar] AS
(
	OPENINGPERIOD([Date.Monthly Calendar].[Date], [Date.Monthly Calendar].CurrentMember)
	, [Measures].[Subscribers]
)
MEMBER [Measures].[Subbase Weekly Calendar] AS
(
	OPENINGPERIOD([Date.Weekly Calendar].[Date], [Date.Weekly Calendar].CurrentMember)
	, [Measures].[Subscribers]
)
MEMBER [Measures].[Subbase Final] AS
(
	IIF([Date.Monthly Calendar].currentmember.level.ordinal > 0
		, [Measures].[Subbase Monthly Calendar]
		, [Measures].[Subbase Weekly Calendar]
	)
)
SELECT
[Measures].[Subbase Final] On Columns,
Non Empty({[Date.Weekly Calendar].[Week].Members}) On Rows
FROM [Subscriber Base]
```

The results are correct. Let's implement this in our **Mondrian OLAP Schema**:

```
<CalculatedMember name="Subbase Monthly Calendar" formatString="#,##0" caption="Subbase" description="Subbase at the first day of the period" formula="(OPENINGPERIOD([Date.Monthly Calendar].[Date], [Date.Monthly Calendar].CurrentMember), [Measures].[Subscribers])" dimension="Measures" visible="false"/>
<CalculatedMember name="Subbase Weekly Calendar" formatString="#,##0" caption="Subbase" description="Subbase at the first day of the period" formula="(OPENINGPERIOD([Date.Weekly Calendar].[Date], [Date.Weekly Calendar].CurrentMember), [Measures].[Subscribers])" dimension="Measures" visible="false"/>
<CalculatedMember name="Subbase" formatString="#,##0" caption="Subbase" description="Subbase at the first day of the period" formula="IIF([Date.Monthly Calendar].currentmember.level.ordinal > 0, [Measures].[Subbase Monthly Calendar], [Measures].[Subbase Weekly Calendar])" dimension="Measures" visible="true"/>
```

As you can see we specified the first two calculated measures as hidden (they are intermediate steps and should not be exposed to the end-user).
The beauty of this is that now our end-user only has to deal with one measure (irrespective of whatever date hiearchy they choose).