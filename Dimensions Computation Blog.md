Dimensions Computation
=====================
Introduction
---------------

In the world of big data many enterprises have a common problem, they have tremendous amounts of data flowing into their systems and they need to observe historical trends in that data. Consider the case of a digital advertisement publisher who is receiving hundreds of thousands of click events every second. Looking at the history of individual clicks and impressions doesn't tell the publisher much about what is going on with their users and advertisements. A useful technique the publisher may try is to look at the total number of clicks and impressions that happened every second, minute, hour, and day. Such a technique would be helpful for finding global trends in their system, but it may not provide enough granularity to take action on more localized trends. For example, looking at the total clicks and impressions for a particular advertiser, a particular geographical area, or a combination of the two could provide some actionable insight. This process of receiving individual events, aggregating them over time, and drilling down into the data using some parameters like "advertiser" and "location" is called Dimensions Computation. 

Dimensions Computation is a powerful mechanism that allows you to spot trends in your streaming data in real-time. In this post we'll cover the key concepts behind Dimensions Computation and outline the process of performing Dimensions Computation. We will also show you how to use Data Torrent's out of the box operators to easily add Dimensions Computation to your application. 

The Process
---------------

Continuing with our example of an advertisement publisher let's trace through the steps of how a publisher can go from massive amounts of raw advertisement data to meaningful historical views of their advertisement events.

### The Data

Typically advertisement publishers receive a packet of information every time an event happens related to their advertisements. Let's say the events a publisher receives look like this:

```
    public class AdEvent
    {
	  //The name of the company that is advertising
      public String advertiser;
      
      //The geographical location of the person initiating the event
      public String location;
      
      //How much the advertiser was charged for the event
      public double cost;
      
      //How much revenue was generated for the advertiser
      public double revenue;
      
      //The number of impressions the advertiser received from this event
      public long impressions;
      
      //The number of clicks the advertiser received from this event
      public long clicks;
      
      //The timestamp of the event in milliseconds
      public long time;

      public AdEvent()
      {
      }

      public AdEvent(String advertiser,
                     String location,
                     double cost,
                     double revenue,
                     long impressions,
                     long clicks,
                     long time)
      {
        this.advertiser = advertiser;
        this.location = location;
        this.cost = cost;
        this.revenue = revenue;
        this.impressions = impressions;
        this.clicks = clicks;
        this.time = time;
      }

      /*
       * Getters and setters go here
       */
    }
```

The **AdEvent** contains two types of data:

* **Aggregates**: The data which is combined using aggregators.
* **Keys**: The data which is used to select aggregations at a finer granularity.

#### Aggregates

The aggregates in our **AdEvent** object are the pieces of data we need to combine using aggregators to obtain a meaningful historical view. It would be meaningful to combine cost, revenue, impressions, and clicks, so those are our aggregates. We won't obtain anything useful by aggregating the location and advertiser strings in our **AdEvent**, so those are not considered aggregates. It's important to note that aggregates are considered seperate entities. This means that the cost field of and **AdEvent** cannot be combined with it's revenue field; cost values can only be aggregated with other cost values and revenue values can only be aggregated with other revenue values.

In summary the aggregates in our **AdEvent** are:

* **cost**
* **revenue**
* **impressions**
* **clicks**

#### Keys

The keys in our **AdEvent** object are used to select aggregations at a finer granularity. It would make sense to look at the number of clicks for a particular advertiser, the number of clicks for a certain location, or the number of clicks for a certain location and advertiser combination, so they are keys. Time is also another key since it is useful to look at the number of clicks received from 12:00 pm - 1:00 pm or 12:00 pm to 12:01 pm.

In summary the keys in our **AdEvent** are:

* **advertiser**
* **location**
* **time**

### Computing The Aggregations

When an advertisement publisher receives a new **AdEvent** the event is added to running aggregations in real time. The keys and aggregates in the **AdEvent** are used to compute aggregations. How the aggregations are computed and the number of aggregations computed are determined by three tunable parameters:

* **Aggregators**
* **Dimensions Combinations**
* **Time Buckets**

#### Aggregators

Dimensions Computation supports more than just one kind of aggregation, and multiple aggregators can be used to combine incoming data at once. Some of the aggregators available out of the box are:

* **Sum**
* **Count**
* **Min**
* **Max**

##### An example of applying aggregators to input data is as follows:

Let's say our publisher is not using the keys in their **AdEvents** and they want to perform a sum and a max aggregation.

