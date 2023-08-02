# (PART\*) Descriptive statistics {-}

# Projects funded by the AFD

In this document are performed and plotted the following descriptive statistics on the PAs funded the AFD :

-   Distribution of PAs among IUCN categories, at country, region and world level.
-   Distribution in terms of ecosystems
-   Distribution of PAs across countries and regions, in terms of numbers
-   Distribution of PAs across countries and regions, in terms of areas
-   Temporal evolution in the number and area of PAs funded by the AFD
-   Distribution in terms of governance types

The statistics are derived from datasets stored in the SSPCloud, and saved into the SSPCloud. Thus specific functions from the aws.S3 package are used (s3read_using() and s3write_using()). These can be replaced by other R functions to read/write locally (fread() typically). The ggplot2::ggsave() function cannot be used to write directly in the SSPCloud storage. Instead, plots from ggplot2 are stored in the temporary memory, then moved to the SSPCloud storage. Working locally, ggsave() can be directly used once the plots are created.

## Importing packages

#```{r setup, include=FALSE, eval = FALSE}
#knitr::opts_knit$set(root.dir = rprojroot::find_rstudio_root_file())
#```


```r
library(tidyverse)
library(stargazer)
library(dplyr)
library(ggplot2)
library(ggrepel)
library(RColorBrewer)
library(data.table)
#library(readxl)
#library(splitstackshape) 
library(janitor)
library(xtable)
library(questionr)
library(tidyterra)
library(terra)
library(sf)
library(mapview)
library(aws.s3)
```

## Importing datasets


```r
#Both datasets are imported in UTF8 encoding, for some variables
##A first dataset with some PA on more than one row (one line per funding for instance)
data_stat = 
  #fread("data_tidy/BDD_DesStat_nofund.csv", encoding = "UTF-8")
  aws.s3::s3read_using(
  FUN = data.table::fread,
  encoding = "UTF-8",
  # Mettre les options de FUN ici
  object = "data_tidy/BDD_DesStat_nofund.csv",
  bucket = "projet-afd-eva-ap",
  opts = list("region" = ""))

##A second dataset with one line per PA, more suited for some statistics
data_stat_nodupl = 
  #fread("data_tidy/BDD_DesStat_nofund_nodupl.csv" , encoding = "UTF-8")
  aws.s3::s3read_using(
  FUN = data.table::fread,
  encoding = "UTF-8",
  # Mettre les options de FUN ici
  object = "data_tidy/BDD_DesStat_nofund_nodupl.csv",
  bucket = "projet-afd-eva-ap",
  opts = list("region" = ""))

##Datasets on aggregated size per country/region/year and at world level, taking into account the overlap.
pa_area_ctry = 
  #fread("data_tidy/area/pa_area_ctry.csv", encoding = "UTF-8")
  aws.s3::s3read_using(
  FUN = data.table::fread,
  encoding = "UTF-8",
  # Mettre les options de FUN ici
  object = "data_tidy/area/pa_area_ctry.csv",
  bucket = "projet-afd-eva-ap",
  opts = list("region" = ""))

pa_area_dr = 
  #fread("data_tidy/area/pa_area_dr.csv", encoding = "UTF-8")
  aws.s3::s3read_using(
  FUN = data.table::fread,
  encoding = "UTF-8",
  # Mettre les options de FUN ici
  object = "data_tidy/area/pa_area_dr.csv",
  bucket = "projet-afd-eva-ap",
  opts = list("region" = ""))

pa_area_wld = 
  #fread("data_tidy/area/pa_area_wld.csv", encoding = "UTF-8")
  aws.s3::s3read_using(
  FUN = data.table::fread,
  encoding = "UTF-8",
  # Mettre les options de FUN ici
  object = "data_tidy/area/pa_area_wld.csv",
  bucket = "projet-afd-eva-ap",
  opts = list("region" = ""))

pa_int_yr = 
  #fread("data_tidy/area/pa_area_dr.csv", encoding = "UTF-8")
  aws.s3::s3read_using(
  FUN = data.table::fread,
  encoding = "UTF-8",
  # Mettre les options de FUN ici
  object = "data_tidy/area/pa_int_yr.csv",
  bucket = "projet-afd-eva-ap",
  opts = list("region" = ""))
```

## Performing descriptive statistics

### IUCN categories

#### Share of PAs by IUCN categories


