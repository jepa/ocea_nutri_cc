---
title: "Post DBEM data analysis"
author: "Juliano Palacios Abrantes, Gabriel Reygondeau, William W.L. Cheung"
date: "2023-04-03"
output: pdf_document
---



```{r setup, results='hide', message=FALSE, echo = F}
library(MyFunctions)

packages <- c(
  "readxl", # Read dataframe
  "tidyverse", # for all data wrangling and ggplot
  "janitor", # for data cleaning
  "sf", #Spatial analysis 
  "sp", #Spatial analysis 
  "rnaturalearth", # For maps
  "doParallel",
  "foreach"
)

my_lib(packages)

# Fix new updates of sf package
sf::sf_use_s2(use_s2 = FALSE)
```
# Overall

This script uses the DBEM runs with the CMIP6 Earth System Models (GFDL, IPSL, MPIS) under SSP 126 and 585 to estimate the percentage change in each species maximum catch potential (MCP). The calculation is made by species in each EEZ and the output is a percentage change by the end and the mid 21st century relative to today.

# Select species

Here we will select the species to model based on the countries selected

## Get EEZs

```{r select_eezs, eval = T, echo = T}

selected_countries <- c(
  "Cook Islands", # Yes
  "Fiji", # Yes
  "Kiribati (Gilbert Islands)", # Yes
  "Kiribati (Line Islands)", # Yes
  "Kiribati (Phoenix Islands)", # Yes
  "Marshall Isl.", # Yes
  "Micronesia (Federated States of)", # Yes
  "Nauru", # Yes
  "Niue (New Zealand)", # Yes
  "Palau", # Yes
  "Papua New Guinea", # Yes
  "Samoa", # Yes
  "Solomon Isl.", # Yes
  "Timor Leste", # Yes
  "Tokelau (New Zealand)", # Yes
  "Tonga", # Yes
  "Tuvalu", # Yes
  "Vanuatu", # Yes
  "American Samoa", # Yes
  "Indonesia (Indian Ocean)", # Yes
  "Indonesia (Eastern)", # Yes
  "Indonesia (Central)", # Yes
  "Malaysia (Peninsula West)", # Yes
  "Malaysia (Peninsula East)", # Yes
  "Malaysia (Sabah)", # Yes
  "Malaysia (Sarawak)", # Yes
  "Philippines", # Yes
  "Australia", # Yes
  "New Zealand", # Yes
  "French Polynesia", # Yes
  "Pitcairn (UK)", # Yes
  "New Caledonia (France)", # Yes
  "Wallis & Futuna Isl. (France)" # Yes 
)


length(selected_countries)

# Read SAU EEZs
sf_sau_eez <- MyFunctions::my_sf("SAU") %>% 
  clean_names()

# Get EEZ names
# sf_sau_eez %>%
#   as.data.frame() %>%
#   select(name) %>%
# View()


sf_region <-
  sf_sau_eez %>% 
  filter(name %in% selected_countries)
  
unique(sf_region$name)
  
length(unique(sf_region$name))

# write_csv(sf_region %>% as.data.frame() %>% select(-geometry),
      # "../data/spatial/region_index.csv")

# Visually make sure
sf_region %>%  st_simplify(preserveTopology = TRUE, dTolerance = 1) %>% 
  st_shift_longitude() %>%
  ggplot() +
  geom_sf(aes())

```

## Get speceis list

We selected all species captured since 2006 within the EEZs of the region. Such species represent 66% of the identified species fished since 2006

