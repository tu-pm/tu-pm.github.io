# How NOT to get panicked using Go channels 

In this article, I'll share some of my experiences about how to work with Golang channels effectively without having panic attacks. Hope you'll enjoy.

## Unbuffered channel vs Buffered channel

Buffered channel can be created by using capacity argument:

```go
c  := make(chan int)     // c is an unbuffered channel
bc := make(chan int, 10) // bc is a buffered channel with the capacity of 10 
```

You can think of unbuffered channels are buffered channels with capacity of zero. But what does *capacity* mean in this context? Just look at these working principles of a channel: 

1. Sender(s) are goroutines that send values to channels, receivers are goroutines that receive values from channels:

    ```go
    c <- x    // send x to channel c
    x := <-c  // read x from channel c
    ```

2. *If a channel is full, sender(s) will be blocked until there's some receivers finished reading from it*. An unbuffered channel is always full and a buffered channel is full if it contains as many items as its capacity. *If there's no receiver reading from a full channel, deadlocks will occur to all of its senders and they will all go panic!*
3. The opposite thing happens when the channel is empty, all receivers are blocked until there's someone sends something to it, and if there's none, they will also panic

The direct result from this is sending to or receiving from an unbuffered channel in the same goroutine *always* causes panic:
```go
c := make(chan int)
c <- 1                  // panicked!
<-c                     // also panicked!
```

Just bear in mind that channeling is a *communicating model*,  if there's a talker (sender) there also have to be a listener (receiver) and otherwise. So to make the above example work, you have to start another goroutine to communicate with the current one:
```go
c := make(chan int)
go func(){
  c <- 1
}()
fmt.Println(<-c)
```

## Properly Closing a Channel

Channels are like teenagers with paranoid personality disorder, they get panicked so damn easily. Reading from an empty channel with no one sending to it causes panic. Sending to a full channel with no one reading from it causes panic. Closing a closed channel causes panic. And sending to a closed channel also causes panic. Whether you're in the role of sender or reader, you always have to be extremely careful with it.

Like normal communications, there are various types of communicating through a channel: one-to-one (one sender, one receiver), one-to-many (one sender, many receivers), many-to-one (many senders, one receiver) and many-to-many (many senders, many receivers). Identifying if a conversation is over yet is a critical task and needs a more effective mechanism than, let's say, waiting for the silence becomes too awkward to stay so that you just stand up and leave and never get invited to hang out with your friends again. In channeling model, this mechanism is channel closing, all parties can just check if the channel is closed to know the communication is over or not. But again, we have to be careful not to cause any panics to happen.

The ultimate rule for closing a channel is: **Don't close a channel when there's still someone intends to send to it**. Let's see if we can achieve our goal without violating this rule.

### One-to-One and One-to-Many Models

These are the easiest cases, because there's only one sender, it can just close the channel, indicating that it has nothing to send anymore.

- Sender:
    ```go
    go func(){
      ...
      close(ch)
      ...
    }()
    ```
- Receivers:
    ```go
    go func(){
      message, ok := <- ch
      if ok{
        // handle message
      } else {
        // channel is closed, stop listening
      }
    }()
    ```

### Many-to-One and Many-to-Many Models

In these cases, only the last sender has the right to close the channel, because if the channel is closed any sooner, it will cause panics. One method can be used to ensure that can be done is keeping track of how many senders joining to the channel and how many that already terminated, so that whenever a sender's about to terminate, it can check if it's the last one and then close the channel. 

- Main goroutine:
    ```go
    type Counter struct {
      total      int 
      terminated int 
      mux        sync.Mutex
    }

    func (counter *Counter) Join() {
      counter.mux.Lock()
      counter.total += 1
      counter.mux.Unlock()
    }

    func (counter *Counter) Terminate() {
      counter.mux.Lock()
      counter.terminated += 1
      counter.mux.Unlock()
    }

    func (counter *Counter) IsLast() {
      counter.mux.Lock()
      defer counter.mux.Unlock()
      return counter.mux.terminated == counter.mux.total-1
    }
    var counter Counter
    ```
- Senders:
    ```go
    go func(){
      counter.Join()
      // do something
      // ...
      // before terminating
      if counter.IsLast(){
        close(ch)
      }
      counter.Terminate()
    }()
    ```
- Receivers:
    ```go
    go func(){
      message, ok := <- ch
      if ok{
        // handle message
      } else {
        // channel is closed, stop listening
      }
    }()
    ```
If you notice, there's a **fatal flaw** in this method: If there's no senders joining the channel before the current one terminates, the channel will be closed. Most of the time, all sender workloads are pretty heavy (that's why we have to use multiple senders in the first place) so this will likely happen to the last one. But there's no way to be ABSOLUTELY sure that it doesn't happen right at the first goroutine.

There is, however, a certain condition this method can work. That is if we know exactly how many senders are there, so that we can confidently close the channel after all of them terminated. Otherwise, closing a multiple-sender channel, with an absolute guaranty that there's no more data sent to it, is just an impossible task. 

In a less ideal circumstance, we need a way to signal other senders about the channel closing event, so that they will not send data to that channel, causing panics and crashing our program. In that case, you might want to check out the article [How to Gracefully Close Channels](https://go101.org/article/channel-closing.html) for more information.

Thanks for reading.
