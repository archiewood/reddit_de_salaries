# How much should you be earning as a Data Engineer?

### Using data from Reddit posts to benchmark data engineering salaries, based on job title, experience and location.


I recently came across this [quarterly salary post](https://www.reddit.com/r/dataengineering/comments/v2ka3w/quarterly_salary_discussion/) on Reddit on r/dataengineering:

<img src="https://i.imgur.com/h0Wq3OM.png" width="100%">

As I was scrolling down the post, I noticed that the salaries were all in pretty much the same format, as suggested by the post. 

<img src="https://i.imgur.com/ytumQpu.png" width="100%">

Such structured data! And well labelled. A dream for a mini data project. Not only that, there were posts every quarter for the last 2 years. So there was effectively a time series!

## Approach

So I set about my approach for the project. As I saw it, I would need to do the following:
1. **Extract the data** from Reddit
2. **Turn this data into a table** 
3. **Clean the data** pretty thoroughly
4. **Present the results** in a easily digestible way

In this post I'll run through the steps of the project, and finish with the final results.

## 1. Extract the data

Reddit has an API. I didn't use it.

I haven't used it before and wasn't sure if they'd even be able to give me the data I wanted. But since the data is in the post, I knew it must be being delivered to the browser somewhere.

So I opened Chrome's trusty devtools and looked at the network tab, looking for a request that looked like it would have the data in. It's turns out it's loaded dynamically. After a bit of a trawl I found it.

Unfortunately websites don't load data in nice tablular formats like CSV, but as .json files. It looks like this (I've hidden some non-useful data with "..." in places):


<pre><code style="display:block;">
</code></pre>
```json_file
{
    "account": {...},
    "authorFlair": {...},
    "commentLists": {...},
    "comments": {
        "t1_iaut7cg": {
            ...
            "permalink": "/r/dataengineering/comments/v2ka3w/quarterly_salary_discussion/iaut7cg/",
            ...
            "media": {
                "richtextContent": {
                    "document": [
                        {"c": [{"c": [{"c": [{"e": "text","t": "Data Engineer"}],"e": "par"}],"e": "li"},
                                {"c": [{"c": [{"e": "text","t": "1 year. Coming from analyst roles though."],"e": "par"}],"e": "li"},
                                {"c": [{"c": [{"e": "text","t": "Utah.  Remote"}],"e": "par"}],"e": "li"},
                                {"c": [{"c": [{"e": "text","t": "95k USD"}],"e": "par"}],"e": "li"},
                                {"c": [{"c": [{"e": "text","t": "25k bonus,  options."}],"e": "par"}],"e": "li"},
                                {"c": [{"c": [{"e": "text","t": "Health care"}],"e": "par"}],"e": "li"},
                                {"c": [{"c": [{"e": "text","t": "Python,  AWS."}],"e": "par"}],"e": "li"}
                            ],
                            "e": "list",
                            "o": true
                        }
                    ]
                },
                "type": "rtjson",
                "rteMode": "richtext"
            },
            "profileImage": "https://styles.redditmedia.com/t5_iphh8/styles/profileIcon_snoo43d49fe6-1eaf-40da-bfbe-c37291a3277e-headshot.png?width=256&height=256&crop=256:256,smart&s=9c8037bf01c641aead748a436f524a18a943e997"
        },
}
```

I saved one of these files for each of the 5 posts from the last 5 quarters.

## 2. Turn the data into a table
I'm not particularly savvy about getting data out of json files. But for me I knew this was going to be a Python job. I booted up a jupyter notebook.

I noticed a few things about the structure of the data:
1. **Generally only "top level" comments had salary data.** Threads under comments were normally asking the commenter for more information. This made the task easier, as I could discard the other comments.
2. **Most commenters stuck to the numbered list format.** However some added a line of text before they started their list. And some used a list, but did not add numbers. This meant I would need a couple of different approaches to getting the data.
3. **Comments with less than 4 lines rarely contained salary data.** Because the post was asking for 4-7 data points, those with fewer lines were normally comments without salary data. I excluded these.

I also included the date the thread was created as a column. 

After a bit of trial and error I got to this. Done in 40 lines of Python. 

