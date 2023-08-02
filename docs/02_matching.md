# (PART\*) Impact analysis {-}

# Matching

In this R Markdown are performed the different steps to obtain a matched dataset, i.e a dataset with control and treated observational units to compute the treatment effect. The treatment here is to be under protected area status, and we look at the impact on deforestation.

The steps are the following

1.  Pre-processing : in a loop for each country

    1.  Create a grid of the country

    2.  Import geospatial data on PAs from the WDPA dataset, and assign each observation unit/pixel to a group : PA funded by the AFD (treated), PA non-funded by the AFD, buffer (closed to but not a PA), other (so potential control).

    3.  Compute the covariates and outcome of interest in all pixels thanks to the mapme.biodiversity package

    4.  Build the matching data frame : each pixel is assigned to a group and has covariates and outcome values.

2.  Post-processing : in a loop for each country

    1.  Load the matching dataframe of the given country

    2.  Perform the matching

    3.  Plot covariate balance and density plots to ensure relevant matching

    4.  Panelize the dataframe

    5.  Perform regressions

## Initial settings

#```{r setup, include=FALSE, eval = FALSE}
#knitr::opts_knit$set(root.dir = rprojroot::find_rstudio_root_file())
# ```


```r
#Install some libraries
install.packages(c("tictoc", "geodata", "wdpar", "exactextractr", "MatchIt", "fixest", "cobalt"))
remotes::install_github("mapme-initiative/mapme.biodiversity", upgrade="always")

#Install the web driver to download wdpa data directly
webdriver::install_phantomjs()

# Load Libraries
library(dplyr)
library(tictoc) #For timing
library(xtable)
library(tidyr)
library(stringr)
library(ggplot2) # For plotting
library(sf) # For handling vector data
library(terra) # For handling raster data
library(raster) # For handling raster data
library(rgeos)
library(geodata) # For getting country files
library(wdpar) # For getting protected areas
library(exactextractr) # For zonal statistics
library(mapme.biodiversity)
library(aws.s3)
library(MatchIt) #For matching
library(fixest) #For estimating the models
library(cobalt) #To visualize density plots and covariate balance from MatchIt outcomes
```


```r
#Import functions
source("scripts/functions/02_fns_matching.R")
```


```r
# Define the path to a temporary, working directory for pre- and post-processing steps
tmp_pre = paste(tempdir(), "matching_pre", sep = "/")
tmp_post = paste(tempdir(), "matching_post", sep = "/")

# Define the file name of the matching frame to save/import
name_output = name_input = "matching_frame_10km"
ext_output = ext_input = ".gpkg"

#####
###Pre-processing
#####

##Buffer and gridsize : make it depends on the country ? For instance, a typical size of grid such that the smaller PA is sampled with at least N pixels ?
# Specify buffer width in meter
buffer_m = 10000
# Specify the grid cell size in meter
gridSize = 10000 

#Load data 
data_pa_full = 
  #fread("data_tidy/BDD_DesStat_nofund_nodupl.csv" , encoding = "UTF-8")
  aws.s3::s3read_using(
  FUN = data.table::fread,
  encoding = "UTF-8",
  object = "data_tidy/BDD_DesStat_nofund_nodupl.csv",
  bucket = "projet-afd-eva-ap",
  opts = list("region" = ""))

#List of countries in the sample
#list_iso = unique(data_pa_full$iso3)
list_iso = c("COM", "MMR")
# Specify a list of WDPA IDs of funded protected areas (treated areas)
# paid = data_stat_nodupl[data_stat_nodupl$iso3 == country,]$wdpaid

# Start year
y_first = 2000
# End year
y_last = 2021

#####
###Post-processing
#####

# Define Column Names of Covariates
colname.travelTime = "minutes_median_5k_110mio"
colname.clayContent = "clay_0.5cm_mean"
# colname.elevation = "elevation_mean"
# colname.tri = "tri_mean"
colname.fcIni = "treecover_2000"


# Prefix of columns for forest cover
colfc.prefix = "treecover"
# Separation between prefix and year
colfc.bind = "_"
# Prefix of columns for forest loss
colfl.prefix = "treeloss"
#Prefix of columns for average forest loss pre-funding
colname.flAvg = "avgLoss_pre_fund"

# Year of treatment
treatment.start = 2014
```

## Matching process

For pre- and post-processing steps, the different functions are called in a loop (one iteration per country). 

### Pre-processing


