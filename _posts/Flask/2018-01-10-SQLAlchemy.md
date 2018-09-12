---
layout: post
title: "SQLAlchemy"
date: 2018-01-10  
description: "flask"
tag: Flask
--- 

# SQLAlchemy,ORM框架

## 介绍

作用: 帮助我们使用类和对象快速实现数据库操作,将类和对象转换成SQL，然后使用数据API执行SQL并获取执行结果。

操作数据库的方式:

1. 原生:

    MySQLdb:py2
    pymysql:py2/pu3
2. ORM框架

    SQLAlchemy

安装: `pip3 install sqlalchemy`
注意:SQLAlchemy本身无法操作数据库，其必须以来pymsql等第三方插件，Dialect用于和数据API进行交流，根据配置文件的不同调用不同的数据库API，从而实现对数据库的操作，如：

```
MySQL-Python
    mysql+mysqldb://<user>:<password>@<host>[:<port>]/<dbname>
    
pymysql
    mysql+pymysql://<username>:<password>@<host>/<dbname>[?<options>]
    
MySQL-Connector
    mysql+mysqlconnector://<user>:<password>@<host>[:<port>]/<dbname>
    
cx_Oracle
    oracle+cx_oracle://user:pass@host:port/dbname[?key=value&key=value...]
    
更多：http://docs.sqlalchemy.org/en/latest/dialects/index.html
```

## 使用

### 执行原生sql语句

```python
Session = sessionmaker(bind=engine)

session = Session()

# 查询
cursor = session.execute('select * from users')
result = cursor.fetchall()

# 添加
cursor = session.execute('insert into users(name) values(:value)',params={"value":'wupeiqi'})
session.commit()
print(cursor.lastrowid)

session.close()
```

### ORM

1. 单表创建
    ```python
    #!/usr/bin/env python
    # -*- coding:utf-8 -*-
    import datetime
    from sqlalchemy import create_engine
    from sqlalchemy.ext.declarative import declarative_base
    from sqlalchemy import Column, Integer, String, Text, ForeignKey, DateTime, UniqueConstraint, Index

    Base = declarative_base()

    class Users(Base):
        __tablename__ = 'users'

        id = Column(Integer, primary_key=True)
        name = Column(String(32), index=True, nullable=False)
        # email = Column(String(32), unique=True)
        # ctime = Column(DateTime, default=datetime.datetime.now)
        # extra = Column(Text, nullable=True)

        __table_args__ = (
            UniqueConstraint('id', 'name', name='uix_id_name') # 联合唯一索引
            Index('ix_id_name', 'name', 'email') # 联合索引
        )

    def init_db():
        """
        根据类创建数据库表
        :return: 
        """
        engine = create_engine(
            "mysql+pymysql://root:123@127.0.0.1:3306/s6?charset=utf8",
            max_overflow=0,  # 超过连接池大小外最多创建的连接
            pool_size=5,  # 连接池大小
            pool_timeout=30,  # 池中没有线程最多等待的时间，否则报错
            pool_recycle=-1  # 多久之后对线程池中的线程进行一次连接的回收（重置）
        )

        Base.metadata.create_all(engine) # 在数据库中创建表


    def drop_db():
        """
        根据类删除数据库表
        :return: 
        """
        engine = create_engine(
            "mysql+pymysql://root:123@127.0.0.1:3306/s6?charset=utf8",
            max_overflow=0,  # 超过连接池大小外最多创建的连接
            pool_size=5,  # 连接池大小
            pool_timeout=30,  # 池中没有线程最多等待的时间，否则报错
            pool_recycle=-1  # 多久之后对线程池中的线程进行一次连接的回收（重置）
        )

        Base.metadata.drop_all(engine) # 在数据库中删除表

    if __name__ == '__main__':
        drop_db()
        init_db()
    ```

2. 创建多个表并包含Fk、M2M关系
    > 一对多示例
    ```python
    class Hobby(Base):
        __tablename__ = 'hobby'
        id = Column(Integer, primary_key=True)
        caption = Column(String(50), default='篮球')


    class Person(Base):
        __tablename__ = 'person'
        nid = Column(Integer, primary_key=True)
        name = Column(String(32), index=True, nullable=True)
        hobby_id = Column(Integer, ForeignKey("hobby.id"))

        # 与生成表结构无关，仅用于查询方便
        # relationship("关联类名", backref='方向查询的名字')
        hobby = relationship("Hobby", backref='pers')
    ```

    > 多对多示例
    ```python
    class Server2Group(Base):
        __tablename__ = 'server2group'
        id = Column(Integer, primary_key=True, autoincrement=True)
        server_id = Column(Integer, ForeignKey('server.id'))
        group_id = Column(Integer, ForeignKey('group.id'))

    class Group(Base):
        __tablename__ = 'group'
        id = Column(Integer, primary_key=True)
        name = Column(String(64), unique=True, nullable=False)

        # 与生成表结构无关，仅用于查询方便
        servers = relationship('Server', secondary='server2group', backref='groups')

    class Server(Base):
        __tablename__ = 'server'

        id = Column(Integer, primary_key=True, autoincrement=True)
        hostname = Column(String(64), unique=True, nullable=False)

    ```

