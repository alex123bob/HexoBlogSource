---
title: Http Status Code 401/403
date: 2018-08-20 14:07:40
tags:
---

I have been asked about the difference between two http status code: 401 and 403. 
Off the top of my head, I will tell ppl 401 means unauthorized request while 403 represents server forbid client's request.
But what the heck does that mean? I have yet thought thoroughly about it.

After doing a few searches, I just put down some key points I learnt from others.
#### **What is 401?**
> The **401 Unauthorized Error** is an HTTP response status code indicating that the request sent by the client could not be authenticated.

I know the definition above is a ~~bullshit~~ coz it's too abstract.
Receiving a 401 response is the server telling you, "you aren’t authenticated–either not authenticated at all or authenticated incorrectly–but please reauthenticate and try again." And your server will help you out by providing a <font color="red"><strong>WWW-Authenticate header</strong></font> that describes how to authenticate.
**Attention:** 401 is told by your server, not from your web application.

#### **In which situation will 401 pop up?**
Most of the time, 401 error is related to client side.
* Client inputs an improper URL that server is not prepared to provide access to.
* Cookie cleaned
* Cache clean
* Log in/Log out

**But** we can't rule server out of 401's root cause.
As modern web servers provide one or more configuration files that allow you to reject requests to some specific directories or ban specific IP address, you will get **401 Unauthorized Error**

#### **What exactly is 403?**
> Receiving a 403 response is the server telling you, "I'm sorry. I know who you are--I believe who you say you are--but you just don’t have permission to access this resource. Maybe if you ask the system administrator nicely, you'll get permission. But please don't bother me again until your predicament changes."

The explanation above is so much better than its original definition.(At least friendly to ppl like me ^_^)

#### **To Sum up**
403 is more concrete, your web application has got your message, and put its own logics in this process, and it's permanent.
While 401 is coming from the server, not your web application. Most of the time, 401 means your request can still be sent to server and pass its authentication and get what you want in the final end as long as you change or put a new credential. 

I'd like to put [Daniel Irvine](http://www.danielirvine.com/)'s quotes to end this post.
> In summary, a 401 Unauthorized response should be used for missing or bad authentication, and a 403 Forbidden response should be used afterwards, when the user is authenticated but isn’t authorized to perform the requested operation on the given resource.


Reference docs for this post:
- [Understanding 403 Forbidden](http://www.dirv.me/blog/2011/07/18/understanding-403-forbidden/index.html)