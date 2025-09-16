# Project Background

This project was carried out to **investigate a sharp drop in performance after a system upgrade** implemented in January 2025. The key issue was a suspected rise in transaction drop-offs, leading to major revenue losses. Critical KPIs such as successful transaction count, total transaction value, drop-off rate, success-after-retry rate, and average transaction value by drop-off reason/page were focused on. 

The analysis covered a 5-month period from October 2024 to February 2025, comparing **pre-upgrade (Oct–Dec 2024)** and **post-upgrade (Jan–Feb 2025)** periods. The insights are intended for the Product teams, who will use the findings to **improve user experience, optimize authentication flows, recover high-value users, and stabilize revenue performance**.

<br/>

<br/>

# Project Questions

This project was designed to assess the impact of a January 2025 system upgrade on transaction performance, focusing on customer abandonment, revenue loss, and retry behavior. The investigation was structured around three Key Questions:

**1. How do retries, failures, and drop-offs influence overall business performance?**

* What is the transaction value and counts of all dropped-off transactions, pre-upgrade and post-upgrade?

* What is the revenue difference between successful vs dropped vs failed transactions per month?

* What percentage of monthly transaction value was lost due to transactions that ended in drop-off?

* What percent of retried transactions were eventually successful after retry?

<br/>

**2. Which drop-off reasons are costing the business the most?**

* What is the total count, value of transactions, and the percentage distribution of each DropOff Reason?

* Which drop-off reason contributes most to loss in transactions above ₦500,000?

* Which is the most common drop-off page after upgrade and which drop-off page has the highest average transaction value lost per drop?

<br/>

<br/>

# Data Structure

The project analysis was conducted using a transactional dataset structured across one main table; FactTransactions.

1. FactTransactions Table

This is the central fact table containing raw records of all user transaction attempts. Each row represents one attempt and it is the backbone of the analysis.

Some of the Columns Used are:

* TransactionID: Unique identifier for each transaction attempt.
* TransactionDateTime: Timestamp used for monthly grouping (to compare pre- and post-upgrade).
* TransactionValue: Monetary value of the transaction. This is the main metric for total transaction value, drop-off value, and failed transaction value calculations.
* TransactionStatus: Categorical column indicating whether the transaction was Successful, Dropped, or Failed.
* DropOffReason: Explains why a transaction dropped (e.g., User Exit, OTP Failure, Timeout).
* DropOffPage: Indicates where in the journey the drop-off occurred (e.g., OTP Verification Page, Transfer Summary Page).

Metrics Derived from This Table: Total monthly transaction count and value, Drop-off share of value (as % of total), Retry success rate, Drop-off reason and page distribution, Segment analysis by transaction value thresholds (e.g., ₦500,000+)


<img src="Data Structure.png"/>

<br/>

<br/>

# Technical Details

This project was executed using SQL Server for data exploration and analysis. The dataset was already clean. Therefore, the focus was on applying **transformations and aggregation logic** to extract insights relevant to post-upgrade performance evaluation and user behavior analysis.

### 1. Skills & Functions Used

Skills: CASE, GROUP BY, CTE, Aggregate Functions (COUNT, SUM, AVG), ROW_NUMBER, DATENAME


### 2. Project Workflow & Methodology

I followed a structured approach broken down into these phases:

**a. Data Familiarization & Filtering**

* Applied date filters (WHERE TransactionDateTime BETWEEN '2024-10-01' AND '2025-02-28') to isolate relevant months before and after the system upgrade.

* Standardized time intervals by using DATENAME(Month, TransactionDateTime) for grouping and comparison across months.

**b. KPI Extraction and Trend Analysis**

* Used aggregations to calculate total TransactionValue, DropOff_Value, and DropOff_Count monthly.

* Employed window functions like ROW_NUMBER() to identify the most frequent drop-off pages and user behavior trends.

* Tracked performance shift using pre/post-upgrade segmentations (CASE WHEN YEAR(TransactionDateTime) = '2024' THEN 'Pre-Upgrade' ELSE 'Post-Upgrade').


**c. Retry Behavior**

* Built Common Table Expressions (CTEs) to isolate retries and calculate retry success rates using:

SUM(CASE WHEN Attempt_Cnt > 1 AND TransactionStatus = 'Successful' THEN 1 ELSE 0 END) AS Successful_After_Retries


