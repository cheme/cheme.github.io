---
title: "MyDHT service usage switch (part 2)"
date: 2017-10-29T10:14:08+02:00
draft: false
categories: 
  - "MyDHT"
  - "rust"
tags:
  - "programming"
  - "p2p"
  - "design"
  - "rust"
---

The point of this post is to give a taste of how service are configured within the [MyDHT](https://www.github.com/cheme/mydht) new design.
we are going to adapt a test case to run in a single thread.

All changes describe are on 'MyDHTConf' trait implementation for 'TestLocalConf' struct (to create an application this big trait must be implemented : basically it contains the associated types and the initialisation functions of every components).

# configuring thread

The test case, 'test/mainloop.rs' test_connect_all_th, is a simple test running a local service proxying the message to the globalservice (useless except for testing), with a global service implementation managing a few touch (ping/pong) commands. The messages transmitted within this test are simple authentication (library standard), simple touch service (global service) and a simple peer query (library standard with code that can be use as global service).

'ThreadPark' spawner is used almost everywhere with an Mpsc channel and use ArcRef over Peer (Arc).

Our new test case 'test_connect_local' will be running as much as possible over a single thread, through MyDHTConf trait for 'TestLocalConf'.  
So the starting point will be a copy of the full multithreaded original test case, and we will progessivly change it to a single thread usage.

## STRef and usage of non Send reference

Firstly, the question of Arc usage without threads. Arc is use other peers (immutable and implementation dependent sized), and can be switch to RC, but that is not true for the library implementation (usage of thread or not is unknown), therefore we will use Rc through RcRef (implements SRef and Ref\<P\>) where we previously use Arc through ArcRef.

```rust
  pub trait Ref<T> : SRef + Borrow<T> {
    fn new(t : T) -> Self;
  }
```

Ref\<P\> is an abstraction of a immutable reference (Borrow\<P\> plus Clone within mydht), and replace old Arc\<P\> in our code. Ref\<P\> needs to be 'Send' (ArcRef) when we spawn threads, but could also not be send (RcRef) when we do not spawn thread (Send trait is a rust marker indicating that a type can be send to other threads, Rc as a counted reference cannot).


```rust
  type PeerRef = RcRef<Self::Peer>;
  //type PeerRef = ArcRef<Self::Peer>;
```

Compiling it will obviously break everywhere : there is no way that our 'MpscChannel' (which is only a wrapper over standard rust mpsc sender and receiver) will run with some Rc\<Peer\> which are not Send.

4 nice errors appears all similar to :
```
error[E0277]: the trait bound `std::rc::Rc<mydht_basetest::node::Node>: std::marker::Send` is not satisfied in `procs::server2::ReadService<test::mainloop::TestLocalConf>`
   --> src/test/mainloop.rs:658:6
    |
658 | impl MyDHTConf for TestLocalConf {
    |      ^^^^^^^^^ `std::rc::Rc<mydht_basetest::node::Node>` cannot be sent between threads safely
    |
    = help: within `procs::server2::ReadService<test::mainloop::TestLocalConf>`, the trait `std::marker::Send` is not implemented for `std::rc::Rc<mydht_basetest::node::Node>`
    = note: required because it appears within the type `mydht_base::utils::RcRef<mydht_basetest::node::Node>`
    = note: required because it appears within the type `procs::server2::ReadService<test::mainloop::TestLocalConf>`
    = note: required because of the requirements on the impl of `mydht_base::service::Spawner<procs::server2::ReadService<test::mainloop::TestLocalConf>, procs::server2::ReadDest<test::mainloop::TestLocalConf>, mydht_base::service::DefaultRecv<procs::server2::ReadCommand, std::sync::mpsc::Receiver<procs::server2::ReadCommand>>>` for `mydht_base::service::ThreadPark`
```

At this point we could change all threaded spawner to have no new thread and no needs for Send, but we want to be able to mix thread and non thread spawner without having to use ArcRef : the use case where we transmit peers as RcRef or CloneRef (for instance if peers is a single ip address) and still use multithreading. 

So the solution will be to simply copy/clone the Rc to the new thread, that is what the SRef trait implementation of RcRef does : give an associated sendable type for Rc\<Peer\> (simply cloning Peer) and after sending thist type in the spawn thread, put it back this to its original type : a RcRef (putting it back may not be suitable but RcRef\<Peer\> is the only kind of peer define in the mydhtconf trait and use in service commands).  
SRef is therefore a way to send a struct by using an associated sendable type, for type that are already sendable (like ArcRef) the associated type is simply 'Self'. The associated type reverts to original type through 'SToRef' implementation.
```rust
pub trait SRef : Sized {
  type Send : SToRef<Self>;
  fn get_sendable(self) -> Self::Send;
}
pub trait SToRef<T : SRef> : Send + Sized {
  fn to_ref(self) -> T;
}
```
Note, Ref\<P\> as an SRef is weaker than an Arc as it does not ensure that the same object is use in two threads after cloning, it simply ensure that we borrow the same origin information (considering P to be kindof immutable).  
This sending of SRef could not be achieve with MpscChannel (MpscChannel is constrained on Send trait), but another Channel implementation is doing it (cloning/unwraping on write and putting back to Ref on recv) : 'MpscChannelRef'

```rust
pub struct MpscChannelRef;
pub struct MpscSenderRef<C : SRef>(MpscSender<C::Send>);
pub struct MpscReceiverRef<C : SRef>(MpscReceiver<C::Send>);
pub struct MpscSenderToRef<CS>(MpscSender<CS>);
pub struct MpscReceiverToRef<CS>(MpscReceiver<CS>);
```
It simply wraps an MpscSender of the sendable inner type of our Ref<P> as an SRef, implementation is straightforward (get_sendable before sending and to_ref after sending).

Se we change all our channels : ApiChannelIn, PeerStoreServiceChannelIn, MainLoopChannelIn, MainLoopChannelOut, ReadChannelIn, WriteChannelIn, PeerMgmtChannelIn, GlobalServiceChannelIn, except localproxy service channels where we already use a 'NoChannel' as input (received message is build from frame in Read service) and a non threaded spawner : see 'localproxyglobal' macro.  
Then after manually implementing SRef for a lot of Command and Reply struct (a macro is really needed eg boilerplate code for mainloop command https://github.com/cheme/mydht/blob/0578b3ceef4678e9f341199bcb5f8deafbe6bfab/mydht/src/procs/mainloop.rs line 291) we still got :
```
error[E0277]: the trait bound `std::rc::Rc<mydht_basetest::node::Node>: std::marker::Send` is not satisfied in `procs::server2::ReadService<test::mainloop::TestLocalConf>`
   --> src/test/mainloop.rs:658:6
    |
658 | impl MyDHTConf for TestLocalConf {
    |      ^^^^^^^^^ `std::rc::Rc<mydht_basetest::node::Node>` cannot be sent between threads safely
    |
    = help: within `procs::server2::ReadService<test::mainloop::TestLocalConf>`, the trait `std::marker::Send` is not implemented for `std::rc::Rc<mydht_basetest::node::Node>`
    = note: required because it appears within the type `mydht_base::utils::RcRef<mydht_basetest::node::Node>`
    = note: required because it appears within the type `procs::server2::ReadService<test::mainloop::TestLocalConf>`
    = note: required because of the requirements on the impl of `mydht_base::service::Spawner<procs::server2::ReadService<test::mainloop::TestLocalConf>, procs::server2::ReadDest<test::mainloop::TestLocalConf>, mydht_base::service::DefaultRecv<procs::server2::ReadCommand, mydht_base::service::MpscReceiverRef<procs::server2::ReadCommand>>>` for `mydht_base::service::ThreadPark`
```
Obviously changing the channel was not enough because its type does not match our ThreadPark spawn. Our thread park spawner also requires content to be Send.  
Also, when spawning a service, a command can be use as first parameter (optional command input in spawn function), letting us run some use case as in localproxyglobal's macro where the service is running with a dummy Channel input and spawning parameter for a single iteration (like a function call). With this usecase, service must be restarted for each new command (service handle indicate that it is finished after its only iteration and service state is use to restart from handle). This usecase also required restart of finished service to be implemented (I may have been lazy on many restart it is plugged in write service and local service at least).

So, the need for sendable content is also true for the Service itself, when starting and restarting from finished handle, the state containing the service itself is send. So SRef must also be implemented for our service, especially those containing our Ref\<P\> implementation (eg 'me' field of most for most of the services).

Consequently the ThreadPark spawn need to use a Send command and similarily to what we did with channel we will use a spawner variant, 'ThreadParkRef' which is going to use the associated sendable variant of our service when spawning the new thread. Then before looping over the iteration of our service call, our type is wrapped back as an RcRef.

In fact, with this approach all Send requirement are more or less replaced by SRef (in service, receiver and sender). Some difficulties occured within some service fields : for instance query cache of kvstore can not easily be sendable when it contains Rc (heavy copy needed) : in this case it was simply not send in the variant and initiated at service start similarily to kvstore storage (boxed initializer), this lazy choice invalidates cache for restart and a proper SRef implementation is required (but not really suitable).

Similarilly, returning result abstraction 'ApiResult' could contains an ArcRef. Another trait implementation will be use to return the SRef::Send : 'ApiResultRef'. 
The further we go with implementation and the more it looks like SRef usage globally could be a good idea.

At this time we only do it for Ref<Peer> as it is the most commonly use SRef in MyDHT, but custom Global and Local service message could contain such Ref and others Ref should be added in MyDHT. The challenge byte vec (unknown size out of implementation) for authentication should possibly be SRef, yet it does not make sense as it does not require to be clone (and Vec\<u8\> is already sendable).

After changing to ThreadParkRef it finally compiles and test pass. 
As a result, 'test_connect_all_local' is configured to mainly use threads but ran with Rc\<Peer\> instead of Arc\<Peer\>, see https://github.com/cheme/mydht/blob/bd098eadba760f2aaa496dfc96f6c3a9e22293de/mydht/src/test/mainloop.rs at line 676.

I did cheat a bit on order of errors, first is Spawner then Channel, but it was simplier blogging in this order.

So what did we do ? Just changing MyDHT internal usage of Arc for peers to Rc usage while staying in a highly multithreaded environment. 

What's next? Obliteration of those threads because non blocking IO allows it.  


## Removing some threads


Next step will be easiest : changing some services to run in the same thread as the main loop service. Please note that this is only possible if the transport is asynchronous, for synch/blocking transport there is a lot of thread restrictions (plus tricks that I may describe in a future post).

### a statefull service

'Api' is a single service that stores Api queries (query requires a unique id for its matching reply) and forwards replies into 'ApiReturn' abstraction. Currently there is no suitable 'ApiReturn' implementation (an old synch struct is used to adapt old blocking test cases), some needs to be implemented (for C as single blocking function, for Rust as single blocking function, for rust as Future, not ipc as 'MainLoopChannelOut' seems enough for this case...).  
'Api' is a non mandatory service (a MyDHTConf trait boolean associated constant is used to switch result to MainLoopChannelOut when Api Query management is unneeded : the management can be external by simply listing on a result channel (unordered results)), it does not suspend (yield) on anything except its receiving message queue : it is easy to see that suspending is statefull in this case (cache is include in the service struct and message handling is out of call function so it is highly unprobable that some of this function inner state is needed).

The marker trait 'ServiceRestartable' is used for this kind of service, it indicates that when Yielding/suspending we can simply end the service and store the state (service struct and current channels) in the handle. When yielding back the handle can simply restart the service from its state.

```rust
impl<MC : MyDHTConf,QC : KVCache<ApiQueryId,(MC::ApiReturn,Instant)>> ServiceRestartable for Api<MC,QC> { }
```

It does not mean that we can restart the service after it is finished (max iteration reached or ended with command), naming is confusing.

An example of local spawner implementation with a suspend/restart strategy is 'RestartOrError' :

```rust
impl<S : Service + ServiceRestartable, D : SpawnSend<<S as Service>::CommandOut>, R : SpawnRecv<S::CommandIn>> Spawner<S,D,R> for RestartOrError {
  type Handle = RestartSameThread<S,D,Self,R>;
  type Yield = NoYield;
``` 

Its handle is 'RestartSameThread', an enum containing the handle state (ended or yielded, no running state as not possible over a single thread without coroutine or local context switch) and the service state (service struct and its channels plus iteration counter). Interesting point is that it also contains the spawner for restart (in our case our spawner is the 0 length struct 'RestartOrError').

Its yield is a dummy 'NoYield' simply returning 'YieldReturn::Return' (meaning that yield calling function must 'wouldblock' error return) so that the spawner just prepare for return/suspend (fill handle with state) and exit.

We simply change the following lines :
```rust 
  // type ApiServiceSpawn = ThreadParkRef;
  type ApiServiceSpawn = RestartOrError;
```
and 
```rust 
  fn init_api_spawner(&mut self) -> Result<Self::ApiServiceSpawn> {
    //Ok(ThreadParkRef)
    Ok(RestartOrError)
  }
```
We can see that initialisation function ('init_api_spawner') is using MyDhtConf as mutable reference. This choice allows more complex spawner usage, for instance if we want to use a threadpool, the initialized pool reference may be include in some Arc Mutex and cloned into every spawners.

Compiles and run pass, nice.

Still, we did not change the channel, we still use 
```rust
  type ApiServiceChannelIn = MpscChannelRef;
```
Useless, our api service is now on the same thread as main loop. We can send command localy by using LocalRcChannel (internally Rc\<RefCell\<VecDeque\<C\>\>\>).
```rust
  //type ApiServiceChannelIn = MpscChannelRef;
  type ApiServiceChannelIn = LocalRcChannel;
```
and 
```rust
  fn init_api_channel_in(&mut self) -> Result<Self::ApiServiceChannelIn> {
//    Ok(MpscChannelRef)
    Ok(LocalRcChannel)
  }
```

Alternatively an unsafe single value buffer should be fine as currently we unyield on each send of command.
That is not strictly mandatory so we keep the generic safe VecDeque choice (optimizing this way is still doable but low value in our case).
Ideally 'NoChannel' (disabled channel struct) should be use but it requires to change unyield function to allow an optional command (currently this could be good as we systematically unyield on each send).

When looking at this restart handle associated types we observe :
```rust
impl<S : Service + ServiceRestartable,D : SpawnSend<<S as Service>::CommandOut>, R : SpawnRecv<S::CommandIn>>
  SpawnHandle<S,D,R> for 
  RestartSameThread<S,D,RestartOrError,R> {
  type WeakHandle = NoWeakHandle;
  ...

  #[inline]
  fn get_weak_handle(&self) -> Option<Self::WeakHandle> {
    None
  }
```

The weakhandle is a dummy type (also true for others non threaded spawner handles), never returned.  
This illustrates the way message passing will switch when changing spawner : without weak handle the other services (eg 'GlobalDest' containing optional api HandleSend) will send ApiCommand to MainLoop which will proxy it (same thread as api), with a WeakHande like previously, the other service can send directly (cf optional weakhandle for spawn handle in previous post).  

The weakhandle is stored in (with a sender) the dest of other service, for instance :

```rust
pub struct GlobalDest<MC : MyDHTConf> {
  pub mainloop : MainLoopSendIn<MC>,
  pub api : Option<ApiHandleSend<MC>>,
}
```

Thinking back about this for api, it looks quite suitable : close to having the query cache in the mainloop and calling a looping function on it, but not costless (inlining everything is theorically fine for service stored in yield handle, even not inline cache implementation is probably already on heap, same for message waiting in channel). Main bad thing is the use of a channel (a simple vecdeque here) for passing command : to be closest to a function call we could simply use a 'Blocker' spawner with a single iteration limit : use of 'localproxyglobal' macro in our test is an example.

### a non statefull service

For some service previous strategy could not be use as some state could not be restore on unyield (the service unyielding on other events than the channel call). That is the default case (no 'RestartSameThread' marker).  
For read or write service, this is related to the fact that reading is streamed inside of the serializer : we cannot save the state of the serializer/deserializer due to the way we serialize/deserialize directly from the streams.  
For reference Read service gets the message through the 'MsgEnc' trait method 'decode_msg_from' :
```rust
  fn decode_msg_from<R : Read>(&self, &mut R) -> MDHTResult<M>;
```
We do not get message directly by reading on a reader which returns 'WouldBlock' if we need to yield, if it was the case it would be easy to put the Read stream in a service field and state will be save.  
The problem is that our Read stream is a composition of blocking oriented read traits and the lower level transport read is not : therefore when the lower level transport Read return a WouldBlock error (meannig that we should yield) the CompExt should be implemented in a way that reading could restart (doable).
Biggest issue is that the serializer (generally a serde backend) used for the implementation of MsgEnc would requires to be resumable on a 'WouldBlock' error (not the case at the moment I think or at least for the majority of implementation).

Therefore, the strategy for yielding on Asynch Read/Write is to use a CompExt layer (composition) over the transport stream, the layer will catch 'WouldBlock' Error and Yield on it (no support of statefull suspend with YieldReturn::Return).
CompExt used are :
```rust
pub struct ReadYield<'a,R : 'a + Read,Y : 'a + SpawnerYield> (pub &'a mut R,pub &'a mut Y);
```
and 
```rust
pub struct WriteYield<'a,W : 'a + Write, Y : 'a + SpawnerYield> (pub &'a mut W, pub &'a mut Y);
```
For threaded spawn, like a thread pool or ThreadPark, the thread simply park (block on a condvar) and resume later through its yield handle resume (unlock condvar).
For local spawn, we need to use a CoRoutine : with CoRoutine spawner (using coroutine crate), the coroutine state does include all inner states (even possible Serde backend implementation) and we can suspend at WriteYield/ReadYield level (using YieldReturn::Loop strategy).

Lets adapt write service.
```rust
  //type WriteChannelIn = MpscChannelRef;
  type WriteChannelIn = LocalRcChannel;
  //type WriteSpawn = ThreadParkRef;
  type WriteSpawn = CoRoutine;
```
Note that with a blocking (synch) write stream, write service could allow suspend (coroutine is useless as statefull), and mydht will still run with an impact depending on transport (for many transport is not a major issue if not threaded). For blocking ReadStream suspending only on message input (blocking transport case) is a huge issue (thread simply blocks on reading and if nothing is received mydht instance is stuck).  

Also note that keeping an MpscChannelRef (or a MpscChannel if using ArcRef\<Peer\>) could still make sense to enable direct message sending to write service through its weak send and handle (but it would also require that CoRoutine implements a WeakHandle (which is currently not the case : it would requires to run a shared mutex over the handle and would be a different usecase)).

Last point, the number of iteration should be unlimited or high to avoid to much coroutine creation.
```rust
  //const SEND_NB_ITER : usize = 1;
  const SEND_NB_ITER : usize = 0; // 0 is unlimited run
```


Applying this change to read is way more interesting (infinite service with really recurring suspend on Read transport stream) and run as smoothly :
```rust
  type ReadChannelIn = LocalRcChannel;
  type ReadSpawn = Coroutine;
```

### a blocking service

Already seen before, macro 'localproxyglobal' used in this test conf is an example of a local service call that does not yield ('Blocker' spawner), with a single iteration and no channel.
The service is spawn at each time like a function call (optional command is set) and never suspends.
```rust
#[macro_export]
macro_rules! localproxyglobal(() => (
  type GlobalServiceCommand = Self::LocalServiceCommand;
  type GlobalServiceReply  = Self::LocalServiceReply;
  type LocalService = DefLocalService<Self>;
  const LOCAL_SERVICE_NB_ITER : usize = 1;
  type LocalServiceSpawn = Blocker;
  type LocalServiceChannelIn = NoChannel;
  #[inline]
  fn init_local_spawner(&mut self) -> Result<Self::LocalServiceSpawn> {
    Ok(Blocker)
  }
  #[inline]
  fn init_local_channel_in(&mut self) -> Result<Self::LocalServiceChannelIn> {
    Ok(NoChannel)
  }
  #[inline]
  fn init_local_service(me : Self::PeerRef, with : Option<Self::PeerRef>) -> Result<Self::LocalService> {
    Ok(DefLocalService{
      from : me,
      with : with,
    })
  }
));
```

For this kind of configuration, the service must always be call with an initial command input, the service must be restarted when its handle indicates that the service is finished. 

At the time, I've been a bit lazy on writing restart service code, truth is I am not really convinced by its usefulness (the number of iteration criterion is not really good, some time limit may be better). Still, it is done in for this case, local service spawning from a read service spawning from mainloop service initiated from mydhtconf.


Conclusion
==========

There is still 3 threads (global service, peerstore service and the mainloop), two being the same use case as Api service (single service from mainloop) and the last required to test on multiple peers without having to change the test method.

SRef seems interesting, yet it absolutely requires some macro derivability, it is an ok addition to the 'Send' marker trait.
I considered removing all 'Send' dependant implemention and keeping only the 'SRef' ones, plus maybe changing service base type to always use SRef variants (Send variants becoming useless). Obviously ArcRef and CloneRef SRef implemetation replacing the previous use case (for all thread we only need the kind of sref constrainted config that was describe for running RcRef over threads).  
https://github.com/rust-lang/rust/issues/42763 seems pretty bad with this SRef abstraction, same with almost all spawner, but I think it needs to be checked.  
If putting Service in its own crate, Clone and Send constrained implementation of spawner and channels are still interesting to avoid global SRef usage.

