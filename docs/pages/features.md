# Features

> **Note**: ⛑ The following use abbreviated examples that skip a complete setup of passing Nexus type definition to Nexus' `makeSchema`. If you are new to Nexus, consider reading the [official Nexus tutorial](https://nxs.li/tutorial) before jumping into Nexus Prisma.

## Type-safe Generated Library Code

Following the same philosophy as Prisma Client, Nexus Prisma uses generation to create an API that feels tailor made for your project.

```prisma
model User {
  id  String  @id
}
```

```ts
import { User } from 'nexus-prisma'
import { objectType } from 'nexus'

objectType({
  name: User.$name
  description: User.$description
  definition(t) {
    t.field({
      type: User.id.type,
      description: User.id.description
    })
  }
})
```

## Project Enums

Every enum defined in your Prisma schema becomes importable as a Nexus enum type definition configuration. This makes it trivial to project enums from your database layer into your API layer.

```prisma
enum SomeEnum {
  foo
  bar
}
```

```ts
import { SomeEnum } from 'nexus-prisma'
import { enumType } from 'nexus'

SomeEnum.name //    'SomeEnum'
SomeEnum.members // ['foo', 'bar']

enumType(SomeEnum)
```

## Project Scalars

Like GraphQL, [Prisma has the concept of scalar types](https://www.prisma.io/docs/reference/api-reference/prisma-schema-reference/#model-field-scalar-types). Some of the Prisma scalars can be naturally mapped to standard GraphQL scalars. The mapping is as follows:

**Prisma Standard Scalar to GraphQL Standard Scalar Mapping**

| Prisma              | GraphQL                                                        |
| ------------------- | -------------------------------------------------------------- |
| `Boolean`           | `Boolean`                                                      |
| `String`            | `String`                                                       |
| `Int`               | `Int`                                                          |
| `Float`             | `Float`                                                        |
| `String` with `@id` | `ID`                                                           |
| `Int` with `@id`    | `ID` \| `Int` ([configurable](#projectidinttographql-id--int)) |

However some of the Prisma scalars do not have a natural standard representation in GraphQL. For these cases Nexus Prisma generates code that references type names matching those scalar names in Prisma. Then, you are expected to define those custom scalar types in your GraphQL API. Nexus Prisma ships with pre-defined mappings in `nexus-prisma/scalars` you _can_ use for convenience. The mapping is as follows:

**Prisma Standard-Scalar to GraphQL Custom-Scalar Mapping**

| Prisma     | GraphQL    | Nexus `t` Helper | GraphQL Scalar Implementation                                         | Additional Info                                                                                              |
| ---------- | ---------- | ---------------- | --------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| `Json`     | `Json`     | `json`           | [JsonObject](https://www.graphql-scalars.dev/docs/scalars/jsonobject) |                                                                                                              |
| `DateTime` | `DateTime` | `dateTime`       | [DateTime](https://www.graphql-scalars.dev/docs/scalars/datetime)     |                                                                                                              |
| `BigInt`   | `BigInt`   | `bigInt`         | [BigInt](https://www.graphql-scalars.dev/docs/scalars/big-int)        | [JavaScript BigInt](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/BigInt) |
| `Bytes`    | `Bytes`    | `bytes`          | [Byte](https://www.graphql-scalars.dev/docs/scalars/byte/)            | [Node.js Buffer](https://nodejs.org/api/buffer.html#buffer_buffer)                                           |
| `Decimal`  | `Decimal`  | `decimal`        | (internal)                                                            | Uses [Decimal.js](https://github.com/MikeMcl/decimal.js)                                                     |

> **Note:** Not all Prisma scalar mappings are implemented yet: `Unsupported`

> **Note:** BigInt is supported in Node.js since version [10.4.0](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/BigInt#browser_compatibility) however to support BigInt in `JSON.parse`/`JSON.stringify` you must use [`json-bigint-patch`](https://github.com/ardatan/json-bigint-patch) otherwise BigInt values will be serialized as strings.

You can use your own GraphQL Scalar Implementation, however, you _must adhere to the above Prisma/GraphQL name mapping defined above_.

Here is an example using Nexus Prisma's pre-defined GraphQL custom scalars:

```ts
import NexusPrismaScalars from 'nexus-prisma/scalars'
import { makeSchema } from 'nexus'

makeSchema({
  types: [NexusPrismaScalars],
})
```

There is a [recipe below](#Supply-custom-custom-scalars-to-your-GraphQL-schema) showing how to add your own custom scalars if you want.

## Project Relations

You can project [relations](https://www.prisma.io/docs/concepts/components/prisma-schema/relations) into your API with Nexus Prisma. Nexus Prisma even includes the resolver you'll need at runtime to fulfill the projection by automating use of your Prisma Client instance.

Please note that not all kinds of relationships are supported yet. Details about projecting each kind of relation are documented in their respective sections. This section only contains general documentation common to all.

To project relations you must by default expose an instance of Prisma Client on the GraphQL context under the key name `prisma`. You can [customize which context property Nexus Prisma should look for your Prisma Client](#prismaclientcontextfield-string).

#### Example: Exposing Prisma Client on GraphQL Context with Apollo Server

```ts
import { ApolloServer } from 'apollo-server'
import { PrismaClient } from '@prisma/client'
import schema from './your/schema/somewhere'

const prisma = new PrismaClient()

new ApolloServer({
  schema,
  context() {
    return {
      prisma,
    }
  },
})
```

## Project 1:1 Relation

You can project [1:1 relationships](https://www.prisma.io/docs/concepts/components/prisma-schema/relations#one-to-one-relations) into your API.

#### Example: Tests

The integration test suite is a useful reference as it is declarative (easy to read) and gives a known-working example spanning from database all the way to executed GraphQL document.

- [Tests](https://github.com/prisma/nexus-prisma/blob/main/tests/integration/relation1To1.test.ts)
- [Snapshots](https://github.com/prisma/nexus-prisma/blob/main/tests/integration/__snapshots__/relation1To1.test.ts.snap)

#### Example: Full 1:1

```prisma
// Database Schema

model User {
  id         String  @id
  profile    Profile @relation(fields: [profileId], references: [id])
  profileId  String
}

model Profile {
  id      String  @id
  user    User?
}
```

```ts
// API Schema

import { User, Profile } from 'nexus-prisma'

queryType({
  definition(t) {
    t.nonNull.list.nonNull.field('users', {
      type: 'User',
      resolve(_, __, ctx) {
        return ctx.prisma.user.findMany()
      },
    })
  },
})

objectType({
  name: User.$name,
  definition(t) {
    t.field(User.id)
    t.field(User.profile)
  },
})

objectType({
  name: Profile.$name,
  definition(t) {
    t.field(Profile.id)
  },
})
```

```graphql
# API Schema Represented in GraphQL SDL (this is generated by Nexus)

type Query {
  users: [User!]!
}

type User {
  id: ID
  profile: Profile
}

type Profile {
  id: ID
}
```

```ts
// Example Database Data (for following example)

await prisma.user.create({
  data: {
    id: 'user1',
    profile: {
      create: {
        id: 'profile1',
      },
    },
  },
})
```

```graphql
# Example API Client Query

query {
  users {
    id
    profile {
      id
    }
  }
}
```

```json
{
  "data": {
    "users": [
      {
        "id": "user1",
        "profile": {
          "id": "profile1"
        }
      }
    ]
  }
}
```

#### Limitation: Nullable on Without-Relation-Scalar Side

Prisma requires that a 1:1 relationship has one side that is optional. For example in the following it is **not** possible for `Profile` to have a required relationship to `User`. For more detail you can read the Prisma docs about this [here](https://www.prisma.io/docs/concepts/components/prisma-schema/relations#one-to-one-relations).

```prisma
model User {
  id         String  @id
  profile    Profile @relation(fields: [profileId], references: [id])
  profileId  String
}

model Profile {
  id      String  @id
  user    User?  // <--  "?" required
}
```

Prisma inherits this limitation from databases. In turn Nexus Prisma inherits this limitation from Prisma. For example consider this projection and then look at the resulting GraphQL SDL representation.

```ts
import { User, Profile } from 'nexus-prisma'

objectType({
  name: User.$name,
  definition(t) {
    t.field(User.id)
    t.field(User.profile)
  },
})

objectType({
  name: Profile.$name,
  definition(t) {
    t.field(Profile.id)
    t.field(User.profile)
  },
})
```

```graphql
type User {
  id: ID
  profile: Profile!
}

type Profile {
  id: ID
  user: User # <-- Nullable!
}
```

This limitation may be a problem for your API. There is an [issue track this that you can subscribe to](https://github.com/prisma/nexus-prisma/issues/34) if interested. As a workaround for now you can do this:

```ts
objectType({
  name: Profile.$name,
  definition(t) {
    t.field(Profile.id)
    t.field({
      ...User.profile,
      type: nonNull(User.profile.type),
    })
  },
})
```

## Project 1:n Relation

You can project [1:n relationships](https://www.prisma.io/docs/concepts/components/prisma-schema/relations#one-to-many-relations) into your API.

#### Example: Tests

The integration test suite is a useful reference as it is declarative (easy to read) and gives a known-working example spanning from database all the way to executed GraphQL document.

- [Tests](https://github.com/prisma/nexus-prisma/blob/main/tests/integration/relation1ToN.test.ts)
- [Snapshots](https://github.com/prisma/nexus-prisma/blob/main/tests/integration/__snapshots__/relation1ToN.test.ts.snap)

#### Example: Full 1:n

```prisma
// Database Schema

model User {
  id         String    @id
  posts      Post[]
}

model Post {
  id        String  @id
  author    User?   @relation(fields: [authorId], references: [id])
  authorId  String
}
```

```ts
// API Schema

import { User, Post } from 'nexus-prisma'

queryType({
  definition(t) {
    t.nonNull.list.nonNull.field('users', {
      type: 'User',
      resolve(_, __, ctx) {
        return ctx.prisma.user.findMany()
      },
    })
  },
})

objectType({
  name: User.$name,
  definition(t) {
    t.field(User.id)
    t.field(User.posts)
  },
})

objectType({
  name: Post.$name,
  definition(t) {
    t.field(Post.id)
  },
})
```

```graphql
# API Schema Represented in GraphQL SDL (this is generated by Nexus)

type Query {
  users: [User]
}

type User {
  id: ID!
  posts: [Post!]!
}

type Post {
  id: ID!
}
```

```ts
// Example Database Data (for following example)

await prisma.user.create({
  data: {
    id: 'user1',
    posts: {
      create: [{ id: 'post1' }, { id: 'post2' }],
    },
  },
})
```

```graphql
# Example API Client Query

query {
  users {
    id
    posts {
      id
    }
  }
}
```

```json
{
  "data": {
    "users": [
      {
        "id": "user1",
        "posts": [
          {
            "id": "post1"
          },
          {
            "id": "post2"
          }
        ]
      }
    ]
  }
}
```

## Projecting Nullability

Currently nullability projection is not configurable. This section describes how Nexus Prisma handles it.

```
                      Nexus Prisma Projects
                                │
                                │
DB Layer (Prisma)           → → ┴ → →           API Layer (GraphQL)
–––––––––––––––––                               –––––––––––––––––––

Nullable Field Relation                         Nullable Field Relation

model A {                                       type A {
  foo Foo?                                        foo: Foo
}                                               }



Non-Nullable Field Relation                     Non-Nullable Field Relation

model A {                                       type A {
  foo Foo                                         foo: Foo!
}                                               }



List Field Relation                             Non-Nullable Field Relation Within Non-Nullable List

model A {                                       type A {
  foos Foo[]                                      foo: [Foo!]!
}                                               }
```

If a `findOne` or `findUnique` for a non-nullable Prisma field return null for some reason (e.g. data corruption in the database) then the standard [GraphQL `null` propagation](https://medium.com/@calebmer/when-to-use-graphql-non-null-fields-4059337f6fc8) will kick in.

#### Prisma Client `rejectOnNotFound` Handling

Prisma Client's [`rejectOnNotFound` feature](https://www.prisma.io/docs/reference/api-reference/prisma-client-reference#rejectonnotfound) is effectively ignored by Nexus Prisma. For example if you [set `rejectOnNotFound` globally on your Prisma Client](https://www.prisma.io/docs/reference/api-reference/prisma-client-reference#enable-globally-for-findunique-and-findfirst) it will not effect Nexus Prisma when it uses Prisma Client. This is because Nexus Prisma [sets `rejectOnNotFound: false` for every `findUnique`/`findFirst`](https://www.prisma.io/docs/reference/api-reference/prisma-client-reference#remarks-2) request it sends.

The reason for this design choice is that when Nexus Prisma's logic is handling a GraphQL resolver that includes how to handle nullability issues which it has full knowledge about.

If you have a use-case for different behaviour [please open a feature request](https://github.com/prisma/nexus-prisma/issues/new?assignees=&labels=type%2Ffeat&template=10-feature.md&title=Better%20rejectOnNotFound%20Handling). Also, remember, you can always override the Nexus Prisma resolvers with your own logic ([recipe](/recipes#project-relation-with-custom-resolver-logic)).

#### Related Issues

- [`#98` Always set rejectOnNotFound to false](https://github.com/prisma/nexus-prisma/issues/98)

## Prisma Schema Docs Propagation

#### As GraphQL schema doc

```prisma
/// A user.
model User {
  /// A stable identifier to find users by.
  id  String  @id
}
```

```ts
import { User } from 'nexus-prisma'
import { objectType } from 'nexus'

User.$description // JSDoc: A user.
User.id.description // JSDoc: A stable identifier to find users by.

objectType({
  name: User.$name
  description: User.$description
  definition(t) {
    t.field(User.id)
  }
})
```

```graphql
"""
A user.
"""
type User {
  """
  A stable identifier to find users by.
  """
  id: ID
}
```

#### As JSDoc

Can be disabled in [gentime settings](https://pris.ly/nexus-prisma/docs/settings/gentime).

```prisma
/// A user.
model User {
  /// A stable identifier to find users by.
  id  String  @id
}
```

```ts
import { User } from 'nexus-prisma'

User // JSDoc: A user.
User.id // JSDoc: A stable identifier to find users by.
```

#### Rich Formatting

It is possible to write multiline documentation in your Prisma Schema file. It is also possible to write markdown or whatever else you want.

```prisma
/// # Foo   _bar_
/// qux
///
/// tot
model Foo {
  /// Foo   bar
  /// qux
  ///
  /// tot
  foo  String
}
```

However, you should understand the formatting logic Nexus Prisma uses as it may limit what you want to achieve. The current logic is:

1. Strip newlines
1. Collapse multi-spaces spaces into single-space

So the above would get extracted by Nexus Prisma as if it was written like this:

```prisma
/// # Foo _bar_ qux tot
model Foo {
  /// Foo bar qux tot
  foo  String
}
```

This formatting logic is conservative. We are open to making it less so, in order to support more expressivity. Please [open an issue](https://github.com/prisma/nexus-prisma/issues/new?assignees=&labels=type%2Ffeat&template=10-feature.md&title=Better%20extraction%20of%20Prisma%20documentation) if you have an idea.

## ESM Support

Nexus Prisma supports both [ESM](https://nodejs.org/api/esm.html) and CJS. There shouldn't be anything you need to "do", things should "just work". Here's the highlights of how it works though:

- We publish both a CJS and ESM build to npm.
- When the generator runs, it emits CJS code to the CJS build _and_ ESM code to the ESM build.
- Nexus Prisma CLI exists both in the ESM and CJS builds but its built to not matter which is used. That said, the package manifest is setup to run the CJS of the CLI and so that is what ends up being used in practice.

## Refined DX

These are finer points that aren't perhaps worth a top-level point but none the less add up toward a thoughtful developer experience.

#### JSDoc

- Generated Nexus configuration for fields and models that you _have not_ documented in your PSL get default JSDoc that teaches you how to do so.
- JSDoc for Enums have their members embedded

#### Default Runtime

When your project is in a state where the generated Nexus Prisma part is missing (new repo clone, reinstalled deps, etc.) Nexus Prisma gives you a default runtime export named `PleaseRunPrismaGenerate` and will error with a clear message.

#### Peer-Dependency Validation

When `nexus-prisma` is imported it will validate that your project has peer dependencies setup correctly.

If a peer dependency is not installed it `nexus-prisma` will log an error and then exit 1. If its version does not satify the range supported by the current version of `nexus-prisma` that you have installed, then a warning will be logged. If you want to opt-out of this validation (e.g. you're [using a bundler](/notes#disable-peer-dependency-check)) then set an envar as follows:

```
NO_PEER_DEPENDENCY_CHECK=true|1
PEER_DEPENDENCY_CHECK=false|0
```

#### Auto-Import Optimized

- `nexus-prisma/scalars` offers a default export you can easily auto-import by name: `NexusPrismaScalars`.
