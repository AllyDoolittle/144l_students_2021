2021\_Bac\_Abundance
================
Ally Doolittle
10/20/2021

# Goal

**Objective:** Visualize our data and identify exponential growth.

# Intro to R Markdown

Create a new code chunk: PC - Ctrl + alt + i

Load packages that we’ll need to analyze our data

``` r
library(tidyverse)
library(readxl)
library(lubridate)
```

We can toggle on/off warnings in chunks if we don’t want them in our
final markdown file

# Import Data

``` r
excel_sheets("144L_2021_BactAbund.xlsx")
```

    ## [1] "Metadata" "Data"

``` r
metadata <- read_excel("144L_2021_BactAbund.xlsx", sheet = "Metadata")

glimpse(metadata)
```

    ## Rows: 80
    ## Columns: 16
    ## $ Experiment           <chr> "144L_2021", "144L_2021", "144L_2021", "144L_2021~
    ## $ Location             <chr> "Goleta Pier", "Goleta Pier", "Goleta Pier", "Gol~
    ## $ Temperature          <dbl> 19, 19, 19, 19, 19, 19, 19, 19, 19, 19, 19, 19, 1~
    ## $ Depth                <dbl> 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1~
    ## $ Bottle               <chr> "A", "A", "A", "A", "A", "A", "A", "A", "A", "A",~
    ## $ Timepoint            <dbl> 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 0, 1, 2, 3, 4, 5, 6~
    ## $ Treatment            <chr> "Control", "Control", "Control", "Control", "Cont~
    ## $ Target_DOC_Amendment <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0~
    ## $ Inoculum_L           <dbl> 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2~
    ## $ Media_L              <dbl> 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5~
    ## $ Datetime             <chr> "2021-10-04T16:00", "2021-10-05T08:00", "2021-10-~
    ## $ TOC_Sample           <lgl> TRUE, FALSE, FALSE, FALSE, TRUE, FALSE, FALSE, FA~
    ## $ Cell_Sample          <lgl> TRUE, TRUE, TRUE, TRUE, TRUE, TRUE, TRUE, TRUE, T~
    ## $ DAPI_Sample          <lgl> TRUE, FALSE, FALSE, FALSE, TRUE, FALSE, FALSE, FA~
    ## $ DNA_Sample           <lgl> TRUE, FALSE, FALSE, FALSE, TRUE, FALSE, FALSE, FA~
    ## $ Nutrient_Sample      <lgl> TRUE, FALSE, FALSE, FALSE, FALSE, FALSE, FALSE, F~

``` r
#unique(metadata$Bottle)
#unique(metadata$Treatment)

data <- read_excel("144L_2021_BactAbund.xlsx", sheet = "Data")

glimpse(data)
```

    ## Rows: 72
    ## Columns: 5
    ## $ Bottle       <chr> "A", "A", "A", "A", "A", "A", "A", "A", "A", "B", "B", "B~
    ## $ Timepoint    <dbl> 0, 1, 2, 3, 4, 5, 6, 7, 8, 0, 1, 2, 3, 4, 5, 6, 7, 8, 0, ~
    ## $ all_cells_uL <chr> "901.48904752420799", "4302.9300548457404", "3944.9457004~
    ## $ LNA_cells_uL <chr> "653.047184033284", "768.27893058466896", "937.2441189022~
    ## $ HNA_cells_uL <chr> "248.441863490923", "3534.65112426107", "3007.70158156820~

``` r
joined <- left_join(metadata, data) #attach data to metadata
#joins right dataset to the left one by using variables that are the same across the two dataframes

#glimpse(joined)
```

# Prepare Data

we will convert the Date and Time column values from characters to
dates, add columns with time elapsed for each treatment, and convert to
cells/L because it will help us match up with the TOC data later. We
will then subset the data for variables of interest and drop NA values.

To do this we are going to use **piping**. Piping is an operation that
allows us to write more efficient code. The way that we’ll use it here
is to manipulate our data sequentially. The pipe operator “%&gt;%, which
basically says like”first do one thing to the data, THEN do this other
thing." (with the %&gt;% operator taking the place of the word THEN in
this scenario). Every cell that we invoke with an additional pipe is
going to take place on the variable (dataframe) that we specify at the
beginning

``` r
cells <- joined %>%
  mutate(Datetime = ymd_hm(Datetime), #splits apart Datetime as specified
         cells_L = as.numeric(all_cells_uL) * 1000000) %>%
  group_by(Treatment, Bottle) %>%
#group our dataset so that we can calculate the time elapsed properly
  mutate(interv = interval(first(Datetime), Datetime),
         s = as.numeric(interv),
         hours = s/3600,
         days = hours/24) %>%
  ungroup() %>%
  select(Experiment:DNA_Sample, cells_L, hours, days) %>%
  drop_na(cells_L)
```

# Plot Growth Curves

We will plot a growth curve for each bottle. Need cell abundance and
days data.

