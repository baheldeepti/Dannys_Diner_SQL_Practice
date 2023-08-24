# sql_practise
SQL Casestudies
/* --------------------
   Case Study Questions
   --------------------*/

-- 1. What is the total amount each customer spent at the restaurant?
SELECT
a.customer_id,
sum(price) as total_price
from  dannys_diner.sales a
left join  dannys_diner.menu b
on a.product_id = b.product_id
left join  dannys_diner.members c
on c.customer_id = a.customer_id
group by 1
order by 1,2;
-- 2. How many days has each customer visited the restaurant?
SELECT
a.customer_id,
count(distinct order_date) as visit_days
from  dannys_diner.sales a
left join  dannys_diner.menu b
on a.product_id = b.product_id
left join  dannys_diner.members c
on c.customer_id = a.customer_id
group by 1
order by 1,2;

-- 3. What was the first item from the menu purchased by each customer?
with cte as (
SELECT
a.customer_id,
a.product_id,
b.product_name,
a.order_date,
row_number() over (partition by a.customer_id order by order_date asc) as transaction_number
from  dannys_diner.sales a
left join  dannys_diner.menu b
on a.product_id = b.product_id
left join  dannys_diner.members c
on c.customer_id = a.customer_id
group by 1,2,3,4
order by 4)
select 
customer_id,
product_name
from cte
where transaction_number = 1
order by 1,2;
-- 4. What is the most purchased item on the menu and how many times was it purchased by all customers?
SELECT
a.product_id,
b.product_name,
count(a.product_id) as count_item
from  dannys_diner.sales a
left join  dannys_diner.menu b
on a.product_id = b.product_id
left join  dannys_diner.members c
on c.customer_id = a.customer_id
group by 1,2 
order by count(a.product_id) desc limit 1;

-- 5. Which item was the most popular for each customer?
with cte as (
SELECT
a.customer_id,
a.product_id,
b.product_name,
count(a.product_id) as count_item
from  dannys_diner.sales a
left join  dannys_diner.menu b
on a.product_id = b.product_id
left join  dannys_diner.members c
on c.customer_id = a.customer_id
group by 1,2,3),
final as 
(select
 customer_id,
 product_id,
 product_name,
 count_item,
 dense_rank() over (partition by customer_id order by count_item desc) as item_rank
 from cte
)
 select
 customer_id,
 product_id,
 product_name,
 count_item,
 item_rank
 from final
 where item_rank = 1
  group by 1,2,3,4,5
 ;
 
-- 6. Which item was purchased first by the customer after they became a member?

with cte as (SELECT
a.customer_id,
a.product_id,
b.product_name,
a.order_date,
c.join_date
from  dannys_diner.sales a
left join  dannys_diner.menu b
on a.product_id = b.product_id
right join  dannys_diner.members c
on c.customer_id = a.customer_id),
final as(
select
customer_id,
product_name,
order_date,
join_date,
  row_number() over (partition by customer_id order by order_date asc) as item_purchase_number
from cte where join_date <= order_date)
select
customer_id,
product_name,
order_date,
join_date,
item_purchase_number
from final
where item_purchase_number = 1;

-- 7. Which item was purchased just before the customer became a member?
with cte as (SELECT
a.customer_id,
a.product_id,
b.product_name,
a.order_date,
c.join_date
from  dannys_diner.sales a
left join  dannys_diner.menu b
on a.product_id = b.product_id
right join  dannys_diner.members c
on c.customer_id = a.customer_id),
final as(
select
customer_id,
product_name,
order_date,
join_date,
  row_number() over (partition by customer_id order by order_date desc) as item_purchase_number
from cte where join_date > order_date)
select
customer_id,
product_name,
order_date,
join_date,
item_purchase_number
from final
where item_purchase_number = 1;

-- 8. What is the total items and amount spent for each member before they became a member?
with cte as (SELECT
a.customer_id,
a.product_id,
b.product_name,
b.price,
a.order_date,
c.join_date
from  dannys_diner.sales a
left join  dannys_diner.menu b
on a.product_id = b.product_id
right join  dannys_diner.members c
on c.customer_id = a.customer_id)
select
customer_id,
count(product_id) as count_products,
sum(price) as total_price
from cte 
where join_date > order_date
GROUP BY 1
ORDER By 2;
-- 9.  If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
with cte as (SELECT
a.customer_id,
a.product_id,
b.product_name,
b.price,
a.order_date,
c.join_date
from  dannys_diner.sales a
left join  dannys_diner.menu b
on a.product_id = b.product_id
right join  dannys_diner.members c
on c.customer_id = a.customer_id)
select
customer_id,
product_name,
case when product_name like 'sushi' then sum(price)*2*10
else sum(price)*10 end as total_points
from cte 
GROUP BY 1,2
ORDER By 3 desc;
-- 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
with cte as (SELECT
a.customer_id,
a.product_id,
b.product_name,
b.price,
a.order_date,
c.join_date
from  dannys_diner.sales a
left join  dannys_diner.menu b
on a.product_id = b.product_id
right join  dannys_diner.members c
on c.customer_id = a.customer_id),
p as (
select
customer_id,
product_name,
price,
join_date,
order_date,
(join_date - order_date) as differenceindays
from cte 
GROUP BY 1,2,3,4,5
ORDER By 3 desc),
final as(
select
customer_id,
product_name,
(case 
when (product_name like 'sushi') or 
(differenceindays >=0 and differenceindays <=7)
then sum(price)*2*10
else sum(price)*10 end) as total_points
from p
GROUP BY 1,2,differenceindays)
select
customer_id,
product_name,
sum(total_points) as total_points
from final
group by 1,2
order by 3;
