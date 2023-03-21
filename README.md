The Stream Inequality join operation is an important tool for real-time streaming applications that require continuous analysis of data with low latency and high processing throughput. This operation is used to join two or more streams of data based on a predefined inequality condition. It allows applications to combine streams of data in real-time and extract meaningful insights from the resulting data.

Another advantage of the Stream Inequality join operation is that it allows applications to combine data from multiple sources, such as social media feeds, sensor data, and financial data, in real-time. This enables applications to gain a more comprehensive understanding of the data and to make more informed decisions based on the results. For example, a smart grid management application might combine data from multiple sensors to detect power outages and predict future demand, while a financial trend indicator application might combine data from multiple financial markets to identify trends and predict future market movements.

Indexing is a technique used in database management systems to speed up the search and retrieval of data. In the context of inequality join, indexing is important because it can significantly improve the performance of the join operation. Inequality join is a type of join operation in which two tables are joined based on a non-equality condition, such as less than or greater than. This type of join can be computationally expensive, especially when dealing with large tables. Indexing can help to reduce the computational complexity of the join operation by creating a data structure that allows for fast lookup and retrieval of rows that satisfy the join condition.

# Solutions
We implement 5 state-of-the-art schemes for stream Inequality join algorithm in Apache Storm DSPS That includes
1. Broad cast hash join
2. Split Join
3. Chain index with check pointing
4. Multi-chain index 
5. Partitioned In-Memory merge tree
# How to run the code
Initially download the .rar file from github and import it into your local Intellij idea.
Run the kafka and Storm cluster For details of kafka and storm cluster follow the links. Apache Kafka [Kafka](https://kafka.apache.org/) and Apache Storm [Storm](https://storm.apache.org/)
Our code contain package choose "StreamWindowJoin", It has following folder that contains implement of following inequality join solution: 
1. BCHJ
2. ChainIndex
3. ChainIndexBroadCaster
4. SplitJoin
5. PartitionedInMemoryJoin
6. StreamAwareJoin
and Test File that contain Main class. In Main class set your kafka and Storm configuration. For example;

```
Config config =new Config();
config.setNumWorkers(12);
config.setFallBackOnJavaSerialization(true);
config.put( Config.TOPOLOGY_SKIP_MISSING_KRYO_REGISTRATIONS,false);
config.put(Config.TOPOLOGY_STATE_CHECKPOINT_INTERVAL,1000 );
config.registerSerialization(org.apache.storm.spout.CheckPointState.class);
config.registerSerialization(com.StreamWindowJoin.SplitJoin.RedBlackBST.class);
config.registerSerialization(com.StreamWindowJoin.PartitionedInMemoryJoin.BPlusTree.class);
long timeOfWindowArchive=10000;
long durationForDeletionOfTree=50000;
TopologyBuilder builder= new TopologyBuilder();
```
However, the example for running the inequality algorithm you should define the kafka spout and bolts carefully and choose the grouping strategy carefully, 
Here I provide you the example details for single self join algorithm for all Algorithm
1. Broad cast hash join
To Run BCHJ: The first `KafkaSpout` is same for all:
```
      builder.setSpout("KafkaSpout", new org.apache.storm.kafka.spout.KafkaSpout<>(kafkaSpoutConfig));
      builder.setBolt("ZipF", new KafkaBolt()).fieldsGrouping("KafkaSpout", new Fields("Tuple")).setNumTasks(5);
      builder.setBolt("BCHJ", new WindowJoinBolt("StreamR","StreamS","Tuple","Tuple",durationForDeletionOfTree,">="), 10).allGrouping("ZipF","StreamR").fieldsGrouping("ZipF","StreamS", new Fields("Tuple")).setNumTasks(50);
      builder.setBolt("RecordBCHJ", new BCHJReceiver()).allGrouping("BCHJ");
```
2. Split Join:
```
builder.setBolt("SplitJoin", new SplitJoinWindowBolt("StreamR","StreamS","Tuple","Tuple",">=",durationForDeletionOfTree),10).shuffleGrouping ("ZipF","StreamR").allGrouping("ZipF","StreamS").setNumTasks(40);
builder.setBolt("RecordSplitJoin", new SplitJoinReceiver()).allGrouping("SplitJoin");
```
3. Chain Index
```
builder.setBolt("Chained", new ChainIndexWindowJoinBolt("StreamR","StreamS","Tuple","Tuple",1000,40000,">="),10).
fieldsGrouping("ZipF","StreamR", new Fields("Tuple")).
fieldsGrouping("ZipF","StreamS", new Fields("Tuple")).setNumTasks(40);  builder.setBolt("RecordChainedJoin", new ChainedReceiver()).allGrouping("Chained");
```
4. ChainIndexBroadcaster:
```
builder.setBolt("MultiChainedIndex", new JoinForChainIndexBoltTest<>("StreamR","StreamS","Tuple","Tuple"),10).fieldsGrouping("ZipF","StreamR", new Fields("Tuple")). allGrouping("ZipF","StreamS").setNumTasks(40);
builder.setBolt("RecordMultiChainJoin", new MultiChainIndexReceiver()).allGrouping("MultiChainedIndex");
```
5. Partitioned InMemory Join:
```
builder.setBolt("PIMTree", new PIMTree("StreamR","StreamS","Tuple","Tuple",4,1000,40000),10). fieldsGrouping("ZipF","StreamR", new Fields("Tuple")). fieldsGrouping("ZipF","StreamS", new Fields("Tuple")).setNumTasks(40);
builder.setBolt("PIMReceiver", new PIMReceiver()).allGrouping("PIMTree");
```
6. Stream-Aware Join :
```
builder.setBolt("StreamPredictor", new AugmentedSketchForSelfJoin<>("StreamR","Tuple")).fieldsGrouping("ZipF","StreamR",new Fields("Tuple"));
builder.setBolt("StreamAwareJoiner",  new StreamAwareJoinBolt("StreamR","StreamS","Tuple","Tuple","Swapped_Status",4,timeOfWindowArchive,durationForDeletionOfTree,">="),10).customGrouping("StreamPredictor",new StreamPartitioningForSelfJoin()).
customGrouping("ZipF","StreamS", new StreamPartitionerForSingleStream()).setNumTasks(40);
 builder.setBolt("StreamAwareJoinerReceiver", new StreamAwareJoinerReceiver()).allGrouping("StreamAwareJoiner");
```
All `ReceiverBolts` for gathering the join results.

# Contact
Please feel free to contact me via email if you encounter any issues while using this project. My email address is <a href="adeel.aslam@unimore.it">adeel.aslam@unimore.it</a>