**d. High-Value Drop-Off Transaction Focus**

* Filtered down drop-offs with high value to analyze where we’re losing premium users.

WHERE TransactionValue > 500000 AND DropOffReason IS NOT NULL

* Counted high-value drop-offs by DropOffReason and DropOffPage to determine user-initiated losses vs technical issues.

**e. Drop-Off Reason & Page-Level Insight**

* Calculated frequency and average transaction value per page


<br/>

<br/>

# Executive Summary

Over the 5-month analysis period spanning October 2024 to February 2025, a critical decline in performance metrics was observed following a major system upgrade at the start of January 2025. **Before the upgrade, the platform recorded ₦47.8 billion in successful transaction value and 147,338 completed transactions. However, post-upgrade, both figures dropped significantly to ₦28.7 billion and 89,486 transactions** respectively, marking a steep **59%** fall in value and almost **60%** drop in transaction count. While one might expect failure rates to decrease with an upgrade, the opposite occurred: drop-off transaction values surpassed successful ones in both January and February, with **47%** of all transaction value lost to drop-offs in those months. 

**The OTP Verification Page alone accounted for 49,619 drop-offs and the highest average transaction loss per session (₦322,211)**, indicating abandonment at one of the most sensitive user points. **Furthermore, success after retries declined from 47% pre-upgrade to 34% post-upgrade**, suggesting customers are not only failing more often but are also increasingly unlikely to recover from failed attempts. Alarmingly, **70%** of all drop-offs were user-initiated exits, not system errors, and User Exits accounted for over 62,000 high-value (₦500K+) transaction losses. These figures underscore eroding user trust and platform experience.

To combat this rapidly worsening trajectory, we recommend that the Product teams **immediately prioritize UX and authentication flow redesign**, particularly of the OTP Verification Page. Without swift intervention, the business risks systemic revenue erosion, poor customer retention, and long-term brand damage.

<br/>

<br/>

# Insights

## 1. How do retries, failures, and drop-offs influence overall business performance?

### 1.1 Overall transaction counts and transaction amount have gone down post-upgrade

        WITH Attempts AS (
        SELECT
        CASE WHEN YEAR(TransactionDateTime) = '2024' THEN 'Pre-Upgrade' ELSE 'Post-Upgrade' END AS Period,
        TransactionGroupID,
        TransactionID,
        TransactionDateTime,
        TransactionStatus,
        TRY_CONVERT(DECIMAL(38,2), TransactionValue) AS TransactionValue,
        ROW_NUMBER() OVER (PARTITION BY TransactionGroupID ORDER BY TransactionDateTime DESC, TransactionID DESC) AS Row_Num
        FROM FactTransactions)

        SELECT
        SUM(CASE WHEN Period = 'Pre-Upgrade' THEN TransactionValue ELSE 0 END) AS Pre_Upgrade_Transaction_Value,
        SUM(CASE WHEN Period = 'Post-Upgrade' THEN TransactionValue ELSE 0 END) AS Post_Upgrade_Transaction_Value,
        CAST(1.0 *(SUM(CASE WHEN Period = 'Post-Upgrade' THEN TransactionValue ELSE 0 END)/
            SUM(CASE WHEN Period = 'Pre-Upgrade' THEN TransactionValue ELSE 0 END)) - 1 AS DECIMAL(4,2)) AS Transaction_Value_Percent_Change,
        CAST(SUM(CASE WHEN Period = 'Pre-Upgrade' THEN 1 ELSE 0 END) AS DECIMAL(14,2)) AS Pre_Upgrade_Count,
        CAST(SUM(CASE WHEN Period = 'Post-Upgrade' THEN 1 ELSE 0 END) AS DECIMAL(14,2)) AS Post_Upgrade_Count,
        CAST(1.0 *(CAST(SUM(CASE WHEN Period = 'Post-Upgrade' THEN 1 ELSE 0 END) AS DECIMAL(14,2))/
            CAST(SUM(CASE WHEN Period = 'Pre-Upgrade' THEN 1 ELSE 0 END) AS DECIMAL(14,2))) - 1 AS DECIMAL(4,2)) AS Count_Percent_Change
        FROM Attempts
        WHERE Row_Num = 1
        AND TransactionStatus = 'Drop-Off';

<img src = "1.1.png">