```r
#Building the relevant dataset
##For all PAs ..
data_cat_iucn = data_stat_nodupl %>%
  group_by(iucn_des) %>%
  #number of PAs per IUCN category
  summarize(n_iucn = n()) %>%
  ungroup() %>%
  #Frequency of IUCN categories
  mutate(n_pa = sum(n_iucn),
         freq_iucn = round(n_iucn/n_pa*100, 1)) %>%
  arrange(desc(iucn_des)) %>%
  mutate(ypos_iucn = cumsum(freq_iucn) - 0.5*freq_iucn) 


##... and for referenced PAs only
data_cat_iucn_ref = data_stat_nodupl %>%
  #Remove not referenced PAs
  subset(!(iucn_des %in% c("Non catégorisée", "Non référencée"))) %>%
  group_by(iucn_des) %>%
  #number of PAs per IUCN category
  summarize(n_iucn = n()) %>%
  ungroup() %>%
  #Frequency of IUCN categories
  mutate(n_pa = sum(n_iucn),
         freq_iucn = round(n_iucn/n_pa*100, 1)) %>%
  arrange(freq_iucn) %>%
  mutate(ypos_iucn = cumsum(freq_iucn) - 0.5*freq_iucn) 

#Latex table
tbl_cat_iucn = data_cat_iucn %>%
  select(c(iucn_des, n_iucn, freq_iucn))
names(tbl_cat_iucn) <- c("Catégories IUCN","Nombre d'AP", "Proportion d'AP (%)")

tbl_cat_iucn_ref = data_cat_iucn_ref %>%
  select(c(iucn_des, n_iucn, freq_iucn))
names(tbl_cat_iucn_ref) <- c("Catégories IUCN","Nombre d'AP", "Proportion d'AP (%)")


#Histogram including non-referenced PAs
hist_cat_iucn = ggplot(data_cat_iucn, 
                       aes(x = reorder(iucn_des, -freq_iucn), 
                           y = freq_iucn, fill = iucn_des)) %>% 
  + geom_bar(stat = "identity", width = 0.50, fill="#3182BD") %>%
  + geom_text(aes(label = round(n_iucn, 1), y = freq_iucn), 
            vjust = -0.1, color="black",
            size=3.5) %>%
  + labs(title = "Proportion d'aires protégées par catégorie IUCN",
         subtitle = paste("Echantillon :", sum(data_cat_iucn$n_iucn), "aires protégées. Nombre d'aires indiqué sur les barres."),
          x = "Catégories IUCN", 
          y = "Nombre (%)") %>%
  + theme(legend.position = "bottom",
      legend.key = element_rect(fill = "white"),
      plot.title = element_text(size = 14, face = "bold"), 
      axis.text.x = element_text(angle = 45,size=9, hjust = .5, vjust = .6),
      panel.background = element_rect(fill = 'white', colour = 'white', 
                                      linewidth = 0.5, linetype = 'solid'),
      panel.grid.major = element_line(colour = 'grey90', linetype = 'solid'),
      panel.grid.minor = element_line(colour = 'grey90', linetype = 'solid'),
      plot.caption = element_text(color = 'grey50', size = 8.5, face = 'plain'))
hist_cat_iucn

#Histogram excluding non-referenced PAs
hist_cat_iucn_ref = ggplot(data_cat_iucn_ref, 
                       aes(x = reorder(iucn_des, -freq_iucn), 
                           y = freq_iucn, fill = iucn_des)) %>% 
  + geom_bar(stat = "identity", width = 0.50, fill="#3182BD") %>%
  + geom_text(aes(label = round(n_iucn, 1), y = freq_iucn), 
            vjust = -0.1, color="black",
            size=3.5) %>%
  + labs(title = "Proportion d'aires protégées par catégorie IUCN (hors AP non-répertoriées)",
         subtitle = paste("Echantillon :", sum(data_cat_iucn_ref$n_iucn), "aires protégées. Nombre d'aires indiqué sur les barres."),
          x = "Catégories IUCN", 
          y = "Nombre (%)") %>%
  + theme(legend.position = "bottom",
      legend.key = element_rect(fill = "white"),
      plot.title = element_text(size = 14, face = "bold"), 
      axis.text.x = element_text(angle = 45,size=9, hjust = .5, vjust = .6),
      panel.background = element_rect(fill = 'white', colour = 'white', 
                                      linewidth = 0.5, linetype = 'solid'),
      panel.grid.major = element_line(colour = 'grey90', linetype = 'solid'),
      panel.grid.minor = element_line(colour = 'grey90', linetype = 'solid'),
      plot.caption = element_text(color = 'grey50', size = 8.5, face = 'plain'))
hist_cat_iucn_ref


#Pie chart INcluding non-referenced PAs
pie_cat_iucn = ggplot(data_cat_iucn, 
                      aes(x="", y= freq_iucn, fill = iucn_des)) %>%
  + geom_bar(width = 1, stat = "identity", color="white") %>%
  + coord_polar("y", start=0) %>%
  + geom_label_repel(aes(x=1.2, label = paste0(round(freq_iucn, 1), "%")), 
             color = "white", 
             position = position_stack(vjust = 0.55), 
             size=2.5, show.legend = FALSE) %>%
  # + geom_label(aes(x=1.4, label = paste0(freq_iucn, "%")), 
  #              color = "white", 
  #              position = position_stack(vjust = 0.7), size=2.5, 
  #              show.legend = FALSE) %>%
  + labs(x = "", y = "",
         title = "Proportion d'aires protégées par catégorie IUCN (%)",
         subtitle = paste("Echantillon :", sum(data_cat_iucn$n_iucn), "aires protégées")) %>%
  + scale_fill_brewer(name = "Catégories", palette = "Dark2") %>%
  + theme_void()
pie_cat_iucn



#Pie chart EXcluding non-referenced PAs
pie_cat_iucn_ref = ggplot(data_cat_iucn_ref, 
                      aes(x="", y= freq_iucn, fill = iucn_des)) %>%
  + geom_bar(width = 1, stat = "identity", color="white") %>%
  + coord_polar("y", start=0) %>%
  + geom_label_repel(aes(x=1.2, label = paste0(round(freq_iucn, 1), "%")), 
             color = "white", 
             position = position_stack(vjust = 0.55), 
             size=2.5, show.legend = FALSE) %>%
  # + geom_label(aes(x=1.4, label = paste0(freq_iucn, "%")), 
  #              color = "white", 
  #              position = position_stack(vjust = 0.7), size=2.5, 
  #              show.legend = FALSE) %>%
  + labs(x = "", y = "",
         title = "Proportion d'aires protégées par catégorie IUCN \nhors aires non répertoriées (%)",
         subtitle = paste("Echantillon :", sum(data_cat_iucn_ref$n_iucn), "aires protégées")) %>%
  + scale_fill_brewer(name = "Catégories", palette = "Dark2") %>%
  + theme_void()
pie_cat_iucn_ref
```


```r
#Saving figures

tmp = paste(tempdir(), "fig", sep = "/")

ggsave(paste(tmp, "hist_cat_iucn.png", sep = "/"),
       plot = hist_cat_iucn,
       device = "png",
       height = 6, width = 9)

ggsave(paste(tmp, "hist_cat_iucn_ref.png", sep = "/"),
       plot = hist_cat_iucn_ref,
       device = "png",
       height = 6, width = 9)

ggsave(paste(tmp, "pie_cat_iucn.png", sep = "/"),
       plot = pie_cat_iucn,
       device = "png",
       height = 6, width = 9)

ggsave(paste(tmp, "pie_cat_iucn_ref.png", sep = "/"),
       plot = pie_cat_iucn_ref,
       device = "png",
       height = 6, width = 9)

print(xtable(tbl_cat_iucn, type = "latex"),
      file = paste(tmp, "tbl_cat_iucn.tex", sep = "/"))

print(xtable(tbl_cat_iucn_ref, type = "latex"),
      file = paste(tmp, "tbl_cat_iucn_ref.tex", sep = "/"))

#Export to S3 storage

##List of files to save in the temp folder
files <- list.files(tmp, full.names = TRUE)
##Add each file in the bucket (same foler for every file in the temp)
for(f in files) 
  {
  cat("Uploading file", paste0("'", f, "'"), "\n")
  aws.s3::put_object(file = f, 
                     bucket = "projet-afd-eva-ap/DesStat/IUCN", 
                     region = "", show_progress = TRUE)
  }

#Erase the files in the temp directory

do.call(file.remove, list(list.files(tmp, full.names = TRUE)))
```

#### IUCN categories by countries and regions


```r
#Build the distribution of IUCN categories at country level ...
data_iucn_ctry = table(data_stat_nodupl$iucn_des,
                       data_stat_nodupl$iso3) %>%
  #Create a table with all iucn categories for each country, and compute the frequencies in percent
  prop.table(2) %>%
  as.data.frame() %>%
  mutate(Freq = round(Freq, 3)*100) %>%
  pivot_wider(names_from = Var2, values_from = Freq) %>%
  rename("iucn_des" = "Var1")

#... and regional level
data_iucn_reg = table(data_stat_nodupl$iucn_des,
                       data_stat_nodupl$direction_regionale) %>%
    #Create a table with all iucn categories for each country, and compute the frequencies in percent
  prop.table(2) %>%
  as.data.frame() %>%
  mutate(Freq = round(Freq, 3)*100) %>%
  pivot_wider(names_from = Var2, values_from = Freq) %>%
  rename("iucn_des" = "Var1")
```


