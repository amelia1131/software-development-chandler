# Dividing a Legacy ERP into Cloud Services using MongoDB Data Architecture

In my role as the lead architect at [Hybrid Web Agency](https://hybridwebagency.com/), one of the most challenging projects I've undertaken involved the conversion of a legacy monolithic ERP system into a modern microservices architecture. The outdated monolith had become a costly burden to maintain and was hindering my client's ability to innovate.

Fortunately, we had encountered similar challenges numerous times during our years of offering custom [Software Development Services in Chandler](https://hybridwebagency.com/chandler-az/best-software-development-company/). We understood the difficulties organizations face when dealing with rigid and hard-to-change systems. That's why I'm eager to share the step-by-step approach we took to revamp the data model and decompose the monolith, both at the code and database levels.

Throughout the migration process, my team and I uncovered some lesser-known best practices for modeling domain entities in a document database like MongoDB to support fully independent microservices. If you've ever wondered how to future-proof your database architecture for the cloud while preserving historical data, you're in for a treat with these strategies.

By the end of this article, you'll have a clear blueprint for migrating your own legacy systems to a modern architecture. I'll provide plenty of practical insights to help you avoid common pitfalls and deliver new features to your customers faster. Let's dive in!

## The Advantages of a Microservices Architecture

Implementing a microservices architecture offers numerous benefits over the traditional monolithic approach. Microservices are independently deployable, allowing for the rapid development and release of features without disrupting the entire application.

Furthermore, individual services can be developed using different programming languages and frameworks, enabling organizations to choose the best technologies for each domain. For example, a recommendation engine might leverage machine learning libraries in Python, while a user interface could be built using React.

This decoupling of concerns empowers specialized teams to work autonomously on separate services. It also facilitates rapid validation of ideas through prototypes before committing to a full-scale rewrite of the monolithic system. Additionally, new team members can quickly contribute to a single service that aligns with their skills.

In a world where technology trends evolve rapidly, microservices help mitigate risks associated with these shifts. Replacements only affect small, isolated parts of the system, rather than requiring a complete overhaul of the entire monolith. Well-defined interfaces make it seamless to migrate components to new implementations.

#### Independent Scalability

Efficient resource management is achieved when services can scale independently in response to demand. For example, a frontend API gateway can efficiently route traffic based on URLs and can securely deploy behind a load balancer to handle traffic spikes. During peak periods, such as holidays, only the order processing service needs additional servers, avoiding unnecessary scaling of the entire system.

```
ordersAPI.scale(replicas: 5);
```

Horizontal scaling at the service level leads to significant cost savings by accurately allocating resources. Idle support microservices, like user profiles, no longer require costly overprovisioning to handle traffic that doesn't impact them.

## Analyzing the Data Model

### Understanding Entity Relationships

The first step in migrating to microservices is a thorough analysis of how entities are interconnected within the monolithic data model. We meticulously examined each collection in the MongoDB database to identify clusters of domains and transaction boundaries.

Common entities like Users, Products, and Orders emerged as the central elements within bounded contexts. Relationships between these core entities became candidates for service decomposition. For instance, we observed that Orders contained foreign keys linking to Users for customer information and Products to represent purchased items.

To gain a deeper understanding of these interdependencies, we created sample documents to visualize associated fields. An important revelation was that legacy code redundantly stored data that now pertained to separate business capabilities. For example, shipping addresses needlessly duplicated user profiles instead of referencing lightweight counterparts.

Analyzing relationships exposed problematic tight couplings between modules, resulting in cascading updates. Normalizing redundant data removed obstacles to the independent development of user profiles and shipping namespaces.

We utilized database tools to explore the intricate web of connections. With MongoDB Compass, we meticulously mapped relationships through $lookup pipelines and ran aggregate queries to count references between entities. This process revealed critical breakpoints for dividing logic into coherent services.

These relationships guided the delineation of domain boundaries and ensured that services presented clean, well-defined interfaces. These well-defined contracts empowered autonomous teams to develop and deploy modules incrementally as micro frontends without hindering each other.

### Identifying Transactional Boundaries

Beyond examining relationships, we delved into transactions within the existing codebase to understand the flow of business processes. This effort helped us identify areas where data modifications needed to be confined within individual services to maintain data consistency and integrity.

For instance, in the realm of order processing, we realized that any updates related to the order itself, associated payments, inventory levels, and shipment notifications had to occur within a single service, in a transactional manner. This insight informed the delineation of our Order Management service boundary.

A meticulous analysis of both relationships and transactions provided crucial insights that guided the refactoring of the data model and logic into independently deployable microservices, each characterized by clearly defined interfaces.

## Refactoring for Microservices

### Normalizing Data Schemas

To support independent services that might deploy to different data stores if needed, we embarked on the normalization of schemas. The aim was to eliminate redundancy and include only the essential data required by each service.

For instance, the original Orders schema encompassed the entire User object. We streamlined this by introducing a lightweight reference:

```
// Before
Orders: {
  user: {
   

 name: 'John',
    address: '123 Main St...' 
  }
  //...
}

// After  
Orders: {
  userId: 1234
  //...  
}
```

Similarly, we extracted product details from Orders and placed them into their own collections. This approach allowed these entities to evolve independently over time.

### Leveraging Domain-Driven Design

We harnessed the concept of bounded contexts from Domain-Driven Design to logically segregate services, such as Order Fulfillment versus User Profiles. Interfaces played a pivotal role as they abstracted data access:

```
interface UserRepository {
  getUser(id): User;
  updateProfile(user: User): void;
}

class MongoUserRepository implements UserRepository {

  users = db.collection('users');

  async getUser(id) {
    return this.users.findOne({_id: id}); 
  }

  //...
}
```

### Evolving Data Access Patterns

Queries and commands underwent significant refactoring to align with the new architectural paradigm. In the past, services directly accessed data using calls like `db.collection.find()`. We introduced abstraction through data access libraries:

```
// order.service.ts

constructor(private ordersRepo: OrderRepository) {}

getOrders() {
  return this.ordersRepo.find();
}

// mongodb.repo.ts

@Injectable()
class MongoOrderRepository implements OrderRepository {

  constructor(private mongoClient: MongoClient) {}

  find() {
   return this.mongoClient.db.collection('orders').find(); 
  }

}
```

This ensured flexibility, allowing us to migrate databases without necessitating changes to consumer code.

## Deploying Microservices

### Independent Scaling

In the world of microservices, autoscaling takes place at the individual service level, departing from the application-wide approach. We implemented scaling logic using Docker Swarm and Kubernetes.

Deployment manifests defined scaling policies based on CPU and memory usage:

```yaml
# docker-compose.yml

services:
  orders:
    image: orders
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 512M
      restart_policy: any
      placement:
        constraints: [node.role == worker]  
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      rollback_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      scaler:
        min_replicas: 2    
        max_replicas: 6
```

Scaling tiers provided reserve capacity and elastic overflow handling. The orders service autonomously monitored its performance and spawned or terminated containers as needed:

```js
// orders.js

const cpusUsed = process.cpuUsage();

if(cpusUsed > 80) {
  swarm.scaleService('orders', 1);
}

if(cpusUsed < 20) {
  swarm.scaleService('orders', -1);  
}
```

Load balancers like Nginx and Traefik effectively routed traffic to scaled replica sets. This optimization of resource utilization bolstered throughput and concurrently reduced costs.

### Implementing Resilience

Our arsenal of resiliency techniques included retry policies, timeouts, and circuit breakers. Rate limiting and throttling acted as safeguards against cascading failures, while the Platform service took charge of transient error policies for dependent services.

Homegrown and open-source solutions such as Polly, Hystrix, and Resilience4j stood guard against potential mishaps. Centralized logging via Elasticsearch played a pivotal role in tracing errors across the distributed applications.

### Ensuring Reliability

Incorporating reliability into microservices necessitated the implementation of various techniques aimed at averting single points of failure. We prioritized automated responses to transient errors and designed measures to tackle overload scenarios.

Leveraging the Resilience4J library, we implemented circuit breakers to gracefully handle faults:

```java
// OrdersService.java

@Slf4j
@Service
public class OrdersService {

  @CircuitBreaker(name="ordersService", fallbackMethod="fallback")
  public List<Order> getOrders() {
    return orderRepository.getOrders();
  }

  public List<Order> fallback(Exception e) {
    log.error("Circuit open, returning empty orders");
    return Collections.emptyList();
  }
}
```

Rate limiting was employed to prevent service flooding during times of stress:

```java 
// RateLimiterConfig.java

@Configuration
public class RateLimiterConfig {

  @Bean
  public RateLimiter ordersLimiter() {
    return RateLimiter.create(maxOrdersPerSecond); 
  }

}
```

Timeouts were introduced to terminate lengthy calls:

```java
// ProductClient.java

@HystrixCommand(fallbackMethod="getProductFallback",
               commandProperties={
                 @HystrixProperty(name="execution.isolation.thread.timeoutInMilliseconds", value="1000")  
               })
public Product getProduct(Long id) {
  return productService.getProduct(id);
}
```

We introduced retry logic through policies defined at the client level:

```java
// RetryConfig.java

@Configuration
public class RetryConfig {

  @Bean
  public RetryTemplate ordersRetryTemplate() {
    SimpleRetryPolicy policy =

 new SimpleRetryPolicy();
    policy.setMaxAttempts(3);
    return a RetryTemplate(policy);
  }

} 
```

These techniques guaranteed consistent responses and thwarted cascading failures across the spectrum of services.

## In Conclusion

The migration of our monolithic ERP system into microservices has been an invaluable learning experience. It transcends the realm of a mere technical migration, signifying an organizational metamorphosis that equips our client to better serve their ever-evolving customer base.

By dismantling tightly coupled layers and instituting clear domain boundaries, our development team has unlocked a new level of agility. Features can now be developed and deployed independently, driven by business priorities rather than architectural constraints. This capacity for rapid experimentation and refinement positions the application to remain responsive to dynamic market demands.

Simultaneously, our operations team now enjoys full visibility and control over each system component. Anomalous behavior can be promptly detected through improved monitoring of discrete services. Scaling and failover mechanisms have transitioned from manual interventions to automated processes, fostering enhanced resilience that will serve as a reliable foundation for our client's continued growth.

While the benefits of microservices are unequivocal, embarking on such a migration is not without its challenges. By dedicating time to meticulous relationship analysis, interface definition, and abstraction introduction—rather than resorting to a crude 'rip and replace' approach—we have cultivated a flexible architecture capable of evolving in tandem with customer needs.

Above all, I am profoundly grateful for the opportunity this project has afforded us, allowing us to collaborate closely with our client on their digital transformation journey. Sharing our experiences and insights in this article is my way of paying it forward, with the hope that it empowers more businesses to embrace modernization. The rewards, both for customers and businesses, are unquestionably worth the endeavor.

## References 

- MongoDB Documentation - Official documentation on data modeling, queries, deployment, and more. [https://docs.mongodb.com/](https://docs.mongodb.com/)

- Microservices Pattern - Martin Fowler's seminal overview of microservices architecture principles. [https://martinfowler.com/articles/microservices.html](https://martinfowler.com/articles/microservices.html)

- Domain-Driven Design - Eric Evans' book introducing DDD concepts for structuring services around business domains. [https://domainlanguage.com/ddd/](https://domainlanguage.com/ddd/)

- Twelve-Factor App Methodology - Best practices for building software-as-a-service apps that are readily deployable to the cloud. [https://12factor.net/](https://12factor.net/)

- Container Journal - In-depth articles on containerization, orchestration, and cloud platforms. [https://containerjournal.com/](https://containerjournal.com/)

- Docker Documentation - Comprehensive guides for building, deploying, and managing containerized apps. [https://docs.docker.com/](https://docs.docker.com/)

- Kubernetes Documentation - Kubernetes, the leading container orchestrator, and its architecture. [https://kubernetes.io/docs/home/](https://kubernetes.io/docs/home/)
