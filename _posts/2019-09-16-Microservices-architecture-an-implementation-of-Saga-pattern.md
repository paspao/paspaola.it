---
layout: single
classes: wide
title:  "Microservices architecture: an implementation of Saga pattern"
description: An implementation of Saga pattern and an Hexagonal architecure.
date:   2019-09-25 16:44:00 +0200
excerpt_separator: <!--more-->
header:
  teaser: /assets/images/ketan-rajput-n-g7dgwNZg4-unsplash.jpg

tags: [docker, kafka, spring, springboot, saga, choreography, microservices, hexagonal, buildkit, multistage]

---

{% include image.html url="/assets/images/ketan-rajput-n-g7dgwNZg4-unsplash.jpg" description="Ketan Rajput" href="https://unsplash.com/photos/n-g7dgwNZg4" %}


In the last years the microservices is one of the hot topic right now in the industry, also in a context where it is not needed. Often, the architecture design is wrong, probably it's more like a micro-monolith service. <!--more-->If you answer "Yes" to one of these basic questions, your architecture is wrong, probably :-D

* Have you a single instance of your service?
* Have you a single database (or schema)?
* Is the communication between services syncronous?
* ...

There are a lot of questions to answer, but in this post I'll show you a simple microservices architecture comply with pattern, based on the book "*Microservices Patterns*" by **Chris Richardson**.

The main idea,in my example, is to build a management software for "McPaspao", my hypothetical fast food :-D. Following, a preliminary domain based analysis:

* Orders Mangement
* Kitchen Management
* Delivery Management

The *Orders Management* manages the hamburger order, the *Kitchen Mangement* manages the kitchen job (eg: cooking hamburger or the fridge management), the *Delivery Management* manages the deliveries of the hamburgers.
So I need three different services at least, each one with it's own database, then each service needs to communicate with each others. 
In this scenario other five components are needed:

* Orders Database
* Kitchen Database
* Delivery Database
* Messaging Service
* API Gateway

In the microservices architecture API Gateway, Messaging Service and Database per Service are common patterns used to solve a lot of problems, for example:

* **Messaging Service**: Services often collaborate to handle many requests so, they must use an inter-process communication protocol. More specifically an asynchronous messaging system.
* **Database per Service**: The service's database must be part of the implementation to ensure loosly coupling so that it can be developed, deployed and scaled indipendently.
* **API Gateway**:  In a microservices architecture there are a lot fo services, protocols, addresses, ports, security policies, redundancy policies, etc the API Gateway pattern tries to solve this problem, it gives to the clients a single entry point managing all the listed aspects and more.

![Architecture](/assets/images/McPaspaoArchitecure.png)

