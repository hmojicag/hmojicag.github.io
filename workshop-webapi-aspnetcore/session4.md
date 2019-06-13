---
layout: course
title: Session 4 - Web API with ASP.Net Core
date: 2019-05-26
---
# Session 4 - User Authentication, Http Clients and Production Environment

1. Execute Queries and Stored Procedures wit EF Core
1. Create an HTTP client that consumes the APIs
1. Add user Authentication with JWT and the Bearer Token Authentication
1. Publish your API to binaries
1. Host and deploy ASP.NET Core by Microsoft

## Execute Queries and Stored Procedures wit EF Core

Entity Framework Core allows you to drop down to raw SQL queries when working with a relational database. This can be useful if the query you want to perform can't be expressed using LINQ, or if using a LINQ query is resulting in inefficient SQL queries. Raw SQL queries can return entity types or, starting with EF Core 2.1, query types that are part of your model.

Let's create an endpoint that will perform a transformation in all the records of the table `Movie`, the query will `sum 1` to the Duration column.

First, let's create the next stored procedure in the database

```sql
CREATE PROCEDURE sp_GetMoviesForStudio
@StudioName varchar(50)
AS
BEGIN

	SELECT Movies.* FROM Movies
	INNER JOIN Studios ON Studios.Id = Movies.StudioId
	WHERE Studios.Name like '%'+@StudioName+'%'

END
GO
```

Create a controller so we can execute some raw queries in the database.

`src\Controllers\RawController`
```csharp
using System.Collections.Generic;
using System.Linq;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using MoviesWebApi.Data;
using MoviesWebApi.Dto;
using MoviesWebApi.Utils;

namespace MoviesWebApi.Controllers {

    [Route("api/raw")]
    [ApiController]
    public class RawController : ControllerBase {

        private readonly AppDbContext db;

        public RawController(AppDbContext db) {
            this.db = db;
        }

        [HttpPost("increaseDuration")]
        //Using a raw SQL query
        public ActionResult IncreaseDuration() {
            //Execute a query that returns nothing
            db.Database.ExecuteSqlCommand("UPDATE Movies SET Duration = Duration + 1");
            return NoContent();
        }

        [HttpGet("movies")]
        //GET api/raw/movies?studioName=<querystring>
        //Return all movies where it's studio name match the querystring
        public ActionResult<IEnumerable<MovieDto>> GetMoviesForStudio([FromQuery] string studioName) {
            //Execute a query (stored procedure) that will return a records of type Movie
            return db.Movies.FromSql("EXECUTE sp_GetMoviesForStudio {0}", studioName)
                .ToList()
                .ConvertAll(Mapper.MapMovie);
        }
    }
}
```

Make calls to it and see what happens:

```
curl -X POST http://localhost:5000/api/raw/increaseDuration
```

```
curl -X GET 'http://localhost:5000/api/raw/movies?studioName=bone'
```


