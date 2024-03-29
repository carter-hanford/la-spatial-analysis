Data Cleaning R Notebook
================
Carter Hanford
(March 20, 2020)

## Introduction

This is the R notebook for the data cleaning / variable calculation step
for the project in SOC5670 - Spatial Demography. In this notebook, we
will clean the data set for our final project and create any new
variables that we will need for our final spatial analysis.

This project is a spatial analysis of the City of Los Angeles.

## Packages

This code chunk loads the necessary R packages that we will need for our
analysis. I tend to work under the `tidy` workflow when it comes to data
analysis/wrangling, so most of the packages are home to the `tidyverse`.

``` r
# tidyverse packages
library(readr)     # read .csv files
library(here)      # file path management
```

    ## here() starts at /Users/carterhanford/Documents/GitHub/spatial-demography-spring-2020/project

``` r
library(tidyr)     # data management
library(dplyr)     # data cleaning
```

    ## 
    ## Attaching package: 'dplyr'

    ## The following objects are masked from 'package:stats':
    ## 
    ##     filter, lag

    ## The following objects are masked from 'package:base':
    ## 
    ##     intersect, setdiff, setequal, union

``` r
# other packages
library(openxlsx)  # write to excel
```

## Load Raw Data

This code chunk loads the raw, uncleaned data into our global
environment from our raw data folder. This data comes from [Social
Explorer](https://www.socialexplorer.com/explore-maps), and is generated
using custom built reports.

``` r
rawdata <- read_csv(here("data", "raw data", "R12488462_SL140.csv"))
```

## Clean / Subset Data

Now we can get started with our data cleaning. The first thing we want
to do is create a subset of the raw data which only includes the
variables that we need for our analysis.

``` r
# subset selection based on variables for analysis
rawdata %>%
  select(Geo_FIPS, SE_A04001_001, SE_A00002_002, SE_A00002_003, SE_A04001_003, 
         SE_A04001_004, SE_A04001_010, SE_B12001_001, SE_B12001_002, SE_B12001_003, 
         SE_B12001_004, SE_A14006_001, SE_A10036_001, SE_A18009_001, SE_A13004_002,
         SE_A13004_003, SE_A13004_004, SE_A13004_001) -> rawdata_sub
```

Now we’ll go through the **fun** process of renaming all of our
variables. This is a really tedious step, but I find it helpful to knock
this out immediately so that: \* 1 - our variable names are easy to
locate \* 2 - calculating new variables becomes easier

The last couple lines of code in this code chunk simply move certain
variables around to where I like them to be in the data frame. This is
completely optional and is just an organization step for me.

``` r
# rename all variables intuitively 
rawdata_sub %>%
  rename(totalpop = SE_A04001_001,
         popdensity = SE_A00002_002,
         area = SE_A00002_003,
         popwhite = SE_A04001_003,
         popblack = SE_A04001_004,
         pophispanic = SE_A04001_010,
         eapop25over = SE_B12001_001,
         less_highschool = SE_B12001_002,
         highschool = SE_B12001_003,
         bachelorplus = SE_B12001_004,
         med_income = SE_A14006_001,
         med_homevalue = SE_A10036_001,
         med_grossrent = SE_A18009_001,
         povratio_under_.5 = SE_A13004_002,
         povratio_.5_.74 = SE_A13004_003,
         povratio_.75_.99 = SE_A13004_004,
         povstatus_pop = SE_A13004_001,
         FIPS = Geo_FIPS) -> projectdata

# clean up variable placement
projectdata %>%
  select(FIPS, area, popdensity, med_homevalue, med_grossrent, totalpop, popwhite, 
         popblack, pophispanic, povstatus_pop, povratio_under_.5, 
         povratio_.5_.74, povratio_.75_.99, med_income, eapop25over, less_highschool, 
         highschool, bachelorplus) -> projectdata
```

## Calculate Poverty Rates

Now that we have our data set cleaned and our variables intuitively
named, we can start with our calculations for certain new variables we
need for our analysis.

We’ll start with calculating normalized `poverty rates` for each census
tract:

The workflow here is as follows:

  -   - calculate new variable

  -   - add new variable to existing data frame

  -   - remove the now irrelevant variables from existing data frame

  -   - remove the calculated data object from the global environment.

I’ve found that the last step of this workflow is really helpful when
trying to keep large projects organized. I used to have very messy
global environments with lots of objects and variables all over the
place with similar names, which makes it difficult to find certain
variables while progressing through a project. Simply keeping track of
them and removing unnecessary ones along the way has done wonders for my
organization\!

``` r
# calculate poverty rate
povrate <- ((projectdata$povratio_under_.5+projectdata$povratio_.5_.74+projectdata$povratio_.75_.99)/projectdata$povstatus_pop)

# add to existing data frame
projectdata <- data.frame(projectdata, povrate)
  
# clean up excess variables
projectdata %>%
  select(-povstatus_pop, -povratio_.5_.74, 
         -povratio_under_.5, 
         -povratio_.75_.99) -> projectdata

# clean up global environment
rm(povrate)
```

## Calculate Normalized Demographic Population Percentages

Now we’ll calculate normalized statistics for tract demographics. We
never want to use raw population counts for each demographic in a census
tract because they are not normalized with the total population of the
tract. A **normalized** demographic variable simply means it is
calculated with the total population taken into account.

For example, we don’t want to use the simple `popblack` variable to
signify a population variable for blacks in a census tract. Instead, we
want to take the black population count and divide it by the total
population count to get a normalized proportion.

We’ll do that for the three demographic groups below, following the same
workflow mentioned previously:

``` r
# calculate normalized demographic percentages
pctwhite <- projectdata$popwhite/projectdata$totalpop
pctblack <- projectdata$popblack/projectdata$totalpop
pcthispanic <- projectdata$pophispanic/projectdata$totalpop

# add to existing data frame
projectdata <- data.frame(projectdata, pctwhite, pctblack, pcthispanic)

# clean up excess variables 
projectdata %>%
  select(-popwhite, -popblack, -pophispanic) -> projectdata

# clean up environment
rm(pctwhite, pctblack, pcthispanic)
```

## Calculate Educational Attainment

Finally, the last variable we’ll need to calculate is educational
attainment. We need to follow a similar calculation workflow as the
demographic code chunk since the existing variable is simply a count,
and NOT normalized.

I broke down the existing educational attainment variables into three
categories:

  -   - `ed_hs_below` - below a high school education

  -   - `ed_hs` - high school education

  -   - `ed_bachelor_plus` - college education or more

We’ll calculate these new variables below and follow the same
organization workflow:

``` r
# calculate normalized educational attainment rates
ed_hs_below <- projectdata$less_highschool/projectdata$eapop25over
ed_hs <- projectdata$highschool/projectdata$eapop25over
ed_bachelor_plus <- projectdata$bachelorplus/projectdata$eapop25over

# add to existing data frame
projectdata <- data.frame(projectdata, ed_hs_below, ed_hs, ed_bachelor_plus)

# clean up excess variables 
projectdata %>%
  select(-eapop25over, -less_highschool, -highschool, -bachelorplus) -> projectdata

# clean up environment
rm(ed_hs, ed_hs_below, ed_bachelor_plus)
```

## Clean Up Final Data Frame

Now that we have our variables created that we need for our analysis, we
can move on to cleaning up the final data frame before exporting\!

There are some missing values in this particular data frame, and we want
to remove those before moving on. Because there are missing values (NA),
when we perform some calculations R will return *NaN* values which stand
for “not a number.” This happens (typically) when performing an
operation where the denominator is NA, which will therefore not complete
the equation.

We’ll remove both NA and NaN values using a simple pipeline of two
functions:

``` r
# remove all NaN and NA values
projectdata %>%
  na_if("NaN") %>%
  na.omit() -> projectdata
```

## Export Final Clean Data Frame

Now we can finally export our cleaned data set\! Typically we would
export this as a .csv value using `readr`, but for our purposes we are
going to export this to an excel file so that we can perform a clean
merge in *ArcPro* later with our Los Angeles City shapefile.

Thanks for following along\!

``` r
# export final project data to clean data folder
write.xlsx(projectdata, here("data", "clean data", "projectdata.xlsx"))
```
