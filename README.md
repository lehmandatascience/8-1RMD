---
title: "DHS Immigration Statistics"
output: html_document
---

Loading necessary libraries below:

```{r}
library(readxl)
library(dplyr)
library(tidyr)
library(ggplot2)
library(plotly)
library(scales)
library(reshape2)
```

First, needed to download data table from https://www.dhs.gov/immigration-statistics/lawful-permanent-residents
https://www.dhs.gov/sites/default/files/publications/YRBK%202016%20LPR%20Excel%20Final_0.zip
https://www.dhs.gov/sites/default/files/publications/LPR2006_0.zip

```{r}
download.file(  "https://www.dhs.gov/sites/default/files/publications/YRBK%202016%20LPR%20Excel%20Final_0.zip", destfile = "file1.zip")

download.file("https://www.dhs.gov/sites/default/files/publications/LPR2006_0.zip", destfile = "file2.zip")



```

```{r}
unzip("file1.zip", list = TRUE)
unzip("file2.zip", list = TRUE)
```
```{r}

unzip("file1.zip", exdir = "downloads/unzipped", files = "YRBK 2016 LPR Excel Final/fy2016_table6.xls")
unzip("file2.zip", exdir = "downloads/unzipped", files = "Table 06D.xls")

```


Second, had to adjust data tables manually in excel to make sure column names were in the same order and had the same number of 
rows/columns. If row names weren't in the same order after merging, merging won't match up properly and there will be a presumed 
loss of data.

Third, uploaded excel files into 'R' to be read.

```{r}
newer_file <- "downloads/unzipped/YRBK 2016 LPR Excel Final/fy2016_table6.xls"
tableSix_2007to2016_all <- 
  read_excel(newer_file, 
             na ="D", skip = 3, col_names = T)
# Get the column names for this table
col_name_list <- colnames(tableSix_2007to2016_all)
# only read in the totals
tableSix_2007to2016_Total <- 
  read_excel(newer_file, 
             na ="D", skip = 3, col_names = col_name_list, range = "a6:k30")
tableSix_2007to2016_Total$type <- "Total"
# only read in the  adjustments
tableSix_2007to2016_Adjust <-
  read_excel(newer_file,
             na ="D", skip = 3, col_names = col_name_list,
             range = "a32:k56")
tableSix_2007to2016_Adjust$type <- "Adjusted Status" 

# only read in the new arrivals
tableSix_2007to2016_New <- 
  read_excel(newer_file,
             na ="D", skip = 3, col_names = col_name_list, 
             range = "a58:k82")
tableSix_2007to2016_New$type <- "New Arrivals"
```

The file from the earlier period
```{r}
older_file <- "downloads/unzipped/Table 06D.xls"
tableSix_1997to2006_all <- read_excel(older_file, 
                                      na ="D", skip = 3,col_names = T)
col_name_old <- colnames(tableSix_1997to2006_all)
# only read in the Totals
tableSix_1997to2006_Total <- read_excel(older_file, 
                                      na ="D", range = "a6:k29", 
                                      col_names = col_name_old )

# only read in the adjusted
tableSix_1997to2006_Adjust <- read_excel(older_file, 
                                      na ="D", range = "a31:k54",
                                      col_names = col_name_old)


# only read in the New arrivals
tableSix_1997to2006_New <- read_excel(older_file, 
                                      na ="D", range = "a56:k79",
                                      col_names = col_name_old)

```

Merging two datatables, (1997 to 2006) and (2007 to 2016) together by row.names to avoid duplicates.