This is the big picture of the architecture, the *API Gateway* is [Kong](https://konghq.com/kong), the *Messaging Service* [Kafka](https://kafka.apache.org/) and the *Database per Service* [MongoDB](https://www.mongodb.com/).
The project is [here](https://github.com/paspao/McPaspaoTakeAway) on Github.

Each Microservice is implemented following the *Hexagonal* architecture style: the core logic is embedded inside a hexagon, and the edges of the hexagon are considered the input and output. The aim is to layer the objects in a way that isolates your core logic from outside elements: the core logic is at the center of the picture and all the other elements are considered as integration points (DB, API, Messaging). We talk about *inbound adapters* that handle requests from the outside by invoking the business logic and about *outbound adapters* that are invoked by the business logic (to invoke external applications). A *port* defines a set of operations that is how the business logic interacts with what's outside of it.

![Hexagonal](/assets/images/Heaxagonal.png)

In the picture *Controller* and *Consumer* are inbound adapters, *Services* are inbound port, *Messaging interfaces* and *DB Interfaces* are outbound port while *DAO* and *Producer* are outbound adapter.

I'll show the details of a single microservice for an explanation of the internal architecture used, the *Delivery Service*. It has a single api to monitor the status of a delivery, it defines an *inbound port IDeliveryAPI*: 

```java
public interface IDeliveryApi {
    @ApiOperation(value = "View delivery status", response = DeliveryDTO.class,responseContainer = "list")
    @RequestMapping(value = "status", produces = MediaType.APPLICATION_JSON_VALUE, method = RequestMethod.GET)
    @ResponseBody
    List<DeliveryDTO> status();
}
```

The class *DeliveryApi* is an *inbound adapter*:

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

The interface *IDeliveryPublisher* is an *outbound port*:

```java
public interface IDeliveryPublisher {
    void sendToOrderCallback(OrderDTO orderDTO) throws JsonProcessingException;
}
```

The class *DeliveryPublisher* is an *outbound adapter*:

```java
@Service
public class DeliveryPublisher implements IDeliveryPublisher {

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

Each microservice (in my example) has this style of architcture internally to ensure an high *loosly coupling* between software layers. But this is only the internally architecture of a single microservice, it's possible that other microservices uses a *Layered* architecture style, for example. 

Well, a simple use case that involve every microservice is the *order management*, that is: a browser makes a request for one hamburger, the *Order Service* receives the order and write it on the database, the order management work is finished but to complete the order it needs to contact the *Kitchen Service*, so it sends a message on a topic (to ensure an asyncronous inter-process-communiction), the *Kitchen Service* is listening on this topic, it consumes the message and it processes the order, giving a feedback to the *Order Service* through another topic. When the *Kitchen Service* has cooked the hamburger it sends a message to *Delivery Service*, the *Delivery Service* processes the message, it delivers the hamburger and it sends a feedback. Every communication between the microservices goes through the message broker, in my example *Kafka*, I have applied a *Choreography Saga pattern*, that is:

> A saga is a sequence of local transactions. Each local transaction updates the database and publishes a message or event to trigger the next local transaction in the saga. If a local transaction fails because it violates a business rule then the saga executes a series of compensating transactions that undo the changes that were made by the preceding local transactions.
>There are two ways of coordination sagas:
>* Choreography - each local transaction publishes domain events that trigger local transactions in other services
>* Orchestration - an orchestrator (object) tells the participants what local transactions to execute
>
> <cite>Chris Richardson</cite>

To see the entire architecture, I use the docker-compose.yml (docker app) listed below:

```yml
version: '3.2'
services:

  order-service:
    image: paspaola/order-service:0.0.1
    ports:
      - 8090:8090
    depends_on:
      - mongodb-order
      - kafkabroker
    networks:
      - mcpaspao

  kitchen-service:
    image: paspaola/kitchen-service:0.0.1
    ports:
      - 8080:8080
    depends_on:
      - mongodb-kitchen
      - kafkabroker
    networks:
      - mcpaspao

  delivery-service:
    image: paspaola/delivery-service:0.0.1
    ports:
      - 8070:8070
    depends_on:
      - mongodb-delivery
      - kafkabroker
    networks:
      - mcpaspao

  mongodb-delivery:
    image: mongo:3.4.22-xenial
    ports:
      - 27017:27017
    networks:
      - mcpaspao

  mongodb-order:
    image: mongo:3.4.22-xenial
    ports:
      - 27018:27017
    networks:
      - mcpaspao

  mongodb-kitchen:
    image: mongo:3.4.22-xenial
    ports:
      - 27019:27017
    networks:
      - mcpaspao

  kafkabroker:
    image: paspaola/kafka-mcpaspao
    ports:
      - 2181:2181
      - 9092:9092
    environment:
      - KAFKA_ADVERTISED_LISTNERS=${advertised.addr}
    networks:
      - mcpaspao

  kong-mcpaspao:
    image: paspaola/kong-mcpaspao:0.0.1
    ports:
      - 8000:8000
      - 8443:8443
      - 8001:8001
      - 8444:8444
    networks:
      - mcpaspao
    depends_on:
      - delivery-service
      - kitchen-service
      - order-service

networks:
  mcpaspao:
```

Like into the big picture above, there are three services and three database, then there is the Kafka broker, a personalized image that already have on borad all the needed topics:

* orderservice
* orderservicecallback
* kitchenservice
* deliveryservice

In the Kafka container there is also an instance of Zookeeper, needed to start Kafka, you can read how to make it [here]({% post_url 2019-09-09-Kafka-in-one-container %}).

The last component is the API Gateway, *Kong*: the classic installation uses a database like *Postgresql*, but it's also possible (for development usage) to start Kong in a declarative way, following the simple configuration of Kong *kong.yml*:

```yml
_format_version: "1.1"

services:
  - name: order-service
    url: http://order-service:8090
    routes:
      - name: order-service
        paths:
          - /order-service

  - name: kitchen-service
    url: http://kitchen-service:8080
    routes:
      - name: kitchen-service
        paths:
          - /kitchen-service

  - name: delivery-service
    url: http://delivery-service:8070
    routes:
      - name: delivery-service
        paths:
          - /delivery-service

plugins:
  - name: request-transformer
    service: kitchen-service
    config:
      add:
        headers:
          - x-forwarded-prefix:/kitchen-service

  - name: request-transformer
    service: order-service
    config:
      add:
        headers:
          - x-forwarded-prefix:/order-service

  - name: request-transformer
    service: delivery-service
    config:
      add:
        headers:
          - x-forwarded-prefix:/delivery-service
```
In this example I'm using the API Gateway in the simplest way, without any Authentication and Authorization service or Service replica or Service Discovery, etc. to avoid confusing on the main aspect: the implementation of *Choreography Saga pattern*.

To build the project, you can use *maven* and then start manually every service, or you can build everything with the multistage Dockerfile (you have to enable the *experimental features* on Docker 19.x):

```bash
docker buildx build --target=order-service -t paspaola/order-service:0.0.1 --load . &&\
docker buildx build --target=kitchen-service -t paspaola/kitchen-service:0.0.1 --load . &&\
docker buildx build --target=delivery-service -t paspaola/delivery-service:0.0.1 --load . &&\
docker buildx build --target=kong-mcpaspao -t paspaola/kong-mcpaspao:0.0.1 --load .
```

and then start with command:

```bash
docker app render -s advertised.addr="your docker host ip" mcpaspao.dockerapp| docker-compose -f - up
```

It's time to test!

You can verify that every microservice is runnig, using the Swagger user interface:

* Order Service: *http://localhost:8000/kitchen-service/swagger-ui.html*
* Kitchen Services *http://localhost:8000/order-service/swagger-ui.html*
* Delivery Services *http://localhost:8000/delivery-service/swagger-ui.html*

**Now I want an hamburger!!!** The kithcen needs some hamburgers, the fridge is empty, so (you have to install [jq](https://stedolan.github.io/jq/)):

```bash    
curl -X POST "http://localhost:8000/kitchen-service/kitchen/add?hamburgerType=KOBE&quantity=2" -H "accept: application/json"|jq -C && \
 \
curl -X GET "http://localhost:8000/kitchen-service/kitchen/status" -H "accept: application/json"|jq -C
```

I have added two hamburgers, now I make a request for an order with two hamburgers:

```bash
printf "\n--START--\n" && \
curl -X POST "http://localhost:8000/order-service/order/create" -H "accept: application/json" -H "Content-Type: application/json" -d "{ \"addressDTO\": { \"number\": \"string\", \"street\": \"string\" }, \"cookingType\": \"BLOOD\", \"hamburgerList\": [ { \"hamburgerType\": \"KOBE\", \"quantity\": 2 } ], \"price\": 10}" |jq -C && \
printf "\n---------\n" && \
 \
curl -X GET "http://localhost:8000/order-service/order/view" -H "accept: application/json"|jq -C  && sleep 5 && \
printf "\n---------\n" && \
curl -X GET "http://localhost:8000/order-service/order/view" -H "accept: application/json"|jq -C && sleep 5 && \
printf "\n---------\n" && \
curl -X GET "http://localhost:8000/order-service/order/view" -H "accept: application/json"|jq -C && sleep 5 && \
printf "\n---------\n" && \
curl -X GET "http://localhost:8000/order-service/order/view" -H "accept: application/json"|jq -C && \
printf "\n---------\n" && \
 \
curl -X GET "http://localhost:8000/delivery-service/delivery/status" -H "accept: application/json"|jq -C && \
printf "\n--END--\n"
```


In the first step the order goes in *WAITING* status, then *COOKING*, *PACKAGING* and *DELIVERED* status. If you run again the script, the system doesn't have enough hamburgers and the next order will be in status *WAITING* and then *ABORTED*.

I hope this guide will help you to clarify the power and the complexity of a microservice architecture, this is only a pratical example implemented using simple and basic components, but you can guess when use or not it. Thank you for reading.








