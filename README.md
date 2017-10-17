# Specification

This document aims to describe all possible interaction patterns between pull-stream modules (the pull-stream protocol). This may serve a module designer to ensure the module behaves correctly in all cases. In turn that will ensure users of pull-stream modules that any combination of them will work correctly in all cases.

This document is intentionally terse and aims for precision and clarity rather than accessibility. If you are looking for a general and quick introduction to pull-streams, look instead at [pull-streams by example](https://github.com/dominictarr/pull-stream-examples). If you want to learn how to design pull-stream modules, look at the [tutorials](https://github.com/pull-stream/pull-stream-workshop).

The presentation is separated in layers, following the [kernel language approach] (https://www.sics.se/~seif/Publications/fdp.pdf). This allows designers to quickly create modules that do not need the full capabilities of the protocol (however modules that are aimed for public usage should follow the full specification). This also eases reasoning about correctness by doing it in steps.

# Terminology

Different pull-stream documents may use different terms to describe the same things. We provide here a table of all terms used in this specification and other synonyms that are used in other documents. The terms and their relationships are illustrated in the following figure:

![Image](./pull-stream.png)

| Term                 | Definition                                                                                                                              | Synonyms                                     |
| :------------------- | :-----------------------------------------------------------------------------------------------                                        | :------------------------------------------- |
| Answer               | Event from a module that happens after a request and goes to the module immediately downstream.                                         | Callback                                     |
| Downstream           | The module(s) in a pipeline that come(s) next when following the flow of values.                                                        |                                              |
| Completed Stream     | Stream that cannot be extended. Stays completed forever. Corresponds to the sequence ````value(V1) value(V2) ... value(Vn) done````.    |                                              |
| Failed Stream        | Stream in which an error occurred. Stays failed forever. Corresponds to the sequence ````value(V1) value(V2) ... value(Vn) error(E)````.|                                              |
| Indication           | Event coming from a module to the module immediately upstream. It expects no answer.                                                    | Read with no callback                        |
| Lazy Generation      | Generation of values only after an explicit requested.                                                                                  |                                              |
| Module               | Element of a pipeline, may be a source, a sink, or a transformer.                                                                       |                                              |
| Partial Stream       | Stream that can be extended by more values in the future. Corresponds to the sequence ````value(V1) value(V2) ... value(Vn)````.      |                                              |
| Pipeline             | Composition of a single source, zero or multiple transformers, and a single sink.                                                       |                                              |
| Primitive Sink       | Elementary sink.                                                                                                                        |                                              |
| Primitive Source     | Elementary source.                                                                                                                      |                                              |
| Request              | Event coming from a module to the module immediately upstream. It expects an answer.                                                    | Read                                         |
| Sink                 | Module that consumes values, may be composed of a sink and zero or more transformer(s).                                                 | Consumer, Reader                             |
| Source               | Module that produces values, may be composed of a source and zero or more transformer(s).                                               | Producer, Readable                           |
| Stream               | Series of values produced by a module. May be partial, complete or failed.                                                              |                                              |
| Transformer          | Module that consumes and produces values, may be composed of multiple transformers.                                                     | Through                                      |
| Upstream             | The module(s) in a pipeline that come(s) before when following the flow of values.                                                      |                                              |

# Module Composition

| Composition                   | Result                 |
| :---------------------------- | :--------------------- |
| Source + Transformer          | Source                 |
| Transformer + Transformer     | Transformer            |
| Transformer + Sink            | Sink                   |
| Source (+ Transformer) + Sink | Pipeline               |

## Properties

* Composition is declarative: users of the modules do not need to care about the pull-stream protocol to compose modules.

# Format for a Sequence of Interaction Events

````
Prefix: Event
````

| Prefixes   | Meaning                      |
| :--------- | :--------------------------- |
| D: Event   | Downstream event.            |
| U: Event   | Upstream event.              |
| TI: Event  | Transformer input event.     |
| TO: Event  | Transformer output event.    |

A basic pairwise interaction corresponds to a flow of values from an upstream module to a downstream module (````U=>D````). A transformer interaction corresponds to a flow of values from an upstream module to transformer input then from the same transformer output to a downstream module (````U=>(TI TO)=>D````). Since the requests propagate in the opposite direction as the values (from downstream to upstream), requests will first happen downstream and propagate upstream, then answers will propagate from upstream towards downstream.

Event A always happens before event B is noted ````A->B````. Events may be written on different lines to denote all possible concurrent executions. A happens B may be written:


    A -> ...
     \-> B
     
or:

      A  -\
    ... -> B

# (1) Base Protocol

The base protocol enables values to flow between modules in a pipeline and completion of the stream by the source when there are no more values.

## Possible Events

| Downstream Requests | Meaning                                                          | Callback Implementation  |
| :------------------ | :--------------------------------------------------------------- | :----------------------- |
| ask(Ans)            | Requests a value and expects the answer in Ans.                  | ````read(false, ans)```` |

| Upstream Answers    | Meaning                                                          | Callback Implementation  |
| :------------------ | :--------------------------------------------------------------- | :----------------------- |
| Ans=value(V)        | Provides the value V.                                            | ````ans(false, v)````    |
| Ans=done            | Indicates that the stream has completed.                         | ````ans(true)````        |

## Properties

* An answer always happens after a request;
* Modules downstream and upstream may regulate the flow rate of the pipeline by respectively delaying their requests or answers;
* Values are generated lazily by the source. The source may therefore be used to represent an infinite stream;
* A module may make multiple requests before obtaining the first answer;
* Answers are always provided in the same order as their corresponding requests;
* Monotonic: either the stream is partial, and can be extended by a value, or complete and stays complete forever. Values cannot be updated after having been produced and new values must not be produced after the `done` event happened.

## Possible Interactions

Request on an empty stream:


    D: ask(Ans) -> U: Ans=done


Sequential requests on a non-empty stream:

    D: ask1(Ans1) -> U: Ans1=value(V1) -> D: ask2(Ans2) -> U: Ans2=done


Concurrent requests on an empty stream:

    D: ask1(Ans1)  -> U: Ans1=done   -\
                  \-> D: ask2(Ans2) -> U: Ans2=done


Concurrent requests for a non-empty stream:

    D: ask1(Ans1)  -> U: Ans1=value(V1)  -\
                  \-> D: ask2(Ans2)     -> U: Ans2=done


### Transformer Interactions

Request on an empty stream:


    D: ask1(Ans1) -> TI: askT1(AnsT1) -> U: AnsT1=done -> TO: Ans1=done
    
Sequential requests on a non-empty stream:

    D: ask1(Ans1) -> TI: askT1(AnsT1) -> U: AnsT1=value(V1) -> TO: Ans1=done -> D: ask2(Ans2) -> TI: askT2(AnsT2) -> U: AnsT2=done -> TO: Ans2=done

Concurrent requests on an empty stream:

    D: ask1(Ans1) -> TI: askT1(AnsT1) -> U: AnsT1=done -\ -> TO: Ans1=done -\
                 \-> D: ask2(Ans2) -> TI: askT2(AnsT2)    ->  U: AnsT2=done          -> TO: Ans2=done


Concurrent requests on a non-empty stream:

    D: ask1(Ans1) -> TI: askT1(AnsT1) -> U: AnsT1=value(V1) -\ -> TO: Ans1=value(V1') -\
                 \-> D: ask2(Ans2) -> TI: askT2(AnsT2)    ->  U: AnsT2=done          -> TO: Ans2=done



# (2) Abortable Protocol: (1) + Early Aborting

## Properties

## Interactions

# (3) Fault-handling Protocol: (1) + (2) + Error Handling

## Properties

## Interactions

# Limitations

1. The protocol does not easily allow parallel processing between the different modules of a pipeline. To do so, the sink needs to make as many concurrent requests as there are modules in the pipeline, it therefore needs to know how many there are. A source-driven pipeline would not require knowledge of the number of downstream modules (but would loose laziness).

# Callback Implementation

# Dataflow Implementation

