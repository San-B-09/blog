---
title: "Todo List App Backend with Go"
date: 2023-08-28
slug: "go-todo-list-backend"
description: "A quick intro to using Go routines for simple asynchronous execution in Go."
summary: "A quick intro to using Go routines for simple asynchronous execution in Go."
categories: ["golang"]
draft: false
---

This blog serves as instructional resource for building a simple todo list application backend in golang. I believe in writing good, self-explainable code and hence this blog might not go through each code line.

We’ll be building a todo app with basic crud operation on todo list items. Additionally we’ll have option to mark an item as completed or revert back to incomplete.

**TLDR**: Here is the [GitHub Link](https://github.com/San-B-09/todo_app_mux) for the complete code.

---

# Prerequisite

We are building a todo app where server will be in golang. We’ll be using mongoDb for database, hosted in a local docker container. Hence, you’re all set if you’ve basic understanding of following
- Go: Must know golang basics. If you’re completely new to go, I would recommend visiting [here](https://gobyexample.com/). You can find more info on go installation [here](https://go.dev/doc/install).
- Docker: Good to have
- MongoDb: Good to have.

Initialise a golang project and this’ll be the main you’re main server directory. You can find more about project initialisation [here](https://go.dev/doc/tutorial/getting-started).
You also need to start a mongo docker container. Make sure you start the container on port `27017` or make sure you make the changes to port in code accordingly. You can find more about starting dockerised mongodb container [here](https://www.mongodb.com/compatibility/docker).

Now that we’re all set, Let’s get to Code!

> *Note: This project is developed outside the GOPATH due to which the local imports are working.*

---

# Project Structure
I’ve been following concentric circle coding architectural style as it has worked really well for me. It keeps good balance in being simple yet maintaining independency with UI, DB and external agents and having good testability at the same time. This coding style makes the application as a plug and play, where any external agent or additional db can be plugged in with minimal changes to central business logic of the application. You can find more about the architectural style [here](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html).

We’ll initialise a golang project as `todo_app`. The `server` directory structure structure will be:
```lua
todo_app
 |- db
 |- domain
 |- webservice
 |- models
 |- main.go
```

Remember, this is 30K feet view of the directory and we’ll be going through individual layer in detail further down the blog. Each directory works as an independent layer and will be functional in following ways:
- `webservice`: Handles and routes all the RESTful requests. It communicates only with domain layer.In a way, except webservice all other layers remains abstracted from user.
- `domain`: Handles all business logic and serves as a central layer where all others layers can be plugged-in.
- `db`: This layer communicates with database (mongoDb in our case) and handle db crud operations. It will be plugged into domain layer and can be only accessed from there.
- `main.go`: This is the entry point of our application. We’ll initialise all layers in main.go and start the server.

Except these directories, additionally we can have a `model` and `constant` directory under parent `server` to declare models and constants being used across all layers. For layer specific models and constants, we can declare them individually under respective layers.

---

# Models

We have a models directory under main `server` directory. Here we are defining models that we need to access across all layers. `models` directory is structured as following:
```lua
todo_app
 |- models
    |- todoModels.go
```

Here we are defining `TodoListItem` struct. This is defined in `todoModels.go` on a global level and will be shared across layers.
```go
package models

type TodoListItem struct {
	Id          string `json:"id" bson:"_id,omitempty"`
	Item        string `json:"item" bson:"item,omitempty"`
	IsCompleted bool   `json:"isCompleted" bson:"isCompleted,omitempty"`
}
```

As, we are using mongo as our database, we have defined `TodoListItem` members with `json` and `bson` keys. `json` keys are used to interact with UI, while `bson` keys are used to interact with mongodb. You can read more about `bson` keys [here](https://www.mongodb.com/docs/drivers/go/v1.11/usage-examples/struct-tagging/).

---

# Mongo/DB Layer

We’ve all db functions required for todo list app in DB layer. Here, we interact with database, create db filters and options but do not perform any modification or operation on data. DB later is structured as:
```go
todo_app
 |- db
    |- mongo
       |- mongo.go
       |- todoItem.go
       |- constants.go
    |- db.go
```

We are first defining a db interface with all db functions needed for todo list app. This is defined in db.go under main db directory as following:

```go
package db

import (
	"context"
	"todo_app_mux/models"
)

type Idb interface {
	AddItemToDb(ctx context.Context, item string) error
	GetItemsFromDb(ctx context.Context) ([]models.TodoListItem, error)
	UpdateItemFromDb(ctx context.Context, itemId, item string) error
	DeleteItemFromDb(ctx context.Context, itemId string) error
	UpdateItemCompletedStatus(ctx context.Context, itemId string, itemCompleteStatus bool) error
}
```

Remember, in go interface makes it easier to wrap any object on it’s functions. Here with db interface we can define any db client with same functions as interface and pass db interface to domain, hence keeping domain logic abstracted of db client used for db operations.

For our todo list app, we are using mongoDb as our database. We have created a `mongoService` struct containing mongo client as its member. We’ve also defined a `New` function that takes `mongoUrl` as argument, creates mongodb client and returns `mongoService` struct as db interface.

> *Note: As I’ve already implemented functions defined in db interface (Idb) with mongoService, returning mongoService as Idb isn’t throwing error otherwise it’ll throw error due to incomplete method implementation.*

```go
package mongo

import (
	"context"
	"go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/options"
	"log"
	"todo_app_mux/db"
)

type mongoService struct {
	db mongo.Client
}

func New(ctx context.Context, mongoUrl string) db.Idb {
	mongoClient, err := mongo.Connect(ctx, options.Client().ApplyURI(mongoUrl))
	if err != nil {
		log.Fatalf("Error initiating mongo client")
		return &mongoService{}
	}

	return &mongoService{
		db: *mongoClient,
	}
}
```

Now the only thing left in our db layer is adding `mongoService` methods, completing db functionalities as discussed at the beginning.

Here we’ve simple `AddItemToDb` method that takes item as string, creates `TodoListItem` model with item passed in argument and add same to the db. We return error in this method incase any error is incurred while adding todo list item to db. In `AddItemToDb`, `defaultDb` and `todoListCollection` are db and collection names defined in `constants.go` under mongo layer.

I’ve added only one mongo methods as sample while you can find implementation for all methods [here](https://github.com/San-B-09/todo_app_mux/blob/master/db/mongo/todoItem.go).
```go
package mongo

import (
	"context"
	"go.mongodb.org/mongo-driver/bson"
	"todo_app_mux/models"
)

func (m *mongoService) AddItemToDb(ctx context.Context, item string) error {
	insertObject := models.TodoListItem{
		Item: item,
	}

	collection := m.db.Database(defaultDb).Collection(todoListCollection)
	_, err := collection.InsertOne(ctx, insertObject)
	if err != nil {
		return err
	}

	return nil
}
```

---

# Domain Layer

Domain layer is the central layer of our application. This is where we perform all business logics and interact with db or any external agent. Domain layer is structured as following:
```lua
todo_app
 |- domain
    |- standard
       |- standard.go
       |- todoItem.go
    |- domain.go
```

We are first defining a domain interface with the domain functions needed for todo list app. This is defined in `domain.go` under main `db` directory as following:
```go
package domain

import (
	"context"
	"todo_app_mux/models"
)

type IDomainService interface {
	AddItemToList(ctx context.Context, item string) error
	GetTodoItems(ctx context.Context) ([]models.TodoListItem, error)
	UpdateItemFromTodo(ctx context.Context, itemId, item string) error
	DeleteItemFromTodo(ctx context.Context, itemId string) error
	MarkItemComplete(ctx context.Context, itemId string) error
	MarkItemIncomplete(ctx context.Context, itemId string) error
}
```

Furthermore, similar to mongo layer, we are defining a domain client `domainService`. Domain client is responsible to implement all domain functions and return `IDomainService` as domain interface that can be called through webservice layer.
```go
package standard

import (
	"todo_app_mux/db"
	"todo_app_mux/domain"
)

type domainService struct {
	db db.Idb
}

func New(dbService db.Idb) domain.IDomainService {
	return &domainService{
		db: dbService,
	}
}
```

We’ve defined a `todoItem.go` under `standard` directory in domain where we’ve defined all `domainService` methods.

We have a method `AddItemToList` that takes `item` as argument and calls db layer to add item to db. Here we’ve also added an additional check to check if item received to domain function is empty and returned error accordingly.

I’ve added only one domain methods as sample while you can find implementation for all methods [here](https://github.com/San-B-09/todo_app_mux/blob/master/domain/standard/todoItem.go).
```go
package standard

import (
	"context"
	"errors"
	"log"
)

func (s *domainService) AddItemToList(ctx context.Context, item string) error {
	if item == "" {
		log.Println(ctx, errors.New("Item can't be empty"))
		return errors.New("Item can't be empty")
	}

	err := s.db.AddItemToDb(ctx, item)
	if err != nil {
		log.Println(ctx, err)
		return err
	}

	return nil
}
```

---

# Webservice Layer
This is the layer that interacts with the outer world. Here we are defining all the routes to our application for the functionalities discussed at the start. Webservice layer is structured as following.
```lua
todo_app
 |- webservice
    |- models.go
    |- webservice.go
    |- routes.go
    |- todoItem.go
```

We’ve defined models for the api response in `models.go`. Here we’ve `meta` and `data` in our response structure. In `meta` we can respond with success/error code and message and in `data` we can respond with the data for read requests.
```go
package webservice

type httpResponse struct {
	Meta httpResponseMeta `json:"meta"`
	Data interface{}      `json:"data"`
}

type httpResponseMeta struct {
	Code    int64  `json:"code"`
	Message string `json:"message"`
}
```

We are now creating a `webservice.go` and where we’ve a `webService` struct containing domain layer field. While creating a new webservice layer, we will pass domain and that way we can now access the domain layer through webservice layer while keeping domain and db as completely abstract.

`webService` has a method `Start` where we are using gorilla mux for creating router. We’ve then added all the routes to initialised router and created a handler as `serverHandler` that listens on local port `8080`.

We are also defining functions `ReturnOKResponse` and `ReturnErrorResponse` that will be used in webservice methods to respond with data or error code and error message as their name suggests.

Lastly we’ve defined a `Ping` function that simply logs the server pings and respond with ping success. `Ping` functions are generally used to monitor the health of the server and generate alerts incase server goes down.
```go

package webservice

import (
	"bytes"
	"context"
	"encoding/json"
	"github.com/gorilla/handlers"
	"github.com/gorilla/mux"
	"log"
	"net/http"
	"todo_app_mux/domain"
)

type webService struct {
	domain domain.IDomainService
}

func New(domain domain.IDomainService) *webService {
	return &webService{
		domain: domain,
	}
}

func (s *webService) Start(ctx context.Context) {
	router := mux.NewRouter().StrictSlash(true)

	s.addBasicRoutes(router)

	serverHandler := handlers.CORS()(router)
	err := http.ListenAndServe(":8080", serverHandler)
	if err != nil {
		log.Println(ctx, err, map[string]interface{}{"msg": "Error starting the server:"})
	}

	return
}

func ReturnOKResponse(ctx context.Context, w http.ResponseWriter, data interface{}) {
	response := httpResponse{
		Meta: httpResponseMeta{
			Code:    http.StatusOK,
			Message: "success",
		},
		Data: data,
	}

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	var buf = new(bytes.Buffer)
	enc := json.NewEncoder(buf)
	err := enc.Encode(response)
	if err != nil {
		log.Println(ctx, err)
	}
	_, err = w.Write(buf.Bytes())
	if err != nil {
		log.Println(ctx, err)
	}
}

func ReturnErrorResponse(ctx context.Context, w http.ResponseWriter, errorCode int64, errorMessage string) {
	response := httpResponse{
		Meta: httpResponseMeta{
			Code:    errorCode,
			Message: errorMessage,
		},
	}

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	var buf = new(bytes.Buffer)
	enc := json.NewEncoder(buf)
	err := enc.Encode(response)
	if err != nil {
		log.Println(ctx, err)
	}
	_, err = w.Write(buf.Bytes())
	if err != nil {
		log.Println(ctx, err)
	}
}

func (s *webService) Ping(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	log.Println(ctx, "Server Pinged")
	ReturnOKResponse(ctx, w, "pong")
}
```

We are now defining the routes in a webservice method `addBasicRoutes` that takes router as argument and registers new application endpoints with routed functions. I’ve followed a specific naming convention for these routes and you can find more about the same [here](https://medium.com/@nadinCodeHat/rest-api-naming-conventions-and-best-practices-1c4e781eb6a5).
```go
package webservice

import (
	"github.com/gorilla/mux"
	"net/http"
)

func (s *webService) addBasicRoutes(router *mux.Router) {
	router.HandleFunc("/ping", s.Ping).Methods(http.MethodGet)

	router.HandleFunc("/item", s.AddItem).Methods(http.MethodPost)
	router.HandleFunc("/items", s.GetItems).Methods(http.MethodGet)
	router.HandleFunc("/item/{item-id}", s.UpdateItem).Methods(http.MethodPut)
	router.HandleFunc("/item/{item-id}", s.DeleteItem).Methods(http.MethodDelete)

	router.HandleFunc("/item/mark-complete/{item-id}", s.MarkItemComplete).Methods(http.MethodPut)
	router.HandleFunc("/item/mark-incomplete/{item-id}", s.MarkItemIncomplete).Methods(http.MethodPut)
}
```

These routed functions registered against the application endpoints are defined in `todoItem.go`. We’ve a webservice method `AddItem` that reads item passed in request body and calls domain layer to add the item to todo list.

I’ve added only one webservice methods as sample while you can find for all methods implementation [here](https://github.com/San-B-09/todo_app_mux/blob/master/webservice/todoItem.go).
```go
package webservice

import (
	"encoding/json"
	"log"
	"net/http"
	"todo_app_mux/models"
)

func (s *webService) AddItem(w http.ResponseWriter, r *http.Request) {
	ctx := UpgradeContext(r.Context())
	var item models.TodoListItem
	err := json.NewDecoder(r.Body).Decode(&item)
	if err != nil {
		log.Println(ctx, err)
		ReturnErrorResponse(ctx, w, http.StatusBadRequest, "Error decoding request")
		return
	}

	err = s.domain.AddItemToList(ctx, item.Item)
	if err != nil {
		log.Println(ctx, err)
		ReturnErrorResponse(ctx, w, http.StatusInternalServerError, "Error adding item to todo list")
		return
	}

	ReturnOKResponse(ctx, w, "success")
	return
}
```

# main.go

Lastly we are defining `main.go` that is the entry point of our application. Here we define all the layers and call `Start` method from webservice layer. While defining the layers, all required layers are plugged in like mongoService layer is plugged into domain layer.
```go
package main

import (
	"context"
	"log"
	"os"
	"os/signal"
	"syscall"
	"todo_app_mux/db/mongo"
	"todo_app_mux/domain/standard"
	"todo_app_mux/webservice"
)

func main() {
	ctx := context.Background()

	mongoService := mongo.New(ctx, "mongodb://localhost:27017/")
	domainService := standard.New(mongoService)
	webService := webservice.New(domainService)
	log.Println(ctx, "Starting Server ")
	webService.Start(ctx)

	done := make(chan os.Signal, 1)
	signal.Notify(done, os.Interrupt, syscall.SIGINT, syscall.SIGTERM)
	<-done
	// Stop and close the connections
	log.Println(ctx, "Stopping Server ")
}
```

---

And Tada!! You’re done.
Start your server, play around with the todo list and feel free to make any additional/enhancements as per you’re requirements.

**You can find the complete code [here](https://github.com/San-B-09/todo_app_mux).**

> *Note: Complete code has some additional enhancements for logging. Here we are generated a new request id every time some http request comes to the server and same is logged while logging context. This way we can track the request logs throughout all layers.*

