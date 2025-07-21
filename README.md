# HealthKart Influencer Campaign ROI Insights Dashboard

This project is a comprehensive analytics solution built to simulate, track, and visualize influencer marketing performance for HealthKart. The system measures key metrics like ROI, Incremental ROAS, influencer effectiveness, and campaign-level insights using SQL-based data modeling and a dashboard interface.

![HK Logo](images_HK.png)

## 📌 Project Overview
Influencer marketing is critical for brand visibility and growth. This open-source project helps HealthKart:

- Track campaign performance across platforms (Instagram, YouTube, Twitter)
- Analyze influencer performance (reach, revenue, ROAS)
- Compute ROI & Incremental ROAS
- Gain actionable insights on products, platforms, and personas

## 🗂 Dataset Overview
  
The project uses simulated data to represent HealthKart’s influencer campaigns across platforms like Instagram, YouTube, and Twitter. The datasets include influencer profiles, social media post metrics, user-level conversion tracking, and payout structures. Together, these tables enable full-funnel performance analysis — from reach and engagement to revenue and ROAS.

## 🧱 Data Model
Four core datasets were simulated:

- **influencers:**	Influencer profiles with category, gender, follower count, and platform
- **posts:**	Post-level metrics: reach, likes, comments
- **tracking_data:**	User-level conversions attributed to influencer campaigns
- **payouts:**	Payout contracts (per post / per order) and amounts paid

## 🧠 Assumptions

### 1. Influencer Identification

- Each influencer is uniquely identified by influencer_id.
- The same influencer may post on multiple platforms but is counted once in influencers.

### 2. Tracking Attribution

- Every tracked order in tracking_data is attributed to a single influencer using influencer_id.
- Orders and revenue in tracking_data come only from campaign traffic (i.e., paid posts).

### 3. Payouts

- Influencer payouts are based either on per post or per order.
- If basis is post, payout = rate × number_of_posts.
- If basis is order, payout = rate × number_of_orders.

### 3. Campaign-Level Metrics

- ROAS (Return on Ad Spend) = Total Revenue / Total Payout
- Incremental ROAS is assumed to be close to standard ROAS because we do not simulate a control group or organic baseline in this version.
- All revenue figures are assumed to be post-discount and net of returns.

### 4. Post Engagement

- Engagement metrics (likes, comments, reach) are simulated but assumed accurate.
- Posts with zero engagement are still valid and may be boosted content or brand-generated.

### 5. Platform Differences

- Instagram, YouTube, and Twitter posts vary in engagement and performance.
- Influencer platform is the main platform for their identity and tracking, even if they post on other platforms.


## 🖥️ Setup

### 1. Table Creation

```sql
#Influencers Table
CREATE TABLE influencers (
    influencer_id VARCHAR(10) PRIMARY KEY,
    name VARCHAR(100),
    category VARCHAR(50),
    gender VARCHAR(10),
    follower_count INT,
    platform VARCHAR(50)
);
```

```sql
#Posts Table
CREATE TABLE posts (
    post_id SERIAL PRIMARY KEY,
    influencer_id VARCHAR(10),
    platform VARCHAR(50),
    post_date DATE,
    url TEXT,
    caption TEXT,
    reach INT,
    likes INT,
    comments INT,
    FOREIGN KEY (influencer_id) REFERENCES influencers(influencer_id)
);
```

```sql
#Tracking Data
CREATE TABLE tracking_data (
    tracking_id SERIAL PRIMARY KEY,
    source VARCHAR(50),
    campaign VARCHAR(50),
    influencer_id VARCHAR(10),
    user_id UUID,
    product VARCHAR(100),
    order_date DATE,
    orders INT,
    revenue FLOAT,
    FOREIGN KEY (influencer_id) REFERENCES influencers(influencer_id)
);
```

```sql
#Payouts Table
CREATE TABLE payouts (
    payout_id SERIAL PRIMARY KEY,
    influencer_id VARCHAR(10),
    basis VARCHAR(10), -- 'post' or 'order'
    rate FLOAT,
    orders INT,
    total_payout FLOAT,
    FOREIGN KEY (influencer_id) REFERENCES influencers(influencer_id)
);
```

### 2. Data Import
Open MySQL Workbench → Select your database (e.g., healthkart) → Right-click on a table (e.g., influencers) → Table Data Import Wizard → Browse and select your CSV file → Match CSV columns with table columns → Click Next → Next → Finish.
Repeat this for each table.

## 📊 SQL Queries

### ✅ 1. Post-Level Performance Summary

Shows how each post performed based on reach, likes, and comments.

