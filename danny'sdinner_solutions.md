**Schema (PostgreSQL v13)**

    CREATE SCHEMA dannys_diner;
    SET search_path = dannys_diner;
    
    CREATE TABLE sales (
      "customer_id" VARCHAR(1),
      "order_date" DATE,
      "product_id" INTEGER
    );
    
    INSERT INTO sales
      ("customer_id", "order_date", "product_id")
    VALUES
      ('A', '2021-01-01', '1'),
      ('A', '2021-01-01', '2'),
      ('A', '2021-01-07', '2'),
      ('A', '2021-01-10', '3'),
      ('A', '2021-01-11', '3'),
      ('A', '2021-01-11', '3'),
      ('B', '2021-01-01', '2'),
      ('B', '2021-01-02', '2'),
      ('B', '2021-01-04', '1'),
      ('B', '2021-01-11', '1'),
      ('B', '2021-01-16', '3'),
      ('B', '2021-02-01', '3'),
      ('C', '2021-01-01', '3'),
      ('C', '2021-01-01', '3'),
      ('C', '2021-01-07', '3');
     
    
    CREATE TABLE menu (
      "product_id" INTEGER,
      "product_name" VARCHAR(5),
      "price" INTEGER
    );
    
    INSERT INTO menu
      ("product_id", "product_name", "price")
    VALUES
      ('1', 'sushi', '10'),
      ('2', 'curry', '15'),
      ('3', 'ramen', '12');
      
    
    CREATE TABLE members (
      "customer_id" VARCHAR(1),
      "join_date" DATE
    );
    
    INSERT INTO members
      ("customer_id", "join_date")
    VALUES
      ('A', '2021-01-07'),
      ('B', '2021-01-09');

---

**Query #1**

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

| customer_id | total_price |
| ----------- | ----------- |
| A           | 76          |
| B           | 74          |
| C           | 36          |

---
**Query #2**

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

| customer_id | visit_days |
| ----------- | ---------- |
| A           | 4          |
| B           | 6          |
| C           | 2          |

---
**Query #3**

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

| customer_id | product_name |
| ----------- | ------------ |
| A           | sushi        |
| B           | curry        |
| C           | ramen        |

---
**Query #4**

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

| product_id | product_name | count_item |
| ---------- | ------------ | ---------- |
| 3          | ramen        | 8          |

---
**Query #5**

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

| customer_id | product_id | product_name | count_item | item_rank |
| ----------- | ---------- | ------------ | ---------- | --------- |
| A           | 3          | ramen        | 3          | 1         |
| B           | 1          | sushi        | 2          | 1         |
| B           | 2          | curry        | 2          | 1         |
| B           | 3          | ramen        | 2          | 1         |
| C           | 3          | ramen        | 3          | 1         |

---
**Query #6**

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

| customer_id | product_name | order_date               | join_date                | item_purchase_number |
| ----------- | ------------ | ------------------------ | ------------------------ | -------------------- |
| A           | curry        | 2021-01-07T00:00:00.000Z | 2021-01-07T00:00:00.000Z | 1                    |
| B           | sushi        | 2021-01-11T00:00:00.000Z | 2021-01-09T00:00:00.000Z | 1                    |

---
**Query #7**

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

| customer_id | product_name | order_date               | join_date                | item_purchase_number |
| ----------- | ------------ | ------------------------ | ------------------------ | -------------------- |
| A           | sushi        | 2021-01-01T00:00:00.000Z | 2021-01-07T00:00:00.000Z | 1                    |
| B           | sushi        | 2021-01-04T00:00:00.000Z | 2021-01-09T00:00:00.000Z | 1                    |

---
**Query #8**

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

| customer_id | count_products | total_price |
| ----------- | -------------- | ----------- |
| A           | 2              | 25          |
| B           | 3              | 40          |

---
**Query #9**

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

| customer_id | product_name | total_points |
| ----------- | ------------ | ------------ |
| B           | sushi        | 400          |
| A           | ramen        | 360          |
| A           | curry        | 300          |
| B           | curry        | 300          |
| B           | ramen        | 240          |
| A           | sushi        | 200          |

---
**Query #10**

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

| customer_id | product_name | total_points |
| ----------- | ------------ | ------------ |
| A           | sushi        | 200          |
| A           | ramen        | 240          |
| B           | ramen        | 240          |
| B           | sushi        | 400          |
| B           | curry        | 450          |
| A           | curry        | 600          |

---

[View on DB Fiddle](https://www.db-fiddle.com/f/2rM8RAnq7h5LLDTzZiRWcd/138)