```{r select_spp, eval = T, echo = T}

# Read in SAU data of after 2006
sau_catch <- read.csv("/Volumes/Enterprise/Data/SAU/SAUCatch_FE_EEZ_FAO_taxa_sector_29May2020.csv") 

# Get species
df_regional_spp <- sau_catch %>% 
  filter(eezid %in% sf_region$eezid,
         year >= 2006) %>% 
  pull(taxon_key) %>% 
  unique()

# Get taxon from witch we have DBEM distributions
dbem_spp <- list.files("/Volumes/Enterprise/Data/Species/Distributions")
dbem_spp <- str_remove(dbem_spp,".csv")
dbem_spp <- str_remove(dbem_spp,"S")

# Read species list
spp_list <- read.csv("/Volumes/Enterprise/Data/SAU/exploited_species_list.csv") %>% 
  janitor::clean_names() %>% 
  filter(taxon_key %in% df_regional_spp,
         taxon_key %in% dbem_spp
         )

# Species percentage 
# 257/392

# Save complete list
# write_csv(spp_list,"../data/species/project_spplist.csv")

# Estimate catches
df_analized_catch <- sau_catch %>% 
  filter(eezid %in% sf_region$eezid,
         year >= 2006) %>% 
  group_by() %>% 
  summarise(total = sum(sumcatch,na.rm = T)) %>% 
  pull(total)

sau_catch %>% 
  filter(eezid %in% sf_region$eezid,
         year >= 2006,
         taxon_key %in% spp_list$taxon_key) %>% 
  group_by() %>% 
  summarise(regional = sum(sumcatch,na.rm = T)/df_analized_catch*100)


```

## Main analysis

We projected the distribution of species using the DBEM coupled with three Earth System Models (ESMs) following two Shared Socioeconomic Pathways (SSPs); SSP 126 representing a low emission / high mitigation scenario and SSP 585 representing a high emission / no mitigation scenario. 

For each species, we projected its future maximum catch potential or "MCP" (a proxy of MSY) within each EEZ from 1951 to 2100. We then determined three time periods from which we estimate the percentage change in MCP. The present represent the average projections from 1995 to 2014 and match the historical data used by the ESMs. The mid of 21st century represents the average MCP projection from 2041 to 2060 and the end of the 21st century represent 2081 to 2100. These two future time periods were chosed to show a shorter and longer time frames affected by climate change. Finally, we estimate the percentage change in MCP ($\Delta{MCP}$) of mid and end of the 21st century relative the the present time frame as:


$$\Delta{MCP} =\frac{MCP_f-MCP_p}{MCP_p}*100$$
Where $MCP_f$ represents the future time periods (mid and end) and $MCP_p$ represents the current time period. As arbritary tules, if $MCP_p$ = 0 & $MCP_f$ > 0 then ($\Delta{MCP}$) = 100, and if $MCP_p$ > 0 & $MCP_f$ < 0 then ($\Delta{MCP}$) = -100.


### Function needed

```{r main_fx, eval = F, echo = T}

mainfx <- function(taxon_key){
  
  to_read <- paste0(taxon_key,"MCP.RData")
  
  for(m in 1:6){
    
    #################################
    # List esm folders
    file_to_read <- paste0(files_path[m],"/",to_read)
    if(file.exists(file_to_read)){
      load(file_to_read)
      
      # Transform it to df
      spp_data <- as.data.frame(sppMCPfnl) %>% 
        rowid_to_column("index") %>% 
        right_join(regional_grid,
                   by = "index")
      colnames(spp_data) <- c("index",(seq(1951,2100,1)),"eez_id")
      rm(sppMCPfnl) # remove data for computing power
      
      # Estimate percentage change
      final_data <- spp_data %>% 
        gather("year","value",`1951`:`2100`) %>% 
        mutate(
          period = ifelse(year < 2014 & year > 1995,"present",
                          ifelse(year >2041 & year <2060,"mid",
                                 ifelse(year > 2081,"end",NA
                                 )
                          )
          )
        ) %>% 
        filter(!is.na(period)) %>% 
        group_by(eez_id,year,period) %>% 
        # Sum total catch per eez per year
        summarise(eez_catch = sum(value, na.rm = T),.groups = "drop") %>% 
        group_by(eez_id,period) %>% 
        # Average yearly data
        summarise(mean_catch = mean(eez_catch,na.rm = T),.groups = "drop") %>% 
        spread(period,mean_catch) %>% 
        mutate(
          per_change_mid = ifelse(present == 0 & mid > 0,100, 
                                  round((mid-present)/present*100)),
          # per_change_mid = round((mid-present)/present*100),
          per_change_end = ifelse(present == 0 & end > 0,100, 
                                  round((end-present)/present*100)),
          # per_change_end = round((end-present)/present*100)
        ) %>% 
        mutate(
          taxon_key = taxon_key,
          esm = ifelse(str_detect(files_path[m],"MPI"),"c6mpi", 
                       str_to_lower(str_sub(files_path[m],49,54)
                       )
          ),
          ssp = ifelse(str_detect(files_path[m],"MPI"),paste0("ssp",
                                                              str_sub(files_path[m],54,55)), 
                       paste0("ssp",str_sub(files_path[m],55,56))
          )
        ) %>% 
        select(taxon_key,eez_id,esm,ssp,present,everything())
      
      # Join data for all three models
      if(m == 1){
        final_output <- final_data
      }else{
        final_output <- bind_rows(final_output,final_data)
      }
      
      file_name <- paste0("/Users/juliano/Data/ocea_nutri_cc/results/",
                          taxon_key,
                          "_per_change.csv")
      
      write_csv(final_output,file_name)
      
    }else{
      return(print(paste("something went wrong with",taxon_key,
                         str_to_lower(str_sub(files_path[m],49,56)))))
    }
  }
  
} # Close function

```

