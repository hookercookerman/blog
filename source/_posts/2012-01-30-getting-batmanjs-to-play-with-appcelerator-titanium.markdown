---
layout: post
title: "Getting BatmanJs to play with Appcelerator Titanium"
date: 2012-01-30 14:43
comments: true
categories: javascript, mobile, titanium, batmanjs 
---

Its true I am now full time working with [Appcelerator](http://www.appcelerator.com/);

Whilst I am going through the pain of learning any new framework is kind
of well painful; I wanted to use my [BatmanJs](http://batmanjs.org/)
models that I have created on the web side of things so I wanted to get
Batman to play nice in the titanium world; So Rich what did you do?

## BatmanJs ##

What is BatmanJs? its a Javascript library with some very nice 
bindings for the cool kids that want that thing; It has a sweet DSL 
which is main reason for using it; its just nice to use; it feels right.

Whilst batmanjs does really lack docs you can go into their test suit
and hunt around; this is not ideal but it does give you alot of
information. But have a look yourselves.

## Cut And Paste ##
At present their is no seperation of batmanjs main source file; like
you would see in [Emberjs](http://emberjs.com/) for instance.
So at the minute I am being lame and basically cutting out all the
DOM and Navigation functionality that I dont need form the main batman.coffee file.

## Main Js File ##

There is normally one main JS file that you have to bootstrap stuff in
titanium; this is what mine looks like for batman stuff

```coffeescript
  global = @ # needed to batman can play in commonjs world 
  App = {} # Bad but we need a namespace for Batman models
  Batman = require "/vendor/batman.titanium"
  Batman.currentApp = App
```

What you do need is use a namespace as we are not going to be using
Batman App and creating an instance that as it does not make sense in
titanium world to be using the boostrapping of that class;

## Request Using Titanium ##

I done the basics to get this to work so I am sure I need to tweak a few
things; like handling different status as Titanium only give you 
one hook onload no complete or success; 

```coffeescript
applyExtra = (Batman) ->

  # `param` and `buildParams` stolen from jQuery
  #
  # jQuery JavaScript Library v1.6.1
  # http://jquery.com/
  #
  # Copyright 2011, John Resig
  # Dual licensed under the MIT or GPL Version 2 licenses.
  # http://jquery.org/license
  rbracket = /\[\]$/
  r20 = /%20/g
  param = (a) ->
    return a if typeof a is 'string'
    s = []
    add = (key, value) ->
      value = value() if typeof value is 'function'
      s[s.length] = encodeURIComponent(key) + "=" + encodeURIComponent(value)

    if Batman.typeOf(a) is 'Array'
      for value, name of a
        add name, value
    else
      for own k, v of a
        buildParams k, v, add
    s.join("&").replace r20, "+"

  buildParams = (prefix, obj, add) ->
    if Batman.typeOf(obj) is 'Array'
      for v, i in obj
        if rbracket.test(prefix)
          add prefix, v
        else
          buildParams prefix + "[" + (if typeof v == "object" or Batman.typeOf(v) is 'Array' then i else "") + "]", v, add
    else if obj? and typeof obj == "object"
      for name of obj
        buildParams prefix + "[" + name + "]", obj[name], add
    else
      add prefix, obj
  
  Batman.Request::send = (data) ->
    data ?= @get('data')
    @fire 'loading'

    options =
      url: @get 'url'
      method: @get 'method'
      type: @get 'type'
      headers: @get 'headers'

      # aka success
      onload: (response) =>
        @set 'response', xhr.responseText
        @set 'status', (xhr?.status or 200)
        @fire 'success', xhr.responseText
        @fire 'loaded'

      # have to wait to see if this fits basically
      onerror: (error) =>
        @set 'response', xhr.responseText || xhr.content
        @set 'status', xhr.status
        @fire 'error', error

    unless @get('formData')
      options.headers['Content-type'] = @get('contentType')

    if options.method in ['PUT', 'POST']
      if @get('formData')
        options.data = @constructor.objectToFormData(data)
      else
        options.data = param(data)
    else if options.method is 'GET' and (params = param(data))
      options.url += "?#{params}"

    # Fires the request. Grab a reference to the xhr object so we can get the status code elsewhere.
    client = Ti.Network.createHTTPClient(options)
    client.open(options.method, options.url)
    client.send()
    xhr = client

if (module? && require?)
  module.exports = applyExtra
else
  applyExtra(Batman)
```

## Last Bit ##

At the end of the batman.titanium.coffee aka the main file I just add
the request

```coffeescript
  (require "vendor/batman.titanium_request")(Batman)
```

then Disco you are ready to Rock and Roll with Batman; 

```coffeescript
CurrentUser = new App.User()
CurrentUser.url = "http://#{HOST}/users/current"
CurrentUser.load (err) -> throw err if err
CurrentUser.on "loaded", =>
  win = ApplicationWindow()
  win.open()
```
