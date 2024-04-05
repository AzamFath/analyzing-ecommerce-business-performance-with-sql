# Analyzing eCommerce Business Performance with SQL
E-Commerce Data Analysis using SQL

## Background
The rapid growth of the e-commerce industry is attracting more people to shop online, driven by convenience, accessibility, and a variety of platform choices. Fierce competition among large platforms and new players pushes companies to gain market share by optimizing services, prices, and user experiences. Business performance analysis is key to identifying trends, understanding customer behavior, and making better strategic decisions in an increasingly competitive landscape. Data-driven approaches leverage information from sales transactions, customer data, and product data to understand customer behavior, optimize marketing strategies, and optimize preferred payment methods. Therefore, in this project, I will attempt to analyze the business performance of an e-commerce company using SQL.

## Objectives
This project will analyze business performance and find out information that can be obtained from an e-commerce company to provide an overview of business performance and provide recommendations for the e-commerce company.

## Research Questions
1. How does the growth of new customers compare to returning customers annually, and what is the monthly count of active customers each year?
2. What are the most purchased products each year, what are the most canceled products each year, and what is the percentage of these compared to the total?
3. What are the most commonly used payment methods by customers?

## Data and Assumptions
The dataset used can be downloaded from [dataset source](https://drive.google.com/file/d/1cdC9nzrjddbOleK9aPZK7ZjCUTJ3uG7U/view?usp=sharing). The assumption in this study is that the processed data belongs to one of the e-commerce companies in the world, although I cannot confirm whether the data is valid or not because here I ONLY process the dataset.

## Data Analysis
In this project, I use SQL to analyze the data. The file `SQL Query.docx` presents detailed analysis steps. However, here I will only display the results of data analysis according to the research questions that have been created.


1. Annual Customer Activity Growth

    A. Average Monthly Active Users
		```sql
		select 
			order_year,
			round(avg(MAU),2) as average_mau
		from	
			(select 
		date_part('year', o.order_purchase_timestamp) as order_year,
		date_part('month', o.order_purchase_timestamp) as order_monnth,
				count(distinct customer_unique_id) as MAU
			from orders_dataset o
			join customer_dataset c on o.customer_id = c.customer_id
			group by 1,2) subq
		group by 1
		```

    B. New Customers
		```sql
		select 
			date_part('year', order_year) as order_year,
			count(1) as new_customer
		from	
			(select 
				c.customer_id,
				min(o.order_purchase_timestamp) as order_year
			from orders_dataset o
			join customer_dataset c on o.customer_id = c.customer_id
			group by 1) subq
		group by 1
		```

    C. Repeat Customers
		```sql
		select
			order_year,
			count(distinct customer_unique_id) as repeating_customers
		from
			(select 
		date_part('year', o.order_purchase_timestamp) as order_year,
				c.customer_unique_id,
				count(1) as purchase_frequency
			from orders_dataset o
			join customer_dataset c on o.customer_id = c.customer_id
			group by 1,2
			having count(1)>1) subq
		group by 1
		```

    D. Average Order Frequency
		```sql
		select
			order_year,
			round(avg(purchase_frequency),3) as avg_purchase_frequency
		from
			(select 
		date_part('year', o.order_purchase_timestamp) as order_year,
				c.customer_unique_id,
				count(1) as purchase_frequency
			from orders_dataset o
			join customer_dataset c on o.customer_id = c.customer_id
			group by 1,2) subq
		group by 1
		```

    E. Combining the Four Metrics
		```sql
		with 

		MAU as(
		select 
			order_year,
			round(avg(MAU),2) as average_mau
		from	
			(select 
		date_part('year', o.order_purchase_timestamp) as order_year,
		date_part('month', o.order_purchase_timestamp) as order_monnth,
				count(distinct customer_unique_id) as MAU
			from orders_dataset o
			join customer_dataset c on o.customer_id = c.customer_id
			group by 1,2) subq
		group by 1),

		new_customer as(
		select 
			date_part('year', order_year) as order_year,
			count(1) as new_customer
		from	
			(select 
				c.customer_id,
				min(o.order_purchase_timestamp) as order_year
			from orders_dataset o
			join customer_dataset c on o.customer_id = c.customer_id
			group by 1) subq
		group by 1),

		repeat_customer as(
		select
			order_year,
			count(distinct customer_unique_id) as repeating_customers
		from
			(select 
		date_part('year', o.order_purchase_timestamp) as order_year,
				c.customer_unique_id,
				count(1) as purchase_frequency
			from orders_dataset o
			join customer_dataset c on o.customer_id = c.customer_id
			group by 1,2
			having count(1)>1) subq
		group by 1),

		avg_purchase_freq as(
		select
			order_year,
			round(avg(purchase_frequency),3) as avg_purchase_frequency
		from
			(select 
		date_part('year', o.order_purchase_timestamp) as order_year,
				c.customer_unique_id,
				count(1) as purchase_frequency
			from orders_dataset o
			join customer_dataset c on o.customer_id = c.customer_id
			group by 1,2) subq
		group by 1)

		select
			mau.order_year,
			mau.average_mau,
			nc.new_customer,
			rc.repeating_customers,
			apf.avg_purchase_frequency
		from MAU mau
		join new_customer nc on mau.order_year = nc.order_year
		join repeat_customer rc on rc.order_year = mau.order_year
		join avg_purchase_freq apf on apf.order_year = mau.order_year
		```

![Data Results](https://github.com/AzamFath/predict-customer-clicked-ads-classification/blob/main/fig%201.png)

2. Product Category Quality Annually

    A. Revenue per Year
		```sql
		create table revenue_per_year as
		select
			date_part('year', order_purchase_timestamp) as year,
			sum(revenue_per_order) as revenue
		from
			(select 
				order_id, 
				sum(price+freight_value) as revenue_per_order
			from order_items_dataset
			group by 1) subq
		join orders_dataset o on subq.order_id = o.order_id
		where o.order_status = 'delivered'
		group by 1
		```

   B. Number of Canceled Orders per Year
		```sql
		create table cancel_per_year as
		select 
			date_part('year', order_purchase_timestamp) as year,
			count(1) as num_canceled_booking
		from orders_dataset
		where order_status = 'canceled'
		group by 1
		```

	C. Categories Generating the Highest Revenue per Year
		```sql
		create table top_product_revenue_per_year as
		select	
			year,
			product_category_name,
			revenue
		from
			(select 
				date_part('year', o.order_purchase_timestamp) as year,
				p.product_category_name,
				sum(price+freight_value) as revenue,
		rank() over(partition by date_part('year', o.order_purchase_timestamp)
				order by sum(price+freight_value) desc)
			from order_items_dataset oi
			join product_dataset p on oi.product_id = p.product_id
			join orders_dataset o on o.order_id = oi.order_id
			where o.order_status = 'delivered'
			group by 1,2) subq
		where rank = 1
		```

	D. Categories Experiencing the Largest Number of Canceled Orders per Year
		```sql
		create table top_cancel_product_per_year as
		select	
			year,
			product_category_name,
			num_canceled
		from
			(select 
				date_part('year', o.order_purchase_timestamp) as year,
				p.product_category_name,
				count(2) as num_canceled,
		rank() over(partition by date_part('year', o.order_purchase_timestamp)
				order by count(1) desc)
			from order_items_dataset oi
			join product_dataset p on oi.product_id = p.product_id
			join orders_dataset o on o.order_id = oi.order_id
			where o.order_status = 'canceled'
			group by 1,2) subq
		where rank = 1
		```

	E. Combining the Four Tables
		```sql
		select 
			cpy.year,
			cpy.num_canceled_booking,
			rpy.revenue,
			tcp.product_category_name top_product_by_canceled,
			tcp.num_canceled,
			tpr.product_category_name  top_product_by_revenue,
			tpr.revenue
		from cancel_per_year cpy
		join revenue_per_year rpy on cpy.year = rpy.year
		join top_cancel_product_per_year tcp on tcp.year = cpy.year
		join top_product_revenue_per_year tpr on tpr.year = cpy.year
		```

![Data Results](https://github.com/AzamFath/predict-customer-clicked-ads-classification/blob/main/fig%201.png)

3. Annual Payment Method Usage

	A. Total Number of Each Payment Type Usage
		```sql
		select 
				payment_type,
				count(1) as num_payment
		from order_payments_dataset
		group by 1
		order by 2 desc
		```

	B. Number of Usage for Each Payment Type
		```sql
		select 
				payment_type,
		sum(case when year = '2016' then num_used else 0 end) as year_2016,
		sum(case when year = '2017' then num_used else 0 end) as year_2017,
		sum(case when year = '2018' then num_used else 0 end) as year_2018
		from
				(select 
					date_part('year', o.order_purchase_timestamp) as year,
					op.payment_type,
					count(2) as num_used
				from order_payments_dataset op 
				join orders_dataset o on o.order_id = op.order_id
				group by 1, 2) subq
		group by 1
		```

![Data Results](https://github.com/AzamFath/predict-customer-clicked-ads-classification/blob/main/fig%201.png)

## Conclusion
The analysis results indicate that:
1. The growth of new customers compared to returning customers annually, 

![Data Results](https://github.com/AzamFath/predict-customer-clicked-ads-classification/blob/main/fig%201.png)

and the monthly count of active customers each year.

![Data Results](https://github.com/AzamFath/predict-customer-clicked-ads-classification/blob/main/fig%201.png)

2. Based on the analysis results, the most purchased product each year is [Product A], 

![Data Results](https://github.com/AzamFath/predict-customer-clicked-ads-classification/blob/main/fig%201.png)

and the most canceled product each year is [Product B]. 

![Data Results](https://github.com/AzamFath/predict-customer-clicked-ads-classification/blob/main/fig%201.png)

Meanwhile, the percentage of the total for the most purchased product is [percentage]%, 

![Data Results](https://github.com/AzamFath/predict-customer-clicked-ads-classification/blob/main/fig%201.png)

and for the most canceled product is [percentage]%.

![Data Results](https://github.com/AzamFath/predict-customer-clicked-ads-classification/blob/main/fig%201.png)

3. Based on the previous analysis results, the payment method most used by customers is Credit Card.

![Data Results](https://github.com/AzamFath/predict-customer-clicked-ads-classification/blob/main/fig%201.png)

## Recommendations
Based on the analysis results, several recommendations that can be considered by the business owner are:

1. **Maintain Monthly Active Customers**
   Keep maintaining the number of monthly active customers as there is an increase every year. This could be achieved by running attractive campaigns or promotions to ensure that monthly active customers are retained and stay on a positive trend.

2. **Further Analysis**
   For further research, it may be worth investigating why in 2018, the product that generated the most revenue, Health Beauty, was also the most canceled product. Is it because there are many manufacturers in the same category or for other reasons?

3. **Lower Interest Rates**
   Since the most commonly used payment method by customers is credit card, reducing interest rates on credit card payments could be beneficial. Additionally, offering incentives such as free shipping promotions can further encourage customers to use credit cards. 
