# Dark Shelf AI - Requirements Specification

## Project Overview

Dark Shelf AI is a retail market intelligence system that identifies "dark shelves" — store-SKU combinations where products are in-stock but significantly underperforming expected demand. The system quantifies revenue at risk, diagnoses root causes with evidence, and recommends corrective actions.

## Problem Statement

Retailers lose revenue when products sit on shelves without selling at expected rates despite being in-stock. Traditional inventory systems only flag out-of-stock situations, missing the hidden opportunity cost of underperforming SKUs. Dark Shelf AI surfaces these invisible losses and provides actionable intelligence.

## Target Users

- Store Operations Managers
- Category Managers
- Pricing Analysts
- Inventory Planners
- Regional Retail Directors

## Functional Requirements

### FR1: Dark Shelf Detection
**Priority:** P0 (Critical)

The system must identify store-SKU combinations where actual sales significantly underperform expected demand.

**Acceptance Criteria:**
- Calculate expected daily sales rate per store-SKU based on historical baseline (trailing 28-day average, excluding anomalies)
- Flag as "dark shelf" when actual sales < 60% of expected for 3+ consecutive days while inventory on-hand > 0
- Support configurable thresholds (underperformance %, duration days, min inventory level)
- Process daily batch updates within 15 minutes for 10,000 store-SKU combinations

### FR2: Revenue Impact Quantification
**Priority:** P0 (Critical)

The system must calculate revenue at risk for each dark shelf case.

**Acceptance Criteria:**
- Calculate daily revenue loss = (Expected Units - Actual Units) × Current Price
- Aggregate cumulative loss over dark shelf duration
- Project 30-day forward loss if trend continues
- Display confidence interval (±20%) based on historical variance
- Rank cases by total revenue at risk (high to low)

### FR3: Root Cause Analysis
**Priority:** P0 (Critical)

The system must diagnose likely causes with supporting evidence.

**Acceptance Criteria:**
- **Price Analysis:** Flag if current price > 115% of category average or > 110% of own 90-day average
- **Conversion Drop:** Flag if impressions/footfall data available and conversion rate dropped >30% vs baseline
- **Overstock/Low Velocity:** Flag if days-of-supply > 45 days and sales velocity declining
- **Promo Ineffectiveness:** Flag if promo active but sales lift < 20% vs non-promo baseline
- **Misallocation:** Flag if same SKU performing well (>120% expected) in nearby stores (<50km radius)
- Assign primary cause (highest confidence score) and secondary causes
- Provide evidence metrics for each flagged cause

### FR4: Action Recommendations
**Priority:** P1 (High)

The system must recommend specific corrective actions with projected impact.

**Acceptance Criteria:**
- **Price Adjustment:** Suggest optimal price reduction (5-20%) to match competitive level, project recovery
- **Micro-Promo:** Recommend targeted promotion (e.g., "15% off this week"), estimate lift
- **Transfer Suggestion:** Identify top 3 nearby stores with higher demand, suggest transfer quantity
- **Markdown Acceleration:** For aged inventory (>60 days), recommend aggressive clearance
- Display projected revenue recovery for each action
- Allow users to accept/reject/modify recommendations

### FR5: Visualization Dashboard
**Priority:** P1 (High)

The system must provide intuitive visual interfaces for exploration.

**Acceptance Criteria:**
- **Store Heatmap:** Geographic view showing dark shelf severity by location (color-coded: red=high, yellow=medium, green=low)
- **Ranked Case List:** Sortable table with store, SKU, revenue at risk, cause, confidence, duration
- **Case Drill-Down:** Time-series chart showing expected vs actual sales trend, inventory level, price changes, promo periods
- **Filters:** By store, category, brand, date range, severity level, cause type
- **Export:** CSV download of filtered results

### FR6: Data Integration
**Priority:** P0 (Critical)

The system must ingest and process retail data from multiple sources.

**Acceptance Criteria:**
- **Required Inputs:** Daily sales transactions (store, SKU, date, units, revenue), inventory on-hand (store, SKU, date, quantity), pricing (store, SKU, date, price, promo_flag)
- **Optional Inputs:** Impressions/footfall (store, date, count), store locations (store_id, lat, lon), SKU metadata (category, brand, cost)
- Support CSV file upload and REST API ingestion
- Validate data quality (completeness, range checks, referential integrity)
- Handle missing data gracefully (skip optional fields, interpolate minor gaps)

## Non-Functional Requirements

### NFR1: Performance
- Dashboard loads in <3 seconds for 1,000 cases
- Batch detection processing completes in <15 minutes for 10,000 store-SKUs
- API response time <500ms for single case lookup

### NFR2: Scalability
- Support 500 stores × 2,000 SKUs = 1M store-SKU combinations
- Retain 90 days of historical data for analysis
- Handle 10 concurrent dashboard users

### NFR3: Usability
- Zero-training required for basic dashboard navigation
- Mobile-responsive design for field managers
- Tooltips explain all metrics and causes

### NFR4: Accuracy
- Detection precision >80% (true dark shelves / flagged cases)
- Revenue loss estimates within ±25% of actual opportunity cost
- Root cause primary diagnosis correct >70% of time (validated against expert review)

## Data Requirements

### Input Data Schema

**sales_transactions.csv**
```
store_id, sku_id, date, units_sold, revenue, promo_flag
```

**inventory_snapshot.csv**
```
store_id, sku_id, date, quantity_on_hand
```

**pricing.csv**
```
store_id, sku_id, date, price, promo_flag, promo_type
```

**impressions.csv** (optional)
```
store_id, sku_id, date, impression_count
```

**stores.csv**
```
store_id, store_name, latitude, longitude, region
```

**skus.csv**
```
sku_id, sku_name, category, brand, cost
```

### Output Data Schema

**dark_shelf_cases.csv**
```
case_id, store_id, sku_id, detection_date, duration_days, expected_daily_units, actual_daily_units, underperformance_pct, revenue_at_risk, confidence_score, primary_cause, secondary_causes, recommended_action, projected_recovery
```

## Success Metrics

- Detect 50+ dark shelf cases in demo dataset (500 stores × 100 SKUs × 30 days)
- Identify $100K+ in revenue at risk
- Provide actionable recommendations for top 20 cases
- Achieve <5 second dashboard interaction time
- Demonstrate 3 different root cause types with evidence

## Out of Scope (for Hackathon)

- Real-time streaming data processing
- Machine learning model training (use rule-based heuristics)
- Multi-language support
- Mobile native apps
- Integration with POS systems
- User authentication/authorization
- Automated action execution (recommendations only)

## Demo Scenario

**Setup:** 100 stores, 50 SKUs, 60 days of synthetic data with planted dark shelf scenarios

**Walkthrough:**
1. Show dashboard with 15 detected dark shelf cases
2. Drill into top case: "Store #42, SKU #789, $12K at risk"
3. Display cause: "Price 25% above category average"
4. Show recommendation: "Reduce price by 15% to $8.99, projected recovery $9K"
5. Show heatmap highlighting Store #42 in red
6. Filter by "Price Too High" cause, show 5 similar cases
7. Export results to CSV

## Assumptions

- Historical data is representative of normal demand patterns
- External factors (weather, events, competition) are not modeled
- Store locations are fixed (no new store openings during analysis period)
- SKU assortment is stable (no new product launches)
- Data is available with 1-day latency (T-1 processing)
