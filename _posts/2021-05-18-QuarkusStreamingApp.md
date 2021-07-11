---
layout: single
classes: wide
title:  "A kafka streaming application with Quarkus (DRAFT)"
description: A kafka streaming applicationt with Quarkus.
date:   2021-05-21 18:43:00 +0200
excerpt_separator: <!--more-->
header:
  teaser: /assets/images/quarkus_logo_kafka.png

tags: [docker, kafka, quarkus, streaming]

---

{% include image_width.html url="/assets/images/quarkus_logo_kafka.png" description="" width="600px" %}

The Kafka streaming application platform is becoming *"de facto"* the modern approach to the ETL (Extract, transform and load) world. Probably you already read a lot about "What is Kafka", "What is a topic" and "What is a partition", but i want to share with you "What is a streaming application", this the aim of this article, i hope i'll achieve the goal.
<!--more-->
The book *"Kafka streams in Action"* defines the stream processing as *"working with data as it’s arriving in your system"* or more refined *"stream processing is the ability to work with an infinite stream of data with continuous computation, as it flows, with no need to collect or store the data
to act on it"*. Usually, we use a stream processing application when we have a lot of data (BigData) and the need to quickly respond to or report on incoming data.

In this scenarious all the data are already present inside Kafka topic, i tryed to draw them in the below picture:
{% include image_width.html url="/assets/images/allTogether.svg" description="" width="800px" %}
So i created four simple entities (each one is a Kafka topic), that is: Person, DriverLicense, Sim, Fee. The data architecure is pretty simple: a person could have a driver license, a person could have sim card and last but not least a person pays the fees unfortunatly.
I marked with red colour the _key_ of a topic and the entire row is the kafka message or the _payload_.

My first goal is to join two simple topic that are _Person_ and _DriverLicense_ because i want to discover how many person already had a driver license, so let's start to join these two topics. 
From a data perspective to join two topics they have to respect the _co-partitioning_ requirements that are:

* The input records for the join must have the same key (or key schema)
* The input records must have the same number of partitions on both sides
* Both sides of the join must have the same partitioning strategy (usually we use the default one offered by the kafka client, pratically ignoring the ability to choose a different one)



{% highlight java linenos %}
public Topology appJoinPersonAndDriverLicense() {

    final KStream<String, Person> personStream = builder.stream(PERSON,
            Consumed.with(Serdes.String(), FactorySerde.getPersonSerde()));
    final KStream<String, DriverLicense> driverLicenseStream = builder.stream(DRIVERLICENSE,
            Consumed.with(Serdes.String(), FactorySerde.getDriverLicenseSerde()));

    KStream<String, Person> peekStream = personStream.peek(loggingForEach);

    KStream<String, PersonDriverLicense> joined = peekStream.join(driverLicenseStream,
            (person, license) -> new PersonDriverLicense(person, license),
            JoinWindows.of(Duration.of(5, ChronoUnit.MINUTES)), StreamJoined.with(
                    Serdes.String(), /* key */
                    FactorySerde.getPersonSerde(),   /* left value */
                    FactorySerde.getDriverLicenseSerde()  /* right value */
            ));

    joined.to(ALL, Produced.with(Serdes.String(), FactorySerde.getPersonDriverLicenseSerde()));

    Topology build = builder.build();
    logger.info(build.describe().toString());
    return build;
}
{% endhighlight %}

I don't want show you the details about the configuration, mainly i want to work on the streaming core: at lines 3 and 5 we defined two streams one for _Person_ topic and one for _DriverLicense_ topic, then just a _peek_ operation to log the _stream person_ and later the _join_ operation at lines 10-16 between Person and DriverLicense. Be careful, I used a _peekStream_ and not the previous one (_personStream_) at line 10 because i added a log inside the flow, i guess we need something to see this topology.
At line 21 i printed out the stream in the console log, it's a lgical desciption of this stream, that is:

