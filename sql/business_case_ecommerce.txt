-- funnel query (users behaviour)
with funnel as
(select user_id, year, MAX(CASE WHEN event_type = 'home' THEN 1 ELSE 0 END) AS visited_home,
  MAX(CASE WHEN event_type = 'department' THEN 1 ELSE 0 END) AS browsed_department,
  MAX(CASE WHEN event_type = 'product' THEN 1 ELSE 0 END) AS viewed_product,
  MAX(CASE WHEN event_type = 'cart' THEN 1 ELSE 0 END) AS added_to_cart,
  MAX(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) AS made_purchase,
  MAX(CASE WHEN event_type = 'cancel' THEN 1 ELSE 0 END) AS cancel
from 
(select user_id, CAST(EXTRACT(YEAR from created_at) as string) as year, event_type from `bussiness-case-tyc.bussiness_case_ecommerce.events` where user_id is not null
) et
group by year, user_id
)

SELECT
  year,
  COUNT(*) AS total_users,
  SUM(visited_home) AS step_1_home,
  SUM(browsed_department) AS step_2_department,
  SUM(viewed_product) AS step_3_product,
  SUM(added_to_cart) AS step_4_cart,
  SUM(made_purchase) AS step_5_purchase,
  SUM(cancel) AS optional_cancel
FROM funnel
group by year order by year

-- order status
select  CAST(EXTRACT(YEAR from created_at) as string) as year, status, count(1) cantidad from `bussiness-case-tyc.bussiness_case_ecommerce.orders` group by year, status order by year, status  

-- successful repurchases and purchases
with purchase_orders as (select * from `bussiness-case-tyc.bussiness_case_ecommerce.orders` where status in ('Complete'))
SELECT
  o1.user_id,
  o1.order_id AS original_order,
  o1.created_at AS original_date,
  MIN(o2.order_id) AS repurchase_order,
  MIN(o2.created_at) AS repurchase_date,
  MIN(DATE_DIFF(o2.created_at,o1.created_at,DAY)) AS days_between
FROM purchase_orders o1
LEFT JOIN purchase_orders o2
  ON o1.user_id = o2.user_id
  AND o2.created_at > o1.created_at
  AND DATE_DIFF(o2.created_at,o1.created_at,DAY) <= 60
GROUP BY o1.user_id, o1.order_id, o1.created_at
ORDER BY o1.user_id, o1.created_at

-- traffic sources with purchase intention
select CAST(EXTRACT(YEAR from created_at) as string) as year,traffic_source, count(1) cantidad from `bussiness-case-tyc.bussiness_case_ecommerce.events` 
where user_id is not null 
and event_type='purchase'
group by year, traffic_source order by year, traffic_source

-- order, user and product info for modeling

with r as 
(select u.id user_id, u.age,u.gender,u.state,u.city, u.country, u.traffic_source,rp.original_order order_id, rp.original_date order_date, case when repurchase_order is null then 0 else 1 end repurchase 
from `bussiness-case-tyc.Ivonne_Heredia_Leon.repurchase_purchase_sucessful` rp
left join `bussiness-case-tyc.bussiness_case_ecommerce.users` u
on rp.user_id = u.id),

r_u as (select r.user_id, r.age,r.gender,r.state,r.city,r.country,r.traffic_source, r.order_date, r.order_id, o.num_of_item, r.repurchase from r left join `bussiness_case_ecommerce.orders` o  on r.order_id=o.order_id),

r_u_i as 
(select r_u.*, o.product_id, o.sale_price item_sale_price, o.inventory_item_id
from r_u left join `bussiness_case_ecommerce.order_items` o on r_u.order_id = o.order_id),
rep_products as
(select  r_u_i.*, p.category product_category, p.name product_name, p.brand product_brand,p.department product_department, p.distribution_center_id product_dc_id, p.retail_price product_retail_price
from r_u_i left join `bussiness_case_ecommerce.products` p on r_u_i.product_id = p.id 
),
final as 
(select rep_products.* , dc.name  dc_name
from rep_products 
left join `bussiness_case_ecommerce.distribution_centers` dc 
on rep_products.product_dc_id = dc.id)

select * from final 

