# Paraguas

**Release Config:**

paraguas/mix.exs add

```elixir
defp deps do
  [{:distillery, "~> 1.5", runtime: false}]
end
```


run

```shell
mix deps.get
mix release.init. #This adds a rel directory with a config.exs file.
```

Notes: paraguas/apps/phoenix_app/config/prod.exs

```elixir
config :phoenix_app, PhoenixAppWeb.Endpoint,
  load_from_system_env: true,
  http: [:inet6, port: {:system, "PORT"}],
  url: [host: "localhost", port: 80],
  cache_static_manifest: "priv/static/cache_manifest.json",
  server: true,
  root: ".",
  version: Application.spec(:phoenix_app, :vsn)
```

Dockerfile

```Dockerfile
# Alias this container as builder:
FROM bitwalker/alpine-elixir-phoenix as builder

WORKDIR /paraguas

ENV MIX_ENV=prod

# Umbrella
COPY mix.exs mix.lock ./
COPY config config

# Apps
COPY apps apps
RUN mix do deps.get, deps.compile

# Build assets in production mode:
WORKDIR /paraguas/apps/phoenix_app/assets
RUN npm install && ./node_modules/brunch/bin/brunch build --production

WORKDIR /paraguas/apps/phoenix_app
RUN MIX_ENV=prod mix phx.digest

WORKDIR /paraguas
COPY rel rel
RUN mix release --env=prod --verbose

### Release

FROM alpine:3.6

RUN apk upgrade --no-cache && \
    apk add --no-cache bash openssl
    # we need bash and openssl for Phoenix

EXPOSE 4000

ENV PORT=4000 \
    MIX_ENV=prod \
    REPLACE_OS_VARS=true \
    SHELL=/bin/bash

WORKDIR /paraguas

COPY --from=builder /paraguas/_build/prod/rel/paraguas/releases/0.1.0/paraguas.tar.gz .

RUN tar zxf paraguas.tar.gz && rm paraguas.tar.gz

RUN chown -R root ./releases

USER root

CMD ["/paraguas/bin/paraguas", "foreground"]
```

build image:
```bash
docker build -t paraguas:0.1.0 .
```

run on CLI:

```bash
docker run --rm -ti \
             -p 4000:4000 \
             -e COOKIE=a_cookie \
             -e BASIC_AUTH_USERNAME=username \
             -e BASIC_AUTH_PASSWORD=password \
             -e BASIC_AUTH_REALM=realm \
             paraguas:0.1.0
```
