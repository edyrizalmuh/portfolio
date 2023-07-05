---
name: "VARim: an R Package"
tools: [R, R-package, devtools]
image: ../assets/images/VARim logo.png
description: VARim is an R package that I developed as part of my master's thesis. This package can be used as an imputation method for multivariate time series data, with or without seasonality.
---

<!-- <h1 align="center">Excel Dashboard</h1> -->
<h1 align="center"><b>VARim: an R Package</b></h1><br>

<p class="text-center">
{% include elements/button.html link="https://github.com/edyrizalmuh/VARim/" text="Project Github" %}
</p>

Time series is a data type whose data points are consecutively related to each other. Therefore, missing values is an important problem in time series analysis, since missing values will inevitable affect analysis of the whole series. One of the common ways of dealing with missing values in time series is data imputation which is a technique to estimate the missing values.

VARim is an R package for time series imputation. This package is based on Vector Autoregression Imputation Method (VAR-IM) proposed by [Bashir and Wei (2018)](https://www.sciencedirect.com/science/article/abs/pii/S0925231217315515). VAR is a time series model, typically used for multivariate time series data. Hence, VARim can also be used to impute missing values in several variables simultaneously, given that the variables are related.

In addition to the original VAR-IM proposed by [Bashir and Wei (2018)](https://www.sciencedirect.com/science/article/abs/pii/S0925231217315515), I also added VAR with Decomposition-IM (VARDEC-IM) and VAR with Differencing-IM (VARD-IM). VARDEC-IM is suitable for data with seasonality, while VARD-IM is suitable for non-stationary data. VARDEC-IM used STL Decomposition, while VARD-IM used differencing before performing imputation.

A more detailed evaluation of VARim can be found [here](https://ojs3.unpatti.ac.id/index.php/barekeng/article/view/6494).