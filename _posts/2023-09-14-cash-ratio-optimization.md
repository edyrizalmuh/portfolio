---
title: Cash Ratio Optimization
tags: [Python, Forecasting, Time Series, VAR model, Seasonality]
style: fill
color: danger
description: Inefficient cash management increases operational expenses and reduces potential cash usage. The objective of this post is to construct a monthly predictive model for accurate cash management. Utilizing a VAR model due to interrelated variables yields a highly accurate forecasting with a 0.06 RMSLE score, just 6% off from the actual values. These precise forecasts support decision-making, optimizing the bank's cash ratio.
---

* TOC
{:toc .floating-toc}

## 1. Introduction
Financial institution use cash ratio to measure the institution ability to cover its short-term liabilities. A high cash ratio means the institution has a strong ability to cover its liabilities while a low cash ratio means the institution may struggle to cover its liabilities. However, an excessively high cash ratio are not a good sign either because the cash could have been used to generate more opportunities. Therefore, an optimal management of a cash ratio is important to prevent losses.

As a state-owned bank, Bank Rakyat Indonesia (BRI) has the same problem. In one of [its hackaton](https://www.kaggle.com/competitions/bri-data-hackathon-cr-optimization/), BRI challenge data analysts to build a machine-learning model to forecast the cash amount in its work unit, ATM, (Automated Teller Machine) and CRM (Cash Recycle Machine) for the next 31 days. An accurate forecast can help in forming a better management that can prevent BRI from losing additional expenses or potential cash usage.

BRI provides a data with 425 rows and 13 variables. The task is to predict the `kas_kantor` and `kas_echannel` columns for the next 31 days. Both variables are defined as $x$
Inline equation: $\pi$

