# Specification

**Superseded by https://arxiv.org/abs/1801.06144, the rest of the document is left here for historical purposes.**

This document aims to describe all possible interaction patterns between pull-stream modules (the pull-stream protocol). This may serve a module designer to ensure the module behaves correctly in all cases. In turn that will ensure users of pull-stream modules that any combination of them will work correctly in all cases.

This document is intentionally terse and aims for precision and clarity rather than accessibility. If you are looking for a general and quick introduction to pull-streams, look instead at [pull-streams by example](https://github.com/dominictarr/pull-stream-examples). If you want to learn how to design pull-stream modules, look at the [tutorials](https://github.com/pull-stream/pull-stream-workshop).

The presentation is separated in layers, following the [kernel language approach] (https://www.sics.se/~seif/Publications/fdp.pdf). This allows designers to quickly create modules that do not need the full capabilities of the protocol (however modules that are aimed for public usage should follow the full specification). This also eases reasoning about correctness by doing it in steps. We first explain what a stream is, the terminology and notation we use in the rest of the document and two possible implementations of pull-stream, one based on callbacks as the original design, and another equivalent implementation based on dataflow variables.

# Streams

Streams are sequences of values that follow one another in time. At any point in time a stream can either (1) be extended with another value, or (2) be completed (terminated) and never be extended again. A stream may be completed because there were a finite number of values, because not all values were needed, or because an error happened during processing. The past values of a stream may never be replaced with different values and the sequence may never be modified to omit some values or add new ones.

When describing a stream we separate its different values with the marker ````,````.  We therefore write ````A,B```` to mean that 'A is followed by B' in time.

A stream may be in one of three states: *partial*, *completed*, or *failed*. We illustrate the three cases with examples using non-zero positive numbers.  

A *partial stream* is a stream that may still be extended with more values and is terminated by the special ````_```` marker. A newly created stream is a partial stream with no values:

    _

A partial stream may also represent some values that were already generated. For example, the first 5 non-zero positive numbers is written:

    1, 2, 3, 4, 5, _
    
A *completed stream* is a stream that will never be extended again and is terminated by the special ````done```` marker. For example, a completed stream that contains the first 5 non-zero positive numbers is written:

     1, 2, 3, 4, 5, done
     
A *failed stream* is a stream whose processing failed at some point and is terminated by the special ````error```` marker that contains an optional error message. The failure may have happened for different reasons such as a network failure, a resource limitation, or a user action. For example, a stream that failed after the first 5 non-zero positive numbers is written:

      1, 2, 3, 4, 5, error('Network error')
      
In the previous examples, the values that make up the stream (ex: 1) are represented using their literal representation as numbers. However, maybe some arbitrary stream might use ````done```` as a value in addition to being used as a termination marker. To remove ambiguity, every value is written by being wrapped with the ````value(V)```` marker. Therefore the numeric value of ````1```` is represented as ````value(1)```` in the rest of the specification. The previous failed stream example is therefore represented as the following sequence:

    value(1), value(2), value(3), value(4), value(5), error('Network error')
           
Stream values are generated by a producer and may be processed by one or more consumers. A *lazy stream* is a stream that is only extended when the consumer of that stream requests the producer to do so.

Pull-stream is a communication protocol between modules to propagate the values of a lazy stream, handles its completion, and handles failures during processing.

# Terminology

Different pull-stream documents may use different terms to describe the same things. We provide here a table of all terms used in this specification and other synonyms that are used in other documents. The terms and their relationships are illustrated in the following figure:

![Image](./pull-stream.png)

| Term                 | Definition                                                                                                                              | Synonyms                                     |
| :------------------- | :-----------------------------------------------------------------------------------------------                                        | :------------------------------------------- |
| Answer               | Event from a module that happens after a request and goes to the module immediately downstream.                                         | Callback                                     |
| Downstream           | The module(s) in a pipeline that come(s) next when following the flow of values.                                                        |                                              |
| Completed Stream     | Stream that cannot be extended. Stays completed forever. Corresponds to the sequence ````value(V1), value(V2), ... value(Vn), done````.    |                                              |
| Failed Stream        | Stream in which an error occurred. Stays failed forever. Corresponds to the sequence ````value(V1), value(V2), ... value(Vn), error(E)````.|                                              |
| Indication           | Event coming from a module to the module immediately upstream. It expects no answer.                                                    | Read with no callback                        |
| Lazy Generation      | Generation of values only after an explicit requested.                                                                                  |                                              |
| Module               | Element of a pipeline, may be a source, a sink, or a transformer.                                                                       |                                              |
| Partial Stream       | Stream that can be extended by more values in the future. Corresponds to the sequence ````value(V1), value(V2), ... value(Vn), _````.      |                                              |
| Pipeline             | Composition of a single source, zero or multiple transformers, and a single sink.                                                       |                                              |
| Primitive Sink       | Sink that is not composed from other modules.                                                                                                                       |                                              |
| Primitive Source     | Source that is not composed from other modules.                                                                                                                      |                                              |
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

# Format for Representing the Protocol as a Sequence of Events

The pull-stream protocol may be implemented with different programming techniques in addition to the original callback-based implementation that was proposed by Dominic Tarr. In order to describe the protocol in a way that is independent of a particular programming technique, we describe the possible interactions between different modules as sequence of events. Two implementations with different programming techniques are therefore equivalent if they generate the same sequence of events.

Moreover, a processing pipeline may be composed of many modules that all follow the same protocol. To avoid the complexity of describing the entire processing pipeline, we describe the sequence of events from the point of view of an abstract module in relationship to adjacent ones that are either upstream or downstream.

A basic pairwise interaction describes the interaction between a module and a source or a sink. Values flow from an upstream module to a downstream module (````U=>D````). To describe a transformer module, two pairwise interactions are necessary, one with the module upstream and one with the module downstream. Values therefore flow from an upstream module to a transformer input then from the same transformer output to a downstream module (````U=>(TI TO)=>D````). In both cases, since the requests propagate in the opposite direction as the values (from downstream to upstream), requests will first happen downstream and propagate upstream, then answers will propagate from upstream towards downstream. 

The (relative) module that generated an event is specified using a prefix before the event, which may be one of the following: 

    Prefix: Event


| Prefixes   | Meaning                      |
| :--------- | :--------------------------- |
| D: Event   | Downstream event.            |
| U: Event   | Upstream event.              |
| TI: Event  | Transformer input event.     |
| TO: Event  | Transformer output event.    |

To remove any specific timing assumption from the specification, we only mention the ordering between different events and not how much time was spent or what other internal or external events may have happened in-between. To express that event A always happens before event B, we write ````A->B````. An event A may happen before or after any two other concurrent events B and C that do not have an order between one another. The concurrency therefore introduces a partial order between A, B, and C. To make the sequence of concurrent events easier to read, we write them on multiple lines so that if event E' happens transitively after event E, it will most often (but not strictly) be at its right horizontally. Any sequence of concurrent events, which when written on a single horizontal timeline, that respects the ordering relationship is a valid interaction within the pull-stream protocol. Any sequence that violates some ordering relationship is not.

To represent ordering relationships between events written on multiple lines such as:
    
    A B
    C D

We approximate a 'diagonal' ````->```` with the  ASCII characters ````\->```` and ````-\```` for ````A->D```` when it is concurrent with ````A->B````:

    A  -> B
    C \-> D

    A  -\ -> B
    C    D

The opposite 'diagonal' ````->```` is written ````/->```` and ````/-```` for ````C->B```` when it is concurrent with ````C->D````:

    A /-> B
    C  -> D

    A    B
    C  -/ -> D
    
The choice of one or the other depends on which one is actually clearer in the specific context.

Sometimes, an event B following an event A may also be written on a second line if there is not enough space on the first
line to write the full sequence. To disambiguate with concurrent lines, we write the second line with a leading arrow ````->```` which therefore implicitly refers to the last item of the first line:
    
    A -> ...
    -> B -> ...
    
When the protocol allows many choices of sequence of events, we write ```` Choice1 | Choice2 | ... | ChoiceN````. We use brackets to disambiguate between sequences (ex: ````(A->B) | C````). 

When an event A may or may not happen before event B, we write ````(A ->)? B````.

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
* Either the stream is partial, and can be extended by a value, or it is complete and stays complete forever. Values cannot be updated after having been produced and new values must not be produced after the `done` event happened;
* Because of the monotonicity of the closing operation, modules can perform closing operations for clean-up and ressource release the first time a done event has happened.

## Possible Interactions

There is a single parameter to consider for the case analysis: the number of values in the stream (N-1).

### Base Case

Request on an empty stream (N == 1):

    D: ask(Ans) -> U: Ans=done

Concurrent requests on an empty stream (N == 1):

    D: ask1(Ans1)  -> U: Ans1=done
                  \-> D: ask2(Ans2) -> U: Ans2=done

### Induction Case

Concurrent requests on a non-empty stream (N > 1)

    D: ask1(Ans1) -> U: Ans1=value(V1)
                  ...
                  \-> D: askN(AnsN) -> U: AnsN=done
                                   \-> D: askN2(AnsN2) -> U: AnsN2=done
                                   ...


The previous concurrent requirements are the minimal synchronization requirements for the protocol to work. They allow some level of concurrency. In practice however, concurrency is restricted by waiting for the answer to a request to be returned before making another request. This amounts to linearizing all the requests. This approach amounts to performing co-routining between two modules, as shown below.

Linear requests on a non-empty stream (N > 1):

    D: ask1(Ans1) -> U: Ans1=value(V1)
    -> ...
    -> D: askN(AnsN) -> U: AnsN=done 
    -> D: askN2(AnsN2) -> U: AnsN2=done 
    -> ...                                                 
                                                   

### Transformer Interactions

Request on an empty stream (N == 1):

    D: ask1(Ans1) -> TI: askT1(AnsT1) -> U: AnsT1=done -> TO: Ans1=done

Concurrent requests on an empty stream (N == 1):

    D: ask1(Ans1) -> TI: askT1(AnsT1) -\ -> U: AnsT1=done -> TO: Ans1=done
                 \-> D: ask2(Ans2) ->  TI: askT2(AnsT2) -> U: AnsT2=done -> TO: Ans2=done


Concurrent requests on a non-empty stream (N > 1):

    D: ask1(Ans1) -> TI: askT1(AnsT1) -\ -> U: AnsT1=value(V1) -> TO: Ans1=value(V1')
                   ...
                   \-> D: askN(AnsN) ->   TI: askTN(AnsTN) -\ -> U: AnsTN=done -> TO: AnsN=done
                                    \-> D: askN2(AnsN2) ->   TI: askTN2(AnsTN2) -> U: AnsTN2=done -> TO: AnsN2=done
                                                    ...
Sequential requests on a non-empty stream (N > 1):

    D: ask1(Ans1) -> TI: askT1(AnsT1) -> U: AnsT1=value(V1) -> TO: Ans1=value(V1')
    -> ...
    -> D: askN(AnsN) -> TI: askTN(AnsTN) -> U: AnsTN=done -> TO: AnsN=done
    -> D: askN2(AnsN2) -> TI: askTN2(AnsTN2) -> U: AnsTN2=done -> TO: AnsN2=done
    -> ...

# (2) Abortable Protocol: (1) + Early Aborting

New events are in *italic*.

| Downstream Requests | Meaning                                                          | Callback Implementation  |
| :------------------ | :--------------------------------------------------------------- | :----------------------- |
| ask(Ans)            | Requests a value and expects the answer in Ans.                  | ````read(false, ans)```` |
| *abort(Ans)*        | Aborts the stream and expects a confirmation Ans=done.           | ````read(true, ans)````  |

| Downstream Indications | Meaning                                                          | Callback Implementation  |
| :--------------------- | :--------------------------------------------------------------- | :----------------------- |
| *done*                 | Aborts the stream and does not wait for a confirmation.          | ````read(true)````       |

| Upstream Answers    | Meaning                                                          | Callback Implementation  |
| :------------------ | :--------------------------------------------------------------- | :----------------------- |
| Ans=value(V)        | Provides the value V.                                            | ````ans(false, v)````    |
| Ans=done            | Indicates that the stream has completed.                         | ````ans(true)````        |

## Properties

In addition to the properties for (1), the Abortable Protocol introduce this new property:

* The stream may be completed before all values have been produced by any module in the pipeline, i.e. the number of requested values may be less than the total number of possible values in the stream.

## Interactions

There are two parameters to consider for the case analysis, the number of requested values (R-1) and the number of values in the stream (N-1).

### The number of requested values is greater than the size of the stream (R > N)

The stream completes before the module downstream aborts it, any subsequent abort has no effect.


Concurrent requests (R > N) on a non-empty stream (N > 1):

    D: ask1(Ans1) -> U: Ans1=value(V1)
                ...
                \-> (D: askN(AnsN) -> U: AnsN=done)
                                    ...
                                    \-> (D: abortR(AnsR) -> U: AnsR=done) 
                                      |  D: doneR
                                        ...
                                        
#### Transformer Interactions:

Concurrent requests on a non-empty stream (N > 1):

    D: ask1(Ans1) -> TI: askT1(AnsT1)   -\ -> U: AnsT1=value(V1) -> TO: Ans1=value(V1')
                   ...
                 \-> D: askN(AnsN) ->   TI: askTR(AnsTR)   -\ -> U: AnsTR=done -> TO: AnsR=done
                                    ...
                                    \-> (D: abortR(AnsR)   -> TI: abortTR(AnsTR) -> U: AnsTR=done -> TO: AnsR=done) 
                                      |  D: doneR
                                      ...

### The number of requested values is equal or less than the size of the stream (R <= N) 

R==1 (0 requested values):

    (D: abort(Ans) -> U: Ans=done) | D: done

Concurrent requests (R > 1) on a non-empty stream (N > 1):

    D: ask1(Ans1) -> U: Ans1=value(V1)
                ...
                \-> (D: abortR(AnsR) -> U: AnsR=done) | D: doneR
                                    \-> (D: askR2(AnsR2) -> U: AnsR2=done) 
                                      | (D: abortR2(AnsR2) -> U: AnsR2=done) 
                                      |  D: doneR2
                                        ...


#### Transformer Interactions:

R==1 (0 requested values):

    D: abort(Ans1) -> TI: abort(AnsT1) -> U: AnsT1=done -> TO: Ans1=done

Concurrent requests on a non-empty stream (R > 1):

    D: ask1(Ans1) -> TI: askT1(AnsT1)   -\ -> U: AnsT1=value(V1) -> TO: Ans1=value(V1')
                   ...
                 \-> D: abortR(AnsR) ->   TI: abortTR(AnsTR)   -\ -> U: AnsTR=done -> TO: AnsR=done
                                    \-> (D: askR2(AnsR2)     -> TI: askTR2(AnsTR2)   -> U: AnsTR2=done -> TO: AnsR2=done) 
                                      | (D: abortR2(AnsR2)   -> TI: abortTR2(AnsTR2) -> U: AnsTR2=done -> TO: AnsR2=done) 
                                      |  D: doneR2
                                      ...
                                      
Early-abort originating from a transformer (R > 1):

    D: ask1(Ans1) -> TI: askT1(AnsT1)   -\ -> U: AnsT1=value(V1) -> TO: Ans1=value(V1')
                   ...
                \-> (D: **askR(AnsR)** ->)? TI: abortTR(AnsTR)            -\ -> U: AnsTR=done (-> TO: AnsR=done)?
                                             \-> (D: askR2(AnsR2)       -> TI: askTR2(AnsTR2)   -> U: AnsTR2=done -> TO: AnsR2=done) 
                                               | (D: abortR2(AnsR2)     -> TI: abortTR2(AnsTR2) -> U: AnsTR2=done -> TO: AnsR2=done) 
                                               |  D: doneR2
                                               ...

# (3) Fault-handling Protocol: (1) + (2) + Error Handling

New events are in *italic*.

| Downstream Requests | Meaning                                                                | Callback Implementation                 |
| :------------------ | :--------------------------------------------------------------------- | :------------------------------------- |
| ask(Ans)            | Requests a value and expects the answer in Ans.                        | ````read(false, ans)````               |
| abort(Ans)          | Aborts the stream and expects a confirmation Ans=done.                 | ````read(true, ans)````                 |
| *error(Msg Ans)*    | Fails the stream and expects a confirmation Ans=done or Ans=error(Msg).| ````read(new Error('Msg'), ans)````  |

| Downstream Indications | Meaning                                                          | Callback Implementation        |
| :--------------------- | :--------------------------------------------------------------- | :----------------------------- |
| done                   | Aborts the stream and does not wait for a confirmation.          | ````read(true)````             |
| *error(Msg)*           | Fails the stream and does not wait for a confirmation.           | ````read(new Error('Msg'))```` |

| Upstream Answers    | Meaning                                                          | Callback Implementation       |
| :------------------ | :--------------------------------------------------------------- | :---------------------------- |
| Ans=value(V)        | Provides the value V.                                            | ````ans(false, v)````         |
| Ans=done            | Indicates that the stream has completed.                         | ````ans(true)````             |
| *Ans=error(Msg)*    | Indicates that the stream has failed.                            | ````ans(new Error('Msg'))```` |


## Properties

Same as for (2) with the additional property:

* A module may explicitly mention the reason for failing the stream. 

## Interactions

The interactions are the same as for (2), except that the error message should be propagated. Once failed, the answer to a request must be the error that failed the stream.

# Limitations

1. The protocol does not easily allow parallel processing between the different modules of a pipeline. To do so, the sink needs to make as many concurrent requests as there are modules in the pipeline, it therefore needs to know how many there are. A source-driven pipeline would not require knowledge of the number of downstream modules (but would loose laziness).

# Callback Implementation

# Dataflow Implementation

