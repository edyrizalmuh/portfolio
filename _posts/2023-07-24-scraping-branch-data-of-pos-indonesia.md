---
title: Scraping branch data of POS Indonesia
tags: [Data Scraping, R, Rvest]
style: fill
color: warning
description: Using Rvest to scrape branch data from POS Indonesia's official website.
# external_url: https://github.com/edyrizalmuh/scraping-courier-data/tree/main
---

{% include elements/figure.html image="assets/images/Pos_indo.png" %}

* TOC
{:toc .floating-toc}

## 1. Introduction
POS Indonesia is an Indonesian state-owned corporation specializing in postal and courier services. Being one of the oldest corporations in Indonesia, POS Indonesia has surely encountered various challenges, including the growth of competitors within the courier-related industry over the past decade. However, one of POS Indonesia's key strengths lies in its extensive reach, even in remote areas, which provides a clear advantage over newer competitors, particularly in this industry. Analyzing the true extent of this advantage becomes an intriguing subject in itself.

This blog post is part of a larger project focused on analyzing the courier industry in Indonesia. Specifically, the article will illustrate the scraping process used to collect data on POS Indonesia's branches throughout the country, utilizing information from [the official website](https://www.posindonesia.co.id/). The main goal is to compile a comprehensive list of all POS Indonesia branches in Indonesia, enabling a comparison with other competitors in the industry.

## 2. Scraping Permission
The first step of the scraping process involves checking the permissions for web scraping on POS Indonesia's [official website](https://www.posindonesia.co.id/), which can be found in the [robots.txt file](https://www.posindonesia.co.id/robots.txt). The robots.txt file shows these lines.

```
User-agent: *
Disallow:
```

These lines confirm that all robots have unrestricted access to visit all pages on the website. This permission allows us to scrape data, including information about the branches. Of course, the scraping process should be done responsibly, ensuring that it is not overly aggressive and will not adversely impact other users. 


