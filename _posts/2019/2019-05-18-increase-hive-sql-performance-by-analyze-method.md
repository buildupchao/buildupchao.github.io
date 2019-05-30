---
title: Hive SQL查询效率提升之Analyze方案的实施
date: 2019-05-19 22:45:41
tags: ['Hive', 'HQL查询效率', 'Analyze Table']
---

## **0.简介**
Analyze，分析表（也称为计算统计信息）是一种内置的Hive操作，可以执行该操作来收集表上的元数据信息。这可以极大的改善表上的查询时间，因为它收集构成表中数据的行计数，文件计数和文件大小（字节），并在执行之前将其提供给查询计划程序。

<!-- more -->

## **1.如何分析表？**
- 基础分析语句
```SQL
ANALYZE TABLE my_database_name.my_table_name COMPUTE STATISTICS;
```
这是一个基础分析语句，不限制是否存在表分区，如果你是分区表更应该定期执行。

- 分析特定分区
```SQL
ANALYZE TABLE my_database_name.my_table_name PARTITION (YEAR=2019, MONTH=5, DAY=12) COMPUTE STATISTICS;
```
这是一个细粒度的分析语句。它收集指定的分区上的元数据，并将该信息存储在Hive Metastore中已进行查询优化。该信息包括每列，不同值的数量，NULL值的数量，列的平均大小，平均值或列中所有值的总和（如果类型为数字）和值的百分数。

- 分析列
```SQL
ANALYZE TABLE my_database_name.my_table_name COMPUTE STATISTICS FOR column1, column2, column3;
```
它收集指定列上的元数据，并将该信息存储在Hive Metastore中以进行查询优化。该信息包括每列，不同值的数量，NULL值的数量，列的平均大小，平均值或列中所有值的总和（如果类型为数字）和值的百分数。

- 分析指定分区上的列
```SQL
ANALYZE TABLE my_database_name.my_table_name PARTITION (YEAR=2019, MONTH=5, DAY=12, HOUR=0) COMPUTE STATISTICS for column1, column2, column3;

ANALYZE TABLE my_database_name.my_table_name PARTITION (YEAR=2019, MONTH=5, DAY=12, HOUR) COMPUTE STATISTICS for column1, column2, column3;

ANALYZE TABLE my_database_name.my_table_name PARTITION (YEAR=2019, MONTH=5, DAY=12, HOUR) COMPUTE STATISTICS FOR COLUMNS;
```
第一个SQL只会分析单小时分区上的列信息；
第二个SQL会分析单天分区上的列信息；
第三个SQL会分析单天分区上的所有列信息。

## **2.效果验证**

### **测试案例1**
- 数据准备
选取KS3线上数据集、TPC-DS基准测试数据集作为样本。结合Hive表分析操作，对多个文件格式以及压缩算法下的数据查询时间进行比对。
```SQL
SELECT count(DISTINCT(uuid)) AS script_appentry_30day_uv
FROM test_hive.document_assistant
WHERE dt >= '2019-03-12'
  AND dt <= '2019-04-10'
  AND p3 = '14'
  AND p5 = 'script_appentry'
```

