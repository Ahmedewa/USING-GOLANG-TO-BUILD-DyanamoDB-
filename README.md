
NAME
AIMS AND GOALS 
DEVELOPMENTAL SETUP
TECH STACK
MAIN CODE
PROBLEMS/DISADVANTAGES
CONCLUSION

 
 

NAME: USING-GOLANG-TO-BUILD-DynamoDB

AIMS/GOALS:
This project implements a relational database-like application using MongoDB and Go. By leveraging MongoDB's collections, embedded documents, and manual joins, we
emulate relational database behavior while retaining NoSQL database qualities. This approach combines the scalability of NoSQL with ACID-compliant data structuring, 
suitable for handling massive datasets.

Key advantages:

A.   Flexible Document Model: Seamlessly handles varying data structures, ideal for dynamic data.
B.   Scalability: Efficiently manages large datasets and high traffic for large-scale applications.
C.   High Performance: Go's performance, combined with MongoDB's data retrieval, delivers a powerful application.
D.   Easy Data Handling: MongoDB's document model aligns with Go structs for straightforward data mapping.
E.   Rapid Development: Flexible schema allows quick adaptation to changing requirements.


TECH STACK:
Project Folder Structure

```plaintext
mongodb-relational-db/
│
├── main.go                       # Entry point
├── models/                       # Schemas/Models
│   ├── user.go                   # User model
│   └── order.go                  # Order model
├── controllers/                  # Business logic
│   ├── userController.go         # User-related logic
│   └── orderController.go        # Order-related logic
├── routes/                       # API routes
│   ├── userRoutes.go             # User routes
│   └── orderRoutes.go            # Order routes
├── database/                     # Database connection
│   └── connection.go             # MongoDB connection
├── docker-compose.yml            # Docker for MongoDB
├── go.mod                        # Go Module file
└── README.md
```

---

2. Building the App

2a. MongoDB Connection

`database/connection.go`
```go
package database

import (
	"context"
	"log"
	"time"

	"go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/options"
)

var Client  mongo.Client

func Connect() {
	client, err := mongo.NewClient(options.Client().ApplyURI("mongodb://localhost:27017"))
	if err != nil {
		log.Fatal(err)
	}

	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	err = client.Connect(ctx)
	if err != nil {
		log.Fatal(err)
	}

	// Ping the database to ensure the connection is established.
	err = client.Ping(ctx, nil)
	if err != nil {
		log.Fatal(err)
	}

	log.Println("Connected to MongoDB!")
	Client = client
}

func GetCollection(collectionName string) *mongo.Collection {
	return Client.Database("relational_db").Collection(collectionName)
}
```

---

2b. Models

    `models/user.go`
```go
package models

import "go.mongodb.org/mongo-driver/bson/primitive"

type User struct {
	ID        primitive.ObjectID `bson:"_id,omitempty"`
	FirstName string             `bson:"first_name"`
	LastName  string             `bson:"last_name"`
	Email     string             `bson:"email"`
}
```


`models/order.go`
```go
package models

import "go.mongodb.org/mongo-driver/bson/primitive"

type Order struct {
	ID       primitive.ObjectID `bson:"_id,omitempty"`
	UserID   primitive.ObjectID `bson:"user_id"`
	Item     string             `bson:"item"`
	Quantity int                `bson:"quantity"`
}
```

---

2c. Controllers

 `controllers/userController.go`
```go
package controllers

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"time"

	"mongodb-relational-db/database"
	"mongodb-relational-db/models"

	"go.mongodb.org/mongo-driver/bson"
	"go.mongodb.org/mongo-driver/bson/primitive"
)

var userCollection = database.GetCollection("users")

func CreateUser(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")

	var user models.User
	err := json.NewDecoder(r.Body).Decode(&user)
	if err != nil {
		http.Error(w, "Invalid input", http.StatusBadRequest)
		return
	}

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	result, err := userCollection.InsertOne(ctx, user)
	if err != nil {
		log.Fatal(err)
	}

	json.NewEncoder(w).Encode(result)
}

func GetUsers(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	cursor, err := userCollection.Find(ctx, bson.M{})
	if err != nil {
		log.Fatal(err)
	}
	defer cursor.Close(ctx)

	var users []models.User
	for cursor.Next(ctx) {
		var user models.User
		cursor.Decode(&user)
		users = append(users, user)
	}

	json.NewEncoder(w).Encode(users)
}
```

 `controllers/orderController.go`
```go
package controllers

import (
	"context"
	"encoding/json"
	"log"
	"net/http"
	"time"

	"mongodb-relational-db/database"
	"mongodb-relational-db/models"

	"go.mongodb.org/mongo-driver/bson"
	"go.mongodb.org/mongo-driver/bson/primitive"
)

var orderCollection = database.GetCollection("orders")

func CreateOrder(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")

	var order models.Order
	err := json.NewDecoder(r.Body).Decode(&order)
	if err != nil {
		http.Error(w, "Invalid input", http.StatusBadRequest)
		return
	}

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	result, err := orderCollection.InsertOne(ctx, order)
	if err != nil {
		log.Fatal(err)
	}

	json.NewEncoder(w).Encode(result)
}

func GetOrders(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	cursor, err := orderCollection.Find(ctx, bson.M{})
	if err != nil {
		log.Fatal(err)
	}
	defer cursor.Close(ctx)

	var orders []models.Order
	for cursor.Next(ctx) {
		var order models.Order
		cursor.Decode(&order)
		orders = append(orders, order)
	}

	json.NewEncoder(w).Encode(orders)
}

func GetOrdersByUserID(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")

	userID := r.URL.Query().Get("user_id")
	if userID == "" {
		http.Error(w, "user_id is required", http.StatusBadRequest)
		return
	}

	objID, _ := primitive.ObjectIDFromHex(userID)

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	cursor, err := orderCollection.Find(ctx, bson.M{"user_id": objID})
	if err != nil {
		log.Fatal(err)
	}
	defer cursor.Close(ctx)

	var orders []models.Order
	for cursor.Next(ctx) {
		var order models.Order
		cursor.Decode(&order)
		orders = append(orders, order)
	}

	json.NewEncoder(w).Encode(orders)
}
```

---

2d. Routes

 `routes/userRoutes.go`
```go
package routes

import (
	"mongodb-relational-db/controllers"
	"net/http"
)

func RegisterUserRoutes() {
	http.HandleFunc("/users", controllers.GetUsers)
	http.HandleFunc("/users/create", controllers.CreateUser)
}
```

 `routes/orderRoutes.go`
```go
package routes

import (
	"mongodb-relational-db/controllers"
	"net/http"
)

func RegisterOrderRoutes() {
	http.HandleFunc("/orders", controllers.GetOrders)
	http.HandleFunc("/orders/create", controllers.CreateOrder)
	http.HandleFunc("/orders/user", controllers.GetOrdersByUserID)
}
```

---

2e. Main Application

 `main.go`
