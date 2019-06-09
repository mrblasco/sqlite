# SQLite

Contents: 

1. Preparing and importing data 
2. Using SQLite from R
3. Handling JSON files

## Preparing and importing data into a local SQLite database

First, check whether your machine has SQLite installed

```
sqlite3
```

The data that we are going to use in this tutorial is the National Energy Efficiency Data - Framework: anonymised data 2014 (NEED) dataset provided by the Department of Energy & Climate Change.

```
curl -O https://www.gov.uk/government/uploads/system/uploads/attachment_data/file/315188/need_public_use_file_2014.csv
```

Once in the directory with the data file, start SQLite by creating a new database called need_data:

```
sqlite3 need_data
```

Type .databases to show all currently available databases:

Then set the column separator to comma and import the need_puf_2014.csv data file as a new table called need:

```
sqlite> .separator "," 
sqlite> .import need_public_use_file_2014.csv need
```

You can check the available tables by running the .tables command:

```
sqlite> .tables
```

Also, we may inspect the structure of the table using a PRAGMA statement:

```
sqlite> PRAGMA table_info('need');
```

The .schema function allows us to print the schema of the table.

```
sqlite> .schema need
```


## Use SQLite from R

As R requires the RSQLite package to make a connection with the SQLite database using the DBI package, we have to download the new versions of DBI and its dependency Rcpp beforehand. Note that in order to install recent releases of these packages from GitHub repositories, you first need to install the devtools package - it allows connectivity with GitHub:

```
> install.packages("devtools")
...#output truncated
> devtools::install_github("RcppCore/Rcpp")
...#output truncated
> devtools::install_github("rstats-db/DBI")
...#output truncated
Then, install and load the RSQLite package:
```

```
> install.packages("RSQLite")
...#output truncated
> library(RSQLite)
```

Let's then create a connection with the need_data SQLite database:

```
> con <- dbConnect(RSQLite::SQLite(), "need_data")
> con
<SQLiteConnection>
```

The dbListTables() and dbListFields() functions provide information on the available tables in the connected database and columns within a specified table, respectively:

```
> dbListTables(con)
[1] "need"
> dbListFields(con, "need")
 [1] "HH_ID"           "REGION"          "IMD_ENG"        
 [4] "IMD_WALES"       "Gcons2005"       "Gcons2005Valid" 
 [7] "Gcons2006"       "Gcons2006Valid"  "Gcons2007"      
[10] "Gcons2007Valid"  "Gcons2008"       "Gcons2008Valid" 
[13] "Gcons2009"       "Gcons2009Valid"  "Gcons2010"      
[16] "Gcons2010Valid"  "Gcons2011"       "Gcons2011Valid" 
[19] "Gcons2012"       "Gcons2012Valid"  "Econs2005"      
[22] "Econs2005Valid"  "Econs2006"       "Econs2006Valid" 
[25] "Econs2007"       "Econs2007Valid"  "Econs2008"      
[28] "Econs2008Valid"  "Econs2009"       "Econs2009Valid" 
[31] "Econs2010"       "Econs2010Valid"  "Econs2011"      
[34] "Econs2011Valid"  "Econs2012"       "Econs2012Valid" 
[37] "E7Flag2012"      "MAIN_HEAT_FUEL"  "PROP_AGE"       
[40] "PROP_TYPE"       "FLOOR_AREA_BAND" "EE_BAND"        
[43] "LOFT_DEPTH"      "WALL_CONS"       "CWI"            
[46] "CWI_YEAR"        "LI"              "LI_YEAR"        
[49] "BOILER"          "BOILER_YEAR"
```

We may now query the data using the dbSendQuery() function; for example, we can retrieve all records from the table for which the value of the FLOOR_AREA_BAND variable equals 1:

```
> query.1 <- dbSendQuery(con, "SELECT * FROM need WHERE FLOOR_AREA_BAND = 1")
> dbGetStatement(query.1)
[1] "SELECT * FROM need WHERE FLOOR_AREA_BAND = 1"
```

In case you want to extract the string representing the SQL query used, you may apply the dbGetStatement() function on the object created by the dbSendQuery() command, as shown in the preceding code.

The results set may now be easily pulled to R (note that all queries and data processing activities run directly on the database, thus saving valuable resources normally used by R processes):

```
> query.1.res <- fetch(query.1, n=50)
> str(query.1.res)
'data.frame':  50 obs. of  50 variables:
 $ HH_ID          : chr  "5" "6" "12" "27" ...
 $ REGION         : chr  "E12000003" "E12000007" "E12000007" "E12000004" ...
 $ IMD_ENG        : chr  "1" "2" "1" "1" ...
 $ IMD_WALES      : chr  "" "" "" "" ...
 $ Gcons2005      : chr  "" "" "" "5500" ...
 $ Gcons2005Valid : chr  "M" "O" "M" "V" ...
...#output truncated
> query.1.res
   HH_ID    REGION IMD_ENG IMD_WALES Gcons2005 Gcons2005Valid
1      5 E12000003       1                                  M
2      6 E12000007       2                                  O
3     12 E12000007       1                                  M
4     27 E12000004       1                5500              V
5     44 W99999999                 1     18000              V
...#output truncated
```

