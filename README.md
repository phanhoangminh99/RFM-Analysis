# RFM Analysis 

## Data Loading and Processing
  #### A. Import libraries

 ```php 
import pandas as pd
import numpy as np
from datetime import datetime, timedelta
import matplotlib.pyplot as plt
import seaborn as sns
import plotly.express as px
```

  #### B. Load dataset

```php
xlsx =pd.ExcelFile("Dataset.xlsx")
Orders=xlsx.parse('Orders')
Product= xlsx.parse('Product')
Location= xlsx.parse('Location')
Customer= xlsx.parse('Customer')
Return= xlsx.parse('Return')
RFM_Score= xlsx.parse('RFM Score')
```


  #### C. Find missing or error values

```php
Orders.isna().any()
Product.isna().any()
Location.isna().any()
Customer.isna().any()
Return.isna().any()
```


## Data Preparation & Visualization

 #### A. Preprocess data 
 
  ```php 
 # Group by returned product id and count
R_count = Return.groupby(["Order ID"])["Returned"].agg("count").reset_index()

  # Merging with left join
Orders_R= Orders.merge(R_count, how="left", on="Order ID")

  # Filling NaN values in the Returned column with 0
Orders_R.fillna(value=0, inplace=True)

  # Calculating the frequency of each order
Count_Frequency=Orders_R.groupby("Order ID").agg(Count_F=("Order ID","count")).reset_index()

```
#### B. Visualize customer segmentation 


 ```php 
sns.countplot(x=Count_Frequency["Count_F"])
plt.title('Number of customer segmentation')
plt.xlabel('Purchase number')
plt.ylabel('Count of Number of purchase')
plt.show()
```

<img width="621" alt="Screenshot 2024-01-08 at 11 19 12 AM" src="https://github.com/phanhoangminh99/Testing/assets/115093313/22f3cd04-2309-4d81-bc9b-651ad580efe4">

## More Data Preparation for RFM Analysis

 #### A. Calculate RFM scores for each customer based on order history


 ```php
# Calculate profit per unit
Orders_R["Profit_per_unit"]=Orders_R["Profit"]/Orders_R["Quantity"]

# Calculate quantity after returned product
Orders_R["Quantity_R"]=Orders_R["Quantity"]- Orders_R["Returned"]

# Calculate sales after returned product
Orders_R["Sales_R"]=Orders_R["Quantity_R"]*(Orders_R["Unit Price"]+Orders_R["Profit_per_unit"])

# Setting a reference date for recency calculation
d = datetime(year=2017, month=12, day=31) + timedelta(days=1)

# Grouping orders to create a customer DataFrame for RFM analysis
df_customers= Orders_R.groupby(["Order ID"]).agg(
    {"Order Date": lambda x: (d-x.max()).days,"Quantity_R":"count",
    "Sales_R":"sum"}).reset_index()
```

#### B. Assign score

 ```php
# Divide RFM results by levels 1,2,3,4,5
df_customers["Recency_score"] = pd.cut(df_customers["Order Date"],5, labels =[5,4,3,2,1])
df_customers["Frequency_score"] = pd.cut(df_customers["Quantity_R"],5, labels =[1,2,3,4,5])
df_customers["Monetary_score"] = pd.cut(df_customers["Sales_R"],5, labels =[1,2,3,4,5])
```

Here I assign scores for Recency (how recently they purchased), Frequency (how often they purchase), and Monetary value (how much they spend).

