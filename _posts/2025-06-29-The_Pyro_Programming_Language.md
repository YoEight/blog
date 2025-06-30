---
layout: default
title: "Pyro: A Toy Language Rooted in Temporal Query Modeling"
date: 2025-06-29
---

# Pyro: A Toy Language Rooted in Temporal Query Modeling

Pyro began as an attempt to find a better way to describe user-defined projections in KurrentDB (formerly EventStoreDB)—specifically, the kind of temporal correlation queries that are difficult to express clearly using existing tools. That exploration led me to design a small programming language inspired by the π-calculus and built with concurrency and messaging as first-class concepts.

While Pyro isn't meant for production use, it evolved into a valuable playground. In this environment, I could model event streams, experiment with concurrency, and reason about time through code [You can view the Pyro compiler on Github].

## What is π-calculus ?

The π-calculus is a mathematical model for describing concurrent systems, where multiple processes run independently and communicate with each other. It focuses on how processes exchange messages through channels, and uniquely, it allows those channels to be created and passed around dynamically.

In essence, π-calculus helps us reason formally about systems that evolve through interaction. It's a framework for understanding how independent parts of a system can send and receive messages, change connections over time, and evolve based on communication. This makes it ideal for modeling things like distributed systems, messaging protocols, or event streams.

## Pyro key features

Pyro is a functionally oriented language that avoids mutable state. All data is currently immutable by design. It features a strong nominal type system, where types are explicitly named and checked for compatibility, helping catch many errors at compile time. Type inference is supported for non-top-level declarations, reducing boilerplate without sacrificing clarity. For flexibility during experimentation or rapid prototyping, type checking can be disabled, in which case type errors or references to undefined variables are deferred to runtime and reported as exceptions.

## Core concepts

At the core of Pyro are processes, the basic units of computation that can run concurrently. Processes interact by sending and receiving messages over channels. A sender transmits a message along a channel, while a receiver waits for a message to arrive on a given channel before continuing execution. Channels themselves can be passed as messages, allowing the communication structure of the system to evolve dynamically. Pyro also includes constructs for parallel composition, which allows multiple processes to run simultaneously, and name restriction, which creates new, private channels. Together, these concepts provide a powerful and minimal foundation for modeling complex, concurrent behaviors.

Sending and receiving messages are first-class concepts in the language, each with dedicated syntax to make communication between processes explicit and expressive.

## How does Pyro look like?

```
print ! "Hello, World!"
```

Unfamiliar syntax? Think of `print` as a process, and the `!` operator as a way to send it a message—in this case, the string `"Hello, World!"`. If you’ve used Scala with the Akka framework, this should feel familiar. It’s similar to sending a message to an Actor, where `print` plays the role of the actor in this example.

The following example shows how receiving a message looks like.

```
echo ? msg = print ! msg
```

`echo` is a process, and the `?` operator means “wait for” a message—here, bound to `msg` variable. When a message arrives, the expression after the `=` is executed. In this case, it sends the received `msg` to the `print` process. Pyro supports pattern matching, so one could have written that alternative instead:

```
echo ? [x] = print ! x
```

We expect an array containing a single element and bind that element to the x variable. Finally, we send x to the print process.

Process communicatioin aside, Pyro is very similar to Lisp and use prefix notation for functions:

```
(+ 1 2)
```

Which is the equivalent of the infix notation `1 + 2`.

You can define functions in Pyro, though they’re more like syntactic sugar for creating processes—essentially an abstraction layer. The syntax might look unfamiliar at first,
but it should still be understandable with a bit of context.

```
(def foobar [x y] = (x ! y | x ! y))
```

As a process abstraction, `foobar` can only return another process. For example, you can't return a number. In this case, `foobar` defines a process that waits for a message: specifically, an array containing exactly two elements.
This is how Pyro supports parameters. At its core, a process can only receive a single message, so if you need to pass multiple arguments, you wrap them in an array. The `|` operator
in Pyro represents parallel execution. In this snippet, we send the message `y` to the process `x` twice in parallel.

Let’s look at a more involved example. Pyro is functional by design and doesn’t include traditional looping constructs like `for` or `while`. Instead, looping behavior can be achieved by defining a process abstraction that uses recursion.

