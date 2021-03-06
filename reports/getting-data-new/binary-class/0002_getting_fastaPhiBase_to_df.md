Getting the fasta file from Phi base to dataframe format
========================================================

In this report, it will be shown how we can split the fasta file from
Phi-base to dataframe format that is more accessible.

``` r
# Change the fasta data from Phi-base to dataframe
phi_base_fasta <- readLines("../../../data/phi-base-current-data/phi-base_current.fasta") %>%
  # Get rid of header
  .[-1] %>%
  # Collapse into a single string
  stringr::str_c(collapse = "#") %>%
  # Get rid of "blank lines"
  stringr::str_replace_all("##", "#") %>%
  # Split by "#>" character
  stringr::str_split("#>") %>%
  unlist() %>%
  # Split into different columns
  stringr::str_split("#") %>% 
  unlist() %>%
  # Create data frame
  matrix(nrow = 7) %>%
  t() %>% 
  as.data.frame() 

phi_base_fasta <- phi_base_fasta %>% 
  `colnames<-`(c("ProteinID",
                 "PHIMolConnID",
                 "Gene",
                 "PathogenID", 
                 "Pathogenspecies", 
                 "MutantPhenotype", 
                 "Sequence")) %>% 
  dplyr::mutate(ProteinID = stringr::str_replace(ProteinID, ">", ""))

# Save the data into RDS object so that it can be loaded easily
# saveRDS(phi_base_fasta, "phi_base_fasta.RDS")
```
