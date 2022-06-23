---
title: Write Simple Web Server with Actix and Sailfish
date: "2021-05-22"
featuredImage: "./actix.png"
---

### What is Actix?

Actix is a powerful, pragmatic, and extremely fast web framework for Rust.

We call `actix-web` a powerful and pragmatic framework. For all intents and purposes it’s a micro-framework with a few twists. If you are already a Rust programmer you will probably find yourself at home quickly, but even if you are coming from another programming language you should find `actix-web` easy to pick up.

Furthermore, **Actix is currently ranked among the top ten fastest frameworks in the world**. Rust is used to create Actix. As a result, it is self-evident that it is one of the fastest and most efficient web development frameworks. Take a look at some of the [benchmarks](https://www.techempower.com/benchmarks/).

### What is Sailfish?

Sailfish is a simple, small, and extremely fast template engine for Rust. These are some of its specialties

- Write a Rust code directly inside templates, supporting many Rust syntax (struct definition, closure, macro invocation, etc.)
- [Built-in filters](https://docs.rs/sailfish/latest/sailfish/runtime/filter/index.html)
- Minimal dependencies (<15 crates in total)
- Template rendering is always type-safe because templates are statically compiled.
- Syntax highlighting ([vscode](http://github.com/Kogia-sima/sailfish/blob/master/syntax/vscode), [vim](http://github.com/Kogia-sima/sailfish/blob/master/syntax/vim))

Furthermore, **this is one of the fastest template engines in Rust**, with lots of features. Take a look at some of the [benchmarks](https://github.com/djc/template-benchmarks-rs).

That’s it for introduction.

### So, What we are going to do in Actix and Sailfish?

We’ll make a simple Web application without using any CSS or JS. I tried to keep things as straightforward as possible. This tutorial will provide you with everything you need to get started as an Actix Web Developer.

Nothing complicated; we’ll simply create a Web App that can store a person’s name in a database (actually, there won’t be a database; instead, a separate module will act as one). On another page, we can see all the names of the people who have been saved and delete any of them. There is no designing and no AJAX.

So, in a nutshell, this is a CRUD Web App. You can found this project on [GitHub](https://github.com/ParampreetR/crud_sailfish).

### Requirements

- A minimal knowledge regarding how web works.
- Some practical knowledge of Rust.
- (Assume anything you already has)

### Basics

Create a new rust project with cargo. I will call this project `crud_sailfish`.

```bash
cargo new crud_sailfish
```

This is a basic setup of Actix. Below is the contents of `src/main.rs`

```rust
use actix_web::{web, App, HttpResponse, HttpServer, Responder};

async fn homepage() -> impl Responder {
	HttpResponse::Ok().body("Hello World")
}

#[actix_web::main]
async fn main() {
	let addr = "localhost:8080";
	let server = HttpServer::new(move || {
	App::new()
	  .route("/", web::get().to(homepage))
	})
	.bind(addr)
	.unwrap()
	.run();
	println!("Server live at http://{}", addr);
	server.await.unwrap();
}
```

Starting with main, main function is declared `async` and `#[actix web::main]` is needed to run `async` main function at runtime. We’ll need to host our web server and set up routes later. This is accomplished using the `HttpServer` struct, which accepts closure as an argument, and this closure contains an App struct that defines routes. There is only one route declared here, which is the homepage. `route()` second argument is something that has the `Route` trait. Simply, it should resemble `web::<Request Type>().to(<Handler Function>)`. Finally, we asked OS for a port to host our server on, then unwrapped and called run() to start our server and `.await` to wait for it to exit (which will never happen unless we explicitly press Ctrl + C).

The handler function is next, which is declared `async` for non-blocking behaviour (for example, one request does not block another), which is required by Actix and without which your application will fail to compile. It gives you something that has the `Responder` trait. This trait will be implemented on those objects that can act like or be converted to a response object, according to the actix-web documentation. So, in essence, we’ll return something that implements the `Responder` trait, for example, something that can be converted to a response. We used another struct `HttpResponse` in the body of the handler function to provide a `200` status code with the body of Hello World. `::Ok()` returns a status code of `200`, and the rest can be found [here](https://docs.rs/actix-web/3.3.2/actix_web/web/struct.HttpResponse.html).

This is all the needed for now. Rest I’ll explain as we code.

## Start building our Web Application

Firstly, this is our directory structure

```
.
├── Cargo.toml
├── src
│   ├── database.rs
│   └── main.rs
└── templates
    ├── home.stpl
    ├── list.stpl
    └── navbar.stpl
```

and our main function inside main file.

```rust
use actix_web::{web, App, HttpResponse, HttpServer, Responder};
mod database;
use database::*;
use std::sync::Mutex;

#[actix_web::main]
async fn main() {
  let addr = "localhost:8080";

	// We need some data across each requests. For example Database instance.
  let db = web::Data::new(Mutex::new(Db::new()));

  let server = HttpServer::new(move || {
    App::new()
      .app_data(db.clone())
      .route("/", web::get().to(homepage))
      .route("/add", web::post().to(add_to_persons))
      .route("/list", web::get().to(get_persons))
      .service(web::resource("/delete/{id}").route(web::get().to(delete_person)))
  })
  .bind(addr)
  .unwrap()
  .run();

  println!("Server live at http://{}", addr);
  server.await.unwrap();
}
```

Somethings may caught attention here but everything is straight forward. We use `actix_web::web::Data::new(Mutex::new(<data>));` and provide returned value to `App::new().app_data()`, when we want some data to be persistent across requests as each request will start a new thread instance, we need something like `Arc<>` with `Mutex<>` to share value across threads. Don't worry for now, this data will be available in handler functions. Here we also imported everything from database file, which mimic functionality of database. And, if you are confused about `.service()`, this is a way we define route when we want to retrieve something from entered URL like we are retrieving something follows URL `/delete/` with variable name of `id`, which again will available to our handler functions.

We will define handler function at last but for now, this is our template structs as required by sailfish in `main.rs`.

```rust
#[derive(TemplateOnce)]
#[template(path = "home.stpl")]
struct Home {}

#[derive(TemplateOnce)]
#[template(path = "list.stpl")]
struct List {
    persons: Vec<String>,
}
```

We need to derive `TemplateOnce` to our template struct. `#[derive(TemplateOnce)]` is required by sailfish. We need to tell sailfish to link our HTML file to our struct this is done by `#[template(path = "<html file>")]`. While `<html file>` is some file under `/templates` directory. In `List` struct we defined a member `persons`, this basically means we need to initialize struct with a vector of string and we can access this vector in our html file with `<% persons %>`.

Following are our HTML files with `stpl` extension

`templates/navbar.stpl`

```html
<ul>
  <li><a href="/">Home</a></li>
  <li><a href="/list">List</a></li>
</ul>
```

`templates/home.stpl`

```html
<html>
  <head>
    <title>Home</title>
  </head>
  <body>
    <% include!("./navbar.stpl"); %>

    <h1>Add Person</h1>
    <form method="POST" action="/add">
      <input type="text" name="name" />
      <input type="submit" value="add" />
    </form>
  </body>
</html>
```

`templates/list.stpl`

```html
<html>
  <head>
    <title>Home</title>
  </head>
  <body>
    <% include!("./navbar.stpl"); %>

    <h1>Persons List</h1>
    <ol>
      <% for (index, person) in persons.iter().enumerate() {%>
      <li><%= person %> <a href="/delete/<%= index %>">delete</a></li>
      <% } %>
    </ol>
  </body>
</html>
```

In following `stpl` files or HTML formatted files, we can use pure rust commands. Further, we can include one file in another file like we did in `templates/list.stpl` with `<% include!() %>`.

And don't care much about `database.rs` file. This is a simple struct to mimic some functionality of database as I said. In reality, there is noting more than a vector of strings.

```rust
pub struct Db {
  pub persons: Vec<String>,
}

impl Db {
  pub fn new() -> Self {
    Self {
      persons: Vec::new(),
    }
  }

  pub fn add(&mut self, name: String) {
    self.persons.push(name);
  }

  pub fn delete(&mut self, index: usize) {
    self.persons.remove(index);
  }

  pub fn get(&self) -> Vec<String> {
    self.persons.clone()
  }
}
```

And last we define our handler functions

```rust
#[derive(Deserialize)]
struct Person {
  name: String,
}

async fn homepage() -> impl Responder {
  HttpResponse::Ok().body(Home {}.render_once().unwrap())
}

async fn add_to_persons(
  person: web::Form<Person>,
  db_mutex: web::Data<Mutex<Db>>,
) -> impl Responder {
  let mut db = db_mutex.lock().unwrap();
  db.add(person.name.clone());
  println!("{:?}", db.persons);
  HttpResponse::Found().header("Location", "/").finish()
}

async fn get_persons(db_mutex: web::Data<Mutex<Db>>) -> impl Responder {
  let db = db_mutex.lock().unwrap();
  let persons_list = db.get();
  HttpResponse::Ok().body(
    List {
      persons: persons_list,
    }
    .render_once()
    .unwrap(),
  )
}

async fn delete_person(
  db_mutex: web::Data<Mutex<Db>>,
  web::Path((id,)): web::Path<(usize,)>,
) -> impl Responder {
  let mut db = db_mutex.lock().unwrap();
  db.delete(id);
  HttpResponse::Found().header("Location", "/list").finish()
}
```

In `homepage()`, we returned a simple HTTP Response with contents of `home.stpl`. Take a note of `.render_once().unwrap()`, we created instance of `Home` and called mentioned method which returns `Result<>`.

Take a look at rest of handler functions, One parameters is `db_mutex` of type `web::Data<Mutex<Db>>`, we provided this data in main function. We need to lock mutex in order access data. While in Rust, `.lock()` returns a `Result<>` to handle if data can be locked or some error occurred.

In `add_to_persons()`, we are receiving some data through form which can be accessed by specifying a parameter of type `web::Form<Person>`. While `Person` is a struct which contain all named GET values that will be submitted by user.

In `get_persons()`, consider how we passed vector of string to `List` struct. This variable will be accessed inside out template file.

At last `delete_person()`, Second parameter is bit weird (Yes somewhat I agree). `Web::Path<>` is used to access any part of URL. In main function, I mentioned we can access `id` in handler function. So this is syntax we need to know to access that data. We are destructing `web::Path<(usize,)>` to get our data inside `id` variable. And our server will respond with redirection to `/list`.

## Conslusion

In Actix, we created a very basic Web Application. How did it go? Is it complicated or simple? Yeah, I guess that was a little more complicated than other frameworks like Ruby on Rails or ExpressJS, where we don't have to worry about low-level issues like threads and data sharing via Mutexes. Actix provided us a lot more control over what was going on behind the scenes. Everything must be taken care of, which becomes more difficult in large-scale projects with asynchronous responses. However, there are other advantages, including speed, performance, and security.

Every day, technology advances. We used to compare programs based on how much space they took up and how quickly they ran. However, because we now have more sophisticated devices than ever before, space complexity is often overlooked. Both are luckily taken care of by rust. We can hope that actix will rise in the near future.

So that was it; I hope it helped you learn anything new or gave you a taste of this amazing project. You can found all code [here](https://github.com/ParampreetR/crud_sailfish)
