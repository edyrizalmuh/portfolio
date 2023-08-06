---
title: Scraping Google Maps
tags: [Data Scraping, Data Cleaning, R, Rvest, Rselenium]
style: fill
color: danger
description: This post describes the process of scraping google maps data. The goal is to scrape branch data of top courier companies in Indonesia, including POS Indonesia. The scraped data can be compared to the data from the official website, detailed in the previous post.
---

* TOC
{:toc .floating-toc}

## 1. Introduction
The [previous post](scraping-pos-indonesia-branch-names-and-addresses.html) describes the scraping process of POS's official website. This post will scrape similar data, but from another source: google maps. The data from google maps can be compared to the data from the official website. This comparison can help identifying accessible and inaccessible branches in terms of helping customer finding the branches.  maps is chosen because it is one of the most complete source and one of the most commonly used app/website for location data.

## 2. Data Scraping
The scraping process is started by loading the required packages and google maps' url.
```R
library(RSelenium)
library(netstat)
library(tidyverse)
library(foreach)
library(doParallel)
main_url = "http://maps.google.com/"
```

It is also necessary to load all keywords that will be inputed to google maps. In this case, the keyword is "pos" followed by city/regency names, such as, "pos aceh barat". The city/regency names were extracted in [previous post](scraping-pos-indonesia-branch-names-and-addresses#3-cities-and-regencies).
```R
business_name = "pos"
daftar_kota = 
  read_csv("data/clean/table_city_tidy.csv") %>% 
  pull(kabupaten_kota) %>% 
  str_to_lower()
paste(business_name, head(daftar_kota)) # examples
```
```
[1] "pos aceh barat"      "pos aceh barat daya"
[3] "pos aceh besar"      "pos aceh jaya"      
[5] "pos aceh selatan"    "pos aceh singkil"
```

To track which city/regency has been and has not been scraped, data from each city/regency can be stored in different files. In this case, the data are stored csv format.
```R
raw_dir = paste0("data/raw/",business_name,"/")
kabkota_done = list.files(raw_dir) %>% str_extract(".+(?=.csv)")
kabkota_remaining = daftar_kota[!daftar_kota %in% kabkota_done]
```

As seen in this example output, some cities/regencies have similar names. As the result, some urls may appear multiple times. For example, here, "kantorpos kotabahagia" appears both on "pos aceh barat" and "pos aceh barat daya".

{% include elements/figure.html image="assets/images/google maps - pos aceh barat.png" caption="POS Aceh Barat"%}

{% include elements/figure.html image="assets/images/google maps - pos aceh barat daya.png" caption="POS Aceh Barat Daya"%}

To prevent the scraping code to visit the same urls, the visited urls can be saved in a separate csv file. The following code will create the file in an empty folder, before any city or regency is scraped.
```R
if (length(kabkota_done) == 0){
  visited_urls = c()
  write.csv(visited_urls, paste0(raw_dir,"visited_urls.csv"), row.names = FALSE)
}
```

### 2.1 Scrolling
Assuming a keyword returns a list of results such as in the previous picture, the scraping process is generally done with these steps:
1. extract urls from all search results
1. visit all urls that are not found in `visited_urls`
1. extract the data

However, in some cases, the list is so long that a scrolling is needed to load all the urls.
![A long list of search results](../assets/gif/google maps - scrolling.gif)


The scrolling process can be done using this code.
```R
scroll = function(client){
  search_list = client$findElements("xpath", '//*[@id="QA0Szd"]/div/div/div[1]/div[2]/div/div[1]/div/div/div[2]/div[1]')
  current_results = search_list[[1]]$getElementText()[[1]]
  end = NULL
  i = 0 # retry
  while (is.null(end)){
    last_results = current_results
    search_list[[1]]$sendKeysToElement(list(key = "page_up"))
    search_list[[1]]$sendKeysToElement(list(key = "page_up"))
    search_list[[1]]$sendKeysToElement(list(key = "end"))
    Sys.sleep(1)
    end = 
      tryCatch({
        suppressMessages(client$findElement("class", "HlvSq"))
      }, error = function(e){
        return(NULL)
      })
    
    # if the results are the same as the previous results (before scrolling)
    # for 10 consecutive times, then refresh
    current_results = client$findElements("xpath", '//*[@id="QA0Szd"]/div/div/div[1]/div[2]/div/div[1]/div/div/div[2]/div[1]')[[1]]$getElementText()[[1]]
     # add retry if the results are same, reset if not
    if (current_results == last_results){i = i + 1} else {i = 0}
    if (i == 30){
      client$refresh()
      current_results = client$findElements("xpath", '//*[@id="QA0Szd"]/div/div/div[1]/div[2]/div/div[1]/div/div/div[2]/div[1]')[[1]]$getElementText()[[1]]
      search_list = client$findElements("xpath", '//*[@id="QA0Szd"]/div/div/div[1]/div[2]/div/div[1]/div/div/div[2]/div[1]')
      i = 0
    }
  }
}
```

The code uses a combination of `page up` and `end` button to get to the end of the loaded list. The `page up` and `end` buttons are pressed multiple times until an element of class `HlvSq` is found. The element is the text `You've reached the end of the list.` found at the end of the list.

There are also some cases where the list failed to load more results. In such cases, the code will retry for 30 times and will refresh the page if the list does not change.

### 2.2 Multiple Search Results?
Although most cities and regencies return multiple results, there are also some cities and regencies that return a single result, meaning that there is only one branch in that particular city or regency. In such cases, the scrolling code will fail since the code will try to find the `HlvSq` forever. 

The following code will check whether the search result is a list.
```R
is_list = function(client){
  image_pane = 
    tryCatch({
      suppressMessages(
        client$findElement("class", "ZKCDEc")
      )
    }, error = function(e){
      return(NULL)
    })
  # if it does not have image, then it is a list
  return(ifelse(is.null(image_pane), TRUE, FALSE)) 
}
```

The code is trying to find an element of class `ZKCDEc`, which is the image pane found in each url.

{% include elements/figure.html image="../assets/images/google maps - pos lhokseumawe.png" caption="An element of class `ZKCDEc`"%}

If the image pane is found, the data can be extracted immeadiately. But, if the image page is not found, there must be a list of results, whose urls will be visited separately before the data is extracted, following the general steps explained in the [previous section](#21-scrolling).

### 2.3 Extracting urls from Each Search Results
As explained in the [previous section](#21-scrolling), if a keyword returns a list of results, urls from each result will be visited separately. To do so, the urls of each result have to be extracted first. The following code extract the urls, but only after the scrolling process ends.
```R
extract_search_url = function(client){
  search_results = client$findElements("class name", 'hfpxzc')
  urls = c()
  for (idx_result in seq(length(search_results))){
    urls = append(urls, search_results[[idx_result]]$getElementAttribute("href")[[1]])
  }
  return(urls)
}
```

`hfpxzc` refers to each search result.

### 2.4 Extracting the Data
There are various data that can be extracted from google maps. In this case, the data that I am interested in are the following:
1. Branch latitude and logitude,
1. Branch name,
1. Branch type,
1. Branch rating,
1. Number of review,
1. Branch address,
1. Branch status, can be either open, closed (outside of working hours), temporarily closed, or permanently closed.

<br>
**Branch latitude and logitude**<br>
Latitude and longitude can be extracted from the branch url. However, there are two different urls. If a keyword returns only one result, the url will contain the latitude and longitude. However, if a keyword returns a list of results, the extracted urls from each of those results will not contain latitude and longitude. Typically, when visiting the urls, the urls will change after a few seconds to the type of urls with latitude and longitude. The following piece of code will extract the latitude and longitude, considering whether the keyword returned a single or multiple results.

```R
if (multiple_results == TRUE){
    lat_detect = FALSE
    long_detect = FALSE
    while ((lat_detect == FALSE) | (long_detect == FALSE)){ # wait until the url changes to coordinate url
      coord_url = client$getCurrentUrl()[[1]]
      lat_detect = coord_url %>% str_detect("(?<=@)[0-9.-]+(?=,)")
      long_detect = coord_url %>% str_detect("(?<=,)[0-9.-]+(?=,)")
    }
  } else (
    coord_url = url
  )
longitude = coord_url %>% str_extract("(?<=,)[0-9.-]+(?=,)")
latitude = coord_url %>% str_extract("(?<=@)[0-9.-]+(?=,)")
```

The latitude and logitude are usually started with an @ and ended with a z (zoom), which can be captured using the above regex.

<br>
**Branch name**<br>
The branch name can be easily identified by class `fontHeadlineLarge`. The following code extract the branch name.

```R
branch_name =
    tryCatch({
      suppressMessages(
        client$findElement("class", "fontHeadlineLarge")$getElementText()[[1]]
      )
    }, error = function(e){
      return(NA)
    })
```

<br>
**Branch type**<br>
The branch type is important, especially for "POS", because "POS" is a generic word that can refers to other places such as police station or military post in Bahasa Indonesia. Extracting this data can be done by catching element of class `DkEaL` using the following code.

```R
type = 
    tryCatch({
      suppressMessages(client$findElement("class", "DkEaL")$getElementText()[[1]])
    }, error = function(e){
      return(NA)
    })
```

<br>
**Branch rating**<br> 
This data can be used to evaluate the performance of each branch. The following code extract the rating of each branch.

```R
rating = 
    tryCatch({
      suppressMessages(
        client$findElement("xpath", '//*[@id="QA0Szd"]/div/div/div[1]/div[2]/div/div[1]/div/div/div[2]/div/div[1]/div[2]/div/div[1]/div[2]/span[1]/span[1]')$getElementText()[[1]]
      ) %>%
        str_replace(",",".") %>%
        as.numeric()
    }, error = function(e){
      return(NA) # temporary or permanently closed place don't have rating or num_review
    })
```

<br>
**Number of review**<br>
The reliability of the branch rating can be measured by number of reviews. Ideally, the more number of reviews, the more reliable the rating is.

```R
num_review = 
    tryCatch({
      suppressMessages(
        client$findElement("xpath", '//*[@id="QA0Szd"]/div/div/div[1]/div[2]/div/div[1]/div/div/div[2]/div/div[1]/div[2]/div/div[1]/div[2]/span[2]/span/span')$getElementText()[[1]] 
      ) %>%
        str_extract("[0-9]+") %>%
        as.integer()
    }, error = function(e){
      return(NA) # temporary or permanently closed place don't have rating or num_review
    })
```

<br>
**Branch address, phone numbers, and plus code**<br>
Every url has different details, some have address and phone number, while some only have adress. Fortunately, these details have the same class, `CsEnBe`. While there are also some elements in google maps that have `CsEnBe` class, such as "Claim this business" and "Send to your phone" buttons, these buttons can be easily excluded since address, phone number, and plus code usually have ":". The following code extract all elements with `CsEnBe` class and excluding elements without ":".

```R
details =
    tryCatch({
      details_list = suppressMessages(client$findElements("class name", "CsEnBe"))
      details = sapply(1:(length(details_list)), function(x) details_list[[x]]$getElementAttribute("aria-label")[[1]])
      details = 
        details[details %>% str_detect(":")] %>% 
        as_tibble() %>%
        mutate(
          detail = str_extract(value, "[A-Za-z ]+(?=:)"),
          value = str_extract(value, "(?<=: ).+") %>% str_trim()
        ) %>%
        relocate(detail)
    }, error = function(e){
      return(NA)
    })
```

<br>
**Branch status**<br>
Status in google maps can be either open, closed (outside of working hours), temporarily closed, or permanently closed. There are some cases where a branch is found, but the branch is temporarily, or even, permanently closed. Extracting this data can help in excluding these kind of branches later in data cleaning.

```R
status = 
    tryCatch({
      suppressMessages(
        client$findElement("class", "o0Svhf")$getElementText()[[1]]
      )
    }, error = function(e){
      return("unknown")
    })
```

<br>
**Defining an extraction function**<br>
The following `extract_data` function combines all extraction processes explained before.

```R
extract_data = function(client, url, multiple_results){
  # Extract the coordinate url while waiting for the load to complete  -------
  
  if (multiple_results == TRUE){
    lat_detect = FALSE
    long_detect = FALSE
    while ((lat_detect == FALSE) | (long_detect == FALSE)){ # wait until the url changes to coordinate url
      coord_url = client$getCurrentUrl()[[1]]
      lat_detect = coord_url %>% str_detect("(?<=@)[0-9.-]+(?=,)")
      long_detect = coord_url %>% str_detect("(?<=,)[0-9.-]+(?=,)")
    }
  } else (
    coord_url = url
  )
  

  # Extract other data ------------------------------------------------------
  
  branch_name =
    tryCatch({
      suppressMessages(
        client$findElement("class", "fontHeadlineLarge")$getElementText()[[1]]
      )
    }, error = function(e){
      return(NA)
    })
  
  type = 
    tryCatch({
      suppressMessages(client$findElement("class", "DkEaL")$getElementText()[[1]])
    }, error = function(e){
      return(NA)
    })
    
  rating = 
    tryCatch({
      suppressMessages(
        client$findElement("xpath", '//*[@id="QA0Szd"]/div/div/div[1]/div[2]/div/div[1]/div/div/div[2]/div/div[1]/div[2]/div/div[1]/div[2]/span[1]/span[1]')$getElementText()[[1]]
      ) %>%
        str_replace(",",".") %>%
        as.numeric()
    }, error = function(e){
      return(NA) # temporary or permanently closed place don't have rating or num_review
    })
  
  num_review = 
    tryCatch({
      suppressMessages(
        client$findElement("xpath", '//*[@id="QA0Szd"]/div/div/div[1]/div[2]/div/div[1]/div/div/div[2]/div/div[1]/div[2]/div/div[1]/div[2]/span[2]/span/span')$getElementText()[[1]] 
      ) %>%
        str_extract("[0-9]+") %>%
        as.integer()
    }, error = function(e){
      return(NA) # temporary or permanently closed place don't have rating or num_review
    })
  
  details =
    tryCatch({
      details_list = suppressMessages(client$findElements("class name", "CsEnBe"))
      details = sapply(1:(length(details_list)), function(x) details_list[[x]]$getElementAttribute("aria-label")[[1]])
      details = 
        details[details %>% str_detect(":")] %>% 
        as_tibble() %>%
        mutate(
          detail = str_extract(value, "[A-Za-z ]+(?=:)"),
          value = str_extract(value, "(?<=: ).+") %>% str_trim()
        ) %>%
        relocate(detail)
    }, error = function(e){
      return(NA)
    })
  
  status = 
    tryCatch({
      suppressMessages(
        client$findElement("class", "o0Svhf")$getElementText()[[1]]
      )
    }, error = function(e){
      return("unknown")
    })
  
  longitude = coord_url %>% str_extract("(?<=,)[0-9.-]+(?=,)")
  latitude = coord_url %>% str_extract("(?<=@)[0-9.-]+(?=,)")
   
  # Output ------------------------------------------------------------------
  return(
    tibble(
      branch_name = branch_name,
      type = type,
      rating = rating,
      num_review = num_review,
      status = status,
      search_url = url,
      coord_url = coord_url,
      longitude = longitude,
      latitude = latitude
    ) %>% 
      bind_cols(details)
  )
}
```

### 2.5 Parallel Scraping
To speed up the scraping process, the scraping can be done in parallel. To do so, all codes above can be combined into one single function named `main`.

```R
main = function(client, business_name, aliases = c(), city, dir){
  # update visited_urls
  all_bussiness_names = append(business_name, aliases)
  all_bussiness_names_encode = URLencode(all_bussiness_names, reserved = TRUE)
  visited_urls = suppressMessages(read_csv(paste0(raw_dir,"visited_urls.csv"))[,1]) %>% pull()
  
  keyword = paste(business_name, city)
  client$navigate(main_url)
  search_box = client$findElement("id", "searchboxinput")
  search_box$clearElement()
  search_box$sendKeysToElement(list(keyword, key = "enter"))
  # client$setTimeout("implicit", 5000)
  Sys.sleep(5)
  
  multiple_results = is_list(client)
  if (multiple_results == FALSE){
    url = client$getCurrentUrl()[[1]]
    if (!url %in% visited_urls){
      data = extract_data(client, url, multiple_results)
      visited_urls = append(visited_urls, url)
    }
  } else if (multiple_results == TRUE){
    not_found = 
      tryCatch({
        suppressMessages(client$findElement("class", "Q2vNVc"))
      }, error = function(e){
        return(NULL) # found search results
      })
    if (is.null(not_found)){
      scroll(client)
      urls = extract_search_url(client)
      urls = urls[str_detect(urls, regex(paste(all_bussiness_names_encode,collapse="|"), ignore_case = T))]
      urls = urls[!urls %in% visited_urls]
      
      # extract the data 
      data = tibble()
      idx_branch = 0
      for (url in urls){
        client$navigate(url)
        layer_button = NULL
        while (is.null(layer_button)){
          layer_button = 
            tryCatch({
              suppressMessages(
                client$findElement("class", "qk5Wte")
              )
            }, error = function(e){
              return(NULL)
            })
        }
        temp = extract_data(client, url, multiple_results)
        data = bind_rows(data, temp)
        rm(temp)
        visited_urls = append(visited_urls, url)
        idx_branch = idx_branch + 1
        cat(paste("\r", city, ": ", idx_branch, " of ", length(urls)))
      }
    } else {
      data = tibble()
    }
  }
  write.csv(data, paste0(dir, city, ".csv"), row.names = FALSE)
  write.csv(visited_urls, paste0(raw_dir,"visited_urls.csv"), row.names = FALSE)
}
```


This `main` function is then be used inside a loop, so that the `main` function will keep scraping data untill all cities or regencies are scraped. The parallel scraping is done using `foreach` and `doParallel` packages.
```R
library(tidyverse)
library(foreach)
library(doParallel)

Sys.sleep(5)
(cl <- (detectCores() - 1) %>%  makeCluster) %>% registerDoParallel
Sys.sleep(5)
clusterEvalQ(cl, {
  library(RSelenium)
  library(netstat)
  library(tidyverse)
  
  source("is_list.R")
  source("extract_data.R")
  source("scroll.R")
  source("extract_search_url.R")
  source("main.R")
  
  rs_obj =
    rsDriver(
      browser = "chrome",
      chromever = "114.0.5735.90",
      port = free_port()
    )
  remDr = rs_obj$client
  
  business_name = "mixue"
  raw_dir = paste0("data/raw/",business_name,"/")
  daftar_kota =
    read_csv("data/clean/table_city_tidy.csv") %>%
    pull(kabupaten_kota) %>%
    str_to_lower()
  main_url = "http://maps.google.com/"
  kabkota_done = list.files(raw_dir) %>% str_extract(".+(?=.csv)")
  kabkota_remaining = daftar_kota[!daftar_kota %in% kabkota_done]
  if (length(kabkota_done) == 0){
    visited_urls = c()
    write.csv(visited_urls, paste0(raw_dir,"visited_urls.csv"), row.names = FALSE)
  }
})

business_name = "mixue"
raw_dir = paste0("data/raw/",business_name,"/")
daftar_kota =
  read_csv("data/clean/table_city_tidy.csv") %>%
  pull(kabupaten_kota) %>%
  str_to_lower()

complete = NULL
while (is.null(complete)){
  kabkota_done = list.files(raw_dir) %>% str_extract(".+(?=.csv)")
  kabkota_remaining = daftar_kota[!daftar_kota %in% kabkota_done]
  complete =
    tryCatch({
      parSapply(
        cl, 
        kabkota_remaining, 
        function(x) 
          main(
            client = remDr, 
            business_name = business_name, 
            # aliases = "ninjaxpress",
            city = x, 
            dir = raw_dir
          )
      )
    }, error = function(e){
      return(NULL)
    })
}
```

Running above codes will open multiple instances. The number of instances will vary based on available cores assigned in `cl`.

When the scraping process is finished, there will be multiple files in the `raw_dir` with different sizes. These different sizes indicate that each city or region has different number of branches.

{% include elements/figure.html image="assets/images/google maps - pos final folder.png" caption="`raw_dir` after the scraping process is finished."%}