## 3. Cities and Regencies
The branches in [the official website](https://www.posindonesia.co.id/) are searchable only by city or regency. Therefore, it is important to scrape all names of cities and regencies in Indonesia before scraping the branches, which can be obtained from [this wikipedia page](https://id.wikipedia.org/wiki/Daftar_kabupaten_dan_kota_di_Indonesia). Of course, the robots.txt permit scraping from such URL.

{% include elements/figure.html image="assets/images/wikipedia - cities and regencies in Indonesia.png" caption="Example of the wikipedia page. Each province has its own table, listing its cities and regencies." %}

### 3.1 Scraping Names of Cities and Regencies
Since each province has its own table, the first step is to extract all of these tables using `rvest`.

```R
# install.packages("rvest")
# install.packages("janitor")
library(rvest)
library(tidyverse) # for data wrangling
library(janitor) # for data cleaning
link = "https://id.wikipedia.org/wiki/Daftar_kabupaten_dan_kota_di_Indonesia"
page = read_html(link)
table_all = 
  page %>% 
  html_elements("table.sortable") %>%
  html_table(dec = ",") # the wiki page is in Indonesian, which uses , (comma) as decimal
```
The syntax above returns `table_all`, which is a list with 39 elements.

{% include elements/figure.html image="assets/images/table_all.png" caption="The first 8 elements of table_all"%}

The first element of `table_all` is a table containing the names of all provinces in Indonesia. The remaining 38 elements are tables for each province, containing the names of its cities and regencies. Therefore, the first element should be separated from the rest 38 elements.

```R
table_prov = table_all[[1]] %>% clean_names() # clean column names to avoid error
prov_names = head(table_prov$provinsi, nrow(table_prov)-1) 
table_city = table_all[-1]
```

### 3.2 Cleaning City and Regency Data

The tables of each province seem to have different dimensions. 

```R
num_of_col = lapply(1:length(table_city), function(x) ncol(table_city[[x]])) %>% as.numeric()
num_of_col
```

```
10 11 10 10 10 10 10 20 10 10 19 10 10 10 10 10 10 10 10 10
10 10 10 10 10 10 10 10 10 10 10 10 10 10 10 10 12 10
```

The output shows that most tables have 10 columns, but some tables have more than 10 columns. The cities/regencies with more than 10 columns are the cities/regencies with these indexes:
```R
idx_messy = which(num_of_col > 10)
idx_messy
```

```
2  8 11 37
```

These indexes are based on `table_city`, corresponding to indexes `2, 9, 12` and `38` in `table_all`. Knowing indexes from these two objects is important for cleaning later, as we do not want to accidentally change the content of these tables.

These column names have to be compared and standardized. The following syntax shows the unique column names and how many province(s) use the names in its table. 

```R
# Extract unique column names from all tables
all_column_names <- unique(unlist(lapply(table_city, colnames)))

# Create an empty matrix with rows representing tables and columns representing unique column names
result_matrix <- matrix(0, nrow = length(table_city), ncol = length(all_column_names))
colnames(result_matrix) <- all_column_names

# Fill the matrix with 1 if the column name is present in the respective table
for (i in 1:length(table_city)) {
  col_indices <- match(colnames(table_city[[i]]), all_column_names)
  result_matrix[i, col_indices] <- 1
}

# count how many cities/regencies have the corresponding column names
result_tibble =
  result_matrix %>%
  t() %>%
  as_tibble() %>%
  mutate(col_names = rownames(t(result_matrix))) %>%
  relocate(col_names)
# colnames(result_tibble) = colnames(result_tibble) %>% str_replace_all("V", "")
result_tibble %>% 
  rowwise() %>%
  mutate(total = sum(c_across(where(is.numeric)))) %>%
  select(col_names, total) %>% 
  arrange(col_names)
```

```
# A tibble: 95 × 2
# Rowwise: 
   col_names                         total
   <chr>                             <dbl>
 1 ""                                    3
 2 "Bupati"                              2
 3 "Bupati/Walikota"                     1
 4 "Bupati/wali kota"                   33
 5 "Bupati/wali kota administrasi"       1
 6 "Distrik"                             6
 7 "Gampong"                             1
 8 "IPM\n(2020)"                         1
 9 "Ibu kota"                           34
10 "Ibu kota[11]"                        1
11 "Jumlah Penduduk (2024)[13]"          1
12 "Jumlah penduduk"                     3
13 "Jumlah penduduk (2015)[16]"          1
14 "Jumlah penduduk (2017)[15]"          1
15 "Jumlah penduduk (2017)[34]"          1
16 "Jumlah penduduk (2017)[35]"          1
17 "Jumlah penduduk (2017)[36]"          1
18 "Jumlah penduduk (2017)[37]"          1
19 "Jumlah penduduk (2017)[38]"          1
20 "Jumlah penduduk (2017)[39]"          1
21 "Jumlah penduduk (2018)[27]"          1
22 "Jumlah penduduk (2020)"              7
23 "Jumlah penduduk (2020)[12]"          1
24 "Jumlah penduduk (2020)[18]"          1
25 "Jumlah penduduk (2020)[20]"          1
26 "Jumlah penduduk (2020)[29]"          1
27 "Jumlah penduduk (2020)[30]"          1
28 "Jumlah penduduk (2020)[31]"          1
29 "Jumlah penduduk (2020)[32]"          1
30 "Jumlah penduduk (2020)[33]"          1
31 "Jumlah penduduk (2020)[40]"          1
32 "Jumlah penduduk (2020)[5]"           1
33 "Jumlah penduduk (2021)[10]"          1
34 "Jumlah penduduk (2021)[9]"           1
35 "Jumlah penduduk (2021)[9][2]"        1
36 "Jumlah penduduk (2023)[23]"          1
37 "Jumlah penduduk (2023)[7]"           1
38 "Jumlah penduduk (Juni 2022)[22]"     1
39 "Jumlah penduduk (SP 2020)[25]"       1
40 "Kabupaten"                           1
41 "Kabupaten/Kota"                      1
42 "Kabupaten/kota"                     34
43 "Kabupaten/kota administrasi[14]"     1
44 "Kapanewon/kemantren"                 1
45 "Kecamatan"                          30
46 "Kelurahan"                           1
47 "Kelurahan/desa"                     28
48 "Kelurahan/kalurahan"                 1
49 "Kelurahan/kampung"                   6
50 "Lambang"                            36
51 "Lambang alt"                         1
52 "Luas wilayah (km2)[12]"              1
53 "Luas wilayah (km2)[13]"              1
54 "Luas wilayah (km2)[17]"              1
55 "Luas wilayah (km2)[21]"              1
56 "Luas wilayah (km2)[23]"              1
57 "Luas wilayah (km2)[24]"              1
58 "Luas wilayah (km2)[28]"              1
59 "Luas wilayah (km2)[2]"               1
60 "Luas wilayah (km2)[30]"              1
61 "Luas wilayah (km2)[31]"              1
62 "Luas wilayah (km2)[33]"              1
63 "Luas wilayah (km2)[34]"              1
64 "Luas wilayah (km2)[35]"              1
65 "Luas wilayah (km2)[36]"              1
66 "Luas wilayah (km2)[37]"              1
67 "Luas wilayah (km2)[38]"              1
68 "Luas wilayah (km2)[39]"              1
69 "Luas wilayah (km2)[3]"               1
70 "Luas wilayah (km2)[4]"               1
71 "Luas wilayah (km2)[6]"               1
72 "Luas wilayah (km²)"                  2
73 "Luas wilayah (km²)[15]"              1
74 "Luas wilayah (km²)[19]"              1
75 "Luas wilayah (km²)[20]"              1
76 "Luas wilayah (km²)[26]"              1
77 "Luas wilayah (km²)[41]"              1
78 "Luas wilayah (km²)[50]"              1
79 "Luas wilayah (km²)[9]"               7
80 "Luas wilayah(km²)[15]"               1
81 "No."                                37
82 "Peta lokasi"                        37
83 "Pusat pemerintahan"                  2
84 "Ref."                                1
85 "X1"                                  1
86 "X10"                                 1
87 "X2"                                  1
88 "X3"                                  1
89 "X4"                                  1
90 "X5"                                  1
91 "X6"                                  1
92 "X7"                                  1
93 "X8"                                  1
94 "X9"                                  1
95 "luas wilayah (km2)[10]"              1
```

Based on the output, there are some common column names such as `No.` and `Peta lokasi`, but even these columns only sums up to 37 tables; there are 38 provinces. It means that there is at least one province table with different or simply missing these columns, especially considering that there are three province tables with blank column(s).

There are also columns with different names, but essentially means the same thing. `Bupati`, `Bupati/Walikota`, `Bupati/wali kota`, and `Bupati/wali kota administrasi` means the head of the government. `Distrik/Kapanewon/kemantren` has the same administrative level as `Kecamatan`, while `Gampong/kalurahan` has the same administrative level as `Kelurahan/desa/kampung`. 

Some column names, such as `Ibu Kota`, `Jumlah Penduduk`, and `Luas Wilayah`, have various values because of numbers in square brackets, which are reference numbering in the original website page. It means that the numbers in one city/regency may be ontained from different sources compared to the numbers from another city/regency. This is obviously less than ideal. Another important thing to consider is how these columns, specifically `Jumlah Penduduk` were obtained from different years. Both problems may be solved by obtaining the most recent data from a single organization such as Badan Pusat Statistik (lit. Centra Agency of Statistics). But, this cleaning process will be postponed until such data are needed for the coming larger project, since the data needed for now is the city and regency names. For now, columns `Jumlah Penduduk` and `Luas Wilayah` will be marked with `_diff_sources_and_years` names.

Other than columns with missing or similar names, there are also some column names that are unique to certain table. Column names such as `Ref.` (Reference), `IPM`, and `Lambang alt` are only found in a table. These column will not be used (including the `Lambang` column which presents in more tables). 

There are also undefined column names that start with `X`. This is found in city number 5.
```R
table_city[[5]]
```

```
# A tibble: 12 × 10
   X1    X2                             X3            X4                        X5                    X6                        X7        X8             X9        X10         
   <chr> <chr>                          <chr>         <chr>                     <chr>                 <chr>                     <chr>     <chr>          <chr>     <chr>       
 1 No.   Kabupaten/kota                 Ibu kota      Bupati/wali kota          Luas wilayah (km2)[8] Jumlah penduduk (2021)[8] Kecamatan Kelurahan/desa "Lambang" "Peta lokas…
 2 1     Kabupaten Batanghari           Muara Bulian  Muhammad Fadhil Arief     5.804,83              301.700                   8         14/110         ""        ""          
 3 2     Kabupaten Bungo                Muara Bungo   Mashuri                   4.659,00              355.927                   17        12/141         ""        ""          
 4 3     Kabupaten Kerinci              Siulak        Adirozal                  3.807,28              254.241                   16        2/285          ""        ""          
 5 4     Kabupaten Merangin             Bangko        Mashuri                   7.668,61              357.315                   24        10/205         ""        ""          
 6 5     Kabupaten Muaro Jambi          Sengeti       Bachyuni Deliansyah (Pj.) 5.246,00              406.799                   11        5/150          ""        ""          
 7 6     Kabupaten Sarolangun           Sarolangun    Henrizal (Pj.)            6.174,00              279.532                   10        9/149          ""        ""          
 8 7     Kabupaten Tanjung Jabung Barat Kuala Tungkal Anwar Sadat               5.009,82              323.466                   13        20/114         ""        ""          
 9 8     Kabupaten Tanjung Jabung Timur Muara Sabak   Romi Hariyanto            5.445,00              232.048                   11        20/73          ""        ""          
10 9     Kabupaten Tebo                 Muara Tebo    Aspan (Pj.)               6.461,00              335.228                   12        5/107          ""        ""          
11 10    Kota Jambi                     -             Syarif Fasha              205,38                621.365                   11        62/-           ""        ""          
12 11    Kota Sungai Penuh              -             Ahmadi Zubir              391,50                97.190                    8         4/65           ""        "" 
```
This table must be the only table with such names, since all `X1` to `X10` present in this table. As shown before in the list of column names, the `X` column names only present in one table.

As seen in the table, the header of the table becomes the first row. This can be fixed using the following syntax.
```R
table_city[[5]] = table_all[[6]][-1,]
```

As for the tables with more columns, i.e. the tables with these indexes:
```R
idx_messy
```
```
2  8 11 37
```

The columns have to be inspected one by one.
```R
table_city[[2]]
```
```
# A tibble: 33 × 11
     No. `Kabupaten/kota`            `Ibu kota` `Bupati/wali kota` Luas wilayah (km2)[3…¹ Jumlah penduduk (202…² `IPM\n(2020)` Kecamatan `Kelurahan/desa` Lambang `Peta lokasi`
   <int> <chr>                       <chr>      <chr>              <chr>                  <chr>                          <dbl>     <int> <chr>            <lgl>   <lgl>        
 1     1 Kabupaten Asahan            Kisaran    Surya              3.737,83               791.174                         70.5        25 27/177           NA      NA           
 2     2 Kabupaten Batu Bara         Limapuluh  Zahir              888,14                 443.816                         68.6        12 10/141           NA      NA           
 3     3 Kabupaten Dairi             Sidikalang Eddy Keleng Ate B… 2.083,60               319.583                         71.8        15 8/161            NA      NA           
 4     4 Kabupaten Deli Serdang      Lubuk Pak… Ashari Tambunan    2.581,23               1.991.108                       75.5        22 14/380           NA      NA           
 5     5 Kabupaten Humbang Hasundut… Dolok San… Dosmar Banjarnahor 2.351,51               204.377                         69.4        10 1/153            NA      NA           
# ℹ 28 more rows
# ℹ abbreviated names: ¹​`Luas wilayah (km2)[3]`, ²​`Jumlah penduduk (2020)`
# ℹ Use `print(n = ...)` to see more rows
```

`IPM` is the only column that does not present in other tables. This table can be easily cleaned by dropping this `IPM` column. Meanwhile, the 8-th table have 20 columns:

```R
table_city[[8]]
```
```
# A tibble: 15 × 20
   No.   `Kabupaten/kota`  `Ibu kota[11]` `Bupati/wali kota` Luas wilayah (km2)[1…¹ Jumlah penduduk (202…² Kecamatan `Kelurahan/desa` Lambang `Peta lokasi`    `` ``    ``   ``           ``       ``         `` ``    ``    ``   
   <chr> <chr>             <chr>          <chr>              <chr>                  <chr>                  <chr>     <chr>            <chr>   <chr>         <int> <chr> <chr><chr>        <chr>    <chr>   <int> <chr> <lgl> <lgl>
 1 No.   Kabupaten/kota    Ibu kota[11]   Bupati/wali kota   Luas wilayah (km2)[12] Jumlah penduduk (2020… Kecamatan Kelurahan/desa   "Lamba… "Peta lokasi"     1 Kabu… Liwa Nukman (Pj.) 2.142,78 302.139    15 5/131 NA    NA   
 2 2     Kabupaten Lampun… Kalianda       Nanang Ermanto     2.219,01               1.064.301              17        4/256            ""      ""               NA NA    NA   NA           NA       NA         NA NA    NA    NA   
 3 3     Kabupaten Lampun… Gunung Sugih   Musa Ahmad         4.544,68               1.460.045              28        10/301           ""      ""               NA NA    NA   NA           NA       NA         NA NA    NA    NA   
 4 4     Kabupaten Lampun… Sukadana       M. Dawam Rahardjo  3.864,69               1.110.340              24        -/264            ""      ""               NA NA    NA   NA           NA       NA         NA NA    NA    NA   
 5 5     Kabupaten Lampun… Kotabumi       Budi Utomo         2.529,54               633 099                23        15/232           ""      ""               NA NA    NA   NA           NA       NA         NA NA    NA    NA   
 # ℹ 10 more rows
```

The only row in the 11-th to the 20-th columns is supposed to be the first row in the first 10 columns. The syntax below will fix this issue.
```R
temp1 = table_all[[9]][1,11:20]
temp2 = table_all[[9]][-1,1:10]
colnames(temp1) = colnames(temp2)
temp1$No. = as.character(temp1$No.)
temp1$`Jumlah penduduk (2020)[12]` = as.character(temp1$`Jumlah penduduk (2020)[12]`)
temp1$Kecamatan = as.character(temp1$Kecamatan)
table_city[[8]] = bind_rows(temp2, temp1)
```

```R
table_city[[11]]
```
```
# A tibble: 6 × 19
  No.   Kabupaten/kota admini…¹ `Pusat pemerintahan` Bupati/wali kota adm…² Luas wilayah(km²)[15…³ Jumlah penduduk (201…⁴ Kecamatan Kelurahan Lambang `Peta lokasi` ``    ``   ``         `` ``        ``    `` ``    ``   
  <chr> <chr>                   <chr>                <chr>                  <chr>                  <chr>                  <chr>     <chr>     <chr>   <chr>         <chr> <chr> <chr>   <dbl> <chr>  <int> <int> <lgl> <lgl>
1 No.   Kabupaten/kota adminis… Pusat pemerintahan   Bupati/wali kota admi… Luas wilayah(km²)[15]  1                      Kecamatan Kelurahan "Lamba… "Peta lokasi" Kabu… Pula… Junaedi  10.2 27.749     2     6 NA    NA   
2 2     Kota Administrasi Jaka… Kembangan            Yani Wahyu Purwoko     124,44                 2.434.511              8         56        ""      ""            NA    NA    NA       NA   NA        NA    NA NA    NA   
3 3     Kota Administrasi Jaka… Menteng              Dhany Sukma            52,38                  1.056.896              8         44        ""      ""            NA    NA    NA       NA   NA        NA    NA NA    NA   
4 4     Kota Administrasi Jaka… Kebayoran Baru       Munjirin               154,32                 2.226.812              10        65        ""      ""            NA    NA    NA       NA   NA        NA    NA NA    NA   
5 5     Kota Administrasi Jaka… Cakung               Muhammad Anwar         182,70                 3.037.139              10        65        ""      ""            NA    NA    NA       NA   NA        NA    NA NA    NA   
6 6     Kota Administrasi Jaka… Koja                 Ali Maulana Hakim      139,99                 1.778.981              6         31        ""      ""            NA    NA    NA       NA   NA        NA    NA NA    NA   
# ℹ abbreviated names: ¹​`Kabupaten/kota administrasi[14]`, ²​`Bupati/wali kota administrasi`, ³​`Luas wilayah(km²)[15]`, ⁴​`Jumlah penduduk (2015)[16]`
```

The 11-th table has the same problem as the 8-th table, therefore, similar approach will fix the issue.
```R
temp1 = table_all[[12]][1,11:19]
temp2 = table_all[[12]][-1,1:10] # 12th table in table_all, the 1st table is the province table

colnames(temp1) = colnames(table_all[[12]][2:10])
temp1 = 
  temp1 %>% 
  mutate('No.' = '1') %>%
  relocate('No.')

temp1$No. = as.character(temp1$No.)
temp1$`Jumlah penduduk (2015)[16]` = as.character(temp1$`Jumlah penduduk (2015)[16]`)
temp1$Kecamatan = as.character(temp1$Kecamatan)
temp1$Kelurahan = as.character(temp1$Kelurahan) 
temp1$`Luas wilayah(km²)[15]` = as.character(temp1$`Luas wilayah(km²)[15]`)
table_city[[11]] = bind_rows(temp1, temp2)
```

```R
table_city[[37]]
```
```
# A tibble: 8 × 12
    No. `Kabupaten/kota`             `Ibu kota`         `Bupati/wali kota` `Luas wilayah (km²)` `Jumlah penduduk` Distrik `Kelurahan/kampung` Lambang `Peta lokasi` Ref.  ``   
  <int> <chr>                        <chr>              <chr>              <chr>                <chr>               <int> <chr>               <lgl>   <lgl>         <chr> <lgl>
1     1 Kabupaten Jayawijaya         Wamena             John Richard Banua 13.925,31            277.923                40 4/328               NA      NA            [42]  NA   
2     2 Kabupaten Lanny Jaya         Tiom               Doren Wakerkwa (P… 6.077,4              201.461                39 1/354               NA      NA            [43]  NA   
3     3 Kabupaten Mamberamo Tengah   Kobakma            Yonas Kenelak (Pl… 3.743,64             51.719                  5 -/59                NA      NA            [44]  NA   
4     4 Kabupaten Nduga              Kenyam             Edison Gwijangge … 12.941,00            109.630                32 -/248               NA      NA            [45]  NA   
5     5 Kabupaten Pegunungan Bintang Oksibil            Spei Yan Birdana   15.683,00            78.466                 34 -/277               NA      NA            [46]  NA   
6     6 Kabupaten Tolikara           Karubaga           Marthen Kogoya (P… 14.564,00            244.345                46 4/541               NA      NA            [47]  NA   
7     7 Kabupaten Yalimo             Elelim             Nahor Nekwek       4.330,29             103.387                 5 -/300               NA      NA            [48]  NA   
8     8 Kabupaten Yahukimo           Sumohai (de jure)… Didimus Yahuli     17.152,00            350.880                51 1/510               NA      NA            [49]  NA   
```

The 37-th table have two more columns: `Ref.` and a blank column. Dropping these columns should fix the issue.
```R
table_city[[37]] = table_all[[38]][,1:10] # the 11-th column is an empty column
```

After inspecting the tables with excess number of columns, the next process is to clean the column names.
```R
# function for cleaning column names
clean_name_table = function(x){
  temp =
    x %>%
    select(-any_of(c("No.", "Ref.", "IPM\n(2020)", "Lambang", "Lambang alt", "Peta lokasi")))
    
  names(temp) =
    names(temp) %>%
    str_replace("Kabupaten(/Kota|/kota)?( administrasi\\[14])?", "kabupaten_kota") %>%
    str_replace("Ibu kota(\\[\\d+\\])?|Pusat pemerintahan", "ibu_kota") %>%
    str_replace("Bupati(/Walikota|/wali kota)?( administrasi)?", "bupati_wali_kota") %>%
    str_replace("[Ll]uas [Ww]ilayah(?:\\s*\\([^\\)]+\\))?(?:\\[\\d+\\])?", "luas_wilayah_diff_sources_and_years") %>%
    str_replace("Jumlah [pP]enduduk(?: \\([^\\)]+\\))?(?:\\[[^\\]]+\\])*(?:\\[[^\\]]+\\])?", "jumlah_penduduk_diff_sources_and_years") %>%
    str_replace("Ibu kota(\\[\\d+\\])?", "ibu_kota") %>%
    str_replace("Kecamatan|Distrik|Kapanewon/kemantren", "kecamatan") %>%
    str_replace("Gampong|Kelurahan(?:\\/desa|\\/kalurahan|\\/kampung)?", "kelurahan_desa")
  
  temp = 
    temp %>%
    mutate(
      kabupaten_kota = as.character(kabupaten_kota),
      ibu_kota = as.character(ibu_kota),
      bupati_wali_kota = as.character(bupati_wali_kota),
      luas_wilayah_diff_sources_and_years = str_replace_all(luas_wilayah_diff_sources_and_years, "\\.", ""),
      luas_wilayah_diff_sources_and_years = str_replace_all(luas_wilayah_diff_sources_and_years, ",", "."),
      luas_wilayah_diff_sources_and_years = str_replace_all(luas_wilayah_diff_sources_and_years, " ", ""),
      luas_wilayah_diff_sources_and_years = as.numeric(luas_wilayah_diff_sources_and_years),
      jumlah_penduduk_diff_sources_and_years = str_replace_all(jumlah_penduduk_diff_sources_and_years, "\\.", ""),
      jumlah_penduduk_diff_sources_and_years = str_replace_all(jumlah_penduduk_diff_sources_and_years, " ", ""),
      jumlah_penduduk_diff_sources_and_years = str_replace_all(jumlah_penduduk_diff_sources_and_years, " ", ""),
      jumlah_penduduk_diff_sources_and_years = as.numeric(jumlah_penduduk_diff_sources_and_years),
      ibu_kota = as.character(ibu_kota),
      kecamatan = as.integer(kecamatan),
      kelurahan_desa = as.character(kelurahan_desa)
    )
  return(temp)
}

table_city_clean_colnames = tibble()
prov_names = head(table_prov$provinsi, nrow(table_prov)-1)
for (i in 1:length(table_city)){
  temp = 
    clean_name_table(table_city[[i]]) %>%
    mutate(provinsi = prov_names[i]) %>%
    relocate(provinsi)
  table_city_clean_colnames = bind_rows(table_city_clean_colnames, temp)
  rm(temp)
}
table_city_clean_colnames
```
```
# A tibble: 514 × 8
   provinsi kabupaten_kota            ibu_kota    bupati_wali_kota       luas_wilayah_diff_sources_and_years jumlah_penduduk_diff_sources_and_years kecamatan kelurahan_desa
   <chr>    <chr>                     <chr>       <chr>                                                <dbl>                                  <dbl>     <int> <chr>         
 1 Aceh     Kabupaten Aceh Barat      Meulaboh    Mahdi Effendi (Pj.)                                  2783.                                 198858        12 322           
 2 Aceh     Kabupaten Aceh Barat Daya Blangpidie  Darmansah (Pj.)                                      1882.                                 153515         9 152           
 3 Aceh     Kabupaten Aceh Besar      Kota Jantho Muhammad Iswanto (Pj.)                               2891.                                 422241        23 604           
 4 Aceh     Kabupaten Aceh Jaya       Calang      Nurdin (Pj.)                                         3872.                                  96049         9 172           
 5 Aceh     Kabupaten Aceh Selatan    Tapak Tuan  Tgk. Amran                                           4175.                                 234169        18 260           
 6 Aceh     Kabupaten Aceh Singkil    Singkil     Marthunis (Pj.)                                      1853.                                 129674        11 116           
 7 Aceh     Kabupaten Aceh Tamiang    Karang Baru Meurah Budiman (Pj.)                                 2188.                                 301800        12 216           
 8 Aceh     Kabupaten Aceh Tengah     Takengon    Teuku Mirzuan (Pj.)                                  4468.                                 219744        14 295           
 9 Aceh     Kabupaten Aceh Tenggara   Kutacane    Syakir (Pj.)                                         4179.                                 227921        16 385           
10 Aceh     Kabupaten Aceh Timur      Idi Rayeuk  Mahyuddin (Pj.)                                      5409.                                 434929        24 513           
# ℹ 504 more rows
# ℹ Use `print(n = ...)` to see more rows
```

Since POS's website cannot search branches if the word `Kabupaten` or `Kota` is included, the final step for this section is to separate `Kabupaten` and `Kota` from the city and regency names.
```R
table_city_tidy =
  table_city_clean_colnames %>%
  mutate(
    kabkota = str_extract(kabupaten_kota, "Kabupaten|Kota"), # is it a regency or a city
    kabkota = ifelse(is.na(kabkota), "Kabupaten", kabkota),
    kabupaten_kota = str_remove(kabupaten_kota, "Kabupaten Administrasi |Kabupaten |Kota Administrasi |Kota "),
    # cities often have - in `ibu_kota`, meaning that the `ibu_kota` should have the same name as in `kabupaten_kota`
    ibu_kota = ifelse(ibu_kota == "-", kabupaten_kota, ibu_kota),
    kelurahan = str_extract(kelurahan_desa, "[0-9	]+(?=/)"),
    kelurahan = ifelse(is.na(kelurahan), 0, kelurahan) %>% as.integer(),
    desa = str_extract(kelurahan_desa, "(?<=/)[0-9	]+"),
    desa = ifelse(is.na(desa), 0, desa) %>% as.integer(),
    total_kelurahan_desa = kelurahan+desa,
    total_kelurahan_desa = 
      ifelse(total_kelurahan_desa==0, kelurahan_desa, total_kelurahan_desa) %>%
      as.integer()
  ) %>%
  select(-c("kelurahan", "desa", "kelurahan_desa")) %>%
  relocate(provinsi, kabupaten_kota, kabkota) 
table_city_tidy
```
```
# A tibble: 514 × 9
   provinsi kabupaten_kota  kabkota   ibu_kota    bupati_wali_kota       luas_wilayah_diff_sources_and_years jumlah_penduduk_diff_sources_and_…¹ kecamatan total_kelurahan_desa
   <chr>    <chr>           <chr>     <chr>       <chr>                                                <dbl>                               <dbl>     <int>                <int>
 1 Aceh     Aceh Barat      Kabupaten Meulaboh    Mahdi Effendi (Pj.)                                  2783.                              198858        12                  322
 2 Aceh     Aceh Barat Daya Kabupaten Blangpidie  Darmansah (Pj.)                                      1882.                              153515         9                  152
 3 Aceh     Aceh Besar      Kabupaten Kota Jantho Muhammad Iswanto (Pj.)                               2891.                              422241        23                  604
 4 Aceh     Aceh Jaya       Kabupaten Calang      Nurdin (Pj.)                                         3872.                               96049         9                  172
 5 Aceh     Aceh Selatan    Kabupaten Tapak Tuan  Tgk. Amran                                           4175.                              234169        18                  260
 6 Aceh     Aceh Singkil    Kabupaten Singkil     Marthunis (Pj.)                                      1853.                              129674        11                  116
 7 Aceh     Aceh Tamiang    Kabupaten Karang Baru Meurah Budiman (Pj.)                                 2188.                              301800        12                  216
 8 Aceh     Aceh Tengah     Kabupaten Takengon    Teuku Mirzuan (Pj.)                                  4468.                              219744        14                  295
 9 Aceh     Aceh Tenggara   Kabupaten Kutacane    Syakir (Pj.)                                         4179.                              227921        16                  385
10 Aceh     Aceh Timur      Kabupaten Idi Rayeuk  Mahyuddin (Pj.)                                      5409.                              434929        24                  513
# ℹ 504 more rows
# ℹ abbreviated name: ¹​jumlah_penduduk_diff_sources_and_years
# ℹ Use `print(n = ...)` to see more rows
```

## 4. Scraping the Branches' Names and Addresses
