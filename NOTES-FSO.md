# 1. GraphQL Server

- GraphQL's philosophy is very different from REST. In REST every resoucr has its own address ie `/users/1`. All actions done to this resource are done via HTTP requests send to this endpoint.

- REST has limitations. For eg: We want to show the list of blogs, that were added by users who have commented on blogs our user follow.

- If the backend does not have an endpoint for this, then frontend has to send a lot of requests, and process them on the backend. This needs a lot of logic in frontend end and a lot of useless data is send from backend to frontend.

- A GraphQL server is well suited for these kind of situations.

- The main philosophy of GraphQL is that the code on the client forms a _query_ describing the data wanted and sends it to API via HTTP POST request. Unlike REST, all send to the same endpoint and their type is POST.

## 1.1 Schema and Queries

### 1.1.1 Schema

- At the heart of all GraphQL applications is a **schema** which describes

  1. The structure of the **types** of data sent between the client and the server. (The most basic _type_ is `Object` type)
  2. The relationship between different _types_.

- The GraphQL schema is defined using a human-readable **Schema Definition Language** For eg: The schema for the phonebook application is as follows:

```
type Person {
  name: String!
  phone: String
  street: String!
  city: String!
  id: ID!
}

type Query {
  personCount: Int!
  allPersons: [Person!]!
  findPerson(name: String!): Person
}
```

The above schema defines two _types_

1. `Person` is an object _type_ with 5 fields. 4/5 fields are of type `String`. 3 out of those 4 are mandatory - marked with `!`. `id` field is of type `ID`. `ID` type is a `String`, GraphQL ensures that they are mandatory.

2. The second type is `Query`. It describes what kind of queries can be made to the API. ()`Query` is the root type for unnamed operations.)

The above _Schema_ desribes 3 different types of queries - `personCount`, `allPersons` and `findPerson`. The `!` means that the returned vallue will not be `null`.

`[Person!]!` means that the query returns a list of `Person` object with the list containing no `null` values.

So, the schema describes what kind of queries can client send to server, what kind of params the query can have and what kind of data the queries return.

So, the schema describes what kind of queries can client send to server, what kind of params the query can have and what kind of data the queries return. In short, schema determines the contract between the client and the server.

### 1.1.2 Queries

The simplese of query `personCount` looks as follows

```
query {
  personCount
}
```

The response will be

```
{
  "data": {
    "personCount": 3
  }
}
```

For a query for which the return type is an Object, the query must specify which _fields_ of the object the query returns. For eg:

```
query {
  allPersons {
    name
    phone
  }
}
```

The response could be

```
{
  "data": {
    "allPersons": [
      {
        "name": "Arto Hellas",
        "phone": "040-123543"
      },
      {
        "name": "Matti Luukkainen",
        "phone": "040-432342"
      },
      {
        "name": "Venla Ruuska",
        "phone": null
      }
    ]
  }
}
```

For queries which take a param, that param needs to be specified inside round parens while the queried fields go inside the curly parens.

```
query {
  findPerson(name: "Arto Hellas") {
    phone
    city
    street
    id
  }
}
```

So, in contrast to REST, in GraphQL the query describes what kind of data it wants. In REST, the API endpoint just gives all the data that the backend is designed to give from that endpoint.

Note: Despite its name GraphQL has nothing to do with how the backend saves the data - relational or a document database or other servers which a GraphQL server can access.

## 1.2 Apollo Server

- `Apollo Server` is the leading JS GraphQL server library. It provides instances of `ApolloServer` which are used to **serve schema** and **handle requests**.

```
const server = new ApolloServer({
  typeDefs,
  resolvers,
})
```

- The Schema definition for `ApolloServer` is called `typeDef` and is a `string` describing the Schema.

- The second param is called an object called `resolvers` which describe how the _queries_ described in the Schema are resolved by the backend.

```
const resolvers = {
  Query: {
    personCount: () => persons.length,
    allPersons: () => persons,
    findPerson: (root, args) =>
      persons.find(p => p.name === args.name)
  }
}
```

