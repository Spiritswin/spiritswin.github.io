---
title: 'Survival Analysis Report — Telco Customer Churn'
date: 2026-04-18
permalink: /posts/2026/04/survival-analysis-telco-churn/
tags:
  - survival analysis
  - data science
  - customer churn
excerpt: 'A comprehensive survival analysis of telco customer churn using Kaplan-Meier estimator, Cox PH model, and AFT model.'
---

## 1. Data Preparation

### 1.1 Dataset

|**Item**|**Value**|
|---|---|
|Source|[IBM Telco Customer Churn](https://github.com/IBM/telco-customer-churn-on-icp4d)|
|Original rows (Bronze)|**7,043**|
|Columns|21|

### 1.2 Filtering for Survival Analysis

Two filters applied:

1. **Contract = "Month-to-month"** only
2. **InternetService ≠ "No"** (internet subscribers only)

| Stage | Rows | % of Original |
|---|---|---|
| Bronze (raw) | 7,043 | 100.0% |
| Silver (filtered) | **3,351** | 47.6% |

### 1.3 Churn Distribution (Silver Table)

|**Churn**|**Count**|**Percentage**|
|---|---|---|
|0 (Retained)|1,795|53.6%|
|1 (Churned)|1,556|46.4%|
|**Total**|**3,351**||

### 1.4 Contract Distribution (Full Dataset)

|**Contract**|**Count**|**Percentage**|
|---|---|---|
|Month-to-month|3,875|55.0%|
|Two year|1,695|24.1%|
|One year|1,473|20.9%|

---

## 2. Kaplan-Meier Estimator

### 2.1 What is Kaplan-Meier?

Kaplan-Meier is a **non-parametric** method that estimates the survival function S(t) — the probability that a customer survives beyond time t. It properly accounts for censored observations.

### 2.2 Population-Level Survival Curve

**Median survival time: 34.0 months**

|**Time Point**|**Survival Probability**|**Interpretation**|
|---|---|---|
|6 months|0.7803|78.0% survive at least 6 months|
|12 months|0.6950|69.5% survive at least 1 year|
|24 months|0.5753|57.5% survive at least 2 years|
|34 months|0.5000|**Median** — half have churned|
|48 months|0.3872|38.7% survive 4 years|
|60 months|0.2890|28.9% survive 5 years|

### 2.3 Covariate-Level Analysis with Log-Rank Test

The log-rank test determines whether survival curves for different groups are statistically distinguishable. **Null hypothesis (H₀):** the groups have the same survival distribution.

#### Results for all 15 categorical variables:

|**Variable**|**Levels**|**Overall p-value**|**Significant (p < 0.05)?**|
|---|---|---|---|
|**onlineSecurity**|3|< 0.000001|✅ Yes|
|**onlineBackup**|3|< 0.000001|✅ Yes|
|**deviceProtection**|3|< 0.000001|✅ Yes|
|**techSupport**|3|< 0.000001|✅ Yes|
|**partner**|2|< 0.000001|✅ Yes|
|**dependents**|2|< 0.000001|✅ Yes|
|**internetService**|2|0.000001|✅ Yes|
|**paymentMethod**|4|< 0.000001|✅ Yes|
|**multipleLines**|3|< 0.000001|✅ Yes|
|**streamingMovies**|2|0.000023|✅ Yes|
|**streamingTV**|2|0.000322|✅ Yes|
|**paperlessBilling**|2|0.003876|✅ Yes|
|gender|2|0.153317|❌ No|
|phoneService|2|0.194432|❌ No|
|seniorCitizen|2|0.723174|❌ No|

#### Key findings:

- **12 out of 15 variables** show statistically significant differences.
    
- **paymentMethod IS significant** (overall p < 0.000001).
    
- **Service-related features** are the most significant.
    

### 2.4 DSL Subscriber Survival Probabilities

|**Month**|**Survival Probability**|
|---|---|
|0|1.0000|
|3|0.8347|
|6|0.7839|
|9|0.7508|
|12|0.7270|

---

## 3. Cox Proportional Hazards Model

### 3.1 What is Cox PH?

Cox Proportional Hazards is a **semi-parametric** regression model:

$$h(t|X) = h_0(t) \times e^{\beta_1 X_1 + \beta_2 X_2 + \cdots + \beta_p X_p}$$

- **HR < 1** → protective (reduces churn risk)
    
- **HR > 1** → risk factor (increases churn risk)
    

### 3.2 Feature Encoding

|**Original Variable**|**Kept Column**|**Dropped (baseline)**|
|---|---|---|
|dependents|dependents_Yes|dependents_No|
|internetService|internetService_DSL|internetService_Fiber optic|
|onlineBackup|onlineBackup_Yes|onlineBackup_No|
|techSupport|techSupport_Yes|techSupport_No|
|paperlessBilling|paperlessBilling_Yes|paperlessBilling_No|

### 3.3 Model Results

|**Covariate**|**Coef (β)**|**Hazard Ratio exp(β)**|**p-value**|**95% CI**|
|---|---|---|---|---|
|**onlineBackup_Yes**|-0.7766|**0.4600**|< 0.001|[0.4096, 0.5165]|
|**techSupport_Yes**|-0.6392|**0.5277**|< 0.001|[0.4553, 0.6117]|
|**dependents_Yes**|-0.3287|**0.7199**|< 0.001|[0.6265, 0.8272]|
|**internetService_DSL**|-0.2173|**0.8047**|0.0002|[0.7167, 0.9034]|
|**Concordance Index: 0.6409**|||||

### 3.4 Interpretation

- **Online Backup (HR = 0.460):** 54% lower hazard of churning.
    
- **Tech Support (HR = 0.528):** 47.2% lower hazard.
    
- **DSL Internet (HR = 0.805):** 19.5% lower hazard compared to Fiber Optic.
    

### 3.5 Proportional Hazards Assumption Check

|**Variable**|**p-value**|**PH Assumption Violated?**|
|---|---|---|
|**internetService_DSL**|< 0.0001|✅ Yes|
|**onlineBackup_Yes**|< 0.0001|✅ Yes|
|**techSupport_Yes**|0.0002|✅ Yes|
|**dependents_Yes**|> 0.05|❌ No|

---

## 4. Accelerated Failure Time (AFT) Model

### 4.1 What is AFT?

AFT models how covariates "accelerate" or "decelerate" the time to event:

$$T = T_0 \times e^{\beta_1 X_1 + \beta_2 X_2 + \cdots + \beta_p X_p}$$

- **exp(β) > 1** → time to churn is **longer** (protective)
    
- **exp(β) < 1** → time to churn is **short** (risk factor)
    

### 4.2 Feature Encoding

9 covariates: partner, multipleLines, internetService_DSL, onlineSecurity, onlineBackup, deviceProtection, techSupport, paymentMethod_Bank, paymentMethod_Credit.

### 4.3 Model Results

**Median survival time: 135.51 months**

|**Metric**|**Cox PH**|**AFT (Log-Logistic)**|
|---|---|---|
|Concordance|0.6409|**0.7306**|

### 4.4 AFT Coefficients

|**Covariate**|**Coef (β)**|**exp(β)**|**p-value**|**Interpretation**|
|---|---|---|---|---|
|onlineSecurity_Yes|0.8616|2.3669|< 0.001|2.37× longer survival|
|onlineBackup_Yes|0.8128|2.2542|< 0.001|2.25× longer survival|
|paymentMethod_Credit|0.7990|2.2234|< 0.001|2.22× longer survival|
|techSupport_Yes|0.6893|1.9923|< 0.001|1.99× longer survival|

---

## 5. Customer Lifetime Value (CLV)

### 5.1 Methodology

$$\text{Expected Profit}_m = S(m) \times \text{Monthly Revenue}$$

$$\text{NPV}_m = \frac{\text{Expected Profit}_m}{(1 + \text{Monthly IRR})^m}$$

### 5.2 Customer Profile

Has dependents, Fiber Optic, has online backup, has tech support.

### 5.3 CLV Table

|**Month**|**Survival Prob**|**Expected Profit**|**NPV**|**Cumulative NPV**|
|---|---|---|---|---|
|1|0.9830|$29.49|$29.24|$29.24|
|6|0.9416|$28.25|$26.90|$168.13|
|12|0.9073|$27.22|$24.64|**$319.76**|
|24|0.8583|$25.75|$21.12|**$591.61**|
|36|0.8118|$24.35|$18.06|**$824.71**|

### 5.4 Profile Comparison (36-Month Cumulative NPV)

|**Profile**|**36-Month Cumulative NPV**|
|---|---|
|DSL + TechSupport|**$1,004.91**|
|Fiber + TechSupport|$907.05|
|Fiber, No TechSupport|**$596.69**|

---

## 6. Summary & Key Takeaways

### 6.1 Method Comparison

|**Method**|**Concordance**|**Key Assumption Met?**|
|---|---|---|
|Cox PH|0.6409|❌ 3/4 violated|
|AFT|**0.7306**|Partially|

### 6.2 Most Important Findings

1. **Median churn time is 34 months** for target segment.
    
2. **Online Backup is the strongest protective factor** (HR = 0.460).
    
3. **The AFT model outperforms Cox PH** (concordance 0.73 vs 0.64).
    
4. **Gender, phoneService, and seniorCitizen do NOT significantly affect churn.**
    

### 6.3 Business Recommendations

- **Prioritize Online Backup and Tech Support** adoption.
    
- **Monitor Fiber Optic customers** closely.
    
- **Use AFT model** for better prediction accuracy.
    

---

## Appendix: Corrections

|**Issue**|**Original**|**Corrected**|
|---|---|---|
|paymentMethod significance|Not significant|**Significant** (p < 0.000001)|
|AFT interpretation|exp(β)>1 = faster churn|**exp(β)>1 = longer survival**|
|CLV 12m NPV|$292.68|**$319.76**|
|CLV 36m NPV|$799.97|**$824.71**|
