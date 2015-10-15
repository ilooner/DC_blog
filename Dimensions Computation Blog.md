Dimensions Computation
=====================
Introduction
---------------

In the world of big data many enterprises have a common problem, they have tremendous amounts of data flowing into their systems and they need to observe historical trends in that data. Consider the case of a digital advertisement publisher who is receiving hundreds of thousands of click events every second. Looking at the history of individual clicks and impressions doesn't tell the publisher much about what is going on with their users and advertisements. A useful technique the publisher may try is to look at the total number of clicks and impressions that happened every second, minute, hour, and day. Such a technique would be helpful for finding global trends in their system, but it may not provide enough granularity to take action on more localized trends. For example, looking at the total clicks and impressions for a particular advertiser, a particular geographical area, or a combination of the two could provide some actionable insight. This process of receiving individual events, aggregating them over time, and drilling down into the data using some parameters like "advertiser" and "location" is called Dimensions Computation. 

In this post we'll first cover the general process of performing dimensions computation in detail, then we will provide concrete examples of how to use the Dimensions Computation operators in Megh within your application.
	
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
      
      //The timestamp of the event
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

**2.** The AdEvent is aggregated to the Sum and Count aggregations.

**3.** Another AdEvent arrives.

**4.** The AdEvent is aggregated to the Sum and Count aggregations.

#### Dimension Combinations

Each **AdEvent** does not necessarily contribute to only one aggregation for each aggregator. In our advertisement example there are 4 **dimension combinations** over which aggregations can be computed.

* **advertiser:** This dimension combination is comprised of just the advertiser value. This means that all the aggregates for **AdEvents** with a particular value for advertiser (e.g. Gamestop) are aggregated.
* **location:** This dimension combination is comprised of just the location value. This means that all the aggregates for **AdEvents** with a particular value for location (e.g. California) are aggregated.
* **advertiser, location:** This dimension combination is comprised the advertiser and location values. This means that all the aggregates for **AdEvents** with the same advertiser and location combination (e.g. Gamestop, California) are aggregated.
* **the empty combination:** This combination is a *global aggregation* in the sense that it doesn't use any of the keys in the **AdEvent**. This means that all the **AddEvents** are aggregated.

Therefore if a publisher is using the four dimension combinations enumerated above along with the sum and count aggregators, the number of aggregations being maintained would be:

NUM_AGGS = 2 x *(number of unique advertisers)* + 2 * (number of unique locations) + 2 * (number of unique advertiser and location combinations) + 2

##### An example of how these NUM_AGGS aggregations are computed is as follows:

1

#### Time Buckets

In addition to computing aggregations for dimensions combinations aggregations can also be performed for time buckets. For example an advertiser may be interested in looking the sum of clicks for a particular 

#### Computing

An example of aggregating data


The Implementation
------------------------

### Computing Pre-Aggregations

### Storing The Aggregations

### Putting it all Together

### Visualizing The Aggregations

