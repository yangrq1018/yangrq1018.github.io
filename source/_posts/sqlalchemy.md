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