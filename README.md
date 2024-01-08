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
<img width="598" alt="Screenshot 2024-01-08 at 8 40 22 PM" src="https://github.com/phanhoangminh99/RFM-Analysis/assets/115093313/f66bddc4-2886-4fd3-9d6d-de48720b2b39">


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

<img width="925" alt="Screenshot 2024-01-08 at 8 37 20 PM" src="https://github.com/phanhoangminh99/RFM-Analysis/assets/115093313/a4c55a7a-d81c-4225-a2c2-282c101a9cdc">


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

<img width="555" alt="Screenshot 2024-01-08 at 8 38 01 PM" src="https://github.com/phanhoangminh99/RFM-Analysis/assets/115093313/3832ac15-6b40-4caa-a704-95ff64342522">

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
<img width="449" alt="Screenshot 2024-01-08 at 8 38 38 PM" src="https://github.com/phanhoangminh99/RFM-Analysis/assets/115093313/8ca7a1ed-fcbf-4bef-801a-55663d8c1e47">


 ```php
# Pie chart of Ship mode
Orders_RFM_ship = Orders_RFM_m.groupby("Ship Mode").agg(Ship_count=("Ship Mode","count")).reset_index()
plt.pie(Orders_RFM_ship["Ship_count"], labels=Orders_RFM_ship["Ship Mode"], autopct='%.0f%%')
plt.title('Percentage of Ship mode')
plt.show()
```

<img width="428" alt="Screenshot 2024-01-08 at 8 38 49 PM" src="https://github.com/phanhoangminh99/RFM-Analysis/assets/115093313/7ad52c16-da05-49c0-98b9-8f0d713279d1">


Next, we analyze sales by product category and sub-category using pie charts to understand product performance.


 ```php
# Pie chart of category
Orders_Product_category = Orders_Product.groupby("Category").agg(Category_count=("Category","count")).reset_index()
plt.pie(Orders_Product_category["Category_count"], labels=Orders_Product_category["Category"], autopct='%.0f%%')
plt.title('Percentage of Category')
plt.show()
```


<img width="545" alt="Screenshot 2024-01-08 at 8 39 04 PM" src="https://github.com/phanhoangminh99/RFM-Analysis/assets/115093313/1c09b835-2be5-4d8f-a972-33dca0855711">


 ```php
# Pie chart of sub-category
Orders_Product_subcategory = Orders_Product.groupby("Sub-Category").agg(SubCategory_count=("Sub-Category","count")).reset_index()
plt.pie(Orders_Product_subcategory["SubCategory_count"], labels=Orders_Product_subcategory["Sub-Category"], autopct='%.0f%%')
plt.title('Percentage of SubCategory')
plt.show()
```
<img width="519" alt="Screenshot 2024-01-08 at 8 39 16 PM" src="https://github.com/phanhoangminh99/RFM-Analysis/assets/115093313/de9a397d-9fd6-4512-aae6-16412bd3f020">



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

<img width="599" alt="Screenshot 2024-01-08 at 8 39 31 PM" src="https://github.com/phanhoangminh99/RFM-Analysis/assets/115093313/c5e5d560-fb39-4061-89ef-e485f6ec2915">


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
<img width="558" alt="Screenshot 2024-01-08 at 8 39 46 PM" src="https://github.com/phanhoangminh99/RFM-Analysis/assets/115093313/2dad3c67-a0ba-4a98-a1ac-6a53bce7bdd7">


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

<img width="577" alt="Screenshot 2024-01-08 at 8 39 55 PM" src="https://github.com/phanhoangminh99/RFM-Analysis/assets/115093313/acfb11c0-70c4-4865-948d-ba384003e019">

