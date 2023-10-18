﻿![Mobile Buy SDK](https://cloud.githubusercontent.com/assets/5244861/26374751/6895a582-3fd4-11e7-80c4-2c1632262d66.png)

[![Tests](https://github.com/Shopify/mobile-buy-sdk-ios/workflows/Test/badge.svg)](https://github.com/Shopify/mobile-buy-sdk-ios/actions/workflows/test.yml)
[![GitHub Release](https://img.shields.io/github/release/shopify/mobile-buy-sdk-ios.svg?style=flat)](https://github.com/Shopify/mobile-buy-sdk-ios/releases/latest)
[![Carthage compatible](https://img.shields.io/badge/Carthage-compatible-4BC51D.svg?style=flat)](https://github.com/Carthage/Carthage)
[![Swift Package Manager compatible](https://img.shields.io/badge/Swift%20Package%20Manager-compatible-4BC51D.svg?style=flat)](https://swift.org/package-manager/)
[![GitHub license](https://img.shields.io/badge/license-MIT-lightgrey.svg?style=flat)](https://github.com/Shopify/mobile-buy-sdk-ios/blob/main/LICENSE)

# Mobile Buy SDK

The Mobile Buy SDK makes it easy to create custom storefronts in your mobile app. The SDK connects to the Shopify platform using the Storefront GraphQL API, and supports a wide range of native storefront experiences.

### Documentation

You can generate a complete HTML and `.docset` documentation by running the `Documentation` scheme. You can then use a documentation browser like [Dash](https://kapeli.com/dash) to access the `.docset` artifact, or browse the HTML directly in the `Docs/Buy` and `Docs/Pay` directories.

The documentation is generated using [Jazzy](https://github.com/realm/jazzy).

## Installation

<a href="https://github.com/Shopify/mobile-buy-sdk-ios/releases/latest">Download the latest version</a>

### Swift Package Manager

This is the recommended approach of integration the SDK with your app. You can follow Apple's guide for [adding a package dependency to your app](https://developer.apple.com/documentation/xcode/adding_package_dependencies_to_your_app) for a thorough walkthrough.

### Dynamic Framework Installation

1. Add `Buy` as a git submodule by running:

```
git submodule add git@github.com:Shopify/mobile-buy-sdk-ios.git
```

2. Ensure that all submodules of `Buy SDK` have also been updated by running:

```
git submodule update --init --recursive
```

3. Drag the `Buy.xcodeproj` into your application project.
4. Add **Buy.framework** target as a dependency:
   1. Navigate to **Build Phases** > **Target Dependencies**.
   2. Add `Buy.framework`.
5. Link `Buy.framework`:
   1. Navigate to **Build Phases** > **Link Binary With Libraries**.
   2. Add `Buy.framework`.
6. Make sure that the framework is copied into the bundle:
   1. Navigate to **Build Phases** > **New Copy Files Phase**.
   2. From the **Destination** dropdown, select **Frameworks**.
   3. Add `Buy.framework`.
7. Import into your project files using `import Buy`.

See the **Storefront** sample app for an example of how to add the `Buy` target a dependency.

### Carthage

1. Add the following line to your Cartfile:

```ruby
github "Shopify/mobile-buy-sdk-ios"
```

2. Run `carthage update`.
3. Follow the [steps to link the dynamic framework](#dynamic-framework-installation-) that Carthage produced.
4. Import the SDK module:

```swift
import Buy
```

### CocoaPods

1. Add the following line to your podfile:

```ruby
pod "Mobile-Buy-SDK"
```

2. Run `pod install`.
3. Import the SDK module:

```swift
import MobileBuySDK
```

Note: If you've forked this repo and are attempting to install from your own git destination, commit, or branch, be sure to include "submodules: true" in the line of your Podfile

## Getting started

The Buy SDK is built on [GraphQL](http://graphql.org/). The SDK handles all the query generation and response parsing, exposing only typed models and compile-time checked query structures. It doesn't require you to write stringed queries, or parse JSON responses.

You don't need to be an expert in GraphQL to start using it with the Buy SDK (but it helps if you've used it before). The sections below provide a brief introduction to this system, and some examples of how you can use it to build secure custom storefronts.

## Migration from SDK v2.0

The previous version of the Mobile Buy SDK (version 2.0) is based on a REST API. With version 3.0, Shopify is migrating the SDK from REST to GraphQL.

Unfortunately, the specifics of generation GraphQL models make it almost impossible to create a migration path from v2.0 to v3.0 (domains models are not backwards compatible). However, the main concepts are the same across the two versions, such as collections, products, checkouts, and orders.

## Code Generation

The Buy SDK is built on a hierarchy of generated classes that construct and parse GraphQL queries and responses. These classes are generated manually by running a custom Ruby script that relies on the [GraphQL Swift Generation](https://github.com/Shopify/graphql_swift_gen) library. Most of the generation functionality and supporting classes live inside the library. It works by downloading the GraphQL schema, generating Swift class hierarchy, and saving the generated files to the specified folder path. In addition, it provides overrides for custom GraphQL scalar types like `DateTime`.

### Request Models

All generated request models are derived from the `GraphQL.AbstractQuery` type. Although this abstract type contains enough functionality to build a query, you should never use it directly. Instead, rely on the typed methods provided in the generated subclasses.

The following example shows a sample query for a shop's name:

```swift
let query = Storefront.buildQuery { $0
    .shop { $0
        .name()
    }
}
```

Never use the abstract class directly:

```swift
// Never do this

let shopQuery = GraphQL.AbstractQuery()
shopQuery.addField(field: "name")

let query = GraphQL.AbstractQuery()
query.addField(field: "shop", subfields: shopQuery)

```

Both of the above queries produce identical GraphQL queries (see below), but the former approach provides auto-completion and compile-time validation against the GraphQL schema. It will surface an error if a requested field doesn't exist, isn't the correct type, or is deprecated. You also might have noticed that the former approach resembles the GraphQL query language structure (this is intentional). The query is both easier to write and much more legible.

```graphql
query {
  shop {
    name
  }
}
```

### Response models

All generated response models are derived from the `GraphQL.AbstractResponse` type. This abstract type provides a similar key-value type interface to a `Dictionary` for accessing field values in GraphQL responses. Just like `GraphQL.AbstractQuery`, you should never use these accessors directly, and instead rely on typed, derived properties in generated subclasses.

The following example builds on the earlier example of accessing the result of a shop name query:

```swift
// response: Storefront.QueryRoot

let name: String = response.shop.name
```

Never use the abstract class directly:

```swift
// Never do this

let response: GraphQL.AbstractResponse

let shop = response.field("shop") as! GraphQL.AbstractResponse
let name = shop.field("name") as! String
```

Again, both of the approaches produce the same result, but the former case is preferred: it requires no casting since it already knows about the expected type.

### The `Node` protocol

The GraphQL schema defines a `Node` interface that declares an `id` field on any conforming type. This makes it convenient to query for any object in the schema given only its `id`. The concept is carried across to the Buy SDK as well, but requires a cast to the correct type. You need to make sure that the `Node` type is of the correct type, otherwise casting to an incorrect type will return a runtime exception.

Given this query:

```swift
let id    = GraphQL.ID(rawValue: "gid://shopify/Product/123")
let query = Storefront.buildQuery { $0
    .node(id: id) { $0
        .onProduct { $0
            .id()
            .title()
        }
    }
}
```

The `Storefront.Product` requires a cast:

```swift
// response: Storefront.QueryRoot

let product = response.node as! Storefront.Product
```

#### Aliases

Aliases are useful when a single query requests multiple fields with the same names at the same nesting level, since GraphQL allows only unique field names. Multiple nodes can be queried by using a unique alias for each one:

```swift
let query = Storefront.buildQuery { $0
    .node(aliasSuffix: "collection", id: GraphQL.ID(rawValue: "NkZmFzZGZhc")) { $0
        .onCollection { $0
            // fields for Collection
        }
    }
    .node(aliasSuffix: "product", id: GraphQL.ID(rawValue: "GZhc2Rm")) { $0
        .onProduct { $0
            // fields for Product
        }
    }
}
```

Accessing the aliased nodes is similar to a plain node:

```swift
// response: Storefront.QueryRoot

let collection = response.aliasedNode(aliasSuffix: "collection") as! Storefront.Collection
let product    = response.aliasedNode(aliasSuffix: "product")    as! Storefront.Product
```

Learn more about [GraphQL aliases](http://graphql.org/learn/queries/#aliases).

## Graph.Client

The `Graph.Client` is a network layer built on top of `URLSession` that executes `query` and `mutation` requests. It also simplifies polling and retrying requests. To get started with `Graph.Client`, you need the following:

- Your shop's `.myshopify.com` domain
- Your API key, which you can find in your shop's admin page
- A `URLSession` (optional), if you want to customize the configuration used for network requests or share your existing `URLSession` with the `Graph.Client`
- (optional) The buyer's current locale. Supported values are limited to locales available to your shop.

```swift
let client = Graph.Client(
	shopDomain: "shoes.myshopify.com",
	apiKey:     "dGhpcyBpcyBhIHByaXZhdGUgYXBpIGtleQ"
)
```

If your store supports multiple languages, then the Storefront API can return translated resource types and fields. Learn more about [translating content](https://shopify.dev/tutorials/manage-app-translations-with-admin-api).

```swift
// Initializing a client to return translated content
let client = Graph.Client(
	shopDomain: "shoes.myshopify.com",
	apiKey:     "dGhpcyBpcyBhIHByaXZhdGUgYXBpIGtleQ",
        locale:     Locale.current
)
```

GraphQL specifies two types of operations: queries and mutations. The `Client` exposes these as two type-safe operations, although it also offers some conveniences for retrying and polling in each one.

### Queries

Semantically, a GraphQL `query` operation is equivalent to a `GET` RESTful call. It guarantees that no resources will be mutated on the server. With `Graph.Client`, you can perform a query operation using:

```swift
public func queryGraphWith(_ query: Storefront.QueryRootQuery, retryHandler: RetryHandler<Storefront.QueryRoot>? = default, completionHandler: QueryCompletion) -> Task
```

The following example shows how you can query for a shop's name:

```swift
let query = Storefront.buildQuery { $0
    .shop { $0
        .name()
    }
}

let task = client.queryGraphWith(query) { response, error in
    if let response = response {
        let name = response.shop.name
    } else {
        print("Query failed: \(error)")
    }
}
task.resume()
```

Learn more about [GraphQL queries](http://graphql.org/learn/queries/).

### Mutations

Semantically a GraphQL `mutation` operation is equivalent to a `PUT`, `POST` or `DELETE` RESTful call. A mutation is almost always accompanied by an input that represents values to be updated and a query to fetch fields of the updated resource. You can think of a `mutation` as a two-step operation where the resource is first modified, and then queried using the provided `query`. The second half of the operation is identical to a regular `query` request.

With `Graph.Client` you can perform a mutation operation using:

```swift
public func mutateGraphWith(_ mutation: Storefront.MutationQuery, retryHandler: RetryHandler<Storefront.Mutation>? = default, completionHandler: MutationCompletion) -> Task
```

The following example shows how you can reset a customer's password using a recovery token:

```swift
let customerID = GraphQL.ID(rawValue: "YSBjdXN0b21lciBpZA")
let input      = Storefront.CustomerResetInput.create(resetToken: "c29tZSB0b2tlbiB2YWx1ZQ", password: "abc123")
let mutation   = Storefront.buildMutation { $0
    .customerReset(id: customerID, input: input) { $0
        .customer { $0
            .id()
            .firstName()
            .lastName()
        }
        .userErrors { $0
            .field()
            .message()
        }
    }
}

let task = client.mutateGraphWith(mutation) { response, error in
    if let mutation = response?.customerReset {

        if let customer = mutation.customer, !mutation.userErrors.isEmpty {
            let firstName = customer.firstName
            let lastName = customer.lastName
        } else {

            print("Failed to reset password. Encountered invalid fields:")
            mutation.userErrors.forEach {
                let fieldPath = $0.field?.joined() ?? ""
                print("  \(fieldPath): \($0.message)")
            }
        }

    } else {
        print("Failed to reset password: \(error)")
    }
}
task.resume()
```

A mutation will often rely on some kind of user input. Although you should always validate user input before posting a mutation, there are never guarantees when it comes to dynamic data. For this reason, you should always request the `userErrors` field on mutations (where available) to provide useful feedback in your UI regarding any issues that were encountered in the mutation query. These errors can include anything from `Invalid email address` to `Password is too short`.

Learn more about [GraphQL mutations](http://graphql.org/learn/queries/#mutations).

### Retry

Both `queryGraphWith` and `mutateGraphWith` accept an optional `RetryHandler<R: GraphQL.AbstractResponse>`. This object encapsulates the retry state and customization parameters for how the `Client` will retry subsequent requests (such as after a set delay, or a number of retries). By default, the `retryHandler` is nil and no retry behavior will be provided. To enable retry or polling, create a handler with a condition. If the `handler.condition` and `handler.canRetry` evaluate to `true`, then the `Client` will continue executing the request:

```swift
let handler = Graph.RetryHandler<Storefront.QueryRoot>() { (query, error) -> Bool in
    if myCondition {
        return true // will retry
    }
    return false // will complete the request, either succeed or fail
}
```

The retry handler is generic, and can handle both `query` and `mutation` requests equally well.

### Caching

Network queries and mutations can be both slow and expensive. For resources that change infrequently, you might want to use caching to help reduce both bandwidth and latency. Since GraphQL relies on `POST` requests, we can't easily take advantage of the HTTP caching that's available in `URLSession`. For this reason, the `Graph.Client` is equipped with an opt-in caching layer that can be enabled client-wide or on a per-request basis.

**IMPORTANT:** Caching is provided only for `query` operations. It isn't available for `mutation` operations or for any other requests that provide a `retryHandler`.

There are four available cache policies:

- `.cacheOnly` - Fetch a response from the cache only, ignoring the network. If the cached response doesn't exist, then return an error.
- `.networkOnly` - Fetch a response from the network only, ignoring any cached responses.
- `.cacheFirst(expireIn: Int)` - Fetch a response from the cache first. If the response doesn't exist or is older than `expireIn`, then fetch a response from the network
- `.networkFirst(expireIn: Int)` - Fetch a response from the network first. If the network fails and the cached response isn't older than `expireIn`, then return cached data instead.

#### Enable client-wide caching

You can enable client-wide caching by providing a default `cachePolicy` for any instance of `Graph.Client`. This sets all `query` operations to use your default cache policy, unless you specify an alternate policy for an individual request.

In this example, we set the client's `cachePolicy` property to `cacheFirst`:

```swift
let client = Graph.Client(shopDomain: "...", apiKey: "...")
client.cachePolicy = .cacheFirst
```

Now, all calls to `queryGraphWith` will yield a task with a `.cacheFirst` cache policy.

If you want to override a client-wide cache policy for an individual request, then specify an alternate cache policy as a parameter of `queryGraphWith`:

```swift
let task = client.queryGraphWith(query, cachePolicy: .networkFirst(expireIn: 20)) { query, error in
    // ...
}
```

In this example, the `task` cache policy changes to `.networkFirst(expireIn: 20)`, which means that the cached response will be valid for 20 seconds from the time the response is received.

### Errors

The completion for either a `query` or `mutation` request will always contain an optional `Graph.QueryError` that represents the current error state of the request. **It's important to note that `error` and `response` are NOT mutually exclusive.** It is perfectly valid to have a non-nil error and response. The presence of an error can represent both a network error (such as a network error, or invalid JSON) or a GraphQL error (such as invalid query syntax, or a missing parameter). The `Graph.QueryError` is an `enum`, so checking the type of error is trivial:

```swift
let task = client.queryGraphWith(query) { response, error in
    if let response = response {
        // Do something
    } else {

        if let error = error, case .http(let statusCode) = error {
            print("Query failed. HTTP error code: \(statusCode)")
        }
    }
}
task.resume()
```

If the error is of type `.invalidQuery`, then an array of `Reason` objects is returned. These will provide more in-depth information about the query error. Keep in mind that these errors are not meant to be displayed to the end-user. **They are for debugging purposes only**.

The following example shows a GraphQL error response for an invalid query:

```json
{
  "errors": [
    {
      "message": "Field 'Shop' doesn't exist on type 'QueryRoot'",
      "locations": [
        {
          "line": 2,
          "column": 90
        }
      ],
      "fields": ["query CollectionsWithProducts", "Shop"]
    }
  ]
}
```

Learn more about [GraphQL errors](http://graphql.org/learn/validation/).

## Search

Some fields and connections accept search terms via the `query` parameter. For example, you can provide a `query` to search for collections that contain a specific search term in any of their fields.

The following example shows how you can find collections that contain the word "shoes":

```swift
let query = Storefront.buildQuery { $0
    .shop { $0
        .collections(first: 10, query: "shoes") { $0
            .nodes { $0
                .id()
                .title()
                .description()
            }
        }
    }
}
```

For more information on the syntax and capabilities of search queries, see [shopify.dev](https://shopify.dev/docs/api/usage/search-syntax).

## Case studies

Getting started with any SDK can be confusing. The purpose of this section is to explore all areas of the Buy SDK that might be necessary to build a custom storefront on iOS and provide a solid starting point for your own implementation.

In this section we're going to assume that you've [set up a client](#graphclient-) somewhere in your source code. Although it's possible to have multiple instances of `Graph.Client`, reusing a single instance offers many behind-the-scenes performance improvements:

```swift
let client: Graph.Client
```

### Fetch shop

Before you display products to the user, you typically need to obtain various metadata about your shop. This can be anything from a currency code to your shop's name:

```swift
let query = Storefront.buildQuery { $0
    .shop { $0
        .name()
    }
}

let task = client.queryGraphWith(query) { response, error in
    let name         = response?.shop.name
}
task.resume()
```

The corresponding GraphQL query looks like this:

```graphql
query {
  shop {
    name
  }
}
```

### Fetch collections and products

In our sample custom storefront, we want to display a collection with a preview of several products. With a conventional RESTful service, this would require one network call for collections and another network call for each collection in that array. This is often referred to as the `n + 1` problem.

The Buy SDK is built on GraphQL, which solves the `n + 1` request problem. In the following example, a single query retrieves 10 collection and 10 products for each collection with just one network request:

```swift
let query = Storefront.buildQuery { $0
    .shop { $0
        .collections(first: 10) { $0
            .edges { $0
                .node { $0
                    .id()
                    .title()
                    .products(first: 10) { $0
                        .edges { $0
                            .node { $0
                                .id()
                                .title()
                                .productType()
                                .description()
                            }
                        }
                    }
                }
            }
        }
    }
}

let task = client.queryGraphWith(query) { response, error in
    let collections  = response?.shop.collections.edges.map { $0.node }
    collections?.forEach { collection in

        let products = collection.products.edges.map { $0.node }
    }
}
task.resume()
```

The corresponding GraphQL query looks like this:

```swift
{
  shop {
    collections(first: 10) {
      edges {
        node {
          id
          title
          products(first: 10) {
            edges {
              node {
                id
                title
                productType
                description
              }
            }
          }
        }
      }
    }
  }
}
```

Since it only retrieves a small subset of properties for each resource, this GraphQL call is also much more bandwidth-efficient than it would be to fetch 100 complete resources via conventional REST.

But what if you need to get more than 10 products in each collection?

### Pagination

Although it might be convenient to assume that a single network request will suffice for loading all collections and products, that might be naive. The best practice is to paginate results. Since the Buy SDK is built on top of GraphQL, it inherits the concept of `edges` and `nodes`.

Learn more about [pagination in GraphQL](http://graphql.org/learn/pagination/).

The following example shows how you can paginate through products in a collection:

```swift
let query = Storefront.buildQuery { $0
    .node(id: collectionID) { $0
        .onCollection { $0
            .products(first: 10, after: productsCursor) { $0
                .pageInfo { $0
                    .hasNextPage()
                }
                .edges { $0
                    .cursor()
                    .node { $0
                        .id()
                        .title()
                        .productType()
                        .description()
                    }
                }
            }
        }
    }
}

let task = client.queryGraphWith(query) { response, error in
    let collection    = response?.node as? Storefront.Collection
    let productCursor = collection?.products.edges.last?.cursor
}
task.resume()
```

The corresponding GraphQL query looks like this:

```graphql
query {
  node(id: "IjoxNDg4MTc3MzEsImxhc3R") {
    ... on Collection {
      products(first: 10, after: "sdWUiOiIxNDg4MTc3M") {
        pageInfo {
          hasNextPage
        }
        edges {
          cursor
          node {
            id
            title
            productType
            description
          }
        }
      }
    }
  }
}
```

Since we know exactly what collection we want to fetch products for, we'll use the [`node` interface](#the-node-protocol-) to query the collection by `id`. You might have also noticed that we're fetching a couple of additional fields and objects: `pageInfo` and `cursor`. We can then use a `cursor` of any product edge to fetch more products `before` it or `after` it. Likewise, the `pageInfo` object provides additional metadata about whether the next page (and potentially previous page) is available or not.

### Fetch product details

In our sample app we likely want to have a detailed product page with images, variants, and descriptions. Conventionally, we'd need multiple REST calls to fetch all the required information. But with the Buy SDK, we can do it with a single query:

```swift
let query = Storefront.buildQuery { $0
    .node(id: productID) { $0
        .onProduct { $0
            .id()
            .title()
            .description()
            .images(first: 10) { $0
                .edges { $0
                    .node { $0
                        .id()
                        .src()
                    }
                }
            }
            .variants(first: 10) { $0
                .edges { $0
                    .node { $0
                        .id()
                        .price()
                        .title()
                        .available()
                    }
                }
            }
        }
    }
}

let task = client.queryGraphWith(query) { response, error in
    let product  = response?.node as? Storefront.Product
    let images   = product?.images.edges.map { $0.node }
    let variants = product?.variants.edges.map { $0.node }
}
task.resume()
```

The corresponding GraphQL query looks like this:

```graphql
{
  node(id: "9Qcm9kdWN0LzMzMj") {
    ... on Product {
      id
      title
      description
      images(first: 10) {
        edges {
          node {
            id
            src
          }
        }
      }
      variants(first: 10) {
        edges {
          node {
            id
            price
            title
            available
          }
        }
      }
    }
  }
}
```

#### Handling errors

The `Graph.Client` can return a non-nil `Graph.QueryError`. **The error and the result are not mutually exclusive.** It is valid to have both an error and a result. However, the error `case`, in this instance, is **always** `.invalidQuery(let reasons)`. You should always evaluate the `error`, make sure that you don't have an invalid query, and then evaluate the result:

```swift
let task = self.client.queryGraphWith(query) { result, error in

    if let error = error, case .invalidQuery(let reasons) = error {
        reasons.forEach {
            print("Error on \($0.line):\($0.column) - \($0.message)")
        }
    }

    if let result = result {
        // Do something with the result
    } else {
        // Handle any other errors
    }
}
task.resume()
```

**IMPORTANT:** `Graph.QueryError` does not contain user-friendly information. Often, it describes the technical reason for the failure, and shouldn't be shown to the end-user. Handling errors is most useful for debugging.

## Customer Accounts

Using the Buy SDK, you can build custom storefronts that let your customers create accounts, browse previously completed orders, and manage their information. Since most customer-related actions modify states on the server, they are performed using various `mutation` requests. Let's take a look at a few examples.

### Creating a customer

Before a customer can log in, they must first create an account. In your application, you can provide a sign-up form that runs the following `mutation` request. In this example, the `input` for the mutation is some basic customer information that will create an account on your shop.

```swift
let input = Storefront.CustomerCreateInput.create(
    email:            .value("john.smith@gmail.com"),
    password:         .value("123456"),
    firstName:        .value("John"),
    lastName:         .value("Smith"),
    acceptsMarketing: .value(true)
)

let mutation = Storefront.buildMutation { $0
    .customerCreate(input: input) { $0
        .customer { $0
            .id()
            .email()
            .firstName()
            .lastName()
        }
        .userErrors { $0
            .field()
            .message()
        }
    }
}
```

Keep in mind that this mutation returns a `Storefront.Customer` object, **not** an access token. After a successful mutation, the customer will still be required to [log in using their credentials](#customer-login-).

### Customer login

Any customer who has an account can log in to your shop. All log-in operations are `mutation` requests that exchange customer credentials for an access token. You can log in your customers using the `customerAccessTokenCreate` mutation. Keep in mind that the return access token will eventually expire. The expiry `Date` is provided by the `expiresAt` property of the returned payload.

```swift
let input = Storefront.CustomerAccessTokenCreateInput.create(
    email:    "john.smith@gmail.com",
    password: "123456"
)

let mutation = Storefront.buildMutation { $0
    .customerAccessTokenCreate(input: input) { $0
        .customerAccessToken { $0
            .accessToken()
            .expiresAt()
        }
        .userErrors { $0
            .field()
            .message()
        }
    }
}
```

Optionally, you can refresh the custom access token periodically using the `customerAccessTokenRenew` mutation.

**IMPORTANT:** It is your responsibility to securely store the customer access token. We recommend using Keychain and best practices for storing secure data.

### Password reset

Occasionally, a customer might forget their account password. The SDK provides a way for your application to reset a customer's password. A minimalistic implementation can simply call the recover mutation, at which point the customer will receive an email with instructions on how to reset their password in a web browser.

The following mutation takes a customer's email as an argument and returns `userErrors` in the payload if there are issues with the input:

```swift
let mutation = Storefront.buildMutation { $0
    .customerRecover(email: "john.smith@gmail.com") { $0
        .userErrors { $0
            .field()
            .message()
        }
    }
}
```

### Create, update, and delete address

You can create, update, and delete addresses on the customer's behalf using the appropriate `mutation`. Keep in mind that these mutations require customer authentication. Each query requires a customer access token as a parameter to perform the mutation.

The following example shows a mutation for creating an address:

```swift
let input = Storefront.MailingAddressInput.create(
    address1:  .value("80 Spadina Ave."),
    address2:  .value("Suite 400"),
    city:      .value("Toronto"),
    country:   .value("Canada"),
    firstName: .value("John"),
    lastName:  .value("Smith"),
    phone:     .value("1-123-456-7890"),
    province:  .value("ON"),
    zip:       .value("M5V 2J4")
)

let mutation = Storefront.buildMutation { $0
    .customerAddressCreate(customerAccessToken: token, address: input) { $0
        .customerAddress { $0
            .id()
            .address1()
            .address2()
        }
        .userErrors { $0
            .field()
            .message()
        }
    }
}
```

### Customer information

Up to this point, our interaction with customer information has been through `mutation` requests. At some point, we'll also need to show the customer their information. We can do this using customer `query` operations.

Just like the address mutations, customer `query` operations are authenticated and require a valid access token to execute. The following example shows how to obtain some basic customer info:

```swift
let query = Storefront.buildQuery { $0
    .customer(customerAccessToken: token) { $0
        .id()
        .firstName()
        .lastName()
        .email()
    }
}
```

#### Customer Addresses

You can obtain the addresses associated with the customer's account:

```swift
let query = Storefront.buildQuery { $0
    .customer(customerAccessToken: token) { $0
        .addresses(first: 10) { $0
            .edges { $0
                .node { $0
                    .address1()
                    .address2()
                    .city()
                    .province()
                    .country()
                }
            }
        }
    }
}
```

#### Customer Orders

You can also obtain a customer's order history:

```swift
let query = Storefront.buildQuery { $0
    .customer(customerAccessToken: token) { $0
        .orders(first: 10) { $0
            .edges { $0
                .node { $0
                    .id()
                    .orderNumber()
                    .totalPrice()
                }
            }
        }
    }
}
```

#### Customer Update

Input objects, like `Storefront.MailingAddressInput`, use `Input<T>` (where `T` is the type of value) to represent optional fields and distinguish `nil` values from `undefined` values (eg. `phone: Input<String>`).

The following example uses `Storefront.CustomerUpdateInput` to show how to update a customer's phone number:

```swift
let input = Storefront.CustomerUpdateInput(
    phone: .value("+16471234567")
)
```

In this example, you create an input object by setting the `phone` field to the new phone number that you want to update the field with. Notice that you need to pass in an `Input.value()` instead of a simple string containing the phone number.

The `Storefront.CustomerUpdateInput` object also includes other fields besides the `phone` field. These fields all default to a value of `.undefined` if you don't specify them otherwise. This means that the fields aren't serialized in the mutation, and will be omitted entirely. The result looks like this:

```graphql
mutation {
  customerUpdate(
    customer: { phone: "+16471234567" }
    customerAccessToken: "..."
  ) {
    customer {
      phone
    }
  }
}
```

This approach works well for setting a new phone number or updating an existing phone number to a new value. But what if the customer wants to remove the phone number completely? Leaving the phone number blank or sending an empty string are semantically different and won't achieve the intended result. The former approach indicates that we didn't define a value, and the latter returns an invalid phone number error. This is where the `Input<T>` is especially useful. You can use it to signal the intention to remove a phone number by specifying a `nil` value:

```swift
let input = Storefront.CustomerUpdateInput(
    phone: .value(nil)
)
```

The result is a mutation that updates a customer's phone number to `null`.

```graphql
mutation {
  customerUpdate(customer: { phone: null }, customerAccessToken: "...") {
    customer {
      phone
    }
  }
}
```

## Sample application

For help getting started, take a look at the [sample iOS app](https://github.com/Shopify/mobile-buy-sdk-ios-sample). It covers the most common use cases of the SDK and how to integrate with it. Use the sample app as a template, a starting point, or a place to cherrypick components as needed. Refer to the app's readme for more details.

## Contributions

We welcome contributions. Please follow the steps in our [contributing guidelines](.github/CONTRIBUTING.md).

## Help

For help with the Mobile Buy SDK, see the [iOS Buy SDK documentation](https://help.shopify.com/en/api/storefront-api/tools/ios-buy-sdk) or post questions on [our forum](https://ecommerce.shopify.com/c/shopify-apis-and-technology), in the `Shopify APIs & SDKs` section.

## License

The Mobile Buy SDK is provided under an [MIT License](LICENSE).
