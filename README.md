# Superstore Sales Analysis
Welcome to my first data analytics project! This repository contains my end-to-end analysis of a 9,994-row e-commerce dataset, focusing on moving past basic reporting and digging into data storytelling.

**Why This Project?**

"Anyone with a basic working knowledge of SQL or spreadsheets can compute a profit margin or flag a delayed shipment. The math is foundational—but simply displaying numbers is no longer enough. Every dataset has a story to tell."

I chose the Superstore dataset because it provides a tight, manageable structure while offering enough depth (nearly 10,000 records) to establish a real narrative. Even though this data is fictional and cleaner than real-world corporate data, it accurately mirrors the chaotic reality of e-commerce operations, warts and all.

> [!WARNING]
Status: 🚧 Work in Progress — Currently building out the data cleaning scripts and initial exploratory analysis.

<details>
<summary>Click here to expand the table</summary>

|row_id|order_id      |order_date|ship_date |ship_mode     |customer_id|customer_name         |segment    |country      |city             |state               |postal_code|region |product_id     |category       |subcategory|product_name                                                                                                                   |sales    |quantity|discount|profit    |
|------|--------------|----------|----------|--------------|-----------|----------------------|-----------|-------------|-----------------|--------------------|-----------|-------|---------------|---------------|-----------|-------------------------------------------------------------------------------------------------------------------------------|---------|--------|--------|----------|
|1     |CA-2016-152156|11/8/2016 |11/11/2016|Second Class  |CG-12520   |Claire Gute           |Consumer   |United States|Henderson        |Kentucky            |42420      |South  |FUR-BO-10001798|Furniture      |Bookcases  |Bush Somerset Collection Bookcase                                                                                              |261.96   |2       |0       |41.9136   |
|2     |CA-2016-152156|11/8/2016 |11/11/2016|Second Class  |CG-12520   |Claire Gute           |Consumer   |United States|Henderson        |Kentucky            |42420      |South  |FUR-CH-10000454|Furniture      |Chairs     |Hon Deluxe Fabric Upholstered Stacking Chairs, Rounded Back                                                                    |731.94   |3       |0       |219.582   |
|3     |CA-2016-138688|6/12/2016 |6/16/2016 |Second Class  |DV-13045   |Darrin Van Huff       |Corporate  |United States|Los Angeles      |California          |90036      |West   |OFF-LA-10000240|Office Supplies|Labels     |Self-Adhesive Address Labels for Typewriters by Universal                                                                      |14.62    |2       |0       |6.8714    |
|4     |US-2015-108966|10/11/2015|10/18/2015|Standard Class|SO-20335   |Sean O'Donnell        |Consumer   |United States|Fort Lauderdale  |Florida             |33311      |South  |FUR-TA-10000577|Furniture      |Tables     |Bretford CR4500 Series Slim Rectangular Table                                                                                  |957.5775 |5       |0.45    |-383.031  |
|5     |US-2015-108966|10/11/2015|10/18/2015|Standard Class|SO-20335   |Sean O'Donnell        |Consumer   |United States|Fort Lauderdale  |Florida             |33311      |South  |OFF-ST-10000760|Office Supplies|Storage    |Eldon Fold 'N Roll Cart System                                                                                                 |22.368   |2       |0.2     |2.5164    |

*Table 1. Superstore Sample*

</details>

---

## The Promotion Leak - SQL

The first thing that I noticed about the dataset is it shows negative profit on some sales and traced it back to discounts as there are no zero-percent discounts that outputs negative profit. The losses might be of significant value so with a quick assessment, I looked into it.

IN:
```sql
SELECT SUM(profit) AS net_profit
FROM superstore.sales;
```

OUT:
> 286397.79

IN:
```sql
SELECT SUM(profit) AS total_loss
FROM superstore.sales
WHERE profit < 0;
```

OUT:
> -156131.86

With some quick maths, it's visible that the company lost about ***MORE THAN A THIRD*** of its earnings due to aggressive discounts. However, if I'm to present these to shareholders, I can't exactly just show these numbers, tell them to cut the discounts, and call it a day. I might not have a degree in business but in a realistic world, space taken in the warehouse that remains dormant is space that's not earning. Discounts are there to expedite the movement of slow-moving inventory, that's why its presence is as constant as taxes.

