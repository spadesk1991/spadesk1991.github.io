## 店铺余额

### 系统简介

本系统主要统计所有授权店铺的余额，每天晚上11点50扫描亚马逊数据表 mysql（mws_financial_event_groups）状态（processing_status）为 Open的数据，并获取当前的汇率，将所有的店铺余额转换成人民币后写入mongodb （  financial_event_group）并计算当天的店铺余额总额，写入mongodb（financial_event_group_summary）

### 系统流程图

~~~flow
```flow
st=>start: 每天晚上11点50或点击刷新按钮
op1=>operation: 扫描mysql（mws_financial_event_groups）状态为Open的数据
op2=>operation: 获取汇率，余额转换成人名币，汇总当天余额，入库mongodb
e=>end: 结束
st->op1->op2->e->
```
~~~

### 系统表结构

####  店铺余额时序表 （financial_event_group）

| 字段                        | 类型   | 说明                                                      |
| --------------------------- | ------ | --------------------------------------------------------- |
| _id                         | string | mongo ID                                                  |
| load_date                   | string | 数据插入日期                                              |
| insert_time                 | time   | 数据入库时间                                              |
| uuid                        | string | 原mysql数据库uuid                                         |
| created_at                  | time   | 原mysql数据库创建时间                                     |
| updated_at                  | time   | 原mysql数据库更新时间                                     |
| deleted_at                  | time   | 原mysql数据库删除时间                                     |
| seller_id                   | string | 店铺ID                                                    |
| financial_event_group_id    | string | 原mysql数据库字段financial_event_group_id                 |
| processing_status           | string | 余额结算情况                                              |
| fund_transfer_status        | string | 资金转移状况                                              |
| fund_transfer_date          | time   | 转账时间                                                  |
| trace_id                    | string | 银行转账流水号                                            |
| account_tail                | string | 账户尾号                                                  |
| original_total_amount       | float  | 店铺余额 mysql 原字段original_total_amount取整数后的值    |
| converted_total_amount      | float  | 原mysql数据库字段converted_total_amount                   |
| beginning_balance_amount    | float  | 结算期开始时余额                                          |
| financial_event_group_start | time   | 结算开始时间                                              |
| financial_event_group_end   | time   | 结算结束时间                                              |
| currency_code               | string | 货币代码                                                  |
| ext_hash                    | string | 扩展字段                                                  |
| exchange_rate               | float  | 入库mongo时汇率                                           |
| cny_balance                 | float  | 转换成人民币后的余额(exchange_rate*original_total_amount) |

####  店铺余额统计表（financial_event_group_summary）

| 字段         | 类型   | 说明                                                         |
| ------------ | ------ | ------------------------------------------------------------ |
| _id          | string | mongodb id                                                   |
| load_date    | string | 数据入库日期                                                 |
| insert_time  | string | 数据入库时间                                                 |
| total_amount | float  | 当天店铺总余额 ; 表financial_event_group 对字段cny_balance的求和 |



### 关于刷新按钮使用频率限制

除admin用户外，其他用户每天仅能刷新一次，基于内存来实现，采用redis过期策略设计模式（惰性过期）每次获取用户的访问次数前判断当天是否已经受限，该key是否过期，

#### 流程图

```flow
st=>start: 刷新
cond1=>condition: 判断当前key是否已经过期（当前时间和过期时间比较）
io1=>inputoutput: 访问次数+1
io2=>inputoutput: 访问次数重置为1，过期时间更新为当天23点59分59秒
cond2=>condition: 判断是否达到最大访问次数
e1=>end: 返回可刷新的次数
e2=>end: 错误提示当天可刷新次数为0
st->cond1
cond1(yes)->io2->e1->
cond1(no)->io1->cond2
cond2(yes)->e2->
cond2(no)->e1->
```
