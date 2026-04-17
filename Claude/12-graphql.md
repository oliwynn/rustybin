# Prompt 12 — GraphQL Endpoint

## Context

You are working on the Rustybin project — a Rust/axum HTTP stub service. Read the existing codebase in `src/` to understand the project structure, shared types, and how routers are merged in `main.rs`.

## Goal

Implement a GraphQL endpoint with a small but realistic schema. This tests Kong's graphql-proxy-caching, graphql-rate-limiting-advanced, and degraphql plugins. The schema needs relationships and enough depth to exercise query-depth limiting.

## What to build

### File: `src/graphql.rs`

### Dependencies to add

- `async-graphql` — GraphQL server library for Rust
- `async-graphql-axum` — axum integration

### Routes

| Route | Method | Behaviour |
|---|---|---|
| `/graphql` | POST | GraphQL query endpoint (standard `{"query": "...", "variables": {...}}` body) |
| `/graphql` | GET | GraphQL Playground / IDE (HTML page for interactive testing) |
| `/graphql/schema` | GET | Return the SDL schema as text |

### Schema

Design a schema with these types and relationships:

```graphql
type Query {
    user(id: ID!): User
    users(limit: Int = 10, offset: Int = 0): [User!]!
    product(id: ID!): Product
    products(category: String, limit: Int = 10): [Product!]!
    order(id: ID!): Order
    orders(userId: ID, status: OrderStatus, limit: Int = 10): [Order!]!
}

type Mutation {
    createUser(input: CreateUserInput!): User!
    updateUser(id: ID!, input: UpdateUserInput!): User
    createOrder(input: CreateOrderInput!): Order!
}

type User {
    id: ID!
    name: String!
    email: String!
    role: UserRole!
    orders: [Order!]!            # Nested relationship — depth!
    createdAt: String!
}

type Product {
    id: ID!
    name: String!
    description: String!
    price: Float!
    category: String!
    inStock: Boolean!
    reviews: [Review!]!          # Another depth level
}

type Order {
    id: ID!
    user: User!                  # Back-reference to user
    items: [OrderItem!]!
    status: OrderStatus!
    total: Float!
    createdAt: String!
}

type OrderItem {
    product: Product!            # Deep nesting: order → items → product → reviews
    quantity: Int!
    unitPrice: Float!
}

type Review {
    id: ID!
    author: User!               # Another back-reference
    rating: Int!
    comment: String!
    createdAt: String!
}

enum UserRole { ADMIN, USER, GUEST }
enum OrderStatus { PENDING, PROCESSING, SHIPPED, DELIVERED, CANCELLED }

input CreateUserInput { name: String!, email: String!, role: UserRole }
input UpdateUserInput { name: String, email: String, role: UserRole }
input CreateOrderInput { userId: ID!, items: [OrderItemInput!]! }
input OrderItemInput { productId: ID!, quantity: Int! }
```

### Hardcoded Data

Populate with 5 users, 10 products (across 3 categories), 8 orders with items, and some reviews. Store everything in static/const data or `lazy_static`. No database needed.

The data should be interesting enough for a demo — real-looking names, products, prices, etc. Not "Test User 1".

### Resolver Behaviour

- All queries return from the hardcoded data
- Mutations "work" but store nothing — they return a plausible response based on the input (generate a new UUID for the ID, use current timestamp, etc.)
- The `orders` field on `User` filters from the orders list
- The `user` field on `Order` resolves from the users list
- This creates genuine N+1 and circular reference patterns that are realistic for gateway testing

### GraphQL Playground

For the GET handler, return the `async-graphql` built-in playground HTML, or a minimal HTML page that loads the GraphiQL or Apollo Sandbox from CDN.

### Router integration

Export `pub fn router() -> Router`. Merge into app router in `main.rs`.

## Verification

1. `cargo build` — compiles cleanly
2. `curl -X POST http://localhost/graphql -H 'Content-Type: application/json' -d '{"query": "{ users { id name email } }"}'` → returns users
3. Deep query: `{"query": "{ users { orders { items { product { reviews { author { name } } } } } } }"}` → resolves full depth
4. `curl http://localhost/graphql` in browser → shows Playground
5. `curl http://localhost/graphql/schema` → returns SDL
6. Mutation: `{"query": "mutation { createUser(input: {name: \"Test\", email: \"test@test.com\"}) { id name } }"}` → returns created user