```r
#Saving figures

tmp = paste(tempdir(), "fig", sep = "/")

print(xtable(data_iucn_ctry, type = "latex"),
      file = paste(tmp, "tbl_iucn_ctry.tex", sep = "/"))
print(xtable(data_iucn_reg, type = "latex"),
      file = paste(tmp, "tbl_iucn_reg.tex", sep = "/"))

#Export to S3 storage

##List of files to save in the temp folder
files <- list.files(tmp, full.names = TRUE)
##Add each file in the bucket (same foler for every file in the temp)
for(f in files) 
  {
  cat("Uploading file", paste0("'", f, "'"), "\n")
  aws.s3::put_object(file = f, 
                     bucket = "projet-afd-eva-ap/DesStat/IUCN", 
                     region = "", show_progress = TRUE)
  }

#Erase the files in the temp directory

do.call(file.remove, list(list.files(tmp, full.names = TRUE)))
```

### Ecosystems (excluding non-referenced PAs)

#### Proportion of PAs by marine or terrestrial areas


```r
#Build datasets
data_eco = data_stat_nodupl %>%
  #subset non-referencded PAs (have NA ecosysteme)
  subset(is.na(marine) == FALSE) %>%
  mutate(marine = as.factor(marine))
data_eco$ecosyst_en = fct_recode(data_eco$marine, 
                              "Terrestrial"="0", 
                              "Coastal"="1", 
                              "Marine"="2")
data_eco$ecosyst_fr = fct_recode(data_eco$marine, 
                              "Terrestre"="0", 
                              "Côtier"="1", 
                              "Marin"="2")

data_eco_hist = data_eco %>%
  group_by(ecosyst_en, ecosyst_fr) %>%
  summarize(n = n(),
            freq = round(n/nrow(data_eco), 1)*100) %>%
  ungroup()

tbl_eco_world_fr = data_eco_hist %>%
  select(c(ecosyst_fr, n, freq)) %>%
  rename("Ecosystème" = "ecosyst_fr",
         "Nombre d'AP" = "n",
         "Proportion d'AP(%)" = "freq")

tbl_eco_world_en = data_eco_hist %>%
  select(c(ecosyst_en, n, freq)) %>%
  rename("Ecosystem" = "ecosyst_en",
         "Number of PAs" = "n",
         "Share of PAs(%)" = "freq")


#Histogram in share (in French)
hist_eco_shr_fr = ggplot(data_eco_hist, 
                     aes(x = ecosyst_fr, y = freq, fill = ecosyst_fr)) %>%
+ geom_bar(width = 0.50, fill= "#3182BD", stat="identity") %>%
+ geom_text(aes(label = round(n, 1), y = freq), 
        vjust = -0.1, color="black",
        size=3.5) %>%
+ labs(title = "Proportion d'aires protégées par type d'écosystème \n(hors AP non-référencées)",
       subtitle = paste("Echantillon :", sum(data_eco_hist$n), "aires protégées. Nombre d'aires indiqué sur les barres."),
         x = "Type d'écosystème",
         y = "Proportion d'aires protégées(%)") %>%
  + theme(legend.position = "bottom",
      legend.key = element_rect(fill = "white"),
      plot.title = element_text(size = 14, face = "bold"), 
      axis.text.x = element_text(angle = 0,size=10, hjust = .5, vjust = .6),
      axis.title.x = element_text(margin = margin(t = 10)),
      panel.background = element_rect(fill = 'white', colour = 'white', 
                                      linewidth = 0.5, linetype = 'solid'),
      panel.grid.major = element_line(colour = 'grey90', linetype = 'solid'),
      panel.grid.minor = element_line(colour = 'grey90', linetype = 'solid'),
      plot.caption = element_text(color = 'grey50', size = 8.5, face = 'plain'))
hist_eco_shr_fr
# 



#Histogram in share (in English)
hist_eco_shr_en = ggplot(data_eco_hist, 
                     aes(x = ecosyst_en, y = freq, fill = ecosyst_en)) %>%
+ geom_bar(width = 0.50, fill= "#3182BD", stat="identity") %>%
+ geom_text(aes(label = round(n, 1), y = freq), 
        vjust = -0.1, color="black",
        size=3.5) %>%
+ labs(title = "Proportion of protected areas by ecosystem type \n(excluding non-references PAs)",
       subtitle = paste("Sample :", sum(data_eco_hist$n), "protected areas. Number of areas indicated above."),
         x = "Ecosystem type",
         y = "Proportion of protected areas(%)") %>%
  + theme(legend.position = "bottom",
      legend.key = element_rect(fill = "white"),
      plot.title = element_text(size = 14, face = "bold"), 
      axis.text.x = element_text(angle = 0,size=10, hjust = .5, vjust = .6),
      axis.title.x = element_text(margin = margin(t = 10)),
      panel.background = element_rect(fill = 'white', colour = 'white', 
                                      linewidth = 0.5, linetype = 'solid'),
      panel.grid.major = element_line(colour = 'grey90', linetype = 'solid'),
      panel.grid.minor = element_line(colour = 'grey90', linetype = 'solid'),
      plot.caption = element_text(color = 'grey50', size = 8.5, face = 'plain'))
#hist_eco_shr_en


  
#Histogram in number (in French)
hist_eco_n_fr = ggplot(data_eco_hist, 
                     aes(x = ecosyst_fr, y = n, fill = ecosyst_fr)) %>%
+ geom_bar(width = 0.50, fill= "#3182BD", stat="identity") %>%
+ labs(title = "Proportion d'aires protégées par type d'écosystème \n(hors AP non-référencées)",
       subtitle = paste("Echantillon :", sum(data_eco_hist$n), "aires protégées"),
         x = "Type d'écosystème",
         y = "Proportion d'aires protégées") %>%
  + theme(legend.position = "bottom",
      legend.key = element_rect(fill = "white"),
      plot.title = element_text(size = 14, face = "bold"), 
      axis.text.x = element_text(angle = 0,size=10, hjust = .5, vjust = .6),
      axis.title.x = element_text(margin = margin(t = 10)),
      panel.background = element_rect(fill = 'white', colour = 'white', 
                                      linewidth = 0.5, linetype = 'solid'),
      panel.grid.major = element_line(colour = 'grey90', linetype = 'solid'),
      panel.grid.minor = element_line(colour = 'grey90', linetype = 'solid'),
      plot.caption = element_text(color = 'grey50', size = 8.5, face = 'plain'))
hist_eco_n_fr


#Histogram in number (in English)
hist_eco_n_en = ggplot(data_eco_hist, 
                     aes(x = ecosyst_en, y = n, fill = ecosyst_en)) %>%
+ geom_bar(width = 0.50, fill= "#3182BD", stat="identity") %>%
+ labs(title = "Proportion of protected areas by ecosystem type \n(excluding non-referenced PAs)",
       subtitle = paste("Sample :", sum(data_eco_hist$n), "protected areas"),
         x = "Ecosystem type",
         y = "Proportion of protected areas") %>%
  + theme(legend.position = "bottom",
      legend.key = element_rect(fill = "white"),
      plot.title = element_text(size = 14, face = "bold"), 
      axis.text.x = element_text(angle = 0,size=10, hjust = .5, vjust = .6),
      axis.title.x = element_text(margin = margin(t = 10)),
      panel.background = element_rect(fill = 'white', colour = 'white', 
                                      linewidth = 0.5, linetype = 'solid'),
      panel.grid.major = element_line(colour = 'grey90', linetype = 'solid'),
      panel.grid.minor = element_line(colour = 'grey90', linetype = 'solid'),
      plot.caption = element_text(color = 'grey50', size = 8.5, face = 'plain'))
#hist_eco_n_en

#Pie chart (in French)
pie_eco_fr = ggplot(data_eco_hist, 
                     aes(x = "", y = freq, fill = ecosyst_fr)) %>%
+ geom_bar(width = 1, stat = "identity",color="white") %>%
+ geom_label(aes(x=1.3, label = paste0(freq, "%")), 
             color = "black", position = position_stack(vjust = 0.55), 
             size=2.5, show.legend = FALSE) %>%
+ coord_polar("y", start=0) %>%
+ labs(title = "Proportion d'aires protégées par type d'écosystème \n(hors AP non-référencées)",
       subtitle = paste("Echantillon :", sum(data_eco_hist$n), "aires protégées"),
         x = "Type d'écosystème",
         y = "Proportion d'aires protégées") %>%
  + scale_fill_brewer(name = "Ecosystème", palette="Paired") %>%
  + theme_void()
#pie_eco_fr


#Histogram in number (in English)
pie_eco_en = ggplot(data_eco_hist, 
                     aes(x = "", y = n, fill = ecosyst_en)) %>%
+ geom_bar(width = 1, stat = "identity",color="white") %>%
+ geom_label(aes(x=1.3, label = paste0(freq, "%")), 
             color = "black", position = position_stack(vjust = 0.55), 
             size=2.5, show.legend = FALSE) %>%
+ coord_polar("y", start=0) %>%
+ labs(title = "Proportion of protected areas by ecosystem type \n(exluding non-referenced PAs)",
       subtitle = paste("Sample :", sum(data_eco_hist$n), "protected areas"),
         x = "Ecosystem type",
         y = "Proportion of protected areas") %>%
  + scale_fill_brewer(name = "Ecosystem", palette="Paired") %>%
  + theme_void()
pie_eco_en
```


