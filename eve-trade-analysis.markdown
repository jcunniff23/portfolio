---
layout: notebook
title: EVE Online Trade Analysis
permalink: /projects/RMT-analysis
---
# EVE Trade-Volume Analysis


## What is EVE?
EVE Online is a Massively Multiplayer Online (MMO) sci-fi game that was released in the early 2000s. In the 20 years that it has been online, the in-game, player driven market has grown and evolved to meet player needs. The market in this game presents a unique case for analysis because  the game's API allows for full access to public trading data. Over the years, the community has been very active and involved in third-party applications to assist in data collection from the developer's API.

The data for this project will be looking at the trade and volume metrics for the trade 'hubs' across the in-game universe. These hubs can be thought of as metroplexes across a nation that have the strongest economies. Unlike the stock market, the items in game must be bought or sold at their precise location.

To help understand how this works, the best analogy for the behavior of the economy is to relate it to the stock market. Instead of being able to remotely buy and sell units of a share, all of the buying or selling must be done at the precise location where the items are located. These trade hubs, or locations where the highest amount of goods are bought and sold, allow for regional diffferences in supply, demand and price fluctation. 


## The EVE Economy and Personal Motivation
The EVE Economy has come under research and reporting before; work from Taylor et. all [2] or Melaza et. all [3] can proide further context to the availability of data. 

A main point of my motivation for this analysis is my life long interest in Sci-Fi media and this game in general. Being able to play with the data over the years has helped me learn SQL, Python and Excel VBA skills that I would not have otherwise had the motivation to learn. Going one step further, I wanted to utilize the publically available data to produce a report to sharpen skills related to my greater portfolio. 

## Data and Methodology
The data for this analysis is from E. Tristram [1] at https://static.adam4eve.eu under `/MarketPricesStationHistory/2024` and `/MarketVolumeStationHistory/2024`. This data comes as raw CSVs that is then digested into a MySQL database to allow for cleaning and processing of data via Python. The SQL scripts are shown below.

The data in this notebook shows the price and volume trends for major trade hubs which was pulled on a 15-minute interval and then published on a weekly basis to the static repository.

### Bibliography
[1] E. Tristram, “Adam4EVE static exports.” [Online]. Available: https://static.adam4eve.eu
[2] N. Taylor, K. Bergstrom, J. Jenson, and S. De Castell, “Alienated Playbour: Relations of Production in EVE Online,” Games and Culture, vol. 10, no. 4, pp. 365–388, Jul. 2015, doi: 10.1177/1555412014565507.
[3] A. M. Belaza, J. Ryckebusch, K. Schoors, L. E. C. Rocha, and B. Vandermarliere, “On the connection between real-world circumstances and online player behaviour: The case of EVE Online,” PLoS ONE, vol. 15, no. 10, p. e0240196, Oct. 2020, doi: 10.1371/journal.pone.0240196.


### SQL


```python
from sqlalchemy import create_engine
import pandas as pd
engine = create_engine("mysql+mysqldb://python:password@localhost:3307/eve_data")
conn = engine.connect()
```


```python
##  Load all 2023 data into MySQL server
from pathlib import Path
price_dump = Path("data/2023/prices")
for entry in price_dump.iterdir():
    df = pd.read_csv(entry, delimiter=";")
    df.to_sql(name='2023_price_history', con=conn, schema='eve_data', if_exists='append')
```


```python
##  Load all 2023 volume data into MySQL server
from pathlib import Path
price_dump = Path("data/2023/volume")
for entry in price_dump.iterdir():
    df = pd.read_csv(entry, delimiter=";")
    df.to_sql(name='volume_history_2023', con=conn, schema='eve_data', if_exists='append')
```

### 2023 Data Exploration
From the cells above, ~16m records are loaded from the static data dump from 2023 price history. This data was taken from a static dataset that will take the public records from a given location every 15 minutes and track changes to build daily values on sale price and volume.


```python
##  Select price data for a single item, using the type_id's

def sql_base(type: int) -> str:
    return f'''  SELECT date, buy_price_high, sell_price_low FROM price_history_2023
                WHERE type_id = {type} AND location_id = 60003760'''

prices_plex = pd.read_sql_query(sql_base(44992), conn)

# prices_trit = pd.read_sql_query(sql_base(34), conn)
# prices_ishtar = pd.read_sql_query(sql_base(12005), conn)
# prices_isogen = pd.read_sql_query(sql_base(37), conn)
```


```python
## Select volume data for a single item, utilizing same type_ids

def sql_vol(type: int) -> str:
    return f'''  SELECT date, buy_volume_avg, sell_volume_avg FROM volume_history_2023
                WHERE type_id = {type} AND location_id = 60003760'''

vol_plex  = pd.read_sql_query(sql_vol(44992), conn)

# vol_trit = pd.read_sql_query(sql_vol(34), conn)
# vol_iso = pd.read_sql_query(sql_vol(37), conn)
# vol_ishtar  = pd.read_sql_query(sql_vol(12005), conn)
```

#### Visualization


```python
## Seaborn
import seaborn as sb
sb.set_theme(style='darkgrid')
```


```python
## Run once -- set index to datetime column
vol_plex['date'] = pd.to_datetime(vol_plex['date'])
vol_plex.set_index('date', inplace=True)

prices_plex['date'] = pd.to_datetime(prices_plex['date'])
prices_plex.set_index('date', inplace=True)
```

#### PLEX Price and Volume Plotting

In EVE, 'PLEX' is an item that is utilized to convert real world money to in-game subscrition time. The typical price for a unit of PLEX is approximately 5 cents without a sale or spending more to lower the price per unit. The 'real world' store sells bundles of $4.99 for 100 units all the way up to $649.99 for 20,000 units. 

Occasionally, to help drive the free market, the game developers, Crowd Control Productions (CCP) will hold sales that impact the real world market realted items. The broad category of items for this are called real money trade (RMT) market. 

##### 2023 Impacts to the RMT Market
In 2023 there were a number of sales that can impact the supply and demand of the RMT market. Some notable ones that will be explored are:
- January 10th
- June 2nd
- October 5th
- October 13th


```python
## Monthly Sell Prices of PLEX over 2023

plex_WK = prices_plex.resample('W').mean()
fig_plex = sb.lineplot(data=plex_WK, x='date', y='sell_price_low')
fig_plex.set_title('PLEX Sale Price 2023')
fig_plex.set_xlabel('Month')
fig_plex.set_ylabel('Sell Price')
```




    Text(0, 0.5, 'Sell Price')




    
![png](PLEX_Analysis_files/PLEX_Analysis_13_1.png)
    



```python
vol_plex_ME = vol_plex.resample('W').mean()
fig_plex = sb.lineplot(data=vol_plex_ME, x='date', y='sell_volume_avg')
fig_plex.set_title('PLEX Sale Volume 2023')
fig_plex.set_xlabel('Month')
fig_plex.set_ylabel('Volume (units)')
```




    Text(0, 0.5, 'Volume (units)')




    
![png](PLEX_Analysis_files/PLEX_Analysis_14_1.png)
    

