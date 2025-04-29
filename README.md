# Introduction to Business Problem
ShopEasy, an online retail business, is facing reduced customer engagement and conversion rates despite launching several new online marketing campaigns. They are reaching out to you to help conduct a detailed analysis and identify areas for improvement in their marketing strategies.

## Key Points:
•	Reduced Customer Engagement: The number of customer interactions and engagement with the site and marketing content has declined.
•	Decreased Conversion Rates: Fewer site visitors are converting into paying customers.
•	High Marketing Expenses: Significant investments in marketing campaigns are not yielding expected returns.
•	Need for Customer Feedback Analysis: Understanding customer opinions about products and services is crucial for improving engagement and conversions.
## Key Performance Indicators (KPIs)
•	Conversion Rate: Percentage of website visitors who make a purchase.
•	Customer Engagement Rate: Level of interaction with marketing content (clicks, likes, comments).
•	Average Order Value (AOV): Average amount spent by a customer per transaction.
•	Customer Feedback Score: Average rating from customer reviews.
## Goals
•	Increase Conversion Rates:
•	Goal: Identify factors impacting the conversion rate and provide recommendations to improve it.
•	Insight: Highlight key stages where visitors drop off and suggest improvements to optimize the conversion funnel.
•	Enhance Customer Engagement:
•	Goal: Determine which types of content drive the highest engagement. 
•	Insight: Analyze interaction levels with different types of marketing content to inform better content strategies.
•	Improve Customer Feedback Scores:
•	Goal: Understand common themes in customer reviews and provide actionable insights.
•	Insight: Identify recurring positive and negative feedback to guide product and service improvements.

## Tools Used: SQL, Python, Power BI

