## 摸鱼不队 RFC
### 选题背景
Tidb使用过程中，发现如下的等值查询与前缀索引结合的组合索引，未能充分利用前缀索引:

select XXX from objects where bucket_id = '.bucket.meta.bkt20211213' and name >'1' order by bucket_id,name asc limit 5;


组合索引为：KEY \`idx\` (\`bucket_id\`,\`name\`(498))，其中name上为长度498的前缀索引



执行计划如下（只保留与问题描述相关的部分），可以看到的是在进行等值查询获得满足条件的 bucket_id 所有数据之后，算子 IndexRangeScan_19 对这些数据全都进行了扫描即 ".bucket.meta.bucket1" "1",".bucket.meta.bucket1" +inf] ，并没有充分利用前缀索引的特性。
```
*************************** 1. row ***************************
id: Projection_8
estRows: 5.00
actRows: 5
.......
*************************** 2. row ***************************
id: └─TopN_10
estRows: 5.00
actRows: 5
.......
*************************** 3. row ***************************
id: └─IndexLookUp_23
estRows: 5.00
actRows: 9071
.......
*************************** 4. row ***************************
id: ├─IndexRangeScan_19(Build)
estRows: 624973.22
actRows: 661666
task: cop[tikv]
access object: table:objects2, index:idx(bucket_id, name)
execution info: time:208.6ms, loops:653, cop_task: {num: 3, max: 231ms, min:
5.72ms, avg: 97.7ms, p95: 231ms, max_proc_keys: 521960, p95_proc_keys: 521960,
tot_proc: 277ms, rpc_num: 3, rpc_time: 293ms, copr_cache: disabled}, tikv_task:
{proc max:210ms, min:5ms, p80:210ms, p95:210ms, iters:659, tasks:3}
operator info: range:(".bucket.meta.bucket1" "1",".bucket.meta.bucket1" +inf],
keep order:false
memory: N/A
disk: N/A
*************************** 5. row ***************************
id: └─TopN_22(Probe)
estRows: 5.00
actRows: 9071
.......
*************************** 6. row ***************************
id: └─Selection_21
estRows: 624973.22
actRows: 661666
.......
*************************** 7. row ***************************
id: └─TableRowIDScan_20
estRows: 624973.22
actRows: 661666
......
```
上述语句在 .bucket.meta.bucket1（总共661666条记录）和 .bucket.meta.bkt20211213（总共 27995202条记录）下 name > "1" 的，并且name最小的 5 条记录用时分别为：

 | .bucket.meta.bucket1（总共661666条记录）|	.bucket.meta.bkt20211213（总共 27995202条记录）| 
 |:---:|:---:|
 | 5 rows in set (0.49 sec) | 	 5 rows in set (16.35 sec)  |
 
27995202 / 661666 = 40.0881 ，16.35 / 0.49 两者基本呈线性增长的关系，当数据仅为两千多万条时的查询时间为16.35秒，无法满足生产上的需求。

### 目标

基于上述问题的背景，本次2021 Hackthon 摸鱼不队的目标为利用前缀索引的优势，对组合索引进行优化，以满足生产环境的需求。

### 问题讨论

我们希望能生成如下所示的执行计划：
```
TopN_21
└─IndexLookUp
 └─Like_IndexRangeScan(Build)
 └─Selection_20(Probe)
 └─TableRowIDScan
 ```
 
其中的 Like_IndexRangeScan 算子为类似 IndexRangeScan 的行为，所实现的逻辑使用以下例子来说明。

当查询语句为 select * from t where a= "a", b>= "dddd" order by a, b asc limit 5; ，索引为 idx(a, b[3]) 时， Like_IndexRangeScan 算子扫描尽可能多的 rowId ，扫描个数为  cn1 + cn2 + cn3 。

以下面的数据来举例：

```
keyNum                col_a               col_b
1                     a                   ddda
2                     a                   dddf
3                     a                   dddc
4                     a                   e
5                     a                   f
6                     a                   f
7                     a                   g			
8                     a                   hadf
9                     a                   hadc
10                    a                   hadgs
11                    a                   j
12                    a                   k
13                    a                   l
```

最终结果正确的行数是：  

```
keyNum 								col_a 							col_b
2                     a                   dddf
4                     a                   e
5                     a                   f
6                     a                   f
7                     a                   g
9                     a                   hadc
```

 cn1 + cn2 + cn3 三个值： 
 
- cn1: 因为查询条件 b>= "dddd" 是被截断的，那么对于截断的 "ddd"，相同的记录 rowId 要返回，这里是前三条
- cn2: cn1 查询来的记录有可能都满足，有可能都不满足，那么按照都不满足的情况，扫描后面四条 
- cn3: 由于cn1 + cn2 至少有 4 条是满足的，那么期望再扫描一条，由于 8、9、10 三条前缀都是 “had”， 那么把这三条也扫描出来  

 最后返回的 rowId 个数为 3 + 4 + 3 = 10 个，对这 10 个进行 TableRowIDScan。


整个流程就是： 
1. IndexLookUp 返回 10 条 rowId 
2. TableRowIDScan 扫描完整记录 Selection 过滤掉 keyNum 为 1 和 3 的记录，返回 8条记录给 tidb 
3. TopN 选出前 5 条，keyNum 为：2、4、5、6、7 

数据同上，查询条件为 select * from t where a= "a", b>= "e" order by a, b asc limit 5; 

cn1 + cn2 + cn3 三个值： 

- cn1：由于 b>="e" 没有被截断，那么 keyNum 为 4 扫描出来就是满足的，这个记录数为 1. 
- cn2:为 5、6、7的记录三条 
- cn3:目前cn1 + cn2 = 4 条，需要再扫描后面连续的具有相同前缀“had”的三条。 
最后返回的 rowId 个数为 1 + 3 + 3 = 7 个，对这 7 个进行 TableRowIDScan  