[Reference](https://docs.microsoft.com/en-us/ef/core/querying/raw-sql)

## Create an HTTP client that consumes the APIs

In this link you can find the source code for a simple CLI client that will consume the REST API we just have build: [Simple CLI Client](https://github.com/hmojicag/workshop-webapi-aspnetcore/tree/master/session4/TestClientApp).

In this Simple CLI Client we are using `Flurl`, an alternative to the `HttpClient` library. Check the next page regarding different HTTP Clients: https://code-maze.com/different-ways-consume-restful-api-csharp/ .

For retrying check Polly: https://github.com/App-vNext/Polly/wiki/Polly-and-HttpClientFactory

## Add user Authentication with JWT and the Bearer Token Authentication

Read this tutorial [Part1](https://jonhilton.net/2017/10/11/secure-your-asp.net-core-2.0-api-part-1---issuing-a-jwt/) and [Part2](https://jonhilton.net/security/apis/secure-your-asp.net-core-2.0-api-part-2---jwt-bearer-authentication/) to understand a little more about `JWT` and Authentication for APIs.

So, each "user" of our API is called `consumer`. Every consumer has an `Id` which identifies it uniquely and an `ApiKey` (you can treat this field as a password)  The consumer will know it's `Id` and `ApiKey` and will treat them as secrets, in the same way you treat a password.

The way the `Bearer Token Authentication` works can be explained using the following analogy:

You have a live long subscription to the cinema theater (our API), but that subscription is only valid for you and every time you go you need to show your credentials (username and password), it's really insecure to be showing your credentials to the cashier every time you want to see a movie (http is insecure if it's not encrypted, a spy hooked up in the connection can steal you username and password if they are sent in every request since they are transmitted as plain text), so instead of showing your credentials every time you want to see a movie you show your credentials once and the cinema gives you a token (a JWT token) that expires in a period of time,  with this token you go to the movies any time you want in that period of time (the JWT token has a expiration date). The token has all the data needed to identify you as well as many other information. If the token expires you just need to show your credentials and a new one will be generated for you. If somebody steals your token, no problem we can revoke it easily and give you a new one.

In order to get a Token and have access to our API a consumer must issue a request to the authorization endpoint `/api/auth` passing `Id` and `ApiKey`, the response will be a `JWT` token, this token will expire in `30 minutes`.
Using this token the consumer must pass in every request to the API in the form of a header:
`Authorization: Bearer <token goes here>`.

The token will identify uniquely a user and if valid then it will process the request, without a token or if the token is expired the API will always throw a `401 Unauthorized` response.

First we need a `Model` where to store all consumer related information, like Name and ApiKey .

`src\Models\Consumer.cs`
```csharp
using System;

namespace MoviesWebApi.Models {
    public class Consumer {
        public Consumer() {
            Id = Guid.NewGuid().ToString();
            ApiKey = Guid.NewGuid().ToString().Replace("-", "");
            CreatedDate = DateTime.Now;
            LastUpdatedDate = DateTime.Now;
        }

        public string Id { get; set; }

        public string ApiKey { get; set; }

        public string Name { get; set; }

        public DateTime CreatedDate { get; set; }

        public DateTime LastUpdatedDate { get; set; }
    }
}
```

Also configure `AppDbContext.cs` to support the table `Consumers`.

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
        public DbSet<Actor> Actors { get; set; }
        public DbSet<Studio> Studios { get; set; }
        public DbSet<MovieActor> MovieActors { get; set; }

        public DbSet<Consumer> Consumers { get; set; }

        protected override void OnModelCreating(ModelBuilder modelBuilder) {
            //We are going to put here all the Model characteristics
            //Like relationships between tables, Indexes, Primary Keys...

            //Configure the navigation property, this is, the relationships
            //The next statement can be read like this:
            //"A movie as many Actors and an Actor has many movies
            modelBuilder.Entity<Movie>()//Many to Many relationship trough MovieActors table
                .HasMany(movie => movie.MovieActors)
                .WithOne(movieActor => movieActor.Movie)
                .HasForeignKey(movieActor => movieActor.MovieId);

            modelBuilder.Entity<Actor>() //Many to Many relationship trough MovieActors table
                .HasMany(actor => actor.MovieActors)
                .WithOne(movieActor => movieActor.Actor)
                .HasForeignKey(movieActor => movieActor.ActorId);

            modelBuilder.Entity<Movie>()//Movie <-> Studio. One to Many
                .HasOne(movie => movie.Studio)
                .WithMany(studio => studio.Movies)
                .HasForeignKey(movie => movie.StudioId);

            modelBuilder.Entity<Studio>() //Movie <-> Studio. One to Many
                .HasMany(studio => studio.Movies)
                .WithOne(movie => movie.Studio)
                .HasForeignKey(movie => movie.StudioId);

            modelBuilder.Entity<MovieActor>()//MovieActors. Configure Primary Key
                .HasKey(movieActor => new {movieActor.ActorId, movieActor.MovieId});

            modelBuilder.Entity<Consumer>()//Configure ApiKey to be a unique index
                .HasIndex(consumer => consumer.ApiKey)
                .IsUnique();
            modelBuilder.Entity<Consumer>()
                .HasIndex(consumer => consumer.Name);

        }
    }
}
```

Now we need a need a `Security Key` that will be used to encode the `JWT` token, add it to `appsettings.json` file:

`/appsettings.json`
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
  },
  "SecurityKey": "HazelSoft-Software-Consulting-Company"
}
```


Let's add a new controller that will issue `JWT` tokens by using the consumer Id and `ApiKey`. Also create the dto class.
`src\dto\TokenRequest.cs`
```csharp
namespace MoviesWebApi.Dto {
    public class TokenRequest {
        public string ConsumerId { get; set; }
        public string ApiKey { get; set; }
    }
}
```

