Getting data as control set
===========================

![Getting the Control
data](/data/images/getting-control-data-flowchart.png)

### Making a list of all of the name of species of non-effector data

``` r
bacteria <- c("Acinetobacter baumannii", 
"Aeromonas hydrophila", 
"Aeromonas salmonicida", 
"Brucella abortus", 
"Burkholderia glumae", 
"Burkholderia pseudomallei", 
"Campylobacter jejuni", 
"Citrobacter rodentium", 
"Clavibacter michiganensis", 
"Coxiella burnetii", 
"Cystobacter fuscus", 
"Edwardsiella ictaluri", 
"Erwinia amylovora", 
"Escherichia coli", 
"Francisella tularensis", 
"Helicobacter pylori", 
"Legionella pneumophila",  
"Listeria monocytogenes", 
"Mycobacterium tuberculosis", 
"Pantoea stewartii", 
"Pseudomonas aeruginosa", 
"Pseudomonas cichorii", 
"Pseudomonas savastanoi", 
"Pseudomonas syringae", 
"Ralstonia solanacearum", 
"Salmonella enterica", 
"Shigella flexneri", 
"Staphylococcus aureus", 
"Vibrio parahaemolyticus", 
"Xanthomonas axonopodis", 
"Xanthomonas campestris", 
"Xanthomonas oryzae", 
"Xylella fastidiosa", 
"Yersinia enterocolitica", 
"Yersinia pestis", 
"Yersinia pseudotuberculosis")

fungi <- c("Beauveria bassiana",
"Blumeria graminis",
"Botrytis cinerea",
"Cercospora apii",
"Cercospora beticola",
"Colletotrichum orbiculare",
"Dothistroma septosporum",
"Fusarium oxysporum",
"Leptosphaeria maculans",
"Magnaporthe oryzae",
"Melampsora lini",
"Parastagonospora nodorum",
"Passalora fulva",
"Penicillium expansum",
"Pseudocercospora fuligena",
"Puccinia striiformis",
"Rhynchosporium commune",
"Verticillium dahliae",
"Ustilago maydis",
"Zymoseptoria tritici")

oomycetes <- c("Hyaloperonospora arabidopsidis", 
"Phytophthora cactorum", 
"Phytophthora capsici", 
"Phytophthora infestans", 
"Phytophthora parasitica", 
"Phytophthora sojae", 
"Pythium aphanidermatum", 
"Plasmopara halstedii", 
"Phytophthora megakarya")

others <- c("Globodera rostochiensis",
"Heterodera glycines",
"Macrosiphum euphorbiae",
"Toxoplasma gondii")
```

``` r
library(tidyverse)

phi_base <- data.table::fread("../../../data/getting-data-old/phi-base-main.csv", header = TRUE)

effector_matched_ids <- phi_base %>%
  tibble::rowid_to_column() %>% 
  dplyr::filter_all(any_vars(str_detect(., 'plant avirulence determinant'))) %>% 
  select(rowid) %>% 
  unlist()

# filter all of the data with 'plant avirulence determinant' information
non_effector_matched_data <- phi_base %>%
  tibble::rowid_to_column() %>% 
  # dplyr::filter_all(all_vars(!str_detect(., 'plant avirulence determinant')))
  dplyr::filter(!(rowid %in% effector_matched_ids)) %>% 
  select(-rowid)

# select the species on the  list above 
non_effector_matched_data <- non_effector_matched_data %>% 
  dplyr::filter(`Pathogen species` %in% c(bacteria, fungi, oomycetes, others)) %>% 
  select(`Pathogen species`, `Protein ID`) %>% 
  mutate(., category = with(., case_when(
  (`Pathogen species` %in% bacteria) ~ "bacteria",
  (`Pathogen species` %in% fungi) ~ "fungi",
  (`Pathogen species` %in% oomycetes) ~ "oomycetes",
  TRUE ~ "Others")))

drop <- c(636, 2477, 2479, 2489)

non_effector_with_uniq_IDs <- non_effector_matched_data %>% 
  filter(`Protein ID` != "no data found") %>% 
  group_by(`Protein ID`, category) %>% 
  slice(1) %>% 
  ungroup() %>% 
  dplyr::filter(!(row_number() %in% drop))

list <- duplicated(non_effector_with_uniq_IDs$`Protein ID`)


# take all of the preotein IDS, find the unique values, and remove all of the rows that the IDs are not available
non_effector_IDs <- non_effector_matched_data  %>% 
  dplyr::select(`Protein ID`) %>% 
  unique() %>% 
  dplyr::filter_all(any_vars(!str_detect(., 'no data found')))

write.table(non_effector_IDs, "../../../data/getting-data-old/0002-getting-control-sets/phi-base-old/phi_base_non_effector_IDs.csv", quote = FALSE, row.names = FALSE, col.names = FALSE)
```

