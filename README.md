# Exploratory Data Analysis - Relax Inc Challenge

## Objective

This repository aims to showcase advanced reports built using SQL. The analyses provided here can be applied to businesses of all sizes that aspire to become more data-driven. These reports will help organizations extract valuable insights from their data, aiding in strategic decision-making.

## Context

The `Relax Inc.` makes productivity and project management software that's popular with both individuals and teams. Founded by several former Facebook employees.
This data analytics take-home assignment, which has been given to data analysts at Relax Inc., asks you to dig into user engagement data.

The main objective is determine who an `“adopted user”` is, which is a user who has logged into the product on three separate days in at least one seven-day period.

Other analyzes were also carried out to better understand the dataset:
* Adopted User Profile
* Period with Highest Users Engagement
* Users Profile who Logged in 1, 2 and 3+ times
* Marketing Drip x Mailing List x None

## Interesting Findings

* **Adopted Users represent 12% of the dataset**
* **Source Creation - Adopted Users were mostly invited by Org_Invite and Guest_Invite**
* **Marketing Drip and Mailing List Opt didn't show anything to interesting compared to users who did not receive any email** 
* **Around 71% of users only accessed it once** 

## Recommendation

In general, users in each group (Marketing Drip and Mailing List Opt) have the same average number of visits, this behavior is repeated for the top 100, 500 and 1000 most active users. This shows that the investment in Drip Marketing and Mailing List Opt is not being effective.

It would be interesting to carry out an A/B test with different types of email marketing to get more "adopted users".

Adopted Users were mostly invited by Org_Invite and Guest_Invite. This tells that it may be advantageous to invest in other types of marketing campaigns aimed at capturing users from these groups.

## Next Steps

Perform a Correlation Analysis and Perform a Prediction Analysis.

## Challenges

It would be great if we can have more data about user activity. With the features given it was difficult to carry out accurate analysis.

# Project Step by Step

## Processes and Tools

The tools used were:
* Postgres: analyze and work with the datasets
* Docker: Create pg4 container
* Looker Studio: Create the graphs

The processes were:
* **Data Gathering → Data Exploration → Data Cleaning → Data Analysis**

Each part of the process will be presented below.

## Contextualization

**User Table:**
("takehome_users") with data on 12,000 users who signed up for the product in the last two years.
This table includes:

* name: the user's name
* object_id: the user's id
* email: email address
* creation_source: how their account was created. 
This takes on one of 5 values:

    PERSONAL_PROJECTS: invited to join another user's personal workspace

    GUEST_INVITE: invited to an organization as a guest (limited permissions)

    ORG_INVITE: invited to an organization (as a full member)

    SIGNUP: signed up via the website

    SIGNUP_GOOGLE_AUTH: signed up using Google Authentication (using a Google email account for their login id)

* creation_time: when they created their account
* last_session_creation_time: unix timestamp of last login
* opted_in_to_mailing_list: whether they have opted into receiving marketing emails
* enabled_for_marketing_drip: whether they are on the regular marketing email drip
* org_id: the organization (group of users) they belong to
* invited_by_user_id: which user invited them to join (if applicable).

