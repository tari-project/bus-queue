 # Lock-free Bounded Non-Blocking Pub-Sub Queue

 This is a publish subscribe pattern queue, where the publisher is never blocked by
 slow subscribers. The side effect is that slow subscribers will miss messages. The intended
 use-case are high throughput streams where receiving the latest message is prioritized over
 receiving the entire stream. Market Data Feeds, Live Streams, etc....

 The underlying data-structure is a vector of Arc(s) eliminating the use of copies.

## Features
 * Lock-Free Write/Read - Lock-Free for Publisher and Lock-Free for Subscribers.
 * Bounded - Constant size of memory used, max is **sizeof(MsgObject)*(queue_size + sub_cnt + 1)**.
   This is an edge-case where each subscriber is holding a ref to an object while the publisher
   has published a full length of queue in the mean time.
 * Non-Blocking - The queue never blocks the publisher, slow subscribers miss data proportinal to
   their speed.
 * Pub-Sub - Every Subscriber that can keep up with the Publisher will recieve all the data the
   Publisher publishes.
 * **sync**/**async** - both interfaces are provided, as well as a bare queue implementation
   without the thread synchronisation ,and futures logic.
 * **std::sync::mpsc** like interface - The API is modeled after the standard library mpsc queue,
   channel function are used to create a tuple of (Publisher, Subscriber), while the Clone trait on Subscriber
   creates additional subscribers to the same Publisher

 **sync::Publisher**, **async::Publisher**, and **BarePublisher** are used to broadcast data to
 **sync::Subscriber**, **async::Subscriber**, and **BareSubscriber** pools. Subscribers are
 clone-able such that many threads, or futures, can receive data simultaneously. The only
 limitation is that Subscribers have to keep up with the frequency of the Publisher. If a
 Subscriber is slow it will drop data.

 ## Disconnection

 The broadcast and receive operations on channels will all return a **Result**
 indicating whether the operation succeeded or not. An unsuccessful operation
 is normally indicative of the other half of a channel having "hung up" by
 being dropped in its corresponding thread.

 Once half of a channel has been deallocated, most operations can no longer
 continue to make progress, so **Err** will be returned. Many applications
 will continue to **unwrap** the results returned from this module,
 instigating a propagation of failure among threads if one unexpectedly dies.


# Examples
## Simple bare usage
```rust
extern crate bus_queue;

use bus_queue::bare_channel;

fn main() {
    let (tx, rx) = bare_channel(10);
    (1..15).for_each(|x| tx.broadcast(x).unwrap());

    let received: Vec<i32> = rx.map(|x| *x).collect();
    // Test that only the last 10 elements are in the received list.
    let expected: Vec<i32> = (5..15).collect();

    assert_eq!(expected, received);
}
```

## Simple synchronous usage
```rust
extern crate bus_queue;

use bus_queue::sync;
fn main() {
    // Create a sync channel
    let (mut tx, rx) = sync::channel(10);
    // spawn tx thread, broadcast all and drop publisher.
    let tx_t = std::thread::spawn(move || {
        (1..15).for_each(|x| tx.broadcast(x).unwrap());
    });
    // small sleep for the tx thread to send and close, before rx thread is called
    std::thread::sleep(std::time::Duration::from_millis(100));

    // spawn rx thread to get all the items left in the buffer
    let rx_t = std::thread::spawn(move || {
        let received: Vec<i32> = rx.map(|x| *x).collect();
        // Test that only the last 10 elements are in the received list.
        let expected: Vec<i32> = (5..15).collect();
        assert_eq!(received, expected);
    });

    tx_t.join().unwrap();
    rx_t.join().unwrap();
}
```
## Simple asynchronous usage
```rust
extern crate bus_queue;
extern crate futures;
extern crate tokio;

use bus_queue::async;
use futures::*;
use tokio::runtime::Runtime;

fn main() {
    let mut rt = Runtime::new().unwrap();
    let (tx, rx) = async::channel(10);
    let sent: Vec<i32> = (1..15).collect();
    let publisher = stream::iter_ok(sent)
        .forward(tx)
        .and_then(|(_, mut sink)| sink.close())
        .map_err(|_| ())
        .map(|_| ());

    rt.spawn(publisher);

    let received: Vec<i32> = rt.block_on(rx.map(|x| *x).collect()).unwrap();
    // Test that only the last 10 elements are in the received list.
    let expected: Vec<i32> = (5..15).collect();
    assert_eq!(expected, received);
}
```
