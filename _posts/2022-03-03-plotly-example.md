---
  layout: post
title: HW 1
---
  
  In this blog post, I'll show you how to create several interesting, interactive data graphics using the NOAA climate data.


# Create a Database 

## §1. Import modules and libraries
You can import the modules and libraries by running:

```python 
import pandas as pd
import sqlite3
import numpy as np
from plotly import express as px
from sklearn.linear_model import LinearRegression
```

## §2. Import the datasets

```python
temps = pd.read_csv("temps_stacked.csv")
temps['countryID'] = temps['ID'].str[:2] #create a new column with abbreviated country names
countries = pd.read_csv('countries.csv')
countries = countries.rename(columns= {"FIPS 10-4": "FIPS_104"})# get rid of spaces to make it easier to read in sql
stations = pd.read_csv('station-metadata.csv')
```

## §3. Convert datasets into SQL database
You can create your database by running the following code.
 ```python
conn = sqlite3.connect("temps.db")
temps.to_sql("temperatures", conn, if_exists="replace", index=False)
countries.to_sql("countries", conn, if_exists="replace", index=False)
stations.to_sql("stations", conn, if_exists="replace", index=False)
# always close your connection
conn.close()
 ```
 
# Write a query function
 
## §1. Query function
The return value of `query_climate_database()` is a Pandas dataframe of temperature readings for the specified country, in the specified date range, in the specified month of the year. 

Run the following code 
```python
def query_climate_database(country, year_begin, year_end, month):
    conn = sqlite3.connect("temps.db") #connects to database
    
    cmd = \
    """ 
    SELECT temperatures.Month, temperatures.Year, temperatures.Temp,  stations.NAME, \
    stations.LATITUDE, stations.LONGITUDE, countries.Name
    FROM temperatures 
    JOIN stations ON temperatures.ID = stations.ID 
    JOIN countries ON countries.FIPS_104 = temperatures.countryID
    WHERE (temperatures.Year>=""" + str(year_begin) + """ ) \
    AND (temperatures.Year<=""" + str(year_end) + """) \
    AND (temperatures.Month= """ + str(month) + """) \
    AND (countries.Name='"""+ country  +"""')
  
    """
    df = pd.read_sql(cmd, conn) #reads the data frame
    return(df)
```

## §2. Call query function
Let's call the query function with the following code

```python
query_climate_database(country = "India", 
                       year_begin = 1980, 
                       year_end = 2020,
                       month = 1)
```
                       
You obtain the following output: 
![image-example.png](/images/dataframe1.png)

#Write a Geographic Scatter Function for Yearly Temperature Increases

## §1.
Let's write a function called `temperature_coefficient_plot()`

The output of this function should be an interactive geographic scatterplot, constructed using Plotly Express, with a point for each station, such that the color of the point reflects an estimate of the yearly change in temperature during the specified month and time period at that station. 

```python
def temperature_coefficient_plot(country, year_begin, year_end, month, min_obs, **kwargs):
    monthDict={1:'January', 2:'February', 3:'March', 4:'April', 5:'May', 6:'June', 7:'July', 8:'August', 9:'September', 10:'October', 11:'November', 12:'December'}

    conn = sqlite3.connect("temps.db")
    
    cmd = \
    """ 
    SELECT temperatures.Month, temperatures.Year, temperatures.Temp,  stations.NAME, \
    stations.LATITUDE, stations.LONGITUDE, countries.Name
    FROM temperatures 
    JOIN stations ON temperatures.ID = stations.ID 
    JOIN countries ON countries.FIPS_104 = temperatures.countryID
    WHERE (temperatures.Year>=""" + str(year_begin) + """ ) \
    AND (temperatures.Year<=""" + str(year_end) + """) \
    AND (temperatures.Month= """ + str(month) + """) \
    AND (countries.Name='"""+ country  +"""')
  
    """
    df = pd.read_sql(cmd, conn) 
    df['count']=df.groupby(['NAME',"Month"])['Year'].transform("count")
    df = df[df["count"]>min_obs]

    # your old friend scikit-learn
    def coef(data_group):
      X = data_group[["Year"]]
      y = data_group["Temp"]
      LR = LinearRegression()
      LR.fit(X, y)
      slope = LR.coef_[0]
      return slope

    coefs = df.groupby(['NAME',"Month","LONGITUDE","LATITUDE"]).apply(coef)
    coefs = coefs.reset_index()

    coefs.columns = ['NAME',"Month","LONGITUDE","LATITUDE","Estimated yearly increase (C)"]

    
    fig = px.scatter_mapbox(coefs, # data for the points you want to plot
                        lat = "LATITUDE", # column name for latitude informataion
                        lon = "LONGITUDE", # column name for longitude information
                        hover_name = "NAME", # what's the bold text that appears when you hover over
                         # how much you want to zoom into the map
                        height = 300, # control aspect ratio
                        
                        # opacity for each data point
                       color="Estimated yearly increase (C)",
                       title = "Estimates of yearly increases in temperature in "+ monthDict.get(month)+\
" for stations in " +country + " from "\
                       +str(year_begin)+"-"+str(year_end),
                       **kwargs) # represent temp using color
    fig.update_layout(margin={"r":0,"t":50,"l":50,"b":0})
    return(fig.show())
```

## §2. Call your function to display geographic scatter plot
You can create your own scatter plot by running: 

```python
color_map = px.colors.diverging.RdGy_r
df2 = temperature_coefficient_plot(country = "India", 
                       year_begin = 1980, 
                       year_end = 2020,
                       month = 1,
                       min_obs = 10,
                       mapbox_style="carto-positron",
                       color_continuous_scale=color_map,
                       zoom = 2)
```
![image-example.png](/images/unknown.png)