```
(def for [min: Integer max: Integer f: ![Integer ^[]] done: ^[]] =
  (def loop x : Integer =
    if (<= x max) then
      (new c : ^[]
        ( f ! [x c] | c?[] = loop ! (+ x 1)))
    else
      done ! []

  loop ! min ))
```

Admittedly, this looks a lot more intimidating than your typical for loop. Let’s break down the syntax to make it more approachable.

`min: Integer` is a standard variable declaration. Arrays in Pyro can have labels, which are optional. Labels introduce variable names bound to their corresponding values, and optionally, their types. When labeling types, `!` denotes a `Client`—a process that can only be sent messages. The `^` symbol represents a `Channel`, which means it can both send and receive messages. In Pyro, a `Client` is write-only, a `Receiver` is read-only, and a `Channel` combines both capabilities.

Now, about `f: ![Integer ^[]]`: this means `f` is a `Client` that expects a message containing an array with two elements. The first is an `Integer`, and the second is a `Channel` that handles an empty array—in other words, a communication endpoint used like a signal.

The `new` keyword creates a new `Channel` and binds it to the variable `c`. With the syntax clarified, let’s look at how the for loop is implemented. The `min` and `max` values define the bounds of the loop: the starting and ending integers for the recursion. The `f` process is invoked on each iteration. Notably, the second element expected by `f` is a `Channel` it must use to signal whether to continue the loop. To do that, `f` sends an empty array `[]` to indicate the loop should proceed.

The `done` `Channel` is used to signal the end of the iteration sequence. The loop itself is implemented through a local process named `loop`, which takes the current iteration value as its parameter—bound to the variable `x`. The logic is straightforward: if `x` is less than or equal to `max`, the loop sends both the current value and a continuation `Channel` to `f`. This `Channel` allows `f` to notify whether to continue the next iteration.

Let’s zoom in on that part:

```
if (<= x max) then
  (new c : ^[]
    ( f ! [x c] | c?[] = loop ! (+ x 1)))
```

In this branch of the code, we run two processes in parallel: `f ! [x c]` and `c?[] = loop ! (+ x 1)`. The first expression, `f ! [x c]`, sends the current iteration value `x` along with the continuation channel `c` to the `f` process. This allows `f` to perform its work and decide whether to continue the loop.

The second expression, `c?[] = loop ! (+ x 1)`, waits for a signal—specifically an empty message `[]`—from `f`. If `f` sends this signal, it means the loop should proceed, and we call loop recursively with the incremented value of `x`.

When the current value of `x` exceeds the `max` bound, we exit the loop by sending `[]` to the `done` channel, as shown here:

```
else
  done ! []
```

The `for` process starts with `loop ! min` which starts the local process `loop` with the `min` value.

This how the `for` process can be used:

```
(new done : ^[]
  ( for ! [1 4 \[x c] = (print ! x | c ! []) done]
  | done?[] = print ! "Done!")))
```

It will produce this output:

```
1
2
3
4
"Done!"
```

The complete snippet is:

```
run
  (def for [min: Integer max: Integer f:![Integer ^[]] done: ^[]] =
    (def loop x:Integer =
        if (<= x max) then
            (new c : ^[]
            ( f ! [x c]
            | c?[] = loop!(+ x 1)))
        else
            done ! []
    loop ! min )
  (new done : ^[]
    ( for! [1 4
        \[x c] = (print ! x | c ! [])
        done]
    | done?[] = print ! "Done!")))
```

# Why I built it?

It all started when I wanted to go beyond what's currently possible with KurrentDB (formerly EventStoreDB) user-defined projections. KurrentDB lets users write projections in JavaScript, which works well for a specific class of queries known as temporal correlation queries, which are common in many business systems. Temporal correlation queries are queries that relate or correlate multiple events based on their temporal (time-based) relationships. These are particularly useful in event-driven or stream processing systems where understanding when something happened and in what order is just as important as what happened. An example of temporal correlation query would be:

```
If temperature > 80°C for 10 consecutive minutes, trigger an alert.
```

In the context of KurrentDB, user-defined projections can’t dynamically change the types of events they listen to. This isn’t a limitation of KurrentDB itself, but rather of the language used to express those projections.

What I needed was a programming language where waiting for a message or an event is a first-class concept, something more expressive than simply calling a function. It's worth noting this wasn't driven by a work or business requirement. It came from a personal belief that the π-calculus paradigm captures my vision of what KurrentDB could be as a programmable data platform.