```sql
SELECT 
    p.influencer_id,
    i.name AS influencer_name,
    p.platform,
    p.date,
    p.url,
    p.reach,
    p.likes,
    p.comments,
    ROUND((p.likes + p.comments) / p.reach * 100, 2) AS engagement_rate_pct
FROM posts p
JOIN influencers i ON p.influencer_id = i.id
ORDER BY p.reach DESC;
```

### ✅ 2. Influencer-Level Performance Summary

Aggregates reach, engagement, orders, and revenue per influencer.

```sql
SELECT 
    i.id AS influencer_id,
    i.name,
    i.platform,
    i.category,
    SUM(p.reach) AS total_reach,
    SUM(p.likes + p.comments) AS total_engagement,
    ROUND(SUM(p.likes + p.comments) / SUM(p.reach) * 100, 2) AS avg_engagement_rate_pct,
    COALESCE(SUM(t.orders), 0) AS total_orders,
    COALESCE(SUM(t.revenue), 0) AS total_revenue
FROM influencers i
LEFT JOIN posts p ON i.id = p.influencer_id
LEFT JOIN tracking_data t ON i.id = t.influencer_id
GROUP BY i.id, i.name, i.platform, i.category
ORDER BY total_revenue DESC;
```

### ✅ 3. ROI Calculation

```sql
SELECT 
    i.id AS influencer_id,
    i.name,
    COALESCE(SUM(t.revenue), 0) AS total_revenue,
    COALESCE(p.total_payout, 0) AS total_payout,
    
    -- ROI Calculation
    CASE 
        WHEN COALESCE(p.total_payout, 0) = 0 THEN NULL
        ELSE ROUND((SUM(t.revenue) - p.total_payout) / p.total_payout, 2)
    END AS roi

FROM influencers i
LEFT JOIN tracking_data t ON i.id = t.influencer_id
LEFT JOIN payouts p ON i.id = p.influencer_id
GROUP BY i.id, i.name, p.total_payout
ORDER BY roi DESC NULLS LAST;
```

### ✅ 3. Incremental ROAS Calculation

```sql
SELECT 
    i.id AS influencer_id,
    i.name,
    i.platform,
    COALESCE(SUM(t.revenue), 0) AS total_revenue,
    COALESCE(p.total_payout, 0) AS total_payout,
    CASE 
        WHEN COALESCE(p.total_payout, 0) = 0 THEN NULL
        ELSE ROUND(SUM(t.revenue) / p.total_payout, 2)
    END AS incremental_roas
FROM influencers i
LEFT JOIN tracking_data t ON i.id = t.influencer_id
LEFT JOIN payouts p ON i.id = p.influencer_id
GROUP BY i.id, i.name, i.platform, p.total_payout
ORDER BY incremental_roas DESC NULLS LAST;
```

### ✅ 4. Filtering by Brand 

```sql
SELECT 
    t.campaign,
    i.name AS influencer_name,
    i.platform,
    t.product,
    t.date,
    t.revenue,
    t.orders
FROM tracking_data t
JOIN influencers i ON t.influencer_id = i.id
WHERE t.product = 'MuscleBlaze' /*Filter by one brand*/

/* Filter by multiple brands: WHERE t.product IN ('MuscleBlaze', 'HKVitals')
Filter by brand and platform: WHERE t.product = 'MuscleBlaze' AND i.platform = 'Instagram'
Filter by brand and influencer category: WHERE t.product = 'Gritzo' AND i.category = 'Fitness'*/

ORDER BY t.revenue DESC;
```
### ✅ 5. Filtering by Product

```sql
SELECT 
    t.campaign,
    i.name AS influencer_name,
    i.platform,
    t.product,
    t.date,
    t.orders,
    t.revenue
FROM tracking_data t
JOIN influencers i ON t.influencer_id = i.id
WHERE t.product = 'Whey Protein'
ORDER BY t.revenue DESC;
```

### ✅ 6. Filtering by Influencer Type

```sql
SELECT 
    i.name AS influencer_name,
    i.category AS influencer_type,
    i.platform,
    t.product,
    t.campaign,
    t.orders,
    t.revenue
FROM influencers i
JOIN tracking_data t ON i.id = t.influencer_id
WHERE i.category = 'Fitness'
ORDER BY t.revenue DESC;
```

### ✅ 7. Filtering by Platform

```sql
SELECT 
    i.name AS influencer_name,
    i.platform,
    i.category,
    t.product,
    t.revenue,
    t.orders
FROM influencers i
JOIN tracking_data t ON i.id = t.influencer_id
WHERE i.platform = 'Instagram'
ORDER BY t.revenue DESC;
```

