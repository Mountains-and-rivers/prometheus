# prometheus监控主机和mysql
### 环境信息

| 插件            | 版本                          | ip                            |
| --------------- | ----------------------------- | ----------------------------- |
| prometheus      | v2.26.0                       | 192.168.31.240                |
| grafana         | v7.5.5                        | 192.168.31.240                |
| mysqld_exporter | v1.1.2                        | 192.168.31.214                |
| node_exporter   | v0.12.1                       | 192.168.31.214                |
| go              | v1.16.3                       | 192.168.31.240                |
| mysql           | v5.7.34                       | 192.168.31.214                |
| os              | CentOS Linux release 8.3.2011 | 192.168.31.240/192.168.31.214 |

### 概述

prometheus（普罗米修斯）是一套开源的监控&报警&时间序列数据库的组合，起始是由SoundCloud公司开发的。现在最常见的Docker、Mesos、Kubernetes容器管理系统中，通常会搭配Prometheus进行监控。

Prometheus基本原理是通过HTTP协议周期性抓取被监控组件的状态，这样做的好处是任意组件只要提供HTTP接口就可以接入监控系统，不需要任何SDK或者其他的集成过程。这样做非常适合虚拟化环境比如VM或者Docker 。

输出被监控组件信息的HTTP接口被叫做exporter。目前互联网公司常用的组件大部分都有exporter可以直接使用，比如Varnish、Haproxy、Nginx、MySQL、Linux 系统信息 (包括磁盘、内存、CPU、网络等等)，具体支持的源看：https://github.com/prometheus。

图片1

### Prometheus各组件运行流程如下：

1、 Prometheus Server：Prometheus Sever是Prometheus组件中的核心部分，负责实现对监控数据的获取，存储及查询。Prometheus Server可以通过静态配置管理监控目标，也可以配合使用Service Discovery(服务发现)的方式动态管理监控目标，并从这些监控目标中获取数据。其次Prometheus Sever需要对采集到的数据进行存储，Prometheus Server本身就是一个实时数据库，将采集到的监控数据按照时间序列的方式存储在本地磁盘当中。Prometheus Server对外提供了自定义的PromQL，实现对数据的查询以及分析。另外Prometheus Server的联邦集群能力可以使其从其他的Prometheus Server实例中获取数据。

2、Exporters：Exporter将监控数据采集的端点通过HTTP服务的形式暴露给Prometheus Server，Prometheus Server通过访问该Exporter提供的Endpoint端点，即可以获取到需要采集的监控数据。可以将Exporter分为2类：

(1)、直接采集：这一类Exporter直接内置了对Prometheus监控的支持，比如cAdvisor，Kubernetes，Etcd，Gokit等，都直接内置了用于向Prometheus暴露监控数据的端点。

（2）、间接采集：原有监控目标并不直接支持Prometheus，因此需要通过Prometheus提供的Client Library编写该监控目标的监控采集程序。例如：Mysql Exporter，JMX Exporter，Consul Exporter等。 

3、AlertManager：在Prometheus Server中支持基于Prom QL创建告警规则，如果满足Prom QL定义的规则，则会产生一条告警。常见的接收方式有：电子邮件，webhook 等。

4、PushGateway:Prometheus数据采集基于Prometheus Server从Exporter pull数据，因此当网络环境不允许Prometheus Server和Exporter进行通信时，可以使用PushGateway来进行中转。

### Prometheus的工作流：

1.Prometheus server定期从配置好的jobs和exporters中拉取metrics，或者接收来自Pushgateway发送过来的metrics，或者从其它的Prometheus server中拉metrics。

2.Prometheus server在本地存储收集到的metrics，并运行定义好的alerts.rules，记录新的时间序列或者向Alert manager推送警报。

3.Alertmanager根据配置文件，对接收到的警报进行处理，发出告警。

4.在图形界面中，可视化采集数据。

实验拓扑图：

图片2

### 安装mysql

参考：https://github.com/Mountains-and-rivers/mysql-master-slave-ha

```
CREATE USER 'mysql_monitor'@'localhost' IDENTIFIED BY 'MyNewPass4' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'mysql_monitor'@'localhost';
flush privileges;

#test
select * from mysql.user;
drop user 'mysql_monitor'@'localhost';

```



#### 安装插件

##### prometheus  192.168.31.240

```
cat > /usr/lib/systemd/system/prometheus.service << EOF
[Unit]
Description=Prometheus Node Exporter
After=network.target
 
[Service]
ExecStart=/usr/local/prometheus/prometheus --config.file=/usr/local/prometheus/prometheus.yml --web.read-timeout=5m  --web.max-connections=10 --storage.tsdb.retention=15d --storage.tsdb.path=/prometheus/data --query.max-concurrency=20 --query.timeout=2m
User=root
[Install]
WantedBy=multi-user.target
EOF
```

###### 启动

```
systemctl start prometheus.service
systemctl enable prometheus.service
systemctl status prometheus.service
```

### node_exporter

```
 tar zxvf /root/mysqld_exporter-0.12.1.linux-amd64.tar.gz -C /usr/local/
```

启动

```
./node_exporter &
```

### mysqld_exporter

```
 /usr/local/mysqld_exporter/mysqld_exporter --config.my-cnf="/usr/local/mysqld_exporter/.my.cnf" &
```

### grafana 192.168.31.240

安装

```
wget https://dl.grafana.com/oss/release/grafana-7.5.5-1.x86_64.rpm

cat xa* > grafana-7.5.5-1.x86_64.rpm
 
rpm -ivh grafana-7.5.5-1.x86_64.rpm
```

卸载

```
yum autoremove `rpm -qa |grep -i grafana` -y
rm -rf /var/lib/grafana/grafana.db
rm -rf /usr/share/grafana
rm -rf /var/lib/grafan
rm -rf /run/grafana
rm -rf /run/grafana/grafana-server.pid
rm -rf /etc/grafana
rm -rf /var/lib/grafana/grafana.db
rm -rf /var/lib/grafana/grafana.db
```

启动

```
systemctl start grafana-server
```

安装饼图插件

```配置
grafana-cli plugins install grafana-piechart-panel
systemctl restart grafana-server.service
```

配置展示