```go
package main

import (
	"log"
	"net/http"

	"mongodb-relational-db/database"
	"mongodb-relational-db/routes"
)

func main() {
	database.Connect()

	routes.RegisterUserRoutes()
	routes.RegisterOrderRoutes()

	log.Println("Server is running on port 8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

---

2f. Docker Compose

 `docker-compose.yml`
```yaml
version: "3.8"
services:
  mongodb:
    image: mongo:5
    container_name: mongodb
    ports:
      - "27017:27017"
    volumes:
      - mongodb-data:/data/db

  app:
    build:
      context: .
    ports:
      - "8080:8080"
    depends_on:
      - mongodb

volumes:
  mongodb-data:
```

---

Tech Stack Diagram

```plaintext
+-------------------+       +----------------------------+
|   Frontend (React)| <---> | Backend API (Go + Gin)     |
+-------------------+       +----------------------------+
                                 |
                                 v
+-------------------+       +----------------------------+
|     MongoDB       | <---> | Monitoring (Prometheus, Grafana) |
+-------------------+       +----------------------------+
```

---

By following these steps, we can simulate a relational database-like application in MongoDB with Go
while maintaining scalability and  additional features or enhancements.



1. Modified Code for Relational Schema with Sort Key

To simulate a relational database in DynamoDB, we use composite primary keys 
(partition key + sort key) and Global Secondary Indexes (GSIs).

-Creating a Table with a Sort Key

We define a primary key with both a partition key (HASH key) and a sort key (RANGE key).

```go
package main

import (
	"fmt"
	"github.com/aws/aws-sdk-go/aws"
	"github.com/aws/aws-sdk-go/aws/session"
	"github.com/aws/aws-sdk-go/service/dynamodb"
)

func main() {
	// Start a session
	sess := session.Must(session.NewSessionWithOptions(session.Options{
		SharedConfigState: session.SharedConfigEnable,
	}))

	// Create DynamoDB service client
	svc := dynamodb.New(sess)

	// Define table schema
	input := &dynamodb.CreateTableInput{
		TableName: aws.String("RelationalTable"),
		KeySchema: []*dynamodb.KeySchemaElement{
			{
				AttributeName: aws.String("PrimaryKey"),
				KeyType:       aws.String("HASH"), // Partition Key
			},
			{
				AttributeName: aws.String("SortKey"),
				KeyType:       aws.String("RANGE"), // Sort Key
			},
		},
		AttributeDefinitions: []*dynamodb.AttributeDefinition{
			{
				AttributeName: aws.String("PrimaryKey"),
				AttributeType: aws.String("S"), // String
			},
			{
				AttributeName: aws.String("SortKey"),
				AttributeType: aws.String("S"), // String
			},
		},
		ProvisionedThroughput: &dynamodb.ProvisionedThroughput{
			ReadCapacityUnits:  aws.Int64(5),
			WriteCapacityUnits: aws.Int64(5),
		},
	}

	// Create the table
	_, err := svc.CreateTable(input)
	if err != nil {
		fmt.Println("Failed to create table:", err.Error())
		return
	}

	fmt.Println("Table created successfully!")
}
```

---
2. Inserting Data into the Table

To insert data with a composite primary key:

```go
func putItem(svc *dynamodb.DynamoDB, tableName, primaryKey, sortKey, data string) {
	input := &dynamodb.PutItemInput{
		TableName: aws.String(tableName),
		Item: map[string]*dynamodb.AttributeValue{
			"PrimaryKey": {
				S: aws.String(primaryKey),
			},
			"SortKey": {
				S: aws.String(sortKey),
			},
			"Data": {
				S: aws.String(data),
			},
		},
	}

	_, err := svc.PutItem(input)
	if err != nil {
		fmt.Println("Error inserting item:", err.Error())
		return
	}

	fmt.Println("Item inserted successfully!")
}
```

---

3. Querying Data by Primary Key

To retrieve data by the partition key and/or sort key:

```go
func queryData(svc *dynamodb.DynamoDB, tableName, primaryKey string) {
	input := &dynamodb.QueryInput{
		TableName: aws.String(tableName),
		KeyConditionExpression: aws.String("PrimaryKey = :pk"),
		ExpressionAttributeValues: map[string]*dynamodb.AttributeValue{
			":pk": {S: aws.String(primaryKey)},
		},
	}

	result, err := svc.Query(input)
	if err != nil {
		fmt.Println("Error querying data:", err.Error())
		return
	}

	fmt.Println("Query succeeded:")
	for _, item := range result.Items {
		fmt.Println(item)
	}
}
```

---

4. Global Secondary Index (GSI)

If you need to query non-primary key attributes, use GSIs.

Modify Table to Add a GSI

```go
input := &dynamodb.CreateTableInput{
	TableName: aws.String("RelationalTable"),
	KeySchema: []*dynamodb.KeySchemaElement{
		{
			AttributeName: aws.String("PrimaryKey"),
			KeyType:       aws.String("HASH"),
		},
		{
			AttributeName: aws.String("SortKey"),
			KeyType:       aws.String("RANGE"),
		},
	},
	AttributeDefinitions: []*dynamodb.AttributeDefinition{
		{
			AttributeName: aws.String("PrimaryKey"),
			AttributeType: aws.String("S"),
		},
		{
			AttributeName: aws.String("SortKey"),
			AttributeType: aws.String("S"),
		},
		{
			AttributeName: aws.String("SecondaryKey"),
			AttributeType: aws.String("S"),
		},
	},
	GlobalSecondaryIndexes: []*dynamodb.GlobalSecondaryIndex{
		{
			IndexName: aws.String("SecondaryIndex"),
			KeySchema: []*dynamodb.KeySchemaElement{
				{
					AttributeName: aws.String("SecondaryKey"),
					KeyType:       aws.String("HASH"),
				},
			},
			Projection: &dynamodb.Projection{
				ProjectionType: aws.String("ALL"),
			},
			ProvisionedThroughput: &dynamodb.ProvisionedThroughput{
				ReadCapacityUnits:  aws.Int64(5),
				WriteCapacityUnits: aws.Int64(5),
			},
		},
	},
	ProvisionedThroughput: &dynamodb.ProvisionedThroughput{
		ReadCapacityUnits:  aws.Int64(5),
		WriteCapacityUnits: aws.Int64(5),
	},
}
```

Query the GSI

```go
func queryGSI(svc *dynamodb.DynamoDB, tableName, indexName, secondaryKey string) {
	input := &dynamodb.QueryInput{
		TableName:              aws.String(tableName),
		IndexName:              aws.String(indexName),
		KeyConditionExpression: aws.String("SecondaryKey = :sk"),
		ExpressionAttributeValues: map[string]*dynamodb.AttributeValue{
			":sk": {S: aws.String(secondaryKey)},
		},
	}

	result, err := svc.Query(input)
	if err != nil {
		fmt.Println("Error querying GSI:", err.Error())
		return
	}

	fmt.Println("Query succeeded:")
	for _, item := range result.Items {
		fmt.Println(item)
	}
}
```

---

5. Conditional Writes for Concurrency

To handle concurrent writes, use the `ConditionExpression` parameter.

```go
func conditionalWrite(svc *dynamodb.DynamoDB, tableName, primaryKey, sortKey, value string) {
	input := &dynamodb.PutItemInput{
		TableName: aws.String(tableName),
		Item: map[string]*dynamodb.AttributeValue{
			"PrimaryKey": {
				S: aws.String(primaryKey),
			},
			"SortKey": {
				S: aws.String(sortKey),
			},
			"Value": {
				S: aws.String(value),
			},
		},
		ConditionExpression: aws.String("attribute_not_exists(Value)"),
	}

	_, err := svc.PutItem(input)
	if err != nil {
		fmt.Println("Error with conditional write:", err.Error())
		return
	}

	fmt.Println("Conditional write succeeded!")
}
```

---

6. Full CRUD Application Example

Here’s a full example of a CRUD application using Go and DynamoDB:

```go
package main

