# How NOT to get panicked using Go channels 

In this article, I'll share some of my experiences about how to work with Golang channels effectively without having a total panic attack. Hope you'll enjoy.

## Unbuffered channel vs Buffered channel

Go channels are like, well, channels, that connect goroutines and help them communicating with each other. A channel is initialized using the `chan` keyword followed by the type of data that get transferred in the channel. It might also take in another capacity paramenter, which is the size of the queue that messages sit in before the channel is ready to transfer, hence buffered channel. Unbuffered channels are channels with buffer size of zero.

```go
c  := make(chan int)     // c is an unbuffered channel
bc := make(chan int, 10) // bc is a buffered channel with the capacity of 10 
```

These are some basic principles that everyone must be familiar with to work with channels properly:

1. Sender(s) are goroutines that send values to channels, receivers are goroutines that receive values from channels:

    ```go
    c <- x    // send x to channel c
    x := <-c  // receive x from channel c
    ```

2. *If a channel buffer is full, sender(s) will be blocked until there's a receiver finish reading from it, if there isn't one, deadlock will occurs on the senders side and the program will panic*
3. The opposite thing happens when the channel is empty, all receivers are blocked until there's someone sends something to it. And if not, they will also panic

Because unbuffered channels have buffer size of zero, that means they are always both full *and*  empty at the same time. The direct consequence from this is if you send to or read from a channel in the same goroutine right after it is initialized, it will cause panic.
```go
c := make(chan int)
c <- 1                  // panicked!
<-c                     // also panicked!
```

Bear in mind that channeling is a *communicating model*, if there's a talker (sender) there also have to be a listener (receiver) and otherwise. So to make the above example work, you have to start another goroutine to communicate with the current one:
```go
c := make(chan int)
go func(){
  c <- 1
}()
fmt.Println(<-c)
```

## Properly Closing a Channel

Channels are like teenagers with paranoid personality disorder, they get panicked so damn easily. Reading from an empty channel with no one sending to it causes panic. Sending to a full channel with no one reading from it causes panic. Closing a closed channel causes panic. And sending to a closed channel also causes panic. Whether you're in the role of sender or receiver, you always have to be extremely careful with it.

Like normal communications, there are various types of communicating through a channel: one-to-one (one sender, one receiver), one-to-many (one sender, many receivers), many-to-one (many senders, one receiver) and many-to-many (many senders, many receivers). Identifying if a conversation is over yet is a critical task and needs a more effective mechanism than, let's say, waiting for the silence becomes too awkward to stay so that you just stand up and leave and never get invited to hang out with your friends again. In channeling model, this mechanism is channel closing: all parties can just check if the channel is closed to know the communication is over or not. But again, we have to be careful not to cause any panics to happen.

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

In these cases, only the last sender has the right to close the channel, because if the channel is closed any sooner, it will cause panics. One method can be used to ensure that it can be done is keeping track of how many senders joining to the channel and how many that already terminated, so that whenever a sender is about to terminate, it can check if it's the last one and then close the channel. 

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

In a less ideal circumstance, we need a way to signal other senders about the channel closing event, so that they will not crash our program by sending more data to it. In that case, you might want to check out the article [How to Gracefully Close Channels](https://go101.org/article/channel-closing.html) for more information.

Thanks for reading.
