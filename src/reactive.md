# [T.B.D.] Reactive programming

- Reactive system (Event-driven / Reactive Programming)
  - Reactive State-Machine
  - Cannels and Capabilities
  - User-defined events / messages

- Software Circuit
  - Component := Context (ex. Output Portals) × Input Channel × StateMachine
  - Circuit := Component Network
  - Declarative Resource Constraints
  - Application := Instantiate and Run Circuit under the declared resource constraints.

---

## Channels and Send/Receive Capabilities

```rust , ignore
/// Communication channel with (re)configurable parameters
/// (e.g. buffer size, back pressure, etc.)
type Channel a = @{...};

/// Send/Receive capabilities.
type SendCap a = SendCap (Channel a);
type RecvCap a = RecvCap (Channel a);

/// send : SendCap a -> a -> ()
*let send = \sc.\msg. proc! {
    send!(sc, msg)
};

/// recv : RecvCap a -> a
*let recv = \rc. proc! {
    recv!(rc)
};

/// portal
///   : SendCap (req, resp -> ())
///  -> SendCap msg
///  -> (resp -> msg)
///  -> req
///  -> ()
*let portal = \sendreq.\sendmsg. {
    let call_out  = send sendreq;
    let call_back = send sendmsg;
    (\toMsg.\req. call_out (req, call_back << toMsg))
};

/// Receive Capability is an Iterator.
impl Iter RecvCap a {
    next = \inbox. Some (recv inbox, inbox);
};
```

## Software Circuit

**Client-Server Example**:
```rust , ignore
// === Echo Server definition ===
// (pseudo code)

type EchoReq
    = Echo ScalarString
    | EchoBye
    ;

type EchoServer = EchoServer;

impl Sink EchoServer (EchoReq, ScalarString -> ()) () {
    update = \self.\(req, replay). match (req) {
        Echo s => {
            reply s;
            self
        },
        EchoBye => {
            reply "Bye!";
            self
        },
    };
    finish = \self. ();
};

// === Echo Client definition ===
// (pseudo code)

type Msg
    = OnReceived ScalarString
    | ...
    ;

type EchoClient = @{
    echo : EchoReq -> (),
};

impl Sink EchoClient Msg () {
    update = \self.\msg. match (msg) {
        OnReceived s => {
            println s;
            self.echo EchoBye;
            self
        },
    };
    finish = \self. ();
};

// === Software Circuit definition ===
let newSystem = \conf. {
    // === channels ===
    let @{s, c} = conf : @{
        s : Channel (EchoReq, ScalarString -> ()),
        c : Channel Msg,
    };

    // === connections ===
    let e = portal (SendCap s) (SendCap c) OnReceived;

    // === entities ===
    let server = EchoServer;
    let client = EchoClient @{ echo = e };

    // === tasks ===
    // (Assume that `task expr` constructs a task abstraction)
    @[
        task {
            collect server (RecvCap s)
        },
        task {
            e (Echo "Hi!");
            collect client (RecvCap c)
        },
    ]
};

// === Instantiate the circuit w/ configuration ===
let system = newSystem @{
    // === channels config./constraints ===
    s = Channel @{...},
    c = Channel @{...},
    // === other config./constraints ===
    // ...
};

// === Execute the circuit instance ===
par system
```
