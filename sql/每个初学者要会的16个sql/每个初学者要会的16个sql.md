# 每个初学者都需要了解的 16 种 SQL 技术

## 从 1 到 10，您的数据仓库技能有多好？

想要超过 7/10？那么这篇文章就是为你准备的。



你的 SQL 有多好？想尽快准备好面试？



这篇博文详细解释了最复杂的数据仓库 SQL 技术。我将使用 BigQuery 标准 SQL 方言写下关于这个主题的一些想法。

## 1.增量表和MERGE

更新表很重要。这确实很重要。理想情况是当您拥有主键、唯一整数和自动递增的事务时。这种情况下的表更新很简单：



```sql
insert target_table (transaction_id)
select transaction_id from source_table where transaction_id > (select max(transaction_id) from target_table);
```





在现代数据仓库中处理非规范化星型模式数据集时，情况并非总是如此。您的任务可能是使用 SQL 创建**会话**和/或仅使用一部分数据增量更新数据集。 `transaction_id`可能不存在，但您必须处理唯一键取决于已知的最新`transaction_id` （或时间戳）的数据模型。例如， `last_online`数据集中的`user_id`取决于最新的已知连接时间戳。在这种情况下，您可能希望`update`现有用户并`insert`新用户。

### MERGE 和增量更新

您可以使用**MERGE** ，也可以将操作拆分为两个操作。一种用新记录更新现有记录，另一种插入不存在的全新记录（LEFT JOIN 情况）。

**MERGE**是关系数据库中常用的语句。 Google BigQuery MERGE 命令是一种数据操作语言 (DML) 语句。它通常用于在一条语句中以原子方式执行三个主要功能。这些函数是 UPDATE、INSERT 和 DELETE。



- 当两个或多个数据匹配时，可以使用 UPDATE 或 DELETE 子句。
- 当两个或多个数据不同且不匹配时，可以使用 INSERT 子句。
- 当给定数据与源不匹配时，也可以使用 UPDATE 或 DELETE 子句。



这意味着 Google BigQuery MERGE 命令使您能够通过更新、插入和删除 Google BigQuery 表中的数据来合并 Google BigQuery 数据。

考虑这个 SQL：





```sql
create temp table last_online as (
    select 1 as user_id
    , timestamp('2000-10-01 00:00:01') as last_online
)
;
create temp table connection_data  (
  user_id int64
  ,timestamp timestamp
)
PARTITION BY DATE(_PARTITIONTIME)
;
insert connection_data (user_id, timestamp)
    select 2 as user_id
    , timestamp_sub(current_timestamp(),interval 28 hour) as timestamp
union all
    select 1 as user_id
        , timestamp_sub(current_timestamp(),interval 28 hour) as timestamp
union all
    select 1 as user_id
        , timestamp_sub(current_timestamp(),interval 20 hour) as timestamp
union all
    select 1 as user_id
    , timestamp_sub(current_timestamp(),interval 1 hour) as timestamp
;

merge last_online t
using (
  select
      user_id
    , last_online
  from
    (
        select
            user_id
        ,   max(timestamp) as last_online

        from 
            connection_data
        where
            date(_partitiontime) >= date_sub(current_date(), interval 1 day)
        group by
            user_id

    ) y

) s
on t.user_id = s.user_id
when matched then
  update set last_online = s.last_online, user_id = s.user_id
when not matched then
  insert (last_online, user_id) values (last_online, user_id)
;
select * from last_online
;
```





## 2. 数词

执行 UNNEST() 并检查您需要的词是否在您需要的列表中可能在许多情况下很有用，即数据仓库情绪分析：





```sql
with titles as (
    select 'Title with word foo' as title union all
    select 'Title with word bar'
)
, data as (
select 
    title, 
    split(title, ' ') as words 
from 
    titles
)
select * from data, unnest(words) words
where
    words in ('bar')
;
```





## 3. 在 SELECT 语句之外使用 IF() 语句

这使我们有机会节省一些代码行并在代码方面更加雄辩。通常你会想把它放到一个子查询中，并在**where**子句中添加一个过滤器，但你可以**这样**做：





```
with daily_revenue as (
select
      current_date() as dt
    , 100          as revenue
    union all
select
      date_sub(current_date(), interval 1 day) as dt
    , 100          as revenue
)
select
*
from daily_revenue
where
    if(revenue >101,1,0) = 1
;
```





另一个示例如何**不**将其用于**分区**表。**不要这样做**。这是一个不好的例子，因为匹配的表后缀可能是动态确定的（基于表中的某些内容），您**将被收取全表扫描费用。**





