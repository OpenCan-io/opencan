![Screenshot](img/sudograph.png)

# Sudograph

Sudograph is a [GraphQL](https://graphql.org/) database for the [Internet Computer](https://dfinity.org/).

It greatly simplifies [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) development by providing GraphQL queries and mutations derived directly from your [GraphQL schema](https://graphql.org/learn/schema/). All you have to do is define your schema using the [GraphQL SDL](https://www.digitalocean.com/community/tutorials/graphql-graphql-sdl).

For example, if you created the following schema:

# As an example, you might define the following types in a file called schema.graphql
<pre><code>
type BlogPost {
    id: String!
    body: String!
    created_at: Date!
    live: Boolean!
    num_words: Int!
    published_at: Date
    title: String!
    updated_at: Date!
}
</code></pre>

Then Sudograph would create the following queries and mutations for you:

<pre><code>
type Query {
  readBlogPost(input: ReadBlogPostInput!): [BlogPost!]!
}

type Mutation {
  createBlogPost(input: CreateBlogPostInput!): [BlogPost!]!
  updateBlogPost(input: UpdateBlogPostInput!): [BlogPost!]!
  deleteBlogPost(input: DeleteBlogPostInput!): [BlogPost!]!
  initBlogPost: Boolean!
}
</code></pre>

There's a lot more being generated for you to get the above to work, but you're seeing the most important parts (the queries and mutations themselves).

With the generated queries/mutations above, you could start writing code like this in any of your clients:

<pre><code>
query {
    readBlogPost(input: {
        live: {
            eq: true
        }
    }) {
        id
        body
        created_at
        live
        num_words
        published_at
        title
        updated_at
    }
}
</code></pre>

The query above will return all blog posts that are "live" (have been published).

Creating a blog post would look something like this:

<pre><code>
mutation {
    createBlogPost(input: {
        id: "0"
        body: "This is my blog post!"
        created_at: "2021-03-21T05:34:31.127Z"
        live: false
        num_words: 5
        published_at: null
        title: "My Blog Post"
        updated_at: "2021-03-21T05:34:31.127Z"
    }) {
        id
    }
}
</code></pre>

## Quick Start

Sudograph is a Rust crate, and thus (for now) you must create a Rust IC canister to use it. You should generally follow the official DFINITY guidance for the [Rust CDK](https://sdk.dfinity.org/docs/rust-guide/rust-intro.html).

If you ever want to see a concrete example of how to implement Sudograph, simply take a look at the examples directory.

Let's imagine you've created a Rust canister called `graphql` in a directory called `graphql`. In the `graphql` directory you should have a `Cargo.toml` file. You'll need to add Sudograph as a dependency. For example:

<pre><code>
[package]
name = "graphql"
version = "0.0.0"
authors = ["Jordan Last <jordan.michael.last@gmail.com>"]
edition = "2018"

[lib]
path = "src/graphql.rs"
crate-type = ["cdylib"]

[dependencies]
sudograph = "0.1.0"
</code></pre>

Next let's define our schema. In the `graphql/src` directory, let's add a file called `schema.graphql`:

<pre><code>
# graphql/src/schema.graphql
type BlogPost {
    id: String!
    body: String!
    created_at: Date!
    live: Boolean!
    num_words: Int!
    published_at: Date
    title: String!
    updated_at: Date!
}
</code></pre>

Your canister should be implemented as a Rust library crate, in this case the source code for our canister is found in `graphql/src/graphql.rs`. You only need to add two lines of code to this file to bootstrap your GraphQL database:

<pre><code>
// graphql/src/graphql.rs
use sudograph::graphql_database;

graphql_database!("canisters/graphql/src/schema.graphql");
</code></pre>

You will also need to add a [Candid](https://sdk.dfinity.org/docs/candid-guide/candid-intro.html) file to `graphql/src`. Let's call it `graphql.did`:

<pre><code>
# graphql/src/graphql.did
service : {
    "graphql_query": (text) -> (text) query;
    "graphql_mutation": (text) -> (text);
}
</code></pre>

Sudograph will automatically create two methods on your canister, the first is called `graphql_query`, which is a query method (will return quickly). The second is called `graphql_mutation`, which is an update method (will return more slowly). You should send all queries to `graphql_query` and all mutations to `graphql_mutation`. If you want the highest security guarantees, you can send all queries to `graphql_mutation`, they will simply take a few seconds to return.

If you have setup your `dfx.json` correctly, then you should be able to deploy your Sudograph canister. Open up a terminal in the root directory of your IC project and start up an IC replica with `dfx start`. Open another terminal, and from the same directory run `dfx deploy`.

You should now have a GraphQL database running inside of your `graphql` canister.

##Preset Sudograph Canister
[Sudograph](https://github.com/ALLiDoizCode/sudograph)

This canister is ready to deploy without having to write any Motoko code. It can be used as-is for quick prototyping and testing, but is also suitable for production environments.