```r
#Saving figures

tmp = paste(tempdir(), "fig", sep = "/")


ggsave(paste(tmp, "hist_eco_shr_fr.png", sep = "/"),
       plot = hist_eco_shr_fr,
       device = "png",
       height = 6, width = 9)
ggsave(paste(tmp, "hist_eco_shr_en.png", sep = "/"),
       plot = hist_eco_shr_en,
       device = "png",
       height = 6, width = 9)
ggsave(paste(tmp, "hist_eco_n_fr.png", sep = "/"),
       plot = hist_eco_n_fr,
       device = "png",
       height = 6, width = 9)
ggsave(paste(tmp, "hist_eco_n_en.png", sep = "/"),
       plot = hist_eco_n_en,
       device = "png",
       height = 6, width = 9)
ggsave(paste(tmp, "pie_eco_fr.png", sep = "/"),
       plot = pie_eco_fr,
       device = "png",
       height = 6, width = 9)
ggsave(paste(tmp, "pie_eco_en.png", sep = "/"),
       plot = pie_eco_en,
       device = "png",
       height = 6, width = 9)

print(xtable(tbl_eco_world_fr, type = "latex"), 
      file = paste(tmp, "tbl_ecosyst_fr.tex", sep = "/"))
print(xtable(tbl_eco_world_en, type = "latex"), 
      file = paste(tmp, "tbl_ecosyst_en.tex", sep = "/"))

#Export to S3 storage

##List of files to save in the temp folder
files <- list.files(tmp, full.names = TRUE)
##Add each file in the bucket (same foler for every file in the temp)
count = 0
for(f in files) 
{
  count = count+1
  cat("Uploading file", paste0(count, "/", length(files), " '", f, "'"), "\n")
  aws.s3::put_object(file = f, 
                     bucket = "projet-afd-eva-ap/DesStat/ecosysteme", 
                     region = "", show_progress = TRUE)
  }

#Erase the files in the temp directory

do.call(file.remove, list(list.files(tmp, full.names = TRUE)))
```

### Distribution of PAs across countries and regions

#### Statistics at country level


```r
data_distrib_ctry = data_stat_nodupl %>%
  group_by(pays, iso3) %>%
  summarize(n = n(),
            freq = round(n/nrow(data_stat_nodupl), 1)*100) %>%
  ungroup() 


data_distrib_ctry_top = data_distrib_ctry %>%
  subset(freq >= 5)

#Histogram in share (in French) for top countries
hist_distrib_ctry_top_shr_fr = ggplot(data_distrib_ctry_top, 
                     aes(x = reorder(pays, -freq), y = freq, fill = pays)) %>%
+ geom_bar(width = 0.50, fill= "#3182BD", stat="identity") %>%
+ geom_text(aes(label = round(n, 1), y = freq), 
        vjust = -0.1, color="black",
        size=3.5) %>%
+ labs(title = "Les pays abritant le plus d'aires protégées",
       subtitle = paste("Echantillon :", sum(data_distrib_ctry$n), "aires protégées. Nombre d'aires indiqué à la base du graphique"),
         x = "Pays",
         y = "Proportion d'aires protégées(%)") %>%
  + theme(legend.position = "bottom",
      legend.key = element_rect(fill = "white"),
      plot.title = element_text(size = 14, face = "bold"), 
      axis.text.x = element_text(angle = 0,size=10, hjust = .5, vjust = .6),
      axis.title.x = element_text(margin = margin(t = 10)),
      panel.background = element_rect(fill = 'white', colour = 'white', 
                                      linewidth = 0.5, linetype = 'solid'),
      panel.grid.major = element_line(colour = 'grey90', linetype = 'solid'),
      panel.grid.minor = element_line(colour = 'grey90', linetype = 'solid'),
      plot.caption = element_text(color = 'grey50', size = 8.5, face = 'plain'))
hist_distrib_ctry_top_shr_fr
# 


#Histogram in number (in French)
hist_distrib_ctry_n_fr = ggplot(data_distrib_ctry, 
                     aes(x = reorder(pays, -n), y = n, fill = pays)) %>%
+ geom_bar(width = 0.50, fill= "#3182BD", stat="identity") %>%
+ geom_text(aes(label = round(n, 1), y = n), 
        vjust = -0.1, color="black",
        size=3.5) %>%
+ labs(title = "Proportion d'aires protégées par pays",
       subtitle = paste("Echantillon :", sum(data_distrib_ctry$n), "aires protégées. Nombre d'aires indiqué sur chaque barre"),
         x = "Pays",
         y = "Proportion d'aires protégées(%)") %>%
  + theme(legend.position = "bottom",
      legend.key = element_rect(fill = "white"),
      plot.title = element_text(size = 14, face = "bold"), 
      axis.text.x = element_text(angle = 45,size=10, hjust = .5, vjust = .6),
      axis.title.x = element_text(margin = margin(t = 10)),
      panel.background = element_rect(fill = 'white', colour = 'white', 
                                      linewidth = 0.5, linetype = 'solid'),
      panel.grid.major = element_line(colour = 'grey90', linetype = 'solid'),
      panel.grid.minor = element_line(colour = 'grey90', linetype = 'solid'),
      plot.caption = element_text(color = 'grey50', size = 8.5, face = 'plain'))
hist_distrib_ctry_n_fr
# 
```


