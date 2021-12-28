# GA总结优化

## 1、GA数仓规范

* GA数仓分层一览图

<table>
  <tr align="center">
    <th>层级</th>
    <th>database</th>
    <th>Event</th>
    <th>Profile</th>
  </tr>
  <tr align="center">
    <td>VIEW</td>
    <td>ga_view</td>
    <td>event_{project_id}</td>
    <td>users/devices_{project_id}</td>
  </tr>
  <tr align="center">
    <td >DWS</td>
    <td>ga_dws</td>
    <td></td>
    <td></td>
  </tr>
  <tr align="center">
    <td rowspan="3">DWM</td>
    <td rowspan="3">ga_dwm</td>
    <td>base_report_event_hourly_{project_id}</td>
    <td></td>
  </tr>
  <tr align="center">
    <td>base_report_order_daily_{project_id}</td>
    <td></td>
  </tr>
   <tr align="center">
    <td>groups_{project_id}_{names}</td>
    <td></td>
  </tr>
  <tr align="center">
    <td rowspan="2">DWD</td>
    <td>ga_kudu</td>
    <td>event_{project_id}</td>
    <td>users/devices_{project_id}</td>
  </tr>
  <tr align="center">
    <td>ga_hive</td>
    <td>event_{project_id}</td>
    <td></td>
  </tr>
   <tr align="center">
    <td>DIM</td>
    <td>ga_dim</td>
    <td></td>
    <td></td>
  <tr align="center">
    <td>TMP</td>
    <td>ga_tmp</td>
    <td>event_{project_id}_{day}</td>
    <td></td>
  </tr>
</table>

* 词根命名规范

固定单词含义，统一既定事物的名称

* 表名命名规范

**分层前缀[dwd|dwm|dws|dim|tmp].主题域\_业务域\_粒度\_XXX**

* 注意事项

**DWD、DIM统一接入配置**

**TMP、DWM、DWS、VIEW可以在符合规则前提下自由建表**



## 2、项目接入

### 1、目标

* 一键式初始化
* 减少沟通成本

## 

### 2、数仓层作业分布

#### 1、表初始化

目前为Java文件，工程为 http://gitlab.bi.club/grow-analytics/pugna.git 

执行 com.tuyoo.framework.pugna.utils.InitProject Main方法即可。

涉及组件参数： Impala Kudu Hive  Redis  版本号

#### 2、Flink实时入库工程

目前为Flink Java工程，地址为  http://gitlab.bi.club/grow-analytics/pugna.git 

|                | 测试服                                               | 正式服                                                       |
| -------------- | ---------------------------------------------------- | ------------------------------------------------------------ |
| **项目位置**   | /home/wangzhaoqi/GA_loadLog/flink                    | /home/bi/ga_pugna                                            |
| **配置位置**   | hdfs 中 /flink/flink_config/test.properties          | hdfs 中   /flink/flink_config/pugna.properties               |
| **offset位置** | **默认setStartFromLatest**                           | **默认setStartFromEarliest**                                   重启时必须指定HDFS checkpoint位置，为offset继续消费位置 |
| **开启脚本**   | root 执行 /home/wangzhaoqi/GA_loadLog/flink/start.sh | 需要参数与checkpoint，查明后重启                             |
| **注意事项**   | yarn资源是否够用                                     | offset消费位置，配置文件参数                                 |

#### 3、dwm层基础报表作业

目前为Java工程，地址为 http://gitlab.bi.club/grow-analytics/mirana.git

目前均已部署在测试服/正式服 oozie中。

|              | Event 报表                           | LTV报表                          |
| ------------ | ------------------------------------ | -------------------------------- |
| **运行逻辑** | 每小时拉取最新svn项目，更新Event报表 | 每天拉取最新svn项目，更新LTV报表 |
| **参数要求** | day  hour    impala jdbc地址         | day    impala jdbc地址           |
| **重跑注意** | 运行日期为日志kudu收录时间           | 运行日期为真实更新LTV日期        |
| **注意事项** | 项目更新操作与重跑数据操作           | 项目更新操作与重跑数据操作       |

#### 4、Kudu导入Hive作业

目前为Spark工程，地址为http://gitlab.bi.club/grow-analytics/omni-knight.git

目前均已部署在测试服/正式服 oozie中。

### 5、新项目接入

新项目接入只需要初始化表即可。

Flink实时入库工程，dwm层基础报表作业，Kudu导入Hive作业 都已经部署长期运行





## 3、开发规范



## 4、项目部署



## 6、模型总结

### 1、漏斗分析

> https://www.sensorsdata.cn/manual/funnel.html
>
>  https://www.jianshu.com/p/4c86a2478cca

满足时间间隔 + 有序漏斗  -->   **带滑动时间窗口的最左子序列问题**

filter      group      sort     search   merge



1. 过滤满足时间间隔

```mysql
SELECT date_time
  , DEFAULT.funnel_count(group_concat(s), '0-0', '7', 'dd') AS funnelValue
FROM (
  SELECT '-' AS date_time
    , concat_ws('-', event1.event.user_id, CAST(event1.event_time AS string), CASE 
      WHEN event1.event_id = 't_game_end' THEN '0'
      ELSE '-1'
    END) AS s
  FROM ga_view.event_100000 event
    INNER JOIN ga_view.event_100000 event1
    ON event.event.user_id = event1.event.user_id
      AND DEFAULT.count_datediff(event1.event_time, event.event_time, 'dd') <= 7
      AND event1.day BETWEEN '2019-10-25' AND '2019-11-06'
      AND (event1.event_id = 't_game_end'
        OR event1.event_id = 't_game_end')
  WHERE event.day BETWEEN '2019-10-25' AND '2019-10-31'
    AND event.event_id = 't_game_end'
) t
GROUP BY date_time, user_id
```

