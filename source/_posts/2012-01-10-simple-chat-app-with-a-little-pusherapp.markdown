---
layout: post
title: "Simple Chat With A Little PusherApp"
date: 2012-01-10 16:50
comments: true
categories: rails, mongodb, RealTime
---

## Goal ##

Make Chat client inside an existing application; where an admin can chat to many users. Only admins can chat to users.

you dont want to read all this crap you just want to see it running cool
ok go [http://pusher-simple-chat.herokuapp.com/](http://pusher-simple-chat.herokuapp.com/)

![example](http://pusher-simple-chat.herokuapp.com/assets/example.png)

[You can found all the code to this example right here](https://github.com/hookercookerman/simple-chat)

## Tools ##

[PusherApp](http://pusherapp.com) using the [presence channels](http://pusher.com/docs/client_api_guide/client_presence_events) feature.

### What is PusherApp and what are Presence Channels?###

The docs at [PusherApp](http://pusherapp.com) are very good please read them then come back here.

### Setup ###

Lets pretend that we have a rails project setup and ready to rock and
roll; if you need reference there is the [coding example here](https://github.com/hookercookerman/simple-chat)

### The Guts ###

* User class with an admin permission; A User has a chat channel
* Channel class to represent a chat channel; A chat channel belongs to a user
* Message class for sending messages on a channel; A message has a chat channel
* None Admin user client side behaviour to send and receive messages.
* Admin user client side behaviour to chat to users.

### A Taster of the Coffescript ###

``` coffeescript
do ($ = jQuery) ->  
  class ClientChat
    @setup: (dev = true)->
      Pusher.channel_auth_endpoint = '/chat/channel/auth';
      if dev
        Pusher.log = (message) ->
          if window.console && window.console.log
            window.console.log(message)
        WEB_SOCKET_DEBUG = true 
       
    @scrollToBottom: =>
      $('#chat_messages').animate
        'scrollTop' : $('#chat_messages')[0].scrollHeight
      
    constructor: (@pusherKey, @chat_id = "#chat") ->
      @adminReadyToChat = false;
      @pusher = new Pusher(@pusherKey)
      @pusher.connection.bind 'connected', @connected
      @pusher.connection.bind 'disconnected', @disconnected

      if $(@chat_id).length
        @channel_name = $(@chat_id).attr("data-channel_name")
        @channel = @pusher.subscribe @channel_name
        @channel.bind "message", @receiveMessage
        @channel.bind "pusher:member_added", @adminOnline
        @channel.bind "pusher:member_removed", @adminOffline
        @channel.bind "pusher:subscription_succeeded", @readyToChat

    readyToChat: (message) =>
      if message["count"] > 1
        $("#no_one_there").html("")
      $.post "/chat/channel/activate", ->

    attachMessageBehaviour: =>
      $("#new_message").append("<input type='hidden' name='socket_id' value=#{@socket_id} />")
      $("#new_message #message_body").keypress (e) ->
        if e.which is 13
          $(@).focus()
          $("#new_message").submit()
          false

      $("#new_message").bind "submit", ->
        if !$("#new_message #message_body").val()
          false

      $("#new_message").bind "ajax:success", (evt, data, status, xhr) =>
        data = JSON.parse(data)
        $("#chat_messages").append("<li class='message'>
          <div class='sent'>
            <img src=#{data['avatar_url']} width='50px' height='50px'/><div class='display-name'>Me, #{data["created_at"]}</div>
            <div class='body'>#{data["message"]}</div>
          </div>
        </li>")
        $("#new_message #message_body").val("")
        ClientChat.scrollToBottom()
        
    adminOnline: (member) =>
      $("#no_one_there").html("")
      $("#chat").show()
      $("#chat_messages").css('opacity', 1)
      $("#chat_messages").append("<li class='message'><div class='notice'><img src=#{member['info']['avatar_url']} width='50px' height='50px'/>#{member['info']['display_name']} is online to chat</div></li>")
      ClientChat.scrollToBottom()

    adminOffline: (member) =>
      $("#chat").show()
      $("#chat_messages").css('opacity', 1)
      $("#chat_messages").append("<li class='message'><div class='notice'><img src=#{member['info']['avatar_url']} width='50px' height='50px'/>#{member['info']['display_name']} is offfline </div></li>")
      ClientChat.scrollToBottom()

    subscriptionError:  (message = "Sorry chat is Dead")->
      $("#chat_status_indicator").removeClass()
      $("#chat_status_indicator").addClass("error")
      $("#chat_messages").append("<li class='message'><div class='notice'>#{message}</div></li>")
      ClientChat.scrollToBottom()
      
    receiveMessage: (message) ->
      if not $("#chat").is(":visible")
        $("#chat").show()
      $("#chat_messages").append("<li class='message'>
        <div class='received'>
          <img src=#{message['avatar_url']} width='50px' height='50px'/><div class='display-name'>#{message['display_name']}, #{message["created_at"]}</div>
          <div class='body'>#{message["message"]}</div>
        </div>
      </li>")
      ClientChat.scrollToBottom()
      
    disconnected: (message) ->
      messageDom = $("<li class='message'><div class='notice'><img src=#{message['avatar_url']} width='50px' height='50px'/>#{message['message']}</div></li>")
      if $("#chat_messages")
        $("#chat_messages").append(messageDom)
        $("#chat_messages").css('opacity', 0.5)
        ClientChat.scrollToBottom()

    connected: =>
      @socket_id = @pusher.connection.socket_id
      @attachMessageBehaviour()

  $(document).ready ->
    if $("#chat").length
      ClientChat.setup()
      client = new ClientChat(PUSHER_KEY)
      
    if $('#chat_window').length
      $('#chat_title').click () -> 
        $('#chat_container').slideToggle()
        $(@).toggleClass('active')

```


## What about testing ? ##

Ok here is a taster for example of an acceptance test for when a user
gets a mesage from an admin

```ruby
require File.expand_path("../../acceptance_helper", __FILE__)

feature "Chat" do
  context "User is signed in" do
    background  do
      @user = FactoryGirl.create(:user, email:  "test@test.com", password: "testing123", password_confirmation: "testing123")
      @user.chat_channel.start_chat!
      sign_in(@user)
    end

    scenario "reciving a message from the admin", js: true do
      save_and_open_page
      page.execute_script(%Q(
        Pusher.dispatch('#{@user.chat_channel.name}', 'message', {'display_name': 'Hookercookerman', 'message':'Just testing', 'channel_name' : '#{@user.chat_channel.name}'});
      ))
      page.should have_content("Just testing")
      page.should have_content("Hookercookerman")
    end
  end
end
```

## Summary ##

Using pusherapp is super simple and it allows making any realtime app a
breeze so give it a go; 

## Whats Next ##

Ok for next time I will attempt to give you custom twitter streams;
What does that mean? I user can create stop and start twitter streams
for various requirements i.e hastag milkandbeans.
