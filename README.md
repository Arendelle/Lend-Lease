# 共享单车租借系统

## 需求分析：

1. 需要储存所有在车棚内储存的单车，每辆单车都有一个独立编号

2. 共享单车有会员制度，只有共享单车的会员用户才可以借车，否则不能借车
3. 每个会员拥有一个余额账户，会员可以充值任意余额，余额可用于缴纳租车押金，支付超时费用
4. 每个余额账户都有一笔小额的信用额度，用于临时抵扣租车费用。信用额度被占用时不能租车
5. 用户可以借车，每人每次可以借一辆（每人可以累计同时借多辆车），但是总数不能超过一个数量限制
6. 每次租车都要预缴押金，押金不能用信用额度抵扣
7. 一辆车只能同时被一个人借用
8. 用户用完单车后需要还车，还车时返还押金。如果超时归还，则根据超时时长计算费用
9. 用户只能还自己借的车，不能还别人借的车
10. 车棚可以补充新的单车，也可以报废旧的单车
11. 如果用户有需要，可以随时加入和退出会员，但是在退出会员前，需要归还所有的已借单车
12. 如果用户丢失或损坏单车，相应的单车报废，且此次租借中止，押金不予返还

## 精化策略：

### Base

定义单车类 `Bicycles` 和人类 `People`，用于描述全体单车和人

定义集合 `Garage`，意为在车棚内的单车

定义集合 `Lent`，意为已经被借出的单车集合

定义一个二元组集合 `Lend_Record`，存储借车人与所借车的记录

定义 Event：`借车`，输入变量 `借车人` 和 `单车`

- 检查单车是否未被租借
- 将单车移出 `Garage` 集合
- 将单车放入 `Lent` 集合

定义 Event：`还车`，输入变量 `还车人` 和 `单车`

- 检查车是否被还车人租借
- 将单车移出 `Lent` 集合
- 将单车放入 `Garage` 集合
  
### Refine 1 ：借车量限制

定义常量 `LEASE_LIMIT`，表示用户最大借车数量

定义集合 `Users`，表示租车系统中的所有用户

增加二元组集合 `User_Lend_Count`，标记某一用户已借车的数量

精化 Event `借车`，只有用户借车数量没有到达 `LEASE_LIMIT`时才允许借车。在借车之后该用户借车数量 `+1`

精化 Event `还车`，还车之后该用户借车数量 `-1`

### Refine 2 ：会员子系统

定义集合 `Membership`，意为加入共享单车会员的用户集合

增加 Event `入会`，用于用户加入会员

增加 Event `退会`，用于用户退出会员

- 只有用户当前借车量为 `0`，才可以退出会员

精化 Event `借车`，只有用户属于会员才允许借车

精化 Event `还车`，只有用户属于会员才允许还车

### Refine 3 ：会员余额子系统

定义二元组集合 `Balance`，意为所有会员的余额账户
 - 余额账户集合内的条目应该在用户入会时创建，并在用户退会时删除
 - 余额可以少量透支（即低于 0 元），但不可以透支超过常量 `CREDIT_THRESHOLD` 的金额

定义常量 `DEPOSIT`，意为租车时需要缴纳的押金，押金直接从余额中扣除

增加 Event `充值`，给指定会员账户增加余额。单次充值不能超过常量 `CHARGE_SINGLE_MAX`

精化 Event `入会`
 - 用户入会时，创建该用户的余额账户

精化 Event `退会`
 - 会员余额为负时，不允许退会
 - 退会时视为结清余额，同时删除该会员的余额账户

精化 Event `借车`
 - 借车前检查余额需要大于押金
 - 押金不能用信用额度抵扣，要保证扣除押金后账户内还有余额

精化 Event `还车`，还车后归还押金

### Refine 4 ：杂项

增加 Event `Vehicle_Add`，用于添加新的单车进入车棚

增加 Event `Vehicle_Discard`，用于报废单车退出车棚
- 报废单车在报废时不能处于被租用状态

增加 Event `Vehicle_Abnormal`，当单车遇到异常情况（如丢失、损毁等），报废对应单车，解除该单车的租用，并对租车账户扣除罚款

增加常量 `DURATION_FREE`，意为免费借车的时长

增加常量 `CHARGING_RATE`，意为超时租用后单位时间收取的费用

精化 `LEASE_LIMIT`，添加借车时的时间戳

精化 `还车`，增加还车时间戳参数，当借用时间超出免费借用时长时，计算超时费用，并扣除相应余额。余额不足部分使用信用额度抵扣，信用额度如果依然不足，则不允许还车（需要用户先执行充值操作）