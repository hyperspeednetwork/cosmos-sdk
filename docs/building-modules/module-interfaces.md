# Module Interfaces

## Prerequisites
* [Application CLI](.//interfaces/cli.md)
* [Application REST Interface](.//interfaces/cli.md)

## Synopsis

This document details how to build CLI and REST interfaces for a module.
- [CLI](#cli)
  + [Transaction Commands](#tx-commands)
  + [Query Commands](#query-commands)
- [REST](#rest)
  + [Request Types](#request-types)
  + [Request Handlers](#request-handlers)
  + [Register Routes](#register-routes)

## CLI

One of the main interfaces for an application is the [command-line interface](../interfaces/cli.md). This entrypoint created by the application developer will add commands from the application's modules to make [**transactions**](../core/transactions.md) and [**queries**]().  

### Transaction Commands

### Query Commands

## REST

Application users may find the most intuitive methods of interfacing with the application are web services that use HTTP requests (e.g. a web wallet like [Lunie.io](lunie.io)). Thus, application developers will also use REST Routes to route HTTP requests to the application's modules. The module developer's responsibility is to define the REST client by defining routes for all possible requests and handlers for each of them.

### Request Types

Request types must be defined for all *transaction* requests. Conventionally, each request is named with the suffix `Req`, e.g. `SendReq` for a Send transaction. Each struct should include a base request `baseReq`, the name of the transaction, and all the arguments the user must provide for the transaction.

`BaseReq` is a type defined in the SDK that encapsulates much of the transaction configurations similar to CLI command flags:

* From
*	Memo
*	ChainID
*	AccountNumber
*	Sequence
*	Fees
*	GasPrices
*	Gas  
*	GasAdjustment
*	Simulate      

### Request Handlers

Request handlers must be defined for both transaction and query requests. Handlers' arguments include a reference to the application's `codec` and the [`CLIContext`](../interfaces/query-lifecycle.md#clicontext) created in the user interaction.

### Register Routes

The entrypoint `RegisterRoutes` function of the application will call the  `registerRoutes` functions of each module utilized by the application.

The router used by the SDK is [Gorilla Mux](https://github.com/gorilla/mux). The router is initialized with the Gorilla Mux `NewRouter()` function. Then, the router's `HandleFunc` function can then be used to route urls with the defined request handlers and the HTTP method (e.g. "POST", "GET") as a route matcher. It is recommended to prefix every route with the name of the module to avoid collisions with other modules that have the same query or transaction names.


#### Examples

Here is an example of a query route from the [nameservice tutorial](https://cosmos.network/docs/tutorial/rest.html):

``` go
// ResolveName Query
r.HandleFunc(fmt.Sprintf("/%s/names/{%s}", storeName, restName), resolveNameHandler(cdc, cliCtx, storeName)).Methods("GET")
```

A few things to note:

* `"/%s/names/{%s}", storeName, restName` is the url for the HTTP request. `storeName` is the name of the module, `restName` is a variable provided by the user to specify what kind of query they are making.
* `resolveNameHandler` is the query request handler defined by the module developer. It also takes the application `codec` and `CLIContext` passed in from the user side, as well as the `storeName`.
* `"GET"` is the HTTP Request method. As to be expected, queries are typically GET requests. Transactions are typically POST and PUT requests.
