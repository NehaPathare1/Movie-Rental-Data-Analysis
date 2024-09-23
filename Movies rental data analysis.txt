SQL Project: Film Rental Database Analysis
Project Overview
This project analyzes a film rental database to understand key aspects such as how films are rented, customer behavior, and the revenue generated from rentals. The following sections break down various SQL queries to derive insights from the data.

Database Overview
The database consists of the following tables:

film: Information about films, including rental rates and lengths.
category: Different categories of films.
customer: Information about customers.
rental: Records of rental transactions.
payment: Details of payments made for rentals.

SQL Queries

1. Foreign Key Constraints in Rental Table
This query displays the fields in the "rental" table that have foreign key constraints.


PRAGMA foreign_key_list(rental);

2. Top Categories by Average Film Length
Find the top 5 film categories with the longest average lengths and compare them to the overall average length of films.


WITH category_avg_length AS (
    SELECT c.name AS name, AVG(f.length) AS avg_length
    FROM film f
    JOIN film_category fc ON f.film_id = fc.film_id
    JOIN category c ON fc.category_id = c.category_id
    GROUP BY c.name
),
overall_avg_length AS (
    SELECT AVG(length) AS overall_avg
    FROM film
)
SELECT cal.name, cal.avg_length, (cal.avg_length - oal.overall_avg) AS difference
FROM category_avg_length cal
CROSS JOIN overall_avg_length oal
ORDER BY cal.avg_length DESC
LIMIT 5;

3. Customers Renting Films from All Categories
Identify customers who have rented films from every category.


WITH category_count AS (
    SELECT COUNT(*) AS total_categories
    FROM category
),
customer_rentals_per_category AS (
    SELECT r.customer_id, fc.category_id
    FROM rental r
    JOIN inventory i ON r.inventory_id = i.inventory_id
    JOIN film_category fc ON i.film_id = fc.film_id
    GROUP BY r.customer_id, fc.category_id
),
customer_category_rentals AS (
    SELECT customer_id, COUNT(DISTINCT category_id) AS category_rentals
    FROM customer_rentals_per_category
    GROUP BY customer_id
)
SELECT ccr.customer_id
FROM customer_category_rentals ccr
JOIN category_count cc ON ccr.category_rentals = cc.total_categories;

4. Average Rental Duration for Films
Calculate the average rental duration for films rented by more than 5 customers.


WITH rental_counts AS (
    SELECT i.film_id, COUNT(DISTINCT r.customer_id) AS customer_count
    FROM rental r
    JOIN inventory i ON r.inventory_id = i.inventory_id
    GROUP BY i.film_id
)
SELECT AVG(f.rental_duration) AS avg_rental_duration
FROM rental_counts rc
JOIN film f ON rc.film_id = f.film_id
WHERE rc.customer_count > 5;

5. Top Films by Number of Rentals
Find the top 3 most rented films in each store.


WITH mydata1 AS (
    WITH mydata AS (
        SELECT i.store_id, i.film_id, title, COUNT(rental_id) AS rental_count
        FROM inventory AS i 
        JOIN film USING(film_id) 
        JOIN rental AS r USING(inventory_id) 
        GROUP BY i.store_id, i.film_id, title
    )
    SELECT *, ROW_NUMBER() OVER(PARTITION BY store_id ORDER BY rental_count DESC) AS rn 
    FROM mydata
)
SELECT store_id, film_id, title, rental_count 
FROM mydata1
WHERE rn <= 3;

6. Actors in Multiple Film Categories
Find actors who have appeared in films from every category.


WITH CategoryCount AS (
    SELECT COUNT(DISTINCT c.category_id) AS total_categories
    FROM category c
),
ActorCategoryCount AS (
    SELECT a.actor_id, a.first_name || ' ' || a.last_name AS actor_name, COUNT(DISTINCT fc.category_id) AS category_count
    FROM actor a
    JOIN film_actor fa ON a.actor_id = fa.actor_id
    JOIN film_category fc ON fa.film_id = fc.film_id
    GROUP BY a.actor_id, a.first_name, a.last_name
)
SELECT acc.actor_id, acc.actor_name
FROM ActorCategoryCount acc
JOIN CategoryCount cc ON acc.category_count = cc.total_categories;

7. Top Countries by Films Rented
Find the top 3 countries with the most films rented by customers from those countries.


