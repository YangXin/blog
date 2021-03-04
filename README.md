# Prometheus指标的由来
## 1、前言
   我最近在学习Pigsty的过程中，其中关于监控模块中，Grafana展示的指标让人耳目一新，故很好奇在Prometheus中能准确查找到对应指标及如何产生的。
## 2、Promthus的指标由来
   众所得知Prometheus是自带时序数据库存储推送过来的数据，首先我们需要找到/etc/prometheus/prometheus.yml的配置文件，如下：

```  
global:
  scrape_interval: 2s
  evaluation_interval: 2s
  scrape_timeout: 1s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - localhost:9093
      scheme: http
      timeout: 10s
      api_version: v1

rule_files:
  - rules/node.yml
  - rules/pgsql.yml

scrape_configs:

  # meta services
  - job_name: meta
    static_configs:
      - targets:
          - 127.0.0.1:3000    # grafana
          - 127.0.0.1:9090    # prometheus
          - 127.0.0.1:9093    # alertmanager
          - 127.0.0.1:9113    # nginx


  # node_exporter | pg_exporter | pgbouncer_exporter | haproxy exporter
  - job_name: pg
    # https://prometheus.io/docs/prometheus/latest/configuration/configuration/#consul_sd_config
    metrics_path: /metrics
    #使用了consul的注册服务
    consul_sd_configs:
      - server: 127.0.0.1:8500
        refresh_interval: 5s
    #consul过滤标签只保留标签是pg、exporter的标签
        tags:
          - pg
          - exporter
    # https://prometheus.io/docs/prometheus/latest/configuration/configuration/#relabel_config
    # final labels: [cls, svc, ins, role, node, ip, instance]
    relabel_configs:
    #重新修改标签
      # cluster label: pg-test, pg-meta, ...
      - source_labels: [__meta_consul_service_metadata_cluster]
        action: replace
        target_label: cls

      # service label: pg-test-primary, pg-test-replica, ...
      - source_labels: [__meta_consul_service_metadata_service]
        action: replace
        target_label: svc

      # instance label: pg-meta-0, pg-test-1, ...
      - source_labels: [__meta_consul_service_metadata_instance]
        action: replace
        target_label: ins

      # role label      `primary|replica`
      - source_labels: [__meta_consul_service_metadata_role]
        action: replace
        target_label: role

      # node/ip address label
      - source_labels: [__meta_consul_address]
        action: replace
        target_label: ip
```
 
      从中可知在consul中标签为pg、exporter的服务器，会向Prometheus推送数据，通过从consul发现分为两大类：
         一是由node_exporter的操作系统数据;
         二是由pg_exporter的数据库数据;
## 3、Prometheus的指标查询
     本人一开始对Prometheus的指标有哪些是完全不知情的，跟关系型数据库比较大的区别就在于Prometheus是按指标查询的，一个指标可以理解成“一张表”，
     那么问题来了， 我们怎么能提前知道指标名及作用。
     
     首先我们需要理解Prometheus定义了4种不同的指标类型(metric type)：Counter（计数器）、Gauge（仪表盘）、Histogram（直方图）、Summary（摘要）；
     注意这对我们理解指标查询非常关键。
     
     其次我们需要到对应的客户机上的配置文件如pg_exporter的/etc/pg_exporter.yaml中查看，如示例部分：
     
