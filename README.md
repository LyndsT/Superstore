# Superstore Sales Analysis
Welcome to my first data analytics project! This repository contains my end-to-end analysis of a 9,994-row e-commerce dataset, focusing on moving past basic reporting and digging into data storytelling.

**Why This Project?**

"Anyone with a basic working knowledge of SQL or spreadsheets can compute a profit margin or flag a delayed shipment. The math is foundational—but simply displaying numbers is no longer enough. Every dataset has a story to tell."

I chose the Superstore dataset because it provides a tight, manageable structure while offering enough depth (nearly 10,000 records) to establish a real narrative. Even though this data is fictional and cleaner than real-world corporate data, it accurately mirrors the chaotic reality of e-commerce operations, warts and all.

> [!WARNING]
Status: 🚧 Work in Progress — Currently building out the data cleaning scripts and initial exploratory analysis.

| Row ID | Order ID | Order Date | Ship Date | Ship Mode | Customer ID | Customer Name | Segment | Country | City | State | Postal Code | Region | Product ID | Category | Sub-Category | Product Name | Sales | Qty | Discount | Profit |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | ---: | ---: | ---: | ---: |
| 1 | CA-2016-152156 | 11/08/2016 | 11/11/2016 | Second Class | CG-12520 | Claire Gute | Consumer | United States | Henderson | Kentucky | 42420 | South | FUR-BO-10001798 | Furniture | Bookcases | Bush Somerset Collection Bookcase | 261.96 | 2 | 0.00% | 41.91 |
| 2 | CA-2016-152156 | 11/08/2016 | 11/11/2016 | Second Class | CG-12520 | Claire Gute | Consumer | United States | Henderson | Kentucky | 42420 | South | FUR-CH-10000454 | Furniture | Chairs | Hon Deluxe Fabric Upholstered Stacking Chairs, Rounded Back | 731.94 | 3 | 0.00% | 219.58 |
| 3 | CA-2016-138688 | 06/12/2016 | 06/16/2016 | Second Class | DV-13045 | Darrin Van Huff | Corporate | United States | Los Angeles | California | 90036 | West | OFF-LA-10000240 | Office Supplies | Labels | Self-Adhesive Address Labels for Typewriters by Universal | 14.62 | 2 | 0.00% | 6.87 |
| 4 | US-2015-108966 | 10/11/2015 | 10/18/2015 | Standard Class | SO-20335 | Sean O'Donnell | Consumer | United States | Fort Lauderdale | Florida | 33311 | South | FUR-TA-10000577 | Furniture | Tables | Bretford CR4500 Series Slim Rectangular Table | 957.58 | 5 | 45.00% | -383.03 |
| 5 | US-2015-108966 | 10/11/2015 | 10/18/2015 | Standard Class | SO-20335 | Sean O'Donnell | Consumer | United States | Fort Lauderdale | Florida | 33311 | South | OFF-ST-10000760 | Office Supplies | Storage | Eldon Fold 'N Roll Cart System | 22.37 | 2 | 20.00% | 2.52 |

*Table 1. Superstore Sample*

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

| Column | Description |
| :--- | :--- |
| **sales** | The final total sale for each transaction (included in the database). |
| **discount** | Percentage of discount that the transaction is made with. This is already factored into "sales" (included in the database). |
| **profit** | Gross profit of the transaction, including both positive and negative values (included in the database). |
| **original_price** | Item price if the discount is zero. |
| **zero_profit_sales** | Absolute lowest item price for the transaction to break even at $0 profit and avoid losses. |
| **max_discount** | The absolute highest discount for the transaction to break even at $0 profit and avoid losses. |

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

| Region | Category | Sub-Category | Product ID | Product Name | Sales | Discount | Profit | Original Price | Zero-Profit Sales | Max Discount |
| :--- | :--- | :--- | :--- | :--- | ---: | ---: | ---: | ---: | ---: | ---: |
| South | Furniture | Bookcases | FUR-BO-10001798 | Bush Somerset Collection Bookcase | 261.96 | 0.00% | 41.91 | 261.96 | 220.05 | 16.00% |
| South | Furniture | Chairs | FUR-CH-10000454 | Hon Deluxe Fabric Upholstered Stacking Chairs, Rounded Back | 731.94 | 0.00% | 219.58 | 731.94 | 512.36 | 30.00% |
| West | Office Supplies | Labels | OFF-LA-10000240 | Self-Adhesive Address Labels for Typewriters by Universal | 14.62 | 0.00% | 6.87 | 14.62 | 7.75 | 46.99% |
| South | Furniture | Tables | FUR-TA-10000577 | Bretford CR4500 Series Slim Rectangular Table | 957.58 | 45.00% | -383.03 | 1,741.05 | 1,340.61 | 23.00% |
| South | Office Supplies | Storage | OFF-ST-10000760 | Eldon Fold 'N Roll Cart System | 22.37 | 20.00% | 2.52 | 27.96 | 19.85 | 29.01% |

*Table 2: The Promotion Leak Sample*

Lastly, to make our work easier, we can make use of SQL as well to see which region - subcategory we need to focus on. We'll be gathering the profit, loss, and the percentage increase if the loss were to breakeven to garner the maximum gain the company can gain

**IN:**

```sql
WITH profit_table AS(
SELECT region,
	sub_category,
	SUM(CASE
		WHEN profit < 0 THEN 0
		ELSE profit
		END) AS profit,
	SUM(CASE
		WHEN profit > 0 THEN 0
		ELSE profit
		END) AS loss
FROM superstore.sales
GROUP BY sub_category, region
ORDER BY region
)

SELECT pt.region,
	pt.sub_category,
	pt.profit,
	pt.loss,
	(ABS(pt.loss) / pt.profit) AS percentage_increase
FROM profit_table pt
ORDER BY percentage_increase DESC;
```