After performing the query, we can obtain additional information, for example its full SQL statement, the structure of the results set, and how many rows it returned:

```
> info <- dbGetInfo(query.1)
> str(info)
List of 6
 $ statement   : chr "SELECT * FROM need WHERE FLOOR_AREA_BAND = 1"
 $ isSelect    : int 1
 $ rowsAffected: int -1
 $ rowCount    : int 50
 $ completed   : int 0
 $ fields      :'data.frame': 50 obs. of  4 variables:
  ..$ name  : chr [1:50] "HH_ID" "REGION" "IMD_ENG" "IMD_WALES" ...
  ..$ Sclass: chr [1:50] "character" "character" "character" "character" ...
  ..$ type  : chr [1:50] "TEXT" "TEXT" "TEXT" "TEXT" ...
  ..$ len   : int [1:50] NA NA NA NA NA NA NA NA NA NA ...
> info
$statement
[1] "SELECT * FROM need WHERE FLOOR_AREA_BAND = 1"
$isSelect
[1] 1
$rowsAffected
[1] -1
$rowCount
[1] 50
$completed
[1] 0
$fields
              name    Sclass type len
1            HH_ID character TEXT  NA
2           REGION character TEXT  NA
3          IMD_ENG character TEXT  NA
4        IMD_WALES character TEXT  NA
5        Gcons2005 character TEXT  NA
...#output truncated
```

Once we complete a particular query, it is recommended to free the resources by clearing the obtained results set:

```
> dbClearResult(query.1)
[1] TRUE
```

We may now run a second query on the need table contained within the need_data SQLite database. This time, we will calculate the average electricity consumption for 2012 grouped by the levels of categorical variables: electricity efficiency band (EE_BAND), property age (PROP_AGE), and property type (PROP_TYPE). The statement will also sort the results set in ascending order, based on the values of two variables, EE_BAND and PROP_TYPE:

```
> query.2 <- dbSendQuery(con, "SELECT EE_BAND, PROP_AGE, PROP_TYPE, 
+                        AVG(Econs2012) AS 'AVERAGE_ELEC_2012' 
+                        FROM need 
+                        GROUP BY EE_BAND, PROP_AGE, PROP_TYPE 
+                        ORDER BY EE_BAND, PROP_TYPE ASC")
```

By running the statement, we simply apply a set of queries on the database; we then need to get the results into R using the fetch() function, just like we did in the first query. If you want to fetch all records, set the n parameter to -1:

```
> query.2.res <- fetch(query.2, n=-1)
```

It's always a good idea to inspect the structure and the size of the results set created with the query:

```
> info2 <- dbGetInfo(query.2)
> info2
$statement
[1] "SELECT EE_BAND, PROP_AGE, PROP_TYPE, \n                       AVG(Econs2012) AS 'AVERAGE_ELEC_2012' \n                       FROM need \n                       GROUP BY EE_BAND, PROP_AGE, PROP_TYPE \n                       ORDER BY EE_BAND, PROP_TYPE ASC"
$isSelect
[1] 1
$rowsAffected
[1] -1
$rowCount
[1] 208
$completed
[1] 1
$fields
               name    Sclass type len
1           EE_BAND character TEXT  NA
2          PROP_AGE character TEXT  NA
3         PROP_TYPE character TEXT  NA
4 AVERAGE_ELEC_2012    double REAL   8
```

From the preceding output, you can see that our results set consists of 208 rows of data with the variables as outlined in the fields attribute of the output. We can finally view the first six rows of data:

```
> head(query.2.res, n=6)
  EE_BAND PROP_AGE PROP_TYPE AVERAGE_ELEC_2012
1       1      101       101          2650.000
2       1      102       101         12162.500
3       1      103       101          3137.500
4       1      104       101          4200.000
5       1      105       101          3933.333
6       1      106       101          5246.774
```

Before disconnecting from the database, you can also export the results set into a new table within the database:

```
> dbWriteTable(con, "query_2_result", query.2.res)
[1] TRUE
```

The new table named query_2_result has now been created in the need_data database:

```
> dbListTables(con)
[1] "need"           "query_2_result"
```

Once you finish all the processing, make sure to clear the results of the most recent query and disconnect from the active connection:

```
> dbClearResult(query.2)
[1] TRUE
> dbDisconnect(con)
[1] TRUE
```

This completes the tutorial on SQLite database connectivity with the R language as a data source for locally run SQL queries. In the next section, we will explore how easily R can operate with a MariaDB database deployed on an Amazon EC2 instance.

