# SQL Mastery Guide
### From "I've studied it before" to "I can answer any interview question"

---

## How This Guide Works

You've studied SQL multiple times and still don't feel confident. That happens for one reason: **you've been learning syntax, not thinking in SQL.** This guide fixes that. Every section has three parts:

1. **The concept explained in plain English** (not jargon)
2. **The syntax** (short and clean)
3. **A real practice problem** using the Olist dataset you already have

Work through one section per day. Write every query by hand — do not copy-paste. The act of typing it yourself is what builds confidence.

---

## Module 1 — Foundations (Days 1–3)

### Day 1: SELECT, WHERE, ORDER BY, LIMIT

**The concept:**
SELECT is how you ask a question. WHERE is how you filter the answer. Think of it like this: SELECT is "show me", WHERE is "but only if".

```sql
-- Show me all delivered orders from São Paulo, most recent first
SELECT 
    order_id,
    order_status,
    order_purchase_timestamp
FROM orders
WHERE order_status = 'delivered'
  AND customer_id IN (
      SELECT customer_id FROM customers WHERE customer_state = 'SP'
  )
ORDER BY order_purchase_timestamp DESC
LIMIT 20;
```

**Rules you must know:**
- SELECT always comes first
- WHERE filters BEFORE any grouping happens
- ORDER BY ASC = smallest to largest, DESC = largest to smallest
- LIMIT always goes last

**Your practice problem:**
Write a query that shows the 5 most expensive single items ever ordered from the order_items table. Show: order_id, product_id, price. Order by price descending.

**Answer:**
```sql
SELECT order_id, product_id, price
FROM order_items
ORDER BY price DESC
LIMIT 5;
```

---

### Day 2: Aggregate Functions — COUNT, SUM, AVG, MIN, MAX

**The concept:**
Aggregates collapse many rows into one number. You always use them with GROUP BY (unless you want one number for the entire table).

**The rule that trips everyone up:**
> Every column in your SELECT that is NOT an aggregate MUST be in your GROUP BY.

```sql
-- How many orders per status?
SELECT 
    order_status,
    COUNT(*) AS total_orders
FROM orders
GROUP BY order_status
ORDER BY total_orders DESC;
```

```sql
-- Total and average revenue per payment type
SELECT 
    payment_type,
    COUNT(*) AS total_transactions,
    ROUND(SUM(payment_value), 2) AS total_revenue,
    ROUND(AVG(payment_value), 2) AS avg_order_value
FROM order_payments
GROUP BY payment_type;
```

**Your practice problem:**
Write a query that shows the average review score per order status. Which status has the lowest average score?

**Answer:**
```sql
SELECT 
    o.order_status,
    ROUND(AVG(r.review_score), 2) AS avg_score,
    COUNT(*) AS total_reviews
FROM orders o
JOIN order_reviews r ON o.order_id = r.order_id
GROUP BY o.order_status
ORDER BY avg_score ASC;
```

---

### Day 3: HAVING vs WHERE — The Most Common Interview Mistake

**The concept:**
- WHERE filters rows BEFORE grouping
- HAVING filters groups AFTER grouping

Think of it this way: WHERE talks to individual rows. HAVING talks to groups.

```sql
-- WRONG: This will error
SELECT customer_state, COUNT(*) AS total
FROM customers
WHERE COUNT(*) > 1000   -- ❌ Cannot use aggregate in WHERE
GROUP BY customer_state;

-- CORRECT
SELECT customer_state, COUNT(*) AS total
FROM customers
GROUP BY customer_state
HAVING COUNT(*) > 1000;  -- ✅ Filter after grouping
```

```sql
-- Real example: Find sellers who have sold more than 100 items
SELECT 
    seller_id,
    COUNT(*) AS items_sold,
    ROUND(SUM(price), 2) AS total_revenue
FROM order_items
GROUP BY seller_id
HAVING COUNT(*) > 100
ORDER BY total_revenue DESC;
```

**Your practice problem:**
Find all product categories that have an average review score below 3.5 and have at least 50 reviews. Show category name (in English), avg score, and review count.

