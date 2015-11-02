Dimensions Computation (Aggregate Navigator) Part 1: Intro
==========================================================
Introduction
---------------

In the world of big data, enterprises have a common problem. They have large volumes of data flowing into their systems for which they need to observe historical trends in real-time. Consider the case of a digital advertising publisher that is receiving hundreds of thousands of click events every second. Looking at the history of individual clicks and impressions doesn't tell the publisher much about what is going on. A technique the publisher might employ is to track the total number of clicks and impressions for every second, minute, hour, and day. Such a technique might help find global trends in their systems, but may not provide enough granularity to take action on localized trends. The technique will need to be powerful enough to spot local trends. For example, the total clicks and impressions for an advertiser, a geographical area, or a combination of the two can provide some actionable insight. This process of receiving individual events, aggregating them over time, and drilling down into the data using some parameters like "advertiser" and "location" is called Dimensions Computation.

Dimensions Computation is a powerful mechanism that allows you to spot trends in your streaming data in real-time. In this post we'll cover the key concepts behind Dimensions Computation and outline the process of performing Dimensions Computation. We will also show you how to use Data Torrent's out-of-the-box enterprise operators to easily add Dimensions Computation to your application.

The Process
------------

Let us continue with our example of an advertising publisher. Let us now see the steps that the publisher might take to ensure that large volumes of raw advertisement data is converted into meaningful historical views of their advertisement events.

### The Data

Typically advertising publishers receive packets of information for each advertising event. The events that a publisher receives might look like this:

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
    }