Now to address this issue, which categorization do I want to split the entries with? Shareholders deal with **strategic categories** that executives can actually make decisions about. However, down the supply chain, store managers will also be able to make use of discount caps for each item so it's better to include it in as well.

```sql
SELECT region,
	category,
	sub_category,
	product_id,
	product_name,
```

For the computation, we'll be needing the following. We can cut some afterwards but it doesn't hurt to keep everything on the result, as it might be helpful in our presentation later. If the database ever struggles to process the information, computation for "original_price" and "zero_profit_sales" can be removed. I decided to keep it in for clarity.

> sales : the final total sale for each transaction; included in the database
> 
> discount : percentage of discount that the transaction is made with. This is already factored in on the "sales"; included in the database
> 
> profit : gross profit of the transaction, includes both positive and negative; included in the database
>
> original_price : item price if discount is zero
> 
> zero_profit_sales : absolute lowest item price for the transaction to break even at $0 sales and avoid losses
> 
> max_discount : the absolute highest discount for the transaction to break even at $0 sales and avoid losses

```sql
SELECT region,
	category,
	sub_category,
	product_id,
	product_name,
	sales,
	discount,
	profit,
	sales / (1 - discount) AS original_price,
	(sales - profit) AS zero_profit_sales,
	1 - ((sales - profit) / (sales / (1 - discount))) AS max_discount
FROM superstore.sales
```

As a last safety measure in the long run, although the highest discount is only at 80%, I've put in failsafes if an event occured that the store would be giving items away for 100% off (Itemizing freebies and giveaways for stock purposes, buy 2 take 1's, who knows. Better safe than sorry.)

**IN:**

```sql
SELECT region,
	category,
	sub_category,
	product_id,
	product_name,
	sales,
	discount,
	profit,
	CASE -- Prevents division by zero if discount is 100% (1.0)
		WHEN discount = 1 THEN NULL 
		ELSE sales / (1 - discount) 
		END AS original_price,
	(sales - profit) AS zero_profit_sales,
	CASE -- Calculates max discount safely
		WHEN sales = 0 THEN NULL -- Prevents division by zero if sales are 0
		WHEN discount = 1 THEN 0   -- If it was given away for free, max break-even discount is effectively 0
		ELSE 1 - ((sales - profit) / (sales / (1 - discount))) 
		END AS max_discount
FROM superstore.sales
```

**OUT:**

|region |category       |sub_category|product_id     |product_name                                                                                                                   |sales   |discount|profit  |original_price        |zero_profit_sales|max_discount           |
|-------|---------------|------------|---------------|-------------------------------------------------------------------------------------------------------------------------------|--------|--------|--------|----------------------|-----------------|-----------------------|
|South  |Furniture      |Bookcases   |FUR-BO-10001798|Bush Somerset Collection Bookcase                                                                                              |261.96  |0.00    |41.91   |261.9600000000000000  |220.05           |0.15998625744388456253 |
|South  |Furniture      |Chairs      |FUR-CH-10000454|Hon Deluxe Fabric Upholstered Stacking Chairs, Rounded Back                                                                    |731.94  |0.00    |219.58  |731.9400000000000000  |512.36           |0.29999726753559034894 |
|West   |Office Supplies|Labels      |OFF-LA-10000240|Self-Adhesive Address Labels for Typewriters by Universal                                                                      |14.62   |0.00    |6.87    |14.6200000000000000   |7.75             |0.46990424076607387141 |
|South  |Furniture      |Tables      |FUR-TA-10000577|Bretford CR4500 Series Slim Rectangular Table                                                                                  |957.58  |0.45    |-383.03 |1741.0545454545454545 |1340.61          |0.23000114872908790908 |
|South  |Office Supplies|Storage     |OFF-ST-10000760|Eldon Fold 'N Roll Cart System                                                                                                 |22.37   |0.20    |2.52    |27.9625000000000000   |19.85            |0.29012069736253911489 |

*Table 2: The Promotion Leak Sample - *

---

## The Promotion Leak - PowerBI

Finally, the math is done. Sadly, not everyone shares our love for data in its raw form. Time to give the story some dashboard-worthy makeover