import (
	"fmt"
	"github.com/aws/aws-sdk-go/aws"
	"github.com/aws/aws-sdk-go/aws/session"
	"github.com/aws/aws-sdk-go/service/dynamodb"
)

func main() {
	sess := session.Must(session.NewSessionWithOptions(session.Options{
		SharedConfigState: session.SharedConfigEnable,
	}))

	svc := dynamodb.New(sess)
	tableName := "RelationalTable"

	// Create an item
	putItem(svc, tableName, "User#1", "Profile#1", "John Doe")

	// Query an item
	queryData(svc, tableName, "User#1")

	// Update an item
	conditionalWrite(svc, tableName, "User#1", "Profile#1", "John Updated")

	// Query GSI
	queryGSI(svc, tableName, "SecondaryIndex", "SomeSecondaryKey")
}

func putItem(svc *dynamodb.DynamoDB, tableName, primaryKey, sortKey, data string) {
	// Insert item code as shown above
}

func queryData(svc *dynamodb.DynamoDB, tableName, primaryKey string) {
	// Query item code as shown above
}

func queryGSI(svc *dynamodb.DynamoDB, tableName, indexName, secondaryKey string) {
	// Query GSI code as shown above
}

func conditionalWrite(svc *dynamodb.DynamoDB, tableName, primaryKey, sortKey, value string) {
	// Conditional write code as shown above
}
```

---

Next Steps
- Implement pagination for large datasets using `LastEvaluatedKey`.
- Use AWS IAM roles to secure DynamoDB access.
- Deploy the Go app using Docker or AWS Lambda.


Developmental environment for the project along with a single tech stack diagram.
A complete setup for the backend, frontend, database, monitoring tools, and deployment.

---

1. DEVELOPMENTAL SETUP

1a. Prerequisites
Before starting, install the following tools:
1. **Go** (Backend): Install Go from [golang.org](https://golang.org/).
   
Prerequisites
1. Go: Install from [golang.org](https://golang.org/).
2. MongoDB: Install MongoDB from [mongodb.com](https://www.mongodb.com/try/download/community).
3. MongoDB Driver for Go: Install the official MongoDB Go Driver:
   ```bash
   go get go.mongodb.org/mongo-driver/mongo
   ```
3. Node.js (Frontend): Install from [nodejs.org](https://nodejs.org/).
4. AWS CLI: Install from [aws.amazon.com/cli](https://aws.amazon.com/cli/).
5. Docker**: Install from [docker.com](https://www.docker.com/).
6. Postman**: For API testing, download from [postman.com](https://www.postman.com/).
7. Grafana & Prometheus: Download Grafana and Prometheus for monitoring.
8. Git: Install Git for version control.

   ```

1b. Project Folder Structure
Here’s the folder structure for the entire project:

```
vulnerability-assessment-app/
│
├── backend/                    # Backend service (Go)
│   ├── main.go
│   ├── go.mod
│   └── Dockerfile
│
├── frontend/                   # Frontend service (React.js)
│   ├── src/
│   ├── public/
│   ├── package.json
│   └── Dockerfile
│
├── database/                   # DynamoDB-related setup
│   ├── setup-dynamodb.go
│
├── monitoring/                 # Monitoring setup (Prometheus + Grafana)
│   ├── prometheus.yml
│   ├── grafana.ini
│
├── logging/                    # Centralized logging setup (ELK Stack)
│   ├── logstash.conf
│
├── ci-cd/                      # GitHub Actions CI/CD pipeline
│   └── ci-cd.yml
│
├── docker-compose.yml          # Docker Compose setup for the full stack
└── README.md
```

---

1c. Backend Setup

1. Initialize a Go Project
   ```bash
   mkdir backend && cd backend
   go mod init backend
   ```

2. Install Dependencies
   ```bash
   go get github.com/gin-gonic/gin
   go get github.com/aws/aws-sdk-go
   go get github.com/prometheus/client_golang/prometheus
   ```

3. Backend Code Example (`main.go`)
   ```go
   package main

   import (
       "github.com/gin-gonic/gin"
       "log"
       "net/http"
   )

   func main() {
       router := gin.Default()

       router.GET("/health", func(c *gin.Context) {
           c.JSON(http.StatusOK, gin.H{"status": "OK"})
       })

       log.Println("Backend service is running on port 8080")
       router.Run(":8080")
   }
   ```

4. Dockerfile for Backend
   ```dockerfile
   FROM golang:1.20

   WORKDIR /app
   COPY . .
   RUN go build -o main .

   EXPOSE 8080
   CMD ["./main"]
   ```

---
1d. Frontend Setup

1. Create a React App
   ```bash
   npx create-react-app frontend
   cd frontend
   npm install
   ```

2. Frontend Code Example (`App.js`)
   ```jsx
   import React from "react";

   function App() {
       return (
           <div>
               <h1>Vulnerability Assessment App</h1>
               <p>The app is running successfully!</p>
           </div>
       );
   }

   export default App;
   ```

3. Dockerfile for Frontend
   ```dockerfile
   FROM node:16

   WORKDIR /app
   COPY . .
   RUN npm install
   RUN npm run build

   EXPOSE 3000
   CMD ["npm", "start"]
   ```

---

1e. DynamoDB Setup

1. Install AWS CLI
   ```bash
   aws configure
   ```

