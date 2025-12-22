# EventQL: A SQL-Inspired Query Language Designed for Event Sourcing

Event sourcing has become an increasingly popular architectural pattern, but querying event streams efficiently has remained a challenge. While events are append-only and immutable, finding specific events or analyzing patterns across thousands or millions of events requires thoughtful query design. This is where EventQL shines.

## The Problem with Querying Events

When working with event-sourced systems, you're dealing with fundamentally different data structures than traditional databases:

- Events have rich metadata: type, subject, timestamp, data payload
- Events are organized hierarchically through subjects (e.g., `/books/42`, `/users/123/orders/456`)
- You need to filter, aggregate, and transform streams of events
- Performance depends heavily on leveraging the right indexes

Traditional NoSQL query interfaces often fall short because they weren't designed with these event-specific characteristics in mind.

## Enter EventQL

EventQL is a query language originally designed for [EventSourcingDB](https://www.thenativeweb.io/products/eventsourcingdb) by The Native Web. What makes it special is how well it captures the essence of event querying while maintaining SQL's familiar expressiveness.

Here's a simple example:

```sql
FROM e IN events
WHERE e.type == "io.eventsourcingdb.library.book-acquired"
  AND e.data.price > 20
PROJECT INTO {
  id: e.id,
  title: e.data.title,
  price: e.data.price
}
```

If you've written SQL, this feels immediately familiar. But notice how it's tailored for events: we're filtering by event type, accessing nested data payloads, and reshaping the output.

## Why EventQL's Design Matters

### 1. **First-Class Event Properties**

EventQL treats event metadata as first-class citizens in the query language:

- `e.type` - Filter by event type
- `e.subject` - Query by subject hierarchy
- `e.id` - Reference specific events
- `e.time` - Time-based sorting and filtering
- `e.data.*` - Deep access into event payloads

Each of these properties represents an **indexing opportunity**. A well-designed event store can create indexes on types, subjects, and timestamps, making these queries blazingly fast.

### 2. **Subject Hierarchies Enable Smart Scoping**

One of EventQL's most powerful features is subject pattern matching:

```sql
FROM e IN events
WHERE e.subject == "/books/42"
ORDER BY e.time DESC
TOP 100
PROJECT INTO e
```

Subject hierarchies like `/books/42` or `/users/123/orders/456` are natural in event sourcing. They represent aggregate boundaries and allow you to:

- Scope queries to specific aggregates
- Create subject-based indexes for rapid lookups
- Build hierarchy-aware queries (though not shown in basic examples)

This makes browsing event data intuitive: "Show me all events for this book" or "What happened to this user's orders?"

### 3. **SQL-Like Expressiveness**

EventQL borrows SQL's proven patterns:

- **WHERE clauses** with full boolean logic (AND, OR, NOT)
- **ORDER BY** for sorting with ASC/DESC
- **GROUP BY** for aggregations
- **TOP/SKIP** for pagination
- **Nested subqueries** for complex transformations

This means you can express intricate queries precisely:

```sql
FROM e IN (
  FROM e IN events
  WHERE e.type == "order-placed"
  PROJECT INTO {
    orderId: e.id,
    total: e.data.total
  }
)
WHERE e.total > 100
ORDER BY e.total DESC
PROJECT INTO e
```

### 4. **Projection as a First-Class Concept**

Unlike SQL's optional SELECT, EventQL requires explicit projection with `PROJECT INTO`. This design choice makes sense for event queries where you often want to:

- Reshape nested event data
- Extract specific fields from payloads
- Build aggregates or computed values

```sql
FROM e IN events
WHERE e.type == "book-acquired"
PROJECT INTO {
  year: YEAR(e.time),
  revenue: SUM(e.data.price)
}
```

The projection syntax supports arbitrary object construction, making it easy to build exactly the output shape you need.

## Index-Friendly by Design

What I particularly appreciate about EventQL is how naturally it guides you toward indexable queries. Consider the common properties used in WHERE clauses:

- **Event type** - Almost always indexed
- **Subject** - Natural partition key
- **Timestamp** - Essential for time-range queries
- **Data fields** - Can be indexed selectively

A query like this:

```sql
FROM e IN events
WHERE e.type == "user-registered"
  AND e.time > "2025-01-01"
  AND e.subject LIKE "/users/*"
ORDER BY e.time DESC
TOP 1000
PROJECT INTO e
```

Maps beautifully to a composite index on (type, time, subject). The language's structure makes it obvious where indexes will help.

## Making the Parser Production-Ready

I originally wrote a parser for EventQL while working on my GethDB database project. It worked, but it was tightly coupled to that specific use case. Recently, I decided to make it production-ready as a standalone library.

The result is a robust Rust parser that:

- Provides detailed error messages with line and column numbers
- Builds a strongly-typed AST suitable for query optimization
- Supports the full EventQL grammar
- Includes comprehensive test coverage
- Can be embedded in any Rust-based event store

### Type Inference: Coming Soon

The GethDB version included a type inference system that I plan to port to this library as well. The type inferencer collects as much type information as possible from the query and catches inconsistencies early.

For example, it would reject queries like this:

```sql
FROM e IN events
WHERE e.data.price == "expensive"  -- price treated as string
  AND e.data.price > 100           -- price treated as number
PROJECT INTO e
```

By tracking how fields are used throughout the query, the type inferencer can rule out nonsensical queries before they ever reach the database. This provides better error messages and prevents runtime failures from type mismatches.

You can find the parser on GitHub: [eventql-parser](https://github.com/YoEight/eventql-parser)

## Why This Matters

Event sourcing is powerful, but it needs good tooling. A well-designed query language makes the difference between an event store that's painful to use and one that's a joy to work with.

EventQL gets the fundamentals right:

- It's **familiar** (SQL-like syntax)
- It's **expressive** (complex queries are possible)
- It's **event-aware** (designed around event properties)
- It's **index-friendly** (natural optimization opportunities)

If you're building an event-sourced system, you need a way to query your events effectively. EventQL shows how to do it right.

## Try It Yourself

The parser is available as a Rust crate. Here's how to get started:

```rust
use event_query_lang::parse_query;

let query = r#"
    FROM e IN events
    WHERE e.type == "order-placed"
    ORDER BY e.time DESC
    TOP 100
    PROJECT INTO e
"#;

match parse_query(query) {
    Ok(ast) => {
        // Use the AST to execute the query
        println!("Parsed successfully!");
    }
    Err(e) => {
        eprintln!("Parse error: {}", e);
    }
}
```

## Conclusion

Good language design is about understanding the domain deeply and creating abstractions that feel natural. EventQL does this beautifully for event sourcing.

It proves that you don't need to reinvent the wheel - SQL's proven patterns work wonderfully when adapted thoughtfully to event streams. The result is a query language that's both powerful and pleasant to use.

If you're working with event sourcing, give EventQL a look. And if you need a production-ready parser, the Rust implementation is ready to use.

---

*This parser was built to support my work on GethDB and other event-sourced systems. If you have feedback or want to contribute, issues and PRs are welcome on GitHub.*
