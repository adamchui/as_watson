
# A.S. Watson Data Warehouse Analysis Series

Thanks for the A.S. Watson Data released during A.S. Watson Hackathon, we would use those retail stores (located in Hong Kong and Macau) information to do some data exploration and analysis.

## Store Segmentation Analysis
A.S. Watson has allocated the store into 12 segments, namely, NULL, Budget, Cosmo, Others, WTC - Baby, WTC - Beautiq, WTC - Commercial, WTC - Destination, WTC - Health, WTC - Residential, WTC - Savers and WTC - Tourists/Conv. Store segments are the store classifications, which will have an influence on the choice of POGs to use.  We believe that store in each segments have  different customers structures and product purchase preferences. 

In this analysis, we would extract and summarize information by Store in those persectives, including
- Item Transaction
- Member Personal Info
- Member Loyalty
- Product Purchased Category

Before this analysis, all data in csv format has been imported to SQL database by SSIS with bulk insert. For details, please reference to another post. 

## Data Retrieval from SQL Database
We would extract and do some initial aggregation, like, sum, counting, average and standard deviation.


### Load all the required library


```python
import pyodbc
import pandas as pd
import numpy as np
from sklearn.cluster import MeanShift
from sklearn.datasets.samples_generator import make_blobs
import matplotlib.pyplot as plt
from matplotlib import style
from sklearn.decomposition import PCA
from sklearn import cluster,manifold
#from sklearn import manifold
from ggplot import *
import googlemaps
from datetime import datetime
import folium
import math
from sklearn import preprocessing
```

    C:\ProgramData\Anaconda3\lib\site-packages\ggplot\utils.py:81: FutureWarning: pandas.tslib is deprecated and will be removed in a future version.
    You can access Timestamp as pandas.Timestamp
      pd.tslib.Timestamp,
    C:\ProgramData\Anaconda3\lib\site-packages\ggplot\stats\smoothers.py:4: FutureWarning: The pandas.lib module is deprecated and will be removed in a future version. These are private functions and can be accessed from pandas._libs.lib instead
      from pandas.lib import Timestamp
    C:\ProgramData\Anaconda3\lib\site-packages\statsmodels\compat\pandas.py:56: FutureWarning: The pandas.core.datetools module is deprecated and will be removed in a future version. Please use the pandas.tseries module instead.
      from pandas.core import datetools
    

In case the python package cannot be found, please install like this

```python
!pip install hdbscan
```

Or you need to specify the pip location

```python
!C:\ProgramData\Anaconda3\Scripts\pip install hdbscan
```


### Item Transaction Data Retreival

In order to capture the item transaction, 

Group the Data by STORE ( STORE_ID )
- Average of Product Type Sold in each order
- Standard Deviation of Product Type Sold in each order
- Average of Product Type Quantity in each order
- Standard Deviation of Product Quantity Sold in each order
- Average of Sales Revenue in each order
- Standard Deviation of Sales Revenue in each order
- Average of Discount in each order
- Standard Deviation of Discount in each order


```python

cnxn = pyodbc.connect('DRIVER={SQL Server};SERVER={fastserver};DATABASE={watson};Trusted_Connection=yes;')

query = """
With RFM_CTE (STORE_ID, AVG_SEG_NUM, STDEV_SEG_NUM)
as
(
SELECT STORE_ID, avg(cast(SEG_NUM as int)) AVG_SEG_NUM, stdev(cast(SEG_NUM as int)) STDEV_SEG_NUM
  FROM [watson].[dbo].[WATSON_AGG_TRANSACTION] a
  inner join [watson].[dbo].[WATSON_FIN_RFM] b on a.CONTACT_ID = b.CONTACT_ID 
  and a.BUSINESS_DATE_DT>=b.SEG_START_DT
  and a.BUSINESS_DATE_DT<=b.SEG_END_DT
  --and a.BUSINESS_DATE<b.SEG_START_DT
  group by STORE_ID
), AGG_TRAN_CTE 
as
(
select cast(a.STORE_ID as varchar(10)) STORE_ID
	,avg([PRODUCT_ID_COUNT]) AVG_SUM_PRODUCT_ID
	,stdev([PRODUCT_ID_COUNT]) STDEV_SUM_PRODUCT_ID
      ,avg([ITEM_QUANTITY]) AVG_SUM_ITEM_QUANTITY
	  ,stdev([ITEM_QUANTITY]) STDEV_SUM_ITEM_QUANTITY
      ,avg([ITEM_AMOUNT]) AVG_SUM_ITEM_AMOUNT
	  ,stdev([ITEM_AMOUNT]) STDEV_SUM_ITEM_AMOUNT
      ,avg([ITEM_DISCOUNT_AMOUNT]) AVG_SUM_ITEM_DISCOUNT_AMOUNT
	  ,stdev([ITEM_DISCOUNT_AMOUNT]) STDEV_SUM_ITEM_DISCOUNT_AMOUNT

  from [dbo].[WATSON_AGG_TRANSACTION] a  
  group by a.STORE_ID
)
select a.*, b.AVG_SEG_NUM, b.STDEV_SEG_NUM from 
AGG_TRAN_CTE a
inner join RFM_CTE b on a.STORE_ID = b.STORE_ID
"""


ITEM_TRAN_df = pd.read_sql(query, cnxn)
ITEM_TRAN_df.head()

```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>STORE_ID</th>
      <th>AVG_SUM_PRODUCT_ID</th>
      <th>STDEV_SUM_PRODUCT_ID</th>
      <th>AVG_SUM_ITEM_QUANTITY</th>
      <th>STDEV_SUM_ITEM_QUANTITY</th>
      <th>AVG_SUM_ITEM_AMOUNT</th>
      <th>STDEV_SUM_ITEM_AMOUNT</th>
      <th>AVG_SUM_ITEM_DISCOUNT_AMOUNT</th>
      <th>STDEV_SUM_ITEM_DISCOUNT_AMOUNT</th>
      <th>AVG_SEG_NUM</th>
      <th>STDEV_SEG_NUM</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>11955</td>
      <td>1.724710</td>
      <td>1.370606</td>
      <td>2.770968</td>
      <td>7.835618</td>
      <td>113.434108</td>
      <td>231.057269</td>
      <td>16.133859</td>
      <td>96.308430</td>
      <td>3</td>
      <td>1.831615</td>
    </tr>
    <tr>
      <th>1</th>
      <td>11972</td>
      <td>1.645778</td>
      <td>1.399311</td>
      <td>2.249968</td>
      <td>7.003476</td>
      <td>98.383218</td>
      <td>304.284756</td>
      <td>6.553552</td>
      <td>50.989477</td>
      <td>3</td>
      <td>1.682894</td>
    </tr>
    <tr>
      <th>2</th>
      <td>12272</td>
      <td>1.919302</td>
      <td>1.764426</td>
      <td>2.959445</td>
      <td>7.996145</td>
      <td>166.950796</td>
      <td>396.427580</td>
      <td>18.517122</td>
      <td>120.005615</td>
      <td>3</td>
      <td>1.958209</td>
    </tr>
    <tr>
      <th>3</th>
      <td>12172</td>
      <td>2.241608</td>
      <td>2.060455</td>
      <td>2.967163</td>
      <td>4.868277</td>
      <td>164.809825</td>
      <td>389.157374</td>
      <td>14.523409</td>
      <td>79.275861</td>
      <td>3</td>
      <td>1.893091</td>
    </tr>
    <tr>
      <th>4</th>
      <td>12193</td>
      <td>1.748568</td>
      <td>1.474737</td>
      <td>3.247996</td>
      <td>10.545632</td>
      <td>158.936115</td>
      <td>489.315244</td>
      <td>16.628632</td>
      <td>100.305847</td>
      <td>3</td>
      <td>1.858842</td>
    </tr>
  </tbody>
</table>
</div>



### Member Segmentation Data Retrieval

Grouped by Store (Store_ID), divide each segment by total of all segment
- Percentage of Segment 1
- Percentage of Segment 2
- Percentage of Segment 3
- Percentage of Segment 4
- Percentage of Segment 5
- Percentage of Segment 6
- Percentage of Segment 7
- Percentage of Segment 8

Segment Number: 1 - VIP, 2 - Loyal, 3 - Infrequent , 4 - One-off,  5 - New, 6 - Lapsing , 7 - Inactive, 8 - Gone Away


```python
query = """

with SEG_NUM_CTE as
(
SELECT STORE_ID, SEG_NUM , count(*) as SEG_NUM_COUNT
  FROM [watson].[dbo].[WATSON_AGG_TRANSACTION] a
  inner join [watson].[dbo].[WATSON_FIN_RFM] b on a.CONTACT_ID = b.CONTACT_ID 
  and a.BUSINESS_DATE_DT>=b.SEG_START_DT
  and a.BUSINESS_DATE_DT<=b.SEG_END_DT
  --and a.BUSINESS_DATE<b.SEG_START_DT
  group by STORE_ID, SEG_NUM
),
SEG_NUM_SUM_CTE as
( 
select STORE_ID, sum(SEG_NUM_COUNT) SUM_SEG_NUM_COUNT
from SEG_NUM_CTE
group by STORE_ID
)
select cast(STORE_ID as varchar(10)) as STORE_ID, [1] as 'SEG_1',[2] as 'SEG_2',[3] as 'SEG_3',[4]  as 'SEG_4',[5]  as 'SEG_5',[6]  as 'SEG_6',[7] as 'SEG_7',[8]   as 'SEG_8'
FROM
(
select a.STORE_ID
,convert(int,a.SEG_NUM) SEG_NUM
, (convert(float,a.SEG_NUM_COUNT)/convert(float, b.SUM_SEG_NUM_COUNT))  as PF 
--, a.SEG_NUM_COUNT  as PF 
from SEG_NUM_CTE a
inner join SEG_NUM_SUM_CTE b 
on a.STORE_ID = b.STORE_ID
) p 
PIVOT
(
SUM(PF)
FOR SEG_NUM IN 
( [1],[2],[3],[4],[5],[6],[7],[8]
--'1','2','3','4','5','6','7','8','9'
)
) as pvt

"""
    
SEG_NUM_df = pd.read_sql(query, cnxn)
SEG_NUM_df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>STORE_ID</th>
      <th>SEG_1</th>
      <th>SEG_2</th>
      <th>SEG_3</th>
      <th>SEG_4</th>
      <th>SEG_5</th>
      <th>SEG_6</th>
      <th>SEG_7</th>
      <th>SEG_8</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>11902</td>
      <td>0.068422</td>
      <td>0.250199</td>
      <td>0.235277</td>
      <td>0.184273</td>
      <td>0.025247</td>
      <td>0.117270</td>
      <td>0.068989</td>
      <td>0.050323</td>
    </tr>
    <tr>
      <th>1</th>
      <td>12160</td>
      <td>0.064260</td>
      <td>0.278961</td>
      <td>0.234503</td>
      <td>0.164508</td>
      <td>0.047054</td>
      <td>0.112171</td>
      <td>0.060239</td>
      <td>0.038306</td>
    </tr>
    <tr>
      <th>2</th>
      <td>12172</td>
      <td>0.042320</td>
      <td>0.292780</td>
      <td>0.242275</td>
      <td>0.166343</td>
      <td>0.037692</td>
      <td>0.104618</td>
      <td>0.056451</td>
      <td>0.057521</td>
    </tr>
    <tr>
      <th>3</th>
      <td>12193</td>
      <td>0.069457</td>
      <td>0.296790</td>
      <td>0.243477</td>
      <td>0.165478</td>
      <td>0.022902</td>
      <td>0.099118</td>
      <td>0.062606</td>
      <td>0.040173</td>
    </tr>
    <tr>
      <th>4</th>
      <td>12073</td>
      <td>0.048635</td>
      <td>0.279899</td>
      <td>0.252977</td>
      <td>0.187558</td>
      <td>0.011985</td>
      <td>0.115209</td>
      <td>0.061133</td>
      <td>0.042603</td>
    </tr>
  </tbody>
</table>
</div>



### MEMBER Personal Info Retrieval

Grouped by Store (Store_ID), Weighted by Transaction
- Mainland Members to Total Members Ratio
- Members to All Customers Ratio
- Average Members' Age
- Standard Deviation Members' Age


```python
query = """

;With MAINLAND_CTE as
(
SELECT a.STORE_ID, b.MAINLAND_CHN_MEMBER_IND, count(*) Quantity
  FROM [watson].[dbo].[WATSON_AGG_TRANSACTION] a
  inner join [watson].[dbo].[WATSON_FIN_MEMBER] b on a.MEMBER_ID = b.MEMBER_ID 
  group by a.STORE_ID, b.MAINLAND_CHN_MEMBER_IND
), MAINLAND_TOTAL_CTE as
(
select STORE_ID, sum(Quantity) Total from MAINLAND_CTE
group by STORE_ID
), MAINLAND_CTE_RESULT as
(
select a.STORE_ID, convert(float,Quantity)/convert(float,Total) as MAIN_MEMBER_IN_TRAN_RATIO from MAINLAND_CTE a
inner join MAINLAND_TOTAL_CTE b on a.STORE_ID = b.STORE_ID
where MAINLAND_CHN_MEMBER_IND = 'Y'
), 
MEMBER_TRAN_CTE as
(
select STORE_ID, MEMBER_TRANSACTION_FLAG, count(*) GP_COUNTER
FROM [watson].[dbo].[WATSON_AGG_TRANSACTION] 
group by STORE_ID, MEMBER_TRANSACTION_FLAG
), MEMBER_TRAN_TOTAL as
(
select STORE_ID, sum(GP_COUNTER) TOTAL
from MEMBER_TRAN_CTE
group by STORE_ID
), MEMEBER_TRAN_CTE_RESULT as
(
select a.STORE_ID, convert(float, GP_COUNTER) /convert(float, TOTAL) MEMBER_PERCENTAGE
from MEMBER_TRAN_CTE a
inner join MEMBER_TRAN_TOTAL b on a.STORE_ID = b.STORE_ID
where MEMBER_TRANSACTION_FLAG = 'Y'
), GENDER_CTE as
(
SELECT a.STORE_ID, b.GENDER, count(*) Quantity
  FROM [watson].[dbo].[WATSON_AGG_TRANSACTION] a
  inner join [watson].[dbo].[WATSON_FIN_MEMBER] b on a.MEMBER_ID = b.MEMBER_ID 
  group by a.STORE_ID, b.GENDER
), GENDER_TOTAL_CTE as
(
select STORE_ID, sum(Quantity) Total from GENDER_CTE
group by STORE_ID
), GENDER_CTE_RESULT as (
select a.STORE_ID, convert(float,Quantity)/convert(float,Total) as M_GENDER_RATIO from GENDER_CTE a
inner join GENDER_TOTAL_CTE b on a.STORE_ID = b.STORE_ID
where GENDER = 'M'
), AGE_CTE_RESULT as
(
SELECT a.STORE_ID, avg(b.AGE) AVG_AGE, stdev(b.AGE) STDEV_AGE
  FROM [watson].[dbo].[WATSON_AGG_TRANSACTION] a
  inner join [watson].[dbo].[WATSON_FIN_MEMBER] b on a.MEMBER_ID = b.MEMBER_ID 
  group by a.STORE_ID
)
select cast(a.STORE_ID as varchar(10)) as STORE_ID, a.MAIN_MEMBER_IN_TRAN_RATIO, b.MEMBER_PERCENTAGE, c.M_GENDER_RATIO, d.AVG_AGE, d.STDEV_AGE from
MAINLAND_CTE_RESULT a 
inner join MEMEBER_TRAN_CTE_RESULT b on a.STORE_ID = b.STORE_ID
inner join GENDER_CTE_RESULT c on a.STORE_ID = c.STORE_ID
inner join AGE_CTE_RESULT d on a.STORE_ID = d.STORE_ID

"""
    
MEMBER_df = pd.read_sql(query, cnxn)
MEMBER_df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>STORE_ID</th>
      <th>MAIN_MEMBER_IN_TRAN_RATIO</th>
      <th>MEMBER_PERCENTAGE</th>
      <th>M_GENDER_RATIO</th>
      <th>AVG_AGE</th>
      <th>STDEV_AGE</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>12172</td>
      <td>0.649773</td>
      <td>0.222892</td>
      <td>0.182826</td>
      <td>63.741199</td>
      <td>34.545323</td>
    </tr>
    <tr>
      <th>1</th>
      <td>12193</td>
      <td>0.475164</td>
      <td>0.201638</td>
      <td>0.180780</td>
      <td>59.396990</td>
      <td>28.529972</td>
    </tr>
    <tr>
      <th>2</th>
      <td>12073</td>
      <td>0.386787</td>
      <td>0.435959</td>
      <td>0.169109</td>
      <td>57.723281</td>
      <td>24.167924</td>
    </tr>
    <tr>
      <th>3</th>
      <td>12271</td>
      <td>0.432872</td>
      <td>0.321882</td>
      <td>0.179190</td>
      <td>58.963774</td>
      <td>27.343983</td>
    </tr>
    <tr>
      <th>4</th>
      <td>11902</td>
      <td>0.555759</td>
      <td>0.344396</td>
      <td>0.135206</td>
      <td>60.904480</td>
      <td>27.381875</td>
    </tr>
  </tbody>
</table>
</div>



### Product Category Info Retrieval

Grouped by Store (Store_ID) and Product Hier Lvl1
- Average Daily Item Quantity / Total Daily Item Quantity



```python
query = """
  with PRODUCT_HIER_CTE
  as
  (
  select a.STORE_ID, a.BUSINESS_DATE, b.PRODUCT_HIER_LVL1, b.PRODUCT_HIER_LVL2, b.PRODUCT_HIER_LVL3, sum(a.ITEM_QUANTITY) SUM_ITEM_QUANTITY from 
  [watson].[dbo].[WATSON_FIN_TRANSACTION] a 
  inner join [watson].[dbo].[WATSON_FIN_PRODUCT] b on a.PRODUCT_ID = b.PRODUCT_ID
  group by a.STORE_ID, a.BUSINESS_DATE, b.PRODUCT_HIER_LVL1, b.PRODUCT_HIER_LVL2, b.PRODUCT_HIER_LVL3
  ) , 
  PRODUCT_HIER_DAILY_AVG_CTE as
  (select STORE_ID, REPLACE(UPPER(PRODUCT_HIER_LVL1),' ', '_') PRODUCT_HIER_LVL1, avg(SUM_ITEM_QUANTITY) AVG_DAILY_ITEM_QUANTITY
  from PRODUCT_HIER_CTE
  --where STORE_ID = 12328
  group by  STORE_ID, PRODUCT_HIER_LVL1
  ), PRODUCT_HIER_DAILY_AVG_TOTAL_CTE as
  (
  select STORE_ID, sum(AVG_DAILY_ITEM_QUANTITY) TOTAL
  from PRODUCT_HIER_DAILY_AVG_CTE
  group by STORE_ID
  )
  select cast(a.STORE_ID as varchar(10)) as STORE_ID, a.PRODUCT_HIER_LVL1, convert(float,a.AVG_DAILY_ITEM_QUANTITY) / convert(float,b.TOTAL) ITEM_QUANTITY_RATIO
  from
  PRODUCT_HIER_DAILY_AVG_CTE a inner join 
  PRODUCT_HIER_DAILY_AVG_TOTAL_CTE b on a.STORE_ID = b.STORE_ID
  
  """

PRODUCT_HIER_df = pd.read_sql(query, cnxn)
PRODUCT_HIER_df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>STORE_ID</th>
      <th>PRODUCT_HIER_LVL1</th>
      <th>ITEM_QUANTITY_RATIO</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>11896</td>
      <td>DAIRY_FROZEN</td>
      <td>0.022381</td>
    </tr>
    <tr>
      <th>1</th>
      <td>11896</td>
      <td>DINING</td>
      <td>0.008842</td>
    </tr>
    <tr>
      <th>2</th>
      <td>12172</td>
      <td>BASIC_GARMENT</td>
      <td>0.026395</td>
    </tr>
    <tr>
      <th>3</th>
      <td>12172</td>
      <td>DINING</td>
      <td>0.014938</td>
    </tr>
    <tr>
      <th>4</th>
      <td>11896</td>
      <td>OTHER_BRANDS</td>
      <td>0.005526</td>
    </tr>
  </tbody>
</table>
</div>



#### Turn PRODUCT_HIER_df to pivot_table
Row - Grouped by Store (Store_ID)

Column - Grouped by PRODUCT_HIER_LVL1

Value - Sum of ITEM_QUANTITY_RATIO


```python
PRODUCT_HIER_TABLE = pd.pivot_table(PRODUCT_HIER_df, values = 'ITEM_QUANTITY_RATIO', index = ['STORE_ID'], columns = ['PRODUCT_HIER_LVL1'])
PRODUCT_HIER_TABLE.head()

```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>PRODUCT_HIER_LVL1</th>
      <th>"</th>
      <th>ACCESSORIES</th>
      <th>ALCOHOL_&amp;_TOBACCO</th>
      <th>A_BRAND</th>
      <th>BAKERY</th>
      <th>BASIC_GARMENT</th>
      <th>BASIC_GROCERY</th>
      <th>BEDDING</th>
      <th>BEVERAGE_WINE&amp;SPIRIT</th>
      <th>BIG_APPLIANCE</th>
      <th>...</th>
      <th>NON_FOOD</th>
      <th>OATML&amp;BR_RICE"</th>
      <th>OTHER_BRANDS</th>
      <th>PERSONAL_CARE</th>
      <th>PRESERVED</th>
      <th>SEASONAL_IN/OUTDOOR</th>
      <th>SHOES</th>
      <th>SKIN_&amp;_COSMETIC</th>
      <th>SMALL_APPLIANCE</th>
      <th>_L</th>
    </tr>
    <tr>
      <th>STORE_ID</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>11668</th>
      <td>NaN</td>
      <td>0.020228</td>
      <td>NaN</td>
      <td>0.011908</td>
      <td>0.031827</td>
      <td>0.019968</td>
      <td>0.036778</td>
      <td>0.012783</td>
      <td>0.022075</td>
      <td>0.010092</td>
      <td>...</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>0.012110</td>
      <td>0.064996</td>
      <td>NaN</td>
      <td>0.014679</td>
      <td>0.013602</td>
      <td>0.046372</td>
      <td>0.041282</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>11669</th>
      <td>NaN</td>
      <td>0.020684</td>
      <td>NaN</td>
      <td>0.011902</td>
      <td>0.012818</td>
      <td>0.020814</td>
      <td>0.041162</td>
      <td>0.012924</td>
      <td>0.031286</td>
      <td>0.009970</td>
      <td>...</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>0.009970</td>
      <td>0.068244</td>
      <td>NaN</td>
      <td>0.015401</td>
      <td>0.022598</td>
      <td>0.042917</td>
      <td>0.028533</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>11671</th>
      <td>NaN</td>
      <td>0.038217</td>
      <td>NaN</td>
      <td>0.012568</td>
      <td>0.012582</td>
      <td>0.023839</td>
      <td>0.030833</td>
      <td>0.011469</td>
      <td>0.059046</td>
      <td>NaN</td>
      <td>...</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>0.013630</td>
      <td>0.066604</td>
      <td>NaN</td>
      <td>0.026626</td>
      <td>0.030668</td>
      <td>0.045603</td>
      <td>0.031388</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>11698</th>
      <td>NaN</td>
      <td>0.024362</td>
      <td>NaN</td>
      <td>0.012149</td>
      <td>0.008508</td>
      <td>0.013832</td>
      <td>0.031911</td>
      <td>0.007999</td>
      <td>0.037601</td>
      <td>0.003927</td>
      <td>...</td>
      <td>NaN</td>
      <td>0.006545</td>
      <td>0.010471</td>
      <td>0.084046</td>
      <td>NaN</td>
      <td>0.015763</td>
      <td>0.009817</td>
      <td>0.052168</td>
      <td>0.024167</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>11700</th>
      <td>NaN</td>
      <td>0.018034</td>
      <td>NaN</td>
      <td>0.017512</td>
      <td>0.019903</td>
      <td>0.019617</td>
      <td>0.033339</td>
      <td>0.011565</td>
      <td>0.020897</td>
      <td>0.009683</td>
      <td>...</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>0.012910</td>
      <td>0.058349</td>
      <td>NaN</td>
      <td>0.011321</td>
      <td>0.014847</td>
      <td>0.044607</td>
      <td>0.045692</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
<p>5 rows × 32 columns</p>
</div>



### Store Info Retrieval


```python
#cnxn = pyodbc.connect('DRIVER={SQL Server};SERVER={fastserver};DATABASE={watson};Trusted_Connection=yes;')
#query = 'select * from WATSON_FIN_WEEK'
query = """
    SELECT cast([STORE_ID] as varchar(10)) as [STORE_ID]
      ,[LOC_NAME]
      ,[ABBREVIATION]
      ,[REGION]
      ,[DISTRICT]
      ,[CITY]
      ,[TYPE]
      ,[SEGMENT]
      ,[NET_TRADING_AREA]
      ,[GROSS_AREA]
      ,[OPENING_DATE]
      ,[LAST_UPDATE_DATE]
      ,[CLOSING_DATE]
      ,[CURRENCY_CODE]
  FROM [watson].[dbo].[WATSON_FIN_STORE]
"""

df_fin_store = pd.read_sql(query, cnxn)
df_fin_store.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>STORE_ID</th>
      <th>LOC_NAME</th>
      <th>ABBREVIATION</th>
      <th>REGION</th>
      <th>DISTRICT</th>
      <th>CITY</th>
      <th>TYPE</th>
      <th>SEGMENT</th>
      <th>NET_TRADING_AREA</th>
      <th>GROSS_AREA</th>
      <th>OPENING_DATE</th>
      <th>LAST_UPDATE_DATE</th>
      <th>CLOSING_DATE</th>
      <th>CURRENCY_CODE</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>11668</td>
      <td>Siu Sai Wan SC</td>
      <td>232</td>
      <td>301</td>
      <td>301</td>
      <td>Hong Kong</td>
      <td>A</td>
      <td>WTC - Residential</td>
      <td>1536.0</td>
      <td>1711.0</td>
      <td>2008-09-30 16:00:00</td>
      <td>2017-05-02 16:00:00</td>
      <td>NaT</td>
      <td>HKD</td>
    </tr>
    <tr>
      <th>1</th>
      <td>11669</td>
      <td>Marina Square</td>
      <td>233</td>
      <td>301</td>
      <td>301</td>
      <td>AP LEI CHAU, HK</td>
      <td>A</td>
      <td>WTC - Residential</td>
      <td>2312.0</td>
      <td>2850.0</td>
      <td>1993-12-09 16:00:00</td>
      <td>2017-05-15 16:00:00</td>
      <td>NaT</td>
      <td>HKD</td>
    </tr>
    <tr>
      <th>2</th>
      <td>11670</td>
      <td>CENTRE MARK</td>
      <td>234</td>
      <td>306</td>
      <td>353</td>
      <td>CENTRAL, HK</td>
      <td>A</td>
      <td>WTC - Destination</td>
      <td>1359.0</td>
      <td>1800.0</td>
      <td>2003-11-27 16:00:00</td>
      <td>2017-05-14 16:00:00</td>
      <td>2012-10-20 16:00:00</td>
      <td>HKD</td>
    </tr>
    <tr>
      <th>3</th>
      <td>11671</td>
      <td>Tung Lo Wan Road</td>
      <td>235</td>
      <td>304</td>
      <td>331</td>
      <td>CAUSEWAY BAY, HK</td>
      <td>A</td>
      <td>WTC - Residential</td>
      <td>1947.0</td>
      <td>2500.0</td>
      <td>2004-06-18 16:00:00</td>
      <td>2017-05-02 16:00:00</td>
      <td>NaT</td>
      <td>HKD</td>
    </tr>
    <tr>
      <th>4</th>
      <td>11698</td>
      <td>Yuen Long</td>
      <td>654</td>
      <td>305</td>
      <td>342</td>
      <td>YUEN LONG, NT</td>
      <td>A</td>
      <td>WTC - Destination</td>
      <td>5716.0</td>
      <td>9270.0</td>
      <td>1986-09-22 16:00:00</td>
      <td>2017-11-24 16:00:00</td>
      <td>NaT</td>
      <td>HKD</td>
    </tr>
  </tbody>
</table>
</div>



## Data Exploration

Explore the Product Hier with NaN more than 90% in all the stores


```python
PRODUCT_HIER_TABLE_NAN_NUM = PRODUCT_HIER_TABLE.isnull().sum()
PRODUCT_HIER_TABLE_NAN_RATIO = PRODUCT_HIER_TABLE_NAN_NUM/PRODUCT_HIER_TABLE.shape[0]
print(PRODUCT_HIER_TABLE_NAN_RATIO[PRODUCT_HIER_TABLE_NAN_RATIO > 0.9])

print("In case the clustering resultis not satisified, we may consider to remove some groups")


```

    PRODUCT_HIER_LVL1
    "                    0.988593
    ALCOHOL_&_TOBACCO    0.996198
    DIGITAL              0.958175
    EXCLUSIVE_BRAND      0.984791
    L                    0.996198
    NON_FOOD             0.946768
    OATML&BR_RICE"       0.977186
    PRESERVED            0.996198
    _L                   0.984791
    dtype: float64
    In case the clustering resultis not satisified, we may consider to remove some groups
    


```python
print("Remove meaningless group with high nan ratio, including "", L and _L")

del PRODUCT_HIER_TABLE['"']
del PRODUCT_HIER_TABLE['L']
del PRODUCT_HIER_TABLE['_L']

print("Rename column names containing symbol, including, %, / and """)
print("Before, column names")
print(PRODUCT_HIER_TABLE.columns)
PRODUCT_HIER_TABLE = PRODUCT_HIER_TABLE.rename(columns = {'OATML&BR_RICE"': 'OATML_AND_BR_RICE'})
PRODUCT_HIER_TABLE = PRODUCT_HIER_TABLE.rename(columns = {'ALCOHOL_&_TOBACCO': 'ALCOHOL_AND_TOBACCO'})
PRODUCT_HIER_TABLE = PRODUCT_HIER_TABLE.rename(columns = {'BEVERAGE_WINE&SPIRIT': 'BEVERAGE_WINE_AND_SPIRIT'})
PRODUCT_HIER_TABLE = PRODUCT_HIER_TABLE.rename(columns = {'SEASONAL_IN/OUTDOOR': 'SEASONAL_IN_OR_OUTDOOR'})
PRODUCT_HIER_TABLE = PRODUCT_HIER_TABLE.rename(columns = {'SKIN_&_COSMETIC': 'SKIN_AND_COSMETIC'})

print("Refreshed column names")
print(PRODUCT_HIER_TABLE.columns)
```

    Remove meaningless group with high nan ratio, including , L and _L
    Rename column names containing symbol, including, %, / and 
    Before, column names
    Index(['ACCESSORIES', 'ALCOHOL_&_TOBACCO', 'A_BRAND', 'BAKERY',
           'BASIC_GARMENT', 'BASIC_GROCERY', 'BEDDING', 'BEVERAGE_WINE&SPIRIT',
           'BIG_APPLIANCE', 'DAIRY_FROZEN', 'DEPT_9', 'DIGITAL', 'DINING',
           'EXCLUSIVE_BRAND', 'HOME_TEXTILE', 'HOME_WORLD', 'HOUSEHOLD_CONSUMABLE',
           'KITCHEN', 'LIFESTYLE_GROCERY', 'MEDICINE', 'NON_FOOD',
           'OATML&BR_RICE"', 'OTHER_BRANDS', 'PERSONAL_CARE', 'PRESERVED',
           'SEASONAL_IN/OUTDOOR', 'SHOES', 'SKIN_&_COSMETIC', 'SMALL_APPLIANCE'],
          dtype='object', name='PRODUCT_HIER_LVL1')
    Refreshed column names
    Index(['ACCESSORIES', 'ALCOHOL_AND_TOBACCO', 'A_BRAND', 'BAKERY',
           'BASIC_GARMENT', 'BASIC_GROCERY', 'BEDDING', 'BEVERAGE_WINE_AND_SPIRIT',
           'BIG_APPLIANCE', 'DAIRY_FROZEN', 'DEPT_9', 'DIGITAL', 'DINING',
           'EXCLUSIVE_BRAND', 'HOME_TEXTILE', 'HOME_WORLD', 'HOUSEHOLD_CONSUMABLE',
           'KITCHEN', 'LIFESTYLE_GROCERY', 'MEDICINE', 'NON_FOOD',
           'OATML_AND_BR_RICE', 'OTHER_BRANDS', 'PERSONAL_CARE', 'PRESERVED',
           'SEASONAL_IN_OR_OUTDOOR', 'SHOES', 'SKIN_AND_COSMETIC',
           'SMALL_APPLIANCE'],
          dtype='object', name='PRODUCT_HIER_LVL1')
    


```python
print("Fill PRODUCT_HIER_TABLE NaN with 0")
PRODUCT_HIER_TABLE =PRODUCT_HIER_TABLE.fillna(0)
PRODUCT_HIER_TABLE
PRODUCT_HIER_TABLE = PRODUCT_HIER_TABLE.reset_index()
PRODUCT_HIER_TABLE.head()
```

    Fill PRODUCT_HIER_TABLE NaN with 0
    




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>PRODUCT_HIER_LVL1</th>
      <th>STORE_ID</th>
      <th>ACCESSORIES</th>
      <th>ALCOHOL_AND_TOBACCO</th>
      <th>A_BRAND</th>
      <th>BAKERY</th>
      <th>BASIC_GARMENT</th>
      <th>BASIC_GROCERY</th>
      <th>BEDDING</th>
      <th>BEVERAGE_WINE_AND_SPIRIT</th>
      <th>BIG_APPLIANCE</th>
      <th>...</th>
      <th>MEDICINE</th>
      <th>NON_FOOD</th>
      <th>OATML_AND_BR_RICE</th>
      <th>OTHER_BRANDS</th>
      <th>PERSONAL_CARE</th>
      <th>PRESERVED</th>
      <th>SEASONAL_IN_OR_OUTDOOR</th>
      <th>SHOES</th>
      <th>SKIN_AND_COSMETIC</th>
      <th>SMALL_APPLIANCE</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>11668</td>
      <td>0.020228</td>
      <td>0.0</td>
      <td>0.011908</td>
      <td>0.031827</td>
      <td>0.019968</td>
      <td>0.036778</td>
      <td>0.012783</td>
      <td>0.022075</td>
      <td>0.010092</td>
      <td>...</td>
      <td>0.039250</td>
      <td>0.0</td>
      <td>0.000000</td>
      <td>0.012110</td>
      <td>0.064996</td>
      <td>0.0</td>
      <td>0.014679</td>
      <td>0.013602</td>
      <td>0.046372</td>
      <td>0.041282</td>
    </tr>
    <tr>
      <th>1</th>
      <td>11669</td>
      <td>0.020684</td>
      <td>0.0</td>
      <td>0.011902</td>
      <td>0.012818</td>
      <td>0.020814</td>
      <td>0.041162</td>
      <td>0.012924</td>
      <td>0.031286</td>
      <td>0.009970</td>
      <td>...</td>
      <td>0.037321</td>
      <td>0.0</td>
      <td>0.000000</td>
      <td>0.009970</td>
      <td>0.068244</td>
      <td>0.0</td>
      <td>0.015401</td>
      <td>0.022598</td>
      <td>0.042917</td>
      <td>0.028533</td>
    </tr>
    <tr>
      <th>2</th>
      <td>11671</td>
      <td>0.038217</td>
      <td>0.0</td>
      <td>0.012568</td>
      <td>0.012582</td>
      <td>0.023839</td>
      <td>0.030833</td>
      <td>0.011469</td>
      <td>0.059046</td>
      <td>0.000000</td>
      <td>...</td>
      <td>0.056374</td>
      <td>0.0</td>
      <td>0.000000</td>
      <td>0.013630</td>
      <td>0.066604</td>
      <td>0.0</td>
      <td>0.026626</td>
      <td>0.030668</td>
      <td>0.045603</td>
      <td>0.031388</td>
    </tr>
    <tr>
      <th>3</th>
      <td>11698</td>
      <td>0.024362</td>
      <td>0.0</td>
      <td>0.012149</td>
      <td>0.008508</td>
      <td>0.013832</td>
      <td>0.031911</td>
      <td>0.007999</td>
      <td>0.037601</td>
      <td>0.003927</td>
      <td>...</td>
      <td>0.043681</td>
      <td>0.0</td>
      <td>0.006545</td>
      <td>0.010471</td>
      <td>0.084046</td>
      <td>0.0</td>
      <td>0.015763</td>
      <td>0.009817</td>
      <td>0.052168</td>
      <td>0.024167</td>
    </tr>
    <tr>
      <th>4</th>
      <td>11700</td>
      <td>0.018034</td>
      <td>0.0</td>
      <td>0.017512</td>
      <td>0.019903</td>
      <td>0.019617</td>
      <td>0.033339</td>
      <td>0.011565</td>
      <td>0.020897</td>
      <td>0.009683</td>
      <td>...</td>
      <td>0.030886</td>
      <td>0.0</td>
      <td>0.000000</td>
      <td>0.012910</td>
      <td>0.058349</td>
      <td>0.0</td>
      <td>0.011321</td>
      <td>0.014847</td>
      <td>0.044607</td>
      <td>0.045692</td>
    </tr>
  </tbody>
</table>
<p>5 rows × 30 columns</p>
</div>



### Pre-Processing Data
As the magnitude of some of Data Dimension is large (>500) compared to others ( between 0 and 1 ), for doing clustering like K-Means, it is required to do the pre-processing to convert the value from large (>500) to between 0 and 1.

#### Pre-Processing ITEM TRANSACTION Records
All Columns in this table is not having the range between 0 and 1, we will do the pre-processing (rescale) of all columns.


```python
ITEM_TRAN_df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>STORE_ID</th>
      <th>AVG_SUM_PRODUCT_ID</th>
      <th>STDEV_SUM_PRODUCT_ID</th>
      <th>AVG_SUM_ITEM_QUANTITY</th>
      <th>STDEV_SUM_ITEM_QUANTITY</th>
      <th>AVG_SUM_ITEM_AMOUNT</th>
      <th>STDEV_SUM_ITEM_AMOUNT</th>
      <th>AVG_SUM_ITEM_DISCOUNT_AMOUNT</th>
      <th>STDEV_SUM_ITEM_DISCOUNT_AMOUNT</th>
      <th>AVG_SEG_NUM</th>
      <th>STDEV_SEG_NUM</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>11955</td>
      <td>1.724710</td>
      <td>1.370606</td>
      <td>2.770968</td>
      <td>7.835618</td>
      <td>113.434108</td>
      <td>231.057269</td>
      <td>16.133859</td>
      <td>96.308430</td>
      <td>3</td>
      <td>1.831615</td>
    </tr>
    <tr>
      <th>1</th>
      <td>11972</td>
      <td>1.645778</td>
      <td>1.399311</td>
      <td>2.249968</td>
      <td>7.003476</td>
      <td>98.383218</td>
      <td>304.284756</td>
      <td>6.553552</td>
      <td>50.989477</td>
      <td>3</td>
      <td>1.682894</td>
    </tr>
    <tr>
      <th>2</th>
      <td>12272</td>
      <td>1.919302</td>
      <td>1.764426</td>
      <td>2.959445</td>
      <td>7.996145</td>
      <td>166.950796</td>
      <td>396.427580</td>
      <td>18.517122</td>
      <td>120.005615</td>
      <td>3</td>
      <td>1.958209</td>
    </tr>
    <tr>
      <th>3</th>
      <td>12172</td>
      <td>2.241608</td>
      <td>2.060455</td>
      <td>2.967163</td>
      <td>4.868277</td>
      <td>164.809825</td>
      <td>389.157374</td>
      <td>14.523409</td>
      <td>79.275861</td>
      <td>3</td>
      <td>1.893091</td>
    </tr>
    <tr>
      <th>4</th>
      <td>12193</td>
      <td>1.748568</td>
      <td>1.474737</td>
      <td>3.247996</td>
      <td>10.545632</td>
      <td>158.936115</td>
      <td>489.315244</td>
      <td>16.628632</td>
      <td>100.305847</td>
      <td>3</td>
      <td>1.858842</td>
    </tr>
  </tbody>
</table>
</div>




```python
ITEM_TRAN_pre_processed_df = ITEM_TRAN_df.copy(deep=True)

#Create a list of columns that requires pre-processing, except STORE_ID
pre_processing_cols = list(ITEM_TRAN_pre_processed_df.columns)
pre_processing_cols.remove('STORE_ID')
pre_processing_cols

ITEM_TRAN_pre_processed_df[pre_processing_cols]=preprocessing.scale(ITEM_TRAN_pre_processed_df[pre_processing_cols])

ITEM_TRAN_pre_processed_df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>STORE_ID</th>
      <th>AVG_SUM_PRODUCT_ID</th>
      <th>STDEV_SUM_PRODUCT_ID</th>
      <th>AVG_SUM_ITEM_QUANTITY</th>
      <th>STDEV_SUM_ITEM_QUANTITY</th>
      <th>AVG_SUM_ITEM_AMOUNT</th>
      <th>STDEV_SUM_ITEM_AMOUNT</th>
      <th>AVG_SUM_ITEM_DISCOUNT_AMOUNT</th>
      <th>STDEV_SUM_ITEM_DISCOUNT_AMOUNT</th>
      <th>AVG_SEG_NUM</th>
      <th>STDEV_SEG_NUM</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>11955</td>
      <td>-0.646365</td>
      <td>-0.798156</td>
      <td>-0.609078</td>
      <td>-0.103066</td>
      <td>-0.101473</td>
      <td>-0.098244</td>
      <td>-0.331138</td>
      <td>-0.136354</td>
      <td>-0.1016</td>
      <td>-0.161135</td>
    </tr>
    <tr>
      <th>1</th>
      <td>11972</td>
      <td>-0.806892</td>
      <td>-0.737811</td>
      <td>-1.194570</td>
      <td>-0.173136</td>
      <td>-0.110491</td>
      <td>-0.087500</td>
      <td>-0.984918</td>
      <td>-0.968525</td>
      <td>-0.1016</td>
      <td>-1.493298</td>
    </tr>
    <tr>
      <th>2</th>
      <td>12272</td>
      <td>-0.250618</td>
      <td>0.029739</td>
      <td>-0.397270</td>
      <td>-0.089549</td>
      <td>-0.069408</td>
      <td>-0.073981</td>
      <td>-0.168499</td>
      <td>0.298786</td>
      <td>-0.1016</td>
      <td>0.972815</td>
    </tr>
    <tr>
      <th>3</th>
      <td>12172</td>
      <td>0.404866</td>
      <td>0.652058</td>
      <td>-0.388596</td>
      <td>-0.352929</td>
      <td>-0.070691</td>
      <td>-0.075048</td>
      <td>-0.441039</td>
      <td>-0.449116</td>
      <td>-0.1016</td>
      <td>0.389528</td>
    </tr>
    <tr>
      <th>4</th>
      <td>12193</td>
      <td>-0.597845</td>
      <td>-0.579250</td>
      <td>-0.073001</td>
      <td>0.125128</td>
      <td>-0.074210</td>
      <td>-0.060352</td>
      <td>-0.297374</td>
      <td>-0.062952</td>
      <td>-0.1016</td>
      <td>0.082747</td>
    </tr>
  </tbody>
</table>
</div>



#### Pre-Processing Member Personal Info Records
All Columns in this table is not having the range between 0 and 1, we will do the pre-processing (rescale) of AVG_AGE & STDEV_AGE.


```python
MEMBER_df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>STORE_ID</th>
      <th>MAIN_MEMBER_IN_TRAN_RATIO</th>
      <th>MEMBER_PERCENTAGE</th>
      <th>M_GENDER_RATIO</th>
      <th>AVG_AGE</th>
      <th>STDEV_AGE</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>12172</td>
      <td>0.649773</td>
      <td>0.222892</td>
      <td>0.182826</td>
      <td>63.741199</td>
      <td>34.545323</td>
    </tr>
    <tr>
      <th>1</th>
      <td>12193</td>
      <td>0.475164</td>
      <td>0.201638</td>
      <td>0.180780</td>
      <td>59.396990</td>
      <td>28.529972</td>
    </tr>
    <tr>
      <th>2</th>
      <td>12073</td>
      <td>0.386787</td>
      <td>0.435959</td>
      <td>0.169109</td>
      <td>57.723281</td>
      <td>24.167924</td>
    </tr>
    <tr>
      <th>3</th>
      <td>12271</td>
      <td>0.432872</td>
      <td>0.321882</td>
      <td>0.179190</td>
      <td>58.963774</td>
      <td>27.343983</td>
    </tr>
    <tr>
      <th>4</th>
      <td>11902</td>
      <td>0.555759</td>
      <td>0.344396</td>
      <td>0.135206</td>
      <td>60.904480</td>
      <td>27.381875</td>
    </tr>
  </tbody>
</table>
</div>




```python
pre_processing_cols = ['AVG_AGE', 'STDEV_AGE']

MEMBER_pre_processed_df = MEMBER_df.copy(deep=True)

MEMBER_pre_processed_df[pre_processing_cols]=preprocessing.scale(MEMBER_df[pre_processing_cols])

MEMBER_pre_processed_df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>STORE_ID</th>
      <th>MAIN_MEMBER_IN_TRAN_RATIO</th>
      <th>MEMBER_PERCENTAGE</th>
      <th>M_GENDER_RATIO</th>
      <th>AVG_AGE</th>
      <th>STDEV_AGE</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>12172</td>
      <td>0.649773</td>
      <td>0.222892</td>
      <td>0.182826</td>
      <td>1.583667</td>
      <td>2.076176</td>
    </tr>
    <tr>
      <th>1</th>
      <td>12193</td>
      <td>0.475164</td>
      <td>0.201638</td>
      <td>0.180780</td>
      <td>0.296808</td>
      <td>0.043197</td>
    </tr>
    <tr>
      <th>2</th>
      <td>12073</td>
      <td>0.386787</td>
      <td>0.435959</td>
      <td>0.169109</td>
      <td>-0.198985</td>
      <td>-1.431023</td>
    </tr>
    <tr>
      <th>3</th>
      <td>12271</td>
      <td>0.432872</td>
      <td>0.321882</td>
      <td>0.179190</td>
      <td>0.168479</td>
      <td>-0.357626</td>
    </tr>
    <tr>
      <th>4</th>
      <td>11902</td>
      <td>0.555759</td>
      <td>0.344396</td>
      <td>0.135206</td>
      <td>0.743363</td>
      <td>-0.344820</td>
    </tr>
  </tbody>
</table>
</div>



### Merge DataFrames into one Big DataFrame


```python
rtn = pd.merge(MEMBER_df, PRODUCT_HIER_TABLE, on = 'STORE_ID')
rtn = pd.merge(rtn, SEG_NUM_df, on = 'STORE_ID')
rtn = pd.merge(rtn, ITEM_TRAN_df, on = 'STORE_ID')
rtn.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>STORE_ID</th>
      <th>MAIN_MEMBER_IN_TRAN_RATIO</th>
      <th>MEMBER_PERCENTAGE</th>
      <th>M_GENDER_RATIO</th>
      <th>AVG_AGE</th>
      <th>STDEV_AGE</th>
      <th>ACCESSORIES</th>
      <th>ALCOHOL_AND_TOBACCO</th>
      <th>A_BRAND</th>
      <th>BAKERY</th>
      <th>...</th>
      <th>AVG_SUM_PRODUCT_ID</th>
      <th>STDEV_SUM_PRODUCT_ID</th>
      <th>AVG_SUM_ITEM_QUANTITY</th>
      <th>STDEV_SUM_ITEM_QUANTITY</th>
      <th>AVG_SUM_ITEM_AMOUNT</th>
      <th>STDEV_SUM_ITEM_AMOUNT</th>
      <th>AVG_SUM_ITEM_DISCOUNT_AMOUNT</th>
      <th>STDEV_SUM_ITEM_DISCOUNT_AMOUNT</th>
      <th>AVG_SEG_NUM</th>
      <th>STDEV_SEG_NUM</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>12172</td>
      <td>0.649773</td>
      <td>0.222892</td>
      <td>0.182826</td>
      <td>63.741199</td>
      <td>34.545323</td>
      <td>0.066544</td>
      <td>0.0</td>
      <td>0.015323</td>
      <td>0.000000</td>
      <td>...</td>
      <td>2.241608</td>
      <td>2.060455</td>
      <td>2.967163</td>
      <td>4.868277</td>
      <td>164.809825</td>
      <td>389.157374</td>
      <td>14.523409</td>
      <td>79.275861</td>
      <td>3</td>
      <td>1.893091</td>
    </tr>
    <tr>
      <th>1</th>
      <td>12193</td>
      <td>0.475164</td>
      <td>0.201638</td>
      <td>0.180780</td>
      <td>59.396990</td>
      <td>28.529972</td>
      <td>0.036787</td>
      <td>0.0</td>
      <td>0.020191</td>
      <td>0.014422</td>
      <td>...</td>
      <td>1.748568</td>
      <td>1.474737</td>
      <td>3.247996</td>
      <td>10.545632</td>
      <td>158.936115</td>
      <td>489.315244</td>
      <td>16.628632</td>
      <td>100.305847</td>
      <td>3</td>
      <td>1.858842</td>
    </tr>
    <tr>
      <th>2</th>
      <td>12073</td>
      <td>0.386787</td>
      <td>0.435959</td>
      <td>0.169109</td>
      <td>57.723281</td>
      <td>24.167924</td>
      <td>0.025088</td>
      <td>0.0</td>
      <td>0.016627</td>
      <td>0.023698</td>
      <td>...</td>
      <td>1.721019</td>
      <td>1.285594</td>
      <td>2.513918</td>
      <td>5.680536</td>
      <td>123.395800</td>
      <td>242.513382</td>
      <td>16.925127</td>
      <td>77.792761</td>
      <td>3</td>
      <td>1.839475</td>
    </tr>
    <tr>
      <th>3</th>
      <td>12271</td>
      <td>0.432872</td>
      <td>0.321882</td>
      <td>0.179190</td>
      <td>58.963774</td>
      <td>27.343983</td>
      <td>0.022752</td>
      <td>0.0</td>
      <td>0.014879</td>
      <td>0.022219</td>
      <td>...</td>
      <td>1.747741</td>
      <td>1.461141</td>
      <td>3.142284</td>
      <td>15.265728</td>
      <td>161.648746</td>
      <td>404.930002</td>
      <td>20.785438</td>
      <td>118.243042</td>
      <td>3</td>
      <td>1.961912</td>
    </tr>
    <tr>
      <th>4</th>
      <td>11902</td>
      <td>0.555759</td>
      <td>0.344396</td>
      <td>0.135206</td>
      <td>60.904480</td>
      <td>27.381875</td>
      <td>0.018324</td>
      <td>0.0</td>
      <td>0.014543</td>
      <td>0.016361</td>
      <td>...</td>
      <td>1.837831</td>
      <td>1.559796</td>
      <td>3.211683</td>
      <td>8.707156</td>
      <td>122.585896</td>
      <td>336.737403</td>
      <td>18.770367</td>
      <td>94.457921</td>
      <td>3</td>
      <td>1.923020</td>
    </tr>
  </tbody>
</table>
<p>5 rows × 53 columns</p>
</div>




```python
rtn_pre_processed = pd.merge(MEMBER_pre_processed_df, PRODUCT_HIER_TABLE, on = 'STORE_ID')
rtn_pre_processed = pd.merge(rtn_pre_processed, SEG_NUM_df, on = 'STORE_ID')
rtn_pre_processed = pd.merge(rtn_pre_processed, ITEM_TRAN_pre_processed_df, on = 'STORE_ID')
rtn_pre_processed.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>STORE_ID</th>
      <th>MAIN_MEMBER_IN_TRAN_RATIO</th>
      <th>MEMBER_PERCENTAGE</th>
      <th>M_GENDER_RATIO</th>
      <th>AVG_AGE</th>
      <th>STDEV_AGE</th>
      <th>ACCESSORIES</th>
      <th>ALCOHOL_AND_TOBACCO</th>
      <th>A_BRAND</th>
      <th>BAKERY</th>
      <th>...</th>
      <th>AVG_SUM_PRODUCT_ID</th>
      <th>STDEV_SUM_PRODUCT_ID</th>
      <th>AVG_SUM_ITEM_QUANTITY</th>
      <th>STDEV_SUM_ITEM_QUANTITY</th>
      <th>AVG_SUM_ITEM_AMOUNT</th>
      <th>STDEV_SUM_ITEM_AMOUNT</th>
      <th>AVG_SUM_ITEM_DISCOUNT_AMOUNT</th>
      <th>STDEV_SUM_ITEM_DISCOUNT_AMOUNT</th>
      <th>AVG_SEG_NUM</th>
      <th>STDEV_SEG_NUM</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>12172</td>
      <td>0.649773</td>
      <td>0.222892</td>
      <td>0.182826</td>
      <td>1.583667</td>
      <td>2.076176</td>
      <td>0.066544</td>
      <td>0.0</td>
      <td>0.015323</td>
      <td>0.000000</td>
      <td>...</td>
      <td>0.404866</td>
      <td>0.652058</td>
      <td>-0.388596</td>
      <td>-0.352929</td>
      <td>-0.070691</td>
      <td>-0.075048</td>
      <td>-0.441039</td>
      <td>-0.449116</td>
      <td>-0.1016</td>
      <td>0.389528</td>
    </tr>
    <tr>
      <th>1</th>
      <td>12193</td>
      <td>0.475164</td>
      <td>0.201638</td>
      <td>0.180780</td>
      <td>0.296808</td>
      <td>0.043197</td>
      <td>0.036787</td>
      <td>0.0</td>
      <td>0.020191</td>
      <td>0.014422</td>
      <td>...</td>
      <td>-0.597845</td>
      <td>-0.579250</td>
      <td>-0.073001</td>
      <td>0.125128</td>
      <td>-0.074210</td>
      <td>-0.060352</td>
      <td>-0.297374</td>
      <td>-0.062952</td>
      <td>-0.1016</td>
      <td>0.082747</td>
    </tr>
    <tr>
      <th>2</th>
      <td>12073</td>
      <td>0.386787</td>
      <td>0.435959</td>
      <td>0.169109</td>
      <td>-0.198985</td>
      <td>-1.431023</td>
      <td>0.025088</td>
      <td>0.0</td>
      <td>0.016627</td>
      <td>0.023698</td>
      <td>...</td>
      <td>-0.653872</td>
      <td>-0.976870</td>
      <td>-0.897947</td>
      <td>-0.284533</td>
      <td>-0.095504</td>
      <td>-0.096563</td>
      <td>-0.277140</td>
      <td>-0.476349</td>
      <td>-0.1016</td>
      <td>-0.090730</td>
    </tr>
    <tr>
      <th>3</th>
      <td>12271</td>
      <td>0.432872</td>
      <td>0.321882</td>
      <td>0.179190</td>
      <td>0.168479</td>
      <td>-0.357626</td>
      <td>0.022752</td>
      <td>0.0</td>
      <td>0.014879</td>
      <td>0.022219</td>
      <td>...</td>
      <td>-0.599527</td>
      <td>-0.607832</td>
      <td>-0.191798</td>
      <td>0.522580</td>
      <td>-0.072585</td>
      <td>-0.072733</td>
      <td>-0.013705</td>
      <td>0.266421</td>
      <td>-0.1016</td>
      <td>1.005988</td>
    </tr>
    <tr>
      <th>4</th>
      <td>11902</td>
      <td>0.555759</td>
      <td>0.344396</td>
      <td>0.135206</td>
      <td>0.743363</td>
      <td>-0.344820</td>
      <td>0.018324</td>
      <td>0.0</td>
      <td>0.014543</td>
      <td>0.016361</td>
      <td>...</td>
      <td>-0.416308</td>
      <td>-0.400438</td>
      <td>-0.113809</td>
      <td>-0.029679</td>
      <td>-0.095989</td>
      <td>-0.082739</td>
      <td>-0.151217</td>
      <td>-0.170334</td>
      <td>-0.1016</td>
      <td>0.657620</td>
    </tr>
  </tbody>
</table>
<p>5 rows × 53 columns</p>
</div>




```python
print('Big DataFrame Column Names')
rtn.columns
```

    Big DataFrame Column Names
    




    Index(['STORE_ID', 'MAIN_MEMBER_IN_TRAN_RATIO', 'MEMBER_PERCENTAGE',
           'M_GENDER_RATIO', 'AVG_AGE', 'STDEV_AGE', 'ACCESSORIES',
           'ALCOHOL_AND_TOBACCO', 'A_BRAND', 'BAKERY', 'BASIC_GARMENT',
           'BASIC_GROCERY', 'BEDDING', 'BEVERAGE_WINE_AND_SPIRIT', 'BIG_APPLIANCE',
           'DAIRY_FROZEN', 'DEPT_9', 'DIGITAL', 'DINING', 'EXCLUSIVE_BRAND',
           'HOME_TEXTILE', 'HOME_WORLD', 'HOUSEHOLD_CONSUMABLE', 'KITCHEN',
           'LIFESTYLE_GROCERY', 'MEDICINE', 'NON_FOOD', 'OATML_AND_BR_RICE',
           'OTHER_BRANDS', 'PERSONAL_CARE', 'PRESERVED', 'SEASONAL_IN_OR_OUTDOOR',
           'SHOES', 'SKIN_AND_COSMETIC', 'SMALL_APPLIANCE', 'SEG_1', 'SEG_2',
           'SEG_3', 'SEG_4', 'SEG_5', 'SEG_6', 'SEG_7', 'SEG_8',
           'AVG_SUM_PRODUCT_ID', 'STDEV_SUM_PRODUCT_ID', 'AVG_SUM_ITEM_QUANTITY',
           'STDEV_SUM_ITEM_QUANTITY', 'AVG_SUM_ITEM_AMOUNT',
           'STDEV_SUM_ITEM_AMOUNT', 'AVG_SUM_ITEM_DISCOUNT_AMOUNT',
           'STDEV_SUM_ITEM_DISCOUNT_AMOUNT', 'AVG_SEG_NUM', 'STDEV_SEG_NUM'],
          dtype='object')



### Define Columns to be feed to algorithm

feat_cols - columns feed to the algorithm

all_cols - all columns (included Store_ID)


```python
def WithoutStoreID(colnames):
    rtn_list = list(colnames)
    rtn_list.remove('STORE_ID')
    return rtn_list
```


```python
feat_cols = WithoutStoreID(rtn.columns)
all_cols = list(rtn.columns)
```

### Prepare Methods/Functions 
We define an unified interface to cluster and reduce dimension

**Cluster** - interface to run clustering, by 'Meanshift', 'KMEANS', 'HDBSCAN'

**ReduceDimention** - interface to reduce dimension, by 'PCA', 'TSNE'


```python

def Cluster(method, data_frame, feat_cols, define_cluster_size=None):
    if ( method.upper() == 'MEANSHIFT'):
        ms = MeanShift()
        ms.fit(data_frame[feat_cols])
        labels = ms.labels_
        cluster_centers = ms.cluster_centers_
        return labels
    #else ( method.upper() == 'PCA')
    elif ( method.upper() =='KMEANS'):
        k_means = cluster.KMeans(n_clusters=define_cluster_size)
        k_means.fit(data_frame[feat_cols]) 
        labels = k_means.labels_         
        return labels
    elif ( method.upper() == "HDBSCAN"):
        clusterer = hdbscan.HDBSCAN(min_cluster_size = define_cluster_size)
        clusterer.fit(data_frame[feat_cols])
        labels = clusterer.labels_
        return labels
    else:
        return None
        
def ReduceDimension(method, data_frame, feat_cols, define_components_size=None):
    if ( method.upper() == 'PCA'):
        pca = PCA(n_components=define_components_size)
        pca_result = pca.fit_transform(data_frame[feat_cols].values)
        return (pca,pca_result)
    elif ( method.upper() == 'TSNE' ):
        tsne = manifold.TSNE(n_components=define_components_size)
        tsne_result = tsne.fit_transform(data_frame[feat_cols].values)
        return (tsne, tsne_result)
    return None
        
    
```

**RunClusteringAndRDChart** - Run Clustering and Create Reduce Dimensions Chart


```python
def RunClusteringAndRDChart(running_df, Clustering_Method, Clustering_Gp_Num, Reducing_Dimenion_Method):
    #running_df = MEMBER_df
    running_feat_cols = WithoutStoreID(running_df.columns)


    k_means_labels = Cluster(Clustering_Method, running_df, running_feat_cols, Clustering_Gp_Num)
    (pca, pca_result) = ReduceDimension(Reducing_Dimenion_Method, running_df, running_feat_cols, 2)

    #KMeans_Data = running_df[running_feat_cols]
    KMeans_Data = running_df
    KMeans_Data['label'] = k_means_labels
    KMeans_Data['label'] = KMeans_Data['label'].apply(lambda i: str(i))
    KMeans_Data['rd-one'] = pca_result[:,0]
    KMeans_Data['rd-two'] = pca_result[:,1] 
    #KMeans_Data['STORE_ID'] = running_df['STORE_ID']

    #X['pca-three'] = pca_result[:,2]

    #print('Explained variation per principal component: {}'.format(pca.explained_variance_ratio_))    

    chart = ggplot( KMeans_Data , aes(x='rd-one', y='rd-two', color='label') ) \
            + geom_point(size=75,alpha=0.8) \
            + ggtitle("First and Second Principal Components colored by digit \n Clustering:" + Clustering_Method + ", Reduce Dimension:" + Reducing_Dimenion_Method)
    return (KMeans_Data, chart)
```

**CreateDataAnalysis** - Use the result of Clustering and Create Tables/Matrix for analysis


```python
def CreateDataAnalysis(labelData, storeData):
    STORE_CUST_LABEL= pd.merge(left=labelData, right=storeData, left_on='STORE_ID', right_on='STORE_ID', how='inner')    
    STORE_CUST_LABEL_SEGMENT_SIZE = STORE_CUST_LABEL.groupby(['label', 'SEGMENT']).size().reset_index()
    STORE_CUST_LABEL_SEGMENT_SIZE.columns = ['label', 'SEGMENT', 'COUNT']

    #C = STORE_CUST_LABEL.groupby(['label', 'SEGMENT'])['label'].agg(['size']).reset_index()
    #D = STORE_CUST_LABEL.groupby(['label'])['label'].agg(['size']).reset_index()
    #D.columns = ['label', 'gp_size']
    #E = pd.merge(left=C, right=D, left_on = 'label', right_on = 'label', how='inner')
    #E['power'] = E['size'] / E['gp_size']
    #B = A.groupby('label', sort=False)['COUNT'].max().reset_index()    
    LABEL_SIZE = STORE_CUST_LABEL.groupby(['label'])['label'].agg(['size']).reset_index()
    LABEL_SIZE.columns = ['label', 'label_SIZE']

    SEGMENT_SIZE = STORE_CUST_LABEL.groupby(['SEGMENT'])['SEGMENT'].agg(['size']).reset_index()
    SEGMENT_SIZE.columns = ['SEGMENT', 'SEGMENT_SIZE']

    RTN = pd.merge(left=STORE_CUST_LABEL_SEGMENT_SIZE, right=LABEL_SIZE, left_on = 'label', right_on = 'label', how='inner')
    RTN = pd.merge(left=RTN, right=SEGMENT_SIZE, left_on = 'SEGMENT', right_on = 'SEGMENT', how='inner')

    RTN['label_power'] = RTN['COUNT'] / RTN['label_SIZE']
    RTN['SEGMENT_power'] = RTN['COUNT'] / RTN['SEGMENT_SIZE']
    RTN['MULTIPLE_power'] = RTN['label_power']  * RTN['SEGMENT_power']
    
    maxPowerGpBySegment = RTN.groupby(['SEGMENT'], sort=False)['MULTIPLE_power'].max().reset_index()
    segmentLabelOptimal = pd.merge(left = RTN, right = maxPowerGpBySegment, left_on = ['SEGMENT', 'MULTIPLE_power'], right_on =['SEGMENT', 'MULTIPLE_power'])
    
    confusionMatrixByCount = pd.pivot_table(RTN, values=['COUNT'], index=['label'],columns=['SEGMENT'], aggfunc=np.sum).fillna(0)
    confusionMatrixByLabelPower = pd.pivot_table(RTN, values=['label_power'], index=['label'],columns=['SEGMENT'], aggfunc=np.sum).fillna(0)
    confusionMatrixBySegmentPower = pd.pivot_table(RTN, values=['SEGMENT_power'], index=['label'],columns=['SEGMENT'], aggfunc=np.sum).fillna(0)
    return (RTN, confusionMatrixByCount, confusionMatrixByLabelPower,confusionMatrixBySegmentPower, segmentLabelOptimal)
        
```


```python
def MergeByStoreID(left, right=None):
    if ( right is  None ):
        return left.copy(deep=True)
    else: 
        return pd.merge(left = left, right = right, on = 'STORE_ID')
```

## Run Clustering & Reduce Dimension for Charting

### Clustering: MeanShift, Reduce Diemsnion: PCA
The Advantage of MeanShift is not necessary to give the cluster size, the optimal size would be found automatically.


```python
target_gp_num = len(MergeByStoreID(df_fin_store, ITEM_TRAN_df)['SEGMENT'].unique())
running_df = rtn_pre_processed
```


```python
#running_df = MergeByStoreID(SEG_NUM_df, None)
#running_df
#(KMean_Data,Pca_Chart) = RunClusteringAndRDChart(running_df, 'KMEANS', target_gp_num)
(Cluster_Label_Data,Rd_Chart) = RunClusteringAndRDChart(running_df.copy(deep=True), 'MeanShift', target_gp_num, 'PCA')
(all_record, confusionMatrixByCount, confusionMatrixByLabelPower, confusionMatrixBySegmentPower, optimal) =CreateDataAnalysis(Cluster_Label_Data, df_fin_store)
Rd_Chart
```


![png](output_46_0.png)





    <ggplot: (139377530218)>




```python
optimal
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>label</th>
      <th>SEGMENT</th>
      <th>COUNT</th>
      <th>label_SIZE</th>
      <th>SEGMENT_SIZE</th>
      <th>label_power</th>
      <th>SEGMENT_power</th>
      <th>MULTIPLE_power</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>11</td>
      <td>WTC - Baby</td>
      <td>1</td>
      <td>8</td>
      <td>4</td>
      <td>0.125000</td>
      <td>0.250000</td>
      <td>0.031250</td>
    </tr>
    <tr>
      <th>1</th>
      <td>0</td>
      <td>WTC - Commercial</td>
      <td>10</td>
      <td>214</td>
      <td>11</td>
      <td>0.046729</td>
      <td>0.909091</td>
      <td>0.042481</td>
    </tr>
    <tr>
      <th>2</th>
      <td>4</td>
      <td>WTC - Destination</td>
      <td>4</td>
      <td>8</td>
      <td>15</td>
      <td>0.500000</td>
      <td>0.266667</td>
      <td>0.133333</td>
    </tr>
    <tr>
      <th>3</th>
      <td>16</td>
      <td>WTC - Health</td>
      <td>1</td>
      <td>1</td>
      <td>14</td>
      <td>1.000000</td>
      <td>0.071429</td>
      <td>0.071429</td>
    </tr>
    <tr>
      <th>4</th>
      <td>0</td>
      <td>WTC - Residential</td>
      <td>112</td>
      <td>214</td>
      <td>119</td>
      <td>0.523364</td>
      <td>0.941176</td>
      <td>0.492578</td>
    </tr>
    <tr>
      <th>5</th>
      <td>0</td>
      <td>WTC - Savers</td>
      <td>34</td>
      <td>214</td>
      <td>38</td>
      <td>0.158879</td>
      <td>0.894737</td>
      <td>0.142154</td>
    </tr>
    <tr>
      <th>6</th>
      <td>0</td>
      <td>WTC - Tourists/Conv</td>
      <td>44</td>
      <td>214</td>
      <td>53</td>
      <td>0.205607</td>
      <td>0.830189</td>
      <td>0.170693</td>
    </tr>
    <tr>
      <th>7</th>
      <td>10</td>
      <td>Others</td>
      <td>1</td>
      <td>1</td>
      <td>5</td>
      <td>1.000000</td>
      <td>0.200000</td>
      <td>0.200000</td>
    </tr>
    <tr>
      <th>8</th>
      <td>12</td>
      <td>Others</td>
      <td>1</td>
      <td>1</td>
      <td>5</td>
      <td>1.000000</td>
      <td>0.200000</td>
      <td>0.200000</td>
    </tr>
    <tr>
      <th>9</th>
      <td>13</td>
      <td>Others</td>
      <td>1</td>
      <td>1</td>
      <td>5</td>
      <td>1.000000</td>
      <td>0.200000</td>
      <td>0.200000</td>
    </tr>
    <tr>
      <th>10</th>
      <td>14</td>
      <td>Others</td>
      <td>1</td>
      <td>1</td>
      <td>5</td>
      <td>1.000000</td>
      <td>0.200000</td>
      <td>0.200000</td>
    </tr>
    <tr>
      <th>11</th>
      <td>8</td>
      <td>Others</td>
      <td>1</td>
      <td>1</td>
      <td>5</td>
      <td>1.000000</td>
      <td>0.200000</td>
      <td>0.200000</td>
    </tr>
    <tr>
      <th>12</th>
      <td>5</td>
      <td>WTC - Beautiq</td>
      <td>1</td>
      <td>1</td>
      <td>2</td>
      <td>1.000000</td>
      <td>0.500000</td>
      <td>0.500000</td>
    </tr>
    <tr>
      <th>13</th>
      <td>7</td>
      <td>WTC - Beautiq</td>
      <td>1</td>
      <td>1</td>
      <td>2</td>
      <td>1.000000</td>
      <td>0.500000</td>
      <td>0.500000</td>
    </tr>
  </tbody>
</table>
</div>



### Clustering: MeanShift, Reduce Diemsnion: TSNE


```python
#running_df = MergeByStoreID(SEG_NUM_df, None)
#(KMean_Data,Pca_Chart) = RunClusteringAndRDChart(running_df, 'KMEANS', target_gp_num)
(Cluster_Label_Data,Rd_Chart) = RunClusteringAndRDChart(running_df.copy(deep=True), 'MeanShift', target_gp_num, 'TSNE')
(all_record, confusionMatrixByCount, confusionMatrixByLabelPower, confusionMatrixBySegmentPower, optimal) =CreateDataAnalysis(Cluster_Label_Data, df_fin_store)
Rd_Chart
```


![png](output_49_0.png)





    <ggplot: (139379009996)>




```python
optimal
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>label</th>
      <th>SEGMENT</th>
      <th>COUNT</th>
      <th>label_SIZE</th>
      <th>SEGMENT_SIZE</th>
      <th>label_power</th>
      <th>SEGMENT_power</th>
      <th>MULTIPLE_power</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>11</td>
      <td>WTC - Baby</td>
      <td>1</td>
      <td>8</td>
      <td>4</td>
      <td>0.125000</td>
      <td>0.250000</td>
      <td>0.031250</td>
    </tr>
    <tr>
      <th>1</th>
      <td>0</td>
      <td>WTC - Commercial</td>
      <td>10</td>
      <td>214</td>
      <td>11</td>
      <td>0.046729</td>
      <td>0.909091</td>
      <td>0.042481</td>
    </tr>
    <tr>
      <th>2</th>
      <td>4</td>
      <td>WTC - Destination</td>
      <td>4</td>
      <td>8</td>
      <td>15</td>
      <td>0.500000</td>
      <td>0.266667</td>
      <td>0.133333</td>
    </tr>
    <tr>
      <th>3</th>
      <td>16</td>
      <td>WTC - Health</td>
      <td>1</td>
      <td>1</td>
      <td>14</td>
      <td>1.000000</td>
      <td>0.071429</td>
      <td>0.071429</td>
    </tr>
    <tr>
      <th>4</th>
      <td>0</td>
      <td>WTC - Residential</td>
      <td>112</td>
      <td>214</td>
      <td>119</td>
      <td>0.523364</td>
      <td>0.941176</td>
      <td>0.492578</td>
    </tr>
    <tr>
      <th>5</th>
      <td>0</td>
      <td>WTC - Savers</td>
      <td>34</td>
      <td>214</td>
      <td>38</td>
      <td>0.158879</td>
      <td>0.894737</td>
      <td>0.142154</td>
    </tr>
    <tr>
      <th>6</th>
      <td>0</td>
      <td>WTC - Tourists/Conv</td>
      <td>44</td>
      <td>214</td>
      <td>53</td>
      <td>0.205607</td>
      <td>0.830189</td>
      <td>0.170693</td>
    </tr>
    <tr>
      <th>7</th>
      <td>10</td>
      <td>Others</td>
      <td>1</td>
      <td>1</td>
      <td>5</td>
      <td>1.000000</td>
      <td>0.200000</td>
      <td>0.200000</td>
    </tr>
    <tr>
      <th>8</th>
      <td>12</td>
      <td>Others</td>
      <td>1</td>
      <td>1</td>
      <td>5</td>
      <td>1.000000</td>
      <td>0.200000</td>
      <td>0.200000</td>
    </tr>
    <tr>
      <th>9</th>
      <td>13</td>
      <td>Others</td>
      <td>1</td>
      <td>1</td>
      <td>5</td>
      <td>1.000000</td>
      <td>0.200000</td>
      <td>0.200000</td>
    </tr>
    <tr>
      <th>10</th>
      <td>14</td>
      <td>Others</td>
      <td>1</td>
      <td>1</td>
      <td>5</td>
      <td>1.000000</td>
      <td>0.200000</td>
      <td>0.200000</td>
    </tr>
    <tr>
      <th>11</th>
      <td>8</td>
      <td>Others</td>
      <td>1</td>
      <td>1</td>
      <td>5</td>
      <td>1.000000</td>
      <td>0.200000</td>
      <td>0.200000</td>
    </tr>
    <tr>
      <th>12</th>
      <td>5</td>
      <td>WTC - Beautiq</td>
      <td>1</td>
      <td>1</td>
      <td>2</td>
      <td>1.000000</td>
      <td>0.500000</td>
      <td>0.500000</td>
    </tr>
    <tr>
      <th>13</th>
      <td>7</td>
      <td>WTC - Beautiq</td>
      <td>1</td>
      <td>1</td>
      <td>2</td>
      <td>1.000000</td>
      <td>0.500000</td>
      <td>0.500000</td>
    </tr>
  </tbody>
</table>
</div>



**Result**

### Clustering: KMeans, Reduce Diemsnion: PCA


```python
#print('Running it with Kmeans and PCA')
#running_df = MergeByStoreID(SEG_NUM_df, None)
#(KMean_Data,Pca_Chart) = RunClusteringAndRDChart(running_df, 'KMEANS', target_gp_num)
(Cluster_Label_Data,Rd_Chart) = RunClusteringAndRDChart(running_df.copy(deep=True), 'KMEANS', target_gp_num,'PCA')
(all_record, confusionMatrixByCount, confusionMatrixByLabelPower, confusionMatrixBySegmentPower, optimal) =CreateDataAnalysis(Cluster_Label_Data, df_fin_store)
Rd_Chart
```


![png](output_53_0.png)





    <ggplot: (139381292885)>




```python
optimal
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>label</th>
      <th>SEGMENT</th>
      <th>COUNT</th>
      <th>label_SIZE</th>
      <th>SEGMENT_SIZE</th>
      <th>label_power</th>
      <th>SEGMENT_power</th>
      <th>MULTIPLE_power</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>0</td>
      <td>WTC - Baby</td>
      <td>2</td>
      <td>32</td>
      <td>4</td>
      <td>0.062500</td>
      <td>0.500000</td>
      <td>0.031250</td>
    </tr>
    <tr>
      <th>1</th>
      <td>0</td>
      <td>WTC - Destination</td>
      <td>10</td>
      <td>32</td>
      <td>15</td>
      <td>0.312500</td>
      <td>0.666667</td>
      <td>0.208333</td>
    </tr>
    <tr>
      <th>2</th>
      <td>5</td>
      <td>WTC - Health</td>
      <td>2</td>
      <td>7</td>
      <td>14</td>
      <td>0.285714</td>
      <td>0.142857</td>
      <td>0.040816</td>
    </tr>
    <tr>
      <th>3</th>
      <td>0</td>
      <td>WTC - Tourists/Conv</td>
      <td>16</td>
      <td>32</td>
      <td>53</td>
      <td>0.500000</td>
      <td>0.301887</td>
      <td>0.150943</td>
    </tr>
    <tr>
      <th>4</th>
      <td>2</td>
      <td>Others</td>
      <td>3</td>
      <td>3</td>
      <td>5</td>
      <td>1.000000</td>
      <td>0.600000</td>
      <td>0.600000</td>
    </tr>
    <tr>
      <th>5</th>
      <td>3</td>
      <td>WTC - Commercial</td>
      <td>7</td>
      <td>65</td>
      <td>11</td>
      <td>0.107692</td>
      <td>0.636364</td>
      <td>0.068531</td>
    </tr>
    <tr>
      <th>6</th>
      <td>6</td>
      <td>WTC - Residential</td>
      <td>62</td>
      <td>92</td>
      <td>119</td>
      <td>0.673913</td>
      <td>0.521008</td>
      <td>0.351114</td>
    </tr>
    <tr>
      <th>7</th>
      <td>1</td>
      <td>WTC - Savers</td>
      <td>13</td>
      <td>44</td>
      <td>38</td>
      <td>0.295455</td>
      <td>0.342105</td>
      <td>0.101077</td>
    </tr>
    <tr>
      <th>8</th>
      <td>5</td>
      <td>WTC - Beautiq</td>
      <td>2</td>
      <td>7</td>
      <td>2</td>
      <td>0.285714</td>
      <td>1.000000</td>
      <td>0.285714</td>
    </tr>
  </tbody>
</table>
</div>



### Clustering: KMeans, Reduce Diemsnion: TSNE


```python
#print('Running it with Kmeans and TSNE')
#running_df = MergeByStoreID(SEG_NUM_df, PRODUCT_HIER_TABLE)
#(KMean_Data,Pca_Chart) = RunClusteringAndRDChart(running_df, 'KMEANS', target_gp_num)
(Cluster_Label_Data,Rd_Chart) = RunClusteringAndRDChart(running_df.copy(deep=True), 'KMEANS', target_gp_num,'TSNE')
(all_record, confusionMatrixByCount, confusionMatrixByLabelPower, confusionMatrixBySegmentPower, optimal) =CreateDataAnalysis(Cluster_Label_Data, df_fin_store)
Rd_Chart
```


![png](output_56_0.png)





    <ggplot: (-9223371897473423921)>



#### Below is the result of segmentation(label vs SEGMENT)
label_power - COUNT / label_SIZE

SEGMENT_power - COUNT / SEGMENT_SIZE

MULTIPLE_power - label_power*SEGMENT_power


**Higher the MULTIPLE_power, better we pick.**


```python
optimal
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>label</th>
      <th>SEGMENT</th>
      <th>COUNT</th>
      <th>label_SIZE</th>
      <th>SEGMENT_SIZE</th>
      <th>label_power</th>
      <th>SEGMENT_power</th>
      <th>MULTIPLE_power</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>6</td>
      <td>WTC - Commercial</td>
      <td>3</td>
      <td>45</td>
      <td>11</td>
      <td>0.066667</td>
      <td>0.272727</td>
      <td>0.018182</td>
    </tr>
    <tr>
      <th>1</th>
      <td>4</td>
      <td>WTC - Destination</td>
      <td>10</td>
      <td>29</td>
      <td>15</td>
      <td>0.344828</td>
      <td>0.666667</td>
      <td>0.229885</td>
    </tr>
    <tr>
      <th>2</th>
      <td>8</td>
      <td>WTC - Health</td>
      <td>2</td>
      <td>7</td>
      <td>14</td>
      <td>0.285714</td>
      <td>0.142857</td>
      <td>0.040816</td>
    </tr>
    <tr>
      <th>3</th>
      <td>0</td>
      <td>WTC - Residential</td>
      <td>60</td>
      <td>90</td>
      <td>119</td>
      <td>0.666667</td>
      <td>0.504202</td>
      <td>0.336134</td>
    </tr>
    <tr>
      <th>4</th>
      <td>2</td>
      <td>WTC - Savers</td>
      <td>12</td>
      <td>38</td>
      <td>38</td>
      <td>0.315789</td>
      <td>0.315789</td>
      <td>0.099723</td>
    </tr>
    <tr>
      <th>5</th>
      <td>6</td>
      <td>WTC - Tourists/Conv</td>
      <td>21</td>
      <td>45</td>
      <td>53</td>
      <td>0.466667</td>
      <td>0.396226</td>
      <td>0.184906</td>
    </tr>
    <tr>
      <th>6</th>
      <td>5</td>
      <td>Others</td>
      <td>3</td>
      <td>3</td>
      <td>5</td>
      <td>1.000000</td>
      <td>0.600000</td>
      <td>0.600000</td>
    </tr>
    <tr>
      <th>7</th>
      <td>4</td>
      <td>WTC - Baby</td>
      <td>2</td>
      <td>29</td>
      <td>4</td>
      <td>0.068966</td>
      <td>0.500000</td>
      <td>0.034483</td>
    </tr>
    <tr>
      <th>8</th>
      <td>8</td>
      <td>WTC - Beautiq</td>
      <td>2</td>
      <td>7</td>
      <td>2</td>
      <td>0.285714</td>
      <td>1.000000</td>
      <td>0.285714</td>
    </tr>
  </tbody>
</table>
</div>



## Conclusions
In this analysis, the segmentation is done by the Item Transactions, Memeber Personal Info, Member's Loyalty and products sold categories. In the initial guess, it should be segmented same/similar to the A.S. Watson segment groups. However, the groups clustered automatically cannot be matched to A.S. Watson group significantly. A.S. Watson may need to think twice whether their stores are selling what they expected / optimized, Watson should revise the segment of the store / change the products to be sold.

For better Illustration of the new groups (automatically clustered ) and old groups ( A.S. Watson Segement ), we have put all in the map.

## New Cluster in Map
Here we show the new cluster on a map. As the Store Table is lack of latitude and longitude, we add it back latitude and longitude through GoogleMap API. Then we mark all the Store on OpenStreetMap with different colors representing different clusters.

For plotting on Google Map, this package could be used: 
https://github.com/tcassou/mapsplotlib 
or
https://github.com/pbugnion/gmaps

googlemaps git: https://github.com/googlemaps/google-maps-services-python


```python
def findGeoCode(name, measure='lat'):
    #gmaps = googlemaps.Client(key='AIzaSyDWxX_F8b8k4QHn2i8fYLB-PpZXHoNP1HM')
    #gmaps = googlemaps.Client(key='AIzaSyBlTYjJiWCwEJBVWeeXt560w86q82MdGuA')    
    gmaps = googlemaps.Client(key='AIzaSyARkKxFGSgvqEmx0NODIhPOQTMAj5kc7qk')    
    rtn = gmaps.geocode(name)
    if len(rtn)> 0:
        if measure == 'lat':
            return rtn[0]['geometry']['location']['lat']
        else:
            return rtn[0]['geometry']['location']['lng']
    else:
        return None

def AppendColumns(df, NEW_COL_NAME, APPENDING_COL_NAME_LIST):    
    df[NEW_COL_NAME] = ""
    
    for x in APPENDING_COL_NAME_LIST:        
        df[NEW_COL_NAME] =df[NEW_COL_NAME] + ' ' + df[x]
        #if df[NEW_COL_NAME].empty:
        #    df[NEW_COL_NAME] = df[x]
        #else:
            
    return df
def CheckLatLngBoundary(df, lat_north, lat_south, lng_west, lng_east):
    return df[(df['lat']>=lat_north) | (df['lat']<=lat_south) | (df['lng']<=lng_west)  | (df['lng']>=lng_east) ]

def AddLatLng(LOC_DT, LOC_NAME, deep=False):
    STORE_LOCATION_GP_DT = LOC_DT.copy(deep=deep)
    lat = STORE_LOCATION_GP_DT[LOC_NAME].apply(findGeoCode, args = ('lat',))
    #lng = df_fin_store['LOC_NAME'].apply(lambda x: gmaps.geocode(x)[0]['geometry']['location']['lng'])
    lng = STORE_LOCATION_GP_DT[LOC_NAME].apply(findGeoCode,args = ('lng',))

    STORE_LOCATION_GP_DT['lat'] = lat
    STORE_LOCATION_GP_DT['lng'] = lng
    return STORE_LOCATION_GP_DT


    
def PasteData(Label_Data, Store_Location):
    Store_Location_Col = list(Store_Location.columns)    
    Remove_Col_List = list(set(Label_Data.columns).intersection(set(Store_Location.columns))).remove('STORE_ID')
    if ( Remove_Col_List is not None ):
        for i, val in enumerate(list(set(Label_Data.columns).intersection(set(Store_Location.columns))).remove('STORE_ID')):
            Store_Location_Col.remove(val)
    return pd.merge(left=Label_Data, right = Store_Location[Store_Location_Col], left_on = 'STORE_ID', right_on = 'STORE_ID', how = 'inner' )

def TurnLabelIntoColor(label):
    color_list = ['red', 'blue', 'green', 'purple', 'orange', 'darkred',
             'lightred', 'beige', 'darkblue', 'darkgreen', 'cadetblue',
             'darkpurple', 'white', 'pink', 'lightblue', 'lightgreen',
             'gray', 'black', 'lightgray']
    return color_list[label]

def printMap(data , mapInitialPoint):
    map_1 = folium.Map(location=mapInitialPoint)
    #df_fin_store.applymap()
    for index, row in data.iterrows():
        if not math.isnan(row['lat']):
            folium.Marker([row['lat'],row['lng']],
                  popup='STORE_ID:'+ row['STORE_ID'] + ', LABEL:' + row['label'] + ', SEGMENT:' + row['SEGMENT'],
                  #icon=folium.Icon(color=TurnLabelIntoColor(int(row['label'])))
                  icon=folium.Icon(color=TurnLabelIntoColor(int(row['label'])))                      
                  #icon=folium.Icon(icon='cloud')
                 ).add_to(map_1)
    return map_1
```


```python
#Get the Lat and Lng the first
Cluster_Label_Store_Data = PasteData(Cluster_Label_Data,df_fin_store.copy(deep=True) )
Cluster_Label_Store_Data_With_Loc = AddLatLng(Cluster_Label_Store_Data,'LOC_NAME', True)

```


```python
#Get the Lat and Lng the first for not in range lat and lng

#Check if the points are closed to Hong Kong and Macau
NotInRangeLocation = CheckLatLngBoundary(Cluster_Label_Store_Data_With_Loc, 22.6,22,113.3,114.4)

#Create a new columns for checking at the map again
NotInRangeLocation_MapCheck = AppendColumns(NotInRangeLocation, 'MapCheck', ['LOC_NAME', 'CITY'])
NotInRangeLocation_MapChecked = AddLatLng(NotInRangeLocation_MapCheck, 'MapCheck')
del NotInRangeLocation_MapChecked['MapCheck']

```

    C:\ProgramData\Anaconda3\lib\site-packages\ipykernel_launcher.py:15: SettingWithCopyWarning: 
    A value is trying to be set on a copy of a slice from a DataFrame.
    Try using .loc[row_indexer,col_indexer] = value instead
    
    See the caveats in the documentation: http://pandas.pydata.org/pandas-docs/stable/indexing.html#indexing-view-versus-copy
      from ipykernel import kernelapp as app
    C:\ProgramData\Anaconda3\lib\site-packages\ipykernel_launcher.py:18: SettingWithCopyWarning: 
    A value is trying to be set on a copy of a slice from a DataFrame.
    Try using .loc[row_indexer,col_indexer] = value instead
    
    See the caveats in the documentation: http://pandas.pydata.org/pandas-docs/stable/indexing.html#indexing-view-versus-copy
    


```python
#drop the existing records not in range
Cluster_Label_Store_Data_With_Loc = Cluster_Label_Store_Data_With_Loc.drop(NotInRangeLocation_MapChecked.index)

#append the corrected records
Cluster_Label_Store_Data_With_Loc = pd.concat([Cluster_Label_Store_Data_With_Loc , NotInRangeLocation_MapChecked])

map_label = printMap(Cluster_Label_Store_Data_With_Loc, [22.28552, 114.15769])
map_label
```




<div style="width:100%;"><div style="position:relative;width:100%;height:0;padding-bottom:60%;"><iframe src="data:text/html;charset=utf-8;base64,PCFET0NUWVBFIGh0bWw+CjxoZWFkPiAgICAKICAgIDxtZXRhIGh0dHAtZXF1aXY9ImNvbnRlbnQtdHlwZSIgY29udGVudD0idGV4dC9odG1sOyBjaGFyc2V0PVVURi04IiAvPgogICAgPHNjcmlwdD5MX1BSRUZFUl9DQU5WQVMgPSBmYWxzZTsgTF9OT19UT1VDSCA9IGZhbHNlOyBMX0RJU0FCTEVfM0QgPSBmYWxzZTs8L3NjcmlwdD4KICAgIDxzY3JpcHQgc3JjPSJodHRwczovL2Nkbi5qc2RlbGl2ci5uZXQvbnBtL2xlYWZsZXRAMS4yLjAvZGlzdC9sZWFmbGV0LmpzIj48L3NjcmlwdD4KICAgIDxzY3JpcHQgc3JjPSJodHRwczovL2FqYXguZ29vZ2xlYXBpcy5jb20vYWpheC9saWJzL2pxdWVyeS8xLjExLjEvanF1ZXJ5Lm1pbi5qcyI+PC9zY3JpcHQ+CiAgICA8c2NyaXB0IHNyYz0iaHR0cHM6Ly9tYXhjZG4uYm9vdHN0cmFwY2RuLmNvbS9ib290c3RyYXAvMy4yLjAvanMvYm9vdHN0cmFwLm1pbi5qcyI+PC9zY3JpcHQ+CiAgICA8c2NyaXB0IHNyYz0iaHR0cHM6Ly9jZG5qcy5jbG91ZGZsYXJlLmNvbS9hamF4L2xpYnMvTGVhZmxldC5hd2Vzb21lLW1hcmtlcnMvMi4wLjIvbGVhZmxldC5hd2Vzb21lLW1hcmtlcnMuanMiPjwvc2NyaXB0PgogICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL2Nkbi5qc2RlbGl2ci5uZXQvbnBtL2xlYWZsZXRAMS4yLjAvZGlzdC9sZWFmbGV0LmNzcyIgLz4KICAgIDxsaW5rIHJlbD0ic3R5bGVzaGVldCIgaHJlZj0iaHR0cHM6Ly9tYXhjZG4uYm9vdHN0cmFwY2RuLmNvbS9ib290c3RyYXAvMy4yLjAvY3NzL2Jvb3RzdHJhcC5taW4uY3NzIiAvPgogICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL21heGNkbi5ib290c3RyYXBjZG4uY29tL2Jvb3RzdHJhcC8zLjIuMC9jc3MvYm9vdHN0cmFwLXRoZW1lLm1pbi5jc3MiIC8+CiAgICA8bGluayByZWw9InN0eWxlc2hlZXQiIGhyZWY9Imh0dHBzOi8vbWF4Y2RuLmJvb3RzdHJhcGNkbi5jb20vZm9udC1hd2Vzb21lLzQuNi4zL2Nzcy9mb250LWF3ZXNvbWUubWluLmNzcyIgLz4KICAgIDxsaW5rIHJlbD0ic3R5bGVzaGVldCIgaHJlZj0iaHR0cHM6Ly9jZG5qcy5jbG91ZGZsYXJlLmNvbS9hamF4L2xpYnMvTGVhZmxldC5hd2Vzb21lLW1hcmtlcnMvMi4wLjIvbGVhZmxldC5hd2Vzb21lLW1hcmtlcnMuY3NzIiAvPgogICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL3Jhd2dpdC5jb20vcHl0aG9uLXZpc3VhbGl6YXRpb24vZm9saXVtL21hc3Rlci9mb2xpdW0vdGVtcGxhdGVzL2xlYWZsZXQuYXdlc29tZS5yb3RhdGUuY3NzIiAvPgogICAgPHN0eWxlPmh0bWwsIGJvZHkge3dpZHRoOiAxMDAlO2hlaWdodDogMTAwJTttYXJnaW46IDA7cGFkZGluZzogMDt9PC9zdHlsZT4KICAgIDxzdHlsZT4jbWFwIHtwb3NpdGlvbjphYnNvbHV0ZTt0b3A6MDtib3R0b206MDtyaWdodDowO2xlZnQ6MDt9PC9zdHlsZT4KICAgIAogICAgICAgICAgICA8c3R5bGU+ICNtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzAgewogICAgICAgICAgICAgICAgcG9zaXRpb24gOiByZWxhdGl2ZTsKICAgICAgICAgICAgICAgIHdpZHRoIDogMTAwLjAlOwogICAgICAgICAgICAgICAgaGVpZ2h0OiAxMDAuMCU7CiAgICAgICAgICAgICAgICBsZWZ0OiAwLjAlOwogICAgICAgICAgICAgICAgdG9wOiAwLjAlOwogICAgICAgICAgICAgICAgfQogICAgICAgICAgICA8L3N0eWxlPgogICAgICAgIAo8L2hlYWQ+Cjxib2R5PiAgICAKICAgIAogICAgICAgICAgICA8ZGl2IGNsYXNzPSJmb2xpdW0tbWFwIiBpZD0ibWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwIiA+PC9kaXY+CiAgICAgICAgCjwvYm9keT4KPHNjcmlwdD4gICAgCiAgICAKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGJvdW5kcyA9IG51bGw7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgdmFyIG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCA9IEwubWFwKAogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgJ21hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCcsCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICB7Y2VudGVyOiBbMjIuMjg1NTIsMTE0LjE1NzY5XSwKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIHpvb206IDEwLAogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgbWF4Qm91bmRzOiBib3VuZHMsCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICBsYXllcnM6IFtdLAogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgd29ybGRDb3B5SnVtcDogZmFsc2UsCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICBjcnM6IEwuQ1JTLkVQU0czODU3CiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgdGlsZV9sYXllcl83MzMyMzg1YWIyZmU0NjUxYjY1MWVlNjMwYmYwNDczMSA9IEwudGlsZUxheWVyKAogICAgICAgICAgICAgICAgJ2h0dHBzOi8ve3N9LnRpbGUub3BlbnN0cmVldG1hcC5vcmcve3p9L3t4fS97eX0ucG5nJywKICAgICAgICAgICAgICAgIHsKICAiYXR0cmlidXRpb24iOiBudWxsLAogICJkZXRlY3RSZXRpbmEiOiBmYWxzZSwKICAibWF4Wm9vbSI6IDE4LAogICJtaW5ab29tIjogMSwKICAibm9XcmFwIjogZmFsc2UsCiAgInN1YmRvbWFpbnMiOiAiYWJjIgp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfOWMxY2I2YTkzMjEwNDRhNjk1Y2Y5MmI0NmZlNWIxZDAgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4xOTQzMDU4LDExMy41NDM4NTg3XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzRmODM1ZjNiNDQ5MDRjYTA4MDI3ZjliYzFjNTc0NTIwID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdsaWdodHJlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl85YzFjYjZhOTMyMTA0NGE2OTVjZjkyYjQ2ZmU1YjFkMC5zZXRJY29uKGljb25fNGY4MzVmM2I0NDkwNGNhMDgwMjdmOWJjMWM1NzQ1MjApOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfZmI2MTgzZTg3ZjI0NDgxYTlmNGViNzhjMTYwYjZkZmUgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfMzYxNGQwNDkyNzczNDJlZDg4OWVhNjZkYzU3Yzk4Y2UgPSAkKCc8ZGl2IGlkPSJodG1sXzM2MTRkMDQ5Mjc3MzQyZWQ4ODllYTY2ZGM1N2M5OGNlIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjE3MiwgTEFCRUw6NiwgU0VHTUVOVDpXVEMgLSBUb3VyaXN0cy9Db252PC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9mYjYxODNlODdmMjQ0ODFhOWY0ZWI3OGMxNjBiNmRmZS5zZXRDb250ZW50KGh0bWxfMzYxNGQwNDkyNzczNDJlZDg4OWVhNjZkYzU3Yzk4Y2UpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl85YzFjYjZhOTMyMTA0NGE2OTVjZjkyYjQ2ZmU1YjFkMC5iaW5kUG9wdXAocG9wdXBfZmI2MTgzZTg3ZjI0NDgxYTlmNGViNzhjMTYwYjZkZmUpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfMWM5MjY2ZGU1YmI4NGJlMGE2NDk4YTM3YmRmNzYxZGYgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4zNjI4MTE1LDExNC4xMDMzOTQ1XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2FlNmVmYzZjMWZmMjRmN2I4ZDQ4YWYwMzdkZWRjOGJkID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfMWM5MjY2ZGU1YmI4NGJlMGE2NDk4YTM3YmRmNzYxZGYuc2V0SWNvbihpY29uX2FlNmVmYzZjMWZmMjRmN2I4ZDQ4YWYwMzdkZWRjOGJkKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzA0OGMzNGZmNTMwNjRmYTJhNmE4NzU2YzdkNDEyOGM5ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzlhODgwMzRiZWM2ZjRjYTBiNmE4YjFkOGYwZGQ2ODU4ID0gJCgnPGRpdiBpZD0iaHRtbF85YTg4MDM0YmVjNmY0Y2EwYjZhOGIxZDhmMGRkNjg1OCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIwNzMsIExBQkVMOjAsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzA0OGMzNGZmNTMwNjRmYTJhNmE4NzU2YzdkNDEyOGM5LnNldENvbnRlbnQoaHRtbF85YTg4MDM0YmVjNmY0Y2EwYjZhOGIxZDhmMGRkNjg1OCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzFjOTI2NmRlNWJiODRiZTBhNjQ5OGEzN2JkZjc2MWRmLmJpbmRQb3B1cChwb3B1cF8wNDhjMzRmZjUzMDY0ZmEyYTZhODc1NmM3ZDQxMjhjOSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl85Y2M0OTI4NWRmZTM0MzA3OWI3ZWRkMWMxYTQ5OGI1OCA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjMyMTA2NTIsMTE0LjIxMzI3NjhdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fZmEzYjhkZTg5ZjllNDNkZWI3NjBkYTBlMTUxMmNhYmYgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2dyZWVuJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzljYzQ5Mjg1ZGZlMzQzMDc5YjdlZGQxYzFhNDk4YjU4LnNldEljb24oaWNvbl9mYTNiOGRlODlmOWU0M2RlYjc2MGRhMGUxNTEyY2FiZik7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9mODY4YThhMGMwN2M0YmM4YjhlMzkwNTQzNjgwYTU5MCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9kMTliODAyOTVkNWU0MmY5YmI1MjljMDk4ZWNkNWQ2MiA9ICQoJzxkaXYgaWQ9Imh0bWxfZDE5YjgwMjk1ZDVlNDJmOWJiNTI5YzA5OGVjZDVkNjIiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMjcxLCBMQUJFTDoyLCBTRUdNRU5UOldUQyAtIEhlYWx0aDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfZjg2OGE4YTBjMDdjNGJjOGI4ZTM5MDU0MzY4MGE1OTAuc2V0Q29udGVudChodG1sX2QxOWI4MDI5NWQ1ZTQyZjliYjUyOWMwOThlY2Q1ZDYyKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfOWNjNDkyODVkZmUzNDMwNzliN2VkZDFjMWE0OThiNTguYmluZFBvcHVwKHBvcHVwX2Y4NjhhOGEwYzA3YzRiYzhiOGUzOTA1NDM2ODBhNTkwKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzI5OTAyNTU4MjkyNTQyYTBhMjQzNTg5ZThlNDM0MzY1ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzg3Mzg5LDExMy45ODAyNzZdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fOTQ4MmYyZDZlMTliNDQ1OTllZDJkNDdkYWI4ODFjOGYgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3JlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl8yOTkwMjU1ODI5MjU0MmEwYTI0MzU4OWU4ZTQzNDM2NS5zZXRJY29uKGljb25fOTQ4MmYyZDZlMTliNDQ1OTllZDJkNDdkYWI4ODFjOGYpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfY2EyMjg0ZTMyNWVjNDgxZjhkZDBhYjE5MzU2NjJlNzggPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfZDdkYWY1YmQ0ZGM1NGNmY2EwNWUyYTRmMTA1NWIzZTggPSAkKCc8ZGl2IGlkPSJodG1sX2Q3ZGFmNWJkNGRjNTRjZmNhMDVlMmE0ZjEwNTViM2U4IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMTkwMiwgTEFCRUw6MCwgU0VHTUVOVDpXVEMgLSBTYXZlcnM8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2NhMjI4NGUzMjVlYzQ4MWY4ZGQwYWIxOTM1NjYyZTc4LnNldENvbnRlbnQoaHRtbF9kN2RhZjViZDRkYzU0Y2ZjYTA1ZTJhNGYxMDU1YjNlOCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzI5OTAyNTU4MjkyNTQyYTBhMjQzNTg5ZThlNDM0MzY1LmJpbmRQb3B1cChwb3B1cF9jYTIyODRlMzI1ZWM0ODFmOGRkMGFiMTkzNTY2MmU3OCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl80OWJlNjgxYmZlMDg0M2E1YTk1Njc4ZjI0M2U1MzdhNCA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjMzNzI5NzEsMTE0LjE3NDUyNzNdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fYTg0ZTBjY2YzNWE1NGJlZDgzZDE5MTNiYWE4MDM5NTcgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ29yYW5nZScsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl80OWJlNjgxYmZlMDg0M2E1YTk1Njc4ZjI0M2U1MzdhNC5zZXRJY29uKGljb25fYTg0ZTBjY2YzNWE1NGJlZDgzZDE5MTNiYWE4MDM5NTcpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfZGE3NzRiMzdmYzkwNDQ3NDhmYWM2OTdlYmIwYmNjYzYgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfOThiNThiYjIwNTg0NGYxYjhlZDYxODgyMDY3ODViNzMgPSAkKCc8ZGl2IGlkPSJodG1sXzk4YjU4YmIyMDU4NDRmMWI4ZWQ2MTg4MjA2Nzg1YjczIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjE2MCwgTEFCRUw6NCwgU0VHTUVOVDpXVEMgLSBEZXN0aW5hdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfZGE3NzRiMzdmYzkwNDQ3NDhmYWM2OTdlYmIwYmNjYzYuc2V0Q29udGVudChodG1sXzk4YjU4YmIyMDU4NDRmMWI4ZWQ2MTg4MjA2Nzg1YjczKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfNDliZTY4MWJmZTA4NDNhNWE5NTY3OGYyNDNlNTM3YTQuYmluZFBvcHVwKHBvcHVwX2RhNzc0YjM3ZmM5MDQ0NzQ4ZmFjNjk3ZWJiMGJjY2M2KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzdiZDVhYjgwZDQzYjQxZTFhZjg4OWYzZjE3OTI5NzEzID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuNTAxMjU3LDExNC4xMjc5NDVdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fZjlhYTk0MDhjYTlmNDM2OGE3Yjg3MTljZGJiNmFjYjUgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3JlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl83YmQ1YWI4MGQ0M2I0MWUxYWY4ODlmM2YxNzkyOTcxMy5zZXRJY29uKGljb25fZjlhYTk0MDhjYTlmNDM2OGE3Yjg3MTljZGJiNmFjYjUpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfZWE2MWE1ZmM0MzE1NGU3MDkzOGY0NDM5ZTY2NzdhZjUgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNWM3MWRjOTAyNmFiNDg5Mzk5NGM4ZTlhY2Y3ZGU2MTkgPSAkKCc8ZGl2IGlkPSJodG1sXzVjNzFkYzkwMjZhYjQ4OTM5OTRjOGU5YWNmN2RlNjE5IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMTg5NiwgTEFCRUw6MCwgU0VHTUVOVDpXVEMgLSBUb3VyaXN0cy9Db252PC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9lYTYxYTVmYzQzMTU0ZTcwOTM4ZjQ0MzllNjY3N2FmNS5zZXRDb250ZW50KGh0bWxfNWM3MWRjOTAyNmFiNDg5Mzk5NGM4ZTlhY2Y3ZGU2MTkpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl83YmQ1YWI4MGQ0M2I0MWUxYWY4ODlmM2YxNzkyOTcxMy5iaW5kUG9wdXAocG9wdXBfZWE2MWE1ZmM0MzE1NGU3MDkzOGY0NDM5ZTY2NzdhZjUpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfN2FhYWI5MjIxNWE2NGQ2OGIyZjE0NGM3OWM2MWUyOTQgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4zMjE5MiwxMTQuMTcyNDc4XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzU2MzUyNjIwYWE2MDQ5ZWY4Y2E2ODJmMzQ4NGY2ODRhID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdncmVlbicsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl83YWFhYjkyMjE1YTY0ZDY4YjJmMTQ0Yzc5YzYxZTI5NC5zZXRJY29uKGljb25fNTYzNTI2MjBhYTYwNDllZjhjYTY4MmYzNDg0ZjY4NGEpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfOTdmYjFiODk0NzRjNDdjZjlkM2FlOGJhMmFmYWViZTEgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfMTRmNWY3ZDgzNWM5NDVjOGE5Mzc5ZmY5M2E2NDE4Y2YgPSAkKCc8ZGl2IGlkPSJodG1sXzE0ZjVmN2Q4MzVjOTQ1YzhhOTM3OWZmOTNhNjQxOGNmIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjI0MiwgTEFCRUw6MiwgU0VHTUVOVDpXVEMgLSBUb3VyaXN0cy9Db252PC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF85N2ZiMWI4OTQ3NGM0N2NmOWQzYWU4YmEyYWZhZWJlMS5zZXRDb250ZW50KGh0bWxfMTRmNWY3ZDgzNWM5NDVjOGE5Mzc5ZmY5M2E2NDE4Y2YpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl83YWFhYjkyMjE1YTY0ZDY4YjJmMTQ0Yzc5YzYxZTI5NC5iaW5kUG9wdXAocG9wdXBfOTdmYjFiODk0NzRjNDdjZjlkM2FlOGJhMmFmYWViZTEpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfNjYxMmNjMGRlNDlkNGM0Nzg3MjE0YjJhMWNhYjNjYzIgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4zMzc1ODQ4LDExNC4xMzc5ODM2XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2MxYTYxYTM3ZWI4YjRiYTQ4ZTk5ZWM1ZmI1YTIwNmVmID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfNjYxMmNjMGRlNDlkNGM0Nzg3MjE0YjJhMWNhYjNjYzIuc2V0SWNvbihpY29uX2MxYTYxYTM3ZWI4YjRiYTQ4ZTk5ZWM1ZmI1YTIwNmVmKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzhiNzk5ZTAwYjE1MTQ1ODFhOWVmYTEwZmU3ZjMyNjkyID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2MwODlkMWRkOGQ3ZTQ4YWNhMzI4MjY2ZTk3NzAxNzY4ID0gJCgnPGRpdiBpZD0iaHRtbF9jMDg5ZDFkZDhkN2U0OGFjYTMyODI2NmU5NzcwMTc2OCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTI0NjAsIExBQkVMOjAsIFNFR01FTlQ6V1RDIC0gVG91cmlzdHMvQ29udjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfOGI3OTllMDBiMTUxNDU4MWE5ZWZhMTBmZTdmMzI2OTIuc2V0Q29udGVudChodG1sX2MwODlkMWRkOGQ3ZTQ4YWNhMzI4MjY2ZTk3NzAxNzY4KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfNjYxMmNjMGRlNDlkNGM0Nzg3MjE0YjJhMWNhYjNjYzIuYmluZFBvcHVwKHBvcHVwXzhiNzk5ZTAwYjE1MTQ1ODFhOWVmYTEwZmU3ZjMyNjkyKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzlhZmIyZjIyYzU1YTQwZTliZDMwM2YyMGU4MmYxNzI3ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuNDQ3ODE1LDExNC4xNjcxNjcxXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzBhNjdhODFlN2ZjMTQ1YzM5NDc1NjEyNzFiZmU3ZTEwID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfOWFmYjJmMjJjNTVhNDBlOWJkMzAzZjIwZTgyZjE3Mjcuc2V0SWNvbihpY29uXzBhNjdhODFlN2ZjMTQ1YzM5NDc1NjEyNzFiZmU3ZTEwKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzY2M2Y1OWU3NjE4YzRiODY5ZDcyYjg1ZTQ5MWQ0NGJkID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2ZhZjMyZmZmZmVmZTQ1NDFhODgwNTZhODYzMzYzNjc3ID0gJCgnPGRpdiBpZD0iaHRtbF9mYWYzMmZmZmZlZmU0NTQxYTg4MDU2YTg2MzM2MzY3NyIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTE3MDAsIExBQkVMOjAsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzY2M2Y1OWU3NjE4YzRiODY5ZDcyYjg1ZTQ5MWQ0NGJkLnNldENvbnRlbnQoaHRtbF9mYWYzMmZmZmZlZmU0NTQxYTg4MDU2YTg2MzM2MzY3Nyk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzlhZmIyZjIyYzU1YTQwZTliZDMwM2YyMGU4MmYxNzI3LmJpbmRQb3B1cChwb3B1cF82NjNmNTllNzYxOGM0Yjg2OWQ3MmI4NWU0OTFkNDRiZCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl83NTI5NWU2N2E5Yjk0NDQzOTMwMzg3N2Q5M2E1ZDFhZiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjM1OTYzMjIsMTE0LjEyNzQ2ODldLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fMDI1NGRkY2FjNDBmNDk0NzlkMTcwMDk4NzllNjBjNDIgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3JlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl83NTI5NWU2N2E5Yjk0NDQzOTMwMzg3N2Q5M2E1ZDFhZi5zZXRJY29uKGljb25fMDI1NGRkY2FjNDBmNDk0NzlkMTcwMDk4NzllNjBjNDIpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfODFiNzY3Y2IyZDBhNDAwN2FmNDU1MmUzMzE5N2YyZTUgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfY2MxNjA5NmM5N2Q0NGJiNjhjNjc1NWU5MzY5MTFlYzcgPSAkKCc8ZGl2IGlkPSJodG1sX2NjMTYwOTZjOTdkNDRiYjY4YzY3NTVlOTM2OTExZWM3IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMTk3NywgTEFCRUw6MCwgU0VHTUVOVDpXVEMgLSBTYXZlcnM8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzgxYjc2N2NiMmQwYTQwMDdhZjQ1NTJlMzMxOTdmMmU1LnNldENvbnRlbnQoaHRtbF9jYzE2MDk2Yzk3ZDQ0YmI2OGM2NzU1ZTkzNjkxMWVjNyk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzc1Mjk1ZTY3YTliOTQ0NDM5MzAzODc3ZDkzYTVkMWFmLmJpbmRQb3B1cChwb3B1cF84MWI3NjdjYjJkMGE0MDA3YWY0NTUyZTMzMTk3ZjJlNSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl8xMTIwYzY4Mjc4NjU0NjA4OGVjMmYxNDkxOWFlYTM1NiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjUwMTI0MjYsMTE0LjEyNjQ4NjRdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fNTk4NzIxOTdjNmJlNGNmOTlkNDAzNWY4YTVkZTk3YmQgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ29yYW5nZScsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl8xMTIwYzY4Mjc4NjU0NjA4OGVjMmYxNDkxOWFlYTM1Ni5zZXRJY29uKGljb25fNTk4NzIxOTdjNmJlNGNmOTlkNDAzNWY4YTVkZTk3YmQpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfODEwMDRiYzA2MDg1NGM1ZGE4NzlhZWM5OTNmOWJlMjkgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfOGE1NDVkYTU0NjJkNGQ2M2FkMzlhMWVlMDA1NWY4NTggPSAkKCc8ZGl2IGlkPSJodG1sXzhhNTQ1ZGE1NDYyZDRkNjNhZDM5YTFlZTAwNTVmODU4IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjYyMCwgTEFCRUw6NCwgU0VHTUVOVDpXVEMgLSBUb3VyaXN0cy9Db252PC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF84MTAwNGJjMDYwODU0YzVkYTg3OWFlYzk5M2Y5YmUyOS5zZXRDb250ZW50KGh0bWxfOGE1NDVkYTU0NjJkNGQ2M2FkMzlhMWVlMDA1NWY4NTgpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl8xMTIwYzY4Mjc4NjU0NjA4OGVjMmYxNDkxOWFlYTM1Ni5iaW5kUG9wdXAocG9wdXBfODEwMDRiYzA2MDg1NGM1ZGE4NzlhZWM5OTNmOWJlMjkpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfNjRlMjYyZjgzZGNiNDIyMTgyM2MzMDk5NmU0YjBjMTAgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi40NTI0ODc2LDExNC4xNjY3MDEyXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2EyZmUzOGZmMjY1ZDQzYzg4ZmRhZGE1MTViZjhhYjBkID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfNjRlMjYyZjgzZGNiNDIyMTgyM2MzMDk5NmU0YjBjMTAuc2V0SWNvbihpY29uX2EyZmUzOGZmMjY1ZDQzYzg4ZmRhZGE1MTViZjhhYjBkKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzRjMjRjZDE2MjA3ZTRjMWE4NzEwYzQxM2YyMjJjZmY1ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzRjM2MwMmExYzJkMzRmYjE4NThmYTg0YzA4YWQwODM0ID0gJCgnPGRpdiBpZD0iaHRtbF80YzNjMDJhMWMyZDM0ZmIxODU4ZmE4NGMwOGFkMDgzNCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTE5NTUsIExBQkVMOjAsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzRjMjRjZDE2MjA3ZTRjMWE4NzEwYzQxM2YyMjJjZmY1LnNldENvbnRlbnQoaHRtbF80YzNjMDJhMWMyZDM0ZmIxODU4ZmE4NGMwOGFkMDgzNCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzY0ZTI2MmY4M2RjYjQyMjE4MjNjMzA5OTZlNGIwYzEwLmJpbmRQb3B1cChwb3B1cF80YzI0Y2QxNjIwN2U0YzFhODcxMGM0MTNmMjIyY2ZmNSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl8zMDExZTc2OWIyNGM0MGYwYWEzYjBlNmY1MWNhYWNiMCA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjI3OTAyNTcsMTE0LjE2NTI4OF0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl8xMWY1MzU4Y2VjMmM0NjQyYWEwNDk1MTExMWFiM2RjNSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnYmVpZ2UnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfMzAxMWU3NjliMjRjNDBmMGFhM2IwZTZmNTFjYWFjYjAuc2V0SWNvbihpY29uXzExZjUzNThjZWMyYzQ2NDJhYTA0OTUxMTExYWIzZGM1KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2Q0Yjg5NDE4OTUwMjRkOTZhZGQ1NTk0ZDg1MjI3MGFlID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzUzNDViOTEyNmFmYTQzNzA5MGE2MmMxODgxODFhNzI3ID0gJCgnPGRpdiBpZD0iaHRtbF81MzQ1YjkxMjZhZmE0MzcwOTBhNjJjMTg4MTgxYTcyNyIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTE5NzIsIExBQkVMOjcsIFNFR01FTlQ6V1RDIC0gQ29tbWVyY2lhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfZDRiODk0MTg5NTAyNGQ5NmFkZDU1OTRkODUyMjcwYWUuc2V0Q29udGVudChodG1sXzUzNDViOTEyNmFmYTQzNzA5MGE2MmMxODgxODFhNzI3KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfMzAxMWU3NjliMjRjNDBmMGFhM2IwZTZmNTFjYWFjYjAuYmluZFBvcHVwKHBvcHVwX2Q0Yjg5NDE4OTUwMjRkOTZhZGQ1NTk0ZDg1MjI3MGFlKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzE5Mjk5YzQ0MWRmOTQyZTQ5ZjFmZjQxM2E5MzBmNjBkID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzEwNTEzLDExNC4yMzgwNDIyXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzhkNzljNTg1NWRjZjQ3MmFhMjA0NDNlNmRmZDg0MjI2ID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdncmVlbicsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl8xOTI5OWM0NDFkZjk0MmU0OWYxZmY0MTNhOTMwZjYwZC5zZXRJY29uKGljb25fOGQ3OWM1ODU1ZGNmNDcyYWEyMDQ0M2U2ZGZkODQyMjYpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfYmQzZTE4YjhlZGIzNGI0NDkzNjIxOTEzMmY5YzAyNTIgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfZWYxMDAwYzc0YzJhNDg3MzhmZmIzZjU4NDMxYWU5ZTEgPSAkKCc8ZGl2IGlkPSJodG1sX2VmMTAwMGM3NGMyYTQ4NzM4ZmZiM2Y1ODQzMWFlOWUxIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjAzNywgTEFCRUw6MiwgU0VHTUVOVDpXVEMgLSBTYXZlcnM8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2JkM2UxOGI4ZWRiMzRiNDQ5MzYyMTkxMzJmOWMwMjUyLnNldENvbnRlbnQoaHRtbF9lZjEwMDBjNzRjMmE0ODczOGZmYjNmNTg0MzFhZTllMSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzE5Mjk5YzQ0MWRmOTQyZTQ5ZjFmZjQxM2E5MzBmNjBkLmJpbmRQb3B1cChwb3B1cF9iZDNlMThiOGVkYjM0YjQ0OTM2MjE5MTMyZjljMDI1Mik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9hYWRiMWM3M2Q0Mjc0NjRkYmE1NWU4MGJkMWMzZGU0ZiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjM2NDQ3NzIsMTE0LjEyNjkxMDNdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fNzA4NjcwMDIzMzg5NGQ2ZGIwN2RkMGU5NWZiMzE4ZmEgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2JlaWdlJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyX2FhZGIxYzczZDQyNzQ2NGRiYTU1ZTgwYmQxYzNkZTRmLnNldEljb24oaWNvbl83MDg2NzAwMjMzODk0ZDZkYjA3ZGQwZTk1ZmIzMThmYSk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF81OTJjNzhlNzRhODE0MWUwYjQ4MzhjNmJlYjU4ZmQ1ZSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9lODJhYTc0ZWFhNjc0NjNkYjc4YzAwOWM4YmY1MzEyMyA9ICQoJzxkaXYgaWQ9Imh0bWxfZTgyYWE3NGVhYTY3NDYzZGI3OGMwMDljOGJmNTMxMjMiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyNjI0LCBMQUJFTDo3LCBTRUdNRU5UOldUQyAtIFNhdmVyczwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNTkyYzc4ZTc0YTgxNDFlMGI0ODM4YzZiZWI1OGZkNWUuc2V0Q29udGVudChodG1sX2U4MmFhNzRlYWE2NzQ2M2RiNzhjMDA5YzhiZjUzMTIzKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfYWFkYjFjNzNkNDI3NDY0ZGJhNTVlODBiZDFjM2RlNGYuYmluZFBvcHVwKHBvcHVwXzU5MmM3OGU3NGE4MTQxZTBiNDgzOGM2YmViNThmZDVlKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzcxMzE2MzA5ZTg5ZjRjYzlhNjNiNThmOTU1MGI2YjgxID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMjQ3OTAxNCwxMTQuMTY2NDE5NV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9jNmYzMzgwZTZkZGE0YmJlODBlMjA2OWVmODJhN2Q3OSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnYmVpZ2UnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfNzEzMTYzMDllODlmNGNjOWE2M2I1OGY5NTUwYjZiODEuc2V0SWNvbihpY29uX2M2ZjMzODBlNmRkYTRiYmU4MGUyMDY5ZWY4MmE3ZDc5KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzQxZDc4MDc0N2Y1OTQyMWFiYjgyYzNmYjA1NjA3Mzg2ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2UxY2QyNGVkZWZlZTQ4M2U4MWU4NjU5OTlhY2M4MWFkID0gJCgnPGRpdiBpZD0iaHRtbF9lMWNkMjRlZGVmZWU0ODNlODFlODY1OTk5YWNjODFhZCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTE5NDUsIExBQkVMOjcsIFNFR01FTlQ6V1RDIC0gQ29tbWVyY2lhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNDFkNzgwNzQ3ZjU5NDIxYWJiODJjM2ZiMDU2MDczODYuc2V0Q29udGVudChodG1sX2UxY2QyNGVkZWZlZTQ4M2U4MWU4NjU5OTlhY2M4MWFkKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfNzEzMTYzMDllODlmNGNjOWE2M2I1OGY5NTUwYjZiODEuYmluZFBvcHVwKHBvcHVwXzQxZDc4MDc0N2Y1OTQyMWFiYjgyYzNmYjA1NjA3Mzg2KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzFjYzAwNGVjODZjZTQyYzU5OGM2YWI4OGE2Y2IyYTNiID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuNDkyMjU0LDExNC4xMzk3NDFdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fMzA0NzU1ZjI4MmU1NDg1MmE1NmQ4MmEyYzE2NmJiNDYgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2JlaWdlJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzFjYzAwNGVjODZjZTQyYzU5OGM2YWI4OGE2Y2IyYTNiLnNldEljb24oaWNvbl8zMDQ3NTVmMjgyZTU0ODUyYTU2ZDgyYTJjMTY2YmI0Nik7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF84YTVmZDljZDVkMTI0ZGFhOTJkNDZkOWU2NTY0YzA0MSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9jYmVmYjZmZGE1ZDQ0OTczYTkyNTVjNmYwOTAxMmQ3NSA9ICQoJzxkaXYgaWQ9Imh0bWxfY2JlZmI2ZmRhNWQ0NDk3M2E5MjU1YzZmMDkwMTJkNzUiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyNjIyLCBMQUJFTDo3LCBTRUdNRU5UOldUQyAtIFRvdXJpc3RzL0NvbnY8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzhhNWZkOWNkNWQxMjRkYWE5MmQ0NmQ5ZTY1NjRjMDQxLnNldENvbnRlbnQoaHRtbF9jYmVmYjZmZGE1ZDQ0OTczYTkyNTVjNmYwOTAxMmQ3NSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzFjYzAwNGVjODZjZTQyYzU5OGM2YWI4OGE2Y2IyYTNiLmJpbmRQb3B1cChwb3B1cF84YTVmZDljZDVkMTI0ZGFhOTJkNDZkOWU2NTY0YzA0MSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9mNGQ1MzE5MjZlNjg0MWJmYWI3YzUzMzgxYTc3N2MwMSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjUwNDYyOSwxMTQuMTI4MjYyXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzkxNjZiOGE4NTZjNzRkMzg5MTZmMGI4ZDc5Y2UxNjNkID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdsaWdodHJlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl9mNGQ1MzE5MjZlNjg0MWJmYWI3YzUzMzgxYTc3N2MwMS5zZXRJY29uKGljb25fOTE2NmI4YTg1NmM3NGQzODkxNmYwYjhkNzljZTE2M2QpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfZTM4ZjYxMzFhNDJlNGJiYTgyOTgzNDRlMmJhMGI4ODIgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNGNkZjBhMjgwM2RkNDFhODhjMjE2YzllODc4NWRkNTcgPSAkKCc8ZGl2IGlkPSJodG1sXzRjZGYwYTI4MDNkZDQxYTg4YzIxNmM5ZTg3ODVkZDU3IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjA3MCwgTEFCRUw6NiwgU0VHTUVOVDpXVEMgLSBUb3VyaXN0cy9Db252PC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9lMzhmNjEzMWE0MmU0YmJhODI5ODM0NGUyYmEwYjg4Mi5zZXRDb250ZW50KGh0bWxfNGNkZjBhMjgwM2RkNDFhODhjMjE2YzllODc4NWRkNTcpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl9mNGQ1MzE5MjZlNjg0MWJmYWI3YzUzMzgxYTc3N2MwMS5iaW5kUG9wdXAocG9wdXBfZTM4ZjYxMzFhNDJlNGJiYTgyOTgzNDRlMmJhMGI4ODIpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfYjBlNmQ3NTJjZDc4NDViOWIzYjM4ZjczYjJlZjAwY2YgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4yODMzNTQ4LDExNC4yMjEyNjM4XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzlkZjJkYWIxN2EyOTQ3ZTI5YTUyOTU2OTM4YWRiMzBmID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdncmVlbicsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl9iMGU2ZDc1MmNkNzg0NWI5YjNiMzhmNzNiMmVmMDBjZi5zZXRJY29uKGljb25fOWRmMmRhYjE3YTI5NDdlMjlhNTI5NTY5MzhhZGIzMGYpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfN2QwZmM1NzBjNGEwNDkwZmIwM2FhMTU0NGE3NzZlMjcgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfMWQwOWU3NWE1MzJiNGM2N2E2NTMxZGM2YWI2MTJiOTggPSAkKCc8ZGl2IGlkPSJodG1sXzFkMDllNzVhNTMyYjRjNjdhNjUzMWRjNmFiNjEyYjk4IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjE4MCwgTEFCRUw6MiwgU0VHTUVOVDpXVEMgLSBTYXZlcnM8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzdkMGZjNTcwYzRhMDQ5MGZiMDNhYTE1NDRhNzc2ZTI3LnNldENvbnRlbnQoaHRtbF8xZDA5ZTc1YTUzMmI0YzY3YTY1MzFkYzZhYjYxMmI5OCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyX2IwZTZkNzUyY2Q3ODQ1YjliM2IzOGY3M2IyZWYwMGNmLmJpbmRQb3B1cChwb3B1cF83ZDBmYzU3MGM0YTA0OTBmYjAzYWExNTQ0YTc3NmUyNyk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl83MTMzNGUzYTE4OTQ0ZDI5OTY0ZjYwYTQ5YTY4NTBhNCA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjMzMzU1NjQsMTE0LjE5Njk3ODNdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fMDdhMjUyYmM5Y2NmNDg1NTljNmNkMmNmNmUwYTlkYjYgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2dyZWVuJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzcxMzM0ZTNhMTg5NDRkMjk5NjRmNjBhNDlhNjg1MGE0LnNldEljb24oaWNvbl8wN2EyNTJiYzljY2Y0ODU1OWM2Y2QyY2Y2ZTBhOWRiNik7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF80OTRjYmIwZTQ0NDY0NWZkYjVjM2UzOWE3OTQ0NTdhYyA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF82NDQ0ODQzNzMyZjU0OTM1YmI3YzM2YWY3MGQ2OWYzMiA9ICQoJzxkaXYgaWQ9Imh0bWxfNjQ0NDg0MzczMmY1NDkzNWJiN2MzNmFmNzBkNjlmMzIiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjExOTk2LCBMQUJFTDoyLCBTRUdNRU5UOldUQyAtIFNhdmVyczwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNDk0Y2JiMGU0NDQ2NDVmZGI1YzNlMzlhNzk0NDU3YWMuc2V0Q29udGVudChodG1sXzY0NDQ4NDM3MzJmNTQ5MzViYjdjMzZhZjcwZDY5ZjMyKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfNzEzMzRlM2ExODk0NGQyOTk2NGY2MGE0OWE2ODUwYTQuYmluZFBvcHVwKHBvcHVwXzQ5NGNiYjBlNDQ0NjQ1ZmRiNWMzZTM5YTc5NDQ1N2FjKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzI2NjE4NjU0NGI5YjQ0YTE5YTA0ZDIwZDVjYjE5Mzk4ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzk2ODAzLDExMy45NzUyODUzXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzljYWJhZTA5ZmMwOTRkNmQ4ODQwNDdmMzJiNTZhZjYyID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfMjY2MTg2NTQ0YjliNDRhMTlhMDRkMjBkNWNiMTkzOTguc2V0SWNvbihpY29uXzljYWJhZTA5ZmMwOTRkNmQ4ODQwNDdmMzJiNTZhZjYyKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2U1YTU3MjM3M2UyZDQxNjk4NGExYjEwMDFiNWZmOWNmID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzlkN2JiZjljOWE2NjQ1MWRhY2M3MjdkYjY2Zjg4MGU2ID0gJCgnPGRpdiBpZD0iaHRtbF85ZDdiYmY5YzlhNjY0NTFkYWNjNzI3ZGI2NmY4ODBlNiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIwNTksIExBQkVMOjAsIFNFR01FTlQ6V1RDIC0gVG91cmlzdHMvQ29udjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfZTVhNTcyMzczZTJkNDE2OTg0YTFiMTAwMWI1ZmY5Y2Yuc2V0Q29udGVudChodG1sXzlkN2JiZjljOWE2NjQ1MWRhY2M3MjdkYjY2Zjg4MGU2KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfMjY2MTg2NTQ0YjliNDRhMTlhMDRkMjBkNWNiMTkzOTguYmluZFBvcHVwKHBvcHVwX2U1YTU3MjM3M2UyZDQxNjk4NGExYjEwMDFiNWZmOWNmKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzkyZWY4OGFlZTI3YjQ1ZDBiNjk4NWYwNzVhNTE3ZGFjID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuNDAyNDEwMSwxMTMuOTc3NjIxM10sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl83MmFiN2M5ZTVhNzM0NjQ2YWRmMmY0NGRiMGY5ODBkMiA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnYmVpZ2UnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfOTJlZjg4YWVlMjdiNDVkMGI2OTg1ZjA3NWE1MTdkYWMuc2V0SWNvbihpY29uXzcyYWI3YzllNWE3MzQ2NDZhZGYyZjQ0ZGIwZjk4MGQyKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2I5ZDc3NGFhMjFiOTRmYmE4ZjhmNWQ5OTYzZDdhMDAxID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2Q2YWNiNzc3MjE2MzQ5NWNiOTZhNDVjYjgwMDIzNTBjID0gJCgnPGRpdiBpZD0iaHRtbF9kNmFjYjc3NzIxNjM0OTVjYjk2YTQ1Y2I4MDAyMzUwYyIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTI0NDMsIExBQkVMOjcsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2I5ZDc3NGFhMjFiOTRmYmE4ZjhmNWQ5OTYzZDdhMDAxLnNldENvbnRlbnQoaHRtbF9kNmFjYjc3NzIxNjM0OTVjYjk2YTQ1Y2I4MDAyMzUwYyk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzkyZWY4OGFlZTI3YjQ1ZDBiNjk4NWYwNzVhNTE3ZGFjLmJpbmRQb3B1cChwb3B1cF9iOWQ3NzRhYTIxYjk0ZmJhOGY4ZjVkOTk2M2Q3YTAwMSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl8zMTgxMjEwNjAyMTg0MWNmOWY3NTNjMmEyZWM4OWIyNyA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjM0MzY0NjIsMTE0LjExMDg2MzJdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fOGU5OTYxNWIxZjk3NDNjNjlhMjNmZTI2OTk0NDExMzIgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2JlaWdlJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzMxODEyMTA2MDIxODQxY2Y5Zjc1M2MyYTJlYzg5YjI3LnNldEljb24oaWNvbl84ZTk5NjE1YjFmOTc0M2M2OWEyM2ZlMjY5OTQ0MTEzMik7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF85NjEwMWMwN2MwZTc0MmU3OTJjYzdlYmExMzU4NDQxNiA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF80MzExZjQxNWU2NDE0YTk2YWFhNzU4NTc2YTgzYWEzMSA9ICQoJzxkaXYgaWQ9Imh0bWxfNDMxMWY0MTVlNjQxNGE5NmFhYTc1ODU3NmE4M2FhMzEiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjExOTU2LCBMQUJFTDo3LCBTRUdNRU5UOldUQyAtIFRvdXJpc3RzL0NvbnY8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzk2MTAxYzA3YzBlNzQyZTc5MmNjN2ViYTEzNTg0NDE2LnNldENvbnRlbnQoaHRtbF80MzExZjQxNWU2NDE0YTk2YWFhNzU4NTc2YTgzYWEzMSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzMxODEyMTA2MDIxODQxY2Y5Zjc1M2MyYTJlYzg5YjI3LmJpbmRQb3B1cChwb3B1cF85NjEwMWMwN2MwZTc0MmU3OTJjYzdlYmExMzU4NDQxNik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl80MmI5NTFkYTA5YmY0ODkwOWVmMzY4YTNlMzYyZWIwNiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjIxMTM0NjcsMTEzLjU1MDU2NDldLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fYTkyZjZmOTFmMGIwNGFlZDk4Y2FhZmNiODY0ZTIxZTggPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2xpZ2h0cmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzQyYjk1MWRhMDliZjQ4OTA5ZWYzNjhhM2UzNjJlYjA2LnNldEljb24oaWNvbl9hOTJmNmY5MWYwYjA0YWVkOThjYWFmY2I4NjRlMjFlOCk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8xMzJlMDY0MDllYjk0ODU1OGUyZjdhOThjYjMwNjI0YiA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF85MTBhNjg1NzM4N2Q0YTAxOWNjNTkxZjlkMTkwYzhlMyA9ICQoJzxkaXYgaWQ9Imh0bWxfOTEwYTY4NTczODdkNGEwMTljYzU5MWY5ZDE5MGM4ZTMiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjExOTgzLCBMQUJFTDo2LCBTRUdNRU5UOldUQyAtIFJlc2lkZW50aWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8xMzJlMDY0MDllYjk0ODU1OGUyZjdhOThjYjMwNjI0Yi5zZXRDb250ZW50KGh0bWxfOTEwYTY4NTczODdkNGEwMTljYzU5MWY5ZDE5MGM4ZTMpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl80MmI5NTFkYTA5YmY0ODkwOWVmMzY4YTNlMzYyZWIwNi5iaW5kUG9wdXAocG9wdXBfMTMyZTA2NDA5ZWI5NDg1NThlMmY3YTk4Y2IzMDYyNGIpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfZjZlZjQyZjdkMTdlNGIzOTg2ZjhkMGYxMTU2MDNhNWYgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4yODE1ODMzLDExNC4xNTE3MTg5XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzNlNTc2NDY4YjUzYzRmMmY5NDY0ZGJiYTk3MTFiYTA1ID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdiZWlnZScsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl9mNmVmNDJmN2QxN2U0YjM5ODZmOGQwZjExNTYwM2E1Zi5zZXRJY29uKGljb25fM2U1NzY0NjhiNTNjNGYyZjk0NjRkYmJhOTcxMWJhMDUpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNTZjMzRhMmNhMzZmNGMyMmI5YWViZTE3ZDc4ZmZhMzcgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfOTY4ZTVlZmZjMWQ3NGY3M2JhNmIwNDllN2I3MmExODAgPSAkKCc8ZGl2IGlkPSJodG1sXzk2OGU1ZWZmYzFkNzRmNzNiYTZiMDQ5ZTdiNzJhMTgwIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjM0NCwgTEFCRUw6NywgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNTZjMzRhMmNhMzZmNGMyMmI5YWViZTE3ZDc4ZmZhMzcuc2V0Q29udGVudChodG1sXzk2OGU1ZWZmYzFkNzRmNzNiYTZiMDQ5ZTdiNzJhMTgwKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfZjZlZjQyZjdkMTdlNGIzOTg2ZjhkMGYxMTU2MDNhNWYuYmluZFBvcHVwKHBvcHVwXzU2YzM0YTJjYTM2ZjRjMjJiOWFlYmUxN2Q3OGZmYTM3KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2YzZTU1NWFjYjE2ZjQzYTg5MjAzN2EwOGVmOTc0MGVmID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMTkzNTMzNCwxMTMuNTM5NzU5M10sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl8xOTQzOWViN2EzYWE0MGZmODQ1ZTRjZDU4NTViOTZmZSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnb3JhbmdlJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyX2YzZTU1NWFjYjE2ZjQzYTg5MjAzN2EwOGVmOTc0MGVmLnNldEljb24oaWNvbl8xOTQzOWViN2EzYWE0MGZmODQ1ZTRjZDU4NTViOTZmZSk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9jNjE1ZmRiOTVhMjA0ZGZmYTI4YTc1ZjlkNGI5ODY1NyA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF82OThmODE2OTQ1N2Y0ZjQ4YTk3ZTljZTlkNDExYjYwMCA9ICQoJzxkaXYgaWQ9Imh0bWxfNjk4ZjgxNjk0NTdmNGY0OGE5N2U5Y2U5ZDQxMWI2MDAiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjExODgwLCBMQUJFTDo0LCBTRUdNRU5UOldUQyAtIERlc3RpbmF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9jNjE1ZmRiOTVhMjA0ZGZmYTI4YTc1ZjlkNGI5ODY1Ny5zZXRDb250ZW50KGh0bWxfNjk4ZjgxNjk0NTdmNGY0OGE5N2U5Y2U5ZDQxMWI2MDApOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl9mM2U1NTVhY2IxNmY0M2E4OTIwMzdhMDhlZjk3NDBlZi5iaW5kUG9wdXAocG9wdXBfYzYxNWZkYjk1YTIwNGRmZmEyOGE3NWY5ZDRiOTg2NTcpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfYTE2NTU0MzU5NTQxNDQyYWI4NTI2MjdmZTRlNzA5MTUgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4yMDE2MTc5LDExNC4wMjY1MDA3XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzI0ZDkxYWI5NTQ5MjQ3MjA5NWM5YmY2MDc3MjIwMWY5ID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdiZWlnZScsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl9hMTY1NTQzNTk1NDE0NDJhYjg1MjYyN2ZlNGU3MDkxNS5zZXRJY29uKGljb25fMjRkOTFhYjk1NDkyNDcyMDk1YzliZjYwNzcyMjAxZjkpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNDM1ZjBiMmYzYTIyNDU3MmFhM2JiOTUxZGQyYzc2M2MgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfMWJmMjFkYTc5MDRkNDIxZjgzZjVkZmI1NjUzZmQyNGMgPSAkKCc8ZGl2IGlkPSJodG1sXzFiZjIxZGE3OTA0ZDQyMWY4M2Y1ZGZiNTY1M2ZkMjRjIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMTkwMSwgTEFCRUw6NywgU0VHTUVOVDpXVEMgLSBUb3VyaXN0cy9Db252PC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF80MzVmMGIyZjNhMjI0NTcyYWEzYmI5NTFkZDJjNzYzYy5zZXRDb250ZW50KGh0bWxfMWJmMjFkYTc5MDRkNDIxZjgzZjVkZmI1NjUzZmQyNGMpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl9hMTY1NTQzNTk1NDE0NDJhYjg1MjYyN2ZlNGU3MDkxNS5iaW5kUG9wdXAocG9wdXBfNDM1ZjBiMmYzYTIyNDU3MmFhM2JiOTUxZGQyYzc2M2MpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfNjUwN2VmYmM0MGRiNDQxMGI0NTVjZjA3MjlhZmVhNjUgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4yNzg3NDYsMTE0LjIyODY0NzZdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fZjhhZjI3MDY0ZTk5NDk5NThmMzFhZmU4NTY0ZmU5ZWQgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3JlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl82NTA3ZWZiYzQwZGI0NDEwYjQ1NWNmMDcyOWFmZWE2NS5zZXRJY29uKGljb25fZjhhZjI3MDY0ZTk5NDk5NThmMzFhZmU4NTY0ZmU5ZWQpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNDcxZmMwMWUzNjVhNDkxMDhhYzBhYjMyYzNlNTAwZGEgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfY2QxMzg0MzhmZWU1NDA4YTllYTFhMWQ2NmY0MGI2YWUgPSAkKCc8ZGl2IGlkPSJodG1sX2NkMTM4NDM4ZmVlNTQwOGE5ZWExYTFkNjZmNDBiNmFlIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMTkzMiwgTEFCRUw6MCwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNDcxZmMwMWUzNjVhNDkxMDhhYzBhYjMyYzNlNTAwZGEuc2V0Q29udGVudChodG1sX2NkMTM4NDM4ZmVlNTQwOGE5ZWExYTFkNjZmNDBiNmFlKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfNjUwN2VmYmM0MGRiNDQxMGI0NTVjZjA3MjlhZmVhNjUuYmluZFBvcHVwKHBvcHVwXzQ3MWZjMDFlMzY1YTQ5MTA4YWMwYWIzMmMzZTUwMGRhKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzdlODU3YmMzOTNiMjQzYmE4YTg0OTZhMGQ2NDg0OTUyID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzA3MzIyMSwxMTQuMTg2NzE2Ml0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl8xOWE4M2FjYTkyNzk0OTVkYTkzYzU4Nzc2OWEzM2I4YSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnZ3JlZW4nLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfN2U4NTdiYzM5M2IyNDNiYThhODQ5NmEwZDY0ODQ5NTIuc2V0SWNvbihpY29uXzE5YTgzYWNhOTI3OTQ5NWRhOTNjNTg3NzY5YTMzYjhhKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzEzZTcyNWZmZTg5ZTQwODI5NDNmYzljZWNhZDU3MGYzID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzUzNzMzYjE3NGVjNzRiYThhNzJjMWY2ZGNkOTM5NzcyID0gJCgnPGRpdiBpZD0iaHRtbF81MzczM2IxNzRlYzc0YmE4YTcyYzFmNmRjZDkzOTc3MiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIwNTMsIExBQkVMOjIsIFNFR01FTlQ6V1RDIC0gSGVhbHRoPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8xM2U3MjVmZmU4OWU0MDgyOTQzZmM5Y2VjYWQ1NzBmMy5zZXRDb250ZW50KGh0bWxfNTM3MzNiMTc0ZWM3NGJhOGE3MmMxZjZkY2Q5Mzk3NzIpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl83ZTg1N2JjMzkzYjI0M2JhOGE4NDk2YTBkNjQ4NDk1Mi5iaW5kUG9wdXAocG9wdXBfMTNlNzI1ZmZlODllNDA4Mjk0M2ZjOWNlY2FkNTcwZjMpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfY2I5MDBkMTEyMThkNGYzMTlhNTg5N2RhN2ZjYzBhNWMgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi40NTMwMTE2LDExNC4xNzU0MTIzXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzY5NTdmYjg3YzBjYTRmZjdiOTYwNWMyNTAzN2UxZDY3ID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdiZWlnZScsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl9jYjkwMGQxMTIxOGQ0ZjMxOWE1ODk3ZGE3ZmNjMGE1Yy5zZXRJY29uKGljb25fNjk1N2ZiODdjMGNhNGZmN2I5NjA1YzI1MDM3ZTFkNjcpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfYjdlM2M5MzVlZTM0NDU1YmFkZmUyMTM5YWYyMzkwN2EgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNGMzMjI4Y2M3NjEyNDYzZmFlN2FlNDVjMTRiZDRlYjkgPSAkKCc8ZGl2IGlkPSJodG1sXzRjMzIyOGNjNzYxMjQ2M2ZhZTdhZTQ1YzE0YmQ0ZWI5IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjU2NSwgTEFCRUw6NywgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfYjdlM2M5MzVlZTM0NDU1YmFkZmUyMTM5YWYyMzkwN2Euc2V0Q29udGVudChodG1sXzRjMzIyOGNjNzYxMjQ2M2ZhZTdhZTQ1YzE0YmQ0ZWI5KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfY2I5MDBkMTEyMThkNGYzMTlhNTg5N2RhN2ZjYzBhNWMuYmluZFBvcHVwKHBvcHVwX2I3ZTNjOTM1ZWUzNDQ1NWJhZGZlMjEzOWFmMjM5MDdhKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzUxN2FkYzk4NTZhODRmNzhiMzMzNTEyYjYyY2QyNmZlID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzcyMzIyNSwxMTQuMTIxODAyNV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9lNDBmYjMzYjU2NmQ0YjY5ODQ3Y2I1MmFjOWE0MjA4YiA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnZ3JlZW4nLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfNTE3YWRjOTg1NmE4NGY3OGIzMzM1MTJiNjJjZDI2ZmUuc2V0SWNvbihpY29uX2U0MGZiMzNiNTY2ZDRiNjk4NDdjYjUyYWM5YTQyMDhiKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzEyYjQ4OGUyNjRlNDQ0NjZhY2FhMGExODJmNzc1NjYyID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2M2ZDdhNjI0OWFjZDRiODc5MGFjNzk0ZmU4OWMzZWM5ID0gJCgnPGRpdiBpZD0iaHRtbF9jNmQ3YTYyNDlhY2Q0Yjg3OTBhYzc5NGZlODljM2VjOSIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTE5NjMsIExBQkVMOjIsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzEyYjQ4OGUyNjRlNDQ0NjZhY2FhMGExODJmNzc1NjYyLnNldENvbnRlbnQoaHRtbF9jNmQ3YTYyNDlhY2Q0Yjg3OTBhYzc5NGZlODljM2VjOSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzUxN2FkYzk4NTZhODRmNzhiMzMzNTEyYjYyY2QyNmZlLmJpbmRQb3B1cChwb3B1cF8xMmI0ODhlMjY0ZTQ0NDY2YWNhYTBhMTgyZjc3NTY2Mik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl8wZGY1YzU1NWI1NWM0OGYyYTdiNGE4ODYzNjFlZTllMSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjM1NTc0OTEsMTE0LjEwODMwMTZdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fZDAwMGQ2Yjc0ZDZiNDUzZTk1ZWY4NmFiOWI2N2I1ODUgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3JlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl8wZGY1YzU1NWI1NWM0OGYyYTdiNGE4ODYzNjFlZTllMS5zZXRJY29uKGljb25fZDAwMGQ2Yjc0ZDZiNDUzZTk1ZWY4NmFiOWI2N2I1ODUpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfZmVkNDEyZWQ1ODdmNDZjMWJhZjBjMWFiYjRiZmYyYWUgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfZmE4NGY0NTc1ZTc5NGVmZjliYjY3YmVhNGQ0YTRmZDYgPSAkKCc8ZGl2IGlkPSJodG1sX2ZhODRmNDU3NWU3OTRlZmY5YmI2N2JlYTRkNGE0ZmQ2IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjMwMywgTEFCRUw6MCwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfZmVkNDEyZWQ1ODdmNDZjMWJhZjBjMWFiYjRiZmYyYWUuc2V0Q29udGVudChodG1sX2ZhODRmNDU3NWU3OTRlZmY5YmI2N2JlYTRkNGE0ZmQ2KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfMGRmNWM1NTViNTVjNDhmMmE3YjRhODg2MzYxZWU5ZTEuYmluZFBvcHVwKHBvcHVwX2ZlZDQxMmVkNTg3ZjQ2YzFiYWYwYzFhYmI0YmZmMmFlKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzdhNjIwZjc5MzBjNTRmYzk5MTkxYzYzZjlmYmUzNzIzID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMjgzMDkxMiwxMTQuMTU3NTIxMV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9lODU4MzhlMjI5ZDI0MWNkOTRlMGIzMTc5Y2I1MmQ3MyA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnbGlnaHRyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfN2E2MjBmNzkzMGM1NGZjOTkxOTFjNjNmOWZiZTM3MjMuc2V0SWNvbihpY29uX2U4NTgzOGUyMjlkMjQxY2Q5NGUwYjMxNzljYjUyZDczKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzdmMzBiZDgzMzkyYTRhMDliNWVkMzY2MDg5MmYwNTNhID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2U5NDM5Y2IwNDhlMDQ3Y2VhZDBhMDlmYjExMGQ4MTJmID0gJCgnPGRpdiBpZD0iaHRtbF9lOTQzOWNiMDQ4ZTA0N2NlYWQwYTA5ZmIxMTBkODEyZiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTI2MTAsIExBQkVMOjYsIFNFR01FTlQ6V1RDIC0gQ29tbWVyY2lhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfN2YzMGJkODMzOTJhNGEwOWI1ZWQzNjYwODkyZjA1M2Euc2V0Q29udGVudChodG1sX2U5NDM5Y2IwNDhlMDQ3Y2VhZDBhMDlmYjExMGQ4MTJmKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfN2E2MjBmNzkzMGM1NGZjOTkxOTFjNjNmOWZiZTM3MjMuYmluZFBvcHVwKHBvcHVwXzdmMzBiZDgzMzkyYTRhMDliNWVkMzY2MDg5MmYwNTNhKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzNlOWY0YTFhMGYzMTQ4Y2NhNWNjODI4NjQ1YWFmNDMxID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMjgyNTE3NSwxMTQuMTI5ODM4M10sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl83NWJkOTBiNDZhOWM0NmMxYWQwODc4YjY4ZGVhOTNhOCA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAncmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzNlOWY0YTFhMGYzMTQ4Y2NhNWNjODI4NjQ1YWFmNDMxLnNldEljb24oaWNvbl83NWJkOTBiNDZhOWM0NmMxYWQwODc4YjY4ZGVhOTNhOCk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9iZTQ2NWRkMjliMTg0NThlOTIyYjc0MjMyMGFmZGRkNSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9mZWFjM2MyNWI4Njk0ODAxYTUwMGEwYjVkYmY4ODM3YSA9ICQoJzxkaXYgaWQ9Imh0bWxfZmVhYzNjMjViODY5NDgwMWE1MDBhMGI1ZGJmODgzN2EiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMDAxLCBMQUJFTDowLCBTRUdNRU5UOldUQyAtIFJlc2lkZW50aWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9iZTQ2NWRkMjliMTg0NThlOTIyYjc0MjMyMGFmZGRkNS5zZXRDb250ZW50KGh0bWxfZmVhYzNjMjViODY5NDgwMWE1MDBhMGI1ZGJmODgzN2EpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl8zZTlmNGExYTBmMzE0OGNjYTVjYzgyODY0NWFhZjQzMS5iaW5kUG9wdXAocG9wdXBfYmU0NjVkZDI5YjE4NDU4ZTkyMmI3NDIzMjBhZmRkZDUpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfYjljMmUxZGQ1YmViNGRjZWE4ZGYwOGIxYzI0ZjM2OTMgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4zMDg0MjI3LDExNC4yMzQ1MzldLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fYzJhZjQ5NjUzZDAzNDEyMzg3MjNmNTdiZDkzZWI5N2UgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3JlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl9iOWMyZTFkZDViZWI0ZGNlYThkZjA4YjFjMjRmMzY5My5zZXRJY29uKGljb25fYzJhZjQ5NjUzZDAzNDEyMzg3MjNmNTdiZDkzZWI5N2UpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfMGQ3YWQ0MmU2YjA2NDhmZWFkMzYxMWNiNWRhYmFhNWIgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfYTY1OWFiZDI2MDQ0NDAwNDg1NGQyOTM1NDJkNGU3MDEgPSAkKCc8ZGl2IGlkPSJodG1sX2E2NTlhYmQyNjA0NDQwMDQ4NTRkMjkzNTQyZDRlNzAxIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjAzNSwgTEFCRUw6MCwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMGQ3YWQ0MmU2YjA2NDhmZWFkMzYxMWNiNWRhYmFhNWIuc2V0Q29udGVudChodG1sX2E2NTlhYmQyNjA0NDQwMDQ4NTRkMjkzNTQyZDRlNzAxKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfYjljMmUxZGQ1YmViNGRjZWE4ZGYwOGIxYzI0ZjM2OTMuYmluZFBvcHVwKHBvcHVwXzBkN2FkNDJlNmIwNjQ4ZmVhZDM2MTFjYjVkYWJhYTViKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzYwNzYxZGU5ODJiNzQ4ZWM4ZDNmZmQxNDI2MDgwYmU0ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzE4MzY2NSwxMTQuMTcxNzU2Nl0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9mNWE3MGJmMzQ0MDk0NjBiOWU2NmRmYjc1Y2E0MmI4YSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnbGlnaHRyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfNjA3NjFkZTk4MmI3NDhlYzhkM2ZmZDE0MjYwODBiZTQuc2V0SWNvbihpY29uX2Y1YTcwYmYzNDQwOTQ2MGI5ZTY2ZGZiNzVjYTQyYjhhKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzVjN2IxYjY1ZWEwYTRiYjk4MTJmYjVmNjE1OTUzOWFhID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzM3N2JkYzg0OWI2ZDQyZjhhZWM0ZjJhM2FiMDRhMDU3ID0gJCgnPGRpdiBpZD0iaHRtbF8zNzdiZGM4NDliNmQ0MmY4YWVjNGYyYTNhYjA0YTA1NyIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIxMTEsIExBQkVMOjYsIFNFR01FTlQ6V1RDIC0gVG91cmlzdHMvQ29udjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNWM3YjFiNjVlYTBhNGJiOTgxMmZiNWY2MTU5NTM5YWEuc2V0Q29udGVudChodG1sXzM3N2JkYzg0OWI2ZDQyZjhhZWM0ZjJhM2FiMDRhMDU3KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfNjA3NjFkZTk4MmI3NDhlYzhkM2ZmZDE0MjYwODBiZTQuYmluZFBvcHVwKHBvcHVwXzVjN2IxYjY1ZWEwYTRiYjk4MTJmYjVmNjE1OTUzOWFhKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2Q3ZDY5ZGQwODhiNTRmYjdiNWM3YmE2YmZkODM1MDRhID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMjgwMzAzOSwxMTQuMTgyNzUxN10sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9kOGVmNzc1MGZhY2I0YzIwODE5MGFlM2E3MzZkMDdjNiA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnb3JhbmdlJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyX2Q3ZDY5ZGQwODhiNTRmYjdiNWM3YmE2YmZkODM1MDRhLnNldEljb24oaWNvbl9kOGVmNzc1MGZhY2I0YzIwODE5MGFlM2E3MzZkMDdjNik7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF81M2M1M2M0MWNlNGM0NDg4OGRiOGIyNGU5MjI1NjM1MSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF84MDM3ZWY0NGQ1ZDY0YmExOTRhZjM0NTVhOTljNTdhOSA9ICQoJzxkaXYgaWQ9Imh0bWxfODAzN2VmNDRkNWQ2NGJhMTk0YWYzNDU1YTk5YzU3YTkiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMzg4LCBMQUJFTDo0LCBTRUdNRU5UOldUQyAtIFRvdXJpc3RzL0NvbnY8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzUzYzUzYzQxY2U0YzQ0ODg4ZGI4YjI0ZTkyMjU2MzUxLnNldENvbnRlbnQoaHRtbF84MDM3ZWY0NGQ1ZDY0YmExOTRhZjM0NTVhOTljNTdhOSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyX2Q3ZDY5ZGQwODhiNTRmYjdiNWM3YmE2YmZkODM1MDRhLmJpbmRQb3B1cChwb3B1cF81M2M1M2M0MWNlNGM0NDg4OGRiOGIyNGU5MjI1NjM1MSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl8yMjQxM2M0YWUzZmE0Y2IzYTNkNGIwMTI0YThmZTlkNiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjQ0NTA1ODQsMTE0LjAyNzQzNjJdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fZDFkZTdiZDQxNWU1NDkyNTgzOGY2ZDZhMzdmODhkYTIgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2xpZ2h0cmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzIyNDEzYzRhZTNmYTRjYjNhM2Q0YjAxMjRhOGZlOWQ2LnNldEljb24oaWNvbl9kMWRlN2JkNDE1ZTU0OTI1ODM4ZjZkNmEzN2Y4OGRhMik7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF83ZTg1MTI3YWEwZmY0Nzc2YjRmYTAxYTk1Y2VkY2QwMiA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF8xZDY2Mjg5MjRjZGE0NWI3YmNmMWI5M2QxOTE5ZjA4YSA9ICQoJzxkaXYgaWQ9Imh0bWxfMWQ2NjI4OTI0Y2RhNDViN2JjZjFiOTNkMTkxOWYwOGEiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjExODQ5LCBMQUJFTDo2LCBTRUdNRU5UOldUQyAtIFRvdXJpc3RzL0NvbnY8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzdlODUxMjdhYTBmZjQ3NzZiNGZhMDFhOTVjZWRjZDAyLnNldENvbnRlbnQoaHRtbF8xZDY2Mjg5MjRjZGE0NWI3YmNmMWI5M2QxOTE5ZjA4YSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzIyNDEzYzRhZTNmYTRjYjNhM2Q0YjAxMjRhOGZlOWQ2LmJpbmRQb3B1cChwb3B1cF83ZTg1MTI3YWEwZmY0Nzc2YjRmYTAxYTk1Y2VkY2QwMik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl85YTUwYjc2MzZiZmE0MDJlYmNhZjM3NTg2MzExZDQxOSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjE4OTIxODMsMTEzLjU1MTMyN10sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl82MzgzYTA5MDY2YjE0OWIwOTZkMjE1YzM0ZjYxZWNkZSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAncHVycGxlJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzlhNTBiNzYzNmJmYTQwMmViY2FmMzc1ODYzMTFkNDE5LnNldEljb24oaWNvbl82MzgzYTA5MDY2YjE0OWIwOTZkMjE1YzM0ZjYxZWNkZSk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9kZDQzOTczMmUyYzE0N2Q4YjA5OGQzMWIwYzBiMmYxZSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9jOGJiMDEwZWIzNWY0YmI5YTkzYWNjNjA4ZmRkMzU4OCA9ICQoJzxkaXYgaWQ9Imh0bWxfYzhiYjAxMGViMzVmNGJiOWE5M2FjYzYwOGZkZDM1ODgiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMTczLCBMQUJFTDozLCBTRUdNRU5UOldUQyAtIFJlc2lkZW50aWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9kZDQzOTczMmUyYzE0N2Q4YjA5OGQzMWIwYzBiMmYxZS5zZXRDb250ZW50KGh0bWxfYzhiYjAxMGViMzVmNGJiOWE5M2FjYzYwOGZkZDM1ODgpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl85YTUwYjc2MzZiZmE0MDJlYmNhZjM3NTg2MzExZDQxOS5iaW5kUG9wdXAocG9wdXBfZGQ0Mzk3MzJlMmMxNDdkOGIwOThkMzFiMGMwYjJmMWUpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfOWQwN2I1YWIzZjBjNDM1N2E3NWMxODk2NTM0ZjdhYjYgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4zMDIwMDUsMTE0LjE4MzcyMV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl8yZWQ0YmFjZmFlNDk0MDY4OWZmMjc3NTBhMGRiN2M5ZCA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnbGlnaHRyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfOWQwN2I1YWIzZjBjNDM1N2E3NWMxODk2NTM0ZjdhYjYuc2V0SWNvbihpY29uXzJlZDRiYWNmYWU0OTQwNjg5ZmYyNzc1MGEwZGI3YzlkKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2EyZTMwODFmYzdkZDRkZTY5ZjZhYjYzZjE1MzRlZDYwID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzg1OTViZWZiMDIxZTRkMzE4ZGVhZDczZTdiOTM2MzcxID0gJCgnPGRpdiBpZD0iaHRtbF84NTk1YmVmYjAyMWU0ZDMxOGRlYWQ3M2U3YjkzNjM3MSIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIzMjgsIExBQkVMOjYsIFNFR01FTlQ6V1RDIC0gVG91cmlzdHMvQ29udjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfYTJlMzA4MWZjN2RkNGRlNjlmNmFiNjNmMTUzNGVkNjAuc2V0Q29udGVudChodG1sXzg1OTViZWZiMDIxZTRkMzE4ZGVhZDczZTdiOTM2MzcxKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfOWQwN2I1YWIzZjBjNDM1N2E3NWMxODk2NTM0ZjdhYjYuYmluZFBvcHVwKHBvcHVwX2EyZTMwODFmYzdkZDRkZTY5ZjZhYjYzZjE1MzRlZDYwKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzUzNmM5OTE5OTI5MDQyMWFiMDk0MjU2MTMyZGZkZmY0ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzA4NjA0NCwxMTQuMTgzNDgzN10sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl8yYTYxZmRhMzIzN2Y0NTlkOTAzYTg1ZWEyZDU5NjUwZSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnZ3JlZW4nLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfNTM2Yzk5MTk5MjkwNDIxYWIwOTQyNTYxMzJkZmRmZjQuc2V0SWNvbihpY29uXzJhNjFmZGEzMjM3ZjQ1OWQ5MDNhODVlYTJkNTk2NTBlKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzA5YTUyOGE0ZmViZTRiNGI5OTFmYWU2M2NjYmI4NTZjID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzA1Y2ViZjA1OWI2ODRkYWE5NGExMWIyZTQyMmRlMWM1ID0gJCgnPGRpdiBpZD0iaHRtbF8wNWNlYmYwNTliNjg0ZGFhOTRhMTFiMmU0MjJkZTFjNSIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTMwODIsIExBQkVMOjIsIFNFR01FTlQ6V1RDIC0gU2F2ZXJzPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8wOWE1MjhhNGZlYmU0YjRiOTkxZmFlNjNjY2JiODU2Yy5zZXRDb250ZW50KGh0bWxfMDVjZWJmMDU5YjY4NGRhYTk0YTExYjJlNDIyZGUxYzUpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl81MzZjOTkxOTkyOTA0MjFhYjA5NDI1NjEzMmRmZGZmNC5iaW5kUG9wdXAocG9wdXBfMDlhNTI4YTRmZWJlNGI0Yjk5MWZhZTYzY2NiYjg1NmMpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfNDNhYmQxMDQxZDViNDYzY2E1YmIzYmFjNzljNWRiMGMgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi40NDc0MTQ4LDExNC4xNjc2NzgyXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzY5ODY3YzA0YWRhYjRlNzVhNjE2ZGEyOTRjNzc2OTE4ID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfNDNhYmQxMDQxZDViNDYzY2E1YmIzYmFjNzljNWRiMGMuc2V0SWNvbihpY29uXzY5ODY3YzA0YWRhYjRlNzVhNjE2ZGEyOTRjNzc2OTE4KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2JmNTQ5MTAyODBmZDQ4Y2M4ZjkwNTFmYWEyNDRkMmZlID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzBiMTI4YWU2NDgyMTRkOTJhNmJlYmJkMGVhNGNiYTI2ID0gJCgnPGRpdiBpZD0iaHRtbF8wYjEyOGFlNjQ4MjE0ZDkyYTZiZWJiZDBlYTRjYmEyNiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTQwMjEsIExBQkVMOjAsIFNFR01FTlQ6V1RDIC0gU2F2ZXJzPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9iZjU0OTEwMjgwZmQ0OGNjOGY5MDUxZmFhMjQ0ZDJmZS5zZXRDb250ZW50KGh0bWxfMGIxMjhhZTY0ODIxNGQ5MmE2YmViYmQwZWE0Y2JhMjYpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl80M2FiZDEwNDFkNWI0NjNjYTViYjNiYWM3OWM1ZGIwYy5iaW5kUG9wdXAocG9wdXBfYmY1NDkxMDI4MGZkNDhjYzhmOTA1MWZhYTI0NGQyZmUpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfOTc3YTEwNzQxZTAwNGUxMTgyNjJhZjM4YmNhMDk2NGQgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4yNzg5MTk0LDExNC4xODk1NTk2XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2MzMmFkNWQwY2JhNzRiYTk5NTM5Mjk4Y2M3YjZmOGZhID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfOTc3YTEwNzQxZTAwNGUxMTgyNjJhZjM4YmNhMDk2NGQuc2V0SWNvbihpY29uX2MzMmFkNWQwY2JhNzRiYTk5NTM5Mjk4Y2M3YjZmOGZhKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2NjZDYwMmZiMmY2YzQxMjNiZmRiZDI3YjNmZDgwMDYzID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzBlMjA0NDE5MGNiODQ2ZTE4MzlkYzc2OWY3M2MyODA3ID0gJCgnPGRpdiBpZD0iaHRtbF8wZTIwNDQxOTBjYjg0NmUxODM5ZGM3NjlmNzNjMjgwNyIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTE2NzEsIExBQkVMOjAsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2NjZDYwMmZiMmY2YzQxMjNiZmRiZDI3YjNmZDgwMDYzLnNldENvbnRlbnQoaHRtbF8wZTIwNDQxOTBjYjg0NmUxODM5ZGM3NjlmNzNjMjgwNyk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzk3N2ExMDc0MWUwMDRlMTE4MjYyYWYzOGJjYTA5NjRkLmJpbmRQb3B1cChwb3B1cF9jY2Q2MDJmYjJmNmM0MTIzYmZkYmQyN2IzZmQ4MDA2Myk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl83NjJiNDVhMWNjZDU0ZDg5OTIzMzg4NmM2MjdmOGVkOSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjQ0ODY3MTksMTE0LjE2NjU3OTRdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fYzA5YTA0NDgzZDk1NDc5OTkyN2VhMGFkYzEyNDE1MzMgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3JlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl83NjJiNDVhMWNjZDU0ZDg5OTIzMzg4NmM2MjdmOGVkOS5zZXRJY29uKGljb25fYzA5YTA0NDgzZDk1NDc5OTkyN2VhMGFkYzEyNDE1MzMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfOWU2Y2FkZGE3ZWJkNDBiOThiZjFjM2E5MzIwZjM1NTQgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfOTBlNzRiMTRiMWZmNDRhMWJiMjg0MjU1YmIyODRjMDAgPSAkKCc8ZGl2IGlkPSJodG1sXzkwZTc0YjE0YjFmZjQ0YTFiYjI4NDI1NWJiMjg0YzAwIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMTk2MCwgTEFCRUw6MCwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfOWU2Y2FkZGE3ZWJkNDBiOThiZjFjM2E5MzIwZjM1NTQuc2V0Q29udGVudChodG1sXzkwZTc0YjE0YjFmZjQ0YTFiYjI4NDI1NWJiMjg0YzAwKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfNzYyYjQ1YTFjY2Q1NGQ4OTkyMzM4ODZjNjI3ZjhlZDkuYmluZFBvcHVwKHBvcHVwXzllNmNhZGRhN2ViZDQwYjk4YmYxYzNhOTMyMGYzNTU0KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2QxOTNhNDU4N2Q1NjRmZjA5MWMyZTBkZTFlZWUxM2FjID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzMyODA4OCwxMTQuMTU2NzU0N10sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9hZWJiZDllZjEzZDA0Y2QwODcwM2M2MzMzZjdlMGIzYiA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAncmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyX2QxOTNhNDU4N2Q1NjRmZjA5MWMyZTBkZTFlZWUxM2FjLnNldEljb24oaWNvbl9hZWJiZDllZjEzZDA0Y2QwODcwM2M2MzMzZjdlMGIzYik7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9hZjI2N2Y0OWRkY2M0NDI3OWQyMmM4MjBhNzg3Yjk2OSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9mYTg4YjFjNTU1NGM0ZjZmYmIxOTAxZDZjZTQyY2JmNiA9ICQoJzxkaXYgaWQ9Imh0bWxfZmE4OGIxYzU1NTRjNGY2ZmJiMTkwMWQ2Y2U0MmNiZjYiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMjc3LCBMQUJFTDowLCBTRUdNRU5UOldUQyAtIFNhdmVyczwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfYWYyNjdmNDlkZGNjNDQyNzlkMjJjODIwYTc4N2I5Njkuc2V0Q29udGVudChodG1sX2ZhODhiMWM1NTU0YzRmNmZiYjE5MDFkNmNlNDJjYmY2KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfZDE5M2E0NTg3ZDU2NGZmMDkxYzJlMGRlMWVlZTEzYWMuYmluZFBvcHVwKHBvcHVwX2FmMjY3ZjQ5ZGRjYzQ0Mjc5ZDIyYzgyMGE3ODdiOTY5KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2E3MzI3YTJmYzdhYzRjMjk5ZmNlMTZlMGYxODE2ZDlkID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzM4NDQ5MSwxMTQuMTg3MDU4XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2UwYzA0ZDY3ODc1MjQzZjViY2IyODg1NDIxZTQ1OGUwID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfYTczMjdhMmZjN2FjNGMyOTlmY2UxNmUwZjE4MTZkOWQuc2V0SWNvbihpY29uX2UwYzA0ZDY3ODc1MjQzZjViY2IyODg1NDIxZTQ1OGUwKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzIyZDBjMTYzYzIzNTQyMjU4NGEzYmU1Njk3YTQ1Y2I0ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzU1Zjg4YzAwNzdkYzQ5Yzc4NjZmYTZiMDcyZmRkZDFlID0gJCgnPGRpdiBpZD0iaHRtbF81NWY4OGMwMDc3ZGM0OWM3ODY2ZmE2YjA3MmZkZGQxZSIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTI1MDcsIExBQkVMOjAsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzIyZDBjMTYzYzIzNTQyMjU4NGEzYmU1Njk3YTQ1Y2I0LnNldENvbnRlbnQoaHRtbF81NWY4OGMwMDc3ZGM0OWM3ODY2ZmE2YjA3MmZkZGQxZSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyX2E3MzI3YTJmYzdhYzRjMjk5ZmNlMTZlMGYxODE2ZDlkLmJpbmRQb3B1cChwb3B1cF8yMmQwYzE2M2MyMzU0MjI1ODRhM2JlNTY5N2E0NWNiNCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl8wZjM1MWYzMTQwYmM0NTA0YTQ5OWI4MmQxMTlkYmEwYSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjQ2ODM0NTEsMTEzLjk5ODY2OTRdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fNTRmODk0ZGMxZWJkNDM2ZTk1M2E1MTE1ZmI0ZDRhZTkgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2JlaWdlJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzBmMzUxZjMxNDBiYzQ1MDRhNDk5YjgyZDExOWRiYTBhLnNldEljb24oaWNvbl81NGY4OTRkYzFlYmQ0MzZlOTUzYTUxMTVmYjRkNGFlOSk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF84Yzk3YzI2N2E4ZGI0YjJlODQ3NTY2NGY4MTFkNGY0YSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF8xZmNhYjMwZTMxYjU0MDg0YWViMTYyNGU3MTkxZmY5ZiA9ICQoJzxkaXYgaWQ9Imh0bWxfMWZjYWIzMGUzMWI1NDA4NGFlYjE2MjRlNzE5MWZmOWYiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMDYxLCBMQUJFTDo3LCBTRUdNRU5UOldUQyAtIFJlc2lkZW50aWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF84Yzk3YzI2N2E4ZGI0YjJlODQ3NTY2NGY4MTFkNGY0YS5zZXRDb250ZW50KGh0bWxfMWZjYWIzMGUzMWI1NDA4NGFlYjE2MjRlNzE5MWZmOWYpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl8wZjM1MWYzMTQwYmM0NTA0YTQ5OWI4MmQxMTlkYmEwYS5iaW5kUG9wdXAocG9wdXBfOGM5N2MyNjdhOGRiNGIyZTg0NzU2NjRmODExZDRmNGEpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfYTAyNDM3OTI1Y2I5NDhiZjk2NjljN2YxNjBlNGQzMDAgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4zMzA5ODU3LDExNC4xODc5NjI0XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2M1ODJlZjJhZmQ2YjQ3ZTNiYzFmZmNlYjUyYzRkYWNmID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdsaWdodHJlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl9hMDI0Mzc5MjVjYjk0OGJmOTY2OWM3ZjE2MGU0ZDMwMC5zZXRJY29uKGljb25fYzU4MmVmMmFmZDZiNDdlM2JjMWZmY2ViNTJjNGRhY2YpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfMjZhMTMwZjBmMzA1NGU4Nzk5M2MwMjFkYTdmN2Q2NmMgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNTA3MTExM2NhODgyNGU3ZGFmMWM5ZjNjOTVhOWRjNjcgPSAkKCc8ZGl2IGlkPSJodG1sXzUwNzExMTNjYTg4MjRlN2RhZjFjOWYzYzk1YTlkYzY3IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjM3NCwgTEFCRUw6NiwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMjZhMTMwZjBmMzA1NGU4Nzk5M2MwMjFkYTdmN2Q2NmMuc2V0Q29udGVudChodG1sXzUwNzExMTNjYTg4MjRlN2RhZjFjOWYzYzk1YTlkYzY3KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfYTAyNDM3OTI1Y2I5NDhiZjk2NjljN2YxNjBlNGQzMDAuYmluZFBvcHVwKHBvcHVwXzI2YTEzMGYwZjMwNTRlODc5OTNjMDIxZGE3ZjdkNjZjKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzAxMGE3YTJhMTBkYTRiZGJiN2M1ZDI1NjVlOWEwYTAwID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMjc0NjMyNiwxMTQuMTcxNzU4Nl0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9iNGY5OTZjNjljMjg0N2Q1ODhiNDE2YWI5NWJmMjc5MyA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAncmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzAxMGE3YTJhMTBkYTRiZGJiN2M1ZDI1NjVlOWEwYTAwLnNldEljb24oaWNvbl9iNGY5OTZjNjljMjg0N2Q1ODhiNDE2YWI5NWJmMjc5Myk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF85NjFjY2RjMDc4MmE0OTVlOTJiNWY3MzA3Y2IzNTFkYyA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9jZjNhNTQzNzg1Y2I0ZDkyYTBmNzUxNzQ1OTA2ZTFhNyA9ICQoJzxkaXYgaWQ9Imh0bWxfY2YzYTU0Mzc4NWNiNGQ5MmEwZjc1MTc0NTkwNmUxYTciIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMzkzLCBMQUJFTDowLCBTRUdNRU5UOldUQyAtIENvbW1lcmNpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzk2MWNjZGMwNzgyYTQ5NWU5MmI1ZjczMDdjYjM1MWRjLnNldENvbnRlbnQoaHRtbF9jZjNhNTQzNzg1Y2I0ZDkyYTBmNzUxNzQ1OTA2ZTFhNyk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzAxMGE3YTJhMTBkYTRiZGJiN2M1ZDI1NjVlOWEwYTAwLmJpbmRQb3B1cChwb3B1cF85NjFjY2RjMDc4MmE0OTVlOTJiNWY3MzA3Y2IzNTFkYyk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl83MGE5N2JhOTRlZGY0ZTYwYWE1ZWE2MDEzZmZhZTE0MCA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjQzMDAyNiwxMTQuMjQzMjIxXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzEwMGExYjVhYmUxNjQwYjQ5ZjI3M2E0NDRlNzcwNDZkID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfNzBhOTdiYTk0ZWRmNGU2MGFhNWVhNjAxM2ZmYWUxNDAuc2V0SWNvbihpY29uXzEwMGExYjVhYmUxNjQwYjQ5ZjI3M2E0NDRlNzcwNDZkKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzkzMDViOGMyMDFiNzQ4MGViMmZhZWU0NzBlY2VlMTAyID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2FmNWY0NWFkZDkwYjRmMDNiN2VkMDNjYzQwMTFkNjI0ID0gJCgnPGRpdiBpZD0iaHRtbF9hZjVmNDVhZGQ5MGI0ZjAzYjdlZDAzY2M0MDExZDYyNCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTMxNDMsIExBQkVMOjAsIFNFR01FTlQ6V1RDIC0gU2F2ZXJzPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF85MzA1YjhjMjAxYjc0ODBlYjJmYWVlNDcwZWNlZTEwMi5zZXRDb250ZW50KGh0bWxfYWY1ZjQ1YWRkOTBiNGYwM2I3ZWQwM2NjNDAxMWQ2MjQpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl83MGE5N2JhOTRlZGY0ZTYwYWE1ZWE2MDEzZmZhZTE0MC5iaW5kUG9wdXAocG9wdXBfOTMwNWI4YzIwMWI3NDgwZWIyZmFlZTQ3MGVjZWUxMDIpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfZDczYTk5N2ViODgyNGE4ZTk1NWNlMjE2YjExNjIzMWUgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4yNTgwMTgzLDExNC4xMzg5Njg1XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzY0YzgzZjhjMGU4MTQyYTJiZDA1M2QxYTc1MGU0Yzg0ID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdncmVlbicsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl9kNzNhOTk3ZWI4ODI0YThlOTU1Y2UyMTZiMTE2MjMxZS5zZXRJY29uKGljb25fNjRjODNmOGMwZTgxNDJhMmJkMDUzZDFhNzUwZTRjODQpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfODM0ZDIwNzdlZDRiNDQ2ZTgwMTI0YjQyMjMyMDE4N2YgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfZDI1ODkzYTZjMGQ2NGFlOGFhYTAxYThjYWU0NWFkMTggPSAkKCc8ZGl2IGlkPSJodG1sX2QyNTg5M2E2YzBkNjRhZThhYWEwMWE4Y2FlNDVhZDE4IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjA3NywgTEFCRUw6MiwgU0VHTUVOVDpXVEMgLSBTYXZlcnM8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzgzNGQyMDc3ZWQ0YjQ0NmU4MDEyNGI0MjIzMjAxODdmLnNldENvbnRlbnQoaHRtbF9kMjU4OTNhNmMwZDY0YWU4YWFhMDFhOGNhZTQ1YWQxOCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyX2Q3M2E5OTdlYjg4MjRhOGU5NTVjZTIxNmIxMTYyMzFlLmJpbmRQb3B1cChwb3B1cF84MzRkMjA3N2VkNGI0NDZlODAxMjRiNDIyMzIwMTg3Zik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9jZThmOWE4ZTEwZjQ0ZDgxOWU4MGVjZjA0YjNhYTAxNiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjI4ODA1MiwxMTQuMTUwOTcyXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2FiNGUwMWUzOTIzNTRmNTBiNGEwOGRmNDA0YTA0NzRhID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdsaWdodHJlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl9jZThmOWE4ZTEwZjQ0ZDgxOWU4MGVjZjA0YjNhYTAxNi5zZXRJY29uKGljb25fYWI0ZTAxZTM5MjM1NGY1MGI0YTA4ZGY0MDRhMDQ3NGEpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfZGFjOGJhY2Q3MGQ1NDI0Yjk1YTM1YWNhYTczODBjOTMgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfYmU5MTQ3MGE2NjY4NDU4MmEwZTQ2MWM4NzU1ODMxZGIgPSAkKCc8ZGl2IGlkPSJodG1sX2JlOTE0NzBhNjY2ODQ1ODJhMGU0NjFjODc1NTgzMWRiIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjI1MSwgTEFCRUw6NiwgU0VHTUVOVDpXVEMgLSBUb3VyaXN0cy9Db252PC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9kYWM4YmFjZDcwZDU0MjRiOTVhMzVhY2FhNzM4MGM5My5zZXRDb250ZW50KGh0bWxfYmU5MTQ3MGE2NjY4NDU4MmEwZTQ2MWM4NzU1ODMxZGIpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl9jZThmOWE4ZTEwZjQ0ZDgxOWU4MGVjZjA0YjNhYTAxNi5iaW5kUG9wdXAocG9wdXBfZGFjOGJhY2Q3MGQ1NDI0Yjk1YTM1YWNhYTczODBjOTMpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfOGEyODNjMTg4NmY4NGYwYjkzMDZmMTgzNzU5OWM3ZjAgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4zMDQ3MTEsMTE0LjE5MDI1MDRdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fZDVhMjI3NTk4NzdkNDMzZTgzZWFhMGRlZGQ3MmJmNDEgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2xpZ2h0cmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzhhMjgzYzE4ODZmODRmMGI5MzA2ZjE4Mzc1OTljN2YwLnNldEljb24oaWNvbl9kNWEyMjc1OTg3N2Q0MzNlODNlYWEwZGVkZDcyYmY0MSk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9jZjg1NDkyYmY0N2U0NzU2YTZhYWRhMmE0ZGEwNTMxNyA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF85MDc0ZDEzNjk5MjU0YTQ3OWY1M2Y5OWQ1ZGYyMGFmNyA9ICQoJzxkaXYgaWQ9Imh0bWxfOTA3NGQxMzY5OTI1NGE0NzlmNTNmOTlkNWRmMjBhZjciIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMzk2LCBMQUJFTDo2LCBTRUdNRU5UOldUQyAtIERlc3RpbmF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9jZjg1NDkyYmY0N2U0NzU2YTZhYWRhMmE0ZGEwNTMxNy5zZXRDb250ZW50KGh0bWxfOTA3NGQxMzY5OTI1NGE0NzlmNTNmOTlkNWRmMjBhZjcpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl84YTI4M2MxODg2Zjg0ZjBiOTMwNmYxODM3NTk5YzdmMC5iaW5kUG9wdXAocG9wdXBfY2Y4NTQ5MmJmNDdlNDc1NmE2YWFkYTJhNGRhMDUzMTcpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfNGNiNWUzYzVhYTg5NDQ0NWJiZGFlYTVhZWViZTM2ZTQgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi40NDU3NTg3LDExNC4wMzUyMjQ4XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2FkZDg1MjI5ODJmMDQ0MDViMjJiZDVkYTZjNjY3YWQzID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdkYXJrYmx1ZScsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl80Y2I1ZTNjNWFhODk0NDQ1YmJkYWVhNWFlZWJlMzZlNC5zZXRJY29uKGljb25fYWRkODUyMjk4MmYwNDQwNWIyMmJkNWRhNmM2NjdhZDMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfOWE0ZGFhZGEwYjgyNGE2YTk3YjkyNWRjMDA0ODRjYzQgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfOTJjMGJiOTZhOWIyNGIyZDk3ZTg4NDZhZDdiYzVlOWEgPSAkKCc8ZGl2IGlkPSJodG1sXzkyYzBiYjk2YTliMjRiMmQ5N2U4ODQ2YWQ3YmM1ZTlhIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMzg2MSwgTEFCRUw6OCwgU0VHTUVOVDpXVEMgLSBUb3VyaXN0cy9Db252PC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF85YTRkYWFkYTBiODI0YTZhOTdiOTI1ZGMwMDQ4NGNjNC5zZXRDb250ZW50KGh0bWxfOTJjMGJiOTZhOWIyNGIyZDk3ZTg4NDZhZDdiYzVlOWEpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl80Y2I1ZTNjNWFhODk0NDQ1YmJkYWVhNWFlZWJlMzZlNC5iaW5kUG9wdXAocG9wdXBfOWE0ZGFhZGEwYjgyNGE2YTk3YjkyNWRjMDA0ODRjYzQpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfZTU3ZjU0ODNlM2FlNGIyZGFkYTIzZWJjZDQ4ZTQwOWIgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4yODI3MjA2LDExNC4yMTIzMDA5XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2QwZWZkMzlhZmYxNjQxYWNiNDkxMjYyYzJhMzljMmUwID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdncmVlbicsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl9lNTdmNTQ4M2UzYWU0YjJkYWRhMjNlYmNkNDhlNDA5Yi5zZXRJY29uKGljb25fZDBlZmQzOWFmZjE2NDFhY2I0OTEyNjJjMmEzOWMyZTApOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfOTkxMmFjODk1Mjk3NGQ4ZjkwZDU4Mjc3NmNlMmYwNTAgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNTA0YzAxOWNiNTA0NDBmZWE4MmM2ZmE5M2NiYTg1YzkgPSAkKCc8ZGl2IGlkPSJodG1sXzUwNGMwMTljYjUwNDQwZmVhODJjNmZhOTNjYmE4NWM5IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMTk0OSwgTEFCRUw6MiwgU0VHTUVOVDpXVEMgLSBTYXZlcnM8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzk5MTJhYzg5NTI5NzRkOGY5MGQ1ODI3NzZjZTJmMDUwLnNldENvbnRlbnQoaHRtbF81MDRjMDE5Y2I1MDQ0MGZlYTgyYzZmYTkzY2JhODVjOSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyX2U1N2Y1NDgzZTNhZTRiMmRhZGEyM2ViY2Q0OGU0MDliLmJpbmRQb3B1cChwb3B1cF85OTEyYWM4OTUyOTc0ZDhmOTBkNTgyNzc2Y2UyZjA1MCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9kMGVjMTEzMjM0NmI0MDRjYTVkODJmYmZkNDc4Zjg1NSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjQ2NzI4NDYsMTEzLjk5NzU3OTNdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fOTk5MGM0Njg4NTIyNGMyYjk2MjA0NjE3Y2VlYTczYTYgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2JlaWdlJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyX2QwZWMxMTMyMzQ2YjQwNGNhNWQ4MmZiZmQ0NzhmODU1LnNldEljb24oaWNvbl85OTkwYzQ2ODg1MjI0YzJiOTYyMDQ2MTdjZWVhNzNhNik7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8zYTliYzQwZjVlYTE0Y2Q5OWE0MGIxMDk3MTI5ZmY5MCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF85MzIzOTNlMGJiZTY0YWFhYTE4NTU5YTk2YmMzNTQyNyA9ICQoJzxkaXYgaWQ9Imh0bWxfOTMyMzkzZTBiYmU2NGFhYWExODU1OWE5NmJjMzU0MjciIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMDYzLCBMQUJFTDo3LCBTRUdNRU5UOldUQyAtIFJlc2lkZW50aWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8zYTliYzQwZjVlYTE0Y2Q5OWE0MGIxMDk3MTI5ZmY5MC5zZXRDb250ZW50KGh0bWxfOTMyMzkzZTBiYmU2NGFhYWExODU1OWE5NmJjMzU0MjcpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl9kMGVjMTEzMjM0NmI0MDRjYTVkODJmYmZkNDc4Zjg1NS5iaW5kUG9wdXAocG9wdXBfM2E5YmM0MGY1ZWExNGNkOTlhNDBiMTA5NzEyOWZmOTApOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfMTlkY2MwYjFjNzBmNDgyNGIxOGQ0NDhlMjJiYTg4ZjIgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4zNzAzNTkxLDExNC4xMzg3MzMyXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzc4MTQyMWEyYTFkNzRjMzFiODY2NzYyNDYyODM2MTQ1ID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfMTlkY2MwYjFjNzBmNDgyNGIxOGQ0NDhlMjJiYTg4ZjIuc2V0SWNvbihpY29uXzc4MTQyMWEyYTFkNzRjMzFiODY2NzYyNDYyODM2MTQ1KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzNmM2RhZmE2YmQ0MjQ1MTY4YzYxMzJmODUyNjY3OWE4ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzJkN2NmYTcyYjA2MDQxMjk5MWZjZTJmYjhjYjA0OTE2ID0gJCgnPGRpdiBpZD0iaHRtbF8yZDdjZmE3MmIwNjA0MTI5OTFmY2UyZmI4Y2IwNDkxNiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIwOTAsIExBQkVMOjAsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzNmM2RhZmE2YmQ0MjQ1MTY4YzYxMzJmODUyNjY3OWE4LnNldENvbnRlbnQoaHRtbF8yZDdjZmE3MmIwNjA0MTI5OTFmY2UyZmI4Y2IwNDkxNik7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzE5ZGNjMGIxYzcwZjQ4MjRiMThkNDQ4ZTIyYmE4OGYyLmJpbmRQb3B1cChwb3B1cF8zZjNkYWZhNmJkNDI0NTE2OGM2MTMyZjg1MjY2NzlhOCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl85OGIwMGVhYmE0MTY0YmU5YmU4ZTZjNjcxYjI1MjkxNSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjMxOTc1LDExNC4xNjMyNTcxXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2M0NDU3ODZjNGRmYTRjZGU4ZDdkNzgwYzBkYzkyZTNlID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfOThiMDBlYWJhNDE2NGJlOWJlOGU2YzY3MWIyNTI5MTUuc2V0SWNvbihpY29uX2M0NDU3ODZjNGRmYTRjZGU4ZDdkNzgwYzBkYzkyZTNlKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzA1MTQwMDVhMzVlNzRmOWY5ZWZkNmUwMTFiZDRiYTA2ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2I2NGEwM2UzM2U5YTQxY2VhYzQ5NWNkODk4NTkzZWQzID0gJCgnPGRpdiBpZD0iaHRtbF9iNjRhMDNlMzNlOWE0MWNlYWM0OTVjZDg5ODU5M2VkMyIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIyOTIsIExBQkVMOjAsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzA1MTQwMDVhMzVlNzRmOWY5ZWZkNmUwMTFiZDRiYTA2LnNldENvbnRlbnQoaHRtbF9iNjRhMDNlMzNlOWE0MWNlYWM0OTVjZDg5ODU5M2VkMyk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzk4YjAwZWFiYTQxNjRiZTliZThlNmM2NzFiMjUyOTE1LmJpbmRQb3B1cChwb3B1cF8wNTE0MDA1YTM1ZTc0ZjlmOWVmZDZlMDExYmQ0YmEwNik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl84M2U2OGExYWNkMjY0MGNkYTNlMmUzMjUwNWJiMTEyMiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjMxNzQ3MzUsMTE0LjE2NDc1NzJdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fNWNhOGUwODZkZGIzNDc0MjhmZGNhOTMwYjg2ODk1OTcgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2dyZWVuJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzgzZTY4YTFhY2QyNjQwY2RhM2UyZTMyNTA1YmIxMTIyLnNldEljb24oaWNvbl81Y2E4ZTA4NmRkYjM0NzQyOGZkY2E5MzBiODY4OTU5Nyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF82ZjU2ZDY4MmI4MmM0ODc2ODNmYjY5OGMwMzU0OTFiZCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF83OGIxMDNmZWJhNjY0ZDE5OGE5NzM2OTYzZTdlNjBkMiA9ICQoJzxkaXYgaWQ9Imh0bWxfNzhiMTAzZmViYTY2NGQxOThhOTczNjk2M2U3ZTYwZDIiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMTMxLCBMQUJFTDoyLCBTRUdNRU5UOldUQyAtIFNhdmVyczwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNmY1NmQ2ODJiODJjNDg3NjgzZmI2OThjMDM1NDkxYmQuc2V0Q29udGVudChodG1sXzc4YjEwM2ZlYmE2NjRkMTk4YTk3MzY5NjNlN2U2MGQyKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfODNlNjhhMWFjZDI2NDBjZGEzZTJlMzI1MDViYjExMjIuYmluZFBvcHVwKHBvcHVwXzZmNTZkNjgyYjgyYzQ4NzY4M2ZiNjk4YzAzNTQ5MWJkKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2VjMDU0OTJkZmEyYTRjNjhhOTY1MzYxNjgyYzYxY2MxID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzcyNzA3MSwxMTQuMTc4NjE2N10sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9kZWMyNzEzNGU1MWY0MzU0YjVlNWZkYjY1MTQ2MTc5ZSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnYmVpZ2UnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfZWMwNTQ5MmRmYTJhNGM2OGE5NjUzNjE2ODJjNjFjYzEuc2V0SWNvbihpY29uX2RlYzI3MTM0ZTUxZjQzNTRiNWU1ZmRiNjUxNDYxNzllKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2Q4MWMxOGRkMjZhMjRhYzc5ODU2NmNhMTg5Y2ZjNDEyID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzgxY2UzMTA0NGNkYjQ4ZmJhYTQ1YTc3ZjAwODQ3MWMxID0gJCgnPGRpdiBpZD0iaHRtbF84MWNlMzEwNDRjZGI0OGZiYWE0NWE3N2YwMDg0NzFjMSIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIzMDgsIExBQkVMOjcsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2Q4MWMxOGRkMjZhMjRhYzc5ODU2NmNhMTg5Y2ZjNDEyLnNldENvbnRlbnQoaHRtbF84MWNlMzEwNDRjZGI0OGZiYWE0NWE3N2YwMDg0NzFjMSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyX2VjMDU0OTJkZmEyYTRjNjhhOTY1MzYxNjgyYzYxY2MxLmJpbmRQb3B1cChwb3B1cF9kODFjMThkZDI2YTI0YWM3OTg1NjZjYTE4OWNmYzQxMik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl8wMTcxOWU2ZjFiZWU0NDY2OGRmNTkxMTczZjk5NDUyNiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjE1ODU0OCwxMTMuNTUzNjg0XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzYzMTYwNTIwM2Y1MjQxMzRiMzk3NzhhNGFhZjg0MWRlID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdsaWdodHJlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl8wMTcxOWU2ZjFiZWU0NDY2OGRmNTkxMTczZjk5NDUyNi5zZXRJY29uKGljb25fNjMxNjA1MjAzZjUyNDEzNGIzOTc3OGE0YWFmODQxZGUpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfYmNkZTZjYzQzODdjNDU3NTkzYmM0MTY1YTRkZDQwODEgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfY2NjZWUwNTZkYjlmNGNhMWI2NmEwZDQ4Njc4ZTZjZjkgPSAkKCc8ZGl2IGlkPSJodG1sX2NjY2VlMDU2ZGI5ZjRjYTFiNjZhMGQ0ODY3OGU2Y2Y5IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjQyOSwgTEFCRUw6NiwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfYmNkZTZjYzQzODdjNDU3NTkzYmM0MTY1YTRkZDQwODEuc2V0Q29udGVudChodG1sX2NjY2VlMDU2ZGI5ZjRjYTFiNjZhMGQ0ODY3OGU2Y2Y5KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfMDE3MTllNmYxYmVlNDQ2NjhkZjU5MTE3M2Y5OTQ1MjYuYmluZFBvcHVwKHBvcHVwX2JjZGU2Y2M0Mzg3YzQ1NzU5M2JjNDE2NWE0ZGQ0MDgxKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2E3NDNkNTIzMTAzMjRjMjBiMWY5Yjk2YzczNzk4NjljID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuNDQ0NTM3NiwxMTQuMDIyMjA3Nl0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl81NDIxN2E5ZWQ0NWU0NzdlYTVlZGQ1MzQ1YmU5NTE2MiA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnb3JhbmdlJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyX2E3NDNkNTIzMTAzMjRjMjBiMWY5Yjk2YzczNzk4NjljLnNldEljb24oaWNvbl81NDIxN2E5ZWQ0NWU0NzdlYTVlZGQ1MzQ1YmU5NTE2Mik7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8wNmViZDhhZTM3MjA0NmQwYjljN2NjYmFkN2QyNDc4OSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF82YTg0MTYyZTZjZjI0ODNiOWE1NzkzMmM5NmQ3NjFmOSA9ICQoJzxkaXYgaWQ9Imh0bWxfNmE4NDE2MmU2Y2YyNDgzYjlhNTc5MzJjOTZkNzYxZjkiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjExNjk4LCBMQUJFTDo0LCBTRUdNRU5UOldUQyAtIERlc3RpbmF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8wNmViZDhhZTM3MjA0NmQwYjljN2NjYmFkN2QyNDc4OS5zZXRDb250ZW50KGh0bWxfNmE4NDE2MmU2Y2YyNDgzYjlhNTc5MzJjOTZkNzYxZjkpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl9hNzQzZDUyMzEwMzI0YzIwYjFmOWI5NmM3Mzc5ODY5Yy5iaW5kUG9wdXAocG9wdXBfMDZlYmQ4YWUzNzIwNDZkMGI5YzdjY2JhZDdkMjQ3ODkpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfODlkYWM5NTNhNTQzNDNmMjhlNDRkYTcyOWYyYjBkN2IgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4zOTUzODc4LDExMy45NzM5MzMxXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2RmYzdkMjk1Y2MwODQxMWNhYjY3YmIyZjM2NDUwOTc4ID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdsaWdodHJlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl84OWRhYzk1M2E1NDM0M2YyOGU0NGRhNzI5ZjJiMGQ3Yi5zZXRJY29uKGljb25fZGZjN2QyOTVjYzA4NDExY2FiNjdiYjJmMzY0NTA5NzgpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfMDllYWNiMDcwZDUzNDJjYThkOTRkNWJjNWI3MTk3ZGIgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfMDY4MTk3YzgwNTczNDcyNzkzYTNjMmZhYThiOTgzYTUgPSAkKCc8ZGl2IGlkPSJodG1sXzA2ODE5N2M4MDU3MzQ3Mjc5M2EzYzJmYWE4Yjk4M2E1IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMTkwOCwgTEFCRUw6NiwgU0VHTUVOVDpXVEMgLSBUb3VyaXN0cy9Db252PC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8wOWVhY2IwNzBkNTM0MmNhOGQ5NGQ1YmM1YjcxOTdkYi5zZXRDb250ZW50KGh0bWxfMDY4MTk3YzgwNTczNDcyNzkzYTNjMmZhYThiOTgzYTUpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl84OWRhYzk1M2E1NDM0M2YyOGU0NGRhNzI5ZjJiMGQ3Yi5iaW5kUG9wdXAocG9wdXBfMDllYWNiMDcwZDUzNDJjYThkOTRkNWJjNWI3MTk3ZGIpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfNDU3NzEyMzJhNjY0NDI5Njk2YjQwNjcwZmQ2Mjk3YTUgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4zMzc3NDIsMTE0LjIwMDA3Ml0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9kNDQzMWU2MTM0NmU0ZDlkYTYzZWQ5YzcxYTg2ODZjMCA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnYmVpZ2UnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfNDU3NzEyMzJhNjY0NDI5Njk2YjQwNjcwZmQ2Mjk3YTUuc2V0SWNvbihpY29uX2Q0NDMxZTYxMzQ2ZTRkOWRhNjNlZDljNzFhODY4NmMwKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2U0ZGI4MzIwMGYxNDQ3M2NiYTJkNjhlYzk5N2U5ZTJhID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzJhZTQzODVlYTc4MTQ5MTBiZjBlODEyNmY3ZWIxZWExID0gJCgnPGRpdiBpZD0iaHRtbF8yYWU0Mzg1ZWE3ODE0OTEwYmYwZTgxMjZmN2ViMWVhMSIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTI2MTUsIExBQkVMOjcsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2U0ZGI4MzIwMGYxNDQ3M2NiYTJkNjhlYzk5N2U5ZTJhLnNldENvbnRlbnQoaHRtbF8yYWU0Mzg1ZWE3ODE0OTEwYmYwZTgxMjZmN2ViMWVhMSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzQ1NzcxMjMyYTY2NDQyOTY5NmI0MDY3MGZkNjI5N2E1LmJpbmRQb3B1cChwb3B1cF9lNGRiODMyMDBmMTQ0NzNjYmEyZDY4ZWM5OTdlOWUyYSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl81MTdlZGVlMGY3ODE0YzJjYjMyYzJlODVmZDhkNGQ5ZiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjMyMDM2NDgsMTE0LjE2OTc3M10sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl8yODA2Y2ZjMzE0N2M0YjAwOTE4MWVhNDZiNWVhYjc1ZiA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnb3JhbmdlJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzUxN2VkZWUwZjc4MTRjMmNiMzJjMmU4NWZkOGQ0ZDlmLnNldEljb24oaWNvbl8yODA2Y2ZjMzE0N2M0YjAwOTE4MWVhNDZiNWVhYjc1Zik7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF80OWFhNzc1NTNkOTQ0M2NlODJhNGNjMmQ0MjYyNDZkOCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF8zMWQyMzVmZDA5ZmM0OTI5OTgxOWM2NTI4YzQxOGQ2MCA9ICQoJzxkaXYgaWQ9Imh0bWxfMzFkMjM1ZmQwOWZjNDkyOTk4MTljNjUyOGM0MThkNjAiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjExNzQwLCBMQUJFTDo0LCBTRUdNRU5UOldUQyAtIERlc3RpbmF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF80OWFhNzc1NTNkOTQ0M2NlODJhNGNjMmQ0MjYyNDZkOC5zZXRDb250ZW50KGh0bWxfMzFkMjM1ZmQwOWZjNDkyOTk4MTljNjUyOGM0MThkNjApOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl81MTdlZGVlMGY3ODE0YzJjYjMyYzJlODVmZDhkNGQ5Zi5iaW5kUG9wdXAocG9wdXBfNDlhYTc3NTUzZDk0NDNjZTgyYTRjYzJkNDI2MjQ2ZDgpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfMzU5YWY2MzAyYjllNDI5Yjg5OWY4YzdiMGEyMmQ1YzEgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi40NDYwOTM1LDExNC4wMjU5MzU1XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzI1ZDgxZDI0NjIwNjQ2ODA5MWU0Mzk3NTJiMWUyNTBhID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdncmVlbicsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl8zNTlhZjYzMDJiOWU0MjliODk5ZjhjN2IwYTIyZDVjMS5zZXRJY29uKGljb25fMjVkODFkMjQ2MjA2NDY4MDkxZTQzOTc1MmIxZTI1MGEpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfZDI2NDQ5OTNhZThjNDE3YTg1N2QyNGJmM2RkMWU3MjkgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNDI2OTQ4NmE5ZWQ4NGNkNzhjOTg0ODEwMGYyODc1M2QgPSAkKCc8ZGl2IGlkPSJodG1sXzQyNjk0ODZhOWVkODRjZDc4Yzk4NDgxMDBmMjg3NTNkIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMTk1NywgTEFCRUw6MiwgU0VHTUVOVDpXVEMgLSBTYXZlcnM8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2QyNjQ0OTkzYWU4YzQxN2E4NTdkMjRiZjNkZDFlNzI5LnNldENvbnRlbnQoaHRtbF80MjY5NDg2YTllZDg0Y2Q3OGM5ODQ4MTAwZjI4NzUzZCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzM1OWFmNjMwMmI5ZTQyOWI4OTlmOGM3YjBhMjJkNWMxLmJpbmRQb3B1cChwb3B1cF9kMjY0NDk5M2FlOGM0MTdhODU3ZDI0YmYzZGQxZTcyOSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl8yODIxYmRkYTEyMzk0MDRhYWU4NTllOTU2MDU4MWYxYyA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjQ4Njk4MiwxMTQuMTQzNzYyNl0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9kZmUwMmRiNjJjNGQ0ZDYyOWIzYWM1MTdiOTM1ZTM4NyA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnYmVpZ2UnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfMjgyMWJkZGExMjM5NDA0YWFlODU5ZTk1NjA1ODFmMWMuc2V0SWNvbihpY29uX2RmZTAyZGI2MmM0ZDRkNjI5YjNhYzUxN2I5MzVlMzg3KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzAyOTcyZDg1M2I4ZjRhNmM4ZmEyY2IwMWVkYWFkM2IyID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzlmMjVmZTljY2QwNDQ3ZDk5ODQxZThmYjg1NTBhNzRjID0gJCgnPGRpdiBpZD0iaHRtbF85ZjI1ZmU5Y2NkMDQ0N2Q5OTg0MWU4ZmI4NTUwYTc0YyIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTE5NTksIExBQkVMOjcsIFNFR01FTlQ6V1RDIC0gU2F2ZXJzPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8wMjk3MmQ4NTNiOGY0YTZjOGZhMmNiMDFlZGFhZDNiMi5zZXRDb250ZW50KGh0bWxfOWYyNWZlOWNjZDA0NDdkOTk4NDFlOGZiODU1MGE3NGMpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl8yODIxYmRkYTEyMzk0MDRhYWU4NTllOTU2MDU4MWYxYy5iaW5kUG9wdXAocG9wdXBfMDI5NzJkODUzYjhmNGE2YzhmYTJjYjAxZWRhYWQzYjIpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfOGRjN2IwMGRmYmZkNGEwMDljNmFkZWI1ZTY2MDI3ZGMgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4zMzU3NzY1LDExNC4xNDA1Njk1XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzJiMTQzODE0YjU0ZTRjMTI5NmVhODc1MzYxN2QzN2M1ID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfOGRjN2IwMGRmYmZkNGEwMDljNmFkZWI1ZTY2MDI3ZGMuc2V0SWNvbihpY29uXzJiMTQzODE0YjU0ZTRjMTI5NmVhODc1MzYxN2QzN2M1KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2E1MTBhNWNjNDc0ZTQxNzdiMzM0ZjRlYzMxZDMyNDBjID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2FiMzFkYzVmM2IyZDQ2MGY4Y2Q5NDE5ZmNkNzc1MDI4ID0gJCgnPGRpdiBpZD0iaHRtbF9hYjMxZGM1ZjNiMmQ0NjBmOGNkOTQxOWZjZDc3NTAyOCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIzMDEsIExBQkVMOjAsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2E1MTBhNWNjNDc0ZTQxNzdiMzM0ZjRlYzMxZDMyNDBjLnNldENvbnRlbnQoaHRtbF9hYjMxZGM1ZjNiMmQ0NjBmOGNkOTQxOWZjZDc3NTAyOCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzhkYzdiMDBkZmJmZDRhMDA5YzZhZGViNWU2NjAyN2RjLmJpbmRQb3B1cChwb3B1cF9hNTEwYTVjYzQ3NGU0MTc3YjMzNGY0ZWMzMWQzMjQwYyk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9lNTRiMTYwNGVlNjk0YjgxYTc0NGM1NzkxMWYwZWJmNyA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjI3ODcyNCwxMTQuMTgxNDA4OF0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9iMDNmZmI4YWMyMmY0NmQ2YmY1ODg3YWEzNjgzZjY0ZCA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnZ3JlZW4nLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfZTU0YjE2MDRlZTY5NGI4MWE3NDRjNTc5MTFmMGViZjcuc2V0SWNvbihpY29uX2IwM2ZmYjhhYzIyZjQ2ZDZiZjU4ODdhYTM2ODNmNjRkKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2EzMDg1M2NhYjVjNDRjNjI4ZjBiMzUwYTkxZDVjYjk4ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzhjMmY3MjZlZmJiOTRlMDdhNGI1ZmU1NGFhZmQ4YmM0ID0gJCgnPGRpdiBpZD0iaHRtbF84YzJmNzI2ZWZiYjk0ZTA3YTRiNWZlNTRhYWZkOGJjNCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIzOTAsIExBQkVMOjIsIFNFR01FTlQ6V1RDIC0gVG91cmlzdHMvQ29udjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfYTMwODUzY2FiNWM0NGM2MjhmMGIzNTBhOTFkNWNiOTguc2V0Q29udGVudChodG1sXzhjMmY3MjZlZmJiOTRlMDdhNGI1ZmU1NGFhZmQ4YmM0KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfZTU0YjE2MDRlZTY5NGI4MWE3NDRjNTc5MTFmMGViZjcuYmluZFBvcHVwKHBvcHVwX2EzMDg1M2NhYjVjNDRjNjI4ZjBiMzUwYTkxZDVjYjk4KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzdlMzAxMmE0ODU2YjRkZGNhYWQ2ODE2YWRmMGZmODUwID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMjYyNTMwMywxMTQuMjM1NTY2OV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl8zMzBkMjA4ZTI3OTQ0ODE4YTE5NDFiZGU3NzJmMWViNCA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAncmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzdlMzAxMmE0ODU2YjRkZGNhYWQ2ODE2YWRmMGZmODUwLnNldEljb24oaWNvbl8zMzBkMjA4ZTI3OTQ0ODE4YTE5NDFiZGU3NzJmMWViNCk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF80YWMwNGRjZmNmMWQ0M2Q5YTljODA0YmVmZTM2MDk1MSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9lZmIwNWIyNGNkNzA0NzNjOGNjM2NkZWRiOTc0NTdiZCA9ICQoJzxkaXYgaWQ9Imh0bWxfZWZiMDViMjRjZDcwNDczYzhjYzNjZGVkYjk3NDU3YmQiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyNDI3LCBMQUJFTDowLCBTRUdNRU5UOldUQyAtIFJlc2lkZW50aWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF80YWMwNGRjZmNmMWQ0M2Q5YTljODA0YmVmZTM2MDk1MS5zZXRDb250ZW50KGh0bWxfZWZiMDViMjRjZDcwNDczYzhjYzNjZGVkYjk3NDU3YmQpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl83ZTMwMTJhNDg1NmI0ZGRjYWFkNjgxNmFkZjBmZjg1MC5iaW5kUG9wdXAocG9wdXBfNGFjMDRkY2ZjZjFkNDNkOWE5YzgwNGJlZmUzNjA5NTEpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfNmM4OTgxMmZkZTAwNGMwZjgxNWJhMTY1OTJhMTdkNGEgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi41MTQ0ODksMTE0LjA2NTY4NF0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9iYTk1NTRhMTdmZDg0YTY4ODM2OTNhZGE5OGQxYTNlZSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnYmVpZ2UnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfNmM4OTgxMmZkZTAwNGMwZjgxNWJhMTY1OTJhMTdkNGEuc2V0SWNvbihpY29uX2JhOTU1NGExN2ZkODRhNjg4MzY5M2FkYTk4ZDFhM2VlKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2E0YzAwZThiZTIzNDQwZTliNjRkODRjMWVmZmYwMmEwID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzgyMTFhMDk3ZGEzMzRlMzE5YWVmNjI1NThlYmQ0MjUyID0gJCgnPGRpdiBpZD0iaHRtbF84MjExYTA5N2RhMzM0ZTMxOWFlZjYyNTU4ZWJkNDI1MiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTMzNjEsIExBQkVMOjcsIFNFR01FTlQ6V1RDIC0gVG91cmlzdHMvQ29udjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfYTRjMDBlOGJlMjM0NDBlOWI2NGQ4NGMxZWZmZjAyYTAuc2V0Q29udGVudChodG1sXzgyMTFhMDk3ZGEzMzRlMzE5YWVmNjI1NThlYmQ0MjUyKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfNmM4OTgxMmZkZTAwNGMwZjgxNWJhMTY1OTJhMTdkNGEuYmluZFBvcHVwKHBvcHVwX2E0YzAwZThiZTIzNDQwZTliNjRkODRjMWVmZmYwMmEwKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzM1MDlhMjJkZjJiNDRmODQ4Y2ZjYjZhYjE0OWJmMWQyID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzcwMDg4LDExNC4xMTY5ODk5XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzE3MDg5MjljMTYwMzQ0OTg5MjVlMjRhMzk2NDE4OTZjID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdncmVlbicsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl8zNTA5YTIyZGYyYjQ0Zjg0OGNmY2I2YWIxNDliZjFkMi5zZXRJY29uKGljb25fMTcwODkyOWMxNjAzNDQ5ODkyNWUyNGEzOTY0MTg5NmMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfM2Y3N2M3MDg3YTA4NDkyMGI5OTc1ZjI0Y2VlMTYyZDcgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNzYyYjg3Y2RmNWViNGYyYzkzYmZkNjcyZDFhYTk2YTggPSAkKCc8ZGl2IGlkPSJodG1sXzc2MmI4N2NkZjVlYjRmMmM5M2JmZDY3MmQxYWE5NmE4IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMTcwOCwgTEFCRUw6MiwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfM2Y3N2M3MDg3YTA4NDkyMGI5OTc1ZjI0Y2VlMTYyZDcuc2V0Q29udGVudChodG1sXzc2MmI4N2NkZjVlYjRmMmM5M2JmZDY3MmQxYWE5NmE4KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfMzUwOWEyMmRmMmI0NGY4NDhjZmNiNmFiMTQ5YmYxZDIuYmluZFBvcHVwKHBvcHVwXzNmNzdjNzA4N2EwODQ5MjBiOTk3NWYyNGNlZTE2MmQ3KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2M4MzcxOGZiNTZjMTRjMmVhYjllNjEzNmQ2MjFmYTg2ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMjA2MTI3NywxMTMuNTUwNTY3XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzJmZjEwNTdlYmFjMzQ0NjM4MzE0MjE3M2U4ZGU3OTQwID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdsaWdodHJlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl9jODM3MThmYjU2YzE0YzJlYWI5ZTYxMzZkNjIxZmE4Ni5zZXRJY29uKGljb25fMmZmMTA1N2ViYWMzNDQ2MzgzMTQyMTczZThkZTc5NDApOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfYjkyMmU1NjE4NjhkNDVkMDlkN2Q3Mjg1ZTQ3YzFjMzggPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfOWFjMzE2ZGQxMzIwNDQzMzg0MTU3ZDM4MzIzMzc3MzcgPSAkKCc8ZGl2IGlkPSJodG1sXzlhYzMxNmRkMTMyMDQ0MzM4NDE1N2QzODMyMzM3NzM3IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMTk4NiwgTEFCRUw6NiwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfYjkyMmU1NjE4NjhkNDVkMDlkN2Q3Mjg1ZTQ3YzFjMzguc2V0Q29udGVudChodG1sXzlhYzMxNmRkMTMyMDQ0MzM4NDE1N2QzODMyMzM3NzM3KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfYzgzNzE4ZmI1NmMxNGMyZWFiOWU2MTM2ZDYyMWZhODYuYmluZFBvcHVwKHBvcHVwX2I5MjJlNTYxODY4ZDQ1ZDA5ZDdkNzI4NWU0N2MxYzM4KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzI3MTczYzVjOWVjMTRmMmZhNjI5ZTNlYTBmZGU5ZGQ4ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzE5NjI4MywxMTQuMTY5OTQ3OV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9mZDA2NzZkZWIyNWI0MTc0YjAxMTM2MmVhMGJiMTJlMSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnb3JhbmdlJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzI3MTczYzVjOWVjMTRmMmZhNjI5ZTNlYTBmZGU5ZGQ4LnNldEljb24oaWNvbl9mZDA2NzZkZWIyNWI0MTc0YjAxMTM2MmVhMGJiMTJlMSk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9jNjZkZDM3YTFhNTI0MzQ0OWFkNTNjZjMxOGViMzlkMSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9iYWQ2MmU2YWJkMDM0ZmVmODc4Y2QyOWYxZGIxZWJkMCA9ICQoJzxkaXYgaWQ9Imh0bWxfYmFkNjJlNmFiZDAzNGZlZjg3OGNkMjlmMWRiMWViZDAiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMjM4LCBMQUJFTDo0LCBTRUdNRU5UOldUQyAtIFRvdXJpc3RzL0NvbnY8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2M2NmRkMzdhMWE1MjQzNDQ5YWQ1M2NmMzE4ZWIzOWQxLnNldENvbnRlbnQoaHRtbF9iYWQ2MmU2YWJkMDM0ZmVmODc4Y2QyOWYxZGIxZWJkMCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzI3MTczYzVjOWVjMTRmMmZhNjI5ZTNlYTBmZGU5ZGQ4LmJpbmRQb3B1cChwb3B1cF9jNjZkZDM3YTFhNTI0MzQ0OWFkNTNjZjMxOGViMzlkMSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl8wZjNiNTgwZDViNjk0ZTU4OWMwNDUyMmEwZTAwM2M1ZCA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjM5MzExNiwxMTMuOTc1NzldLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fZmY2NmNmOWVkYjU5NDlhYTlmN2EzOTdiMzFkYzcyYTIgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2xpZ2h0cmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzBmM2I1ODBkNWI2OTRlNTg5YzA0NTIyYTBlMDAzYzVkLnNldEljb24oaWNvbl9mZjY2Y2Y5ZWRiNTk0OWFhOWY3YTM5N2IzMWRjNzJhMik7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9iOTM4YjYwN2YzOWI0ZGUzYWNhOWJhNjU2OGFhOTQ3ZSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF8wYmM4ZjY2MmI3NDM0ODgzYWRjOWI0NGQxNGZiOWYyZSA9ICQoJzxkaXYgaWQ9Imh0bWxfMGJjOGY2NjJiNzQzNDg4M2FkYzliNDRkMTRmYjlmMmUiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjExODk5LCBMQUJFTDo2LCBTRUdNRU5UOldUQyAtIFRvdXJpc3RzL0NvbnY8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2I5MzhiNjA3ZjM5YjRkZTNhY2E5YmE2NTY4YWE5NDdlLnNldENvbnRlbnQoaHRtbF8wYmM4ZjY2MmI3NDM0ODgzYWRjOWI0NGQxNGZiOWYyZSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzBmM2I1ODBkNWI2OTRlNTg5YzA0NTIyYTBlMDAzYzVkLmJpbmRQb3B1cChwb3B1cF9iOTM4YjYwN2YzOWI0ZGUzYWNhOWJhNjU2OGFhOTQ3ZSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl83MTk4MzBjYmViMDE0MDkyODM0NWEzYWFjZjkyMGNhZCA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjI4MjM4MzUsMTE0LjE1NjYxNTRdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fNDljODdiMjQxYTgwNDJiOWI1ZTg3YWZhYTRjODY1NTMgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2dyZWVuJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzcxOTgzMGNiZWIwMTQwOTI4MzQ1YTNhYWNmOTIwY2FkLnNldEljb24oaWNvbl80OWM4N2IyNDFhODA0MmI5YjVlODdhZmFhNGM4NjU1Myk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8wYzk5NTFjYTYxNWE0Mzk1OTE4Yzc3N2UwMDUwMDZlOCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF84ZDJiYmU1MDZmYzU0ZTdlOGMxNDU3YjI3MjFiY2I5NiA9ICQoJzxkaXYgaWQ9Imh0bWxfOGQyYmJlNTA2ZmM1NGU3ZThjMTQ1N2IyNzIxYmNiOTYiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMDU0LCBMQUJFTDoyLCBTRUdNRU5UOldUQyAtIEhlYWx0aDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMGM5OTUxY2E2MTVhNDM5NTkxOGM3NzdlMDA1MDA2ZTguc2V0Q29udGVudChodG1sXzhkMmJiZTUwNmZjNTRlN2U4YzE0NTdiMjcyMWJjYjk2KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfNzE5ODMwY2JlYjAxNDA5MjgzNDVhM2FhY2Y5MjBjYWQuYmluZFBvcHVwKHBvcHVwXzBjOTk1MWNhNjE1YTQzOTU5MThjNzc3ZTAwNTAwNmU4KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzY4MmUwNmYzM2E3OTQwNDVhZjdmZDhkZmIzY2M4ODg1ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuNDQ0MjI2MiwxMTQuMDMwNzUxN10sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9iNDE5ZTJhOGMxOWI0MTVlOWYwYWFhOTFlMjYxOWRmYyA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnbGlnaHRyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfNjgyZTA2ZjMzYTc5NDA0NWFmN2ZkOGRmYjNjYzg4ODUuc2V0SWNvbihpY29uX2I0MTllMmE4YzE5YjQxNWU5ZjBhYWE5MWUyNjE5ZGZjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2Q5NWY2OGE0NWI4OTQ3Yzg4MzQxOTM4MmVjMWU5Yzg1ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzY2ZGU2YjUyYTM2YzQyMDE4YmQyOTMwZmY5N2U0NGRlID0gJCgnPGRpdiBpZD0iaHRtbF82NmRlNmI1MmEzNmM0MjAxOGJkMjkzMGZmOTdlNDRkZSIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTI1NjYsIExBQkVMOjYsIFNFR01FTlQ6V1RDIC0gVG91cmlzdHMvQ29udjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfZDk1ZjY4YTQ1Yjg5NDdjODgzNDE5MzgyZWMxZTljODUuc2V0Q29udGVudChodG1sXzY2ZGU2YjUyYTM2YzQyMDE4YmQyOTMwZmY5N2U0NGRlKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfNjgyZTA2ZjMzYTc5NDA0NWFmN2ZkOGRmYjNjYzg4ODUuYmluZFBvcHVwKHBvcHVwX2Q5NWY2OGE0NWI4OTQ3Yzg4MzQxOTM4MmVjMWU5Yzg1KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzM1ZmM2NDg1MDk1OTQ2NjE4Mjg3ZjI3MzRmMjM0ZjBjID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzUzNjU3LDExNC4xMzU0Ml0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl80YzNhZTNkYmRmZjA0ZGMyOGRiNzUzYTYwY2NhZmFjOCA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAncmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzM1ZmM2NDg1MDk1OTQ2NjE4Mjg3ZjI3MzRmMjM0ZjBjLnNldEljb24oaWNvbl80YzNhZTNkYmRmZjA0ZGMyOGRiNzUzYTYwY2NhZmFjOCk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9mY2Y2YzQ1Mzg5ZTQ0YTViOGIwNDdhOWVmOThhM2FhNSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF81ZDA0N2I3ZDIwOGM0ZGVlYTU0ZGRkODY2OWUzNTE5NiA9ICQoJzxkaXYgaWQ9Imh0bWxfNWQwNDdiN2QyMDhjNGRlZWE1NGRkZDg2NjllMzUxOTYiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyOTgzLCBMQUJFTDowLCBTRUdNRU5UOldUQyAtIFJlc2lkZW50aWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9mY2Y2YzQ1Mzg5ZTQ0YTViOGIwNDdhOWVmOThhM2FhNS5zZXRDb250ZW50KGh0bWxfNWQwNDdiN2QyMDhjNGRlZWE1NGRkZDg2NjllMzUxOTYpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl8zNWZjNjQ4NTA5NTk0NjYxODI4N2YyNzM0ZjIzNGYwYy5iaW5kUG9wdXAocG9wdXBfZmNmNmM0NTM4OWU0NGE1YjhiMDQ3YTllZjk4YTNhYTUpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfYWU2NDhlY2ZhYjA1NDlhYmE3ZjY5YzU0MzcxZTgwZWYgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi40MDYzNDYsMTEzLjk3MTYyOF0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl85NThiYTNlMWJhN2I0N2M4OTg4NDNlMTU2ZDVlMjA0MyA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAncmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyX2FlNjQ4ZWNmYWIwNTQ5YWJhN2Y2OWM1NDM3MWU4MGVmLnNldEljb24oaWNvbl85NThiYTNlMWJhN2I0N2M4OTg4NDNlMTU2ZDVlMjA0Myk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8zYTJjYWM5ZWRiODQ0NGFmODQ5YjFkMGI2YmQyMWIxMyA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF8wOWM5NTBjZmZkYTA0ZWY2YWNkMTVhNGNlMDE2ZmJlZSA9ICQoJzxkaXYgaWQ9Imh0bWxfMDljOTUwY2ZmZGEwNGVmNmFjZDE1YTRjZTAxNmZiZWUiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMDY0LCBMQUJFTDowLCBTRUdNRU5UOldUQyAtIFNhdmVyczwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfM2EyY2FjOWVkYjg0NDRhZjg0OWIxZDBiNmJkMjFiMTMuc2V0Q29udGVudChodG1sXzA5Yzk1MGNmZmRhMDRlZjZhY2QxNWE0Y2UwMTZmYmVlKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfYWU2NDhlY2ZhYjA1NDlhYmE3ZjY5YzU0MzcxZTgwZWYuYmluZFBvcHVwKHBvcHVwXzNhMmNhYzllZGI4NDQ0YWY4NDliMWQwYjZiZDIxYjEzKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2YzNmRkY2U5NzY0MDQ1MWU5ZTg5ZTI4YTgwMDg4NjM2ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzY4MDQxNiwxMTQuMDg0NDE3Nl0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9iMzJmMGE4Y2JhMDk0MTNkYWM0Njg3M2I2ZTMwZDEyNyA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAncmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyX2YzNmRkY2U5NzY0MDQ1MWU5ZTg5ZTI4YTgwMDg4NjM2LnNldEljb24oaWNvbl9iMzJmMGE4Y2JhMDk0MTNkYWM0Njg3M2I2ZTMwZDEyNyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9lM2EyMGQyM2U1Mjg0ZWEwOWVmZjkxYjI2MTI4NDE5ZCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF85NmNjM2RmZWVhYjg0MTZiYjU1YTQ4MTBkMjM3MWQ4MCA9ICQoJzxkaXYgaWQ9Imh0bWxfOTZjYzNkZmVlYWI4NDE2YmI1NWE0ODEwZDIzNzFkODAiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMjgwLCBMQUJFTDowLCBTRUdNRU5UOldUQyAtIFJlc2lkZW50aWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9lM2EyMGQyM2U1Mjg0ZWEwOWVmZjkxYjI2MTI4NDE5ZC5zZXRDb250ZW50KGh0bWxfOTZjYzNkZmVlYWI4NDE2YmI1NWE0ODEwZDIzNzFkODApOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl9mMzZkZGNlOTc2NDA0NTFlOWU4OWUyOGE4MDA4ODYzNi5iaW5kUG9wdXAocG9wdXBfZTNhMjBkMjNlNTI4NGVhMDllZmY5MWIyNjEyODQxOWQpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfYjA2OGZmZjIxYzNhNDQ5MGIwMmRlNzgyNDUyM2E5Y2QgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi41MDA5Njk5LDExNC4xMzQ0MTExXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzFlMDI4NTY2MDAyNjQzMTliMjBiNDlmMzcxYTQ0ZDY1ID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdiZWlnZScsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl9iMDY4ZmZmMjFjM2E0NDkwYjAyZGU3ODI0NTIzYTljZC5zZXRJY29uKGljb25fMWUwMjg1NjYwMDI2NDMxOWIyMGI0OWYzNzFhNDRkNjUpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfMjhjYzU3ZDUzZmU3NGUzNmI5YjhkYzE2YTk1N2U0ZDYgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfYzdiMzgwZDc3Y2ZmNDFlMWJkODkyZDlkZGI3OGNlZGQgPSAkKCc8ZGl2IGlkPSJodG1sX2M3YjM4MGQ3N2NmZjQxZTFiZDg5MmQ5ZGRiNzhjZWRkIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjQzOSwgTEFCRUw6NywgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMjhjYzU3ZDUzZmU3NGUzNmI5YjhkYzE2YTk1N2U0ZDYuc2V0Q29udGVudChodG1sX2M3YjM4MGQ3N2NmZjQxZTFiZDg5MmQ5ZGRiNzhjZWRkKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfYjA2OGZmZjIxYzNhNDQ5MGIwMmRlNzgyNDUyM2E5Y2QuYmluZFBvcHVwKHBvcHVwXzI4Y2M1N2Q1M2ZlNzRlMzZiOWI4ZGMxNmE5NTdlNGQ2KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzdmY2EyZjZmOThiZTRjODFhM2VmOGY2OTgzMGRiMjhkID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMjk3MzU1NywxMTQuMTczNTQxOF0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9iMzY2MzUzMDQ3MTI0NjdkOWYwODA0M2NmOTE1OGNlNCA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnbGlnaHRyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfN2ZjYTJmNmY5OGJlNGM4MWEzZWY4ZjY5ODMwZGIyOGQuc2V0SWNvbihpY29uX2IzNjYzNTMwNDcxMjQ2N2Q5ZjA4MDQzY2Y5MTU4Y2U0KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzM5NzEzZjI1ODhhNTQ0NDVhNDQwZTkyYzRhMDg2OGU5ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzM4NWJiY2ViMTZjMTRkMmZiMTk2MjJmYThmZTE3NjNmID0gJCgnPGRpdiBpZD0iaHRtbF8zODViYmNlYjE2YzE0ZDJmYjE5NjIyZmE4ZmUxNzYzZiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTI0ODAsIExBQkVMOjYsIFNFR01FTlQ6V1RDIC0gVG91cmlzdHMvQ29udjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMzk3MTNmMjU4OGE1NDQ0NWE0NDBlOTJjNGEwODY4ZTkuc2V0Q29udGVudChodG1sXzM4NWJiY2ViMTZjMTRkMmZiMTk2MjJmYThmZTE3NjNmKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfN2ZjYTJmNmY5OGJlNGM4MWEzZWY4ZjY5ODMwZGIyOGQuYmluZFBvcHVwKHBvcHVwXzM5NzEzZjI1ODhhNTQ0NDVhNDQwZTkyYzRhMDg2OGU5KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2ZkNGNmNzY1MWZmZjRlYmFiZGZlOGMyYzZlNDVlZTI1ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuNDU2ODIwMywxMTQuMDAzOTAxNV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl8wNWVmM2FiNjk1MmI0MTczYWJiNGRlY2JlY2NlZjg1OSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnYmVpZ2UnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfZmQ0Y2Y3NjUxZmZmNGViYWJkZmU4YzJjNmU0NWVlMjUuc2V0SWNvbihpY29uXzA1ZWYzYWI2OTUyYjQxNzNhYmI0ZGVjYmVjY2VmODU5KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzQwNjRlNjFhMTEwMDQxN2E4M2QyOTIzZGE1MWE4ZWVjID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2M1NDk2NWZhMTU1YTRkZWM5YTExOTAwOGYwZGY2ODc2ID0gJCgnPGRpdiBpZD0iaHRtbF9jNTQ5NjVmYTE1NWE0ZGVjOWExMTkwMDhmMGRmNjg3NiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTI2MjUsIExBQkVMOjcsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzQwNjRlNjFhMTEwMDQxN2E4M2QyOTIzZGE1MWE4ZWVjLnNldENvbnRlbnQoaHRtbF9jNTQ5NjVmYTE1NWE0ZGVjOWExMTkwMDhmMGRmNjg3Nik7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyX2ZkNGNmNzY1MWZmZjRlYmFiZGZlOGMyYzZlNDVlZTI1LmJpbmRQb3B1cChwb3B1cF80MDY0ZTYxYTExMDA0MTdhODNkMjkyM2RhNTFhOGVlYyk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9mNTg3M2JlYzlkMGQ0NWJhYjA0OGI1N2VkZThmYzc2YSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjI2NzE4NjcsMTE0LjI0NjQ4NThdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fYzI3M2NlMTE1YTFiNDNhMWE2NjU3NDZjYTQ4MGY3NTIgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3JlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl9mNTg3M2JlYzlkMGQ0NWJhYjA0OGI1N2VkZThmYzc2YS5zZXRJY29uKGljb25fYzI3M2NlMTE1YTFiNDNhMWE2NjU3NDZjYTQ4MGY3NTIpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfYjMwZWFhNTYyYzE1NDk1ODliMGNiNzU5OWRlYjhkODQgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNGVhZTRkMDZiMDQ0NDhjNGIzODYxYmM1MThiMDNhYjEgPSAkKCc8ZGl2IGlkPSJodG1sXzRlYWU0ZDA2YjA0NDQ4YzRiMzg2MWJjNTE4YjAzYWIxIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMTY2OCwgTEFCRUw6MCwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfYjMwZWFhNTYyYzE1NDk1ODliMGNiNzU5OWRlYjhkODQuc2V0Q29udGVudChodG1sXzRlYWU0ZDA2YjA0NDQ4YzRiMzg2MWJjNTE4YjAzYWIxKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfZjU4NzNiZWM5ZDBkNDViYWIwNDhiNTdlZGU4ZmM3NmEuYmluZFBvcHVwKHBvcHVwX2IzMGVhYTU2MmMxNTQ5NTg5YjBjYjc1OTlkZWI4ZDg0KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzVmMTQyZjJlOGVjMDRjMTRiMTRlMDk5NzVkNmY4NmIwID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzgxMjQwOCwxMTMuOTY0MzgyN10sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl85NjRkOWZkYTc5Nzc0ZWRhYTRmMDlmYWNkYjY2Y2Q5MiA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnYmVpZ2UnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfNWYxNDJmMmU4ZWMwNGMxNGIxNGUwOTk3NWQ2Zjg2YjAuc2V0SWNvbihpY29uXzk2NGQ5ZmRhNzk3NzRlZGFhNGYwOWZhY2RiNjZjZDkyKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2JhZmVlYzhkMTkxYzRjNzNhZjYxNmY4NjM3OWNhZWQwID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzgxYTIwMjA0ZjM3NDRhYzg4YjQzOGUwY2Q2ZWY2Y2ZmID0gJCgnPGRpdiBpZD0iaHRtbF84MWEyMDIwNGYzNzQ0YWM4OGI0MzhlMGNkNmVmNmNmZiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTE4OTcsIExBQkVMOjcsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2JhZmVlYzhkMTkxYzRjNzNhZjYxNmY4NjM3OWNhZWQwLnNldENvbnRlbnQoaHRtbF84MWEyMDIwNGYzNzQ0YWM4OGI0MzhlMGNkNmVmNmNmZik7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzVmMTQyZjJlOGVjMDRjMTRiMTRlMDk5NzVkNmY4NmIwLmJpbmRQb3B1cChwb3B1cF9iYWZlZWM4ZDE5MWM0YzczYWY2MTZmODYzNzljYWVkMCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl8zM2ExOWNiMTY5Mjk0YzkzODljZjY3Yjg5NWU5M2JiMiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjI4OTE2MywxMTMuOTQxMjY5XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzY0NjRhMGI4MTdiYTRhM2RhZjc3YmFmNmIwZWJlNDllID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdwdXJwbGUnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfMzNhMTljYjE2OTI5NGM5Mzg5Y2Y2N2I4OTVlOTNiYjIuc2V0SWNvbihpY29uXzY0NjRhMGI4MTdiYTRhM2RhZjc3YmFmNmIwZWJlNDllKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzY2NDIwOTc1NTE2YjQxZmM4ZWQ2MTIxNjUzN2NkNmI3ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzI1NGYyZDFmYjZlMjQ4YzJiYzRiMWRlMjZlNzQwZDkxID0gJCgnPGRpdiBpZD0iaHRtbF8yNTRmMmQxZmI2ZTI0OGMyYmM0YjFkZTI2ZTc0MGQ5MSIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIwMTQsIExBQkVMOjMsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzY2NDIwOTc1NTE2YjQxZmM4ZWQ2MTIxNjUzN2NkNmI3LnNldENvbnRlbnQoaHRtbF8yNTRmMmQxZmI2ZTI0OGMyYmM0YjFkZTI2ZTc0MGQ5MSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzMzYTE5Y2IxNjkyOTRjOTM4OWNmNjdiODk1ZTkzYmIyLmJpbmRQb3B1cChwb3B1cF82NjQyMDk3NTUxNmI0MWZjOGVkNjEyMTY1MzdjZDZiNyk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl83NmQxNDlhNzY0YzE0NTlkODdiOGFlZGVkYzQxMWIwOSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjM2Mzc4OTYsMTE0LjEzODM2MTZdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fNDQ2MDVjODdlZjVmNDljY2JkMDQwNjU2YjcwOTc1ZDMgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3JlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl83NmQxNDlhNzY0YzE0NTlkODdiOGFlZGVkYzQxMWIwOS5zZXRJY29uKGljb25fNDQ2MDVjODdlZjVmNDljY2JkMDQwNjU2YjcwOTc1ZDMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfMjkyNzU2MDUzMzhmNDZlMWJhMjkzNTViMjRhNTZkZGQgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfZTRjY2E0NDE5YTE1NGI3OWI3MDAyNThjMTgxMDYwZDQgPSAkKCc8ZGl2IGlkPSJodG1sX2U0Y2NhNDQxOWExNTRiNzliNzAwMjU4YzE4MTA2MGQ0IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjA1NywgTEFCRUw6MCwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMjkyNzU2MDUzMzhmNDZlMWJhMjkzNTViMjRhNTZkZGQuc2V0Q29udGVudChodG1sX2U0Y2NhNDQxOWExNTRiNzliNzAwMjU4YzE4MTA2MGQ0KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfNzZkMTQ5YTc2NGMxNDU5ZDg3YjhhZWRlZGM0MTFiMDkuYmluZFBvcHVwKHBvcHVwXzI5Mjc1NjA1MzM4ZjQ2ZTFiYTI5MzU1YjI0YTU2ZGRkKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzg3ZGM5M2E2NjNjOTQ5OTZiZDI2OGY1ODRhN2FiOWZlID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzY3ODEzNCwxMTQuMTgwNjIxN10sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl83ZmYzOTdhYjhiMWI0YTVhYWNhYWMzMTlmYmZkYjBjMSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnYmVpZ2UnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfODdkYzkzYTY2M2M5NDk5NmJkMjY4ZjU4NGE3YWI5ZmUuc2V0SWNvbihpY29uXzdmZjM5N2FiOGIxYjRhNWFhY2FhYzMxOWZiZmRiMGMxKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2Q1Y2JlNmFkMTBjNTRlN2M4OGM3YzIyOGNkNTY1NDRkID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzhmNWVlYTUxNDk3MjRkOTlhYjE5N2YwNWM2ZDJiOGNlID0gJCgnPGRpdiBpZD0iaHRtbF84ZjVlZWE1MTQ5NzI0ZDk5YWIxOTdmMDVjNmQyYjhjZSIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIzMDksIExBQkVMOjcsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2Q1Y2JlNmFkMTBjNTRlN2M4OGM3YzIyOGNkNTY1NDRkLnNldENvbnRlbnQoaHRtbF84ZjVlZWE1MTQ5NzI0ZDk5YWIxOTdmMDVjNmQyYjhjZSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzg3ZGM5M2E2NjNjOTQ5OTZiZDI2OGY1ODRhN2FiOWZlLmJpbmRQb3B1cChwb3B1cF9kNWNiZTZhZDEwYzU0ZTdjODhjN2MyMjhjZDU2NTQ0ZCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9jNDhmZDNmZjkwMzg0Njg3ODliMThhOTEyYmZkZTU0YSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjM0NTkzMDIsMTE0LjE5MTQ2OTFdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fMGU2NjVjZmIzNDc0NGRjZTgzNGEwZjBkNTFiYzk3ZGUgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2JlaWdlJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyX2M0OGZkM2ZmOTAzODQ2ODc4OWIxOGE5MTJiZmRlNTRhLnNldEljb24oaWNvbl8wZTY2NWNmYjM0NzQ0ZGNlODM0YTBmMGQ1MWJjOTdkZSk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF82YTBlNDhmYjEwZDg0OGRiOGFiNzlhMDcxYmNiM2I5ZSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF8wMjY1YWQ1Y2FlNzk0MzY4YmU4NjQ4NzUyOGM0ODhjYyA9ICQoJzxkaXYgaWQ9Imh0bWxfMDI2NWFkNWNhZTc5NDM2OGJlODY0ODc1MjhjNDg4Y2MiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyNjEzLCBMQUJFTDo3LCBTRUdNRU5UOldUQyAtIFNhdmVyczwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNmEwZTQ4ZmIxMGQ4NDhkYjhhYjc5YTA3MWJjYjNiOWUuc2V0Q29udGVudChodG1sXzAyNjVhZDVjYWU3OTQzNjhiZTg2NDg3NTI4YzQ4OGNjKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfYzQ4ZmQzZmY5MDM4NDY4Nzg5YjE4YTkxMmJmZGU1NGEuYmluZFBvcHVwKHBvcHVwXzZhMGU0OGZiMTBkODQ4ZGI4YWI3OWEwNzFiY2IzYjllKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2I2YzRjNjhmYzE1NjRlOWQ5MDM2Zjk1MjdhOTg3MGVhID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMjk1MDY4OSwxMTQuMTY2ODY2MV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl82ODQwN2NhZjcwZjU0NGNjODA5YTVkNGFkNTdmYzNiYyA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnZGFya2JsdWUnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfYjZjNGM2OGZjMTU2NGU5ZDkwMzZmOTUyN2E5ODcwZWEuc2V0SWNvbihpY29uXzY4NDA3Y2FmNzBmNTQ0Y2M4MDlhNWQ0YWQ1N2ZjM2JjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2Q4ZDAwZWUyMjNhZDRmMGRiYmRiN2MxZTA2MmUwMmMxID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzE0ODg3NzNhYzY3NjQ4OGZhZWY0Mzk2ZWYzYzhiMjAxID0gJCgnPGRpdiBpZD0iaHRtbF8xNDg4NzczYWM2NzY0ODhmYWVmNDM5NmVmM2M4YjIwMSIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTMyNjEsIExBQkVMOjgsIFNFR01FTlQ6V1RDIC0gSGVhbHRoPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9kOGQwMGVlMjIzYWQ0ZjBkYmJkYjdjMWUwNjJlMDJjMS5zZXRDb250ZW50KGh0bWxfMTQ4ODc3M2FjNjc2NDg4ZmFlZjQzOTZlZjNjOGIyMDEpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl9iNmM0YzY4ZmMxNTY0ZTlkOTAzNmY5NTI3YTk4NzBlYS5iaW5kUG9wdXAocG9wdXBfZDhkMDBlZTIyM2FkNGYwZGJiZGI3YzFlMDYyZTAyYzEpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfMTVlOGFmY2EyN2Q2NDhmYjgzOGI3Njk4MGU3NjZhOTIgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4yODIzOTcyLDExNC4xOTIyMzgyXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzUyYmUyOWI0ZTBmODRjMmQ5ZDc0Yjg0NTJhZjhmZmFkID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdsaWdodHJlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl8xNWU4YWZjYTI3ZDY0OGZiODM4Yjc2OTgwZTc2NmE5Mi5zZXRJY29uKGljb25fNTJiZTI5YjRlMGY4NGMyZDlkNzRiODQ1MmFmOGZmYWQpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfMWVkMzJjN2E0ZDM2NGZjOGE0NGMyMjk1ZDlkZGY2ZTQgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfYjdjYzk0NDYyOWNjNGFiODhiNjUxZGNjN2UwNTBjMjkgPSAkKCc8ZGl2IGlkPSJodG1sX2I3Y2M5NDQ2MjljYzRhYjg4YjY1MWRjYzdlMDUwYzI5IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjAwMiwgTEFCRUw6NiwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMWVkMzJjN2E0ZDM2NGZjOGE0NGMyMjk1ZDlkZGY2ZTQuc2V0Q29udGVudChodG1sX2I3Y2M5NDQ2MjljYzRhYjg4YjY1MWRjYzdlMDUwYzI5KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfMTVlOGFmY2EyN2Q2NDhmYjgzOGI3Njk4MGU3NjZhOTIuYmluZFBvcHVwKHBvcHVwXzFlZDMyYzdhNGQzNjRmYzhhNDRjMjI5NWQ5ZGRmNmU0KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2FiZWJlZjRjYmMyNTQzNWY5NmRhZDJkNDJhNWQ2NDU4ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuNTAxMDQwOCwxMTQuMTMxMzE1Ml0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9jMDE2ZTNkNjdmNzI0ZDA0OGM0OThlM2VhOTljYzc5ZCA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnYmVpZ2UnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfYWJlYmVmNGNiYzI1NDM1Zjk2ZGFkMmQ0MmE1ZDY0NTguc2V0SWNvbihpY29uX2MwMTZlM2Q2N2Y3MjRkMDQ4YzQ5OGUzZWE5OWNjNzlkKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2E5YjliNjdlNWExYjQ1ZTQ4Nzg4OWZiNmJlNjUxMjI0ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzNkYmIwMjAyMjExMjQ1Yjc4M2MxZmU4MDEzMTAxMDk2ID0gJCgnPGRpdiBpZD0iaHRtbF8zZGJiMDIwMjIxMTI0NWI3ODNjMWZlODAxMzEwMTA5NiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIwNTUsIExBQkVMOjcsIFNFR01FTlQ6V1RDIC0gVG91cmlzdHMvQ29udjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfYTliOWI2N2U1YTFiNDVlNDg3ODg5ZmI2YmU2NTEyMjQuc2V0Q29udGVudChodG1sXzNkYmIwMjAyMjExMjQ1Yjc4M2MxZmU4MDEzMTAxMDk2KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfYWJlYmVmNGNiYzI1NDM1Zjk2ZGFkMmQ0MmE1ZDY0NTguYmluZFBvcHVwKHBvcHVwX2E5YjliNjdlNWExYjQ1ZTQ4Nzg4OWZiNmJlNjUxMjI0KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzFkZGM5NTI1MmY1MjQzODA4YTYyNmM5Njk4MzEwNzcxID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuNDIyMDc0NSwxMTQuMjI1ODM3Ml0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9hZGE1Njc5MjljYzU0MmNlOWJkZWQwYzE4NGMyYWFjNiA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAncmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzFkZGM5NTI1MmY1MjQzODA4YTYyNmM5Njk4MzEwNzcxLnNldEljb24oaWNvbl9hZGE1Njc5MjljYzU0MmNlOWJkZWQwYzE4NGMyYWFjNik7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF85NmUwOTQwNGY3Y2Q0NzdiODYxOWI4MmE4YzZiYTZiYSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9lMGEwMDA2MjBiMmM0NGE1OTdkZTBmZTM5MzIyZmU0ZSA9ICQoJzxkaXYgaWQ9Imh0bWxfZTBhMDAwNjIwYjJjNDRhNTk3ZGUwZmUzOTMyMmZlNGUiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMDY5LCBMQUJFTDowLCBTRUdNRU5UOldUQyAtIFNhdmVyczwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfOTZlMDk0MDRmN2NkNDc3Yjg2MTliODJhOGM2YmE2YmEuc2V0Q29udGVudChodG1sX2UwYTAwMDYyMGIyYzQ0YTU5N2RlMGZlMzkzMjJmZTRlKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfMWRkYzk1MjUyZjUyNDM4MDhhNjI2Yzk2OTgzMTA3NzEuYmluZFBvcHVwKHBvcHVwXzk2ZTA5NDA0ZjdjZDQ3N2I4NjE5YjgyYThjNmJhNmJhKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzc2NDA0ZTQxMjNmNTQ4NDZiODIwOWQ1NDIxMDMzYjJmID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMTQ3MDMzMSwxMTMuNTU5ODM5XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzVkZjEwMzVkZjEzZTRmZDliNTU5OGMxY2ViNDU0ZmMwID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdwdXJwbGUnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfNzY0MDRlNDEyM2Y1NDg0NmI4MjA5ZDU0MjEwMzNiMmYuc2V0SWNvbihpY29uXzVkZjEwMzVkZjEzZTRmZDliNTU5OGMxY2ViNDU0ZmMwKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzM2NTI1NWNkZjliZjQ0NTY5ZDE1MGFiYzMyOTdmYmU2ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzVkZTMyMzMyYTcxMjQ5MWJiMTE1MmZmNjNkMmE1MTkwID0gJCgnPGRpdiBpZD0iaHRtbF81ZGUzMjMzMmE3MTI0OTFiYjExNTJmZjYzZDJhNTE5MCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIxOTksIExBQkVMOjMsIFNFR01FTlQ6V1RDIC0gVG91cmlzdHMvQ29udjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMzY1MjU1Y2RmOWJmNDQ1NjlkMTUwYWJjMzI5N2ZiZTYuc2V0Q29udGVudChodG1sXzVkZTMyMzMyYTcxMjQ5MWJiMTE1MmZmNjNkMmE1MTkwKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfNzY0MDRlNDEyM2Y1NDg0NmI4MjA5ZDU0MjEwMzNiMmYuYmluZFBvcHVwKHBvcHVwXzM2NTI1NWNkZjliZjQ0NTY5ZDE1MGFiYzMyOTdmYmU2KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzUxMzgyNDY2NzJhNzQ2ZDdhNWNmYjRjYzhjYWRkYThiID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzMxMDE3NSwxMTQuMTU5ODIwMl0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl82YTkxNGY5MjgyOTQ0YzU3OGU0NmM4ODUwY2I2NzA4MiA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAncmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzUxMzgyNDY2NzJhNzQ2ZDdhNWNmYjRjYzhjYWRkYThiLnNldEljb24oaWNvbl82YTkxNGY5MjgyOTQ0YzU3OGU0NmM4ODUwY2I2NzA4Mik7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8yNTFkODg3YzQyYjY0ZjUzYTIwZmJkMjRiOTFmNWY4OCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF83NjBjZTUzNTRjYzk0YjVjOTMyZjZkMGE3ODA3ZGIzOCA9ICQoJzxkaXYgaWQ9Imh0bWxfNzYwY2U1MzU0Y2M5NGI1YzkzMmY2ZDBhNzgwN2RiMzgiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMzc4LCBMQUJFTDowLCBTRUdNRU5UOldUQyAtIFJlc2lkZW50aWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8yNTFkODg3YzQyYjY0ZjUzYTIwZmJkMjRiOTFmNWY4OC5zZXRDb250ZW50KGh0bWxfNzYwY2U1MzU0Y2M5NGI1YzkzMmY2ZDBhNzgwN2RiMzgpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl81MTM4MjQ2NjcyYTc0NmQ3YTVjZmI0Y2M4Y2FkZGE4Yi5iaW5kUG9wdXAocG9wdXBfMjUxZDg4N2M0MmI2NGY1M2EyMGZiZDI0YjkxZjVmODgpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfNzA2Y2JhYWMxNjE1NDIzNzk0NGUzZGJkYWNlODAzOTkgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4zMDU0ODE5LDExNC4xODU3ODQ0XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzJjYjdhYTQ3MzA3ZDRjYjQ4NjFmNzE1MzI0ZTA4YTA2ID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfNzA2Y2JhYWMxNjE1NDIzNzk0NGUzZGJkYWNlODAzOTkuc2V0SWNvbihpY29uXzJjYjdhYTQ3MzA3ZDRjYjQ4NjFmNzE1MzI0ZTA4YTA2KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzE0MzFmMzRiMGUyYTQyODc4M2ZlMDQzZThiMjhhYjE2ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzllYWIwY2YzNGRkYTQwOTRiMzk4MTAxY2I3MGM3MzA0ID0gJCgnPGRpdiBpZD0iaHRtbF85ZWFiMGNmMzRkZGE0MDk0YjM5ODEwMWNiNzBjNzMwNCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTI1MDYsIExBQkVMOjAsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzE0MzFmMzRiMGUyYTQyODc4M2ZlMDQzZThiMjhhYjE2LnNldENvbnRlbnQoaHRtbF85ZWFiMGNmMzRkZGE0MDk0YjM5ODEwMWNiNzBjNzMwNCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzcwNmNiYWFjMTYxNTQyMzc5NDRlM2RiZGFjZTgwMzk5LmJpbmRQb3B1cChwb3B1cF8xNDMxZjM0YjBlMmE0Mjg3ODNmZTA0M2U4YjI4YWIxNik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl8wNjBiZDVhNTUxNzY0MGNmYmViM2JkNjRmMzEzZGI2ZCA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjM1ODQ4NywxMTQuMTA3MjYxXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzVmMDk2NGFiZDhjNDQyMmRiMDhmZWFjZWJlNWQ5ZTc2ID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfMDYwYmQ1YTU1MTc2NDBjZmJlYjNiZDY0ZjMxM2RiNmQuc2V0SWNvbihpY29uXzVmMDk2NGFiZDhjNDQyMmRiMDhmZWFjZWJlNWQ5ZTc2KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzFmYjNjNTUzZDY0ZTRiZGU4ZGRhZmNlMzJjOTRmMWQ5ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzliMTRiY2ViZjE3ODQwMjJiYjUzYTQwOTM2Njg3Y2QzID0gJCgnPGRpdiBpZD0iaHRtbF85YjE0YmNlYmYxNzg0MDIyYmI1M2E0MDkzNjY4N2NkMyIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTI2MTksIExBQkVMOjAsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzFmYjNjNTUzZDY0ZTRiZGU4ZGRhZmNlMzJjOTRmMWQ5LnNldENvbnRlbnQoaHRtbF85YjE0YmNlYmYxNzg0MDIyYmI1M2E0MDkzNjY4N2NkMyk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzA2MGJkNWE1NTE3NjQwY2ZiZWIzYmQ2NGYzMTNkYjZkLmJpbmRQb3B1cChwb3B1cF8xZmIzYzU1M2Q2NGU0YmRlOGRkYWZjZTMyYzk0ZjFkOSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9lOWY5ZjJhMGNmYTU0ODQwODY5NGU0ZjNjZGUwZjRlNSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjI5NDcwMywxMTQuMTY4OTIzNV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl82NDY2ZjM2MzliMWM0YjEwODc0Yjg0NmVlNzExZjQ5MSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnb3JhbmdlJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyX2U5ZjlmMmEwY2ZhNTQ4NDA4Njk0ZTRmM2NkZTBmNGU1LnNldEljb24oaWNvbl82NDY2ZjM2MzliMWM0YjEwODc0Yjg0NmVlNzExZjQ5MSk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8yMWNjMTc4YTg0OWQ0OGRmOGZkM2YxMTlhMTRjNzk4ZSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF82ZjI1YzZhMTFkYzA0M2RjYmUxOGRhYmVmZDk2YWQ1NyA9ICQoJzxkaXYgaWQ9Imh0bWxfNmYyNWM2YTExZGMwNDNkY2JlMThkYWJlZmQ5NmFkNTciIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjExNzQ1LCBMQUJFTDo0LCBTRUdNRU5UOldUQyAtIERlc3RpbmF0aW9uPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8yMWNjMTc4YTg0OWQ0OGRmOGZkM2YxMTlhMTRjNzk4ZS5zZXRDb250ZW50KGh0bWxfNmYyNWM2YTExZGMwNDNkY2JlMThkYWJlZmQ5NmFkNTcpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl9lOWY5ZjJhMGNmYTU0ODQwODY5NGU0ZjNjZGUwZjRlNS5iaW5kUG9wdXAocG9wdXBfMjFjYzE3OGE4NDlkNDhkZjhmZDNmMTE5YTE0Yzc5OGUpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfNzYxMjM2ZWYyYTY2NGVlMWI4YTc3Yzc4MTM4MzJlN2IgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4yNDE0NjEzLDExNC4xNTYwOTQyXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2QwYzdjYzA4MTQwZjQ5NTZiMjdlZWZmYzIwMmZhYjI2ID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdiZWlnZScsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl83NjEyMzZlZjJhNjY0ZWUxYjhhNzdjNzgxMzgzMmU3Yi5zZXRJY29uKGljb25fZDBjN2NjMDgxNDBmNDk1NmIyN2VlZmZjMjAyZmFiMjYpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNDZhMWIwNGI3ZThiNGYzMWI2MGFkYzY5YWQxNGEwNDYgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfYTkzNTdlMjdlMmI3NGU1Yzk4MTg4MGNhMTJlNWVlZTMgPSAkKCc8ZGl2IGlkPSJodG1sX2E5MzU3ZTI3ZTJiNzRlNWM5ODE4ODBjYTEyZTVlZWUzIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMTk3MCwgTEFCRUw6NywgU0VHTUVOVDpXVEMgLSBTYXZlcnM8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzQ2YTFiMDRiN2U4YjRmMzFiNjBhZGM2OWFkMTRhMDQ2LnNldENvbnRlbnQoaHRtbF9hOTM1N2UyN2UyYjc0ZTVjOTgxODgwY2ExMmU1ZWVlMyk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzc2MTIzNmVmMmE2NjRlZTFiOGE3N2M3ODEzODMyZTdiLmJpbmRQb3B1cChwb3B1cF80NmExYjA0YjdlOGI0ZjMxYjYwYWRjNjlhZDE0YTA0Nik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl80MTJjMmRkMmQ0ZGU0NDhkOWIxMmM2MzNlNDUzNTU2MSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjI4NjMyMywxMTQuMTM3MTk3Nl0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl8xYWFmMGUyMTUyNWE0NWRlYWU4OTFhMTgwM2ZlNjgxNiA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAncmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzQxMmMyZGQyZDRkZTQ0OGQ5YjEyYzYzM2U0NTM1NTYxLnNldEljb24oaWNvbl8xYWFmMGUyMTUyNWE0NWRlYWU4OTFhMTgwM2ZlNjgxNik7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF81ZTM5NTkyZTBiODA0NTFkOWMwMDBlYTIzNzJlNDQ2OCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9iMTUyNjQzODA3MGE0NmYyOGJiMmVlYmVhNWU4YTIyNSA9ICQoJzxkaXYgaWQ9Imh0bWxfYjE1MjY0MzgwNzBhNDZmMjhiYjJlZWJlYTVlOGEyMjUiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMDAwLCBMQUJFTDowLCBTRUdNRU5UOldUQyAtIFJlc2lkZW50aWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF81ZTM5NTkyZTBiODA0NTFkOWMwMDBlYTIzNzJlNDQ2OC5zZXRDb250ZW50KGh0bWxfYjE1MjY0MzgwNzBhNDZmMjhiYjJlZWJlYTVlOGEyMjUpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl80MTJjMmRkMmQ0ZGU0NDhkOWIxMmM2MzNlNDUzNTU2MS5iaW5kUG9wdXAocG9wdXBfNWUzOTU5MmUwYjgwNDUxZDljMDAwZWEyMzcyZTQ0NjgpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfNWEyOWVlMmUzNDIxNGZhZDg4YzMwMDc4OGYxOTJlNmEgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4yODM1NjI5LDExNC4xMzA0OTgyXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzE1YzEyOWI4ZWU3MzQ1OTk5Yzc2YmJjN2NlZTRmOGJmID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfNWEyOWVlMmUzNDIxNGZhZDg4YzMwMDc4OGYxOTJlNmEuc2V0SWNvbihpY29uXzE1YzEyOWI4ZWU3MzQ1OTk5Yzc2YmJjN2NlZTRmOGJmKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2VjNDg3MTdhN2NiMTQ1YTI4ZDZlMDlkZDg3MmE4YWJlID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzBkMjYzZDg2NTkwODRlMjM4YTI1OWVkMGI5OTI0NzFiID0gJCgnPGRpdiBpZD0iaHRtbF8wZDI2M2Q4NjU5MDg0ZTIzOGEyNTllZDBiOTkyNDcxYiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIxOTEsIExBQkVMOjAsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2VjNDg3MTdhN2NiMTQ1YTI4ZDZlMDlkZDg3MmE4YWJlLnNldENvbnRlbnQoaHRtbF8wZDI2M2Q4NjU5MDg0ZTIzOGEyNTllZDBiOTkyNDcxYik7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzVhMjllZTJlMzQyMTRmYWQ4OGMzMDA3ODhmMTkyZTZhLmJpbmRQb3B1cChwb3B1cF9lYzQ4NzE3YTdjYjE0NWEyOGQ2ZTA5ZGQ4NzJhOGFiZSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl80MjhhYzU0ZDEzY2Q0MTMwOTZkYmVkM2ZmYTA2YTFjOSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjM2ODg2OTEsMTE0LjExNzU4ODNdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fNDk4NzhhOGVmMTE0NGY1OWE2M2ZiM2I2N2M5OTI4YTcgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2xpZ2h0cmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzQyOGFjNTRkMTNjZDQxMzA5NmRiZWQzZmZhMDZhMWM5LnNldEljb24oaWNvbl80OTg3OGE4ZWYxMTQ0ZjU5YTYzZmIzYjY3Yzk5MjhhNyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9jOTdiM2Q0MTI2ZjU0MzU2OGQyNDQ5OGI2ZWRkODU3MCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF85NDFjYzkxODQyMTI0NTY4ODQ1ZTVjMTI2NjJjYWQ1MyA9ICQoJzxkaXYgaWQ9Imh0bWxfOTQxY2M5MTg0MjEyNDU2ODg0NWU1YzEyNjYyY2FkNTMiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjExNzA3LCBMQUJFTDo2LCBTRUdNRU5UOldUQyAtIFJlc2lkZW50aWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9jOTdiM2Q0MTI2ZjU0MzU2OGQyNDQ5OGI2ZWRkODU3MC5zZXRDb250ZW50KGh0bWxfOTQxY2M5MTg0MjEyNDU2ODg0NWU1YzEyNjYyY2FkNTMpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl80MjhhYzU0ZDEzY2Q0MTMwOTZkYmVkM2ZmYTA2YTFjOS5iaW5kUG9wdXAocG9wdXBfYzk3YjNkNDEyNmY1NDM1NjhkMjQ0OThiNmVkZDg1NzApOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfZmRiN2JiOGNjODVhNDNjZWJhODRlNGM3MjQ4M2EzYWMgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4zMzY0NDI2LDExNC4xOTg5MjU5XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2I3OWE5YjdmZTA0NjQwMDc4YmUxMjQ1ZGU4ZTliMWFlID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfZmRiN2JiOGNjODVhNDNjZWJhODRlNGM3MjQ4M2EzYWMuc2V0SWNvbihpY29uX2I3OWE5YjdmZTA0NjQwMDc4YmUxMjQ1ZGU4ZTliMWFlKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzIxNDEyOWVhMmZjODRlYjRhOTE1M2NmOTg3Njg5MWRiID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2I3ZTA5ZmNiMWZkYzQwNmJhMDg1ZDA2YmU0M2VjZDYxID0gJCgnPGRpdiBpZD0iaHRtbF9iN2UwOWZjYjFmZGM0MDZiYTA4NWQwNmJlNDNlY2Q2MSIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTE3NDcsIExBQkVMOjAsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzIxNDEyOWVhMmZjODRlYjRhOTE1M2NmOTg3Njg5MWRiLnNldENvbnRlbnQoaHRtbF9iN2UwOWZjYjFmZGM0MDZiYTA4NWQwNmJlNDNlY2Q2MSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyX2ZkYjdiYjhjYzg1YTQzY2ViYTg0ZTRjNzI0ODNhM2FjLmJpbmRQb3B1cChwb3B1cF8yMTQxMjllYTJmYzg0ZWI0YTkxNTNjZjk4NzY4OTFkYik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9iYzI1YjM4YzJhMWI0NDEyOWMzOWVmY2M4M2Y5ZTdhNSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjUwMTA0MDgsMTE0LjEzMTMxNTJdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fYTQ1YTZkYjhiZmZlNDA5M2FmN2Y0Y2QzMTgzMGMxZTMgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2xpZ2h0cmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyX2JjMjViMzhjMmExYjQ0MTI5YzM5ZWZjYzgzZjllN2E1LnNldEljb24oaWNvbl9hNDVhNmRiOGJmZmU0MDkzYWY3ZjRjZDMxODMwYzFlMyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF84Y2ViNDhkNTRkN2Y0NzhkYTUxMjliYjIwNjE2OTg3MCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF85NjFjODU1ZDRjZGQ0YjMwOTMyMzQxODU3ZmRkNmZiNyA9ICQoJzxkaXYgaWQ9Imh0bWxfOTYxYzg1NWQ0Y2RkNGIzMDkzMjM0MTg1N2ZkZDZmYjciIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjExOTc5LCBMQUJFTDo2LCBTRUdNRU5UOldUQyAtIFRvdXJpc3RzL0NvbnY8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzhjZWI0OGQ1NGQ3ZjQ3OGRhNTEyOWJiMjA2MTY5ODcwLnNldENvbnRlbnQoaHRtbF85NjFjODU1ZDRjZGQ0YjMwOTMyMzQxODU3ZmRkNmZiNyk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyX2JjMjViMzhjMmExYjQ0MTI5YzM5ZWZjYzgzZjllN2E1LmJpbmRQb3B1cChwb3B1cF84Y2ViNDhkNTRkN2Y0NzhkYTUxMjliYjIwNjE2OTg3MCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl83NTNmNDE2NmUwOWI0MDc1YTVmZmFhMDMxNWZlNGZiYSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjMyMTcyMTcsMTE0LjIxNjE5MDldLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fMTNlNzJlMGQxOTM4NDBmMTg3YzU2MTBjZDE2NDVjMTAgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3JlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl83NTNmNDE2NmUwOWI0MDc1YTVmZmFhMDMxNWZlNGZiYS5zZXRJY29uKGljb25fMTNlNzJlMGQxOTM4NDBmMTg3YzU2MTBjZDE2NDVjMTApOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNjQ4YWY3NTFjOGIwNDU5OWIyNGJkZjQ4NTExNjcxYWYgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfZGJmMTVmNmY5ZjVjNDYxMWI2ZmUyN2RiMTI4NjQwMmUgPSAkKCc8ZGl2IGlkPSJodG1sX2RiZjE1ZjZmOWY1YzQ2MTFiNmZlMjdkYjEyODY0MDJlIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjA1MCwgTEFCRUw6MCwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNjQ4YWY3NTFjOGIwNDU5OWIyNGJkZjQ4NTExNjcxYWYuc2V0Q29udGVudChodG1sX2RiZjE1ZjZmOWY1YzQ2MTFiNmZlMjdkYjEyODY0MDJlKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfNzUzZjQxNjZlMDliNDA3NWE1ZmZhYTAzMTVmZTRmYmEuYmluZFBvcHVwKHBvcHVwXzY0OGFmNzUxYzhiMDQ1OTliMjRiZGY0ODUxMTY3MWFmKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2Y2Y2MxODFjZDZkZDQxYjViMTk5ZDhlMjE1ZTkyMWQ2ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzc1OTQxLDExNC4xNzg2MjZdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fMTYyOWZiOWU3NTFkNDQyY2JlMWMxZGI2Y2U0NTFhOWUgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3JlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl9mNmNjMTgxY2Q2ZGQ0MWI1YjE5OWQ4ZTIxNWU5MjFkNi5zZXRJY29uKGljb25fMTYyOWZiOWU3NTFkNDQyY2JlMWMxZGI2Y2U0NTFhOWUpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfMGY1NzY3YTUzZTFhNDMzZWIxZTY3NDhlZjM4MTFlZTkgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfZWVjMTY4YTZkMzI5NDQ4ZGJlZmJhYTk1ZGYyN2Y3M2QgPSAkKCc8ZGl2IGlkPSJodG1sX2VlYzE2OGE2ZDMyOTQ0OGRiZWZiYWE5NWRmMjdmNzNkIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjMwNywgTEFCRUw6MCwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMGY1NzY3YTUzZTFhNDMzZWIxZTY3NDhlZjM4MTFlZTkuc2V0Q29udGVudChodG1sX2VlYzE2OGE2ZDMyOTQ0OGRiZWZiYWE5NWRmMjdmNzNkKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfZjZjYzE4MWNkNmRkNDFiNWIxOTlkOGUyMTVlOTIxZDYuYmluZFBvcHVwKHBvcHVwXzBmNTc2N2E1M2UxYTQzM2ViMWU2NzQ4ZWYzODExZWU5KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzAzZTgxMzg1YmQ1NDRkYjc5NDgxNTc4Nzg1YjA2ZGNiID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzI5Njg2MywxMTQuMTg3NTkyN10sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl83NzZjNTFkYjg4YjA0OWFhYmQ5ZjVmNjZkZDg4Y2NmMiA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAncmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzAzZTgxMzg1YmQ1NDRkYjc5NDgxNTc4Nzg1YjA2ZGNiLnNldEljb24oaWNvbl83NzZjNTFkYjg4YjA0OWFhYmQ5ZjVmNjZkZDg4Y2NmMik7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8zODA4ZTFmZjVlMjQ0YjFiOGU1Mzk1NjBhNjIyMGY4OSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF82ZGZlODc0NjVkMTQ0ZDI4YmJiN2NhYjcwMzAzOWU4MyA9ICQoJzxkaXYgaWQ9Imh0bWxfNmRmZTg3NDY1ZDE0NGQyOGJiYjdjYWI3MDMwMzllODMiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMzczLCBMQUJFTDowLCBTRUdNRU5UOldUQyAtIFJlc2lkZW50aWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8zODA4ZTFmZjVlMjQ0YjFiOGU1Mzk1NjBhNjIyMGY4OS5zZXRDb250ZW50KGh0bWxfNmRmZTg3NDY1ZDE0NGQyOGJiYjdjYWI3MDMwMzllODMpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl8wM2U4MTM4NWJkNTQ0ZGI3OTQ4MTU3ODc4NWIwNmRjYi5iaW5kUG9wdXAocG9wdXBfMzgwOGUxZmY1ZTI0NGIxYjhlNTM5NTYwYTYyMjBmODkpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfYTlmYmFkYzZhZjRiNDBkY2I3NzQ4YTllMDBmYmI5NDEgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4zNzQxNTE1LDExNC4xMTUyMjk4XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzYwMDgyMDcxMzQzMzQzOTViOTk4Mzk2ZGE1MTk3MTkxID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdncmVlbicsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl9hOWZiYWRjNmFmNGI0MGRjYjc3NDhhOWUwMGZiYjk0MS5zZXRJY29uKGljb25fNjAwODIwNzEzNDMzNDM5NWI5OTgzOTZkYTUxOTcxOTEpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfZGFiOTdmYzYzZmRlNGVjZDk1YTFjMWM0MDMwMTEzNDggPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfMzJlMGRhMmVjZTA2NDM1N2I3MWY1MzFhZTcwMjA3YmEgPSAkKCc8ZGl2IGlkPSJodG1sXzMyZTBkYTJlY2UwNjQzNTdiNzFmNTMxYWU3MDIwN2JhIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMzI0MSwgTEFCRUw6MiwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfZGFiOTdmYzYzZmRlNGVjZDk1YTFjMWM0MDMwMTEzNDguc2V0Q29udGVudChodG1sXzMyZTBkYTJlY2UwNjQzNTdiNzFmNTMxYWU3MDIwN2JhKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfYTlmYmFkYzZhZjRiNDBkY2I3NzQ4YTllMDBmYmI5NDEuYmluZFBvcHVwKHBvcHVwX2RhYjk3ZmM2M2ZkZTRlY2Q5NWExYzFjNDAzMDExMzQ4KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzU5ODZjYTkzM2NjNDRlMTU4MGMxOTFlODE1NzNmNjBkID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzcwNzM1LDExNC4xMTEzMDldLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fYmQxYWI3OTdjODBhNDAxY2FkZmJjNzk2MzlhNWRjODcgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3JlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl81OTg2Y2E5MzNjYzQ0ZTE1ODBjMTkxZTgxNTczZjYwZC5zZXRJY29uKGljb25fYmQxYWI3OTdjODBhNDAxY2FkZmJjNzk2MzlhNWRjODcpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfYThhNWYxZmFhZThkNDA4Y2JmZDRjNTExODIzZmI5NDEgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfMTBhMDA4YjhjOWFiNGNkNjgzOGVkY2VlYWZkNDUxYzAgPSAkKCc8ZGl2IGlkPSJodG1sXzEwYTAwOGI4YzlhYjRjZDY4MzhlZGNlZWFmZDQ1MWMwIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMTc0OSwgTEFCRUw6MCwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfYThhNWYxZmFhZThkNDA4Y2JmZDRjNTExODIzZmI5NDEuc2V0Q29udGVudChodG1sXzEwYTAwOGI4YzlhYjRjZDY4MzhlZGNlZWFmZDQ1MWMwKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfNTk4NmNhOTMzY2M0NGUxNTgwYzE5MWU4MTU3M2Y2MGQuYmluZFBvcHVwKHBvcHVwX2E4YTVmMWZhYWU4ZDQwOGNiZmQ0YzUxMTgyM2ZiOTQxKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzRmYWU1ZGQwMzEyYjRhYTdiY2ZjNTkyN2FhYmUyZWY1ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzI1NjYwNSwxMTQuMTc3OTM1OF0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl8zMTdlNGE0ZDc0OGQ0N2I1YjAyOWY3MGNjYzcyZjlhMyA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnbGlnaHRyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfNGZhZTVkZDAzMTJiNGFhN2JjZmM1OTI3YWFiZTJlZjUuc2V0SWNvbihpY29uXzMxN2U0YTRkNzQ4ZDQ3YjViMDI5ZjcwY2NjNzJmOWEzKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzBmNTdmYjQ0YmE2OTQzZDI4ZDg0NmNiYThkYzYzM2M0ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzhiMjg2MzVkOTI5MzRlYzNiY2JmMDUyZDQ3NzI2Yzg0ID0gJCgnPGRpdiBpZD0iaHRtbF84YjI4NjM1ZDkyOTM0ZWMzYmNiZjA1MmQ0NzcyNmM4NCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTE4NDgsIExBQkVMOjYsIFNFR01FTlQ6V1RDIC0gVG91cmlzdHMvQ29udjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMGY1N2ZiNDRiYTY5NDNkMjhkODQ2Y2JhOGRjNjMzYzQuc2V0Q29udGVudChodG1sXzhiMjg2MzVkOTI5MzRlYzNiY2JmMDUyZDQ3NzI2Yzg0KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfNGZhZTVkZDAzMTJiNGFhN2JjZmM1OTI3YWFiZTJlZjUuYmluZFBvcHVwKHBvcHVwXzBmNTdmYjQ0YmE2OTQzZDI4ZDg0NmNiYThkYzYzM2M0KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzBkNTZkNTI4NjYzMzQ3Y2M5N2ZiZTQ4ZTQ3YzQwNzQ4ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzgyNTM2MywxMTQuMTkwMzM1N10sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl85NGZlMDMxODJhZDU0MGI1OWUwOGQ1NjRhZDZlNTFjNyA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnb3JhbmdlJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzBkNTZkNTI4NjYzMzQ3Y2M5N2ZiZTQ4ZTQ3YzQwNzQ4LnNldEljb24oaWNvbl85NGZlMDMxODJhZDU0MGI1OWUwOGQ1NjRhZDZlNTFjNyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF85ZTQzY2FkYjBlNDg0NjMzODJiYzQzY2YxZmU0ZjZjMCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF85YjFiZWYzN2FmNGM0ZGJhYmRiYWI3NjBlNzE0ZWU1NiA9ICQoJzxkaXYgaWQ9Imh0bWxfOWIxYmVmMzdhZjRjNGRiYWJkYmFiNzYwZTcxNGVlNTYiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjExOTYyLCBMQUJFTDo0LCBTRUdNRU5UOldUQyAtIEJhYnk8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzllNDNjYWRiMGU0ODQ2MzM4MmJjNDNjZjFmZTRmNmMwLnNldENvbnRlbnQoaHRtbF85YjFiZWYzN2FmNGM0ZGJhYmRiYWI3NjBlNzE0ZWU1Nik7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzBkNTZkNTI4NjYzMzQ3Y2M5N2ZiZTQ4ZTQ3YzQwNzQ4LmJpbmRQb3B1cChwb3B1cF85ZTQzY2FkYjBlNDg0NjMzODJiYzQzY2YxZmU0ZjZjMCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9iNWE1NjYwYWQ0ZTg0ZDdlOTM0MWY3Y2UxNmJjM2ZiYyA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjQwNzMxNDYsMTEzLjk2MjgxMTFdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fNjIwYjc3Y2M0NGY3NDE1Mjk3NGRmYzZjYTg3N2MzNzMgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3JlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl9iNWE1NjYwYWQ0ZTg0ZDdlOTM0MWY3Y2UxNmJjM2ZiYy5zZXRJY29uKGljb25fNjIwYjc3Y2M0NGY3NDE1Mjk3NGRmYzZjYTg3N2MzNzMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNjIxMGJhZDU4YzBiNDZmOTgwZDNiMzI1MjMzZjdkZjggPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfYjcyN2Q0OGVkZTcxNDE2Nzg3YWFhZWZmNTNhNDlhODggPSAkKCc8ZGl2IGlkPSJodG1sX2I3MjdkNDhlZGU3MTQxNjc4N2FhYWVmZjUzYTQ5YTg4IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjA2MiwgTEFCRUw6MCwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNjIxMGJhZDU4YzBiNDZmOTgwZDNiMzI1MjMzZjdkZjguc2V0Q29udGVudChodG1sX2I3MjdkNDhlZGU3MTQxNjc4N2FhYWVmZjUzYTQ5YTg4KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfYjVhNTY2MGFkNGU4NGQ3ZTkzNDFmN2NlMTZiYzNmYmMuYmluZFBvcHVwKHBvcHVwXzYyMTBiYWQ1OGMwYjQ2Zjk4MGQzYjMyNTIzM2Y3ZGY4KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzJlMTlmOGI5NWJhNjRlOGZhMzk1MmJmMzljOWM3MTdhID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzI4OTk3NSwxMTQuMTg3MDA1XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzRmMDk4NGVmY2E5YjQ2MDVhMGI3OWQ3NTc2ZTc4M2NhID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdncmVlbicsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl8yZTE5ZjhiOTViYTY0ZThmYTM5NTJiZjM5YzljNzE3YS5zZXRJY29uKGljb25fNGYwOTg0ZWZjYTliNDYwNWEwYjc5ZDc1NzZlNzgzY2EpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfZDU2NDY0ODZkZTUwNDBmMTliZGY5MzgzNjZjMDNiNjQgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNmVlMzhmYzVmNTM4NDRhZDliNWE1OWQzNWQwZGUwYjUgPSAkKCc8ZGl2IGlkPSJodG1sXzZlZTM4ZmM1ZjUzODQ0YWQ5YjVhNTlkMzVkMGRlMGI1IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjEwNiwgTEFCRUw6MiwgU0VHTUVOVDpXVEMgLSBTYXZlcnM8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2Q1NjQ2NDg2ZGU1MDQwZjE5YmRmOTM4MzY2YzAzYjY0LnNldENvbnRlbnQoaHRtbF82ZWUzOGZjNWY1Mzg0NGFkOWI1YTU5ZDM1ZDBkZTBiNSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzJlMTlmOGI5NWJhNjRlOGZhMzk1MmJmMzljOWM3MTdhLmJpbmRQb3B1cChwb3B1cF9kNTY0NjQ4NmRlNTA0MGYxOWJkZjkzODM2NmMwM2I2NCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9jM2E0MWYzOWJiYzE0MGNhYWMxN2I3YTI3MjEwYzI5MCA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjQ1MDY2ODcsMTE0LjE2MDczMTVdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fNTJkYmQ4NDc0YzMyNDUxOGI1OGY5MzVmNjcyZjJjZjAgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3JlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl9jM2E0MWYzOWJiYzE0MGNhYWMxN2I3YTI3MjEwYzI5MC5zZXRJY29uKGljb25fNTJkYmQ4NDc0YzMyNDUxOGI1OGY5MzVmNjcyZjJjZjApOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfZjQ3Zjk0OWZjZjMyNGM0MmJmMTI3ZDI0Zjg1ZmQxOTIgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfYjA5MjJjZGQ0MjJhNDMyZmJlNjdlNjgzYjRiMTk4MmUgPSAkKCc8ZGl2IGlkPSJodG1sX2IwOTIyY2RkNDIyYTQzMmZiZTY3ZTY4M2I0YjE5ODJlIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMTcwNCwgTEFCRUw6MCwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfZjQ3Zjk0OWZjZjMyNGM0MmJmMTI3ZDI0Zjg1ZmQxOTIuc2V0Q29udGVudChodG1sX2IwOTIyY2RkNDIyYTQzMmZiZTY3ZTY4M2I0YjE5ODJlKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfYzNhNDFmMzliYmMxNDBjYWFjMTdiN2EyNzIxMGMyOTAuYmluZFBvcHVwKHBvcHVwX2Y0N2Y5NDlmY2YzMjRjNDJiZjEyN2QyNGY4NWZkMTkyKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2UzMDJlMzJiOGJjYjQxMTliM2U2ODYyMzMxZmY4ZTkyID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzMyNDkzOSwxMTQuMTYwMTEzXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2IzZGFmMTAzMTEyNTRkZTY4YzlmNGE2Y2FlODk0YWViID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfZTMwMmUzMmI4YmNiNDExOWIzZTY4NjIzMzFmZjhlOTIuc2V0SWNvbihpY29uX2IzZGFmMTAzMTEyNTRkZTY4YzlmNGE2Y2FlODk0YWViKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2NhNTk4OTc3ZDcxZjQ5NjE5YzhhMDc4YzYzYWY4N2YxID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzFmMTU5YzQzYjkwMjRhZGE4YWE0Mjk4ZGM4OWI5MTI3ID0gJCgnPGRpdiBpZD0iaHRtbF8xZjE1OWM0M2I5MDI0YWRhOGFhNDI5OGRjODliOTEyNyIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIyNzUsIExBQkVMOjAsIFNFR01FTlQ6V1RDIC0gU2F2ZXJzPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9jYTU5ODk3N2Q3MWY0OTYxOWM4YTA3OGM2M2FmODdmMS5zZXRDb250ZW50KGh0bWxfMWYxNTljNDNiOTAyNGFkYThhYTQyOThkYzg5YjkxMjcpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl9lMzAyZTMyYjhiY2I0MTE5YjNlNjg2MjMzMWZmOGU5Mi5iaW5kUG9wdXAocG9wdXBfY2E1OTg5NzdkNzFmNDk2MTljOGEwNzhjNjNhZjg3ZjEpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfMzYwMzE2YmExNzYwNDIwYjgzZjZkNGJlNWZlZjA4OTkgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4yOTY0MjIxLDExNC4xNzExNDNdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fNjRiOTQwZDE2ZDA0NDU4NjgzYjFmYTNlMjk0NzkwZjcgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ29yYW5nZScsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl8zNjAzMTZiYTE3NjA0MjBiODNmNmQ0YmU1ZmVmMDg5OS5zZXRJY29uKGljb25fNjRiOTQwZDE2ZDA0NDU4NjgzYjFmYTNlMjk0NzkwZjcpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNWE1NTIzZDUyNTg3NGY1MjhiYzBlODk1ODk0ZDJjODYgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfZTU2ZjYzYTJiNzc1NDg3MTgzYWZkNmY0MTVhZDc0MWIgPSAkKCc8ZGl2IGlkPSJodG1sX2U1NmY2M2EyYjc3NTQ4NzE4M2FmZDZmNDE1YWQ3NDFiIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjMyNywgTEFCRUw6NCwgU0VHTUVOVDpXVEMgLSBEZXN0aW5hdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNWE1NTIzZDUyNTg3NGY1MjhiYzBlODk1ODk0ZDJjODYuc2V0Q29udGVudChodG1sX2U1NmY2M2EyYjc3NTQ4NzE4M2FmZDZmNDE1YWQ3NDFiKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfMzYwMzE2YmExNzYwNDIwYjgzZjZkNGJlNWZlZjA4OTkuYmluZFBvcHVwKHBvcHVwXzVhNTUyM2Q1MjU4NzRmNTI4YmMwZTg5NTg5NGQyYzg2KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzY4Mjk5MzcwOTk1YTQ0MGZhZDBkM2Q2YjUzOTU3OGU0ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMjk2ODc0OCwxMTQuMjM5MTQyNl0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl82Y2UzNzgzYzI4YzE0MWQ2YWI1YmQ4NmUyM2MzN2EzMCA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnZ3JlZW4nLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfNjgyOTkzNzA5OTVhNDQwZmFkMGQzZDZiNTM5NTc4ZTQuc2V0SWNvbihpY29uXzZjZTM3ODNjMjhjMTQxZDZhYjViZDg2ZTIzYzM3YTMwKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2M5ZjRiYjIwYmJiMjRiZTViNTQ1MzI2Njg1MWFiOGQ0ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzVmMTAyMzljNzQ3MjQwMmViOWRjM2QwOGM5NTQzOTQ3ID0gJCgnPGRpdiBpZD0iaHRtbF81ZjEwMjM5Yzc0NzI0MDJlYjlkYzNkMDhjOTU0Mzk0NyIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIzODMsIExBQkVMOjIsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2M5ZjRiYjIwYmJiMjRiZTViNTQ1MzI2Njg1MWFiOGQ0LnNldENvbnRlbnQoaHRtbF81ZjEwMjM5Yzc0NzI0MDJlYjlkYzNkMDhjOTU0Mzk0Nyk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzY4Mjk5MzcwOTk1YTQ0MGZhZDBkM2Q2YjUzOTU3OGU0LmJpbmRQb3B1cChwb3B1cF9jOWY0YmIyMGJiYjI0YmU1YjU0NTMyNjY4NTFhYjhkNCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl81ZGVhZWQ5OTQyZTQ0ZmZhOWU3M2M4NjQyZDY2YmRjNSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjI4OTE2MywxMTMuOTQxMjY5XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzQ0OTNkY2ZkYzdjNzQ2MWE4OGMxYmRhZGFjMDg0YmNiID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfNWRlYWVkOTk0MmU0NGZmYTllNzNjODY0MmQ2NmJkYzUuc2V0SWNvbihpY29uXzQ0OTNkY2ZkYzdjNzQ2MWE4OGMxYmRhZGFjMDg0YmNiKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzdlMTczNjJhZThmYzQyNmNiYWY0NmE4NDhlMjhiZGRjID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzdlMzBiNTkwYTQ2MTRkMzI4Y2MxNWQxMmRjYWYxZTcxID0gJCgnPGRpdiBpZD0iaHRtbF83ZTMwYjU5MGE0NjE0ZDMyOGNjMTVkMTJkY2FmMWU3MSIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTI0MDMsIExBQkVMOjAsIFNFR01FTlQ6V1RDIC0gSGVhbHRoPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF83ZTE3MzYyYWU4ZmM0MjZjYmFmNDZhODQ4ZTI4YmRkYy5zZXRDb250ZW50KGh0bWxfN2UzMGI1OTBhNDYxNGQzMjhjYzE1ZDEyZGNhZjFlNzEpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl81ZGVhZWQ5OTQyZTQ0ZmZhOWU3M2M4NjQyZDY2YmRjNS5iaW5kUG9wdXAocG9wdXBfN2UxNzM2MmFlOGZjNDI2Y2JhZjQ2YTg0OGUyOGJkZGMpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfOWRkZWNhYjE1OTIwNDNmOGFiN2I0YjE1YWUwMjgwODMgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4zMzcyODAyLDExNC4xNTY0NTczXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzE0YjEwOTZmYTY0YjRkMmViMmUxMTEyZGJjYmIzZWM3ID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfOWRkZWNhYjE1OTIwNDNmOGFiN2I0YjE1YWUwMjgwODMuc2V0SWNvbihpY29uXzE0YjEwOTZmYTY0YjRkMmViMmUxMTEyZGJjYmIzZWM3KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzUwNmQ1ODIwNTcyMjRiMGE5N2ZjMmJlMzBjYzI1MDYzID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2I0YzAzYmU4ZjRhZDQxZDJhMTM5NTkzMTg4M2I5M2ZlID0gJCgnPGRpdiBpZD0iaHRtbF9iNGMwM2JlOGY0YWQ0MWQyYTEzOTU5MzE4ODNiOTNmZSIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTI2MTIsIExBQkVMOjAsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzUwNmQ1ODIwNTcyMjRiMGE5N2ZjMmJlMzBjYzI1MDYzLnNldENvbnRlbnQoaHRtbF9iNGMwM2JlOGY0YWQ0MWQyYTEzOTU5MzE4ODNiOTNmZSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzlkZGVjYWIxNTkyMDQzZjhhYjdiNGIxNWFlMDI4MDgzLmJpbmRQb3B1cChwb3B1cF81MDZkNTgyMDU3MjI0YjBhOTdmYzJiZTMwY2MyNTA2Myk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl8zNTA1NDM5OWRlMWY0ZWFjYjU2YzliYjc1N2ExMWQ3YiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjI3ODYxNjYsMTE0LjE3ODE4OTFdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fNGJhZWM3MDc3N2VjNDYwY2FkZjllNGFhYjllNDk2Y2QgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2xpZ2h0cmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzM1MDU0Mzk5ZGUxZjRlYWNiNTZjOWJiNzU3YTExZDdiLnNldEljb24oaWNvbl80YmFlYzcwNzc3ZWM0NjBjYWRmOWU0YWFiOWU0OTZjZCk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8xZTE1NTI0YjBmYzg0OTVmOGYzMzFjZDEzYzkwZGJjNSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF8xYzUzZDBmNTc5NjE0ZjA4ODM1Njg1N2RhNDlkMzdlZCA9ICQoJzxkaXYgaWQ9Imh0bWxfMWM1M2QwZjU3OTYxNGYwODgzNTY4NTdkYTQ5ZDM3ZWQiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMTc4LCBMQUJFTDo2LCBTRUdNRU5UOldUQyAtIFRvdXJpc3RzL0NvbnY8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzFlMTU1MjRiMGZjODQ5NWY4ZjMzMWNkMTNjOTBkYmM1LnNldENvbnRlbnQoaHRtbF8xYzUzZDBmNTc5NjE0ZjA4ODM1Njg1N2RhNDlkMzdlZCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzM1MDU0Mzk5ZGUxZjRlYWNiNTZjOWJiNzU3YTExZDdiLmJpbmRQb3B1cChwb3B1cF8xZTE1NTI0YjBmYzg0OTVmOGYzMzFjZDEzYzkwZGJjNSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9iNDIyOGY2YzgzNDM0MDhiYjI0Y2RhODNjN2U2ZTMwMCA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjMyODY5NywxMTQuMTY0NTQ2XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzg4NGE0M2MzMWY3OTRiNTQ5YzE2NDcxNDE3NDk3ZWQ1ID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfYjQyMjhmNmM4MzQzNDA4YmIyNGNkYTgzYzdlNmUzMDAuc2V0SWNvbihpY29uXzg4NGE0M2MzMWY3OTRiNTQ5YzE2NDcxNDE3NDk3ZWQ1KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzFmNjE0MDlhZGU1NzRjMGE4MTdjMDJlZTgwNzE1YTY2ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzAwODlmOTdmODg0ZDQ0MGRiMjlhZDIwZDliZjg5ZTIyID0gJCgnPGRpdiBpZD0iaHRtbF8wMDg5Zjk3Zjg4NGQ0NDBkYjI5YWQyMGQ5YmY4OWUyMiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIxODgsIExBQkVMOjAsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzFmNjE0MDlhZGU1NzRjMGE4MTdjMDJlZTgwNzE1YTY2LnNldENvbnRlbnQoaHRtbF8wMDg5Zjk3Zjg4NGQ0NDBkYjI5YWQyMGQ5YmY4OWUyMik7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyX2I0MjI4ZjZjODM0MzQwOGJiMjRjZGE4M2M3ZTZlMzAwLmJpbmRQb3B1cChwb3B1cF8xZjYxNDA5YWRlNTc0YzBhODE3YzAyZWU4MDcxNWE2Nik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9lZDRhZjM4MjI1OWM0ZDYzYjg0OWFjNDRjOWMzYmVlMiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjI5MjEwMzUsMTE0LjE5NjMxNjVdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fZDNlNDdlNGQzYTgzNGNkNGJkYWE3ODUzMWMxZTAxYjAgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3JlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl9lZDRhZjM4MjI1OWM0ZDYzYjg0OWFjNDRjOWMzYmVlMi5zZXRJY29uKGljb25fZDNlNDdlNGQzYTgzNGNkNGJkYWE3ODUzMWMxZTAxYjApOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfMDg5NjczYjE5YWU5NDU4NTkwOWNkNTQ3MzkyMGJkOTIgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfODljOTlkNmEzZTgxNDA0YjhmMTE3NGZlYTRkNmQ2MWIgPSAkKCc8ZGl2IGlkPSJodG1sXzg5Yzk5ZDZhM2U4MTQwNGI4ZjExNzRmZWE0ZDZkNjFiIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjE5NCwgTEFCRUw6MCwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMDg5NjczYjE5YWU5NDU4NTkwOWNkNTQ3MzkyMGJkOTIuc2V0Q29udGVudChodG1sXzg5Yzk5ZDZhM2U4MTQwNGI4ZjExNzRmZWE0ZDZkNjFiKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfZWQ0YWYzODIyNTljNGQ2M2I4NDlhYzQ0YzljM2JlZTIuYmluZFBvcHVwKHBvcHVwXzA4OTY3M2IxOWFlOTQ1ODU5MDljZDU0NzM5MjBiZDkyKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzY0ZjI3ZjhjZmY4YjRiZGRhNDk1MmNmNWE4OTdkNzgyID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzA4ODkyOSwxMTQuMjYyNDk2OV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl85NDBkOGFhYjg4ZGE0ZDlkYTU1ZjhiODg0ZGYxM2RmOCA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAncmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzY0ZjI3ZjhjZmY4YjRiZGRhNDk1MmNmNWE4OTdkNzgyLnNldEljb24oaWNvbl85NDBkOGFhYjg4ZGE0ZDlkYTU1ZjhiODg0ZGYxM2RmOCk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9jOGJiMDllMDUwNDI0YzYxOGNlMjg0MGQ1MjQ2ZWQwOSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9iYTI1ZDI2NzQ4ZGU0N2Q5YmI1ZjJkYWFkM2Y0NmYxNiA9ICQoJzxkaXYgaWQ9Imh0bWxfYmEyNWQyNjc0OGRlNDdkOWJiNWYyZGFhZDNmNDZmMTYiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMzg1LCBMQUJFTDowLCBTRUdNRU5UOldUQyAtIFJlc2lkZW50aWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9jOGJiMDllMDUwNDI0YzYxOGNlMjg0MGQ1MjQ2ZWQwOS5zZXRDb250ZW50KGh0bWxfYmEyNWQyNjc0OGRlNDdkOWJiNWYyZGFhZDNmNDZmMTYpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl82NGYyN2Y4Y2ZmOGI0YmRkYTQ5NTJjZjVhODk3ZDc4Mi5iaW5kUG9wdXAocG9wdXBfYzhiYjA5ZTA1MDQyNGM2MThjZTI4NDBkNTI0NmVkMDkpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfODM0NjdkYTZlYzdlNDBjY2I3MWIzNWFkNzM4NmE5OTUgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi40NDQ5MjYzLDExNC4wMjM4MjIyXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzU2ZjY4YmZmNGMwZjRmMmE5YWYwOGNiNWNiMTRmZTA0ID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfODM0NjdkYTZlYzdlNDBjY2I3MWIzNWFkNzM4NmE5OTUuc2V0SWNvbihpY29uXzU2ZjY4YmZmNGMwZjRmMmE5YWYwOGNiNWNiMTRmZTA0KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2QxNjk1YzRkMDNiMTRhNjQ5YzUxNjEyOTc4NDBmYTZlID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzYyMGI3ZDc0MTFhMTQwZmNhOTU3OTgyZTMxZDg2MzYwID0gJCgnPGRpdiBpZD0iaHRtbF82MjBiN2Q3NDExYTE0MGZjYTk1Nzk4MmUzMWQ4NjM2MCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTI0NDEsIExBQkVMOjAsIFNFR01FTlQ6V1RDIC0gVG91cmlzdHMvQ29udjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfZDE2OTVjNGQwM2IxNGE2NDljNTE2MTI5Nzg0MGZhNmUuc2V0Q29udGVudChodG1sXzYyMGI3ZDc0MTFhMTQwZmNhOTU3OTgyZTMxZDg2MzYwKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfODM0NjdkYTZlYzdlNDBjY2I3MWIzNWFkNzM4NmE5OTUuYmluZFBvcHVwKHBvcHVwX2QxNjk1YzRkMDNiMTRhNjQ5YzUxNjEyOTc4NDBmYTZlKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzk5MmM2MjJiOTU2ZTQzM2NhMjZjNmYxNjU3YjA1MGE4ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuNDUxMjUyOCwxMTQuMDAxMjc3Nl0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9mNzhmNWFiOGQ0NTg0ZGU2YWEwOTIxNjU3Y2Q0OTFmNSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAncmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzk5MmM2MjJiOTU2ZTQzM2NhMjZjNmYxNjU3YjA1MGE4LnNldEljb24oaWNvbl9mNzhmNWFiOGQ0NTg0ZGU2YWEwOTIxNjU3Y2Q0OTFmNSk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF80NmRjMGZiY2IzODA0Zjk4OTFkN2JmNjhkNmZiMjgzZSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9mZmI4NDkzOGYwNzc0NDcyYjVkYjA5Yjc3ZDFlNWQwMCA9ICQoJzxkaXYgaWQ9Imh0bWxfZmZiODQ5MzhmMDc3NDQ3MmI1ZGIwOWI3N2QxZTVkMDAiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyNTI5LCBMQUJFTDowLCBTRUdNRU5UOldUQyAtIFJlc2lkZW50aWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF80NmRjMGZiY2IzODA0Zjk4OTFkN2JmNjhkNmZiMjgzZS5zZXRDb250ZW50KGh0bWxfZmZiODQ5MzhmMDc3NDQ3MmI1ZGIwOWI3N2QxZTVkMDApOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl85OTJjNjIyYjk1NmU0MzNjYTI2YzZmMTY1N2IwNTBhOC5iaW5kUG9wdXAocG9wdXBfNDZkYzBmYmNiMzgwNGY5ODkxZDdiZjY4ZDZmYjI4M2UpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfNDg2MDM0YjNjZDMzNDI5MjhjZDBjZGI3YmYxNGI0ZjYgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4zNDYwMjIsMTE0LjE5NzY1XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzcxMWEzNDAzZjk3MjQ2ZDhiN2M1ZDBmMzA1MzFhYjI5ID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfNDg2MDM0YjNjZDMzNDI5MjhjZDBjZGI3YmYxNGI0ZjYuc2V0SWNvbihpY29uXzcxMWEzNDAzZjk3MjQ2ZDhiN2M1ZDBmMzA1MzFhYjI5KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzIyYWUwNDRkODU3ODQ1NWFhMTFiYjFmZjY4OWExYjAxID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzE1NWIxODNiZmMxOTQ0MWFiMTg4ODk2ODA5MjZhNmQ0ID0gJCgnPGRpdiBpZD0iaHRtbF8xNTViMTgzYmZjMTk0NDFhYjE4ODg5NjgwOTI2YTZkNCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTE3NDEsIExBQkVMOjAsIFNFR01FTlQ6V1RDIC0gU2F2ZXJzPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8yMmFlMDQ0ZDg1Nzg0NTVhYTExYmIxZmY2ODlhMWIwMS5zZXRDb250ZW50KGh0bWxfMTU1YjE4M2JmYzE5NDQxYWIxODg4OTY4MDkyNmE2ZDQpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl80ODYwMzRiM2NkMzM0MjkyOGNkMGNkYjdiZjE0YjRmNi5iaW5kUG9wdXAocG9wdXBfMjJhZTA0NGQ4NTc4NDU1YWExMWJiMWZmNjg5YTFiMDEpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfOGViODQ4NjU5MTkzNDUyMjkzNzlhYTY5OTUzNGI2ZGMgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4zNzkyMDA0LDExNC4xMzU3NjA1XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2IxMjlhNmYzMmRmMjQ0Zjk5NTkzZWZhYTRkZDkzZWQzID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdiZWlnZScsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl84ZWI4NDg2NTkxOTM0NTIyOTM3OWFhNjk5NTM0YjZkYy5zZXRJY29uKGljb25fYjEyOWE2ZjMyZGYyNDRmOTk1OTNlZmFhNGRkOTNlZDMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfN2JjNTZlOGM1Y2VkNGJhN2I2OWJlOGIyMjhlMDdjZmYgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfMGE2MTMyZGJhM2FjNDE3NTg0NmYyNmM0NDFkZDVjZDcgPSAkKCc8ZGl2IGlkPSJodG1sXzBhNjEzMmRiYTNhYzQxNzU4NDZmMjZjNDQxZGQ1Y2Q3IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMTc2MSwgTEFCRUw6NywgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfN2JjNTZlOGM1Y2VkNGJhN2I2OWJlOGIyMjhlMDdjZmYuc2V0Q29udGVudChodG1sXzBhNjEzMmRiYTNhYzQxNzU4NDZmMjZjNDQxZGQ1Y2Q3KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfOGViODQ4NjU5MTkzNDUyMjkzNzlhYTY5OTUzNGI2ZGMuYmluZFBvcHVwKHBvcHVwXzdiYzU2ZThjNWNlZDRiYTdiNjliZThiMjI4ZTA3Y2ZmKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzA2YzU1NDI3ODQ4YTQwNGU5MTcxMWU2Nzk2MzgxNzNkID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzgxMTg5LDExNC4xOTUyNDldLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fNTgxYzUwMThiNjg5NDBlZGFlMDk1MjYxNTI3ODIwZjMgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2xpZ2h0cmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzA2YzU1NDI3ODQ4YTQwNGU5MTcxMWU2Nzk2MzgxNzNkLnNldEljb24oaWNvbl81ODFjNTAxOGI2ODk0MGVkYWUwOTUyNjE1Mjc4MjBmMyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9mMDM3NTI1ZDUxZDc0MzFhOGNkZjQ4ZDJjNmY3N2VkMiA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9mYzMyM2EyNGRmMTM0NGRhODE0ZTM2NDY2YjQzNDMzOCA9ICQoJzxkaXYgaWQ9Imh0bWxfZmMzMjNhMjRkZjEzNDRkYTgxNGUzNjQ2NmI0MzQzMzgiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMDE3LCBMQUJFTDo2LCBTRUdNRU5UOldUQyAtIFJlc2lkZW50aWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9mMDM3NTI1ZDUxZDc0MzFhOGNkZjQ4ZDJjNmY3N2VkMi5zZXRDb250ZW50KGh0bWxfZmMzMjNhMjRkZjEzNDRkYTgxNGUzNjQ2NmI0MzQzMzgpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl8wNmM1NTQyNzg0OGE0MDRlOTE3MTFlNjc5NjM4MTczZC5iaW5kUG9wdXAocG9wdXBfZjAzNzUyNWQ1MWQ3NDMxYThjZGY0OGQyYzZmNzdlZDIpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfNjA5YTNlMjMzZDgzNDg5NTgwZDg1YTk5OTEwNzk5MjkgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4zMzEzMjUxLDExNC4xNTY5MDUxXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2ZkYjc1NTU4ZGQyZDQ4OTliMzg0YTRmMWFlMDMyYjA2ID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfNjA5YTNlMjMzZDgzNDg5NTgwZDg1YTk5OTEwNzk5Mjkuc2V0SWNvbihpY29uX2ZkYjc1NTU4ZGQyZDQ4OTliMzg0YTRmMWFlMDMyYjA2KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzZkMDRjMTgzYzIxNzQ2MGQ4MTQ3MWIyY2RmMmY1YzhjID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzdhMjA3OGFhMjJlMTQ4MTFhMTU0ZmUxZWM4OTJmOTBmID0gJCgnPGRpdiBpZD0iaHRtbF83YTIwNzhhYTIyZTE0ODExYTE1NGZlMWVjODkyZjkwZiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIxMDksIExBQkVMOjAsIFNFR01FTlQ6V1RDIC0gU2F2ZXJzPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF82ZDA0YzE4M2MyMTc0NjBkODE0NzFiMmNkZjJmNWM4Yy5zZXRDb250ZW50KGh0bWxfN2EyMDc4YWEyMmUxNDgxMWExNTRmZTFlYzg5MmY5MGYpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl82MDlhM2UyMzNkODM0ODk1ODBkODVhOTk5MTA3OTkyOS5iaW5kUG9wdXAocG9wdXBfNmQwNGMxODNjMjE3NDYwZDgxNDcxYjJjZGYyZjVjOGMpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfYzFmNGQzZjc5Nzk4NGVkOGIwNTA4YTFjMGQ5YWRhOTAgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4yODQ1Mzk2LDExNC4yMTcwNjc5XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzZmZDQ5MzFiYTdlMzQwMmI4NzM3NzlkZTNiNmZiNjQ5ID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdncmVlbicsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl9jMWY0ZDNmNzk3OTg0ZWQ4YjA1MDhhMWMwZDlhZGE5MC5zZXRJY29uKGljb25fNmZkNDkzMWJhN2UzNDAyYjg3Mzc3OWRlM2I2ZmI2NDkpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfYjIyN2JkOGY1NGQ2NDFlOGIxMzk0ZTgzOTEyYjNlYWIgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfYTg1YzJkOTljZDY2NGNhNGI5OTFiOGY2NGM3N2Q4NGIgPSAkKCc8ZGl2IGlkPSJodG1sX2E4NWMyZDk5Y2Q2NjRjYTRiOTkxYjhmNjRjNzdkODRiIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjM0NSwgTEFCRUw6MiwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfYjIyN2JkOGY1NGQ2NDFlOGIxMzk0ZTgzOTEyYjNlYWIuc2V0Q29udGVudChodG1sX2E4NWMyZDk5Y2Q2NjRjYTRiOTkxYjhmNjRjNzdkODRiKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfYzFmNGQzZjc5Nzk4NGVkOGIwNTA4YTFjMGQ5YWRhOTAuYmluZFBvcHVwKHBvcHVwX2IyMjdiZDhmNTRkNjQxZThiMTM5NGU4MzkxMmIzZWFiKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzUxZjZhNzBlNTgwNDQxMGQ4OTcwNDVjYTE3OTIyMjRlID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzQyMDMwMywxMTQuMTkyMzQ3NF0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl8yN2E4MGMxOTg3ZmI0NDBkODc0N2ZhYWViYzIxOTA0YyA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnZ3JlZW4nLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfNTFmNmE3MGU1ODA0NDEwZDg5NzA0NWNhMTc5MjIyNGUuc2V0SWNvbihpY29uXzI3YTgwYzE5ODdmYjQ0MGQ4NzQ3ZmFhZWJjMjE5MDRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzFlZDI0NzFlYjY4MzQyZjY4YmQzOWJiMDNmMDNlMzRjID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzM2MWMxYTZhN2I1ZDRlZGI4ZDc0YTNlMDljNDg5ZWY0ID0gJCgnPGRpdiBpZD0iaHRtbF8zNjFjMWE2YTdiNWQ0ZWRiOGQ3NGEzZTA5YzQ4OWVmNCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTMwMjEsIExBQkVMOjIsIFNFR01FTlQ6V1RDIC0gQmFieTwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMWVkMjQ3MWViNjgzNDJmNjhiZDM5YmIwM2YwM2UzNGMuc2V0Q29udGVudChodG1sXzM2MWMxYTZhN2I1ZDRlZGI4ZDc0YTNlMDljNDg5ZWY0KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfNTFmNmE3MGU1ODA0NDEwZDg5NzA0NWNhMTc5MjIyNGUuYmluZFBvcHVwKHBvcHVwXzFlZDI0NzFlYjY4MzQyZjY4YmQzOWJiMDNmMDNlMzRjKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzQzODU2NjVlYTZjMzQ4MjZiNmQ2ZWFkZGY1ODMxNmU0ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzA4NDU3MSwxMTQuMTg3ODcyNl0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl8zODlkMTY4ZGUzYjY0MGQ4YjE1ZjA3OTBjNTU1MWU0MSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAncmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzQzODU2NjVlYTZjMzQ4MjZiNmQ2ZWFkZGY1ODMxNmU0LnNldEljb24oaWNvbl8zODlkMTY4ZGUzYjY0MGQ4YjE1ZjA3OTBjNTU1MWU0MSk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF80MTI2MmQ3YjIxZjk0Y2QyODBhYzIwY2EzMjA1MDhkMiA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9iNjk4MTM2NzZhMDU0YzRiODEzYjQ5ZWM1Yjk3ZDQzYiA9ICQoJzxkaXYgaWQ9Imh0bWxfYjY5ODEzNjc2YTA1NGM0YjgxM2I0OWVjNWI5N2Q0M2IiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjExNzQyLCBMQUJFTDowLCBTRUdNRU5UOldUQyAtIENvbW1lcmNpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzQxMjYyZDdiMjFmOTRjZDI4MGFjMjBjYTMyMDUwOGQyLnNldENvbnRlbnQoaHRtbF9iNjk4MTM2NzZhMDU0YzRiODEzYjQ5ZWM1Yjk3ZDQzYik7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzQzODU2NjVlYTZjMzQ4MjZiNmQ2ZWFkZGY1ODMxNmU0LmJpbmRQb3B1cChwb3B1cF80MTI2MmQ3YjIxZjk0Y2QyODBhYzIwY2EzMjA1MDhkMik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9kYzM4NzVlZTcxMjI0MDMzYjdiOWZlYTJmYWE3NDU5OSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjIxMTYyNSwxMTMuNTUxODldLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fMDdhMjcwYjNiOGY1NDUwZmFmZGRlZjEyOTVmYzQwZTcgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2xpZ2h0cmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyX2RjMzg3NWVlNzEyMjQwMzNiN2I5ZmVhMmZhYTc0NTk5LnNldEljb24oaWNvbl8wN2EyNzBiM2I4ZjU0NTBmYWZkZGVmMTI5NWZjNDBlNyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9lZjMyOTQ0MWViMWY0ZTcyYjc3Y2JlOTRlNWQ0ZDI0OSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF8wMmI4ZmJiOTY3MDE0YjlkOGRlZWZhODg0MGQ1ODgyMCA9ICQoJzxkaXYgaWQ9Imh0bWxfMDJiOGZiYjk2NzAxNGI5ZDhkZWVmYTg4NDBkNTg4MjAiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjExODc5LCBMQUJFTDo2LCBTRUdNRU5UOldUQyAtIFJlc2lkZW50aWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9lZjMyOTQ0MWViMWY0ZTcyYjc3Y2JlOTRlNWQ0ZDI0OS5zZXRDb250ZW50KGh0bWxfMDJiOGZiYjk2NzAxNGI5ZDhkZWVmYTg4NDBkNTg4MjApOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl9kYzM4NzVlZTcxMjI0MDMzYjdiOWZlYTJmYWE3NDU5OS5iaW5kUG9wdXAocG9wdXBfZWYzMjk0NDFlYjFmNGU3MmI3N2NiZTk0ZTVkNGQyNDkpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfYzQzOGVlNWM0MzJlNGFiNDg1N2FkY2ExZWVjNDEwMWQgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4zOTMwODU1LDExMy45NzY0NDU5XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2Q2ZWY5NzgxMDhjNDRhNDA4N2ZkNTczMjMyMTFkOGZiID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdsaWdodHJlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl9jNDM4ZWU1YzQzMmU0YWI0ODU3YWRjYTFlZWM0MTAxZC5zZXRJY29uKGljb25fZDZlZjk3ODEwOGM0NGE0MDg3ZmQ1NzMyMzIxMWQ4ZmIpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNTE5ZDc1MDY0ZjcwNDUzZTlhMzQ4MzlhMWE3NDYyNzggPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfZTg2MGIyOWFiMjljNDU0Mzk0OGViMGE0M2Q5NzAwOWQgPSAkKCc8ZGl2IGlkPSJodG1sX2U4NjBiMjlhYjI5YzQ1NDM5NDhlYjBhNDNkOTcwMDlkIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMTg5OCwgTEFCRUw6NiwgU0VHTUVOVDpXVEMgLSBIZWFsdGg8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzUxOWQ3NTA2NGY3MDQ1M2U5YTM0ODM5YTFhNzQ2Mjc4LnNldENvbnRlbnQoaHRtbF9lODYwYjI5YWIyOWM0NTQzOTQ4ZWIwYTQzZDk3MDA5ZCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyX2M0MzhlZTVjNDMyZTRhYjQ4NTdhZGNhMWVlYzQxMDFkLmJpbmRQb3B1cChwb3B1cF81MTlkNzUwNjRmNzA0NTNlOWEzNDgzOWExYTc0NjI3OCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl82ZWMyODUzMWY1ZDg0ZWZhODNmNmVkMzFhYjMyMDliZSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjIwMjg0NTksMTEzLjU0NzE5ODNdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fZjJmZmY0OTAwYTk3NDFmNzlhODhkM2ZhZjlkZGE1ZGQgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2xpZ2h0cmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzZlYzI4NTMxZjVkODRlZmE4M2Y2ZWQzMWFiMzIwOWJlLnNldEljb24oaWNvbl9mMmZmZjQ5MDBhOTc0MWY3OWE4OGQzZmFmOWRkYTVkZCk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9iZjQxNzU4ODhiMDE0OTM4OThhMWM3YmY0ZjI2OTc3OSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9mY2U0NTZmMmI3Njk0ZGU4YjM0MGVhMTRmZjc4NmU5MiA9ICQoJzxkaXYgaWQ9Imh0bWxfZmNlNDU2ZjJiNzY5NGRlOGIzNDBlYTE0ZmY3ODZlOTIiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjExOTY2LCBMQUJFTDo2LCBTRUdNRU5UOldUQyAtIFJlc2lkZW50aWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9iZjQxNzU4ODhiMDE0OTM4OThhMWM3YmY0ZjI2OTc3OS5zZXRDb250ZW50KGh0bWxfZmNlNDU2ZjJiNzY5NGRlOGIzNDBlYTE0ZmY3ODZlOTIpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl82ZWMyODUzMWY1ZDg0ZWZhODNmNmVkMzFhYjMyMDliZS5iaW5kUG9wdXAocG9wdXBfYmY0MTc1ODg4YjAxNDkzODk4YTFjN2JmNGYyNjk3NzkpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfNGViNjc4NGRjZjYzNDdkM2IwMWZjYmMxYWZjOTJiOGQgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi41MTEyMDA4LDExNC4xMjYwOTkxXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2Q4ZWIxY2NmY2FmZjQwMDE4ZjY2NjYzMDIwZmU0NWVlID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdvcmFuZ2UnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfNGViNjc4NGRjZjYzNDdkM2IwMWZjYmMxYWZjOTJiOGQuc2V0SWNvbihpY29uX2Q4ZWIxY2NmY2FmZjQwMDE4ZjY2NjYzMDIwZmU0NWVlKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzVhNzdkMzNiNDA1MDQwNWVhN2EwMzhlMzYyZGIzZTgzID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzNlM2VlY2FiMTIyMDQ1YjU4ZTZkMjE0M2ExZjMyYWNmID0gJCgnPGRpdiBpZD0iaHRtbF8zZTNlZWNhYjEyMjA0NWI1OGU2ZDIxNDNhMWYzMmFjZiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTE5OTgsIExBQkVMOjQsIFNFR01FTlQ6V1RDIC0gVG91cmlzdHMvQ29udjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNWE3N2QzM2I0MDUwNDA1ZWE3YTAzOGUzNjJkYjNlODMuc2V0Q29udGVudChodG1sXzNlM2VlY2FiMTIyMDQ1YjU4ZTZkMjE0M2ExZjMyYWNmKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfNGViNjc4NGRjZjYzNDdkM2IwMWZjYmMxYWZjOTJiOGQuYmluZFBvcHVwKHBvcHVwXzVhNzdkMzNiNDA1MDQwNWVhN2EwMzhlMzYyZGIzZTgzKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzM4YjBiYmQzNmIwNDRhNWJhOTljMGRiM2E1MWE5YjMyID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzczODAxNCwxMTQuMTE2MzQ0N10sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl84NDk0OWFlNTcxNDM0N2I5YjIxNDFlNzI3YmFjZGMyZiA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAncmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzM4YjBiYmQzNmIwNDRhNWJhOTljMGRiM2E1MWE5YjMyLnNldEljb24oaWNvbl84NDk0OWFlNTcxNDM0N2I5YjIxNDFlNzI3YmFjZGMyZik7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9kNDhmMmQyM2I5YzQ0YzI0YmJlNzVkODE1ZjBmYjE1YyA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF80OTAwNWY0YTBjYzU0Y2Q3ODc3Y2JlOTA3ODJkNWQ3NiA9ICQoJzxkaXYgaWQ9Imh0bWxfNDkwMDVmNGEwY2M1NGNkNzg3N2NiZTkwNzgyZDVkNzYiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjExOTc2LCBMQUJFTDowLCBTRUdNRU5UOldUQyAtIFNhdmVyczwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfZDQ4ZjJkMjNiOWM0NGMyNGJiZTc1ZDgxNWYwZmIxNWMuc2V0Q29udGVudChodG1sXzQ5MDA1ZjRhMGNjNTRjZDc4NzdjYmU5MDc4MmQ1ZDc2KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfMzhiMGJiZDM2YjA0NGE1YmE5OWMwZGIzYTUxYTliMzIuYmluZFBvcHVwKHBvcHVwX2Q0OGYyZDIzYjljNDRjMjRiYmU3NWQ4MTVmMGZiMTVjKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzVjMjM5MDU3M2Y2NjQ3NGRiMDc3MjJkNzAwNzMzOWE4ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuNTE0NDY0LDExNC4wNjU2NDddLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fNTY0NzBkMTAxNzUwNGU4MmJkNDExZDk5Yzk5MTA0MTEgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2xpZ2h0cmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzVjMjM5MDU3M2Y2NjQ3NGRiMDc3MjJkNzAwNzMzOWE4LnNldEljb24oaWNvbl81NjQ3MGQxMDE3NTA0ZTgyYmQ0MTFkOTljOTkxMDQxMSk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF81YzA2YmYxZjEyYWI0MjRlYThkNjAxZTc3ODlkN2JhMCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF80ZDM3MGNmYWZmMjU0OWNhOTIwNzk5ZDEzZWQxODk1MyA9ICQoJzxkaXYgaWQ9Imh0bWxfNGQzNzBjZmFmZjI1NDljYTkyMDc5OWQxM2VkMTg5NTMiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjExOTgxLCBMQUJFTDo2LCBTRUdNRU5UOldUQyAtIEhlYWx0aDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNWMwNmJmMWYxMmFiNDI0ZWE4ZDYwMWU3Nzg5ZDdiYTAuc2V0Q29udGVudChodG1sXzRkMzcwY2ZhZmYyNTQ5Y2E5MjA3OTlkMTNlZDE4OTUzKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfNWMyMzkwNTczZjY2NDc0ZGIwNzcyMmQ3MDA3MzM5YTguYmluZFBvcHVwKHBvcHVwXzVjMDZiZjFmMTJhYjQyNGVhOGQ2MDFlNzc4OWQ3YmEwKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzM5MDFhZmUyMGVmMDRkNjU5ODBlODAyZmMyYzM5ZDk2ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzE1NjIsMTE0LjE4MTQ3XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2I2MzZjYmZiMTk4MTQzZWE4MDUxMjRhYjIxZjNkZDgyID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdncmVlbicsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl8zOTAxYWZlMjBlZjA0ZDY1OTgwZTgwMmZjMmMzOWQ5Ni5zZXRJY29uKGljb25fYjYzNmNiZmIxOTgxNDNlYTgwNTEyNGFiMjFmM2RkODIpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfYTM5ZjBhY2EzMGY0NDJiNGJlZmVhM2FhZjVmYTRiYzUgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfZDRlYjAyOWY5NzhlNDE0N2IyYmMzNmIxZjMzOWQ1MGEgPSAkKCc8ZGl2IGlkPSJodG1sX2Q0ZWIwMjlmOTc4ZTQxNDdiMmJjMzZiMWYzMzlkNTBhIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjI0MSwgTEFCRUw6MiwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfYTM5ZjBhY2EzMGY0NDJiNGJlZmVhM2FhZjVmYTRiYzUuc2V0Q29udGVudChodG1sX2Q0ZWIwMjlmOTc4ZTQxNDdiMmJjMzZiMWYzMzlkNTBhKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfMzkwMWFmZTIwZWYwNGQ2NTk4MGU4MDJmYzJjMzlkOTYuYmluZFBvcHVwKHBvcHVwX2EzOWYwYWNhMzBmNDQyYjRiZWZlYTNhYWY1ZmE0YmM1KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzAxMjU1MDNmZDQ0NjQ4MjY5ZWU4ZDFlOWJjNmI0NTM3ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMjc3OTQ4NywxMTQuMTc1NzgyNV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9jMzdhOTk0ZGRiZDA0MDY3YWJhMDA4YjA3NjQ4ZDZmMiA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnbGlnaHRyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfMDEyNTUwM2ZkNDQ2NDgyNjllZThkMWU5YmM2YjQ1Mzcuc2V0SWNvbihpY29uX2MzN2E5OTRkZGJkMDQwNjdhYmEwMDhiMDc2NDhkNmYyKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzFlNDUzNGNkMTM3NjQ0YzI5YWIxMDNjYWQzZDZiOWE4ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2Q3MGE5NWI2ZTI5ODRhMGE5YTkwZGM0YWY4YWE3NTQ2ID0gJCgnPGRpdiBpZD0iaHRtbF9kNzBhOTViNmUyOTg0YTBhOWE5MGRjNGFmOGFhNzU0NiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIyNDksIExBQkVMOjYsIFNFR01FTlQ6V1RDIC0gQ29tbWVyY2lhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMWU0NTM0Y2QxMzc2NDRjMjlhYjEwM2NhZDNkNmI5YTguc2V0Q29udGVudChodG1sX2Q3MGE5NWI2ZTI5ODRhMGE5YTkwZGM0YWY4YWE3NTQ2KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfMDEyNTUwM2ZkNDQ2NDgyNjllZThkMWU5YmM2YjQ1MzcuYmluZFBvcHVwKHBvcHVwXzFlNDUzNGNkMTM3NjQ0YzI5YWIxMDNjYWQzZDZiOWE4KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzk0M2Q0M2ExYWJhNjQ4ZGJhMmQwNjE5YjQwMGE0YzFiID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzIxMDQyNywxMTQuMTcwODAzN10sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9mNDI4YzA0OGYxYzE0YjZhOGJjZTg5ZTAwMWRmOGY5ZSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnZ3JlZW4nLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfOTQzZDQzYTFhYmE2NDhkYmEyZDA2MTliNDAwYTRjMWIuc2V0SWNvbihpY29uX2Y0MjhjMDQ4ZjFjMTRiNmE4YmNlODllMDAxZGY4ZjllKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzhjMDg3ZjlmMGM3YzRmMWRiNjJmM2U1YTljZTU4MTE0ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2NhNTYxMWE2MTVhODQ0YjNiM2RiYTQxZDc0YzlhNWM0ID0gJCgnPGRpdiBpZD0iaHRtbF9jYTU2MTFhNjE1YTg0NGIzYjNkYmE0MWQ3NGM5YTVjNCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIzODYsIExBQkVMOjIsIFNFR01FTlQ6V1RDIC0gVG91cmlzdHMvQ29udjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfOGMwODdmOWYwYzdjNGYxZGI2MmYzZTVhOWNlNTgxMTQuc2V0Q29udGVudChodG1sX2NhNTYxMWE2MTVhODQ0YjNiM2RiYTQxZDc0YzlhNWM0KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfOTQzZDQzYTFhYmE2NDhkYmEyZDA2MTliNDAwYTRjMWIuYmluZFBvcHVwKHBvcHVwXzhjMDg3ZjlmMGM3YzRmMWRiNjJmM2U1YTljZTU4MTE0KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzJhY2NhNzA3ZmIyYTQxZGM5NjMzNWZmN2M4MGRhMzE4ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzY4NTI5LDExNC4xMzY1OF0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl80MmE0ZTQxMDY2MmU0NGIxYmYwZTRhMGVlNTFkMDZiMSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAncmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzJhY2NhNzA3ZmIyYTQxZGM5NjMzNWZmN2M4MGRhMzE4LnNldEljb24oaWNvbl80MmE0ZTQxMDY2MmU0NGIxYmYwZTRhMGVlNTFkMDZiMSk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9mZTNjOThiMTcyNjg0OTdmYmNlMTY3ZmYxMzIzY2NjZiA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF8zNWE1NzgzOGNlNGU0ZWYyYWJiYWIyMDliZDQ5MGY1OCA9ICQoJzxkaXYgaWQ9Imh0bWxfMzVhNTc4MzhjZTRlNGVmMmFiYmFiMjA5YmQ0OTBmNTgiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEzMDgzLCBMQUJFTDowLCBTRUdNRU5UOldUQyAtIFNhdmVyczwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfZmUzYzk4YjE3MjY4NDk3ZmJjZTE2N2ZmMTMyM2NjY2Yuc2V0Q29udGVudChodG1sXzM1YTU3ODM4Y2U0ZTRlZjJhYmJhYjIwOWJkNDkwZjU4KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfMmFjY2E3MDdmYjJhNDFkYzk2MzM1ZmY3YzgwZGEzMTguYmluZFBvcHVwKHBvcHVwX2ZlM2M5OGIxNzI2ODQ5N2ZiY2UxNjdmZjEzMjNjY2NmKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2I3NjZkMGU4NDAzYzQ4MmRiM2JlNTU2ODA1ZTRiOTkxID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuNDI1MzI4MiwxMTQuMjMxMDYyNF0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl8wMDJiNmYzN2E5YjA0ZjZlODQzNzIxMzAxYmMzMDc1OSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAncmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyX2I3NjZkMGU4NDAzYzQ4MmRiM2JlNTU2ODA1ZTRiOTkxLnNldEljb24oaWNvbl8wMDJiNmYzN2E5YjA0ZjZlODQzNzIxMzAxYmMzMDc1OSk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8wYjU0YWJhNzI5OTk0NTA1ODFmNjE4YzQxMzhjOWNkMiA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9kMTFjMWE3ZDE1OTk0ZjdkOTJhNDc3ODVhYWJiM2M5OCA9ICQoJzxkaXYgaWQ9Imh0bWxfZDExYzFhN2QxNTk5NGY3ZDkyYTQ3Nzg1YWFiYjNjOTgiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjExNzk3LCBMQUJFTDowLCBTRUdNRU5UOldUQyAtIFJlc2lkZW50aWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8wYjU0YWJhNzI5OTk0NTA1ODFmNjE4YzQxMzhjOWNkMi5zZXRDb250ZW50KGh0bWxfZDExYzFhN2QxNTk5NGY3ZDkyYTQ3Nzg1YWFiYjNjOTgpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl9iNzY2ZDBlODQwM2M0ODJkYjNiZTU1NjgwNWU0Yjk5MS5iaW5kUG9wdXAocG9wdXBfMGI1NGFiYTcyOTk5NDUwNTgxZjYxOGM0MTM4YzljZDIpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfNTgwN2MxOGExMjg2NDdiYThhOWI1YTMwNzlhZGQ2NjYgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4xODg3NiwxMTMuNTU0MTJdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fNmFhYzgxYzJmMWUwNDJlNzljNjRhYTMwMTNmODMxYTMgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3B1cnBsZScsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl81ODA3YzE4YTEyODY0N2JhOGE5YjVhMzA3OWFkZDY2Ni5zZXRJY29uKGljb25fNmFhYzgxYzJmMWUwNDJlNzljNjRhYTMwMTNmODMxYTMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfMmRlMWFiZTE1ZDlmNDk3NDk0NGY0ZGVkMThlZjI4M2MgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfZWUyMWI2NWQ0Yzk3NDA4ZmEyY2U3MWNiZDE3MzQ3OWMgPSAkKCc8ZGl2IGlkPSJodG1sX2VlMjFiNjVkNGM5NzQwOGZhMmNlNzFjYmQxNzM0NzljIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMTg4MSwgTEFCRUw6MywgU0VHTUVOVDpXVEMgLSBUb3VyaXN0cy9Db252PC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8yZGUxYWJlMTVkOWY0OTc0OTQ0ZjRkZWQxOGVmMjgzYy5zZXRDb250ZW50KGh0bWxfZWUyMWI2NWQ0Yzk3NDA4ZmEyY2U3MWNiZDE3MzQ3OWMpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl81ODA3YzE4YTEyODY0N2JhOGE5YjVhMzA3OWFkZDY2Ni5iaW5kUG9wdXAocG9wdXBfMmRlMWFiZTE1ZDlmNDk3NDk0NGY0ZGVkMThlZjI4M2MpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfYzU2MGM3ZWZmZjMyNGIwZmE2ZDcwZjc1MDc5NGM5ZjEgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4zOTA4Mjk1LDExMy45NzI1MTI2XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzI1ZTU0YzEyYzZmYjQ0N2U4MTY4NmQzMDQ2NTU5ODU3ID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdvcmFuZ2UnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfYzU2MGM3ZWZmZjMyNGIwZmE2ZDcwZjc1MDc5NGM5ZjEuc2V0SWNvbihpY29uXzI1ZTU0YzEyYzZmYjQ0N2U4MTY4NmQzMDQ2NTU5ODU3KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2M5NmRlNGU0MTI1ZjQyZjZhZTFhNTZmOTgzZTEyMTJjID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2Q1N2I5Y2IwZDQ1YjQ0ZWViMTlmMTEzNGRhYjU5YTMyID0gJCgnPGRpdiBpZD0iaHRtbF9kNTdiOWNiMGQ0NWI0NGVlYjE5ZjExMzRkYWI1OWEzMiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTE5MDAsIExBQkVMOjQsIFNFR01FTlQ6V1RDIC0gRGVzdGluYXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2M5NmRlNGU0MTI1ZjQyZjZhZTFhNTZmOTgzZTEyMTJjLnNldENvbnRlbnQoaHRtbF9kNTdiOWNiMGQ0NWI0NGVlYjE5ZjExMzRkYWI1OWEzMik7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyX2M1NjBjN2VmZmYzMjRiMGZhNmQ3MGY3NTA3OTRjOWYxLmJpbmRQb3B1cChwb3B1cF9jOTZkZTRlNDEyNWY0MmY2YWUxYTU2Zjk4M2UxMjEyYyk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl8wOTNmMmExMzkwM2M0NWRjYWMyMjJhYjJkZDFlY2E0OCA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjM4NjE0OTMsMTE0LjE5MjkyNzRdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fYTA3Njc1ZTNlYTZkNDQxNDk3ZWE2OTc0ODM0ZWE0MDcgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3JlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl8wOTNmMmExMzkwM2M0NWRjYWMyMjJhYjJkZDFlY2E0OC5zZXRJY29uKGljb25fYTA3Njc1ZTNlYTZkNDQxNDk3ZWE2OTc0ODM0ZWE0MDcpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfZDQ5ZTUzZDRmMDVkNDA0OWFiYjdkNmQ1MWFjN2IyMDUgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNTFlNzZlYjc1YmZlNDU4ZDg5NDAzYjc1ZDExMDIyODEgPSAkKCc8ZGl2IGlkPSJodG1sXzUxZTc2ZWI3NWJmZTQ1OGQ4OTQwM2I3NWQxMTAyMjgxIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjAxOCwgTEFCRUw6MCwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfZDQ5ZTUzZDRmMDVkNDA0OWFiYjdkNmQ1MWFjN2IyMDUuc2V0Q29udGVudChodG1sXzUxZTc2ZWI3NWJmZTQ1OGQ4OTQwM2I3NWQxMTAyMjgxKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfMDkzZjJhMTM5MDNjNDVkY2FjMjIyYWIyZGQxZWNhNDguYmluZFBvcHVwKHBvcHVwX2Q0OWU1M2Q0ZjA1ZDQwNDlhYmI3ZDZkNTFhYzdiMjA1KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2ViNmQ2NGRmMDRjYjQzNzdhYjE3YzhhOGFmYTljNjcwID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzUxNTUxOCwxMTQuMjAwNDEyXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzRiYTg1MGFkNTQ4ZDQ0Zjg4NjI4NzBlMjdjYWRkNWU3ID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfZWI2ZDY0ZGYwNGNiNDM3N2FiMTdjOGE4YWZhOWM2NzAuc2V0SWNvbihpY29uXzRiYTg1MGFkNTQ4ZDQ0Zjg4NjI4NzBlMjdjYWRkNWU3KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2RjMWQ4ODM1ODY4YTQ2NGZhZmUyYTgzNGRmYTRhY2Y2ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2I4OGY3MWFmMzc2NjRhZjBhMDQ2NjAwMzU4NTJlOGY5ID0gJCgnPGRpdiBpZD0iaHRtbF9iODhmNzFhZjM3NjY0YWYwYTA0NjYwMDM1ODUyZThmOSIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIxMTAsIExBQkVMOjAsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2RjMWQ4ODM1ODY4YTQ2NGZhZmUyYTgzNGRmYTRhY2Y2LnNldENvbnRlbnQoaHRtbF9iODhmNzFhZjM3NjY0YWYwYTA0NjYwMDM1ODUyZThmOSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyX2ViNmQ2NGRmMDRjYjQzNzdhYjE3YzhhOGFmYTljNjcwLmJpbmRQb3B1cChwb3B1cF9kYzFkODgzNTg2OGE0NjRmYWZlMmE4MzRkZmE0YWNmNik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl84YTU0YmE0ZWU0YWE0YTI1OTMwMDZmYTJmMTc0YTc2YiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjQ0NDc0OTgsMTE0LjAyNjE3NTVdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fMmE3NWZhMDAyOTA3NDM4NGE4NjlkZDRmY2I0OTU0NjMgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2xpZ2h0cmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzhhNTRiYTRlZTRhYTRhMjU5MzAwNmZhMmYxNzRhNzZiLnNldEljb24oaWNvbl8yYTc1ZmEwMDI5MDc0Mzg0YTg2OWRkNGZjYjQ5NTQ2Myk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF83NjdjZjQ5OTU4ZjY0ZGFhYjExZTY1MDU5YjJkZjUzNyA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF8xODIxN2YwZmZhMWM0M2NkOGEzZjU5ODg2NGUwZGY0YiA9ICQoJzxkaXYgaWQ9Imh0bWxfMTgyMTdmMGZmYTFjNDNjZDhhM2Y1OTg4NjRlMGRmNGIiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyNjIzLCBMQUJFTDo2LCBTRUdNRU5UOldUQyAtIFRvdXJpc3RzL0NvbnY8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzc2N2NmNDk5NThmNjRkYWFiMTFlNjUwNTliMmRmNTM3LnNldENvbnRlbnQoaHRtbF8xODIxN2YwZmZhMWM0M2NkOGEzZjU5ODg2NGUwZGY0Yik7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzhhNTRiYTRlZTRhYTRhMjU5MzAwNmZhMmYxNzRhNzZiLmJpbmRQb3B1cChwb3B1cF83NjdjZjQ5OTU4ZjY0ZGFhYjExZTY1MDU5YjJkZjUzNyk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl83ZWQ4YzVjM2RhOGE0NTY2ODM5ZWZhZTNlOWZiNmQ0ZCA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjI4MzM0NjYsMTE0LjE1NjQ1MjhdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fZTc0Y2ZkZDNkZGQ2NDlkMDg5M2Q2YzQ1NDM5MDc4YTcgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2dyZWVuJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzdlZDhjNWMzZGE4YTQ1NjY4MzllZmFlM2U5ZmI2ZDRkLnNldEljb24oaWNvbl9lNzRjZmRkM2RkZDY0OWQwODkzZDZjNDU0MzkwNzhhNyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF81MTU2MjE4YWEyYzM0YjFlYTJlZDcyNmU5OWNjN2ZmZCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9jOWEzM2E3MDBlNjE0ODg3OGUxZmI2OGQxMWUxOTIxZiA9ICQoJzxkaXYgaWQ9Imh0bWxfYzlhMzNhNzAwZTYxNDg4NzhlMWZiNjhkMTFlMTkyMWYiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjExOTUxLCBMQUJFTDoyLCBTRUdNRU5UOldUQyAtIENvbW1lcmNpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzUxNTYyMThhYTJjMzRiMWVhMmVkNzI2ZTk5Y2M3ZmZkLnNldENvbnRlbnQoaHRtbF9jOWEzM2E3MDBlNjE0ODg3OGUxZmI2OGQxMWUxOTIxZik7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzdlZDhjNWMzZGE4YTQ1NjY4MzllZmFlM2U5ZmI2ZDRkLmJpbmRQb3B1cChwb3B1cF81MTU2MjE4YWEyYzM0YjFlYTJlZDcyNmU5OWNjN2ZmZCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl81ZDMxMWNlNTM2ZTA0NmQyOTJjOTkzNzk5OGJlZDE5NiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjMxMTA0MDgsMTE0LjI1ODk1MDldLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fMWJhMGU5OGQ4Yjk2NDJhY2ExMjUwMTk4NGU3N2ViMGYgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3JlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl81ZDMxMWNlNTM2ZTA0NmQyOTJjOTkzNzk5OGJlZDE5Ni5zZXRJY29uKGljb25fMWJhMGU5OGQ4Yjk2NDJhY2ExMjUwMTk4NGU3N2ViMGYpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNjhjMzFlNWJjOGJlNDQ1Yjk1ZGU3YjdlZDJhNGVjMTIgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfZWM2ZTcwYWNiNjM2NDY2Yzk4MmExN2IyMDM0NDdmZDIgPSAkKCc8ZGl2IGlkPSJodG1sX2VjNmU3MGFjYjYzNjQ2NmM5ODJhMTdiMjAzNDQ3ZmQyIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjAzOSwgTEFCRUw6MCwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNjhjMzFlNWJjOGJlNDQ1Yjk1ZGU3YjdlZDJhNGVjMTIuc2V0Q29udGVudChodG1sX2VjNmU3MGFjYjYzNjQ2NmM5ODJhMTdiMjAzNDQ3ZmQyKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfNWQzMTFjZTUzNmUwNDZkMjkyYzk5Mzc5OThiZWQxOTYuYmluZFBvcHVwKHBvcHVwXzY4YzMxZTViYzhiZTQ0NWI5NWRlN2I3ZWQyYTRlYzEyKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzU3MDA1MjUzNWFjNzRkYmE5NzdjODk4YjQyNWVkMTU4ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuNTQ2NjEyLDExNC4yMjIzMDZdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fOTdiMDMwZDY5ZWM2NDkxMDk2OGRkMjg1ZGY2YzBmNjcgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2JsdWUnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfNTcwMDUyNTM1YWM3NGRiYTk3N2M4OThiNDI1ZWQxNTguc2V0SWNvbihpY29uXzk3YjAzMGQ2OWVjNjQ5MTA5NjhkZDI4NWRmNmMwZjY3KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzA3Mzk3NzI3NWY1ZjRmMmM5NWJkMTQ5MzI5MWViZGYyID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzhmYTU2MzRlOTc4NzQ5N2U4YjUxNjk1OTY3YTA4NzU0ID0gJCgnPGRpdiBpZD0iaHRtbF84ZmE1NjM0ZTk3ODc0OTdlOGI1MTY5NTk2N2EwODc1NCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIwNjUsIExBQkVMOjEsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzA3Mzk3NzI3NWY1ZjRmMmM5NWJkMTQ5MzI5MWViZGYyLnNldENvbnRlbnQoaHRtbF84ZmE1NjM0ZTk3ODc0OTdlOGI1MTY5NTk2N2EwODc1NCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzU3MDA1MjUzNWFjNzRkYmE5NzdjODk4YjQyNWVkMTU4LmJpbmRQb3B1cChwb3B1cF8wNzM5NzcyNzVmNWY0ZjJjOTViZDE0OTMyOTFlYmRmMik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl85MjdjZmZjZGJjN2I0OWFmOTE2NzU4MzIwNjI5NWVhNiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjUxNDQ2NCwxMTQuMDY1NjQ3XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2EwOGUwZjdmNjJmMjRiNTlhZGRmNDA2ZTVmYjRkZGJiID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdsaWdodHJlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl85MjdjZmZjZGJjN2I0OWFmOTE2NzU4MzIwNjI5NWVhNi5zZXRJY29uKGljb25fYTA4ZTBmN2Y2MmYyNGI1OWFkZGY0MDZlNWZiNGRkYmIpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfZTY0NjU3YTBiOTExNDM3ZTgxZThlNGFiMDRiNmEyNDggPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfMDgyNTBkYTY5ZGIxNDBjNDhiNTExOTU4YjJlNzMwYmIgPSAkKCc8ZGl2IGlkPSJodG1sXzA4MjUwZGE2OWRiMTQwYzQ4YjUxMTk1OGIyZTczMGJiIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjA2NiwgTEFCRUw6NiwgU0VHTUVOVDpXVEMgLSBUb3VyaXN0cy9Db252PC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9lNjQ2NTdhMGI5MTE0MzdlODFlOGU0YWIwNGI2YTI0OC5zZXRDb250ZW50KGh0bWxfMDgyNTBkYTY5ZGIxNDBjNDhiNTExOTU4YjJlNzMwYmIpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl85MjdjZmZjZGJjN2I0OWFmOTE2NzU4MzIwNjI5NWVhNi5iaW5kUG9wdXAocG9wdXBfZTY0NjU3YTBiOTExNDM3ZTgxZThlNGFiMDRiNmEyNDgpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfNDc5YTU5N2MwYmQ5NDA2N2JlMWE3ZWVlMDI3YjM2ZjAgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi40NDk4ODcsMTE0LjE2OTAyMV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl84NWZlYmUyZTJmOTk0NTM2ODI1NDAwYTYwYWZkYTdhMSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAncmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzQ3OWE1OTdjMGJkOTQwNjdiZTFhN2VlZTAyN2IzNmYwLnNldEljb24oaWNvbl84NWZlYmUyZTJmOTk0NTM2ODI1NDAwYTYwYWZkYTdhMSk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8wN2UzZjNlN2ViNjc0MTJkYTMzYmZhMWViYzI2MDJiZSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF81NTVlMDM2YjFjMGI0OWM4OWVlYzg5NzUwYzEyMWIzMyA9ICQoJzxkaXYgaWQ9Imh0bWxfNTU1ZTAzNmIxYzBiNDljODllZWM4OTc1MGMxMjFiMzMiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMDcyLCBMQUJFTDowLCBTRUdNRU5UOldUQyAtIEhlYWx0aDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMDdlM2YzZTdlYjY3NDEyZGEzM2JmYTFlYmMyNjAyYmUuc2V0Q29udGVudChodG1sXzU1NWUwMzZiMWMwYjQ5Yzg5ZWVjODk3NTBjMTIxYjMzKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfNDc5YTU5N2MwYmQ5NDA2N2JlMWE3ZWVlMDI3YjM2ZjAuYmluZFBvcHVwKHBvcHVwXzA3ZTNmM2U3ZWI2NzQxMmRhMzNiZmExZWJjMjYwMmJlKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2ZjMGQwZTY1YmE1YjRkNGY4ZDU5ZTY4ODVjNTRiZWViID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzI4NTg5OSwxMTQuMTYwMjg0Nl0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl82YzliZmU2ODdhODQ0Y2QyOWIzOWZhMjQ5ZTQ4NGM3ZSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnZ3JlZW4nLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfZmMwZDBlNjViYTViNGQ0ZjhkNTllNjg4NWM1NGJlZWIuc2V0SWNvbihpY29uXzZjOWJmZTY4N2E4NDRjZDI5YjM5ZmEyNDllNDg0YzdlKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2E3MTFlMGNjYmFjMTQyMTlhNmVkYTllZWJiM2EwNDc5ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzExMmQyZWM4OWQwNjRhMTliOTBhMWJlM2JjZWMwNTgwID0gJCgnPGRpdiBpZD0iaHRtbF8xMTJkMmVjODlkMDY0YTE5YjkwYTFiZTNiY2VjMDU4MCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIyNDMsIExBQkVMOjIsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2E3MTFlMGNjYmFjMTQyMTlhNmVkYTllZWJiM2EwNDc5LnNldENvbnRlbnQoaHRtbF8xMTJkMmVjODlkMDY0YTE5YjkwYTFiZTNiY2VjMDU4MCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyX2ZjMGQwZTY1YmE1YjRkNGY4ZDU5ZTY4ODVjNTRiZWViLmJpbmRQb3B1cChwb3B1cF9hNzExZTBjY2JhYzE0MjE5YTZlZGE5ZWViYjNhMDQ3OSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl84ZGJiNDFjNTAyMTY0NDk0YTEwMzBmMjQ2MDVmZTFiNCA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjMxNzEyMTgsMTE0LjI2NjI2MTVdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fMGFjYWQ0ODM1NjRlNDZkZWEwYzgzNTFkZjkwZjg5OTQgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3JlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl84ZGJiNDFjNTAyMTY0NDk0YTEwMzBmMjQ2MDVmZTFiNC5zZXRJY29uKGljb25fMGFjYWQ0ODM1NjRlNDZkZWEwYzgzNTFkZjkwZjg5OTQpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfYWIyYzI3MzZlZjI5NDQ2NmJjNjVkYTkyZTA2ZDA4YTEgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfOGMyZWM4NzgzMzhhNGM0ODgzNWQ2Yzg5YjQ0OWUyN2UgPSAkKCc8ZGl2IGlkPSJodG1sXzhjMmVjODc4MzM4YTRjNDg4MzVkNmM4OWI0NDllMjdlIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjUxMywgTEFCRUw6MCwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfYWIyYzI3MzZlZjI5NDQ2NmJjNjVkYTkyZTA2ZDA4YTEuc2V0Q29udGVudChodG1sXzhjMmVjODc4MzM4YTRjNDg4MzVkNmM4OWI0NDllMjdlKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfOGRiYjQxYzUwMjE2NDQ5NGExMDMwZjI0NjA1ZmUxYjQuYmluZFBvcHVwKHBvcHVwX2FiMmMyNzM2ZWYyOTQ0NjZiYzY1ZGE5MmUwNmQwOGExKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzgxZmMzNGZmZjk1ODRlY2Y4YTI2MDRmNjEyN2RkMGI4ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzEyNDA2NywxMTQuMTc4ODYyNl0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl82ZGI1NTk5MzliNTU0OTE0ODhiNDNmMWUwNzE1NDhmNCA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAncmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzgxZmMzNGZmZjk1ODRlY2Y4YTI2MDRmNjEyN2RkMGI4LnNldEljb24oaWNvbl82ZGI1NTk5MzliNTU0OTE0ODhiNDNmMWUwNzE1NDhmNCk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8zYjlmZTUzYWI4OGU0MDUyYWY0YWExMDFlZmE1ZDIyZSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF83ZmVkNWI5ZmQ0MTk0ZjZmYmE0MGI1Y2MwZjA1ZWViMyA9ICQoJzxkaXYgaWQ9Imh0bWxfN2ZlZDViOWZkNDE5NGY2ZmJhNDBiNWNjMGYwNWVlYjMiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyNjE2LCBMQUJFTDowLCBTRUdNRU5UOldUQyAtIFNhdmVyczwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfM2I5ZmU1M2FiODhlNDA1MmFmNGFhMTAxZWZhNWQyMmUuc2V0Q29udGVudChodG1sXzdmZWQ1YjlmZDQxOTRmNmZiYTQwYjVjYzBmMDVlZWIzKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfODFmYzM0ZmZmOTU4NGVjZjhhMjYwNGY2MTI3ZGQwYjguYmluZFBvcHVwKHBvcHVwXzNiOWZlNTNhYjg4ZTQwNTJhZjRhYTEwMWVmYTVkMjJlKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2FjZjRmYTkwMmZjODQ1MDI5NzYyZDQ3N2UwOWNiNWRlID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMTUwNTIyLDExMy41NTIzODVdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fM2FjMTg0NDkyOTQ2NGZjYmJmZDhkMTRjYTJmYjNlM2IgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3B1cnBsZScsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl9hY2Y0ZmE5MDJmYzg0NTAyOTc2MmQ0NzdlMDljYjVkZS5zZXRJY29uKGljb25fM2FjMTg0NDkyOTQ2NGZjYmJmZDhkMTRjYTJmYjNlM2IpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfN2IwNTQ3ZDEwYjAzNDkxMWIzZDNlMmUzNmU3YjkyNTIgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfMmY4ZjM4NmI3YzhlNGE1MTg2MTgzYjQyMWQxZmU0MTUgPSAkKCc8ZGl2IGlkPSJodG1sXzJmOGYzODZiN2M4ZTRhNTE4NjE4M2I0MjFkMWZlNDE1IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjg4MSwgTEFCRUw6MywgU0VHTUVOVDpXVEMgLSBCYWJ5PC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF83YjA1NDdkMTBiMDM0OTExYjNkM2UyZTM2ZTdiOTI1Mi5zZXRDb250ZW50KGh0bWxfMmY4ZjM4NmI3YzhlNGE1MTg2MTgzYjQyMWQxZmU0MTUpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl9hY2Y0ZmE5MDJmYzg0NTAyOTc2MmQ0NzdlMDljYjVkZS5iaW5kUG9wdXAocG9wdXBfN2IwNTQ3ZDEwYjAzNDkxMWIzZDNlMmUzNmU3YjkyNTIpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfM2I3ODhmNDJlYzQyNGUyZDlhOWI4MTVmNzU1YTQxZmMgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi40NDQ1Mzc2LDExNC4wMjIyMDc2XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzFlYTAxOGU5OGM0ZTRmMTI4YmJmYWU0NDM2NDQ1M2M3ID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdncmVlbicsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl8zYjc4OGY0MmVjNDI0ZTJkOWE5YjgxNWY3NTVhNDFmYy5zZXRJY29uKGljb25fMWVhMDE4ZTk4YzRlNGYxMjhiYmZhZTQ0MzY0NDUzYzcpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfYzdlZGJhMDdlODExNDlkYjg5MjZjM2IzZDQyMTZlY2YgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNjJjMDEwZDlhMGNlNDViM2JhNWNhYjFiMzFjNWI5M2IgPSAkKCc8ZGl2IGlkPSJodG1sXzYyYzAxMGQ5YTBjZTQ1YjNiYTVjYWIxYjMxYzViOTNiIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMTcyMiwgTEFCRUw6MiwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfYzdlZGJhMDdlODExNDlkYjg5MjZjM2IzZDQyMTZlY2Yuc2V0Q29udGVudChodG1sXzYyYzAxMGQ5YTBjZTQ1YjNiYTVjYWIxYjMxYzViOTNiKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfM2I3ODhmNDJlYzQyNGUyZDlhOWI4MTVmNzU1YTQxZmMuYmluZFBvcHVwKHBvcHVwX2M3ZWRiYTA3ZTgxMTQ5ZGI4OTI2YzNiM2Q0MjE2ZWNmKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzU5NmM3N2QxZjg0NjQ1NjBhNjMzMjc5MWQzY2YwYzAxID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzE3MzM3MywxMTQuMTg3OTY5MV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9mN2YxM2E4YjlkYTQ0MmQ1YTlkOWE0MGI5M2EwOTZiZSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAncmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzU5NmM3N2QxZjg0NjQ1NjBhNjMzMjc5MWQzY2YwYzAxLnNldEljb24oaWNvbl9mN2YxM2E4YjlkYTQ0MmQ1YTlkOWE0MGI5M2EwOTZiZSk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8xN2ZmYTczYjQyZTk0YTE2YWU1YjRkNjYzMjBhYjMwMyA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF84M2IxMmU5OGMwOTU0NzAwODczYjY0OGYzMjRhZmM2MyA9ICQoJzxkaXYgaWQ9Imh0bWxfODNiMTJlOThjMDk1NDcwMDg3M2I2NDhmMzI0YWZjNjMiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjExNzQ0LCBMQUJFTDowLCBTRUdNRU5UOldUQyAtIFJlc2lkZW50aWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8xN2ZmYTczYjQyZTk0YTE2YWU1YjRkNjYzMjBhYjMwMy5zZXRDb250ZW50KGh0bWxfODNiMTJlOThjMDk1NDcwMDg3M2I2NDhmMzI0YWZjNjMpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl81OTZjNzdkMWY4NDY0NTYwYTYzMzI3OTFkM2NmMGMwMS5iaW5kUG9wdXAocG9wdXBfMTdmZmE3M2I0MmU5NGExNmFlNWI0ZDY2MzIwYWIzMDMpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfOTQyYTRkMjZjMTA1NGMyN2I5NTZkMDU1ZjVlMjI5MjEgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi40NDI3NTU3LDExNC4wMjg0MjMzXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzk2MzhjZDE0MzdlZTRkMDhiYmUzNjZkYWZhNzhiMjVlID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfOTQyYTRkMjZjMTA1NGMyN2I5NTZkMDU1ZjVlMjI5MjEuc2V0SWNvbihpY29uXzk2MzhjZDE0MzdlZTRkMDhiYmUzNjZkYWZhNzhiMjVlKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2E1YWFmYmExMTM0NjQzZDFhOGE4NGFlMDE0MzY1OWQ3ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzk3MWQwODQzODgxYTQyZGQ5OTM5YTc0NmIyZGU2NzU5ID0gJCgnPGRpdiBpZD0iaHRtbF85NzFkMDg0Mzg4MWE0MmRkOTkzOWE3NDZiMmRlNjc1OSIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTE5MDksIExBQkVMOjAsIFNFR01FTlQ6V1RDIC0gVG91cmlzdHMvQ29udjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfYTVhYWZiYTExMzQ2NDNkMWE4YTg0YWUwMTQzNjU5ZDcuc2V0Q29udGVudChodG1sXzk3MWQwODQzODgxYTQyZGQ5OTM5YTc0NmIyZGU2NzU5KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfOTQyYTRkMjZjMTA1NGMyN2I5NTZkMDU1ZjVlMjI5MjEuYmluZFBvcHVwKHBvcHVwX2E1YWFmYmExMTM0NjQzZDFhOGE4NGFlMDE0MzY1OWQ3KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2M5NmNlYjQ3ODE4ZjRlNzA4ZmNjNzE3YjQ4YzBlZjlhID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzAwNTI3NCwxMTQuMTY3NzkzMl0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9lNWY4MmYwZjQyZWQ0YTliYjkyYTliYWNhNDBhMTAwNSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnb3JhbmdlJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyX2M5NmNlYjQ3ODE4ZjRlNzA4ZmNjNzE3YjQ4YzBlZjlhLnNldEljb24oaWNvbl9lNWY4MmYwZjQyZWQ0YTliYjkyYTliYWNhNDBhMTAwNSk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9jZTQ0YmUwOWJmZDY0MDgzYmIzYTUyM2Q2MmUxM2RhMCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF80MWYzODI2ZTFkOWY0NzA5OGU0OTRmNThjZDA2MjE3YyA9ICQoJzxkaXYgaWQ9Imh0bWxfNDFmMzgyNmUxZDlmNDcwOThlNDk0ZjU4Y2QwNjIxN2MiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMTA0LCBMQUJFTDo0LCBTRUdNRU5UOldUQyAtIFRvdXJpc3RzL0NvbnY8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2NlNDRiZTA5YmZkNjQwODNiYjNhNTIzZDYyZTEzZGEwLnNldENvbnRlbnQoaHRtbF80MWYzODI2ZTFkOWY0NzA5OGU0OTRmNThjZDA2MjE3Yyk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyX2M5NmNlYjQ3ODE4ZjRlNzA4ZmNjNzE3YjQ4YzBlZjlhLmJpbmRQb3B1cChwb3B1cF9jZTQ0YmUwOWJmZDY0MDgzYmIzYTUyM2Q2MmUxM2RhMCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl85ODg3YTdlMTI5NjI0MGNkOTExNmIzODFlMDViNGE1ZCA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjI5NTA2ODksMTE0LjE2Njg2NjFdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fNzgyNGUyYWI2Y2RiNDI2NDkxYzhmM2U1NTUyNjYzYmUgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2RhcmtibHVlJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzk4ODdhN2UxMjk2MjQwY2Q5MTE2YjM4MWUwNWI0YTVkLnNldEljb24oaWNvbl83ODI0ZTJhYjZjZGI0MjY0OTFjOGYzZTU1NTI2NjNiZSk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF83ZTU1YmUxYTk0ZjE0YzllYjk3ODE4MzQxOGMwMDAwMiA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9jZWZiMzA0ZDEwNjA0NzhkYmE2YjYwNDEzZTliY2ZlOSA9ICQoJzxkaXYgaWQ9Imh0bWxfY2VmYjMwNGQxMDYwNDc4ZGJhNmI2MDQxM2U5YmNmZTkiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMTA1LCBMQUJFTDo4LCBTRUdNRU5UOldUQyAtIEhlYWx0aDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfN2U1NWJlMWE5NGYxNGM5ZWI5NzgxODM0MThjMDAwMDIuc2V0Q29udGVudChodG1sX2NlZmIzMDRkMTA2MDQ3OGRiYTZiNjA0MTNlOWJjZmU5KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfOTg4N2E3ZTEyOTYyNDBjZDkxMTZiMzgxZTA1YjRhNWQuYmluZFBvcHVwKHBvcHVwXzdlNTViZTFhOTRmMTRjOWViOTc4MTgzNDE4YzAwMDAyKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2UwNWExYWUzNDVmZjQ5OTY5YWQxOGU1NzNhZjlhNDUwID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzA2OTk5MywxMTQuMjMzNTk3Ml0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9hZWQ3OTAyYTljMjY0OTZkYWJhMTBjNGMwZGZiM2YxMCA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnZGFya2JsdWUnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfZTA1YTFhZTM0NWZmNDk5NjlhZDE4ZTU3M2FmOWE0NTAuc2V0SWNvbihpY29uX2FlZDc5MDJhOWMyNjQ5NmRhYmExMGM0YzBkZmIzZjEwKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzlkNTBlNTIyMTgyOTRkNjg4NGZhYzk4ODBmM2Y1YzE4ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2JlNDBjOGNkZmFlNzQ3MDVhMTY5MDgzMjY4MmE2ZDRkID0gJCgnPGRpdiBpZD0iaHRtbF9iZTQwYzhjZGZhZTc0NzA1YTE2OTA4MzI2ODJhNmQ0ZCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIzMzAsIExBQkVMOjgsIFNFR01FTlQ6V1RDIC0gU2F2ZXJzPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF85ZDUwZTUyMjE4Mjk0ZDY4ODRmYWM5ODgwZjNmNWMxOC5zZXRDb250ZW50KGh0bWxfYmU0MGM4Y2RmYWU3NDcwNWExNjkwODMyNjgyYTZkNGQpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl9lMDVhMWFlMzQ1ZmY0OTk2OWFkMThlNTczYWY5YTQ1MC5iaW5kUG9wdXAocG9wdXBfOWQ1MGU1MjIxODI5NGQ2ODg0ZmFjOTg4MGYzZjVjMTgpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfOGEwMzFlYTllNDFmNDZiMTg0OGM3NTU1ZGM4ZTEzOGQgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4zMzMzNTExLDExNC4yMDkwODY0XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2JmZDBhZDgxYzQyYjRhNmJhMGNlNDExYjk4OTA1ZWFhID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdncmVlbicsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl84YTAzMWVhOWU0MWY0NmIxODQ4Yzc1NTVkYzhlMTM4ZC5zZXRJY29uKGljb25fYmZkMGFkODFjNDJiNGE2YmEwY2U0MTFiOTg5MDVlYWEpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNjI5MzQ1NDYwMDFmNDI5YTgxYzFiNzRlODY4ZThjOWEgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNWYwMmI1YjU2OTYwNGNlZmE5ZWY0NmYxZTU5NTY2ZTggPSAkKCc8ZGl2IGlkPSJodG1sXzVmMDJiNWI1Njk2MDRjZWZhOWVmNDZmMWU1OTU2NmU4IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMzEwMSwgTEFCRUw6MiwgU0VHTUVOVDpXVEMgLSBTYXZlcnM8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzYyOTM0NTQ2MDAxZjQyOWE4MWMxYjc0ZTg2OGU4YzlhLnNldENvbnRlbnQoaHRtbF81ZjAyYjViNTY5NjA0Y2VmYTllZjQ2ZjFlNTk1NjZlOCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzhhMDMxZWE5ZTQxZjQ2YjE4NDhjNzU1NWRjOGUxMzhkLmJpbmRQb3B1cChwb3B1cF82MjkzNDU0NjAwMWY0MjlhODFjMWI3NGU4NjhlOGM5YSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl81YzE5ZmFjYmRiZmU0ZjhkOTRlZThlMjViYzBmMmZiZSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjM1NTc0OTEsMTE0LjEwODMwMTZdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fMjdiMjBmODI0MzA4NDdmZmE2MjcwNmI1NWVlOTc2YWUgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3JlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl81YzE5ZmFjYmRiZmU0ZjhkOTRlZThlMjViYzBmMmZiZS5zZXRJY29uKGljb25fMjdiMjBmODI0MzA4NDdmZmE2MjcwNmI1NWVlOTc2YWUpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfZWVmOTdlODNkZTdmNDJlYWE1M2M0ZDNkM2M0ZjM4NDYgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfMDkxMmU4ZmZjNDQyNDdhZjhkZGNmNGRlYjg0MTQ0ZWMgPSAkKCc8ZGl2IGlkPSJodG1sXzA5MTJlOGZmYzQ0MjQ3YWY4ZGRjZjRkZWI4NDE0NGVjIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMzE0MiwgTEFCRUw6MCwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfZWVmOTdlODNkZTdmNDJlYWE1M2M0ZDNkM2M0ZjM4NDYuc2V0Q29udGVudChodG1sXzA5MTJlOGZmYzQ0MjQ3YWY4ZGRjZjRkZWI4NDE0NGVjKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfNWMxOWZhY2JkYmZlNGY4ZDk0ZWU4ZTI1YmMwZjJmYmUuYmluZFBvcHVwKHBvcHVwX2VlZjk3ZTgzZGU3ZjQyZWFhNTNjNGQzZDNjNGYzODQ2KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzY4NTI5YjQzZDI5NzQwYTQ4NTY5YjE3ZTc3NGQ2NDJhID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzgzMTkxOSwxMTQuMTg5NDA0NF0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9mYjI1MjkzYzU5NDg0ODQxOGViNzcyNTYzMWM4ODkwMSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnb3JhbmdlJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzY4NTI5YjQzZDI5NzQwYTQ4NTY5YjE3ZTc3NGQ2NDJhLnNldEljb24oaWNvbl9mYjI1MjkzYzU5NDg0ODQxOGViNzcyNTYzMWM4ODkwMSk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF84ZTk3Zjk1ODQ5YTk0ZjcxYTI4MmIyOWExYzAzNGM1NSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF83YWYyNDRiYzI4ZWY0YmZkYTU0YjZiNWViZDVhODgzMCA9ICQoJzxkaXYgaWQ9Imh0bWxfN2FmMjQ0YmMyOGVmNGJmZGE1NGI2YjVlYmQ1YTg4MzAiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjExOTY0LCBMQUJFTDo0LCBTRUdNRU5UOldUQyAtIFRvdXJpc3RzL0NvbnY8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzhlOTdmOTU4NDlhOTRmNzFhMjgyYjI5YTFjMDM0YzU1LnNldENvbnRlbnQoaHRtbF83YWYyNDRiYzI4ZWY0YmZkYTU0YjZiNWViZDVhODgzMCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzY4NTI5YjQzZDI5NzQwYTQ4NTY5YjE3ZTc3NGQ2NDJhLmJpbmRQb3B1cChwb3B1cF84ZTk3Zjk1ODQ5YTk0ZjcxYTI4MmIyOWExYzAzNGM1NSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl84YTBkN2MzMTI5YWM0YzM4YjNhZjdhZmU2N2VhN2E1YSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjQ1MjA4MjksMTEzLjk5OTI2XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2ZjYjkwNGIyYzllZTQ0MmQ4MDhmZmVlM2FkMjkzMzRmID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfOGEwZDdjMzEyOWFjNGMzOGIzYWY3YWZlNjdlYTdhNWEuc2V0SWNvbihpY29uX2ZjYjkwNGIyYzllZTQ0MmQ4MDhmZmVlM2FkMjkzMzRmKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2M5MmNiNjljMzE2YjQxYjc5ZmUxMWRiMmNjOWMyNzc2ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2M3MGQ2MDQ5NzUzNDQ0Y2JhOGE2MzIwZDA3N2ViNzczID0gJCgnPGRpdiBpZD0iaHRtbF9jNzBkNjA0OTc1MzQ0NGNiYThhNjMyMGQwNzdlYjc3MyIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIwNjAsIExBQkVMOjAsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2M5MmNiNjljMzE2YjQxYjc5ZmUxMWRiMmNjOWMyNzc2LnNldENvbnRlbnQoaHRtbF9jNzBkNjA0OTc1MzQ0NGNiYThhNjMyMGQwNzdlYjc3Myk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzhhMGQ3YzMxMjlhYzRjMzhiM2FmN2FmZTY3ZWE3YTVhLmJpbmRQb3B1cChwb3B1cF9jOTJjYjY5YzMxNmI0MWI3OWZlMTFkYjJjYzljMjc3Nik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9hY2ViNzIzZTM1Y2E0MGVkOGNlZWE2MjQ2MzYzYTdjMiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjM3MDE1OTgsMTE0LjExNDEwMTNdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fNzg3ZjcxNzM0NmZlNGIxNWJjYWYwYWI4YTMzYmNjMGIgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2xpZ2h0cmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyX2FjZWI3MjNlMzVjYTQwZWQ4Y2VlYTYyNDYzNjNhN2MyLnNldEljb24oaWNvbl83ODdmNzE3MzQ2ZmU0YjE1YmNhZjBhYjhhMzNiY2MwYik7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8zMzZlZDAzZWU5MzM0NTc5ODBlZWNiYjk5N2Q2ZGUwZSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF83MzkyZWMyYzVkZTE0MTdhOWJiMTNjMWM0NzZiOTA2OCA9ICQoJzxkaXYgaWQ9Imh0bWxfNzM5MmVjMmM1ZGUxNDE3YTliYjEzYzFjNDc2YjkwNjgiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMDY3LCBMQUJFTDo2LCBTRUdNRU5UOldUQyAtIFRvdXJpc3RzL0NvbnY8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzMzNmVkMDNlZTkzMzQ1Nzk4MGVlY2JiOTk3ZDZkZTBlLnNldENvbnRlbnQoaHRtbF83MzkyZWMyYzVkZTE0MTdhOWJiMTNjMWM0NzZiOTA2OCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyX2FjZWI3MjNlMzVjYTQwZWQ4Y2VlYTYyNDYzNjNhN2MyLmJpbmRQb3B1cChwb3B1cF8zMzZlZDAzZWU5MzM0NTc5ODBlZWNiYjk5N2Q2ZGUwZSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl8zN2Q1OTgyYzgyMWM0OGZjOGZmNmNiZGVmZjg4ZTIyMyA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjMxNjc5NzQsMTE0LjE2MTQ2OTJdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fYWViODFiYzlmM2FiNDJjOWE1MDJjY2RiM2NhMGJhNmYgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2xpZ2h0cmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzM3ZDU5ODJjODIxYzQ4ZmM4ZmY2Y2JkZWZmODhlMjIzLnNldEljb24oaWNvbl9hZWI4MWJjOWYzYWI0MmM5YTUwMmNjZGIzY2EwYmE2Zik7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF83YzRiYmZiZTczNzQ0YWE4ODJlYjY5MDUwMDFiNDExOSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF81Njk4ZWI0ODE3NmU0MTI1OTY5MjkzNTBiNzU5OTVjYSA9ICQoJzxkaXYgaWQ9Imh0bWxfNTY5OGViNDgxNzZlNDEyNTk2OTI5MzUwYjc1OTk1Y2EiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMTMwLCBMQUJFTDo2LCBTRUdNRU5UOldUQyAtIFJlc2lkZW50aWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF83YzRiYmZiZTczNzQ0YWE4ODJlYjY5MDUwMDFiNDExOS5zZXRDb250ZW50KGh0bWxfNTY5OGViNDgxNzZlNDEyNTk2OTI5MzUwYjc1OTk1Y2EpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl8zN2Q1OTgyYzgyMWM0OGZjOGZmNmNiZGVmZjg4ZTIyMy5iaW5kUG9wdXAocG9wdXBfN2M0YmJmYmU3Mzc0NGFhODgyZWI2OTA1MDAxYjQxMTkpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfYzYzZDRlZTkxOTA1NDljOWI0NzIzZjgxM2E2OGMwMjggPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4yNjM5OTEzLDExNC4yMzcwNzM5XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzkxODA4NjBiZjk0ZTRlMzViZGI2ODU0OTIzMDYwNWVjID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfYzYzZDRlZTkxOTA1NDljOWI0NzIzZjgxM2E2OGMwMjguc2V0SWNvbihpY29uXzkxODA4NjBiZjk0ZTRlMzViZGI2ODU0OTIzMDYwNWVjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzk5ZjJlMjljNGI1ODQ0NWZiZDk0YmZhY2YwYTYwNGM5ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzMwNjVjN2E4ZTc1YjQyNGE4NWE3NDc4MWEwZTNkMzM2ID0gJCgnPGRpdiBpZD0iaHRtbF8zMDY1YzdhOGU3NWI0MjRhODVhNzQ3ODFhMGUzZDMzNiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIyNTAsIExBQkVMOjAsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzk5ZjJlMjljNGI1ODQ0NWZiZDk0YmZhY2YwYTYwNGM5LnNldENvbnRlbnQoaHRtbF8zMDY1YzdhOGU3NWI0MjRhODVhNzQ3ODFhMGUzZDMzNik7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyX2M2M2Q0ZWU5MTkwNTQ5YzliNDcyM2Y4MTNhNjhjMDI4LmJpbmRQb3B1cChwb3B1cF85OWYyZTI5YzRiNTg0NDVmYmQ5NGJmYWNmMGE2MDRjOSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl81ZTI2NWQzMDQyZGY0MTA4OTFhNjQ0MDVlYjljZDYyZSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjM4MzY4OSwxMTQuMjcwNzg2N10sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9mZmIwNzkzMTQ3Yzc0Mzc3YTIyMjUyZTEwODM5ZWE3MCA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAncmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzVlMjY1ZDMwNDJkZjQxMDg5MWE2NDQwNWViOWNkNjJlLnNldEljb24oaWNvbl9mZmIwNzkzMTQ3Yzc0Mzc3YTIyMjUyZTEwODM5ZWE3MCk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9lODQ3NWE1YmM3MGQ0MjU0YmNlZjc4ZDcxYjJjODM0OCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF82ZjRhZDY3OWVhN2I0MmViOGVkODUyYWYzZTMyZDZjYiA9ICQoJzxkaXYgaWQ9Imh0bWxfNmY0YWQ2NzllYTdiNDJlYjhlZDg1MmFmM2UzMmQ2Y2IiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMjY5LCBMQUJFTDowLCBTRUdNRU5UOldUQyAtIFJlc2lkZW50aWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9lODQ3NWE1YmM3MGQ0MjU0YmNlZjc4ZDcxYjJjODM0OC5zZXRDb250ZW50KGh0bWxfNmY0YWQ2NzllYTdiNDJlYjhlZDg1MmFmM2UzMmQ2Y2IpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl81ZTI2NWQzMDQyZGY0MTA4OTFhNjQ0MDVlYjljZDYyZS5iaW5kUG9wdXAocG9wdXBfZTg0NzVhNWJjNzBkNDI1NGJjZWY3OGQ3MWIyYzgzNDgpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfOTdmYzBmN2RkNmNjNDY2Zjk1MWU2MDlmOWEyZGNiN2YgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4zMDYzMDg1LDExNC4yNjA4NTE0XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2FlMzUwN2QyNjc5MDQ4OWY5OTU3MzgxYjE0MjQ5M2I4ID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdwdXJwbGUnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfOTdmYzBmN2RkNmNjNDY2Zjk1MWU2MDlmOWEyZGNiN2Yuc2V0SWNvbihpY29uX2FlMzUwN2QyNjc5MDQ4OWY5OTU3MzgxYjE0MjQ5M2I4KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzc4MzIxODg3Y2FlZTQ2ZGRhODUwNTI5ZTZkMDBiYTYyID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2FiZDQ5NGM3Y2ZjMTRmMTI5NDQzOTI3NWYyYWYwMDA0ID0gJCgnPGRpdiBpZD0iaHRtbF9hYmQ0OTRjN2NmYzE0ZjEyOTQ0MzkyNzVmMmFmMDAwNCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIyOTUsIExBQkVMOjMsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzc4MzIxODg3Y2FlZTQ2ZGRhODUwNTI5ZTZkMDBiYTYyLnNldENvbnRlbnQoaHRtbF9hYmQ0OTRjN2NmYzE0ZjEyOTQ0MzkyNzVmMmFmMDAwNCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzk3ZmMwZjdkZDZjYzQ2NmY5NTFlNjA5ZjlhMmRjYjdmLmJpbmRQb3B1cChwb3B1cF83ODMyMTg4N2NhZWU0NmRkYTg1MDUyOWU2ZDAwYmE2Mik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl82Mzg2MDlkMmRiMTU0YzhkYjBmNGE1ODRjNjk0ODE5OCA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjMxODYxMDgsMTE0LjIzNDg5NjldLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fYWZmYmEwY2RhN2ViNDY1ZThjZDA2ZDlmNmUyYTJkZjAgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3JlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl82Mzg2MDlkMmRiMTU0YzhkYjBmNGE1ODRjNjk0ODE5OC5zZXRJY29uKGljb25fYWZmYmEwY2RhN2ViNDY1ZThjZDA2ZDlmNmUyYTJkZjApOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfN2I2OTcxNTZjMjcwNGE4NGE2ODFkZWRhNGY2NzhjN2IgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfYmZhYzhlMjM0M2QzNDg4NGEyMDNjZTNmNDA0ZThiM2IgPSAkKCc8ZGl2IGlkPSJodG1sX2JmYWM4ZTIzNDNkMzQ4ODRhMjAzY2UzZjQwNGU4YjNiIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjQ4MiwgTEFCRUw6MCwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfN2I2OTcxNTZjMjcwNGE4NGE2ODFkZWRhNGY2NzhjN2Iuc2V0Q29udGVudChodG1sX2JmYWM4ZTIzNDNkMzQ4ODRhMjAzY2UzZjQwNGU4YjNiKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfNjM4NjA5ZDJkYjE1NGM4ZGIwZjRhNTg0YzY5NDgxOTguYmluZFBvcHVwKHBvcHVwXzdiNjk3MTU2YzI3MDRhODRhNjgxZGVkYTRmNjc4YzdiKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzQwM2U0ZDFkZWU4YjQ4YTQ5NzkwOTE1MjlhYTU3N2U0ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMjkxNTA2OSwxMTQuMTk2NTI0M10sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl8yNWNjY2FlMTU1MTU0MGZhYmY3ZTE5YWE4ZGE2ZTY3NyA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAncmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzQwM2U0ZDFkZWU4YjQ4YTQ5NzkwOTE1MjlhYTU3N2U0LnNldEljb24oaWNvbl8yNWNjY2FlMTU1MTU0MGZhYmY3ZTE5YWE4ZGE2ZTY3Nyk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9hZTNiNDE2MTZlMDE0OWE0Yjc5MDU1N2ZhNDViODEzOCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF81MTNlODkxZmZlYWE0ZWFiODFmMmE5NzNjZWE3M2Q0NiA9ICQoJzxkaXYgaWQ9Imh0bWxfNTEzZTg5MWZmZWFhNGVhYjgxZjJhOTczY2VhNzNkNDYiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMTkzLCBMQUJFTDowLCBTRUdNRU5UOldUQyAtIFNhdmVyczwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfYWUzYjQxNjE2ZTAxNDlhNGI3OTA1NTdmYTQ1YjgxMzguc2V0Q29udGVudChodG1sXzUxM2U4OTFmZmVhYTRlYWI4MWYyYTk3M2NlYTczZDQ2KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfNDAzZTRkMWRlZThiNDhhNDk3OTA5MTUyOWFhNTc3ZTQuYmluZFBvcHVwKHBvcHVwX2FlM2I0MTYxNmUwMTQ5YTRiNzkwNTU3ZmE0NWI4MTM4KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2M3ZjZmODJmZDZhNjQ4OWI5YmJmNWM2ZDRjNjI0Mjc4ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzEyNDc2LDExNC4xNzEwMjZdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fNjFjZmUxZGE1YjUzNDc0ZjliM2FmM2RmY2IyZGQwNTEgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2xpZ2h0cmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyX2M3ZjZmODJmZDZhNjQ4OWI5YmJmNWM2ZDRjNjI0Mjc4LnNldEljb24oaWNvbl82MWNmZTFkYTViNTM0NzRmOWIzYWYzZGZjYjJkZDA1MSk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9iOGU4ZTJmNTIyNzU0YzE1ODcxMjI5MmIyYTU5MmRjMiA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9kMTUzMWI0N2FiNzI0N2I4Yjk1N2IzNmQwYTU2N2JmNCA9ICQoJzxkaXYgaWQ9Imh0bWxfZDE1MzFiNDdhYjcyNDdiOGI5NTdiMzZkMGE1NjdiZjQiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMjcyLCBMQUJFTDo2LCBTRUdNRU5UOldUQyAtIFRvdXJpc3RzL0NvbnY8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2I4ZThlMmY1MjI3NTRjMTU4NzEyMjkyYjJhNTkyZGMyLnNldENvbnRlbnQoaHRtbF9kMTUzMWI0N2FiNzI0N2I4Yjk1N2IzNmQwYTU2N2JmNCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyX2M3ZjZmODJmZDZhNjQ4OWI5YmJmNWM2ZDRjNjI0Mjc4LmJpbmRQb3B1cChwb3B1cF9iOGU4ZTJmNTIyNzU0YzE1ODcxMjI5MmIyYTU5MmRjMik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl85YjNlZDA5YTE1NGE0M2QyOGE4NmI3MjI2NDNlMmMxOCA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzMyLjgwODQzNiwtOTcuMzM2NjU4XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2ViMTJlMzEzZjU0YTQ1MjI5NjRmZDYwMWZkODUzYWRiID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfOWIzZWQwOWExNTRhNDNkMjhhODZiNzIyNjQzZTJjMTguc2V0SWNvbihpY29uX2ViMTJlMzEzZjU0YTQ1MjI5NjRmZDYwMWZkODUzYWRiKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzdiYjg1MGU3NGE2OTQ5NThhOTI4NDY4NTAwYzMxMDM4ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2Y3YzE0M2M2ZmQzMDRmZWE4NjE2MGM2OTczYjI4Yzk1ID0gJCgnPGRpdiBpZD0iaHRtbF9mN2MxNDNjNmZkMzA0ZmVhODYxNjBjNjk3M2IyOGM5NSIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIyMzksIExBQkVMOjAsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzdiYjg1MGU3NGE2OTQ5NThhOTI4NDY4NTAwYzMxMDM4LnNldENvbnRlbnQoaHRtbF9mN2MxNDNjNmZkMzA0ZmVhODYxNjBjNjk3M2IyOGM5NSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzliM2VkMDlhMTU0YTQzZDI4YTg2YjcyMjY0M2UyYzE4LmJpbmRQb3B1cChwb3B1cF83YmI4NTBlNzRhNjk0OTU4YTkyODQ2ODUwMGMzMTAzOCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl84NTg1NzdmNmMzYWQ0ZjlkOTcyYjg1YjU3NWRkMmE5ZiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzI2LjY0MjUzMzIsLTgxLjg2Mjg1NDRdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fYzE3OWMzNDk4ZjJlNDA2MDg2NGJiM2Y0ZjA5NjBmZTEgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3B1cnBsZScsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl84NTg1NzdmNmMzYWQ0ZjlkOTcyYjg1YjU3NWRkMmE5Zi5zZXRJY29uKGljb25fYzE3OWMzNDk4ZjJlNDA2MDg2NGJiM2Y0ZjA5NjBmZTEpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNmQ2YzRjY2RmMWNiNDU1YTg2YTkwMmJmNTFkNTdmOWQgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfMzAxZWZhMThiZGQxNDJkMGI5MGI2YzdiOGUzNTNlNjkgPSAkKCc8ZGl2IGlkPSJodG1sXzMwMWVmYTE4YmRkMTQyZDBiOTBiNmM3YjhlMzUzZTY5IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjQ0MiwgTEFCRUw6MywgU0VHTUVOVDpXVEMgLSBTYXZlcnM8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzZkNmM0Y2NkZjFjYjQ1NWE4NmE5MDJiZjUxZDU3ZjlkLnNldENvbnRlbnQoaHRtbF8zMDFlZmExOGJkZDE0MmQwYjkwYjZjN2I4ZTM1M2U2OSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzg1ODU3N2Y2YzNhZDRmOWQ5NzJiODViNTc1ZGQyYTlmLmJpbmRQb3B1cChwb3B1cF82ZDZjNGNjZGYxY2I0NTVhODZhOTAyYmY1MWQ1N2Y5ZCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl81MmY2N2FiMjVlNjY0YTQxOGFkYmU1Y2E4OGNlYmQ4OCA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjQ2MTEwOCwxMTMuOTk3MzQzXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2I5OGEyYzczZmFhOTQxYTlhZDIxZmUwZWRlM2QyZmRjID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfNTJmNjdhYjI1ZTY2NGE0MThhZGJlNWNhODhjZWJkODguc2V0SWNvbihpY29uX2I5OGEyYzczZmFhOTQxYTlhZDIxZmUwZWRlM2QyZmRjKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2FkZWMwMjVkNTM2YjRlNmY4YmVkMDgyNGYyOWIwMzg4ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzY5MDZmNzRkYzBkNDQ4MDM4NjNhZjM0MGE5MTkzMTc4ID0gJCgnPGRpdiBpZD0iaHRtbF82OTA2Zjc0ZGMwZDQ0ODAzODYzYWYzNDBhOTE5MzE3OCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIwOTEsIExBQkVMOjAsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2FkZWMwMjVkNTM2YjRlNmY4YmVkMDgyNGYyOWIwMzg4LnNldENvbnRlbnQoaHRtbF82OTA2Zjc0ZGMwZDQ0ODAzODYzYWYzNDBhOTE5MzE3OCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzUyZjY3YWIyNWU2NjRhNDE4YWRiZTVjYTg4Y2ViZDg4LmJpbmRQb3B1cChwb3B1cF9hZGVjMDI1ZDUzNmI0ZTZmOGJlZDA4MjRmMjliMDM4OCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl8zOTE1OTk1NzI5YmQ0MjEwYjcxYzEzZDVjMjQxNjUxOCA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjI3NzQ5ODUsMTE0LjE2NjMyMjVdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fM2ExNjRjYWZkYWNlNDM2NzgwNDJmZjQ4ZDM1ZjdlMzAgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2dyZWVuJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzM5MTU5OTU3MjliZDQyMTBiNzFjMTNkNWMyNDE2NTE4LnNldEljb24oaWNvbl8zYTE2NGNhZmRhY2U0MzY3ODA0MmZmNDhkMzVmN2UzMCk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9iMDQzZDQxMjg1Zjg0ZjdlODBiMjIxMjM5YzY1MWEyOSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9hYWZkODM3YjRjMmI0MjYxOTdkZTQzOWVmMWY1NzZmYSA9ICQoJzxkaXYgaWQ9Imh0bWxfYWFmZDgzN2I0YzJiNDI2MTk3ZGU0MzllZjFmNTc2ZmEiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMzg3LCBMQUJFTDoyLCBTRUdNRU5UOldUQyAtIEhlYWx0aDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfYjA0M2Q0MTI4NWY4NGY3ZTgwYjIyMTIzOWM2NTFhMjkuc2V0Q29udGVudChodG1sX2FhZmQ4MzdiNGMyYjQyNjE5N2RlNDM5ZWYxZjU3NmZhKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfMzkxNTk5NTcyOWJkNDIxMGI3MWMxM2Q1YzI0MTY1MTguYmluZFBvcHVwKHBvcHVwX2IwNDNkNDEyODVmODRmN2U4MGIyMjEyMzljNjUxYTI5KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzFiOWQzNDE5YTQxNzQ3MmU4M2Y2YjhkZTVmMDA5ZWE5ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzgyNTY5MSwxMTQuMTg3NTExMV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl8wYWRhN2YwY2ZkOTk0NmI3YjdlNjJkNmY1NmNkNTM2OCA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnZGFya2JsdWUnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfMWI5ZDM0MTlhNDE3NDcyZTgzZjZiOGRlNWYwMDllYTkuc2V0SWNvbihpY29uXzBhZGE3ZjBjZmQ5OTQ2YjdiN2U2MmQ2ZjU2Y2Q1MzY4KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzA1NDI1YWM0MGIwYzQ3MzRiODJmNzczZTBhNmQ3Mzc4ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzQzNWM4ZDI4MjQ4MzQ3YzdiNTIxODk5NWVmMWQ0MGI3ID0gJCgnPGRpdiBpZD0iaHRtbF80MzVjOGQyODI0ODM0N2M3YjUyMTg5OTVlZjFkNDBiNyIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTMyMDEsIExBQkVMOjgsIFNFR01FTlQ6V1RDIC0gQmVhdXRpcTwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMDU0MjVhYzQwYjBjNDczNGI4MmY3NzNlMGE2ZDczNzguc2V0Q29udGVudChodG1sXzQzNWM4ZDI4MjQ4MzQ3YzdiNTIxODk5NWVmMWQ0MGI3KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfMWI5ZDM0MTlhNDE3NDcyZTgzZjZiOGRlNWYwMDllYTkuYmluZFBvcHVwKHBvcHVwXzA1NDI1YWM0MGIwYzQ3MzRiODJmNzczZTBhNmQ3Mzc4KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2MxZGM5MmIxZWFkNzQ2ZWI5NzdlMjhkZjRiOWNlN2JmID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzI0NDc4MSwxMTQuMjE2MzU1Nl0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl84ZTdjZWZjYzk2ZTk0N2NmODkwMDc1NmM2MzI3NjlkZCA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAncmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyX2MxZGM5MmIxZWFkNzQ2ZWI5NzdlMjhkZjRiOWNlN2JmLnNldEljb24oaWNvbl84ZTdjZWZjYzk2ZTk0N2NmODkwMDc1NmM2MzI3NjlkZCk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF80OTZkMTI5MzBjZjU0ZjNjYjQ2Y2VkZWI5Mjc2MzE3OSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9mNjAzMWEzZDQyMTA0NGU0OTVmZjk5NzM2YmJkOWY5OSA9ICQoJzxkaXYgaWQ9Imh0bWxfZjYwMzFhM2Q0MjEwNDRlNDk1ZmY5OTczNmJiZDlmOTkiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMTI4LCBMQUJFTDowLCBTRUdNRU5UOldUQyAtIFJlc2lkZW50aWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF80OTZkMTI5MzBjZjU0ZjNjYjQ2Y2VkZWI5Mjc2MzE3OS5zZXRDb250ZW50KGh0bWxfZjYwMzFhM2Q0MjEwNDRlNDk1ZmY5OTczNmJiZDlmOTkpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl9jMWRjOTJiMWVhZDc0NmViOTc3ZTI4ZGY0YjljZTdiZi5iaW5kUG9wdXAocG9wdXBfNDk2ZDEyOTMwY2Y1NGYzY2I0NmNlZGViOTI3NjMxNzkpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfNjYxNmZhZGMxY2U1NDZmOWJjNmQxMzI0NWU3NWZhYWQgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi40NDg1NDU1LDExNC4xNjYyNzYxXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzc1MzVlMDMxMDgxMDQzZDQ5NzllMzE4Mzg4ZGYzODIzID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdiZWlnZScsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl82NjE2ZmFkYzFjZTU0NmY5YmM2ZDEzMjQ1ZTc1ZmFhZC5zZXRJY29uKGljb25fNzUzNWUwMzEwODEwNDNkNDk3OWUzMTgzODhkZjM4MjMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNGUyMWY0NTgzZWNiNDM2OTgyZWVmMWE1NzUwMjFhNTcgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfN2VhNmQyYWE5YTVkNDQ2YWE1OTM0Njg4OTJhMzExZDcgPSAkKCc8ZGl2IGlkPSJodG1sXzdlYTZkMmFhOWE1ZDQ0NmFhNTkzNDY4ODkyYTMxMWQ3IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMTk1OCwgTEFCRUw6NywgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNGUyMWY0NTgzZWNiNDM2OTgyZWVmMWE1NzUwMjFhNTcuc2V0Q29udGVudChodG1sXzdlYTZkMmFhOWE1ZDQ0NmFhNTkzNDY4ODkyYTMxMWQ3KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfNjYxNmZhZGMxY2U1NDZmOWJjNmQxMzI0NWU3NWZhYWQuYmluZFBvcHVwKHBvcHVwXzRlMjFmNDU4M2VjYjQzNjk4MmVlZjFhNTc1MDIxYTU3KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2M0ODFmMWNlMWM4MTQxZDg4YzFiMmFkNzRhOTY1ZWE1ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMjc2NjE5NSwxMTQuMTczMjYyNF0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl8wNWExZWE3MjQ0YmE0M2Q5YWU5Njg4NjgzYTBhM2YxZiA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnbGlnaHRyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfYzQ4MWYxY2UxYzgxNDFkODhjMWIyYWQ3NGE5NjVlYTUuc2V0SWNvbihpY29uXzA1YTFlYTcyNDRiYTQzZDlhZTk2ODg2ODNhMGEzZjFmKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzY1NWUzNGEyNzlkOTQ0M2E4NTRhYTg3M2EzZjg0NDljID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2NjYjRkNTM1YzFmYTRlMWU4NGRkOTI1ODI2Mzg2MWU2ID0gJCgnPGRpdiBpZD0iaHRtbF9jY2I0ZDUzNWMxZmE0ZTFlODRkZDkyNTgyNjM4NjFlNiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTE5NzMsIExBQkVMOjYsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzY1NWUzNGEyNzlkOTQ0M2E4NTRhYTg3M2EzZjg0NDljLnNldENvbnRlbnQoaHRtbF9jY2I0ZDUzNWMxZmE0ZTFlODRkZDkyNTgyNjM4NjFlNik7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyX2M0ODFmMWNlMWM4MTQxZDg4YzFiMmFkNzRhOTY1ZWE1LmJpbmRQb3B1cChwb3B1cF82NTVlMzRhMjc5ZDk0NDNhODU0YWE4NzNhM2Y4NDQ5Yyk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9iZTUyNDliZjZhMTk0NGMyODQ4M2UwM2FkYWI0NTc4NiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjM3NjQwMTIsMTE0LjExMTU0OF0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl85MmJjZDliMDIwZWE0MTg1OTE5NTIwMTVjOWNjMmY4MSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAncmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyX2JlNTI0OWJmNmExOTQ0YzI4NDgzZTAzYWRhYjQ1Nzg2LnNldEljb24oaWNvbl85MmJjZDliMDIwZWE0MTg1OTE5NTIwMTVjOWNjMmY4MSk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8wM2IwYTM5ODkwOTQ0MDgxYWU1MzU2ZmU1YmNlNzkwMyA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9jM2IwZTIzMzBiMjE0ZjJhYTAxZjlhMTY1Yjg4YzVmOCA9ICQoJzxkaXYgaWQ9Imh0bWxfYzNiMGUyMzMwYjIxNGYyYWEwMWY5YTE2NWI4OGM1ZjgiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjExNzYyLCBMQUJFTDowLCBTRUdNRU5UOldUQyAtIFJlc2lkZW50aWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8wM2IwYTM5ODkwOTQ0MDgxYWU1MzU2ZmU1YmNlNzkwMy5zZXRDb250ZW50KGh0bWxfYzNiMGUyMzMwYjIxNGYyYWEwMWY5YTE2NWI4OGM1ZjgpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl9iZTUyNDliZjZhMTk0NGMyODQ4M2UwM2FkYWI0NTc4Ni5iaW5kUG9wdXAocG9wdXBfMDNiMGEzOTg5MDk0NDA4MWFlNTM1NmZlNWJjZTc5MDMpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfZTE4NTdiYTcyYmI1NDIyYmI2Y2JkMTdkYjNiZTg0NmQgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4zNzA3OTEsMTE0LjA5OTI4N10sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl83NTQzMzk1ODI0Yzc0MTA0OWZkNTRkYWFlMmZlNGRkMyA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnYmVpZ2UnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfZTE4NTdiYTcyYmI1NDIyYmI2Y2JkMTdkYjNiZTg0NmQuc2V0SWNvbihpY29uXzc1NDMzOTU4MjRjNzQxMDQ5ZmQ1NGRhYWUyZmU0ZGQzKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2RmODM1YzhhNDJhODRlY2E4N2QyYTdjNjA5OTEzMWI1ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2EyYmYyMGU2Yjg1YTRiZTRhMzNlN2I5ZmI0ODg2NmFmID0gJCgnPGRpdiBpZD0iaHRtbF9hMmJmMjBlNmI4NWE0YmU0YTMzZTdiOWZiNDg4NjZhZiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTMxMDIsIExBQkVMOjcsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2RmODM1YzhhNDJhODRlY2E4N2QyYTdjNjA5OTEzMWI1LnNldENvbnRlbnQoaHRtbF9hMmJmMjBlNmI4NWE0YmU0YTMzZTdiOWZiNDg4NjZhZik7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyX2UxODU3YmE3MmJiNTQyMmJiNmNiZDE3ZGIzYmU4NDZkLmJpbmRQb3B1cChwb3B1cF9kZjgzNWM4YTQyYTg0ZWNhODdkMmE3YzYwOTkxMzFiNSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9kMjA1ZmZmZjViOGE0NDFiYTI3NDc0NDcyMDhiZTA0MiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjI4MTUyNzksMTE0LjE4NDIxODldLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fYmJiYWRiNzA3NDM5NGIzZmJkMjA3OWE1NjlmMTBhMjYgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2RhcmtibHVlJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyX2QyMDVmZmZmNWI4YTQ0MWJhMjc0NzQ0NzIwOGJlMDQyLnNldEljb24oaWNvbl9iYmJhZGI3MDc0Mzk0YjNmYmQyMDc5YTU2OWYxMGEyNik7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9mNDBkYWM0NDE5N2E0NTAyYjUyN2NjYzI1NTZiZDc0MiA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9iNzMyMjNkYzQzZmU0OTM1ODU2MjQyYzdjNDMzYWE3NyA9ICQoJzxkaXYgaWQ9Imh0bWxfYjczMjIzZGM0M2ZlNDkzNTg1NjI0MmM3YzQzM2FhNzciIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjExOTc1LCBMQUJFTDo4LCBTRUdNRU5UOldUQyAtIEJlYXV0aXE8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2Y0MGRhYzQ0MTk3YTQ1MDJiNTI3Y2NjMjU1NmJkNzQyLnNldENvbnRlbnQoaHRtbF9iNzMyMjNkYzQzZmU0OTM1ODU2MjQyYzdjNDMzYWE3Nyk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyX2QyMDVmZmZmNWI4YTQ0MWJhMjc0NzQ0NzIwOGJlMDQyLmJpbmRQb3B1cChwb3B1cF9mNDBkYWM0NDE5N2E0NTAyYjUyN2NjYzI1NTZiZDc0Mik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl85ZTllMDI4MDdkNDY0OTllYjE2ZWVjZGJhNjMzMTMwOSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjMyNDM5NjUsMTE0LjIxMjU2NjNdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fMWQyNGMwNmYyZTIzNGNjZWIyYTM1NGRjMmQ5MTJjODcgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3B1cnBsZScsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl85ZTllMDI4MDdkNDY0OTllYjE2ZWVjZGJhNjMzMTMwOS5zZXRJY29uKGljb25fMWQyNGMwNmYyZTIzNGNjZWIyYTM1NGRjMmQ5MTJjODcpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfZDc4MTFmZDA0OTEwNGY4Zjg0NWUwNTBkMjc2N2QzNjQgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfZGM0Y2ViOWYzMTk0NGJlZWE5ZjJkYmM1MWQ2ZTNmZDcgPSAkKCc8ZGl2IGlkPSJodG1sX2RjNGNlYjlmMzE5NDRiZWVhOWYyZGJjNTFkNmUzZmQ3IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjMyOSwgTEFCRUw6MywgU0VHTUVOVDpXVEMgLSBEZXN0aW5hdGlvbjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfZDc4MTFmZDA0OTEwNGY4Zjg0NWUwNTBkMjc2N2QzNjQuc2V0Q29udGVudChodG1sX2RjNGNlYjlmMzE5NDRiZWVhOWYyZGJjNTFkNmUzZmQ3KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfOWU5ZTAyODA3ZDQ2NDk5ZWIxNmVlY2RiYTYzMzEzMDkuYmluZFBvcHVwKHBvcHVwX2Q3ODExZmQwNDkxMDRmOGY4NDVlMDUwZDI3NjdkMzY0KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzlhZDAzZTIyZGRiNjQ0OTdiYmYxM2JhMjM3OWQyNzY5ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzcxNzA1MywxMTMuOTg5NDcxMl0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9lMjdjZGNkOWIxYWY0YjAyOWY4MTBiMTcxZTUwMTVmZSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnYmVpZ2UnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfOWFkMDNlMjJkZGI2NDQ5N2JiZjEzYmEyMzc5ZDI3Njkuc2V0SWNvbihpY29uX2UyN2NkY2Q5YjFhZjRiMDI5ZjgxMGIxNzFlNTAxNWZlKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2JkNDIwNDU3Y2ZhZTQ2YjI5OGE5ZTBiMmQ3MTM0ZWMxID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2NjOGQ5Njk1YmI4ODQ3M2Y4Mzg4OTcyNDk4N2NlNmNjID0gJCgnPGRpdiBpZD0iaHRtbF9jYzhkOTY5NWJiODg0NzNmODM4ODk3MjQ5ODdjZTZjYyIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTI2MjEsIExBQkVMOjcsIFNFR01FTlQ6V1RDIC0gVG91cmlzdHMvQ29udjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfYmQ0MjA0NTdjZmFlNDZiMjk4YTllMGIyZDcxMzRlYzEuc2V0Q29udGVudChodG1sX2NjOGQ5Njk1YmI4ODQ3M2Y4Mzg4OTcyNDk4N2NlNmNjKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfOWFkMDNlMjJkZGI2NDQ5N2JiZjEzYmEyMzc5ZDI3NjkuYmluZFBvcHVwKHBvcHVwX2JkNDIwNDU3Y2ZhZTQ2YjI5OGE5ZTBiMmQ3MTM0ZWMxKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzU0OTFiN2RlNjBjYzQ5MDE5ZDZlMmU1MzdiNjgxOTExID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzA4NjgxNCwxMTQuMTcwOTY3XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2FiNzI0YTY5MmZmMzQzN2Y5ZjdjOTM0OGQxZGFmYjQyID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdwdXJwbGUnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfNTQ5MWI3ZGU2MGNjNDkwMTlkNmUyZTUzN2I2ODE5MTEuc2V0SWNvbihpY29uX2FiNzI0YTY5MmZmMzQzN2Y5ZjdjOTM0OGQxZGFmYjQyKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzYzYWY0OTNkOGEwNjQyNzA5YzQ0ODZiOTExYTc2MTM4ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzU3MTBmYmIzNGEzMDQ2NWU5MzNkZDE1MzllNzA2MTYwID0gJCgnPGRpdiBpZD0iaHRtbF81NzEwZmJiMzRhMzA0NjVlOTMzZGQxNTM5ZTcwNjE2MCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIxNjEsIExBQkVMOjMsIFNFR01FTlQ6V1RDIC0gVG91cmlzdHMvQ29udjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNjNhZjQ5M2Q4YTA2NDI3MDljNDQ4NmI5MTFhNzYxMzguc2V0Q29udGVudChodG1sXzU3MTBmYmIzNGEzMDQ2NWU5MzNkZDE1MzllNzA2MTYwKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfNTQ5MWI3ZGU2MGNjNDkwMTlkNmUyZTUzN2I2ODE5MTEuYmluZFBvcHVwKHBvcHVwXzYzYWY0OTNkOGEwNjQyNzA5YzQ0ODZiOTExYTc2MTM4KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2QwNzEzY2VmYTNlMDRkMTk4MmM5NzZlMmQ4OTdiNjA2ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzcxOTUxNSwxMTQuMTAxNzg4N10sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9iNzViOGUzY2U2NzY0YmZjODczYmM1MWZjYjQxNWY0MCA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnYmVpZ2UnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfZDA3MTNjZWZhM2UwNGQxOTgyYzk3NmUyZDg5N2I2MDYuc2V0SWNvbihpY29uX2I3NWI4ZTNjZTY3NjRiZmM4NzNiYzUxZmNiNDE1ZjQwKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzBhNzgwNWJlMjc4MDQxNTA5N2QyZmVlM2Y2NDdhZThjID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2IzNDNmZTM3OWU2OTQwM2E5ZTFjODU2YzJlYmY0Yjc0ID0gJCgnPGRpdiBpZD0iaHRtbF9iMzQzZmUzNzllNjk0MDNhOWUxYzg1NmMyZWJmNGI3NCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTI1MjgsIExBQkVMOjcsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzBhNzgwNWJlMjc4MDQxNTA5N2QyZmVlM2Y2NDdhZThjLnNldENvbnRlbnQoaHRtbF9iMzQzZmUzNzllNjk0MDNhOWUxYzg1NmMyZWJmNGI3NCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyX2QwNzEzY2VmYTNlMDRkMTk4MmM5NzZlMmQ4OTdiNjA2LmJpbmRQb3B1cChwb3B1cF8wYTc4MDViZTI3ODA0MTUwOTdkMmZlZTNmNjQ3YWU4Yyk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9jOTk3MDA0MmQwMDg0Y2RkOWQ0M2IwZDY3YjE0NjVmMiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjMxNjI2MiwxMTQuMjY1MzQxNV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl8yNDI4ZjEzNDExNmY0NTAxYjA3ZjQzZjg3ZmI4MDJhNCA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnZ3JlZW4nLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfYzk5NzAwNDJkMDA4NGNkZDlkNDNiMGQ2N2IxNDY1ZjIuc2V0SWNvbihpY29uXzI0MjhmMTM0MTE2ZjQ1MDFiMDdmNDNmODdmYjgwMmE0KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzBlYTBjODQwMDU1YjRjYjQ5YzkwZGI5Yzk1NjQxOTUyID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzhlZGYyZTYxYWU3NTRkOGNiMzliY2IwNjg1MzRlZmYxID0gJCgnPGRpdiBpZD0iaHRtbF84ZWRmMmU2MWFlNzU0ZDhjYjM5YmNiMDY4NTM0ZWZmMSIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTQwNjEsIExBQkVMOjIsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzBlYTBjODQwMDU1YjRjYjQ5YzkwZGI5Yzk1NjQxOTUyLnNldENvbnRlbnQoaHRtbF84ZWRmMmU2MWFlNzU0ZDhjYjM5YmNiMDY4NTM0ZWZmMSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyX2M5OTcwMDQyZDAwODRjZGQ5ZDQzYjBkNjdiMTQ2NWYyLmJpbmRQb3B1cChwb3B1cF8wZWEwYzg0MDA1NWI0Y2I0OWM5MGRiOWM5NTY0MTk1Mik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl8wMjllMjgwYjE0MGQ0NzM2YTZmMzE4ZjgxNjM2ZjdiNiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjI3MjE2NTEsMTE0LjE4MTA1MDNdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fMDZiZTVkZDczYjc3NGI4MWEwNTJiZDY2NDhlYTdmMmUgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2xpZ2h0cmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzAyOWUyODBiMTQwZDQ3MzZhNmYzMThmODE2MzZmN2I2LnNldEljb24oaWNvbl8wNmJlNWRkNzNiNzc0YjgxYTA1MmJkNjY0OGVhN2YyZSk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8xY2JmYjY4ZmVjY2E0ZjliYWIyMjJlYmQxYjdmMGRkMSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF8wYTFlMDYxNGYxOWU0MWRhOWJiNWUyOTZiMzhmMDc2ZSA9ICQoJzxkaXYgaWQ9Imh0bWxfMGExZTA2MTRmMTllNDFkYTliYjVlMjk2YjM4ZjA3NmUiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMDc2LCBMQUJFTDo2LCBTRUdNRU5UOldUQyAtIFJlc2lkZW50aWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8xY2JmYjY4ZmVjY2E0ZjliYWIyMjJlYmQxYjdmMGRkMS5zZXRDb250ZW50KGh0bWxfMGExZTA2MTRmMTllNDFkYTliYjVlMjk2YjM4ZjA3NmUpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl8wMjllMjgwYjE0MGQ0NzM2YTZmMzE4ZjgxNjM2ZjdiNi5iaW5kUG9wdXAocG9wdXBfMWNiZmI2OGZlY2NhNGY5YmFiMjIyZWJkMWI3ZjBkZDEpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfMjUyMzdjY2U1ZDkzNDg4ZWFhMDk0YTUwMzBkYjY1ZTEgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4yMjA0MjIsMTE0LjIxMjY2ODVdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fZWQ1ZDRhZmY3ODE4NDUzM2I5ZDU2OGI2YTBiMWIxNDMgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3JlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl8yNTIzN2NjZTVkOTM0ODhlYWEwOTRhNTAzMGRiNjVlMS5zZXRJY29uKGljb25fZWQ1ZDRhZmY3ODE4NDUzM2I5ZDU2OGI2YTBiMWIxNDMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNzNlMzQ3NjMyZjE5NGM2NmFmZDQ3Mzg1NTg2NjU0NDEgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNGEyZTBlYzcwYTE3NDQyMmE0OGI0NWQ5Y2RlNzU0ZTcgPSAkKCc8ZGl2IGlkPSJodG1sXzRhMmUwZWM3MGExNzQ0MjJhNDhiNDVkOWNkZTc1NGU3IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMTk0NywgTEFCRUw6MCwgU0VHTUVOVDpXVEMgLSBUb3VyaXN0cy9Db252PC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF83M2UzNDc2MzJmMTk0YzY2YWZkNDczODU1ODY2NTQ0MS5zZXRDb250ZW50KGh0bWxfNGEyZTBlYzcwYTE3NDQyMmE0OGI0NWQ5Y2RlNzU0ZTcpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl8yNTIzN2NjZTVkOTM0ODhlYWEwOTRhNTAzMGRiNjVlMS5iaW5kUG9wdXAocG9wdXBfNzNlMzQ3NjMyZjE5NGM2NmFmZDQ3Mzg1NTg2NjU0NDEpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfNTBkNGIyZjc5YjY4NGIwY2JhOWEwMmEwNWEzNjE4MmYgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4yNzg2MzcxLDExNC4xNzk3MjldLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fODljZTNiNWRhZmJiNDEyMDk5NGE2NTYxYWJmOGJmMTAgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3B1cnBsZScsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl81MGQ0YjJmNzliNjg0YjBjYmE5YTAyYTA1YTM2MTgyZi5zZXRJY29uKGljb25fODljZTNiNWRhZmJiNDEyMDk5NGE2NTYxYWJmOGJmMTApOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNmEzZTRkNzNhNGM0NDE0ZmI5NjRlYzA1MjgyMDRkYjUgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfYjczZjVjZWQ2NDUzNGM3YTkzZTZlNzY4NDlmZmY1ODEgPSAkKCc8ZGl2IGlkPSJodG1sX2I3M2Y1Y2VkNjQ1MzRjN2E5M2U2ZTc2ODQ5ZmZmNTgxIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMTk3NCwgTEFCRUw6MywgU0VHTUVOVDpXVEMgLSBDb21tZXJjaWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF82YTNlNGQ3M2E0YzQ0MTRmYjk2NGVjMDUyODIwNGRiNS5zZXRDb250ZW50KGh0bWxfYjczZjVjZWQ2NDUzNGM3YTkzZTZlNzY4NDlmZmY1ODEpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl81MGQ0YjJmNzliNjg0YjBjYmE5YTAyYTA1YTM2MTgyZi5iaW5kUG9wdXAocG9wdXBfNmEzZTRkNzNhNGM0NDE0ZmI5NjRlYzA1MjgyMDRkYjUpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfYzUzYTY0OTliZTM0NDE0ZjlhMWFjMWYyY2RlNzk2NGUgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFs0MC43MDQ0MjQ2LC03My43MTY4MTM4OTk5OTk5OV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl81NmEwOTJmZWYxMWY0YTYwYjczYjAwYWNlZDgyMGUyOSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnbGlnaHRyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfYzUzYTY0OTliZTM0NDE0ZjlhMWFjMWYyY2RlNzk2NGUuc2V0SWNvbihpY29uXzU2YTA5MmZlZjExZjRhNjBiNzNiMDBhY2VkODIwZTI5KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzg5ZTY0ZWExODFmNjQ1Zjc4NmIyZDg0OTg1OWM3ZjVmID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzU2YjllZWJjMGYyZTQ3MGVhMjU5Y2QxMDUxZDIwYjM0ID0gJCgnPGRpdiBpZD0iaHRtbF81NmI5ZWViYzBmMmU0NzBlYTI1OWNkMTA1MWQyMGIzNCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTE5MTAsIExBQkVMOjYsIFNFR01FTlQ6V1RDIC0gVG91cmlzdHMvQ29udjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfODllNjRlYTE4MWY2NDVmNzg2YjJkODQ5ODU5YzdmNWYuc2V0Q29udGVudChodG1sXzU2YjllZWJjMGYyZTQ3MGVhMjU5Y2QxMDUxZDIwYjM0KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfYzUzYTY0OTliZTM0NDE0ZjlhMWFjMWYyY2RlNzk2NGUuYmluZFBvcHVwKHBvcHVwXzg5ZTY0ZWExODFmNjQ1Zjc4NmIyZDg0OTg1OWM3ZjVmKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzdkMDgzYzE3NWM5MzRiYTJiNzRkOWRhMzY0OTAwY2JkID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzA2MTEwOSwxMTQuMjI3OTE3Nl0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl80Yjg1NzNmMjMxYWE0ZDAyYWJjYjYzYzhlZmM5MTQwNCA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnZ3JlZW4nLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfN2QwODNjMTc1YzkzNGJhMmI3NGQ5ZGEzNjQ5MDBjYmQuc2V0SWNvbihpY29uXzRiODU3M2YyMzFhYTRkMDJhYmNiNjNjOGVmYzkxNDA0KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzExNDBkODExZDZhZTRlNTE5NjVjY2IxYmY0MWI2ZDVkID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzcyZmQxODhlNzhmNzQ3NGY4YjNkZjk3NDQ5NDNkNTlmID0gJCgnPGRpdiBpZD0iaHRtbF83MmZkMTg4ZTc4Zjc0NzRmOGIzZGY5NzQ0OTQzZDU5ZiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIxMDMsIExBQkVMOjIsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzExNDBkODExZDZhZTRlNTE5NjVjY2IxYmY0MWI2ZDVkLnNldENvbnRlbnQoaHRtbF83MmZkMTg4ZTc4Zjc0NzRmOGIzZGY5NzQ0OTQzZDU5Zik7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzdkMDgzYzE3NWM5MzRiYTJiNzRkOWRhMzY0OTAwY2JkLmJpbmRQb3B1cChwb3B1cF8xMTQwZDgxMWQ2YWU0ZTUxOTY1Y2NiMWJmNDFiNmQ1ZCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9hOGE4NmZmYWNhMTA0NGFmYjdmZjYyYTZjYjRiMDY5NiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzMzLjYyMjYzMDUsLTg0LjE5OTE5OTFdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fZWZhYjk1YTk3NjIyNDRmMmFlZjEyZjQwYzM0OWE1YWUgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2xpZ2h0cmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyX2E4YTg2ZmZhY2ExMDQ0YWZiN2ZmNjJhNmNiNGIwNjk2LnNldEljb24oaWNvbl9lZmFiOTVhOTc2MjI0NGYyYWVmMTJmNDBjMzQ5YTVhZSk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF85NjE1NTBjNGFlNzM0MDE2YTQ2ODE3ZDE1YTNjMWIyYiA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9jOWRkYmFiMTViZjI0NmZlOTUyYzVkZDdmMDE1NzliNiA9ICQoJzxkaXYgaWQ9Imh0bWxfYzlkZGJhYjE1YmYyNDZmZTk1MmM1ZGQ3ZjAxNTc5YjYiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMjc0LCBMQUJFTDo2LCBTRUdNRU5UOldUQyAtIFJlc2lkZW50aWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF85NjE1NTBjNGFlNzM0MDE2YTQ2ODE3ZDE1YTNjMWIyYi5zZXRDb250ZW50KGh0bWxfYzlkZGJhYjE1YmYyNDZmZTk1MmM1ZGQ3ZjAxNTc5YjYpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl9hOGE4NmZmYWNhMTA0NGFmYjdmZjYyYTZjYjRiMDY5Ni5iaW5kUG9wdXAocG9wdXBfOTYxNTUwYzRhZTczNDAxNmE0NjgxN2QxNWEzYzFiMmIpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfNmNmYWJjZTkyYWQ3NDk0M2EwMDY2MTY3OTI1YmE4ZjUgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4yNzgyMDc5LDExNC4xODIyOTk0XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzY0MTQ1NzE5NTc2YTRmMDQ5Y2NlMjAwZTg4MTkyYTJkID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdvcmFuZ2UnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfNmNmYWJjZTkyYWQ3NDk0M2EwMDY2MTY3OTI1YmE4ZjUuc2V0SWNvbihpY29uXzY0MTQ1NzE5NTc2YTRmMDQ5Y2NlMjAwZTg4MTkyYTJkKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzgzMTFkMTc1Y2RjMDQxZDNiOTZhNWUyNzM0ZWNhMjEyID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzIwM2QwMTY0NmRiNTRhMjg4ODYyOGE5NDlhYjYwZTgwID0gJCgnPGRpdiBpZD0iaHRtbF8yMDNkMDE2NDZkYjU0YTI4ODg2MjhhOTQ5YWI2MGU4MCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIzODksIExBQkVMOjQsIFNFR01FTlQ6V1RDIC0gSGVhbHRoPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF84MzExZDE3NWNkYzA0MWQzYjk2YTVlMjczNGVjYTIxMi5zZXRDb250ZW50KGh0bWxfMjAzZDAxNjQ2ZGI1NGEyODg4NjI4YTk0OWFiNjBlODApOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl82Y2ZhYmNlOTJhZDc0OTQzYTAwNjYxNjc5MjViYThmNS5iaW5kUG9wdXAocG9wdXBfODMxMWQxNzVjZGMwNDFkM2I5NmE1ZTI3MzRlY2EyMTIpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfMmZiNzY1NGRlYWNiNGYwNGFhMmU5MzJiNjhiZWE2YmIgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4zMjE3NzcsMTE0LjE4OTMyMl0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl82ODgwYjE1OGE0MWM0NDZkOWNiZWJkNWRmMzkxZGQ5NSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAncmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzJmYjc2NTRkZWFjYjRmMDRhYTJlOTMyYjY4YmVhNmJiLnNldEljb24oaWNvbl82ODgwYjE1OGE0MWM0NDZkOWNiZWJkNWRmMzkxZGQ5NSk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF82NTFhMzAyMjdhZTk0M2E2YjdiYTljZDBhNjM1ODM2MyA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF80MTJhNTk1YmUwYmE0YWY0OWE0MDNjZDUxMDc0YmJiMCA9ICQoJzxkaXYgaWQ9Imh0bWxfNDEyYTU5NWJlMGJhNGFmNDlhNDAzY2Q1MTA3NGJiYjAiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjExNzM4LCBMQUJFTDowLCBTRUdNRU5UOldUQyAtIFJlc2lkZW50aWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF82NTFhMzAyMjdhZTk0M2E2YjdiYTljZDBhNjM1ODM2My5zZXRDb250ZW50KGh0bWxfNDEyYTU5NWJlMGJhNGFmNDlhNDAzY2Q1MTA3NGJiYjApOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl8yZmI3NjU0ZGVhY2I0ZjA0YWEyZTkzMmI2OGJlYTZiYi5iaW5kUG9wdXAocG9wdXBfNjUxYTMwMjI3YWU5NDNhNmI3YmE5Y2QwYTYzNTgzNjMpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfMDA2YTRlMjRlNTFkNDZiMDkyNDMwMjI5ZDhmMGU1ZTkgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4zODE3MzYxLDExNC4xODg2MTM4XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzcyZmVkZmQxZWUyMjRiZDQ5MjUzOTBiOWFiNTJmNzFiID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdvcmFuZ2UnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfMDA2YTRlMjRlNTFkNDZiMDkyNDMwMjI5ZDhmMGU1ZTkuc2V0SWNvbihpY29uXzcyZmVkZmQxZWUyMjRiZDQ5MjUzOTBiOWFiNTJmNzFiKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzFlNjg3NmNlNDA2MDQwZjQ5MWQyZDg5NWQ2OGRiOGY2ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzhkOWQxYTYyMGI5YTQ3ZGY5YWQ3NmYyY2VjOGY3MmI5ID0gJCgnPGRpdiBpZD0iaHRtbF84ZDlkMWE2MjBiOWE0N2RmOWFkNzZmMmNlYzhmNzJiOSIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTE5NzgsIExBQkVMOjQsIFNFR01FTlQ6V1RDIC0gRGVzdGluYXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzFlNjg3NmNlNDA2MDQwZjQ5MWQyZDg5NWQ2OGRiOGY2LnNldENvbnRlbnQoaHRtbF84ZDlkMWE2MjBiOWE0N2RmOWFkNzZmMmNlYzhmNzJiOSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzAwNmE0ZTI0ZTUxZDQ2YjA5MjQzMDIyOWQ4ZjBlNWU5LmJpbmRQb3B1cChwb3B1cF8xZTY4NzZjZTQwNjA0MGY0OTFkMmQ4OTVkNjhkYjhmNik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl85OGMwNGVmNzA0NDg0MTkxYWQ1MTE0ODIxZGExY2QxYiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjQwMDc1MDEsMTEzLjk3NDQ0MzhdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fN2IyODBhOWZjNGFlNDIwMjhkNjM1MDU4NzFhZWRlZTAgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3JlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl85OGMwNGVmNzA0NDg0MTkxYWQ1MTE0ODIxZGExY2QxYi5zZXRJY29uKGljb25fN2IyODBhOWZjNGFlNDIwMjhkNjM1MDU4NzFhZWRlZTApOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNjZjN2RiZmFhMmU1NGI2ZDg1NzkwZGQxNzAwMTUzM2IgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfMDExNDNjNjYzNzQ0NDUwNWIwMWYwZDU2MjYwOTI3MjYgPSAkKCc8ZGl2IGlkPSJodG1sXzAxMTQzYzY2Mzc0NDQ1MDViMDFmMGQ1NjI2MDkyNzI2IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjA1NiwgTEFCRUw6MCwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNjZjN2RiZmFhMmU1NGI2ZDg1NzkwZGQxNzAwMTUzM2Iuc2V0Q29udGVudChodG1sXzAxMTQzYzY2Mzc0NDQ1MDViMDFmMGQ1NjI2MDkyNzI2KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfOThjMDRlZjcwNDQ4NDE5MWFkNTExNDgyMWRhMWNkMWIuYmluZFBvcHVwKHBvcHVwXzY2YzdkYmZhYTJlNTRiNmQ4NTc5MGRkMTcwMDE1MzNiKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2JkODAzZTQyMzQyODRiMGFiZTJlODc2OWUyYmY3OGQxID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzM0NzIyLDExNC4xNDk1NjJdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fZjVmNTg0MjAzMDQzNDM5YWEyZTIwZWNkYjdjZjcyMWIgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3JlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl9iZDgwM2U0MjM0Mjg0YjBhYmUyZTg3NjllMmJmNzhkMS5zZXRJY29uKGljb25fZjVmNTg0MjAzMDQzNDM5YWEyZTIwZWNkYjdjZjcyMWIpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfN2IxYzBmZmU2NjcxNDNkZTk2NWQzOGI1MjdhZDM4M2MgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfZmVmODBlNjIxZjEyNGRlMTgxYmE5OTQxYzNmNTU0YzkgPSAkKCc8ZGl2IGlkPSJodG1sX2ZlZjgwZTYyMWYxMjRkZTE4MWJhOTk0MWMzZjU1NGM5IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjI4NCwgTEFCRUw6MCwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfN2IxYzBmZmU2NjcxNDNkZTk2NWQzOGI1MjdhZDM4M2Muc2V0Q29udGVudChodG1sX2ZlZjgwZTYyMWYxMjRkZTE4MWJhOTk0MWMzZjU1NGM5KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfYmQ4MDNlNDIzNDI4NGIwYWJlMmU4NzY5ZTJiZjc4ZDEuYmluZFBvcHVwKHBvcHVwXzdiMWMwZmZlNjY3MTQzZGU5NjVkMzhiNTI3YWQzODNjKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2MwOTI2ODRkZTZjMDQ1ZWNhZWNmMWUyMjdiNDQzYzg4ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzQxOTA4NSwxMTQuMTkyOTkzMl0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl85YWQ1ODkzZDhjNWU0YmY5YmNhZWY0MjM1ZTU2MGEyNSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnbGlnaHRyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfYzA5MjY4NGRlNmMwNDVlY2FlY2YxZTIyN2I0NDNjODguc2V0SWNvbihpY29uXzlhZDU4OTNkOGM1ZTRiZjliY2FlZjQyMzVlNTYwYTI1KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzEwNmRkMGIzNGRjODQ3OWU5NmU4NmMwNjE3ZWFmNjY4ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzJjM2VjZjJkZjcyYzQwYzNiYjZhY2ZhNTJhZWQzNzFiID0gJCgnPGRpdiBpZD0iaHRtbF8yYzNlY2YyZGY3MmM0MGMzYmI2YWNmYTUyYWVkMzcxYiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTI0NzksIExBQkVMOjYsIFNFR01FTlQ6V1RDIC0gQ29tbWVyY2lhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMTA2ZGQwYjM0ZGM4NDc5ZTk2ZTg2YzA2MTdlYWY2Njguc2V0Q29udGVudChodG1sXzJjM2VjZjJkZjcyYzQwYzNiYjZhY2ZhNTJhZWQzNzFiKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfYzA5MjY4NGRlNmMwNDVlY2FlY2YxZTIyN2I0NDNjODguYmluZFBvcHVwKHBvcHVwXzEwNmRkMGIzNGRjODQ3OWU5NmU4NmMwNjE3ZWFmNjY4KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzI3NzJjZDlmOGM1YzRlZWM5NjQyMDdkNGM5NzNlNGZkID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMjg0Mzc3LDExNC4xNTgwNTZdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fMjA3MjEzNmNiYmQzNDllNzhmODRiOTBjNDY4YmIwYjYgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3JlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl8yNzcyY2Q5ZjhjNWM0ZWVjOTY0MjA3ZDRjOTczZTRmZC5zZXRJY29uKGljb25fMjA3MjEzNmNiYmQzNDllNzhmODRiOTBjNDY4YmIwYjYpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNmI3YmNjZWFmNDk5NGUyODgzYmEzY2FiMWJmOGUwYjYgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNjRlOGZlMmE3NGJhNGFhMzg3ZGEyZDI3MGI2ZmI2NzMgPSAkKCc8ZGl2IGlkPSJodG1sXzY0ZThmZTJhNzRiYTRhYTM4N2RhMmQyNzBiNmZiNjczIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMTk0MywgTEFCRUw6MCwgU0VHTUVOVDpXVEMgLSBDb21tZXJjaWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF82YjdiY2NlYWY0OTk0ZTI4ODNiYTNjYWIxYmY4ZTBiNi5zZXRDb250ZW50KGh0bWxfNjRlOGZlMmE3NGJhNGFhMzg3ZGEyZDI3MGI2ZmI2NzMpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl8yNzcyY2Q5ZjhjNWM0ZWVjOTY0MjA3ZDRjOTczZTRmZC5iaW5kUG9wdXAocG9wdXBfNmI3YmNjZWFmNDk5NGUyODgzYmEzY2FiMWJmOGUwYjYpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfMTFlMjI4MWYwMDQ1NGQzYTkxY2JmMjkwYzE1ZjU0ODYgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4zMTYyNjIsMTE0LjI2NTM0MTVdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fYmVmOTYxYzJmNDFkNGM5Yjg5NDkwZGQwODNhNWJlMTAgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2dyZWVuJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzExZTIyODFmMDA0NTRkM2E5MWNiZjI5MGMxNWY1NDg2LnNldEljb24oaWNvbl9iZWY5NjFjMmY0MWQ0YzliODk0OTBkZDA4M2E1YmUxMCk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9iN2VhNmRiMDBhZWM0MmQxODhiMGE1MTg5MmI1ODA3NiA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9mNmY0ZGQxZWE1M2I0ZThhOGY1MGNiYzBkZTYyZmM3ZiA9ICQoJzxkaXYgaWQ9Imh0bWxfZjZmNGRkMWVhNTNiNGU4YThmNTBjYmMwZGU2MmZjN2YiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMDM4LCBMQUJFTDoyLCBTRUdNRU5UOldUQyAtIFJlc2lkZW50aWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9iN2VhNmRiMDBhZWM0MmQxODhiMGE1MTg5MmI1ODA3Ni5zZXRDb250ZW50KGh0bWxfZjZmNGRkMWVhNTNiNGU4YThmNTBjYmMwZGU2MmZjN2YpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl8xMWUyMjgxZjAwNDU0ZDNhOTFjYmYyOTBjMTVmNTQ4Ni5iaW5kUG9wdXAocG9wdXBfYjdlYTZkYjAwYWVjNDJkMTg4YjBhNTE4OTJiNTgwNzYpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfMDczNWYyNTAyNTA4NDQwZGJiZmM4YzJkZGMzYjk4ZDggPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4yOTAyMDQzLDExNC4xOTQ5NzgzXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2Q5OTVhM2RiNjY0ZDRkYzQ4YzQ5YjU5MTRhZTQxY2ZjID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdncmVlbicsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl8wNzM1ZjI1MDI1MDg0NDBkYmJmYzhjMmRkYzNiOThkOC5zZXRJY29uKGljb25fZDk5NWEzZGI2NjRkNGRjNDhjNDliNTkxNGFlNDFjZmMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNDcwZDUzNWNmNzY1NDk2ZDkxZGExOTg2N2ZmMmNkMjcgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfYTU4NWZjN2YwZjAyNDJmODllZjlhNmMxMjZhOWIwNTkgPSAkKCc8ZGl2IGlkPSJodG1sX2E1ODVmYzdmMGYwMjQyZjg5ZWY5YTZjMTI2YTliMDU5IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjA1MSwgTEFCRUw6MiwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNDcwZDUzNWNmNzY1NDk2ZDkxZGExOTg2N2ZmMmNkMjcuc2V0Q29udGVudChodG1sX2E1ODVmYzdmMGYwMjQyZjg5ZWY5YTZjMTI2YTliMDU5KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfMDczNWYyNTAyNTA4NDQwZGJiZmM4YzJkZGMzYjk4ZDguYmluZFBvcHVwKHBvcHVwXzQ3MGQ1MzVjZjc2NTQ5NmQ5MWRhMTk4NjdmZjJjZDI3KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2NmYmQzMGMyZDE2NzRjODE5ZjVlMWI0MThkMjVlYTM1ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMjk5NTU1LDExNC4xNzcxNDQyXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2VlYjhhNGQyOGE5NzQxOTE5N2ZkZTVjOTg2NDc1ZTdmID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdvcmFuZ2UnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfY2ZiZDMwYzJkMTY3NGM4MTlmNWUxYjQxOGQyNWVhMzUuc2V0SWNvbihpY29uX2VlYjhhNGQyOGE5NzQxOTE5N2ZkZTVjOTg2NDc1ZTdmKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzQ1NmI3OGUyNGYyMjQxMmViZjQ0ODVkMDNkMWUyMmE4ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2MyZWVjMjZkYWRmMTQ0NGE5YTUzMmM5NTc4M2M0NmFkID0gJCgnPGRpdiBpZD0iaHRtbF9jMmVlYzI2ZGFkZjE0NDRhOWE1MzJjOTU3ODNjNDZhZCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIzNzYsIExBQkVMOjQsIFNFR01FTlQ6V1RDIC0gVG91cmlzdHMvQ29udjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNDU2Yjc4ZTI0ZjIyNDEyZWJmNDQ4NWQwM2QxZTIyYTguc2V0Q29udGVudChodG1sX2MyZWVjMjZkYWRmMTQ0NGE5YTUzMmM5NTc4M2M0NmFkKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfY2ZiZDMwYzJkMTY3NGM4MTlmNWUxYjQxOGQyNWVhMzUuYmluZFBvcHVwKHBvcHVwXzQ1NmI3OGUyNGYyMjQxMmViZjQ0ODVkMDNkMWUyMmE4KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2ZhYWRhNTdmY2JiMjQwZGQ4MjY5ZDYxYTg5MjUwM2ZhID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzE3OTEzNywxMTQuMTY4NzQ0Ml0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl8xOTMwMWQ0NWM4ODc0YmU3YjU5YjlmNGYyMTVlY2YyMSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnbGlnaHRyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfZmFhZGE1N2ZjYmIyNDBkZDgyNjlkNjFhODkyNTAzZmEuc2V0SWNvbihpY29uXzE5MzAxZDQ1Yzg4NzRiZTdiNTliOWY0ZjIxNWVjZjIxKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2Y0MmJlMThlY2NmZTQ5MTBiM2ZhN2YxZjVmMzJiZDc4ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzU0YWE1Y2QzYjBmNzQ0NDFiMjZjOTE1NjcwNDI5MWM4ID0gJCgnPGRpdiBpZD0iaHRtbF81NGFhNWNkM2IwZjc0NDQxYjI2YzkxNTY3MDQyOTFjOCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTE3NDMsIExBQkVMOjYsIFNFR01FTlQ6V1RDIC0gVG91cmlzdHMvQ29udjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfZjQyYmUxOGVjY2ZlNDkxMGIzZmE3ZjFmNWYzMmJkNzguc2V0Q29udGVudChodG1sXzU0YWE1Y2QzYjBmNzQ0NDFiMjZjOTE1NjcwNDI5MWM4KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfZmFhZGE1N2ZjYmIyNDBkZDgyNjlkNjFhODkyNTAzZmEuYmluZFBvcHVwKHBvcHVwX2Y0MmJlMThlY2NmZTQ5MTBiM2ZhN2YxZjVmMzJiZDc4KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2JjZTUzOThjOWUzMjRhZWI4MjUwNThiYzM2YmQyNjQ4ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzMuNzgzMDExNCwtMTE4LjI4MTE4MzNdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fZDIxM2Q0ODRhNGFmNGMyZDljMmY4NTQyODBkZTA2ZDcgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ29yYW5nZScsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl9iY2U1Mzk4YzllMzI0YWViODI1MDU4YmMzNmJkMjY0OC5zZXRJY29uKGljb25fZDIxM2Q0ODRhNGFmNGMyZDljMmY4NTQyODBkZTA2ZDcpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfZDBhZWE2ZDFiMjRlNDQ3YTkxNWM3YTAxZWY5YWE0ZWUgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfMDJjZGNhMTFmNmY3NDU1YzhlZmY4OTAzOGY4OTdlOGEgPSAkKCc8ZGl2IGlkPSJodG1sXzAyY2RjYTExZjZmNzQ1NWM4ZWZmODkwMzhmODk3ZThhIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjM3NSwgTEFCRUw6NCwgU0VHTUVOVDpXVEMgLSBUb3VyaXN0cy9Db252PC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9kMGFlYTZkMWIyNGU0NDdhOTE1YzdhMDFlZjlhYTRlZS5zZXRDb250ZW50KGh0bWxfMDJjZGNhMTFmNmY3NDU1YzhlZmY4OTAzOGY4OTdlOGEpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl9iY2U1Mzk4YzllMzI0YWViODI1MDU4YmMzNmJkMjY0OC5iaW5kUG9wdXAocG9wdXBfZDBhZWE2ZDFiMjRlNDQ3YTkxNWM3YTAxZWY5YWE0ZWUpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfMjhjMGU4Nzk4MWM1NGExNzg0ZDljODFjZWUxM2VlNWMgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4yODA0MzQ1LDExNC4xODY2MTIxXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzA3Mjk0YWY0NzQ5ZDQ1NTg5NzkxOWY2OTI3ODkxMGNiID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdvcmFuZ2UnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfMjhjMGU4Nzk4MWM1NGExNzg0ZDljODFjZWUxM2VlNWMuc2V0SWNvbihpY29uXzA3Mjk0YWY0NzQ5ZDQ1NTg5NzkxOWY2OTI3ODkxMGNiKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzU0N2QwOTFmM2NmYzRiOTliYTkyMjY4ZTNhZTg4NWQ0ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2RjNDQxNjFhMGEyODQ1YzI4ZDdiMDMzMTk2ZTNjMjY0ID0gJCgnPGRpdiBpZD0iaHRtbF9kYzQ0MTYxYTBhMjg0NWMyOGQ3YjAzMzE5NmUzYzI2NCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIwMDMsIExBQkVMOjQsIFNFR01FTlQ6V1RDIC0gVG91cmlzdHMvQ29udjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNTQ3ZDA5MWYzY2ZjNGI5OWJhOTIyNjhlM2FlODg1ZDQuc2V0Q29udGVudChodG1sX2RjNDQxNjFhMGEyODQ1YzI4ZDdiMDMzMTk2ZTNjMjY0KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfMjhjMGU4Nzk4MWM1NGExNzg0ZDljODFjZWUxM2VlNWMuYmluZFBvcHVwKHBvcHVwXzU0N2QwOTFmM2NmYzRiOTliYTkyMjY4ZTNhZTg4NWQ0KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzA1YjNlZTg0MmM4ODQ2NzliZDBlMzU2ZjZiY2RlYjMyID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzA0NTQyNywxMTQuMTcxMzYxOF0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9mNjgzZjg1Nzc0NTU0NzU4YTY1OTIyNzBjYzVhNWVjMSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnbGlnaHRyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfMDViM2VlODQyYzg4NDY3OWJkMGUzNTZmNmJjZGViMzIuc2V0SWNvbihpY29uX2Y2ODNmODU3NzQ1NTQ3NThhNjU5MjI3MGNjNWE1ZWMxKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzc4MDk1OTExNmJiMDRhODQ5ZmI2NWUwMWNmZDljYTdlID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzUwYjZiZjM2NzRlMDQ4ZmY5MjdjNGQwNTM0MTFiMGExID0gJCgnPGRpdiBpZD0iaHRtbF81MGI2YmYzNjc0ZTA0OGZmOTI3YzRkMDUzNDExYjBhMSIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTE4NDcsIExBQkVMOjYsIFNFR01FTlQ6V1RDIC0gVG91cmlzdHMvQ29udjwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNzgwOTU5MTE2YmIwNGE4NDlmYjY1ZTAxY2ZkOWNhN2Uuc2V0Q29udGVudChodG1sXzUwYjZiZjM2NzRlMDQ4ZmY5MjdjNGQwNTM0MTFiMGExKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfMDViM2VlODQyYzg4NDY3OWJkMGUzNTZmNmJjZGViMzIuYmluZFBvcHVwKHBvcHVwXzc4MDk1OTExNmJiMDRhODQ5ZmI2NWUwMWNmZDljYTdlKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzVlYjA0NTZmNDQ1NjQ1ZTk4OTFkNjVlYjE4MmNjZGRjID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzI1MzQ3LDExNC4xNjc4NTUyXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2I3NWI2MDZiNmY1YTQ5Yzg4NTE3Y2MxODE4ODFlZDc4ID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdsaWdodHJlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl81ZWIwNDU2ZjQ0NTY0NWU5ODkxZDY1ZWIxODJjY2RkYy5zZXRJY29uKGljb25fYjc1YjYwNmI2ZjVhNDljODg1MTdjYzE4MTg4MWVkNzgpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNzE5NWFjNzUwYzRiNDE3ZDg1YzBiNzE1YTJhYTYxMDggPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfMjVhNWVkNTZmYmYwNDUyNzllMDI5MWQ3OWNhNDg5MTUgPSAkKCc8ZGl2IGlkPSJodG1sXzI1YTVlZDU2ZmJmMDQ1Mjc5ZTAyOTFkNzljYTQ4OTE1IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjI3OCwgTEFCRUw6NiwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNzE5NWFjNzUwYzRiNDE3ZDg1YzBiNzE1YTJhYTYxMDguc2V0Q29udGVudChodG1sXzI1YTVlZDU2ZmJmMDQ1Mjc5ZTAyOTFkNzljYTQ4OTE1KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfNWViMDQ1NmY0NDU2NDVlOTg5MWQ2NWViMTgyY2NkZGMuYmluZFBvcHVwKHBvcHVwXzcxOTVhYzc1MGM0YjQxN2Q4NWMwYjcxNWEyYWE2MTA4KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzhhMDQ2M2ExOGNmYzQ1YWE4Njg2N2VlYzEzNWUwZWJjID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMjc4MTU1NiwxMTQuMTgyMDNdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fY2EwNTE0ZWI0Y2RiNDU1YmFkZTE4YzRhMzY5MWZkMDEgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ29yYW5nZScsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl84YTA0NjNhMThjZmM0NWFhODY4NjdlZWMxMzVlMGViYy5zZXRJY29uKGljb25fY2EwNTE0ZWI0Y2RiNDU1YmFkZTE4YzRhMzY5MWZkMDEpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfM2FhNjMzMmEyZDU4NDdiNWE0ZDZlOGY5NTk0MTlhYmQgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfOTQ5NTNmZjljZjZlNDIzN2I2YWJiMmI2OTFhNTIwYjEgPSAkKCc8ZGl2IGlkPSJodG1sXzk0OTUzZmY5Y2Y2ZTQyMzdiNmFiYjJiNjkxYTUyMGIxIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMzA4MSwgTEFCRUw6NCwgU0VHTUVOVDpXVEMgLSBCYWJ5PC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8zYWE2MzMyYTJkNTg0N2I1YTRkNmU4Zjk1OTQxOWFiZC5zZXRDb250ZW50KGh0bWxfOTQ5NTNmZjljZjZlNDIzN2I2YWJiMmI2OTFhNTIwYjEpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl84YTA0NjNhMThjZmM0NWFhODY4NjdlZWMxMzVlMGViYy5iaW5kUG9wdXAocG9wdXBfM2FhNjMzMmEyZDU4NDdiNWE0ZDZlOGY5NTk0MTlhYmQpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfYWQ4N2JmMjQ2YzZiNDM0Y2FjM2NmYmUyZmE2MzNiNDUgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4yNzgyMDc5LDExNC4xODIyOTk0XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzE4Y2E1NWExYTZiNjRhNDU5MzhlMjg2ZmYyMWI2NTdkID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdvcmFuZ2UnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfYWQ4N2JmMjQ2YzZiNDM0Y2FjM2NmYmUyZmE2MzNiNDUuc2V0SWNvbihpY29uXzE4Y2E1NWExYTZiNjRhNDU5MzhlMjg2ZmYyMWI2NTdkKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzllNmI2NWY5MmJmODRkMTk5YTVjY2MyOTkwNDhjZDg0ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzYwZDcxMGNiNWI3YjQ3ZmE4ZTRlMTNjZjdjYTFmN2QyID0gJCgnPGRpdiBpZD0iaHRtbF82MGQ3MTBjYjViN2I0N2ZhOGU0ZTEzY2Y3Y2ExZjdkMiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIzOTQsIExBQkVMOjQsIFNFR01FTlQ6V1RDIC0gSGVhbHRoPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF85ZTZiNjVmOTJiZjg0ZDE5OWE1Y2NjMjk5MDQ4Y2Q4NC5zZXRDb250ZW50KGh0bWxfNjBkNzEwY2I1YjdiNDdmYThlNGUxM2NmN2NhMWY3ZDIpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl9hZDg3YmYyNDZjNmI0MzRjYWMzY2ZiZTJmYTYzM2I0NS5iaW5kUG9wdXAocG9wdXBfOWU2YjY1ZjkyYmY4NGQxOTlhNWNjYzI5OTA0OGNkODQpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfN2IzYzY5YmE5ZDFhNDkwN2JjNDVjZDg1YTgxNTIzNTcgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi40MjQxNzUzLDExNC4yMzE3OF0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl8zMzExOThlN2VhMGM0YjRhYjcyZWNmN2NiN2Y4YzA5OCA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAncmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzdiM2M2OWJhOWQxYTQ5MDdiYzQ1Y2Q4NWE4MTUyMzU3LnNldEljb24oaWNvbl8zMzExOThlN2VhMGM0YjRhYjcyZWNmN2NiN2Y4YzA5OCk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF83NTVlMjE4ZGFkOTM0MTRjYWM4NDhmMjU5NjVlM2JmOSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9lZWFlYjRmNjBhMDU0NmIwYmJhNDYyYTFkMGU3Nzg4MyA9ICQoJzxkaXYgaWQ9Imh0bWxfZWVhZWI0ZjYwYTA1NDZiMGJiYTQ2MmExZDBlNzc4ODMiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjExNzAyLCBMQUJFTDowLCBTRUdNRU5UOldUQyAtIFJlc2lkZW50aWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF83NTVlMjE4ZGFkOTM0MTRjYWM4NDhmMjU5NjVlM2JmOS5zZXRDb250ZW50KGh0bWxfZWVhZWI0ZjYwYTA1NDZiMGJiYTQ2MmExZDBlNzc4ODMpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl83YjNjNjliYTlkMWE0OTA3YmM0NWNkODVhODE1MjM1Ny5iaW5kUG9wdXAocG9wdXBfNzU1ZTIxOGRhZDkzNDE0Y2FjODQ4ZjI1OTY1ZTNiZjkpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfOTBiZGRlOGYzOGI5NDFmMDgzNWQ5MWUxOWM5YjgyYmYgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4yNzg1NDYsMTE0LjE2NTIwNzldLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fM2QyMjlmMzFhNmViNDUwNjk1YWJlOTM3MmU3MWU2NzYgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3JlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl85MGJkZGU4ZjM4Yjk0MWYwODM1ZDkxZTE5YzliODJiZi5zZXRJY29uKGljb25fM2QyMjlmMzFhNmViNDUwNjk1YWJlOTM3MmU3MWU2NzYpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNDJiZTAwZGQ5NWVjNDAzOGFhZTgxOGY4YmU1NTk3YTYgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfOTBlMWNjNjMwMzI2NDAyM2E4YjIzMTc1MjJkY2VjYjQgPSAkKCc8ZGl2IGlkPSJodG1sXzkwZTFjYzYzMDMyNjQwMjNhOGIyMzE3NTIyZGNlY2I0IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMTk1MiwgTEFCRUw6MCwgU0VHTUVOVDpXVEMgLSBDb21tZXJjaWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF80MmJlMDBkZDk1ZWM0MDM4YWFlODE4ZjhiZTU1OTdhNi5zZXRDb250ZW50KGh0bWxfOTBlMWNjNjMwMzI2NDAyM2E4YjIzMTc1MjJkY2VjYjQpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl85MGJkZGU4ZjM4Yjk0MWYwODM1ZDkxZTE5YzliODJiZi5iaW5kUG9wdXAocG9wdXBfNDJiZTAwZGQ5NWVjNDAzOGFhZTgxOGY4YmU1NTk3YTYpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfMjQ2OWJmMjg0ODVjNDliNWI2ODk2MzUyMWExYjBiODQgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi40NDQwMDIsMTE0LjE2ODcwOV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl80N2E1OTg1YWNlYTg0MzhjYTNlMzdkNWM0MjI5ZDZhNyA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnYmVpZ2UnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfMjQ2OWJmMjg0ODVjNDliNWI2ODk2MzUyMWExYjBiODQuc2V0SWNvbihpY29uXzQ3YTU5ODVhY2VhODQzOGNhM2UzN2Q1YzQyMjlkNmE3KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzhmZjgzNWQ2ZmE0OTQzZTQ4OTI2ZmYyNzY1MmI4ZDBlID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzNkZmI4Njc4MTU5ZjQwMjM5NjIxOTRlNzEzMWYzYWU3ID0gJCgnPGRpdiBpZD0iaHRtbF8zZGZiODY3ODE1OWY0MDIzOTYyMTk0ZTcxMzFmM2FlNyIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTE5NjEsIExBQkVMOjcsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzhmZjgzNWQ2ZmE0OTQzZTQ4OTI2ZmYyNzY1MmI4ZDBlLnNldENvbnRlbnQoaHRtbF8zZGZiODY3ODE1OWY0MDIzOTYyMTk0ZTcxMzFmM2FlNyk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzI0NjliZjI4NDg1YzQ5YjViNjg5NjM1MjFhMWIwYjg0LmJpbmRQb3B1cChwb3B1cF84ZmY4MzVkNmZhNDk0M2U0ODkyNmZmMjc2NTJiOGQwZSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl82OTVhN2ZjOTE1Nzc0OGEyYjk4NDQ0ZWUxNTg3ODFhYiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjI5MDkxNzUsMTE0LjE5ODEwNjJdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fY2NkYjRiOGM0N2U0NDIyNjgxYjQ1ZGY0NjYzMjI4ZDkgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2dyZWVuJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzY5NWE3ZmM5MTU3NzQ4YTJiOTg0NDRlZTE1ODc4MWFiLnNldEljb24oaWNvbl9jY2RiNGI4YzQ3ZTQ0MjI2ODFiNDVkZjQ2NjMyMjhkOSk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9mZTU4OTZmZTc1ZGM0ZTUwOGQ4YmM1NWFlOGI5YmJmOSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF8yMDU2ZWRmNTI0Yzk0OTI1ODY1MWVlNzUwMTY3NGIzZCA9ICQoJzxkaXYgaWQ9Imh0bWxfMjA1NmVkZjUyNGM5NDkyNTg2NTFlZTc1MDE2NzRiM2QiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjExOTY5LCBMQUJFTDoyLCBTRUdNRU5UOldUQyAtIFNhdmVyczwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfZmU1ODk2ZmU3NWRjNGU1MDhkOGJjNTVhZThiOWJiZjkuc2V0Q29udGVudChodG1sXzIwNTZlZGY1MjRjOTQ5MjU4NjUxZWU3NTAxNjc0YjNkKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfNjk1YTdmYzkxNTc3NDhhMmI5ODQ0NGVlMTU4NzgxYWIuYmluZFBvcHVwKHBvcHVwX2ZlNTg5NmZlNzVkYzRlNTA4ZDhiYzU1YWU4YjliYmY5KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2ZmZmIyZmI2ZmI0NzQxYzZiNTE0NDEwMzA5MmVjMzVjID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMjg2NTMxLDExNC4xNDIyODJdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fNjllN2YxYWExZWE0NGU3MWIwNTk2N2RkYzM3ODFhOTMgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3JlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl9mZmZiMmZiNmZiNDc0MWM2YjUxNDQxMDMwOTJlYzM1Yy5zZXRJY29uKGljb25fNjllN2YxYWExZWE0NGU3MWIwNTk2N2RkYzM3ODFhOTMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfOTg1M2U0MzUxNDE4NDc0OWIxODczMmYxNTYxNTQyOWIgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfMjc1ZGU3MWY2ZDBjNDI4ZDkzNzhlNzYwZDVlMWE5YWEgPSAkKCc8ZGl2IGlkPSJodG1sXzI3NWRlNzFmNmQwYzQyOGQ5Mzc4ZTc2MGQ1ZTFhOWFhIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjA3OSwgTEFCRUw6MCwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfOTg1M2U0MzUxNDE4NDc0OWIxODczMmYxNTYxNTQyOWIuc2V0Q29udGVudChodG1sXzI3NWRlNzFmNmQwYzQyOGQ5Mzc4ZTc2MGQ1ZTFhOWFhKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfZmZmYjJmYjZmYjQ3NDFjNmI1MTQ0MTAzMDkyZWMzNWMuYmluZFBvcHVwKHBvcHVwXzk4NTNlNDM1MTQxODQ3NDliMTg3MzJmMTU2MTU0MjliKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzk2ZjcwNzEyZWRhNzQxYWU5N2RjNmY5NjViYzFkMTk3ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMjkyMTQwNiwxMTQuMDEwMTQ2N10sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9iOWQ5ZmZlMmNkMTE0MWRmYTI3ODE1ODA0ZWFiYTdlZCA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnYmVpZ2UnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfOTZmNzA3MTJlZGE3NDFhZTk3ZGM2Zjk2NWJjMWQxOTcuc2V0SWNvbihpY29uX2I5ZDlmZmUyY2QxMTQxZGZhMjc4MTU4MDRlYWJhN2VkKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzJjNWQzZjdkNWNjZDQyYzBiOWI3MGNiYjY4ZjNlOWMyID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzlmMmJlNmZlODE0ODRhZTBiZjdmM2VjMGFiZjhjMGIzID0gJCgnPGRpdiBpZD0iaHRtbF85ZjJiZTZmZTgxNDg0YWUwYmY3ZjNlYzBhYmY4YzBiMyIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIwMTUsIExBQkVMOjcsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzJjNWQzZjdkNWNjZDQyYzBiOWI3MGNiYjY4ZjNlOWMyLnNldENvbnRlbnQoaHRtbF85ZjJiZTZmZTgxNDg0YWUwYmY3ZjNlYzBhYmY4YzBiMyk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzk2ZjcwNzEyZWRhNzQxYWU5N2RjNmY5NjViYzFkMTk3LmJpbmRQb3B1cChwb3B1cF8yYzVkM2Y3ZDVjY2Q0MmMwYjliNzBjYmI2OGYzZTljMik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9iMTA0MmYxNzMwYjU0Y2MyOGFlNmQ1N2FjNmU4M2E5NyA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjM3NTI0ODQsMTEzLjk2MzA2MzJdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fZjM0NjIwNDYzYzIwNDkwNGE4MjFmNzk3ODdkNTY0NDQgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3JlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl9iMTA0MmYxNzMwYjU0Y2MyOGFlNmQ1N2FjNmU4M2E5Ny5zZXRJY29uKGljb25fZjM0NjIwNDYzYzIwNDkwNGE4MjFmNzk3ODdkNTY0NDQpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfOTUwMjBiMDQwMzg5NDY2NmJiNmY0NmE1YTQ2MTFkMmEgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfZTliZGQwMTc0ZTVjNDllYzhmN2E1MzNjMzk0MGVlYWIgPSAkKCc8ZGl2IGlkPSJodG1sX2U5YmRkMDE3NGU1YzQ5ZWM4ZjdhNTMzYzM5NDBlZWFiIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjAxNiwgTEFCRUw6MCwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfOTUwMjBiMDQwMzg5NDY2NmJiNmY0NmE1YTQ2MTFkMmEuc2V0Q29udGVudChodG1sX2U5YmRkMDE3NGU1YzQ5ZWM4ZjdhNTMzYzM5NDBlZWFiKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfYjEwNDJmMTczMGI1NGNjMjhhZTZkNTdhYzZlODNhOTcuYmluZFBvcHVwKHBvcHVwXzk1MDIwYjA0MDM4OTQ2NjZiYjZmNDZhNWE0NjExZDJhKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzgxYTBhNTkzYzVkYjQxNTJiZTYyYjllMDVhOWQ1YmZkID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuNDU0OTEzOCwxMTQuMDAzOTc2XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzcyNzU2NjA3NTgxMTQxNzc5NDJmY2NkZjVmYmU3ZWQwID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdwdXJwbGUnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfODFhMGE1OTNjNWRiNDE1MmJlNjJiOWUwNWE5ZDViZmQuc2V0SWNvbihpY29uXzcyNzU2NjA3NTgxMTQxNzc5NDJmY2NkZjVmYmU3ZWQwKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzFkNjVlODM3ZDJlMjQ0NGQ4YTdiNjgzODJkZjA4NjVjID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2U0YTA0MjE4ZmQ4ZTRlZTk5YThiYmU1NjFjYmEzMWQ4ID0gJCgnPGRpdiBpZD0iaHRtbF9lNGEwNDIxOGZkOGU0ZWU5OWE4YmJlNTYxY2JhMzFkOCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTQwODEsIExBQkVMOjMsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzFkNjVlODM3ZDJlMjQ0NGQ4YTdiNjgzODJkZjA4NjVjLnNldENvbnRlbnQoaHRtbF9lNGEwNDIxOGZkOGU0ZWU5OWE4YmJlNTYxY2JhMzFkOCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzgxYTBhNTkzYzVkYjQxNTJiZTYyYjllMDVhOWQ1YmZkLmJpbmRQb3B1cChwb3B1cF8xZDY1ZTgzN2QyZTI0NDRkOGE3YjY4MzgyZGYwODY1Yyk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9iYzliZDdkYmM1ZTc0OTEzOGNkNDFiZjI1OWZjY2NkMSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjM2ODU0LDExNC4xMjE0ODI0XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2UwYWFmMDEwM2ViYTQzZmM5ZDRkNjMwMzlmZjU2Y2MwID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfYmM5YmQ3ZGJjNWU3NDkxMzhjZDQxYmYyNTlmY2NjZDEuc2V0SWNvbihpY29uX2UwYWFmMDEwM2ViYTQzZmM5ZDRkNjMwMzlmZjU2Y2MwKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzk3YzViM2MxYzRkODRmM2RiZjVmZjkwY2U0NDkyZGRhID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2E3ODBlYTVkYzQzMzRkOTc5MTJmNjMwNzQ1N2RkNGZiID0gJCgnPGRpdiBpZD0iaHRtbF9hNzgwZWE1ZGM0MzM0ZDk3OTEyZjYzMDc0NTdkZDRmYiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIxOTAsIExBQkVMOjAsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzk3YzViM2MxYzRkODRmM2RiZjVmZjkwY2U0NDkyZGRhLnNldENvbnRlbnQoaHRtbF9hNzgwZWE1ZGM0MzM0ZDk3OTEyZjYzMDc0NTdkZDRmYik7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyX2JjOWJkN2RiYzVlNzQ5MTM4Y2Q0MWJmMjU5ZmNjY2QxLmJpbmRQb3B1cChwb3B1cF85N2M1YjNjMWM0ZDg0ZjNkYmY1ZmY5MGNlNDQ5MmRkYSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl81ZjM1N2NkNzQwMDI0MWI0YjJhNzQzYmZiMWZhYzEzNSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjMyMDUzMTQsMTE0LjIyMjY1N10sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9mMTIwMmQ0MDZkYmM0MTExYTVlODRmOTg5YTRjYjBjZSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAncmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzVmMzU3Y2Q3NDAwMjQxYjRiMmE3NDNiZmIxZmFjMTM1LnNldEljb24oaWNvbl9mMTIwMmQ0MDZkYmM0MTExYTVlODRmOTg5YTRjYjBjZSk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8zZDNhODM4MWIwOTg0YmViOTE1ZjBmOGNiNmIxYjI1YiA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF84ODUxMWQ5ZDQ2NmY0OTdjYmIzMjg1ZDk1Y2ZhMmFhNSA9ICQoJzxkaXYgaWQ9Imh0bWxfODg1MTFkOWQ0NjZmNDk3Y2JiMzI4NWQ5NWNmYTJhYTUiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMzMxLCBMQUJFTDowLCBTRUdNRU5UOldUQyAtIFNhdmVyczwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfM2QzYTgzODFiMDk4NGJlYjkxNWYwZjhjYjZiMWIyNWIuc2V0Q29udGVudChodG1sXzg4NTExZDlkNDY2ZjQ5N2NiYjMyODVkOTVjZmEyYWE1KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfNWYzNTdjZDc0MDAyNDFiNGIyYTc0M2JmYjFmYWMxMzUuYmluZFBvcHVwKHBvcHVwXzNkM2E4MzgxYjA5ODRiZWI5MTVmMGY4Y2I2YjFiMjViKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzI2NWVhYzRjOWViODRlODY4NmVhMGEyN2UyYTgwNTZhID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMjQ4OTQ2OSwxMTQuMTUzODc5MV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl83ZGQzYzM3YTgyN2I0NDQ3OWRiOWQzZGE1YzM2MjgyMiA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnYmVpZ2UnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfMjY1ZWFjNGM5ZWI4NGU4Njg2ZWEwYTI3ZTJhODA1NmEuc2V0SWNvbihpY29uXzdkZDNjMzdhODI3YjQ0NDc5ZGI5ZDNkYTVjMzYyODIyKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzExMWRlYjNkMDVkYzRiZTc5NGJhOGJmOTU1MTZmYjJjID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2U1ODY1ZDg4OTFmNzQ2NDBhYjZmZWE3MDRlMTI3MTY3ID0gJCgnPGRpdiBpZD0iaHRtbF9lNTg2NWQ4ODkxZjc0NjQwYWI2ZmVhNzA0ZTEyNzE2NyIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIxODksIExBQkVMOjcsIFNFR01FTlQ6V1RDIC0gRGVzdGluYXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzExMWRlYjNkMDVkYzRiZTc5NGJhOGJmOTU1MTZmYjJjLnNldENvbnRlbnQoaHRtbF9lNTg2NWQ4ODkxZjc0NjQwYWI2ZmVhNzA0ZTEyNzE2Nyk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzI2NWVhYzRjOWViODRlODY4NmVhMGEyN2UyYTgwNTZhLmJpbmRQb3B1cChwb3B1cF8xMTFkZWIzZDA1ZGM0YmU3OTRiYThiZjk1NTE2ZmIyYyk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9lMzk2YmE2OTdmZDk0NWUyOGYxY2U3Nzg3MDljMmY0YiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjMwNDIxNiwxMTQuMjUxOTc0XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzRkNGExZDQ1NTQzYzQ5YWY5ZGI5YWJlYzM2MGJhNWM2ID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdiZWlnZScsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl9lMzk2YmE2OTdmZDk0NWUyOGYxY2U3Nzg3MDljMmY0Yi5zZXRJY29uKGljb25fNGQ0YTFkNDU1NDNjNDlhZjlkYjlhYmVjMzYwYmE1YzYpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfZWFlMTA5NTkxMmE3NGYzMmFmNzg4ZTg1YmRkM2FkMDggPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfYmRmMWIxODMxMmMyNDg2MWI5ZjJhMWFhMjBmZTNiNGYgPSAkKCc8ZGl2IGlkPSJodG1sX2JkZjFiMTgzMTJjMjQ4NjFiOWYyYTFhYTIwZmUzYjRmIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjM4NCwgTEFCRUw6NywgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfZWFlMTA5NTkxMmE3NGYzMmFmNzg4ZTg1YmRkM2FkMDguc2V0Q29udGVudChodG1sX2JkZjFiMTgzMTJjMjQ4NjFiOWYyYTFhYTIwZmUzYjRmKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfZTM5NmJhNjk3ZmQ5NDVlMjhmMWNlNzc4NzA5YzJmNGIuYmluZFBvcHVwKHBvcHVwX2VhZTEwOTU5MTJhNzRmMzJhZjc4OGU4NWJkZDNhZDA4KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzAxNTNkODg4NWUzNjQ3MGRhYTc4MDY4ZDU1MzM5YmIwID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzkzOTEzLDExMy45NzU0ODhdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fMWIzODcwZGUzNWVkNDQwMGJlNTM2M2M3NWEzYTMxYTUgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ29yYW5nZScsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl8wMTUzZDg4ODVlMzY0NzBkYWE3ODA2OGQ1NTMzOWJiMC5zZXRJY29uKGljb25fMWIzODcwZGUzNWVkNDQwMGJlNTM2M2M3NWEzYTMxYTUpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfMmFmMWRjMzJmMGMxNDY4YWJhYjMzZjZjYzM3MTM5NGUgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfMTk0NjIyOGRiNDJhNDJhY2JjNDkzZmM4YTdkMDk2MWUgPSAkKCc8ZGl2IGlkPSJodG1sXzE5NDYyMjhkYjQyYTQyYWNiYzQ5M2ZjOGE3ZDA5NjFlIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMzQ0MSwgTEFCRUw6NCwgU0VHTUVOVDpXVEMgLSBIZWFsdGg8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzJhZjFkYzMyZjBjMTQ2OGFiYWIzM2Y2Y2MzNzEzOTRlLnNldENvbnRlbnQoaHRtbF8xOTQ2MjI4ZGI0MmE0MmFjYmM0OTNmYzhhN2QwOTYxZSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzAxNTNkODg4NWUzNjQ3MGRhYTc4MDY4ZDU1MzM5YmIwLmJpbmRQb3B1cChwb3B1cF8yYWYxZGMzMmYwYzE0NjhhYmFiMzNmNmNjMzcxMzk0ZSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl8xMTBhMjRmOTBlY2M0NDUxYWI1ZTAzZGJmY2M3OTQ5YiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjM0MTM3MDEsMTE0LjE5NDMwMzRdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fYjM0NWUwY2UyM2I5NDRlYWJhNjUwNjg0MGNjOWIwMDkgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3JlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl8xMTBhMjRmOTBlY2M0NDUxYWI1ZTAzZGJmY2M3OTQ5Yi5zZXRJY29uKGljb25fYjM0NWUwY2UyM2I5NDRlYWJhNjUwNjg0MGNjOWIwMDkpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfYjM1YmQyZGI2ZjY3NGUzMmEwYWI3Yjg3YzljMDNmZmMgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNTEyYTE3MWM0ODg2NDRlZmI1MDA5ZGI3OWQzMTdkOWEgPSAkKCc8ZGl2IGlkPSJodG1sXzUxMmExNzFjNDg4NjQ0ZWZiNTAwOWRiNzlkMzE3ZDlhIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMzkyMSwgTEFCRUw6MCwgU0VHTUVOVDpXVEMgLSBTYXZlcnM8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2IzNWJkMmRiNmY2NzRlMzJhMGFiN2I4N2M5YzAzZmZjLnNldENvbnRlbnQoaHRtbF81MTJhMTcxYzQ4ODY0NGVmYjUwMDlkYjc5ZDMxN2Q5YSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzExMGEyNGY5MGVjYzQ0NTFhYjVlMDNkYmZjYzc5NDliLmJpbmRQb3B1cChwb3B1cF9iMzViZDJkYjZmNjc0ZTMyYTBhYjdiODdjOWMwM2ZmYyk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9jYzU3ZTQ3YmY4OGU0YTVlOWNhYTFhMWIwNjk1ZTk3YiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjM2NjIyNywxMTQuMDU4OTExXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2E4Yjc3M2I2ZjMwODQ0ZjlhNDE5NTc2YjEzOWQ3OTJiID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdsaWdodHJlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl9jYzU3ZTQ3YmY4OGU0YTVlOWNhYTFhMWIwNjk1ZTk3Yi5zZXRJY29uKGljb25fYThiNzczYjZmMzA4NDRmOWE0MTk1NzZiMTM5ZDc5MmIpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfMDhhMjM5YTVlMjI0NDZjMTgwZWU5MGQ2NzIyMGNhZTMgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfMjg2ODQ3OGQzNGJjNGZjM2JkYjRmNWNiMzQ4YmMwZDAgPSAkKCc8ZGl2IGlkPSJodG1sXzI4Njg0NzhkMzRiYzRmYzNiZGI0ZjVjYjM0OGJjMGQwIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMTc5NiwgTEFCRUw6NiwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMDhhMjM5YTVlMjI0NDZjMTgwZWU5MGQ2NzIyMGNhZTMuc2V0Q29udGVudChodG1sXzI4Njg0NzhkMzRiYzRmYzNiZGI0ZjVjYjM0OGJjMGQwKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfY2M1N2U0N2JmODhlNGE1ZTljYWExYTFiMDY5NWU5N2IuYmluZFBvcHVwKHBvcHVwXzA4YTIzOWE1ZTIyNDQ2YzE4MGVlOTBkNjcyMjBjYWUzKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzdiMDU1NGIxYjg4MTQzMDQ5OTdkYmU2M2I4MTVhZmJiID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMjg2NjY2MywxMTQuMjE3NDE2OF0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9hZjQzNmJkNzQzZjk0NjcxOWFhNDlkNWVjNDRhOWJhNSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnZ3JlZW4nLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfN2IwNTU0YjFiODgxNDMwNDk5N2RiZTYzYjgxNWFmYmIuc2V0SWNvbihpY29uX2FmNDM2YmQ3NDNmOTQ2NzE5YWE0OWQ1ZWM0NGE5YmE1KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2E2Yjk4ODA5ZTRlODRhOGQ4ZGVlZjNhYzBkMjJlM2U0ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2NkMjAzYjkzY2VlYTQ3NWM5MjMwMzY3MTEzZTc3YTJkID0gJCgnPGRpdiBpZD0iaHRtbF9jZDIwM2I5M2NlZWE0NzVjOTIzMDM2NzExM2U3N2EyZCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTE5MjgsIExBQkVMOjIsIFNFR01FTlQ6V1RDIC0gRGVzdGluYXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2E2Yjk4ODA5ZTRlODRhOGQ4ZGVlZjNhYzBkMjJlM2U0LnNldENvbnRlbnQoaHRtbF9jZDIwM2I5M2NlZWE0NzVjOTIzMDM2NzExM2U3N2EyZCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzdiMDU1NGIxYjg4MTQzMDQ5OTdkYmU2M2I4MTVhZmJiLmJpbmRQb3B1cChwb3B1cF9hNmI5ODgwOWU0ZTg0YThkOGRlZWYzYWMwZDIyZTNlNCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl81MjdkMWYwZmFkYTM0NmNjYTI4ODk4YjQ3MTRhNmEwNyA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjMwNDIyMTUsMTE0LjE3MTU2MTRdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fZWNlODE1ZjdkZDkxNDBkNjkwNGQzNjdlNGFmNDFiYjggPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ29yYW5nZScsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl81MjdkMWYwZmFkYTM0NmNjYTI4ODk4YjQ3MTRhNmEwNy5zZXRJY29uKGljb25fZWNlODE1ZjdkZDkxNDBkNjkwNGQzNjdlNGFmNDFiYjgpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfMGI2MjI5NmIwOTA5NGQyOWJkYjMzNDQxYWE0YzQ1MTYgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfM2Y1NTQ4MTJmNjcyNDFiYzlmODk3ZDZkYjVhZDlkYWEgPSAkKCc8ZGl2IGlkPSJodG1sXzNmNTU0ODEyZjY3MjQxYmM5Zjg5N2Q2ZGI1YWQ5ZGFhIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjI5OCwgTEFCRUw6NCwgU0VHTUVOVDpXVEMgLSBUb3VyaXN0cy9Db252PC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8wYjYyMjk2YjA5MDk0ZDI5YmRiMzM0NDFhYTRjNDUxNi5zZXRDb250ZW50KGh0bWxfM2Y1NTQ4MTJmNjcyNDFiYzlmODk3ZDZkYjVhZDlkYWEpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl81MjdkMWYwZmFkYTM0NmNjYTI4ODk4YjQ3MTRhNmEwNy5iaW5kUG9wdXAocG9wdXBfMGI2MjI5NmIwOTA5NGQyOWJkYjMzNDQxYWE0YzQ1MTYpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfZTIyOTZiNjA2NGM3NGRjNzlmMmUxMGNmMTU1YjczNzIgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi41MDI0MzQsMTE0LjEyOTA0MV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl80MzFmZTM4NWY5YmM0NTIwYTA5ZTM3N2YxNDA4OTI0OSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnb3JhbmdlJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyX2UyMjk2YjYwNjRjNzRkYzc5ZjJlMTBjZjE1NWI3MzcyLnNldEljb24oaWNvbl80MzFmZTM4NWY5YmM0NTIwYTA5ZTM3N2YxNDA4OTI0OSk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9lYzFlZDQyYWI3MjM0ZmM4YTQzY2NiMmZmYWI3NDk5YyA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF80YmI3YzY1MmMxYjM0NGQxOWM5MzFmYzIxMzYxM2Q5OCA9ICQoJzxkaXYgaWQ9Imh0bWxfNGJiN2M2NTJjMWIzNDRkMTljOTMxZmMyMTM2MTNkOTgiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyNDQwLCBMQUJFTDo0LCBTRUdNRU5UOldUQyAtIFRvdXJpc3RzL0NvbnY8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2VjMWVkNDJhYjcyMzRmYzhhNDNjY2IyZmZhYjc0OTljLnNldENvbnRlbnQoaHRtbF80YmI3YzY1MmMxYjM0NGQxOWM5MzFmYzIxMzYxM2Q5OCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyX2UyMjk2YjYwNjRjNzRkYzc5ZjJlMTBjZjE1NWI3MzcyLmJpbmRQb3B1cChwb3B1cF9lYzFlZDQyYWI3MjM0ZmM4YTQzY2NiMmZmYWI3NDk5Yyk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl84ZDgxNmExYjgzN2M0N2QxYTRkYjVlYTM4NGE0ODFjMyA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjQ4NTg4MDEsMTE0LjE0MTI5MV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl82ZjRjMjc3MDg1ODg0MGRlOTllZDkxZTZjNmE1NGZhYSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnZ3JlZW4nLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfOGQ4MTZhMWI4MzdjNDdkMWE0ZGI1ZWEzODRhNDgxYzMuc2V0SWNvbihpY29uXzZmNGMyNzcwODU4ODQwZGU5OWVkOTFlNmM2YTU0ZmFhKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzliNjE4MDVhOGE0MzRhZWFiNTUyODMzZmVmNTNjYjBkID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzEyNmRhNzRjZTZiNDQ5ZDk4M2U1NmQyNDA4NmVjZTZiID0gJCgnPGRpdiBpZD0iaHRtbF8xMjZkYTc0Y2U2YjQ0OWQ5ODNlNTZkMjQwODZlY2U2YiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTE3MjAsIExBQkVMOjIsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzliNjE4MDVhOGE0MzRhZWFiNTUyODMzZmVmNTNjYjBkLnNldENvbnRlbnQoaHRtbF8xMjZkYTc0Y2U2YjQ0OWQ5ODNlNTZkMjQwODZlY2U2Yik7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzhkODE2YTFiODM3YzQ3ZDFhNGRiNWVhMzg0YTQ4MWMzLmJpbmRQb3B1cChwb3B1cF85YjYxODA1YThhNDM0YWVhYjU1MjgzM2ZlZjUzY2IwZCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9lOWZmMWU4MDlkYWE0YjZmYjc0NDkwZmJhZjNhZGMyMCA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzM0LjA1ODgzMzgsLTExOC4yMzc2MTU2XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2I1M2NmNjUzNDJmYTQ4M2FhODY4MDE4YTliYzAyMDVkID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfZTlmZjFlODA5ZGFhNGI2ZmI3NDQ5MGZiYWYzYWRjMjAuc2V0SWNvbihpY29uX2I1M2NmNjUzNDJmYTQ4M2FhODY4MDE4YTliYzAyMDVkKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzk4YTc1ZDkwNDJhOTRkODhhYWNiNmI1MWNmNDMyYzYzID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzJmZjQ5ZjgwMWY1YjRjOWQ5M2Q3MTQ5ZDg0NTFjZTJkID0gJCgnPGRpdiBpZD0iaHRtbF8yZmY0OWY4MDFmNWI0YzlkOTNkNzE0OWQ4NDUxY2UyZCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIwNjgsIExBQkVMOjAsIFNFR01FTlQ6V1RDIC0gRGVzdGluYXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzk4YTc1ZDkwNDJhOTRkODhhYWNiNmI1MWNmNDMyYzYzLnNldENvbnRlbnQoaHRtbF8yZmY0OWY4MDFmNWI0YzlkOTNkNzE0OWQ4NDUxY2UyZCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyX2U5ZmYxZTgwOWRhYTRiNmZiNzQ0OTBmYmFmM2FkYzIwLmJpbmRQb3B1cChwb3B1cF85OGE3NWQ5MDQyYTk0ZDg4YWFjYjZiNTFjZjQzMmM2Myk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl82ZGZkNmVhOTc2MGY0N2RiOWJkZDE5NDc1YTQ3YTk5MCA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjMyMjgzMjIsMTE0LjI1ODQwOV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl84Yjg2OGViNWJhNmM0NDA1YmE4NTcwMzBmNDVkZTU3NyA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnZ3JlZW4nLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfNmRmZDZlYTk3NjBmNDdkYjliZGQxOTQ3NWE0N2E5OTAuc2V0SWNvbihpY29uXzhiODY4ZWI1YmE2YzQ0MDViYTg1NzAzMGY0NWRlNTc3KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzYwYTZjZDg1YTNiYjQ3MWY4NmNjOTA5MTNmMjIxMGQ2ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzA5MTA2ZGZmOWZjZjRhYmM4OWQxMWUzNzMyNWMwY2NkID0gJCgnPGRpdiBpZD0iaHRtbF8wOTEwNmRmZjlmY2Y0YWJjODlkMTFlMzczMjVjMGNjZCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIyNDAsIExBQkVMOjIsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzYwYTZjZDg1YTNiYjQ3MWY4NmNjOTA5MTNmMjIxMGQ2LnNldENvbnRlbnQoaHRtbF8wOTEwNmRmZjlmY2Y0YWJjODlkMTFlMzczMjVjMGNjZCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzZkZmQ2ZWE5NzYwZjQ3ZGI5YmRkMTk0NzVhNDdhOTkwLmJpbmRQb3B1cChwb3B1cF82MGE2Y2Q4NWEzYmI0NzFmODZjYzkwOTEzZjIyMTBkNik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9jZTExMzBkMDExZTE0MjMzYjlhZDVkNGY5NzY5MTJhMyA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjI5NjQ1MzEsMTE0LjIzODY5ODJdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fNzJlNjc0MmM2ZTM1NDg5ZThhNTI5Nzc0ZDVlMmZmNjQgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2dyZWVuJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyX2NlMTEzMGQwMTFlMTQyMzNiOWFkNWQ0Zjk3NjkxMmEzLnNldEljb24oaWNvbl83MmU2NzQyYzZlMzU0ODllOGE1Mjk3NzRkNWUyZmY2NCk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8wMDZjOTA4MmNkYTg0OWY0YmI0ODQzOWVhMTVjYjhjNCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF82YmEwNmM5MzNkODY0NDgxOGU3ZjRkOWQ1MTgwMzVjOSA9ICQoJzxkaXYgaWQ9Imh0bWxfNmJhMDZjOTMzZDg2NDQ4MThlN2Y0ZDlkNTE4MDM1YzkiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMjg1LCBMQUJFTDoyLCBTRUdNRU5UOldUQyAtIFNhdmVyczwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMDA2YzkwODJjZGE4NDlmNGJiNDg0MzllYTE1Y2I4YzQuc2V0Q29udGVudChodG1sXzZiYTA2YzkzM2Q4NjQ0ODE4ZTdmNGQ5ZDUxODAzNWM5KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfY2UxMTMwZDAxMWUxNDIzM2I5YWQ1ZDRmOTc2OTEyYTMuYmluZFBvcHVwKHBvcHVwXzAwNmM5MDgyY2RhODQ5ZjRiYjQ4NDM5ZWExNWNiOGM0KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzk2OTZkZmY2ZGRjYTQxZWY5MTMyYTNjZTg4ZGQ2YzNlID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMjk5MTE4MSwxMTQuMTc0MTcxXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzg4OWFhMmMyZjM1ZDQyY2E4MjRlYWVlZTliNTQ0N2QwID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdvcmFuZ2UnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfOTY5NmRmZjZkZGNhNDFlZjkxMzJhM2NlODhkZDZjM2Uuc2V0SWNvbihpY29uXzg4OWFhMmMyZjM1ZDQyY2E4MjRlYWVlZTliNTQ0N2QwKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzE1YmNjMDIwNTAwMTQ0ZGY5YjYzNjY3NTIwMmZlNGY4ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzQ4YTc1Yzg4OGEwYTQxNTA5ZmQ2NzViMWI0NGM0ZTEwID0gJCgnPGRpdiBpZD0iaHRtbF80OGE3NWM4ODhhMGE0MTUwOWZkNjc1YjFiNDRjNGUxMCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIzNzcsIExBQkVMOjQsIFNFR01FTlQ6V1RDIC0gRGVzdGluYXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzE1YmNjMDIwNTAwMTQ0ZGY5YjYzNjY3NTIwMmZlNGY4LnNldENvbnRlbnQoaHRtbF80OGE3NWM4ODhhMGE0MTUwOWZkNjc1YjFiNDRjNGUxMCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzk2OTZkZmY2ZGRjYTQxZWY5MTMyYTNjZTg4ZGQ2YzNlLmJpbmRQb3B1cChwb3B1cF8xNWJjYzAyMDUwMDE0NGRmOWI2MzY2NzUyMDJmZTRmOCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl8zZTk4M2ZjNzExNDE0OTczOWEyM2Q0MDc3NzkyZjdhNCA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjMzMzk5NywxMTQuMjA5ODM5XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzNmOWU5ODE0NjdkMTQyYjI5MDY4YzZmMjZkNGEzMzIyID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfM2U5ODNmYzcxMTQxNDk3MzlhMjNkNDA3Nzc5MmY3YTQuc2V0SWNvbihpY29uXzNmOWU5ODE0NjdkMTQyYjI5MDY4YzZmMjZkNGEzMzIyKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzdkODRhMDg3NjRhMjRmZjE5Y2NkZjljZDI1MTBkMzk4ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzI3MjQ0MmY2ZTU3NDQ0M2ViOTUyODczMmRkNTBjY2ZlID0gJCgnPGRpdiBpZD0iaHRtbF8yNzI0NDJmNmU1NzQ0NDNlYjk1Mjg3MzJkZDUwY2NmZSIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTIyNzYsIExBQkVMOjAsIFNFR01FTlQ6V1RDIC0gUmVzaWRlbnRpYWw8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzdkODRhMDg3NjRhMjRmZjE5Y2NkZjljZDI1MTBkMzk4LnNldENvbnRlbnQoaHRtbF8yNzI0NDJmNmU1NzQ0NDNlYjk1Mjg3MzJkZDUwY2NmZSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzNlOTgzZmM3MTE0MTQ5NzM5YTIzZDQwNzc3OTJmN2E0LmJpbmRQb3B1cChwb3B1cF83ZDg0YTA4NzY0YTI0ZmYxOWNjZGY5Y2QyNTEwZDM5OCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9kYjgxMmFlM2Y2NzA0MWUxOWJlNzNjNjAxYzQ1YTQ2MyA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzIyLjI5MDk3MiwxMTQuMjAyNjQzXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2Y1MmRiOTExMmM4ZDRkZjA4ODJiY2E4N2MyZmRiZGRhID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdiZWlnZScsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl9kYjgxMmFlM2Y2NzA0MWUxOWJlNzNjNjAxYzQ1YTQ2My5zZXRJY29uKGljb25fZjUyZGI5MTEyYzhkNGRmMDg4MmJjYTg3YzJmZGJkZGEpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfMzI1NjVkYzdmMjVmNDMzMzk2YTgxMzBiMDZkMzNjZDEgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfMzYzZDVjOGZhYWUzNGFmOThlN2Q3MTM2ZGI0NGEyZmUgPSAkKCc8ZGl2IGlkPSJodG1sXzM2M2Q1YzhmYWFlMzRhZjk4ZTdkNzEzNmRiNDRhMmZlIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjM5MSwgTEFCRUw6NywgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMzI1NjVkYzdmMjVmNDMzMzk2YTgxMzBiMDZkMzNjZDEuc2V0Q29udGVudChodG1sXzM2M2Q1YzhmYWFlMzRhZjk4ZTdkNzEzNmRiNDRhMmZlKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfZGI4MTJhZTNmNjcwNDFlMTliZTczYzYwMWM0NWE0NjMuYmluZFBvcHVwKHBvcHVwXzMyNTY1ZGM3ZjI1ZjQzMzM5NmE4MTMwYjA2ZDMzY2QxKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2Q0OTYwZmVmOTQzYTQ5MjVhZWM4NWI1ZWIxYjQwNmE4ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuNTAyNzA3NiwxMTQuMTI3ODM4NF0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl9hM2NkM2QxODJlMGQ0YzZmOGJmMGQwYTkyNGIyYjhmYSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnb3JhbmdlJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyX2Q0OTYwZmVmOTQzYTQ5MjVhZWM4NWI1ZWIxYjQwNmE4LnNldEljb24oaWNvbl9hM2NkM2QxODJlMGQ0YzZmOGJmMGQwYTkyNGIyYjhmYSk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF85ZGZmMzU3MjI2Yzk0N2ZhODJkODlkNmZiYWZhMWQ1YyA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9iMDYzYTQyNThhZWQ0MDUwYjkwNTMwN2NiMjY2NjU3YyA9ICQoJzxkaXYgaWQ9Imh0bWxfYjA2M2E0MjU4YWVkNDA1MGI5MDUzMDdjYjI2NjY1N2MiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyOTgyLCBMQUJFTDo0LCBTRUdNRU5UOldUQyAtIEhlYWx0aDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfOWRmZjM1NzIyNmM5NDdmYTgyZDg5ZDZmYmFmYTFkNWMuc2V0Q29udGVudChodG1sX2IwNjNhNDI1OGFlZDQwNTBiOTA1MzA3Y2IyNjY2NTdjKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfZDQ5NjBmZWY5NDNhNDkyNWFlYzg1YjVlYjFiNDA2YTguYmluZFBvcHVwKHBvcHVwXzlkZmYzNTcyMjZjOTQ3ZmE4MmQ4OWQ2ZmJhZmExZDVjKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzM5MDY5NTMzYzM0NTQzNTc5MTU3MTUwMTM1NTM4ZTRhID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMzgyNjg4LDExNC4xOTEyMzldLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fOWY0YjIzNzhiMDgyNDc5N2I3YTA5MDliZGEyOTQxZjYgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2xpZ2h0cmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzM5MDY5NTMzYzM0NTQzNTc5MTU3MTUwMTM1NTM4ZTRhLnNldEljb24oaWNvbl85ZjRiMjM3OGIwODI0Nzk3YjdhMDkwOWJkYTI5NDFmNik7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF83MmY5MzdkNGMzODg0ODM3ODcyMTNhZDdhZjFiOTllZiA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF8yZWNmZjJmZDRmMjI0NmE0OGRmMDI0MzM1NTExNGI4OSA9ICQoJzxkaXYgaWQ9Imh0bWxfMmVjZmYyZmQ0ZjIyNDZhNDhkZjAyNDMzNTUxMTRiODkiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjExOTA3LCBMQUJFTDo2LCBTRUdNRU5UOldUQyAtIFJlc2lkZW50aWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF83MmY5MzdkNGMzODg0ODM3ODcyMTNhZDdhZjFiOTllZi5zZXRDb250ZW50KGh0bWxfMmVjZmYyZmQ0ZjIyNDZhNDhkZjAyNDMzNTUxMTRiODkpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl8zOTA2OTUzM2MzNDU0MzU3OTE1NzE1MDEzNTUzOGU0YS5iaW5kUG9wdXAocG9wdXBfNzJmOTM3ZDRjMzg4NDgzNzg3MjEzYWQ3YWYxYjk5ZWYpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfNTFmZjBjNGY4MTRiNDc5Zjg2NTFjMDc5NmViMDY0NTYgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4zOTU1MzIyLDExNC4xOTkwNTZdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fMTAxYTE3MmE5MzVkNDNiZWIwMGVhZjQ3NmZjYTc1NjggPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2JlaWdlJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzUxZmYwYzRmODE0YjQ3OWY4NjUxYzA3OTZlYjA2NDU2LnNldEljb24oaWNvbl8xMDFhMTcyYTkzNWQ0M2JlYjAwZWFmNDc2ZmNhNzU2OCk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF81ZGEzNmIzOGZlYjk0YTAyODFkMzgwZDQ3YjI3ZjMwOSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF8zMjYyYmQwZWUxOTA0YjgwODc1NTJjZGE5ODllMzAwMCA9ICQoJzxkaXYgaWQ9Imh0bWxfMzI2MmJkMGVlMTkwNGI4MDg3NTUyY2RhOTg5ZTMwMDAiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyNTY3LCBMQUJFTDo3LCBTRUdNRU5UOldUQyAtIFJlc2lkZW50aWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF81ZGEzNmIzOGZlYjk0YTAyODFkMzgwZDQ3YjI3ZjMwOS5zZXRDb250ZW50KGh0bWxfMzI2MmJkMGVlMTkwNGI4MDg3NTUyY2RhOTg5ZTMwMDApOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl81MWZmMGM0ZjgxNGI0NzlmODY1MWMwNzk2ZWIwNjQ1Ni5iaW5kUG9wdXAocG9wdXBfNWRhMzZiMzhmZWI5NGEwMjgxZDM4MGQ0N2IyN2YzMDkpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfZWQxNTExNTgyZmZlNDYxNThkZjc1M2EzMzM3MDAzMGMgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4yNDM4MzAxLDExNC4xNDY3NzMxXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uX2FiYTcwNGI3ODU1ZDRkZTBiMGI4ZDg1OWZiNThlYTljID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdiZWlnZScsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl9lZDE1MTE1ODJmZmU0NjE1OGRmNzUzYTMzMzcwMDMwYy5zZXRJY29uKGljb25fYWJhNzA0Yjc4NTVkNGRlMGIwYjhkODU5ZmI1OGVhOWMpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfMGE4ZmI1ZDRiNGQ5NDk0YjkzNjYzMDgzY2I3Nzc0OWIgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfZmY1YTE2NjFjODUzNDMyZDk1YzQ1MTMwMTAwMTcwNzIgPSAkKCc8ZGl2IGlkPSJodG1sX2ZmNWExNjYxYzg1MzQzMmQ5NWM0NTEzMDEwMDE3MDcyIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMTY2OSwgTEFCRUw6NywgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMGE4ZmI1ZDRiNGQ5NDk0YjkzNjYzMDgzY2I3Nzc0OWIuc2V0Q29udGVudChodG1sX2ZmNWExNjYxYzg1MzQzMmQ5NWM0NTEzMDEwMDE3MDcyKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfZWQxNTExNTgyZmZlNDYxNThkZjc1M2EzMzM3MDAzMGMuYmluZFBvcHVwKHBvcHVwXzBhOGZiNWQ0YjRkOTQ5NGI5MzY2MzA4M2NiNzc3NDliKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2YwNTk0MDhhYzQ4OTRkMjZhMjk2ZmYyMGI4Nzk2NzQ2ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuNTAwMzgxNywxMTQuMTQzNjk3Nl0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwXzRiMjcwNmM5NmQxNDRjNzU4OTlhNDZkZjVmYjBjOGMwKTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl80YTEyYzI1NzFhM2Q0ZmI2ODhlYjM0NTY5ZDg5YzNhNCA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ2luZm8tc2lnbicsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAncmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyX2YwNTk0MDhhYzQ4OTRkMjZhMjk2ZmYyMGI4Nzk2NzQ2LnNldEljb24oaWNvbl80YTEyYzI1NzFhM2Q0ZmI2ODhlYjM0NTY5ZDg5YzNhNCk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF84Mzc0YTY5MGZiZTk0YTBlYTM4ODYyYTI1YTNmNDlhYSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF9kM2Q4N2IzODFmMDk0N2NkYjQxYzgwNmYxMDI2MTRhYyA9ICQoJzxkaXYgaWQ9Imh0bWxfZDNkODdiMzgxZjA5NDdjZGI0MWM4MDZmMTAyNjE0YWMiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjExOTk5LCBMQUJFTDowLCBTRUdNRU5UOldUQyAtIFJlc2lkZW50aWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF84Mzc0YTY5MGZiZTk0YTBlYTM4ODYyYTI1YTNmNDlhYS5zZXRDb250ZW50KGh0bWxfZDNkODdiMzgxZjA5NDdjZGI0MWM4MDZmMTAyNjE0YWMpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl9mMDU5NDA4YWM0ODk0ZDI2YTI5NmZmMjBiODc5Njc0Ni5iaW5kUG9wdXAocG9wdXBfODM3NGE2OTBmYmU5NGEwZWEzODg2MmEyNWEzZjQ5YWEpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfZTQxMmJiMWNmMWI5NGU1OWIzNDU1Njc1M2M2N2VkOTcgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi40MzEyMzIsMTE0LjIwNDQ2NzddLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF80YjI3MDZjOTZkMTQ0Yzc1ODk5YTQ2ZGY1ZmIwYzhjMCk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fNWJlNTgyYmNiZWI0NDg4ZGE5NjI1N2ZjMjMwM2QxNTMgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdpbmZvLXNpZ24nLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2xpZ2h0cmVkJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyX2U0MTJiYjFjZjFiOTRlNTliMzQ1NTY3NTNjNjdlZDk3LnNldEljb24oaWNvbl81YmU1ODJiY2JlYjQ0ODhkYTk2MjU3ZmMyMzAzZDE1Myk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9lNDJhMGVhMGI1MDk0YzAzYmM1ZDY3NTY1OGZlZjNmZSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF83Y2UzYTQ0Mzg0NGQ0MjY5YmM1YWIzMDVmYWFiNTVkNiA9ICQoJzxkaXYgaWQ9Imh0bWxfN2NlM2E0NDM4NDRkNDI2OWJjNWFiMzA1ZmFhYjU1ZDYiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPlNUT1JFX0lEOjEyMDcxLCBMQUJFTDo2LCBTRUdNRU5UOldUQyAtIFJlc2lkZW50aWFsPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9lNDJhMGVhMGI1MDk0YzAzYmM1ZDY3NTY1OGZlZjNmZS5zZXRDb250ZW50KGh0bWxfN2NlM2E0NDM4NDRkNDI2OWJjNWFiMzA1ZmFhYjU1ZDYpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl9lNDEyYmIxY2YxYjk0ZTU5YjM0NTU2NzUzYzY3ZWQ5Ny5iaW5kUG9wdXAocG9wdXBfZTQyYTBlYTBiNTA5NGMwM2JjNWQ2NzU2NThmZWYzZmUpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfOTI0ZTVmMWYwYWIxNGJhYmFjY2Q0YTJlNjFmNTI1OTggPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFsyMi4zODY2NTI2LDExNC4yMDQwNjE0XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzc0YWEyNzZjNDdhNDQyZDRhMTVhNjJlNTkzNDkzZTk5ID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdncmVlbicsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl85MjRlNWYxZjBhYjE0YmFiYWNjZDRhMmU2MWY1MjU5OC5zZXRJY29uKGljb25fNzRhYTI3NmM0N2E0NDJkNGExNWE2MmU1OTM0OTNlOTkpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfYzg3MTZmZWQ3MDNhNGU0M2JmZDc2NzBlYWVhMjcyZjEgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfYzA5MzAwMjVlNWMyNGUzYWI0MTRmOTk2YzllOGM2ODcgPSAkKCc8ZGl2IGlkPSJodG1sX2MwOTMwMDI1ZTVjMjRlM2FiNDE0Zjk5NmM5ZThjNjg3IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5TVE9SRV9JRDoxMjQ2NSwgTEFCRUw6MiwgU0VHTUVOVDpXVEMgLSBSZXNpZGVudGlhbDwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfYzg3MTZmZWQ3MDNhNGU0M2JmZDc2NzBlYWVhMjcyZjEuc2V0Q29udGVudChodG1sX2MwOTMwMDI1ZTVjMjRlM2FiNDE0Zjk5NmM5ZThjNjg3KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfOTI0ZTVmMWYwYWIxNGJhYmFjY2Q0YTJlNjFmNTI1OTguYmluZFBvcHVwKHBvcHVwX2M4NzE2ZmVkNzAzYTRlNDNiZmQ3NjcwZWFlYTI3MmYxKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzI2MWQzYjU1MGI4OTRkMTc4NmJlZDJkMDUxZWVmYmM3ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMjIuMjc5MzcyNywxMTQuMTg0NDU5XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNGIyNzA2Yzk2ZDE0NGM3NTg5OWE0NmRmNWZiMGM4YzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzU5MDBkNDNhMWIxMzQ5NjZhZTc3ODE5ZDQ4YjllODkxID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnaW5mby1zaWduJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdvcmFuZ2UnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfMjYxZDNiNTUwYjg5NGQxNzg2YmVkMmQwNTFlZWZiYzcuc2V0SWNvbihpY29uXzU5MDBkNDNhMWIxMzQ5NjZhZTc3ODE5ZDQ4YjllODkxKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzAyMGU1YjRhN2VjNzRjNzc5OTNmMGIzMzMzMTU2Yjc5ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzg3NTM4YWUwNzkyNzRhYTQ5NWNmYWI1YzA0MzgzNGVlID0gJCgnPGRpdiBpZD0iaHRtbF84NzUzOGFlMDc5Mjc0YWE0OTVjZmFiNWMwNDM4MzRlZSIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+U1RPUkVfSUQ6MTI2MTEsIExBQkVMOjQsIFNFR01FTlQ6V1RDIC0gRGVzdGluYXRpb248L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzAyMGU1YjRhN2VjNzRjNzc5OTNmMGIzMzMzMTU2Yjc5LnNldENvbnRlbnQoaHRtbF84NzUzOGFlMDc5Mjc0YWE0OTVjZmFiNWMwNDM4MzRlZSk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzI2MWQzYjU1MGI4OTRkMTc4NmJlZDJkMDUxZWVmYmM3LmJpbmRQb3B1cChwb3B1cF8wMjBlNWI0YTdlYzc0Yzc3OTkzZjBiMzMzMzE1NmI3OSk7CgogICAgICAgICAgICAKICAgICAgICAKPC9zY3JpcHQ+" style="position:absolute;width:100%;height:100%;left:0;top:0;border:none !important;" allowfullscreen webkitallowfullscreen mozallowfullscreen></iframe></div></div>


