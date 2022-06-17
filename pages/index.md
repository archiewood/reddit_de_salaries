# test

```select10
SELECT * FROM salary_data

```

```avg_salary
SELECT AVG(salary_usd) FROM salary_data
```


```avg_salary_by_currency
select 
salary_currency,
AVG(salary_usd) as avg_salary, 
count(*) as reports
from salary_data
group by salary_currency
order by avg_salary desc
```

<BarChart data={avg_salary_by_currency}/>

```median_salary
SELECT salary_usd FROM salary_data

ORDER BY salary_usd
LIMIT 1
OFFSET (SELECT COUNT(*)
        FROM salary_data) / 2
```