```r
#Saving figures

tmp = paste(tempdir(), "fig", sep = "/")

ggsave(paste(tmp, "hist_distrib_ctry_top_shr_fr.png", sep = "/"),
       plot = hist_distrib_ctry_top_shr_fr,
       device = "png",
       height = 6, width = 9)
ggsave(paste(tmp, "hist_distrib_ctry_n_fr.png", sep = "/"),
       plot = hist_distrib_ctry_n_fr,
       device = "png",
       height = 6, width = 9)

#Export to S3 storage

##List of files to save in the temp folder
files <- list.files(tmp, full.names = TRUE)
##Add each file in the bucket (same foler for every file in the temp)
count = 0
for(f in files) 
{
  count = count+1
  cat("Uploading file", paste0(count, "/", length(files), " '", f, "'"), "\n")
  aws.s3::put_object(file = f, 
                     bucket = "projet-afd-eva-ap/DesStat/distribution", 
                     region = "", show_progress = TRUE)
  }

#Erase the files in the temp directory

do.call(file.remove, list(list.files(tmp, full.names = TRUE)))
```

#### Statistics at region level


```r
data_distrib_reg = data_stat_nodupl %>%
  group_by(direction_regionale) %>%
  summarize(n = n(),
            freq = round(n/nrow(data_stat_nodupl), 3)*100) %>%
  ungroup() %>%
  mutate(region = gsub("Dr ", "", direction_regionale),
         .after = "direction_regionale")


data_distrib_reg_top = data_distrib_reg %>%
  subset(freq >= 5)

#Histogram in share (in French) for top countries
hist_distrib_reg_top_shr_fr = ggplot(data_distrib_reg_top, 
                     aes(x = reorder(region, -freq), y = freq, fill = region)) %>%
+ geom_bar(width = 0.50, fill= "#3182BD", stat="identity") %>%
+ geom_text(aes(label = round(n, 1), y = freq), 
        vjust = -0.1, color="black",
        size=3.5) %>%
+ labs(title = "Les directions régionales finançant le plus d'aires protégées",
       subtitle = paste("Echantillon :", sum(data_distrib_reg$n), "aires protégées. Nombre d'aires indiqué sur chaque barre."),
         x = "Direction régionale",
         y = "Proportion d'aires protégées(%)") %>%
  + theme(legend.position = "bottom",
      legend.key = element_rect(fill = "white"),
      plot.title = element_text(size = 14, face = "bold"), 
      axis.text.x = element_text(angle = 0,size=10, hjust = .5, vjust = .6),
      axis.title.x = element_text(margin = margin(t = 10)),
      panel.background = element_rect(fill = 'white', colour = 'white', 
                                      linewidth = 0.5, linetype = 'solid'),
      panel.grid.major = element_line(colour = 'grey90', linetype = 'solid'),
      panel.grid.minor = element_line(colour = 'grey90', linetype = 'solid'),
      plot.caption = element_text(color = 'grey50', size = 8.5, face = 'plain'))
hist_distrib_reg_top_shr_fr


#Histogram in number (in French)
hist_distrib_reg_n_fr = ggplot(data_distrib_reg, 
                     aes(x = reorder(direction_regionale, -n), y = n, fill = direction_regionale)) %>%
+ geom_bar(width = 0.50, fill= "#3182BD", stat="identity") %>%
+ geom_text(aes(label = round(n, 1), y = n), 
        vjust = -0.1, color="black",
        size=3.5) %>%
+ labs(title = "Proportion d'aires protégées par région",
       subtitle = paste("Echantillon :", sum(data_distrib_ctry$n), "aires protégées. Nombre d'aires indiqué sur chaque barre."),
         x = "Direction régionale",
         y = "Proportion d'aires protégées(%)") %>%
  + theme(legend.position = "bottom",
      legend.key = element_rect(fill = "white"),
      plot.title = element_text(size = 14, face = "bold"), 
      axis.text.x = element_text(angle = 45,size=10, hjust = .5, vjust = .6),
      axis.title.x = element_text(margin = margin(t = 10)),
      panel.background = element_rect(fill = 'white', colour = 'white', 
                                      linewidth = 0.5, linetype = 'solid'),
      panel.grid.major = element_line(colour = 'grey90', linetype = 'solid'),
      panel.grid.minor = element_line(colour = 'grey90', linetype = 'solid'),
      plot.caption = element_text(color = 'grey50', size = 8.5, face = 'plain'))
hist_distrib_reg_n_fr
```


```r
#Saving figures

tmp = paste(tempdir(), "fig", sep = "/")

ggsave(paste(tmp, "hist_distrib_reg_top_shr_fr.png", sep = "/"),
       plot = hist_distrib_reg_top_shr_fr,
       device = "png",
       height = 6, width = 9)
ggsave(paste(tmp, "hist_distrib_reg_n_fr.png", sep = "/"),
       plot = hist_distrib_reg_n_fr,
       device = "png",
       height = 6, width = 9)

#Export to S3 storage

##List of files to save in the temp folder
files <- list.files(tmp, full.names = TRUE)
##Add each file in the bucket (same foler for every file in the temp)
count = 0
for(f in files) 
{
  count = count+1
  cat("Uploading file", paste0(count, "/", length(files), " '", f, "'"), "\n")
  aws.s3::put_object(file = f, 
                     bucket = "projet-afd-eva-ap/DesStat/distribution", 
                     region = "", show_progress = TRUE)
  }

#Erase the files in the temp directory

do.call(file.remove, list(list.files(tmp, full.names = TRUE)))
```

### Surface of PAs

#### Distribution of surfaces