**1.** An AdEvent arrives.
![enter image description here](https://docs.google.com/drawings/d/1upf5hv-UDT4BKhm7yTrcuFZYqnI263vMTXioKhr_qTo/pub?w=960&h=720)
**2.** The AdEvent is aggregated to the Sum and Count aggregations.
![enter image description here](https://docs.google.com/drawings/d/18kkK7NSioxW8Q7xV63v2IUHsu8TpYH5U5CFTQG5OXVw/pub?w=960&h=720)
**3.** Another AdEvent arrives.
![enter image description here](https://docs.google.com/drawings/d/10gTXjMyxanYo9UFc76IShPxOi5G7U5tvQKtfwqGyIws/pub?w=960&h=720)
**4.** The AdEvent is aggregated to the Sum and Count aggregations.
![enter image description here](https://docs.google.com/drawings/d/1SZkRDxkWMx-u_1mQnGgL1LC-RUkZV5sOCEX_9YWfpHE/pub?w=960&h=720)

As can be seen from the example above, each **AdEvent** contributes to two aggregations.

#### Dimension Combinations

Each **AdEvent** does not necessarily contribute to only one aggregation for each aggregator. In our advertisement example there are 4 **dimension combinations** over which aggregations can be computed.

* **advertiser:** This dimension combination is comprised of just the advertiser value. This means that all the aggregates for **AdEvents** with a particular value for advertiser (e.g. Gamestop) are aggregated.
* **location:** This dimension combination is comprised of just the location value. This means that all the aggregates for **AdEvents** with a particular value for location (e.g. California) are aggregated.
* **advertiser, location:** This dimension combination is comprised the advertiser and location values. This means that all the aggregates for **AdEvents** with the same advertiser and location combination (e.g. Gamestop, California) are aggregated.
* **the empty combination:** This combination is a *global aggregation* in the sense that it doesn't use any of the keys in the **AdEvent**. This means that all the **AddEvents** are aggregated.

Therefore if a publisher is using the four dimension combinations enumerated above along with the sum and max aggregators, the number of aggregations being maintained would be:

NUM_AGGS = 2 x *(number of unique advertisers)* + 2 * *(number of unique locations)* + 2 * *(number of unique advertiser and location combinations)* + 2

And each individual **AdEvent** will contribute to *(number of aggregators)* x *(number of dimension combinations)* aggregations.

##### An example of how these NUM_AGGS aggregations are computed is as follows:

**1.** An **AdEvent** arrives.
![enter image description here](https://docs.google.com/drawings/d/1qx8gLu615KneLDspsGkAS0_OlkX-DyvCUA7DAJtYJys/pub?w=960&h=720)
**2.** The **AdEvent** is applied to aggregations for each aggregator and each dimension combination.
![enter image description here](https://docs.google.com/drawings/d/1EXtR6vNws0LbQQeLPq4TPtuQinU2vPkZjCQbvp9vyVw/pub?w=960&h=720)
**3.** Another **AdEvent** arrives.
![enter image description here](https://docs.google.com/drawings/d/1FA2IyxewwzXtJ9A8JfJPrKtx-pfWHtHpVXp8lb8lKmE/pub?w=960&h=720)
**4.** The **AdEvent** is applied to aggregations for each aggregator and each dimension combination.
![enter image description here](https://docs.google.com/drawings/d/1-4BwRr3SCpvsxpSXSCEy1ySmvFB24aiAfFK0K7QN-P0/pub?w=960&h=720)
**5.** Another **AdEvent** arrives.
![enter image description here](https://docs.google.com/drawings/d/15sxwfZeYOKBiapoD2o721M4rZs-bZBxhF3MeXelnu6M/pub?w=960&h=720)
**6.** The **AdEvent** is applied to aggregations for each aggregator and each dimension combination.
![enter image description here](https://docs.google.com/drawings/d/1E1kBvH7azYwmDyJYsZY-2rym6SK6j1Yi5R_eKr-AOC8/pub?w=960&h=720)

As can be seen from the example above each **AdEvent** contributes to 2 x 4 = 8 aggregations and there are 2 x 2 + 2 x 2 + 2 x 3 + 2 = 16 aggregations in total. 

#### Time Buckets

In addition to computing multiple aggregations for each dimension combination, aggregations can also be performed over time buckets. Time buckets are a window of time (e.g 1:00 pm - 1:01 pm or 1:00 pm - 2:00 pm) and they are specified by a simple notation: 1m is one minute, 1h is one hour, 1d is one day. When aggregations are performed over time buckets, separate aggregations are maintained for each time bucket, and the aggregations for a time bucket are only comprised of events with a time stamp that falls into that time bucket.

##### An example of how these time bucketed aggregations are computed is as follows:

* Let's say our advertisement publisher is interested in computing the Sum and Max of **AdEvents** for the dimension combinations comprised of just **advertiser** and **location**.
* Also Let's say our publisher is computing aggregations for 1 minute and 1 hour time buckets.

**1.** An **AdEvent** arrives.

![enter image description here](https://docs.google.com/drawings/d/1W3qKJnWD5K_P2R2HpPrqCYgQrU9cMVrK3CE9UbDayC8/pub?w=960&h=720)

**2.** The **AdEvent** is applied to the aggregations for the appropriate aggregator, dimension combination and time bucket.

![enter image description here](https://docs.google.com/drawings/d/1vGmRoCFfqLqv8Mql5JjkB2FbqOPZ78qvzm8mBBOefJs/pub?w=960&h=720)

**3.** Another **AdEvent** arrives.

![enter image description here](https://docs.google.com/drawings/d/1ffovsxWZfHnSc_Z30RzGIXgzQeHjCnyZBoanO_xT_e4/pub?w=960&h=720)

**4.**  The **AdEvent** is applied to the aggregations for the appropriate aggregator, dimension combination and time bucket.

![enter image description here](https://docs.google.com/drawings/d/17kD7aapEKZyPxvLLp2RCl7le81qGI8gxEU_MmNoORdc/pub?w=960&h=720)

#### Conclusion

In summary, the three tunable parameters discussed above (**Aggregators**, **Dimension Combinations**, and **Time Buckets**) determine how aggregations are computed. Please note that in the examples provided in the **Aggregators**, **Dimension Combinations**, and **Time Buckets** sections above, we have incrementally increased the complexity in which the aggregations are performed. The examples provided in the **Aggregators**, and **Dimension Combinations** sections were for illustration purposes only; the example provided in the **Time Buckets** section provides an accurate view of how aggregations are computed within Data Torrent.

The Implementation
------------------------

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