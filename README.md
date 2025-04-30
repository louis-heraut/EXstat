# EXstat [<img src="figures/flower_hex.png" align="right" width=160 height=160 alt=""/>](https://github.com/louis-heraut/CARD)

<!-- badges: start -->
[![R-CMD-check](https://github.com/louis-heraut/EXstat/actions/workflows/R-CMD-check.yaml/badge.svg)](https://github.com/louis-heraut/EXstat/actions/workflows/R-CMD-check.yaml)
[![Lifecycle: stable](https://img.shields.io/badge/lifecycle-stable-green)](https://lifecycle.r-lib.org/articles/stages.html)
![](https://img.shields.io/github/last-commit/louis-heraut/EXstat)
[![Contributor Covenant](https://img.shields.io/badge/Contributor%20Covenant-2.1-4baaaa.svg)](code_of_conduct.md) 
<!-- badges: end -->

**EXstat** is a R package which provide an efficient and simple solution to aggregate and analyze the stationarity of time series.

EXstat is highly customizable, but the **CARD** extension provides a simpler solution for performing common hydroclimatic aggregations. See the  [CARD documentation]() of the [CARD package](https://github.com/louis-heraut/CARD) for advance understanding.

This project was carried out for National Research Institute for Agriculture, Food and the Environment (Institut National de Recherche pour l’Agriculture, l’Alimentation et l’Environnement, [INRAE](https://agriculture.gouv.fr/inrae-linstitut-national-de-recherche-pour-lagriculture-lalimentation-et-lenvironnement) in french) and is at the core of [MAKAHO](https://github.com/louis-heraut/MAKAHO) which won the [2024 Open Science Research Data Award](https://www.enseignementsup-recherche.gouv.fr/fr/remise-des-prix-science-ouverte-des-donnees-de-la-recherche-2024-98045) in the “Creating the Conditions for Reuse” category.


## Installation
For latest development version
``` R
remotes::install_github("louis-heraut/EXstat")
```


## Documentation
### Extraction process
Based on [dplyr](https://dplyr.tidyverse.org/), **input data** format is a `tibble` of at least a column of **date**, some columns of **numeric value** and one or more **character** columns for names of time series in order to identify them uniquely. Thus it is possible to have a `tibble` with multiple time series which can be grouped by their names.

For example, we can use the following `tibble` : 

``` R
library(dplyr)

# Date
Start = as.Date("1972-01-01")
End = as.Date("2020-12-31")
Date = seq.Date(Start, End, by="day")

# Value to analyse
set.seed(100)
X = seq(1, length(Date))/1e4 + runif(length(Date), -100, 100)
X[as.Date("2000-03-01") <= Date & Date <= as.Date("2000-09-30")] = NA

# Creation of tibble
data = tibble(Date=Date, ID="serie A", X=X)
```

Which looks like that :
``` R
> data
# A tibble: 17,898 × 3
   Date       ID           X
   <date>     <chr>    <dbl>
 1 1972-01-01 serie A -38.4 
 2 1972-01-02 serie A -48.5 
 3 1972-01-03 serie A  10.5 
 4 1972-01-04 serie A -88.7 
 5 1972-01-05 serie A  -6.29
 6 1972-01-06 serie A  -3.25
 7 1972-01-07 serie A  62.5 
 8 1972-01-08 serie A -25.9 
 9 1972-01-09 serie A   9.31
10 1972-01-10 serie A -65.9 
# ℹ 17,888 more rows
# ℹ Use `print(n = ...)` to see more rows
```

The process of **variable extraction** (for example the yearly mean of time series) is realised with the `process_extraction()` function.
Minimum arguments are :
* Input `data` described above
* The function `funct` (for example `mean`) you want to use. Arguments of the chosen function can be passed to this extraction process and the function can be previously defined.

Some of the optional arguments are :
* `period` A vector of two dates (or two unambiguous character strings that can be coerced to dates) to restrict the period of analysis. As an example, it can be `c("1950-01-01", "2020-12-31")` to select data from the 1st January of 1950 to the end of December of 2020. The default option is `period=NULL`, which considers all available data for each time serie.
* `time_step` A character string specifying the time step of the variable extraction process. Possible values are :
  - "year" for a value per year
  - "month" for a value for each month of the year (so 12 values if at least a full year is given)
  - "year-month" for a value for each month of each year (so 12 times the number of given year values at the end)
  - "season" for a value for each season of th year (so by default 4 values)
  - "year-season" for a value for each season of each year (so by default 4 times the number of given year values at the end)
  - "yearday" for one value per day of the year (so 365 values at the end if at least a full year is given... but more than one year seems obviously more interesting)
  - "none" if you want to extract a unique value for the whole time serie
* `sampling_period` A character string or a vector of two character strings that will indicate how to sample the data for each time step defined by `time_step`. Hence, the choice of this argument needs to be link with the choice of the time step. For example, for a yearly extraction so if `time_step` is set to `"year"`, `sampling_period` needs to be formated as `%m-%d` (a month - a day of the year) in order to indicate the start of the sampling of data for the current year. More precisly, if `time_step="year"` and `sampling_period="03-19"`, `funct` will be apply on every data from the 3rd march of each year to the 2nd march of the following one. In this way, it is possible to create a sub-year sampling with a vector of two character strings as `sampling_period=c("02-01", "07-31")` in order to process data only if the date is between the 1st february and the 31th jully of each year.

More parameters are available, for example, to :
* handle missing values,
* use suffixes to simplify expressions, and
* manage variables related to seasonality.

In this way
``` R
dataEX = process_extraction(data=data,
                            funct=max,
                            funct_args=list("X", na.rm=TRUE),
                            time_step="year",
                            sampling_period=c("05-01",
                                              "11-30"),
                            period=c(as.Date("1990-01-01"),
                                     as.Date("2020-12-31")))
```

will perform a yearly extraction of the maximum value between may and november, from the 1th march of 1990 to the 31th october of 2020, ignoring `NA` values.

The output is also a `tibble` with a column of **date**, of **character** for the name of time series and a **numerical** column with the extracted variable from the time series.

``` R
> dataEX
# A tibble: 31 × 3
   ID      Date           X
   <chr>   <date>     <dbl>
 1 serie A 1990-05-01 100. 
 2 serie A 1991-05-01 101. 
 3 serie A 1992-05-01 100. 
 4 serie A 1993-05-01  99.9
 5 serie A 1994-05-01  99.0
 6 serie A 1995-05-01 100. 
 7 serie A 1996-05-01 100. 
 8 serie A 1997-05-01 101. 
 9 serie A 1998-05-01  99.6
10 serie A 1999-05-01 101. 
# ℹ 21 more rows
# ℹ Use `print(n = ...)` to see more rows
```

Other examples of more complex cases are available in the package documentation. Try starting with 
``` R
library(EXstat)
?EXstat
```


### Trend analyse
The stationarity analyse is computed with the `process_trend()` function on the extracted data `dataEX`. The **statistical test** used here is the **Mann-Kendall test**[^mann][^kendall].

Hence, the following expression

``` R
trendEX = process_trend(data=dataEX)
```

produces the result below

```
# A tibble: 1 × 12
  ID      variable_en level H          p      a     b period_trend
  <chr>   <chr>       <dbl> <lgl>  <dbl>  <dbl> <dbl> <list>      
1 serie A X             0.1 TRUE  0.0958 0.0260  99.3 <date [2]>

  mean_period_trend a_normalise a_normalise_min a_normalise_max
  <lgl>                   <dbl>           <dbl>           <dbl>
1 NA                     0.0260          0.0260          0.0260
```

It is a `tibble` which precises, among other information, the name of the time serie the **p value** and `a` the **Theil-Sen's slope**[^theil][^sen], for each row.

Finaly, as the **p value** is below 0.1, the previous time serie shows an **increasing linear trend** which can be represented by the equation `Y = 0.0260*X + b` with a **type I error** of 10 % or a **trust** of 90 %. 


## FAQ
📬 — **I would like an upgrade / I have a question / Need to reach me**  
Feel free to [open an issue](https://github.com/louis-heraut/EXstat/issues) ! I’m actively maintaining this project, so I’ll do my best to respond quickly.  
I’m also reachable on my institutional INRAE [email](mailto:louis.heraut@inrae.fr?subject=%5BEXstat%5D) for more in-depth discussions.

🛠️ — **I found a bug**  
- *Good Solution* : Search the existing issue list, and if no one has reported it, create a new issue !  
- *Better Solution* : Along with the issue submission, provide a minimal reproducible code sample.  
- *Best Solution* : Fix the issue and submit a pull request. This is the fastest way to get a bug fixed.

🚀 — **Want to contribute ?**  
If you don't know where to start, [open an issue](https://github.com/louis-heraut/EXstat/issues).

If you want to try by yourself, why not start by also [opening an issue](https://github.com/louis-heraut/EXstat/issues) to let me know you're working on something ? Then:

- Fork this repository  
- Clone your fork locally and make changes (or even better, create a new branch for your modifications)
- Push to your fork and verify everything works as expected
- Open a Pull Request on GitHub and describe what you did and why
- Wait for review
- For future development, keep your fork updated using the GitHub “Sync fork” functionality or by pulling changes from the original repo (or even via remote upstream if you're comfortable with Git). Otherwise, feel free to delete your fork to keep things tidy ! 

If we’re connected through work, why not reach out via email to see if we can collaborate more closely on this repo by adding you as a collaborator !

Refer to the [CONTRIBUTING file](.github/CONTRIBUTING.md) for contribution guidelines and help.


## Code of Conduct
Please note that this project is released with a [Contributor Code of Conduct](CODE_OF_CONDUCT.md). By participating in this project you agree to abide by its terms.

[^mann]: [Mann, H. B. (1945). Nonparametric tests against trend. Econometrica: Journal of the econometric society, 245-259.](https://www.jstor.org/stable/1907187)
[^kendall]: [Kendall, M.G. (1975) Rank Correlation Methods. 4th Edition, Charles Grifin, London.](https://www.scirp.org/reference/ReferencesPapers.aspx?ReferenceID=2223266)
[^theil]: [Theil, H. (1950). A rank-invariant method of linear and polynomial regression analysis. Indagationes mathematicae, 12(85), 173.](https://ir.cwi.nl/pub/8270/8270D.pdf)
[^sen]: [Sen, P. K. (1968). Estimates of the regression coefficient based on Kendall's tau. Journal of the American statistical association, 63(324), 1379-1389.](https://www.tandfonline.com/doi/abs/10.1080/01621459.1968.10480934)
