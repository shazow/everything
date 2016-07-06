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

The *receiver* of the `Close()` method is the `(srv *Server)` part. This says 
that inside of the `Close()` method declaration, the scope will have an `s` 
variable that is a reference to the instance of the `Server` that it's being 
called on.  That is:

```go
myServer := &Server{}
myServer.Close()
```

In this case, the `s` that is referenced inside of the `myServer.Close()` is 
effectively the same variable as `myServer`. They're both references to the same 
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
without := Server.Name
```

Receivers can be passed by reference or passed by value.

```go
func (byValue Server) Hello() { ... }
func (byReference *Server) Bye() { ... }
```

This is to illustrate that struct methods in Go are merely thin sugar over 
traditional C-style struct helper declarations, an equivalent C method might 
look like this:

```c
void server_close(server *srv) { ... }
```

Go helps by namespacing the methods and implicitly passing the receiver when 
called on an instance, but otherwise there is very little magic going on.

In other languages where `this` and `self` is a thing (Python, Ruby, JavaScript, 
and so on) it's a much more complicated situation. These are not vanilla local 
variables wearing fancy pants. The thing we might expect `this` to represent 
inside of a method could actually represent something very different once 
inheretance or metaclasses had their way. In effect, it might not make any sense 
to give contextual names to `self` in Python, but it definitely makes sense in 
Go.

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

If we agree that contextually named receivers are meaningful, then maybe we can 
utilize this opportunity for an even greater advantage.

What if we named our receivers based on the interface that they're implementing 
(if any)? Let's say we add `io.Writer` and `io.Reader` interfaces to our Server:

```go
func (w *Server) Write(p []byte) (n int, err error) {
    // Send p to all clients
}

func (r *Server) Read(p []byte) (n int, err error) {
    // Receive data from all clients
}
```
Maybe we also want to add the `http.Handler` interface to provide a dashboard 
for our server.

```go
func (handler *Server) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // Render dashboard
}
```

There are a few benefits to doing it this way:

The receivers enhance the self-documenting nature of our code. It becomes 
clearer which interface each method is attempting to implement.

If we were implementing these wrappers outside of the `Server` struct, we would 
likely be using similarly named variabls for intermediate code. By naming the 
receiver in a corresponding way, it makes it easier to move the code inline 
without changing much.

Further, as we add interface-specific functionality, it's likely that we'll need 
to add more fields to our struct to manage various related state. The code can 
look more meaningful when a read buffer is being accessed from a `r` receiver to 
imply that its purpose is specifically for this functionality rather than it 
being a more general buffer for the server as a whole.


## Name of the Receiver

Carefully naming our receivers can have lots of tangible benefits, especially as 
our project grows and code gets moved around. It can make our inner method code 
much more readable without needing to be aware of which struct it's embedded 
into. It can even add an opportunity to indicate higher-level layout of our 
struct's interface implementation.

Picking a fixed name for all receivers like `self` can have negative effects 
like mixing up context when code gets moved around. It removes a decision during 
writing, but the cost creeps up when we go back to read the code or refactor it.

Go forth and give your receivers the names they deserve.


