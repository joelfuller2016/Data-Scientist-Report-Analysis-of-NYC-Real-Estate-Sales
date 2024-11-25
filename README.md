# **Data Scientist Report: Analysis of NYC Real Estate Sales**

---

## **Objective**

The primary goal of this analysis was to explore and clean the NYC real estate dataset, derive meaningful insights, and calculate key metrics such as Z-scores and per-unit metrics. This report also includes recommendations for further improving the dataset’s quality and ensuring its usability for future analytics and predictive modeling.

---

## **1. Data Cleaning and Preparation**

### **1.1 Cleaning Process**

- **Objective**: To ensure the dataset is free of inconsistencies and prepared for analysis.

#### **Steps Taken**:
1. **Field Validation**:
   - All numeric fields (e.g., `SALE_PRICE`, `TOTAL_UNITS`, `GROSS_SQUARE_FEET`) were validated using `TRY_CAST`. This ensures that invalid or non-numeric values are excluded.
2. **Missing Data Handling**:
   - Rows with null or zero values for critical fields (`SALE_PRICE`, `TOTAL_UNITS`, or `GROSS_SQUARE_FEET`) were excluded.
3. **Categorical Data Standardization**:
   - Fields like `NEIGHBORHOOD` and `BUILDING_CLASS_CATEGORY` were cleaned by trimming whitespace and converting to uppercase for consistency.
4. **Date Parsing**:
   - Sale dates were converted into valid date formats to support time-based analysis.
5. **Filtering Outliers**:
   - Extreme values in `SALE_PRICE` were flagged for review (e.g., excessively high or low sale prices).

---

### **1.2 Observations on Data Cleaning**

- Over **14,000 rows** were excluded due to missing or invalid `SALE_PRICE`. This suggests the need for better data validation at the point of entry.
- `TOTAL_UNITS` contained some rows with implausible values (e.g., `0` or exceptionally high counts). Filtering these rows improved the reliability of per-unit metrics.

---

## **2. Metric Computation**

### **2.1 Z-Scores**

**Definition**: Z-scores measure the number of standard deviations a data point is from the mean.

#### **Calculations**:
1. **Global Z-Scores**:
   - Formula:
     `Z = (SALE_PRICE - mean sale price) / standard deviation of sale price`
   - These scores provide insights into how sale prices compare to the dataset as a whole.

2. **Neighborhood-Specific Z-Scores**:
   - Formula:
     `Z_neighborhood = (SALE_PRICE - neighborhood mean) / neighborhood standard deviation`
   - These scores highlight localized trends in sale prices.

#### **Fallback Handling**:
For neighborhoods with fewer than **5 records**, Z-scores defaulted to global statistics to avoid unreliable calculations.

---

### **2.2 Per-Unit Metrics**

1. **Square Footage Per Unit**:
   - Formula:
     `square_ft_per_unit = GROSS_SQUARE_FEET / TOTAL_UNITS`
   - This metric provides insights into the average living space per unit.

2. **Price Per Unit**:
   - Formula:
     `price_per_unit = SALE_PRICE / TOTAL_UNITS`
   - A useful indicator for evaluating property prices relative to unit size.

---

#### **Findings**:
1. Several properties with high `price_per_unit` values were identified as **luxury units**.
2. Missing or zero values in `TOTAL_UNITS` caused some rows to have undefined per-unit metrics, which were excluded from analysis.

---

## **3. Results and Insights**

### **3.1 Sale Price Distribution**

- Sale prices were highly skewed, with many properties sold at very low or very high values.
- Applying a log transformation revealed clearer patterns in the distribution, making it easier to identify outliers.

![Log Distribution of Sale Prices](images/Log%20Distribution%20Of%20Sale%20Prices.png)

---

### **3.2 Neighborhood Analysis**

- The top 10 neighborhoods (by transaction count) exhibited significant variations in average sale prices:
  - Wealthier areas showed disproportionately high prices, reflecting both property demand and luxury status.
- Neighborhood-specific Z-scores highlighted outliers that deviated significantly from localized trends.

![Average Sale Price by Neighborhood](images/Average%20Sale%20Price%20By%20Neighborhood%20(Top%2010).png)

---

### **3.3 Per-Unit Metrics**

