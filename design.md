# Dark Shelf AI - Technical Design Document

## Executive Summary

Dark Shelf AI is a batch analytics system that identifies retail revenue leakage by detecting store-SKU combinations where products are in-stock but significantly underperforming expected demand. The system processes daily sales, inventory, and pricing data to flag "dark shelves," diagnose root causes with evidence-based scoring, and recommend corrective actions with projected recovery estimates.

### Goals (Hackathon Scope)

- **Detect dark shelves:** Identify store-SKU pairs where actual sales < 60% of expected for 3+ consecutive days with inventory > 0
- **Quantify impact:** Calculate revenue at risk with confidence intervals
- **Diagnose causes:** Implement 5 rule-based root cause analyzers (price, conversion, overstock, promo ineffectiveness, misallocation)
- **Recommend actions:** Generate specific recommendations (price adjustment, micro-promo, transfer, markdown) with projected recovery
- **Visualize insights:** Build interactive Streamlit dashboard with heatmap, ranked list, drill-down, and export
- **Performance:** Process 10,000 store-SKU combinations in <15 minutes on a single machine

### Non-Goals (Out of Scope)

- Real-time streaming detection (daily batch only)
- Machine learning models (rule-based heuristics sufficient)
- Automated action execution (recommendations only)
- Multi-user authentication/authorization
- Production-grade infrastructure (Kubernetes, microservices)
- Integration with live POS/OMS systems
- Competitor pricing data ingestion

---

## High-Level Architecture

The system follows a linear batch pipeline architecture optimized for daily processing:

```
┌─────────────────┐
│  Data Sources   │  CSV files: sales, inventory, pricing, impressions, stores, skus
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Ingestion &    │  Load CSVs, validate schemas, check data quality
│  Validation     │  Output: validated DataFrames
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Feature Store  │  Build store-SKU panel with derived features:
│  Panel Builder  │  - 28-day trailing averages, variance, velocity
└────────┬────────┘  - Days-of-supply, price ratios, conversion rates
         │
         ▼
┌─────────────────┐
│  Baseline       │  Calculate expected daily demand per store-SKU
│  Expected       │  Exclude anomalies (>3σ from mean)
│  Demand Engine  │  Output: expected_daily_units, confidence_interval
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Dark Shelf     │  Apply detection rule: actual < 60% expected
│  Detection      │  for 3+ consecutive days AND inventory > 0
└────────┬────────┘  Output: flagged cases with duration, underperformance %
         │
         ▼
┌─────────────────┐
│  Root Cause     │  Run 5 analyzers in parallel:
│  Analysis (RCA) │  - Price, Conversion, Overstock, Promo, Misallocation
└────────┬────────┘  Score each cause, select primary + secondary
         │
         ▼
┌─────────────────┐
│  Recommendation │  Generate actions based on primary cause:
│  Engine         │  - Price adjustment, micro-promo, transfer, markdown
└────────┬────────┘  Calculate projected recovery for each action
         │
         ▼
┌─────────────────┐
│  Output Store   │  Write dark_shelf_cases.csv
│                 │  Store intermediate results for dashboard
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Streamlit      │  Interactive dashboard: heatmap, ranked list,
│  Dashboard      │  drill-down charts, filters, CSV export
└─────────────────┘
```


---

## Data Pipeline & Scheduling

### Batch Processing Schedule