During the pre-upgrade period, TransactIQ recorded a total drop-off transaction value of **₦47.8 billion**, spread across **147,338 transactions**. In contrast, the period after the upgrade (January to February 2025) saw this drop sharply to **₦28.7 billion** across **89,486 transactions**. This represents a **59.03%** decline in drop-off transaction value and a **59.74%** decline in drop-off transaction count.

**On the surface, this drop may seem like an improvement**. However, the numbers reveal a more concerning truth: **overall transaction activity is declining, not just drop-offs**. This fewer drop-offs aren’t the result of a better system, they are the result of fewer people using the system.


<br/>

### 1.2 Even though overall transactions reduced, the drop-off share got stronger post-upgrade, to the point where drop-off transaction value is now higher than successful transaction value

        WITH Attempts AS (
        SELECT
        DATENAME(MONTH, [Date]) AS Month,
        TransactionGroupID,
        TransactionDateTime,
        TransactionStatus,
        TRY_CONVERT(DECIMAL(38,2), TransactionValue) AS TransactionValue,
        ROW_NUMBER() OVER (PARTITION BY TransactionGroupID ORDER BY TransactionDateTime DESC, TransactionID DESC) AS Row_Num
        FROM FactTransactions)

        SELECT
        Month,
        CAST(SUM(CASE WHEN TransactionStatus = 'Successful' THEN TransactionValue END) AS DECIMAL(38,2)) AS Successful_Transaction_Value,
        CAST(SUM(CASE WHEN TransactionStatus = 'Drop-Off' THEN TransactionValue END) AS DECIMAL(38,2)) AS DropOff_Transaction_Value,
        CAST(SUM(CASE WHEN TransactionStatus = 'Failed' THEN TransactionValue END) AS DECIMAL(38,2)) AS Failed_Transaction_Value,
        CAST(SUM(CASE WHEN TransactionStatus = 'Successful' THEN TransactionValue END) - 
            SUM(CASE WHEN TransactionStatus = 'Drop-Off' THEN TransactionValue END) AS DECIMAL(38,2)) AS Successful_vs_DropOff,
        CAST(SUM(CASE WHEN TransactionStatus = 'Successful' THEN TransactionValue END) - 
            SUM(CASE WHEN TransactionStatus = 'Failed' THEN TransactionValue END) AS DECIMAL(38,2)) AS Successful_vs_Failed
        FROM Attempts
        WHERE Row_Num = 1
        GROUP BY Month
        ORDER BY  MIN(DATEPART(YEAR, TransactionDateTime)), MIN(DATEPART(MONTH, TransactionDateTime))

<img src = "1.2.png">

Before the upgrade, TransactIQ’s platform maintained a healthy transaction ecosystem. **Every month during this period, the value of successful transactions outperformed that of both dropped and failed transactions by a significant margin**. This positive gap continued in November and December.

**However, the narrative dramatically reversed after the upgrade**. In January 2025, drop-offs hit **₦16.8 billion, overtaking successful transactions, which fell to ₦12.4 billion**, a ₦4.3 billion shortfall. By February, the gap remained, with drop-offs at **₦11.9 billion** and successful transactions down to **₦9 billion**. This disturbing shift indicates that something within the upgraded system is discouraging completion.

It’s no longer just that fewer transactions are happening, it's that **when they do happen, they’re more likely to be abandoned before completion**.

<br/>

### 1.3 Not only do drop-off values surpass successful transaction value, but also swallow almost half of all monthly transaction value post-upgrade as nearly 47% of all transaction value in January and February was lost due to drop-offs

        WITH Attempts AS(
        SELECT
        DATENAME(MONTH, [Date]) AS Month,
        TransactionGroupID,
        TransactionDateTime,
        TransactionStatus,
        TRY_CONVERT(DECIMAL(38,2), TransactionValue) AS TransactionValue,
        ROW_NUMBER() OVER (PARTITION BY TransactionGroupID ORDER BY TransactionDateTime DESC, TransactionID DESC) AS Row_Num
        FROM FactTransactions)

        SELECT
        Month,
        SUM(TransactionValue) AS Transaction_Value,
        CAST(SUM(CASE WHEN TransactionStatus = 'Drop-Off' THEN TransactionValue END) AS DECIMAL(38,2)) AS DropOff_Value,
        CAST(SUM(CASE WHEN TransactionStatus = 'Drop-Off' THEN TransactionValue END)/SUM(TransactionValue) * 100 AS DECIMAL(38,2)) AS Percent_
        FROM Attempts
        WHERE Row_Num = 1
        GROUP BY Month
        ORDER BY  MIN(DATEPART(YEAR, TransactionDateTime)), MIN(DATEPART(MONTH, TransactionDateTime))

