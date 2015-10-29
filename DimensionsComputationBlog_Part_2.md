Dimensions Computation (Aggregate Navigator) Part 2: Implementation
===================================================================

### Overview

While the theory of how to compute the aggregations is correct, some more work is required to provide a scalable implementation of Dimensions Computation. As can be seen from the formulas provided in the sections above, the number of aggregations to maintain grows rapidly as the number of unique key values, aggregators, dimension combinations, and time buckets grows. Additionally, a scalable implementation of Dimensions Computation must be capable of handling hundreds of thousands of events per second. In order to achieve this level of performance a balance must be struck between the speed afforded by in memory processing and the need to persist large quantities of data. This balance is achieved by performing dimensions computation in three phases:

1. The **Pre-Aggregation** phase.
2. The **Unification** phase.
3. The **Aggregation Storage** phase.

The sections below will describe the details of each phase of Dimensions Computation, and will also provide the code snippets required to implement each phase in Data Torrent.

### Pre-Aggregation

#### The Theory

This phase allows Dimensions Computation to scale by reducing the number of events entering the system. How this is achieved can be described by the following example:

* Let's say we have 500,000 **AdEvents** / second entering our system, and we want to perform Dimension Computation on those events.

Although each **AdEvent** will contribute to many aggregations (as described by the formulas above) the number of unique values of keys in the **AdEvents** will likely be much smaller than 500,000. So the total number of aggregations produced by 500,000 events will also be much smaller than 500,000. Let's say for the sake of this example that the number of aggregations produced will be on the order of 10,000. This means that if we perform Dimension Computation on batches of 500,000 tuples we can reduce 500,000 events to 10,000 aggregations.

The process can be sped up even further by utilizing partitioning. If a partition can handle 500,000 events / second, then 8 partitions would be able to handle 4,000,000 events / second, and those 4,000,000 events / seconds would then be compressed into 80,000 aggregations / second. These aggregations are then passed on to the Unification stage of processing.

**Note** that these 80,000 aggregations will not be complete aggregations for two reasons:

1. The aggregations do not incorporate the values of events received in previous batches. This draw back is corrected by the **Aggregation Storage** phase.
2. The aggregations computed by different partitions may share the same key values and time buckets. This draw back is corrected by the **Unification** phase.

#### The Code

Setting up the Pre-Aggregation phase of Dimension Computation involves configurating a Dimension Computation operator. There are several flavors of the Dimension Computation operator, the easiest to use out of the box for Java and dtAssemble is **DimensionsComputationFlexibleSingleSchemaPOJO**. This operator can receive any POJO as input (like our AdEvent) and requires the following configuration:

* **A JSON Schema:** The JSON schema specifies the keys, aggregates, aggregators, dimension combinations, and time buckets to be used for Dimension Computation. An example of a schema that could be used for **AdEvents** is the following:
```
{"keys":[{"name":"advertiser","type":"string"},
         {"name":"location","type":"string"}],
 "timeBuckets":["1m","1h","1d"],
 "values":
  [{"name":"impressions","type":"long","aggregators":["SUM","MAX","MIN"]},
   {"name":"clicks","type":"long","aggregators":["SUM","MAX","MIN"]},
   {"name":"cost","type":"double","aggregators":["SUM","MAX","MIN"]},
   {"name":"revenue","type":"double","aggregators":["SUM","MAX","MIN"]}],
 "dimensions":
  [{"combination":[]},
   {"combination":["location"]},
   {"combination":["advertiser"]},
   {"combination":["advertiser","location"]}]
}
```
* A map from key names to the java expression used to extract the key from an incoming POJO.
* A map from aggregate names to the java expression used to extract the aggregate from an incoming POJO.

An example of how to configure a Dimensions Computation operator to process **AdEvents** is as follows:

```
DimensionsComputationFlexibleSingleSchemaPOJO dimensions = dag.addOperator("DimensionsComputation", DimensionsComputationFlexibleSingleSchemaPOJO.class);

Map<String, String> keyToExpression = Maps.newHashMap();
keyToExpression.put("advertiser", "getAdvertiser()");
keyToExpression.put("location", "getLocation()");
keyToExpression.put("time", "getTime()");

Map<String, String> aggregateToExpression = Maps.newHashMap();
aggregateToExpression.put("cost", "getCost()");
aggregateToExpression.put("revenue", "getRevenue()");
aggregateToExpression.put("impressions", "getImpressions()");
aggregateToExpression.put("clicks", "getClicks()");

dimensions.setKeyToExpression(keyToExpression);
dimensions.setAggregateToExpression(aggregateToExpression);
//Here eventSchema is a string containing the JSON listed above.
dimensions.setConfigurationSchemaJSON(eventSchema);
```

### Unification

#### The Theory

The Unification phase is relatively simple. It combines the outputs of all the partitions in the Pre-Aggregation phase into a single single stream which can be passed on to the storage phase. It also has the added benefit of reducing the number of aggregations even further. This is because the aggregations produced by different partitions which share the same key and time bucket can be combined to produce a single aggregation. For example, if the Unification phase is receiving 80,000 aggregations / second it is not unreasonable to expect 20,000 aggregations / second after unification.

#### The Code

The Unification phase is implemented as a unifier which can be set on your dimensions computation operator.

```
dimensions.setUnifier(new DimensionsComputationUnifierImpl<InputEvent, Aggregate>());
```

### Aggregation Storage

#### The Theory

Since the total number of aggregations produced by Dimension Computation is large and increases with time (due to time bucketing) aggregations are persisted to HDFS using HDHT. This persistence is performed by the Dimensions Store and serves two purposes:

* Storage so that aggregations can be retrieved for visualization.
* Storage so that aggregations can be combined with incomplete aggregates produced by Unification.

##### Visualization

The Dimension Store allows you to visualize your aggregations over time. This is done by allowing queries and responses to be received from and sent to the UI via websocket.

##### Aggregation

The store produces complete aggregations by combining the incomplete aggregations received from the Unification stage with aggregations persisted to HDFS.

##### Scalability

Since the work done by the Dimension Store is IO intensive, it cannot handle hundreds of thousands of events. The purpose of the the Pre-Aggregation and Unification phases is to reduce the cardinality of events so that the Store will almost always have a small number of events to handle. However, in cases where there are many unique values for keys, the Pre-Aggregation and Unification phases will not be sufficient to reduce the cardinality of events handled by the Dimension Store. In such cases it is possible to partition the Dimensions Store so that each partition handles the aggregates for a subset of the dimension combinations and time buckets.

#### The Code

Configuration of the DimensionsStore involves the following:

* Setting the JSON Schema.
* Connecting Query and Result operators which are used to send queries to and receive results from the Dimension Store respectively.
* Setting an HDHT File Implementation.
* Setting an HDFS path in which to store aggregation data.

An example of configuring the store is as follows:

```
AppDataSingleSchemaDimensionStoreHDHT store = dag.addOperator("Store", AppDataSingleSchemaDimensionStoreHDHT.class);

TFileImpl hdsFile = new TFileImpl.DTFileImpl();
hdsFile.setBasePath(basePath);
store.setFileStore(hdsFile);
store.setConfigurationSchemaJSON(eventSchema);

String gatewayAddress = dag.getValue(DAG.GATEWAY_CONNECT_ADDRESS);
URI uri = URI.create("ws://" + gatewayAddress + "/pubsub");

PubSubWebSocketAppDataQuery wsIn = dag.addOperator("Query", PubSubWebSocketAppDataQuery.class);
wsIn.setUri(uri);
wsIn.setTopic("Query Topic");

PubSubWebSocketAppDataResult wsOut = dag.addOperator("QueryResult", PubSubWebSocketAppDataResult.class);
wsOut.setUri(uri);
wsOut.setTopic("Result Topic");

dag.addStream("Query", wsIn.outputPort, store.query);
dag.addStream("QueryResult", store.queryResult, wsOut.input);
```