#### C. RFM Score Integration and Customer Grouping

 ```php
# Integrate Recency, Frequency, and Monetary scores into RFM scores
df_customers["RFM_score"] = df_customers.apply(lambda row: str(row["Recency_score"])+
                                              str(row["Frequency_score"])+
                                              str(row["Monetary_score"]), axis=1)
# Split and restructure RFM scores for further analysis
RFM_Score['RFM_Score_List'] = RFM_Score['RFM Score'].str.split(', ')
RFM_Score_split = RFM_Score.explode('RFM_Score_List')
RFM_Score_split.drop(columns=["RFM Score"], inplace=True)
RFM_Score_split.rename(columns={"RFM_Score_List": "RFM_score"}, inplace=True)

# Merge customer RFM scores with original data and grouping by customer
customers_RFM = df_customers.merge(RFM_Score_split, how="left", on ="RFM_score")

# Group orders and aggregating quantities by customer ID
Orders_group= Orders_R.groupby(["Order ID", "Customer ID"])["Quantity"].sum().reset_index()

# Merge customer RFM data with order quantities
Orders_RFM = customers_RFM.merge(Orders_group, how="left", on="Order ID").reset_index()
```

## Customer Segmentation Analysis

 #### A. Create treemap and countplot for customer segmentation

 ```php
# Build a treemap of customer segmentation
treemap_data= Orders_RFM.groupby("Segment").agg(Segment_count=("Segment", "count")).reset_index()
fig = px.treemap(treemap_data, path=['Segment'], values='Segment_count', title='Treemap of customer segmentation')
fig.show()
```

Visualize customer segments using a treemap and countplot to understand their distribution.

<img width="944" alt="Screenshot 2024-01-08 at 11 28 50 AM" src="https://github.com/phanhoangminh99/Testing/assets/115093313/5f7fc39c-66b2-4b12-89aa-aad69742324d">


#### B. Visualize sales and profit by segmentation

 ```php
# Seaborn Countplot of customer segmentation
sns.countplot(x=Orders_RFM["Segment"])
plt.title('Number of customer segmentation')
plt.xlabel('Segmentation')
plt.ylabel('Count of segmentation')
plt.xticks(rotation=45)
plt.show()
```
<img width="624" alt="Screenshot 2024-01-08 at 11 30 36 AM" src="https://github.com/phanhoangminh99/Testing/assets/115093313/ae8ff4db-2fcc-48ba-b1c1-b92b81925a25">

Analyze sales and profit by segmentation using bar plots to identify high-value segments.

## Sales and Distribution Analysis:

#### A. Data Preparation 


 ```php
# Orders_RFM left join with Orders_new
Orders_new = Orders.drop_duplicates(subset='Order ID')
Orders_RFM_m = Orders_RFM.merge(Orders_new, how="left", on=["Order ID","Customer ID"])

# Orders_RFM_m left join with Product_new
Product_new= Product.drop_duplicates(subset='Product ID')
Orders_Product =Orders_RFM_m.merge(Product_new, how="left", on="Product ID")
```

#### B. Visualizations for Various Metrics

First, I'll explore the distribution of orders across channels (e.g., online, retail) and ship modes (e.g., standard, express) using pie charts.

 ```php
# Pie chart of Channel
Orders_RFM_channel = Orders_RFM_m.groupby("Channel").agg(Channel_count=("Channel", "count")).reset_index()
plt.pie(Orders_RFM_channel["Channel_count"], labels=Orders_RFM_channel["Channel"], autopct='%1.0f%%')
plt.title('Percentage of channel')
plt.show()
```

<img width="414" alt="Screenshot 2024-01-08 at 11 32 21 AM" src="https://github.com/phanhoangminh99/Testing/assets/115093313/ccc8abd5-f2de-4fa8-80c1-121dc77c9368">

 ```php
# Pie chart of Ship mode
Orders_RFM_ship = Orders_RFM_m.groupby("Ship Mode").agg(Ship_count=("Ship Mode","count")).reset_index()
plt.pie(Orders_RFM_ship["Ship_count"], labels=Orders_RFM_ship["Ship Mode"], autopct='%.0f%%')
plt.title('Percentage of Ship mode')
plt.show()
```

<img width="423" alt="Screenshot 2024-01-08 at 11 32 46 AM" src="https://github.com/phanhoangminh99/Testing/assets/115093313/1779711b-c77c-4f57-974b-1d40bd0acc50">