假如时间窗口为7天（一天内就不用join了）， 那么需要自身left join， 右表需要 endday + 7 ， 

满足左表第一个 event     和   右表所有    event_time   在7天间隔内。

2. user_id    event_time 作为主键的话，就不用排序
3. 搜索是否满足漏洞模型时， 向前遍历  和  向后遍历  速度优化
4. merge 



### 2、间隔分析

> https://www.sensorsdata.cn/manual/interval.html

初始行为和后续行为  之间   时间间隔      的   人数   总时间和   中位数

多次排序， 归并排序合并



中位数计算：

* 内存够

uid1  间隔数字排序

uid2  间隔数字排序

group_concat()  ====》  维度      间隔数字

=====>  归并排序     计算数组大小 与 排序后

中位数为  arr[数组大小/2-1]

**impala hive udf / hbase协处理器  第一步都是uid有序，再次归并排序即可**

* 内存不够

分桶排序思想

```
1.整数int型，按照32位计算机来说，占4Byte，可以表示4G个不同的值。原始数据总共有10G个数，需要8Byte才能保证能够完全计数。而内存是2G，所以共分成2G/8Byte=250M个不同的组，每组统计4G/250M=16个相邻数的个数。也就是构造一个双字数组(即每一个元素占8Byte)统计计数，数组包含250M个元素，总共占空间8Byte*250M=2G，恰好等于内存2G，即可以全部读入内存。第一个元素统计0-15区间中的数字出现的总个数，第二个元素统计16-31区间中的数字出现的总个数，最后一个元素统计(4G-16)到(4G-1)区间中的数字出现的总个数，这样遍历一遍10G的原始数据，得到这个数组值。

2.定义一个变量sum，初始化为0。从数组第一个元素开始遍历，并把元素值加入到sum。如果加入某个元素的值之前，sum<5G；而加入这个元素的值之后，sum>5G，则说明中位数位于这个元素所对应统计的16个相邻的数之中，并记录下加入这个元素的值之前的sum值(此时sum是小于5G的最大值)。如果这个元素是数组中第m个元素(m从0开始计算)，则对应的这个区间就是[16m,16m+15]。

3.再次定义一个双字数组统计计数，数组包含16个元素，分别统计(16m)到(16m+15)区间中的每一个数字出现的个数，其他数字忽略。这样再次遍历一遍10G的原始数据，得到这个数组值。

4.定义一个变量sum2，sum2的初始值是sum(即上述第二步中记录的小于5G的最大值)。从新数组第一个元素开始遍历，并把元素值加入到sum2。如果加入某个元素的值之前，sum2<5G；而加入这个元素的值之后，sum2>5G，则说明中位数就是这个元素所对应的数字。如果这个元素是新数组中的第n个元素(n从0开始计算)，则对应的数字就是16m+n，这就是这10G个数字中的中位数。

算法过程如上，需要遍历2遍原始数据，即O(2N)，还需要遍历前后2个数组，O(k).总时间复杂度O(2N+k)
```



### 3、留存分析











## 赛马问题

**有36匹马，通过赛马寻找出最快的3匹。跑道可容纳6匹马同时赛跑，请问最快需要几次赛马可以找到最快的3匹马？**

>  一个最简单的方案。36匹马随机分为6组，分别进行赛跑，那么每组的后3名将被淘汰(这些马不可能是最快的)，余下18匹马。将剩余的18匹马再次 分为3组进行赛跑，余下9匹。在最后9匹中随机选择6匹进行赛跑，将最快的3匹马与剩下3匹马进行赛跑，最后胜出的3匹马即为所求。总共赛马次数为 6+3+1+1=11
>
> **让我们回顾一下上述的赛马过程。我们发现，最初的6次赛马之后，剩余的18匹马实际上是局部有序 的，每一组赛马的3匹优胜马都是有序的，很显然上面的做法过于简单，存在冗余操作。我们接下来要做的工作实际上类似于归并排序。那么怎么做可以用最少的赛马次数从18匹马中挑选出最快的3匹呢？  尽可能的淘汰最多的马**
>
> 答案是这样的：将第一组中3匹优胜马按排名令为A1,A2,A3，其中A1最快；同理第二组令为B1,B2,B3；第三组C1,C2,C3；以此类推，直 至F1,F2,F3。取A1,B1,C1,D1,E1,F1进行赛跑，为了方便说名，假设结果为 A1>B1>C1>D1>E1>F1。很显然，D1,E1,F1，将被淘汰，于是D、E、F组的其余马也将被淘汰，因为他 们比这3匹马还慢。再看C组，在剩余有竞争力的9匹马中(分别是A1,A2,A3,B1,B2,B3，C1,C2,C3)，C1最多只能排第三，那么 C2,C3不可能成为最快的3匹马之一，将其淘汰。同理观察B组，可知B3也不具备成为前三甲的可能，淘汰之！现在只剩下 A1,A2,A3,B1,B2,C1共6匹马，再次进行赛马即得到答案。总共赛马次数为6+1+1=8