```{r}
## Add code to deal with the change of name of the "Special immigrants" row

tableSix_2007to2016_New$`Type and class of admission`[tableSix_2007to2016_New$`Type and class of admission` ==
'Fourth: Certain special immigrants'] <-
'Fourth: Special immigrants'

tableSix_2007to2016_Total$`Type and class of admission`[tableSix_2007to2016_Total$`Type and class of admission` ==
'Fourth: Certain special immigrants'] <-
'Fourth: Special immigrants'

tableSix_2007to2016_Adjust$`Type and class of admission`[tableSix_2007to2016_Adjust$`Type and class of admission` ==
'Fourth: Certain special immigrants'] <-
'Fourth: Special immigrants'

```
From ew: Now instead of one merge here, we do three separate merges.
This could definitely be automated more but not really worth it for just
three
```{r}
combo_data_Total <- merge.data.frame(x = tableSix_1997to2006_Total, 
                                     y = tableSix_2007to2016_Total, 
                                     by = "Type and class of admission", 
                                     all = TRUE,
                                     sort = F, 
                                     no.dups = T)

combo_data_Adjust <- merge.data.frame(x = tableSix_1997to2006_Adjust, 
                                      y = tableSix_2007to2016_Adjust, 
                                      by = "Type and class of admission",
                                      all = TRUE,
                                      sort = F, 
                                      no.dups = T)


combo_data_New <- merge.data.frame(x = tableSix_1997to2006_New, 
                                   y = tableSix_2007to2016_New, 
                                   by = "Type and class of admission",
                                    all = TRUE,
                                   sort = F, 
                                   no.dups = T)
```

Now because we have the type variable that tells us what kind of data each is
we can put the three separate data frames back into one big one.

```{r}
combo_data <- rbind(combo_data_Total, combo_data_Adjust, combo_data_New)
```


(SIgh of relief)


ew I really meant for you to do this later but it works fine here
Changing data table format from 'wide' to 'long'.

```{r}
long <- gather(combo_data, year, value, "1997":"2016")

```

Using lapply function to change all -'s dashes in the dataset to 0s (only years 2007-2016 for the "Asylees" row had dashes)

```{r}

long[2:4] <- lapply(long[2:4], function(col) as.character( gsub("-$|\\,", "0", col) ) )

```

Using lapply function to change the class for all the years from "character" to "numeric" , 3:4 representing column 3 through 4.

```{r}
ix <- 3:4
long[ix] <- lapply(long[ix], as.numeric)

```

  
Filtered 'long' dataset to just include 'Total' for all years from 1997 to 2016.


```{r}
long_sub <- filter(long,  
      long$`Type and class of admission` %in% c("Total"))
```
 
Created a test plot for only the 'Total' row for all years from 1997 to 2016.
***this graph is a stacked graph

```{r}
g1 <- ggplot(long_sub, aes(year, value, fill = type))+
  geom_col(show.legend = TRUE, na.rm = TRUE) + ggtitle ("Number of Total Legal Permanent Residents from 1997 to 2016") + 
  labs(x= "Year", y= "Number of Legal Permanent Residents", caption ="Source: Department of Homeland Security Immigration Statistics")

g1 + scale_x_continuous(breaks=c(1997, 1999, 2001, 2003, 2005, 2007, 2009, 2011, 2013, 2016)) + scale_y_continuous(breaks = c(0, 500000, 1000000, 1500000, 2000000, 2500000), labels = comma)


```
Total for all years 1997-2016 (same as above just in position= "dodge") 
*** dodge means the bars are side by side

```{r}

p <- ggplot(data=long_sub) + geom_col(mapping = aes(year, value, fill = type),  position = "dodge") 
p + scale_x_continuous(breaks=c(1997, 1999, 2001, 2003, 2005, 2007, 2009, 2011, 2013, 2016)) +  scale_y_continuous( labels = comma)

```
Graph for Total compared to Family total compared to Employment total
```{r}
long_sub2 <- filter(long,  
      long$`Type and class of admission` %in% c("Total", "Family-sponsored preferences" ,"Employment-based preferences" ), long$type %in% "Total")
```
CUTS WHEN TRYING TO GET THE GRAPH TO WORK
```{r}
levels(long$`Type and class of admission`) <- c("Total", "Family-sponsored preferences", "Employment-based preferences")

ggplot(long_sub2, aes(year, value, fill="Type and class of admission")) +
  geom_bar(stat = "identity")
```
Dr. Elin's Graph of the First subcategory under employment: 
**First she had to filter from the "long" dataset to create a dataset with the variable she is interested in which is "First: Priority workers"
```{r}

long_sub <- filter(long,  
      long$`Type and class of admission` %in% c("Total" ) & long$type != "Total")
long_priority1 <- filter(long,  
      long$`Type and class of admission` %in% c("First: Priority workers"  
                                                ) & long$type != "Total")
```
Created a test plot for only the 'Priority 1' data for all years from 1997 to 2016.
```{r}
g2 <- ggplot(long_priority1, aes(x = year, y = value, fill = type )) +
  geom_col() +
  theme(legend.position="bottom")

g2

```
