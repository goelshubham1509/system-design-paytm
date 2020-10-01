# system-design-paytm
Design A Google Analytic like Backend System. We need to provide Google Analytic like services to our customers. Please provide a high level solution design for the backend system. Feel free to choose any open source tools as you want.

Requirements
Handle large write volume: Billions of write events per day.
Handle large read/query volume: Millions of merchants wish to gain insight into their business. Read/Query patterns are time-series related metrics.
Provide metrics to customers with at most one hour delay.
Run with minimum downtime.
Have the ability to reprocess historical data in case of bugs in the processing logic.

## Analysis of given requirements 
### Functional Requirements
- The system should gather and process events data that is generated on the client's webpage or mobile application.
- The backend system will get data from frontend apps. 
- Backend system should clean and process the data that can be presented as meaningful data to the users using reporting applications.
- We should be able to reprocess historical data.
- The input data will not be the same as the output data (for reporting) so we need services to clean, process, and transform input data into time-series format and store in a large database store. Most of the data could be unstructured so we will use NoSQL databases.

### Non-functional Requirements
- It should be highly robust and scalable to handle large volume of write events. We are expecting billions of write events.
- There could be millions of customers who would want to analyze their data in 'almost' real-time environment. We can have a maximum lag time of 1 hour only.
- System should be highly available so focus will be to achieve five nines availability. 
- While we should show almost real-time data to customers, we should also be able to show data and trends from historical data. So we need a capability to store historical time-series data for the given input volume.

## Assumptions and design decisions
- Input data is coming in from web apps and mobile apps.
- Frontend apps can produce data as byte stream.
- Our services know the algorithm to clean and transform input data into meaningful data that we will be displaying on reporting apps. I am not focusing on the data transformation algorithms here.

## High-level System Design
- Assuming the front-end apps have a capability to produce data in byte stream format or JSON format or .js files, we need to design services using streaming APIs such as Kafka Streams that ingests data to Kafka brokers. 
- I would use Apache Kafka as a event processing application here because it can capture real-time data from different sources and ingest data into Kafka topics.
- Once the messages are in Kafka topics, we need another set of services to perform data transformation, cleaning and then store in a persistent layer. Because here we are talking about billions of operations per day, I would go with a NoSQL database such as Cassandra that is highly scalable and available database system.

## Description of each component
I have marked each component in the diagram with an alphabet. Below is the description of each component:

### (A) UI Layer
- I am assuming all the traffic is coming in from customer's web apps and mobile apps. 
- We need to have UI plugins that could send the events data to backend system. 
- We can also enable customer browser’s session cookies to send some more customer information. 

### (B) Istio Service Layer 
- The entire backend system will be protected by Istio security protocols. We will deploy Istio on Kubernetes cluster.
- Istio will provide features such as load balancing, traffic routing, and service-to-service authentication. 
- Entire system will be secured from outside traffic and internal services will have secure communication. 
- Istio will also provide features such as service discovery that is required for this system which has multiple services.
- We can make use of dashboard tools such as Kiali to view how each backend component is integrated with each other and how it is performing.

### (C) Event Processing Services
- We will be building cloud-native microservice which will act as an event listener application. 
- When the UI layer component (A) sends data, event listener will be executed. 
- This layer will make use of Kafka producer APIs to send messages to Kafka topics.
- We will have this service deployed as pods in K8 cluster for high scalability and ease of deployment.

### (D) Kafka Brokers
- Because we are talking about billions of events per day, we will use Kafka that will provide high-throughput, low-latency platform for building real-time event streaming solution.
- Kafka provides a robust infrastructure that has high throughput. 

### (E) Set of services to do data transformation and ingest data into high scale database such as Cassandra
- At this layer, we need to build services that could perform data transformation and send the cleaned data to Cassandra cluster.
- Services at this layer will implement intelligent algorithms to crunch the input data, filter out what is not important and transfer the useful data into time-series events data that could be stored in Cassandra tables.

### (F) Cassandra Layer 
- We can implement a database persistence layer such as Cassandra to store very high volume of time-series data.
- Data in Cassandra is stored as a set of rows that are organized into tables. It provides the ease of storing data the way we do in relational DB. 
- It does the data replication for us, so we know our data is replicated and highly available. 

### (G) Event listener service to create AMQP messages to send data to reporting services
- In component F, we have stored time-series data that we are going to use to create meaningful analytical reports for the customers. But the question is how to use this data from Cassandra. Making rest call to fetch data from Cassandra for each operation could be a time intensive operation and may not work with a large customer base. So, we need to have database layer closer to reporting websites to decrease the fetch time.
- In order to do so, we will write another set of event listener service that will listen to changes (new data insertions) happening in Cassandra layer, and they will generate messages and send them to message service. We can use messaging technologies such as Rabbit MQ.

### (H) Message Queues - RabbitMQ
- In the above layer (G), event processing service will publish event messages to AMQP topics. 
- There are different ways to subscribe to messages in AMQP topics. 
- The reason we can use AMQP technology here is that it holds the messages until they are delivered and it guarantees the delivery of messages. 

### (I) Report Adapter Service
- These are the services that will expose REST endpoints that reporting UI will use to get reporting data. 
- Each reporting microservice, will be cloud-native application deployed in a containerized environment using Kubernetes. 
- These services subscribe to AMQP exchange and topics from which it will continuously receive data and persist into a temporary cache layer (next component) 
- We can use caching solutions such a Reddis, Postgres etc. These databases will be attached services to reporting microservices.
- These adapter will also expose internal REST endpoint to trigger resync mechanism to reprocess the historical data in case of bugs (component L)

### (J) Cache Layer
I will implement a caching layer using databases such as Redis, postgres etc. These DBs will be closer to UI apps and will store only a limited amount of data. One important to note here is that these cache databases will store only recent data, for example, 1 day of data or 1 week of data based on the use-case that is most common for a user to view in their reports. 
Because we have a continuous flow of data generated by customer web pages/mobile app (component A), and that data is continuously being transformed/cleaned and persisted in Cassandra DB (Component F), the same amount of data will also flow in these Cache DBs, so we need an additional implementation, for example nightly jobs to clear the cache data.

### (K) Reporting Apps that customers will use to view analytical data
These apps will comprise of Software-as-a-Service based web apps or mobile apps that will make REST calls to fetch the requested report data. We might be displaying data in layered format which basically means dashboards with a small snapshot of data and then the details grids to display a complete set of data.

### (L) Resync Process - An ability to reprocess historical data in case of bugs in the processing logic
Our adapter microservices will also have an ability to execute an on-demand resync process that will fetch the required set of data from Cassandra (time-series store) directly instead of fetching from local cache DB. Such an ability could be provided by exposing REST api that the system admin will have access to and can execute when there are bugs or data loss issue and using this process, we can resync the local cache DB. Let's say, we have our pipelines in Jenkins then we can also implement Jenkins jobs to trigger such a resync process.


## Best design aspects taken care of
- Logging: Logging is a very important aspect of backend systems. We can use tools such as Spring Slueth or Elastic search, Logstash, Kibana stack to implement logging. 
- We will be using DevOps and cloud-native architecture to design and build our applications 
- We will use 12-factor app design principals while designing and developing our microservices
- We will focus on data replication to store multiple copies of data to minimize data loss
- We will be good focus on security aspects, we will use technologies such as Istio security, REST OAuth 2, firewalls to protect our internal services
- High focus on scalability. Making use of Kubernetes environment will make deployment and scaling up or down easy
- We will provide low response time to customers by storing most frequently viewed data in local cache DBs