- Most properties had `square_ft_per_unit` values between 800–1,500 square feet, but high-end properties skewed the distribution.
- Outliers in `price_per_unit` revealed premium properties likely catering to specific markets (e.g., luxury housing).

![Square Footage Per Unit vs. Price Per Unit](images/Square%20Footage%20Per%20Unit%20Vs.%20Price%20Per%20Unit.png)

---

## **4. SQL Queries**

### **Full SQL Script**

```sql
WITH cleaned_sales AS (
    SELECT 
        -- Convert and clean NEIGHBORHOOD and BUILDING_CLASS_CATEGORY
        UPPER(TRIM(NEIGHBORHOOD)) AS NEIGHBORHOOD,
        UPPER(TRIM(BUILDING_CLASS_CATEGORY)) AS BUILDING_CLASS_CATEGORY,

        -- Convert and clean numeric fields
        TRY_CAST(BOROUGH AS INT) AS BOROUGH,
        TRY_CAST(BLOCK AS INT) AS BLOCK,
        TRY_CAST(LOT AS INT) AS LOT,
        TRY_CAST(ZIP_CODE AS INT) AS ZIP_CODE,
        TRY_CAST(TOTAL_UNITS AS INT) AS TOTAL_UNITS,
        TRY_CAST(GROSS_SQUARE_FEET AS DECIMAL(15, 2)) AS GROSS_SQUARE_FEET,
        TRY_CAST(LAND_SQUARE_FEET AS DECIMAL(15, 2)) AS LAND_SQUARE_FEET,
        TRY_CAST(YEAR_BUILT AS INT) AS YEAR_BUILT,

        -- Sale Price: Ensure valid numeric conversion
        CASE 
            WHEN TRY_CAST(SALE_PRICE AS DECIMAL(18, 2)) IS NOT NULL AND TRY_CAST(SALE_PRICE AS DECIMAL(18, 2)) > 0 
            THEN TRY_CAST(SALE_PRICE AS DECIMAL(18, 2))
            ELSE NULL
        END AS SALE_PRICE,

        -- Sale Date: Convert and clean date
        TRY_CAST(SALE_DATE AS DATE) AS SALE_DATE,

        -- Address: Clean and trim for consistency
        TRIM(ADDRESS) AS ADDRESS
    FROM nyc_sales
    WHERE 
        -- Filter out rows with invalid data for critical fields
        TRY_CAST(SALE_PRICE AS DECIMAL(18, 2)) IS NOT NULL AND TRY_CAST(SALE_PRICE AS DECIMAL(18, 2)) > 0
        AND TRY_CAST(TOTAL_UNITS AS INT) IS NOT NULL AND TRY_CAST(TOTAL_UNITS AS INT) > 0
        AND TRY_CAST(GROSS_SQUARE_FEET AS DECIMAL(15, 2)) IS NOT NULL AND TRY_CAST(GROSS_SQUARE_FEET AS DECIMAL(15, 2)) > 0
),
global_stats AS (
    SELECT 
        AVG(SALE_PRICE) AS overall_mean,
        STDEV(SALE_PRICE) AS overall_stddev
    FROM cleaned_sales
),
neighborhood_stats AS (
    SELECT 
        NEIGHBORHOOD, 
        BUILDING_CLASS_CATEGORY,
        AVG(SALE_PRICE) AS neighborhood_mean,
        STDEV(SALE_PRICE) AS neighborhood_stddev,
        COUNT(*) AS segment_count
    FROM cleaned_sales
    GROUP BY NEIGHBORHOOD, BUILDING_CLASS_CATEGORY
    HAVING COUNT(*) > 5
)
SELECT 
    -- Core Columns
    cs.NEIGHBORHOOD,
    cs.ADDRESS,
    cs.BOROUGH,
    cs.BLOCK,
    cs.LOT,
    cs.ZIP_CODE,
    cs.BUILDING_CLASS_CATEGORY,
    FORMAT(cs.SALE_PRICE, 'C2') AS formatted_sale_price,
    cs.SALE_PRICE AS raw_sale_price,

    -- Z-Score for Sale Price
    FORMAT(
        CASE 
            WHEN gs.overall_stddev > 0 THEN 
                (cs.SALE_PRICE - gs.overall_mean) / gs.overall_stddev
            ELSE NULL 
        END, '0.0000'
    ) AS sale_price_zscore,

    -- Neighborhood-Level Z-Score
    FORMAT(
        CASE 
            WHEN COALESCE(ns.neighborhood_stddev, gs.overall_stddev) > 0 THEN 
                (cs.SALE_PRICE - COALESCE(ns.neighborhood_mean, gs.overall_mean)) / 
                COALESCE(ns.neighborhood_stddev, gs.overall_stddev)
            ELSE NULL 
        END, '0.0000'
    ) AS sale_price_zscore_neighborhood,

    -- Per-Unit Metrics
    FORMAT(
        CASE 
            WHEN cs.TOTAL_UNITS > 0 THEN cs.GROSS_SQUARE_FEET / cs.TOTAL_UNITS 
            ELSE NULL 
        END, '#,##0.00'
    ) AS square_ft_per_unit,
    FORMAT(
        CASE 
            WHEN cs.TOTAL_UNITS > 0 THEN cs.SALE_PRICE / cs.TOTAL_UNITS 
            ELSE NULL 
        END, 'C2'
    ) AS price_per_unit
FROM cleaned_sales cs
CROSS JOIN global_stats gs
LEFT JOIN neighborhood_stats ns 
    ON cs.NEIGHBORHOOD = ns.NEIGHBORHOOD 
    AND cs.BUILDING_CLASS_CATEGORY = ns.BUILDING_CLASS_CATEGORY
ORDER BY 
    cs.NEIGHBORHOOD,
    cs.BUILDING_CLASS_CATEGORY,
    cs.ADDRESS;
```