Next, we analyze sales by product category and sub-category using pie charts to understand product performance.


 ```php
# Pie chart of category
Orders_Product_category = Orders_Product.groupby("Category").agg(Category_count=("Category","count")).reset_index()
plt.pie(Orders_Product_category["Category_count"], labels=Orders_Product_category["Category"], autopct='%.0f%%')
plt.title('Percentage of Category')
plt.show()
```

<img width="524" alt="Screenshot 2024-01-08 at 11 33 30 AM" src="https://github.com/phanhoangminh99/Testing/assets/115093313/94f44ba4-7704-4f1a-a69c-d05ce302e71f">



 ```php
# Pie chart of sub-category
Orders_Product_subcategory = Orders_Product.groupby("Sub-Category").agg(SubCategory_count=("Sub-Category","count")).reset_index()
plt.pie(Orders_Product_subcategory["SubCategory_count"], labels=Orders_Product_subcategory["Sub-Category"], autopct='%.0f%%')
plt.title('Percentage of SubCategory')
plt.show()
```

<img width="480" alt="Screenshot 2024-01-08 at 11 34 12 AM" src="https://github.com/phanhoangminh99/Testing/assets/115093313/816143ef-99ed-43e5-8b1c-edf74a9fee2b">


Then, we analyze sales and profit by segmentation using a bar plot 


 ```php
# Total Sales by Segmentation
Sales_seg=Orders_RFM_m.groupby("Segment").agg(sum_sales=("Sales", "sum")).reset_index()
plt.bar(Sales_seg["Segment"],Sales_seg["sum_sales"])
for i, v in enumerate(round(Sales_seg["sum_sales"],2)):
    plt.text(i-.25, v+0.5, str(v), color='orange', fontweight='bold', ha='center', va='bottom')
plt.title('Total Sales by Segmentation')
plt.xlabel('Segmentation')
plt.ylabel('Total Sales')
plt.xticks(rotation=45)
plt.show()
```

<img width="647" alt="Screenshot 2024-01-08 at 11 34 59 AM" src="https://github.com/phanhoangminh99/Testing/assets/115093313/caafe51a-7054-4095-97b1-c5c70a2dd02c">



 ```php
# Total Profit by Segmentation
Profit_seg=Orders_RFM_m.groupby("Segment").agg(sum_profit=("Profit", "sum")).reset_index()
plt.bar(Profit_seg["Segment"],Profit_seg["sum_profit"])
for i, v in enumerate(round(Profit_seg["sum_profit"],2)):
    plt.text(i-.25, v+0.5, str(v), color='green', fontweight='bold', ha='center', va='bottom')
plt.title('Total Profit by Segmentation')
plt.xlabel('Segmentation')
plt.ylabel('Total profit')
plt.xticks(rotation=45)
plt.show()
```

<img width="643" alt="Screenshot 2024-01-08 at 11 35 12 AM" src="https://github.com/phanhoangminh99/Testing/assets/115093313/dee2edeb-56ce-4f4a-a8b9-4d269c1fb180">

Finally, we want to identify sales by region using a bar plot to identify regional trends and potential geographic targeting opportunities.

 ```php
# Total Sales by Region
Sales_Region=Orders_Product_Location.groupby("Region").agg(sum_sales=("Sales", "sum")).reset_index()
plt.bar(Sales_Region["Region"],Sales_Region["sum_sales"])
for i, v in enumerate(round(Sales_Region["sum_sales"],2)):
    plt.text(i-.25, v+0.5, str(v), color='orange', fontweight='bold', ha='center', va='bottom')
plt.title('Total Sales by Region')
plt.xlabel('Region')
plt.ylabel('Total Sales')
plt.xticks(rotation=45)
plt.show()
```

<img width="639" alt="Screenshot 2024-01-08 at 11 35 41 AM" src="https://github.com/phanhoangminh99/Testing/assets/115093313/3c7826cf-504e-4adf-9be7-96a1b55490f2">