### Putting it all Together

Combing all the pieces described above into an application that visualizes **AdEvents** looks like this:

```
@ApplicationAnnotation(name="AdEventDemo")
public class AdEventDemo implements StreamingApplication
{
  public static final String EVENT_SCHEMA = "adsGenericEventSchema.json";

  @Override
  public void populateDAG(DAG dag, Configuration conf)
  {
    //This loads the eventSchema.json file which is a jar resource file.
    String eventSchema = SchemaUtils.jarResourceFileToString("eventSchema.json");

    //Operator that receives Ad Events
    AdEventReceiver receiver = dag.addOperator("Event Receiver", AdEventReceiver.class);

    //Dimension Computation
    DimensionsComputationFlexibleSingleSchemaPOJO dimensions = dag.addOperator("DimensionsComputation", DimensionsComputationFlexibleSingleSchemaPOJO.class);

    Map<String, String> keyToExpression = Maps.newHashMap();
    keyToExpression.put("advertiser", "getAdvertiser()");
    keyToExpression.put("location", "getLocation()");
    keyToExpression.put("time", "getTime()");

    Map<String, String> aggregateToExpression = Maps.newHashMap();
    aggregateToExpression.put("cost", "getCost()");
    aggregateToExpression.put("revenue", "getRevenue()");
    aggregateToExpression.put("impressions", "getImpressions()");
    aggregateToExpression.put("clicks", "getClicks()");

    dimensions.setKeyToExpression(keyToExpression);
    dimensions.setAggregateToExpression(aggregateToExpression);
    dimensions.setConfigurationSchemaJSON(eventSchema);

    dimensions.setUnifier(new DimensionsComputationUnifierImpl<InputEvent, Aggregate>());

    //Dimension Store
    AppDataSingleSchemaDimensionStoreHDHT store = dag.addOperator("Store", AppDataSingleSchemaDimensionStoreHDHT.class);

    TFileImpl hdsFile = new TFileImpl.DTFileImpl();
    hdsFile.setBasePath("dataStorePath");
    store.setFileStore(hdsFile);
    store.setConfigurationSchemaJSON(eventSchema);

    String gatewayAddress = dag.getValue(DAG.GATEWAY_CONNECT_ADDRESS);
    URI uri = URI.create("ws://" + gatewayAddress + "/pubsub");

    PubSubWebSocketAppDataQuery wsIn = dag.addOperator("Query", PubSubWebSocketAppDataQuery.class);
    wsIn.setUri(uri);
    wsIn.setTopic("Query Topic");

    PubSubWebSocketAppDataResult wsOut = dag.addOperator("QueryResult", PubSubWebSocketAppDataResult.class);
    wsOut.setUri(uri);
    wsOut.setTopic("Result Topic");

    //Configure Streams

    dag.addStream("Query", wsIn.outputPort, store.query);
    dag.addStream("QueryResult", store.queryResult, wsOut.input);

    dag.addStream("InputStream", receiver.output, dimensions.input);
    dag.addStream("DimensionalData", dimensions.output, store.input);
  }
}
```

### Visualizing The Aggregations

When you launch your application you can visualize the aggregations of AdEvents over time by adding a widget to a visualization dashboard.

![enter image description here](https://docs.google.com/drawings/d/1wcHlgORqQYRdlnvkp3K7R9-2BllsW2jrgSBERqOq2jg/pub?w=960&h=720)

### Conclusion

Aggregating huge amounts of data in real time is a major challenge that many enterprises face today. Dimension Computation provides a valuable way in which to think about the problem of aggregating data, and Data Torrent provides an implementation of of Dimension Computation that allows users to integrate data aggregation with their applications with minimal effort.