```r
tbl_distrib_area_no0 = summary(
  filter(data_stat_nodupl, superficie != 0)$superficie) %>%
  format(scientific = FALSE, big.mark = " ") %>%
  as.array() %>%
  t() %>%
  as.data.frame() %>%
  select(-c("1st Qu.","3rd Qu."))
```

#### Average surface in countries and regions


```r
#Distribution of PA WITH SURFACE >0 across countries and regions
##Country
data_distrib_no0_ctry = data_stat_nodupl %>%
  filter(superficie != 0) %>%
  group_by(iso3, pays) %>%
  summarize(n = n(),
            freq = round(n/nrow(data_stat_nodupl), 1)*100) %>%
  ungroup()

##Region
data_distrib_no0_dr = data_stat_nodupl %>%
  filter(superficie != 0) %>%
  group_by(direction_regionale) %>%
  summarize(n = n(),
            freq = round(n/nrow(data_stat_nodupl), 1)*100) %>%
  ungroup() %>%
  mutate(region = gsub("Dr ", "", direction_regionale),
         .after = "direction_regionale")

#By country..
tbl_area_avg_ctry = data_distrib_no0_ctry %>%
  select(-freq) %>%
  left_join(pa_area_ctry, by = "iso3") %>%
  select(-c(iso3, sprfc_tot_km2, tot_area_int)) %>%
  mutate(sprfc_avg_noint_km2 = format(sprfc_tot_noint_km2/n, big.mark = " ", scientific = FALSE, digits = 1),
         sprfc_tot_noint_km2 = format(sprfc_tot_noint_km2, big.mark  = " ", scientific = FALSE, digits = 1),
         )
names(tbl_area_avg_ctry) = c("Pays", "Nombre d'AP", "Superficie totale (km2)", "Superficie moyenne (km2)")

#By region
tbl_area_avg_dr = data_distrib_no0_dr %>%
  select(-freq) %>%
  left_join(pa_area_dr, by = "direction_regionale") %>%
  select(-c(sprfc_tot_km2, tot_area_int, direction_regionale)) %>%
  mutate(sprfc_avg_noint_km2 = format(sprfc_tot_noint_km2/n, big.mark = " ", scientific = FALSE, digits = 1),
         sprfc_tot_noint_km2 = format(sprfc_tot_noint_km2, big.mark  = " ", scientific = FALSE, digits = 1),
         )
names(tbl_area_avg_dr) = c("Direction régionale", "Nombre d'AP", "Superficie totale (km2)", "Superficie moyenne (km2)")
```


```r
#Saving figures

tmp = paste(tempdir(), "fig", sep = "/")

print(xtable(tbl_distrib_area_no0, type = "latex"), 
      file = paste(tmp, "tbl_distrib_area_no0.tex", sep = "/"))
print(xtable(tbl_area_avg_ctry, type = "latex"), 
      file = paste(tmp, "tbl_area_avg_ctry.tex", sep = "/"))
print(xtable(tbl_area_avg_dr, type = "latex"), 
      file = paste(tmp, "tbl_area_avg_dr.tex", sep = "/"))

#Export to S3 storage

##List of files to save in the temp folder
files <- list.files(tmp, full.names = TRUE)
##Add each file in the bucket (same foler for every file in the temp)
count = 0
for(f in files) 
{
  count = count+1
  cat("Uploading file", paste0(count, "/", length(files), " '", f, "'"), "\n")
  aws.s3::put_object(file = f, 
                     bucket = "projet-afd-eva-ap/DesStat/surface", 
                     region = "", show_progress = TRUE)
  }

#Erase the files in the temp directory

do.call(file.remove, list(list.files(tmp, full.names = TRUE)))
```

### Temporal evolution

#### Number of PAs


```r
data_time_range = data.frame(annee = 
                              c(min(data_stat_nodupl$annee_octroi):max(data_stat_nodupl$annee_octroi))
)


data_time_n = data_stat_nodupl %>%
  group_by(annee_octroi) %>%
  summarize(n = n()) %>%
  full_join(data_time_range, by = c("annee_octroi" = "annee")) %>%
  mutate(n = case_when(is.na(n)~0, TRUE~n)) %>%
  arrange(annee_octroi) %>%
  mutate(n_cum = cumsum(n)) 

#Number of PAs funded by year 
fig_n_pa = ggplot(data_time_n,
                  aes(x = factor(annee_octroi), y = n)) %>%
  + geom_bar(stat = 'identity', fill = "#3182BD") %>% 
  + geom_text(aes(y = n, label = ifelse(n == 0, NA, n)), 
              color = "black", size=4, vjust = -0.3) %>%
  + labs(title = "Nombre d'aires protégées appuyées par année",
         subtitle = paste("Echantillon :", sum(data_time_n$n), "aires protégées"),
         x = "Année",
         y = "Nombre d'aires protégées") %>%
  + theme(legend.position = "bottom",
      legend.key = element_rect(fill = "white"),
      plot.title = element_text(size = 14, face = "bold"), 
      axis.text.x = element_text(angle = 45,size=10, hjust = .5, vjust = .6),
      axis.title.x = element_text(margin = margin(t = 10)),
      panel.background = element_rect(fill = 'white', colour = 'white', 
                                      linewidth = 0.5, linetype = 'solid'),
      panel.grid.major = element_line(colour = 'grey90', linetype = 'solid'),
      panel.grid.minor = element_line(colour = 'grey90', linetype = 'solid'),
      plot.caption = element_text(color = 'grey50', size = 8.5, face = 'plain'))
fig_n_pa



#Cumulative number over time
fig_ncum_pa = ggplot(data_time_n,
                  aes(x = factor(annee_octroi), y = n_cum)) %>%
  # + geom_point(color = "#3182BD", size = 1.5) %>%
  # + geom_line(color = "#3182BD", size = 1) %>% 
  + geom_bar(stat = 'identity', fill = "#3182BD") %>%
  + geom_text(aes(y = n_cum, label = n_cum), 
              color = "black", size=4, vjust = -0.3) %>%
  + labs(title = "Evolution cumulée du nombre d'aires protégées appuyées par l'AFD",
         subtitle = paste("Echantillon :", sum(data_time_n$n), "aires protégées"),
         x = "Année",
         y = "Nombre d'aires protégées") %>%
  + theme(legend.position = "bottom",
      legend.key = element_rect(fill = "white"),
      plot.title = element_text(size = 14, face = "bold"), 
      axis.text.x = element_text(angle = 45,size=10, hjust = .5, vjust = .6),
      axis.title.x = element_text(margin = margin(t = 10)),
      panel.background = element_rect(fill = 'white', colour = 'white', 
                                      linewidth = 0.5, linetype = 'solid'),
      panel.grid.major = element_line(colour = 'grey90', linetype = 'solid'),
      panel.grid.minor = element_line(colour = 'grey90', linetype = 'solid'),
      plot.caption = element_text(color = 'grey50', size = 8.5, face = 'plain'))
fig_ncum_pa
```


