---
layout: course
title: Session 2 - Web API with ASP.Net Core
date: 2019-05-26
---

# Session 2 - HTTP and ASP.Net Core

1. Networking basics for Web Developers
1. Introduction to HTTP for Web Developers
1. Introduction to ASP.Net Core
1. Our first Hello World Web API
1. Defining the Basic Endpoints for our API (Request/response object, status codes and headers)
1. HTTP Conventions for our API
1. Building the Models and Controllers
1. Sharing the API with other devices in the LAN
1. Connecting a database for persisting the data
1. Introduction to EF Core

## Networking basics for Web Developers

As a web developer we are going to develop applications that live in the Application layer of the OSI model.
We are not going to worry about how the data (a web page, a JSON payload, a JPG image...) is transited over the network and that is going the make it easy for us to focus in our task, which is build an API that users can consume.

Although we need to understand some of the basics of HTTP and networking so we can troubleshoot problems if they arise or just to have them as general knowledge.

Remember **HTTP** is a request/response protocol, and every request and it's correspondent response are stateless, it means they do not store any session related information.
As an example, imagine you have a web server which only you can access, nobody else can, then you request a web page from that web server (at least one request/response) is made between the client (your web browser) and the server, this page is a Poll for evaluating somebody, you fill all the fields in the poll then you click the submit button that sends data to the server and refreshes the page, the server will treat each request separately and should not try to relate them, every request must contain enough information within to tell the server what to do (user info, data...).

![Simplification of the HTTP Request/Response Model](../assets/images/workshop-webapi-aspnetcore/http-request-response.png)

![TCP/IP 5 layers](../assets/images/workshop-webapi-aspnetcore/TCPIP5.png)