**Answer:**
```sql
SELECT 
    t.product_category_name_english,
    ROUND(AVG(r.review_score), 2) AS avg_score,
    COUNT(*) AS review_count
FROM order_reviews r
JOIN orders o ON r.order_id = o.order_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
JOIN product_category_translation t 
    ON p.product_category_name = t.product_category_name
GROUP BY t.product_category_name_english
HAVING AVG(r.review_score) < 3.5 AND COUNT(*) >= 50
ORDER BY avg_score ASC;
```

---

## Module 2 — Joins (Days 4–6)

### Day 4: INNER JOIN — Only matching rows

**The concept:**
INNER JOIN returns rows that have a match in BOTH tables. If a row exists in table A but not table B — it disappears.

```sql
-- Orders with their customer state (only orders that have a matching customer)
SELECT 
    o.order_id,
    o.order_status,
    c.customer_state,
    c.customer_city
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_status = 'delivered'
LIMIT 10;
```

**When to use it:** When you only care about records that exist in both tables. Most common join you'll use.

**Your practice problem:**
Write a query showing each order's total payment value, alongside the customer's state. Use INNER JOIN between orders, customers, and order_payments.

---

### Day 5: LEFT JOIN — Keep everything from the left table

**The concept:**
LEFT JOIN keeps ALL rows from the left table, even if there's no match in the right table. Where there's no match, you get NULL.

This is crucial for finding gaps in data.

```sql
-- Find orders that have NO review (left join reveals missing matches)
SELECT 
    o.order_id,
    o.order_status,
    r.review_score
FROM orders o
LEFT JOIN order_reviews r ON o.order_id = r.order_id
WHERE r.review_id IS NULL;  -- NULL means no match was found
```

```sql
-- Find products that have never been ordered
SELECT 
    p.product_id,
    p.product_category_name
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
WHERE oi.order_id IS NULL;
```

**The pattern to memorise:**
> LEFT JOIN + WHERE right_table.id IS NULL = "find everything in A with no match in B"

**Your practice problem:**
Find all orders that were delivered but have no payment record. What does this tell you about the data quality?

---

### Day 6: Multiple JOINs + Aliases

**The concept:**
Real queries join 3, 4, or 5 tables. The key is to work outward from your central table and give every table a short alias.

```sql
-- Full picture: order + customer + items + product category + review score
SELECT 
    o.order_id,
    c.customer_state,
    t.product_category_name_english AS category,
    ROUND(SUM(oi.price), 2) AS order_value,
    r.review_score
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
JOIN product_category_translation t 
    ON p.product_category_name = t.product_category_name
LEFT JOIN order_reviews r ON o.order_id = r.order_id
WHERE o.order_status = 'delivered'
GROUP BY o.order_id, c.customer_state, t.product_category_name_english, r.review_score
ORDER BY order_value DESC
LIMIT 20;
```

**How to build multi-join queries:**
1. Start with your main table (usually orders or the fact table)
2. Ask: what else do I need? Add one JOIN at a time
3. Use LEFT JOIN when the related table might not always have a match
4. Always alias your tables (o, c, oi, p, t, r)

**Your practice problem:**
Write a query showing the top 5 sellers by total revenue. Include: seller_id, seller_state, total items sold, total revenue. You'll need order_items and sellers.

---

## Module 3 — Subqueries & CTEs (Days 7–9)

### Day 7: Subqueries — A query inside a query

**The concept:**
A subquery is just a SELECT inside another SELECT, FROM, or WHERE. Use it when you need to calculate something first before using it.

```sql
-- Find customers who spent more than the average order value
SELECT 
    customer_unique_id,
    total_spent
FROM (
    SELECT 
        c.customer_unique_id,
        SUM(p.payment_value) AS total_spent
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    JOIN order_payments p ON o.order_id = p.order_id
    GROUP BY c.customer_unique_id
) AS customer_totals
WHERE total_spent > (SELECT AVG(payment_value) FROM order_payments)
ORDER BY total_spent DESC;
```

**Three places to use subqueries:**
1. In FROM — treat the result as a temporary table (like above)
2. In WHERE — filter based on a calculated value
3. In SELECT — calculate a value per row (use sparingly, can be slow)

---

