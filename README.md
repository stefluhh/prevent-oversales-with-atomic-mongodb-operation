# How We Identified and Fixed Over-Selling in an Online Shop with Very High Order Volumes

One day in 2022, we offered the new PlayStation 5 for sale in our online shop. The demand for the PS5 was immense and was, in fact, the most significant checkout traffic I have ever witnessed. We experienced up to 20,000 requests per second on the Checkout microservice and had to scale it to around 120 replica with 2 CPU each. With GCP and Kubernetes this isn’t really a problem, but: After releasing about 1,000 units of stock for sale, it quickly became evident that there was an issue with over-selling in a partner team, responsible for the inventory microservice. 
Slightly more than 1050 units were sold, i.e. 50 pieces were oversold. This had quickly pulled management attention, so me and another colleague were supporting in analyzing this issue.

## Our findings

The codebase was quite dated and was only maintained without any further active development. What we encountered was a colorful potpourri of microservices (a total of nine), technologies, and antipatterns. For instance, Spring Repositories were not used consistently, and we found MongoDB queries executed via `MongoTemplate` embedded directly within code were it simply doesn't belong.

After several hours of analysis and reproducing the over-selling issue using jMeter tests, I was finally able to pinpoint the problem (here’s a highly simplified code example):

```java
public void findAndReduceStock(String productId, int qty) {
  int currentAvailability = availabilityRepository.findByProductId(productId);

  if (currentAvailability < qty) {
    throw new OutOfStockException("No sufficient stock found for product " + productId);
  }

  //
  // 1. Potential long running task
  //

  // 2. Non atomic use of MongoTemplate#save operation
  availabilityRepository.save(productId, currentAvailability - qty);
}
```

In an onlineshop that has hundrets of thousands of SKU's, this method is doing fine. 
However at the time the Playstation 5's perceived a flood of purchases, it became apparent this code caused the overselling. In order to understand the reason better, we have to do a little math first: 
The PS5 product was 1 SKU only. We had a peak purchase traffic of 80 orders per second on that SKU. On average, that is 1 sale every 12.5ms. If we imagine this code construction (in reality consisting of a lot of 
orchestration, services, config-repositories involved etc), being called 80 times per second in parallel, it becomes clear why there were these massive oversales:

### Read-Modify-Write Race Condition:
- The method fetches the current stock level (`currentAvailability`) using `availabilityRepository.findByProductId(productId)` in step 1.
- Before the stock reduction is saved in `availabilityRepository.save`, there’s a time window during which other threads or requests might also read the same `currentAvailability`.
- Each thread processes the logic independently and reduces the stock based on the same initial value, leading to multiple reductions beyond the available stock.

### Non-Atomic Operation:
- The critical operations (checking stock availability and reducing it) are not performed atomically.
- The `availabilityRepository.save` operation is separate and does not guarantee consistency in a concurrent environment. MongoDB itself doesn’t enforce transactional guarantees for this logic unless explicitly configured with transactions.

## Example Scenario

- **Initial stock**: `currentAvailability = 1`.
- One request every 12.5ms, all attempting to purchase 1 unit:
  1. 1st request reads `currentAvailability = 1`.
  2. 1st request passes the `if (currentAvailability < qty)` check because `1 >= 1`.
  3. 1st request enters into the `potential long running task`-block.
  4. 2nd request (on another thread) reads 'currentAvailability = 1'.
  5. 2nd request passes the `if (currentAvailability < qty)` check because `1 >= 1`.
  6. 2nd request enters into the `potential long running task`-block.
  7. 1st request reduces stock by 1 and saves `currentAvailability = 0`.
  8. 2nd request also reduces stock by 1, unaware of 1st request's action, and saves `currentAvailability = 0` as well.

As a result, the system is overselling, the more concurrency I configured in jMeter the worse it got.

## Solution

There are a couple of solution approaches to solve this, however the solution we chose is the one that MongoDB ships natively and is realtively simple: [db.collection.findAndModify](https://www.mongodb.com/docs/manual/reference/method/db.collection.findAndModify/). The important detail can be found in the docs:

> When modifying a single document, both db.collection.findAndModify() and the updateOne() method atomically update the document.

`findAndModify` takes a `Query` and `Update` statement and both operations are executed on native-DB level as an atomic operation, i.e. you can achieve the logic we needed: "Only reduce stock if demand is fulfillable`:

```java
public void findAndReduceStock(String productId, int qty) {
  Criteria criteria = Criteria.where("productId").is(productId)
                              .and("onHandStock").gte(qty);

  Update update = new Update().inc("onHandStock", -1 * qty);
  ItemAvailability availability = mongoTemplate.findAndModify(new Query(criteria), update, ItemAvailability.class);
  if (result == null) {
    throw new OutOfStockException("No sufficient stock found for product " + productId);
  }
}
```

## Key Takeaways

1. **Know Your Use Cases**: Understand the specific scenarios your system needs to handle, especially under high concurrency or load.
2. **Load Test Thoroughly**: Simulate realistic traffic and edge cases to identify potential issues, such as race conditions or bottlenecks, before they occur in production.

The issue was relatively simple in nature and occurred mainly because the project had been implemented several years ago under time pressure and with suboptimal requirements management. The primary use cases were not sufficiently specified, and due to the time constraints, there was no opportunity for load testing the application towards the end of the development cycle. 

In hindsight, one could argue that it should have been clear from the beginning that this code structure wouldn’t withstand high load, but diving deeper into that would only be hindsight bias (it’s always easier to be wise after the event). Mistakes are human, and the best way to prevent such severe issues with significant business impact is through thorough and clean testing.