2. Create a DynamoDB Table (`setup-dynamodb.go`)
   ```go
   package main

   import (
       "fmt"
       "github.com/aws/aws-sdk-go/aws"
       "github.com/aws/aws-sdk-go/aws/session"
       "github.com/aws/aws-sdk-go/service/dynamodb"
   )

   func main() {
       sess := session.Must(session.NewSession())
       svc := dynamodb.New(sess)

       input := &dynamodb.CreateTableInput{
           TableName: aws.String("VulnerabilityTable"),
           KeySchema: []*dynamodb.KeySchemaElement{
               {
                   AttributeName: aws.String("PrimaryKey"),
                   KeyType:       aws.String("HASH"),
               },
           },
           AttributeDefinitions: []*dynamodb.AttributeDefinition{
               {
                   AttributeName: aws.String("PrimaryKey"),
                   AttributeType: aws.String("S"),
               },
           },
           ProvisionedThroughput: &dynamodb.ProvisionedThroughput{
               ReadCapacityUnits:  aws.Int64(5),
               WriteCapacityUnits: aws.Int64(5),
           },
       }

       _, err := svc.CreateTable(input)
       if err != nil {
           fmt.Println("Error creating table:", err)
           return
       }

       fmt.Println("Table created successfully")
   }
   ```

---

   1f. Monitoring Setup (Prometheus + Grafana)

1. Prometheus Configuration (`prometheus.yml`)
   ```yaml
   global:
     scrape_interval: 15s

   scrape_configs:
     - job_name: "backend"
       static_configs:
         - targets: ["backend:8080"]
   ```

2. Grafana Configuration (`grafana.ini`)
   ```ini
   [server]
   http_port = 3000

   [auth.anonymous]
   enabled = true
   ```

---

1g. Docker Compose
Combine all services in a `docker-compose.yml` file.

```yaml
version: "3.8"
services:
  backend:
    build: ./backend
    ports:
      - "8080:8080"

  frontend:
    build: ./frontend
    ports:
      - "3000:3000"

  prometheus:
    image: prom/prometheus
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana
    volumes:
      - ./monitoring/grafana.ini:/etc/grafana/grafana.ini
    ports:
      - "3001:3000"
```

---

1h. CI/CD Pipeline
Add a GitHub Actions Workflow file for CI/CD.

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Go
        uses: actions/setup-go@v4
        with:
          go-version: 1.20

      - name: Install dependencies
        run: go mod tidy

      - name: Run tests
        run: go test ./...

      - name: Build Docker images
        run: docker-compose build
```

---

2. Tech Stack Diagram

Here’s a simple diagram representing the tech stack for the project:

```plaintext
+-------------------+         +-----------------------+
|   Frontend (UI)   | <-----> | Backend (Go + Gin)    |
| React + Bootstrap |         | REST API             |
+-------------------+         +-----------------------+
          |
          v
+-------------------+         +-----------------------+
|     Database      | <-----> | Monitoring & Logging  |
| DynamoDB          |         | Prometheus + Grafana  |
+-------------------+         +-----------------------+
          |
          v
+-------------------+
| Deployment Tools  |
| Docker + CI/CD    |
+-------------------+
```

---

Next Steps
- Run `docker-compose up` to start all services.
- Test the app in the browser (`http://localhost:3000` for the frontend, `http://localhost:8080/health` for the backend).
- Monitor services via Prometheus (`http://localhost:9090`) and Grafana (`http://localhost:3001`).

..AWS DynamoDB Transactions, Logstash configuration, Idempotence, Feedback handling, and Security implementation for our Go app.

---

1. `Go` Example Using the `AWS SDK` for DynamoDB Transactions**

DynamoDB supports transactions for multiple operations that need to be executed atomically, such as `TransactWriteItems`
for writing multiple items and `TransactGetItems` for reading multiple items.

-  Writing a Transaction
```go
package main

import (
	"fmt"
	"log"

	"github.com/aws/aws-sdk-go/aws"
	"github.com/aws/aws-sdk-go/aws/session"
	"github.com/aws/aws-sdk-go/service/dynamodb"
)

func main() {
	// Initialize AWS Session
	sess := session.Must(session.NewSession(&aws.Config{
		Region: aws.String("us-east-1"),
	}))

	svc := dynamodb.New(sess)

	// Define transaction items
	input := &dynamodb.TransactWriteItemsInput{
		TransactItems: []*dynamodb.TransactWriteItem{
			{
				Put: &dynamodb.Put{
					TableName: aws.String("UsersTable"),
					Item: map[string]*dynamodb.AttributeValue{
						"UserId":    {S: aws.String("User123")},
						"FirstName": {S: aws.String("John")},
						"LastName":  {S: aws.String("Doe")},
					},
				},
			},
			{
				Update: &dynamodb.Update{
					TableName: aws.String("OrdersTable"),
					Key: map[string]*dynamodb.AttributeValue{
						"OrderId": {S: aws.String("Order456")},
					},
					UpdateExpression: aws.String("SET OrderStatus = :status"),
					ExpressionAttributeValues: map[string]*dynamodb.AttributeValue{
						":status": {S: aws.String("Processed")},
					},
				},
			},
		},
	}

	// Execute transaction
	_, err := svc.TransactWriteItems(input)
	if err != nil {
		log.Fatalf("Failed to execute transaction: %v", err)
	}

	fmt.Println("Transaction executed successfully")
}
```

- Reading in a Transaction
```go
input := &dynamodb.TransactGetItemsInput{
    TransactItems: []*dynamodb.TransactGetItem{
        {
            Get: &dynamodb.Get{
                TableName: aws.String("UsersTable"),
                Key: map[string]*dynamodb.AttributeValue{
                    "UserId": {S: aws.String("User123")},
                },
            },
        },
        {
            Get: &dynamodb.Get{
                TableName: aws.String("OrdersTable"),
                Key: map[string]*dynamodb.AttributeValue{
                    "OrderId": {S: aws.String("Order456")},
                },
            },
        },
    },
}

result, err := svc.TransactGetItems(input)
if err != nil {
    log.Fatalf("Failed to execute transaction: %v", err)
}
fmt.Println("Transaction result:", result)
```

---

2. Configure `Logstash` for Multiple Entries

Logstash can be configured to handle multiple input sources and send logs to multiple outputs.

- Logstash Configuration
Create a `logstash.conf` file:

```plaintext
input {
  file {
    path => "/var/log/app1.log"
    start_position => "beginning"
    type => "app1"
  }

  file {
    path => "/var/log/app2.log"
    start_position => "beginning"
    type => "app2"
  }
}

filter {
  if [type] == "app1" {
    grok {
      match => { "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:loglevel} %{GREEDYDATA:message}" }
    }
  }

  if [type] == "app2" {
    grok {
      match => { "message" => "%{TIMESTAMP_ISO8601:timestamp} %{GREEDYDATA:message}" }
    }
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "%{type}-logs"
  }

  stdout { codec => rubydebug }
}
```

- Explanation:
- Inputs: Logs from two files (`app1.log` and `app2.log`).
- Filters: Parses logs differently based on their type.
- Outputs: Sends logs to Elasticsearch and the console.

---

3. Implementing Idempotence in the Design Concept

Idempotence ensures that repeated operations produce the same result, preventing duplicate writes or updates in DynamoDB.

- Idempotent Writes
Use a unique identifier (e.g., `RequestId`) to ensure an operation is not processed twice.