```python_JSON_unpacking

import json

# run it for each post file
dates=['2021-06-01','2021-09-01','2021-12-01','2022-03-01','2022-06-01']
array = []

for date in dates:
    with open('posts_'+date+'.json', 'r') as f:
        data = json.load(f)
        comment_no = 0
        #  "comments","t1_hmtrrkz", "media", "richtextContent", "document", "c"
        for key in data:
            if key == "comments":
                for comment in data[key]:
                    row=[]
                    row.append(date)
                    for i in range(0,7):
                        # if the data is in a list ie 1. 2. 3. 
                        try: 
                            value=data[key][comment]['media']["richtextContent"]["document"][0]['c'][i]['c'][0]['c'][0]['t']
                            row.append(value)
                        except:
                            # if the data is in a list, but has a non-list sentence first. (Posters often add a preamble)
                            try: 
                                value=data[key][comment]['media']["richtextContent"]["document"][1]['c'][i]['c'][0]['c'][0]['t']
                                row.append(value)
                            except:
                                try: 
                                    # this works if the data is not in a list
                                    value=data[key][comment]['media']["richtextContent"]["document"][i]['c'][0]['t']
                                    row.append(value)
                                except:
                                    pass
                    # remove results with less than 4 lines - these tend to be comments that do not contain salary data (which has 5-7 lines)
                    if len(row)>3:
                        array.append(row)
                    comment_no += 1

import pandas as pd
df=pd.DataFrame(array)
df.to_csv('salary_data.csv', index=False)
```

The final thing I do is to put it into a pandas DataFrame. Pandas is the de-facto standard for manipulating tables in python. It also makes it trivial to export to csv.

After all this, I have the following data:

```raw_salary_data
Select * from salary_data_raw
```

335 rows of tagged salary data!
It's a great start, but it's not usable yet. Time for...

## 3. Data Cleaning

Perhaps unsurprisingly, this was the most time consuming part of the project.

I find the easiest way to look at this kind of data is to use a spreadsheet, as you can see all the data visually.

To begin, there were quite a lot of rows that didn't contain salary data. 
And there were also a few rows where the poster hadn't followed the suggested format. Normally they had missed a field, or added an extra field.

I did a quick pass, tagging rows without salary data for removal, and adjusting rows which hadn't followed the suggested format manually.

After this I re-exported the data to a csv file, now with 297 rows, and opened a <a href="https://www.github.com/archiewood/reddit_de_salaries/data_analysis/salary_data_clean.ipynb" target="_blank">new jupyter notebook</a> to look at the data.

There were some steps I took to clean all the columns:
- Convert to lowercase
- Remove any contents in brackets

And then then there were steps I took for specific columns. Here for, categorical data, I aimed reduce the number of distinct groups that had the same label. For example, if two rows contained the titles "Sr. data engineer ii" and Senior DE 2" I would want these to be in the same group. For continuous data I was trying to extract the numbers correctly.

- **Title**: Remove punctuation, convert some common abbreviations, (e.g. `DE` -> `data engineer`, `jr` -> `junior`), convert numbers to Roman numerals

- **Experience**: Extract numbers, assume it is referring to years of experience, unless the string contains `month` or `mon`

- **Salary**: Extract numbers into one `local_currency` column, modifying for units (e.g. multiply by 1000 if there a `k` is present). Extract 3 digit codes (`GBP`) and currency symbols (`$`) to infer currency in another column. Create a column with salaries converted to USD. Clean out clear outliers.

- **Location**: Aggregate and consolidate US cities and regions (the majority) where possible. For non-US, aggregate to the country level, unless it is a major hub with many results (e.g. London, Toronto).

- **Industry**: Group similar industries, predominantly using keywords. E.g. `medical`, `healthcare`, `telemed` all classified as `healthcare`. This field was optional and many are blank.


After this, it's not perfect but it's retaining 297 rows with very usable data. You can see & download it below.

```all_data
SELECT *
FROM salary_data
```


<DataTable data={data.all_data}/>


## 4. Present the data

<!-- <Histogram data={data.all_data} bins=100/> -->





```most_common_titles
with all_titles as(
select
title,
count(*) as responses,
ROW_NUMBER () OVER ( ORDER BY count(*) desc ) row_num
from salary_data
group by title
order by responses desc)

select 
case 
when row_num <= 10 then title
else "all other" end as title_group,
sum(responses)
from all_titles
group by title_group
order by row_num
```

