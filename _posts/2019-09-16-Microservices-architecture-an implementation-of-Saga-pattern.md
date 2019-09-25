---
layout: single
classes: wide
title:  "Microservices architecture: an implementation of Saga pattern (DRAFT)"
description: An implementation of Saga pattern and an Hexagonal architecure.
date:   2019-09-25 16:44:00 +0200
excerpt_separator: <!--more-->
header:
  teaser: /assets/images/ketan-rajput-n-g7dgwNZg4-unsplash.jpg

tags: [docker, kafka, spring, springboot, saga, choreography, microservices, hexagonal, buildkit, multistage]

---

{% include image.html url="/assets/images/ketan-rajput-n-g7dgwNZg4-unsplash.jpg" description="Ketan Rajput" href="https://unsplash.com/photos/n-g7dgwNZg4" %}


In the last years the microservices is one of the hot topic right now in the industry, also in a context where it is not needed. Often, the design of the architecture is wrong, probably it's more like a micro-monolith service. <!--more-->If you answer "Yes" to one of these basic questions, your architecture is wrong, probably :-D

* Have you a single instance of your service?
* Have you a single database (schema)?
* Is the communication between services syncronous?
* ...

There are a lot of questions to answer, but in this post I'll show you a simple microservices architecture comply with pattern, based on the reading of the book "*Microservices Patterns*" by **Chris Richardson**.

The main idea is to build a management software for "McPaspao", my hypothetical fast food :-D. Following, a preliminary domain based analysis:

* Orders Mangement
* Kitchen Management
* Delivery Management

The *Orders Management* manages the hamburger order, the *Kitchen Mangement* manages the kitchen like cooking hamburger and the fridge management, the *Delivery Management* manages the deliveries of the hamburgers.
So I need three different services at least, each one with it's own database, then each service needs to communicate with each others. So I can add other 5 components:

* Orders Database
* Kitchen Database
* Delivery Database
* Messaging Service
* API Gateway

In the microservices architecture API Gateway, Messaging Service and Database per Service are common patterns used to solve a lot of problems, for example:

* **Messaging Service**: Services often collaborate to handle those requests so, they must use an inter-process communication protocol. More specifically an asynchronous messaging system for inter-service communication.
* **Databse per Service**: The service's database is part of the implementation to ensure loosly coupling so that it can be developed, deployed and scaled indipendently.
* **API Gateway**:  In a microservices architecture there are a lot fo services, protocols, addresses, ports, security policies, redundancy policies, ecc the API Gateway pattern tries to solve this problem, it gives to the clients a single entry point managing all the listed aspects and more.

![Architecture](/assets/images/McPaspaoArchitecure.png)

This is the big picture of the architecture, the *API Gateway* is [Kong](https://konghq.com/kong), the *Messaging Service* [Kafka](https://kafka.apache.org/) and the *Database per Service* [MongoDB](https://www.mongodb.com/).
The project is [here](https://github.com/paspao/McPaspaoTakeAway) on Github.

Each Microservice is implemented following the *Hexagonal* architecture style: the core logic is embedded inside a hexagon, and the edges of the hexagon are considered the input and output. The aim is layering the objects in a way that isolates your core logic from outside elements: the core logic is at the center of the picture and all the other elements are considered like integration points (DB, API, Messaging). We talk of *inbound adapters* that handle requests from the outside by invoking the business logic and of *outbound adapters* that are invoked by the business logic (inboking external applications). A *port* defines a set of operations and is how the business logic interacts with what's outside of it.

![Hexagonal](/assets/images/Heaxagonal.png)

In the picture *Controller* and *Consumer* are inbound adapters, *Services* are inbound port, *Messaging interfaces* and *DB Interfaces* are outbound port while *DAO* and *Producer* are outbound adapter.

I'll show the details of a single microservice for an explanation of the internal architecture used, the *Delivery Service*. It has a single api to monitor the status of a delivery, it defines an *inbound port IDeliveryAPI* 

```java
public interface IDeliveryApi {
    @ApiOperation(value = "View delivery status", response = DeliveryDTO.class,responseContainer = "list")
    @RequestMapping(value = "status", produces = MediaType.APPLICATION_JSON_VALUE, method = RequestMethod.GET)
    @ResponseBody
    List<DeliveryDTO> status();
}
```

The class *DeliveryApi* is an *inbound adapter*

```java
@RestController
@RequestMapping("/delivery/")
@Api(tags = "DeliveryServices")
public class DeliveryApi implements IDeliveryApi {

    @Autowired
    private DeliveryService deliveryService;

    @Override
    public List<DeliveryDTO> status() {
        return deliveryService.getAll();
    }
}
```

The class *DeliveryService* represents the *business logic*:

```java
@Service
public class DeliveryService {
    @Autowired
    private DeliveryRepository deliveryRepository;

    @Autowired
    private DozerBeanMapper dozerBeanMapper;

    public List<DeliveryDTO> getAll()
    {
        List<Delivery> deliveryList  =deliveryRepository.findAll();
        List<DeliveryDTO> res=null;
        if(deliveryList!=null)
        {
            res=new ArrayList<>();
            for(Delivery delivery:deliveryList)
            {
                DeliveryDTO deliveryDTO=dozerBeanMapper.map(delivery,DeliveryDTO.class);
                res.add(deliveryDTO);
            }
        }
        return res;
    }
}
```

The interface *IdeliveryPublisher* is an *outbound port*

```java
public interface IDeliveryPublisher {

    void sendToOrderCallback(OrderDTO orderDTO) throws JsonProcessingException;
}
```

The class *DeliveryPublisher* is an *outbound adapter*

```java
@Service
public class DeliveryPublisher implements IDeliveryPublisher {

    private final static String TOPIC_ORDER_CALLBACK ="orderservicecallback";

    @Autowired
    private ObjectMapper objectMapper;

    @Autowired
    private KafkaTemplate kafkaTemplate;


    @Override
    public void sendToOrderCallback(OrderDTO orderDTO) throws JsonProcessingException {
        kafkaTemplate.send(TOPIC_ORDER_CALLBACK,objectMapper.writeValueAsString(orderDTO));
    }
}
```

Each microservice has this style of architcture internally to ensure an high *loosly coupling* between software layers. But this only the internally architecture of a single microservice, it's possible that others microservices uses a *Layered* architecture style, the microservices architecture pattern controls every thing outside the microservice.

Well, a simple use case that involve every microservice is the *order management*, that is: You make a request for one hamburger, the *Order Service* receives the order and write it on the database, the order management is finished but to complete the order it needs to contact the kitchen service, it sends a message on a topic
