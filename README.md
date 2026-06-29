# E-Commerce Platform — CTEs & SQL Window Functions

**Course:** C11665 - DPR400210: Database Programming  
**Instructor:** Eric Maniraguha  
**Student:** Mpano Eugene  
**GitHub:** [ZyroIV](https://github.com/ZyroIV)  
**University:** UNILAK, Kigali, Rwanda

---

## Business Problem

Retail businesses operating online struggle to gain meaningful insights from raw transaction data. This project models an **E-Commerce Platform** that manages customers, products, orders, and order details — and uses advanced SQL techniques (CTEs and Window Functions) to extract actionable business intelligence such as identifying high-value customers, tracking revenue trends, and ranking top-performing products.

---

## Database Schema

The system is built on **4 relational tables** in Oracle SQL:

| Table | Description |
|---|---|
| `Customers` | Stores customer identity and location |
| `Products` | Product catalog with pricing and stock |
| `Orders` | Order records linked to customers |
| `Order_Details` | Line items per order (product, quantity, price) |

### Table Definitions

```sql
CREATE TABLE Customers (
    customer_id   NUMBER PRIMARY KEY,
    customer_name VARCHAR2(100) NOT NULL,
    phone         VARCHAR2(15),
    city          VARCHAR2(50)
);

CREATE TABLE Products (
    product_id   NUMBER PRIMARY KEY,
    product_name VARCHAR2(100) NOT NULL,
    category     VARCHAR2(50),
    price        NUMBER(10,2),
    stock        NUMBER
);

CREATE TABLE Orders (
    order_id    NUMBER PRIMARY KEY,
    customer_id NUMBER,
    order_date  DATE,
    CONSTRAINT fk_customer FOREIGN KEY (customer_id) REFERENCES Customers(customer_id)
);

CREATE TABLE Order_Details (
    detail_id  NUMBER PRIMARY KEY,
    order_id   NUMBER,
    product_id NUMBER,
    quantity   NUMBER,
    unit_price NUMBER(10,2),
    CONSTRAINT fk_order   FOREIGN KEY (order_id)   REFERENCES Orders(order_id),
    CONSTRAINT fk_product FOREIGN KEY (product_id) REFERENCES Products(product_id)
);
```

---



## CTE Implementations

### A1 — Simple CTE
**Goal:** Calculate total revenue per order.  
**Business Value:** Enables management to instantly see which orders generated the most revenue without repeating complex subqueries.

```sql
WITH OrderRevenue AS (
    SELECT order_id, SUM(quantity * unit_price) AS total_revenue
    FROM Order_Details
    GROUP BY order_id
)
SELECT o.order_id, c.customer_name, o.order_date, r.total_revenue
FROM OrderRevenue r
JOIN Orders o    ON r.order_id    = o.order_id
JOIN Customers c ON o.customer_id = c.customer_id
ORDER BY r.total_revenue DESC;
```

📸 *Screenshot: `screenshots/a1_simple_cte.png`*

---

### A2 — Multiple CTEs
**Goal:** Identify customers whose total spending exceeds the platform average.  
**Business Value:** Pinpoints high-value customers for loyalty programs and premium service targeting.

```sql
WITH OrderRevenue AS (
    SELECT order_id, SUM(quantity * unit_price) AS total_revenue
    FROM Order_Details GROUP BY order_id
),
CustomerSpend AS (
    SELECT o.customer_id, SUM(r.total_revenue) AS total_spend
    FROM OrderRevenue r JOIN Orders o ON r.order_id = o.order_id
    GROUP BY o.customer_id
),
AvgSpend AS (
    SELECT AVG(total_spend) AS avg_spend FROM CustomerSpend
)
SELECT c.customer_name, cs.total_spend, ROUND(a.avg_spend,2) AS avg_spend
FROM CustomerSpend cs
JOIN Customers c ON cs.customer_id = c.customer_id
CROSS JOIN AvgSpend a
WHERE cs.total_spend > a.avg_spend
ORDER BY cs.total_spend DESC;
```

📸 *Screenshot: `screenshots/a2_multiple_ctes.png`*

---

### A3 — Recursive CTE
**Goal:** Generate a continuous date series from first to last order to detect zero-sales days.  
**Business Value:** Gap analysis reveals slow trading periods for planning promotions or staffing.

```sql
WITH DateSeries (order_date, end_date) AS (
    SELECT MIN(order_date), MAX(order_date) FROM Orders
    UNION ALL
    SELECT order_date + 1, end_date FROM DateSeries
    WHERE order_date + 1 <= end_date
)
SELECT TO_CHAR(ds.order_date,'YYYY-MM-DD') AS sale_date,
       COUNT(o.order_id) AS orders_placed
FROM DateSeries ds
LEFT JOIN Orders o ON TRUNC(o.order_date) = TRUNC(ds.order_date)
GROUP BY ds.order_date
ORDER BY ds.order_date;
```

📸 *Screenshot: `screenshots/a3_recursive_cte.png`*

---

### A4 — CTE with Aggregation
**Goal:** Rank product categories by total revenue, including percentage contribution.  
**Business Value:** Guides marketing budget allocation and inventory investment toward the highest-earning categories.

```sql
WITH CategoryRevenue AS (
    SELECT p.category,
           SUM(od.quantity * od.unit_price) AS total_revenue,
           COUNT(DISTINCT od.order_id)       AS total_orders,
           SUM(od.quantity)                  AS units_sold
    FROM Order_Details od JOIN Products p ON od.product_id = p.product_id
    GROUP BY p.category
)
SELECT category, total_revenue, total_orders, units_sold,
       ROUND(total_revenue / SUM(total_revenue) OVER () * 100, 2) AS revenue_pct
FROM CategoryRevenue
ORDER BY total_revenue DESC;
```

📸 *Screenshot: `screenshots/a4_cte_aggregation.png`*

---

### A5 — CTE with JOINs
**Goal:** Find the top-selling product (by quantity) in each category.  
**Business Value:** Enables buyers to focus restocking efforts on the star product in every category.

```sql
WITH ProductSales AS (
    SELECT od.product_id, p.product_name, p.category,
           SUM(od.quantity) AS total_qty,
           SUM(od.quantity * od.unit_price) AS total_revenue
    FROM Order_Details od JOIN Products p ON od.product_id = p.product_id
    GROUP BY od.product_id, p.product_name, p.category
),
CategoryBest AS (
    SELECT category, MAX(total_qty) AS max_qty FROM ProductSales GROUP BY category
)
SELECT ps.category, ps.product_name AS top_product, ps.total_qty, ps.total_revenue
FROM ProductSales ps JOIN CategoryBest cb
  ON ps.category = cb.category AND ps.total_qty = cb.max_qty
ORDER BY ps.total_revenue DESC;
```

📸 *Screenshot: `screenshots/a5_cte_with_joins.png`*

---

## Window Function Implementations

### B1 — Ranking Functions
**Functions:** `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`, `PERCENT_RANK()`  
**Goal:** Rank customers by total spending to enable tiered segmentation.

📸 *Screenshot: `screenshots/b1_ranking_functions.png`*

---

### B2 — Aggregate Window Functions
**Functions:** `SUM() OVER()`, `AVG() OVER()`, `MIN() OVER()`, `MAX() OVER()`  
**Goal:** Show running totals and compare each order's value against the overall average.

📸 *Screenshot: `screenshots/b2_aggregate_window.png`*

---

### B3 — Navigation Functions
**Functions:** `LAG()`, `LEAD()`  
**Goal:** Compare each order's revenue to the previous and next order to detect growth trends.

📸 *Screenshot: `screenshots/b3_lag_lead.png`*

---

### B4 — Distribution Functions
**Functions:** `NTILE(4)`, `CUME_DIST()`  
**Goal:** Segment products into revenue quartiles and measure cumulative distribution.

📸 *Screenshot: `screenshots/b4_distribution.png`*

---

## Analysis and Findings

### Descriptive Analysis — What Happened?
- **Total orders:** 10 orders placed between June 1–20, 2026
- **Highest single order:** Orders 1001 and 1006 each reached **980,000 RWF** (Laptop + Mouse / Laptop only)
- **Top revenue category:** Electronics dominates with the majority of total revenue driven by Laptops and Monitors
- **Most active city:** Kigali had the highest concentration of customers (3 out of 10)

### Diagnostic Analysis — Why Did It Happen?
- Electronics generated the most revenue because high-unit-price items (Laptop at 950,000 RWF) were purchased multiple times across different orders
- Customers in Kigali ordered more frequently, likely reflecting urban purchasing power and better internet access for e-commerce
- The USB Flash Drive (order 1009: qty 10) had the highest quantity sold in a single order, suggesting bulk purchasing behavior for low-cost storage items
- Several days in June had zero orders (gaps visible in the recursive CTE output), indicating the platform does not yet have consistent daily traffic

### Prescriptive Analysis — What Should Be Done?
1. **Launch a VIP loyalty tier** for customers spending above the average (identified in A2) — offer free shipping or early access to new products
2. **Run flash sales on Electronics** during the zero-sales days identified by the date-gap query to smooth out revenue across the month
3. **Restock Accessories category products** (Mouse, Keyboard, Headphones) — they appear in many orders and have moderate stock levels
4. **Target Kigali customers** with location-specific promotions; then expand campaigns to other cities (Musanze, Huye) where purchasing has been lower
5. **Use NTILE quartile data** (B4) to consider discontinuing or repositioning bottom-quartile products like the Office Chair, which had low overall revenue despite a high unit price

---

## References

- Oracle Documentation — Common Table Expressions: https://docs.oracle.com/en/database/oracle/oracle-database/21/sqlrf/SELECT.html
- Oracle Documentation — Analytic Functions: https://docs.oracle.com/en/database/oracle/oracle-database/21/sqlrf/Analytic-Functions.html
- UNILAK Course Materials — DPR400210: Database Programming (Eric Maniraguha, 2026)
- W3Schools SQL Window Functions Reference: https://www.w3schools.com/sql/sql_window_functions.asp

---

## Repository Structure

```
database_programming_assignment1_[studentID]_Eugene/
│
├── README.md                          ← This file
├── assignment1_CTEs_WindowFunctions.sql  ← All SQL (setup + Part A + Part B)
└── screenshots/
    ├── er_diagram.png
    ├── a1_simple_cte.png
    ├── a2_multiple_ctes.png
    ├── a3_recursive_cte.png
    ├── a4_cte_aggregation.png
    ├── a5_cte_with_joins.png
    ├── b1_ranking_functions.png
    ├── b2_aggregate_window.png
    ├── b3_lag_lead.png
    └── b4_distribution.png
```

---

## Academic Integrity Statement

I, **Mpano Eugene**, confirm that this submission is my own original work. I have not copied any part of this assignment from classmates or online repositories. All SQL queries, analysis, and documentation were written independently in fulfillment of the requirements of DPR400210: Database Programming at UNILAK.

Any external references used are cited in the References section above.

> *"Whoever is faithful in very little is also faithful in much."* — Luke 16:10
