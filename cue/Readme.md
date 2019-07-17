<!-- toc -->

- [Valid Cue Examples](#valid-cue-examples)
- [Principles of Cue](#principles-of-cue)
  - [1. "constraints as primitives" give you this unique ability to treat types as values](#1-constraints-as-primitives-give-you-this-unique-ability-to-treat-types-as-values)
  - [2. If your configuration has the following properties, then merging configuration becomes both trivial & lossless:](#2-if-your-configuration-has-the-following-properties-then-merging-configuration-becomes-both-trivial--lossless)
- [Cue Primitive types](#cue-primitive-types)

* [Real-world Example](#real-world-example)
  - [Evaluating Cue Files](#evaluating-cue-files)
  - [Exporting to JSON](#exporting-to-json)
* [How this _might_ fit into Prisma](#how-this-_might_-fit-into-prisma)
  - [Given the following schema](#given-the-following-schema)
    - [Postgres Datasource](#postgres-datasource)
    - [SQLite Datasource](#sqlite-datasource)
    - [`user.prisma`](#userprisma)
    - [`post.prisma`](#postprisma)
    - [`comment.prisma`](#commentprisma)
    - [schema.prisma](#schemaprisma)
  - [Possible Connector Integration](#possible-connector-integration)
    - [Application 1: Schema Validation](#application-1-schema-validation)
    - [Application 2: Safe merging of models with the same name](#application-2-safe-merging-of-models-with-the-same-name)
    - [Application 3: 80% of Runtime Validation is Generated](#application-3-80%25-of-runtime-validation-is-generated)
    - [Application 4: Automated Database Migration](#application-4-automated-database-migration)

<!-- tocstop -->

## Valid Cue Examples

From more specific to less specific

```cue
moscow: {
  name: "Moscow"
  pop: 11.92M
  capital: true
}
```

```cue
largeCapital: {
  name: string
  pop: >5M
  capital: true
}
```

```cue
municipality: {
  name: string
  pop: int
  capital: bool
}
```

- Constraints are used for both validation and reducing boilerplate
- Defining a schema is cumbersome, keeping them up-to-date is much worse.
- Interestingly, an existing valid JSON blob is valid cue. You can use the existing JSON as a hyper-specific cue file, then relax the cue file by adding
  constraints.
- Kubernetes generates protocol definition files from Go code (it's "code-first")

## Principles of Cue

- Cue adds constraints as a primitive. This allows them to say "types are values".
- Cue configuration is Associative, Commutative & Idempotent.

### 1. "constraints as primitives" give you this unique ability to treat types as values

```
uint      >=0
uint8     >=0 & <=255
int8      >=-128 & <=127
uint16    >=0 & <=65536
int16     >=-32_768 & <=32_767
rune      >=0 & <=0x10FFFF
uint32    >=0 & <=4_294_967_296
int32     >=-2_147_483_648 & <=2_147_483_647
uint64    >=0 & <=18_446_744_073_709_551_615
int64     >=-9_223_372_036_854_775_808 & <=9_223_372_036_854_775_807
uint128   >=0 & <=340_282_366_920_938_463_463_374_607_431_768_211_455
int128    >=-170_141_183_460_469_231_731_687_303_715_884_105_728 &
           <=170_141_183_460_469_231_731_687_303_715_884_105_727
```

### 2. If your configuration has the following properties, then merging configuration becomes both trivial & lossless:

- Associative `(A & B) & C = A & (B & C)`
- Commutative `A & B = B & A`
- Idempotent `A & A = A`

Given these properties, it seems like we could have all these databases with certain constraints and then if you want to work with them together or abstract
over them, you could take the intersection of all those configurations.

```
                                     ┌────────────┐
                                     │            │
                                     │    .cue    │
                                     │            │
                                     └────────────┘
                                            ▲
  ┌───────────┐           ┌─────────────────┼────────────────┐
  │   merge   │─ ─ ─ ─ ─ ─│─ ─ ─ ─ ─ ─ ─ ─ ─│─ ─ ─ ─ ─ ─ ─ ─ ┼ ─ ─ ─ ─ ─
  └───────────┘           │                 │                │
                    ┌───────────┐     ┌───────────┐    ┌───────────┐
                    │   .cue    │     │   .cue    │    │   .cue    │
                    └───────────┘     └───────────┘    └───────────┘
                          ▲                 ▲                ▲
┌─────────────┐           │                 │                │
│ introspect  │─ ─ ─ ─ ─ ─│─ ─ ─ ─ ─ ─ ─ ─ ─│─ ─ ─ ─ ─ ─ ─ ─ ┼ ─ ─ ─ ─ ─
└─────────────┘           │                 │                │
                    ┌───────────┐    ┌────────────┐    ┌───────────┐
                    │   MySQL   │    │  Postgres  │    │   Mongo   │
                    └───────────┘    └────────────┘    └───────────┘
```

## Cue Primitive types

```
null
bool
string
bytes
int
float
```

# Real-world Example

## Evaluating Cue Files

**scratch.cue**

```
{
  types: {
    integer: int64 & int32
    text: string & "hello"
    real: float32 & float64
    map: { one: 1, two: 2 } & { two: 2 }
    list: [1,2] | [3,4]
  }
}
```

```sh
cue eval scratch.cue
```

```
{
  types: {
      text:    string
      integer: int32
      real:    float32
      map: {
          one: 1
          two: 2
      }
      list: [1, 2] | [3, 4]
  }
}
```

**Open Question:** Map seems to do the opposite of what I was expecting here.

## Exporting to JSON

**scratch.cue**

```
positive: uint
byte: uint8
word: int32

{
  a: positive & 1
  b: string & "hi"
  c: word & 2_000_000_000
}
```

```sh
cue export scratch.cue
{
    "a": 1,
    "b": "hi",
    "c": 2000000000,
}
```

Let's add a bound:

```
positive: uint
byte:     uint8
word:     int32

{
    a: positive & 1
    b: string & "hi"
    c: word & 2_000_000_000
    d: >=1 & <=10
}
```

```sh
cue export scratch.cue
cue: marshal error at path d: cannot convert incomplete value "(>=1 & <=10)" to JSON
```

This makes sense, how can you export `>=1 <=10` to JSON? Let's look at if we take the intersection with a value outside of the range

```
positive: uint
byte:     uint8
word:     int32

{
    a: positive & 1
    b: string & "hi"
    c: word & 2_000_000_000
    d: >=1 & <=10 & 20
}
```

```sh
cue export scratch.cue
d:20 not within bound <=10
```

Pretty cool. Let's fix this

```
positive: uint
byte:     uint8
word:     int32

{
    a: positive & 1
    b: string & "hi"
    c: word & 2_000_000_000
    d: >=1 & <=10 & 5
}
```

```sh
cue export scratch.cue
{
    "a": 1,
    "b": "hi",
    "c": 2000000000,
    "d": 5
}
```

Our constraints resolve to a single set.

# How this _might_ fit into Prisma

## Given the following schema

Users are backed in Postgres, posts and comments are backed in SQLite:

### Postgres Datasource

```sql
create extension if not exists citext;

create table if not exists users (
  id integer not null primary key,
  firstName text not null,
  lastName text not null,
  active boolean not null default true,
  email citext not null
);
```

### SQLite Datasource

```sql
create table if not exists posts (
  id integer not null primary key,
  title text not null,
  age real not null default 0.0,
  draft integer not null default 1,
  user_id integer not null -- references users (id) in postgres
);

create table if not exists comments (
  id integer not null primary key,
  post_id integer not null references posts (id),
  comment text not null
);
```

### `user.prisma`

```groovy
datasource pg {
  provider = "postgres"
  url = "postgres://localhost:5432/db"
}

datasource sq {
  provider = "sqlite"
  url = "file://local.db"
}

model User {
  id Int @id
  firstName String
  lastName String
  email String @pg.Citext
  posts Post[]
}
```

### `post.prisma`

```groovy
model Post {
  id Int @id
  title String
  user User
  comments Comment[]
}
```

### `comment.prisma`

```groovy
model Comment {
  id Int @id
  comment String
  post Post
}
```

### schema.prisma

```groovy
import "./user.prisma"
import "./post.prisma"
import "./comment.prisma"
```

## Possible Connector Integration

It's important to have type system autocomplete in our IDEs via the Prisma language server. Right now both SQLite & PostgreSQL connectors will be embedded in
the Query Engine binary. We can query the binary for all the type definitions from the language server and save them in-memory.

In cue that may look like this:

**postgres.cue**

```
{
  coreTypes: {
    Int: int64
    String: string
    Boolean: true | false
    Float: float64
  },
  typeSpecs: {
    Citext: coreTypes.String
  }
}
```

**sqlite.cue**

```
{
  coreTypes: {
    Int: int32
    String: string
    Boolean: 0 | 1 // booleans are represented as 1 or 0 in SQLite
    Float: float64
  }
}
```

### Application 1: Schema Validation

Type specifications aren't ambiguous. Using the above example, we can tell the difference between the following:

- **Valid:** `type Email String @pg.citext`
- **Invalid:** `type Email Int @pg.citext`

Because we can do the following:

- **Valid:** `coreTypes.String & typeSpecs.Citext`
- **Invalid:** `coreTypes.Int & typeSpecs.Citext`

**Open Questions:** This would work for the type-specifications. It's less clear to me how this would work for attributes like `@unique`. I still like the idea
of `type Email pg.citext` and handle type resolution at the connector level with the core primitives.

### Application 2: Safe merging of models with the same name

**base.prisma**

```groovy
model User {
  id Int @id
  email String @pg.Citext
  age Int @pg.Int64
}
```

**user.prisma**

```groovy
model User {
  id Int @id
  email String
  age Int @pg.Int32
}
```

**schema.prisma**

```groovy
import "base.prisma"
import "user.prisma"
```

**schema.prisma (resolved)**

```groovy
model User {
  id Int @id
  email String @pg.Citext
  age Int @pg.Int32
}
```

### Application 3: 80% of Runtime Validation is Generated

Cue has a concept of a runtime version (called [cuego](https://github.com/cuelang/cue/blob/master/cuego/examples_scratch.go)). You can validate user input from
queries against the cue schema.

### Application 4: Automated Database Migration

By creating a connector interface based on constraints, we can support multiple definitions of what the core types are, depending on the data source. Then if
you try to move a model from one datasource to another, we can take the intersection of those models and let you know if this is a safe migration path.

```groovy
model User {
  @@db(pg)
  firstName String
  active Boolean
  age Int
}
```

to:

```groovy
model User {
  @@db(sq)
  firstName String
  active Boolean
  age Int
}
```

Can be modeled in the following way:

```
pg.User & sq.User
```

Using the above connector API, our model resolves to:

```
{
  firstName: string
  active: true | false
  age: int64
}
&
{
  firstName: string
  active: 1 | 0
  age: int32
}
```

And merges to:

```
{
  firstName: string
  active: (true | false) & (1 | 0)
  age: int64 & int32
}
```

Which cannot be evaluated because the `(true | false) & (1 | 0)` will evaluate to an empty set. We have a couple options at this point. The naive approach would
be to just fail and say we're incompatible. However, I think we have all the information we need to automatically generate a migration or test if it's possible
on the data.

We could write "casting" rules on how to cast from a "true" to an "integer". In inefficient psuedo-code:

```sql
const users = await pg.query(`select * from users`)
const newUsers = users.map(user => {
  user.active = user.active ? 1 : 0
  if (user.age > sq.Int32.MAX) {
    throw new Error("cannot migrate this user to sqlite")
  }
  return user
})
newUsers.each(user => {
  await sq.query(`insert into users (firstName, active, age) values (?, ?, ?)`, user)
})
```
