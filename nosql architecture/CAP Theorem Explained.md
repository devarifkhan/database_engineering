# CAP Theorem Explained

## Overview

The CAP Theorem (also called Brewer's Theorem) states that a distributed data store can only guarantee two out of three properties simultaneously:

- **C**onsistency
- **A**vailability  
- **P**artition Tolerance

## The Three Properties

### Consistency (C)
Every read receives the most recent write or an error. All nodes see the same data at the same time.

```
Client writes X=5 to Node A
↓
All nodes immediately reflect X=5
↓
Client reads from Node B → Gets X=5 (consistent)
```

**Example:**
```python
# Strong consistency
client.write('balance', 100)  # Write to Node A
time.sleep(0.001)
value = client.read('balance')  # Read from Node B
# value = 100 (guaranteed latest value)
```

### Availability (A)
Every request receives a response (success or failure), without guarantee that it contains the most recent write. The system remains operational.

```
Client sends request to any node
↓
Node responds (even if data is stale)
↓
No timeouts, no errors
```

**Example:**
```python
# High availability
client.write('balance', 100)  # Write to Node A
value = client.read('balance')  # Read from Node B immediately
# value = 90 (old value, but got a response)
```

### Partition Tolerance (P)
The system continues to operate despite network partitions (communication breakdowns between nodes).

```
Network Split:
┌─────────┐     X     ┌─────────┐
│ Node A  │ ←──/──→   │ Node B  │
└─────────┘           └─────────┘
Can't communicate, but both still serve requests
```

**Example:**
```python
# Network partition occurs
# Node A and Node B can't communicate
# System continues operating (partition tolerance)
client.write('balance', 100)  # Goes to Node A
client.read('balance')         # Goes to Node B (may be stale)
```

## Why Only Two?

### Scenario: Network Partition Occurs

```
Initial State:
┌─────────────┐         ┌─────────────┐
│   Node A    │ ←─────→ │   Node B    │
│   X = 10    │         │   X = 10    │
└─────────────┘         └─────────────┘

Network Partition:
┌─────────────┐    X    ┌─────────────┐
│   Node A    │ ←──/──→ │   Node B    │
│   X = 10    │         │   X = 10    │
└─────────────┘         └─────────────┘

Client writes X = 20 to Node A
```

**Now you must choose:**

#### Option 1: CP (Consistency + Partition Tolerance)
```
Node A: X = 20 (updated)
Node B: X = 10 (can't sync due to partition)

Read request to Node B:
→ Return error (sacrifice availability)
→ Ensures consistency (no stale data)
```

#### Option 2: AP (Availability + Partition Tolerance)
```
Node A: X = 20 (updated)
Node B: X = 10 (can't sync due to partition)

Read request to Node B:
→ Return X = 10 (sacrifice consistency)
→ Ensures availability (always responds)
```

#### Option 3: CA (Consistency + Availability)
```
Not possible in distributed systems!
Network partitions will happen.
Can't guarantee both C and A when partition occurs.
```

## CAP Trade-offs

### CP Systems (Consistency + Partition Tolerance)
**Sacrifice: Availability**

When partition occurs, reject requests to maintain consistency.

```
┌─────────────┐    X    ┌─────────────┐
│   Node A    │ ←──/──→ │   Node B    │
│   X = 20    │         │   X = 10    │
└─────────────┘         └─────────────┘

Read from Node B → Error (503 Service Unavailable)
Better to fail than return wrong data
```

**Examples:**
- MongoDB (with default settings)
- HBase
- Redis (with wait command)
- Zookeeper
- Consul

**Use Cases:**
- Banking systems
- Financial transactions
- Inventory management
- Any system where stale data is unacceptable

**Code Example:**
```python
# MongoDB with majority write concern (CP)
from pymongo import MongoClient

client = MongoClient()
db = client.mydb

# Write with majority acknowledgment
db.accounts.update_one(
    {'_id': 'account1'},
    {'$set': {'balance': 1000}},
    write_concern={'w': 'majority'}  # Wait for majority of nodes
)

# If partition occurs and can't reach majority:
# → Write fails (maintains consistency)
# → Sacrifices availability
```

### AP Systems (Availability + Partition Tolerance)
**Sacrifice: Consistency**

When partition occurs, serve requests with potentially stale data.

```
┌─────────────┐    X    ┌─────────────┐
│   Node A    │ ←──/──→ │   Node B    │
│   X = 20    │         │   X = 10    │
└─────────────┘         └─────────────┘

Read from Node B → Returns X = 10 (stale but available)
System stays operational
```

**Examples:**
- Cassandra
- DynamoDB
- CouchDB
- Riak
- Voldemort

**Use Cases:**
- Social media feeds
- Product catalogs
- Caching layers
- Analytics dashboards
- Systems where eventual consistency is acceptable

**Code Example:**
```python
# Cassandra (AP system)
from cassandra.cluster import Cluster

cluster = Cluster(['node1', 'node2', 'node3'])
session = cluster.connect('mykeyspace')

# Write with ONE consistency (fast, available)
session.execute(
    "UPDATE users SET balance = 1000 WHERE id = 'user1'",
    consistency_level='ONE'  # Only wait for one node
)

# If partition occurs:
# → Write succeeds on available nodes
# → Other nodes get update later (eventual consistency)
# → Maintains availability
```

### CA Systems (Consistency + Availability)
**Sacrifice: Partition Tolerance**

Only works in single-node or non-distributed systems.

```
┌─────────────┐
│  Single DB  │  No network = No partition
└─────────────┘

Always consistent and available
But can't scale horizontally
```

**Examples:**
- Traditional RDBMS (single instance)
- PostgreSQL (single node)
- MySQL (single node)

**Reality:**
- In distributed systems, network partitions WILL happen
- CA is not a realistic choice for distributed databases
- Must choose between CP or AP

## Real-World Examples

### Example 1: Banking System (CP)
```python
# Transfer money between accounts
def transfer(from_account, to_account, amount):
    # Must be consistent - can't have money disappear or duplicate
    
    with transaction():
        balance = get_balance(from_account)
        
        if balance < amount:
            raise InsufficientFunds
        
        # Both operations must succeed or fail together
        debit(from_account, amount)   # -100
        credit(to_account, amount)    # +100
    
    # If partition occurs during transaction:
    # → Transaction fails (unavailable)
    # → Maintains consistency (no partial transfers)
```

### Example 2: Social Media Feed (AP)
```python
# Post to social media
def create_post(user_id, content):
    post = {
        'id': generate_id(),
        'user_id': user_id,
        'content': content,
        'timestamp': now()
    }
    
    # Write to available nodes
    db.posts.insert(post, write_concern='ONE')
    
    # If partition occurs:
    # → Post saved on some nodes
    # → Other nodes get it later (eventual consistency)
    # → User sees success immediately (available)
    # → Some followers might see post delayed (acceptable)

def get_feed(user_id):
    # Read from any available node
    posts = db.posts.find({'user_id': user_id}, read_preference='NEAREST')
    
    # Might get slightly stale data, but always responds
    return posts
```

### Example 3: E-commerce Inventory (CP vs AP)

**CP Approach (Accurate inventory):**
```python
def purchase_item(item_id, quantity):
    # Must be accurate - can't oversell
    with transaction():
        stock = get_stock(item_id)
        
        if stock < quantity:
            return "Out of stock"
        
        # Decrement stock atomically
        update_stock(item_id, stock - quantity)
        create_order(item_id, quantity)
    
    # If partition: reject order (unavailable)
    # Better than selling items you don't have
```

**AP Approach (Optimistic inventory):**
```python
def purchase_item(item_id, quantity):
    # Optimistically accept order
    create_order(item_id, quantity)
    decrement_stock(item_id, quantity)
    
    # If partition: order succeeds
    # Might oversell, handle later with:
    # - Backorders
    # - Cancellations
    # - Apologies + discounts
    
    # Better user experience, occasional issues
```

## Eventual Consistency (AP Systems)

### How It Works
```
Time: T0
┌─────────────┐         ┌─────────────┐
│   Node A    │ ←─────→ │   Node B    │
│   X = 10    │         │   X = 10    │
└─────────────┘         └─────────────┘

Time: T1 (Write to Node A)
┌─────────────┐    X    ┌─────────────┐
│   Node A    │ ←──/──→ │   Node B    │
│   X = 20    │         │   X = 10    │
└─────────────┘         └─────────────┘

Time: T2 (Partition heals)
┌─────────────┐         ┌─────────────┐
│   Node A    │ ←─────→ │   Node B    │
│   X = 20    │  sync   │   X = 10    │
└─────────────┘    →    └─────────────┘

Time: T3 (Eventually consistent)
┌─────────────┐         ┌─────────────┐
│   Node A    │ ←─────→ │   Node B    │
│   X = 20    │         │   X = 20    │
└─────────────┘         └─────────────┘
```

### Conflict Resolution Strategies

#### 1. Last Write Wins (LWW)
```python
# Use timestamp to resolve conflicts
Node A: X = 20 (timestamp: 1000)
Node B: X = 15 (timestamp: 1001)

Result: X = 15 (newer timestamp wins)
```

#### 2. Version Vectors
```python
# Track causality
Node A: X = 20 [A:1, B:0]
Node B: X = 15 [A:0, B:1]

Result: Conflict detected, application resolves
```

#### 3. Application-Level Resolution
```python
# Shopping cart example
Node A: cart = ['item1', 'item2']
Node B: cart = ['item1', 'item3']

Result: Merge → cart = ['item1', 'item2', 'item3']
```

## PACELC Theorem (Extended CAP)

CAP only describes behavior during partitions. PACELC extends this:

**If Partition (P):**
- Choose Availability (A) or Consistency (C)

**Else (E):**
- Choose Latency (L) or Consistency (C)

### Examples

**PA/EL (Cassandra, DynamoDB):**
- Partition: Choose Availability
- Normal: Choose Low Latency
- Eventual consistency

**PC/EC (HBase, MongoDB):**
- Partition: Choose Consistency
- Normal: Choose Consistency
- Strong consistency, higher latency

**PA/EC (Cosmos DB with eventual consistency):**
- Partition: Choose Availability
- Normal: Choose Consistency
- Configurable

## Choosing the Right System

### Choose CP When:
- Data accuracy is critical
- Stale data causes problems
- Financial transactions
- Inventory management
- Booking systems
- Medical records

### Choose AP When:
- Availability is critical
- Stale data is acceptable
- Social media
- Content delivery
- Analytics
- Caching
- Shopping carts

## Common Misconceptions

### Myth 1: "CA systems exist in distributed environments"
**Reality:** Network partitions will happen. Must choose CP or AP.

### Myth 2: "CAP is binary"
**Reality:** It's a spectrum. Can tune consistency levels.

```python
# Cassandra - tunable consistency
# Strong consistency (CP-like)
session.execute(query, consistency_level='QUORUM')

# Weak consistency (AP-like)
session.execute(query, consistency_level='ONE')
```

### Myth 3: "AP means no consistency"
**Reality:** AP means eventual consistency, not no consistency.

### Myth 4: "Must choose one forever"
**Reality:** Can choose per operation.

```python
# Critical operation - strong consistency
db.update(user, write_concern='majority')

# Non-critical operation - weak consistency
db.update(view_count, write_concern='ONE')
```

## Database Classification

### CP Databases
- MongoDB (default)
- HBase
- Redis (with WAIT)
- Zookeeper
- Consul
- Etcd

### AP Databases
- Cassandra
- DynamoDB
- Riak
- CouchDB
- Voldemort

### Tunable (CP ↔ AP)
- Cosmos DB
- Cassandra (with consistency levels)
- MongoDB (with read/write concerns)

## Practical Example: Distributed Counter

### CP Implementation
```python
def increment_counter(counter_id):
    # Lock counter across all nodes
    with distributed_lock(counter_id):
        value = read_from_majority(counter_id)
        new_value = value + 1
        write_to_majority(counter_id, new_value)
    
    # If partition: operation fails (unavailable)
    # Guarantees: accurate count
```

### AP Implementation
```python
def increment_counter(counter_id):
    # Increment on local node
    local_increment(counter_id)
    
    # Async replicate to other nodes
    async_replicate(counter_id)
    
    # If partition: operation succeeds (available)
    # Guarantees: eventually accurate count
    # Trade-off: temporary inaccuracy
```

## Software Engineering Example: Building a Ride-Sharing App

Let's build a ride-sharing application (like Uber/Lyft) and see how CAP theorem applies to different features.

### System Architecture
```
┌──────────────────────────────────────────────────┐
│           Ride-Sharing Application               │
├──────────────────────────────────────────────────┤
│  - Driver Location Tracking                      │
│  - Ride Matching                                 │
│  - Payment Processing                            │
│  - User Profiles                                 │
│  - Ride History                                  │
└──────────────────────────────────────────────────┘
         ↓              ↓              ↓
    ┌─────────┐    ┌─────────┐    ┌─────────┐
    │ Node 1  │    │ Node 2  │    │ Node 3  │
    │ US-East │    │ US-West │    │ Europe  │
    └─────────┘    └─────────┘    └─────────┘
```

### Feature 1: Driver Location Tracking (AP System)

**Requirement:** Track thousands of drivers' locations in real-time.

**Why AP?**
- Availability is critical (drivers must always be visible)
- Slight location staleness is acceptable (1-2 seconds old)
- High write throughput (location updates every second)

```python
from cassandra.cluster import Cluster
import time

class DriverLocationService:
    def __init__(self):
        self.cluster = Cluster(['node1', 'node2', 'node3'])
        self.session = self.cluster.connect('rideshare')
    
    def update_location(self, driver_id, lat, lng):
        """
        Update driver location - AP system
        Prioritizes availability over consistency
        """
        query = """
            INSERT INTO driver_locations (driver_id, lat, lng, timestamp)
            VALUES (%s, %s, %s, %s)
        """
        
        # Write to ONE node (fast, always available)
        self.session.execute(
            query,
            (driver_id, lat, lng, time.time()),
            consistency_level='ONE'  # AP choice
        )
        
        # Result:
        # ✓ Always succeeds (available)
        # ✓ Fast response (low latency)
        # ✗ Other nodes might have stale location briefly
        # ✓ Eventually consistent (acceptable for this use case)
    
    def get_nearby_drivers(self, user_lat, user_lng, radius_km):
        """
        Find nearby drivers - read from nearest node
        """
        query = """
            SELECT driver_id, lat, lng, timestamp
            FROM driver_locations
            WHERE lat > %s AND lat < %s
              AND lng > %s AND lng < %s
        """
        
        # Read from nearest node (fast)
        results = self.session.execute(
            query,
            (user_lat - radius_km, user_lat + radius_km,
             user_lng - radius_km, user_lng + radius_km),
            consistency_level='ONE'  # AP choice
        )
        
        # Result:
        # ✓ Fast response
        # ✓ Always available
        # ✗ Might miss a driver who just came online
        # ✓ Acceptable - will see them in next refresh (1-2 sec)
        
        return list(results)

# Usage
service = DriverLocationService()

# Driver app updates location every second
while driver_is_active:
    current_location = gps.get_location()
    service.update_location(
        driver_id='driver_123',
        lat=current_location.lat,
        lng=current_location.lng
    )
    time.sleep(1)

# User app finds nearby drivers
nearby = service.get_nearby_drivers(
    user_lat=37.7749,
    user_lng=-122.4194,
    radius_km=5
)
print(f"Found {len(nearby)} drivers nearby")
```

**What happens during network partition?**
```
Scenario: US-East and US-West nodes can't communicate

Driver in SF updates location → Goes to US-West node ✓
User in SF searches → Reads from US-West node ✓
User in NY searches → Reads from US-East node (stale) ✗

Result:
- System stays available (AP)
- SF users see correct data
- NY users see slightly stale SF driver locations
- Acceptable trade-off for this feature
```

### Feature 2: Payment Processing (CP System)

**Requirement:** Process ride payments accurately.

**Why CP?**
- Consistency is critical (can't charge twice or lose money)
- Brief unavailability is acceptable (user can retry)
- Financial accuracy > availability

```python
from pymongo import MongoClient
from decimal import Decimal

class PaymentService:
    def __init__(self):
        self.client = MongoClient(
            'mongodb://node1,node2,node3/?replicaSet=rs0'
        )
        self.db = self.client.rideshare
    
    def process_payment(self, ride_id, user_id, driver_id, amount):
        """
        Process payment - CP system
        Prioritizes consistency over availability
        """
        session = self.client.start_session()
        
        try:
            with session.start_transaction(
                write_concern={'w': 'majority'}  # CP choice
            ):
                # 1. Check user balance
                user = self.db.users.find_one(
                    {'_id': user_id},
                    session=session
                )
                
                if user['balance'] < amount:
                    raise InsufficientFunds("Not enough balance")
                
                # 2. Deduct from user
                self.db.users.update_one(
                    {'_id': user_id},
                    {'$inc': {'balance': -amount}},
                    session=session
                )
                
                # 3. Credit to driver
                self.db.drivers.update_one(
                    {'_id': driver_id},
                    {'$inc': {'balance': amount}},
                    session=session
                )
                
                # 4. Record transaction
                self.db.transactions.insert_one({
                    'ride_id': ride_id,
                    'user_id': user_id,
                    'driver_id': driver_id,
                    'amount': amount,
                    'status': 'completed',
                    'timestamp': time.time()
                }, session=session)
                
                # Commit transaction
                session.commit_transaction()
                
                # Result:
                # ✓ All operations succeed or all fail (consistent)
                # ✓ No partial payments
                # ✓ Accurate balances
                # ✗ Might fail if partition occurs (unavailable)
                
                return {'status': 'success', 'transaction_id': '...'}
                
        except Exception as e:
            session.abort_transaction()
            
            # If partition occurs:
            # → Transaction fails
            # → User sees error: "Payment failed, please retry"
            # → Better than incorrect payment
            
            raise PaymentError(f"Payment failed: {e}")
        
        finally:
            session.end_session()

# Usage
payment_service = PaymentService()

try:
    result = payment_service.process_payment(
        ride_id='ride_789',
        user_id='user_123',
        driver_id='driver_456',
        amount=Decimal('25.50')
    )
    print("Payment successful!")
except PaymentError as e:
    # Show error to user
    print(f"Payment failed: {e}")
    print("Please try again")
    # User can retry - temporary unavailability is acceptable
```

**What happens during network partition?**
```
Scenario: Can't reach majority of nodes

User completes ride → Payment initiated
↓
Can't reach majority of MongoDB nodes
↓
Transaction fails (CP choice)
↓
User sees: "Payment failed, please retry"

Result:
- System becomes unavailable (CP)
- No incorrect charges
- No money lost
- User retries when partition heals
- Correct trade-off for financial data
```

### Feature 3: User Profile (Tunable)

**Requirement:** Store and retrieve user profiles.

**Why Tunable?**
- Different operations have different requirements
- Profile updates: can be CP (consistency matters)
- Profile reads: can be AP (availability matters)

```python
class UserProfileService:
    def __init__(self):
        self.client = MongoClient(
            'mongodb://node1,node2,node3/?replicaSet=rs0'
        )
        self.db = self.client.rideshare
    
    def update_profile(self, user_id, updates):
        """
        Update profile - CP (consistency important)
        User expects their changes to be saved correctly
        """
        result = self.db.users.update_one(
            {'_id': user_id},
            {'$set': updates},
            write_concern={'w': 'majority'}  # CP: wait for majority
        )
        
        if result.modified_count == 0:
            raise UpdateError("Profile update failed, please retry")
        
        # Result:
        # ✓ Changes are durable
        # ✓ Won't lose user's edits
        # ✗ Might fail during partition (acceptable - user retries)
        
        return {'status': 'success'}
    
    def get_profile(self, user_id):
        """
        Get profile - AP (availability important)
        Slightly stale profile is acceptable
        """
        profile = self.db.users.find_one(
            {'_id': user_id},
            read_preference='nearest'  # AP: read from nearest node
        )
        
        # Result:
        # ✓ Always available
        # ✓ Fast response
        # ✗ Might be slightly stale (acceptable)
        # Example: Profile photo updated 1 second ago might not show yet
        
        return profile
    
    def get_profile_for_payment(self, user_id):
        """
        Get profile for payment - CP (consistency critical)
        Need accurate payment method info
        """
        profile = self.db.users.find_one(
            {'_id': user_id},
            read_preference='primary',  # CP: read from primary
            read_concern={'level': 'majority'}  # CP: majority read
        )
        
        # Result:
        # ✓ Guaranteed up-to-date payment info
        # ✗ Might fail during partition
        # ✓ Correct trade-off for payment operations
        
        return profile

# Usage
profile_service = UserProfileService()

# Scenario 1: User updates profile picture (CP)
try:
    profile_service.update_profile(
        user_id='user_123',
        updates={'profile_picture': 'new_photo.jpg'}
    )
    print("Profile updated!")
except UpdateError:
    print("Update failed, please retry")

# Scenario 2: Display profile in app (AP)
profile = profile_service.get_profile('user_123')
show_profile_ui(profile)  # Might be slightly stale, acceptable

# Scenario 3: Process payment (CP)
profile = profile_service.get_profile_for_payment('user_123')
process_payment(profile['payment_method'])  # Must be accurate
```

### Feature 4: Ride History (AP System)

**Requirement:** Store and display past rides.

**Why AP?**
- Availability is important (users want to see history anytime)
- Eventual consistency is fine (ride completed 1 second ago can appear with slight delay)
- High read throughput

```python
class RideHistoryService:
    def __init__(self):
        self.cluster = Cluster(['node1', 'node2', 'node3'])
        self.session = self.cluster.connect('rideshare')
    
    def save_ride(self, ride_data):
        """
        Save completed ride - AP system
        """
        query = """
            INSERT INTO ride_history 
            (ride_id, user_id, driver_id, start_time, end_time, 
             fare, start_location, end_location)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s)
        """
        
        self.session.execute(
            query,
            (ride_data['ride_id'], ride_data['user_id'], 
             ride_data['driver_id'], ride_data['start_time'],
             ride_data['end_time'], ride_data['fare'],
             ride_data['start_location'], ride_data['end_location']),
            consistency_level='ONE'  # AP: fast, available
        )
        
        # Result:
        # ✓ Always succeeds
        # ✓ Fast
        # ✗ Might not be visible immediately on all nodes
        # ✓ Will appear within seconds (acceptable)
    
    def get_user_rides(self, user_id, limit=20):
        """
        Get user's ride history - AP system
        """
        query = """
            SELECT * FROM ride_history
            WHERE user_id = %s
            ORDER BY end_time DESC
            LIMIT %s
        """
        
        results = self.session.execute(
            query,
            (user_id, limit),
            consistency_level='ONE'  # AP: fast, available
        )
        
        # Result:
        # ✓ Always available
        # ✓ Fast response
        # ✗ Might miss very recent ride (appears in 1-2 seconds)
        # ✓ Acceptable for history view
        
        return list(results)

# Usage
history_service = RideHistoryService()

# Save ride after completion
history_service.save_ride({
    'ride_id': 'ride_789',
    'user_id': 'user_123',
    'driver_id': 'driver_456',
    'start_time': '2024-01-01 10:00:00',
    'end_time': '2024-01-01 10:25:00',
    'fare': 25.50,
    'start_location': 'Downtown',
    'end_location': 'Airport'
})

# User opens ride history
rides = history_service.get_user_rides('user_123')
for ride in rides:
    print(f"Ride to {ride.end_location}: ${ride.fare}")
```

### Complete System Design Decision Matrix

```python
# Ride-Sharing App - CAP Decisions

FEATURES = {
    'driver_location': {
        'system': 'AP (Cassandra)',
        'reason': 'Availability critical, staleness acceptable',
        'consistency': 'Eventual',
        'partition_behavior': 'Stay available, serve stale data'
    },
    
    'payment_processing': {
        'system': 'CP (MongoDB with majority)',
        'reason': 'Accuracy critical, brief unavailability OK',
        'consistency': 'Strong',
        'partition_behavior': 'Become unavailable, maintain accuracy'
    },
    
    'user_profile_read': {
        'system': 'AP (MongoDB with nearest)',
        'reason': 'Fast display, slight staleness OK',
        'consistency': 'Eventual',
        'partition_behavior': 'Stay available'
    },
    
    'user_profile_write': {
        'system': 'CP (MongoDB with majority)',
        'reason': 'User expects changes saved correctly',
        'consistency': 'Strong',
        'partition_behavior': 'Fail write, user retries'
    },
    
    'ride_history': {
        'system': 'AP (Cassandra)',
        'reason': 'Always accessible, eventual consistency fine',
        'consistency': 'Eventual',
        'partition_behavior': 'Stay available'
    },
    
    'ride_matching': {
        'system': 'CP (MongoDB with majority)',
        'reason': 'Can\'t match same driver to two riders',
        'consistency': 'Strong',
        'partition_behavior': 'Fail match, user retries'
    }
}
```

### Real-World Partition Scenario

```python
# Network partition occurs between US-East and US-West

class PartitionScenario:
    """
    Simulating what happens during a real network partition
    """
    
    def scenario_driver_location(self):
        """
        Driver location tracking during partition (AP)
        """
        print("=== Driver Location (AP System) ===")
        print("Partition occurs between US-East and US-West")
        print("")
        
        # Driver in SF (US-West)
        print("Driver in SF updates location:")
        print("  → Writes to US-West node ✓")
        print("  → US-East node doesn't get update (partition)")
        print("")
        
        # User in SF (US-West)
        print("User in SF searches for drivers:")
        print("  → Reads from US-West node ✓")
        print("  → Sees SF driver ✓")
        print("  → System available ✓")
        print("")
        
        # User in NY (US-East)
        print("User in NY searches for SF drivers:")
        print("  → Reads from US-East node ✓")
        print("  → Sees stale SF driver locations ✗")
        print("  → System still available ✓")
        print("  → Impact: Minor (locations update every second anyway)")
        print("")
        
        print("Result: AP choice correct for this feature")
        print("")
    
    def scenario_payment(self):
        """
        Payment processing during partition (CP)
        """
        print("=== Payment Processing (CP System) ===")
        print("Partition occurs, can't reach majority of nodes")
        print("")
        
        print("User completes ride, payment initiated:")
        print("  → Tries to write to majority of nodes")
        print("  → Can't reach majority (partition) ✗")
        print("  → Transaction aborted ✗")
        print("  → User sees: 'Payment failed, please retry'")
        print("")
        
        print("Alternative (if we chose AP):")
        print("  → Payment succeeds on available nodes ✓")
        print("  → But might charge user twice ✗✗✗")
        print("  → Or lose payment record ✗✗✗")
        print("  → Unacceptable for financial data")
        print("")
        
        print("Result: CP choice correct for this feature")
        print("User can retry in a few seconds when partition heals")
        print("")

# Run scenarios
scenario = PartitionScenario()
scenario.scenario_driver_location()
scenario.scenario_payment()
```

### Key Takeaways for Software Engineers

1. **Not all features need the same consistency model**
   - Use CP for critical data (payments, bookings)
   - Use AP for non-critical data (locations, history)

2. **Think about partition scenarios**
   - What happens when network fails?
   - Which is worse: unavailability or stale data?

3. **User experience matters**
   - Payment failure with retry > incorrect charge
   - Stale location for 1 second > no drivers shown

4. **Use tunable consistency**
   - Same database, different consistency per operation
   - Profile read (AP) vs Profile write (CP)

5. **Design for failure**
   - Partitions will happen
   - Choose the right trade-off for each feature
   - Communicate clearly with users when unavailable

## Conclusion

The CAP Theorem is fundamental to understanding distributed systems:

1. **Network partitions will happen** - must be partition tolerant
2. **Choose between Consistency and Availability** during partitions
3. **No perfect solution** - only trade-offs
4. **Context matters** - choose based on requirements
5. **Tunable systems** offer flexibility

Understanding CAP helps make informed decisions about database selection and system design in distributed environments.
