# Hk Test | Task 2 - Solution

## Problem description

The problem is, that the endpoint ```temperature``` is either returning the temperature ("13.5") or throwing an internal server error (HTTP-Statuscode: 500). So the api-endpoint is not reliable, because the response is changing intermittently between server error and success (HTTP-Statuscode 200).

### HTTP-Statuscode: 200
![success response](success.png)

### Internal Server Error (HTTP-Statuscode: 500)
![internal error response](error.png)

This response code indicates that the server encountered an unexpected condition that prevented it from fulfilling the request.

The server couldn't find any better response code - such as 501 Not Implemented, 502 Bad Gateway, (...) .

### Debugging

There are multiple ways to debug this problem:

1. Check the logs from heroku with ```heroku logs``` or logs in heroku-dashboard
2. If the app is not hosted on heroku (like in this case), check the `nginx` or `apache` logs, or also check the application-logfiles.

### Analysis of this case

This problem should be solvable without touching the codebase. That means this intermittent internal server error is e.g. not related to a script with bad performance or bad design.

There was a similiar case, which occured during the work on [aufgabe1](https://gitlab.christianratz.de/christian/hk-test-aufgabe-1). 

The endpoint `PUT https://hk-test-api.herokuapp.com/albums/:id` needs an `Accept`-header, such as `Accept: application/json` or `Accept: */*`, otherwise the api will through also an internal server error (HTTP-Statuscode `500`). 

The difference in this case is, that the server always returned an internal server error, as soon as there will be set an explicit header. 

In my opinion, this additional header (with an explicit content type such as `Accept: application/json`), so an additional constraint for the api-endpoint, is more stable and the frontend, which is requesting the api-endpoint can be sure, that the response from the api is reliable and in the desired content type.

In this case, for task2, the server returns intermittently an internal server error.

As far as i know heroku can easy handle higher loads of users, so it shouldn't be a problem with exceeded memory or overload of the infrastructure. Also it shouldn't be a problem with the underlying cowboy which is used from heroku as a http-server to provide all apps.

So it can't be a problem with apache or nginx and a fastcgi-timeout, because cowboy is used to serve all applications hosted on heroku.

The only reasons i can imagine or which came in mind, where issues with an unreliable mysql-server? Maybe the temperature will be read from a database. If an error such as [mysql server has gone away](https://dev.mysql.com/doc/refman/8.0/en/gone-away.html) occurs, the server will throw an internal server error. Another possible case could be if there is another service which is updating the temperature-entry in the database, so there will be a small timeframe of a lock during `INSERT`, and if in this timeframe someone requests the temperature from database, an internal server error will be thrown [deadlock](https://dev.mysql.com/doc/refman/8.0/en/innodb-deadlocks.html).
* solution: change configuration of the mysql-server, use a master(write)-slave(read)-configuration with two mysql-servers or for example a mysql-server (write) in combination with a redis-cache (read).

### Another application

I experienced or faced already issues with similiar behaviour with laravel, where the server randomly threw internal server errors, when requesting an endpoint. The solution was a misconfiguration / corrupt config-files.
* solution: fix the misconfiguration or use another framework

Besides it could be also a misconfiguration of apache or nginx. Also when the `fastcgi-process` exceeded and timeout. Other possible reason could be misconfiguration with apache and `mod_http2`.

* solution: fix the misconfiguration or use another http-server

The underlying infrastructure behind an application is simply overloaded or to weak, because of high user-load or exceeded memory. In this case users can also possibly face issues with intermittent internal server errors.
* solution: use a more powerful server or switch to another subscription or another server provider

### UPDATE: Alternative: Handle unreliable API on frontend-side (within the application)

In my opinion we have the following solutions for an unreliable api: 

When sending a request to the api and it will return an error like this, we could implement an retry-functionality for requests, with an increasing delay each time the request fails. 

Extending this with an additional check after an executed request, so for example the album administrator: 
We send a POST-Request / or PUT-Request to insert or update an entity. The api response failed -> internal server error.
In this case, we can not be sure, if the entity was already inserted or updated, so before we retry the request, we could send another request to get the information about the inserted or updated entity and check if its already exists or has already been updated. If that’s not the case, we retry the failed request again.

The retry-options should have an “exit”-case, so maybe after five failed requests, definitely inform / alarm about those five failed request and try to fix it - 1. inform the users of your application, 2. check again the api-documentation for any changes, 3. inform the api-provider, and so on..

If the api-provider is not fixing errors or not at least trying to fix them, it could be also a possibility to review the usage or necessity of this api until the errors are handled.

No matter which way you take to handle an unreliable api or another unreliable module of your application, the goal which should be focused on, that you provide an application which is reliable. 

Another completely different case, could be, when the api-provider updated the logic behind the api-endpoint - which would be a really bad decision, best practice for apis is definitely versioning.
The difference is, that these cases could be mostly identified by an 4xx error.