```go
func putItem(svc *dynamodb.DynamoDB, tableName, requestId string) error {
	input := &dynamodb.PutItemInput{
		TableName: aws.String(tableName),
		Item: map[string]*dynamodb.AttributeValue{
			"RequestId": {S: aws.String(requestId)},
			"Data":      {S: aws.String("Some data")},
		},
		ConditionExpression: aws.String("attribute_not_exists(RequestId)"),
	}

	_, err := svc.PutItem(input)
	if err != nil {
		return fmt.Errorf("failed to write item: %w", err)
	}

	return nil
}
```

- `ConditionExpression`: Ensures the `RequestId` is unique.

---

4. Receiving Feedback

-REST API Endpoint
```go
r.POST("/feedback", func(c *gin.Context) {
	var feedback struct {
		UserId  string `json:"userId"`
		Message string `json:"message"`
	}

	if err := c.ShouldBindJSON(&feedback); err != nil {
		c.JSON(400, gin.H{"error": "Invalid input"})
		return
	}

	// Save feedback to DynamoDB
	input := &dynamodb.PutItemInput{
		TableName: aws.String("FeedbackTable"),
		Item: map[string]*dynamodb.AttributeValue{
			"UserId":  {S: aws.String(feedback.UserId)},
			"Message": {S: aws.String(feedback.Message)},
		},
	}

	_, err := svc.PutItem(input)
	if err != nil {
		c.JSON(500, gin.H{"error": err.Error()})
		return
	}

	c.JSON(200, gin.H{"message": "Feedback received"})
})
```

---

5. Implement Security for DynamoDB Database

Best Practices for DynamoDB Security
1. Use AWS IAM Roles: Attach least-privileged policies.
2. Enable Encryption: Use DynamoDB encryption at rest.
3. Audit Access: Enable AWS CloudTrail for audit logs.
4. Restrict IPs: Use VPC endpoints for private access.

-IAM Policy Example
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:Query",
        "dynamodb:GetItem",
        "dynamodb:PutItem"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/MyTable"
    }
  ]
}
```

---

6. Implement Advanced Security for APIs

1. Rate Limiting
Use Gin Middleware for rate limiting:
```bash
go get github.com/gin-contrib/limiter
```

```go
package main

import (
	"github.com/gin-contrib/limiter"
	"github.com/gin-gonic/gin"
	"github.com/ulule/limiter/v3"
	"github.com/ulule/limiter/v3/drivers/store/memory"
)

func main() {
	store := memory.NewStore()
	rate, _ := limiter.NewRateFromFormatted("10-S") // 10 requests per second
	middleware := limiter.NewMiddleware(limiter.New(store, rate))

	r := gin.New()
	r.Use(middleware)

	r.GET("/", func(c *gin.Context) {
		c.String(200, "Welcome!")
	})

	r.Run(":8080")
}
```

---

2. Input Validation
Use `validator` for payload validation:
```bash
go get github.com/go-playground/validator/v10
```

```go
type Feedback struct {
	UserId  string `json:"userId" validate:"required,uuid"`
	Message string `json:"message" validate:"required,min=10"`
}

func main() {
	validate := validator.New()

	r.POST("/feedback", func(c *gin.Context) {
		var feedback Feedback
		if err := c.ShouldBindJSON(&feedback); err != nil {
			c.JSON(400, gin.H{"error": "Invalid input"})
			return
		}

		if err := validate.Struct(feedback); err != nil {
			c.JSON(400, gin.H{"error": err.Error()})
			return
		}

		c.JSON(200, gin.H{"message": "Feedback received"})
	})
}
```

---

3. JWT Authentication
```bash
go get github.com/dgrijalva/jwt-go
```

```go
import (
	"time"
	"github.com/dgrijalva/jwt-go"
)

var jwtSecret = []byte("your-secret-key")

func generateToken(userId string) (string, error) {
	claims := jwt.MapClaims{
		"userId": userId,
		"exp":    time.Now().Add(time.Hour * 1).Unix(),
	}

	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	return token.SignedString(jwtSecret)
}
```

---

Using these implementations, you can achieve robust functionality, security, and
scalability for our DynamoDB-powered Go application. 









IDEMPOTENCE/ FEED-BACK/SECURITY FOR THE APPLICATION

-AWS DynamoDB Transactions, Logstash configuration, Idempotence, Feedback handling, and Security implementation for our Go app.

---
1. `Go`  Using the `AWS SDK` for DynamoDB Transactions**

DynamoDB supports ,transaction for multiple operations that need to be executed atomically, such as `TransactWriteItems` for writing multiple items, and
`TransactGetItems` for reading multiple items.

- Writing a Transaction
```go
package main

import (
	"fmt"
	"log"

	"github.com/aws/aws-sdk-go/aws"
	"github.com/aws/aws-sdk-go/aws/session"
	"github.com/aws/aws-sdk-go/service/dynamodb"
)

func main() {
	// Initialize AWS Session
	sess := session.Must(session.NewSession(&aws.Config{
		Region: aws.String("us-east-1"),
	}))

	svc := dynamodb.New(sess)

	// Define transaction items
	input := &dynamodb.TransactWriteItemsInput{
		TransactItems: []*dynamodb.TransactWriteItem{
			{
				Put: &dynamodb.Put{
					TableName: aws.String("UsersTable"),
					Item: map[string]*dynamodb.AttributeValue{
						"UserId":    {S: aws.String("User123")},
						"FirstName": {S: aws.String("John")},
						"LastName":  {S: aws.String("Doe")},
					},
				},
			},
			{
				Update: &dynamodb.Update{
					TableName: aws.String("OrdersTable"),
					Key: map[string]*dynamodb.AttributeValue{
						"OrderId": {S: aws.String("Order456")},
					},
					UpdateExpression: aws.String("SET OrderStatus = :status"),
					ExpressionAttributeValues: map[string]*dynamodb.AttributeValue{
						":status": {S: aws.String("Processed")},
					},
				},
			},
		},
	}

	// Execute transaction
	_, err := svc.TransactWriteItems(input)
	if err != nil {
		log.Fatalf("Failed to execute transaction: %v", err)
	}

	fmt.Println("Transaction executed successfully")
}
```

- Reading in a Transaction
```go
input := &dynamodb.TransactGetItemsInput{
    TransactItems: []*dynamodb.TransactGetItem{
        {
            Get: &dynamodb.Get{
                TableName: aws.String("UsersTable"),
                Key: map[string]*dynamodb.AttributeValue{
                    "UserId": {S: aws.String("User123")},
                },
            },
        },
        {
            Get: &dynamodb.Get{
                TableName: aws.String("OrdersTable"),
                Key: map[string]*dynamodb.AttributeValue{
                    "OrderId": {S: aws.String("Order456")},
                },
            },
        },
    },
}

