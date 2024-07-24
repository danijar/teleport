# 🌀 Teleport

Efficiently send large arrays across machines.

## Overview

Teleport provides a `Server` that you can bind functions to and a `Client` that
can call the functions and receive their results. The function arguments and
return values are both trees of **Numpy arrays**. The data is sent efficiently
without serialization to maximize throughput.

## Installation

```sh
pip install git+https://github.com/danijar/teleport.git
```

## Example

This example runs the server and client in the same Python program using
subprocesses, but they could also be separate Python scripts running on
different machines.

```python
def server():
  import teleport
  server = teleport.Server('tcp://*:2222')
  server.bind('add', lambda data: {'result': data['foo'] + data['bar']})
  server.bind('msg', lambda data: print('Message from client:', data['msg']))
  server.run()

def client():
  import teleport
  client = teleport.Client('tcp://localhost:2222')
  client.connect()
  future = client.add({'foo': 1, 'bar': 1})
  result = future.result()
  print(result)  # {'result': 2}
  client.msg({'msg': 'Hello World'})

if __name__ == '__main__':
  import teleport
  server_proc = teleport.Process(server, start=True)
  client_proc = teleport.Process(client, start=True)
  client_proc.join()
  server_proc.terminate()
```

## Features

Several productivity and performance features are available:

- **Request batching:** The server can batch requests together so that the user
  function receives a dict of stacked arrays and the function result will be
  split and sent back to the corresponding clients.
- **Multithreading:** Servers can use a thread pool to process multiple
  requests in parallel. Optionally, each function can also request its own
  thread pool to allow functions to block (e.g. for rate limiting) without
  blocking other functions.
- **Async clients:** Clients can send multiple overlapping requests and wait
  on the results when needed using `Future` objects. The maximum number of
  inflight requests can be limited to avoid requests building up when the
  server is slower than the client.
- **Error handling:** Exceptions raised in server functions are reported to the
  client and raised in `future.result()` or, if the user did not store the
  future object, on the next request. Worker exception can also be reraised in
  the server application using `server.check()`.
- **Heartbeating:** Clients can send ping requests when they have not received
  a result from the server for a while, allowing to wait for results that take
  a long time to compute without assuming connection loss.
- **Concurrency:** `Thread` and `Process` implementations with exception
  forwarding that can be forcefully terminated by the parent, which Python
  threads do not natively support. Stoppable threads and processes are also
  available for coorperative shutdown.
- **GIL load reduction:** The `ProcServer` behaves just like the normal
  `Server` but uses a background process to batch requests and fan out results,
  substantially reducing GIL load for the server workers in the main process.

## Questions

If you have a question, please [file an issue][issue].

[issue]: https://github.com/danijar/teleport/issues
