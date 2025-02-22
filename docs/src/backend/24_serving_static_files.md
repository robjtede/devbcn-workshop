# Serving static files

In this section of the backend part of the workshop we'll learn how to **serve static files** with [Actix Web](https://actix.rs) and [Shuttle](https://shuttle.rs).

The main goal here is to serve the statics files present in a folder called `static`. 

So the API will serve `statics` in the root path `/` and the `API endpoints` in the `/api` path.

For this to happen we will need to refactor a little bit our `api-shuttle` main code.

## Shuttle dependencies

Read the [Shuttle documentation for static files](https://docs.shuttle.rs/resources/shuttle-static-folder).

Some of the **caveats** that you will find explained there **will apply to us** as we are using a workspace, but let's start from the beginning.

Let's add the `shuttle-static-folder` and the [actix-files](https://docs.rs/actix-files/latest/actix_files/) dependencies to our `api-shuttle` crate.

```toml
[dependencies]
# static
actix-files = "0.6.2"
shuttle-static-folder = "0.20.0"
```

## Serving the static files

Now, let's refactor our `main.rs` file to serve the static files.

Add this parameter to the `actix_web` function:

```rust
 #[shuttle_static_folder::StaticFolder(folder = "static")] static_folder: PathBuf,
```

And then, let's modify our `ServiceConfig` to serve static files in the `/` path and the API in the `/api` path:

```diff
- cfg.app_data(film_repository)
-   .configure(api_lib::health::service)
-   .configure(api_lib::films::service::<api_lib::film_repository::PostgresFilmRepository>);
+ cfg.service(
+     web::scope("/api")
+         .app_data(film_repository)
+         .configure(api_lib::health::service)
+         .configure(
+             api_lib::films::service::<api_lib::film_repository::PostgresFilmRepo+ sitory>,
+         ),
+ )
+ .service(
+     actix_files::Files::new("/", static_folder)
+         .show_files_listing()
+         .index_file("index.html"),
+ );
```

~~~admonish tip title="Final Code" collapsible=true
```rust
use actix_web::web::{self, ServiceConfig};
use shuttle_actix_web::ShuttleActixWeb;
use shuttle_runtime::CustomError;
use sqlx::Executor;
use std::path::PathBuf;

#[shuttle_runtime::main]
async fn actix_web(
    #[shuttle_shared_db::Postgres()] pool: sqlx::PgPool,
    #[shuttle_static_folder::StaticFolder(folder = "static")] static_folder: PathBuf,
) -> ShuttleActixWeb<impl FnOnce(&mut ServiceConfig) + Send + Clone + 'static> {
    // initialize the database if not already initialized
    pool.execute(include_str!("../../db/schema.sql"))
        .await
        .map_err(CustomError::new)?;

    let film_repository = api_lib::film_repository::PostgresFilmRepository::new(pool);
    let film_repository = web::Data::new(film_repository);

    let config = move |cfg: &mut ServiceConfig| {
        cfg.service(
            web::scope("/api")
                .app_data(film_repository)
                .configure(api_lib::health::service)
                .configure(
                    api_lib::films::service::<api_lib::film_repository::PostgresFilmRepository>,
                ),
        )
        .service(
            actix_files::Files::new("/", static_folder)
                .show_files_listing()
                .index_file("index.html"),
        );
    };

    Ok(config.into())
}
```
~~~

You will get a **runtime error**:

```bash
[Running 'cargo shuttle run']
    Building /home/roberto/GIT/github/robertohuertasm/devbcn-dry-run
   Compiling api-shuttle v0.1.0 (/home/roberto/GIT/github/robertohuertasm/devbcn-dry-run/api/shuttle)
    Finished dev [unoptimized + debuginfo] target(s) in 9.00s
2023-07-02T18:49:07.514534Z ERROR cargo_shuttle: failed to load your service error="Custom error: failed to provision shuttle_static_folder :: StaticFolder"
[Finished running. Exit status: 1]
```

That's mainly because the static folder doesn't exist yet.

Create a folder called `static` in the `api-shuttle` crate and add a file called `index.html` with this content:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Hello Shuttle</title>
  </head>
  <body>
    Hello Shuttle
  </body>
</html>
```

Now if you browse to [http://localhost:8000](http://localhost:8000) you should be able to see the `index.html` file.

```admonish warning
Remember that we have changed the path for the API to `/api` so you will need to change that too in your `api.http` file or Postman configuration.
```

## Ignoring the static folder

As the `static` folder will be generated by the `frontend`, we don't want to commit it to our repository.

Add this to the `.gitignore` file:

```bash
# Ignore the static folder
static/
```

Now, to solve a [Shuttle issue affecting static folders in workspaces](https://docs.shuttle.rs/resources/shuttle-static-folder), we need to create a `.ignore` file in the root folder with the following content:

```bash
!static/
```

Commit your changes:

```bash
git add .
git commit -m "serve static files"
```

Now, in order to deploy to the cloud and avoid having issues with the `static` folder not being found (remember there's currently an issue in the Shuttle static folder implementation), copy the `static` folder to the root of your project and deploy:

```bash
cargo shuttle deploy
```