- **Frequency:** Daily at 2:00 AM (configurable via cron/scheduler)
- **Data Latency:** T-1 (process previous day's data)
- **Processing Window:** 15 minutes for 10,000 store-SKU combinations
- **Execution:** Single-threaded Python script with pandas (parallelization optional for scale)

### Pipeline Stages

1. **Ingestion (2 min):** Load 6 CSV files, parse dates, validate schemas
2. **Feature Engineering (3 min):** Build panel with 28-day rolling windows, join tables
3. **Baseline Calculation (2 min):** Compute expected demand per store-SKU
4. **Detection (1 min):** Apply consecutive-days rule, filter candidates
5. **RCA (5 min):** Run 5 analyzers, score causes, select primary/secondary
6. **Recommendations (1 min):** Generate actions, calculate projected recovery
7. **Output (1 min):** Write CSV, update dashboard data store

### Performance Optimization Notes

- Use pandas vectorized operations (avoid row-by-row iteration)
- Pre-filter to active store-SKUs (inventory > 0 in last 7 days)
- Cache category averages, store distances for reuse
- Limit historical lookback to 90 days (configurable)
- For >10K combos, consider Dask or multiprocessing

---

## Data Model & Schemas

### Input Schemas


#### sales_transactions.csv (Required)
```
store_id       : int      # Unique store identifier
sku_id         : int      # Unique SKU identifier
date           : date     # Transaction date (YYYY-MM-DD)
units_sold     : float    # Units sold (can be fractional for weighted items)
revenue        : float    # Total revenue in USD
promo_flag     : bool     # 1 if promo active, 0 otherwise
```

#### inventory_snapshot.csv (Required)
```
store_id       : int      # Unique store identifier
sku_id         : int      # Unique SKU identifier
date           : date     # Snapshot date (YYYY-MM-DD)
quantity_on_hand : float  # Current inventory level
```

#### pricing.csv (Required)
```
store_id       : int      # Unique store identifier
sku_id         : int      # Unique SKU identifier
date           : date     # Pricing date (YYYY-MM-DD)
price          : float    # Current price in USD
promo_flag     : bool     # 1 if promo active, 0 otherwise
promo_type     : string   # e.g., "BOGO", "15% off", null if no promo
```

#### impressions.csv (Optional)
```
store_id       : int      # Unique store identifier
sku_id         : int      # Unique SKU identifier
date           : date     # Impression date (YYYY-MM-DD)
impression_count : int    # Number of customer impressions/views
```


#### stores.csv (Optional)
```
store_id       : int      # Unique store identifier
store_name     : string   # Store name/label
latitude       : float    # Latitude coordinate
longitude      : float    # Longitude coordinate
region         : string   # Region/district code
```

#### skus.csv (Optional)
```
sku_id         : int      # Unique SKU identifier
sku_name       : string   # Product name
category       : string   # Product category (e.g., "Beverages", "Snacks")
brand          : string   # Brand name
cost           : float    # Unit cost in USD (for margin calculations)
```

### Output Schema

#### dark_shelf_cases.csv
```
case_id              : string   # Unique case identifier (store_sku_date)
store_id             : int      # Store identifier
sku_id               : int      # SKU identifier
detection_date       : date     # Date dark shelf was first detected
duration_days        : int      # Number of consecutive days underperforming
expected_daily_units : float    # Baseline expected units per day
actual_daily_units   : float    # Actual average units per day during dark shelf period
underperformance_pct : float    # (expected - actual) / expected * 100
revenue_at_risk      : float    # Cumulative revenue loss in USD
confidence_score     : float    # 0-100 score based on historical variance
primary_cause        : string   # Top-ranked root cause
secondary_causes     : string   # Comma-separated list of other causes
recommended_action   : string   # Specific action recommendation
projected_recovery   : float    # Estimated revenue recovery in USD
```


### Key Derived Features (Feature Store)

For each store-SKU combination, compute:

```python
# Baseline features (28-day trailing window)
baseline_mean_units      : float   # Mean daily units (excluding anomalies)
baseline_std_units       : float   # Standard deviation of daily units
baseline_cv              : float   # Coefficient of variation (std/mean)
baseline_mean_revenue    : float   # Mean daily revenue

# Velocity features
sales_velocity_7d        : float   # Units per day, last 7 days
sales_velocity_28d       : float   # Units per day, last 28 days
velocity_trend           : float   # (7d - 28d) / 28d (positive = accelerating)

# Inventory features
current_inventory        : float   # Latest quantity on hand
days_of_supply           : float   # current_inventory / sales_velocity_28d
inventory_age_days       : int     # Days since last restock (if detectable)

# Price features
current_price            : float   # Latest price
price_90d_avg            : float   # Own 90-day average price
category_price_avg       : float   # Category average price (across all stores)
price_vs_own_history     : float   # current_price / price_90d_avg
price_vs_category        : float   # current_price / category_price_avg

# Conversion features (if impressions available)
conversion_rate_current  : float   # units_sold / impressions (last 7 days)
conversion_rate_baseline : float   # units_sold / impressions (28-day avg)
conversion_drop_pct      : float   # (baseline - current) / baseline * 100

# Promo features
promo_active             : bool    # Is promo currently active?
promo_lift_pct           : float   # (promo_sales - non_promo_baseline) / non_promo_baseline * 100
```


---

## Algorithms & Formulas

### 1. Baseline Expected Demand Calculation

**Objective:** Compute expected daily units per store-SKU using trailing 28-day average, excluding anomalies.

**Algorithm:**

```python
def calculate_baseline_expected_demand(sales_history_28d):
    """
    Calculate expected daily demand with anomaly exclusion.
    
    Args:
        sales_history_28d: DataFrame with columns [date, units_sold]
    
    Returns:
        expected_daily_units, confidence_interval_lower, confidence_interval_upper
    """
    # Step 1: Calculate initial mean and std
    initial_mean = sales_history_28d['units_sold'].mean()
    initial_std = sales_history_28d['units_sold'].std()
    
    # Step 2: Exclude anomalies (>3 sigma from mean)
    lower_bound = initial_mean - 3 * initial_std
    upper_bound = initial_mean + 3 * initial_std
    filtered_sales = sales_history_28d[
        (sales_history_28d['units_sold'] >= lower_bound) &
        (sales_history_28d['units_sold'] <= upper_bound)
    ]
    
    # Step 3: Recalculate mean and std on filtered data
    if len(filtered_sales) < 14:  # Need at least 14 days
        return None, None, None  # Insufficient data
    
    expected_daily_units = filtered_sales['units_sold'].mean()
    std_daily_units = filtered_sales['units_sold'].std()
    
    # Step 4: Calculate 95% confidence interval (±1.96 std for ~20% CI)
    # Adjust for sample size using t-distribution approximation
    n = len(filtered_sales)
    margin_of_error = 1.96 * (std_daily_units / (n ** 0.5))
    
    ci_lower = expected_daily_units - margin_of_error
    ci_upper = expected_daily_units + margin_of_error
    
    return expected_daily_units, ci_lower, ci_upper
```


**Anomaly Exclusion Rationale:**
- Removes outliers caused by one-time events (stockouts, flash sales, data errors)
- 3-sigma rule captures 99.7% of normal distribution, excludes extreme values
- Recalculation after filtering ensures robust baseline
- Minimum 14 days required to avoid overfitting to noise

### 2. Dark Shelf Detection Rule

**Objective:** Identify store-SKU combinations underperforming for 3+ consecutive days with inventory.

**Algorithm:**

```python
def detect_dark_shelves(store_sku_panel, threshold=0.60, min_consecutive_days=3):
    """
    Detect dark shelf cases using consecutive-days rule.
    
    Args:
        store_sku_panel: DataFrame with [store_id, sku_id, date, actual_units, 
                         expected_units, inventory_on_hand]
        threshold: Underperformance threshold (default 0.60 = 60%)
        min_consecutive_days: Minimum consecutive days (default 3)
    
    Returns:
        DataFrame of detected cases with duration and metrics
    """
    cases = []
    
    # Group by store-SKU
    for (store_id, sku_id), group in store_sku_panel.groupby(['store_id', 'sku_id']):
        group = group.sort_values('date')
        
        # Calculate underperformance ratio
        group['performance_ratio'] = group['actual_units'] / group['expected_units']
        group['is_underperforming'] = (
            (group['performance_ratio'] < threshold) & 
            (group['inventory_on_hand'] > 0)
        )
        
        # Find consecutive sequences
        group['streak_id'] = (group['is_underperforming'] != 
                              group['is_underperforming'].shift()).cumsum()
        
        # Filter to underperforming streaks >= min_consecutive_days
        streaks = group[group['is_underperforming']].groupby('streak_id')
        
        for streak_id, streak_data in streaks:
            if len(streak_data) >= min_consecutive_days:
                cases.append({
                    'store_id': store_id,
                    'sku_id': sku_id,
                    'detection_date': streak_data['date'].min(),
                    'duration_days': len(streak_data),
                    'expected_daily_units': streak_data['expected_units'].mean(),
                    'actual_daily_units': streak_data['actual_units'].mean()
                })
    
    return pd.DataFrame(cases)
```


### 3. Underperformance Percentage

```python
underperformance_pct = ((expected_daily_units - actual_daily_units) / expected_daily_units) * 100
```

Example: Expected 10 units/day, actual 3 units/day → (10-3)/10 * 100 = 70% underperformance

### 4. Revenue at Risk

```python
def calculate_revenue_at_risk(case, pricing_data):
    """
    Calculate cumulative revenue loss during dark shelf period.
    
    Args:
        case: Dict with store_id, sku_id, detection_date, duration_days,
              expected_daily_units, actual_daily_units
        pricing_data: DataFrame with current price for store-SKU
    
    Returns:
        revenue_at_risk (float)
    """
    # Get current price (or average price during period)
    current_price = pricing_data[
        (pricing_data['store_id'] == case['store_id']) &
        (pricing_data['sku_id'] == case['sku_id'])
    ]['price'].iloc[-1]
    
    # Daily revenue loss
    daily_units_lost = case['expected_daily_units'] - case['actual_daily_units']
    daily_revenue_loss = daily_units_lost * current_price
    
    # Cumulative loss over duration
    revenue_at_risk = daily_revenue_loss * case['duration_days']
    
    return revenue_at_risk
```

### 5. Confidence Score

**Objective:** Quantify detection reliability based on historical variance and volume.

```python
def calculate_confidence_score(expected_units, std_units, actual_units, sample_size):
    """
    Calculate confidence score (0-100) for dark shelf detection.
    
    Higher confidence when:
    - Low coefficient of variation (stable demand)
    - Large sample size (more historical data)
    - Large gap between expected and actual
    
    Args:
        expected_units: Baseline expected daily units
        std_units: Standard deviation of historical units
        actual_units: Actual daily units during dark shelf period
        sample_size: Number of days in baseline calculation
    
    Returns:
        confidence_score (0-100)
    """
    # Component 1: Stability score (inverse of coefficient of variation)
    cv = std_units / expected_units if expected_units > 0 else 1.0
    stability_score = max(0, 1 - cv)  # Lower CV = higher stability
    
    # Component 2: Sample size score (more data = higher confidence)
    sample_score = min(1.0, sample_size / 28.0)  # Max at 28 days
    
    # Component 3: Gap magnitude score (larger gap = higher confidence)
    gap_ratio = (expected_units - actual_units) / expected_units
    gap_score = min(1.0, gap_ratio / 0.5)  # Max at 50% gap
    
    # Weighted combination
    confidence_score = (
        0.4 * stability_score +
        0.3 * sample_score +
        0.3 * gap_score
    ) * 100
    
    return round(confidence_score, 1)
```

**Confidence Interpretation:**
- 80-100: High confidence (stable demand, clear underperformance)
- 60-79: Medium confidence (some variance, but significant gap)
- 40-59: Low confidence (high variance or small gap)
- <40: Very low confidence (noisy data, borderline case)


---

## Root Cause Analysis (RCA)

### RCA Framework

Each root cause analyzer:
1. Computes a cause-specific score (0-100)
2. Collects evidence metrics
3. Returns confidence level and supporting data

The system selects:
- **Primary cause:** Highest scoring cause (if score > 50)
- **Secondary causes:** All causes with score > 40 (excluding primary)

### Cause 1: Price Too High

**Detection Rule:** Current price > 115% of category average OR > 110% of own 90-day average

```python
def analyze_price_cause(case, pricing_data, category_avg_price):
    """
    Analyze if high price is causing underperformance.
    
    Returns:
        score (0-100), evidence (dict)
    """
    current_price = pricing_data['price'].iloc[-1]
    own_90d_avg = pricing_data['price'].tail(90).mean()
    
    # Calculate price ratios
    price_vs_category = current_price / category_avg_price
    price_vs_own_history = current_price / own_90d_avg
    
    # Scoring logic
    score = 0
    evidence = {
        'current_price': current_price,
        'category_avg_price': category_avg_price,
        'own_90d_avg_price': own_90d_avg,
        'price_vs_category_pct': (price_vs_category - 1) * 100,
        'price_vs_own_history_pct': (price_vs_own_history - 1) * 100
    }
    
    # Score based on severity of price premium
    if price_vs_category > 1.15:  # >15% above category
        score += 50
        score += min(30, (price_vs_category - 1.15) * 100)  # +1% = +1 point
    
    if price_vs_own_history > 1.10:  # >10% above own history
        score += 20
    
    return min(100, score), evidence
```


### Cause 2: Conversion Drop

**Detection Rule:** Impressions available AND conversion rate dropped >30% vs baseline

```python
def analyze_conversion_cause(case, impressions_data, sales_data):
    """
    Analyze if conversion rate drop is causing underperformance.
    
    Returns:
        score (0-100), evidence (dict)
    """
    if impressions_data is None or len(impressions_data) == 0:
        return 0, {'reason': 'No impression data available'}
    
    # Calculate current conversion rate (last 7 days)
    recent_sales = sales_data.tail(7)['units_sold'].sum()
    recent_impressions = impressions_data.tail(7)['impression_count'].sum()
    current_conversion = recent_sales / recent_impressions if recent_impressions > 0 else 0
    
    # Calculate baseline conversion rate (28-day average, excluding anomalies)
    baseline_sales = sales_data.tail(28)['units_sold'].mean()
    baseline_impressions = impressions_data.tail(28)['impression_count'].mean()
    baseline_conversion = baseline_sales / baseline_impressions if baseline_impressions > 0 else 0
    
    # Calculate drop percentage
    conversion_drop_pct = ((baseline_conversion - current_conversion) / baseline_conversion * 100) if baseline_conversion > 0 else 0
    
    evidence = {
        'current_conversion_rate': round(current_conversion, 4),
        'baseline_conversion_rate': round(baseline_conversion, 4),
        'conversion_drop_pct': round(conversion_drop_pct, 1),
        'recent_impressions': recent_impressions
    }
    
    # Scoring logic
    score = 0
    if conversion_drop_pct > 30:
        score = 60 + min(40, (conversion_drop_pct - 30))  # Base 60, +1 per % over 30%
    
    return min(100, score), evidence
```


### Cause 3: Overstock / Low Velocity

**Detection Rule:** Days-of-supply > 45 days AND sales velocity declining

```python
def analyze_overstock_cause(case, inventory_data, sales_velocity_7d, sales_velocity_28d):
    """
    Analyze if overstock/low velocity is causing underperformance.
    
    Returns:
        score (0-100), evidence (dict)
    """
    current_inventory = inventory_data['quantity_on_hand'].iloc[-1]
    
    # Calculate days of supply
    days_of_supply = current_inventory / sales_velocity_28d if sales_velocity_28d > 0 else 999
    
    # Calculate velocity trend
    velocity_trend = ((sales_velocity_7d - sales_velocity_28d) / sales_velocity_28d * 100) if sales_velocity_28d > 0 else 0
    
    evidence = {
        'current_inventory': current_inventory,
        'days_of_supply': round(days_of_supply, 1),
        'sales_velocity_7d': round(sales_velocity_7d, 2),
        'sales_velocity_28d': round(sales_velocity_28d, 2),
        'velocity_trend_pct': round(velocity_trend, 1)
    }
    
    # Scoring logic
    score = 0
    if days_of_supply > 45:
        score += 40
        score += min(30, (days_of_supply - 45) / 2)  # +1 per 2 days over 45
    
    if velocity_trend < -10:  # Declining velocity
        score += 30
    
    return min(100, score), evidence
```


### Cause 4: Promo Ineffectiveness

**Detection Rule:** Promo active BUT sales lift < 20% vs non-promo baseline

```python
def analyze_promo_cause(case, sales_data, pricing_data):
    """
    Analyze if ineffective promo is causing underperformance.
    
    Returns:
        score (0-100), evidence (dict)
    """
    # Check if promo is currently active
    current_promo = pricing_data['promo_flag'].iloc[-1]
    
    if not current_promo:
        return 0, {'reason': 'No promo currently active'}
    
    # Calculate promo sales (last 7 days with promo)
    promo_sales = sales_data[pricing_data['promo_flag'] == 1].tail(7)['units_sold'].mean()
    
    # Calculate non-promo baseline (last 28 days without promo, before current promo)
    non_promo_sales = sales_data[pricing_data['promo_flag'] == 0].tail(28)['units_sold'].mean()
    
    # Calculate lift percentage
    promo_lift_pct = ((promo_sales - non_promo_sales) / non_promo_sales * 100) if non_promo_sales > 0 else 0
    
    evidence = {
        'promo_active': True,
        'promo_type': pricing_data['promo_type'].iloc[-1],
        'promo_sales_avg': round(promo_sales, 2),
        'non_promo_baseline': round(non_promo_sales, 2),
        'promo_lift_pct': round(promo_lift_pct, 1)
    }
    
    # Scoring logic
    score = 0
    if promo_lift_pct < 20:
        score = 70 - promo_lift_pct  # Lower lift = higher score
        if promo_lift_pct < 0:  # Negative lift (worse than no promo)
            score = 90
    
    return min(100, score), evidence
```


### Cause 5: Misallocation

**Detection Rule:** Same SKU performing well (>120% expected) in nearby stores (<50km radius)

```python
def analyze_misallocation_cause(case, all_store_sku_data, stores_data):
    """
    Analyze if misallocation is causing underperformance.
    
    Returns:
        score (0-100), evidence (dict)
    """
    from math import radians, sin, cos, sqrt, atan2
    
    def haversine_distance(lat1, lon1, lat2, lon2):
        """Calculate distance between two points in km."""
        R = 6371  # Earth radius in km
        dlat = radians(lat2 - lat1)
        dlon = radians(lon2 - lon1)
        a = sin(dlat/2)**2 + cos(radians(lat1)) * cos(radians(lat2)) * sin(dlon/2)**2
        c = 2 * atan2(sqrt(a), sqrt(1-a))
        return R * c
    
    # Get current store location
    current_store = stores_data[stores_data['store_id'] == case['store_id']].iloc[0]
    
    # Find nearby stores with same SKU
    nearby_high_performers = []
    for _, other_store in stores_data.iterrows():
        if other_store['store_id'] == case['store_id']:
            continue
        
        distance = haversine_distance(
            current_store['latitude'], current_store['longitude'],
            other_store['latitude'], other_store['longitude']
        )
        
        if distance <= 50:  # Within 50km
            # Check performance of same SKU at other store
            other_performance = all_store_sku_data[
                (all_store_sku_data['store_id'] == other_store['store_id']) &
                (all_store_sku_data['sku_id'] == case['sku_id'])
            ]
            
            if len(other_performance) > 0:
                perf_ratio = other_performance['actual_units'].mean() / other_performance['expected_units'].mean()
                if perf_ratio > 1.20:  # Performing >120% of expected
                    nearby_high_performers.append({
                        'store_id': other_store['store_id'],
                        'distance_km': round(distance, 1),
                        'performance_ratio': round(perf_ratio, 2)
                    })
    
    evidence = {
        'nearby_high_performers': nearby_high_performers[:3],  # Top 3
        'count_high_performers': len(nearby_high_performers)
    }
    
    # Scoring logic
    score = 0
    if len(nearby_high_performers) > 0:
        score = 50 + min(50, len(nearby_high_performers) * 15)  # +15 per high performer
    
    return min(100, score), evidence
```


### RCA Orchestration

```python
def run_root_cause_analysis(case, all_data):
    """
    Run all 5 RCA analyzers and select primary + secondary causes.
    
    Returns:
        primary_cause, secondary_causes, all_evidence
    """
    causes = {}
    
    # Run all analyzers
    causes['price_too_high'] = analyze_price_cause(case, all_data['pricing'], all_data['category_avg'])
    causes['conversion_drop'] = analyze_conversion_cause(case, all_data['impressions'], all_data['sales'])
    causes['overstock'] = analyze_overstock_cause(case, all_data['inventory'], all_data['velocity_7d'], all_data['velocity_28d'])
    causes['promo_ineffective'] = analyze_promo_cause(case, all_data['sales'], all_data['pricing'])
    causes['misallocation'] = analyze_misallocation_cause(case, all_data['all_store_sku'], all_data['stores'])
    
    # Sort by score
    sorted_causes = sorted(causes.items(), key=lambda x: x[1][0], reverse=True)
    
    # Select primary (highest score > 50)
    primary_cause = None
    if sorted_causes[0][1][0] > 50:
        primary_cause = sorted_causes[0][0]
    
    # Select secondary (score > 40, excluding primary)
    secondary_causes = [
        cause_name for cause_name, (score, _) in sorted_causes[1:]
        if score > 40
    ]
    
    # Collect all evidence
    all_evidence = {cause_name: evidence for cause_name, (score, evidence) in causes.items()}
    
    return primary_cause, secondary_causes, all_evidence
```

---

## Recommendation Engine

The recommendation engine generates specific actions based on the primary root cause, calculates projected recovery, and applies guardrails.


### Recommendation 1: Price Adjustment

**Trigger:** Primary cause = "price_too_high"

```python
def recommend_price_adjustment(case, evidence, sku_cost=None):
    """
    Recommend optimal price reduction.
    
    Returns:
        recommendation (string), projected_recovery (float)
    """
    current_price = evidence['current_price']
    category_avg = evidence['category_avg_price']
    own_90d_avg = evidence['own_90d_avg_price']
    
    # Determine target price (lower of category avg or own history)
    target_price = min(category_avg, own_90d_avg)
    
    # Calculate reduction percentage (5-20% range)
    reduction_pct = ((current_price - target_price) / current_price) * 100
    reduction_pct = max(5, min(20, reduction_pct))  # Clamp to 5-20%
    
    suggested_price = current_price * (1 - reduction_pct / 100)
    
    # Guardrail: Don't go below cost (if available)
    if sku_cost and suggested_price < sku_cost * 1.05:  # Maintain 5% margin
        suggested_price = sku_cost * 1.05
        reduction_pct = ((current_price - suggested_price) / current_price) * 100
    
    # Project recovery (assume price elasticity of -1.5)
    # Sales increase = reduction_pct * 1.5
    expected_sales_increase_pct = reduction_pct * 1.5
    recovered_units = case['expected_daily_units'] * (expected_sales_increase_pct / 100)
    projected_recovery = recovered_units * suggested_price * 30  # 30-day projection
    
    recommendation = f"Reduce price by {reduction_pct:.0f}% from ${current_price:.2f} to ${suggested_price:.2f}"
    
    return recommendation, projected_recovery
```


### Recommendation 2: Micro-Promo

**Trigger:** Primary cause = "conversion_drop" OR "overstock"

```python
def recommend_micro_promo(case, evidence):
    """
    Recommend targeted promotion.
    
    Returns:
        recommendation (string), projected_recovery (float)
    """
    # Suggest promo intensity based on severity
    if evidence.get('conversion_drop_pct', 0) > 50:
        promo_discount = 20
    elif evidence.get('days_of_supply', 0) > 60:
        promo_discount = 15
    else:
        promo_discount = 10
    
    # Assume promo lift of 30-50% (conservative estimate)
    expected_lift_pct = 30 + (promo_discount - 10) * 2  # Higher discount = higher lift
    
    # Project recovery (7-day promo duration)
    units_gap = case['expected_daily_units'] - case['actual_daily_units']
    recovered_units = units_gap * (expected_lift_pct / 100) * 7  # 7-day promo
    current_price = evidence.get('current_price', case['revenue_at_risk'] / case['duration_days'] / case['actual_daily_units'])
    projected_recovery = recovered_units * current_price * 0.9  # Account for discount
    
    recommendation = f"Run {promo_discount}% off promo for 7 days (estimated {expected_lift_pct}% lift)"
    
    return recommendation, projected_recovery
```

### Recommendation 3: Transfer Suggestion

**Trigger:** Primary cause = "misallocation"

```python
def recommend_transfer(case, evidence):
    """
    Recommend inventory transfer to high-performing stores.
    
    Returns:
        recommendation (string), projected_recovery (float)
    """
    nearby_high_performers = evidence.get('nearby_high_performers', [])
    
    if len(nearby_high_performers) == 0:
        return "No transfer opportunity identified", 0
    
    # Calculate transfer quantity (50% of current inventory or 30 days supply at target store)
    transfer_qty = case.get('current_inventory', 0) * 0.5
    
    # Top 3 target stores
    targets = nearby_high_performers[:3]
    target_list = ", ".join([f"Store #{t['store_id']} ({t['distance_km']}km)" for t in targets])
    
    # Project recovery (assume transferred units sell at target store rate)
    avg_target_performance = sum([t['performance_ratio'] for t in targets]) / len(targets)
    recovered_units = transfer_qty * (avg_target_performance - 0.6)  # Incremental over current 60%
    current_price = evidence.get('current_price', 10)  # Fallback
    projected_recovery = recovered_units * current_price
    
    recommendation = f"Transfer {transfer_qty:.0f} units to: {target_list}"
    
    return recommendation, projected_recovery
```


### Recommendation 4: Markdown Acceleration

**Trigger:** Primary cause = "overstock" AND inventory age > 60 days

```python
def recommend_markdown(case, evidence):
    """
    Recommend aggressive clearance markdown.
    
    Returns:
        recommendation (string), projected_recovery (float)
    """
    days_of_supply = evidence.get('days_of_supply', 0)
    
    # Aggressive markdown for aged inventory
    if days_of_supply > 90:
        markdown_pct = 40
    elif days_of_supply > 60:
        markdown_pct = 30
    else:
        markdown_pct = 20
    
    # Project recovery (assume markdown clears 70% of inventory in 14 days)
    current_inventory = case.get('current_inventory', 0)
    clearance_units = current_inventory * 0.7
    current_price = evidence.get('current_price', 10)
    clearance_price = current_price * (1 - markdown_pct / 100)
    projected_recovery = clearance_units * clearance_price
    
    recommendation = f"Markdown by {markdown_pct}% to clear aged inventory (${current_price:.2f} → ${clearance_price:.2f})"
    
    return recommendation, projected_recovery
```

### Recommendation Orchestration

```python
def generate_recommendation(case, primary_cause, evidence, sku_cost=None):
    """
    Generate recommendation based on primary cause.
    
    Returns:
        recommended_action (string), projected_recovery (float)
    """
    if primary_cause == 'price_too_high':
        return recommend_price_adjustment(case, evidence['price_too_high'], sku_cost)
    
    elif primary_cause == 'conversion_drop':
        return recommend_micro_promo(case, evidence['conversion_drop'])
    
    elif primary_cause == 'overstock':
        # Check if aged inventory
        if evidence['overstock'].get('days_of_supply', 0) > 60:
            return recommend_markdown(case, evidence['overstock'])
        else:
            return recommend_micro_promo(case, evidence['overstock'])
    
    elif primary_cause == 'promo_ineffective':
        return "End current promo and revert to regular pricing", case['revenue_at_risk'] * 0.5
    
    elif primary_cause == 'misallocation':
        return recommend_transfer(case, evidence['misallocation'])
    
    else:
        return "Manual review required", 0
```


---

## Dashboard Design

The Streamlit dashboard provides interactive exploration of dark shelf cases with 4 main views.

### View 1: Store Heatmap (Geographic)

**Purpose:** Visualize dark shelf severity across store locations

**Implementation:**
```python
import streamlit as st
import pydeck as pdk

def render_store_heatmap(cases_df, stores_df):
    """
    Render geographic heatmap of dark shelf severity.
    """
    # Aggregate revenue at risk by store
    store_summary = cases_df.groupby('store_id').agg({
        'revenue_at_risk': 'sum',
        'case_id': 'count'
    }).reset_index()
    
    # Join with store locations
    map_data = store_summary.merge(stores_df, on='store_id')
    
    # Color coding by severity
    map_data['color'] = map_data['revenue_at_risk'].apply(lambda x:
        [255, 0, 0, 160] if x > 50000 else      # Red: High (>$50K)
        [255, 165, 0, 160] if x > 20000 else    # Orange: Medium ($20K-$50K)
        [255, 255, 0, 160]                       # Yellow: Low (<$20K)
    )
    
    # Render map
    st.pydeck_chart(pdk.Deck(
        map_style='mapbox://styles/mapbox/light-v9',
        initial_view_state=pdk.ViewState(
            latitude=map_data['latitude'].mean(),
            longitude=map_data['longitude'].mean(),
            zoom=6,
            pitch=0,
        ),
        layers=[
            pdk.Layer(
                'ScatterplotLayer',
                data=map_data,
                get_position='[longitude, latitude]',
                get_color='color',
                get_radius='revenue_at_risk / 100',  # Size by revenue
                pickable=True,
            ),
        ],
        tooltip={'text': 'Store {store_id}\n{case_id} cases\n${revenue_at_risk:,.0f} at risk'}
    ))
```


### View 2: Ranked Case List

**Purpose:** Sortable table of all dark shelf cases with filters

**Implementation:**
```python
def render_case_list(cases_df, skus_df):
    """
    Render filterable, sortable case list.
    """
    # Sidebar filters
    st.sidebar.header("Filters")
    
    # Store filter
    selected_stores = st.sidebar.multiselect(
        "Stores",
        options=cases_df['store_id'].unique(),
        default=[]
    )
    
    # Category filter (if SKU metadata available)
    if skus_df is not None:
        cases_with_category = cases_df.merge(skus_df[['sku_id', 'category']], on='sku_id')
        selected_categories = st.sidebar.multiselect(
            "Categories",
            options=cases_with_category['category'].unique(),
            default=[]
        )
    
    # Cause filter
    selected_causes = st.sidebar.multiselect(
        "Primary Cause",
        options=cases_df['primary_cause'].unique(),
        default=[]
    )
    
    # Revenue threshold
    min_revenue = st.sidebar.slider(
        "Min Revenue at Risk",
        min_value=0,
        max_value=int(cases_df['revenue_at_risk'].max()),
        value=0
    )
    
    # Apply filters
    filtered_df = cases_df.copy()
    if selected_stores:
        filtered_df = filtered_df[filtered_df['store_id'].isin(selected_stores)]
    if selected_causes:
        filtered_df = filtered_df[filtered_df['primary_cause'].isin(selected_causes)]
    filtered_df = filtered_df[filtered_df['revenue_at_risk'] >= min_revenue]
    
    # Display table
    st.dataframe(
        filtered_df[[
            'case_id', 'store_id', 'sku_id', 'duration_days',
            'revenue_at_risk', 'confidence_score', 'primary_cause',
            'recommended_action', 'projected_recovery'
        ]].sort_values('revenue_at_risk', ascending=False),
        use_container_width=True
    )
    
    # Export button
    csv = filtered_df.to_csv(index=False)
    st.download_button(
        label="Export to CSV",
        data=csv,
        file_name="dark_shelf_cases.csv",
        mime="text/csv"
    )
```


### View 3: Case Drill-Down

**Purpose:** Detailed time-series analysis of a specific dark shelf case

**Implementation:**
```python
import plotly.graph_objects as go
from plotly.subplots import make_subplots

def render_case_drilldown(case_id, sales_df, inventory_df, pricing_df):
    """
    Render detailed drill-down for a specific case.
    """
    case = cases_df[cases_df['case_id'] == case_id].iloc[0]
    
    # Filter data for this store-SKU
    store_id, sku_id = case['store_id'], case['sku_id']
    case_sales = sales_df[(sales_df['store_id'] == store_id) & (sales_df['sku_id'] == sku_id)]
    case_inventory = inventory_df[(inventory_df['store_id'] == store_id) & (inventory_df['sku_id'] == sku_id)]
    case_pricing = pricing_df[(pricing_df['store_id'] == store_id) & (pricing_df['sku_id'] == sku_id)]
    
    # Create subplots
    fig = make_subplots(
        rows=3, cols=1,
        subplot_titles=('Sales: Expected vs Actual', 'Inventory Level', 'Price & Promos'),
        vertical_spacing=0.1
    )
    
    # Plot 1: Expected vs Actual Sales
    fig.add_trace(
        go.Scatter(x=case_sales['date'], y=case_sales['expected_units'], 
                   name='Expected', line=dict(color='green', dash='dash')),
        row=1, col=1
    )
    fig.add_trace(
        go.Scatter(x=case_sales['date'], y=case_sales['units_sold'], 
                   name='Actual', line=dict(color='red')),
        row=1, col=1
    )
    
    # Highlight dark shelf period
    fig.add_vrect(
        x0=case['detection_date'], 
        x1=case['detection_date'] + pd.Timedelta(days=case['duration_days']),
        fillcolor="red", opacity=0.1, layer="below", line_width=0,
        row=1, col=1
    )
    
    # Plot 2: Inventory Level
    fig.add_trace(
        go.Scatter(x=case_inventory['date'], y=case_inventory['quantity_on_hand'],
                   name='Inventory', line=dict(color='blue')),
        row=2, col=1
    )
    
    # Plot 3: Price & Promos
    fig.add_trace(
        go.Scatter(x=case_pricing['date'], y=case_pricing['price'],
                   name='Price', line=dict(color='purple')),
        row=3, col=1
    )
    
    # Mark promo periods
    promo_periods = case_pricing[case_pricing['promo_flag'] == 1]
    for _, promo in promo_periods.iterrows():
        fig.add_vrect(
            x0=promo['date'], x1=promo['date'] + pd.Timedelta(days=1),
            fillcolor="green", opacity=0.2, layer="below", line_width=0,
            row=3, col=1
        )
    
    fig.update_layout(height=800, showlegend=True)
    st.plotly_chart(fig, use_container_width=True)
    
    # Display case details
    col1, col2, col3 = st.columns(3)
    col1.metric("Revenue at Risk", f"${case['revenue_at_risk']:,.0f}")
    col2.metric("Underperformance", f"{case['underperformance_pct']:.0f}%")
    col3.metric("Confidence", f"{case['confidence_score']:.0f}/100")
    
    st.subheader("Root Cause Analysis")
    st.write(f"**Primary Cause:** {case['primary_cause']}")
    st.write(f"**Secondary Causes:** {case['secondary_causes']}")
    
    st.subheader("Recommendation")
    st.info(case['recommended_action'])
    st.write(f"**Projected Recovery:** ${case['projected_recovery']:,.0f}")
```


### View 4: Summary Dashboard

**Purpose:** High-level KPIs and distribution charts

**Implementation:**
```python
def render_summary_dashboard(cases_df):
    """
    Render summary KPIs and charts.
    """
    # Top KPIs
    col1, col2, col3, col4 = st.columns(4)
    col1.metric("Total Cases", len(cases_df))
    col2.metric("Total Revenue at Risk", f"${cases_df['revenue_at_risk'].sum():,.0f}")
    col3.metric("Avg Confidence", f"{cases_df['confidence_score'].mean():.0f}/100")
    col4.metric("Stores Affected", cases_df['store_id'].nunique())
    
    # Charts
    col1, col2 = st.columns(2)
    
    with col1:
        st.subheader("Cases by Primary Cause")
        cause_counts = cases_df['primary_cause'].value_counts()
        st.bar_chart(cause_counts)
    
    with col2:
        st.subheader("Revenue at Risk by Cause")
        cause_revenue = cases_df.groupby('primary_cause')['revenue_at_risk'].sum().sort_values(ascending=False)
        st.bar_chart(cause_revenue)
    
    # Duration distribution
    st.subheader("Dark Shelf Duration Distribution")
    st.histogram(cases_df['duration_days'], bins=20)
```

---

## Error Handling & Data Quality

### Data Validation Rules

```python
def validate_input_data(sales_df, inventory_df, pricing_df):
    """
    Validate input data quality and completeness.
    
    Raises:
        ValueError if critical issues found
    """
    errors = []
    warnings = []
    
    # Check required columns
    required_sales_cols = ['store_id', 'sku_id', 'date', 'units_sold', 'revenue']
    if not all(col in sales_df.columns for col in required_sales_cols):
        errors.append(f"Sales data missing required columns: {required_sales_cols}")
    
    # Check for nulls in critical fields
    if sales_df[['store_id', 'sku_id', 'date']].isnull().any().any():
        errors.append("Sales data contains nulls in key fields")
    
    # Check date ranges
    if (sales_df['date'].max() - sales_df['date'].min()).days < 28:
        warnings.append("Sales data has <28 days history; baseline may be unreliable")
    
    # Check for negative values
    if (sales_df['units_sold'] < 0).any():
        warnings.append("Sales data contains negative units_sold; will be filtered")
    
    # Check for gaps in date series
    date_range = pd.date_range(sales_df['date'].min(), sales_df['date'].max())
    missing_dates = set(date_range) - set(sales_df['date'].unique())
    if len(missing_dates) > 0:
        warnings.append(f"Sales data has {len(missing_dates)} missing dates; will interpolate")
    
    # Check referential integrity
    sales_store_skus = set(zip(sales_df['store_id'], sales_df['sku_id']))
    inventory_store_skus = set(zip(inventory_df['store_id'], inventory_df['sku_id']))
    missing_inventory = sales_store_skus - inventory_store_skus
    if len(missing_inventory) > 0:
        warnings.append(f"{len(missing_inventory)} store-SKUs in sales but not in inventory")
    
    if errors:
        raise ValueError(f"Data validation failed: {'; '.join(errors)}")
    
    if warnings:
        print(f"Data quality warnings: {'; '.join(warnings)}")
    
    return True
```


### Missing Data Handling

```python
def handle_missing_data(df, column, strategy='forward_fill'):
    """
    Handle missing data with appropriate strategy.
    
    Strategies:
    - forward_fill: Use last known value (for prices, inventory)
    - interpolate: Linear interpolation (for sales)
    - drop: Remove rows with missing values (for key identifiers)
    - zero: Fill with 0 (for optional metrics)
    """
    if strategy == 'forward_fill':
        df[column] = df.groupby(['store_id', 'sku_id'])[column].fillna(method='ffill')
    elif strategy == 'interpolate':
        df[column] = df.groupby(['store_id', 'sku_id'])[column].interpolate(method='linear')
    elif strategy == 'drop':
        df = df.dropna(subset=[column])
    elif strategy == 'zero':
        df[column] = df[column].fillna(0)
    
    return df
```

### Error Recovery

- **Insufficient historical data:** Skip store-SKU, log warning
- **Missing pricing data:** Use category average price
- **Missing inventory data:** Assume in-stock if recent sales exist
- **Missing optional data (impressions, stores, skus):** Disable dependent analyzers
- **Calculation errors (division by zero):** Return null, flag for manual review

---

## Extensibility

### Future Enhancement 1: ML-Based Baseline

**Current:** Rule-based 28-day trailing average  
**Future:** Time-series forecasting model (Prophet, LSTM)

**Benefits:**
- Capture seasonality, trends, holidays
- Improve accuracy for new products (cold start)
- Adapt to changing demand patterns

**Implementation Path:**
1. Collect 6+ months of historical data
2. Train per-category or per-SKU models
3. A/B test ML baseline vs rule-based
4. Gradually roll out if accuracy improves >10%


### Future Enhancement 2: External Signals

**Current:** Internal data only (sales, inventory, pricing)  
**Future:** Integrate competitor pricing, weather, local events, social media sentiment

**Benefits:**
- Detect external causes (competitor promo, bad weather)
- Improve RCA accuracy
- Provide context for recommendations

**Implementation Path:**
1. Add data connectors for external APIs
2. Extend feature store with external signals
3. Add new RCA analyzers (e.g., "competitor_undercut", "weather_impact")
4. Train correlation models to weight external factors

### Future Enhancement 3: Real-Time Detection

**Current:** Daily batch processing (T-1 latency)  
**Future:** Streaming pipeline with hourly/real-time alerts

**Benefits:**
- Faster response to emerging issues
- Intraday action recommendations
- Reduce revenue leakage window

**Implementation Path:**
1. Migrate to streaming architecture (Kafka, Flink)
2. Implement incremental baseline updates
3. Add alerting system (email, Slack, SMS)
4. Build mobile app for field managers

### Future Enhancement 4: Automated Action Execution

**Current:** Recommendations only (manual execution)  
**Future:** Closed-loop system with automated price changes, promo triggers, transfer orders

**Benefits:**
- Eliminate manual intervention lag
- Scale to 100K+ store-SKUs
- Measure actual impact vs projected

**Implementation Path:**
1. Integrate with POS/pricing systems (APIs)
2. Add approval workflows and guardrails
3. Implement A/B testing framework
4. Build feedback loop to refine recommendations

---

## Testing Plan

### Synthetic Data Generation

Create test dataset with planted dark shelf scenarios:

```python
def generate_synthetic_data(num_stores=100, num_skus=50, num_days=60):
    """
    Generate synthetic sales, inventory, pricing data with planted dark shelves.
    """
    # Scenario 1: Price too high (10 cases)
    # - Store 1-10, SKU 1: price increased 25% on day 30
    # - Expected: sales drop to 40% of baseline
    
    # Scenario 2: Conversion drop (5 cases)
    # - Store 11-15, SKU 2: impressions stable, conversion drops 50% on day 35
    
    # Scenario 3: Overstock (8 cases)
    # - Store 16-23, SKU 3: inventory 3x normal, velocity declining
    
    # Scenario 4: Promo ineffective (7 cases)
    # - Store 24-30, SKU 4: promo active but no lift
    
    # Scenario 5: Misallocation (5 cases)
    # - Store 31-35, SKU 5: underperforming, but Store 36-40 overperforming
    
    # Generate baseline normal data for remaining store-SKUs
    # Add realistic noise, seasonality, missing data
    
    return sales_df, inventory_df, pricing_df, impressions_df, stores_df, skus_df
```


### Unit Tests

```python
# test_baseline.py
def test_baseline_calculation_excludes_anomalies():
    """Test that 3-sigma outliers are excluded from baseline."""
    sales = pd.DataFrame({
        'date': pd.date_range('2024-01-01', periods=28),
        'units_sold': [10]*25 + [100, 5, 10]  # Two outliers
    })
    expected, ci_lower, ci_upper = calculate_baseline_expected_demand(sales)
    assert 9.5 < expected < 10.5  # Should be ~10, not skewed by outliers

def test_baseline_requires_minimum_data():
    """Test that baseline returns None if insufficient data."""
    sales = pd.DataFrame({
        'date': pd.date_range('2024-01-01', periods=10),
        'units_sold': [10]*10
    })
    expected, _, _ = calculate_baseline_expected_demand(sales)
    assert expected is None

# test_detection.py
def test_dark_shelf_detection_consecutive_days():
    """Test that detection requires 3+ consecutive days."""
    panel = pd.DataFrame({
        'store_id': [1]*5,
        'sku_id': [100]*5,
        'date': pd.date_range('2024-01-01', periods=5),
        'actual_units': [3, 3, 3, 10, 3],  # Only 3 consecutive
        'expected_units': [10]*5,
        'inventory_on_hand': [50]*5
    })
    cases = detect_dark_shelves(panel, threshold=0.60, min_consecutive_days=3)
    assert len(cases) == 1
    assert cases.iloc[0]['duration_days'] == 3

def test_dark_shelf_requires_inventory():
    """Test that detection requires inventory > 0."""
    panel = pd.DataFrame({
        'store_id': [1]*3,
        'sku_id': [100]*3,
        'date': pd.date_range('2024-01-01', periods=3),
        'actual_units': [3, 3, 3],
        'expected_units': [10]*3,
        'inventory_on_hand': [0, 0, 0]  # Out of stock
    })
    cases = detect_dark_shelves(panel)
    assert len(cases) == 0

# test_rca.py
def test_price_cause_detection():
    """Test that price analyzer flags high prices."""
    case = {'store_id': 1, 'sku_id': 100}
    pricing = pd.DataFrame({'price': [12.00]})
    category_avg = 10.00
    score, evidence = analyze_price_cause(case, pricing, category_avg)
    assert score > 50  # Should flag 20% premium
    assert evidence['price_vs_category_pct'] == 20.0

# test_recommendations.py
def test_price_adjustment_respects_cost_floor():
    """Test that price recommendations don't go below cost."""
    case = {'expected_daily_units': 10, 'actual_daily_units': 4}
    evidence = {'current_price': 10.00, 'category_avg_price': 8.00, 'own_90d_avg_price': 9.00}
    sku_cost = 8.50
    recommendation, recovery = recommend_price_adjustment(case, evidence, sku_cost)
    suggested_price = float(recommendation.split('$')[-1])
    assert suggested_price >= sku_cost * 1.05  # Maintain 5% margin
```


### Integration Tests

```python
# test_pipeline.py
def test_end_to_end_pipeline():
    """Test full pipeline from ingestion to output."""
    # Generate synthetic data with 5 planted dark shelves
    sales, inventory, pricing, impressions, stores, skus = generate_synthetic_data()
    
    # Run pipeline
    cases = run_dark_shelf_pipeline(sales, inventory, pricing, impressions, stores, skus)
    
    # Assertions
    assert len(cases) >= 5  # Should detect planted cases
    assert all(cases['revenue_at_risk'] > 0)
    assert all(cases['confidence_score'] >= 0) and all(cases['confidence_score'] <= 100)
    assert all(cases['primary_cause'].notna())
    assert all(cases['recommended_action'].notna())
    
    # Validate output schema
    required_columns = [
        'case_id', 'store_id', 'sku_id', 'detection_date', 'duration_days',
        'expected_daily_units', 'actual_daily_units', 'underperformance_pct',
        'revenue_at_risk', 'confidence_score', 'primary_cause', 'secondary_causes',
        'recommended_action', 'projected_recovery'
    ]
    assert all(col in cases.columns for col in required_columns)

def test_pipeline_performance():
    """Test that pipeline meets performance requirements."""
    import time
    
    # Generate 10,000 store-SKU combinations
    sales, inventory, pricing, _, _, _ = generate_synthetic_data(
        num_stores=500, num_skus=20, num_days=60
    )
    
    start_time = time.time()
    cases = run_dark_shelf_pipeline(sales, inventory, pricing)
    elapsed_time = time.time() - start_time
    
    assert elapsed_time < 900  # 15 minutes = 900 seconds
    print(f"Pipeline processed 10,000 combos in {elapsed_time:.1f} seconds")
```

### Validation Tests

```python
# test_validation.py
def test_planted_scenario_detection():
    """Test that each planted scenario is correctly detected and diagnosed."""
    sales, inventory, pricing, impressions, stores, skus = generate_synthetic_data()
    cases = run_dark_shelf_pipeline(sales, inventory, pricing, impressions, stores, skus)
    
    # Scenario 1: Price too high (Store 1-10, SKU 1)
    price_cases = cases[(cases['store_id'].isin(range(1, 11))) & (cases['sku_id'] == 1)]
    assert len(price_cases) >= 8  # Should detect most
    assert (price_cases['primary_cause'] == 'price_too_high').sum() >= 6
    
    # Scenario 2: Conversion drop (Store 11-15, SKU 2)
    conversion_cases = cases[(cases['store_id'].isin(range(11, 16))) & (cases['sku_id'] == 2)]
    assert len(conversion_cases) >= 3
    assert (conversion_cases['primary_cause'] == 'conversion_drop').sum() >= 2
    
    # Scenario 3: Overstock (Store 16-23, SKU 3)
    overstock_cases = cases[(cases['store_id'].isin(range(16, 24))) & (cases['sku_id'] == 3)]
    assert len(overstock_cases) >= 5
    assert (overstock_cases['primary_cause'] == 'overstock').sum() >= 4
    
    # Scenario 4: Promo ineffective (Store 24-30, SKU 4)
    promo_cases = cases[(cases['store_id'].isin(range(24, 31))) & (cases['sku_id'] == 4)]
    assert len(promo_cases) >= 4
    assert (promo_cases['primary_cause'] == 'promo_ineffective').sum() >= 3
    
    # Scenario 5: Misallocation (Store 31-35, SKU 5)
    misalloc_cases = cases[(cases['store_id'].isin(range(31, 36))) & (cases['sku_id'] == 5)]
    assert len(misalloc_cases) >= 3
    assert (misalloc_cases['primary_cause'] == 'misallocation').sum() >= 2
```

---

## Implementation Notes

### Technology Stack

- **Language:** Python 3.9+
- **Data Processing:** pandas, numpy
- **Visualization:** Streamlit, plotly, pydeck
- **Testing:** pytest
- **Scheduling:** cron (Linux/Mac) or Task Scheduler (Windows)
- **Storage:** CSV files (upgrade to SQLite/PostgreSQL for scale)

### Project Structure

```
dark_shelf_ai/
├── data/
│   ├── input/           # CSV input files
│   ├── output/          # Generated cases
│   └── synthetic/       # Test data
├── src/
│   ├── ingestion.py     # Data loading & validation
│   ├── features.py      # Feature engineering
│   ├── baseline.py      # Expected demand calculation
│   ├── detection.py     # Dark shelf detection
│   ├── rca.py           # Root cause analysis
│   ├── recommendations.py  # Action recommendations
│   └── pipeline.py      # Orchestration
├── dashboard/
│   └── app.py           # Streamlit dashboard
├── tests/
│   ├── test_baseline.py
│   ├── test_detection.py
│   ├── test_rca.py
│   ├── test_recommendations.py
│   └── test_pipeline.py
├── notebooks/
│   └── exploration.ipynb  # Data exploration
├── requirements.txt
└── README.md
```

### Development Workflow

1. **Day 1:** Data ingestion, validation, feature engineering
2. **Day 2:** Baseline calculation, detection logic, unit tests
3. **Day 3:** RCA analyzers (5 causes), recommendation engine
4. **Day 4:** Dashboard (heatmap, list, drill-down), integration tests
5. **Day 5:** Synthetic data generation, validation, demo prep

### Performance Optimization Tips

- Use `pandas.eval()` for complex expressions
- Vectorize operations with `.apply()` only as last resort
- Pre-compute category averages, store distances
- Use `groupby().agg()` instead of loops
- Consider `dask` for >100K store-SKUs
- Profile with `cProfile` to identify bottlenecks

---

## Success Criteria

The hackathon demo will be considered successful if:

1. **Detection:** Identify 30+ dark shelf cases in synthetic dataset (500 stores × 50 SKUs × 60 days)
2. **Accuracy:** Correctly diagnose primary cause for 70%+ of planted scenarios
3. **Impact:** Quantify $200K+ in revenue at risk across all cases
4. **Recommendations:** Generate specific actions for top 20 cases with projected recovery
5. **Performance:** Process 10,000 store-SKU combinations in <15 minutes
6. **Usability:** Dashboard loads in <3 seconds, supports filtering and drill-down
7. **Extensibility:** Code is modular and documented for future enhancements

---

## Appendix: Key Formulas Summary

```python
# Baseline Expected Demand
expected_daily_units = mean(sales_last_28_days, exclude_outliers_3_sigma)

# Dark Shelf Detection
is_dark_shelf = (actual < 0.60 * expected) AND (inventory > 0) for 3+ consecutive days

# Underperformance Percentage
underperformance_pct = (expected - actual) / expected * 100

# Revenue at Risk
revenue_at_risk = (expected - actual) * current_price * duration_days

# Confidence Score
confidence = 0.4 * stability + 0.3 * sample_size + 0.3 * gap_magnitude

# Price Cause Score
price_score = 50 if (price > 1.15 * category_avg) else 0
price_score += (price / category_avg - 1.15) * 100

# Projected Recovery (Price Adjustment)
recovered_units = expected * (reduction_pct * 1.5) / 100  # Elasticity -1.5
projected_recovery = recovered_units * new_price * 30  # 30-day projection
```