Getting the FASTA data
----------------------

``` r
library(seqinr)

# Read FASTA file
uniprot_fasta_data <- seqinr::read.fasta("../../../data/getting-data-old/0002-getting-control-sets/phi-base-old/uniprot_non_effector.fasta")
```

``` r
uniprot_fasta_data[[1]]
```

``` r
parse_fasta_data_uniprot <- function(file_path) {
  # Read FASTA file
  fasta_data <- seqinr::read.fasta(file_path)
  # Number of entries
  num_data <- fasta_data %>% length()


  # Create empty data frame
  parsed_data <- data.frame(
    protein_id = rep(NA, num_data),
    pathogen = rep(NA, num_data),
    sequence = rep(NA, num_data)
  )

  for (i in 1:num_data) {
    # Read 'Annot' attribute and parse the string between 'OS=' and 'OX='
    pathogen <- fasta_data[[i]] %>%
      attr("Annot") %>%
      sub(".*OS= *(.*?) *OX=.*", "\\1", .)

    protein_id <- fasta_data[[i]] %>%
      attr("name") %>%
      strsplit("\\|") %>%
      unlist() %>%
      .[[2]]

    # Concatenate the vector of the sequence into a single string
    sequence <- fasta_data[[i]] %>%
      as.character() %>%
      toupper() %>%
      paste(collapse = "")

    # Input values into data frame
    parsed_data[i,] <- cbind(protein_id, pathogen, sequence)
  }

  return(parsed_data)
}
```

``` r
uniprot_noneffector_path <- "../../../data/getting-data-old/0002-getting-control-sets/phi-base-old/uniprot_non_effector.fasta"

noneffector_data <- parse_fasta_data_uniprot(uniprot_noneffector_path)
```

Using protein IDS above, we can retrive the data from `uniprot`:

``` r
# read the mapping result from phi-base unique proteinIDs to uniprot -- retrieved in .csv
uniprot_raw <- data.table::fread("../../../data/getting-data-old/0002-getting-control-sets/phi-base-old/uniprot_non_effector_data_raw_corrected.csv", fill = TRUE, sep = ",")

# rename a column of protein IDs obtained from phi-base, and the entry IDs from
uniprot_raw <- uniprot_raw %>%
  rename(phibase_ids = `yourlist:M201906098471C63D39733769F8E060B506551E121CB19AI`, uniprot_ids = `Entry`)

# uniprot_raw %>% 
#   dplyr::filter(stringr::str_detect(phibase_ids, "(\\,)"))

# getting the ids which are different between phi-base and uniprot
diff_phi_uniprot_ids <- uniprot_raw %>%
  dplyr::filter(!(phibase_ids %in% intersect(uniprot_raw[['phibase_ids']], uniprot_raw[['uniprot_ids']]))) %>%
  select(`phibase_ids`, `uniprot_ids`, `Organism`)
```