### 数据库操作

```python
#!/usr/bin/env python
# -*- coding:utf-8 -*-
from sqlalchemy.orm import sessionmaker
from sqlalchemy import create_engine
from models import Users
  
engine = create_engine("mysql+pymysql://root:123@127.0.0.1:3306/s6", max_overflow=0, pool_size=5)
Session = sessionmaker(bind=engine)
  
# 每次执行数据库操作时，都需要创建一个session
session = Session()
  
# ############# 执行ORM操作 #############
obj1 = Users(name="alex1")
session.add(obj1)
# 多个操作时，使用session.add_all(obj1,obj2)
  
# 提交事务
session.commit()
# 关闭session
session.close()
```

#### 基本增删改查

```python
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy import Column, Integer, String, ForeignKey, UniqueConstraint, Index
from sqlalchemy.orm import sessionmaker, relationship
from sqlalchemy import create_engine
from sqlalchemy.sql import text

from db import Users, Hosts

engine = create_engine("mysql+pymysql://root:123@127.0.0.1:3306/s6", max_overflow=0, pool_size=5)
Session = sessionmaker(bind=engine)

session = Session()

# 添加
obj1 = Users(name='guodengjian')
session.add(obj1)
session.add_all([
    Users(name='a'),
    Users(name='b'),
    Users(name='c'),
])
session.commit()

# 删除
session.query(Users).filter(Users.id > 2).delete()
session.commit()

# 修改
session.query(Users).filter(Users.id > 0).update({"name" : "099"})
# 字符串拼接
session.query(Users).filter(Users.id > 0).update({Users.name: Users.name + "099"}, synchronize_session=False)
# 数字加减法
session.query(Users).filter(Users.id > 0).update({"age": Users.age + 1}, synchronize_session="evaluate")
session.commit()

# 查询,结果都是对象，需要进行二次取值处理,去掉all可以直接看到SQL语句
r1 = session.query(Users).all()
# label('xx')别名:select name as xx from Users
r2 = session.query(Users.name.label('xx'), Users.age).all()
r3 = session.query(Users).filter(Users.name == "alex").all()
# filter_by内部还是执行的filter
r4 = session.query(Users).filter_by(name='alex').all()
r5 = session.query(Users).filter_by(name='alex').first()
r6 = session.query(Users).filter(text("id<:value and name=:name")).params(value=224, name='fred').order_by(Users.id).all()
r7 = session.query(Users).from_statement(text("SELECT * FROM users where name=:name")).params(name='ed').all()
```

#### 常用操作

