# **ShopAI**
Rapidly Connects Customers to Products

---

# ShopAI API Documentation `v0.1`

## Overview

The ShopAI backend is the core of ShopAI. Currently we are running one backend instance for each customer. Each instance has it's own address to connect to. The API of each instance enables the following functionalities:
- Initializing new chat sessions
- Sending messages
- Retrieving the chat history
- Sending heartbeats
- Retrieving the server status
- (Custom functionalities)

The communication with the backend is done via HTTP and Websockets.



## HTTP

### Initializing a new chat session

#### `GET /init_session`

- **Description**: Initializes a new chat session
- **Parameters**:
  - `license_key`: The instance license key
  - `lang`: The language of the chat session
- **Example**:
    ```
    GET /init_session?license_key=123456&lang=en
    ```
- **Response**:
    ```json
    {
        "status": "ok",
        "chat_token": "d42e7dc9-..."
    }
    ```
- **Errors**:
    - **Invalid language**:
        ```json
        {
            "status": "error",
            "message": "Invalide language, supported languages: ['hu', 'en']"
        }
        ```
    - **Invalid license key**:
        ```json
        {
            "status": "error",
            "message": "Invalid license key"
        }
        ```

## Websocket

### Concept
Unlike HTTP, the websocket system is built in such a way that a response is not necessarily expected when a message is sent. Response messages are also not clearly recognizable as such and should not be treated as such.
It is better to think of messages as actions that are intended to trigger something on the other side. Incoming messages should be treated in exactly the same way. 
- **There are no server responses, only messages from the server.**
- **You should always be prepared for every message type.** (No matter what you sent)

### Connecting to the websocket

#### `GET /shpaiws`
- **Description**: Establishes a websocket connection to the chat session
- **Parameters**:
  - `chat_token`: The chat token received from the `init_session` call
- **Headers** (In most cases these are automatically set by the websocket client):
  - `Sec-Websocket-Version`: The websocket version.
  - `Sec-Websocket-Key`: The websocket key.
  - `Sec-Websocket-Extensions`: The websocket extensions.
  - `Connection`: The connection type, in this case `Upgrade`
  - `Host`: The host of the websocket server
- **Example**:
    ```
    GET /shpaiws?chat_token=d42e7dc9-...
    ```
- **Response**:
    - The websocket connection is established. The server will send the following websocket message:
        ```json
        {
            "type": "status", 
            "status": "operational"
        }
        ```
- **Errors**:
    - A websocket connection will be established, but the server will close it immediately. This will be improved in future versions.

### 💬 Sending heartbeats
**Trigger the server to send a heartbeat message.**

To prevent the websocket connection from timing out, the client should send a heartbeat message every 15 seconds. This triggers the server to send a heartbeat message as well.
Sending a heartbeat message is done by sending the following message to the websocket:
```json
{
    "type": "heartbeat"
}
```
We are triggering the server to send a heartbeat message as well:
```json
{
    "type": "heartbeat"
}
```
When the server is currently processing something, it might not respond to the heartbeat message immediately.

### 💬 Sending user messages
**Trigger the server to process a message.**

> ⚠️ **Important:** Only send messages to the server when it is confirmed to be in the `operational` state. After sending a message, disable message sending until the server confirms it remains in the `operational` state.

The client can send messages to the server by sending the following message to the websocket:
```json
{
    "type": "message",
    "message": "Hello, I would like to buy a new phone."
}
```
The maximum length of the message is `512 characters` by default.
When the message is too long, the server will send the following message:
```json
{
    "type": "error",
    "message": "Message is too long, maximum length is {max message length} characters"
}
```
> **Note:** In future versions, the server will simply close the connection when the client is causing an error.

### 💬 Retrieving chat history
**Trigger the server to send the chat history.**

> **Note:** We are expecting you to keep track of the chat history on the client side. A history message from the server should always overwrite the local chat history.

The client can request the chat history by sending the following message to the websocket:
```json
{
    "type": "get_history"
}
```

A message from the server with the chat history will look like this:
```json
{
    "type": "history",
    "history": [
        {
            "type": "user",
            "content": "Hello, I would like to buy a new phone."
        },
        {
            "type": "ai",
            "content": "Hello, how can I help you?"
        }
    ]
}
```

### 🗨️ Tokens
> **What is a token?** In the context of large language models (LLMs) like GPT, "tokens" refer to the pieces of text that the model processes. These can be words, parts of words, or punctuation marks, depending on how the model's tokenizer breaks down the input text. Tokens are the basic units that the model uses to understand and generate language.

While the server is processing the message, it will repeatedly send messages with the type `token`. A token message contains the last generated token.
Example:
```json
{
    "type": "token",
    "token": " examp"
}
```
*Future tokens may be: "le", " of", " text"*

Do handle these messages the following way:
- Check if the last chat message is a message of `"type": "ai"`
  - If not: Add a message with the type `ai` and the content of the token to the local chat history
  - If so: Concatenate the token to the last message of the local chat history

> **Important:** When the system is done generating the response, it will send a message of type `history` with the full response. The history should always overwrite the local chat history. The system will also send a message of type `status` with the status `operational`.