```sql
SELECT *
FROM `firebase.events`
WHERE IF(condition,
         _TABLE_SUFFIX BETWEEN '20170101' AND '20170117',
         _TABLE_SUFFIX BETWEEN '20160101' AND '20160117')
;
```





您还可以在`HAVING`子句和`AGGREGATE`函数中使用它。

## 4. 使用 GROUP BY ROLLUP

ROLLUP 函数用于在多个级别执行聚合。当您必须使用维度图时，这很有用。





![img](data:image/svg+xml,%3csvg%20xmlns=%27http://www.w3.org/2000/svg%27%20version=%271.1%27%20width=%27800%27%20height=%27367.1153846153846%27/%3e)![图片由作者提供](https://hackernoon.imgix.net/images/R5lNhYPYD8WvrV5FEBdiHpJNWTh2-j3b2arp.png?auto=format&fit=max&w=1920)



图片由作者提供



以下查询按**where**子句中指定的交易类型 (is_gift) 返回每天的总积分支出，它还显示每天的总支出以及所有可用日期的总支出。





```sql
with data as (
    select
    current_timestamp() as ts           
    ,'stage'            as context_type 
    ,1                  as user_id      
    ,100                as credit_value 
    , true              as is_gift
union all
    select
    timestamp_sub(current_timestamp(), interval 24 hour) as ts           
    ,'user'             as context_type 
    ,1                  as user_id      
    ,200                as credit_value 
    ,false              as is_gift
union all
    select
    timestamp_sub(current_timestamp(), interval 24*2 hour) as ts           
    ,'user'             as context_type 
    ,3                  as user_id      
    ,300                as credit_value 
    ,true               as is_gift

)
, results as (
select 
     date(ts) as date 
    ,context_type
    ,sum(credit_value)/100 as daily_credits_spend
from data    

group by rollup(1, context_type)
order by 1
)

select
  date
  ,if(context_type is null, 'total', context_type) as context_type
  ,daily_credits_spend
from results
order by date
; 
```





## 5.将表转换为JSON

假设您需要将表转换为 JSON 对象，其中每条记录都是嵌套数组的一个元素。这是`to_json_string()`函数有用的地方：





```sql
with mytable as (
 select 1 as x, 'foo' as y, true as z union all
 select 2, 'bar', false
)
select 
    concat("{", "\"MyTable\":", "[", string_agg(to_json_string(t), ","), "]", "}")
from mytable as t
;
```





然后您可以在任何地方使用它：日期、营销渠道、指数、直方图等。

## 6.使用分区

给定`user_id` 、 `date`和`total_cost`列。对于每个日期，您如何在保留所有行的同时显示每个客户的总收入值？你可以这样实现：





```sql
select
     date
    ,user_id
    ,total_cost
    ,sum(total_cost) over (partition by date,user_id) as revenue_per_day
from production.payment_transaction
;
```





## 7.移动平均线

BI 开发人员的任务通常是在他们的报告和出色的仪表盘中添加移动平均线。这可能是 7、14、30 天/月甚至年 MA 线图。那么我们该怎么做呢？



```sql
with dates as (
select
    dt
from 
    unnest(generate_date_array(date_sub(current_date(), interval 90 day), current_date(), interval 1 day)) as dt
)

, data as (
    select dt
        , CEIL(RAND()*1000) as revenue -- just some random data.
    from
        dates
)
select
  dt
, revenue
, AVG(revenue) OVER(ORDER BY unix_date(dt) RANGE BETWEEN 6 PRECEDING AND CURRENT ROW) as seven_day_moving_average
from data
;
```







## 8. 日期数组

当您处理**用户保留**或想要检查某些数据集是否存在缺失值（即日期）时，它会变得非常方便。 BigQuery 有一个名为`GENERATE_DATE_ARRAY`函数：





```sql
select
 dt
from 
    unnest(generate_date_array('2019–12–04', '2020–09–17', interval 1 day)) as dt
;
```



## 9.行号()

这对于从您的数据中获取最新信息很有用，即最新更新的记录等，甚至可以删除重复项：





```sql
with reputation_data as (
select
      1     as user_id
    , 100   as reputation
    , 1     as reputation_level
    , timestamp_sub(current_timestamp(), interval 3 hour) as ts
union all
select
      1     as user_id
    , 101   as reputation
    , 1     as reputation_level
    , timestamp_sub(current_timestamp(), interval 2 hour)
union all
select
      1     as user_id
    , 200   as reputation
    , 2     as reputation_level
    , timestamp_sub(current_timestamp(), interval 1 hour)
)
select *
from reputation_data a
qualify row_number() over (partition by a.user_id order by a.ts desc) = 1
;
```





## 10.NTILE()

另一个编号功能。如果您有移动应用程序，这对于监控诸如`Login duration in seconds`之类的事情非常有用。例如，我将我的应用程序连接到 Firebase，当用户`login`时，我可以看到他们花了多长时间。





![img](data:image/svg+xml,%3csvg%20xmlns=%27http://www.w3.org/2000/svg%27%20version=%271.1%27%20width=%27800%27%20height=%27434.26573426573424%27/%3e)![图片由作者提供](https://hackernoon.imgix.net/images/R5lNhYPYD8WvrV5FEBdiHpJNWTh2-y4a2axa.png?auto=format&fit=max&w=1920)



图片由作者提供



此函数根据行顺序将行划分为`constant_integer_expression`存储桶，并返回分配给每行的从 1 开始的存储桶编号。桶中的行数最多可以相差 1。余数值（行数除以桶的余数）从桶 1 开始分配给每个桶。如果`constant_integer_expression`计算结果为 NULL、0 或负数，提供了一个错误。





```sql
select (case when tile = 50 then 'median' when tile = 95 then '95%' else '5%' end) as tile
    , dt
    , max(cast( round(duration/1000) as numeric)/1000 ) max_duration_s
    , min(cast( round(duration/1000) as numeric)/1000 ) min_duration_s

from (
    select 
         trace_info.duration_us duration
        , ntile(100) over (partition by (date(event_timestamp)) order by trace_info.duration_us) tile
        , date(event_timestamp) dt

    from firebase_performance.my_mobile_app 
    where 
        date(_partitiontime) >= parse_date('%y%m%d', @ds_start_date) and date(_partitiontime) <= parse_date('%y%m%d', @ds_end_date)
        and 
        date(event_timestamp) >= parse_date('%y%m%d', @ds_start_date)
        and 
        date(event_timestamp) <= parse_date('%y%m%d', @ds_end_date)
    and lower(event_type) = "duration_trace"
    and lower(event_name) = 'logon'
) x
WHERE tile in (5, 50, 95)
group by dt, tile
order by dt
;
```





## 11.排名/dense_rank

它们也称为**编号**函数。我倾向于使用`DENSE_RANK`**作为默认排名函数**，因为它不会跳过下一个可用排名，而`RANK`会。它返回连续的排名值。您可以将它与将结果分成不同的桶的分区一起使用。如果每个分区中的行具有相同的值，它们将获得相同的排名。**例子：**





```sql
with top_spenders as (
    select 1 as user_id, 100 as total_spend, 11   as reputation_level union all
    select 2 as user_id, 250 as total_spend, 11   as reputation_level union all
    select 3 as user_id, 250 as total_spend, 11   as reputation_level union all
    select 4 as user_id, 300 as total_spend, 11   as reputation_level union all
    select 11 as user_id, 1000 as total_spend, 22   as reputation_level union all
    select 22 as user_id, 1500 as total_spend, 22   as reputation_level union all
    select 33 as user_id, 1500 as total_spend, 22   as reputation_level union all
    select 44 as user_id, 2500 as total_spend, 22   as reputation_level 

)

select 
    user_id
    , rank() over(partition by reputation_level order by total_spend desc) as rank
    , dense_rank() over(partition by reputation_level order by total_spend desc) as dense_rank
from
    top_spenders
;
```





**产品价格的另一个例子：**





```sql
with products as (
    
    select
        2                    as product_id      
        , 'premium_account'  as product_type    
        , 100                as total_cost   
    union all
    select
        1                    as product_id      
        , 'premium_group'    as product_type
        , 200                as total_cost
    union all
    select
        111                  as product_id      
        , 'bots'             as product_type    
        , 300                as total_cost      
    union all
    select
        112                  as product_id      
        , 'bots'             as product_type    
        , 400                as total_cost      
    union all
    select
        113                  as product_id      
        , 'bots'             as product_type    
        , 500                as total_cost      
    union all
    select
        213                  as product_id      
        , 'bots'             as product_type    
        , 300                as total_cost      
  
)
select * from (
	select
		  product_id
		, product_type
		, total_cost as product_price
		, dense_rank () over ( 
			partition by product_type
			order by total_cost desc
		) price_rank 
	from
		products
) t
where price_rank < 3
;
```





## 12. 旋转/旋转

Pivot 将行更改为列。这就是它所做的一切。 Unpivot 做[相反的](https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax?ref=hackernoon.com#unpivot_operator)事情。





```sql
select * from
(
  -- #1 from_item
  select 
     extract(month from dt) as mo         
    ,product_type    
    ,revenue   
  from (
    select
        date(current_date()) as dt              
        , 'premium_account'  as product_type    
        , 100                as revenue   
    union all
    select
        date_sub(current_date(), interval 1 month) as dt
        , 'premium_group'    as product_type
        , 200                as revenue
    union all
    select
        date_sub(current_date(), interval 2 month) as dt
        , 'bots'             as product_type
        , 300                as revenue
  )
)
pivot
(
  -- #2 aggregate
  avg(revenue) as avg_revenue_
  -- #3 pivot_column
  for product_type in ('premium_account', 'premium_group')
)
;
```





## 13. 首值/末值

这是另一个有用的函数，它有助于获取每一行相对于该特定分区中第一个/最后一个值的增量。





```sql
with top_spenders as (
    select 1 as user_id, 100 as total_spend, 11   as reputation_level union all
    select 2 as user_id, 150 as total_spend, 11   as reputation_level union all
    select 3 as user_id, 250 as total_spend, 11   as reputation_level union all
    select 11 as user_id, 1000 as total_spend, 22   as reputation_level union all
    select 22 as user_id, 1500 as total_spend, 22   as reputation_level union all
    select 33 as user_id, 2500 as total_spend, 22   as reputation_level 

)
, data as (
    select
        user_id
        ,total_spend
        ,reputation_level
        ,first_value(total_spend)
    over (partition by reputation_level order by total_spend desc
    rows between unbounded preceding and unbounded following) as top_spend
  from top_spenders
)

select
    user_id
    ,reputation_level
    ,total_spend
    ,top_spend          as top_spend_by_rep_level
    ,total_spend - top_spend as delta_in_usd
from data
;
```





## 14. 将表转换为结构数组并将它们传递给 UDF

当您需要将具有某些复杂逻辑的用户定义函数 (UDF) 应用于每一行或一个表时，这非常有用。您始终可以将您的表视为一组 TYPE STRUCT 对象，然后将它们中的每一个传递给 UDF。这取决于你的逻辑。例如，我用它来计算购买过期时间：





```sql
select 
     target_id
    ,product_id
    ,product_type_id
    ,production.purchase_summary_udf()(
        ARRAY_AGG(
            STRUCT(
                target_id
                , user_id
                , product_type_id
                , product_id
                , item_count
                , days
                , expire_time_after_purchase
                , transaction_id 
                , purchase_created_at 
                , updated_at
            ) 
            order by purchase_created_at
        )
    ) AS processed

from new_batch
;
```





以类似的方式，您可以创建表而无需使用**UNION ALL** 。例如，我用它来模拟单元测试的一些测试数据。这样，您只需在编辑器中使用`Alt` + `Shift` + `Down`即可非常快速地完成此操作。





```sql
select * from unnest([
        struct
        (     
            1                                 as user_id
        ,   111                               as reputation
        ,   timestamp('2021-12-16 13:00:01')  as update_time
        
        ),

        (
            2                                 --as user_id
        ,   111                               --as reputation
        ,   timestamp('2011-12-16 13:00:01')  --as update_time
        ),

        (
            3                                 --as user_id
        ,   111                               --as reputation
        ,    timestamp(format_timestamp("%Y-%m-%d 12:59:01 UTC" ,timestamp(date_sub(current_date(), interval 0 day))))   --as update_time
        )
        ]
    ) as t
```





## 15. 使用 FOLLOWING 和 UNBOUNDED FOLLOWING 创建事件漏斗

营销渠道就是一个很好的例子。您的数据集可能包含不断重复的相同类型的事件，但理想情况下您希望将每个事件与下一个不同类型的事件链接起来。当您需要获取某项列表（即事件、购买等）以构建渠道数据集时，这可能很有用。使用 PARTITION BY 它可以让您有机会对所有以下事件进行分组，而不管每个分区中存在多少事件。





```sql
with d as (
select * from unnest([
  struct('0003f' as user_pseudo_id, 12322175 as user_id, timestamp '2020-10-10 16:46:59.878 UTC' as event_timestamp, 'join_group' as event_name),
  ('0003',12,timestamp '2022-10-10 16:50:03.394 UTC','set_avatar'),
  ('0003',12,timestamp '2022-10-10 17:02:38.632 UTC','set_avatar'),
  ('0003',12,timestamp '2022-10-10 17:09:38.645 UTC','set_avatar'),
  ('0003',12,timestamp '2022-10-10 17:10:38.645 UTC','join_group'),
  ('0003',12,timestamp '2022-10-10 17:15:38.645 UTC','create_group'),
  ('0003',12,timestamp '2022-10-10 17:17:38.645 UTC','create_group'),
  ('0003',12,timestamp '2022-10-10 17:18:38.645 UTC','in_app_purchase'),
  ('0003',12,timestamp '2022-10-10 17:19:38.645 UTC','spend_virtual_currency'),
  ('0003',12,timestamp '2022-10-10 17:19:45.645 UTC','create_group'),
  ('0003',12,timestamp '2022-10-10 17:20:38.645 UTC','set_avatar')
  ]
  ) as t)

  , event_data as (
SELECT 
    user_pseudo_id
  , user_id
  , event_timestamp
  , event_name
  , ARRAY_AGG(
        STRUCT(
              event_name AS event_name
            , event_timestamp AS event_timestamp
        )
    ) 
    OVER(PARTITION BY user_pseudo_id ORDER BY event_timestamp ROWS BETWEEN 1 FOLLOWING AND  UNBOUNDED FOLLOWING ) as next_events

FROM d
WHERE
DATE(event_timestamp) = "2022-10-10" 

)
select
    user_pseudo_id
  , user_id
  , event_timestamp
  , event_name
  , (SELECT 
        event_name FROM UNNEST(next_events) next_event
    WHERE t.event_name != event_name
    ORDER BY event_timestamp  LIMIT 1
    -- change to ORDER BY event_timestamp desc if prev event needed
  ) next_event
  , (SELECT 
        event_timestamp FROM UNNEST(next_events) next_event
    WHERE t.event_name != event_name
    ORDER BY event_timestamp  LIMIT 1
    -- change to ORDER BY event_timestamp desc if prev event needed
  ) next_event_ts

from event_data t
;
```





## 16.正则表达式

如果您需要从非结构化数据中提取某些内容，例如外汇汇率、自定义分组等，您会使用它。

### 使用正则表达式处理货币汇率

考虑这个带有汇率数据的例子：





```sql
-- One or more digits (\d+), optional period (\.?), zero or more digits (\d*).
with object as
(select  '{"aed":3.6732,"afn":78.45934,"all":110.586428}' as rates)

, data as (
select "usd" as base_currency,
  regexp_extract_all(rates, r'"[^"]+":\d+\.?\d*') as pair
from object
)
, splits as (
select base_currency, pair, split(pair, ':') positions 
from data cross join unnest (pair) as pair
)
select base_currency, pair,  positions[offset(0)] as rate_currency,  positions[offset(1)] as rate
from splits  
;
```





### 使用正则表达式处理应用程序版本

有时您可能想使用`regexp`为您的应用程序获取**主要**版本、**发布版本**或**修改**版本，并创建自定义报告：





```sql
with events as (
  select  'open_chat' as event_name, '10.1.0' as app_display_version union all
  select  'open_chat' as event_name, '10.1.9' as app_display_version union all
  select  'open_chat' as event_name, '9.1.4' as app_display_version union all
  select  'open_chat' as event_name, '9.0.0' as app_display_version
)
select
     app_display_version
    ,REGEXP_EXTRACT(app_display_version, '^[^.^]*') main_version
    ,safe_cast(REGEXP_EXTRACT(app_display_version, '[0-9]+.[0-9]+') as float64) release_version
    ,safe_cast(REGEXP_EXTRACT(app_display_version, r"^[a-zA-Z0-9_.+-]+.[a-zA-Z0-9-]+\.([a-zA-Z0-9-.]+$)") as int64) as mod_version
from events
;
```





## 结论

SQL 是一种有助于操作数据的强大工具。希望这些来自数字营销的 SQL 用例对您有用。这确实是一项方便的技能，可以帮助您完成许多项目。这些 SQL 片段让我的生活变得轻松多了，我几乎每天都在工作中使用它们。此外，SQL 和现代数据仓库是数据科学的必备工具。其强大的方言功能允许轻松建模和可视化数据。因为 SQL 是数据仓库和商业智能专业人员使用的语言，所以如果您想与他们共享数据，它是一个很好的选择。这是与市场上几乎所有数据仓库/湖解决方案进行通信的最常见方式。

摘自[@datamike](https://hackernoon.com/u/datamike)(https://medium.com/@mshakhomirov?ref=hackernoon.com)发表的[hackernoon.com]([16 SQL Techniques Every Beginner Needs to Know | HackerNoon](https://hackernoon.com/16-sql-techniques-every-beginner-needs-to-know))

