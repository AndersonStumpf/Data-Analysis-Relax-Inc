# Exploratory Data Analysis - Relax Inc Challenge

## Objective

This repository demonstrates advanced SQL reporting and analysis skills, showcasing how businesses can extract valuable insights to drive strategic decision-making.

## Context

`Relax Inc.` makes productivity and project management software that's popular with both individuals and teams. Founded by several former Facebook employees.
This data analytics take-home assignment, which has been given to data analysts at Relax Inc., asks you to dig into user engagement data.

The goal of this take-home assignment is to identify adopted users — those who logged in on three separate days within a seven-day period — and derive insights about user engagement patterns to inform strategic decisions.

Other analyzes were also carried out to better understand the dataset:
* Adopted User Profile
* Period with Highest Users Engagement
* Users Profile who Logged in 1, 2 and 3+ times
* Marketing Drip x Mailing List x None

## Key Findings

* **Adopted users account for 12% of the dataset, highlighting a significant minority of highly engaged users**
* **Most adopted users were invited via Org_Invite or Guest_Invite, suggesting the importance of organizational referrals**
* **Marketing Drip and Mailing List Opt didn't show anything to interesting compared to users who did not receive any email** 
* **Approximately 71% of users logged in only once, indicating challenges with user retention** 

## Recommendation

In general, users in each group (Marketing Drip and Mailing List Opt) have the same average number of visits, this behavior is repeated for the top 100, 500 and 1000 most active users. This shows that the investment in Drip Marketing and Mailing List Opt is not being effective.

Conduct A/B testing with different email marketing strategies to increase user adoption rates. Additionally, focus marketing efforts on campaigns targeting users similar to those invited via Org_Invite and Guest_Invite channels.

## Next Steps

Perform a Correlation Analysis and Perform a Prediction Analysis.

## Challenges

The analysis was limited by the availability of user activity data, making it difficult to perform more nuanced analyses or establish causal relationships.

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
    -- Count the number of users grouped by their account creation source.
    SELECT creation_source, 
        COUNT(creation_source) AS quant -- Count the occurrences of each creation source
    FROM raw_takehome_user
    GROUP BY creation_source -- Group by the creation source field
    ORDER BY quant DESC; -- Sort the results in descending order by count
```
![Creation_Source](https://github.com/AndersonStumpf/Data-Analysis-Relax-Inc/blob/main/pics/creation_surce_distribution.png)

**Visits Over Time**
```sql
    -- Count the number of user visits, grouped by year and month.
    SELECT year, 
        month, 
        COUNT(*) AS visit_count -- Count the total visits for each year and month
    FROM cleaned_takehome_user_engagement
    GROUP BY year, month -- Group the data by year and month
    ORDER BY year, month; -- Sort by year and month in ascending order

```
![Creation_Source_Visites_over_time](https://github.com/AndersonStumpf/Data-Analysis-Relax-Inc/blob/main/pics/creation_source_visites.png)

## Data Cleaning

**Detecting Duplicates**
```sql
    -- Identify duplicate records in the raw_takehome_user table.
    SELECT COUNT(*) AS duplicate_count -- Count the number of duplicate rows
    FROM (
        SELECT object_id, 
            name, 
            COUNT(*) AS records -- Count the occurrences of each unique user ID and name
        FROM raw_takehome_user
        GROUP BY object_id, name -- Group by user ID and name
    ) subquery
    WHERE records > 1; -- Filter for groups with more than one record (duplicates)

```

**Handling Nulls**

* last_session_creation_time – 3177 Nulls
* invited_by_user_id - 5583 Nulls

```sql
    -- Fill NULL values in last_session_creation_time with the account creation time.
    UPDATE cleaned_takehome_user
    SET last_session_creation_time = COALESCE(last_session_creation_time::timestamp, creation_time::timestamp);

    -- Replace NULL values in invited_by_user_id with '0' to indicate no inviter.
    UPDATE cleaned_takehome_user
    SET invited_by_user_id = COALESCE(invited_by_user_id, '0');