`src\Controllers\AuthorizationController.cs`
```csharp
using System;
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Text;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Configuration;
using Microsoft.IdentityModel.Tokens;
using MoviesWebApi.Data;
using MoviesWebApi.Dto;

namespace MoviesWebApi.Controllers {
    [Route("api/auth")]
    [ApiController]
    public class AuthorizationController : ControllerBase {
        private IConfiguration configuration { get; }

        private AppDbContext db;

        public AuthorizationController(IConfiguration configuration, AppDbContext db) {
            this.configuration = configuration;
            this.db = db;
        }

        [AllowAnonymous]
        [HttpPost]
        public IActionResult RequestToken([FromBody] TokenRequest request) {

            if (request?.ConsumerId == null || request.ApiKey == null) {
                return BadRequest();
            }

            //Search for the consumer using the consumer Id:
            var consumer = db.Consumers.Find(request.ConsumerId);
            if (consumer == null || !consumer.ApiKey.Equals(request.ApiKey)) {
                return BadRequest("Not Authenticated");
            }

            var claims = new[] {
                new Claim(ClaimTypes.NameIdentifier, request.ConsumerId)
            };

            var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(configuration["SecurityKey"]));
            var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

            var token = new JwtSecurityToken(
                issuer: "academicos.com.mx",
                audience: "academicos.com.mx",
                claims: claims,
                expires: DateTime.Now.AddMinutes(30),
                signingCredentials: creds);

            return Ok(new {
                token = new JwtSecurityTokenHandler().WriteToken(token)
            });
        }
    }
}
```

Now we need to configure the `pipeline` of request, or also called `middleware` in order to validate that the token passed to the API is valid.

`/Startup.cs`
```csharp
using System.Text;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.DependencyInjection.Extensions;
using Microsoft.IdentityModel.Tokens;
using MoviesWebApi.Data;
using MoviesWebApi.Services;
using Newtonsoft.Json;

namespace MoviesWebApi {
    public class Startup {
        public Startup(IConfiguration configuration) {
            Configuration = configuration;
        }

        public IConfiguration Configuration { get; }

        // This method gets called by the runtime. Use this method to add services to the container.
        public void ConfigureServices(IServiceCollection services) {

            //See: https://jonhilton.net/2017/10/11/secure-your-asp.net-core-2.0-api-part-1---issuing-a-jwt/
            //Need more control over what users execute what endpoint? See: https://jonhilton.net/identify-users-permissions-with-jwts-and-asp-net-core-webapi/
            services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
                .AddJwtBearer(options =>
                {
                    options.TokenValidationParameters = new TokenValidationParameters
                    {
                        ValidateIssuer = true,
                        ValidateAudience = true,
                        ValidateLifetime = true,
                        ValidateIssuerSigningKey = true,
                        ValidIssuer = "academicos.com.mx",
                        ValidAudience = "academicos.com.mx",
                        IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(Configuration["SecurityKey"]))
                    };
                });

            services.AddMvc()
                .AddJsonOptions(options => {
                    options.SerializerSettings.ReferenceLoopHandling = Newtonsoft.Json.ReferenceLoopHandling.Ignore;
                    options.SerializerSettings.NullValueHandling = NullValueHandling.Ignore;
                })
                .SetCompatibilityVersion(CompatibilityVersion.Version_2_2);
            services.AddMemoryCache();
            services.AddDbContext<AppDbContext>(options =>
                options.UseSqlServer(Configuration.GetConnectionString("AppDbContextDB")));
            services.TryAddScoped<IMoviesService, MoviesService>();
            services.TryAddScoped<IActorsService, ActorsService>();
            services.TryAddScoped<IStudiosService, StudiosService>();
        }

        // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
        public void Configure(IApplicationBuilder app, IHostingEnvironment env) {
            if (env.IsDevelopment()) {
                app.UseDeveloperExceptionPage();
            }

            app.UseAuthentication();

            app.UseMvc();
        }
    }
}
```

Let's create a simple controller just demonstrate how the validation works.

`src\Controllers\AuthTestController.cs`
```csharp
using System.Security.Claims;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using MoviesWebApi.Data;
using MoviesWebApi.Models;

namespace MoviesWebApi.Controllers {

    [Route("api/authtest")]
    [ApiController]
    public class AuthTestController : ControllerBase {
        private AppDbContext db;

        public AuthTestController(AppDbContext db) {
            this.db = db;
        }

        [HttpPost]
        [Authorize]
        public ActionResult<Consumer> TestAuthentication() {
            var claim = User.FindFirst(ClaimTypes.NameIdentifier);
            var consumerId = claim != null ? claim.Value : null;

            if (consumerId != null) {
                var consumer = db.Consumers.Find(consumerId);
                return consumer;
            }

            return BadRequest("You broke the internet xd...");
        }

    }
}
```

Now you can add the `Attribute` called `[Authorize]` to every endpoint you want to restrict.

Create a new `Migration` so the table `Consumers` is added.
To do so, execute:

```
dotnet ef migrations add AddingConsumers
dotnet ef database update
```

Add a new Consumer manually by using the next SQL script.

