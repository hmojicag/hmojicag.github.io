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
1. Defining the Endpoints for our API (Request/response object, status codes and headers)
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



[Reference](https://microchipdeveloper.com/tcpip:tcp-ip-five-layer-model)

## Introduction to HTTP for Web Developer




[Reference](https://developer.mozilla.org/en-US/docs/Web/HTTP)

## Introduction to ASP.Net Core
https://dotnet.microsoft.com/learn/web/what-is-aspnet-core