<img src = "1.3.png">

**In the months before the upgrade (October to December 2024)**, the share of total transaction value lost to drop-offs was relatively moderate, **remaining within a narrow and manageable range of 23% to 26%**. This consistent downward slope reflected a system that, while not perfect, was increasingly efficient at guiding transactions to completion and recovering potential revenue.

However, a massive operational and financial failure happened post-upgrade. In January 2025, **nearly 47% of all transaction value, specifically, ₦16.8 billion out of ₦35.8 billion was lost due to users dropping off mid-process**. This pattern held steady in February, with drop-offs swallowing **46.6%** of total value. This means that nearly half of the money that entered the transaction funnel never made it to the finish line.

<br/>

### 1.4 Customers are also less likely to recover when they retry, as success after retry falls steadily from 47% pre-upgrade to 34% post-upgrade

        WITH PerGroup AS (
        SELECT
        DATENAME(MONTH, [Date]) AS Month,
        TransactionGroupID,
        TransactionDateTime,
        ROW_NUMBER() OVER (PARTITION BY TransactionGroupID ORDER BY TransactionDateTime DESC, TransactionID DESC) AS Row_Num,
        COUNT(*) OVER (PARTITION BY TransactionGroupID) AS Attempt_Cnt,
        TransactionStatus AS TransactionStatus
        FROM FactTransactions),

        Final AS (
        SELECT Month, TransactionDateTime, Attempt_Cnt, TransactionStatus
        FROM PerGroup
        WHERE Row_Num = 1)

        SELECT
        Month,
        SUM(CASE WHEN Attempt_Cnt > 1 AND TransactionStatus = 'Successful' THEN 1 ELSE 0 END) AS Successful_After_Retries,
        SUM(CASE WHEN Attempt_Cnt > 1 THEN 1 ELSE 0 END) AS Total_Retries,
        CAST(100.0 * SUM(CASE WHEN Attempt_Cnt > 1 AND TransactionStatus = 'Successful' THEN 1 ELSE 0 END) / NULLIF(SUM(CASE WHEN Attempt_Cnt > 1 THEN 1 ELSE 0 END), 0) AS decimal(6,2) ) AS Percent_Successful_of_Retries
        FROM Final
        GROUP BY Month
        ORDER BY  MIN(DATEPART(YEAR, TransactionDateTime)), MIN(DATEPART(MONTH, TransactionDateTime));

<img src = "1.4.png">

**Before upgrade, customers who encountered issues during transactions could retry and often succeed**. In October 2024, **47% of all retries ended in success**. Then, in the months following, that downward trajectory worsened. By January 2025, only 34.9% of retries were successful, and in February, it held just slightly above at 34.5%. This means that **almost two-thirds of customers who attempted to retry still failed to complete their transactions**.

**The steady decline from 47% to 34% suggests the upgrade not only failed to improve the customer journey, it compounded user frustration.**

<br/>

<br/>

## 2. What Drop-Off Reasons and Interface Pages Are Costing the Business the Most and Why?

### 2.1 Most customers are choosing to abandon transactions intentionally rather than due to system errors

        WITH Count_Value AS(
        SELECT
        DropOffReason,
        COUNT(*) AS Counts,
        SUM(TransactionValue) AS TransactionValue
        FROM FactTransactions
        WHERE DropOffReason IS NOT NULL
        GROUP BY DropOffReason)

        SELECT
        DropOffReason,
        Counts,
        TransactionValue,
        CAST(100.0 * Counts/SUM(Counts) OVER() AS DECIMAL(4,2)) AS Percent_Count,
        CAST(100.0 * TransactionValue/SUM(TransactionValue) OVER() AS DECIMAL(4,2)) AS Percent_Value
        FROM Count_Value;

<img src = "2.1.png">

**Our customers are walking away.** Between October 2024 and February 2025, "User Exit" accounted for over **70% of all drop-offs**, both in count **(252,641 transactions) and value (₦81.7 billion)**. In contrast, **OTP Failures made up 27% (97,295 transactions worth ₦31.5 billion)**, while **Timeouts barely registered at 3%**.