```

The class **AdEvent** contains two types of data:

* **Aggregates**: The data that is combined using aggregators.
* **Keys**: The data that is used to select aggregations at a finer granularity.

#### Aggregates

The aggregates in our **AdEvent** object are the pieces of data, which we must combine using aggregators in order to obtain a meaningful historical view. In this case, we can think of combining cost, revenue, impressions, and clicks. So these are our aggregates. We won't obtain anything useful by aggregating the location and advertiser strings in our **AdEvent**, so those are not considered aggregates. It's important to note that aggregates are considered separate entities. This means that the cost field of and **AdEvent** cannot be combined with its revenue field; cost values can only be aggregated with other cost values and revenue values can only be aggregated with other revenue values.

In summary the aggregates in our **AdEvent** are:

* **cost**
* **revenue**
* **impressions**
* **clicks**

#### Keys

The keys in our **AdEvent** object are used for selecting aggregations at a finer granularity. For example, it would make sense to look at the number of clicks for a particular advertiser, the number of clicks for a certain location, and the number of clicks for a certain location and advertiser combination. So location and advertiser are keys. Time is also another key since it is useful to look at the number of clicks received in a particular time range (For example, 12:00 pm through 1:00 pm or 12:00 pm through 12:01 pm.

In summary the keys in our **AdEvent** are:

* **advertiser**
* **location**
* **time**

### Computing The Aggregations

When the publisher receives a new **AdEvent** the event is added to running aggregations in real time. The keys and aggregates in the **AdEvent** are used to compute aggregations. How the aggregations are computed and the number of aggregations computed are determined by three tunable parameters:

* **Aggregators**
* **Dimensions Combinations**
* **Time Buckets**

#### Aggregators

Dimensions Computation supports more than just one type of aggregation, and multiple aggregators can be used to combine incoming data at once. Some of the aggregators available out-of-the-box are:

* **Sum**
* **Count**
* **Min**
* **Max**

As an example, suppose the publisher is not using the keys in their **AdEvents** and this publisher wants to perform a sum and a max aggregation.

**1.** An AdEvent arrives. The AdEvent is aggregated to the Sum and Count aggregations.
![Adding Aggregate](https://docs.google.com/drawings/d/1upf5hv-UDT4BKhm7yTrcuFZYqnI263vMTXioKhr_qTo/pub?w=960&h=720)
**2.** Another AdEvent arrives. The AdEvent is aggregated to the Sum and Count aggregations.
![Adding Aggregate](https://docs.google.com/drawings/d/10gTXjMyxanYo9UFc76IShPxOi5G7U5tvQKtfwqGyIws/pub?w=960&h=720)

As can be seen from the example above, each **AdEvent** contributes to two aggregations.

#### Dimension Combinations

Each **AdEvent** does not necessarily contribute to only one aggregation for each aggregator. In our advertisement example there are 4 **dimension combinations** over which aggregations can be computed.

* **advertiser:** This dimension combination is comprised of just the advertiser value. This means that all the aggregates for **AdEvents** with a particular value for advertiser (for example, Gamestop) are aggregated.
* **location:** This dimension combination is comprised of just the location value. This means that all the aggregates for **AdEvents** with a particular value for location (for example, California) are aggregated.
* **advertiser, location:** This dimension combination is comprised the advertiser and location values. This means that all the aggregates for **AdEvents** with the same advertiser and location combination (for example, Gamestop, California) are aggregated.
* **the empty combination:** This combination is a *global aggregation* because it doesn't use any of the keys in the **AdEvent**. This means that all the **AddEvents** are aggregated.

Therefore if a publisher is using the four dimension combinations enumerated above along with the sum and max aggregators, the number of aggregations being maintained would be:

NUM_AGGS = 2 x *(number of unique advertisers)* + 2 * *(number of unique locations)* + 2 * *(number of unique advertiser and location combinations)* + 2

And each individual **AdEvent** will contribute to *(number of aggregators)* x *(number of dimension combinations)* aggregations.

Here is an example of how NUM_AGGS aggregations are computed:

**1.** An **AdEvent** arrives. The **AdEvent** is applied to aggregations for each aggregator and each dimension combination.
![Adding Aggregate](https://docs.google.com/drawings/d/1qx8gLu615KneLDspsGkAS0_OlkX-DyvCUA7DAJtYJys/pub?w=960&h=720)
**2.** Another **AdEvent** arrives. The **AdEvent** is applied to aggregations for each aggregator and each dimension combination.
![Adding Aggregate](https://docs.google.com/drawings/d/1FA2IyxewwzXtJ9A8JfJPrKtx-pfWHtHpVXp8lb8lKmE/pub?w=960&h=720)
**3.** Another **AdEvent** arrives. The **AdEvent** is applied to aggregations for each aggregator and each dimension combination.
![Adding Aggregate](https://docs.google.com/drawings/d/15sxwfZeYOKBiapoD2o721M4rZs-bZBxhF3MeXelnu6M/pub?w=960&h=720)

As can be seen from the example above each **AdEvent** contributes to 2 x 4 = 8 aggregations and there are 2 x 2 + 2 x 2 + 2 x 3 + 2 = 16 aggregations in total.

#### Time Buckets

In addition to computing multiple aggregations for each dimension combination, aggregations can also be performed over time buckets. Time buckets are windows of time (for example, 1:00 pm through 1:01 pm) that are specified by a simple notation: 1m is one minute, 1h is one hour, 1d is one day. When aggregations are performed over time buckets, separate aggregations are maintained for each time bucket. Aggregations for a time bucket are comprised only of events with a time stamp that falls into that time bucket.

An example of how these time bucketed aggregations are computed is as follows:

Let's say our advertisement publisher is interested in computing the Sum and Max of **AdEvents** for the dimension combinations comprised of **advertiser** and **location** over 1 minute and 1 hour time buckets.

**1.** An **AdEvent** arrives. The **AdEvent** is applied to the aggregations for the appropriate aggregator, dimension combination and time bucket.

![Adding Aggregate](https://docs.google.com/drawings/d/11voOdqkagpGKcWn5HOiWWAn78fXlpGl7aXUa3tG5sQc/pub?w=960&h=720)

**3.** Another **AdEvent** arrives. The **AdEvent** is applied to the aggregations for the appropriate aggregator, dimension combination and time bucket.

![Adding Aggregate](https://docs.google.com/drawings/d/1ffovsxWZfHnSc_Z30RzGIXgzQeHjCnyZBoanO_xT_e4/pub?w=960&h=720)

#### Conclusion

In summary, the three tunable parameters discussed above (**Aggregators**, **Dimension Combinations**, and **Time Buckets**) determine how aggregations are computed. In the examples provided in the **Aggregators**, **Dimension Combinations**, and **Time Buckets** sections respectively, we have incrementally increased the complexity in which the aggregations are performed. The examples provided in the **Aggregators**, and **Dimension Combinations** sections were for illustration purposes only; the example provided in the **Time Buckets** section provides an accurate view of how aggregations are computed within Data Torrent's enterprise operators.