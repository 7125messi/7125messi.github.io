---
title: "Spark_Hive_RDBMS读写操作"
date: 2020-09-10T21:58:37+08:00
draft: true
---

[项目总结提炼]

前面我们在做数据工程化过程中会大量用到数据的读写操作,先总结如下！！！

主要有以下几个文件：
```
config.ini                         配置文件

func_forecast.sh                   工程执行文件

mysql-connector-java-8.0.11.jar    MySQL连接jdbc驱动

predict_pred.py                    执行主代码

utils.py                           工具项代码
```

# config.ini

主要定义参数值

```ini
[spark]
executor_memory = 2g
driver_memory = 2g
sql_execution_arrow_enabled = true
executor_cores = 2
executor_instances = 2
jar = /root/spark-rdbms/mysql-connector-java-8.0.11.jar

[mysql]
host = 11.23.32.16
port = 3306
db = test
user = test
password = 123456

[postgresql]
host = 11.23.32.16
port = 3306
db = test
user = test
password = 123456
```

# utils.py

主要定义各个函数

```python
from configparser import ConfigParser
from pyspark import SparkConf
import psycopg2
import pymysql

# pip install pymysql
# pip install psycopg2-binary


def get_config():
    """
    获取整个配置文件信息
    
    Returns
    -------
        cf: ConfigParser
            配置文件信息
    """
    cf = ConfigParser()
    cf.read('config.ini', encoding='utf-8')
    return cf

def get_user_pwd():
    """
    获取用户、密码
    
    Returns
    -------
        strUser: Str
            连接用户
        strPassword: Str
            连接密码
    """
    cf = get_config()
    
    strUser = cf.get('mysql', 'user')
    strPassword = cf.get('mysql', 'password')
    
    return strUser, strPassword

def get_conn_url():
    """
    通过jdbc方式获取连接URL
    
    Returns
    -------
        strUrl: Str
            连接URL
    """
    cf = get_config()
    
    host = cf.get('mysql', 'host')
    port = cf.get('mysql', 'port')
    db = cf.get('mysql', 'db')
    strUrl = f'mysql://{host}:{port}/{db}?useSSL=false'
    return strUrl

def get_conn_properties():
    """
    获取连接用户名+密码字典
    
    Returns
    -------
        strProperties: Dict
            用户、密码组成的字典
    """
    cf = get_config()
    
    user = cf.get('mysql', 'user')
    password = cf.get('mysql', 'password')
    strProperties = {'user': user, 'password': password}
    
    return strProperties

def get_spark_conf():
    """
    获取SparkConf
    
    Returns
    -------
        sparkConf: pyspark.sql.SparkConf
            spark配置参数
    """
    cf = get_config()
    sparkConf = SparkConf()
    sparkConf.set("spark.executor.memoryOverhead", "1g")
    sparkConf.set("spark.driver.maxResultSize", "4g")
    sparkConf.set('spark.some.config.option', 'some-value')
    sparkConf.set('spark.executor.memory', cf.get('spark', 'executor_memory'))
    sparkConf.set('spark.driver.memory', cf.get('spark', 'driver_memory'))
    sparkConf.set('spark.executor.instances', cf.get('spark', 'executor_instances'))
    sparkConf.set('spark.executor.cores', cf.get('spark', 'executor_cores'))
    sparkConf.set('spark.sql.execution.arrow.enabled', cf.get('spark', 'sql_execution_arrow_enabled'))
    
    return sparkConf

def read_dataset(spark, strURL, strDBTable, strUser, strPassword, isConcurrent,
                 partitionColumn='', lowerBound='', upperBound='', numPartitions=1):
    """
    Spark读取数据 jdbc
    
    Parameters
    ----------
        spark: pyspark.sql.SparkSession
            SparkSession
        strURL: Str
            连接URL
        strDBTable: Str
            表名
        strUser: Str
            用户
        strPassword: Str
            密码
        isConcurrent: Bool
            是否并发读取（默认并发为1）
        partitionColumn: Str
            Must be a numeric, date, or timestamp column from the table.
        lowerBound: Str
            Used to decide the partition stride.
        upperBound: Str
            Used to decide the partition stride.
        numPartitions: Str
            The maximum number of partitions that can be used for parallelism
    Returns
    -------
        pyspark.sql.DataFrame
    """
    if isConcurrent:
        jdbcDf=spark.read.format("jdbc") \
                         .option('url', f"jdbc:{strURL}") \
                         .option('dbtable', strDBTable) \
                         .option('user', strUser) \
                         .option('password', strPassword) \
                         .option('partitionColumn', partitionColumn) \
                         .option('lowerBound', lowerBound) \
                         .option('upperBound', upperBound) \
                         .option('numPartitions', numPartitions) \
                         .load()
        return jdbcDf
    else:
        jdbcDf=spark.read.format("jdbc") \
                         .option('url', f"jdbc:{strURL}") \
                         .option('dbtable', strDBTable) \
                         .option('user', strUser) \
                         .option('password', strPassword) \
                         .load()
        return jdbcDf

def write_dataset(srcDF, strURL, desDBTable, dictProperties, mode="append"):
    """
    Spark写入数据 jdbc
    
    Parameters
    ----------
        srcDF: pyspark.sql.DataFrame
            待写入源DataFrame
        strURL: Str
            连接URL
        desDBTable: Str
            写入数据库的表名
        dictProperties: Dict
            用户密码组成的字典
        mode: Str
            插入模式（"append": 增量更新； "overwrite": 全量更新）

    """
    try:
        srcDF.write.jdbc(
            f'jdbc:{strURL}', 
            table=desDBTable, 
            mode=mode, 
            properties=dictProperties
        )
    except BaseException as e:
        print(repr(e))

def psycopg_execute(strSql):
    """
    使用psycopg2方式执行sql
    
    Parameters
    ----------
        strSql: Str
            待执行的sql
    """
    cf = get_config()
    
    # 连接数据库
    conn = psycopg2.connect(
        dbname=cf.get('postgresql', 'db'),
        user=cf.get('postgresql', 'user'),
        password=cf.get('postgresql', 'password'), 
        host=cf.get('postgresql', 'host'), 
        port=cf.get('postgresql', 'port')
    )

    # 创建cursor以访问数据库
    cur = conn.cursor()

    try:
        # 执行操作
        cur.execute(strSql)
        # 提交事务
        conn.commit()
    except psycopg2.Error as e:
        print('DML操作异常')
        print(e)
        # 有异常回滚事务
        conn.rollback()
    finally:
        # 关闭连接
        cur.close()
        conn.close()

def get_sql_conn():
    """
    连接sql数据库的engine：可用于使用数据库连接
    
    Parameters
    ----------
        conn: Str
            数据库的engine
    """
    cf = get_config()
    conn = pymysql.connect(
        host=cf.get('mysql', 'host'), 
        port=int(cf.get('mysql', 'port')),
        user=cf.get('mysql', 'user'),
        password=cf.get('mysql', 'password'),
        database=cf.get('mysql', 'db'),
        charset='utf8'   # charset="utf8"，编码不要写成"utf-8"
    )
    return conn

def mysql_query_data(strSql):
    """
    使用mysql查询
    
    Parameters
    ----------
        strSql: Str
            待执行的sql

    Returns
    -------
        list[dict]
    """    
    # 连接数据库
    conn = get_sql_conn()

    # 创建cursor以访问数据库
    cur = conn.cursor(pymysql.cursors.DictCursor)

    try:
        # 执行操作
        cur.execute(strSql)
        return cur.fetchall()
    finally:
        # 关闭连接
        cur.close()
        conn.close()

def mysql_insert_or_update_data(strSql):
    """
    使用mysql执行insert或update操作
    
    Parameters
    ----------
        strSql: Str
            待执行的sql
    """    
    # 连接数据库
    conn = get_sql_conn()

    # 创建cursor以访问数据库
    cur = conn.cursor()

    try:
        # 执行操作
        cur.execute(strSql)
        # 提交事务
        conn.commit()
    except Exception:
        print('DML操作异常')
        # 有异常回滚事务
        conn.rollback()
    finally:
        # 关闭连接
        cur.close()
        conn.close()
```