``` r
noneffector_data_with_proteinID_correction <- noneffector_data %>%
  mutate(protein_id = ifelse(protein_id == diff_phi_uniprot_ids[['uniprot_ids']], 
                             diff_phi_uniprot_ids[['phibase_ids']], protein_id)) 

# %>%
  # select(`protein_id`, `pathogen_short`, `sequence`)
noneffector_data_with_proteinID_correction <- noneffector_data %>%
  left_join(
    diff_phi_uniprot_ids %>% dplyr::rename(protein_id = uniprot_ids) %>% dplyr::select(-Organism),
    by = "protein_id"
  ) %>%
  dplyr::mutate(protein_id = ifelse(is.na(phibase_ids), protein_id, phibase_ids)) %>% 
  dplyr::select(-phibase_ids)


# replace the pathogen name as the one 
control_set_data <- noneffector_data_with_proteinID_correction %>% 
  left_join(non_effector_with_uniq_IDs %>% dplyr::rename(protein_id = `Protein ID`), 
            by = "protein_id") %>% 
  dplyr::select(protein_id, sequence, `Pathogen species`, category)

write.csv(control_set_data, "../../../data/getting-data-old/0002-getting-control-sets/phi-base-old/control_set_data.csv", row.names = FALSE)
```

``` r
library(tidyverse)
control_set_data <- data.table::fread("../../../data/getting-data-old/0002-getting-control-sets/phi-base-old/control_set_data.csv") %>% 
  mutate(label = as.factor(0))
```

``` r
# print the number of data
data.frame(table(control_set_data$category))
```

    ##        Var1 Freq
    ## 1  bacteria 1244
    ## 2     fungi  991
    ## 3 oomycetes   30
    ## 4    Others   29

Since we do not have enough data for oomycetes, we will get sample data
from NCBI

``` r
parse_fasta_data_ncbi <- function(file_path) {
  # Read FASTA file
  fasta_data <- seqinr::read.fasta(file_path)
  # Number of entries
  num_data <- fasta_data %>% length()


  # Create empty data frame
  parsed_data <- data.frame(
    protein_id = rep(NA, num_data),
    protein_fun = rep(NA, num_data),
    pathogen = rep(NA, num_data),
    sequence = rep(NA, num_data)
  )

  for (i in 1:num_data) {
    # Read 'Annot' attribute and parse the string between 'OS=' and 'OX='
    pathogen <- fasta_data[[i]] %>%
      attr("Annot") %>%
      sub(".*\\[ *(.*?) *\\].*", "\\1", .)

    protein_id <- fasta_data[[i]] %>%
      attr("name")

    protein_fun <- fasta_data[[i]] %>%
      attr("Annot") %>%
      stringr::str_remove(protein_id) %>%
      sub(".*> *(.*?) *\\[.*", "\\1", .)

    # Concatenate the vector of the sequence into a single string
    sequence <- fasta_data[[i]] %>%
      as.character() %>%
      toupper() %>%
      paste(collapse = "")

    # Input values into data frame
    parsed_data[i,] <- cbind(protein_id, protein_fun, pathogen, sequence)
  }

  return(parsed_data)
}
```

``` r
library(tidyverse)
ncbi_oomycetes_path <- "../../../data/getting-data-old/ncbi-oomycetes.fasta"
ncbi_oomycetes_parsed <- parse_fasta_data_ncbi(ncbi_oomycetes_path) %>% 
  select(protein_id, sequence, pathogen) %>% 
  rename(`Pathogen species` = pathogen) %>% 
  mutate(category = "oomycetes", label = as.factor(0))

control_set_data_complete <- control_set_data %>% 
  bind_rows(., ncbi_oomycetes_parsed)

write.csv(control_set_data_complete, "../../../data/getting-data-old/0002-getting-control-sets/phi-base-old/control_set_data_complete.csv", row.names = FALSE)
```

``` r
# getting information about the complete data sets

library(tidyverse)
control_set_data <- data.table::fread("../../../data/getting-data-old/0002-getting-control-sets/phi-base-old/control_set_data_complete.csv")

# print the number of data
data.frame(table(control_set_data$category))
```

    ##        Var1 Freq
    ## 1  bacteria 1244
    ## 2     fungi  991
    ## 3 oomycetes  125
    ## 4    Others   29

Encoding the data
-----------------

The encoding process is done in a separated file script.

``` python
import numpy as np

x_control_data = np.load('../../../data/getting-data-old/0002-getting-control-sets/phi-base-old/control-data/x_control.npy')
print("Shape of the ecoded sequence data:", x_control_data.shape)
```

    ## Shape of the ecoded sequence data: (2375, 2500, 20)
