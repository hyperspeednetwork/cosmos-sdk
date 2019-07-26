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

One of the main interfaces for an application is the [command-line interface](../interfaces/cli.md). This entrypoint created by the application developer will add commands from the application's modules to make [**transactions**](../core/transactions.md) and [**queries**](../building-modules/messages-and-queries.md).  The CLI files are typically found in the module's `/client/cli` folder.

### Transaction Commands

[Transactions](../core/transactions.md) are created by users to wrap messages that trigger state changes in applications. Transaction commands typically have their own `tx.go` file in the module `/client/cli` folder. The commands are specified in getter functions prefixed with `GetCmd` followed by the name of the command. Getter functions should do the following:

- **Argument: `codec`**. The getter function takes in an application `codec` as an argument and returns a reference to the `cobra.command`.
- **Construct the command.** Read the [Cobra Documentation](https://github.com/spf13/cobra) for details on how to create commands.
- **`RunE.`** The function should be specified as a `RunE` to allow for errors to be returned. This function encapsulates all of the logic to create a new transaction that is ready to be relayed to nodes.
  + The function should first initialize a [`TxBuilder`](,,/core/transactions.md#txbuilder) with the application `codec`'s `TxEncoder`, as well as a new [`CLIContext`](./query-lifecycle.md#clicontext) with the `codec` and `AccountDecoder` from the application `codec`.
  + If applicable, the `CLIContext` is used to retrieve any parameters such as the transaction originator's address to be used in the transaction.
  + A [message](./messages.md) is created using all parameters parsed from the command arguments and `CLIContext`.
  + The transaction is either generated offline or signed and broadcasted to the preconfigured node, depending on what the user wants.
- **Flags.** Add any [flags](#flags) to the command.

Finally, the module needs to have a `GetTxCmd`, which aggregates all of the transaction commands of the module. Application developers wishing to include the module's transactions will call this function to add them as subcommands in their CLI.

### Query Commands

[Queries](./query.md) allow users to gather information about the application or network state. Query commands typically have their own `query.go` file in the module `/client/cli` folder. Like transaction commands, they are specified in getter functions and have the prefix `GetCmdQuery`. Getter functions should do the following:
- **Arguments: `codec` and `queryRoute`.** In addition to taking in the application `codec`, query command getters also take a `queryRoute` used to construct a path [Baseapp](../core/baseapp.md) uses to route the query in the application.
- **Construct the command.** Read the [Cobra Documentation](https://github.com/spf13/cobra) for details on how to create commands.
- **`RunE`.** The function should be specified as a `RunE` to allow for errors to be returned. This function encapsulates all of the logic to create a new query that is ready to be relayed to nodes.
  + The function should first initialize a new [`CLIContext`](./query-lifecycle.md#clicontext) with the application `codec`.
  + If applicable, the `CLIContext` is used to retrieve any parameters (e.g. the query originator's address to be used in the query) and marshal them with the query parameter type, in preparation to be relayed to a node.
  + Use the `queryRoute` to construct a route Baseapp will use to route the query to the appropriate [querier](./querier.md). Module queries are `custom` type queries.  
  + Call the `CLIContext` query function to relay the query to a node and retrieve the response.
  + Unmarshal the response and use the `CLIContext` to print the output back to the user.
- **Flags.** Add any [flags](#flags) to the command.


Finally, the module also needs a `GetQueryCmd`, which aggregates all of the query commands of the module. Application developers wishing to include the module's queries will call this function to add them as subcommands in their CLI.

### Flags

[Flags](../interfaces/cli.md#flags) are entered by the user and allow for command customizations. Examples include the fees or gas prices users are willing to pay for their transactions.

The flags for a module are typically found in the `flags.go` file in the `/client/cli` folder. Module developers can create a list of possible flags including the value type, default value, and a description displayed if the user uses a `help` command. In each transaction getter function, they can add flags to the commands and, optionally, mark flags as _required_ so that an error is thrown if the user does not provide values for them.

For full details on flags, visit the [Cobra Documentation](https://github.com/spf13/cobra).

## REST

Application users may find the most intuitive methods of interfacing with the application are web services that use HTTP requests (e.g. a web wallet like [Lunie.io](lunie.io)). Thus, application developers will also use REST Routes to route HTTP requests to the application's modules. The module developer's responsibility is to define the REST client by defining routes for all possible requests and handlers for each of them. The REST interface file is typically found in the module's `/client/rest` folder.

### Request Types

Request types must be defined for all *transaction* requests. Conventionally, each request is named with the suffix `Req`, e.g. `SendReq` for a Send transaction. Each struct should include a base request [`baseReq`](../interfaces/rest.md#basereq), the name of the transaction, and all the arguments the user must provide for the transaction.

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