```sql
INSERT INTO [dbo].[Consumers]
           ([Id]
           ,[ApiKey]
           ,[Name]
           ,[CreatedDate]
           ,[LastUpdatedDate])
     VALUES
           (NEWID(),
		   '7CEEFA4CA8C1415CA5D7CE162CA6D053',
		   'Test Consumer',
		   GETDATE(),
		   GETDATE())
```

Now test it. First get a new token by calling:
```
curl -X POST \
  http://localhost:5000/api/auth \
  -H 'Content-Type: application/json' \
  -d '{
	"consumerid":"70F1BE89-958A-41D1-9C66-83846F84739B",
	"apikey":"7CEEFA4CA8C1415CA5D7CE162CA6D053"
}'
```

You will get a token something like:
```
{
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJodHRwOi8vc2NoZW1hcy54bWxzb2FwLm9yZy93cy8yMDA1LzA1L2lkZW50aXR5L2NsYWltcy9uYW1lIjoiNzBGMUJFODktOTU4QS00MUQxLTlDNjYtODM4NDZGODQ3MzlCIiwiZXhwIjoxNTYwMTI5NDg3LCJpc3MiOiJ5b3VyZG9tYWluLmNvbSIsImF1ZCI6InlvdXJkb21haW4uY29tIn0.Fts9GymKjfr80Enouh-4g_PuY9w9yehAvZnwTRnOcW0"
}
```

![Get a new JWT](../assets/images/workshop-webapi-aspnetcore/session4-getJWT.PNG)

Now copy that token and make a `POST` request to `/api/authtest` sending the token in the `Authorization` header.
```
curl -X POST \
  http://localhost:5000/api/authtest \
  -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJodHRwOi8vc2NoZW1hcy54bWxzb2FwLm9yZy93cy8yMDA1LzA1L2lkZW50aXR5L2NsYWltcy9uYW1laWRlbnRpZmllciI6IjcwRjFCRTg5LTk1OEEtNDFEMS05QzY2LTgzODQ2Rjg0NzM5QiIsImV4cCI6MTU2MDM2OTc1NywiaXNzIjoiYWNhZGVtaWNvcy5jb20ubXgiLCJhdWQiOiJhY2FkZW1pY29zLmNvbS5teCJ9.DJPxsCAONXjyJpABeRYHMi7p2ulqiMQFsHX5tz32138'
```

![Get a new JWT](../assets/images/workshop-webapi-aspnetcore/session4-authTest.PNG)

And that's it, now your API is secured by using the Bearer Token Authentication and JWT.
Now every call must contain a valid token (for the specified endpoints) or it will be rejected.


## Publish your API to binaries

You can use the `dotnet publish` command to create the binaries for your project and have it ready for production.
Check the official documentation for more info: [link](https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet-publish?tabs=netcore21).

Execute the next command:

```
dotnet publish -c Release
```

All the binaries needed for run and deployment are now compiled in `bin\Release\netcoreapp2.2\publish`.
If you want to run that binary just do:

```
dotnet bin\Release\netcoreapp2.2\publish\MoviesWebApi.dll
```

Now, that `dll` can be deployed to **IIS**, **Azure**, configured a reverse proxy with **Nginx**, put into a **Docker** container or in any way that fits your requirements.

In the next section you can see many types of deployments available for an ASP.Net Core application.

## Host and deploy ASP.NET Core by Microsoft

There are many ways you can deploy your `Asp.net Core` app, this document from the official page from Microsoft summarizes the most common ones:

[Host and deploy ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/?view=aspnetcore-2.2)

If you are unsure of what method to use I can foresee 3 scenarios that you may encounter and I provide the recommended way to handle it.

1. You have a physical server, and you can to install in there everything you need. This is called `On Premise`. Either you have Linux or Windows on that server I recommend you install SQL Server directly in the bare metal. And deploy your `Asp.Net Core` app using a docker container, see [here](https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/docker/?view=aspnetcore-2.2) for more info. This way you won't worry about setting up any server or reverse proxy, you just run the docker image and that's it. This also apply you rent a `VPS`.
1. You have a hosting provider, something like [SmarterASP.Net](https://www.smarterasp.net/) or [Host4Asp](https://host4asp.net/). In that case they will manage everything for you, and you just need copy the binaries (`*.dll`) into the appropriate directory, that could be done using FTP or git, you may already know how to do this. To generate the binaries run `dotnet publish -c Release`. More info see [here](https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet-publish?tabs=netcore21).
1. You are paying `Azure`, `AWS` or some other cloud computing provider. That means you have a lot of money :) and that you can follow this guide [here](https://docs.microsoft.com/en-us/azure/app-service/app-service-web-get-started-dotnet) to publish your API.