result, err := svc.TransactGetItems(input)
if err != nil {
    log.Fatalf("Failed to execute transaction: %v", err)
}
fmt.Println("Transaction result:", result)
```

---

 Configuring `Logstash` for Multiple Entries:

Logstash can be configured to handle multiple input sources and send logs to multiple outputs.

- Logstash Configuration
Create a `logstash.conf` file:

```plaintext
input {
  file {
    path => "/var/log/app1.log"
    start_position => "beginning"
    type => "app1"
  }

  file {
    path => "/var/log/app2.log"
    start_position => "beginning"
    type => "app2"
  }
}

filter {
  if [type] == "app1" {
    grok {
      match => { "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:loglevel} %{GREEDYDATA:message}" }
    }
  }

  if [type] == "app2" {
    grok {
      match => { "message" => "%{TIMESTAMP_ISO8601:timestamp} %{GREEDYDATA:message}" }
    }
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "%{type}-logs"
  }

  stdout { codec => rubydebug }
}
```

 Explanation:
- Inputs: Logs from two files (`app1.log` and `app2.log`).
- Filters: Parses logs differently based on their type.
- Outputs: Sends logs to Elasticsearch and the console.

---

3. Implementing Idempotence in the Design Concept:

Idempotence ensures that repeated operations produce the same result, preventing duplicate writes or updates in DynamoDB.

- Idempotent Writes
Use a unique identifier (e.g., `RequestId`) to ensure an operation is not processed twice.

```go
func putItem(svc *dynamodb.DynamoDB, tableName, requestId string) error {
	input := &dynamodb.PutItemInput{
		TableName: aws.String(tableName),
		Item: map[string]*dynamodb.AttributeValue{
			"RequestId": {S: aws.String(requestId)},
			"Data":      {S: aws.String("Some data")},
		},
		ConditionExpression: aws.String("attribute_not_exists(RequestId)"),
	}

	_, err := svc.PutItem(input)
	if err != nil {
		return fmt.Errorf("failed to write item: %w", err)
	}

	return nil
}
```

-   `ConditionExpression`: Ensures the `RequestId` is unique.

---

  4. Receiving Feedback

  REST API Endpoint
```go
r.POST("/feedback", func(c *gin.Context) {
	var feedback struct {
		UserId  string `json:"userId"`
		Message string `json:"message"`
	}

	if err := c.ShouldBindJSON(&feedback); err != nil {
		c.JSON(400, gin.H{"error": "Invalid input"})
		return
	}

	// Save feedback to DynamoDB
	input := &dynamodb.PutItemInput{
		TableName: aws.String("FeedbackTable"),
		Item: map[string]*dynamodb.AttributeValue{
			"UserId":  {S: aws.String(feedback.UserId)},
			"Message": {S: aws.String(feedback.Message)},
		},
	}

	_, err := svc.PutItem(input)
	if err != nil {
		c.JSON(500, gin.H{"error": err.Error()})
		return
	}

	c.JSON(200, gin.H{"message": "Feedback received"})
})
```

---

5. Implement Security for DynamoDB Database

Best Practices for DynamoDB Security
2. Enable Encryption: Use DynamoDB encryption at rest.
3. Audit Access: Enable AWS CloudTrail for audit logs.
4. Restrict IPs: Use VPC endpoints for private access.

  IAM Policy Example
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:Query",
        "dynamodb:GetItem",
        "dynamodb:PutItem"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/MyTable"
    }
  ]
}
```

---

  6. Implement Advanced Security for APIs

  1. Rate Limiting
 We use Gin Middleware for rate limiting:
```bash
go get github.com/gin-contrib/limiter
```

```go
package main

import (
	"github.com/gin-contrib/limiter"
	"github.com/gin-gonic/gin"
	"github.com/ulule/limiter/v3"
	"github.com/ulule/limiter/v3/drivers/store/memory"
)

func main() {
	store := memory.NewStore()
	rate, _ := limiter.NewRateFromFormatted("10-S") // 10 requests per second
	middleware := limiter.NewMiddleware(limiter.New(store, rate))

	r := gin.New()
	r.Use(middleware)

	r.GET("/", func(c *gin.Context) {
		c.String(200, "Welcome!")
	})

	r.Run(":8080")
}
```

---

  2. Input Validation
Use `validator` for payload validation:
```bash
go get github.com/go-playground/validator/v10
```

```go
type Feedback struct {
	UserId  string `json:"userId" validate:"required,uuid"`
	Message string `json:"message" validate:"required,min=10"`
}

func main() {
	validate := validator.New()

	r.POST("/feedback", func(c *gin.Context) {
		var feedback Feedback
		if err := c.ShouldBindJSON(&feedback); err != nil {
			c.JSON(400, gin.H{"error": "Invalid input"})
			return
		}

		if err := validate.Struct(feedback); err != nil {
			c.JSON(400, gin.H{"error": err.Error()})
			return
		}

		c.JSON(200, gin.H{"message": "Feedback received"})
	})
}
```

---

  3. JWT Authentication
```bash
go get github.com/dgrijalva/jwt-go
```

```go
import (
	"time"
	"github.com/dgrijalva/jwt-go"
)

var jwtSecret = []byte("your-secret-key")

func generateToken(userId string) (string, error) {
	claims := jwt.MapClaims{
		"userId": userId,
		"exp":    time.Now().Add(time.Hour * 1).Unix(),
	}

	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	return token.SignedString(jwtSecret)
}
```

---

By using these implementations, we can achieve robust functionality, security, and scalability for your
DynamoDB-powered Go application!










How to implement : debugging, error handling, testing, APIs, message queues, parallel processing, failure frameworks, monitoring, and
central logging, for our DynamoDB application. 

---

1. Debugging/Error Handling/Testing

-Error Handling in Go
Use `defer`, `recover`, and proper error handling for DynamoDB operations.

```go
package main

import (
	"fmt"
	"log"
	"runtime/debug"

	"github.com/aws/aws-sdk-go/aws"
	"github.com/aws/aws-sdk-go/aws/session"
	"github.com/aws/aws-sdk-go/service/dynamodb"
)

func main() {
	defer handlePanic() // Recover from panics

	sess, err := session.NewSession()
	if err != nil {
		log.Fatalf("Failed to create session: %v", err)
	}

	svc := dynamodb.New(sess)

	// Example: Querying DynamoDB
	_, err = svc.GetItem(&dynamodb.GetItemInput{
		TableName: aws.String("NonExistentTable"),
		Key: map[string]*dynamodb.AttributeValue{
			"PrimaryKey": {S: aws.String("Key1")},
		},
	})
	if err != nil {
		log.Printf("Error occurred: %v", err)
	}
}

func handlePanic() {
	if r := recover(); r != nil {
		fmt.Println("Recovered from panic:", r)
		fmt.Println("Stack trace:", string(debug.Stack()))
	}
}
```

---

-Testin

-Unit Testing in Go
```bash
go test ./... -v
```

Example of a unit test for a DynamoDB function:
```go
package main

import (
	"testing"

	"github.com/aws/aws-sdk-go/service/dynamodb"
)

func TestPutItem(t *testing.T) {
	mockSvc := &dynamodb.DynamoDB{}
	err := putItem(mockSvc, "TestTable", "Key1", "Value1")
	if err != nil {
		t.Errorf("Expected no error, got %v", err)
	}
}
```

