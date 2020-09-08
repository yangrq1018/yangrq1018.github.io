---
title: sqlalchemy
permalink: article/sqlalchemy
toc:
  number: false
  enable: true
date: 2020-09-07 13:16:17
tags: sqlalchemy
categories: coding
---

## SQLAlchemy使用笔记

`SQLAlchemy`是一个Python的通用数据库接口库，支持市面上几乎所有的商用/开源数据库和Python的交互。支持SQL/ORM方式操作数据。`SQLAlchemy`的ORM模型的核心是declarative model class，用户继承`Base`类，自定义property后，每个类对应物理数据库中的一张表。通过`create_engine`方法生产一个`engine` 实例，指定物理数据库的位置和类型之后，`SQLAlchemy`将以Python代码表达的抽象逻辑编译为SQL语句，与数据库沟通执行。

<!-- more -->

下面是两个作为例子的类。`AlphaFactor`类对应Oracle数据库里的`alpha_factor`表，因为Oracle表是case-insensitive的，所以表叫做`ALPHA_FACTOR`，而Python类叫`AlphaFactor`，符合Python的命名规范。

`AlphaFactor`是计算出来的各种信号时间序列，对应证券代码`security_code`，信号名称`factor_name`，和算法版本`algo_version`。`BondDailyPrice`很简单，就是储存历史转债收盘数据（日频率）的全量数据表，每一行对应一只转债在一个交易日的收盘数据。

```python
class AlphaFactor(Base):
    __tablename__ = 'alpha_factor'

    snapshot_date = Column(DateTime, primary_key=True, nullable=False)
    factor_name = Column(VARCHAR(100), primary_key=True, nullable=False)
    value = Column(Float)
    error_code = Column(Integer)
    error_msg = Column(VARCHAR(100))
    security_code = Column(VARCHAR(100), primary_key=True, nullable=False)
    std_err = Column(Float)
    algo_version = Column(VARCHAR(100), primary_key=True, nullable=False)

    __table_args__ = (ForeignKeyConstraint([snapshot_date, security_code],
                                           [BondDailyPrice.timestamp, BondDailyPrice.bond_id]),
                      {})
    quote = relationship("BondDailyPrice", back_populates="alpha")
```

```python
class BondDailyPrice(Base):
    __tablename__ = 'bar_daily_cb'

    timestamp = Column(DateTime, index=True)
    bond_id = Column(VARCHAR(100), primary_key=True, nullable=False, index=True)
    open = Column(NUMBER(38, 3, False))
    high = Column(NUMBER(38, 3, False))
    low = Column(NUMBER(38, 3, False))
    close = Column(NUMBER(38, 3, False))
    pre_close = Column(NUMBER(38, 3, False))
    volume = Column(NUMBER(38, 0, False))
    amt = Column(Float)
    pct_chg = Column(Float)
    last_trade_day = Column(DateTime)
    trade_status = Column(VARCHAR(100))
    unix_ts = Column(Float, primary_key=True, nullable=False)

    alpha = relationship("AlphaFactor", uselist=False)  # One to one
```

`SQLAlchemy`的ORM模型提供方便直观的联表方法。比如，很常见的一个操作是把信号表和转债价格表联起来，每天、每只转债的实际（收盘）价格，和这里作为信号的LSM价格对比。所以，`AlphaFactor` **join** `BondDailyPrice`的公共key一个是时间，一个是转债代码。