### Day 8: CTEs — The cleaner way to write complex queries

**The concept:**
A CTE (Common Table Expression) does the same thing as a subquery but is much easier to read. You define it at the top with WITH, then use it like a table.

```sql
-- Same query as Day 7 but with a CTE — much cleaner
WITH customer_totals AS (
    SELECT 
        c.customer_unique_id,
        SUM(p.payment_value) AS total_spent
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    JOIN order_payments p ON o.order_id = p.order_id
    GROUP BY c.customer_unique_id
),
avg_spend AS (
    SELECT AVG(payment_value) AS avg_value
    FROM order_payments
)
SELECT 
    ct.customer_unique_id,
    ct.total_spent
FROM customer_totals ct, avg_spend a
WHERE ct.total_spent > a.avg_value
ORDER BY ct.total_spent DESC;
```

**Rule of thumb:**
- If your query has more than 2 levels of nesting → use a CTE
- CTEs make your logic readable and debuggable
- You can define multiple CTEs by separating them with commas

**Your practice problem:**
Using a CTE, find the top 3 product categories by average review score, but only include categories with at least 100 reviews.

---

### Day 9: Window Functions — The most powerful SQL tool

**The concept:**
Window functions calculate a value across a set of rows related to the current row — without collapsing them like GROUP BY does. They use OVER().

```sql
-- Rank sellers by revenue within each state
SELECT 
    s.seller_state,
    s.seller_id,
    ROUND(SUM(oi.price), 2) AS revenue,
    RANK() OVER (
        PARTITION BY s.seller_state 
        ORDER BY SUM(oi.price) DESC
    ) AS rank_in_state
FROM order_items oi
JOIN sellers s ON oi.seller_id = s.seller_id
GROUP BY s.seller_state, s.seller_id
ORDER BY s.seller_state, rank_in_state;
```

**The four window functions you must know:**

| Function | What it does |
|----------|-------------|
| ROW_NUMBER() | Unique number per row (1,2,3,4...) |
| RANK() | Rank with gaps for ties (1,2,2,4...) |
| DENSE_RANK() | Rank without gaps (1,2,2,3...) |
| LAG() / LEAD() | Get the previous / next row's value |

```sql
-- Month-over-month order growth using LAG
WITH monthly_orders AS (
    SELECT 
        DATE_FORMAT(order_purchase_timestamp, '%Y-%m') AS month,
        COUNT(*) AS total_orders
    FROM orders
    GROUP BY DATE_FORMAT(order_purchase_timestamp, '%Y-%m')
)
SELECT 
    month,
    total_orders,
    LAG(total_orders) OVER (ORDER BY month) AS prev_month_orders,
    total_orders - LAG(total_orders) OVER (ORDER BY month) AS growth
FROM monthly_orders
ORDER BY month;
```

---

## Module 4 — Interview Preparation (Days 10–12)

### Day 10: CASE WHEN — Conditional logic in SQL

```sql
-- Categorise orders by delivery performance
SELECT 
    order_id,
    DATEDIFF(order_delivered_customer_date, order_estimated_delivery_date) AS days_diff,
    CASE 
        WHEN order_delivered_customer_date <= order_estimated_delivery_date 
            THEN 'On Time'
        WHEN DATEDIFF(order_delivered_customer_date, 
                      order_estimated_delivery_date) <= 3 
            THEN 'Slightly Late'
        ELSE 'Significantly Late'
    END AS delivery_status
FROM orders
WHERE order_status = 'delivered'
  AND order_delivered_customer_date IS NOT NULL;
```

```sql
-- Pivot-style: count orders by status per state (top 5 states)
SELECT 
    c.customer_state,
    SUM(CASE WHEN o.order_status = 'delivered' THEN 1 ELSE 0 END) AS delivered,
    SUM(CASE WHEN o.order_status = 'cancelled' THEN 1 ELSE 0 END) AS cancelled,
    SUM(CASE WHEN o.order_status = 'shipped' THEN 1 ELSE 0 END) AS shipped
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
GROUP BY c.customer_state
ORDER BY delivered DESC
LIMIT 5;
```

---

### Day 11: The 10 Most Common SQL Interview Questions

