# AmazIEEE system desgin

## Functional Requirements
- system should recieve orders from external source (customer)

- system should check inventory and availability for order items

- once order is confirmed system should allocate robots to handle it

- robots should report their health (status, location, battery_level)

- system need to handle robots failures 

- operators should see live orders status (RECIEVED, CONFIRMED, PICKING, COLLECTED, PACKED, READY, CANCELLED, FAILED)

- system should notify and alert operators about importatnt events (LOW_STOCK, ROBOT_FAILURE, DELAYED_ORDER, ROBOT_LOW_BATTERY)

## Non-Functional Requirements
- system need to have high consistency for handling orders and order items to prevent double selling

- for the analytics and monitoring some staleness wouldn't be an issue, so eventual consistency is accepted here

- system should prevent a product being sold to more than one customer if it's the last piece of that item

- system have to have fault tolerance when robots fails and report that correctly for the operator

- we need to make sure only one robot is assigned to one order item, so two robots don't get in the way of each other

- we need durability for notifications, so that we have a full history of the robot life cycle


## Data Model

### User (operator or admin)
- id (PK)
- ...other user metadata

### Inventory
- id (PK)

### InventoryZone
- id (PK)
- inventoryId (FK -> inventory.id)

### InventoryShelf
- id (PK)
- inventoryId (FK -> inventory.id)
- zoneId (FK -> zone.id)

### Product
- id (PK)
- inventoryShelfId (FK -> inventoryShelf.id)
- name
- description
- weight
- stock
- size

### Order
- id (PK)
- customerId (index)
- createdAt
- status: `RECIEVED | CONFIRMED | PICKING | COLLECTED | PACKED | READY | CANCELLED | FAILED`
- items: OrderItem[]

### OrderItem
- id (PK)
- productId (FK -> product.id)
- quantity
- orderId (FK -> order.id) (index)

### Robot
- id (PK)
- battery
- location

### Task
- id (PK)
- robotId (FK -> robot.id) (index)
- orderId (FK -> order.id) (index)
- orderItem (FK -> orderItem.id)
- status: `RECIEVED | MOVING_TO_SHELF | ITEM_PICKED | MOVE_TO_STATION | COMPLETED`
- createdAt
- updatedAt

with a unique constraint on (robotId, orderItem) so one robot handle one item at a time

### Notification
- id (PK)
- type: `LOW_STOCK | ROBOT_FAILURE | DELAYED_ORDER | ROBOT_LOW_BATTERY`

## API Design
- we handle authentication using JWT which is stateless and scales very well in distributed architecture, and we would have the `role` in the payload to identify different clients

or even better have different apis one is customer facing and the other for internal operators

- for communication with robots we could use websockets but it would be overkill, since this is not really a bidirectional communication, so for health checks, the robots should ping the server every maybe 5 seconds
and when that ping stops for 10 sec we notify the operator

for recieveing commands we could use SSE (Server-sent events) so basically the robot opens a connection to the server and the server sends back when there is a task for that robot, or we could also use polling too to get latest tasks for this robot

- `POST /orders` -> 201 order created
{
    items: OrderItem[]
}

- `GET /orders?pageSize=num&cursor=lastOrderId` -> Order[]

- `GET /robots?limit=num&offset=num` -> Robot[]

- `PATCH /robots/{robotId}` -> {success: true}
{
    battery: float,
    location,
    status
}

- `PATCH /tasks/{taskId}` -> {success: true}
{
    status: `MOVING_TO_SHELF | ITEM_PICKED | MOVE_TO_STATION | COMPLETED`
}


## High-Level Architecture
in the high-level design we split the system into multiple services
### Inventory service
- manages the warehouse inventory and product availability
- it adjusts the stock on created orders
- it should readjust the stock for cancelled orders

### Orders service
- user could easily get live orders status with polling or SSE, websockets won't be neccessary here and would be a unneeded overhead
- dispatches a message (orders.created) when a new order is created

### Robots service
- this one controls the whole robot life cycle
- the robot should poll this service to get new tasks
- this consume the `orders.created` message and runs an algorithm for getting the right robots for a job, and create tasks accordingly
- robot reports it's health by making a patch request every 5 seconds

- an order is ready when all the order's tasks are completed, this would work like this, when a task is compeleted the robot triggers an endpoint that would update that task and in the asyncronously check the other tasks for that order, and if all is completed it change the order status and notify the operator

we also have some cron jobs
- one for checking the robot battery every 10 minutes, and get the ones that are at a critical level and notify the operator

- anothe one is for checking delayed orders, it does so by checking the if the order was created more than an hour ago and it's status is not yet ready and notify the operator

### system flow
customer makes an order -> order (RECIEVED) -> check no overselling and payment successfull -> order (CONFIRMED) -> system send (PICK) task to matching robots -> robot task (RECIEVED, MOVING_TO_SHELF, ITEM_PICKED, MOVE_TO_STATION, COMPLETED) -> all robots finished -> move order to packing -> shipping

## Deep dives
- to prevent the same item from being sold twice, we used a distirbuted lock, that locks the item for that specific order, and check that lock before deducting the item again from the inventory

- we also added a unique constraint to the task `robot.id && orderItem.id` so that two robots don't conflict and ensure one item is handled by one robot

- we also presist the notifications to the database so operator can refer to them later

- as for the database I would add Cassandra for handling the robots activity since it's very good with large write throughput

- for handling the task events we could use Kafka streams, which will offer much better visiblity for operators with back in time features, and could be used for real-time viewing too, this would be a good fit for monitoring the robot health events

- when making an order we could check the items availability and adjust the stock in one transaction, to ensure consistency which is possible with postgres ACID properties but that would be with a tradeoff that the actuall inventory would be inconsistent untill the item is actually picked up by a robot, we could accept this approach with that trade-off to avoid the overhead needed to handle that time gap between the database stock and actual stock

- identifying failed orders is a little hard in this system because we have alot of transient state, at some points we could identify these, like if a robot fails, or item is missing from it's shelf and we just update the order status to failed, other than this it would be up to the admin, he could fetch orders that took so long and mark them as failed to retrigger another order, or let the customer know

## Back-Of-The-Envelope Estimation
- we could expect 10M order daily (115 write QPS)
- each order would have 3 items on average that is 30M task a day for the robots

- orders table size 10M * 1KB * 30 = 300GB / month
- tasks table size about 1TB / month