### Control panel

```{r protocol_control_panel, eval = F, echo = T}

# Get species list
ssp_list <- read_csv("../data/species/project_spplist.csv")

par_spp_list <- ssp_list %>% 
  pull(taxon_key)

# Get the regional grid 
region_index <- read_csv("../data/spatial/region_index.csv") 

# Get list of index by EEZ
eez_grid_id <- my_path("Spa","DBEM", "EEZ_CellID.xlsx", read = TRUE)
colnames(eez_grid_id) <- c("eez_id","index")

regional_grid <- eez_grid_id %>% 
  filter(eez_id %in% region_index$eezid)

# Get the DBEM path
files_path <- list.files("/Volumes/Enterprise/Data/dbem/dbem_cmip6/r_data",
                         full.names = T)

# Get ran species
run_spp <- str_sub(list.files("/Users/juliano/Data/ocea_nutri_cc/results/",
                              full.names = F),1,6)

# Run error spp
spp_to_run <- ssp_list %>% 
  filter(!taxon_key %in% run_spp) %>%
  pull(taxon_key)

```

### Run protocol (Parallel)

```{r run_protocol, eval = F, echo = T}
# get the cores from your computer.
# No need for more than 12 cores 
# Never use all cores so you computer doesn't crash
cores <- ifelse(detectCores() > 12, 12, detectCores()-6)  
cl <- makeCluster(cores[1])
registerDoParallel(cl)


run <- foreach(i = 1:length(spp_to_run), .packages = "tidyverse") %dopar% {
  mainfx(spp_to_run[i])
}

stopCluster(cl)
gc()


```

# Test runs

Testing results for a taxon to make sure all EEZs are included

```{r test_result_data, eval = T, echo = T}
#  Note you need to run firt two chunks

# Load result test
result_df <- read_csv("~/Data/ocea_nutri_cc/results/600107_per_change.csv")

unique(result_df$esm) #c6gfdl

unique(result_df$ssp) # "ssp26" "ssp85"

summary(result_df)
```


```{r test_result_map, eval = T, echo = T}

sf_region %>%  
  st_simplify(preserveTopology = TRUE, dTolerance = 1) %>% # Picaso style map to load faster
  st_shift_longitude() %>% # Center projection on Pacific Islands
  rename(eez_id = eezid) %>% # To mathc SAU shapefile
  left_join(result_df) %>% 
  ggplot() +
  geom_sf(
    aes(
      fill = per_change_mid
    )
  ) + facet_wrap(~ssp, ncol = 1) +
  scale_fill_gradient2()
  



```







	