`SQLAlchemy`实际上只有一种方法指定composite foreign key，官方[文档](https://docs.sqlalchemy.org/en/13/core/constraints.html?highlight=check#metadata-foreignkeys)中说的很清楚

> It’s important to note that the **ForeignKeyConstraint** is the only way to define a composite foreign key. While we could also have placed individual ForeignKey objects on both the `invoice_item.invoice_id` and` invoice_item.ref_num` columns, SQLAlchemy would not be aware that these two values should be paired together - it would be two individual foreign key constraints instead of a single composite foreign key referencing two columns.

如果在两个Column分别指定ForeignKey，`SQLAlchemy`会认为这是两个独立的ForeignKey限制，而不是一个composite key。Composite key

`ForeignKeyConstraint([snapshot_date, security_code], [BondDailyPrice.timestamp, BondDailyPrice.bond_id])`写在`AlphaFactor`里的`__table_args__`里面，传递给model的Constructor。

## 如何联表

综合来说，`SQLAlchemy`有三种join table的方法

1. 以上面这个`AlphaFactor` join `BondDailyPrice`的例子来说，如果我们在定义里声明了relationship，可以

   ```python
   session.query(AlphaFactor).outerjoin(BondDailyPrice).add_entity(BondDailyPrice).all()
   ```

   因为我们已经告诉了`SQLAlchemy`两个表之间的ForeignKey关系，所以可以直接`outerjoin()`。`add_entity`使最终查询到的是两个表而不是只有`AlphaFactor`一个。

2. 或者

   ```python
   from sqlalchemy import and_
   session.query(AlphaFactor).outerjoin(BondDailyPrice, and_(
   		AlphaFactor.security_code == BondDailyPrice.bond_id,
       AlphaFactor.snapshot_date == BondDailyPrice.timestamp
   )).add_entity(BondDailyPrice).all()
   ```

   如果没有声明relationship，可以在`outerjoin`的时候提供`on` clause。

3. 最原始的方法

   ```python
   session.query(AlphaFactor, BondDailyPrice).filter(
     	and_(
   				AlphaFactor.security_code == BondDailyPrice.bond_id,
       		AlphaFactor.snapshot_date == BondDailyPrice.timestamp
     	)
   ).all()
   ```

   这样实际上是生成两个表的Cartesian product然后筛选结果，这实际上是一个inner join，而上面两个方法都是left outer join，如果对结果需要保留的行有要求的话，第三个就有点问题。

## 用ORM操作数据库

ORM指的是Object-relational mapping，将数据库中的每一行，抽象为一个Python下的对象，column对应为该对象的attribute属性。声明了Model之后，可以通过`session`对象查询。比方说我们有一张`Demo`表，声明如下

```python
class Demo(Base):
    __tablename__ = 'demo'

    id = Column(Integer, primary_key=True)
    key1 = Column(VARCHAR(40))
    key2 = Column(VARCHAR(40))
```

很简单的一张表，`id`列为主键，有两个属性`key1`和`key2`。我们查询任意一行，

```python
demo = session.query(Demo).first()
demo
# <__main__.Demo at 0x7f8639399790>
demo.id
# 1
```

`demo`对象对应的这一行，已经加载在Python的内存里了，识别这一行的identity是`id`属性为1。

添加另外一行

```python
session.add(Demo(id=2, key1=)'hello', 'world')
session.commit()
```

调用`session.commit()`使`SQLAlchemy`向数据库发出SQL语句，persist `id=2`的这一行。

我们可以查看SQL语句，

```
from sqlalchemy import create_engine
engine = create_engine("<YOUR DATABASE URL>", echo=True)
```

然后绑定在这个`engine`上的`session`对象，`commit`时会echo出来具体执行的SQL语句。

`create_engine`方法接受一个database URL，具体如何构建URL参见各个数据库，以Oracle数据库为例，

```python
DB_URL = "oracle://<schema>:<password>@<host>/<instance>"
```

> 除了`sqlalchemy`之外，和Oracle数据库通讯还需要出现`cx_Oracle`包和Oracle的客户端`instant_client`。

我们不仅可以增加和删除，也可以在我们query到已有的行之后更改对象，再让`SQLAlchemy`把我们的更改映射到数据库里。

```python
demo = session.query(Demo).filter(Demo.id == 1)
demo.key1
# 'foo'
demo.key1 = 'I have changed'
session.commit()
```

得到如下的echo

```
2020-09-08 11:19:53,671 INFO sqlalchemy.engine.base.Engine UPDATE demo SET key1=:key1 WHERE demo.id = :demo_id
2020-09-08 11:19:53,671 INFO sqlalchemy.engine.base.Engine {'key1': 'I have changed', 'demo_id': 1}
2020-09-08 11:19:53,672 INFO sqlalchemy.engine.base.Engine COMMIT
```

可以看到，`SQLAlchemy`把我们更改`key1`属性，翻译为一个`UPDATE`语句。

> Caveat:
>
> 当我们delete一行，然后添加一个同样主键的行，再commit。`SQLAlchemy`会自动翻译为一个`UPDATE`语句，而不是一个`DELETE`和一个`INSERT`语句（添加一个不同主键的行是在这个情况）。
>
> ```python
> d1 = session.query(Demo).filter(Demo.id == 1).first()
> session.delete(d1)
> session.add(Demo(id=1, key1='new key1', key2='new key2'))
> session.commit()
> ```
>
> echo如下
>
> ```
> 2020-09-08 11:26:19,162 INFO sqlalchemy.engine.base.Engine BEGIN (implicit)
> 2020-09-08 11:26:19,164 INFO sqlalchemy.engine.base.Engine SELECT demo_id, demo_key1, demo_key2 
> FROM (SELECT demo.id AS demo_id, demo.key1 AS demo_key1, demo.key2 AS demo_key2 
> FROM demo 
> WHERE demo.id = :id_1) 
> WHERE ROWNUM <= :param_1
> 2020-09-08 11:26:19,165 INFO sqlalchemy.engine.base.Engine {'id_1': 1, 'param_1': 1}
> 2020-09-08 11:26:19,168 INFO sqlalchemy.engine.base.Engine UPDATE demo SET key1=:key1, key2=:key2 WHERE demo.id = :demo_id
> 2020-09-08 11:26:19,168 INFO sqlalchemy.engine.base.Engine {'key1': 'foo', 'key2': 'bar', 'demo_id': 1}
> 2020-09-08 11:26:19,169 INFO sqlalchemy.engine.base.Engine COMMIT
> ```
>
> SQLAlchemy仅仅是更换了发生了变化的字段`UPDATE demo SET key1=:key1, key2=:key2 WHERE demo.id = :demo_id`。其他的字段是保持原有的值。这有时候是需要的behavior，但有时候可能会产生意想不到效果，比如说某些字段是有默认值而我们重新插入一行，期望的是这些字段会回复到默认值上。
>
> 解决方法当然有，可以在`session.delete(demo)`之后马上`session.commit()`，但同时我们也没办法保证两个操作是一个transaction了。

---

`SQLAlchemy`支持几乎所有的数据库操作，而且比直接写SQL语句更加灵活直观。

```python
demo = session.query(Demo).filter(Demo.id == 1).first()
demo.key1
# 'foo'
demo.key1 = 'I am not foo'
session.rollback()  # stash uncommitted change
demo.key1
# 'foo'
```

调用`session.rollback()`之后，echo信息

```
2020-09-08 11:31:05,620 INFO sqlalchemy.engine.base.Engine ROLLBACK
2020-09-08 11:31:06,428 INFO sqlalchemy.engine.base.Engine BEGIN (implicit)
2020-09-08 11:31:06,428 INFO sqlalchemy.engine.base.Engine SELECT demo.id AS demo_id, demo.key1 AS demo_key1, demo.key2 AS demo_key2 
FROM demo 
WHERE demo.id = :param_1
2020-09-08 11:31:06,429 INFO sqlalchemy.engine.base.Engine {'param_1': 1}
```

重新查询了`id=`的这一行的值，加载到`demo`对象中。我们对Python对象的更改、删除、增加，只要不调用`commit`，都只存在于Python内。