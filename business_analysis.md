# Part B: Business Case Analysis

## Scenario: Promotion Effectiveness at a Fashion Retail Chain

### B1. Problem Formulation

**(a) Machine Learning Problem Formulation**
*   **Target Variable:** `items_sold` (Continuous numerical value).
*   **Candidate Input Features:** 
    *   **Store Attributes:** `location_type`, `store_size`, `competition_density`.
    *   **Temporal Features:** `month`, `is_weekend`, `is_festival`, `is_month_end`.
    *   **Promotion Details:** `promotion_type`.
*   **Type of ML Problem:** This is a **Regression** problem because the target variable (`items_sold`) is a continuous quantity. The goal is to predict a specific count of items based on environmental and promotional factors.

**(b) Reliability of Target Variable**
*   Using `items_sold` (sales volume) is more reliable than total sales revenue because revenue can be heavily skewed by the price of items sold. For example, a few high-value items could inflate revenue without indicating a successful promotion that was intended to drive footfall or inventory turnover. 
*   **Broader Principle:** This illustrates the principle of **Metric Alignment**. The target variable should directly reflect the *outcome* the business process (the promotion) is trying to influence. In retail, promotion effectiveness is often measured by the "lift" in volume to clear stock or gain market share.

**(c) Modeling Strategy**
*   **Alternative Strategy:** Instead of one global model, I propose a **Clustered Modeling** strategy or a **Hierarchical Model**. Since stores in different locations (Urban vs. Rural) respond differently, we can cluster stores by their attributes (using the K-Means results from Part A) and train separate models for each cluster. 
*   **Justification:** A global model might average out the nuances of rural vs. urban behavior. Clustered models capture the specific sensitivities of different demographic segments while still benefiting from more data than individual store-level models.

---

### B2. Data and EDA Strategy

**(a) Data Integration**
*   **Joining Process:** 
    1.  Start with the `transactions` table.
    2.  Left Join `store_attributes` on `store_id`.
    3.  Left Join `promotion_details` on `promotion_id` (or similar key).
    4.  Join `calendar` table on `transaction_date`.
*   **Grain of Dataset:** One row = **One unique store-day combination** (Store ID + Date).
*   **Aggregations:** If raw data is at the transaction level, we must sum `items_sold` and `revenue` per store per day. We might also aggregate "Average Discount %" or "Total Promotion Duration" if applicable.

**(b) Proposed EDA Analyses**
1.  **Promotion Lift Analysis (Bar Chart):** Compare average `items_sold` during "No Promotion" vs. each specific `promotion_type`. This identifies which promotions have the highest baseline impact.
2.  **Seasonality Heatmap:** Monthly sales vs. Store Location. This reveals if certain locations have stronger seasonal peaks (e.g., Rural stores peaking during harvest festivals).
3.  **Competition Impact Scatter Plot:** `competition_density` vs. `items_sold`. This helps determine if promotions need to be more aggressive in high-competition areas.
4.  **Promotion Lag Analysis:** Plotting sales for 7 days *after* a promotion ends. This reveals if the promotion drove long-term brand loyalty or just a temporary spike.

**(c) Handling Imbalance (80% No Promotion)**
*   **Impact:** The model might become biased towards predicting the baseline sales (no promotion) and fail to capture the high-variance spikes caused by promotions.
*   **Steps to Address:**
    *   **Stratified Sampling:** Ensure training and test sets have a similar ratio of promotion vs. non-promotion days.
    *   **Oversampling:** Replicate promotion days in the training set (SMOTE for regression is also an option).
    *   **Residual Analysis:** Specifically check errors on promotion days to see if the model is consistently under-predicting during campaigns.

---

### B3. Model Evaluation and Deployment

**(a) Temporal Split Rationale**
*   **Setup:** Use the first 30 months for training and the final 6 months for testing (preserving chronological order).
*   **Why random split is inappropriate:** Random splitting leads to "look-ahead bias." In a real-world scenario, we cannot use December's sales data to help predict November's. Temporal splits ensure the model is evaluated on its ability to forecast the future using only the past.
*   **Metrics:** **MAE (Mean Absolute Error)** is preferred over RMSE here because it is more interpretable for business owners (e.g., "The model is off by 15 items on average").

**(b) Investigating Campaign Disparity**
*   **Investigation:**
    1.  Check the **Feature Importance** of the Random Forest model. If `month_December` is a top feature, the Loyalty Bonus success might be driven by holiday spending rather than the promotion itself.
    2.  Analyze **Store 12's Historical Baseline**: Did Store 12 always perform well in December?
*   **Communication:** I would present a "Promotion Lift" report showing sales *relative to the historical average* for those specific months. This "de-seasonalizes" the data to show that the Loyalty Bonus was more effective because it capitalized on high intent, whereas the Flat Discount in March failed to generate its own traffic.

**(c) Deployment and Monitoring**
*   **Deployment:** 
    1.  Save the trained pipeline using `joblib` or `pickle`.
    2.  Deploy as a REST API using Flask or FastAPI.
*   **Data Preparation:** At the start of each month, the system fetches the planned `promotion_type` and `calendar` flags (is_festival, etc.) for all 50 stores for the upcoming 30 days.
*   **Monitoring:** 
    1.  **Data Drift:** Monitor if the distribution of input features (like `competition_density`) changes over time.
    2.  **Model Decay:** Calculate "Rolling MAE" every month. If the error exceeds a pre-defined threshold (e.g., 20%), trigger a model retraining pipeline using the most recent 6 months of data.
