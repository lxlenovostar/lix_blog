---
title: Spark实现TopN 
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

6. collect all local top 10s and create a final top 10 list
7. output the final top 10 list