![User_Table](https://github.com/AndersonStumpf/Data-Analysis-Relax-Inc/blob/main/pics/user_table.png)

**User Engagement Table:**
("takehome_user_engagement") that has a row for each day that a user logged into the product.
This table includes:

* time_stamp: when user logged
* user_id: the user's id
* visited

![User_Table_Engagement](https://github.com/AndersonStumpf/Data-Analysis-Relax-Inc/blob/main/pics/user_engagement.png)

## Data Gathering

The data was obtained from kaggle, but you can use csv or sql archive located in this repo for your analysis.

The datasets were with the name `raw_takehome_user` and `raw_takehome_user_engagement`. Subsequently, the data was cleaned and two new tables were created `cleaned_takehome_user` and `cleaned_takehome_user_engagement`

## Data Exploration

Some data exploration was performed for better understanding of the data.

**Frequency Creation Source**

  ```sql
    SELECT creation_source, COUNT(creation_source) AS quant
    FROM raw_takehome_user
    GROUP BY 1
    ORDER BY 2 DESC
```
![Creation_Source](https://github.com/AndersonStumpf/Data-Analysis-Relax-Inc/blob/main/pics/creation_surce_distribution.png)

**Visits Over Time**
```sql
    SELECT year, month, count(*)
    FROM cleaned_takehome_user_engagement
    GROUP BY 1,2
    ORDER BY 1,2

```
![Creation_Source_Visites_over_time](https://github.com/AndersonStumpf/Data-Analysis-Relax-Inc/blob/main/pics/creation_source_visites.png)

## Data Cleaning

**Detecting Duplicates**
```sql
    SELECT COUNT(*)
    FROM 
    (
        SELECT object_id, name, COUNT(*) AS records
        FROM raw_takehome_user
        GROUP BY 1
    ) aa
    WHERE records > 1

```

**Nulls**

* last_session_creation_time – 3177 Nulls
* invited_by_user_id - 5583 Nulls

```sql
    UPDATE cleaned_takehome_user
    SET last_session_creation_time = COALESCE(last_session_creation_time::timestamp, creation_time::timestamp);

    UPDATE cleaned_takehome_user
    SET invited_by_user_id = COALESCE(invited_by_user_id, '0');

```

## Data Analysis

**Adopted Users History**
```sql
CREATE VIEW adopted_users_history AS
    WITH visite_history AS (
        SELECT user_id, time_stamp, LAG(time_stamp) OVER(PARTITION BY user_id ORDER BY time_stamp) AS lag_1, LAG(time_stamp, 2) OVER(PARTITION BY user_id ORDER BY time_stamp) AS lag_2
        FROM cleaned_takehome_user_engagement
        ORDER BY 2)

    SELECT user_id, time_stamp, lag_1, lag_2,
        CASE WHEN (time_stamp - lag_2) < '8 days' THEN 1 ELSE 0 END AS adopted_user
    FROM visite_history
    WHERE lag_2 IS NOT NULL

```
**Adopted Users**
```sql
CREATE VIEW adopted_users AS
    WITH adopted_users AS (
        SELECT DISTINCT user_id, adopted_user
        FROM adopted_users_history
        WHERE adopted_user = 1)

    SELECT c.*, a.adopted_user
    FROM cleaned_takehome_user c
    LEFT JOIN adopted_users a ON c.object_id = a.user_id

```

**Users Profile who Logged in 1, 2 and 3+ times**
```sql
    WITH number_visits AS (
        SELECT user_id, 
            CASE WHEN COUNT(visited) < 2 THEN '1'
            WHEN COUNT(visited) < 3 THEN '2'
            ELSE '3+' 
            END AS visits_quant
        FROM cleaned_takehome_user_engagement
        GROUP BY 1)

    SELECT visits_quant, COUNT(visits_quant), ROUND(COUNT(visits_quant) *100 /	SUM(COUNT(visits_quant)) OVER(PARTITION BY 1),2) AS pct
    FROM number_visits
    GROUP BY 1
    ORDER BY 2 DESC

```

**Creation Source of the Users Profile who Logged 3+ times**
```sql
    WITH number_visits AS (
        SELECT user_id::"varchar", 
            CASE WHEN COUNT(visited) < 2 THEN '1'
            WHEN COUNT(visited) < 3 THEN '2'
            ELSE '3+' 
            END AS visits_quant
        FROM cleaned_takehome_user_engagement
        GROUP BY 1)

    SELECT c.creation_source, COUNT(n.user_id), ROUND(COUNT(n.user_id) *100 /	SUM(COUNT(n.user_id)) OVER(PARTITION BY 1),2) 
    FROM number_visits n 
    RIGHT JOIN cleaned_takehome_user c ON n.user_id = c.object_id
    WHERE visits_quant = '3+'
    GROUP BY 1
    ORDER BY 2 DESC

```

**Marketing Drip x Mailing List x None**
```sql
    WITH number_visits AS (
        SELECT user_id::"varchar", 
            CASE WHEN COUNT(visited) < 2 THEN '1'
            WHEN COUNT(visited) < 3 THEN '2'
            ELSE '3+' 
            END AS visits_quant
        FROM cleaned_takehome_user_engagement
        GROUP BY 1)

    SELECT c.enabled_for_marketing_drip, opted_in_to_mailing_list,COUNT(n.user_id) 
    FROM number_visits n 
    RIGHT JOIN cleaned_takehome_user c ON n.user_id = c.object_id
    WHERE visits_quant = '3+'
    GROUP BY 1, 2
    ORDER BY 3 DESC

```

## Next steps

* **Perform a Correlation Analysis**
* **Perform a Prediction Analysis**

Please note any factors you considered or investigation you did, even if they did not pan out. Feel free to identify any further research or data you think would be valuable.
Thank you.