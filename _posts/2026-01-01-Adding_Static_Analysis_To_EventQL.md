---
layout: default
title: "Adding Static Analysis to EventQL: Type Safety for Event Queries"
date: 2026-01-01
---

# Adding Static Analysis to EventQL: Type Safety for Event Queries

## Introduction

When building query languages, there's a fundamental tradeoff between catching errors early (at parse time) versus catching them late (at runtime). Runtime errors are costly. They waste resources, produce confusing error messages, and can lead to incorrect results being silently propagated through your system.

This article explores how I added comprehensive static analysis to EventQL. By implementing type checking and variable scoping analysis, the system can now catch entire classes of errors before a query ever executes.

## The Problem: Runtime Type Errors

Before static analysis, EventQL queries could fail in subtle ways:

```sql
FROM e IN events
WHERE e.data.foo == "foobar" AND e.data.foo == 42 AND f.type == "foobar"
PROJECT INTO e
```

This query has multiple problems:
1. `e.data.foo` is expected to be a `String` and a `Number` at the same time
2. The variable `f` is never declared
3. Without static analysis, there's no way to know until runtime whether `e.data.foo` exists or is a number

These errors would only surface when the query runs against actual data, potentially after significant processing has already occurred.

## The Solution: Compile-Time Type Checking

The static analysis implementation validates queries in multiple dimensions:

### 1. Type Checking

Every expression is validated against an expected type. The type checker ensures:
- Arithmetic operations use numbers: `price + 10` ✓, `price + "text"` ✗
- Boolean operators use booleans: `active AND verified` ✓, `"yes" AND "no"` ✗
- Comparisons use compatible types: `age > 18` ✓, `age > "adult"` ✗

### 2. Variable Scoping

All variable references must be properly declared:

```sql
-- ✗ Error: variable 'e' is undeclared
FROM f in events
WHERE e.id == "10"
PROJECT INTO e

-- ✓ Correct: 'e' is bound in FROM clause
FROM e in events
WHERE e.id == "10"
PROJECT INTO e
```

The analyzer tracks which variables are in scope and prevents:
- References to undeclared variables
- Duplicate bindings in the same scope
- Invalid shadowing

### 3. Field Access Validation

Record field accesses are validated against type information:

```sql
-- ✓ 'id' is a known field in CloudEvents
FROM e in events
WHERE e.id == "10"
PROJECT INTO e

-- ✗ 'unknownField' doesn't exist in the event type
FROM e in events
WHERE e.unknownField == 3
PROJECT INTO e
```

### 4. Function Call Validation

Function calls are checked for correct arity and argument types:

```sql
-- ✓ ROUND takes a number, returns a number
FROM e in events
PROJECT INTO { id: e.id, price: ROUND(e.data.price) }

-- ✗ ROUND expects a number, not a string
FROM e in events
PROJECT INTO { id: e.id, price: ROUND(e.source) }
```

## Implementation Architecture

### The Type System

At the core of the static analysis is a type system that represents the types of EventQL expressions:

```rust
pub enum Type {
    Unspecified,
    Number,
    String,
    Bool,
    Subject,
    Array(Vec<Type>),
    Record(BTreeMap<String, Type>),
    App { args: Vec<Type>, result: Box<Type> },
}
```

The `Unspecified` type is particularly important. It allows gradual typing for dynamic JSON data in the `data` field, where the schema may not be known at parse time.

### Marker Types for Type Safety

The implementation uses Rust's type pattern to distinguish analyzed queries from raw queries at the type level:

```rust
pub struct Query<A> {
    pub sources: Vec<Source<A>>,
    pub projection: Expr,
    // ... other fields
    pub meta: A,
}

// Raw query (parsed but not analyzed)
let raw_query: Query<Raw> = parse(input)?;

// Typed query (passed static analysis)
let typed_query: Query<Typed> = raw_query.run_static_analysis(&options)?;
```

This gives **compile-time guarantees** that an unvalidated query cannot be accidentally executed.

### The Analysis Algorithm

The analysis algorithm performs a single pass over the query AST, using bidirectional type checking:

1. **Expected Type Propagation**: The context provides an expected type
2. **Type Inference**: The expression is analyzed to determine its actual type
3. **Type Unification**: Expected and actual types are unified

For example, in `WHERE age > 18`:
- Context expects `Bool` (WHERE clauses must be boolean)
- The `>` operator is analyzed and infers `Bool` as its result type
- The operands `age` and `18` are checked against `Number` (comparison operands)

### Scope Management

Variable scoping is handled through a stack of scope frames:

