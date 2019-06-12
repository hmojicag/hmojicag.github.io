---
layout: course
title: Session 3 - Web API with ASP.Net Core
date: 2019-05-26
---

# Session 3 - ASP.Net Core and WebAPI Basics

1. Dependency Injection
1. The Service Layer
1. Migrations (Initial Creation)
1. In Memory Cache
1. Relationships
1. Migrations (Migrate changes)
1. Return Resources instead of Entities

Let's take a look into the good practices when building a web api, this good practices will help us build robust and maintainable software.

# Dependency Injection

A software developer writes a lot of code that is tightly coupled; and when complexity grows, the code will eventually deteriorate into spaghetti code; in other words, the application design being a bad design.

Dependency Injection (DI) is a pattern where objects are not responsible for creating their own dependencies. Dependency Injection is a way to remove hard-coded dependencies among objects, making it easier to replace an object's dependencies, either for testing (using mock objects in unit test) or to change run-time behavior.

### Tight Coupling
When a class is dependent on a concrete dependency, it is said to be tightly coupled to that class. A tightly coupled object is dependent on another object; that means changing one object in a tightly coupled application often requires changes to a number of other objects. It is not difficult when an application is small but in an enterprise level application, it is too difficult to make the changes.

### Loose Coupling
It means two objects are independent and an object can use another object without being dependent on it. It is a design goal that seeks to reduce the inter- dependencies among components of a system with the goal of reducing the risk that changes in one component will require changes in any other component.

Now in short, Dependency Injection is a pattern that makes objects loosely coupled instead of tightly coupled. When we are designed classes with DI, they are more loosely coupled because they do not have direct, hard-coded dependencies on their collaborators.

## The Service Layer

The ASP.NET Core itself provides basic built in IoC container that is represented by IserviceProvider interface. It supports constructor dependency injection by default. ASP.NET Core uses DI for for instantiating all its components and services. The container is configured in ConfigureService method of the startup.cs class as this class is entry point to application.

