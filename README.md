# Cloudfoundry Phoenix
This is a placeholder for now with some problems I've run into deploying phoenix to cloud foundry. I intend to work on these problems and make deployment easier. Also I intend to write up a blog post to help others with common problems.

Be sure to always keep `cf logs <app-name>` running when deploying your new app.

## Known issues

### `prod.secrets.exs` is not available

You may notice:
```
 ERR     ** (Code.LoadError) could not load /tmp/app/config/prod.secret.exs
```
We don't need to use the secrets file on CF. Instead you can use an env var.

Firstly setup a secure secret key in an env var:

```
cf set-env SECRET_KEY_BASE <something-secure>
```

Then edit the prod config like so:

```diff
--- a/config/prod.exs
+++ b/config/prod.exs
@@ -63,4 +63,6 @@ config :logger, level: :info

 # Finally import the config/prod.secret.exs
 # which should be versioned separately.
-import_config "prod.secret.exs"
+
+config :my_app, MyApp.Endpoint,
+  secret_key_base: System.get_env("SECRET_KEY_BASE") || "abc123"
```


### Database URL cannot be accessed in VCAP_SERVICES
You will need to manually set some env var for this:

In order to find what it should be take a look at the VCAP_SERVICES:

```
cf env <app-name>
```

Then find the credentials in the output and update the env vars you need.

```
cf set-env DATABASE_URL <database-url>
```

and reference it in the config using like so:

```
config :my_app, MyApp.Repo,
  adapter: Ecto.Adapters.Postgres,
  url: System.get_env("DATABASE_URL"),
  pool_size: 20
```

### App URL cannot be accessed in VCAP_APPLICATION
You will need to manually set some env var for this:

```
cf set-env APP_URL myapp.cfapps.io
```

and reference it in the config using `System.get_env("APP_URL")` like so:

```diff
--- a/config/prod.exs
+++ b/config/prod.exs
@@ -13,7 +13,7 @@ use Mix.Config
 # which you typically run after static files are built.
 config :my_app, MyApp.Endpoint,
   http: [port: {:system, "PORT"}],
-  url: [host: "example.com", port: 80],
+  url: [host: System.get_env("APP_URL"), port: 80],
   cache_static_manifest: "priv/static/manifest.json",
   server: true
```

### Uploading tonnes of files
Add the following to a `.cfignore` file.

```
/_build
/db
/deps
/*.ez
erl_crash.dump
/node_modules
/rel
```

### Static assets are not available

You may notice:

```
[error] Could not find static manifest at "/home/vcap/app/_build/prod/lib/drivethrough/priv/static/manifest.json". Run "mix phoenix.digest" after building your static files or remove the configuration from "config/prod.exs."
```

Assuming you are using the `.cfignore` I referred to above you can build your assets locally before pushing to cloud foundry. Try the following to deploy:

```
MIX_ENV=prod mix phoenix.digest && cf push
```

If this doesn't fix the problem make sure that you don't have `/priv/static` in your `.cfignore` file.
