# Introduction

This document describes how the query API is generated for a given Prisma Schema

Dependencies:

- data model
- datasources (and their connector)

The resulting query API is expressed in a format codenamed dmmf. This format is all that is needed to generate a photon client for a given language, ensuring separation between connectors and generators.

![image-20190708182058056](/Users/sorenbs/code/prisma/specs/query-capabilities/high-level-diagram.png)

As you can see, DMMF is consumed by Prisma client generators, but is generic enough to also be used by GUI applications that interact with data sources as well as code frameworks that use prisma as the data access component.

### Example - from data model to js client

The following Prisma schema has a simple data model and a a single data source using the MongoDB connector as well as a photonjs generator:

```groovy
datasource main {
  provider = "postgresql"
  url 		 = "..."
}

generator photonjs {
  provider = "photonjs"
}

model User {
  posts: Post[]
}

model Post {
  category: String
}
```

The following steps are taken in order to generate the JS Client:

1. The DMMF Generator takes the data model (`model User {...}` and `model Post{...}`) as well as the list of data sources (`datasource main {...}`) and generate the DMMF.
2. Photon.js takes this DMMF (and nothing else) and generates the code for the JS Client.

An example query supported by the generated JS Client:

```js
client.user.findMany({ where: { posts_any: { category: 'Rust' } } })
```

The exact same data model works on MongoDB:

```groovy
datasource main {
  provider = "mongodb"
  url 		 = "..."
}

generator photonjs {
  provider = "photonjs"
}

model User {
  posts: Post[]
}

model Post {
  category: String
}
```

Again, the same two steps are taken to generate the JS Client. But there is something interesting going on. The above query is no longer supported. Particularly the section wrapped in `[[[...]]]`:

```js
client.user.findMany({where: { [[[posts_any: {category: "Rust"}]]] }})
```

MongoDB is a sharded database, so queries that filter across relations can't be implemented efficiently, so at the moment the MongoDB connector does not implement this query capability.

The important takeaway is that neither the DMMF Generator nor any of the client generators know anything about the particular query capabilities of MongoDB. Instead, the MongoDB connector has a declarative description of query capabilities which the DMMF generator combines with the concrete data model to produce a full DMMF that can be consumed by any client generator.

# Components and the Interfaces between them

This section should describe the architectural components in more detail as well as the formats they exchange. Large parts of this section might be references to other specs.

# Query Capabilities

This section should describe all query capabilities in great detail. It would be ideal if we can find a way to group them.

This list should include everything listed in https://github.com/prisma/specs/issues/8