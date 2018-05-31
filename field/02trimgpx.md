Trim GPX for QGIS
================
Michelle Stuart
6/7/2017

``` r
library(tidyverse) # for pipe functions
library(lubridate) # for changing time zones
library(stringr) # for combining dates and times into dttm
source("readGPXGarmin.R")
source("writeGPXGarmin.R")
```

Get the dive info from the excel spreadsheet

``` r
excel_file <- ("data/GPSSurveys2017.xlsx")
surv <- readxl::read_excel(excel_file, sheet = "diveinfo", col_names=TRUE)
names(surv) <- stringr::str_to_lower(names(surv))
```

Reformat to reduce the number of columns and get rid of weird dates from excel

``` r
surv <- surv %>% 
  select(divenum, date, starttime, endtime, pausestart, pauseend, gps)

surv <- surv %>% 
  separate(starttime, into = c("baddate", "starttime"), sep = " ") %>%
  separate(endtime, into = c("baddate2", "endtime"), sep = " ") %>%
  separate(pausestart, into = c("baddate4", "pausestart"), sep = " ") %>%
  separate(pauseend, into = c("baddate5", "pauseend"), sep = " ") %>% 
  select(-contains("bad")) 
```

Combine date and time to form a dttm column and set the time zone to PHT, Asia/Manila

``` r
surv$start <- str_c(surv$date, surv$starttime, sep = " ")  
surv$start <- ymd_hms(surv$start)
surv$start <- force_tz(surv$start, tzone = "Asia/Manila")
surv$end <- str_c(surv$date, surv$endtime, sep = " ")  
surv$end <- ymd_hms(surv$end)
surv$end <- force_tz(surv$end, tzone = "Asia/Manila")
surv$paust <- str_c(surv$date, surv$pausestart, sep = " ")  
surv$paust <- ymd_hms(surv$paust)
surv$paust <- force_tz(surv$paust, tzone = "Asia/Manila")
surv$pausend <- str_c(surv$date, surv$pauseend, sep = " ")  
surv$pausend <- ymd_hms(surv$pausend)
surv$pausend <- force_tz(surv$pausend, tzone = "Asia/Manila")
```

Change time zone to UTC

``` r
surv$start <- with_tz(surv$start, tzone = "UTC")
surv$end <- with_tz(surv$end, tzone = "UTC")
surv$paust <- with_tz(surv$paust, tzone = "UTC")
surv$pausend <- with_tz(surv$pausend, tzone = "UTC")
```

Read in each GPX file, find the survey that matches, and trim the file to fit the survey, write an output file in gpx format. **This is taking a very long time to run.** Need to find a better way to do this.

``` r
folders  <-  list.files(path = "data", pattern ="gps") # move any empty folders out of the data folder
for (l in 1:length(folders)){
  files <- list.files(path = paste("data/",folders[l], sep = ""), pattern = "*Track*")
  for(i in 1:length(files)){ # for each file
    infile <- readGPXGarmin(paste("data/", folders[l], "/", files[i], sep="")) 
    header <- infile$header
    data <- infile$data
    data$time <- ymd_hms(data$time)
    data <- arrange(data, time)
    # start time for this GPX track
    instarttime <-  data$time[1]
    # end time for this GPX track
    inendtime = data$time[nrow(data)]
    # change elevation to zero
    data$elev <- 0
    data$unit <- substr(folders[l],4,4)
    
    
    # which survey started after the gpx and ended before the gpx ####
    inds <- surv %>% 
      filter(start >= instarttime & end <= inendtime & !is.na(divenum))
    
    # if none of the surveys fit ####
    if(nrow(inds) == 0){
        print(str_c("File", files[i], "does not cover a complete survey"))
      
      # find a survey that at least starts or ends within this GPX track
      # which survey ended before the gpx and ended after the gpx started or started before the gpx ended and started after the gpx started
        inds <- surv %>% 
          filter((end <= inendtime & end >= instarttime)| (start <= inendtime & start >= instarttime))
        if(nrow(inds) == 0){
            print(str_c("EVEN WORSE:", files[i], "does not cover even PART of a survey"))
        }
    }
    if(nrow(inds) > 0){
        for(j in 1:nrow(inds)){ # step through each survey that fits within this track (one or more)
            # output all: not just if this was a dive for collecting APCL
          # if no pause
          if(is.na(inds$paust[j])){ 
            # find the GPX points that fit within the survey
                k <- data %>% 
                  filter(time >= inds$start[j] & time <= inds$end[j])
                k$lat <- as.character(k$lat)
                k$lon <- as.character(k$lon)
                k$elev <- as.character(k$elev)
                k$time <- as.character(k$time)
                outfile <- k
            
                
                writeGPX(filename = str_c("data/gpx_trimmed/GPS", outfile$unit[1], "_", inds$divenum[j], "_", files[i], sep=""), outfile = outfile)
            }
            if(!is.na(inds$paust[j])){ # account for a pause if need be
              k1 <- data %>% 
                filter(time >= inds$start[j] & time <= inds$paust[j])
              k1$lat <- as.character(k1$lat)
              k1$lon <- as.character(k1$lon)
              k1$elev <- as.character(k1$elev)
              k1$time <- as.character(k1$time)
              outfile1 <- k1
                
              k2 <- data %>% 
                filter(time >= inds$pausend[j] & time <= inds$end[j])
              k2$lat <- as.character(k2$lat)
              k2$lon <- as.character(k2$lon)
              k2$elev <- as.character(k2$elev)
              k2$time <- as.character(k2$time)
              outfile2 <- k2
            
              writeGPX(filename = paste("data/gpx_trimmed/GPS", outfile$unit[1], "_",inds$divenum[j], "_", files[i], '_1.gpx', sep=''), outfile = outfile1) # write as two tracks
              writeGPX(filename = paste("data/gpx_trimmed/GPS", outfile$unit[1], "_",inds$divenum[j], "_", files[i], '_2.gpx', sep=''), outfile = outfile2)
              

            }
        }
    }
  }
}
```

    ## [1] "FileTrack_2017-05-24 08.12.14 Day.gpxdoes not cover a complete survey"
    ## [1] "EVEN WORSE:Track_2017-05-24 08.12.14 Day.gpxdoes not cover even PART of a survey"