# 测试代码 predict_pred.py

```python
from pyspark.sql import SparkSession
from pyspark.sql import functions as fn
from pyspark.ml import Pipeline, util
from pyspark.ml import feature as ft

import pandas as pd
import time

import sys
print(sys.path)
sys.path.append('/root/spark-rdbms') #把模块目录加到sys.path列表中
print(sys.path)

from utils import get_config
from utils import get_user_pwd
from utils import get_conn_url
from utils import get_conn_properties
from utils import get_spark_conf
from utils import read_dataset
from utils import write_dataset
from utils import get_sql_conn
from utils import mysql_query_data
from utils import mysql_insert_or_update_data


if __name__ == '__main__':
    # (1) spark读取hive
    # 启动spark
    spark = SparkSession.\
        builder.\
        appName("spark_io").\
        config(conf=get_spark_conf()).\
        enableHiveSupport().\
        getOrCreate()
    
    select_dt = "2020-09-09"
    strSql = f"""
        select * from yth_src.kclb where dt = '{select_dt}'
    """
    df = spark.sql(strSql)
    print(df.show())


    # (2) spark写入hive
    # dt = time.strftime("%Y-%m-%d", time.localtime()) 
    # print(dt)
    # # 打开动态分区
    # spark.sql("set hive.exec.dynamic.partition.mode=nonstrict")
    # spark.sql("set hive.exec.dynamic.partition=true")
    # spark.sql(f"""
    # insert overwrite table yth_dw.shop_inv_turnover_rate partition (dt)
    # select 
    #     shop_code, 
    #     pro_code,

    #     current_timestamp as created_time, 
    #     dt = '{dt}'
    # from shop_inv_turnover_rate_table_db
    # """)

    # (2) spark读取MySQL
    # 获取用户密码
    strUser, strPwd = get_user_pwd()
    jdbcDf = read_dataset(
        spark = spark, 
        strURL = get_conn_url(), 
        strDBTable = 'hive_kclb', 
        strUser = strUser, 
        strPassword = strPwd,
        isConcurrent = False
    )
    print(jdbcDf.show())

    # (2) spark写入mysql
    # write_dataset(
    #     srcDF = jdbcDf,
    #     strURL = get_conn_url(),
    #     table='hive_kclb2', 
    #     mode='append', 
    #     properties=get_conn_properties()
    # )

    # (3) pymysql读取MySQL
    strSql2 = f"""
        select * from hive_kclb limit 5
    """
    df2 = mysql_query_data(strSql2)   # 返回 list[dict]
    df2 = pd.DataFrame(df2)
    print(df2)

    # (3) pymysql读取MySQL————pandas读取
    engine = get_sql_conn()
    df3 = pd.read_sql(strSql2,con = engine)
    print(df3)
```