- 测试结果
![](https://github.com/buildupchao/ImgStore/blob/master/blog/analyze_table.png?raw=true)
![](https://github.com/buildupchao/ImgStore/blob/master/blog/analyze_table2.png?raw=true)

### **测试案例2**
- 数据准备（TPC-DS基础测试）
  - 美国事务处理效能委员会(TPC,Transaction Processing Performance Council) ：是目前最知名的非赢利的数据管理系统评测基准标准化组织。它定义了多组标准测试集用于客观地、可重现地评测数据库的性能。
  - TPC-DS测试基准是TPC组织推出的用于替代TPC-H的下一代决策支持系统测试基准：TPC-DS采用星型、雪花型等多维数据模式。它包含7张事实表，17张维度表，平均每张表有18列。
  - TPC-DS 特点：
    - 共99个测试案例，遵循SQL'99和SQL 2003的语法标准，SQL案例比较复杂；
    - 分析的数据量大，并且测试案例是在回答真实的商业问题；
    - 测试案例中包含各种业务模型(如分析报告型，迭代式的联机分析型，数据挖掘型等)；
    - 几乎所有的测试案例都有很高的IO负载和CPU计算需求。

场景：单事实表、多个维表，复杂的Join
Store_Sales表记录数：2,879,987,999
事实表存储大小（GB）：Text：390， Parquet(Gzip)：116， Orc(Zlib):131
![](https://github.com/buildupchao/ImgStore/blob/master/blog/analyze_table3.png?raw=true)

query27.sql:
```SQL
-- start query 1 in stream 0 using template query27.tpl and seed 2017787633
select  i_item_id,
        s_state, grouping(s_state) g_state,
        avg(ss_quantity) agg1,
        avg(ss_list_price) agg2,
        avg(ss_coupon_amt) agg3,
        avg(ss_sales_price) agg4
 from store_sales, customer_demographics, date_dim, store, item
 where ss_sold_date_sk = d_date_sk and
       ss_item_sk = i_item_sk and
       ss_store_sk = s_store_sk and
       ss_cdemo_sk = cd_demo_sk and
       cd_gender = 'M' and
       cd_marital_status = 'U' and
       cd_education_status = '2 yr Degree' and
       d_year = 2001 and
       s_state in ('SD','FL', 'MI', 'LA', 'MO', 'SC')
 group by rollup (i_item_id, s_state)
 order by i_item_id
         ,s_state
 limit 100;

-- end query 1 in stream 0 using template query27.tpl
```

query28.sql:
```SQL
-- start query 1 in stream 0 using template query28.tpl and seed 444293455
select  *
from (select avg(ss_list_price) B1_LP
            ,count(ss_list_price) B1_CNT
            ,count(distinct ss_list_price) B1_CNTD
      from store_sales
      where ss_quantity between 0 and 5
        and (ss_list_price between 11 and 11+10
             or ss_coupon_amt between 460 and 460+1000
             or ss_wholesale_cost between 14 and 14+20)) B1,
     (select avg(ss_list_price) B2_LP
            ,count(ss_list_price) B2_CNT
            ,count(distinct ss_list_price) B2_CNTD
      from store_sales
      where ss_quantity between 6 and 10
        and (ss_list_price between 91 and 91+10
          or ss_coupon_amt between 1430 and 1430+1000
          or ss_wholesale_cost between 32 and 32+20)) B2,
     (select avg(ss_list_price) B3_LP
            ,count(ss_list_price) B3_CNT
            ,count(distinct ss_list_price) B3_CNTD
      from store_sales
      where ss_quantity between 11 and 15
        and (ss_list_price between 66 and 66+10
          or ss_coupon_amt between 920 and 920+1000
          or ss_wholesale_cost between 4 and 4+20)) B3,
     (select avg(ss_list_price) B4_LP
            ,count(ss_list_price) B4_CNT
            ,count(distinct ss_list_price) B4_CNTD
      from store_sales
      where ss_quantity between 16 and 20
        and (ss_list_price between 142 and 142+10
          or ss_coupon_amt between 3054 and 3054+1000
          or ss_wholesale_cost between 80 and 80+20)) B4,
     (select avg(ss_list_price) B5_LP
            ,count(ss_list_price) B5_CNT
            ,count(distinct ss_list_price) B5_CNTD
      from store_sales
      where ss_quantity between 21 and 25
        and (ss_list_price between 135 and 135+10
          or ss_coupon_amt between 14180 and 14180+1000
          or ss_wholesale_cost between 38 and 38+20)) B5,
     (select avg(ss_list_price) B6_LP
            ,count(ss_list_price) B6_CNT
            ,count(distinct ss_list_price) B6_CNTD
      from store_sales
      where ss_quantity between 26 and 30
        and (ss_list_price between 28 and 28+10
          or ss_coupon_amt between 2513 and 2513+1000
          or ss_wholesale_cost between 42 and 42+20)) B6
limit 100;

-- end query 1 in stream 0 using template query28.tpl
```

query43.sql:
```SQL
-- start query 1 in stream 0 using template query43.tpl and seed 1819994127
select  s_store_name, s_store_id,
        sum(case when (d_day_name='Sunday') then ss_sales_price else null end) sun_sales,
        sum(case when (d_day_name='Monday') then ss_sales_price else null end) mon_sales,
        sum(case when (d_day_name='Tuesday') then ss_sales_price else  null end) tue_sales,
        sum(case when (d_day_name='Wednesday') then ss_sales_price else null end) wed_sales,
        sum(case when (d_day_name='Thursday') then ss_sales_price else null end) thu_sales,
        sum(case when (d_day_name='Friday') then ss_sales_price else null end) fri_sales,
        sum(case when (d_day_name='Saturday') then ss_sales_price else null end) sat_sales
 from date_dim, store_sales, store
 where d_date_sk = ss_sold_date_sk and
       s_store_sk = ss_store_sk and
       s_gmt_offset = -6 and
       d_year = 1998
 group by s_store_name, s_store_id
 order by s_store_name, s_store_id,sun_sales,mon_sales,tue_sales,wed_sales,thu_sales,fri_sales,sat_sales
 limit 100;

-- end query 1 in stream 0 using template query43.tpl
```

query67.sql:
```SQL
-- start query 1 in stream 0 using template query67.tpl and seed 1819994127
select  *
from (select i_category
            ,i_class
            ,i_brand
            ,i_product_name
            ,d_year
            ,d_qoy
            ,d_moy
            ,s_store_id
            ,sumsales
            ,rank() over (partition by i_category order by sumsales desc) rk
      from (select i_category
                  ,i_class
                  ,i_brand
                  ,i_product_name
                  ,d_year
                  ,d_qoy
                  ,d_moy
                  ,s_store_id
                  ,sum(coalesce(ss_sales_price*ss_quantity,0)) sumsales
            from store_sales
                ,date_dim
                ,store
                ,item
       where  ss_sold_date_sk=d_date_sk
          and ss_item_sk=i_item_sk
          and ss_store_sk = s_store_sk
          and d_month_seq between 1212 and 1212+11
       group by  rollup(i_category, i_class, i_brand, i_product_name, d_year, d_qoy, d_moy,s_store_id))dw1) dw2
where rk <= 100
order by i_category
        ,i_class
        ,i_brand
        ,i_product_name
        ,d_year
        ,d_qoy
        ,d_moy
        ,s_store_id
        ,sumsales
        ,rk
limit 100;

-- end query 1 in stream 0 using template query67.tpl
```

query46.sql:
```SQL
-- start query 1 in stream 0 using template query46.tpl and seed 803547492
select  c_last_name
       ,c_first_name
       ,ca_city
       ,bought_city
       ,ss_ticket_number
       ,amt,profit
 from
   (select ss_ticket_number
          ,ss_customer_sk
          ,ca_city bought_city
          ,sum(ss_coupon_amt) amt
          ,sum(ss_net_profit) profit
    from store_sales,date_dim,store,household_demographics,customer_address
    where store_sales.ss_sold_date_sk = date_dim.d_date_sk
    and store_sales.ss_store_sk = store.s_store_sk
    and store_sales.ss_hdemo_sk = household_demographics.hd_demo_sk
    and store_sales.ss_addr_sk = customer_address.ca_address_sk
    and (household_demographics.hd_dep_count = 2 or
         household_demographics.hd_vehicle_count= 1)
    and date_dim.d_dow in (6,0)
    and date_dim.d_year in (1998,1998+1,1998+2)
    and store.s_city in ('Cedar Grove','Wildwood','Union','Salem','Highland Park')
    group by ss_ticket_number,ss_customer_sk,ss_addr_sk,ca_city) dn,customer,customer_address current_addr
    where ss_customer_sk = c_customer_sk
      and customer.c_current_addr_sk = current_addr.ca_address_sk
      and current_addr.ca_city <> bought_city
  order by c_last_name
          ,c_first_name
          ,ca_city
          ,bought_city
          ,ss_ticket_number
  limit 100;

-- end query 1 in stream 0 using template query46.tpl
```

query7.sql:
```SQL
-- start query 1 in stream 0 using template query7.tpl and seed 1930872976
select  i_item_id,
        avg(ss_quantity) agg1,
        avg(ss_list_price) agg2,
        avg(ss_coupon_amt) agg3,
        avg(ss_sales_price) agg4
 from store_sales, customer_demographics, date_dim, item, promotion
 where ss_sold_date_sk = d_date_sk and
       ss_item_sk = i_item_sk and
       ss_cdemo_sk = cd_demo_sk and
       ss_promo_sk = p_promo_sk and
       cd_gender = 'F' and
       cd_marital_status = 'W' and
       cd_education_status = 'Primary' and
       (p_channel_email = 'N' or p_channel_event = 'N') and
       d_year = 1998
 group by i_item_id
 order by i_item_id
 limit 100;

-- end query 1 in stream 0 using template query7.tpl
```

query73.sql:
```SQL
-- start query 1 in stream 0 using template query73.tpl and seed 1971067816
select c_last_name
       ,c_first_name
       ,c_salutation
       ,c_preferred_cust_flag
       ,ss_ticket_number
       ,cnt from
   (select ss_ticket_number
          ,ss_customer_sk
          ,count(*) cnt
    from store_sales,date_dim,store,household_demographics
    where store_sales.ss_sold_date_sk = date_dim.d_date_sk
    and store_sales.ss_store_sk = store.s_store_sk
    and store_sales.ss_hdemo_sk = household_demographics.hd_demo_sk
    and date_dim.d_dom between 1 and 2
    and (household_demographics.hd_buy_potential = '>10000' or
         household_demographics.hd_buy_potential = 'unknown')
    and household_demographics.hd_vehicle_count > 0
    and case when household_demographics.hd_vehicle_count > 0 then
             household_demographics.hd_dep_count/ household_demographics.hd_vehicle_count else null end > 1
    and date_dim.d_year in (2000,2000+1,2000+2)
    and store.s_county in ('Mobile County','Maverick County','Huron County','Kittitas County')
    group by ss_ticket_number,ss_customer_sk) dj,customer
    where ss_customer_sk = c_customer_sk
      and cnt between 1 and 5
    order by cnt desc;

-- end query 1 in stream 0 using template query73.tpl
```

- 测试结果
![](https://github.com/buildupchao/ImgStore/blob/master/blog/analyze_table4.png?raw=true)

## **3.结论**
- Hive执行表分析后能大幅加速查询速度
  - 查询耗时（压缩算法）：None > Snappy > Gzip/Zlib
  - 查询耗时（文件格式）：Text > Parquet > Orc

- 当前测试场景下，ORC格式查询耗时最低
  - Parquet与Orc查询耗时接近