- The `resolvers` defines `Query` as a field. (This is NOT GraphQL schema's `type Query`. ). The `resolvers.Query` is the implementation for `Query` defined in the schema. ( Also, in the above example `persons` has to be present somewhere in the code, Apollo Server can not magically create it for you)

- We say that every query has a resolver.

## 1.3 Apollo Studio Explorer

- When we run the dev server, we get a nice GUI "Apollo Studio Explorer" running at `4000`. Quite handy for sending queries.

## 1.4 Params of a Resolver

Now lets look at the resolver for `findPerson` ie `(root, args) => persons.find(p => p.name === args.name)`.

The first param `arg` is a placeholder which we can ignore. The second param `args` is an object which contains all the params of the query.

PS: All resolver functions are given 4 params - `root`, `args`, `context`, and `info`.

## 1.5 The default Resolver

- A GraphQL server must define resolvers for **each field of every type** in the schema.

For the following query

```
query {
  findPerson(name: "Arto Hellas") {
    phone
    city
    street
  }
}
```

How does the server know exactly which fields to return ? (By the query, duh)

Since we did not define resolvers for each field of type `Person`, Apollo has defined default resolvers for them. The following 2 piece of code are equivalent

```
const resolvers = {
  Query: {
    personCount: () => persons.length,
    allPersons: () => persons,
    findPerson: (root, args) =>
      persons.find(p => p.name === args.name)
  }
}
```

```
const resolvers = {
  Query: {
    personCount: () => persons.length,
    allPersons: () => persons,
    findPerson: (root, args) => persons.find(p => p.name === args.name)
  },
  Person: {
    name: (root) => root.name,
    phone: (root) => root.phone,
    street: (root) => root.street,
    city: (root) => root.city,
    id: (root) => root.id
  }
}
```

(Aside: `root` refers to the `Person` type returned by the previous resolver. More on this later)

The default resolver returns the value of the corresponding field of the object. The object iself can be accessed through the first param of the resolver, `root`.

We can use a custome resolver instead of the default resolver.

```Person: {
    street: (root) => 'Manhattan',
    city: (root) => 'New York',
  },
```

Adding the above would hardcode the make the `street` and `city` to always return the following.

ASIDE:

Assume we have the following query

```
query {
  findPerson(name: "Alice") {
    name
    phone
  }
}
```

Then, the execution flow is as following

1. `findPerson: (root, args) => persons.find(p => p.name === args.name)` In this function call, GraphQL executes the function and returns the matching `Person`
2. Then GraphQL uses default resolvers to resolve the relevant fields. In this second function call, `root` refers to the returned object from the first resolver.

```
Person: {
  name: (root) => root.name,
  phone: (root) => root.phone
}
```

## 1.6 Object within an Object

Lets modify the schema so that the `Person` has a field with type `Address` which in turn has fields `street` and `city`

Now we want the following query

```
query {
  findPerson(name: "Arto Hellas") {
    phone
    address {
      city
      street
    }
  }
}
```

to return

```
{
  "data": {
    "findPerson": {
      "phone": "040-123543",
      "address":  {
        "city": "Espoo",
        "street": "Tapiolankatu 5 A"
      }
    }
  }
}
```

Note that the `Person` is still saved as before in the server.

Note that contrary to `Person` type `Address` type does not have an `ID` field because they are not saved as a separate data structure.

Now we have to add `address` field in `Person` type with custom resolver as the default resolver is not enough.

## 1.7 Mutations

In GraphQL, all operations which cause a change are done with **mutations**. Mutations are described in the Schema as key of type `Mutation`.

```
type Mutation {
  addPerson(
    name: String!
    phone: String
    street: String!
    city: String!
  ): Person
}
```

In the above mutation `addPerson`, the params are the details of the person and it is defined to return the type `Person`.

Mutations also require a Resolver.

The resolver function is responsible for saving the data in the correct form and sending the data to the server in the correct form.

## 1.8 Error Handling

If we give invalid params the server responds with an error message.

An error can be handled by throwing GraphQL Error with a proper Error Code.

In GraphQL, even when an `GraphQLError` occurs, the server still returns a `200 OK` unless there is a network or server-level failure.

- GraphQL treats errors as part of response bodies, not status codes

- Strange that with field missing GraphQL is responding 400, but with invalid data GraphQL is responding 200. It is normal and expected in GraphQL for some errors to return `400` but for some to return `200`.

## 1.9 Enum

In a GraphQL Schema, an Enum defines a type that can take only a specific set of predefined values. These serves key purpose like Validation, Documentation, Readablity and TypeSafety.

## 1.10 More on Queries

In GraphQL, it is possible to combine multiple fields of type `Query` (aka separate Queries) into 1 query.

For example

```
query {
  personCount
  allPersons {
    name
  }
}
```

Combine queries can also use the same query multiple times. However, the queries should have different names.

```
query {
  havePhone: allPersons(phone: YES){
    name
  }
  phoneless: allPersons(phone: NO){
    name
  }
}
```

# 2. React and GraphQL

- Its a common practise to send GraphQL requests to `/graphql` instead of `/`.

In practise we can use `axios` or `fetch` to send GraphQL requests, but in practise we use a higher order library to abstract away unnecessary details of the communication.

**Apollo Client** is the most popular of the two.

## 2.1 Apollo Client

We use the following to create a `ApolloClient`

```
const client = new ApolloClient({
  uri: 'http://localhost:4000',
  cache: new InMemoryCache(),
})
```

We use the following to send a query

```
const query = gql`
  query {
    allPersons  {
      name,
      phone,
      address {
        street,
        city
      }
      id
    }
  }
`

client.query({ query })
  .then((response) => {
    console.log(response.data)
  })
```

The application can communicate with a GraphQL server using the `client` object. The `client` can be made accessible to all component by wrapping the `App` within an `ApolloProvider`.

## 2.2 Making Queries

The most popular way to make query is by `useQuery` hook. It takes the actual query string as argument and returns the result.

## 2.3 Named Queries and Variables

We run the query `findPerson` to fetch details by name as follows

```
const result = useQuery(FIND_PERSON, {
  variables: { nameToSearch },
  skip: !nameToSearch,
})
```

`skip` determines whether the query is to be run or not.
`variables: { nameToSearch }, provides the params to the query

## 2.4 Cache

Apollo Client saves the result of query in a cache automatically.

## 2.5 Doing Mutations

`useMutation` hook is used for this.

We define the function by using the hook `const [ createPerson ] = useMutation(CREATE_PERSON)`, and then call it within the event handler.

Apollo Client does not automatically update the cache after causing the mutation.

## 2.6 Updating the Cache

- One way is short-polling. It causes needless web traffic.

- Other is defining the mutation as following

```
const [ createPerson ] = useMutation(CREATE_PERSON, {
    refetchQueries: [ { query: ALL_PERSONS } ]
})
```

Now there is just one more request to get the new data from the server. However changes does not show to other users at the same time.

One way to update cache is by defining a suitable callback function. Old Cache can cause hard to find problems.

## 2.7 Apollo Client and the Application State

Management of the App state becomes mostly the responsibility of the Apollo client.

# 5. Fragments And Subscriptions

# 5.1 Fragments

It is pretty common in GraphQL that multiple queries return similar results. For example, the following 2 queries have a lot of common fields

```
query {
  findPerson(name: "Pekka Mikkola") {
    name
    phone
    address{
      street 
      city
    }
  }
}
```
```
query {
  allPersons {
    name
    phone
    address{
      street 
      city
    }
  }
}
```

For this we define a **fragment** and use the query in a compact form.
For eg:
```
fragment PersonDetails on Person {
  name
  phone 
  address {
    street 
    city
  }
}

query {
  allPersons {
    ...PersonDetails
  }
}

query {
  findPerson(name: "Pekka Mikkola") {
    ...PersonDetails
  }
}
```
Fragments ARE NOT DEFINED on the server side.

# 5.2 Subscriptions

Along with `Query` and `Mutation` type, GraphQL offers a third operation type - `Subscription`. 

With `Subscription` client can subscribe to changes in the server. When a change occurs on the server, the server sends a notification using **WebSocket** to all of its subscribers.