[Reference](https://microchipdeveloper.com/tcpip:tcp-ip-five-layer-model)

## Introduction to HTTP for Web Developer

See [this HTTP presentation](https://drive.google.com/open?id=1GMkh4y9zXNYoBcKUdpH7dAFjwIU4vBTy)

[Reference](https://developer.mozilla.org/en-US/docs/Web/HTTP)

## Introduction to ASP.Net Core

ASP.NET is a popular web-development framework for building web apps on the .NET platform.

ASP.NET Core is the open-source version of ASP.NET, that runs on macOS, Linux, and Windows. ASP.NET Core was first released in 2016 and is a re-design of earlier Windows-only versions of ASP.NET.

Open a terminal and run the next commands:

```csharp
mkdir webAppTest
cd webAppTest
dotnet new webApp --no-https
dotnet run
```

The previous commands will create a new project based on a standard web app template and launch it, you can see it in action in your browser by going to **localhost:5000**.

Our **Web API** will use the same framework (ASP.Net Core) that is used for **Web Apps** with the difference that instead of returning web pages based on a template or a View (Razor Pages) we are going to return Data (a Json representation).

[Reference](https://dotnet.microsoft.com/learn/web/what-is-aspnet-core)

## Our first Hello World Web API

Open a terminal and run the next commands for creating a new Web API project based on a the standard project template provided by Microsoft.

```csharp
mkdir MoviesWebApi
cd MoviesWebApi
dotnet new webapi --no-https
dotnet run
```

Open postman and make a request to:

```
GET http://localhost:5000/api/Values
```

You will get hardcoded values:

```
[
    "value1",
    "value2"
]
```

## Defining the Basic Endpoints for our API (Request/response object, status codes and headers)

Here below is the web API we are going to create

|API | Description | Request body | Response body | Status Codes
|--- | ---- | ---- | ---- |---- |
|GET /api/movies | Get all movies | None | Array of movies| 200
|GET /api/movies/{id} | Get a movie by ID | None | movie| 200, 404
|POST /api/movies | Add a new movie | movie | movie | 201, 422, 400
|PUT /api/movies/{id} | Update an existing movie | movie | movie | 200, 404, 422, 400
|PUT /api/movies/{movieId}/actors | Add an actor to a movie | actor | movie | 200, 404, 422, 400
|DELETE /api/movies/{id} | Delete a movie | None | None| 204, 404
|POST /api/actors | Add a new actor | actor | actor | 201, 422, 400
|POST /api/studios | Add a new studios | studios | studios | 201, 422, 400


## HTTP Conventions for our API

### API naming conventions
By convention the names of the endpoints must be **nouns**, that is, they must be **things**, not actions. And the should be plural.
For example:

```
GET api/getMovies           # This is bad
GET api/movies              # This is good

GET api/{id}/movie          # This is bad
GET api/movies/{id}         # This is good

GET api/actorsOfMovie/{id}  # This is bad
GET api/movies/{id}/actors  # This is good
```

### HTTP common status codes

It is good to return the appropriate status code according to request type and response. I list here the most common response status codes.

#### Level 200
* 200 - Ok.         (Request fulfilled)
* 201 - Created.    (Successfully created)
* 204 - No Content. (Successfully created, No content returned)

### Level 400 Client Mistakes (Errors)
* 400 - Bad Request (Generic bad request response)
* 401 - Unauthorized (User is not logged in)
* 403 - Forbidden (User is logged in but has no permissions)
* 422 - Unprocessable entity (Semantic mistakes, failed validations)

### Level 500 Server Mistakes (Faults)
* 500 - Internal Server Error

## Building the Models and Controllers

Using the project we have just created using the **webapi** template create a `src` folder and move the `Controllers` inside it (It's not mandatory but I like to keep all my code inside an `src` folder).

Now Create a folder called `src/Models` and inside it a class called `Movie`.
This class is going to be entity we are trying to manipulate (Get, Create, Update, Delete), this class will also serve as Model for mapping to a Table in a SQL Server database.

`src/Models/Movie.cs`
```csharp
using System;

namespace MoviesWebApi.Models {
    public class Movie {

        public Movie() {
            Id = Guid.NewGuid().ToString();
            CreatedDate = DateTime.Now;
            LastUpdatedDate = DateTime.Now;
        }

        // This field is going to be recognized automatically by
        // EF Core as the Primary Key of the table
        public string Id { get; set; }

        public string Name { get; set; }

        public string Genre { get; set; }

        public int Year { get; set; }

        public int Duration { get; set; }

        // These fields are here to know when this record
        // was first created and when it was last updated
        public DateTime CreatedDate { get; set; }

        public DateTime LastUpdatedDate { get; set; }
    }
}
```

Now let's create the **Controller** for handling all the HTTP request.
In the folder `src/Controllers` create a new class called `MoviesController`:

`src/Controllers/MoviesController.cs`
```csharp
using System.Collections.Generic;
using Microsoft.AspNetCore.Mvc;
using MoviesWebApi.Models;

namespace MoviesWebApi.Controllers {

    [Route("api/movies")]
    [ApiController]
    public class MoviesController : ControllerBase {

        [HttpGet]
        //GET api/movies
        public ActionResult<IEnumerable<Movie>> GetAllMovies() {
            List<Movie> movies = new List<Movie>();
            Movie mrBrooks = new Movie();
            mrBrooks.Name = "Mr. Brooks";
            mrBrooks.Genre = "Psychological thriller";
            mrBrooks.Year = 2007;
            mrBrooks.Duration = 120;

            Movie sword = new Movie() {
                Name = "Sword of the Stranger",
                Genre = "Anime",
                Year = 2007,
                Duration = 103
            };

            movies.Add(mrBrooks);
            movies.Add(sword);

            return movies;
        }

        [HttpGet("{id}")]
        //GET api/movies/{id}
        public ActionResult<Movie> GetMovie(string id) {
            Movie sword = new Movie() {
                Name = "Sword of the Stranger",
                Genre = "Anime",
                Year = 2007,
                Duration = 103
            };

            return sword;
        }

        [HttpPost]
        //POST api/movies
        //Payload: A Json representing the Movie object
        public ActionResult<Movie> CreateMovie(Movie movie) {
            return movie;
        }

        [HttpPut("{id}")]
        //PUT api/movies/{id}
        //Payload: A Json representing the Movie object
        public ActionResult<Movie> UpdateMovie(string id, Movie movie) {
            return movie;
        }

        [HttpDelete("{id}")]
        //DELETE api/movies/{id}
        public ActionResult DeleteMovie(string id) {
            return NoContent();
        }
    }
}
```

As you can see, the controller will not persist any data, and will only provide hardcoded static information.

Open postman and test all the requests, here I put also the `curl` equivalent:

**Get All movies**
```
curl -X GET \
  http://localhost:5000/api/movies \
```
![Hardcoded Get All movies endpoint](../assets/images/workshop-webapi-aspnetcore/session2-hardcoded-getall.PNG)

**Create a movie**
```
curl -X POST \
  http://localhost:5000/api/movies \
  -H 'Content-Type: application/json' \
  -H 'cache-control: no-cache' \
  -d '{
        "name": "Le fabuleux destin d Amelie Poulain",
        "genre": "Comedy",
        "year": 2001,
        "duration": 122
    }'
```
![Hardcoded POSt movie endpoint](../assets/images/workshop-webapi-aspnetcore/session2-hardcoded-post.PNG)

**Get a movie**
```
curl -X GET \
  http://localhost:5000/api/movies/840ac292-ce91-4c86-aae1-45c344189b70
```
![Hardcoded Get By movie Id endpoint](../assets/images/workshop-webapi-aspnetcore/session2-hardcoded-get.PNG)

**Update a movie**
```
curl -X PUT \
  http://localhost:5000/api/movies/840ac292-ce91-4c86-aae1-45c344189b70 \
  -H 'Content-Type: application/json' \
  -d '{
        "name": "Sword of the Stranger",
        "genre": "Action",
        "year": 2001,
        "duration": 122
}'
```
![Hardcoded Update a movie endpoint](../assets/images/workshop-webapi-aspnetcore/session2-hardcoded-put.PNG)

## Sharing the API with other devices in the LAN

So far we have our API "working", it returns hardcoded values but we are advancing at good pace towards our goal which is to build a full API.

The **web api** is published locally using the Kestrel embedded server, it is configured to only listen to `http://localhost:5000`, we are going to tweak that so we can share our API with other developers connected to the same LAN.

Add `.UseUrls("http://*:5000")` to the WebHostBuilder in `Program.cs`.
So instead of just listening to `localhost` it will listen to any income connection to the port `5000`.

`Program.cs`
```csharp
using Microsoft.AspNetCore;
using Microsoft.AspNetCore.Hosting;

namespace MoviesWebApi
{
    public class Program
    {
        public static void Main(string[] args)
        {
            CreateWebHostBuilder(args).Build().Run();
        }

        public static IWebHostBuilder CreateWebHostBuilder(string[] args) =>
            WebHost.CreateDefaultBuilder(args)
                .UseUrls("http://*:5000")
                .UseStartup<Startup>();
    }
}
```

Test, ask another developer to make a request to your machine.
For this example my computer was running with IP `192.168.0.7`, so the request would be:

```
curl -X GET \
  http://192.168.0.7:5000/api/movies/
```

You can get your local `ip` by running `ipconfig` in a terminal in windows.

## Connecting a database for persisting the data

Now we are going to add a database connection for persisting the data.
First, let's create a database and the basic schema for the only table we have.

Execute the next command in a `New Query` editor in SSMS.

```
CREATE DATABASE MoviesDB
GO
USE MoviesDB

CREATE TABLE Movies (
	Id varchar(45) NOT NULL PRIMARY KEY,
	[Name] varchar(250),
	Genre  varchar(250),
	[Year] int,
	Duration int,
	CreatedDate [datetime2](7),
	LastUpdatedDate [datetime2](7)
)
```

As you can see we are creating a database called `MoviesDB` and a table in it called `Movies`, which is the plural for the entity class `Movie` we have in our Models folder. Each record of the table `Movies`  will mapped to an instance of the class `Movie` by Entity Framework Core.

Now let's add a `DbContext`, you can see a DbContext as a representation of the database, it will hold a mapping of all the tables in the database that are going to be used by our web api.

Create a folder called `src\Data` and create a class called `AppDbContext` in it:

`src\Data\AppDbContext.cs`
```csharp
using Microsoft.EntityFrameworkCore;
using MoviesWebApi.Models;

namespace MoviesWebApi.Data {
    public class AppDbContext : DbContext {
        public AppDbContext(DbContextOptions<AppDbContext> options)
            : base(options) {
        }

        //This is our table Movies
        public DbSet<Movie> Movies { get; set; }

        protected override void OnModelCreating(ModelBuilder modelBuilder) {
            //We are going to put here all the Model characteristics
            //Like relationships between tables, Indexes, Primary Keys...
        }
    }
}
```

Now we need a way to let know our DbContext how to connect to the database, we need a connection string for that job, add a new connection string to the properties file called `appsettings.json`.

For the connection string use the database name, user and passwords appropriate for the database instance you have.

`\appsettings.json`
```javascript
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning"
    }
  },
  "AllowedHosts": "*",
  "ConnectionStrings": {
    "AppDbContextDB": "Server=MEXMON740L\\SQLEXPRESS; Initial Catalog=MoviesDB;user id=moviesUser;password=123456789;"
  }
}
```

I have seen many different types of Connection Strings, you need to play with it if for some reason it doesn't seems to work. For example:

I'm sure the next one works for working with remote databases like the one hosted in Smarterasp.net
```
"Data Source=<DbServerHostName>.site4now.net; Initial Catalog=DB_<userId>_<user>; User
Id=DB_<userId>_<user>_admin;Password=YOUR_DB_PASSWORD;"
```

The next connection string is used for connection to a SQL Server 2017 spinned up with docker container in the local machine
```
"Data Source=localhost,1401; Initial Catalog=DATABASE_NAME; user id=DB_USER; password=YOUR_DB_PASSWORD;";
```

Now let's register our DbContext as a service for our app, add the next line to the `ConfigureServices` method in the `Startup` class.

```
services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(Configuration.GetConnectionString("AppDbContextDB")));
```

You may end up with a `Startup` class very similar to this one:

`\Startup.cs`
```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using MoviesWebApi.Data;

namespace MoviesWebApi {
    public class Startup {
        public Startup(IConfiguration configuration) {
            Configuration = configuration;
        }

        public IConfiguration Configuration { get; }

        // This method gets called by the runtime. Use this method to add services to the container.
        public void ConfigureServices(IServiceCollection services) {
            services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_2_2);
            services.AddDbContext<AppDbContext>(options =>
                options.UseSqlServer(Configuration.GetConnectionString("AppDbContextDB")));
        }

        // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
        public void Configure(IApplicationBuilder app, IHostingEnvironment env) {
            if (env.IsDevelopment()) {
                app.UseDeveloperExceptionPage();
            }

            app.UseMvc();
        }
    }
}
```
Now run your app, it may not do anything in particular, but watch the logs, the must no throw any error related to database connection, if the application loads correctly then we did our job very well.

## Introduction to EF Core

EF Core can serve as an object-relational mapper (O/RM), enabling .NET developers to work with a database using .NET objects, and eliminating the need for most of the data-access code they usually need to write.

So, in other words, EF Core will cover for us the underlying logic that gets executed against a database and we will only focus in what we want to retrieve from it.

Let's change all our controllers so you see for yourself.

`src\Controllers\MoviesController.cs`
```csharp
using System.Collections.Generic;
using System.Linq;
using Microsoft.AspNetCore.Mvc;
using MoviesWebApi.Data;
using MoviesWebApi.Models;

namespace MoviesWebApi.Controllers {

    [Route("api/movies")]
    [ApiController]
    public class MoviesController : ControllerBase {

        private AppDbContext db;

        //Inject the AppDbContext object into this controlle class
        public MoviesController(AppDbContext db) {
            this.db = db;
        }

        [HttpGet]
        //GET api/movies
        public ActionResult<IEnumerable<Movie>> GetAllMovies() {
            var movies = db.Movies.ToList();
            return movies;
        }

        [HttpGet("{id}")]
        //GET api/movies/{id}
        public ActionResult<Movie> GetMovie(string id) {
            Movie movie = db.Movies.Find(id);

            if (movie == null) {
                return NotFound();
            }

            return movie;
        }

        [HttpPost]
        //POST api/movies
        //Payload: A Json representing the Movie object
        public ActionResult<Movie> CreateMovie(Movie movie) {
            db.Movies.Add(movie);
            db.SaveChanges();
            return movie;
        }

        [HttpPut("{id}")]
        //PUT api/movies/{id}
        //Payload: A Json representing the Movie object
        public ActionResult<Movie> UpdateMovie(string id, Movie movie) {
            Movie movieFromDb = db.Movies.Find(id);

            if (movieFromDb == null) {
                return NotFound();
            }

            //Copy all modifiable fields
            movieFromDb.Name = movie.Name;
            movieFromDb.Year = movie.Year;
            movieFromDb.Genre = movie.Genre;
            movieFromDb.Duration = movie.Duration;

            db.Movies.Update(movieFromDb);
            db.SaveChanges();

            return movieFromDb;
        }

        [HttpDelete("{id}")]
        //DELETE api/movies/{id}
        public ActionResult DeleteMovie(string id) {
            Movie movieFromDb = db.Movies.Find(id);

            if (movieFromDb == null) {
                return NotFound();
            }

            db.Movies.Remove(movieFromDb);
            db.SaveChanges();

            return NoContent();
        }
    }
}
```

Watch closely the terminal and see what SQL code gets executed, for example, for a creation of an entity I got the next log message:

![Log showing the SQL code executed by EF Core](../assets/images/workshop-webapi-aspnetcore/session2-EFCore-SQL.PNG)

Congratulations, you have your first fully functionally API.
In the next Session we will see good practices about developing web apis and why is not really good to have all the logic in the Controller, but for now fell happy, you did it!!
