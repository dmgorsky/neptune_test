## Test Assignment

### Task Overview

Your task is to design and implement a high-performance RESTful service capable of handling the rigorous demands of high-frequency trading systems. This service will act as a component in the company ABC trading infrastructure, managing and analysing financial data in near real-time.

Your service must support two HTTP-based API endpoints communicating via JSON

1. POST /add_batch/
    * **Purpose**: Allows the bulk addition of consecutive trading data points for a
      specific symbol. (in-memory storage)
    * **Input**:
        * `symbol` String identifier for the financial instrument.
            * `values` Array of floating-point numbers representing sequential trading prices.
    * **Response**: Confirmation of the batch data addition.
2. GET /stats/
    * **Purpose**: To provide rapid statistical analyses of recent trading data for specified symbols.
    * **Input**:
        * `symbol` The financial instrument's identifier.
        * `k` An integer from 1 to 8, specifying the number of **last 1e{k}** data points to analyze
    * **Response**:
        * `min` Minimum price in the last 1e{k} points.
        * `max` Maximum price in the last 1e{k} points.
        * `last` Most recent trading price.
        * `avg` Average price over the last 1e{k} points.
        * `var` Variance of prices over the last 1e{k} points.

### Technical Requirements

* **Data Handling**: Implement an efficient data structure for real-time data
  insertion and retrieval of specified requests.
* We are looking for a **single-node, in-memory (no persistent storage)** implementation, assuming the server has enough RAM;
* **Limits**: There will be no more than 10 unique symbols;
* **Language & Framework**: You may use any backend programming language and framework you find suitable for near-real-time data processing and RESTful API implementation.
* **Concurrency & Performance**: The solution must efficiently handle a high volume of concurrent data entries and statistical requests;
    * **No two concurrent add or/and get requests will occur simultaneously within a given symbol**.

## Solution Design

We need to build a web server utilizing a stateful (in-memory) service that updates prices and calculates statistics for specific financial identifiers (symbols).
I started with `Rust` implementation, adding Scala and Java later.
The service functionality should consist of two parts:

### 1. services' registry (e.g. associative array) for different symbols.
Simple, we manage a `map` of services (either Actors or stateful objects), adding new on receiving `add_batch` for absent symbol, and guarding the total symbols' limit.
Also, we dispatch `add_batch` and `stats` requests to corresponding service.

#### Tests needed:
* requests dispatch
* total symbols limit

### 2. Actual stateful service for one symbol
The requirement about sequential requests on one symbol service hints that we can use actor paradigm:
* safely update our internal prices' array
* avoid locks when accessing internal state
    * **But** at first I decided to also segregate reads and writes, blocking only on writes and allowing reads simultaneously ("*poor man's CQRS*"), so developed a service w/o actors paradigm. Then it translates into actors model easily.

In the service we need to implement
* `push_prices` - appends an input array of prices to our "buffer"
    * Obviously, to avoid any allocations, we need some sort of `Circular buffer` to store financial data.
    * I didn't want to mess with begin/end pointers, so, as a quick and dirty solution, I used a heap-allocated array initially pre-allocated with goal `1e8` capacity. On adding prices, if that `10_000_000` limit is reached, I remove (drain) beginning elements from array. "*Poor man's ring buffer*"
    * Such design came because I implemented multi-threading service with reading and writing prices simultaneously at first. Of course, in case of state safety in actors' model, one should better use real circular buffer without additional allocations.
* `get_stats`
    * here we just need an `iterator` providing the stream of numbers to calculate statistics on
    * this stream should end at the end of our prices array and begin at corresponding offset
    * we need to do the second pass to calculate variance

#### Tests needed:
* prices array limits of `add_batch`
* correct borders of calculation stream in `get_stats`
* actual data in `get_stats` after `add_batch`

## Implementations
1. **Rust** - w/o Actors (*Actix-web*)
   https://github.com/dmgorsky/neptune_test_rust
2. *WIP* Rust - with Actors (*Actix-web, Actix*)
   https://github.com/dmgorsky/neptune_test_rust_actors
3. **Scala** - with Actors (*Akka*)
   https://github.com/dmgorsky/neptune_test_scala
4. *WIP* Java (*Play!, Apache Pekko* or *Spring Boot*)
   https://github.com/dmgorsky/neptune_test_java