{% highlight properties %}
Topologies:
Sub-topology: 0
Source: KSTREAM-SOURCE-0000000000 (topics: [person])
--> KSTREAM-PEEK-0000000002
Processor: KSTREAM-PEEK-0000000002 (stores: [])
--> KSTREAM-WINDOWED-0000000003
<-- KSTREAM-SOURCE-0000000000
Source: KSTREAM-SOURCE-0000000001 (topics: [driverlicense])
--> KSTREAM-WINDOWED-0000000004
Processor: KSTREAM-WINDOWED-0000000003 (stores: [KSTREAM-JOINTHIS-0000000005-store])
--> KSTREAM-JOINTHIS-0000000005
<-- KSTREAM-PEEK-0000000002
Processor: KSTREAM-WINDOWED-0000000004 (stores: [KSTREAM-JOINOTHER-0000000006-store])
--> KSTREAM-JOINOTHER-0000000006
<-- KSTREAM-SOURCE-0000000001
Processor: KSTREAM-JOINOTHER-0000000006 (stores: [KSTREAM-JOINTHIS-0000000005-store])
--> KSTREAM-MERGE-0000000007
<-- KSTREAM-WINDOWED-0000000004
Processor: KSTREAM-JOINTHIS-0000000005 (stores: [KSTREAM-JOINOTHER-0000000006-store])
--> KSTREAM-MERGE-0000000007
<-- KSTREAM-WINDOWED-0000000003
Processor: KSTREAM-MERGE-0000000007 (stores: [])
--> KSTREAM-SINK-0000000008
<-- KSTREAM-JOINTHIS-0000000005, KSTREAM-JOINOTHER-0000000006
Sink: KSTREAM-SINK-0000000008 (topic: alltogether)
<-- KSTREAM-MERGE-0000000007
{% endhighlight %}
We can use this description to draw the topology through a tool like [https://zz85.github.io/kafka-streams-viz/](https://zz85.github.io/kafka-streams-viz/), below the result:

{% include image_width.html url="/assets/images/firstTopology.png" description="" width="600px" %}

Well now we have a graphical rappresentation of our simple topology as well. It's more clear to see the topics, the streams, the _peek_ operations we alredy discussed, but now we can see more clearly what happen during a join operation. At line 12 we used a _JoinWindow_, the _join_ operation is a stateful transformation we need a state store to collect all of the records received so far within the defined window boundary. With Kafka we have the ability to consider the time as data resource, so we don't want to process all the updates received a for a specific record but only what is changed in the last 5 minutes for example, in this way i will process only few kilobyte of data (in a BigData context we could have topic with Terabyte of data). So, for each element of the join operation a state store will be created (the two cylinder on the right side of the picture) and a corresponding topic will be created on kafka as well, these topics are named as _store-changelog_ topics, this to ensure the the foult-talerant capabilities of the Kafka Streams. As described in the confluent documentation: _"These changelog topics are partitioned as well so that each local state store instance, and hence the task accessing the store, has its own dedicated changelog topic partition. Log compaction is enabled on the changelog topics so that old data can be purged safely to prevent the topics from growing indefinitely. If tasks run on a machine that fails and are restarted on another machine, Kafka Streams guarantees to restore their associated state stores to the content before the failure by replaying the corresponding changelog topics prior to resuming the processing on the newly started tasks. As a result, failure handling is completely transparent to the end user."_

So, in the final step at line 18 the data will be joined by the key and the result in the _alltogether_ topic will be somethin like:

{% highlight json %}
{
   "person": {
      "ssn": "789",
      "name": "Giacchino",
      "surname": "Murat",
      "sex": "M",
      "dateOfBirth": "1981-06-28",
      "phoneNumber": "1113245798"
   },
   "driverLicense": {
      "id": "789",
      "type": "CAR",
      "licenseNumber": "4213245768"
   }
}
{% endhighlight %}

*TO BE CONTINUE...*

[https://github.com/paspao/quarkus-streaming-app](https://github.com/paspao/quarkus-streaming-app)