**OUT:**

| Region | Sub-Category | Profit | Loss | % Increase |
| :--- | :--- | ---: | ---: | ---: |
| East | Tables | 20.97 | -11,046.36 | 526.77% |
| Central | Supplies | 198.40 | -860.28 | 4.34% |
| East | Supplies | 406.67 | -1,561.78 | 3.84% |
| Central | Bookcases | 949.11 | -2,947.03 | 3.11% |
| Central | Furnishings | 2,038.46 | -5,944.64 | 2.92% |
| Central | Tables | 3,008.69 | -6,568.37 | 2.18% |
| South | Tables | 4,217.71 | -8,840.77 | 2.10% |
| Central | Machines | 1,418.06 | -2,904.13 | 2.05% |
| West | Bookcases | 2,577.17 | -4,223.67 | 1.64% |
| Central | Appliances | 5,991.06 | -8,629.67 | 1.44% |

*Table 3. Points of Interest - Biggest % Increase*

While we're at it, let's also consider the region - subcategory combination with the biggest net loss by altering the last line of the last query

**IN:**

```sql
...
ORDER BY pt.loss ASC;
```

**OUT:**

| Region | Sub-Category | Profit | Loss | % Increase |
| :--- | :--- | ---: | ---: | ---: |
| Central | Binders | 20,865.75 | -21,909.46 | 1.05% |
| East | Machines | 20,918.85 | -13,990.20 | 0.67% |
| East | Tables | 20.97 | -11,046.36 | 526.77% |
| South | Tables | 4,217.71 | -8,840.77 | 2.10% |
| Central | Appliances | 5,991.06 | -8,629.67 | 1.44% |
| South | Binders | 12,305.16 | -8,404.51 | 0.68% |
| South | Machines | 6,196.34 | -7,635.24 | 1.23% |
| East | Phones | 19,030.53 | -6,715.83 | 0.35% |
| Central | Tables | 3,008.69 | -6,568.37 | 2.18% |
| East | Binders | 17,239.58 | -5,971.66 | 0.35% |

*Table 4. Points of Interest - Biggest Net Loss*

Those are quite the significant findings no? Very interesting data that we can look into in a deeper manner.
---

## The Promotion Leak - PowerBI

Finally, the math is done. Sadly, not everyone shares our love for data in its raw form. Time to give the story some dashboard-worthy makeover

Since the goal is to identify the profit leak, it's best to see the most prevalent one. There are subcategories that balances their profit by earning more than they lose. This dashboard lets us see the subcategories that are the leading profit losers.

<img width="1408" height="791" alt="image" src="https://github.com/user-attachments/assets/10b630a3-8643-4632-9f33-d2053d05c8ef" />

For the first dashboard, it shows the overall generalized overview of the company's gross profit, as well as its products sold. The slicers on the left are used to see the differences between the Regions and the Categories. In turn, it shows the movement of profit per subcategory on the right.

<img width="1407" height="791" alt="image" src="https://github.com/user-attachments/assets/72e704c0-1791-4f1b-9b85-1f3c1b95d336" />

For the second dashboard, it caters for a closer look on the sales. The subcategories are made into slicers as well. For detailed analysis, a table indicates each item's subcategory, together with its Current Discount (Avg), Discount Cap (Avg), Items Sold (Count), and Profit (Sum).

---

## Insights

The data indicates that the **Tables Subcategory** represents the most significant profit drag, culminating in a net loss of $ -11.05k due to aggressive discounts. This profit loss is anomalously significant as adopting the discount cap for each item would bump the profit for a +526.77% increase in the subcategory. For the bigger picture, it alone would increase the entire profit of East Region by +12.06%. 

<img width="1114" height="392" alt="image" src="https://github.com/user-attachments/assets/683b3fc4-d94b-4a28-b122-f21a62aabd46" />



---

The second point of interest leads away from subcategories and onto region. 6 out of the 10 leading profit leakage comes from the **Central region** . While seemingly harmless unlike Eastern region's Tables, its losses cumulatively adds up to a $ -27,854.12 net loss, turning it into the region with the lowest profit. More on this on the next finding.

---

Lastly, I admittedly only recognized that what I missed was just the raw value alone. I was so fixated on the discounts and their percentages that it went over my head. And it was also the reason why I made the prompt for Table 4.

<details>
  <summary> <b> Click here for Table 4 recap </b> </summary>

| Region | Sub-Category | Profit | Loss | % Increase |
| :--- | :--- | ---: | ---: | ---: |
| Central | Binders | 20,865.75 | -21,909.46 | 1.05% |
| East | Machines | 20,918.85 | -13,990.20 | 0.67% |
| East | Tables | 20.97 | -11,046.36 | 526.77% |
| South | Tables | 4,217.71 | -8,840.77 | 2.10% |
| Central | Appliances | 5,991.06 | -8,629.67 | 1.44% |
| South | Binders | 12,305.16 | -8,404.51 | 0.68% |
| South | Machines | 6,196.34 | -7,635.24 | 1.23% |
| East | Phones | 19,030.53 | -6,715.83 | 0.35% |
| Central | Tables | 3,008.69 | -6,568.37 | 2.18% |
| East | Binders | 17,239.58 | -5,971.66 | 0.35% |

</details>

Leading the biggest net loss the Central Binders, Followed by East Machines and East Tables (which was already discussed above)
