# enigh2010

This package contains INEGI's National Income and Expenditure Household Survey (ENIGH 2010, new construction) data. It includes the 15 tables which are available at the INEGI portal (<http://www.inegi.org.mx>) and one additional table with AGEB data, which was acquired through a special request to INEGI. As described by INEGI, an AGEB is an Urban Basic Geostatistical Area. It's the territorial extent which constitutes the basic unit of the National Geostatistical Framework. AGEBs are crucial to (approximately) geo-locate the households included in the survey. 

In the following example we will plot commuting expenditure as a percentage of total income, by income quartile. 

## Installation and loading
Install the package with devtools.
```r
devtools::install_github('ricardoochoa/ageb2010')
```
Once that the package is installed, just call it. Please note that in the following examples we will use **ggplot2** and **plyr** as well to plot and merge data, respectively. 
```r
library(enigh2010)
library(ggplot2)
library(plyr)
```
In the following example we will load four specific data sets: 
For transit-related expenditure we will use the **G_diario** table, which contains household daily expense for food, beberages, tobacco and transit. 
For gasoline and diesel expenditure we will use the **Gastos** table, which contains general household expenditure.
For income, we will use the **Ingresos** table, which contains income and capital perceptions for all household members. 
And finally, for the expansion factor we will use the **Hogares** table, which contains dwelling and household characteristics.
```r
data(G_diario)
data(Gastos)
data(Ingresos)
data(Hogares)
```
## Data setup
First of all, we will create simplified versions of the original tables data(G_diario, Ingresos and Hogares), by selecting only the columns that we need. The **transit**, **private** and **income** tables require an extra aggregation step. The variable **clave** in table **Gastos** includes various types of expenses. In our case, we will subset the dataframe to obtain only those that match with F007 Magna gasoline, F008 Premium gasoline, F009 diesel or gas. In other hand, the variable **clave** in table **G_diario** includes, again, several types of daily expenses. In our case, we will subset the data so that it will include only those related to transit: B001 metro or light rail, B002 urban bus, B003 trolley or BRT, B004 vans or microbus B005 taxi and B006 non urban bus and B007 other transit modes.
```r
# transit
transit <- subset(G_diario, clave == "B001" |
                           clave == "B002" |
                           clave == "B003" |
                           clave == "B004" |
                           clave == "B005" |
                           clave == "B006" |
                           clave == "B007")
transit <- 
  aggregate(gas_tri~folioviv + foliohog, 
            data = transit, 
            FUN = sum)

# private
private <- subset(Gastos, clave == "F007" |
                          clave == "F008" |
                          clave == "F009")
private <- 
  aggregate(gas_tri~folioviv + foliohog, 
            data = private, 
            FUN = sum)
            
# commuting
commuting <- 
  aggregate(gas_tri~folioviv + foliohog, 
            data = rbind(transit, private), 
            FUN = sum)

# income
income <- 
  aggregate(ing_tri~folioviv + foliohog, 
            data = Ingresos, 
            FUN = sum)
            
households <- Hogares[c("folioviv", "foliohog", "factor")]

rm(G_diario, Gastos, Hogares, Ingresos, private, transit)
```
Merge all tables by "folioviv" and "foliohog" and use the variable with the expansion factor to expand the data. 

```r
d <- join_all(dfs = list(commuting, income, households), 
              by = c("folioviv", "foliohog"))
d <- subset(d, !is.na(ing_tri))
d <- d[ rep(1:nrow(d), d[ , "factor"]), ]
```
Now we will add few variables. First, lets estimate income quartiles. 
```r
d$group <- cut(d$ing_tri,
               breaks = quantile(x = d$ing_tri, 
                                probs=c(0.00, 0.25, 0.50 ,0.75, 1.00), 
                                        na.rm = T), 
                      include.lowest = TRUE, 
                      labels = c("I", "II", "III", "IV"))
summary(d$group)
```
Now we only need to add a **percent** valiable with the commuting expenditure as a percentage of the total income. 
```r
d$percent <- 100 * d$gas_tri/d$ing_tri
summary(d$percent)
```
Finally, we can plot the data. As can be seen from the figure, the percentage of income spent in commuting increases in lower-income groups.
```r
ggplot() +
  geom_histogram(data = d, 
                 aes(x = percent), 
                 binwidth = 1) +
  xlim(0,50) + theme_bw() + 
  ylab("number of observations") +
  xlab("expenditure as a percentage of total income") +
  facet_grid(group~.)
```  

[[https://github.com/ricardoochoa/enigh2010/blob/master/img/example01.png|alt=example01]]
