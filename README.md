# Analysis of Actorless Chatapp from Actix

https://github.com/actix/examples/tree/master/websockets/chat-actorless


# Client.rs#
## Overview
The client connects to a WebSocket server at ws://127.0.0.1:8080/ws.
It listens for user input from the terminal and sends that input as text messages to the WebSocket server.
The client also listens for messages from the WebSocket server, logs them, and responds to ping frames.
The program runs in an asynchronous loop, where it waits for either a WebSocket message or a 
command from the terminal input, handling both concurrently using tokio::select.

## Main Function

- The main function initializes the logger and then sets up the WebSocket client.
- It spawns a thread to handle user input and relay it to the main async task using an mpsc::unbounded_channel. 
- The cmd_tx sends messages from the input thread, and cmd_rx is the receiving end that will be used in the tokio::select loop.
- The `awc::Client::ws` function initiates a WebSocket connection to the server at `ws://127.0.0.1:8080/ws`.

## Input Thread

This thread reads from stdin (terminal) and sends the input to the main task via the cmd_tx channel. 
It runs on a separate thread since terminal I/O can block (because of `read_line`, 
and we want the WebSocket communication to remain non-blocking.

## WebSocket Loop:

After successfully connecting to the WebSocket server, the client enters an asynchronous loop that listens for both 
WebSocket messages and terminal input simultaneously: 

- Receiving WebSocket Messages: 
It uses the ws.next() stream to listen for WebSocket frames (messages from the server). When a Text frame is received, 
the client logs the message. If a Ping frame is received, it responds with a Pong frame. 

- Terminal Input (Command): 
When the user types something in the terminal, the command is received from the cmd_rx channel. If the input is non-empty, 
it sends that as a Text WebSocket message to the server. The loop breaks if there are no more messages from either the WebSocket server 
or the terminal.

# handler.rs

## echo_heartbeat_ws method
Echos text & binary messages received from the client, respond to ping messages, and monitor
connection health to detect network issues and free up resources.

Steps:
- Define `HEARTBEAT_INTERVAL` and `CLIENT_TIME_OUT`
- Define a `echo_heartbeat_ws` function to handle client's messages and observe clients heartbeat
  - Here arguments are     `mut session: actix_ws::Session` representing unique connections, 
    and `mut msg_stream: actix_ws::MessageStream` to read messages from 
  - track last heartbeat based on`Instant.now()`
  - interval is defined based on `HEARTBEAT_INTERVAL` value 
  - in a loop
    - Get interval tick and pint it. We pin it since tick returns a future and want to make sure it is not moved out of memory. 
     Staying safely in memory is important because we do not want to poll something that is moved.
    - then using the code below we wait either for a msg to arrive or tick to resolve (`heartbeat` amount is passed)
  
      ```rust
           match future::select(msg_stream.next(), tick).await {
                  ...
           }
      ```
    - Either of two things happen first:
      - **Received Messages**: When a message is received, it can be text, binary, close, ping, or pong, even continuation and 
        close. The function for the most part echoes text and binary messages back to the client.
        if message received is error or none, then break and return none
      - **heartbeat Monitoring**: If a heartbeat ping/pong is not received within CLIENT_TIMEOUT, the connection is closed.
      Otherwise, we just send a heartbeat using ping back (attempt to check client health)
  
  - When loop is done close websocket connection gracefully

## broadcast_ws
Broadcast text & binary messages received from a client, respond to ping messages, and monitor
connection health to detect network issues and free up resources.

Steps: 
- Takes
  `mut session: actix_ws::Session` unique connection of client, used to send messages to them or close them
  `mut msg_stream: actix_ws::MessageStream`  This is a stream of incoming messages from the client.
  `mut rx: broadcast::Receiver<web::Bytes>` A receiver for messages sent through a broadcast channel.
  This parameter is used to receive messages broadcasted to multiple clients. The broadcast::Receiver allows the
  WebSocket session to listen for messages that other parts of the application may send to all connected clients. 
  The mut keyword indicates that this receiver can be modified, allowing it to receive messages in a loop.

- Similar heartbeat and timeout mechanism is setup
- A loop to handle messages and check heartbeat is established
- if a message meant for broadcasting to the client is received, then the message is parsed and sent to the client,
in case of error we break the loop

## Simple Echo
Can run `cargo run --bin websocket-client`.
Some changes for more clarity. 
1. Started logging receiving of pings from the server in case `HEARTBEAT_TIMEOUT` is passed (default every 5 sec).
It makes it more clear how server and client interact. 

2. Modified this a bit to send a ping after 3 seconds to the server to which a pong will be received. 
just to test that client can also ping the server. It is a two way street

Now there are a lot of ping pongs lol

```bash
[2024-10-03T01:13:37Z INFO  websocket_client] starting echo WebSocket client
[2024-10-03T01:13:37Z INFO  websocket_client] connected; server will echo messages sent
[2024-10-03T01:13:37Z INFO  websocket_client] Got pinged by server
[2024-10-03T01:13:40Z INFO  websocket_client] Sending ping to the server
[2024-10-03T01:13:40Z INFO  websocket_client] Got Pong back from server
[2024-10-03T01:13:42Z INFO  websocket_client] Got pinged by server
[2024-10-03T01:13:45Z INFO  websocket_client] Sending ping to the server
[2024-10-03T01:13:45Z INFO  websocket_client] Got Pong back from server

```


## Additional way 
[index.html](./static/index.html) is another way for a client to establish connection and send message.

## Broadcast
Took me a while to realize how it works. Easiest way is to open a terminal for each desired client, imagine
3 people joining a room or group in chat. 

We will use `websocat` to perform subscribing to the broadcast messages by these 3 clients. A fourth client
will use `curl` to **send** a message to all the clients intently listening to the broadcast.

1. Bring up the server using 
`cargo run --bin websocket-server`

2. Clients joining the broadcast (chat group), then run this in terminal:
`websocat ws://127.0.0.1:8080/ws-broadcast `

3. A client who needs to send messages will use this
`curl -X POST http://localhost:8080/send -d 'hey guyz privately telling you this...'`

4. Should see the message sent in all those terminals who subscribed