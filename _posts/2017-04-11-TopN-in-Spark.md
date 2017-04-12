---
title: Spark实现TopN 
categories: Spark
---
## 概述
对超过单机内存容量的(K, V)对进行TopN排序，在键值K是否重复的情况   
需要分别处理，因为K重复，我们需要先去重。

## Spark Implementation: Unique Keys 
### 1. 建立和Spark的连接
```
/*create a connection to the Spark master*/
JavaSparkContext ctx = new JavaSparkContext();
```

### 2. 从HDFS读取数据并建立RDD 
```
JavaRDD<String> lines = ctx.textFile(inputPath, 1);
``` 

### 3. 从初始RDD转化为pair RDD
```
 JavaPairRDD<String,Integer> pairs = lines.mapToPair(new PairFunction<String, String, Integer>() {
         @Override
         public Tuple2<String,Integer> call(String s) {
            String[] tokens = s.split(","); // cat7,234
            return new Tuple2<String,Integer>(tokens[0], Integer.parseInt(tokens[1]));
         }
      });
```
使用mapToPair函数用于创建pair RDD. 

### 4. 在每个partition中建立本地TopN
```
      /*
      * <U> JavaRDD<U> mapPartitions(FlatMapFunction<java.util.Iterator<T>,U> f)
      * Return a new RDD by applying a function to each partition of this RDD.
      *
      * mapPartitions(func)	Similar to map, but runs separately on each partition   
      * (block) of the RDD, so func must be of type Iterator<T> => Iterator<U> when   
      * running on an RDD of type T.
      *
      * public interface FlatMapFunction<T,R> extends java.io.Serializable
      * A function that returns zero or more output records from each input record.
      *
      * java.lang.Iterable<R> call(T t) throws java.lang.Exception
      * */
      // STEP-5: create a local top-10
      JavaRDD<SortedMap<Integer, String>> partitions = pairs.mapPartitions(
         new FlatMapFunction<Iterator<Tuple2<String,Integer>>, SortedMap<Integer, String>>() {
             public Iterable<SortedMap<Integer, String>> call(Iterator<Tuple2<String,Integer>> iter) {
             SortedMap<Integer, String> top10 = new TreeMap<Integer, String>();
             while (iter.hasNext()) {
                Tuple2<String,Integer> tuple = iter.next();
                top10.put(tuple._2, tuple._1);
                // keep only top N 
                if (top10.size() > 10) {
                   top10.remove(top10.firstKey());
                }  
             }
             //return Collections.singletonList(top10).iterator();
             return Collections.singletonList(top10);
         }
      });
```
根据《Learning Spark》的4.5节Data Patitioning的论述:
> Spark’s partitioning is available on all RDDs of key/value pairs, and causes the system
to group elements based on a function of each key. Although Spark does not give
explicit control of which worker node each key goes to (partly because the system is
designed to work even if specific nodes fail), it lets the program ensure that a set of
keys will appear together on some node.   

Spark会自动对数据分区，而在分区的基础上mapPartitions进行工作。


### 5. 通过对各个分区的TopN处理，获得最终的TopN
```
  SortedMap<Integer, String> finaltop10 = new TreeMap<Integer, String>();
      List<SortedMap<Integer, String>> alltop10 = partitions.collect();
      for (SortedMap<Integer, String> localtop10 : alltop10) {
          //System.out.println(tuple._1 + ": " + tuple._2);
          //weight/count = tuple._1
          //catname/URL = tuple._2
          for (Map.Entry<Integer, String> entry : localtop10.entrySet()) {
              //System.out.println(entry.getKey() + "--" + entry.getValue());
              finaltop10.put(entry.getKey(), entry.getValue());
              // keep only top 10 
              if (finaltop10.size() > 10) {
                 finaltop10.remove(finaltop10.firstKey());
              }
          }
      }
```
collect()将整个RDD的内容返回，因此其要求所有数据必须能一同放入当台机器的内存中。   
这里也可以使用reduce()函数并行整合RDD中所有数据。

## Spark Implementation: Nonunique Keys 