```most_common_countries
with all_countries as(
select
country,
count(*) as responses,
ROW_NUMBER () OVER ( ORDER BY count(*) desc ) row_num
from salary_data
group by country
order by responses desc)

select 
case 
when row_num <= 10 then country
else "all other" end as country_group,
sum(responses)
from all_countries
group by country_group
order by row_num
```

```most_common_industries
with all_ind as(
select
industry,
count(*) as responses,
ROW_NUMBER () OVER ( ORDER BY count(*) desc ) row_num
from salary_data
group by industry
order by responses desc)

select 
case 
when row_num <= 10 then industry
else "all other" end as industry_group,
sum(responses)
from all_ind
group by industry_group
order by row_num
```

### Summary
Let's start with a few summary statistics:


<BarChart
        data={data.most_common_titles}
        title='Most Common Titles' 
        swapXY=true
        sort=false
        yAxisTitle="Responses"
/>

<BarChart
        data={data.most_common_countries}
        title='Most Common Countries' 
        swapXY=true
        sort=false
        yAxisTitle="Responses"
/>

Many responses did not include industry. But of those that did, tech, finance and healthcare are the most common.

<BarChart
        data={data.most_common_industries}
        title='Most Common Industries'
        subtitle='Optional field' 
        swapXY=true
        sort=false
        yAxisTitle="Responses"
/>


For continuous fields, let's visualize them in a histogram.

```experience_avg
select
avg(years_experience)
from salary_data
```

```experience_levels
select 
--round up to nearest whole year
cast ( years_experience as int ) + ( years_experience > cast ( years_experience as int )) as years_experience_rounded,
count(*)
from salary_data

where years_experience >0
group by years_experience_rounded
```



The average level of experience is <Value data={data.experience_avg}/> years, with the majority of posts are from those with less than 5 years of experience:

<BarChart 
        data={data.experience_levels}
        title='Years of Experience'
/>




```avg_salary
SELECT round(AVG(salary_usd)/1000) FROM salary_data
```


```salary_distribution
select
  floor(salary_usd/10000.00)*10000 as bin_floor_usd,
  count(*) as responses
from salary_data
group by bin_floor_usd
order by bin_floor_usd
```

The $10k range with most salaries in is $60-70k, but the centre of the distribution appears to be around $90-100k.




<BarChart
        title='Salary Distribution'
        data={data.salary_distribution}
/>

The average salary is $<Value data={data.avg_salary}/>k, dragged up by a few high values in the dataset.

```salary_data_india
select
  salary_usd,
  salary_local,
  country
from salary_data
where country = 'india'
```



### More Interesting Analysis

Firstly, salary levels tend to vary by country. Lets see how this looks:



```avg_salary_by_country
with all_countries as (
select 
country,
round(AVG(salary_usd)) as avg_salary_usd, 
count(*) as responses,
ROW_NUMBER () OVER ( ORDER BY  AVG(salary_usd) desc ) row_num
from salary_data
group by country
order by responses desc)

select 
case 
        when responses > 3 then country 
        else "all other" end 
        as country_group,
avg_salary_usd,
sum(responses),
sum(row_num) as row_sum
from all_countries
group by country_group
order by row_sum
```

<BarChart 
        title='Average Salary by Country'
        subtitle='(Countries with > 3 responses)'
        data={avg_salary_by_country}
        x='country_group'
        swapXY=true
        sort=False
/>


Plotting salary vs years of experience, it seems the two are correlated.

```salary_vs_experience
select
        years_experience,
        round(salary_usd) as salary_usd
from salary_data
where years_experience >0.5
```

<ScatterPlot
        data={data.salary_vs_experience}
/>

This is easier to see if we group up by years of experience, and average the salary. There is a steep salary trajectory for those with less than 5 years of experience



        
```salary_by_experience
select cast ( years_experience as int ) + ( years_experience > cast ( years_experience as int )) as years_experience_rounded,
round(AVG(salary_usd)) as avg_salary_usd,
count(*) as responses
from salary_data
where years_experience >0
group by years_experience_rounded
```


<BubbleChart
        data={data.salary_by_experience}
        size='responses'
        title='Salary by Years of Experience'
/>



```salary_by_industry


```country_sort
select * from salary_data
order by country
```