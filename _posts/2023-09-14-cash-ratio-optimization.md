---
title: Cash Ratio Optimization
tags: [Python, Forecasting, Time Series, VAR model, Seasonality]
style: fill
color: danger
usemathjax: true
description: Inefficient cash management increases operational expenses and reduces potential cash usage. The objective of this post is to construct a monthly predictive model for accurate cash management. Utilizing a VAR model due to interrelated variables yields a highly accurate forecasting with a 0.06 RMSLE score, just 6% off from the actual values. These precise forecasts support decision-making, optimizing the bank's cash ratio.
---

* TOC
{:toc .floating-toc}

## 1. Introduction
Financial institution use cash ratio to measure the institution ability to cover its short-term liabilities. A high cash ratio means the institution has a strong ability to cover its liabilities while a low cash ratio means the institution may struggle to cover its liabilities. However, an excessively high cash ratio is not a good sign either because the cash could have been used to generate more opportunities. Therefore, an optimal management of a cash ratio is important to prevent additional expenses or potential losses.

As a state-owned bank, Bank Rakyat Indonesia (BRI) has the same problem. In one of [its hackaton](https://www.kaggle.com/competitions/bri-data-hackathon-cr-optimization/), BRI challenged data analysts to build a machine-learning model to forecast the cash amount in its work unit, ATM, (Automated Teller Machine), and CRM (Cash Recycle Machine). An accurate forecast can help in forming a better management that can prevent BRI from losing additional expenses or potential cash usage.

BRI provides a data with 425 rows and 13 variables. The task is to predict the `kas_kantor` and `kas_echannel` columns for the next 31 days. Both variables are defined as

$$
kas\_kantor_{t} = kas\_kantor_{t-1} + cash\_in\_kantor_{t} + cash\_out\_kantor_{t} \\
kas\_echannel_{t} = kas\_echannel_{t-1} + cash\_in\_echannel_{t} + cash\_out\_echannel_{t}
$$

where `t` is time. Since `kas_kantor` and `kas_echannel` can be calculated using these formula, essentially, the task is to forecast the four `cash_in` and `cash_out` variables.



## 2. Load data

First, load the train data.
```python
import pandas as pd
df = pd.read_csv("../data/raw/train.csv")
df.head()
```

```
      periode  cash_in_echannel  cash_out_echannel  cash_in_kantor  cash_out_kantor  cr_ketetapan_total_bkn_sum          giro      deposito     kewajiban_lain      tabungan  rata_dpk_mingguan    kas_kantor  kas_echannel  
0  2019-07-31      7.303000e+08      -1.304400e+09    1.436722e+11    -1.106104e+11                         3.0  9.867358e+11  8.048153e+11       1.419685e+10  7.072647e+11       3.135744e+11  1.928940e+09  2.939100e+09  
1  2019-08-01      7.322000e+08      -8.321500e+08    3.144131e+11    -6.710987e+10                         3.0  8.962459e+11  8.125611e+11       1.234062e+10  7.011995e+11       3.135744e+11  2.492322e+11  2.839150e+09  
2  2019-08-02      1.169800e+09      -6.214000e+08    1.251294e+09    -1.142332e+09                         3.0  9.059714e+11  8.127225e+11       1.182022e+10  6.922787e+11       3.135744e+11  2.493411e+11  3.387550e+09  
3  2019-08-03      9.134500e+08      -4.240500e+08    0.000000e+00     0.000000e+00                         3.0  9.057127e+11  8.127253e+11       1.199640e+10  6.867224e+11       3.135744e+11  2.493411e+11  3.876950e+09  
4  2019-08-04      7.752500e+08      -7.779500e+08    9.883331e+10    -8.729274e+10                         3.0  9.788347e+11  8.124711e+11       1.232962e+10  6.813438e+11       3.135744e+11  2.608817e+11  3.874250e+09  
```

Train data consists of 13 variables, all of which are float except for `periode`.
```python
df.dtypes
```

```
periode                        object
cash_in_echannel              float64
cash_out_echannel             float64
cash_in_kantor                float64
cash_out_kantor               float64
cr_ketetapan_total_bkn_sum    float64
giro                          float64
deposito                      float64
kewajiban_lain                float64
tabungan                      float64
rata_dpk_mingguan             float64
kas_kantor                    float64
kas_echannel                  float64
dtype: object
```