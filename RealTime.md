# Real Time

This document records my learning note from [Real Time -- channels](https://hexdocs.pm/phoenix/channels.html)

## Channels

- First, Client connect to server using some transport (ie, WebSocket).
- After that, client need to join one or more topic to push or receive messages from channel server.
  - Channel server can also receives messages from their connected clients and can push messages to them.
  - Message is send and receive in channel (topic) even across different nodes.
  - So in other words, only topic matters.
- Channels are the highest level abstraction for real-time communication components in Phoenix.

### The Moving Parts

- One channel server process < -- > {per client, per topic}.
- Each channel < -- > `%Phoenix.Socket{}` and its state is in `socket.assigns`.
- Local PubSub and Remote PubSub.

### Server Endpoint

On server, config how transport is supported. Such as

```elixir
socket "/socket", HelloWeb.UserSocket,
  websocket: true,
  longpoll: false
```

### Client Handlers

On client, we could connect the above endpoint using javascript as

```javascript
let socket = new Socket("/socket", { params: { token: window.userToken } });
```

On the server, Phoenix will invoke `HelloWeb.UserSocket.connect/2`, passing your parameters and the initial socket state.

### Channel Routes and topics

- Channels handle events similar to Controllers.
- Each channel will implement one or more clauses:

  - `join/3`
  - `terminate/2`
  - `handle_in/3`
  - `handle_out/3`

- Topics
  - Convention: "topic:subtopic"

### [Message](https://hexdocs.pm/phoenix/Phoenix.Socket.Message.html)

- A struct with the following keys
  - topic
  - event
  - payload
  - ref

### [PubSub](https://hexdocs.pm/phoenix_pubsub/2.1.3/Phoenix.PubSub.html)

- Subscribe topic
- Broadcast to topic
- If your deployment environment does not support distributed Elixir or direct communication between servers, Phoenix also ships with a [Redis Adapter](https://hexdocs.pm/phoenix_pubsub_redis/Phoenix.PubSub.Redis.html) that uses Redis to exchange PubSub data.

## [A simple chat application](https://hexdocs.pm/phoenix/channels.html#tying-it-all-together)

1. Generating a socket

```sh
mix phx.gen.socket User
* creating lib/hello_web/channels/user_socket.ex
* creating assets/js/user_socket.js

Add the socket handler to your `lib/hello_web/endpoint.ex`, for example:

    socket "/socket", HelloWeb.UserSocket,
      websocket: true,
      longpoll: false

For the front-end integration, you need to import the `user_socket.js`
in your `assets/js/app.js` file:

    import "./user_socket.js"
```

- We generate two files which establish a websocket connection between client and server.
- On client, we use JavaScript to connect to our server (assets/js/app.js).
- On server

  - We enable the transport which uses websocket (lib/hello_web/endpoint.ex).
  - Define how a message get routed to a channel (lib/hello_web/channels/user_socket.ex).

    ```elixir
    defmodule HelloWeb.UserSocket do
    use Phoenix.Socket

    ## Channels
    channel "room:*", HelloWeb.RoomChannel
    ...
    ```

    - Now, whenever a client sends a message whose topic starts with "room:", it will be routed to our `RoomChannel`.
    - It is very similar with how Http request is routed to a controller.

2. Create Channel Module to handle message

- Create `lib/hello_web/channels/room_channel.ex` to define `RoomChannel` module.
- In `Channel`, we define how client join a given topic.

### Summary of Establishing Connection.

`mix phx.gen.socket User` generates two coordiate files to set up a connection between client and server.

On client side:

- Create a socket connection to server.
- With socket is connected: join different channels with topic.
- Notice: When join a channel, client must specify a topic. Make sure it match the topics defined in the channel module from server (see below).

On server side:

- Define use websocket for connection (`endpoint.ex`).
- Define a channel and create its corresponding channel module (`user_socket.ex`).
- Define how client join a topic in the channel module (`XyzChanel.ex`).

So far, we should see "Joined successfully" in the browser's JavaScript console. Our client and server are now talking over a persistent connection.

### Make it useful by enabling chat

On client side (`user_socket.js`)

- Define two UI elements with id and obtain their instance using JS code

  ```js
  let chatInput = document.querySelector("#chat-input");
  let messagesContainer = document.querySelector("#messages");
  ```

- Push message to through channel with event

  ```js
  chatInput.addEventListener("keypress", (event) => {
    if (event.key === "Enter") {
      channel.push("new_msg", { body: chatInput.value });
      chatInput.value = "";
    }
  });
  ```

- Receive and update messages so far

  ```js
  channel.on("new_msg", (payload) => {
    let messageItem = document.createElement("p");
    messageItem.innerText = `[${Date()}] ${payload.body}`;
    messagesContainer.appendChild(messageItem);
  });
  ```

- On server side (`room_channel.ex`)

  ```elixir
  def handle_in("new_msg", %{"body" => body}, socket) do
    broadcast!(socket, "new_msg", %{body: body})
    {:noreply, socket}
  end
  ```

  - We simply match on `new_msg` event and grab the payload.
  - And notify all other subscribers to `room:lobby` with `broadcast!/3`.
  - Notice the map key is string:
    - On client, the payload is `{ body: chatInput.value }`.
    - On server, the payload is ` %{"body" => body}`.
  - Each channel (including our own) subscriber can choose to intercept the event and have their handle_out/3 callback triggered

    ```elixir
    intercept ["new_msg"]

    def handle_out("new_msg", msg, socket) do
    push(
      socket,
      "new_msg",
      Map.merge(
        msg,
        %{is_editable: false}
      )
    )

    {:noreply, socket}
    end
    ```

    - First, define what message could be intercept.
    - Then, define the corresponding `handle_out/3`.
    - Now, the client could receive message with appended extra info.

At this point, fire up multiple browser tabs and you should see your messages being pushed and broadcasted to all windows!

## [How to authenticate client](https://hexdocs.pm/phoenix/channels.html#using-token-authentication)

The general idea is

- Associate every user with an assigned token in connection.
- After that, verify the user token in connection.

1. Assign a Token in the Connection

- For a user in `conn.assigns`, we use `Phoenix.Token.sign` to sign a token from that user (using his/her id) and set that token in conn

```elixir
current_user = conn.assigns[:current_user]
token = Phoenix.Token.sign(conn, "user socket", current_user.id)
assign(conn, :user_token, token)
```

2. Pass the Token to the JavaScript

- Get the assigned token from connection.
- To make sure every place of our frontend has this, we do this in `web/templates/layout/app.html.heex`.

```html
<script>
  window.userToken = "<%= assigns[:user_token] %>";
</script>
```

3. Client connect to the socket in JavaScript with token

We need to pass the assigned token during socket connection.

```js
let socket = new Socket("/socket", { params: { token: window.userToken } });
```

4. Server verify during socket connection

We match and verify the user token in `lib/hello_web/channels/user_socket.ex`

```elixir
def connect(%{"token" => token}, socket, _connect_info) do
  # max_age: 1209600 is equivalent to two weeks in seconds
  case Phoenix.Token.verify(socket, "user socket", token, max_age: 1209600) do
    {:ok, user_id} ->
      {:ok, assign(socket, :current_user, user_id)}
    {:error, reason} ->
      :error
  end
end
```

Notice: the "user socket" is matched with the string value we used to sign the token in step 1.

## [Presence](https://hexdocs.pm/phoenix/presence.html#content)

- Phoenix.Presence is a module that you can use to keep track of which users are on particular 'pages' of your app.
- In other words, it help us to show online users in some context.
- For example, we want to track all users are editing a particular file.
- See other references
  - [Can you hear me now? - Using Phoenix Presence](https://bendyworks.com/blog/can-you-hear-me-now-using-phoenix-presence)
  - [Using Phoenix Presence](https://whatdidilearn.info/2018/03/11/using-phoenix-presence.html)
  - [Phoenix Presence with Phoenix LiveView](https://fullstackphoenix.com/tutorials/phoenix-presence-with-phoenix-liveview)