**Q1: What is the difference between WHERE and HAVING?**
> WHERE filters rows before grouping. HAVING filters after grouping. Use WHERE for row-level conditions, HAVING for aggregate conditions.

**Q2: What is the difference between INNER JOIN and LEFT JOIN?**
> INNER JOIN returns only matching rows from both tables. LEFT JOIN returns all rows from the left table, with NULLs where no match exists in the right table.

**Q3: What is a subquery?**
> A query nested inside another query. Used in FROM (as a derived table), WHERE (to filter based on a calculated value), or SELECT (to compute a per-row value).

**Q4: What is a CTE and when would you use it?**
> A Common Table Expression defined with WITH. Used to break complex queries into readable, named steps. Preferable to nested subqueries when logic has multiple levels.

**Q5: What is the difference between RANK() and DENSE_RANK()?**
> RANK() leaves gaps after ties (1,2,2,4). DENSE_RANK() does not (1,2,2,3).

**Q6: How do you find duplicates in a table?**
```sql
SELECT column_name, COUNT(*) 
FROM table_name 
GROUP BY column_name 
HAVING COUNT(*) > 1;
```

**Q7: How do you find the second highest value?**
```sql
SELECT MAX(price) 
FROM order_items 
WHERE price < (SELECT MAX(price) FROM order_items);
-- Or with DENSE_RANK:
SELECT price FROM (
    SELECT price, DENSE_RANK() OVER (ORDER BY price DESC) AS rnk
    FROM order_items
) ranked WHERE rnk = 2 LIMIT 1;
```

**Q8: What is the difference between DELETE, TRUNCATE, and DROP?**
> DELETE removes specific rows (can be rolled back, WHERE clause applies). TRUNCATE removes all rows fast (cannot filter). DROP removes the entire table including its structure.

**Q9: What is normalisation?**
> The process of organising data to reduce redundancy. First normal form (1NF): no repeating groups. Second (2NF): no partial dependencies. Third (3NF): no transitive dependencies. Your Olist project is a good example of a normalised schema.

**Q10: How would you optimise a slow query?**
> Check if indexes exist on JOIN and WHERE columns. Avoid SELECT *. Filter early with WHERE before grouping. Use EXPLAIN to see the query execution plan. Avoid functions on indexed columns in WHERE clauses.

---

### Day 12: Practice Interview — Answer These Out Loud

Sit down, open MySQL Workbench, and answer each of these as if you are in an interview. No notes.

1. Write a query to find the top 3 customers by total spend, using customer_unique_id.
2. Find all orders where the actual delivery date was more than 7 days after the estimated date.
3. Write a query using a window function to rank products by revenue within each category.
4. Find the month with the highest number of orders in the dataset.
5. Write a query using a CTE to identify sellers who have an average review score below 3.0 with at least 20 reviews.

**After each query, ask yourself:**
- Can I explain what every line does?
- Can I explain why I used JOIN vs LEFT JOIN here?
- Can I explain what would happen if I removed the HAVING clause?

If you can answer those three questions for every query you write — you are interview-ready.

---

## Study Schedule (12 Days)

| Day | Topic | Time |
|-----|-------|------|
| 1 | SELECT, WHERE, ORDER BY, LIMIT | 45 min |
| 2 | COUNT, SUM, AVG, GROUP BY | 45 min |
| 3 | HAVING vs WHERE | 45 min |
| 4 | INNER JOIN | 45 min |
| 5 | LEFT JOIN + finding gaps | 45 min |
| 6 | Multiple JOINs + aliases | 1 hour |
| 7 | Subqueries | 1 hour |
| 8 | CTEs | 1 hour |
| 9 | Window functions | 1 hour |
| 10 | CASE WHEN | 45 min |
| 11 | Interview Q&A — read and memorise | 45 min |
| 12 | Full mock interview — answer out loud | 1 hour |

**Total: ~11 hours over 12 days**

---

## The Most Important Rule

> **Type every query yourself. Do not copy-paste.**
>
> The goal is not to have a file of correct queries.
> The goal is to be able to write them from memory under pressure.
> That only comes from typing them, breaking them, fixing them, and understanding why they broke.