```
{
  "annotations": {
    "list": [
      {
        "builtIn": 1,
        "datasource": "-- Grafana --",
        "enable": true,
        "hide": true,
        "iconColor": "rgba(0, 211, 255, 1)",
        "name": "Annotations & Alerts",
        "type": "dashboard"
      }
    ]
  },
  "description": "",
  "editable": true,
  "gnetId": 11323,
  "graphTooltip": 1,
  "id": 31,
  "iteration": 1603186191702,
  "links": [],
  "panels": [
    {
      "collapsed": false,
      "datasource": "$datasource",
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 0
      },
      "id": 382,
      "panels": [],
      "repeat": null,
      "title": "",
      "type": "row"
    },
    {
      "cacheTimeout": null,
      "datasource": "$datasource",
      "description": "**Uptime**\n\nThe amount of time since the last restart of the MySQL server process.",
      "fieldConfig": {
        "defaults": {
          "custom": {},
          "decimals": 1,
          "mappings": [],
          "nullValueMode": "connected",
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "rgba(245, 54, 54, 0.9)",
                "value": null
              },
              {
                "color": "rgba(237, 129, 40, 0.89)",
                "value": 300
              },
              {
                "color": "rgba(50, 172, 45, 0.97)",
                "value": 3600
              }
            ]
          },
          "unit": "s"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 3,
        "w": 8,
        "x": 0,
        "y": 1
      },
      "id": 12,
      "interval": "1m",
      "links": [],
      "maxDataPoints": 100,
      "options": {
        "colorMode": "value",
        "fieldOptions": {
          "calcs": [
            "lastNotNull"
          ]
        },
        "graphMode": "none",
        "justifyMode": "auto",
        "orientation": "horizontal",
        "reduceOptions": {
          "calcs": [
            "mean"
          ],
          "fields": "",
          "values": false
        }
      },
      "pluginVersion": "7.0.4",
      "targets": [
        {
          "calculatedInterval": "10m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "mysql_global_status_uptime{job=~\"$job\", instance=~\"$instance\"}",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "{{instance}}",
          "metric": "",
          "refId": "A",
          "step": 300
        }
      ],
      "title": "Uptime",
      "type": "stat"
    },
    {
      "cacheTimeout": null,
      "datasource": "$datasource",
      "description": "**Current QPS**\n\nBased on the queries reported by MySQL's ``SHOW STATUS`` command, it is the number of statements executed by the server within the last second. This variable includes statements executed within stored programs, unlike the Questions variable. It does not count \n``COM_PING`` or ``COM_STATISTICS`` commands.",
      "fieldConfig": {
        "defaults": {
          "custom": {},
          "decimals": 2,
          "mappings": [],
          "nullValueMode": "connected",
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "rgba(245, 54, 54, 0.9)",
                "value": null
              },
              {
                "color": "rgba(237, 129, 40, 0.89)",
                "value": 35
              },
              {
                "color": "rgba(50, 172, 45, 0.97)",
                "value": 75
              }
            ]
          },
          "unit": "short"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 3,
        "w": 8,
        "x": 8,
        "y": 1
      },
      "id": 13,
      "interval": "1m",
      "links": [
        {
          "targetBlank": true,
          "title": "MySQL Server Status Variables",
          "url": "https://dev.mysql.com/doc/refman/5.7/en/server-status-variables.html#statvar_Queries"
        }
      ],
      "maxDataPoints": 100,
      "options": {
        "colorMode": "value",
        "fieldOptions": {
          "calcs": [
            "lastNotNull"
          ]
        },
        "graphMode": "area",
        "justifyMode": "auto",
        "orientation": "horizontal",
        "reduceOptions": {
          "calcs": [
            "mean"
          ],
          "fields": "",
          "values": false
        }
      },
      "pluginVersion": "7.0.4",
      "targets": [
        {
          "calculatedInterval": "10m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "rate(mysql_global_status_queries{job=~\"$job\", instance=~\"$instance\"}[$__interval])",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "{{instance}}",
          "metric": "",
          "refId": "A",
          "step": 20
        }
      ],
      "title": "Current QPS",
      "type": "stat"
    },
    {
      "cacheTimeout": null,
      "datasource": "$datasource",
      "description": "**InnoDB Buffer Pool Size**\n\nInnoDB maintains a storage area called the buffer pool for caching data and indexes in memory.  Knowing how the InnoDB buffer pool works, and taking advantage of it to keep frequently accessed data in memory, is one of the most important aspects of MySQL tuning. The goal is to keep the working set in memory. In most cases, this should be between 60%-90% of available memory on a dedicated database host, but depends on many factors.",
      "fieldConfig": {
        "defaults": {
          "custom": {},
          "decimals": 0,
          "mappings": [],
          "nullValueMode": "connected",
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "rgba(50, 172, 45, 0.97)",
                "value": null
              },
              {
                "color": "rgba(237, 129, 40, 0.89)",
                "value": 90
              },
              {
                "color": "rgba(245, 54, 54, 0.9)",
                "value": 95
              }
            ]
          },
          "unit": "bytes"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 3,
        "w": 8,
        "x": 16,
        "y": 1
      },
      "id": 51,
      "interval": "1m",
      "links": [
        {
          "targetBlank": true,
          "title": "Tuning the InnoDB Buffer Pool Size",
          "url": "https://www.percona.com/blog/2015/06/02/80-ram-tune-innodb_buffer_pool_size/"
        }
      ],
      "maxDataPoints": 100,
      "options": {
        "colorMode": "value",
        "fieldOptions": {
          "calcs": [
            "lastNotNull"
          ]
        },
        "graphMode": "none",
        "justifyMode": "auto",
        "orientation": "horizontal",
        "reduceOptions": {
          "calcs": [
            "mean"
          ],
          "fields": "",
          "values": false
        }
      },
      "pluginVersion": "7.0.4",
      "targets": [
        {
          "calculatedInterval": "10m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "mysql_global_variables_innodb_buffer_pool_size{job=~\"$job\", instance=~\"$instance\"}",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "{{instance}}",
          "metric": "",
          "refId": "A",
          "step": 300
        }
      ],
      "title": "InnoDB Buffer Pool",
      "type": "stat"
    },
    {
      "collapsed": false,
      "datasource": "$datasource",
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 4
      },
      "id": 383,
      "panels": [],
      "repeat": null,
      "title": "Connections",
      "type": "row"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "decimals": 0,
      "description": "**Max Connections** \n\nMax Connections is the maximum permitted number of simultaneous client connections. By default, this is 151. Increasing this value increases the number of file descriptors that mysqld requires. If the required number of descriptors are not available, the server reduces the value of Max Connections.\n\nmysqld actually permits Max Connections + 1 clients to connect. The extra connection is reserved for use by accounts that have the SUPER privilege, such as root.\n\nMax Used Connections is the maximum number of connections that have been in use simultaneously since the server started.\n\nConnections is the number of connection attempts (successful or not) to the MySQL server.",
      "editable": true,
      "error": false,
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 2,
      "fillGradient": 0,
      "grid": {},
      "gridPos": {
        "h": 7,
        "w": 12,
        "x": 0,
        "y": 5
      },
      "height": "250px",
      "hiddenSeries": false,
      "id": 92,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": false,
        "max": true,
        "min": true,
        "show": true,
        "sort": "avg",
        "sortDesc": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 2,
      "links": [
        {
          "targetBlank": true,
          "title": "MySQL Server System Variables",
          "url": "https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_max_connections"
        }
      ],
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [
        {
          "alias": "Max Connections",
          "fill": 0
        }
      ],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "sum(max_over_time(mysql_global_status_threads_connected{job=~\"$job\", instance=~\"$instance\"}[$__interval]))",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Connections",
          "metric": "",
          "refId": "A",
          "step": 20
        },
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "sum(mysql_global_status_max_used_connections{job=~\"$job\", instance=~\"$instance\"})",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Max Used Connections",
          "metric": "",
          "refId": "C",
          "step": 20,
          "target": ""
        },
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "sum(mysql_global_variables_max_connections{job=~\"$job\", instance=~\"$instance\"})",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Max Connections",
          "metric": "",
          "refId": "B",
          "step": 20,
          "target": ""
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "MySQL Connections",
      "tooltip": {
        "msResolution": false,
        "shared": true,
        "sort": 0,
        "value_type": "cumulative"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "label": "",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        },
        {
          "format": "short",
          "label": "",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "decimals": 2,
      "description": "**MySQL Active Threads**\n\nThreads Connected is the number of open connections, while Threads Running is the number of threads not sleeping.",
      "editable": true,
      "error": false,
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 2,
      "fillGradient": 0,
      "grid": {},
      "gridPos": {
        "h": 7,
        "w": 12,
        "x": 12,
        "y": 5
      },
      "hiddenSeries": false,
      "id": 10,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": false,
        "max": true,
        "min": true,
        "rightSide": false,
        "show": true,
        "sort": "avg",
        "sortDesc": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 2,
      "links": [],
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [
        {
          "alias": "Peak Threads Running",
          "color": "#E24D42",
          "lines": false,
          "pointradius": 1,
          "points": true
        },
        {
          "alias": "Peak Threads Connected",
          "color": "#1F78C1"
        },
        {
          "alias": "Avg Threads Running",
          "color": "#EAB839"
        }
      ],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "sum(max_over_time(mysql_global_status_threads_connected{job=~\"$job\", instance=~\"$instance\"}[$__interval]))",
          "format": "time_series",
          "hide": false,
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Peak Threads Connected",
          "metric": "",
          "refId": "A",
          "step": 20
        },
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "sum(max_over_time(mysql_global_status_threads_running{job=~\"$job\", instance=~\"$instance\"}[$__interval]))",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Peak Threads Running",
          "metric": "",
          "refId": "B",
          "step": 20
        },
        {
          "expr": "sum(avg_over_time(mysql_global_status_threads_running{job=~\"$job\", instance=~\"$instance\"}[$__interval]))",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Avg Threads Running",
          "refId": "C",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "MySQL Client Thread Activity",
      "tooltip": {
        "msResolution": false,
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": [
          "total"
        ]
      },
      "yaxes": [
        {
          "format": "short",
          "label": "Threads",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        },
        {
          "format": "short",
          "label": "",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": false
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "collapsed": false,
      "datasource": "$datasource",
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 12
      },
      "id": 384,
      "panels": [],
      "repeat": null,
      "title": "Table Locks",
      "type": "row"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "decimals": 2,
      "description": "**MySQL Questions**\n\nThe number of statements executed by the server. This includes only statements sent to the server by clients and not statements executed within stored programs, unlike the Queries used in the QPS calculation. \n\nThis variable does not count the following commands:\n* ``COM_PING``\n* ``COM_STATISTICS``\n* ``COM_STMT_PREPARE``\n* ``COM_STMT_CLOSE``\n* ``COM_STMT_RESET``",
      "editable": true,
      "error": false,
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 2,
      "fillGradient": 0,
      "grid": {},
      "gridPos": {
        "h": 7,
        "w": 12,
        "x": 0,
        "y": 13
      },
      "hiddenSeries": false,
      "id": 53,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": false,
        "max": true,
        "min": true,
        "rightSide": false,
        "show": true,
        "sort": "avg",
        "sortDesc": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 2,
      "links": [
        {
          "targetBlank": true,
          "title": "MySQL Queries and Questions",
          "url": "https://www.percona.com/blog/2014/05/29/how-mysql-queries-and-questions-are-measured/"
        }
      ],
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "rate(mysql_global_status_questions{job=~\"$job\", instance=~\"$instance\"}[$__interval])",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "{{instance}}",
          "metric": "",
          "refId": "A",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "MySQL Questions",
      "tooltip": {
        "msResolution": false,
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "decimals": 2,
      "description": "**MySQL Thread Cache**\n\nThe thread_cache_size variable sets how many threads the server should cache to reuse. When a client disconnects, the client's threads are put in the cache if the cache is not full. It is autosized in MySQL 5.6.8 and above (capped to 100). Requests for threads are satisfied by reusing threads taken from the cache if possible, and only when the cache is empty is a new thread created.\n\n* *Threads_created*: The number of threads created to handle connections.\n* *Threads_cached*: The number of threads in the thread cache.",
      "editable": true,
      "error": false,
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 2,
      "fillGradient": 0,
      "grid": {},
      "gridPos": {
        "h": 7,
        "w": 12,
        "x": 12,
        "y": 13
      },
      "hiddenSeries": false,
      "id": 11,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": false,
        "max": true,
        "min": true,
        "rightSide": false,
        "show": true,
        "sort": "avg",
        "sortDesc": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 2,
      "links": [
        {
          "title": "Tuning information",
          "url": "https://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_thread_cache_size"
        }
      ],
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [
        {
          "alias": "Threads Created",
          "fill": 0
        }
      ],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "sum(mysql_global_variables_thread_cache_size{job=~\"$job\", instance=~\"$instance\"})",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Thread Cache Size",
          "metric": "",
          "refId": "B",
          "step": 20
        },
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "sum(mysql_global_status_threads_cached{job=~\"$job\", instance=~\"$instance\"})",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Threads Cached",
          "metric": "",
          "refId": "C",
          "step": 20
        },
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "sum(rate(mysql_global_status_threads_created{job=~\"$job\", instance=~\"$instance\"}[$__interval]))",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Threads Created",
          "metric": "",
          "refId": "A",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "MySQL Thread Cache",
      "tooltip": {
        "msResolution": false,
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "collapsed": false,
      "datasource": "$datasource",
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 20
      },
      "id": 385,
      "panels": [],
      "repeat": null,
      "title": "Temporary Objects",
      "type": "row"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "decimals": 2,
      "editable": true,
      "error": false,
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 2,
      "fillGradient": 0,
      "grid": {},
      "gridPos": {
        "h": 7,
        "w": 12,
        "x": 0,
        "y": 21
      },
      "hiddenSeries": false,
      "id": 22,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": false,
        "max": true,
        "min": true,
        "rightSide": false,
        "show": true,
        "sort": "avg",
        "sortDesc": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 2,
      "links": [],
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "sum(rate(mysql_global_status_created_tmp_tables{job=~\"$job\", instance=~\"$instance\"}[$__interval]))",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Created Tmp Tables",
          "metric": "",
          "refId": "A",
          "step": 20
        },
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "sum(rate(mysql_global_status_created_tmp_disk_tables{job=~\"$job\", instance=~\"$instance\"}[$__interval]))",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Created Tmp Disk Tables",
          "metric": "",
          "refId": "B",
          "step": 20
        },
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "sum(rate(mysql_global_status_created_tmp_files{job=~\"$job\", instance=~\"$instance\"}[$__interval]))",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Created Tmp Files",
          "metric": "",
          "refId": "C",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "MySQL Temporary Objects",
      "tooltip": {
        "msResolution": false,
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "decimals": 2,
      "description": "**MySQL Select Types**\n\nAs with most relational databases, selecting based on indexes is more efficient than scanning an entire table's data. Here we see the counters for selects not done with indexes.\n\n* ***Select Scan*** is how many queries caused full table scans, in which all the data in the table had to be read and either discarded or returned.\n* ***Select Range*** is how many queries used a range scan, which means MySQL scanned all rows in a given range.\n* ***Select Full Join*** is the number of joins that are not joined on an index, this is usually a huge performance hit.",
      "editable": true,
      "error": false,
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 2,
      "fillGradient": 0,
      "grid": {},
      "gridPos": {
        "h": 7,
        "w": 12,
        "x": 12,
        "y": 21
      },
      "height": "250px",
      "hiddenSeries": false,
      "id": 311,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": false,
        "hideZero": true,
        "max": true,
        "min": true,
        "rightSide": false,
        "show": true,
        "sort": "avg",
        "sortDesc": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 2,
      "links": [],
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "sum(rate(mysql_global_status_select_full_join{job=~\"$job\", instance=~\"$instance\"}[$__interval]))",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Select Full Join",
          "metric": "",
          "refId": "A",
          "step": 20
        },
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "sum(rate(mysql_global_status_select_full_range_join{job=~\"$job\", instance=~\"$instance\"}[$__interval]))",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Select Full Range Join",
          "metric": "",
          "refId": "B",
          "step": 20
        },
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "sum(rate(mysql_global_status_select_range{job=~\"$job\", instance=~\"$instance\"}[$__interval]))",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Select Range",
          "metric": "",
          "refId": "C",
          "step": 20
        },
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "sum(rate(mysql_global_status_select_range_check{job=~\"$job\", instance=~\"$instance\"}[$__interval]))",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Select Range Check",
          "metric": "",
          "refId": "D",
          "step": 20
        },
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "sum(rate(mysql_global_status_select_scan{job=~\"$job\", instance=~\"$instance\"}[$__interval]))",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Select Scan",
          "metric": "",
          "refId": "E",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "MySQL Select Types",
      "tooltip": {
        "msResolution": false,
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "collapsed": false,
      "datasource": "$datasource",
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 28
      },
      "id": 386,
      "panels": [],
      "repeat": null,
      "title": "Sorts",
      "type": "row"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "decimals": 2,
      "description": "**MySQL Sorts**\n\nDue to a query's structure, order, or other requirements, MySQL sorts the rows before returning them. For example, if a table is ordered 1 to 10 but you want the results reversed, MySQL then has to sort the rows to return 10 to 1.\n\nThis graph also shows when sorts had to scan a whole table or a given range of a table in order to return the results and which could not have been sorted via an index.",
      "editable": true,
      "error": false,
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 2,
      "fillGradient": 0,
      "grid": {},
      "gridPos": {
        "h": 7,
        "w": 12,
        "x": 0,
        "y": 29
      },
      "hiddenSeries": false,
      "id": 30,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": false,
        "hideZero": true,
        "max": true,
        "min": true,
        "rightSide": false,
        "show": true,
        "sort": "avg",
        "sortDesc": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 2,
      "links": [],
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "sum(rate(mysql_global_status_sort_rows{job=~\"$job\", instance=~\"$instance\"}[$__interval]))",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Sort Rows",
          "metric": "",
          "refId": "A",
          "step": 20
        },
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "sum(rate(mysql_global_status_sort_range{job=~\"$job\", instance=~\"$instance\"}[$__interval]))",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Sort Range",
          "metric": "",
          "refId": "B",
          "step": 20
        },
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "sum(rate(mysql_global_status_sort_merge_passes{job=~\"$job\", instance=~\"$instance\"}[$__interval]))",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Sort Merge Passes",
          "metric": "",
          "refId": "C",
          "step": 20
        },
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "sum(rate(mysql_global_status_sort_scan{job=~\"$job\", instance=~\"$instance\"}[$__interval]))",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Sort Scan",
          "metric": "",
          "refId": "D",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "MySQL Sorts",
      "tooltip": {
        "msResolution": false,
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "decimals": 2,
      "description": "**MySQL Slow Queries**\n\nSlow queries are defined as queries being slower than the long_query_time setting. For example, if you have long_query_time set to 3, all queries that take longer than 3 seconds to complete will show on this graph.",
      "editable": true,
      "error": false,
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 2,
      "fillGradient": 0,
      "grid": {},
      "gridPos": {
        "h": 7,
        "w": 12,
        "x": 12,
        "y": 29
      },
      "hiddenSeries": false,
      "id": 48,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": false,
        "max": true,
        "min": true,
        "show": true,
        "sort": "avg",
        "sortDesc": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 2,
      "links": [],
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "sum(rate(mysql_global_status_slow_queries{job=~\"$job\", instance=~\"$instance\"}[$__interval]))",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Slow Queries",
          "metric": "",
          "refId": "A",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "MySQL Slow Queries",
      "tooltip": {
        "msResolution": false,
        "shared": true,
        "sort": 0,
        "value_type": "cumulative"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "label": "",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        },
        {
          "format": "short",
          "label": "",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "collapsed": false,
      "datasource": "$datasource",
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 36
      },
      "id": 387,
      "panels": [],
      "repeat": null,
      "title": "Aborted",
      "type": "row"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "decimals": 2,
      "description": "**Aborted Connections**\n\nWhen a given host connects to MySQL and the connection is interrupted in the middle (for example due to bad credentials), MySQL keeps that info in a system table (since 5.6 this table is exposed in performance_schema).\n\nIf the amount of failed requests without a successful connection reaches the value of max_connect_errors, mysqld assumes that something is wrong and blocks the host from further connection.\n\nTo allow connections from that host again, you need to issue the ``FLUSH HOSTS`` statement.",
      "editable": true,
      "error": false,
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 2,
      "fillGradient": 0,
      "grid": {},
      "gridPos": {
        "h": 7,
        "w": 12,
        "x": 0,
        "y": 37
      },
      "hiddenSeries": false,
      "id": 47,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": false,
        "max": true,
        "min": true,
        "show": true,
        "sort": "avg",
        "sortDesc": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 2,
      "links": [],
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "sum(rate(mysql_global_status_aborted_connects{job=~\"$job\", instance=~\"$instance\"}[$__interval]))",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Aborted Connects (attempts)",
          "metric": "",
          "refId": "A",
          "step": 20
        },
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "sum(rate(mysql_global_status_aborted_clients{job=~\"$job\", instance=~\"$instance\"}[$__interval]))",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Aborted Clients (timeout)",
          "metric": "",
          "refId": "B",
          "step": 20,
          "target": ""
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "MySQL Aborted Connections",
      "tooltip": {
        "msResolution": false,
        "shared": true,
        "sort": 0,
        "value_type": "cumulative"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "label": "",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        },
        {
          "format": "short",
          "label": "",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "decimals": 2,
      "description": "**Table Locks**\n\nMySQL takes a number of different locks for varying reasons. In this graph we see how many Table level locks MySQL has requested from the storage engine. In the case of InnoDB, many times the locks could actually be row locks as it only takes table level locks in a few specific cases.\n\nIt is most useful to compare Locks Immediate and Locks Waited. If Locks waited is rising, it means you have lock contention. Otherwise, Locks Immediate rising and falling is normal activity.",
      "editable": true,
      "error": false,
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 2,
      "fillGradient": 0,
      "grid": {},
      "gridPos": {
        "h": 7,
        "w": 12,
        "x": 12,
        "y": 37
      },
      "hiddenSeries": false,
      "id": 32,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": false,
        "max": true,
        "min": true,
        "rightSide": false,
        "show": true,
        "sort": "avg",
        "sortDesc": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 2,
      "links": [],
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "sum(rate(mysql_global_status_table_locks_immediate{job=~\"$job\", instance=~\"$instance\"}[$__interval]))",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Table Locks Immediate",
          "metric": "",
          "refId": "A",
          "step": 20
        },
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "sum(rate(mysql_global_status_table_locks_waited{job=~\"$job\", instance=~\"$instance\"}[$__interval]))",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Table Locks Waited",
          "metric": "",
          "refId": "B",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "MySQL Table Locks",
      "tooltip": {
        "msResolution": false,
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "collapsed": false,
      "datasource": "$datasource",
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 44
      },
      "id": 388,
      "panels": [],
      "repeat": null,
      "title": "Network",
      "type": "row"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "decimals": 2,
      "description": "**MySQL Network Traffic**\n\nHere we can see how much network traffic is generated by MySQL. Outbound is network traffic sent from MySQL and Inbound is network traffic MySQL has received.",
      "editable": true,
      "error": false,
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 6,
      "fillGradient": 0,
      "grid": {},
      "gridPos": {
        "h": 7,
        "w": 24,
        "x": 0,
        "y": 45
      },
      "hiddenSeries": false,
      "id": 9,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": false,
        "max": true,
        "min": true,
        "rightSide": false,
        "show": true,
        "sort": "avg",
        "sortDesc": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 2,
      "links": [],
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": true,
      "steppedLine": false,
      "targets": [
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "sum(rate(mysql_global_status_bytes_received{job=~\"$job\", instance=~\"$instance\"}[$__interval]))",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Inbound",
          "metric": "",
          "refId": "A",
          "step": 20
        },
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "sum(rate(mysql_global_status_bytes_sent{job=~\"$job\", instance=~\"$instance\"}[$__interval]))",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Outbound",
          "metric": "",
          "refId": "B",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "MySQL Network Traffic",
      "tooltip": {
        "msResolution": false,
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "Bps",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        },
        {
          "format": "none",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "collapsed": false,
      "datasource": "$datasource",
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 52
      },
      "id": 389,
      "panels": [],
      "repeat": null,
      "title": "Memory",
      "type": "row"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "decimals": 0,
      "description": "***System Memory***: Total Memory for the system.\\\n***InnoDB Buffer Pool Data***: InnoDB maintains a storage area called the buffer pool for caching data and indexes in memory.\\\n***TokuDB Cache Size***: Similar in function to the InnoDB Buffer Pool,  TokuDB will allocate 50% of the installed RAM for its own cache.\\\n***Key Buffer Size***: Index blocks for MYISAM tables are buffered and are shared by all threads. key_buffer_size is the size of the buffer used for index blocks.\\\n***Adaptive Hash Index Size***: When InnoDB notices that some index values are being accessed very frequently, it builds a hash index for them in memory on top of B-Tree indexes.\\\n ***Query Cache Size***: The query cache stores the text of a SELECT statement together with the corresponding result that was sent to the client. The query cache has huge scalability problems in that only one thread can do an operation in the query cache at the same time.\\\n***InnoDB Dictionary Size***: The data dictionary is InnoDB ‘s internal catalog of tables. InnoDB stores the data dictionary on disk, and loads entries into memory while the server is running.\\\n***InnoDB Log Buffer Size***: The MySQL InnoDB log buffer allows transactions to run without having to write the log to disk before the transactions commit.",
      "editable": true,
      "error": false,
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 6,
      "fillGradient": 0,
      "grid": {},
      "gridPos": {
        "h": 7,
        "w": 24,
        "x": 0,
        "y": 53
      },
      "hiddenSeries": false,
      "id": 50,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": false,
        "hideEmpty": true,
        "hideZero": true,
        "max": true,
        "min": true,
        "rightSide": true,
        "show": true,
        "sort": "avg",
        "sortDesc": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 2,
      "links": [
        {
          "title": "Detailed descriptions about metrics",
          "url": "https://www.percona.com/doc/percona-monitoring-and-management/dashboard.mysql-overview.html#mysql-internal-memory-overview"
        }
      ],
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [
        {
          "alias": "System Memory",
          "fill": 0,
          "stack": false
        }
      ],
      "spaceLength": 10,
      "stack": true,
      "steppedLine": false,
      "targets": [
        {
          "expr": "sum(mysql_global_status_innodb_page_size{job=~\"$job\", instance=~\"$instance\"} * on (instance) mysql_global_status_buffer_pool_pages{job=~\"$job\", instance=~\"$instance\", state=\"data\"})",
          "format": "time_series",
          "hide": false,
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "InnoDB Buffer Pool Data",
          "refId": "A",
          "step": 20
        },
        {
          "expr": "sum(mysql_global_variables_innodb_log_buffer_size{job=~\"$job\", instance=~\"$instance\"})",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "InnoDB Log Buffer Size",
          "refId": "D",
          "step": 20
        },
        {
          "expr": "sum(mysql_global_variables_innodb_additional_mem_pool_size{job=~\"$job\", instance=~\"$instance\"})",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 2,
          "legendFormat": "InnoDB Additional Memory Pool Size",
          "refId": "H",
          "step": 40
        },
        {
          "expr": "sum(mysql_global_status_innodb_mem_dictionary{job=~\"$job\", instance=~\"$instance\"})",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "InnoDB Dictionary Size",
          "refId": "F",
          "step": 20
        },
        {
          "expr": "sum(mysql_global_variables_key_buffer_size{job=~\"$job\", instance=~\"$instance\"})",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Key Buffer Size",
          "refId": "B",
          "step": 20
        },
        {
          "expr": "sum(mysql_global_variables_query_cache_size{job=~\"$job\", instance=~\"$instance\"})",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Query Cache Size",
          "refId": "C",
          "step": 20
        },
        {
          "expr": "sum(mysql_global_status_innodb_mem_adaptive_hash{job=~\"$job\", instance=~\"$instance\"})",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Adaptive Hash Index Size",
          "refId": "E",
          "step": 20
        },
        {
          "expr": "sum(mysql_global_variables_tokudb_cache_size{job=~\"$job\", instance=~\"$instance\"})",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "TokuDB Cache Size",
          "refId": "I",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "MySQL Internal Memory Overview",
      "tooltip": {
        "msResolution": false,
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "bytes",
          "label": "",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "collapsed": false,
      "datasource": "$datasource",
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 60
      },
      "id": 390,
      "panels": [],
      "repeat": null,
      "title": "Command, Handlers, Processes",
      "type": "row"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "decimals": 2,
      "description": "**Top Command Counters**\n\nThe Com_{{xxx}} statement counter variables indicate the number of times each xxx statement has been executed. There is one status variable for each type of statement. For example, Com_delete and Com_update count [``DELETE``](https://dev.mysql.com/doc/refman/5.7/en/delete.html) and [``UPDATE``](https://dev.mysql.com/doc/refman/5.7/en/update.html) statements, respectively. Com_delete_multi and Com_update_multi are similar but apply to [``DELETE``](https://dev.mysql.com/doc/refman/5.7/en/delete.html) and [``UPDATE``](https://dev.mysql.com/doc/refman/5.7/en/update.html) statements that use multiple-table syntax.",
      "editable": true,
      "error": false,
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 2,
      "fillGradient": 0,
      "grid": {},
      "gridPos": {
        "h": 7,
        "w": 24,
        "x": 0,
        "y": 61
      },
      "hiddenSeries": false,
      "id": 14,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": false,
        "hideEmpty": false,
        "hideZero": false,
        "max": true,
        "min": true,
        "rightSide": true,
        "show": true,
        "sort": "avg",
        "sortDesc": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 2,
      "links": [
        {
          "title": "Server Status Variables (Com_xxx)",
          "url": "https://dev.mysql.com/doc/refman/5.7/en/server-status-variables.html#statvar_Com_xxx"
        }
      ],
      "nullPointMode": "null as zero",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "topk(5, rate(mysql_global_status_commands_total{job=~\"$job\", instance=~\"$instance\"}[$__interval])>0)",
          "format": "time_series",
          "hide": false,
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Com_{{ command }}",
          "metric": "",
          "refId": "B",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "Top Command Counters",
      "tooltip": {
        "msResolution": false,
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "decimals": 2,
      "description": "**MySQL Handlers**\n\nHandler statistics are internal statistics on how MySQL is selecting, updating, inserting, and modifying rows, tables, and indexes.\n\nThis is in fact the layer between the Storage Engine and MySQL.\n\n* `read_rnd_next` is incremented when the server performs a full table scan and this is a counter you don't really want to see with a high value.\n* `read_key` is incremented when a read is done with an index.\n* `read_next` is incremented when the storage engine is asked to 'read the next index entry'. A high value means a lot of index scans are being done.",
      "editable": true,
      "error": false,
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 2,
      "fillGradient": 0,
      "grid": {},
      "gridPos": {
        "h": 7,
        "w": 24,
        "x": 0,
        "y": 68
      },
      "hiddenSeries": false,
      "id": 8,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": false,
        "hideZero": true,
        "max": true,
        "min": true,
        "rightSide": true,
        "show": true,
        "sort": "avg",
        "sortDesc": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 2,
      "links": [],
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "rate(mysql_global_status_handlers_total{instance=~\"$host\", handler!~\"commit|rollback|savepoint.*|prepare\"}[$__interval]) or irate(mysql_global_status_handlers_total{instance=~\"$host\", handler!~\"commit|rollback|savepoint.*|prepare\"}[5m])",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "{{ handler }}",
          "metric": "",
          "refId": "J",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "MySQL Handlers",
      "tooltip": {
        "msResolution": false,
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "decimals": 2,
      "editable": true,
      "error": false,
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 2,
      "fillGradient": 0,
      "grid": {},
      "gridPos": {
        "h": 7,
        "w": 24,
        "x": 0,
        "y": 75
      },
      "hiddenSeries": false,
      "id": 28,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": false,
        "hideZero": true,
        "max": true,
        "min": true,
        "rightSide": true,
        "show": true,
        "sort": "avg",
        "sortDesc": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 2,
      "links": [],
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "rate(mysql_global_status_handlers_total{instance=~\"$host\", handler=~\"commit|rollback|savepoint.*|prepare\"}[$__interval]) or irate(mysql_global_status_handlers_total{instance=~\"$host\", handler=~\"commit|rollback|savepoint.*|prepare\"}[5m])",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "{{ handler }}",
          "metric": "",
          "refId": "A",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "MySQL Transaction Handlers",
      "tooltip": {
        "msResolution": false,
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "decimals": 2,
      "editable": true,
      "error": false,
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 0,
      "fillGradient": 0,
      "grid": {},
      "gridPos": {
        "h": 7,
        "w": 24,
        "x": 0,
        "y": 82
      },
      "hiddenSeries": false,
      "id": 40,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": false,
        "hideZero": true,
        "max": true,
        "min": false,
        "rightSide": true,
        "show": true,
        "sort": "avg",
        "sortDesc": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 2,
      "links": [],
      "nullPointMode": "null as zero",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "mysql_info_schema_threads{job=~\"$job\", instance=~\"$instance\"}",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "{{ state }}",
          "metric": "",
          "refId": "A",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "Process States",
      "tooltip": {
        "msResolution": false,
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": true,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "decimals": 2,
      "editable": true,
      "error": false,
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 6,
      "fillGradient": 0,
      "grid": {},
      "gridPos": {
        "h": 7,
        "w": 24,
        "x": 0,
        "y": 89
      },
      "hiddenSeries": false,
      "id": 49,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": false,
        "hideZero": true,
        "max": true,
        "min": false,
        "rightSide": true,
        "show": true,
        "sort": "avg",
        "sortDesc": true,
        "total": false,
        "values": true
      },
      "lines": false,
      "linewidth": 2,
      "links": [],
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": true,
      "steppedLine": false,
      "targets": [
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "topk(5, avg_over_time(mysql_info_schema_threads{job=~\"$job\", instance=~\"$instance\"}[1h]))",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "{{ state }}",
          "metric": "",
          "refId": "A",
          "step": 3600
        }
      ],
      "thresholds": [],
      "timeFrom": "24h",
      "timeRegions": [],
      "timeShift": null,
      "title": "Top Process States Hourly",
      "tooltip": {
        "msResolution": false,
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "collapsed": false,
      "datasource": "$datasource",
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 96
      },
      "id": 391,
      "panels": [],
      "repeat": null,
      "title": "Query Cache",
      "type": "row"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "decimals": 2,
      "description": "**MySQL Query Cache Memory**\n\nThe query cache has huge scalability problems in that only one thread can do an operation in the query cache at the same time. This serialization is true not only for SELECTs, but also for INSERT/UPDATE/DELETE.\n\nThis also means that the larger the `query_cache_size` is set to, the slower those operations become. In concurrent environments, the MySQL Query Cache quickly becomes a contention point, decreasing performance. MariaDB and AWS Aurora have done work to try and eliminate the query cache contention in their flavors of MySQL, while MySQL 8.0 has eliminated the query cache feature.\n\nThe recommended settings for most environments is to set:\n  ``query_cache_type=0``\n  ``query_cache_size=0``\n\nNote that while you can dynamically change these values, to completely remove the contention point you have to restart the database.",
      "editable": true,
      "error": false,
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 2,
      "fillGradient": 0,
      "grid": {},
      "gridPos": {
        "h": 7,
        "w": 12,
        "x": 0,
        "y": 97
      },
      "hiddenSeries": false,
      "id": 46,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": false,
        "max": true,
        "min": true,
        "rightSide": false,
        "show": true,
        "sort": "avg",
        "sortDesc": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 2,
      "links": [],
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "mysql_global_status_qcache_free_memory{job=~\"$job\", instance=~\"$instance\"}",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Free Memory",
          "metric": "",
          "refId": "F",
          "step": 20
        },
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "mysql_global_variables_query_cache_size{job=~\"$job\", instance=~\"$instance\"}",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Query Cache Size",
          "metric": "",
          "refId": "E",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "MySQL Query Cache Memory",
      "tooltip": {
        "msResolution": false,
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "bytes",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "decimals": 2,
      "description": "**MySQL Query Cache Activity**\n\nThe query cache has huge scalability problems in that only one thread can do an operation in the query cache at the same time. This serialization is true not only for SELECTs, but also for INSERT/UPDATE/DELETE.\n\nThis also means that the larger the `query_cache_size` is set to, the slower those operations become. In concurrent environments, the MySQL Query Cache quickly becomes a contention point, decreasing performance. MariaDB and AWS Aurora have done work to try and eliminate the query cache contention in their flavors of MySQL, while MySQL 8.0 has eliminated the query cache feature.\n\nThe recommended settings for most environments is to set:\n``query_cache_type=0``\n``query_cache_size=0``\n\nNote that while you can dynamically change these values, to completely remove the contention point you have to restart the database.",
      "editable": true,
      "error": false,
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 2,
      "fillGradient": 0,
      "grid": {},
      "gridPos": {
        "h": 7,
        "w": 12,
        "x": 12,
        "y": 97
      },
      "height": "",
      "hiddenSeries": false,
      "id": 45,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": false,
        "max": true,
        "min": true,
        "rightSide": false,
        "show": true,
        "sort": "avg",
        "sortDesc": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 2,
      "links": [],
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "rate(mysql_global_status_qcache_hits{job=~\"$job\", instance=~\"$instance\"}[$__interval]) or irate(mysql_global_status_qcache_hits{job=~\"$job\", instance=~\"$instance\"}[5m])",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Hits",
          "metric": "",
          "refId": "B",
          "step": 20
        },
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "rate(mysql_global_status_qcache_inserts{job=~\"$job\", instance=~\"$instance\"}[$__interval]) or irate(mysql_global_status_qcache_inserts{job=~\"$job\", instance=~\"$instance\"}[5m])",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Inserts",
          "metric": "",
          "refId": "C",
          "step": 20
        },
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "rate(mysql_global_status_qcache_not_cached{job=~\"$job\", instance=~\"$instance\"}[$__interval]) or irate(mysql_global_status_qcache_not_cached{job=~\"$job\", instance=~\"$instance\"}[5m])",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Not Cached",
          "metric": "",
          "refId": "D",
          "step": 20
        },
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "rate(mysql_global_status_qcache_lowmem_prunes{job=~\"$job\", instance=~\"$instance\"}[$__interval]) or irate(mysql_global_status_qcache_lowmem_prunes{job=~\"$job\", instance=~\"$instance\"}[5m])",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Prunes",
          "metric": "",
          "refId": "F",
          "step": 20
        },
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "mysql_global_status_qcache_queries_in_cache{job=~\"$job\", instance=~\"$instance\"}",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Queries in Cache",
          "metric": "",
          "refId": "E",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "MySQL Query Cache Activity",
      "tooltip": {
        "msResolution": false,
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "collapsed": false,
      "datasource": "$datasource",
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 104
      },
      "id": 392,
      "panels": [],
      "repeat": null,
      "title": "Files and Tables",
      "type": "row"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "decimals": 2,
      "editable": true,
      "error": false,
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 2,
      "fillGradient": 0,
      "grid": {},
      "gridPos": {
        "h": 7,
        "w": 12,
        "x": 0,
        "y": 105
      },
      "hiddenSeries": false,
      "id": 43,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": false,
        "max": true,
        "min": true,
        "rightSide": false,
        "show": true,
        "sort": "avg",
        "sortDesc": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 2,
      "links": [],
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "rate(mysql_global_status_opened_files{job=~\"$job\", instance=~\"$instance\"}[$__interval]) or irate(mysql_global_status_opened_files{job=~\"$job\", instance=~\"$instance\"}[5m])",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Openings",
          "metric": "",
          "refId": "A",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "MySQL File Openings",
      "tooltip": {
        "msResolution": false,
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "decimals": 2,
      "editable": true,
      "error": false,
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 2,
      "fillGradient": 0,
      "grid": {},
      "gridPos": {
        "h": 7,
        "w": 12,
        "x": 12,
        "y": 105
      },
      "hiddenSeries": false,
      "id": 41,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": false,
        "max": true,
        "min": true,
        "rightSide": false,
        "show": true,
        "sort": "avg",
        "sortDesc": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 2,
      "links": [],
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "mysql_global_status_open_files{job=~\"$job\", instance=~\"$instance\"}",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Open Files",
          "metric": "",
          "refId": "A",
          "step": 20
        },
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "mysql_global_variables_open_files_limit{job=~\"$job\", instance=~\"$instance\"}",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Open Files Limit",
          "metric": "",
          "refId": "D",
          "step": 20
        },
        {
          "expr": "mysql_global_status_innodb_num_open_files{job=~\"$job\", instance=~\"$instance\"}",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "InnoDB Open Files",
          "refId": "B",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "MySQL Open Files",
      "tooltip": {
        "msResolution": false,
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "collapsed": false,
      "datasource": "$datasource",
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 112
      },
      "id": 393,
      "panels": [],
      "repeat": null,
      "title": "Table Openings",
      "type": "row"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "decimals": 2,
      "description": "**MySQL Table Open Cache Status**\n\nThe recommendation is to set the `table_open_cache_instances` to a loose correlation to virtual CPUs, keeping in mind that more instances means the cache is split more times. If you have a cache set to 500 but it has 10 instances, each cache will only have 50 cached.\n\nThe `table_definition_cache` and `table_open_cache` can be left as default as they are auto-sized MySQL 5.6 and above (ie: do not set them to any value).",
      "editable": true,
      "error": false,
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 2,
      "fillGradient": 0,
      "grid": {},
      "gridPos": {
        "h": 7,
        "w": 12,
        "x": 0,
        "y": 113
      },
      "hiddenSeries": false,
      "id": 44,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": false,
        "max": true,
        "min": true,
        "rightSide": false,
        "show": true,
        "sort": "avg",
        "sortDesc": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 2,
      "links": [
        {
          "title": "Server Status Variables (table_open_cache)",
          "url": "http://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_table_open_cache"
        }
      ],
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [
        {
          "alias": "Table Open Cache Hit Ratio",
          "yaxis": 2
        }
      ],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "rate(mysql_global_status_opened_tables{job=~\"$job\", instance=~\"$instance\"}[$__interval]) or irate(mysql_global_status_opened_tables{job=~\"$job\", instance=~\"$instance\"}[5m])",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Openings",
          "metric": "",
          "refId": "A",
          "step": 20
        },
        {
          "expr": "rate(mysql_global_status_table_open_cache_hits{job=~\"$job\", instance=~\"$instance\"}[$__interval]) or irate(mysql_global_status_table_open_cache_hits{job=~\"$job\", instance=~\"$instance\"}[5m])",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Hits",
          "refId": "B",
          "step": 20
        },
        {
          "expr": "rate(mysql_global_status_table_open_cache_misses{job=~\"$job\", instance=~\"$instance\"}[$__interval]) or irate(mysql_global_status_table_open_cache_misses{job=~\"$job\", instance=~\"$instance\"}[5m])",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Misses",
          "refId": "C",
          "step": 20
        },
        {
          "expr": "rate(mysql_global_status_table_open_cache_overflows{job=~\"$job\", instance=~\"$instance\"}[$__interval]) or irate(mysql_global_status_table_open_cache_overflows{job=~\"$job\", instance=~\"$instance\"}[5m])",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Misses due to Overflows",
          "refId": "D",
          "step": 20
        },
        {
          "expr": "(rate(mysql_global_status_table_open_cache_hits{job=~\"$job\", instance=~\"$instance\"}[$__interval]) or irate(mysql_global_status_table_open_cache_hits{job=~\"$job\", instance=~\"$instance\"}[5m]))/((rate(mysql_global_status_table_open_cache_hits{job=~\"$job\", instance=~\"$instance\"}[$__interval]) or irate(mysql_global_status_table_open_cache_hits{job=~\"$job\", instance=~\"$instance\"}[5m]))+(rate(mysql_global_status_table_open_cache_misses{job=~\"$job\", instance=~\"$instance\"}[$__interval]) or irate(mysql_global_status_table_open_cache_misses{job=~\"$job\", instance=~\"$instance\"}[5m])))",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Table Open Cache Hit Ratio",
          "refId": "E",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "MySQL Table Open Cache Status",
      "tooltip": {
        "msResolution": false,
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        },
        {
          "format": "percentunit",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "decimals": 2,
      "description": "**MySQL Open Tables**\n\nThe recommendation is to set the `table_open_cache_instances` to a loose correlation to virtual CPUs, keeping in mind that more instances means the cache is split more times. If you have a cache set to 500 but it has 10 instances, each cache will only have 50 cached.\n\nThe `table_definition_cache` and `table_open_cache` can be left as default as they are auto-sized MySQL 5.6 and above (ie: do not set them to any value).",
      "editable": true,
      "error": false,
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 2,
      "fillGradient": 0,
      "grid": {},
      "gridPos": {
        "h": 7,
        "w": 12,
        "x": 12,
        "y": 113
      },
      "hiddenSeries": false,
      "id": 42,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": false,
        "max": true,
        "min": true,
        "rightSide": false,
        "show": true,
        "sort": "avg",
        "sortDesc": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 2,
      "links": [
        {
          "title": "Server Status Variables (table_open_cache)",
          "url": "http://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_table_open_cache"
        }
      ],
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "mysql_global_status_open_tables{job=~\"$job\", instance=~\"$instance\"}",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Open Tables",
          "metric": "",
          "refId": "B",
          "step": 20
        },
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "mysql_global_variables_table_open_cache{job=~\"$job\", instance=~\"$instance\"}",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Table Open Cache",
          "metric": "",
          "refId": "C",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "MySQL Open Tables",
      "tooltip": {
        "msResolution": false,
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "collapsed": false,
      "datasource": "$datasource",
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 120
      },
      "id": 394,
      "panels": [],
      "repeat": null,
      "title": "MySQL Table Definition Cache",
      "type": "row"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "decimals": 2,
      "description": "**MySQL Table Definition Cache**\n\nThe recommendation is to set the `table_open_cache_instances` to a loose correlation to virtual CPUs, keeping in mind that more instances means the cache is split more times. If you have a cache set to 500 but it has 10 instances, each cache will only have 50 cached.\n\nThe `table_definition_cache` and `table_open_cache` can be left as default as they are auto-sized MySQL 5.6 and above (ie: do not set them to any value).",
      "editable": true,
      "error": false,
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 2,
      "fillGradient": 0,
      "grid": {},
      "gridPos": {
        "h": 7,
        "w": 24,
        "x": 0,
        "y": 121
      },
      "hiddenSeries": false,
      "id": 54,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": false,
        "max": true,
        "min": true,
        "rightSide": false,
        "show": true,
        "sort": "avg",
        "sortDesc": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 2,
      "links": [
        {
          "title": "Server Status Variables (table_open_cache)",
          "url": "http://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_table_open_cache"
        }
      ],
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [
        {
          "alias": "Opened Table Definitions",
          "yaxis": 2
        }
      ],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "mysql_global_status_open_table_definitions{job=~\"$job\", instance=~\"$instance\"}",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Open Table Definitions",
          "metric": "",
          "refId": "B",
          "step": 20
        },
        {
          "calculatedInterval": "2m",
          "datasourceErrors": {},
          "errors": {},
          "expr": "mysql_global_variables_table_definition_cache{job=~\"$job\", instance=~\"$instance\"}",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Table Definitions Cache Size",
          "metric": "",
          "refId": "C",
          "step": 20
        },
        {
          "expr": "rate(mysql_global_status_opened_table_definitions{job=~\"$job\", instance=~\"$instance\"}[$__interval]) or irate(mysql_global_status_opened_table_definitions{job=~\"$job\", instance=~\"$instance\"}[5m])",
          "format": "time_series",
          "interval": "1m",
          "intervalFactor": 1,
          "legendFormat": "Opened Table Definitions",
          "refId": "A",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "MySQL Table Definition Cache",
      "tooltip": {
        "msResolution": false,
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    }
  ],
  "refresh": "10s",
  "schemaVersion": 25,
  "style": "dark",
  "tags": [],
  "templating": {
    "list": [
      {
        "current": {
          "selected": false,
          "text": "prometheus",
          "value": "prometheus"
        },
        "hide": 0,
        "includeAll": false,
        "label": null,
        "multi": false,
        "name": "datasource",
        "options": [],
        "query": "prometheus",
        "refresh": 1,
        "regex": "",
        "skipUrlSync": false,
        "type": "datasource"
      },
      {
        "allValue": null,
        "current": {
          "selected": true,
          "text": "hosted-grafana/cloudsql-proxy-mysql-exporter",
          "value": [
            "hosted-grafana/cloudsql-proxy-mysql-exporter"
          ]
        },
        "datasource": "$datasource",
        "definition": "label_values(mysql_up, job)",
        "hide": 0,
        "includeAll": true,
        "label": null,
        "multi": true,
        "name": "job",
        "options": [],
        "query": "label_values(mysql_up, job)",
        "refresh": 1,
        "regex": "",
        "skipUrlSync": false,
        "sort": 0,
        "tagValuesQuery": "",
        "tags": [],
        "tagsQuery": "",
        "type": "query",
        "useTags": false
      },
      {
        "allValue": null,
        "current": {
          "selected": true,
          "tags": [],
          "text": "All",
          "value": [
            "$__all"
          ]
        },
        "datasource": "$datasource",
        "definition": "label_values(mysql_up, instance)",
        "hide": 0,
        "includeAll": true,
        "label": null,
        "multi": true,
        "name": "instance",
        "options": [],
        "query": "label_values(mysql_up, instance)",
        "refresh": 1,
        "regex": "",
        "skipUrlSync": false,
        "sort": 0,
        "tagValuesQuery": "",
        "tags": [],
        "tagsQuery": "",
        "type": "query",
        "useTags": false
      }
    ]
  },
  "time": {
    "from": "now-1h",
    "to": "now"
  },
  "timepicker": {
    "collapse": false,
    "enable": true,
    "hidden": false,
    "notice": false,
    "now": true,
    "refresh_intervals": [
      "10s",
      "30s",
      "1m",
      "5m",
      "15m",
      "30m",
      "1h",
      "2h",
      "1d"
    ],
    "status": "Stable",
    "time_options": [
      "5m",
      "15m",
      "1h",
      "6h",
      "12h",
      "24h",
      "2d",
      "7d",
      "30d"
    ],
    "type": "timepicker"
  },
  "timezone": "",
  "title": "MySQL",
  "uid": "549c2bf8936f7767ea6ac47c47b00f2a",
  "version": 1
}
```

验证

图片3

图片4