End-to-End Testing with Postman
1. Export your API as a Postman collection.
2. Use Postman to test all endpoints (e.g., `GET`, `POST`).
3. Automate tests with Postman’s   Collection Runner.

---

2. API Example

Create a REST API to interact with DynamoDB.

-Go REST API with Gin
```bash
go get -u github.com/gin-gonic/gin
```

```go
package main

import (
	"github.com/gin-gonic/gin"
	"github.com/aws/aws-sdk-go/aws"
	"github.com/aws/aws-sdk-go/aws/session"
	"github.com/aws/aws-sdk-go/service/dynamodb"
)

func main() {
	r := gin.Default()
	sess := session.Must(session.NewSession())
	svc := dynamodb.New(sess)

	r.GET("/item/:id", func(c *gin.Context) {
		id := c.Param("id")
		input := &dynamodb.GetItemInput{
			TableName: aws.String("MyTable"),
			Key: map[string]*dynamodb.AttributeValue{
				"PrimaryKey": {S: aws.String(id)},
			},
		}

		result, err := svc.GetItem(input)
		if err != nil {
			c.JSON(500, gin.H{"error": err.Error()})
			return
		}
		c.JSON(200, result)
	})

	r.Run(":8080")
}
```

---

  3. Message Queue (RabbitMQ)

Integrate RabbitMQ for asynchronous tasks like processing DynamoDB updates.


-Producer
```go
package main

import (
	"log"

	"github.com/streadway/amqp"
)

func main() {
	conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
	if err != nil {
		log.Fatal(err)
	}
	defer conn.Close()

	ch, err := conn.Channel()
	if err != nil {
		log.Fatal(err)
	}
	defer ch.Close()

	q, err := ch.QueueDeclare("task_queue", true, false, false, false, nil)
	if err != nil {
		log.Fatal(err)
	}

	body := "Hello RabbitMQ!"
	err = ch.Publish("", q.Name, false, false, amqp.Publishing{
		ContentType: "text/plain",
		Body:        []byte(body),
	})
	if err != nil {
		log.Fatal(err)
	}

	log.Printf(" [x] Sent %s", body)
}
```

  -Consumer
```go
func main() {
	conn, _ := amqp.Dial("amqp://guest:guest@localhost:5672/")
	ch, _ := conn.Channel()
	q, _ := ch.QueueDeclare("task_queue", true, false, false, false, nil)

	msgs, _ := ch.Consume(q.Name, "", true, false, false, false, nil)

	for msg := range msgs {
		log.Printf("Received: %s", msg.Body)
	}
}
```

---

  4. Apache Spark

Use Spark for data lake or parallel processing of DynamoDB exports.

-Apache Spark  in Python
```python
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("DynamoDBProcessing").getOrCreate()

. Load DynamoDB export data (e.g., from S3)
data = spark.read.json("s3://my-bucket/dynamodb-export.json")

. Perform transformations
filtered_data = data.filter(data["status"] == "active")

. Write results back to S3
filtered_data.write.json("s3://my-bucket/processed-data")
```

---

## **5. Failure/Retry Framework

  -Circuit Breaker with `opossum` in Node.js
```bash
npm install opossum
```

```javascript
const CircuitBreaker = require('opossum');

function riskyOperation() {
  return new Promise((resolve, reject) => {
    // Simulate failure
    Math.random() > 0.5 ? resolve("Success") : reject("Error");
  });
}

const breaker = new CircuitBreaker(riskyOperation, { timeout: 5000 });

breaker.fallback(() => "Service unavailable. Please try again later.");
breaker.on("open", () => console.log("Circuit breaker opened!"));

breaker.fire().then(console.log).catch(console.error);
```

---

6. Monitoring & Logging

-Prometheus Metrics in Go
```go
package main

import (
	"net/http"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

var requestCount = prometheus.NewCounterVec(
	prometheus.CounterOpts{
		Name: "http_requests_total",
		Help: "Total number of HTTP requests",
	},
	[]string{"method"},
)

func init() {
	prometheus.MustRegister(requestCount)
}

func main() {
	http.Handle("/metrics", promhttp.Handler())
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		requestCount.WithLabelValues(r.Method).Inc()
		w.Write([]byte("Hello, Prometheus!"))
	})
	http.ListenAndServe(":8080", nil)
}
```

  -Grafana Dashboard
1. Connect Grafana to Prometheus.
2. Create a dashboard with panels for metrics like `http_requests_total`.

---

7. Central Logging with ELK Stack

ELK (Elasticsearch, Logstash, Kibana) enables centralized logging.

  - Logstash Configuration
Create `logstash.conf`:
```plaintext
input {
  file {
    path => "/var/log/myapp.log"
    start_position => "beginning"
  }
}

filter {
  grok {
    match => { "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:loglevel} %{GREEDYDATA:message}" }
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "myapp-logs"
  }
}
```

  Go Logging
```go
package main

import (
	"log"
	"os"
)

func main() {
	file, err := os.OpenFile("myapp.log", os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0666)
	if err != nil {
		log.Fatal(err)
	}
	defer file.Close()

	logger := log.New(file, "INFO: ", log.Ldate|log.Ltime|log.Lshortfile)
	logger.Println("Application started")
}
```

### **View Logs in Kibana**
1. Launch Kibana from the ELK stack.
2. Create an index pattern (`myapp-logs*`).
3. View logs in the **Discover** tab.

---

By implementing these components, we can achieve a robust, scalable, and fully monitored DynamoDB application with modern best practices. 







Implementing :pagination using `LastEvaluatedKey`, securing DynamoDB access with AWS IAM roles,
and deploying the Go application using :Docker, AWS Lambda, and GitHub Actions workflows pipeline for CI/CD.

---

1. Pagination for Large Data Sheets Using `LastEvaluatedKey`

When working with large datasets in DynamoDB, we can paginate query results using the `LastEvaluatedKey`.

  Paginated Query

```go
package main

import (
	"fmt"
	"log"

	"github.com/aws/aws-sdk-go/aws"
	"github.com/aws/aws-sdk-go/aws/session"
	"github.com/aws/aws-sdk-go/service/dynamodb"
)

func main() {
	// Initialize DynamoDB session
	sess := session.Must(session.NewSessionWithOptions(session.Options{
		SharedConfigState: session.SharedConfigEnable,
	}))

	svc := dynamodb.New(sess)
	tableName := "RelationalTable"
	partitionKey := "User#1"

	// Fetch paginated results
	paginatedQuery(svc, tableName, partitionKey)
}

func paginatedQuery(svc *dynamodb.DynamoDB, tableName, partitionKey string) {
	var lastEvaluatedKey map[string]*dynamodb.AttributeValue
	for {
		input := &dynamodb.QueryInput{
			TableName:              aws.String(tableName),
			KeyConditionExpression: aws.String("PrimaryKey = :pk"),
			ExpressionAttributeValues: map[string]*dynamodb.AttributeValue{
				":pk": {S: aws.String(partitionKey)},
			},
			Limit: aws.Int64(5), // Fetch 5 items per request
			ExclusiveStartKey: lastEvaluatedKey, // Use the last evaluated key to fetch the next set of data
		}

		result, err := svc.Query(input)
		if err != nil {
			log.Fatalf("Failed to query data: %v", err)
		}

		for _, item := range result.Items {
			fmt.Println(item)
		}

		// Check if there are more items to fetch
		if result.LastEvaluatedKey == nil {
			break
		}

		lastEvaluatedKey = result.LastEvaluatedKey
	}
}
```

   - Explanation:
- `Limit`: Specifies the maximum number of items to fetch in a single query.
- `ExclusiveStartKey`: Used to fetch the next set of results starting after the last evaluated key.
- `LastEvaluatedKey`: Returned by DynamoDB to indicate there are more results to fetch.

---

2. AWS IAM Roles to Secure DynamoDB Access:

IAM roles and policies ensure secure access to DynamoDB. We  can create a role with restricted permissions for our application.

   -  IAM Policy for Restricted DynamoDB Access

We create a policy that allows only specific actions (e.g., `Query`, `GetItem`, `PutItem`) on specific tables.

  - Policy JSON:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:Query",
        "dynamodb:GetItem",
        "dynamodb:PutItem"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/RelationalTable"
    }
  ]
}
```

    Steps to Create the Role:
1. Go to the  AWS Management Console   →   IAM →   Roles.
2. Create a role with the above policy attached.
3. Assign the role to the EC2 instance, Lambda function, or ECS task running our application.

---

3. DEPLOYMENT:

3a. Deploy Using 'DOCKER':

-  Dockerfile
```dockerfile
. Start from the official Golang image
FROM golang:1.20

.  Set the working directory
WORKDIR /app

.  Copy the Go app and dependencies
COPY go.mod go.sum ./
RUN go mod download

COPY . .

.  Build the Go application
RUN go build -o main .

.  Expose the port the app runs on
EXPOSE 8080

.  Run the app
CMD ["./main"]
```


-  Build and Run the Docker Image
```bash
. Build Docker Image
docker build -t my-dynamodb-app .

. Run Docker Container
docker run -p 8080:8080 my-dynamodb-app
```

---

3b. Deploy Using AWS Lambda

You can deploy the Go application as a Lambda function.

Steps:
1. Ensure our Go app is Lambda-compatible by using the AWS Lambda Go SDK:
   ```bash
   go get github.com/aws/aws-lambda-go/lambda
   ```

.  Example Lambda Handler:
```go
package main

import (
	"context"
	"fmt"

	"github.com/aws/aws-lambda-go/lambda"
)

type MyEvent struct {
	PrimaryKey string `json:"primaryKey"`
}

func handleRequest(ctx context.Context, event MyEvent) (string, error) {
	return fmt.Sprintf("Received PrimaryKey: %s", event.PrimaryKey), nil
}

func main() {
	lambda.Start(handleRequest)
}
```

- Build and Package for Lambda:
1. Build the binary for Linux:
   ```bash
   GOOS=linux GOARCH=amd64 go build -o main
   ```
2. Package the binary in a zip file:
   ```bash
   zip function.zip main
   ```
3. Deploy to Lambda via the AWS Management Console or CLI:
   ```bash
   aws lambda create-function \
       --function-name MyLambdaFunction \
       --runtime go1.x \
       --role arn:aws:iam::123456789012:role/execution_role \
       --handler main \
       --zip-file fileb://function.zip
   ```

---

3c. GitHub Actions Workflow Pipeline for CI/CD
-  GitHub Actions Workflow for Docker and AWS Lambda Deployment

Create a `.github/workflows/ci-cd.yml` file:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

      . Step 1:Set up Go
         - name: Checkout code
        uses: actions/checkout@v3

      . Step 2: Set up Go
      - name: Set up Go
        uses: actions/setup-go@v4
        with:
          go-version: 1.20

      . Step 3: Install dependencies and test
      - name: Install dependencies
        run: go mod tidy

      - name: Run tests
        run: go test ./...

      . Step 4: Build Docker image
      - name: Build Docker image
        run: |
          docker build -t my-dynamodb-app .

      . Step 5: Push Docker image to Docker Hub
      - name: Push Docker image
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      - name: Push Image
        run: docker push ${{ secrets.DOCKER_USERNAME }}/my-dynamodb-app:latest

      .  Step 6: Deploy to AWS Lambda
      - name: Deploy to AWS Lambda
        uses: aws-actions/aws-lambda-deploy@v1
        with:
          function-name: MyLambdaFunction
          zip-file: ./function.zip
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          region: us-east-1
```

 ... Key Features:
- Automatically builds a Docker image and pushes it to Docker Hub.
- Deploys the app to AWS Lambda.

---

This setup enables : pagination for large datasets, secures DynamoDB access with IAM roles, 
and provides deployment pipelines for Docker, AWS Lambda, and GitHub Actions pipeline CI/CD workflows. 

7. PROBLEMS/DISADVANTAGES:
MongoDB and GoLang present a steep learning curve for new developers. MongoDB's flexible schema can lead to data inconsistencies if not carefully managed, and its transaction support
is less robust than traditional relational databases. Combining MongoDB with GoLang introduces complexity, especially with intricate queries or data relationships.
Managing both relational and non-relational databases adds further complexity, making data consistency and integration challenging, potentially increasing costs.

CONCLUSION:
Using GoLang to Build MongoDB Database App
 Combining Go and MongoDB offers a powerful solution for building scalable and efficient applications. MongoDB's flexible document model simplifies handling dynamic data, 
 aligning well with Go's structs for straightforward data mapping and rapid development. Its scalability features efficiently manage large datasets and high traffic.
 Real-world examples like Ojje, an adaptive literacy platform, demonstrate the effectiveness of this combination in personalized instruction and student progress tracking. 
 Go and MongoDB are also applicable to real-time analytics, content management, and IoT applications. While a learning curve and data consistency considerations exist
 , these can be managed with careful planning.

Integrating relational and non-relational databases provides flexibility and scalability for diverse data types and use cases. 
Relational databases excel at structured data and complex transactions, while non-relational databases handle large volumes of unstructured data efficiently, 
leading to improved performance and better data management. This combination enables deeper data insights and more informed decisions. E-commerce platforms, social media platforms,
and IoT applications are examples where this combination is beneficial. While complexity and data consistency require attention, the benefits of improved performance, better
data management, and increased data insights make it a valuable approach for organizations seeking to unlock the full potential of their data. 
By combining the strengths of both database types, organizations can achieve optimal performance, maintainability, and scalability while handling massive datasets and adhering 
to ACID properties.