WITH CountryRentalCounts AS (
    SELECT 
        co.country_id,
        co.country,
        COUNT(r.rental_id) AS rental_count
    FROM country co
    JOIN city ci ON co.country_id = ci.country_id
    JOIN address a ON ci.city_id = a.city_id
    JOIN customer c ON a.address_id = c.address_id
    JOIN rental r ON c.customer_id = r.customer_id
    GROUP BY co.country_id, co.country
)
SELECT country_id, country, rental_count
FROM CountryRentalCounts
ORDER BY rental_count DESC
LIMIT 3;

8. Total Revenue from Rentals by Customers in Cities Starting with 'S'
Calculate the total revenue generated from rentals by customers living in cities that start with 'S'.

SELECT 
    SUM(p.amount) AS total_revenue
FROM 
    city ci
JOIN 
    address a ON ci.city_id = a.city_id
JOIN 
    customer c ON a.address_id = c.address_id
JOIN 
    rental r ON c.customer_id = r.customer_id
JOIN 
    payment p ON r.rental_id = p.rental_id
WHERE 
    ci.city LIKE 'S%';

9. Percentage of Customers with Multiple Rentals
Determine the percentage of customers who have rented the same film more than once.


WITH rental_counts AS (
    SELECT customer_id, film_id, COUNT(*) AS rental_count
    FROM rental r
    JOIN inventory i ON r.inventory_id = i.inventory_id
    GROUP BY customer_id, film_id
),
repeat_customers AS (
    SELECT DISTINCT customer_id 
    FROM rental_counts
    WHERE rental_count > 1
)
SELECT
    (SELECT COUNT(DISTINCT customer_id) FROM rental) AS total_customers,
    (SELECT COUNT(DISTINCT customer_id) FROM repeat_customers) AS repeat_customers,
    (SELECT COUNT(DISTINCT customer_id) FROM repeat_customers) * 1.0 /
    (SELECT COUNT(DISTINCT customer_id) FROM rental) * 100 AS percentage_repeat_customers;

10. Top Revenue Categories and Average Revenues
Find the top 5 categories by total revenue and compare their average revenues to the overall average revenue of films.


WITH CategoryRevenue AS (
    SELECT 
        fc.category_id,
        c.name,
        SUM(p.amount) AS total_revenue,
        AVG(p.amount) AS avg_revenue
    FROM 
        film_category fc
    JOIN 
        film f ON fc.film_id = f.film_id
    JOIN 
        inventory i ON f.film_id = i.film_id
    JOIN 
        rental r ON i.inventory_id = r.inventory_id
    JOIN 
        payment p ON r.rental_id = p.rental_id
    JOIN 
        category c ON fc.category_id = c.category_id
    GROUP BY 
        fc.category_id, c.name
),
OverallAverage AS (
    SELECT 
        AVG(total_revenue) AS overall_avg_revenue
    FROM (
        SELECT 
            SUM(p.amount) AS total_revenue
        FROM 
            film f
        JOIN 
            inventory i ON f.film_id = i.film_id
        JOIN 
            rental r ON i.inventory_id = r.inventory_id
        JOIN 
            payment p ON r.rental_id = p.rental_id
        GROUP BY 
            f.film_id
    ) AS FilmRevenue
)
SELECT 
    cr.category_id,
    cr.name,
    cr.total_revenue,
    (cr.avg_revenue / oa.overall_avg_revenue) * 100 AS percentage_of_overall_avg_revenue
FROM 
    CategoryRevenue cr, OverallAverage oa
ORDER BY 
    cr.total_revenue DESC
LIMIT 5;

11. Percentage of Revenue from Films in Top 10% Rental Rate Range
Calculate the percentage of revenue from films with the highest rental rates (top 10%).


WITH top_10_percent_rental_rate AS (
    SELECT 
        rental_rate
    FROM 
        film
    ORDER BY 
        rental_rate DESC
    LIMIT 
        (SELECT CEIL(COUNT(*) * 0.1) FROM film)
),
top_10_percent_revenue AS (
    SELECT 
        SUM(payment.amount) AS top_10_percent_revenue
    FROM 
        payment
    JOIN 
        rental ON payment.rental_id = rental.rental_id
    JOIN 
        inventory ON rental.inventory_id = inventory.inventory_id
    JOIN 
        film ON inventory.film_id = film.film_id
    WHERE 
        film.rental_rate IN (SELECT * FROM top_10_percent_rental_rate)
)
SELECT 
    (top_10_percent_revenue.top_10_percent_revenue * 1.0 / (SELECT SUM(amount) FROM payment)) *