# Task 1– Data Cleaning in SQl Server
There are 5 different tables which we uploaded to SQL Server. The tables are:
•	Products
•	Geography
•	Engagement Data
•	Customers
•	Customer Reviews
•	Customer Journey
![image](https://github.com/user-attachments/assets/3c1aa6c0-048b-4ab6-a477-a1d5424ebd07)
 

## Categorizing products based on their price in Products table

```sql
SELECT 
    ProductID,  
    ProductName,
    Price,  
	Category, 

    CASE 
        WHEN Price < 50 THEN 'Low'  
        WHEN Price BETWEEN 50 AND 200 THEN 'Medium'  
        ELSE 'High'  
    END AS PriceCategory  

FROM 
    dbo.products;
```
![image](https://github.com/user-attachments/assets/83925900-e0af-4452-a69c-30e8a7bdad5e)

## Joining customers table with geography table to enrich customer data with geographic information
```sql
  SELECT 
    c.CustomerID, 
    c.CustomerName, 
    c.Email,  
    c.Gender, 
    c.Age, 
    g.Country,  
    g.City  
FROM 
    dbo.customers as c  
LEFT JOIN
    dbo.geography g  
ON 
    c.GeographyID = g.GeographyID;
 ```
![image](https://github.com/user-attachments/assets/02e27496-a22a-4f0b-8a51-18f41ee8c092)

## Cleaning whitespace issues in the Review Text column of Customer Review Table

```sql
SELECT 
    ReviewID,  
    CustomerID,  
    ProductID,  
    ReviewDate, 
    Rating,  
    REPLACE(ReviewText, '  ', ' ') AS ReviewText
FROM 
    dbo.customer_reviews;  
```
 ![image](https://github.com/user-attachments/assets/4c5e4a1f-1a7c-427e-8857-20c954f0ac2a)

## Cleaning and normalizing the engagement_data table

```sql
SELECT 
    EngagementID,  
    ContentID,  
	CampaignID,
    ProductID, 
    UPPER(REPLACE(ContentType, 'Socialmedia', 'Social Media')) AS ContentType,  
    LEFT(ViewsClicksCombined, CHARINDEX('-', ViewsClicksCombined) - 1) AS Views,  
    RIGHT(ViewsClicksCombined, LEN(ViewsClicksCombined) - CHARINDEX('-', ViewsClicksCombined)) AS Clicks,  
    Likes, 
    FORMAT(CONVERT(DATE, EngagementDate), 'dd.MM.yyyy') AS EngagementDate 
FROM 
    dbo.engagement_data 
WHERE 
    ContentType != 'Newsletter';
```
 ![image](https://github.com/user-attachments/assets/8de8acc2-6aed-4674-85a6-8bfc0c16c1d2)

## Common Table Expression (CTE) to identify and tag duplicate records

```sql
WITH DuplicateRecords AS (
    SELECT 
        JourneyID,  
        CustomerID, 
        ProductID,  
        VisitDate,  
        Stage,  
        Action,  
        Duration,  
        ROW_NUMBER() OVER (
            PARTITION BY CustomerID, ProductID, VisitDate, Stage, Action  
            ORDER BY JourneyID  
        ) AS row_num  
    FROM 
        dbo.customer_journey 
)
    
SELECT *
FROM DuplicateRecords
ORDER BY JourneyID

    
SELECT 
    JourneyID,  
    CustomerID, 
    ProductID,  
    VisitDate,  
    Stage,  
    Action, 
    COALESCE(Duration, avg_duration) AS Duration  
FROM 
    (
        SELECT 
            JourneyID,  
            CustomerID, 
            ProductID,  
            VisitDate,  
            UPPER(Stage) AS Stage, 
            Action, 
            Duration, 
            AVG(Duration) OVER (PARTITION BY VisitDate) AS avg_duration,  
            ROW_NUMBER() OVER (
                PARTITION BY CustomerID, ProductID, VisitDate, UPPER(Stage), Action  
                ORDER BY JourneyID  
            ) AS row_num  
        FROM 
            dbo.customer_journey  
    ) AS subquery 
WHERE 
    row_num = 1;  
``` 
![image](https://github.com/user-attachments/assets/610fd9cd-9dc6-409d-8761-8e22600cc9e1)

## Using Python to fetch data from SQL Server and using VADER sentiment intensity analyzer for analyzing the sentiment of text data from Customer Reviews

```python
import pandas as pd
import pyodbc
import nltk
from nltk.sentiment.vader import SentimentIntensityAnalyzer

nltk.download('vader_lexicon')

def fetch_data_from_sql():
    conn_str = (
        "Driver={SQL Server};"  
        "Server=LAPTOP-3HRLKDAT\SQLEXPRESS;"  
        "Database=ShopEasy_MarketingAnalysis;"  
        "Trusted_Connection=yes;"  
    )
    conn = pyodbc.connect(conn_str)
    
    query = "SELECT ReviewID, CustomerID, ProductID, ReviewDate, Rating, ReviewText FROM fact_customer_reviews"
    
    df = pd.read_sql(query, conn)
    
    conn.close()
    
    return df

customer_reviews_df = fetch_data_from_sql()

sia = SentimentIntensityAnalyzer()

def calculate_sentiment(review):
    sentiment = sia.polarity_scores(review)
    return sentiment['compound']

def categorize_sentiment(score, rating):
    if score > 0.05:  
        if rating >= 4:
            return 'Positive'  
        elif rating == 3:
            return 'Mixed Positive' 
        else:
            return 'Mixed Negative' 
    elif score < -0.05:  
        if rating <= 2:
            return 'Negative'  
        elif rating == 3:
            return 'Mixed Negative'  
        else:
            return 'Mixed Positive'  
    else:  
        if rating >= 4:
            return 'Positive'  
        elif rating <= 2:
            return 'Negative'  
        else:
            return 'Neutral'  

def sentiment_bucket(score):
    if score >= 0.5:
        return '0.5 to 1.0'  
    elif 0.0 <= score < 0.5:
        return '0.0 to 0.49' 
    elif -0.5 <= score < 0.0:
        return '-0.49 to 0.0' 
    else:
        return '-1.0 to -0.5'  

customer_reviews_df['SentimentScore'] = customer_reviews_df['ReviewText'].apply(calculate_sentiment)

customer_reviews_df['SentimentCategory'] = customer_reviews_df.apply(
    lambda row: categorize_sentiment(row['SentimentScore'], row['Rating']), axis=1)

customer_reviews_df['SentimentBucket'] = customer_reviews_df['SentimentScore'].apply(sentiment_bucket)

print(customer_reviews_df.head())

customer_reviews_df.to_csv('fact_customer_reviews_with_sentiment.csv', index=False)
```

## Modified DataSet : https://github.com/Jyotiprakash01/ShopEasy_Marketing-Analysis/blob/main/fact_customer_reviews_enrich.csv

## Now, Loading the data from SQL Server and CSV to Power BI. 
- Change all the column types according to their type
- Reload the data with the SQL query changes done earlier by entering the SQL query in advanced options. 
![image](https://github.com/user-attachments/assets/47711c0f-0acd-4863-9213-a995626ec89a)

 
## Create the relationship between tables.
- Create a new table ‘Calendar’ using the DAX script below and connect with the dates in the other tables

```
Calendar = 
ADDCOLUMNS (
    CALENDAR ( DATE ( 2023, 1, 1 ), DATE ( 2025, 12, 31 ) ),
    "DateAsInteger", FORMAT ( [Date], "YYYYMMDD" ),
    "Year", YEAR ( [Date] ),
    "Monthnumber", FORMAT ( [Date], "MM" ),
    "YearMonthnumber", FORMAT ( [Date], "YYYY/MM" ),
    "YearMonthShort", FORMAT ( [Date], "YYYY/mmm" ),
    "MonthNameShort", FORMAT ( [Date], "mmm" ),
    "MonthNameLong", FORMAT ( [Date], "mmmm" ),
    "DayOfWeekNumber", WEEKDAY ( [Date] ),
    "DayOfWeek", FORMAT ( [Date], "dddd" ),
    "DayOfWeekShort", FORMAT ( [Date], "ddd" ),
    "Quarter", "Q" & FORMAT ( [Date], "Q" ),
    "YearQuarter",
        FORMAT ( [Date], "YYYY" ) & "/Q"
            & FORMAT ( [Date], "Q" )
)
``` 
![image](https://github.com/user-attachments/assets/690c6b23-d28f-430f-a50f-742e5c05072e)

## The POWER BI data visualization are created as follows:
![image](https://github.com/user-attachments/assets/bca03397-4ca6-4a58-a6b2-fe3e591918e6)
![image](https://github.com/user-attachments/assets/f15a0cbe-6822-4e7b-8a4f-da9da23572f7)
![image](https://github.com/user-attachments/assets/46100987-6ffe-41d5-a26a-a18dc202a3a7)
![image](https://github.com/user-attachments/assets/76e8f034-9400-4fc7-b474-414cb150a26c)


Overview:
•	Decreased Conversion Rates: The conversion rate demonstrated a strong rebound in December, reaching 10.2%, despite a notable dip to 5.0% in October.
•	Reduced Customer Engagement:
•	There is a decline in overall social media engagement, with views dropping throughout the year.
•	While clicks and likes are low compared to views, the click-through rate stands at 15.37%, meaning that engaged users are still interacting effectively.
•	Customer Feedback Analysis:
•	Customer ratings have remained consistent, averaging around 3.7 throughout the year.
•	Although stable, the average rating is below the target of 4.0, suggesting a need for focused improvements in customer satisfaction, for products below 3,5.

Decreased Conversion Rates:
 
•	General Conversion Trend:
•	Throughout the year, conversion rates varied, with higher numbers of products converting successfully in months like February and July. This suggests that while some products had strong seasonal peaks, there is potential to improve conversions in lower-performing months through targeted interventions.
•	Lowest Conversion Month:
•	May experienced the lowest overall conversion rate at 4.3%, with no products standing out significantly in terms of conversion. This indicates a potential need to revisit marketing strategies or promotions during this period to boost performance.
•	Highest Conversion Rates:
•	January recorded the highest overall conversion rate at 18.5%, driven significantly by the Ski Boots with a remarkable 150% conversion. This indicates a strong start to the year, likely fueled by seasonal demand and effective marketing strategies.

Reduced Customer Engagement:
 
•	Declining Views:
•	Views peaked in February and July but declined from August and on, indicating reduced audience engagement in the later half of the year.
•	Low Interaction Rates:
•	Clicks and likes remained consistently low compared to views, suggesting the need for more engaging content or stronger calls to action.
•	Content Type Performance:
•	Blog content drove the most views, especially in April and July, while social media and video content maintained steady but slightly lower engagement.
Customer Feedback Analysis:
 
•	Customer Ratings Distribution:
•	The majority of customer reviews are in the higher ratings, with 140 reviews at 4 stars and 135 reviews at 5 stars, indicating overall positive feedback. Lower ratings (1-2 stars) account for a smaller proportion, with 26 reviews at 1 star and 57 reviews at 2 stars.
•	Sentiment Analysis:
•	Positive sentiment dominates with 275 reviews, reflecting a generally satisfied customer base. Negative sentiment is present in 82 reviews, with a smaller number of mixed and neutral sentiments, suggesting some areas for improvement but overall strong customer approval.
•	Opportunity for Improvement:
•	The presence of mixed positive and mixed negative sentiments suggests that there are opportunities to convert those mixed experiences into more clearly positive ones, potentially boosting overall ratings. Addressing the specific concerns in mixed reviews could elevate customer satisfaction.

# Goals & Actions:
Goals:
•	Increase Conversion Rates:
•	Goal: Identify factors impacting the conversion rate and provide recommendations to improve it.
•	Insight: Highlight key stages where visitors drop off and suggest improvements to optimize the conversion funnel.
•	Enhance Customer Engagement:
•	Goal: Determine which types of content drive the highest engagement. 
•	Insight: Analyze interaction levels with different types of marketing content to inform better content strategies.
•	Improve Customer Feedback Scores:
•	Goal: Understand common themes in customer reviews and provide actionable insights.
•	Insight: Identify recurring positive and negative feedback to guide product and service improvements.
Actions:
•	Increase Conversion Rates:
•	Target High-Performing Product Categories: Focus marketing efforts on products with demonstrated high conversion rates, such as Kayaks, Ski Boots, and Baseball Gloves. Implement seasonal promotions or personalized campaigns during peak months (e.g., January and September) to capitalize on these trends.
•	Enhance Customer Engagement:
•	Revitalize Content Strategy: To turn around declining views and low interaction rates, experiment with more engaging content formats, such as interactive videos or user-generated content. Additionally, boost engagement by optimizing call-to-action placement in social media and blog content, particularly during historically lower-engagement months (September-December).
•	Improve Customer Feedback Scores:
•	Address Mixed and Negative Feedback: Implement a feedback loop where mixed and negative reviews are analyzed to identify common issues. Develop improvement plans to address these concerns. Consider following up with dissatisfied customers to resolve issues and encourage re-rating, aiming to move average ratings closer to the 4.0 target.