### ✅ 8. Top Influencers (by revenue or orders)

```sql
SELECT 
    i.name AS influencer_name,
    i.platform,
    i.category,
    SUM(t.revenue) AS total_revenue,
    SUM(t.orders) AS total_orders
FROM influencers i
JOIN tracking_data t ON i.id = t.influencer_id
GROUP BY i.name, i.platform, i.category
ORDER BY total_revenue DESC
LIMIT 10; /*Shows the top 10 influencers based on total revenue*/
```

### ✅ 9. Best Personas (top influencer categories by ROI)

```sql
SELECT 
    i.category AS persona,
    COUNT(DISTINCT i.id) AS total_influencers,
    SUM(t.revenue) AS total_revenue,
    SUM(p.total_payout) AS total_payout,
    ROUND((SUM(t.revenue) - SUM(p.total_payout)) / NULLIF(SUM(p.total_payout), 0), 2) AS roi
FROM influencers i
JOIN tracking_data t ON i.id = t.influencer_id
JOIN payouts p ON i.id = p.influencer_id
GROUP BY i.category
ORDER BY roi DESC
LIMIT 5; /* Top 5*/
```

### ✅ 10. Poor ROIs (bottom influencers by ROI)

```sql
SELECT 
    i.name AS influencer_name,
    i.category,
    i.platform,
    SUM(t.revenue) AS total_revenue,
    SUM(p.total_payout) AS total_payout,
    ROUND((SUM(t.revenue) - SUM(p.total_payout)) / NULLIF(SUM(p.total_payout), 0), 2) AS roi
FROM influencers i
JOIN tracking_data t ON i.id = t.influencer_id
JOIN payouts p ON i.id = p.influencer_id
GROUP BY i.name, i.category, i.platform
HAVING roi IS NOT NULL
ORDER BY roi ASC
LIMIT 10; /*Bottom 10*/
```

## 🔍 Results

### Top Influencers (by revenue or orders)

| influencer_name      | platform  | category  | total_revenue | total_orders |
|----------------------|-----------|-----------|---------------|--------------|
| Andre Johnson        | Twitter   | Fitness   | 76,656        | 16           |
| Shelby Torres        | Instagram | Nutrition | 37,183        | 19           |
| Destiny Campos       | Instagram | Fitness   | 27,902        | 7            |
| Mary Brooks          | Twitter   | Nutrition | 24,954        | 6            |
| Michael Davis        | Instagram | Fitness   | 26,256        | 6            |
| Christopher Turner   | YouTube   | Fitness   | 21,274        | 11           |
| Erin Butler          | Twitter   | Wellness  | 14,184        | 6            |
| Ariana Reese         | Twitter   | Wellness  | 7,210         | 7            |
| Samantha Barrett     | YouTube   | Wellness  | 6,664         | 2            |
| Jonathan Alexander   | Twitter   | Nutrition | 3,880         | 19           |


### Best Personas (top influencer categories by ROI)

| persona    | total_influencers | total_revenue | total_payout | roi   |
|------------|-------------------|---------------|--------------|-------|
| Fitness    | 5                 | 152,088       | 104,573      | 0.45  |
| Wellness   | 4                 | 32,058        | 28,558       | 0.12  |
| Nutrition  | 11                | 72,917        | 78,459       | -0.07 |


### Poor ROIs (bottom influencers by ROI)

| influencer_name      | category  | platform  | total_revenue | total_payout | roi    |
|----------------------|-----------|-----------|---------------|--------------|--------|
| Stephanie Finley     | Wellness  | Instagram | 405.11        | 574          | -0.29  |
| Mary Brooks          | Nutrition | Twitter   | 24,954        | 24,954       | 0.00   |
| Jonathan Alexander   | Nutrition | Twitter   | 3,880         | 3,880        | 0.00   |
| Erin Butler          | Wellness  | Twitter   | 14,184        | 14,184       | 0.00   |
| Ariana Reese         | Wellness  | Twitter   | 7,210         | 7,210        | 0.00   |
| Samantha Barrett     | Wellness  | YouTube   | 6,664         | 6,664        | 0.00   |
| Holly Tucker         | Nutrition | Twitter   | 946           | 946          | 0.00   |
| Megan Koch           | Fitness   | Instagram | 4,247         | 4,247        | 0.00   |
| Paul Davis           | Nutrition | Twitter   | 3,898         | 3,898        | 0.00   |
| Heather Rice         | Nutrition | Twitter   | 3,245         | 3,245        | 0.00   |