```r
#For each country in the list, the different steps of the pre-processing are performed
count = 0 #Initialize counter
max_i = length(list_iso) #Max value of the counter
tic_pre = tic() #Start timer
for (i in list_iso)
{
  #Update counter and display progress
  count = count+1
  print(paste0(i, " : country ", count, "/", max_i))
  
  #Generate observation units
  print("--Generating observation units")
  output_pre_grid = fn_pre_grid(iso = i, path_tmp = tmp_pre, gridSize = gridSize)
  
  #Load the outputs 
  utm_code = output_pre_grid$utm_code
  gadm_prj = output_pre_grid$ctry_shp_prj
  grid = output_pre_grid$grid
  
  #Determining Group IDs and WDPA IDs for all observation units
  print("--Determining Group IDs and WDPA IDs")
  grid_param = fn_pre_group(iso = i, path_tmp = tmp_pre, utm_code = utm_code,
                            buffer_m = buffer_m, data = data_pa_full,
                            gadm_prj = gadm_prj, grid = grid, 
                            gridSize = gridSize)
  
  #Calculating outcome and other covariates for all observation units
  print("--#Calculating outcome and other covariates")
  fn_pre_mf(grid.param = grid_param, path_tmp = tmp_pre, iso = i,
            name_output = name_output, ext_output = ext_output)
  
  
}

toc_pre = toc() #end timer
```

### Post-processing


```r
#For each country in the list, the different steps of the post-processing are performed
count = 0 #Initialize counter
max_i = length(list_iso) #Max value of the counter
tic_post = tic() #start timer
for (i in list_iso)
{
  #Update counter and show progress
  count = count+1
  print(paste0(i, " : country ", count, "/", max_i))
  
  #Load the matching frame
  print("--Loading the matching frame")
  mf_ini = fn_post_load_mf(iso = i)
  
  
  #Add average pre-loss
  print("--Add covariate : average tree loss pre-funding")
  mf = fn_post_avgLoss_prefund(mf = mf_ini, yr_start = treatment.start - 5,
                               yr_end = treatment.start - 1, colfl.prefix = colfl.prefix)
  
  #Define cut-offs
  print("--Define cutoffs")
  ##CAREFUL : ADD elevation and TRI when available
  lst_cutoffs = fn_post_cutoff(mf = mf, 
                               colname.travelTime = colname.travelTime, 
                               colname.clayContent = colname.clayContent,
                               colname.fcIni = colname.fcIni, 
                               colname.flAvg = colname.flAvg
                               )
  
  #Run Coarsened Exact Matching
  print("--Run CEM")
  out.cem = fn_post_cem(mf = mf, iso = i, path_tmp = tmp_post,
                        lst_cutoffs = lst_cutoffs,
                       colname.travelTime = colname.travelTime, 
                       colname.clayContent = colname.clayContent,
                       colname.fcIni = colname.fcIni, 
                       colname.flAvg = colname.flAvg)
  
  #Plots : covariates
  print("--Some plots : covariates")
  print("----Covariate balance")
  fn_post_plot_covbal(out.cem = out.cem, 
                      colname.travelTime = colname.travelTime, 
                       colname.clayContent = colname.clayContent,
                       colname.fcIni = colname.fcIni, 
                       colname.flAvg = colname.flAvg,
                      iso = i,
                      path_tmp = tmp_post)
  print("----Density plots")
  fn_post_plot_density(out.cem = out.cem, 
                      colname.travelTime = colname.travelTime, 
                       colname.clayContent = colname.clayContent,
                       colname.fcIni = colname.fcIni, 
                       colname.flAvg = colname.flAvg,
                      iso = i,
                      path_tmp = tmp_post)
  
  #Panelize dataframes
  print("----Panelize (Un-)Matched Dataframe")
  output_post_panel = fn_post_panel(out.cem = out.cem, mf = mf, colfc.prefix = colfc.prefix, colfc.bind = colfc.bind)
  matched.wide = output_post_panel$matched.wide
  unmatched.wide = output_post_panel$unmatched.wide
  matched.long = output_post_panel$matched.long
  unmatched.long = output_post_panel$unmatched.long  
  
  #Plots : trend
  print("----Plots again : trend")
  fn_post_plot_trend(matched.long = matched.long, unmatched.long = unmatched.long, iso = i)
  
  
}

toc_post = toc() #end timer

#Notes
## Automate the definition of cutoffs for CEM
### Coder 5.5.3 de Iacus et al. 2012 ? Permet de savoir le gain de matched units pour une modification des seuils d'une variable
## Loop for different treatment years : need to adapt fn_post_avgLoss_prefund function
## Allow to enter a list of any covariates to perform the matching
## Function to plot Fig. 3 in Iacus et al. 2012
## Il faut contrôler le nombre de paired units à la fin ! Typiquement on veut que tous les pixels traités soient retenus, idéalement.
## On veut ATE ou ATT ?? Je dirai ATT car on ne veut pas estimer l'effet de mettre une AP, mais l'effet des AP financés par l'AFD 
```