```r
#Saving figures

tmp = paste(tempdir(), "fig", sep = "/")

ggsave(paste(tmp, "fig_n_pa.png", sep = "/"),
       fig_n_pa,
       device = "png",
       height = 6, width = 9)
ggsave(paste(tmp, "fig_ncum_pa.png", sep = "/"),
       fig_ncum_pa,
       device = "png",
       height = 6, width = 9)



#Export to S3 storage

##List of files to save in the temp folder
files <- list.files(tmp, full.names = TRUE)
##Add each file in the bucket (same foler for every file in the temp)
count = 0
for(f in files) 
{
  count = count+1
  cat("Uploading file", paste0(count, "/", length(files), " '", f, "'"), "\n")
  aws.s3::put_object(file = f, 
                     bucket = "projet-afd-eva-ap/DesStat/time_evolution", 
                     region = "", show_progress = TRUE)
  }

#Erase the files in the temp directory

do.call(file.remove, list(list.files(tmp, full.names = TRUE)))
```

#### Surface of PAs


```r
data_time_range = data.frame(annee = 
                              c(min(data_stat_nodupl$annee_octroi):max(data_stat_nodupl$annee_octroi))
)


data_time_area = data_stat_nodupl %>%
  group_by(annee_octroi) %>%
  summarize(tot_area_km2 = sum(superficie)) %>%
  left_join(pa_int_yr, by = c("annee_octroi" = "annee_int")) %>%
  full_join(data_time_range, by = c("annee_octroi" = "annee")) %>%
  mutate(tot_area_km2 = case_when(is.na(tot_area_km2)~0, TRUE~tot_area_km2),
         tot_int_km2 = case_when(is.na(tot_int_km2)~0, TRUE~tot_int_km2),
         tot_area_noint_km2 = tot_area_km2 - tot_int_km2) %>%
  arrange(annee_octroi)

#Evolution of area over time
fig_area_pa = ggplot(data_time_area,
                  aes(x = factor(annee_octroi), y = tot_area_noint_km2)) %>%
  + geom_bar(stat = 'identity', fill = "#3182BD") %>%
  + geom_text(aes(y = tot_area_noint_km2, 
                  label = ifelse(tot_area_noint_km2 != 0,
                                 format(tot_area_noint_km2, 
                                        digits = 2, 
                                        scientific = TRUE),
                                 NA)), 
              color = "black", size=3, vjust = -0.3) %>%
  + labs(title = "Evolution de la superficie des aires protégées",
         subtitle = paste("Echantillon :", sum(data_time_n$n), "aires protégées couvrant", format(sum(data_time_area$tot_area_noint_km2), digits = 1, scientific = FALSE, big.mark = " "), "km2"),
         x = "Année",
         y = "Superficie (km2)") %>%
  + theme(legend.position = "bottom",
      legend.key = element_rect(fill = "white"),
      plot.title = element_text(size = 14, face = "bold"), 
      axis.text.x = element_text(angle = 45,size=10, hjust = .5, vjust = .6),
      axis.title.x = element_text(margin = margin(t = 10)),
      panel.background = element_rect(fill = 'white', colour = 'white', 
                                      linewidth = 0.5, linetype = 'solid'),
      panel.grid.major = element_line(colour = 'grey90', linetype = 'solid'),
      panel.grid.minor = element_line(colour = 'grey90', linetype = 'solid'),
      plot.caption = element_text(color = 'grey50', size = 8.5, face = 'plain'))
fig_area_pa



#Cumulative area over time
fig_area_cum_pa = ggplot(data_time_area,
                  aes(x = factor(annee_octroi),
                      y = cumsum(tot_area_noint_km2))) %>%
  # + geom_point(color = "#3182BD", size = 1.5) %>%
  # + geom_line(color = "#3182BD", size = 1) %>% 
  + geom_bar(stat = 'identity', fill = "#3182BD") %>%
  + geom_text(aes(y = cumsum(tot_area_noint_km2), 
                  label = format(cumsum(tot_area_noint_km2), 
                                 digits = 2,
                                 scientific = TRUE)), 
              color = "black", size=3, vjust = -0.3) %>%
  + labs(title = "Evolution cumulée de la superficie des aires protégées",
          subtitle = paste("Echantillon :", sum(data_time_n$n), "aires protégées couvrant", format(sum(data_time_area$tot_area_noint_km2), digits = 1, scientific = FALSE, big.mark = " "), "km2"),
         x = "Année",
         y = "Superficie (km2)") %>%
  + theme(legend.position = "bottom",
      legend.key = element_rect(fill = "white"),
      plot.title = element_text(size = 14, face = "bold"), 
      axis.text.x = element_text(angle = 45,size=10, hjust = .5, vjust = .6),
      axis.title.x = element_text(margin = margin(t = 10)),
      panel.background = element_rect(fill = 'white', colour = 'white', 
                                      linewidth = 0.5, linetype = 'solid'),
      panel.grid.major = element_line(colour = 'grey90', linetype = 'solid'),
      panel.grid.minor = element_line(colour = 'grey90', linetype = 'solid'),
      plot.caption = element_text(color = 'grey50', size = 8.5, face = 'plain'))
fig_area_cum_pa
```


```r
#Saving figures

tmp = paste(tempdir(), "fig", sep = "/")

ggsave(paste(tmp, "fig_area_pa.png", sep = "/"),
       fig_area_pa,
       device = "png",
       height = 6, width = 9)
ggsave(paste(tmp, "fig_area_cum_pa.png", sep = "/"),
       fig_area_cum_pa,
       device = "png",
       height = 6, width = 9)

#Export to S3 storage

##List of files to save in the temp folder
files <- list.files(tmp, full.names = TRUE)
##Add each file in the bucket (same foler for every file in the temp)
count = 0
for(f in files) 
{
  count = count+1
  cat("Uploading file", paste0(count, "/", length(files), " '", f, "'"), "\n")
  aws.s3::put_object(file = f, 
                     bucket = "projet-afd-eva-ap/DesStat/time_evolution", 
                     region = "", show_progress = TRUE)
  }

#Erase the files in the temp directory

do.call(file.remove, list(list.files(tmp, full.names = TRUE)))
```

### Governance