```

## Data Analysis

**Adopted Users History**
```sql
    -- Identify adopted users based on their login history.
    CREATE VIEW adopted_users_history AS
    WITH visit_history AS (
        -- Create a history of user visits with timestamps of the previous two logins.
        SELECT user_id, 
            time_stamp, 
            LAG(time_stamp) OVER (PARTITION BY user_id ORDER BY time_stamp) AS lag_1, -- Previous login
            LAG(time_stamp, 2) OVER (PARTITION BY user_id ORDER BY time_stamp) AS lag_2 -- Two logins ago
        FROM cleaned_takehome_user_engagement
    )
    -- Mark users as adopted if they logged in on three separate days within a seven-day period.
    SELECT user_id, 
        time_stamp, 
        lag_1, 
        lag_2,
        CASE 
            WHEN (time_stamp - lag_2) < '8 days' THEN 1 -- Adopted user if difference between timestamps is less than 8 days
            ELSE 0
        END AS adopted_user
    FROM visit_history
    WHERE lag_2 IS NOT NULL; -- Exclude users without enough login history
```
**Adopted Users**
```sql
    -- Create a list of adopted users and join it with the cleaned user table for additional information.
    CREATE VIEW adopted_users AS
    WITH adopted_users_list AS (
        -- Select distinct adopted users from the history table.
        SELECT DISTINCT user_id, adopted_user
        FROM adopted_users_history
        WHERE adopted_user = 1
    )
    SELECT c.*, 
        a.adopted_user -- Include the adopted user flag
    FROM cleaned_takehome_user c
    LEFT JOIN adopted_users_list a 
    ON c.object_id = a.user_id; -- Match users by ID

```

**Users Profile who Logged in 1, 2 and 3+ times**
```sql
    -- Categorize users based on their login frequency and calculate percentages.
    WITH number_visits AS (
        SELECT user_id, 
            CASE 
                WHEN COUNT(visited) < 2 THEN '1' -- 1 login
                WHEN COUNT(visited) < 3 THEN '2' -- 2 logins
                ELSE '3+' -- 3 or more logins
            END AS visits_quant
        FROM cleaned_takehome_user_engagement
        GROUP BY user_id
    )
    SELECT visits_quant, 
        COUNT(visits_quant) AS count, -- Count the number of users in each category
        ROUND(COUNT(visits_quant) * 100.0 / SUM(COUNT(visits_quant)) OVER(), 2) AS pct -- Calculate percentage
    FROM number_visits
    GROUP BY visits_quant
    ORDER BY count DESC; -- Sort by the number of users in descending order
```

**Creation Source of the Users Profile who Logged 3+ times**
```sql
    -- Analyze the creation source of users who logged in 3 or more times.
    WITH number_visits AS (
        SELECT user_id::varchar, 
            CASE 
                WHEN COUNT(visited) < 2 THEN '1' -- 1 login
                WHEN COUNT(visited) < 3 THEN '2' -- 2 logins
                ELSE '3+' -- 3 or more logins
            END AS visits_quant
        FROM cleaned_takehome_user_engagement
        GROUP BY user_id
    )
    SELECT c.creation_source, 
        COUNT(n.user_id) AS user_count, -- Count the number of users for each creation source
        ROUND(COUNT(n.user_id) * 100.0 / SUM(COUNT(n.user_id)) OVER(), 2) AS pct -- Calculate percentage
    FROM number_visits n
    RIGHT JOIN cleaned_takehome_user c 
    ON n.user_id = c.object_id
    WHERE visits_quant = '3+' -- Focus on users with 3+ logins
    GROUP BY c.creation_source
    ORDER BY user_count DESC; -- Sort by the number of users in descending order

```

**Marketing Drip x Mailing List x None**
```sql
    -- Compare user engagement based on marketing drip and mailing list participation.
    WITH number_visits AS (
        SELECT user_id::varchar, 
            CASE 
                WHEN COUNT(visited) < 2 THEN '1' -- 1 login
                WHEN COUNT(visited) < 3 THEN '2' -- 2 logins
                ELSE '3+' -- 3 or more logins
            END AS visits_quant
        FROM cleaned_takehome_user_engagement
        GROUP BY user_id
    )
    SELECT c.enabled_for_marketing_drip, 
        c.opted_in_to_mailing_list, 
        COUNT(n.user_id) AS user_count -- Count the number of users in each category
    FROM number_visits n
    RIGHT JOIN cleaned_takehome_user c 
    ON n.user_id = c.object_id
    WHERE visits_quant = '3+' -- Focus on users with 3+ logins
    GROUP BY c.enabled_for_marketing_drip, c.opted_in_to_mailing_list
    ORDER BY user_count DESC; -- Sort by the number of users in descending order
```

## Next steps

* **Perform a Correlation Analysis**
* **Perform a Prediction Analysis**

Please note any factors you considered or investigation you did, even if they did not pan out. Feel free to identify any further research or data you think would be valuable.
Thank you.