<br/>

### 2.2 Customers' actions of intentional abandonment have also contributed to the loss of high-value transactions (above ₦500,000). These are customers we can't afford to lose

        SELECT 
        DropOffReason,
        COUNT(*) AS Counts
        FROM FactTransactions
        WHERE TransactionValue > 500000 AND DropOffReason IS NOT NULL
        GROUP BY DropOffReason

<img src = "2.2.png">

**Customers are also taking the highest-value transactions with them**. Of all drop-offs involving transactions above **₦500,000**, User Exit was responsible for **62,450** cases (over **70%** of all high-value drop-offs). In comparison, **OTP Failure accounted for 24,183** cases, and **Timeouts just 2,686**.

<br/>

### 2.3 The OTP Verification page is the most frequent point of abandonment and also the point where the highest-value transactions are lost

        SELECT
        DropOffPage, 
        COUNT(*) AS Frequency,
        AVG(TransactionValue) AS Avg_TransactionValue
        FROM FactTransactions
        WHERE DropOffPage IS NOT NULL AND YEAR(TransactionDateTime) = '2025'
        GROUP BY DropOffPage
        ORDER BY Frequency DESC;

<img src = "2.3.png">

Post-upgrade, the **OTP verification page** alone saw **49,619 drop-offs**, narrowly edging out the **Transfer Summary Page (49,381)**. But while both pages had nearly equal abandonment frequencies, the OTP Verification Page stood out for one reason; it carried the **highest average transaction value per drop (₦322,211)**. That’s **₦1,439** more per transaction than the Transfer Summary Page, which averaged **₦320,771**.


<br/>

<br/>

# Recommendations

### 1.1 Overall transaction counts and transaction amount have gone down post-upgrade

* The product team should conduct a **full UX and load performance audit of the post-upgrade system**. This must include stress-testing each core journey for time-to-complete and click abandonment points.

<br/>

### 1.2 Even though overall transactions reduced, the drop-off share got stronger post-upgrade, to the point where drop-off transaction value is now higher than successful transaction value

* Fixing the OTP page and transfer summary stage first will give the most immediate uplift.

* UX researchers should run micro user testing to understand hesitation patterns.

<br/>

### 1.3 Not only do drop-off values surpass successful transaction value, but also swallow almost half of all monthly transaction value post-upgrade as nearly 47% of all transaction value in January and February was lost due to drop-offs

* Monitor real-time exits and feed insights back into development sprints using session analytics tools like [Hotjar](https://www.hotjar.com/), [Smartlook](https://www.smartlook.com/)

<br/>

### 1.4 Customers are also less likely to recover when they retry, as success after retry falls steadily from 47% pre-upgrade to 34% post-upgrade

* Redesign the retry flow to include confidence-boosting elements like **“Retry now”** with expected completion time.

* Fix issues that make customers retry in the first place.

<br/>

-- <br/>

### 2.1 Most customers are choosing to abandon transactions intentionally rather than due to system errors

* Review all pages' UX design, transaction flow, perceived risk, or a lack of clarity at key moments. 

* Create and share feedback forms with customers to understand why customers opt out.

<br/>

### 2.2 Customers' actions of intentional abandonment have also contributed to the loss of high-value transactions (above ₦500,000). These are customers we can't afford to lose

* Optimise pages to remove complicated process as upgraded experience is not retaining premium customers. 

<br/>

### 2.3 The OTP Verification page is the most frequent point of abandonment and also the point where the highest-value transactions are lost

* Urgent Review and Optimisation: The OTP Verification page needs to be **simplified, secured, and stress-tested** to ensure smoother user experience as it has become the most critical failure point in the new interface

<br/>

<br/>

# Caveats and Assumptions

* High-Value Threshold at ₦500,000:

For high-value analysis, it was assumed that ₦500,000 is an appropriate business-defined threshold to classify transactions as high-value. 

* OTP Page Complexity:

The OTP Verification Page appears prominently in drop-offs and value loss, but without OTP delivery logs (e.g., failed/missed deliveries), we cannot isolate whether the issue is UX-related or backend delivery failure.

<br/>

<br/>

<br/>

<br/>

Go to my Github *[Homepage](https://github.com/Blessing336)*