```
#┃  # Here is an example of metrics collector definition
#┃  pg_primary_only:       <---- Collector branch name. Must be UNIQUE among entire configuration
#┃    name: pg             <---- Collector namespace, used as METRIC PREFIX, set to branch name by default, can be over
ride
#┃                               Same namespace may contain multiple collector branch. It's user's responsibility
#┃                               to make sure that AT MOST ONE collector is picked for each namespace.
#┃
#┃    desc: PostgreSQL basic information (on primary)                 <---- Collector description
#┃    query: |                                                        <---- SQL string
#┃
#┃      SELECT extract(EPOCH FROM CURRENT_TIMESTAMP)                  AS timestamp,
#┃             pg_current_wal_lsn() - '0/0'                           AS lsn,
#┃             pg_current_wal_insert_lsn() - '0/0'                    AS insert_lsn,
#┃             pg_current_wal_lsn() - '0/0'                           AS write_lsn,
#┃             pg_current_wal_flush_lsn() - '0/0'                     AS flush_lsn,
#┃             extract(EPOCH FROM now() - pg_postmaster_start_time()) AS uptime,
#┃             extract(EPOCH FROM now() - pg_conf_load_time())        AS conf_reload_time,
#┃             pg_is_in_backup()                                      AS is_in_backup,
#┃             extract(EPOCH FROM now() - pg_backup_start_time())     AS backup_time;
#┃
#┃                                <---- [OPTIONAL] metadata fields, control collector behavior
#┃    ttl: 1                     <---- Cache TTL: in seconds, how long will pg_exporter cache this collector's query re
sult.
#┃    timeout: 0.1                <---- Query Timeout: in seconds, query exceed this limit will be canceled.
#┃    min_version: 100000         <---- minimal supported version, boundary IS included. In server version number forma
t,
#┃    max_version: 130000         <---- maximal supported version, boundary NOT included, In server version number form
at
#┃    fatal: false                <---- Collector marked `fatal` fails, the entire scrape will abort immediately and ma
rked as fail
#┃    skip: false                 <---- Collector marked `skip` will not be installed during planning procedure
#┃
#┃    tags: [cluster, primary]    <---- tags is a list of string, which could be:
#┃                                        * 'cluster' marks this query as cluster level, so it will only execute once f
or same PostgreSQL Server
#┃                                        * 'primary' or 'master'  mark this query can only run on a primary instance (
WILL NOT execute if pg_is_in_recovery())
#┃                                        * 'standby' or 'replica' mark this query can only run on a replica instance (
WILL execute if pg_is_in_recovery())
#┃                                      some special tag prefix have special interpretation:
#┃                                        * 'dbname:<dbname>' means this query will ONLY be executed on database with n
ame '<dbname>'
#┃                                        * 'username:<user>' means this query will only be executed when connect with
user '<user>'
#┃                                        * 'extension:<extname>' means this query will only be executed when extension
'<extname>' is installed
#┃                                        * 'schema:<nspname>' means this query will only by executed when schema '<nsp
name>' exist
#┃                                        * 'not:<negtag>' means this query WILL NOT be executed when exporter is tagge
d with '<negtag>'
#┃                                        * '<tag>' means this query WILL be executed when exporter is tagged with '<ta
g>'
#┃                                           ( <tag> could not be cluster,primary,standby,master,replica,etc...)
#┃
#┃
#┃    metrics:                    <---- List of returned columns, each column must have a `name` and `usage`, `rename`
and `description` are optional
#┃      - timestamp:              <---- Column name, should be exactly same as returned column name
#┃          rename: ts            <---- Alias, optional, alias will be used instead of column name
#┃          usage: GAUGE          <---- Metric type, `usage` could be
#┃                                        * DISCARD: completely ignoring this field
#┃                                        * LABEL:   use columnName=columnValue as a label in metric
#┃                                        * GAUGE:   Mark column as a gauge metric, fullname will be '<query.name>_<col
umn.name>'
#┃                                        * COUNTER: Same as above, except for it is a counter rather than gauge.
#┃
#┃          description: database current timestamp <----- Description of the column, will be used as metric descriptio
n, optional
#┃
#┃      - lsn:
#┃          usage: COUNTER
#┃          description: log sequence number, current write location (on primary)
#┃      - insert_lsn:
#┃          usage: COUNTER
#┃          description: primary only, location of current wal inserting
#┃      - write_lsn:
#┃          usage: COUNTER
#┃          description: primary only, location of current wal writing
#┃      - flush_lsn:
#┃          usage: COUNTER
#┃          description: primary only, location of current wal syncing
#┃      - uptime:
#┃          usage: GAUGE
#┃          description: seconds since postmaster start
#┃      - conf_reload_time:
#┃          usage: GAUGE
#┃          description: seconds since last configuration reload
#┃      - is_in_backup:
#┃          usage: GAUGE
#┃          description: 1 if backup is in progress
#┃      - backup_time:
#┃          usage: Gauge
#┃          description: seconds since current backup start. null if don't have one
#┃
#┃
```
     我们从中看到查询了在主节点查询流复制的SQL语句，其中name为pg ，metrics下 sql列名称为lsn的字段为counter类型，说明数据收集的指标名是pg_lsn{}拼接而成，如果usage为label
  那么为这个name下的指标可以附带条件查询，如假设：
  ```
            -ip  
                usage:label  
                description:ip
           - lsn:
               usage: COUNTER
               description: log sequence number, current write location (on primar)
   ```
  那么指标为pg_lsn{ip},所以监控指标的名称的由来，解决了大多数人无从下手查询的难题。
  
## 4、衍生指标
     这个衍生指标的生成花了我不少时间，终于查找到是如何产生的。情况是这样的，有时候我们会发现一些指标并不在客户端的yaml上存在，但能确确实实能查询到该指标。
答案就在普罗米修斯的rules下配置的，衍生指标是基于监控指标产生的。
```
- name: pgsql-rules
    rules:

      #==============================================================#
      #                        Aliveness                             #
      #==============================================================#
      # TODO: change these to your pg_exporter & pgbouncer_exporter port
      - record: pg_exporter_up
        expr: up{instance=~".*:9185"}

      - record: pgbouncer_exporter_up
        expr: up{instance=~".*:9127"}


      #==============================================================#
      #                        Identity                              #
      #==============================================================#
      - record: pg_is_primary
        expr: 1 - pg_in_recovery
      - record: pg_is_replica
        expr: pg_in_recovery
      - record: pg_status
        expr: (pg_up{} * 2) +  (1 - pg_in_recovery{})
```
  其中pg_exporter_up是由up{instance=~".*:9185"}得来。
## 5、结语
  通过上面的描述，我们知道指标是怎么收集的，指标的产生，指标命名规则，指标查询，指标类型以及衍生指标等信息，加深对于Prometheusd这款优秀软件的了解。
  
  
  
  
  
  
  
  
  
  
  
  
  
  
    
