# Req

A work-in-progress HTTP client.

# Features

  * Extensibility via request, response, and error steps

  * Automatic body decompression (via `decompress/2` step)

  * Automatic body decoding (via `decode/2` step)

  * Retries on errors (via `retry/2` step)

## High-level API

```elixir
Req.get!("https://api.github.com/repos/elixir-lang/elixir").body["description"]
#=> "Elixir is a dynamic, functional language designed for building scalable and maintainable applications"
```

## Low-level API

Under the hood, Req works by passing a request through a series of steps.

The request struct, `%Req.Request{}`, initially contains data like HTTP method,
request headers, etc. You can also add request, response, and error steps to it.

Request steps are used to refine the data that will be sent to the server.

Response steps are used to refine the response we got from the server.

Error steps are executed after we got an error from the socket.

Calling `Req.run/1` on the request runs the request through the steps.

```elixir
Req.build(:get, "https://api.github.com/repos/elixir-lang/elixir")
|> IO.inspect()
|> Req.add_request_steps([
  &Req.default_headers/1
])
|> Req.add_response_steps([
  &Req.decode/2
])
|> Req.run()
#=> {:ok, %{body: %{"description" => "Elixir is a dynamic," <> ...}, ...}, ...}
```

We can also build more complex flows like returning a response from a request step
or an error from a response step. We will explore those complex flows next.

### Request steps

Request step is a function that accepts a `request` and returns one of the following:

  * A `request`

  * A `{request, response_or_error}` tuple. In that case no further request steps are executed
    and the return value goes through response or error steps

```elixir
def default_headers(request) do
  update_in(request.headers, &[{"user-agent", "req/0.1.0-dev"} | &1])
end

def read_from_cache(request) do
  case ResponseCache.fetch(request) do
    {:ok, response} -> {request, response}
    :error -> request
  end
end
```

### Response and error steps

A response step is a function that accepts a `request` and a `response` and returns one of the
following:

  * A `{request, response}` tuple

  * A `{request, exception}` tuple. In that case, no further response steps are executed but the
    exception goes through error steps

Similarly, an error step is a function that accepts a `request` and an `exception` and returns one
of the following:

  * A `{request, exception}` tuple

  * A `{request, response}` tuple. In that case, no further error steps are executed but the
    response goes through response steps

```elixir
def decode(request, response) do
  case List.keyfind(response.headers, "content-type", 0) do
    {_, "application/json" <> _} ->
      {request, update_in(response.body, &Jason.decode!/1)}

    _ ->
      {request, response}
  end
end

def log_error(request, exception) do
  Logger.error(["#{request.method} #{request.url}: ", Exception.message(exception)])
  {request, exception}
end
```

### Halting

Any step can call `Req.Request.halt/1` to halt the pipeline. This will prevent any further steps
from being invoked.

```elixir
def circuit_breaker(request) do
  if CircuitBreaker.open?() do
    {Req.Request.halt(request), RuntimeError.exception("circuit breaker is open")}
  else
    request
  end
end
```

## License

Copyright (c) 2021 Wojtek Mach

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at [http://www.apache.org/licenses/LICENSE-2.0](http://www.apache.org/licenses/LICENSE-2.0)

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