# 执行代码 func_forecast.sh

```shell
# !/bin/bash

# spark解释器
spark_interpreter="/opt/spark-2.4.4/bin/spark-submit"

# python解释器
python_interpreter="/app/anaconda3/bin/python"

function func_forecast()
{
    cd /root/spark-rdbms

    # (1) 
    # --packages  jar包的maven地址
    # --packages  mysql:mysql-connector-java:5.1.27 --repositories http://maven.aliyun.com/nexus/content/groups/public/
    #                                               --repositories 为mysql-connector-java包的maven地址，若不给定，则会使用该机器安装的maven默认源中下载
    # 若依赖多个包，则重复上述jar包写法，中间以逗号分隔
    # 默认下载的包位于当前用户根目录下的.ivy/jars文件夹中
    # 应用场景：本地可以没有，集群中服务需要该包的的时候，都是从给定的maven地址，直接下载

    # (2)
    # --jars JARS
    # 应用场景：要求本地必须要有对应的jar文件
    # --driver-class-path 作用于driver的额外类路径，使用–jar时会自动添加路径,多个包之间用冒号(:)分割

    # 执行程序
    ${spark_interpreter} --master yarn \
                        --conf spark.pyspark.python=${python_interpreter} \
                        --driver-class-path mysql-connector-java-8.0.11.jar \
                        --jars mysql-connector-java-8.0.11.jar predict_pred.py

    if [[ $? -ne 0 ]]; then
        echo "--> 执行失败"
    exit 1
    fi
}
func_forecast
```
