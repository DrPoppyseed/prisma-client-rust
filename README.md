# Prisma Client Rust

<a href="https://github.com/Brendonovich/prisma-client-rust">
    <img src="https://img.shields.io/static/v1?label=lib&message=v0.3.0&color=blue&logo=github&style=flat-square" alt="Latest crate version is 0.3.0">
</a>
<a href="https://crates.io/crates/prisma-client-rust-cli">
    <img src="https://img.shields.io/static/v1?label=cli&message=v0.3.0&color=blue&logo=rust&style=flat-square" alt="Latest crate version is 0.3.0">
</a>
<a href="https://prisma.io">
    <img src="https://img.shields.io/static/v1?label=prisma&message=v3.10.0&color=blue&logo=prisma&style=flat-square" alt="Latest supported Prisma version is 3.10.0">
</a>

An ergonomic and autogenerated database client that uses [Prisma](https://prisma.io).

## Development Warning

Prisma Client Rust is still under active development and is subject to change suddenly. Breaking features that are yet to come include:

- Error handling: Queries do not return `Result` types, but will in a future version.
- Required fields type inference: `create` currently requires using field specifiers for all fields, but in future this will only be required for optional fields.

## Installation

1. [Create a Prisma schema](https://www.prisma.io/docs/concepts/components/prisma-client) and [migrate your database](https://www.prisma.io/docs/concepts/components/prisma-migrate)
2. Install the `prisma-client-rust` CLI
   ```
   $ cargo install prisma-client-rust-cli
   ```
3. Add `prisma-client-rust` and `serde` as dependencies in `Cargo.toml`

   ```toml
   prisma-client-rust = { git = "https://github.com/Brendonovich/prisma-client-rust", tag = "0.3.0" }
   serde = { version = "1.0", features = ["derive"] }
   ```

   **Note**: `prisma-client-rust` cannot be installed from crates.io as it uses code from [Prisma's engines](https://github.com/prisma/prisma-engines). This means that your binary or library also cannot be published to crates.io. If you would like to do so, please make an issue and I will consider publishing a separate version that is less performant, but is crates.io compatible.

4. Add `prisma-client-rust` as a generator to your Prisma schema
   ```prisma
   generator client {
       provider = "prisma-client-rust"
       // The file that the client should be generated to, relative to your schema file
       output = "../src/db.rs"
   }
   ```
5. Generate the rust module
    ```
    $ prisma-client-rust generate
    ```
6. Include the generated module in your code and connect a new Prisma client

   ```rs
   // Name of the module will be the file specified in the generator's 'output'
   pub mod db;

   use db::PrismaClient;

   // Any async runtime can be used, tokio is just an example
   #[tokio::main]
   async fn main() {
       let client = PrismaClient::new().await;
   }
   ```

## Queries

The following samples use this schema:

```prisma
datasource db {
    provider = "postgresql"
    url      = env("DATABASE_URL")
}

generator client {
    provider      = "prisma-client-rust"
    output        = "../src/db.rs"
}

model User {
    username    String @id
    displayName String

    posts           Post[]
    comments        Comment[]
}

model Post {
    id      String @id
    content String

    comments Comment[] @relation()

    User         User   @relation(fields: [userUsername], references: [username])
    userUsername String
}

model Comment {
    id String @id

    postId String
    post   Post   @relation(fields: [postId], references: [id])

    User         User?   @relation(fields: [userUsername], references: [username])
    userUsername String?
}
```

### Create

```rust
pub mod db::{PrismaClient, User, Post};

#[tokio::main]
async fn main() {
    let client = PrismaClient::new().await;

    // Required values are passed in as separate arguments
    let user = client
        .user()
        .create(
            User::username().set("user0".to_string()),
            User::display_name().set("User 0".to_string()),
            // Optional arguments can be added in a Vec as the last parameter
            vec![],
        )
        .exec()
        .await;

    let post = client
        .post()
        .create(
            Post::id().set("0".to_string()),
            Post::content().set("Some post content".to_string()),
            // Relations can be linked by specifying where queries
            Post::user().link(User::username().equals(user.username.to_string())),
            vec![],
        )
        .exec()
        .await;
}
```

### Find

```rust
pub mod db::{PrismaClient, Post, User};

#[tokio::main]
async fn main() {
    let client = PrismaClient::new().await;

    // Find a single record using a unique field
    let unique_user = client
        .user()
        .find_unique(User::username().equals("some user".to_string()))
        .exec()
        .await;

    // Find the first record that matches some parameters
    let first_user = client
        .user()
        .find_first(vec![User::username().contains("user".to_string())])
        .exec()
        .await;

    // Find all records that match some parameters
    let many_users = client
        .user()
        .find_many(vec![
            // Querying on relations is also supported
            User::posts().some(vec![
                Post::content().contains("content".to_string())
            ]),
        ])
        .exec()
        .await;
}
```

### Delete

```rust
pub mod db::{PrismaClient, Post};

#[tokio::main]
async fn main() {
    let client = PrismaClient::new().await;

    // Delete a single record matching a given condition
    // Also works with find_unique
    client
        .post()
        .find_unique(Post::id().equals("0".to_string()))
        .delete()
        .exec()
        .await;

    // Delete many records matching given conditions
    // In this case, deletes every user
    client
        .user()
        .find_many(vec![])
        .delete()
        .exec()
        .await;
}
```

## Acknowledgements
- [steebchen](https://github.com/steebchen) and all other contributors to [Prisma Client Go](https://github.com/prisma/prisma-client-go) for writing a lot of code that Prisma Client Rust basically copies
- [seunlanlege](https://github.com/seunlanlege) for their work on [prisma-client-rs](https://github.com/polytope-labs/prisma-client-rs) which was used while integrating Prisma's query engine crates
- [Prisma](https://prisma.io) for developing a brilliant and flexible open source ORM