![Service Lifetimes](http://social.technet.microsoft.com/wiki/cfs-file.ashx/__key/communityserver-wikis-components-files/00-00-00-00-05/172024.1.PNG)

It's very common to create the **Service Layer** as **Scoped** instances, mainly because most of the time they have a dependency on a database service which is also `scoped`.

It's a good practice to always put all the business logic in the Service Layer, the controller layer should only handle the request and no more. So let's register a new Singleton service for handling all the movies logic and inject it into the controller.

Create a new folder called `src\Services` and inside it a new Interface IMoviesService and a class `MoviesService` that implements that interface.
And register the service to the IoC engine in `Startup`.

`src\Services\IMoviesService.cs`
```csharp
using System.Collections.Generic;
using MoviesWebApi.Models;

namespace MoviesWebApi.Services {
    public interface IMoviesService {
        List<Movie> GetAllMovies();
        Movie GetMovie(string id);
        Movie CreateMovie(Movie movie);
        Movie UpdateMovie(string id, Movie movie);
        void DeleteMovie(string id);
    }
}
```

`src\Services\MoviesService.cs`
```csharp
using System.Collections.Generic;
using System.Linq;
using MoviesWebApi.Data;
using MoviesWebApi.Models;

namespace MoviesWebApi.Services {
    public class MoviesService : IMoviesService {

        private AppDbContext db;

        public MoviesService(AppDbContext db) {
            this.db = db;
        }

        public List<Movie> GetAllMovies() {
            return db.Movies.ToList();
        }

        public Movie GetMovie(string id) {
            return db.Movies.Find(id);
        }

        public Movie CreateMovie(Movie movie) {
            db.Movies.Add(movie);
            db.SaveChanges();
            return movie;
        }

        public Movie UpdateMovie(string id, Movie movie) {
            Movie movieFromDb = db.Movies.Find(id);

            if (movieFromDb == null) {
                return null;
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

        public void DeleteMovie(string id) {
            Movie movieFromDb = db.Movies.Find(id);

            if (movieFromDb == null) {
                return;
            }

            db.Movies.Remove(movieFromDb);
            db.SaveChanges();
        }
    }
}
```

Update the controller.

```csharp
using System.Collections.Generic;
using Microsoft.AspNetCore.Mvc;
using MoviesWebApi.Models;
using MoviesWebApi.Services;

namespace MoviesWebApi.Controllers {

    [Route("api/movies")]
    [ApiController]
    public class MoviesController : ControllerBase {

        private IMoviesService moviesService;

        public MoviesController(IMoviesService moviesService) {
            this.moviesService = moviesService;
        }

        [HttpGet]
        //GET api/movies
        public ActionResult<IEnumerable<Movie>> GetAllMovies() {
            return moviesService.GetAllMovies();
        }

        [HttpGet("{id}")]
        //GET api/movies/{id}
        public ActionResult<Movie> GetMovie(string id) {
            Movie movie = moviesService.GetMovie(id);

            if (movie == null) {
                return NotFound();
            }

            return movie;
        }

        [HttpPost]
        //POST api/movies
        //Payload: A Json representing the Movie object
        public ActionResult<Movie> CreateMovie(Movie movie) {
            return moviesService.CreateMovie(movie);
        }

        [HttpPut("{id}")]
        //PUT api/movies/{id}
        //Payload: A Json representing the Movie object
        public ActionResult<Movie> UpdateMovie(string id, Movie movie) {
            Movie updatedMovie = moviesService.UpdateMovie(id, movie);

            if (updatedMovie == null) {
                return NotFound();
            }

            return updatedMovie;
        }

        [HttpDelete("{id}")]
        //DELETE api/movies/{id}
        public ActionResult DeleteMovie(string id) {
            moviesService.DeleteMovie(id);
            return NoContent();
        }
    }
}
```

And add the next line to the method `ConfigureServices` in the class `Startup`.

```csharp
services.TryAddScoped<IMoviesService, MoviesService>();
```

[Reference](https://social.technet.microsoft.com/wiki/contents/articles/37218.asp-net-core-overview-of-dependency-injection.aspx)


## Migrations (Initial Creation)

We have manually created all the schema of the database, that is, using SQL code. If we are developers we have the option of EF Core handling all that for us as well. That is called `Migrations`.

As quoted by the Microsoft page:

> A data model changes during development and gets out of sync with the database. You can drop the database and let EF create a new one that matches the model, but this procedure results in the loss of data. The migrations feature in EF Core provides a way to incrementally update the database schema to keep it in sync with the application's data model while preserving existing data in the database.

This has advantages and disadvantages, I personally think is a very cool feature:

Pros:
1. SQL Server schema gets generated automagically by EF Core, which means you do not need to invest more time writing SQL Code.
1. EF Core not only auto-generated the code for the schema, it also generated the code for a roll back, so in case something goes wrong you can quickly roll back.
1. You can version the code, since it's C# code which will get stored in the project itself it's easily versioned along with all your other changes

Cons:
1. You will need to fire all your database developers (just kidding).

Let's re-create the database using `Migrations` so we can add changes later and see how this cool feature works.

First Delete the database we created in the previous session.

```
DROP DATABASE MoviesDB
```

Now go to the terminal, in the root of the our project (where the file `MoviesWebApi.csproj` is located) run the next command:

```
dotnet ef migrations add InitialMigration
```

The previous command, if executed correctly, created a migration with the name `InitialMigration`. Look for a folder called `Migrations` in the project, you will see some new files created there, these files represent the migrations, each file contains C# code which serves for making database changes.

For this first migration all the code you see there is for Creating the database if not exists and for creating the table with the structure we specified in the Model.

Now execute the next command for applying the pending migrations to the database, for our scenario it will apply the migration called `InitialMigration` which is the only one there.

```
dotnet ef database update
```

Using the Db Configuration we setup in our project the previous command will apply the migration to the SQL Server database. If executed correctly you should now see the database `MoviesDB` again in SSMS and the correct schema in there.

[Reference](https://docs.microsoft.com/en-us/ef/core/managing-schemas/migrations/)

## In Memory Cache

There are different ways to cache responses to clients so our service won't need to process the same response again and again, this way we boost the performance of our application, take away database bottlenecks and maybe save some bucks if we were planning to scale our application to a more powerful server or use a server-farm or more EC2 instances.

One of this ways, which I think is pretty cool is the `In Memory Cache`.

I will quote here microsoft:

> Caching can significantly improve the performance and scalability of an app by reducing the work required to generate content. Caching works best with data that changes infrequently. Caching makes a copy of data that can be returned much faster than from the original source. Apps should be written and tested to never depend on cached data.

> ASP.NET Core supports several different caches. The simplest cache is based on the IMemoryCache, which represents a cache stored in the memory of the web server. Apps that run on a server farm of multiple servers should ensure that sessions are sticky when using the in-memory cache.

### Cache guidelines
> Code should always have a fallback option to fetch data and not depend on a cached value being available.
> * The cache uses a scarce resource, memory. Limit cache growth:
> * Do not use external input as cache keys.
> * Use expirations to limit cache growth.
> * Use SetSize, Size, and SizeLimit to limit cache size. The ASP.NET Core runtime does not limit cache size based on memory pressure. It's up to the developer to limit cache size.

Now, hands on. In-memory caching is a service that's referenced from your app using Dependency Injection. Call `AddMemoryCache` in `ConfigureServices`:

```csharp
services.AddMemoryCache();
```

Now, we need a class to store the keys, the keys are going to be used to individually identify each entry in the cache.
We will also add a utility class for aiding us generating those keys.

`src\MemoryCache\MemoryCacheKey.cs`
```csharp
namespace MoviesWebApi.MemoryCache {
    public enum MemoryCacheKey {
        MOVIES_ALL,
        MOVIE_BY_ID
    }

    public static class MemoryCacheKeyGenerator {
        public static string Generate(MemoryCacheKey key, string identifier = "") {
            if (string.IsNullOrEmpty(identifier)) {
                return key.ToString();
            }
            return $"{key.ToString()}_{identifier}";
        }
    }
}
```

Now inject the cache into the `MoviesService` and use it:

`src\Services\MoviesService.cs`
```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using Microsoft.Extensions.Caching.Memory;
using MoviesWebApi.Data;
using MoviesWebApi.MemoryCache;
using MoviesWebApi.Models;

namespace MoviesWebApi.Services {
    public class MoviesService : IMoviesService {

        private AppDbContext db;
        private IMemoryCache memoryCache;

        public MoviesService(AppDbContext db, IMemoryCache memoryCache) {
            this.db = db;
            this.memoryCache = memoryCache;
        }

        //Now we will get all movies from the InMemoryCache which are stored under the key "MOVIES_ALL"
        //If the entry in the cache does not exists (first time, cache was expired or evicted)
        //then it will call the lambda function which receives an ICacheEntry as parameter and returns
        //the values for that entry.
        public List<Movie> GetAllMovies() {
            return memoryCache.GetOrCreate(
                MemoryCacheKey.MOVIES_ALL.ToString(), cacheEntry => {
                    cacheEntry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30);
                    return db.Movies.ToList();
                });
        }


        public Movie GetMovie(string id) {
            return memoryCache.GetOrCreate(
                MemoryCacheKeyGenerator.Generate(MemoryCacheKey.MOVIE_BY_ID, id),
                cacheEntry => GetMovieFromDb(cacheEntry, id));
        }

        //Instead of creating an anonymous method, call this one for fetching data from db
        private Movie GetMovieFromDb(ICacheEntry cacheEntry, string id) {
            cacheEntry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30);
            return db.Movies.Find(id);
        }

        public Movie CreateMovie(Movie movie) {
            db.Movies.Add(movie);
            db.SaveChanges();
            //Evict cache with all movies
            memoryCache.Remove(MemoryCacheKey.MOVIES_ALL.ToString());
            return movie;
        }

        public Movie UpdateMovie(string id, Movie movie) {
            Movie movieFromDb = db.Movies.Find(id);

            if (movieFromDb == null) {
                return null;
            }

            //Copy all modifiable fields
            movieFromDb.Name = movie.Name;
            movieFromDb.Year = movie.Year;
            movieFromDb.Genre = movie.Genre;
            movieFromDb.Duration = movie.Duration;
            movieFromDb.LastUpdatedDate = DateTime.Now;

            db.Movies.Update(movieFromDb);
            db.SaveChanges();

            //Evict this entry from cache
            memoryCache.Remove(MemoryCacheKeyGenerator.Generate(MemoryCacheKey.MOVIE_BY_ID, id));
            memoryCache.Remove(MemoryCacheKey.MOVIES_ALL.ToString());

            return movieFromDb;
        }

        public void DeleteMovie(string id) {
            Movie movieFromDb = db.Movies.Find(id);

            if (movieFromDb == null) {
                return;
            }

            db.Movies.Remove(movieFromDb);
            db.SaveChanges();

            //Evict this entry from cache
            memoryCache.Remove(MemoryCacheKeyGenerator.Generate(MemoryCacheKey.MOVIE_BY_ID, id));
            memoryCache.Remove(MemoryCacheKey.MOVIES_ALL.ToString());
        }
    }
}
```

Now for testing purposes, run your app and call twice the endpoint `GetAllMovies`. Watch closely the logs, you will only see one call to database been made by the EF Core engine, not two.

[Reference](https://docs.microsoft.com/en-us/aspnet/core/performance/caching/memory?view=aspnetcore-2.2)

## Relationships

We have now understand the basics about EF Core, it's time to add some more tables so we can practice in how to make relationships between them.

We have a `movies`, so let's add `actors` and `producers` too. A movie has one producer and producers can have many movies, this is a one to many relationship. In a movie can participate many actors, and an actor can participate in many movies, since this is a many to many relationship we need a third table for storing this relationship.

Let's begin adding the Models, create these new 3 models under the `src\Models` directory and update the `Movie` model:

`src\Models\Actor.cs`
```csharp
using System;
using System.Collections.Generic;

namespace MoviesWebApi.Models {
    public class Actor {
        public Actor() {
            Id = Guid.NewGuid().ToString();
            CreatedDate = DateTime.Now;
            LastUpdatedDate = DateTime.Now;
        }
        public string Id { get; set; }
        public string Name { get; set; }
        public DateTime CreatedDate { get; set; }
        public DateTime LastUpdatedDate { get; set; }
        /**Navigation Properties**/
        public ICollection<MovieActor> MovieActors { get; set; } //Many to Many
    }
}
```

`src\Models\Studio.cs`
```csharp
using System;
using System.Collections.Generic;

namespace MoviesWebApi.Models {
    public class Studio {
        public Studio() {
            Id = Guid.NewGuid().ToString();
            CreatedDate = DateTime.Now;
            LastUpdatedDate = DateTime.Now;
        }

        public string Id { get; set; }
        public string Name { get; set; }
        public DateTime CreatedDate { get; set; }
        public DateTime LastUpdatedDate { get; set; }
        /**Navigation Properties**/
        public ICollection<Movie> Movies { get; set; } //One to Many
    }
}
```

`src\Models\MovieActor.cs`
```csharp
using System;

namespace MoviesWebApi.Models {
    public class MovieActor {
        public string MovieId { get; set; }
        public string ActorId { get; set; }
        public DateTime CreatedDate { get; set; }
        /**Navigation Properties**/
        public Movie Movie { get; set; }
        public Actor Actor { get; set; }
    }
}
```

`src\Models\Movie.cs`
```csharp
using System;
using System.Collections.Generic;

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
        public string StudioId { get; set; }

        // These fields are here to know when this record
        // was first created and when it was last updated
        public DateTime CreatedDate { get; set; }
        public DateTime LastUpdatedDate { get; set; }

        /**Navigation Properties**/
        public Studio Studio { get; set; }                //One to Many
        public ICollection<MovieActor> MovieActors { get; set; } //Many to Many
    }
}
```

Add 2 new controllers with their respective services.

`src\Conrollers\ActorsController.cs`
```csharp
using Microsoft.AspNetCore.Mvc;
using MoviesWebApi.Data;
using MoviesWebApi.Models;
using MoviesWebApi.Services;

namespace MoviesWebApi.Controllers {

    [Route("api/actors")]
    [ApiController]
    public class ActorsController : ControllerBase {

        private IActorsService actorsService;

        public ActorsController(IActorsService actorsService) {
            this.actorsService = actorsService;
        }

        [HttpPost]
        //POST api/actors
        //Payload: A Json representing the Actor object
        public ActionResult<Actor> CreateActor(Actor actor) {
            return actorsService.CreateActor(actor);
        }

    }
}
```

`src\Services\IActorsService.cs`
```csharp
using MoviesWebApi.Models;

namespace MoviesWebApi.Services {
    public interface IActorsService {
        Actor CreateActor(Actor actor);
    }
}
```

`src\Services\ActorsService.cs`
```csharp
using MoviesWebApi.Data;
using MoviesWebApi.Models;

namespace MoviesWebApi.Services {
    public class ActorsService : IActorsService {
        private AppDbContext db;

        public ActorsService(AppDbContext db) {
            this.db = db;
        }

        public Actor CreateActor(Actor actor) {
            db.Actors.Add(actor);
            db.SaveChanges();
            return actor;
        }
    }
}
```

`src\Conrollers\StudiosController.cs`
```csharp
using Microsoft.AspNetCore.Mvc;
using MoviesWebApi.Models;
using MoviesWebApi.Services;

namespace MoviesWebApi.Controllers {
    [Route("api/studios")]
    [ApiController]
    public class StudiosController : ControllerBase {
        private IStudiosService studiosService;

        public StudiosController(IStudiosService studiosService) {
            this.studiosService = studiosService;
        }

        [HttpPost]
        //POST api/studios
        //Payload: A Json representing the Studio object
        public Studio CreateStudio(Studio studio) {
            return studiosService.CreateStudio(studio);
        }
    }
}
```

`src\Services\IStudiosService.cs`
```csharp
using MoviesWebApi.Models;

namespace MoviesWebApi.Services {
    public interface IStudiosService {
        Studio CreateStudio(Studio studio);
    }
}
```

`src\Services\StudiosService.cs`
```csharp
using MoviesWebApi.Data;
using MoviesWebApi.Models;

namespace MoviesWebApi.Services {
    public class StudiosService : IStudiosService {
        private AppDbContext db;

        public StudiosService(AppDbContext db) {
            this.db = db;
        }

        public Studio CreateStudio(Studio studio) {
            db.Studios.Add(studio);
            db.SaveChanges();
            return studio;
        }
    }
}
```

Add a new endpoint and it's respective service method to movies (for inserting actors into a movie):
`src\Conrollers\MoviesController.cs`
```csharp
using System.Collections.Generic;
using Microsoft.AspNetCore.Mvc;
using MoviesWebApi.Models;
using MoviesWebApi.Services;

namespace MoviesWebApi.Controllers {

    [Route("api/movies")]
    [ApiController]
    public class MoviesController : ControllerBase {

        private IMoviesService moviesService;

        public MoviesController(IMoviesService moviesService) {
            this.moviesService = moviesService;
        }

        [HttpGet]
        //GET api/movies
        public ActionResult<IEnumerable<Movie>> GetAllMovies() {
            return moviesService.GetAllMovies();
        }

        [HttpGet("{id}")]
        //GET api/movies/{id}
        public ActionResult<Movie> GetMovie(string id) {
            Movie movie = moviesService.GetMovie(id);

            if (movie == null) {
                return NotFound();
            }

            return movie;
        }

        [HttpPost]
        //POST api/movies
        //Payload: A Json representing the Movie object
        public ActionResult<Movie> CreateMovie(Movie movie) {
            return moviesService.CreateMovie(movie);
        }

        [HttpPut("{id}")]
        //PUT api/movies/{id}
        //Payload: A Json representing the Movie object
        public ActionResult<Movie> UpdateMovie(string id, Movie movie) {
            Movie updatedMovie = moviesService.UpdateMovie(id, movie);

            if (updatedMovie == null) {
                return NotFound();
            }

            return updatedMovie;
        }

        [HttpDelete("{id}")]
        //DELETE api/movies/{id}
        public ActionResult DeleteMovie(string id) {
            moviesService.DeleteMovie(id);
            return NoContent();
        }

        [HttpPut("{movieId}/actors")]
        //PUT api/movies/{movieId}/actors
        //Payload: A Json array of actors ids
        public ActionResult<Movie> updateActors(string movieId, List<string> actorsIds) {
            Movie updatedMovie = moviesService.updateActors(movieId, actorsIds);

            if (updatedMovie == null) {
                return NotFound();
            }

            return updatedMovie;
        }

    }
}
```
`src\Services\IMoviesService.cs`
```csharp
using System.Collections.Generic;
using MoviesWebApi.Models;

namespace MoviesWebApi.Services {
    public interface IMoviesService {
        List<Movie> GetAllMovies();
        Movie GetMovie(string id);
        Movie CreateMovie(Movie movie);
        Movie UpdateMovie(string id, Movie movie);
        void DeleteMovie(string id);
        Movie updateActors(string movieId, List<string> actorsIds);
    }
}
```

`src\Services\MoviesService.cs`
```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Caching.Memory;
using MoviesWebApi.Data;
using MoviesWebApi.MemoryCache;
using MoviesWebApi.Models;

namespace MoviesWebApi.Services {
    public class MoviesService : IMoviesService {

        private AppDbContext db;
        private IMemoryCache memoryCache;

        public MoviesService(AppDbContext db, IMemoryCache memoryCache) {
            this.db = db;
            this.memoryCache = memoryCache;
        }

        //Now we will get all movies from the InMemoryCache which are stored under the key "MOVIES_ALL"
        //If the entry in the cache does not exists (first time, cache was expired or evicted)
        //then it will call the lambda function which receives an ICacheEntry as parameter and returns
        //the values for that entry.
        public List<Movie> GetAllMovies() {
            return memoryCache.GetOrCreate(
                MemoryCacheKey.MOVIES_ALL.ToString(), cacheEntry => {
                    cacheEntry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30);
                    return db.Movies
                        .Include(movie => movie.Studio)
                        .Include(movie => movie.MovieActors)
                        .ThenInclude(movieActor => movieActor.Actor)
                        .ToList();
                });
        }


        public Movie GetMovie(string id) {
            return memoryCache.GetOrCreate(
                MemoryCacheKeyGenerator.Generate(MemoryCacheKey.MOVIE_BY_ID, id),
                cacheEntry => GetMovieFromDb(cacheEntry, id));
        }

        //Instead of creating an anonymous method, call this one for fetching data from db
        private Movie GetMovieFromDb(ICacheEntry cacheEntry, string id) {
            cacheEntry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30);
            return db.Movies
                .Include(movie => movie.Studio)
                .Include(movie => movie.MovieActors)
                .ThenInclude(movieActor => movieActor.Actor)
                .First(movie => movie.Id.Equals(id));
        }

        public Movie CreateMovie(Movie movie) {
            db.Movies.Add(movie);
            db.SaveChanges();
            //Evict cache with all movies
            memoryCache.Remove(MemoryCacheKey.MOVIES_ALL.ToString());
            return movie;
        }

        public Movie UpdateMovie(string id, Movie movie) {
            Movie movieFromDb = db.Movies.Find(id);

            if (movieFromDb == null) {
                return null;
            }

            //Copy all modifiable fields
            movieFromDb.Name = movie.Name;
            movieFromDb.Year = movie.Year;
            movieFromDb.Genre = movie.Genre;
            movieFromDb.Duration = movie.Duration;
            movieFromDb.StudioId = movie.StudioId;
            movieFromDb.LastUpdatedDate = DateTime.Now;

            db.Movies.Update(movieFromDb);
            db.SaveChanges();

            //Evict this entry from cache
            memoryCache.Remove(MemoryCacheKeyGenerator.Generate(MemoryCacheKey.MOVIE_BY_ID, id));
            memoryCache.Remove(MemoryCacheKey.MOVIES_ALL.ToString());

            return movieFromDb;
        }

        public void DeleteMovie(string id) {
            Movie movieFromDb = db.Movies.Find(id);

            if (movieFromDb == null) {
                return;
            }

            db.Movies.Remove(movieFromDb);
            db.SaveChanges();

            //Evict movies from cache
            memoryCache.Remove(MemoryCacheKeyGenerator.Generate(MemoryCacheKey.MOVIE_BY_ID, id));
            memoryCache.Remove(MemoryCacheKey.MOVIES_ALL.ToString());
        }

        public Movie updateActors(string movieId, List<string> actorsIds) {
            var movieFromDb = db.Movies.Find(movieId);
            //Check if movie Id exists
            if (movieFromDb == null) {
                return null;
            }

            //Verify if all actors exists and build the actors to insert
            var movieActorsToInsert = new List<MovieActor>();
            foreach (var actorId in actorsIds) {
                var actor = db.Actors.Find(actorId);
                if (actor == null) {
                    //If an actor does not exist, return
                    return null;
                }
                var movieActor = new MovieActor() {
                    MovieId = movieFromDb.Id,
                    ActorId = actor.Id
                };
                movieActorsToInsert.Add(movieActor);
            }

            //Delete all records in MovieActors for this movieId
            var movieActorsToRemove = db.MovieActors.Where(movieActor => movieActor.MovieId == movieId).ToList();
            db.MovieActors.RemoveRange(movieActorsToRemove);
            db.SaveChanges();

            //Save the new relationships
            db.MovieActors.AddRange(movieActorsToInsert);
            db.SaveChanges();

            //Evict this entry from cache
            memoryCache.Remove(MemoryCacheKeyGenerator.Generate(MemoryCacheKey.MOVIE_BY_ID, movieId));

            //Return a refreshed instance of Movie
            return GetMovie(movieId);
        }
    }
}
```

Register the newly created services into Startup and tweak a little the JSON Serializer.

`Startup.cs`
```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.DependencyInjection.Extensions;
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

            app.UseMvc();
        }
    }
}
```

Create a new `Migration` and apply it to the database:
```
dotnet ef migrations add AddingMoreRelationships
dotnet ef database update
```
You will see how the new column `StudioId` was added to the table Movies as an FK, and how all the tables were created with their respective relationships.

## Return Resources instead of Entities

If you see the serialized JSON response for a `Movie` entity you may notice something odd, the field `movieActors` holds an array of a serialized `MovieActors` objects, where the `Movie` part is always the same, which is the parent `Movie` object. The same happening with the `Studio` object.
This is very useful for Entity objects managed by `EF Core` but not that useful for exposing that data as a response of a REST web service.

A more useful approach would be to only have the fields `actors` and `studio` inside the `Movie` object.

For doing this, we need to have a dedicated object to be serialized for each entity type object, these dedicated objects are sometimes called `DTO` (data transfer object).

For example instead of returning:

`GET http://localhost:5000/api/movies/`
```javascript
[
    {
        "id": "f8cafcac-0384-422a-b551-a7cdfcc1ee0e",
        "name": "Sword of the Strangers updated",
        "genre": "Anime",
        "year": 2001,
        "duration": 123,
        "studioId": "380a04fd-4408-4171-9ca4-7d1269b153fc",
        "createdDate": "2019-05-29T10:27:09.8873847",
        "lastUpdatedDate": "2019-05-29T10:27:09.9161859",
        "studio": {
            "id": "380a04fd-4408-4171-9ca4-7d1269b153fc",
            "name": "Bones",
            "createdDate": "2019-06-06T22:59:47.0024312",
            "lastUpdatedDate": "2019-06-06T22:59:47.0025235",
            "movies": []
        },
        "movieActors": [
            {
                "movieId": "f8cafcac-0384-422a-b551-a7cdfcc1ee0e",
                "actorId": "9f1fd5b1-0f01-4dc8-9c57-91aee00655de",
                "createdDate": "0001-01-01T00:00:00",
                "actor": {
                    "id": "9f1fd5b1-0f01-4dc8-9c57-91aee00655de",
                    "name": "Sweeney Todd",
                    "createdDate": "2019-05-29T13:38:16.9242563",
                    "lastUpdatedDate": "2019-05-29T13:38:16.9243121",
                    "movieActors": []
                }
            }
        ]
    }
]
```

Better return:
```javascript
[
    {
        "id": "f8cafcac-0384-422a-b551-a7cdfcc1ee0e",
        "name": "Sword of the Stranger",
        "genre": "Anime",
        "year": 2001,
        "duration": 122,
        "createdDate": "2019-05-29T10:27:09.8873847",
        "lastUpdatedDate": "2019-05-29T10:27:09.9161859",
        "studio": {
            "id": "380a04fd-4408-4171-9ca4-7d1269b153fc",
            "name": "Bones",
            "createdDate": "2019-06-06T22:59:47.0024312",
            "lastUpdatedDate": "2019-06-06T22:59:47.0025235"
        },
        "actors": [
            {
              "id": "9f1fd5b1-0f01-4dc8-9c57-91aee00655de",
              "name": "Sweeney Todd",
              "createdDate": "2019-05-29T13:38:16.9242563",
              "lastUpdatedDate": "2019-05-29T13:38:16.9243121"
            }
        ]
    }
]
```

Now, create these `DTO` classes in a new folder in `src/Dto`:

`src\Dto\StudioDto.cs`
```csharp
using System;

namespace MoviesWebApi.Dto {
    public class StudioDto {
        public string Id { get; set; }
        public string Name { get; set; }
        public DateTime CreatedDate { get; set; }
        public DateTime LastUpdatedDate { get; set; }
    }
}
```

`src\Dto\ActorDto.cs`
```csharp
using System;

namespace MoviesWebApi.Dto {
    public class ActorDto {
        public string Id { get; set; }
        public string Name { get; set; }
        public DateTime CreatedDate { get; set; }
        public DateTime LastUpdatedDate { get; set; }
    }
}
```

`src\Dto\MovieDto.cs`
```csharp
using System;
using System.Collections.Generic;

namespace MoviesWebApi.Dto {
    public class MovieDto {
        public string Id { get; set; }
        public string Name { get; set; }
        public string Genre { get; set; }
        public int Year { get; set; }
        public int Duration { get; set; }
        public StudioDto Studio { get; set; }
        public List<ActorDto> Actors { get; set; }
        public DateTime CreatedDate { get; set; }
        public DateTime LastUpdatedDate { get; set; }
    }
}
```

Create a static `utils` class that will map the entity object to a DTO type object.

`src\Utils\Mapper.cs`
```csharp
using System.Collections.Generic;
using MoviesWebApi.Dto;
using MoviesWebApi.Models;

namespace MoviesWebApi.Utils {
    public static class Mapper {

        public static ActorDto MapActor(Actor actor) {
            var actorDto = new ActorDto();
            actorDto.Id = actor.Id;
            actorDto.Name = actor.Name;
            actorDto.LastUpdatedDate = actor.LastUpdatedDate;
            actorDto.CreatedDate = actor.CreatedDate;
            return actorDto;
        }

        public static StudioDto MapStudio(Studio studio) {
            var studioDto = new StudioDto() {
                Id = studio.Id,
                Name = studio.Name,
                LastUpdatedDate = studio.LastUpdatedDate,
                CreatedDate = studio.CreatedDate
            };
            return studioDto;
        }

        public static MovieDto MapMovie(Movie movie) {
            var movieDto = new MovieDto() {
                Id = movie.Id,
                Name = movie.Name,
                Genre = movie.Genre,
                Year = movie.Year,
                Duration = movie.Duration,
                LastUpdatedDate = movie.LastUpdatedDate,
                CreatedDate = movie.CreatedDate
            };

            if (movie.Studio != null) {
                movieDto.Studio = MapStudio(movie.Studio);
            }

            if (movie.MovieActors != null && movie.MovieActors.Count > 0) {
                movieDto.Actors = new List<ActorDto>();
                foreach (var movieActor in movie.MovieActors) {
                    movieDto.Actors.Add(MapActor(movieActor.Actor));
                }
            }

            return movieDto;
        }
    }
}
```

Now update the Controller and Services to return the DTO object instead of the entity object, also add the method that will map the dto object

`src\Controllers\MoviesController.cs`
```csharp
using System.Collections.Generic;
using Microsoft.AspNetCore.Mvc;
using MoviesWebApi.Dto;
using MoviesWebApi.Models;
using MoviesWebApi.Services;

namespace MoviesWebApi.Controllers {

    [Route("api/movies")]
    [ApiController]
    public class MoviesController : ControllerBase {

        private IMoviesService moviesService;

        public MoviesController(IMoviesService moviesService) {
            this.moviesService = moviesService;
        }

        [HttpGet]
        //GET api/movies
        public ActionResult<IEnumerable<MovieDto>> GetAllMovies() {
            return moviesService.GetAllMovies();
        }

        [HttpGet("{id}")]
        //GET api/movies/{id}
        public ActionResult<MovieDto> GetMovie(string id) {
            var movie = moviesService.GetMovie(id);

            if (movie == null) {
                return NotFound();
            }

            return movie;
        }

        [HttpPost]
        //POST api/movies
        //Payload: A Json representing the Movie object
        public ActionResult<MovieDto> CreateMovie(Movie movie) {
            return moviesService.CreateMovie(movie);
        }

        [HttpPut("{id}")]
        //PUT api/movies/{id}
        //Payload: A Json representing the Movie object
        public ActionResult<MovieDto> UpdateMovie(string id, Movie movie) {
            var updatedMovie = moviesService.UpdateMovie(id, movie);

            if (updatedMovie == null) {
                return NotFound();
            }

            return updatedMovie;
        }

        [HttpDelete("{id}")]
        //DELETE api/movies/{id}
        public ActionResult DeleteMovie(string id) {
            moviesService.DeleteMovie(id);
            return NoContent();
        }

        [HttpPut("{movieId}/actors")]
        //PUT api/movies/{movieId}/actors
        //Payload: A Json array of actors ids
        public ActionResult<MovieDto> updateActors(string movieId, List<string> actorsIds) {
            var updatedMovie = moviesService.UpdateActors(movieId, actorsIds);

            if (updatedMovie == null) {
                return NotFound();
            }

            return updatedMovie;
        }

    }
}
```

`src\Services\IMoviesService`
```csharp
using System.Collections.Generic;
using MoviesWebApi.Dto;
using MoviesWebApi.Models;

namespace MoviesWebApi.Services {
    public interface IMoviesService {
        List<MovieDto> GetAllMovies();
        MovieDto GetMovie(string id);
        MovieDto CreateMovie(Movie movie);
        MovieDto UpdateMovie(string id, Movie movie);
        void DeleteMovie(string id);
        MovieDto UpdateActors(string movieId, List<string> actorsIds);
    }
}
```

`src\Services\MoviesService`
```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Caching.Memory;
using MoviesWebApi.Data;
using MoviesWebApi.Dto;
using MoviesWebApi.MemoryCache;
using MoviesWebApi.Models;
using MoviesWebApi.Utils;

namespace MoviesWebApi.Services {
    public class MoviesService : IMoviesService {

        private AppDbContext db;
        private IMemoryCache memoryCache;

        public MoviesService(AppDbContext db, IMemoryCache memoryCache) {
            this.db = db;
            this.memoryCache = memoryCache;
        }

        //Now we will get all movies from the InMemoryCache which are stored under the key "MOVIES_ALL"
        //If the entry in the cache does not exists (first time, cache was expired or evicted)
        //then it will call the lambda function which receives an ICacheEntry as parameter and returns
        //the values for that entry.
        public List<MovieDto> GetAllMovies() {
            return memoryCache.GetOrCreate(
                MemoryCacheKey.MOVIES_ALL.ToString(), cacheEntry => {
                    cacheEntry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30);
                    return db.Movies
                        .Include(movie => movie.Studio)
                        .Include(movie => movie.MovieActors)
                        .ThenInclude(movieActor => movieActor.Actor)
                        .ToList()
                        .ConvertAll(Mapper.MapMovie);
                });
        }


        public MovieDto GetMovie(string id) {
            var movie = memoryCache.GetOrCreate(
                MemoryCacheKeyGenerator.Generate(MemoryCacheKey.MOVIE_BY_ID, id),
                cacheEntry => GetMovieFromDb(cacheEntry, id));
            return Mapper.MapMovie(movie);
        }

        //Instead of creating an anonymous method, call this one for fetching data from db
        private Movie GetMovieFromDb(ICacheEntry cacheEntry, string id) {
            cacheEntry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30);
            return db.Movies
                .Include(movie => movie.Studio)
                .Include(movie => movie.MovieActors)
                .ThenInclude(movieActor => movieActor.Actor)
                .First(movie => movie.Id.Equals(id));
        }

        public MovieDto CreateMovie(Movie movie) {
            db.Movies.Add(movie);
            db.SaveChanges();
            //Evict cache with all movies
            memoryCache.Remove(MemoryCacheKey.MOVIES_ALL.ToString());
            return Mapper.MapMovie(movie);
        }

        public MovieDto UpdateMovie(string id, Movie movie) {
            Movie movieFromDb = db.Movies.Find(id);

            if (movieFromDb == null) {
                return null;
            }

            //Copy all modifiable fields
            movieFromDb.Name = movie.Name;
            movieFromDb.Year = movie.Year;
            movieFromDb.Genre = movie.Genre;
            movieFromDb.Duration = movie.Duration;
            movieFromDb.StudioId = movie.StudioId;
            movieFromDb.LastUpdatedDate = DateTime.Now;

            db.Movies.Update(movieFromDb);
            db.SaveChanges();

            //Evict this entry from cache
            memoryCache.Remove(MemoryCacheKeyGenerator.Generate(MemoryCacheKey.MOVIE_BY_ID, id));
            memoryCache.Remove(MemoryCacheKey.MOVIES_ALL.ToString());

            //Return a refreshed instance of Movie
            return GetMovie(id);
        }

        public void DeleteMovie(string id) {
            Movie movieFromDb = db.Movies.Find(id);

            if (movieFromDb == null) {
                return;
            }

            db.Movies.Remove(movieFromDb);
            db.SaveChanges();

            //Evict movies from cache
            memoryCache.Remove(MemoryCacheKeyGenerator.Generate(MemoryCacheKey.MOVIE_BY_ID, id));
            memoryCache.Remove(MemoryCacheKey.MOVIES_ALL.ToString());
        }

        public MovieDto UpdateActors(string movieId, List<string> actorsIds) {
            var movieFromDb = db.Movies.Find(movieId);
            //Check if movie Id exists
            if (movieFromDb == null) {
                return null;
            }

            //Verify if all actors exists and build the actors to insert
            var movieActorsToInsert = new List<MovieActor>();
            foreach (var actorId in actorsIds) {
                var actor = db.Actors.Find(actorId);
                if (actor == null) {
                    //If an actor does not exist, return
                    return null;
                }
                var movieActor = new MovieActor() {
                    MovieId = movieFromDb.Id,
                    ActorId = actor.Id
                };
                movieActorsToInsert.Add(movieActor);
            }

            //Delete all records in MovieActors for this movieId
            var movieActorsToRemove = db.MovieActors.Where(movieActor => movieActor.MovieId == movieId).ToList();
            db.MovieActors.RemoveRange(movieActorsToRemove);
            db.SaveChanges();

            //Save the new relationships
            db.MovieActors.AddRange(movieActorsToInsert);
            db.SaveChanges();

            //Evict this entry from cache
            memoryCache.Remove(MemoryCacheKeyGenerator.Generate(MemoryCacheKey.MOVIE_BY_ID, movieId));

            //Return a refreshed instance of Movie
            return GetMovie(movieId);
        }
    }
}
```


And that's it, run it to test that everything works as expected.
Now your API is returning more meaningful values to the clients that consume it.
