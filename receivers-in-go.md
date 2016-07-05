# Neither self nor this: Receivers in Go

When getting started with Go, there is a strong temptation to bring baggage from
your previous language. It’s a heuristic which is usually helpful, but sometimes
counter-productive and inevitably results in regret.

Go does not have classes and objects, but it does have types that we can make
many instances of. Further, we can attach methods to these types and they
kind-of start looking like the classes we’re used to. When we attach a method
to a type, the receiver is the instance of the type for which it was called.


## Quick refresher on Go structs

```go
type Server struct {
    ...
}

func (srv *Server) Close() error {
    ...
}

func (srv Server) Name() string {
    ...
}
```

Every time I start writing somekind of server, it starts out looking like this.  
It's a type that holds information about our server, like a `net.Listener` 
socket, and it has a `Close()` method that shuts down the server. Easy enough.

The *receiver* of the `Close()` method is the `(s *Server)` part. This says that 
inside of the `Close()` method declaration, the scope will have an `s` variable 
that is a reference to the instance of the `Server` that it's being called on.  
That is:

```go
myServer := &Server{}
myServer.Close()
```

In this case, the `s` that is referenced inside of the `myserver.Close()` is 
effectively the same variable as `myserver`. They're both references to the same 
`Server` instance.

## Naming the receiver

In a lot of Go code, it's common to use the first letter or a short abbreviation 
as the name of the receiver. If the name of the struct is `Server`, we'll 
usually see `s` or `srv` or even `server`. All of these are fine.

But why not `self` or `this`? Coming from Python, or Ruby, or JavaScript, it's 
tempting to do something like:

```go
func (this *Server) Close() error {
    ...
}
```

That's one less decision to make every time we declare a struct. *All* of our 
methods could use the same receiver. Any time we see `this` in the code, we'll 
*know* that we're talking about the receiver, not some random local variable. It 
will be GREAT!.. or will it?

## Facts about methods and receivers

While we can call a method on a type instance and get the receiver implicitly, 
it can also be called explicitly:

```go
myServer := &Server{}
Server.Name(myServer)
(*Server).Close(&myServer)
```

These functions can be passed around as references just like any other function, 
with the implicit receiver or without:

```go
withReceiver := myServer.Name
without := myServer.Name
```

Receivers can be passed by reference or passed by value.

```go
func (byValue Server) Hello() { ... }
func (byReference *Server) Bye() { ... }
```

## How are structs born?

Taking a step back for a moment, let's look at how we end up with a struct like 
`Server`. Probably, we wrote some code like this:

```go
clients = []net.Conn{}
srv, err := net.Listen("tcp", ":2000")
if err != nil { ... }

for {
    conn, err := srv.Accept()
    if err != nil { ... }

    clients = append(clients, conn)
    for c := range clients {
        // Do something interesting, like send the latest client manifest to 
        // everyone
    }
}

srv.Close()
```

We start a listener server that accepts connections, keeps a list of clients, 
and does something per new client. Soon, we want to expand the logic around what 
happens every time a new client joins, maybe add helpers for adding and removing 
clients. Also would be great to control the state of the server from outside of 
itself--maybe force-close it from another goroutine.

Quickly it becomes clear that we want to something like this instead:

```go
srv, err := StartServer(":2000")
if err != nil { ... }

// Somewhere else:
srv.Close()
```

All the pieces of looping over new connections, keeping track of them, doing 
something relative to the state of the server, keeping internal goroutines safe 
from each other--it should be embedded in a container that we can control from 
the outside.


## Reshaping our code

As we start refactoring our code, we take a chunk of code that is already 
functional and put it in another context where it allows for more flexibility 
towards a higher-level goal.

The first step of taking flat code and embedding it into a struct method is 
usually fine regardless of what the receiver is named. At least in the 
beginning.

Later, once we start moving pieces of code *between* levels abstractions or from 
higher levels of abstraction into a lower level, this is when consistently 
well-named receivers make a huge difference.

Imagine taking this snippet from a higher-level container like `Room` which 
holds groups of users in a server, and moving it up or down one level:

```go
func (room *Room) Announce() {
    srv := room.Server()
    for _, c := range srv.Clients() {
        // Send announcement to all clients about a new room
        c.Send(srv.RenderAnnouncement(room))
    }
}

// Moved between...

func (srv *Server) AddRoom(room *Room) {
    for _, c := range srv.Clients() {
        // Send announcement to all clients about a new room
        c.Send(srv.RenderAnnouncement(room))
    }
}
```

This is a great little pattern to keep everything working despite moving between 
layers of abstraction. Note how the inner code stays identical and all we're 
doing is adding a little extra context outside of it.

On the other hand:

```go
func (this *Room) Announce() {
    srv := this.Server()
    for _, c := range srv.Clients() {
        // Send announcement to all clients about a new room
        c.Send(srv.RenderAnnouncement(this))
    }
}

// Moved between...

func (this *Server) AddRoom(room *Room) {
    for _, c := range this.Clients() {
        // Send announcement to all clients about a new room
        c.Send(this.RenderAnnouncement(room))
    }
}
```

When using `this`, there is confusion about whether we're referring to the 
server or the room as we're moving the code between.

```diff
-       c.Send(this.RenderAnnouncement(room))
+       c.Send(srv.RenderAnnouncement(this))
```

Yes, this is going to cause some bugs that hopefully the compiler catches (or 
maybe not, if the interfaces happen to be compatible). More importantly, it 
makes moving code around more tedious.


## Should receivers be special?

The suggested strategy for naming Go receivers is the same strategy for naming 
normal local variables. If they're named similarly, then these code blocks can 
be moved wholesale between layers of abstraction with minimal hassle.

On the other hand, by naming receivers as `this` or `self`, we're actually 
making receivers *special* in a way that is counter-productive. Imagine naming 
every local variable with the same name, all the time, regardless of what it 
represents? A scary thought.


## Advanced naming technique