-- user behaviour from orders
select t.user_id, t.order_id, t.order_date, 
date_diff(order_date, max(CASE WHEN o.status = 'Complete' THEN o.created_at END), day) days_between_orders,
sum(case when o.status ='Cancelled' then 1 else 0 end) cancelled_orders,
sum(case when o.status ='Returned' then 1 else 0 end) returned_orders,
sum(case when o.status ='Complete' then 1 else 0 end) num_past_purchase   
from `bussiness-case-tyc.Ivonne_Heredia_Leon.order_user_product_data` t
left join `bussiness_case_ecommerce.orders` o
on t.user_id = o.user_id and t.order_date > o.created_at
group by t.user_id, t.order_id, t.order_date

--user exposure 
select t.user_id, t.order_id, t.order_date, 
COUNT(CASE WHEN e.created_at >= TIMESTAMP_SUB(t.order_date, INTERVAL 7 DAY) AND t.order_date > e.created_at THEN 1 END)  events_7_days_all,
COUNT(CASE WHEN e.created_at >= TIMESTAMP_SUB(t.order_date, INTERVAL 7 DAY) AND t.order_date > e.created_at AND e.traffic_source ='Organic'  THEN 1 END)  events_7_days_organic,
COUNT(CASE WHEN e.created_at >= TIMESTAMP_SUB(t.order_date, INTERVAL 7 DAY) AND t.order_date > e.created_at AND e.traffic_source in('Facebook', 'YouTube')  THEN 1 END)  events_7_days_social_network,
COUNT(CASE WHEN e.created_at >= TIMESTAMP_SUB(t.order_date, INTERVAL 7 DAY) AND t.order_date > e.created_at AND e.traffic_source='Adwords'  THEN 1 END)  events_7_days_add,
COUNT(CASE WHEN e.created_at >= TIMESTAMP_SUB(t.order_date, INTERVAL 14 DAY) AND t.order_date > e.created_at THEN 1 END)  events_14_days_all,
COUNT(CASE WHEN e.created_at >= TIMESTAMP_SUB(t.order_date, INTERVAL 14 DAY) AND t.order_date > e.created_at AND e.traffic_source ='Organic'  THEN 1 END)  events_14_days_organic,
COUNT(CASE WHEN e.created_at >= TIMESTAMP_SUB(t.order_date, INTERVAL 14 DAY) AND t.order_date > e.created_at AND e.traffic_source in('Facebook', 'YouTube')  THEN 1 END)  events_14_days_social_network,
COUNT(CASE WHEN e.created_at >= TIMESTAMP_SUB(t.order_date, INTERVAL 14 DAY) AND t.order_date > e.created_at AND e.traffic_source='Adwords'  THEN 1 END)  events_14_days_add,
COUNT(CASE WHEN e.created_at >= TIMESTAMP_SUB(t.order_date, INTERVAL 30 DAY) AND t.order_date > e.created_at AND e.traffic_source='Adwords'  THEN 1 END)  events_30_days_add
from `bussiness-case-tyc.Ivonne_Heredia_Leon.order_user_product_data` t
left join `bussiness_case_ecommerce.events` e
on t.user_id = e.user_id and t.order_date > e.created_at
group by t.user_id, t.order_id, t.order_date

-- training data
select d.*,b.days_between_orders, b.cancelled_orders,b.returned_orders,b.num_past_purchase, CAST(EXTRACT(HOUR from d.order_date) as string) order_hour, CAST(EXTRACT(DAY from d.order_date) as string) order_day, CAST(EXTRACT(MONTH from d.order_date) as string) order_month 
from (select t.*, 
e.events_7_days_add, e.events_7_days_organic, e.events_7_days_social_network, e.events_7_days_all,
e.events_14_days_add, e.events_14_days_organic, e.events_14_days_social_network, e.events_14_days_all,
e.events_14_days_add
from `Ivonne_Heredia_Leon.order_user_product_data` t 
left join `Ivonne_Heredia_Leon.user_events` e on t.user_id = e.user_id and t.order_id=e.order_id) d
left join `Ivonne_Heredia_Leon.user_order_behaviour` b
on d.user_id = b.user_id and d.order_id=b.order_id