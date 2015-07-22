# Introduction #

The server library is an abstract PHP class that you can subclass to implement an OAuth 2.0 authorization or resource server.

## Why is it abstract? ##

An OAuth 2.0 server needs to persistently store and retrieve various pieces of data, like access tokens and authorization codes.

Our goal is to create a library that handles the details of the OAuth protocol, and lets a user plug in the persistent storage details.

## Can I see an example? ##

We created a very simple OAuth server that uses Mongo DB as a persistent store.  Check out the /server directory in the source.