```python
# 条件
ret = session.query(Users).filter(Users.id > 1, Users.name == 'eric').all(）
# between...and
ret = session.query(Users).filter(Users.id.between(1, 3), Users.name == 'eric').all()
# in,取反~
ret = session.query(Users).filter(Users.id.in_([1,3,4])).all()
ret = session.query(Users).filter(~Users.id.in_([1,3,4])).all()
# 子查询
session.query(Users).filter(Users.id.in_(session.query(Users.id).filter_by(name='eric'))).all()
# and,or
from sqlalchemy import and_, or_
# 默认为and，可以不写
ret = session.query(Users).filter(and_(Users.id > 3, Users.name == 'eric')).all()
ret = session.query(Users).filter(or_(Users.id < 2, Users.name == 'eric')).all()
ret = session.query(Users).filter(
    or_(
        Users.id < 2,
        and_(Users.name == 'eric', Users.id > 3),
        Users.extra != ""
    )).all()

# 通配符
ret = session.query(Users).filter(Users.name.like('e%')).all()
ret = session.query(Users).filter(~Users.name.like('e%')).all()

# 切片，通过索引取值，可以做分页
ret = session.query(Users)[1:2]

# 排序，desc()从大到小,asc()从小到大
ret = session.query(Users).order_by(Users.name.desc()).all()
# 先按name从大到小排列，如果name同名，则按照id从小到大排列
ret = session.query(Users).order_by(Users.name.desc(), Users.id.asc()).all()

# group_by 分组
from sqlalchemy.sql import func

ret = session.query(Users).group_by(Users.extra).all()

ret = session.query(
    func.max(Users.id), # 根据条件对分组之后的数据聚合
    func.sum(Users.id),
    func.min(Users.id)).group_by(Users.name).all()

# 对聚合后的数据进行二次筛选
ret = session.query(
    func.max(Users.id),
    func.sum(Users.id),
    func.min(Users.id)).group_by(Users.name).having(func.min(Users.id) >2).all()

# 连表
ret = session.query(Users, Favor).filter(Users.id == Favor.nid).all()
ret = session.query(Person).join(Favor).all()
ret = session.query(Person).join(Favor, isouter=True).all()

# 组合，上下连接
“”“
select id,name from q1
UNION
select id,name from q2;
”“”
# 对结果去重
q1 = session.query(Users.name).filter(Users.id > 2)
q2 = session.query(Favor.caption).filter(Favor.nid < 2)
ret = q1.union(q2).all()

#对结果不去重
q1 = session.query(Users.name).filter(Users.id > 2)
q2 = session.query(Favor.caption).filter(Favor.nid < 2)
ret = q1.union_all(q2).all()
```

#### 基于relationship操作ForeignKey

```python
# 表结构
class Hobby(Base):
    __tablename__ = 'hobby'
    id = Column(Integer, primary_key=True)
    caption = Column(String(50), default='篮球')


class Person(Base):
    __tablename__ = 'person'
    nid = Column(Integer, primary_key=True)
    name = Column(String(32), index=True, nullable=True)
    hobby_id = Column(Integer, ForeignKey("hobby.id"))

    # 与生成表结构无关，仅用于查询方便
    hobby = relationship("Hobby", backref='pers')

# 普通添加
session.add_all([
    Hobby(caption='乒乓球'),
    Hobby(caption='羽毛球'),
    Person(name='张三', hobby_id=3),
    Person(name='李四', hobby_id=4),
])

# 基于relationship添加
person = Person(name='张九', hobby=Hobby(caption='姑娘'))
session.add(person)

hb = Hobby(caption='人妖')
hb.pers = [Person(name='文飞'), Person(name='博雅')]
session.add(hb)

session.commit()

# 使用relationship正向查询
v = session.query(Person).first()
print(v.name)
print(v.hobby.caption)

# 使用relationship反向查询
v = session.query(Hobby).first()
print(v.caption)
print(v.pers)
```

#### 基于relationship操作m2m

```python
# 表结构
class Server2Group(Base):
    __tablename__ = 'server2group'
    id = Column(Integer, primary_key=True, autoincrement=True)
    server_id = Column(Integer, ForeignKey('server.id'))
    group_id = Column(Integer, ForeignKey('group.id'))

class Group(Base):
    __tablename__ = 'group'
    id = Column(Integer, primary_key=True)
    name = Column(String(64), unique=True, nullable=False)

    # 与生成表结构无关，仅用于查询方便
    servers = relationship('Server', secondary='server2group', backref='groups')

class Server(Base):
    __tablename__ = 'server'

    id = Column(Integer, primary_key=True, autoincrement=True)
    hostname = Column(String(64), unique=True, nullable=False)

# 普通添加
session.add_all([
    Server(hostname='c1.com'),
    Server(hostname='c2.com'),
    Group(name='A组'),
    Group(name='B组'),
])
session.commit()

s2g = Server2Group(server_id=1, group_id=1)
session.add(s2g)
session.commit()

# 基于relationship添加
gp = Group(name='C组')
gp.servers = [Server(hostname='c3.com'),Server(hostname='c4.com')]
session.add(gp)
session.commit()

ser = Server(hostname='c6.com')
ser.groups = [Group(name='F组'),Group(name='G组')]
session.add(ser)
session.commit()
```

#### 连接数据库的两种方式

```python
# 方式一
Session = sessionmaker(bind=engine)

def task():
    # 去连接池取一个连接
    session = Session()
    ret = session.query()
    # 归还连接
    session.close()

# 方式二：基于scoped_session
from sqlalchemy.orm import scoped_session
# 内部基于threadinglocal,维护每一个线程
session = scoped_session(Session)

def task():
    ret = session.query()
    # 归还连接
    session.close()
```

### 组件

>参考文件flaskContents。