First let’s set some aesthetics for our plot.

``` r
#assign hex colors to our different treatments

custom.colors <- c("Control" = "#3399FF", "Kelp Exudate" = "#339900", "Kelp Exudate_Nitrate_Phosphate" = "#CC0033", "Glucose_Nitrate_Phosphate" = "#FFCC33")
unique(metadata$Treatment)


#assign levels to control what order things appear in the legend
levels <- c("Control", "Kelp Exudate", "Kelp Exudate_Nitrate_Phosphate", "Glucose_Nitrate_Phosphate")

#now let's use a handy package, ggplot to visualize our data.

cells %>%
  mutate(dna = ifelse(DNA_Sample == T, "*", NA)) %>%
  ggplot(aes(x=days, y=cells_L, group = interaction(Treatment, Bottle))) +
  geom_line(aes(color = factor(Treatment, levels = levels)), size = 1) +
  geom_point(aes(fill = factor(Treatment, levels = levels)), size = 3, color = "black", shape = 21) +
  geom_text(aes(label = dna), size = 12, color = "#E41A1C") +
  labs(x = "Days", y = expression(paste("Cells, L"^-1)), fill = "") +
  guides(color = "none") +
  scale_color_manual(values = custom.colors) +
  scale_fill_manual(values = custom.colors) +
  #facet_grid(rows = "Treatment", scales = "free")
  #scales not all even across treatment plots, get rid of scales to make all the same value
  theme_light()
```

![](2021_Bac_Abundance_files/figure-gfm/unnamed-chunk-3-1.png)<!-- -->

# Next Steps

We can calculate:

-total change in cells from initial condition to the end of the
experiment. -specific growth rate as the slope of ln(abundance) v time
during exponential growth phase. -doubling time as ln(2) divided by the
specific growth rate. -mean of each of these parameters across each
treatment

1st, we need to determine **where** exponential growth is ocurring in
each of our bottles, if it does. To do this, we’ll plot ln(abundance) vs
time.

# Identify Exponential Phase of Growth of Our Remineralization Experiments

**NOTE about logarithms in R**

log(x) gives the natural log of x, not log base 10 log10(x) gives the
log base 10 log2(x) gives log base 2

``` r
ln_cells <- cells %>%
  group_by(Treatment, Bottle) %>%
  mutate(ln_cells = log(cells_L), 
         diff_ln_cells = ln_cells - lag(ln_cells, default = first(ln_cells)))
```

Now let’s plot our newly calculated data!

``` r
ln_cells %>%
  mutate(dna = ifelse(DNA_Sample == T, "*", NA)) %>%
  ggplot(aes(x=days, y=diff_ln_cells, group = interaction(Treatment, Bottle))) +
  geom_line(aes(color = factor(Treatment, levels = levels)), size = 1) +
  geom_point(aes(fill = factor(Treatment, levels = levels)), size = 3, color = "black", shape = 21) +
  geom_text(aes(label = dna), size = 12, color = "#E41A1C") +
  labs(x = "Days", y = expression(paste("Delta*ln cells, L"^-1)), fill = "") +
  guides(color = "none") +
  scale_color_manual(values = custom.colors) +
  scale_fill_manual(values = custom.colors) + facet_wrap("Bottle", ncol =2)
```

![](2021_Bac_Abundance_files/figure-gfm/unnamed-chunk-5-1.png)<!-- -->

``` r
  #facet_grid(rows = "Treatment", scales = "free")
 
  theme_light()
```

Exponential growth seems to be occurring right at the beginning of the
experiment between 0-1 days for most of the bottles.

Let’s try plotting ln\_cells to see if that can help us identify
exponential growth in the control a little better.

``` r
ln_cells %>%
  mutate(dna = ifelse(DNA_Sample == T, "*", NA)) %>%
  ggplot(aes(x=days, y=ln_cells, group = interaction(Treatment, Bottle))) +
  geom_line(aes(color = factor(Treatment, levels = levels)), size = 1) +
  geom_point(aes(fill = factor(Treatment, levels = levels)), size = 3, color = "black", shape = 21) +
  geom_text(aes(label = dna), size = 12, color = "#E41A1C") +
  labs(x = "Days", y = expression(paste("ln cells, L"^-1)), fill = "") +
  guides(color = "none") +
  scale_color_manual(values = custom.colors) +
  scale_fill_manual(values = custom.colors) + facet_wrap("Bottle", ncol =2)
```

![](2021_Bac_Abundance_files/figure-gfm/unnamed-chunk-6-1.png)<!-- -->

``` r
  #facet_grid(rows = "Treatment", scales = "free")
  theme_bw()
```

ln data showing exponential growth at the beginning of the experiment
for most of the bottles.

**Summary:** Prepared data sets for plotting then learned ggplot.
Plotted growth curves for our remineralization experiment. Also added
colors and layers to the plots to more efficiently visualize the data.
Plotted ln and ln(abundance) to identify where exponential growth
occurred.
