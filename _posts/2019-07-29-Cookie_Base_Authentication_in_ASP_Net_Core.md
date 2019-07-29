---
layout: post
title: "Cookie Base Authentication in ASP.Net Core"
date: 2019-07-29
---

# Cookie Base Authentication in ASP.Net Core (without Asp.Net Core Identity)

So in this post I want to write about how to use Cookies for Authenticating a user but without using Identity. This post is based on the official Microsoft instructions for that: [Use cookie authentication without ASP.NET Core Identity](dotnet new webapp -o CookieBasedWebApp).

If you do not know what's Asp.net Core Identity check this page: [Introduction to Identity on ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/identity?view=aspnetcore-2.2&tabs=visual-studio). Basically Identity is an awesome piece of code created by Microsoft that allows you to add Cookie Based Authentication to you Razor Pages project, all User and Roles are going to be handled for you along with database interaction, you can even scaffold the user authentication pages so you don't even need to write code for that.

But some times when you are creating a Web System that needs to integrate with some legacy code and pre-existing user data in a database then you don't want all the auto-generated code of Identity  since you don't want new tables to be created for handling users and roles since they are already there.

Here is the full code developed for this example: [hmojicag/Cookie_Base_Authentication_in_ASP_Net_Core](https://github.com/hmojicag/Cookie_Base_Authentication_in_ASP_Net_Core)

Let's start by creating a new `webapp` project:

```
dotnet new webapp -o CookieBasedWebApp
```

First thing we need is to add the `Authentication Middleware Service`, your `Startup` class may end up looking like, the new code added to `Startup` is enclosed with comments and it's self descriptive.

```csharp
using System;
using Microsoft.AspNetCore.Authentication.Cookies;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;

namespace Surfingweb {
    public class Startup {
        public Startup(IConfiguration configuration) {
            Configuration = configuration;
        }

        public IConfiguration Configuration { get; }

        // This method gets called by the runtime. Use this method to add services to the container.
        public void ConfigureServices(IServiceCollection services) {

            /*-------Adding Cookie Based Authentication-------*/
            services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme).AddCookie();

            services.Configure<CookiePolicyOptions>(options => {
                // This lambda determines whether user consent for non-essential cookies is needed for a given request.
                options.CheckConsentNeeded = context => true;
                options.MinimumSameSitePolicy = SameSiteMode.Strict;
            });

            services.ConfigureApplicationCookie(options => {
                options.ExpireTimeSpan = TimeSpan.FromHours(1);
                options.LoginPath = "/Account/Login";                //Redirect to this Path if user is not Authenticated
                options.AccessDeniedPath = "/Account/AccessDenied";  //Redirect to this Path if user does not have permissions for a page
                options.LogoutPath = "/Account/Logout";              //POST to this path to logout
            });
            /*-------Adding Cookie Based Authentication-------*/

            services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_2_2);
        }

        // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
        public void Configure(IApplicationBuilder app, IHostingEnvironment env) {
            if (env.IsDevelopment()) {
                app.UseDeveloperExceptionPage();
            }
            else {
                app.UseExceptionHandler("/Error");
            }

            app.UseStaticFiles();
            /*-------Adding Cookie Based Authentication-------*/
            app.UseCookiePolicy();
            app.UseAuthentication();
            /*-------Adding Cookie Based Authentication-------*/
            app.UseMvc();
        }
    }
}
```

Now let's build a Login Page:

`/Pages/Account/Login.cshtml`
```csharp
@page
@model LoginModel
@{
    ViewData["Title"] = "Login";
}

<div class="row">
  <div class="col">
    <!--https://getbootstrap.com/docs/4.0/components/forms/-->
    <form method="post">
      <div class="form-group">
        <label for="exampleInputEmail1">Email address</label>
        <input type="email" class="form-control" name="email" id="exampleInputEmail1" aria-describedby="emailHelp" placeholder="Enter email">
        <small id="emailHelp" class="form-text text-muted">We'll never share your email with anyone else.</small>
      </div>
      <div class="form-group">
        <label for="exampleInputPassword1">Password</label>
        <input type="password" name="password" class="form-control" id="exampleInputPassword1" placeholder="Password">
      </div>
      <div class="form-check">
        <input type="checkbox" class="form-check-input" id="exampleCheck1">
        <label class="form-check-label" for="exampleCheck1">Check me out</label>
      </div>
      <button type="submit" class="btn btn-primary">Submit</button>
    </form>
  </div>
</div>
```

`/Pages/Account/Login.cshtml.cs`
```csharp
using System.Collections.Generic;
using System.Security.Claims;
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Authentication.Cookies;
using Microsoft.AspNetCore.Mvc.RazorPages;

namespace CookieBasedWebApp.Pages.Account {
    public class LoginModel : PageModel {
        public void OnGet() {

        }

        public void OnPost(string email, string password) {

            if (!string.Equals(email, "awesome@email.com") || string.Equals(password, "SuperStrongPassword1234")) {
                return;
            }

            //Test different Roles: Admin, MortalUser, Maintainer
            var claims = new List<Claim> {
                new Claim(ClaimTypes.Name, email),
                new Claim("FullName", email),
                new Claim(ClaimTypes.Role, "MortalUser,Maintainer")//This user has 2 roles
            };

            var claimsIdentity = new ClaimsIdentity(
                claims, CookieAuthenticationDefaults.AuthenticationScheme);

            var authProperties = new AuthenticationProperties {
                // Refreshing the authentication session should be allowed.
                AllowRefresh = true,

                // The time at which the authentication ticket expires. A
                // value set here overrides the ExpireTimeSpan option of
                // CookieAuthenticationOptions set with AddCookie.
                //ExpiresUtc = DateTimeOffset.UtcNow.AddMinutes(120),

                // Whether the authentication session is persisted across
                // multiple requests. When used with cookies, controls
                // whether the cookie's lifetime is absolute (matching the
                // lifetime of the authentication ticket) or session-based.
                IsPersistent = true

                //IssuedUtc = <DateTimeOffset>,
                // The time at which the authentication ticket was issued.

                //RedirectUri = <string>
                // The full path or absolute URI to be used as an http
                // redirect response value.
            };

            HttpContext.SignInAsync(
                CookieAuthenticationDefaults.AuthenticationScheme,
                new ClaimsPrincipal(claimsIdentity),
                authProperties);
        }

    }
}
```

Before you test, add some logic to `Index` to recognize the user that is logged. And also add the `[Authorize]` attribute to denote that nobody can access this page if it's not authenticated first.

`/Pages/Index.cshtml`
```csharp
@page
@model IndexModel
@{
    ViewData["Title"] = "Home page";
}

<div class="text-center">
    <h1 class="display-4">Welcome</h1>
    <h2>@Model.LoggedUserName</h2>
    <form method="post" asp-page="/Account/Logout" asp-route-returnUrl="@Url.Page("/Account/Login")">
        <input type="submit" value="LogOut"/>
    </form>
</div>



```

`/Pages/Index.cshtml.cs`
```csharp
using System.Security.Claims;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc.RazorPages;

namespace CookieBasedWebApp.Pages {

    [Authorize]
    public class IndexModel : PageModel {
        public string LoggedUserName = "";

        public void OnGet() {
            var claimEmail = HttpContext.User.FindFirst(ClaimTypes.Name);
            var claimFullName = HttpContext.User.FindFirst("FullName");
            if (claimEmail != null && claimFullName != null) {
                LoggedUserName = $"{claimEmail.Value} - {claimFullName.Value}";
            }
        }
    }
}
```

Now test, run your app and try to go to `http://localhost:5000` (or whatever port you are using) which will try to access `/Index` and since you are not logged in it will redirect you to `/Account/Login` now introduce the hardcoded credentials, hit Submit and watch how it logs you in and takes you to `/Index` where the your name (well, the hardcoded user name) will be displayed.

Also try the log out button.

Now let's finish this post by adding some extra features.

Create the `AccessDenied` page:

`/Account/AccessDenied.cshtml`
```csharp
@page
@model AccessDeniedModel

<div class="text-center">
    <h1 class="display-4">You are a mere mortal, you cannot access this page</h1>
</div>
```

`/Account/AccessDenied.cshtml.cs`
```csharp
using Microsoft.AspNetCore.Mvc.RazorPages;

namespace CookieBasedWebApp.Pages.Account {
    public class AccessDeniedModel : PageModel {
        public void OnGet() {

        }
    }
}
```

Now add a page that only Admins can see so you test this AccessDenied feature.

`/AdminOnly.cshtml`
```csharp
@page
@model AdminOnlyModel

@{
    ViewData["Title"] = "Admin Only page";
}

<div class="text-center">
    <h1 class="display-4">Only Admins here</h1>
</div>
```

`/AdminOnly.cshtml.cs`
```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc.RazorPages;

namespace CookieBasedWebApp.Pages {
    [Authorize(Roles = "Admin")]
    public class AdminOnlyModel : PageModel {
        public void OnGet() {

        }
    }
}
```

Awesome, now run again the code, log in and try to access `/AdminOnly`, you should get redirected to `/Account/AccessDenied`.

And that's it, you added Authentication to your page, be happy now.
