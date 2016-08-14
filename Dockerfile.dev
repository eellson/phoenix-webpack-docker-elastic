# Use elixir base image
FROM elixir:1.3

# Install node
RUN curl -sL https://deb.nodesource.com/setup_5.x | \
    bash - && apt-get install -y nodejs inotify-tools

# Install Elixir Deps
WORKDIR /app
ADD mix.* ./
RUN mix local.rebar --force
RUN mix local.hex --force
RUN mix deps.get

# Install Node Deps
WORKDIR /tmp
ADD package.json ./
RUN mkdir -p deps
RUN cp -a /app/deps/phoenix ./deps/phoenix
RUN cp -a /app/deps/phoenix_html ./deps/phoenix_html
RUN npm install

# Install app
WORKDIR /app
ADD . .
RUN cp -a /tmp/node_modules .
RUN mix compile
