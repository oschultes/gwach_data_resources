# Randomization tutorial


## Randomization in REDCap

In the UW REDCap instance, there is the option to program randomization
into the project. The below resources provide guidance on setting up the
randomization module in REDCap.

- <https://www.ctsi.ufl.edu/files/2018/12/Setting-Up-the-Randomization-Module-in-REDCap.pdf>

- <https://cri.uchicago.edu/wp-content/uploads/2020/02/REDCap-Randomization-Module.pdf>

REDCap requires the user to upload allocation tables; that is, you must
produce a table of ordered randomized assignments for REDCap to use to
allocate participants to intervention groups. This tutorial walks
through an example of how to programmatically produce an allocation
table for a project with a complex randomization model.

## Producing an allocation table programmatically

We need to produce two allocation tables, one for testing while the
REDCap project is in draft mode and one for implementation when the
project is in production mode. While we can use formulas such as RAND()
to conduct simple randomization in Excel, this is insufficient for more
complex randomization models that incorporate stratified randomization
or randomization blocks of varying sizes. For these cases, we can use
the blockrand package in R to specify the randomization model.

### Arguments

- n specifies the number of random assignments to generate

  - It is best practice to produce many more random assignments than are
    expected to be necessary because it is not possible to change the
    list after the project is in production mode

  - This protects against unforeseen circumstances; for instance, if a
    participant is randomized and later deemed ineligible for
    participation

  - So for instance, if expect to enroll 300 participants, we might
    provide 350 randomized allocations

- num.levels specifies the number of intervention groups

  - In the below example, we are assigning participants into one of two
    intervention groups

- block.sizes specifies the block group size for blocked randomization

  - In this example, we use blocks varying in size from 2 to 12 (double
    the value of the block.sizes argument)

- stratum is used to label which strata the randomization applies to

  - When using stratified randomization, we need to create a separate
    randomization table for each level of the stratified variable

  - In the below example, we produce two tables for two different strata

  - Note that in this example because we expect more enrolled
    participants to belong to strata 2, we produced a longer
    randomization list for strata 2 using the n argument

  - If there is stratification by more than one variable (for instance,
    by site and by a clinical variable), then you would need to produce
    a randomization table for each unique combination of strata

``` r
### setup
pacman::p_load(tidyverse, blockrand)


### development randomization model (testing stage)

set.seed(12345)

dev_strata_1 = blockrand(n = 300,                 # number of random assignments (i.e. individuals)
                         num.levels = 2,          # number of intervention groups
                         stratum = "Strata 1 ",   # specify randomization strata
                         block.sizes = c(1:6))    # block group sizes

dev_strata_2 = blockrand(n = 600, 
                         num.levels = 2,
                         stratum = "Strata 2",
                         block.sizes = c(1:6))
```



Note that because we are using block randomization with blocks of
varying lengths, there may be more than the exact number of assignments
specified depending on the set.seed. The blockrand function will always
produce at least the number of random assignments specified by the n
argument. So for instance, in dev_strata_2, the last randomization block
(block 95) begins at the 599th participant and has a length of 8,
meaning that the list has 606 assignments instead of 600.

``` r
dim(dev_strata_1)
```

    [1] 300   5

``` r
dim(dev_strata_2)
```

    [1] 606   5

After producing a randomization table for each strata, we will merge the
tables and update the formatting to match the template allocation table
produced by REDCap. Here, the variables redcap_randomization_group and
strata_variable_name are placeholders for the variable names in the
downloaded template allocation table. The variables are recoded as
numeric, to match the numeric coding of variables in REDCap.

``` r
dev_random = rbind(dev_strata_1, dev_strata_2)

dev_random = dev_random |>
  select(treatment, stratum) |>
  rename(redcap_randomization_group = treatment, 
         strata_variable_name = stratum) |>
  mutate(redcap_randomization_group = ifelse(redcap_randomization_group=="A", 1, 2),
         strata_variable_name = ifelse(strata_variable_name=="Strata 1", 1, 2))
```

We repeat the procedure using a different set.seed for the production
randomization model, then export both tables as .csv files for uploading
to REDCap.

``` r
### production randomization model (participant randomization)

set.seed(54321)

prod_strata_1 = blockrand(n = 300, 
                            num.levels = 2,
                            # levels = c(1,2)
                            stratum = "HIV+",
                            block.sizes = c(1:6))

prod_strata_2 = blockrand(n = 600, 
                           num.levels = 2,
                           stratum = "HEU",
                           block.sizes = c(1:6))

prod_random = rbind(prod_strata_1, prod_strata_2)

prod_random = prod_random |>
  select(treatment, stratum) |>
  rename(redcap_randomization_group = treatment, scr_inf_hiv_status = stratum) |>
  mutate(redcap_randomization_group = ifelse(redcap_randomization_group=="A", 1, 2),
         scr_inf_hiv_status = ifelse(scr_inf_hiv_status=="HIV+", 1, 2))
```

``` r
### export
write.csv(dev_random, "development_randomization.csv", row.names = FALSE)
write.csv(prod_random, "production_randomization.csv", row.names = FALSE)
```

## Additional tips

- Stratification variable

  - When setting up the randomization model, the stratification variable
    must be categorical and must be assigned to the same form where the
    variable exists

  - When randomizing

    - After clicking the randomization button, the stratification
      variable will appear so the user can verify the value

    - This pop-up question is modifiable, so do not change the value
      unless you know the previously marked value is incorrect

    - It is *very important* to verify that the stratification value is
      correct because after randomizing, the value of the stratification
      variable and the randomization assignment will both be locked
