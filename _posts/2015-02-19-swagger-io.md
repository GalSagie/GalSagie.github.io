---
layout: post
author: gsagie
title: Introduction to Swagger.io
date: 2015-02-19 16:25:06 -0700
categories:
- API
tags:
- REST
- API
---

API’s are becoming a big thing in the networking world, the rise of SDN and NFV put a big spotlight on API’s in a community where people are used to configure systems using command line interface and SNMP.

We see more and more vendors for networking functions and equipment expose different API’s from the traditional CLI and SNMP.
We start to see all kinds of XML-RPC, JSON-RPC and of course REST API.

I came to realize that SNMP fails to perform as a configuration protocol for many reasons (there is even an IETF document stating these failures), its great for monitoring and simple to implement but certainly not simple to use for the consumers.

You might not agree with this statement, and i am soon going to post a comparison between SNMP and NETCONF with YANG data model, but after spending a lot of time around SNMP there are just better alternatives that fits current times.

One of these alternative is the REST API, if you dont know what REST is, go visit [this site](http://www.restapitutorial.com/lessons/whatisrest.html) for more information.

Creating a good REST API, or any API for that matter, is a very delicate job, you want to keep the API simple, usable and easily extendable while keeping the complexity in the inner implementation and not exposing it in the API  (something that doesnt always happen).

In this post i am presenting a specification/eco-system which will save you a lot of time when building REST API to your device/application/network function.

# [Swagger.io](http://swagger.io)

“Swagger is a simple yet powerful representation of your RESTful API. With the largest ecosystem of API tooling on the planet, thousands of developers are supporting Swagger in almost every modern programming language and deployment environment. With a Swagger-enabled API, you get interactive documentation, client SDK generation and discoverability.”

Basically its a text file that you easily express your REST API in a specific format and then have an entire ecosystem of tools that help you create use full implementations from this file.

There is a wide range of use cases that can be easily addressed with the specification file, and since there are many projects and contributions to swagger you can expect many more to come in the future.

I am going to point out some use full use cases and things that can be done with swagger.
You can read the [swagger specification here.](https://github.com/swagger-api/swagger-spec)

### Integration with Common REST Server Frameworks

You can turn the specification file into a REST server implementation in your favorite framework, supported frameworks include Spring MVC, Django, Dropwizard, Flask, Pyramid and more…

You can also generate Java POJO’s and other language specific server implementations.

### Generating API Clients

Easily turn an API spec file into client SDK’s in many different languages , supported programing languages include Go, Java, Scala, Javascript, Groovy, Go, .Net, PHP, Perl, Python, Clojure.

This is a great feature, which let you adjust to your user needs without any effort.

### Generating Documentation

There are many different tools to generate documentation to your API, from HTML to JavaDoc to plain text in all different languages and bindings.
Documentations is usually a required task, which can again be automatically achieved from one source, the specification file.

### Swagger UI

API Inspector is an important feature for API consumers, it helps you understand the syntax of the requests data and the responses before you have an actual configured system.
You can simulate with it things and create applications which consume the REST API without having an actual functioning environment.

If you dont know what i mean, take a look at the [swagger UI example here.](http://petstore.swagger.io)

You can experiment with the API’s, again this is generated automatically from the specification file and you can embed this page in your control/management platform.
(An example of this is the API Inspector in Nicira’s NVP solution which helped me greatly in the past.)

### Integration With Build Systems

There are integrations with maven that can add all the above auto generation into your regular build process
For example [this plugin](https://github.com/kongchen/swagger-maven-plugin)

### Swagger Editor

There are many tools for editing and creating the specification file, validating its correctness and other useful features.
You can look at [this example](http://editor.swagger.io/#/) to get a feel of the swagger specification and an auto generated API inspector that is generated as you edit the file.

You can visit the [swagger github repository](https://github.com/swagger-api/swagger-spec) and check the “Additional Libraries” section, you will find many other usefull libraries and tools that work on a swagger speification file.

To summarize, REST API’s are becoming more and more dominant in the networking world just like in the rest of it, picking swagger with its big and increasing ecosystem can help you produce valuable features/implementations in close to zero time.


