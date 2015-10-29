Dimensions Computation (Aggregate Navigator) Part 1: Intro
==========================================================
Introduction
---------------

In the world of big data many enterprises have a common problem, they have tremendous amounts of data flowing into their systems and they need to observe trends in that data in real-time. Consider the case of a digital advertisement publisher who is receiving hundreds of thousands of click events every second. Looking at the history of individual clicks and impressions doesn't tell the publisher much about what is going on with their users and advertisements. A useful technique the publisher may try is to look at the total number of clicks and impressions that happened every second, minute, hour, and day. Such a technique would be helpful for finding global trends in their system, but it may not provide enough granularity to take action on more localized trends. For example, looking at the total clicks and impressions for a particular advertiser, a particular geographical area, or a combination of the two could provide some actionable insight. This process of receiving individual events, aggregating them over time, and drilling down into the data using some parameters like "advertiser" and "location" is called Dimensions Computation.

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
    }
```

The **AdEvent** contains two types of data:

* **Aggregates**: The data which is combined using aggregators.
* **Keys**: The data which is used to select aggregations at a finer granularity.

#### Aggregates

The aggregates in our **AdEvent** object are the pieces of data we need to combine using aggregators to obtain a meaningful historical view. It would be meaningful to combine cost, revenue, impressions, and clicks, so those are our aggregates. We won't obtain anything useful by aggregating the location and advertiser strings in our **AdEvent**, so those are not considered aggregates. It's important to note that aggregates are considered seperate entities. This means that the cost field of and **AdEvent** cannot be combined with its revenue field; cost values can only be aggregated with other cost values and revenue values can only be aggregated with other revenue values.

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

**1.** An AdEvent arrives. The AdEvent is aggregated to the Sum and Count aggregations.
![Adding Aggregate](https://docs.google.com/drawings/d/1upf5hv-UDT4BKhm7yTrcuFZYqnI263vMTXioKhr_qTo/pub?w=960&h=720)
**2.** Another AdEvent arrives. The AdEvent is aggregated to the Sum and Count aggregations.
![Adding Aggregate](https://docs.google.com/drawings/d/10gTXjMyxanYo9UFc76IShPxOi5G7U5tvQKtfwqGyIws/pub?w=960&h=720)

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

**1.** An **AdEvent** arrives. The **AdEvent** is applied to aggregations for each aggregator and each dimension combination.
![Adding Aggregate](https://docs.google.com/drawings/d/1qx8gLu615KneLDspsGkAS0_OlkX-DyvCUA7DAJtYJys/pub?w=960&h=720)
**2.** Another **AdEvent** arrives. The **AdEvent** is applied to aggregations for each aggregator and each dimension combination.
![Adding Aggregate](https://docs.google.com/drawings/d/1FA2IyxewwzXtJ9A8JfJPrKtx-pfWHtHpVXp8lb8lKmE/pub?w=960&h=720)
**3.** Another **AdEvent** arrives. The **AdEvent** is applied to aggregations for each aggregator and each dimension combination.
![Adding Aggregate](https://docs.google.com/drawings/d/15sxwfZeYOKBiapoD2o721M4rZs-bZBxhF3MeXelnu6M/pub?w=960&h=720)

As can be seen from the example above each **AdEvent** contributes to 2 x 4 = 8 aggregations and there are 2 x 2 + 2 x 2 + 2 x 3 + 2 = 16 aggregations in total.

#### Time Buckets

In addition to computing multiple aggregations for each dimension combination, aggregations can also be performed over time buckets. Time buckets are a window of time (e.g 1:00 pm - 1:01 pm or 1:00 pm - 2:00 pm) and they are specified by a simple notation: 1m is one minute, 1h is one hour, 1d is one day. When aggregations are performed over time buckets, separate aggregations are maintained for each time bucket, and the aggregations for a time bucket are only comprised of events with a time stamp that falls into that time bucket.

##### An example of how these time bucketed aggregations are computed is as follows:

* Let's say our advertisement publisher is interested in computing the Sum and Max of **AdEvents** for the dimension combinations comprised of just **advertiser** and **location**.
* Also Let's say our publisher is computing aggregations for 1 minute and 1 hour time buckets.

**1.** An **AdEvent** arrives. The **AdEvent** is applied to the aggregations for the appropriate aggregator, dimension combination and time bucket.

![Adding Aggregate](https://docs.google.com/drawings/d/11voOdqkagpGKcWn5HOiWWAn78fXlpGl7aXUa3tG5sQc/pub?w=960&h=720)

**3.** Another **AdEvent** arrives. The **AdEvent** is applied to the aggregations for the appropriate aggregator, dimension combination and time bucket.

![Adding Aggregate](https://docs.google.com/drawings/d/1ffovsxWZfHnSc_Z30RzGIXgzQeHjCnyZBoanO_xT_e4/pub?w=960&h=720)

#### Conclusion

In summary, the three tunable parameters discussed above (**Aggregators**, **Dimension Combinations**, and **Time Buckets**) determine how aggregations are computed. Please note that in the examples provided in the **Aggregators**, **Dimension Combinations**, and **Time Buckets** sections above, we have incrementally increased the complexity in which the aggregations are performed. The examples provided in the **Aggregators**, and **Dimension Combinations** sections were for illustration purposes only; the example provided in the **Time Buckets** section provides an accurate view of how aggregations are computed within Data Torrent.