```rust
struct Analysis<'a> {
    options: &'a AnalysisOptions,
    prev_scopes: Vec<Scope>,
    scope: Scope,
}
```

When entering a subquery, the analyzer pushes a new scope frame. When exiting, it pops the frame. This naturally handles nested scopes and subqueries:

```sql
SELECT e.data
FROM events AS e
WHERE e.id IN (
    SELECT inner.id FROM events AS inner WHERE inner.type = 'user-created'
)

FROM e IN (
  -- New scope: only 'f' is visible here
  FROM f IN events
  WHERE f.type == "io.eventsourcingdb.library.book-acquired"
  PROJECT INTO { orderId: f.data.foobar, value: f.data.total }
)
WHERE e.value > 100
PROJECT INTO e
```

### Handling Dynamic Data

Events can contain arbitrary JSON in their `data` field. The implementation handles this with special rules:

1. The `data` field starts with type `Unspecified`
2. When accessed, it becomes `Record` with `Unspecified` fields
3. Field accesses in dynamic contexts create fields on-demand
4. Type information propagates as the query is analyzed

This allows queries like:

```sql
FROM e IN events
WHERE e.data.foo == "foobar" AND e.data.baz == 42
PROJECT INTO e
```

Even though the schema of `data` is not known upfront, the type checker builds it incrementally and ensures consistency.

## Built-in Functions

The default scope includes type signatures for 30+ built-in functions:

**Math**: `ABS`, `CEIL`, `FLOOR`, `ROUND`, `COS`, `EXP`, `POW`, `SQRT`, `RAND`, `PI`

**String**: `LOWER`, `UPPER`, `TRIM`, `LTRIM`, `RTRIM`, `LEN`, `INSTR`, `SUBSTRING`, `REPLACE`, `STARTSWITH`, `ENDSWITH`

**Date/Time**: `NOW`, `YEAR`, `MONTH`, `DAY`, `HOUR`, `MINUTE`, `SECOND`, `WEEKDAY`

**Aggregates**: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `MEDIAN`, `STDDEV`, `VARIANCE`, `UNIQUE`

Each function has a precise type signature. For example:

```rust
Type::App {
    args: vec![Type::String, Type::String, Type::String],
    result: Box::new(Type::String),
}
```

This represents a function taking three strings and returning a string (like `REPLACE`).

## Error Reporting

When type checking fails, the system provides precise error messages with line and column information:

```rust
pub enum AnalysisError {
    TypeMismatch(u32, u32, Type, Type),
    VariableUndeclared(u32, u32, String),
    FieldUndeclared(u32, u32, String),
    // ... and more
}
```

## Usage Example

Here's how to use static analysis in practice:

```rust
use eventql_parser::{parse_query, AnalysisOptions};

// Parse the query
let query = parse("FROM e in events PROJECT INTO e")?;

// Run static analysis
let typed_query = query.run_static_analysis(&AnalysisOptions::default())?;

// typed_query is now guaranteed to be type-safe
// You can access type information:
println!("Projection type: {:?}", typed_query.meta.project);
println!("Variables in scope: {:?}", typed_query.meta.scope);
```

## Benefits

The static analysis system provides several key benefits:

### 1. Early Error Detection
Catch type errors, undefined variables, and invalid field accesses at parse time, not runtime.

### 2. Better Error Messages
Precise line/column information and clear explanations of what went wrong.

### 3. Type Safety Guarantees
The type system ensures that operations are only performed on compatible types.

### 4. Compile-Time Verification
Rust's type system enforces that queries must be analyzed before execution.

### 5. Documentation
Type information serves as documentation for what a query expects and produces.

### 6. IDE Support
Type information enables autocomplete, hover tooltips, and other IDE features.

## Future Enhancements

Potential future improvements:

1. **Type Inference for Subqueries**: Propagate more type information between subqueries
2. **Schema Registry Integration**: Load event schemas from a registry
3. **Custom Type Definitions**: Allow users to define custom types

## Conclusion

Adding static analysis to EventQL transforms it from a runtime-validated query language to a compile-time-safe one. By catching errors early, providing precise error messages, and leveraging Rust's type system, I've made EventQL queries more reliable and easier to work with.

The implementation demonstrates several important techniques:
- Bidirectional type checking for inference
- Marker types for compile-time guarantees
- Gradual typing for dynamic data
- Scope management for variable tracking

These patterns are applicable to any language implementation and show how type systems can provide safety without sacrificing flexibility.

The full implementation is available on [GitHub], and contributions and feedback from the community are welcome.

[GitHub]: https://github.com/YoEight/eventql-parser
