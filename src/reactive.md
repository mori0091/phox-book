# [T.B.D.] Reactive programming

## GUI applications

(T.B.D.)

---

## Headless Service Agents

```rust , ignore
// `st` : type of service's state
// `ev` : type of events / messages
// `a`  : type of resulting value
// `Action ev`   : type of command request to the run-time system or other entities.
// `WatchSet ev` : type of subscription for watching the run-time system or other entities.
type ServiceST st ev a
  = Continue st (Action ev) (WatchSet ev)
  | Done a
;

type Service st ev a = @{
  /// `init` is called by the run-time system when the service is started.
  init   : ServiceST st ev a,
  /// `update` is called by the run-time system when a event occurred.
  update : st -> ev -> ServiceST st ev a,
};

/// Constructs service instance and execute it.
///  serve : ServiceST st ev a -> (st -> ev -> ServiceST st ev a) -> a
*let serve = \ini.\upd. {
  let s = Service @{ init = ini, update = upd };
  ::core::service:execute s
};
```

---

## Rough sketch of service-execution loop:

```rust , ignore
*let execute = \Service @{init, update}. {
    let ctx = proc! { ::core::service::new_context!() };
    let Done result = for
        init
        (\sv. match (sv) {
            Done _ => false,
            _      => true,
        })
        (\sv. match (sv) {
            Done _ => sv,
            Continue st act wset => {
                // submit action and watch-set, then wait for events / messages
                let msg = proc! { ::core::service::perform!(ctx, act, wset) };
                update st msg
            },
        })
    ;
    result
};
```
