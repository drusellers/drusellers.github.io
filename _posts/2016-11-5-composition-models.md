---
layout: post
title:  "Composing Application Styles"
date:   2016-11-05 05:58:21
categories: architecture
published: false
disqus_id: 8301429b-f964-406d-8827-5317dc351a34
---

# C# with IoC/DI Container

- leverage interfaces and classes
- compose through the constructor

## Service
```csharp
public interface IRepository { void Save<T>(T entity); }
```

## Implementation
```csharp
public class DatabaseRepository
{
    string _connectionString;

    public DatabaseRepository(string connectionString)
    {
        _connectionString = connectionString;
    }

    public void Save<T>(T entity)
    {
        /* elided */
    }
}
```

## Usage in Controller

```csharp
public class OrderController
{
    readonly IRepository _repository;

    public OrderController(IRepository repository)
    {
      _repository = repository;
    }

    // Assume a JSON -> Order deserialization
    public ActionResult Post(Order order)
    {
        _repository.Save(order);
    }
}
```

What is effectively happening when we are saving an object looks like this.

```
new DatabaseRepository("a connection string").Save(someObject);

// with controller
new OrderController(new DatabaseRepository("a connection string")).Post(anOrder);
```

This reminds me a lot of how I would compose the same problem in F#.

```fsharp
// meta environment detection
let connString = System.Configuration.ConfigurationManager::AppSetting["dbconn"];

module DatabaseRepository
  let databaseSaveEntity (connectionString : string ) (entity : 'a) : () =
    // elided
    ()

// Done by IoC + Interface Implementation
module Repository
  let saveEntity = databaseSaveEntity connString

// based on Suave.io
module OrdersActions

  let orderPost (httpContent) -> () =
    let order = getOrder httpContent
    saveEntity order
    ()
```

Both of these languages are composing their functions.

```
(ns environment)

  (def connection-string (env "dbConn"))

(ns repository.database)
  (defn save [connection-string entity]
    (# do stuff))

(ns repository
  (:require [repository.database :as database]
            [environment]))


  (def save-entity (partial database/save environment/connection-string))

(ns http-actions
  (: require [repository :as repo]))

  (defn raw-order-post [repository-save request]
    (let [order (get-order request)]
      (repository-save order)))

  (def order-post (partial raw-order-post repository/save))
```

This is how you might do the same thing with Clojure, but clojure has a really
nice model for composing applications called [Component](https://github.com/stuartsierra/component)
whose real goal is to manage the lifecycle of stateful objects.

So, now the question is. How do we do this in Rust?
