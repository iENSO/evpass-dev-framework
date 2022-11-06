Introduction
============

iENSO IPC C++ async library to communicate between services.

Description
===========

This implementation of the iENSO IPC protocol is written in C++, it's async and [`asio-based`](https://think-async.com/Asio/).
The library methods aren't thread safe and it's intended that user will use single-threaded `asio` event loop. All library methods should be called from this event loop.

### IPC Primitives
The library supports following IPC primitives:
- `method` - a `method` can be registered on server side and it can be called by client side using `call(method, args)` function. The call accepts `args` which is a JSON value. The `call` returns a result which is also a JSON value.
- `state` - a `state` can be registered on server side. In fact it's represented by a JSON value. A `state` should have an initial value which should be returned immediately as an initiation. A server can publish the `state` updates whenever it's necessary. A client can subscribe to a state using `subscribe(state, path)` function. The function returns receiver which should be used to receive the `state` updates. The `path` argument (JSON pointer) is optional and can be used to receive only a concrete fragment of a state.
- `signal` - a `signal` is in fact a state which doesn't have initial value.

### Introspection
The library has builtin introspection support. The introspection implemented as a builtin `method` `GetIntrospection` which accepts `null` JSON value as an `args`.

The introspections returns all primitives (`methods`, `states`, `signals`) which the server supports in the following format:
```json
{
  "methods": [
    { "name": "methodA", "args_schema": ARGS_JSON_SCHEMA_A },
    { "name": "methodB" },
    ...
  ],
  "signals": [ "signalA", "signalB", ... ],
  "states": [ "stateA", "stateB", ... ],
}
```

Client's workflow examples
=================

The below examples shows how to setup an environment for client and make a remote call to a service:
```c++
#include <ienso/ipc/client.hpp>

#include <asio.hpp>
#include <asio/co_spawn.hpp>

#include <iostream>

using namespace ienso;

asio::io_context ctx {};

nlohmann::json call_args
{
  { "ssid", "AP" },
  { "psk", "12345678" }
};

void propagate_exceptions(std::exception_ptr ex)
{
  if (ex)
    std::rethrow_exception(ex);
}

asio::awaitable<void> coro()
{
  auto client = co_await ipc::make_client(
    ctx, "com.ienso.nm-controller", asio::use_awaitable);

  client.start(asio::detached);
  auto result = co_await client.call("ConnectToWiFi", call_args, asio::use_awaitable);
  std::cout << "call result: " << result.dump() << std::endl;
}

int main()
{
  asio::co_spawn(ctx, &coro, &propagate_exceptions);
  ctx.run();
}

```

by replacing `coro()` function of the above we can demonstrate how to use `state`:
```c++
...
#include <ienso/api/camera-control.hpp>
using namespace ienso::api::camera_control;
...
asio::awaitable<void> coro()
{
  auto client = co_await ipc::make_client(ctx, bus_name, asio::use_awaitable);

  client.start(asio::detached);
  auto rcv = co_await client.subscribe(state::settings, asio::use_awaitable);
  while (true)
  {
    auto state = co_await rcv.async_receive(asio::use_awaitable);
    std::cout << "state: " << state.dump() << std::endl;
  }
}
```

Dependencies
============

* Asio:

  - URI: https://github.com/chriskohlhoff/asio
  - tag: asio-1-18-2

* googletest:

  - URI: git://github.com/google/googletest.git
  - version: 1.10.0

* nlohmann-json

  - URI: git://github.com/nlohmann/json.git
  - version: 3.9.1

* iENSO async

  - URI: git://git@github.com/iENSO/async.git
  - version: ienso/main/dev