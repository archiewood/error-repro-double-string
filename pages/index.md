---
title: Double being interpreted as string
---

```sql marketing_spend_demo
select
  lpad(date_part('week', date),2,'0') as week,
  date_part('year', date) as year,
  channel,
  sum(total_daily_cost_usd) as total_daily_cost_usd,
  sum(num_orders) as num_orders,
  sum(total_sales_usd) as total_sales_usd,
  sum(total_daily_cost_usd) / sum(num_orders) as cpa,
  lag(cpa) over (partition by channel order by year, week) as last_week_cpa,
  cpa / last_week_cpa - 1 as cpa_growth
from (select
  date_trunc('day', order_datetime) as date,
  date_trunc('month', order_datetime) as month_begin,
  orders.channel,
  first(daily_cost_usd) as total_daily_cost_usd,
  count(sales) as num_orders,
  sum(sales) as total_sales_usd,
from orders
left join (select
  month_begin,
  lead(month_begin) over (partition by marketing_channel order by month_begin) as next_month_begin,
  case 
    when next_month_begin is not null then datediff('day',month_begin, next_month_begin)
    else 31 end
  as days_in_month,
  marketing_channel,
  spend / days_in_month as daily_cost_usd
from marketing_spend) as spend
  on marketing_channel = channel 
  and spend.month_begin = date_trunc('month', order_datetime)
group by 1,2,3
order by 1,2)
group by 1,2,3
order by 1,2,5 desc
```

This is an error reproduction.

The following columns `last_week_cpa` and `cpa_growth` should are of type double, so should be treated as numbers, but are showing up as strings. You can see this in the above query.


## Testing

This is last week CPA, unformatted: **<Value data={marketing_spend_demo} column="last_week_cpa" row=10/>**

Now let me try and format it as a currency: <Value data={marketing_spend_demo} column="last_week_cpa" row=10 fmt=usd0 />

Now let me try to use it in a BigValue comparison.

<BigValue
  data={marketing_spend_demo}
  value=cpa
  fmt=usd2
  comparison=last_week_cpa
  comparisonFmt=usd0
/>


