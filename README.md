# Phoenix/Webpack/Docker/Elastic Beanstalk starter pack

This project aims to create a simple and repeatable solution for getting up and
running with dev and production [Phoenix](https://www.phoenixframework.org)
environments.

This approach swaps out brunch for [webpack](https://webpack.github.io/) (with
sass and ES2015 support), makes use of [Docker](https://www.docker.com/) to
allow devs to get their environment up and running easily, and is ready for
deployment on [Elastic Beanstalk](http://docs.aws.amazon.com/elasticbeanstalk/latest/dg/Welcome.html).

## Getting started

First look at the [Gotchas](#gotchas) section for common issues.

Now clone this repo:

`$ git clone https://github.com/eellson/phoenix_starter.git`

## Using in development

### Updating existing project

Copy files into project directory:

```bash
$ cd phoenix-webpack-docker-elastic
$ cp -a .ebextensions/ path_to_project/
$ cp .dockerignore \
     Docker* \
     docker-compose.yml \
     webpack.config.js path_to_project/
```

Ensure you have all the `dependencies` and `devDependencies` from our
package.json in yours, and remove any brunch dependencies. If you're using the
default package.json (you've not added/removed any libs) you can just drop ours
in, and the diff should look something like this:

```diff
 {
   "repository": {},
+  "scripts": {
+    "start": "webpack --watch-stdin --progress --color",
+    "compile": "NODE_ENV=production webpack -p"
   },
   "dependencies": {
-    "brunch": "^1.8.5",
-    "babel-brunch": "^5.1.1",
-    "clean-css-brunch": ">= 1.0 < 1.8",
-    "css-brunch": ">= 1.0 < 1.8",
-    "javascript-brunch": ">= 1.0 < 1.8",
-    "uglify-js-brunch": ">= 1.0 < 1.8"
+    "phoenix": "file:deps/phoenix",
+    "phoenix_html": "file:deps/phoenix_html"
+  },
+  "devDependencies": {
+    "babel-core": "^6.13.2",
+    "babel-loader": "^6.2.4",
+    "babel-preset-es2015": "^6.13.2",
+    "bootstrap-sass": "^3.3.7",
+    "copy-webpack-plugin": "^3.0.1",
+    "css-loader": "^0.23.1",
+    "extract-text-webpack-plugin": "^1.0.1",
+    "file-loader": "^0.9.0",
+    "node-sass": "^3.8.0",
+    "sass-loader": "^4.0.0",
+    "style-loader": "^0.13.1",
+    "url-loader": "^0.5.7",
+    "webpack": "^1.13.1"
 }
```

Update database configuration to point to postgres host in `docker-compose.yml`.
You'll want to make this change in any environments you want to run using
docker-compose:

```diff
config :name_of_project, PhoenixStarter.Repo,
  adapter: Ecto.Adapters.Postgres,
  username: "postgres",
  password: "postgres",
  database: "phoenix_starter_dev",
-  hostname: "localhost",
+  hostname: "postgres",
  pool_size: 10
```

We assume you want webpack, not brunch, so need to update the watchers
instruction in `config/dev.exs` for this. Similarly, if you've got a
`brunch-config.yml` floating around, you can likely get rid:

```diff
config :name_of_project, PhoenixStarter.Endpoint,
  http: [port: 4000],
  debug_errors: true,
  code_reloader: true,
  check_origin: false,
-  watchers: [node: ["node_modules/brunch/bin/brunch", "watch", "--stdin",
-                    cd: Path.expand("../", __DIR__)]]
+  watchers: [npm: ["start", cd: Path.expand("../", __DIR__)]]
```

We also assume you'll want to use sass, compiles to css by webpack, so update
your original plain css:

```bash
$ mv web/static/css/app.{css,scss}
```
### Building docker environment

Build Docker environment:

```bash
$ docker-compose build
```

Setup database:

```bash
$ docker-compose run web bash
> mix do ecto.create, ecto.migrate # run this in docker shell
```

### Running

Run app:

```bash
$ docker-compose up
```

You can run tests in web container with:

```bash
$ docker-compose run web mix test
```

## Setting up production

This setup should be good to go for an elastic beanstalk deployment, with a
couple of tweaks. We assume you've got the `aws` and `eb` clis configured.

### Updating existing project

We use git for our eb deployments, but don't keep our production secrets in
source control, instead they live on the box. We inject these into the Docker
container at build time, so they can be present in your compiled production
config. Anything you expect to read from here should use `System.get_env/1`.

We also need to tweak the production config slightly. Once done you should have
something like:

```elixir
config :name_of_project, NameOfProject.Endpoint,
  http: [port: {:system, "PORT"}, compress: true],
  url: [scheme: "http", host: System.get_env("HOST"), port: {:system, "PORT"}],
  secret_key_base: System.get_env("SECRET_KEY_BASE"),
  code_reloader: false,
  cache_static_manifest: "priv/static/manifest.json",
  server: true

config :name_of_project, NameOfProject.Repo,
  adapter: Ecto.Adapters.Postgres,
  username: System.get_env("RDS_USERNAME"),
  password: System.get_env("RDS_PASSWORD"),
  database: System.get_env("RDS_DATABASE"),
  hostname: System.get_env("RDS_HOSTNAME"),
  port: System.get_env("RDS_PORT") || 5432,
  pool_size: 20,
  ssl: true
```

### Creating production database

Next, create an RDS instance. It's best to
[keep this outside](http://www.michaelgallego.fr/blog/2013/10/26/do-not-associate-rds-instance-with-beanstalk-environment/)
of the elastic beanstalk environment and manage separately. There are lots of
[options for this](http://docs.aws.amazon.com/cli/latest/reference/rds/create-db-instance.html),
but here's a sample:

```bash
$ aws rds create-db-instance \
	  --db-instance-identifier yourrdsinstanceidentifier \
	  --db-instance-class db.t2.micro \
	  --engine postgres \
	  --db-name your_db_name \
	  --allocated-storage 5 \
	  --master-username yourdbusername \
	  --master-user-password dbpass
```

This will take some time to spin up, we can use the cli to get a picture of the
status here. This simply returns when complete:

```bash
$ aws rds wait db-instance-available \
	  --db-instance-identifier yourrdsinstanceidentifier
```

We can now get the hostname for our db with:

```
$ aws rds describe-db-instances \
	  --db-instance-identifier yourrdsinstanceidentifier | grep Address
```

### Creating production app environment

This gives us all the info we need to create our eb environment. We can specify
most of the config options at this stage:

```bash
$ eb create environmentname \
     --envvars MIX_ENV=prod,SECRET_KEY_BASE=secret,PORT=4000,HOST=cname.elasticbeanstalk.com,RDS_USERNAME=yourdbusername,RDS_PASSWORD=dbpass,RDS_HOSTNAME=rdshostname,RDS_DATABASE=your_db_name,RDS_PORT=5432
```

Finally, we need to allow our eb environment access to the rds instance. There
is a good run through of the steps to do this in Amazon's documentation,
[here](http://docs.aws.amazon.com/elasticbeanstalk/latest/dg/AWSHowTo.RDS.html#d0e32700).

## Gotchas

### My app won't deploy because my config is incorrect, but I can't update my config because my app is in an invalid state! What can I do?

Sadly it's easy to get into this state when creating the production environment.
The simplest way to resolve this is to deploy a basic Dockerfile you know works,
(such as the example [here](http://docs.aws.amazon.com/elasticbeanstalk/latest/dg/docker-singlecontainer-deploy.html)
from amazon), set your config to the desired values, then redeploy with the
correct Dockerfile.

### Webpack isn't compiling assets on change/live-reload isn't working.

There is an issue (https://github.com/webpack/webpack-dev-server/issues/143)
with detecting file system changes when using the virtualbox driver. Using an
alternative driver, or a solution like
[Docker for Mac](https://docs.docker.com/engine/installation/mac/) should
resolve this.

### Webpack fails on production with "command not found"

Check your NODE_ENV. We need to install `devDependencies` in order to run
webpack etc on the server, so we should not set `NODE_ENV=production` (note we
set this explicitly when compiling assets for production).

## Acknowledgements

* This approach was heavily inspired by James Robert Somers' post [here](https://robots.thoughtbot.com/deploying-elixir-to-aws-elastic-beanstalk-with-docker)
on Giant Robots Smashing into Other Giant Robots.
* Strategy for writing ENV into Dockerfile on eb from Vladimir Zhukov's insights
[here](https://vladimirzhukov.com/insights-into-deploying-elixir-app-with-amazon-elastic-beanstalk-and-docker-846b31feca7f#.d8hqevkns).
* Webpack config heavily influenced by [Matthew Lehner](http://matthewlehner.net/using-webpack-with-phoenix-and-elixir/)
and [Andrew Stewart](https://andrew.stwrt.ca/posts/phoenix-assets-with-webpack/)'s
approaches.
