---
layout: course
title: Web API with ASP.Net Core - Workshop
date: 2019-05-26
---

Hello I'm Hazael Mojica and this is the main page of the workshop in how to build a WebAPI using ASP.Net. Core.

# [Course Content]({{ site.baseurl }}{% link workshop-webapi-aspnetcore/index.md %})

We will start with an introduction to .Net Core and C# in general, then we will proceed to the ASP.Net Core environment and how it will work for out purposes.
After we have learned at least a little bit of the tool that we are going to use then we will start learning the basics in the HTTP protocol which dominates the industry nowadays and lastly how to build an API for it.

This course is oriented to experienced developers who have worked with other programming languages and are looking to migrate to C# using the .Net Core environment.

## Course Requirements

1. You can use any computer you like, it could be running Windows, Linux or Mac. Since we are using .Net Core it's ok to use any operative system.
2. You need to have installed the **.Net Core SDK** (latest version is 2.2): [.Net Core SDK](https://dotnet.microsoft.com/download). This is Free.
3. You need a **SQL Server database** instance running for our testing purposes, you can choose **Developer or Express editions (both are free)**. [SQL Server Dowload](https://www.microsoft.com/en-us/sql-server/sql-server-downloads). I recommend [Running SQL Server in Docker](https://hub.docker.com/_/microsoft-mssql-server) as we are going to use it for development only.
4. As an **IDE** you need to have installed either:
4.1. [Visual Studio](https://code.visualstudio.com/docs?dv=win&wt.mc_id=DX_841432&sku=codewin) Windows Only. This is Free.
4.2. [Visual Studio Code](https://code.visualstudio.com/?wt.mc_id=DX_841432). Any Operative System. This is Free.
4.3. Or [Jetbrains Rider] (https://www.jetbrains.com/rider/). Any operative System. This is Paid version.
5. You need a SQL Server Client to manage Database Operations:
5.1 SQL Server Management Studio (latest version 18.0): [SSMS](https://docs.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms?view=sql-server-2017). Windows Only. This is Free
5.2 Or [Azure Data Studio](https://docs.microsoft.com/en-us/sql/azure-data-studio/download?view=sql-server-2017). Any Operative System. This is Free
6. As we are going to develop a Web API we need to have a  tool that makes request to this API, we could of course use a CLI tool like **curl**, but for starters it's better to use [Postman](https://www.getpostman.com/products). This is Free and works for any Operative System.
7. We are going to deploy our API to SmarterASP.Net hosting service, and this is going to be accomplished using FTP. So we need and FTP client like [FileZilla](https://filezilla-project.org/download.php?type=client)
8. And of course we are going to use terminal, and having a good terminal emulator is always a pleasant experience, so if you are using Windows I highly recommend installing [Cmder](https://cmder.net/). Install the full version which includes git. It's free.

## Course Content

1. [Session 1 - Introduction to C# and .Net Core]({{ site.baseurl }}{% link workshop-webapi-aspnetcore/session1.md %})
2. [Session 2 - HTTP and ASP.Net Core]({{ site.baseurl }}{% link workshop-webapi-aspnetcore/session2.md %})
3. [Session 3 - ASP.Net Core and WebAPI Basics]({{ site.baseurl }}{% link workshop-webapi-aspnetcore/session3.md %})
4. [Session 4 - Http Clients and Production Environment]({{ site.baseurl }}{% link workshop-webapi-aspnetcore/session4.md %})