That vision also led to another experimental project: [GethDB], a database designed to embody this idea of programmability at its core. I plan to talk more about that project another time. For now, Pyro is the language that lets me explore and realize those ideas.

# Design Decisions and Trade-offs

The project is divided in four parts:

1. **pyro-core** : A library that contains core types, tokenizer, parser and type system.
1. **pyro-runtime**: A library that contains the actual runtime of Pyro.
1. **pyro-repl**: Is Read-Eval-Print-Loop or REPL program for Pyro. It uses both `pyro-core` and `pyro-runtime`. It's a simple interactive programming environment where the user inputs expressions.
1. **pyro**: Like `pyro-repl`, it uses both `pyro-core` and `pyro-runtime`. It runs Pyro programs. The difference with `pyro-repl` is it expects a complete program, not just expression.

I splited the project in smaller part because I wanted the language to be embeddable. Having `pyro-runtime` as library allows me to add different built-ins functions based on the program I want to use Pyro. For example,
when I integrated Pyro in my GethDB database, I added functions (should I say processes) that are specific to GethDB. I won't go over all the nitty gritty details but it looks like this.

```rust
pub fn create_pyro_runtime(client: SubscriptionClient, name: &String) -> eyre::Result<PyroRuntime> {
    // ...
    // Setup code that declares most of the variables that we use below.
    // ...
    let engine = Engine::with_nominal_typing()
        .stdlib(env)
        .register_type::<EventEntry>("Entry")
        .register_type::<EventRecord>("EventRecord")
        .register_value("output", ProgramOutput(send_output))
        .register_function("subscribe", move |stream_name: String| {
            // Code that actually plug code from the GethDB internal API to the Pyro plugins.
        })
        .build()?;
    // ...
}
```

### The compiler's frontend

The frontend of the compiler uses a handcrafted tokenizer and parser. Contrary to popular belief, this approach often leads to faster iteration. You can power through implementation details without getting bogged down by things like grammar ambiguities, since you usually have enough context at each step to make the right decision. Error reporting tends to be significantly better, too, because that same context allows you to produce more meaningful messages for the user. Debugging is also more straightforward; you're not dealing with opaque parser generator state machines or tangled semantic actions, but rather stepping through plain, understandable code.

Pyro has a strong nominal type system. Nominal means we use names to classify our types. The vast majority of programming languages have nominal type systems. Pyro also has constraints, which are essentially a specialized version of Rust's traits or Haskell's typeclasses. Currently, users cannot create their own constraints. The existing constraints are only used by `print`, `!`, and `?`. You can only use `print` on something that can be represented as a `String`. You can only use `!` on something that is a `Client` and you can only use `?` on something that is a `Server`. A type can have multiple constraints, but there is currently no constraint "inheritance" system. You cannot have a constraint that implies another constraint is also met. Nothing prevents this feature from being implemented; it simply does not exist yet.

For local declarations, the type system can infer the type of a value. The implementation uses a bottom-up algorithm, meaning we traverse down the abstract syntax tree and compile a list of evidence about what a variable's type might be. When reaching the top, we use this evidence to determine the most probable type. If no type can be determined, that type becomes universally quantified will all the contraints that we might find attached to it, similar to what Haskell does. The language supports universal quantification but does not allow users to define universally quantified values.

Another useful feature of Pyro's type system is that it can be completely disabled. This is helpful when you want to quickly hack something together and iterate rapidly. It represents a major user experience improvement because nobody likes fighting the type system when quickly testing ideas. When disabled, all errors are reported at runtime, including nonexistent variables or operations that are not supported by the types being used.

### The runtime

For the backend, I opted to write an interpreter for simplicity. It's a toy project, after all. While generating native code with something like LLVM would have been an exciting challenge, it would also have required a significant time investment. I even considered targeting JVM bytecode instead, since it abstracts away many of the lower-level concerns like calling conventions, register and stack management, memory layout, and threading. In the end, I chose to build an interpreter in Rust and use the Tokio library. Tokio provides built-in support for channels, complete with sender and receiver ends, which fits perfectly with Pyro’s process model. It also includes features like channel closure detection, which makes it easier to implement resource cleanup in more contrived scenarios. It’s far from production-ready, but for a toy project, it does the job well.

A Pyro interpreter main entrypoint is basically this:

```rust
impl PyroProcess {
    pub async fn run(self) -> eyre::Result<()> {
        let mut work_items = Vec::new();
        let (sender, mut mailbox) = mpsc::unbounded_channel::<Msg>();

        for tag in self.program {
            work_items.push(Suspend {
                runtime: self.runtime.clone(),
                proc: tag,
            });
        }

        let handle = tokio::spawn(async move {
            let mut gauge = 0i32;
            let mut proc_id_gen = 0u64;

            while let Some(msg) = mailbox.recv().await {
                match msg {
                    Msg::Job(out, s) => {
                        let proc_id = proc_id_gen;
                        proc_id_gen += 1;

                        tokio::spawn(async move {
                            let local_runtime = s.runtime.clone();
                            match execute_proc(s.runtime, s.proc).await {
                                Err(e) => {
                                    local_runtime
                                        .println(format!("UNEXPECTED RUNTIME ERROR: {}", e));
                                }

                                Ok(jobs) => {
                                    for next in jobs {
                                        let _ = out.send(Msg::Job(out.clone(), next));
                                    }
                                }
                            };

                            let _ = out.send(Msg::Completed(proc_id));
                        });

                        gauge += 1;
                    }

                    Msg::Completed(_proc_id) => {
                        gauge -= 1;
                        if gauge <= 0 {
                            break;
                        }
                    }
                }
            }
        });

        for job in work_items {
            let _ = sender.send(Msg::Job(sender.clone(), job));
        }

        handle.await?;
        Ok(())
    }
}
```
The interpreter runs as a manager thread that keeps track of active processes. When no processes are left, it knows it can safely exit. This is managed through a variable named `gauge`—admittedly a poor name. While this tracking approach is quite rudimentary, it turned out to be surprisingly effective given how little time it took to implement. I can’t formally prove its correctness, but in practice, I haven’t encountered any divergent behavior so far.

This is what a process is in Pyro:

```rust
#[derive(Debug, PartialEq, Eq, Clone)]
pub struct Tag<I, A> {
    pub item: I,
    pub tag: A,
}

#[derive(Debug, PartialEq, Eq, Clone)]
pub enum Proc<A> {
    Output(Tag<Val<A>, A>, Tag<Val<A>, A>),
    Input(Tag<Val<A>, A>, Tag<Abs<A>, A>),
    Null, // ()
    Parallel(Vec<Tag<Proc<A>, A>>),
    Decl(Tag<Decl<A>, A>, Box<Tag<Proc<A>, A>>),
    Cond(Tag<Val<A>, A>, Box<Tag<Proc<A>, A>>, Box<Tag<Proc<A>, A>>),
}
```

I reused the `Proc` definition from the Abstract Syntax Tree (AST). It's a generic type, which makes it versatile enough to be shared across multiple stages of the compiler, such as annotation and type checking. I realize there’s a lot to unpack here, but I don’t intend to cover every detail of the compilation pipeline. Even as a toy project, each part of the compiler could warrant its own in-depth explanation. While much of this isn’t unique to Pyro, it still serves as a gentle introduction to fundamental concepts in compiler engineering. That alone might be enough to spark someone’s curiosity and encourage them to dive deeper into how compilers work.

Now, back to `Proc`. Here’s a breakdown of what each variant represents:

- **`Output`**: Represents sending a message on a channel. The left side must be a channel that can be written to (also called a `Client`), while the right side can be any value — like `foo ! 42`.

- **`Input`**: Represents receiving a message from a channel. The left side must be a channel that can be read from (also called a `Receiver`), and the right side acts like a callback to handle the received message — for example, `foo ? x = print ! x`.

- **`Null`**: Represents an empty process that does nothing — written as `()`.

- **`Parallel`**: Represents the parallel execution operator `|`. It can be repeated for multiple processes — for instance, `a | b | c`.

- **`Decl`**: Represents a named process definition. While this is similar to a function or method in traditional programming languages, I chose not to use that terminology to avoid confusion with π-calculus concepts — for example, `(def foobar x: Integer = print ! x)`.

- **`Cond`**: Represents a conditional branch. The first part must evaluate to a boolean, and the two branches must be processes — like `if (== x y) then foo else bar`.

Let’s take a look at how processes are interpreted internally. I won’t dive too deep into the implementation details, as that would make this article overly long. The goal here is simply to introduce a few key entry points:

```rust
async fn execute_proc(
    mut runtime: Runtime,
    proc: Tag<Proc<Ann>, Ann>,
) -> eyre::Result<Vec<Suspend>> {
    let mut sus = Vec::new();

    match proc.item {
        Proc::Output(target, param) => {
            if let Some(suspend) = execute_output(&mut runtime, target, param).await? {
                sus.push(suspend);
            }
        }

        Proc::Input(source, abs) => {
            if let Some(suspend) = execute_input(&mut runtime, source, abs).await? {
                sus.push(suspend);
            }
        }
        Proc::Null => {}

        Proc::Parallel(procs) => {
            for proc in procs {
                sus.push(Suspend {
                    runtime: runtime.clone(),
                    proc,
                });
            }
        }

        Proc::Decl(decl, proc) => {
            runtime.register(decl.item);
            sus.push(Suspend {
                runtime,
                proc: *proc,
            });
        }

        Proc::Cond(test, if_proc, else_proc) => {
            let test = interpret(&mut runtime, test.item).await?;
            let test = test.bool()?;
            let proc = if test { if_proc } else { else_proc };

            sus.push(Suspend {
                runtime,
                proc: *proc,
            })
        }
    };

    Ok(sus)
}
```

The `Runtime` type maintains the current state of the interpreter, including a registry of all variables and their associated scopes. The process being executed is represented as a full tree, annotated with both source code positions and type information:

```rust
#[derive(Clone, Debug, Eq, PartialEq)]
pub struct Ann {
    pub pos: Pos,
    pub r#type: TypePointer,
}
```

`TypePointer` is a somewhat complex type, but at its core, it defines how a type expression should be evaluated, making type checking easier to implement. During interpretation, if executing a process yields a new process, we wrap it in a `Suspension`. A suspension refers to a computation

```rust
struct Suspend {
    runtime: Runtime,
    proc: Tag<Proc<Ann>, Ann>,
}
```

When produced, a suspension will be executed later on by the manager thread that I showed above.

# What I learned?

Constraining yourself helps sharpen your understanding of what truly matters in the runtime. I'm not entirely convinced that the exit condition of the runtime (when no channels remain in use) is 100% foolproof. In the worst case scenario, the runtime will simply idle. I'm confident that I understand the core features I need from a runtime perspective, so I won't waste time experimenting if I ever want to make the runtime more production-ready. Another possibility would be using LLVM to generate machine code. I'm unsure whether we can benefit enough from that level of control to justify the optimization gains, considering the cost of implementing the LLVM backend.

Building from the ground up with concurrency in mind changes the language's shape drastically. The syntax is very unfamiliar and requires a paradigm shift when thinking about functions. These are all processes, even those created through anonymous/lambda expressions. Everything revolves around message passing. It's incredible how Smalltalk was ahead of its time in this regard. In Pyro's case, it allows expressing dynamic network topologies where interconnections between components can change over time. Not every theoretical idea translates to good syntax or runtime behavior, and that tension is part of the fun.

# Final Thoughts

Pyro is not a production tool. It's a learning artifact, one that helped me internalize a complex body of knowledge through concrete experimentation. If you're curious, I encourage you to browse the code, run the interpreter, or even  fork the project and build your own toy language.

Learning is most powerful when it's hands-on, and nothing is more hands-on than building your own tools.

# Acknowledgements

Special thanks to Greg Young, the original author of the EventStore database, who suggested I look into π-calculus theory when I discussed my ideas for improving temporal query modeling.

Pyro also draws significant inspiration from the Pict programming language, which is likely the first implementation of π-calculus theory.

## References

* Benjamin C. Pierce, David N. Turner [The Pict Programming Language]
* [Pict Presentation Slides]
* [Pict Tutorial]

[You can view the Pyro compiler on Github]: https://github.com/YoEight/pyro
[GethDB]: https://github.com/YoEight/GethDB
[The Pict Programming Language]: https://www.cis.upenn.edu/~bcpierce/papers/pict/Html/Pict.html
[Pict Presentation Slides]: https://www-sop.inria.fr/mimosa/Pascal.Zimmer/mobility/pict.pdf
[Pict Tutorial]: https://www.cs.rpi.edu/academics/courses/spring04/dci/picttutorial.pdf
