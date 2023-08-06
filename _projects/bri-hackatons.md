---
name: "BRI Data Hackathon"
tools: [R, Machine Learning, Random Forest]
image: ../assets/images/BRI hackaton logo.png
description: This is a data science competition held by Bank Rakyat Indonesia (BRI). The data can be accessed in kaggle.
---

<!-- <h1 align="center">Excel Dashboard</h1> -->
<h1 align="center"><b>BRI Data Hackathon</b></h1><br>

* TOC
{:toc .floating-toc}

## 1. Overview
BRI Data Hackatons is a data science competition held by Bank Rakyat Indonesia (BRI). The data and competition can be accessed in [kaggle](https://www.kaggle.com/competitions/bri-data-hackathon-pa/overview). The goal is to help HR to predict employee performance for one year ahead using historical KPI data. The competition's participant is tasked to make a machine learning model that can accurately predict employee's performance. The target variable is classified into two class: whether employees are within the best performance category or not.

## 2. Explanatory Data Analysis
The first step is to load all required packages and data.
```R
library(tidyverse)
library(tidymodels)
library(skimr)
library(GGally)
library(ggpubr)
library(ggmosaic)
library(naniar) # visualizing missing data
library(vcd) # visualizing categorical data

df_train <- 
  read_csv("data/train.csv") %>% 
  mutate(source = "train") %>% # for identification
  janitor::clean_names() 
df_test <- 
  read_csv("data/test.csv")  %>% 
  mutate(source = "test") %>% # for identification
  janitor::clean_names()
df <- bind_rows(df_train, df_test) # combine train dan test
```

### 2.1 Data Cleaning
```R
skim(df_train)
```
```
── Data Summary ────────────────────────
                           Values  
Name                       df_train
Number of rows             11153   
Number of columns          22      
_______________________            
Column type frequency:             
  character                5       
  numeric                  17      
________________________           
Group variables            None    

── Variable type: character ──────────────────────────────────────────────────────────
  skim_variable              n_missing complete_rate min max empty n_unique whitespace
1 job_level                          0             1   4   4     0        3          0
2 person_level                       0             1   4   4     0        8          0
3 employee_type                      0             1   9   9     0        3          0
4 marital_status_maried_y_n          0             1   1   1     0        2          0
5 education_level                    0             1   7   7     0        6          0

── Variable type: numeric ─────────────────────────────────────────────────────────────────────────────────────────────────────  
   skim_variable                         n_missing complete_rate     mean     sd      p0      p25     p50     p75    p100 hist 
 1 job_duration_in_current_job_level             0          1       1.43   0.431    0       1.22     1.35    1.41    2.96 ▁▂▇▁▁  
 2 job_duration_in_current_person_level          0          1       1.35   0.325    0       1.22     1.35    1.39    2.83 ▁▁▇▁▁  
 3 job_duration_in_current_branch                0          1       1.03   0.417    0       0.707    1.12    1.22    2.68 ▁▇▇▁▁  
 4 gender                                        0          1       1.74   0.441    1       1        2       2       2    ▃▁▁▁▇  
 5 age                                           0          1    1986.     4.63  1963    1985     1987    1989    1997    ▁▁▂▇▁  
 6 number_of_dependences                         0          1       0.996  0.881    0       0        1       2       7    ▇▃▁▁▁  
 7 gpa                                           0          1       3.18  13.3      0       2.82     3.07    3.27  378    ▇▁▁▁▁  
 8 year_graduated                                0          1    2009.     4.12  1982    2008     2010    2012    2019    ▁▁▁▇▃  
 9 job_duration_from_training                    0          1       6.28   5.03     2       4        5       6      36    ▇▁▁▁▁  
10 branch_rotation                               0          1       3.72   2.40     1       2        3       4      22    ▇▁▁▁▁  
11 job_rotation                                  0          1       3.51   1.82     1       2        3       4      15    ▇▃▁▁▁  
12 assign_of_otherposition                       0          1       1.20   2.58     0       0        0       1      29    ▇▁▁▁▁  
13 annual_leave                                  0          1       3.66   2.65     0       2        3       5      21    ▇▃▁▁▁  
14 sick_leaves                                   0          1       1.10   2.71     0       0        0       1      77    ▇▁▁▁▁  
15 last_achievement_percent                      1          1.00   72.2   23.0      4.51   56.6     71.7    88.2   130    ▁▃▇▅▂  
16 achievement_above_100_percent_during3quartal  1          1.00    0.679  1.11     0       0        0       1       3    ▇▁▁▁▂  
17 best_performance                              0          1       0.147  0.354    0       0        0       0       1    ▇▁▁▁▂  

```

The train data has 22 columns: 5 character columns and 17 numeric columns. The data are fairly clean since there is only one missing values in `last_achievement_percent` and `achievement_above_100_percent_during3quartal`. Plotting the missing values shows that the missing values are from the same observation.

```R
gg_miss_upset(df_train)
```

{% include elements/figure.html image="assets/images/BRI - data hilang.png" caption="Missing Data in `df_train`"%}

Since the data has 11,153 rows, removing one row is a sensible option to deal with the missing values.
```R
df_train = df_train %>% filter(!is.na(last_achievement_percent))
```

Although the train data do not have missing values now, but there are messy data in `gpa`. As shown in the following plot, there are some outlier in `gpa` values. 

```R
df %>%
  ggplot(aes(as.factor(education_level), gpa, fill = source)) +
  geom_boxplot(outlier.alpha = 0.5) +
  xlab("Education Level") +
  ylab("GPA")
```

{% include elements/figure.html image="assets/images/BRI - gpa outliers.png" caption="Outliers in `gpa`"%}

Based on the plot, some outliers are between 0-100, while some are beyond 100 values. If the education level is based on Indonesia's education system, then the education level, consecutively starting from `level_0` to `level_5` should be SD, SMP, SMA, S1, S2, and S3. This make sense since there are only one row that is either `level_0` (SD), `level_1` (SMP), or `level_2` (SMA) and has more than 100 `gpa`. This could be a typo and can be solved by removing the row, or dividing the `gpa` by 10.

```R
df_train %>% filter(gpa > 100 & education_level %in% c("level_0", "level_1", "level_2"))
```
```
# A tibble: 1 × 23
  job_level job_duration_in_current_job_le…¹ person_level job_duration_in_curr…² job_duration_in_curr…³ employee_type gender   age marital_status_marie…⁴ number_of_dependences
  <chr>                                <dbl> <chr>                         <dbl>                  <dbl> <chr>          <dbl> <dbl> <chr>                                  <dbl>
1 JG05                                  2.24 PG06                           2.83                   1.22 RM_type_A          1  1969 Y                                          3
# ℹ abbreviated names: ¹​job_duration_in_current_job_level, ²​job_duration_in_current_person_level, ³​job_duration_in_current_branch, ⁴​marital_status_maried_y_n
# ℹ 13 more variables: education_level <chr>, gpa <dbl>, year_graduated <dbl>, job_duration_from_training <dbl>, branch_rotation <dbl>, job_rotation <dbl>,
#   assign_of_otherposition <dbl>, annual_leave <dbl>, sick_leaves <dbl>, last_achievement_percent <dbl>, achievement_above_100_percent_during3quartal <dbl>,
#   best_performance <dbl>, source <chr>
```

Meanwhile, there are only 20 rows that are either `level_3` (S1), `level_4` (S2), or `level_5` (S3) and have more than 4 `gpa`, but never more than 400 `gpa`. It also indicates that these 20 cases are typo and can be solved by removing the rows, or dividing the `gpa` by 100, return them to range 0 to 4.
```R
df_train %>% filter(gpa > 100 & education_level %in% c("level_3", "level_4", "level_5"))
```
```
# A tibble: 20 × 23
   job_level job_duration_in_current_job_l…¹ person_level job_duration_in_curr…² job_duration_in_curr…³ employee_type gender   age marital_status_marie…⁴ number_of_dependences
   <chr>                               <dbl> <chr>                         <dbl>                  <dbl> <chr>          <dbl> <dbl> <chr>                                  <dbl>
 1 JG04                                2.83  PG04                          1                      0     RM_type_A          2  1982 Y                                          0
 2 JG04                                1.35  PG03                          1.35                   1.19  RM_type_B          2  1987 Y                                          2
 3 JG04                                2.65  PG04                          2.35                   0.707 RM_type_A          1  1975 Y                                          3
 4 JG04                                1.26  PG03                          1.26                   1.19  RM_type_A          2  1988 Y                                          2
 5 JG05                                2.83  PG06                          2.83                   0.707 RM_type_A          1  1964 Y                                          2
 6 JG05                                1.35  PG05                          1.35                   0.707 RM_type_A          1  1983 Y                                          2
 7 JG04                                1.35  PG03                          1.35                   0.707 RM_type_A          2  1988 Y                                          1
 8 JG04                                1.35  PG03                          1.35                   1.66  RM_type_C          2  1989 Y                                          1
 9 JG04                                1.35  PG03                          1.35                   1.19  RM_type_B          2  1982 Y                                          1
10 JG04                                1.41  PG04                          0                      1.19  RM_type_A          2  1990 Y                                          1
11 JG04                                1.47  PG03                          1.47                   1.19  RM_type_A          1  1989 Y                                          1
12 JG04                                1.12  PG03                          1.12                   0.707 RM_type_A          1  1986 Y                                          0
13 JG04                                2.83  PG04                          2.83                   1.78  RM_type_A          2  1969 Y                                          2
14 JG04                                2.83  PG04                          2.83                   2.31  RM_type_A          1  1972 Y                                          3
15 JG04                                2.83  PG04                          1.87                   1.56  RM_type_A          2  1978 Y                                          2
16 JG05                                0.911 PG05                          0.911                  1.56  RM_type_A          2  1979 Y                                          2
17 JG05                                1.47  PG06                          0                      0.707 RM_type_A          2  1975 Y                                          1
18 JG04                                2.83  PG04                          2.45                   1.19  RM_type_A          1  1975 Y                                          3
19 JG04                                2.5   PG04                          1.73                   1.22  RM_type_A          1  1980 Y                                          2
20 JG05                                1.47  PG06                          0                      1.58  RM_type_A          1  1979 Y                                          2
# ℹ abbreviated names: ¹​job_duration_in_current_job_level, ²​job_duration_in_current_person_level, ³​job_duration_in_current_branch, ⁴​marital_status_maried_y_n
# ℹ 13 more variables: education_level <chr>, gpa <dbl>, year_graduated <dbl>, job_duration_from_training <dbl>, branch_rotation <dbl>, job_rotation <dbl>,
#   assign_of_otherposition <dbl>, annual_leave <dbl>, sick_leaves <dbl>, last_achievement_percent <dbl>, achievement_above_100_percent_during3quartal <dbl>,
#   best_performance <dbl>, source <chr>
```

Test data has similar problems to train data, but without missing values.
```R
skim(df_test)
```
```
── Data Summary ────────────────────────
                           Values 
Name                       df_test
Number of rows             6000   
Number of columns          22     
_______________________           
Column type frequency:            
  character                6      
  numeric                  16     
________________________          
Group variables            None   

── Variable type: character ────────────────────────────────────────────────────────────
  skim_variable             n_missing complete_rate min max empty n_unique whitespace
1 job_level                         0             1   4   4     0        4          0
2 person_level                      0             1   4   4     0        7          0
3 employee_type                     0             1   9   9     0        3          0
4 marital_status_maried_y_n         0             1   1   1     0        2          0
5 education_level                   0             1   7   7     0        6          0
6 source                            0             1   4   4     0        1          0

── Variable type: numeric ────────────────────────────────────────────────────────────────────────────────────────────────────────────
   skim_variable                                n_missing complete_rate     mean     sd     p0      p25     p50     p75    p100 hist 
 1 job_duration_in_current_job_level                    0             1    1.43   0.421    0      1.22     1.35    1.41    2.92 ▁▁▇▁▁  
 2 job_duration_in_current_person_level                 0             1    1.35   0.317    0      1.22     1.35    1.39    2.83 ▁▁▇▁▁  
 3 job_duration_in_current_branch                       0             1    1.03   0.413    0      0.707    1.12    1.22    2.74 ▁▇▇▁▁  
 4 gender                                               0             1    1.73   0.444    1      1        2       2       2    ▃▁▁▁▇  
 5 age                                                  0             1 1986.     4.43  1963   1985     1987    1989    1995    ▁▁▂▇▃  
 6 number_of_dependences                                0             1    0.985  0.867    0      0        1       2       4    ▇▇▅▁▁  
 7 gpa                                                  0             1    3.28  14.6      0      2.83     3.08    3.28  381    ▇▁▁▁▁  
 8 year_graduated                                       0             1 2009.     3.98  1984   2008     2010    2012    2020    ▁▁▁▇▂  
 9 job_duration_from_training                           0             1    6.22   4.87     2      4        5       6      35    ▇▁▁▁▁  
10 branch_rotation                                      0             1    3.70   2.34     1      2        3       4      17    ▇▁▁▁▁  
11 job_rotation                                         0             1    3.48   1.80     1      2        3       4      14    ▇▃▁▁▁  
12 assign_of_otherposition                              0             1    1.14   2.50     0      0        0       1      27    ▇▁▁▁▁  
13 annual_leave                                         0             1    3.70   2.65     0      2        3       5      21    ▇▃▁▁▁  
14 sick_leaves                                          0             1    1.17   3.43     0      0        0       1     115    ▇▁▁▁▁  
15 last_achievement_percent                             0             1   72.1   22.7     10.6   56.5     71.1    88.2   130    ▁▅▇▅▂  
16 achievement_above_100_percent_during3quartal         0             1    0.684  1.11     0      0        0       1       3    ▇▁▁▁▂  
```

The histograms in train data and those in test data are similar, meaning that the train data should have represent the condition in test data. The train data, however, have an imbalanced class, as shown in the following section.

### 2.2 Imbalanced Class
`best_performance` is the target variable that will be predicted. The following barplot shows number of employee that were classified as "best performance" (1) and "not best performance" (0).

```R
df_train %>% 
  group_by(bp) %>% 
  count() %>% 
  ggplot(aes(x = as.factor(bp), y = n)) + 
  geom_bar(stat = "identity") +
  xlab("Best Performance") +
  ylab("Number of rows in train dataset")
```

{% include elements/figure.html image="assets/images/BRI - best performance.png" caption="Number of employee based on `best_performance`."%}

The barplot clearly shows an imbalanced dataset, where the number of employee that were not classified as best performance is 4 to 5 times higher than the number of best performance employee. Therefore, resampling may improve model performance.

### 2.3 Association Test

The five categorical variables have classes with few rows, which can be shown in the following barplots.

```R
df_train_char = df_train %>% select(where(is.character), best_performance)
df_train_char %>%
  pivot_longer(-best_performance) %>%
  group_by(best_performance, name, value) %>%
  mutate(
    total = n(),
    best_performance = ifelse(best_performance == 1, "Yes", "No"),
    best_performance = factor(best_performance, levels = c("No", "Yes"))
  ) %>%
  ggplot(aes(as.factor(value), total, fill = best_performance)) +
  geom_bar(stat = "identity", position = position_dodge()) +
  facet_wrap(vars(name), scale = "free_x", nrow = 3) +
  xlab("") +
  ylab("Number of employee") +
  guides(fill=guide_legend(title="Best\nPerformance"))
```

{% include elements/figure.html image="assets/images/BRI - barplot of character variables.png" caption="Number of employee based on the five character variables."%}

Because of these few rows, the expected frequencies from contingency table are less than 5 for some of the classes. Therefore, Fisher Exact Test will be used for association test between the categorical variables.

```R
# pair two categorical variables
catg_pair_list <- 
  df_train %>%
  select(is.character, best_performance, -source) %>%
  colnames() %>%
  combn(2, simplify = F)

# chi-square test for each pair
fisher_res <- data.frame() 
for (i in seq_along(catg_pair_list)) {
  v1 <- catg_pair_list[[i]][1] # getting the 1st variable in the i-th pair, a string
  v2 <- catg_pair_list[[i]][2] # getting the 2nd variable in the i-th pair, a string 
  
  # extract from df_train
  p1 <- df_train[[v1]]
  p2 <- df_train[[v2]]
  
  # chi-squared test
  fisher <- fisher.test(p1,p2,simulate.p.value = T)
  pvalue <- c("V1"=v1, "V2"=v2, "p-value"=round(fisher$p.value,5)) # variable 1, variable 2, p-value
  fisher_res <- rbind(fisher_res,pvalue) # update the empty dataframe
}

# modify the fisher_res
fisher_res <- 
  fisher_res %>% 
  rename(V1 = X.job_level., 
         V2 = X.person_level.,
         pvalue = X.5e.04.) %>% 
  mutate(pvalue = as.numeric(pvalue)) # change pvalue to numeric

fisher_res %>% filter(V2 == "best_performance") %>% arrange(pvalue)
```
```
                         V1               V2  pvalue
1              person_level best_performance 0.01349
2             employee_type best_performance 0.04498
3 marital_status_maried_y_n best_performance 0.18732
4                 job_level best_performance 0.37631
5           education_level best_performance 0.58921
```

Based on the chi-square test, only `person_level` and `employee_type` that are significantly associated with `best_performance`, that is, with a note that `person_level` and `employee_type` are actually associated. The `person_level` should be the only one used because of its lower p-value.

```R
fisher_res %>% 
  filter(V2 != "best_performance") %>%
  filter(V1 == "person_level" | V2 == "person_level") %>%
  arrange(pvalue)
```
```
            V1                        V2 pvalue
1    job_level              person_level  5e-04
2 person_level             employee_type  5e-04
3 person_level marital_status_maried_y_n  5e-04
4 person_level           education_level  5e-04
```

### 2.4 Correlation test
The correlation between numerical variables are measured using spearman correlation. The following are the correlation plot.

```R
df_train_temp <- 
  df_train %>% 
  rename(
    "jdic_job_lv" = "job_duration_in_current_job_level",
    "jdic_person_lv" = "job_duration_in_current_person_level",
    "jdic_branch" = "job_duration_in_current_branch",
    "dependences" = "number_of_dependences",
    "jdf_training" = "job_duration_from_training",
    "married" = "marital_status_maried_y_n",
    "best_performance" = "best_performance",
    "annual_leave" = "annual_leave",
    "Last_ach_perc" = "last_achievement_percent",
    "Ach_above_100perc" = "achievement_above_100_percent_during3quartal"
  )

df_train_temp %>%
  select(where(is.numeric)) %>%
  cor() %>%
  corrplot::corrplot()
```

{% include elements/figure.html image="assets/images/BRI - correlation.png" caption="Correlation between numerical variables."%}

Based on the plot, there are some variables that are highly correlated, positively or negatively, with other variables. Removing some of these highly correlated variables may improve modeling efficiency and/or avoid multicollinearity.

## 3. Random Forest
The results from Explanatory Data Analysis suggest a few approach that can be used. 
1. There is no missing data, but `gpa` should be removed.
1. The train data is imbalanced so a resampling technique such as SMOTE may improve model accuracy.
1. Among the categorical variables, only `person_level` should be used because of its statistically significant with `best_performance`.
1. 

Recommendation:
1. try different model
1. try encoding variable such as gender
1. 