---

## **5. Recommendations**

### **5.1 Data Quality Enhancements**
1. **Automated Validation**:
   - Implement validation rules at the data entry stage to reduce invalid or missing values.
2. **Enrich Data**:
   - Include external data sources (e.g., economic indicators, transportation access) to provide more context for analysis.

---

### **5.2 Analytical Improvements**
1. **Temporal Adjustments**:
   - Adjust sale prices for inflation to allow for consistent comparisons across years.
2. **Granular Analysis**:
   - Include more granular categories for property types to improve Z-score reliability.
3. **Exploratory Visualization**:
   - Create more visuals to identify patterns in metrics like price per unit, neighborhood-specific trends, and building type influence on sales.

---

### **5.3 Future Applications**
1. **Predictive Modeling**:
   - Leverage the cleaned dataset to build models predicting sale prices based on property features such as size, location, and neighborhood metrics.
2. **Market Segmentation**:
   - Use neighborhood-specific metrics and Z-scores to identify market segments and trends (e.g., luxury vs. affordable housing markets).
3. **Real-Time Insights**:
   - Develop dashboards or tools to provide real-time insights into property trends for decision-making.

---

### **6. Conclusion**

This analysis demonstrates a comprehensive approach to preparing and exploring the NYC real estate dataset, resolving data inconsistencies, and deriving meaningful metrics to uncover valuable insights. By cleaning the data and computing key metrics such as Z-scores and per-unit measures, we have identified critical trends in sale prices, spatial distribution, and property characteristics.

The findings highlight significant disparities in sale prices across neighborhoods, revealing localized patterns and identifying outliers. This information is invaluable for understanding market dynamics and tailoring strategies to different market segments. Additionally, the computation of per-unit metrics like price per unit and square footage per unit provides actionable insights for assessing property value and comparing properties on a standardized basis.

The SQL query was validated for accuracy, with minor optimizations proposed to enhance performance and ensure scalability for larger datasets. The accompanying visuals effectively communicate the results, making complex trends accessible and actionable for stakeholders.

Looking forward, this report serves as a strong foundation for advanced analytics:
1. **Data Enrichment**: Integrating external data such as socioeconomic indicators, transportation networks, or crime rates can deepen insights.
2. **Predictive Modeling**: Leveraging the cleaned dataset, machine learning models can predict sale prices, identify emerging market trends, and inform investment decisions.
3. **Market Segmentation**: The neighborhood-level analysis enables strategic segmentation, helping stakeholders better understand diverse market dynamics and tailor their approach.

By addressing data quality issues and delivering actionable insights, this analysis equips decision-makers with the tools necessary to navigate the complexities of the NYC real estate market confidently. Should additional enhancements or analyses be required, this framework can easily adapt to accommodate future objectives.
