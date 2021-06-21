---
layout: post
title: "Connect to SQL Server via an SSH Tunnel"
date: 2021-06-17
---

# How to connect to SQL Server via an SSH Tunnel

In this entry I describe how to connect to SQL Server via an SSH Tunnel, a useful knowledge to have in case you ever need to access that SQL Server database that is not accesible from your current network.

## Motivation

Imagine you are at office behind a corporate firewall or restricted network and for some reason you need to access a SQL Server database that is in "the outside" using any client, SSMS (SQL Server Management Studio), SQuirrel, an application you are coding via Entity Framework...

For some reason you can SSH into a server in the outside but port 1433 used by SQL Server is blocked.

I had this same scenario some time ago, I was doing training courses at office because I was off project and needed to test something with SQL Server, I was not able to install a SQL Server instance in my local, nor Docker nor anything useful, I happened to have a SQL Server database hosted in Azure but corporate firewall was blocking that port, and I also happened to have a Windows Server in the outside, this one hosted in Huawei Cloud.

Now you see, I could just told my manager that I was blocked and cannot properly study to pass that interview for the next client I was going to be assigned to or I could do something about it, I had some spare time so I did.

The setup looked as following image:

![](/assets/images/SQLServer_SSH_Tunnel.jpg)


### First step, install a SSH Server in the Jump Server

I call it Jump Server, but this is just lingo, you can basically use any computer you can connect using port 22.
In my case this computer is a Windows Server, so I manually installed an SSH server (OpenSSH) on it by following this guide:

https://www.server-world.info/en/note?os=Windows_Server_2016&p=openssh

Test you can connect to that server:

`ssh user@.hostname.com`

### Modify the hosts file

Change the hosts file located in `C:\Windows\System32\drivers\etc\hosts` appending a line: `127.0.0.1 sqlserverdb.database.windows.net`.

Put the hostname you usally use to connect to database.

What we are doing here is like setting up a local DNS, so every time there's a network call to `sqlserverdb.database.windows.net` in my example that address will get resolved as `127.0.0.1`, that is localhost.

### Do an SSH Tunnel and connect to database

Tunnel all communication for port 1433 to the Jump Server, you can do that by running:

`ssh -L 1433:sqlserverdb.database.windows.net:1433 user@.hostname.com`

Now test again the database connection, this time all communication to `sqlserverdb.database.windows.net` will locally send to `127.0.0.1` into the SSH tunnel which will in tun forward it to the Jump Server and from there actually connect to `sqlserverdb.database.windows.net`.


### Troubleshooting

If you don't use the hosts file and you just connect to your localhost tunneled connection you will get the next error: `A connection was successfully established with the server, but then an error occurred during the pre-login handshake. (provider: SSL Provider, error: 0 - The wait operation timed out.)`. It looks like using `localhost`/`127.0.0.1` or any equivalent won't work because of some sort of SSL authentiation?