```r
#Table of the governance type distribution
##English version
data_gov_en = data_stat_nodupl %>%
  mutate(gov_type = case_when(gov_type == "" ~ "Not referenced",
                              TRUE ~ gov_type)) %>%
  group_by(gov_type) %>%
  summarize(n = n()) %>%
  mutate(n_tot = sum(n),
         freq = round(n/n_tot*100,1)) %>%
  select(-n_tot) %>%
  arrange(-freq)

tbl_gov_en = data_gov_en
names(tbl_gov_en) = c("Governance","Number of PAs","Share of PAs (%)")


##French Version
data_gov_fr = data_stat_nodupl %>%
  mutate(gov_type = case_when(gov_type == "" ~ "Non référencée",
                              gov_type == "Collaborative governance" ~ "Gouvernance collaborative",
                              gov_type == "Federal or national ministry or agency" ~ "Ministère ou agence, fédérale ou nationale",
                              gov_type == "Federal or national ministry or agency" ~ "Ministère ou agence, fédérale ou nationale",
                              gov_type == "Government-delegated management" ~ "Gestion déléguée par le gouvernement",
                              gov_type == "Indigenous peoples" ~ "Peuples indigènes",
                              gov_type == "Joint governance" ~ "Gouvernance conjointe",
                              gov_type == "Local communities" ~ "Communautés locales",
                              gov_type == "Non-profit organisations" ~ "Organisations non-lucratives",
                              gov_type == "Not Reported" ~ "Non rapportée",
                              gov_type == "Sub-national ministry or agency" ~ "Ministère ou agence sous-nationale",
                              TRUE ~ gov_type)) %>%
  group_by(gov_type) %>%
  summarize(n = n()) %>%
  mutate(n_tot = sum(n),
         freq = round(n/n_tot*100,1)) %>%
  select(-n_tot) %>%
  arrange(-freq)

tbl_gov_fr = data_gov_fr
names(tbl_gov_fr) = c("Gouvernance","Nombre d'AP","Proportion (%)")



#PAs with nureported or unreferenced governance types are removed
##Tables
###English
data_gov_knwn_en = data_stat_nodupl %>%
  mutate(gov_type = case_when(gov_type == "" ~ "Not referenced",
                              TRUE ~ gov_type)) %>%
  filter(gov_type != "Not Reported" & gov_type != "Not referenced") %>%
  group_by(gov_type) %>%
  summarize(n = n()) %>%
  mutate(n_tot = sum(n),
         freq = round(n/n_tot*100,1)) %>%
  select(-n_tot) %>%
  arrange(-freq)

tbl_gov_knwn_en = data_gov_knwn_en
names(tbl_gov_knwn_en) = c("Governance","Number of PAs","Share of PAs (%)")

###French
data_gov_knwn_fr = data_stat_nodupl %>%
  mutate(gov_type = case_when(gov_type == "" ~ "Non référencée",
                              gov_type == "Collaborative governance" ~ "Gouvernance collaborative",
                              gov_type == "Federal or national ministry or agency" ~ "Ministère ou agence, fédérale ou nationale",
                              gov_type == "Federal or national ministry or agency" ~ "Ministère ou agence, fédérale ou nationale",
                              gov_type == "Government-delegated management" ~ "Gestion déléguée par le gouvernement",
                              gov_type == "Indigenous peoples" ~ "Peuples indigènes",
                              gov_type == "Joint governance" ~ "Gouvernance conjointe",
                              gov_type == "Local communities" ~ "Communautés locales",
                              gov_type == "Non-profit organisations" ~ "Organisations non-lucratives",
                              gov_type == "Not Reported" ~ "Non rapportée",
                              gov_type == "Sub-national ministry or agency" ~ "Ministère ou agence sous-nationale",
                              TRUE ~ gov_type)) %>%
  filter(gov_type != "Non référencée" & gov_type != "Non rapportée") %>%
  group_by(gov_type) %>%
  summarize(n = n()) %>%
  mutate(n_tot = sum(n),
         freq = round(n/n_tot*100,1)) %>%
  select(-n_tot) %>%
  arrange(-freq)

tbl_gov_knwn_fr = data_gov_knwn_fr
names(tbl_gov_knwn_fr) = c("Gouvernance","Nombre d'AP","Proportion (%)")
 

##Pie charts
###English
pie_gov_knwn_en = 
  ggplot(data_gov_knwn_en, 
       aes(x="", y= freq, fill= gov_type)) %>%
  + geom_bar(width = 1, stat = "identity", color="white") %>%
  + geom_label(aes(x=1.3, 
                   label = paste0(format(freq, digits = 2), "%")), 
               color = "black", 
               position = position_stack(vjust = 0.55), 
               size=2.5, show.legend = FALSE) %>%
  + coord_polar("y", start=0) %>%
  + labs(title = "Governance type of PAs, except not referenced/reported PAs",
         subtitle = paste("Sample :", sum(data_gov_knwn_en$n), "PAs")) %>%
  + scale_fill_brewer(name = "Governance", palette="Paired") %>%
  + theme_void()



###French
pie_gov_knwn_fr =
  ggplot(data_gov_knwn_fr, 
       aes(x="", y= freq, fill= gov_type)) %>%
  + geom_bar(width = 1, stat = "identity", color="white") %>%
  + geom_label(aes(x=1.3, 
                   label = paste0(format(freq, digits = 2), "%")), 
               color = "black", 
               position = position_stack(vjust = 0.55), 
               size=2.5, show.legend = FALSE) %>%
  + coord_polar("y", start=0) %>%
  + labs(title = "Gouvernance, hors aires protégées avec une gouvernance \nnon-rapportée/référencée",
         subtitle = paste("Echantillon :", sum(data_gov_knwn_fr$n), "aires protégées")) %>%
  + scale_fill_brewer(name = "Gouvernance", palette="Paired") %>%
  + theme_void()
pie_gov_knwn_fr
```


```r
#Saving figures

tmp = paste(tempdir(), "fig", sep = "/")

print(xtable(tbl_gov_en, type = "latex"),
      file = paste(tmp, "tbl_gov_en.tex", sep  ="/"))
print(xtable(tbl_gov_fr, type = "latex"),
      file = paste(tmp, "tbl_gov_fr.tex", sep  ="/"))
print(xtable(tbl_gov_knwn_en, type = "latex"),
      file = paste(tmp, "tbl_gov_knwn_en.tex", sep  ="/"))
print(xtable(tbl_gov_knwn_fr, type = "latex"),
      file = paste(tmp, "tbl_gov_knwn_fr.tex", sep  ="/"))

ggsave(paste(tmp, " pie_gov_knwn_en.png", sep = "/"),
       plot =  pie_gov_knwn_en,
       device = "png",
       height = 6, width = 9)
ggsave(paste(tmp, "pie_gov_knwn_fr.png", sep = "/"),
       plot = pie_gov_knwn_fr,
       device = "png",
       height = 6, width = 9)

#Export to S3 storage

##List of files to save in the temp folder
files <- list.files(tmp, full.names = TRUE)
##Add each file in the bucket (same foler for every file in the temp)
count = 0
for(f in files) 
{
  count = count+1
  cat("Uploading file", paste0(count, "/", length(files), " '", f, "'"), "\n")
  aws.s3::put_object(file = f, 
                     bucket = "projet-afd-eva-ap/DesStat/gouvernance", 
                     region = "", show_progress = TRUE)
  }

#Erase the files in the temp directory

do.call(file.remove, list(list.files(tmp, full.names = TRUE)))
```