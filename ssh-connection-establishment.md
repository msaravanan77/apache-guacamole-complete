# Apache Guacamole - SSH Connection Establishment Flow

**Scope:** This document covers the complete flow from when a user clicks an SSH connection in the browser UI until the terminal appears on screen.

---

## Overview

The connection establishment involves 4 major components:
1. **Browser (JavaScript/Angular)** - User interface
2. **Java Backend (Tomcat)** - REST APIs and WebSocket handler
3. **guacd (C daemon)** - Protocol multiplexer
4. **Target SSH Server** - Final destination

---

## Detailed Flow with Code References

### Phase 1: User Authentication (Prerequisites)

Before connecting to any session, the user must authenticate.

**Step 1.1: User Login**
- **Location:** Browser UI
- **What Happens:** User enters username/password
- **Code:** `guacamole-client/guacamole/src/main/frontend/src/app/rest/services/authenticationService.js`
- **Result:** Receives authentication token (stored in browser session)

---

### Phase 2: Connection Selection

**Step 2.1: List Available Connections**
- **Location:** Browser
- **Code:** `guacamole-client/guacamole/src/main/frontend/src/app/rest/services/connectionService.js`
- **API Call:** `GET /api/session/data/{dataSource}/connections`
- **Handler:** `guacamole-client/guacamole/src/main/java/org/apache/guacamole/rest/connection/ConnectionDirectoryResource.java:getConnections()`
- **Returns:** List of connection objects including SSH connections

**Step 2.2: User Selects SSH Connection**
- **Location:** Browser UI
- **What Happens:** User clicks on SSH connection (e.g., "Production Server 192.168.1.10")
- **Navigation:** Redirects to client view with connection ID

---

### Phase 3: Client Initialization (Browser)

**Step 3.1: Create Guacamole Client**
- **File:** `guacamole-client/guacamole-common-js/src/main/webapp/modules/Client.js:31`
- **Function:** `new Guacamole.Client(tunnel)`
- **Purpose:** Creates protocol client that handles Guacamole instructions

**Step 3.2: Create WebSocket Tunnel**
- **File:** `guacamole-client/guacamole-common-js/src/main/webapp/modules/Tunnel.js:742`
- **Function:** `new Guacamole.WebSocketTunnel(tunnelURL)`
- **WebSocket URL Format:**
  ```
  ws://hostname/websocket-tunnel?token={authToken}&GUAC_DATA_SOURCE={dataSource}&GUAC_ID={connectionId}&GUAC_TYPE=c
  ```
- **Parameters:**
  - `token`: Authentication token from login
  - `GUAC_DATA_SOURCE`: Data source identifier (e.g., "mysql", "postgresql")
  - `GUAC_ID`: Connection ID (e.g., "5")
  - `GUAC_TYPE`: Type - "c" for connection, "g" for connection group

**Step 3.3: Establish WebSocket Connection**
- **File:** `Tunnel.js:991`
- **Code:** `socket = new WebSocket(tunnelURL + "?" + data, "guacamole");`
- **Protocol:** Uses WebSocket subprotocol "guacamole"
- **State:** Client state → `CONNECTING`

---

### Phase 4: Java Backend WebSocket Handling

**Step 4.1: WebSocket Endpoint Receives Connection**
- **File:** `guacamole-client/guacamole/src/main/java/org/apache/guacamole/tunnel/websocket/RestrictedGuacamoleWebSocketTunnelEndpoint.java:97`
- **Function:** `createTunnel(Session session, EndpointConfig config)`
- **Actions:**
  1. Extracts `TunnelRequest` from user properties
  2. Gets `TunnelRequestService`
  3. Calls `tunnelRequestService.createTunnel(tunnelRequest)`

**Step 4.2: Tunnel Request Service Processes Request**
- **File:** `guacamole-client/guacamole/src/main/java/org/apache/guacamole/tunnel/TunnelRequestService.java:330`
- **Function:** `createTunnel(TunnelRequest request)`
- **Detailed Steps:**

```java
// Line 334: Parse request parameters
String authToken = request.getAuthenticationToken();
String id = request.getIdentifier();  // Connection ID
TunnelRequestType type = request.getType();  // CONNECTION or CONNECTION_GROUP

// Line 340: Validate auth token and get session
GuacamoleSession session = authenticationService.getGuacamoleSession(authToken);
AuthenticatedUser authenticatedUser = session.getAuthenticatedUser();
UserContext userContext = session.getUserContext(authProviderIdentifier);

// Line 352: Create connected tunnel
GuacamoleTunnel tunnel = createConnectedTunnel(userContext, type, id, info, tokens);

// Line 359: Associate tunnel with session (monitoring, cleanup)
return createAssociatedTunnel(tunnel, authToken, session, userContext, type, id);
```

**Step 4.3: Retrieve Connection Configuration**
- **File:** `TunnelRequestService.java:204`
- **Function:** `createConnectedTunnel()`
- **Actions:**
  ```java
  // Line 210: Get Connection object from UserContext
  Connectable connectable = type.getConnectable(context, id);

  // Line 216: Call connect() method - this triggers protocol handshake
  GuacamoleTunnel tunnel = connectable.connect(info, tokens);
  ```

**What is `GuacamoleConfiguration`?**
- Contains: `protocol = "ssh"`
- Parameters from database/config:
  - `hostname`: "192.168.1.10"
  - `port`: "22"
  - `username`: "admin"
  - `password`: "secret" (or private-key)
  - `font-name`: "monospace"
  - `font-size`: "12"
  - `color-scheme`: "gray-black"
  - `width`: "1920"
  - `height`: "1080"
  - ...and 50+ other SSH parameters

---

### Phase 5: Connecting to guacd

**Step 5.1: Create TCP Socket to guacd**
- **File:** `guacamole-client/guacamole-common/src/main/java/org/apache/guacamole/net/InetGuacamoleSocket.java:87`
- **Function:** `InetGuacamoleSocket(String hostname, int port)`
- **Default guacd Address:** `localhost:4822`
- **Code:**
  ```java
  // Line 100: Create socket and connect
  sock = new Socket();
  sock.connect(new InetSocketAddress(hostname, port), SOCKET_TIMEOUT);

  // Line 104: Set timeout and TCP_NODELAY
  sock.setSoTimeout(SOCKET_TIMEOUT);
  sock.setTcpNoDelay(true);

  // Line 111: Create I/O streams (UTF-8)
  reader = new ReaderGuacamoleReader(new InputStreamReader(sock.getInputStream(), "UTF-8"));
  writer = new WriterGuacamoleWriter(new OutputStreamWriter(sock.getOutputStream(), "UTF-8"));
  ```

**Step 5.2: Perform Guacamole Protocol Handshake**
- **File:** `guacamole-client/guacamole-common/src/main/java/org/apache/guacamole/protocol/ConfiguredGuacamoleSocket.java:203`
- **Function:** `ConfiguredGuacamoleSocket(GuacamoleSocket socket, GuacamoleConfiguration config, GuacamoleClientInformation info)`

**Guacamole Protocol Format:**
```
instruction_length.instruction_name,arg1_length.arg1,arg2_length.arg2,...;
```

Example: `6.select,3.ssh;` means "select" instruction with argument "ssh"

**Handshake Steps:**

```java
// Line 220: Send "select" instruction
writer.writeInstruction(new GuacamoleInstruction("select", "ssh"));
// Sent: "6.select,3.ssh;"

// Line 223: Wait for "args" instruction from guacd
GuacamoleInstruction args = expect(reader, "args");
// Received: "4.args,12.VERSION_1_5_0,8.hostname,4.port,8.username,8.password,9.font-name,..."

// Line 262: Send "size" instruction (screen dimensions)
writer.writeInstruction(new GuacamoleInstruction("size", "1920", "1080", "96"));
// Sent: "4.size,4.1920,4.1080,2.96;"

// Line 272: Send "audio" instruction (supported formats)
writer.writeInstruction(new GuacamoleInstruction("audio", info.getAudioMimetypes()));
// Sent: "5.audio,9.audio/L16,..."

// Line 278: Send "video" instruction
writer.writeInstruction(new GuacamoleInstruction("video", info.getVideoMimetypes()));

// Line 287: Send "image" instruction
writer.writeInstruction(new GuacamoleInstruction("image", info.getImageMimetypes()));

// Line 307: Send "connect" instruction with ALL parameters
writer.writeInstruction(new GuacamoleInstruction("connect",
    "12.192.168.1.10",  // hostname
    "2.22",             // port
    "5.admin",          // username
    "6.secret",         // password
    "9.monospace",      // font-name
    "2.12",             // font-size
    "10.gray-black",    // color-scheme
    ...
));
// Sent: "7.connect,14.12.192.168.1.10,4.2.22,7.5.admin,..."

// Line 310: Wait for "ready" instruction
GuacamoleInstruction ready = expect(reader, "ready");
// Received: "5.ready,37.$a5b2c3d4-e5f6-7890-abcd-ef1234567890;"

// Line 316: Extract connection ID
id = ready.getArgs().get(0);  // "$a5b2c3d4-e5f6-7890-abcd-ef1234567890"
```

---

### Phase 6: guacd Processing

**Step 6.1: guacd Daemon Accepts Connection**
- **File:** `guacamole-server/src/guacd/daemon.c:544`
- **Function:** `main()` - Main event loop
- **Code:**
  ```c
  // Line 550: Accept incoming connection
  connected_socket_fd = accept(socket_fd, (struct sockaddr*) &client_addr, &client_addr_len);

  // Line 564: Set TCP_NODELAY
  setsockopt(connected_socket_fd, IPPROTO_TCP, TCP_NODELAY, &SO_TRUE, sizeof(SO_TRUE));

  // Line 582: Spawn connection thread
  pthread_create(&child_thread, NULL, guacd_connection_thread, params);
  ```

**Step 6.2: Connection Thread Routes Request**
- **File:** `guacamole-server/src/guacd/connection.c:366`
- **Function:** `guacd_connection_thread(void* data)`
- **Actions:**
  ```c
  // Line 394: Open guac_socket from file descriptor
  socket = guac_socket_open(connected_socket_fd);

  // Line 398: Route connection (performs handshake)
  guacd_route_connection(map, socket);
  ```

**Step 6.3: Parse "select" Instruction**
- **File:** `guacamole-server/src/guacd/connection.c:240`
- **Function:** `guacd_route_connection(guacd_proc_map* map, guac_socket* socket)`
- **Code:**
  ```c
  // Line 242: Allocate parser
  guac_parser* parser = guac_parser_alloc();

  // Line 249: Read "select" instruction
  guac_parser_expect(parser, socket, GUACD_USEC_TIMEOUT, "select");

  // Line 275: Extract protocol identifier
  const char* identifier = parser->argv[0];  // "ssh"
  ```

**Step 6.4: Create New Process for SSH**
- **File:** `guacamole-server/src/guacd/connection.c:298`
- **Code:**
  ```c
  // Check if identifier is a protocol name (not a connection ID)
  if (identifier[0] != GUAC_CLIENT_ID_PREFIX) {  // Not starting with '$'
      // Line 303: Create new process
      proc = guacd_create_proc(identifier);  // identifier = "ssh"
  }
  ```

**Step 6.5: Fork Process and Load SSH Plugin**
- **File:** `guacamole-server/src/guacd/proc.c:325`
- **Function:** `guacd_exec_proc(guacd_proc* proc, const char* protocol)`
- **Code:**
  ```c
  // Line 338: Load protocol plugin (shared library)
  if (guac_client_load_plugin(client, protocol)) {
      // Loads: /usr/lib/guacd/libguac-client-ssh.so
  }
  ```

**Step 6.6: Initialize SSH Client**
- **File:** `guacamole-server/src/protocols/ssh/client.c:66`
- **Function:** `guac_client_init(guac_client* client)`
- **Actions:**
  ```c
  // Line 72: Allocate SSH client data structure
  guac_ssh_client* ssh_client = guac_mem_zalloc(sizeof(guac_ssh_client));
  client->data = ssh_client;

  // Line 76: Set handlers
  client->join_handler = guac_ssh_user_join_handler;
  client->free_handler = guac_ssh_client_free_handler;
  ```

**Step 6.7: Send "args" Instruction Back**
- **Purpose:** Tell Java client which parameters are needed
- **Sent:** `args,VERSION_1_5_0,hostname,port,username,password,font-name,font-size,...`

**Step 6.8: Receive Connection Parameters**
- **Receives:** `connect,192.168.1.10,22,admin,secret,monospace,12,gray-black,...`
- **Parses:** All SSH parameters into settings structure

---

### Phase 7: User Join and SSH Connection

**Step 7.1: Add User to Client**
- **File:** `guacamole-server/src/guacd/proc.c:81`
- **Function:** `guacd_user_thread(void* data)`
- **Code:**
  ```c
  // Line 93: Create skeleton user
  guac_user* user = guac_user_alloc();
  user->socket = socket;
  user->client = client;
  user->owner = params->owner;  // TRUE for first user

  // Line 99: Handle user connection (performs user handshake)
  guac_user_handle_connection(user, GUACD_USEC_TIMEOUT);
  ```

**Step 7.2: SSH User Join Handler**
- **File:** `guacamole-server/src/protocols/ssh/user.c:40`
- **Function:** `guac_ssh_user_join_handler(guac_user* user, int argc, char** argv)`
- **Code:**
  ```c
  // Line 46: Parse SSH settings from arguments
  guac_ssh_settings* settings = guac_ssh_parse_args(user, argc, (const char**) argv);

  // Line 60: If owner (first user), start SSH client thread
  if (user->owner) {
      ssh_client->settings = settings;
      pthread_create(&(ssh_client->client_thread), NULL, ssh_client_thread, (void*) client);
  }

  // Line 79: Set input handlers
  user->key_handler = guac_ssh_user_key_handler;
  user->mouse_handler = guac_ssh_user_mouse_handler;
  user->size_handler = guac_ssh_user_size_handler;
  ```

**Step 7.3: SSH Client Thread Starts**
- **File:** `guacamole-server/src/protocols/ssh/ssh.c:226`
- **Function:** `ssh_client_thread(void* data)`
- **This is THE MAIN SSH CONNECTION LOGIC**

**Detailed SSH Connection Steps:**

```c
// Line 270: Initialize SSH libraries
guac_common_ssh_init(client);

// Line 293: Create terminal options
guac_terminal_options* options = guac_terminal_options_create(
    settings->width,      // 1920
    settings->height,     // 1080
    settings->resolution  // 96
);

// Configure terminal options (lines 297-305)
options->font_name = settings->font_name;           // "monospace"
options->font_size = settings->font_size;           // 12
options->color_scheme = settings->color_scheme;     // "gray-black"
options->backspace = settings->backspace;

// Line 308: CREATE TERMINAL EMULATOR
ssh_client->term = guac_terminal_create(client, client->socket, options);

// Line 315: Get user credentials
ssh_client->user = guac_ssh_get_user(client);  // Prompts if needed

// Line 320: Create SSH session to target server
ssh_client->session = guac_common_ssh_create_session(
    client,
    settings->hostname,     // "192.168.1.10"
    settings->port,         // "22"
    ssh_client->user,
    settings->server_alive_interval,
    settings->host_key,
    guac_ssh_get_credential  // Callback for additional credentials
);

// Line 340: Authenticate to SSH server
if (guac_common_ssh_authenticate(ssh_client->session) != 0) {
    // Authentication failed
    return NULL;
}

// Line 347: Open terminal channel
ssh_client->term_channel = libssh2_channel_open_session(session);

// Line 353: Request PTY (pseudo-terminal)
libssh2_channel_request_pty_ex(
    ssh_client->term_channel,
    settings->terminal_type,  // "linux"
    strlen(settings->terminal_type),
    ssh_ttymodes,
    sizeof(ssh_ttymodes),
    ssh_client->term->term_width,
    ssh_client->term->term_height,
    0, 0
);

// Line 370: Start shell
libssh2_channel_shell(ssh_client->term_channel);

// Line 374: Start input thread (reads from terminal, writes to SSH)
pthread_create(&input_thread, NULL, ssh_input_thread, (void*) client);

// Line 378: MAIN LOOP - Read from SSH, write to terminal
char buffer[8192];
int bytes_read;
while ((bytes_read = libssh2_channel_read(ssh_client->term_channel, buffer, sizeof(buffer))) > 0) {
    // Write data to terminal emulator
    guac_terminal_write(ssh_client->term, buffer, bytes_read);
}
```

**Step 7.4: Terminal Emulation & Rendering**
- **File:** `guacamole-server/src/terminal/terminal.c`
- **Function:** `guac_terminal_write(guac_terminal* term, const char* buffer, int length)`
- **Purpose:**
  - Processes VT102/VT220 escape sequences
  - Renders text using Pango/Cairo
  - Converts to Guacamole protocol instructions

**Terminal Processing:**
```c
// Parse VT escape sequences (CSI, OSC, etc.)
// Update internal terminal buffer
// Render changed regions to images
// Send Guacamole protocol instructions:
//   - "size,layer,width,height" - Set layer size
//   - "png,layer,x,y,base64_data" - Draw PNG image at position
//   - "copy,src_layer,sx,sy,w,h,mask,dst_layer,dx,dy" - Copy regions
//   - "cursor,x,y,layer,sx,sy,w,h" - Set cursor
```

**Step 7.5: Send "ready" Instruction**
- **Sent to Java:** `ready,$a5b2c3d4-e5f6-7890-abcd-ef1234567890`
- **Connection ID:** Unique identifier for this session

---

### Phase 8: Display in Browser

**Step 8.1: Java Receives "ready"**
- **File:** `ConfiguredGuacamoleSocket.java:310`
- **Extracts:** Connection ID
- **Returns:** ConfiguredGuacamoleSocket to WebSocket handler

**Step 8.2: WebSocket Sends Data to Browser**
- **File:** `RestrictedGuacamoleWebSocketTunnelEndpoint.java`
- **Forwards:** All Guacamole protocol instructions from guacd → Browser

**Step 8.3: Browser Receives Instructions**
- **File:** `Tunnel.js:1014`
- **Function:** `socket.onmessage = function(event) { parser.receive(event.data); }`
- **Parses:** Guacamole protocol instructions

**Step 8.4: Client Instruction Handler**
- **File:** `Client.js:1818`
- **Function:** `tunnel.oninstruction = function(opcode, parameters)`
- **Dispatches:** To specific instruction handlers

**Example Instructions Received:**

```javascript
// Instruction: "size,0,1920,1080"
instructionHandlers["size"] = function(parameters) {
    var layer = getLayer(0);  // Default layer
    display.resize(layer, 1920, 1080);
};

// Instruction: "png,0,0,0,iVBORw0KGgoAAAANSUh..."
instructionHandlers["png"] = function(parameters) {
    var layer = getLayer(0);
    var x = parseInt(parameters[2]);
    var y = parseInt(parameters[3]);
    var data = parameters[4];
    display.draw(layer, x, y, "data:image/png;base64," + data);
};

// Instruction: "sync,12345678,0"
instructionHandlers["sync"] = function(parameters) {
    display.flush(function() {
        tunnel.sendMessage("sync", timestamp);
    });
    setState(Guacamole.Client.State.CONNECTED);  // NOW CONNECTED!
};
```

**Step 8.5: Display Rendering**
- **File:** `guacamole-common-js/src/main/webapp/modules/Display.js`
- **Creates:** HTML5 Canvas layers
- **Draws:** Terminal images onto canvas
- **Result:** User sees SSH terminal in browser!

---

## Summary: Complete Path

```
1. User clicks SSH connection in browser
   ↓
2. Browser creates WebSocket: ws://host/websocket-tunnel?token=X&GUAC_ID=5
   ↓
3. Java WebSocket handler (RestrictedGuacamoleWebSocketTunnelEndpoint.java:97)
   ↓
4. TunnelRequestService.createTunnel() (TunnelRequestService.java:330)
   ↓
5. Creates TCP socket to guacd (InetGuacamoleSocket.java:100)
   ↓
6. Sends: select,ssh; size,1920,1080,96; connect,hostname,port,user,pass,...
   ↓
7. guacd daemon accepts connection (daemon.c:550)
   ↓
8. guacd routes to connection thread (connection.c:398)
   ↓
9. Creates new process, loads SSH plugin (proc.c:338, client.c:66)
   ↓
10. User join handler starts SSH client thread (user.c:66)
   ↓
11. ssh_client_thread() connects to target SSH server (ssh.c:226)
    - Creates terminal emulator
    - Connects to 192.168.1.10:22
    - Authenticates
    - Opens PTY channel
    - Starts shell
   ↓
12. Main loop: SSH server → libssh2 → terminal emulator → Guacamole protocol
   ↓
13. guacd sends: size,0,1920,1080; png,0,0,0,<base64>; sync,12345678
   ↓
14. Java forwards instructions → WebSocket → Browser
   ↓
15. Browser parses instructions (Client.js:1818)
   ↓
16. Renders to HTML5 Canvas (Display.js)
   ↓
17. USER SEES TERMINAL! ✅
```

---

## Key Files Reference

### Browser (JavaScript)
- `Tunnel.js:991` - WebSocket creation
- `Client.js:31` - Protocol client
- `Client.js:1818` - Instruction handler
- `Display.js` - Canvas rendering

### Java Backend
- `RestrictedGuacamoleWebSocketTunnelEndpoint.java:97` - WebSocket handler
- `TunnelRequestService.java:330` - Tunnel creation
- `InetGuacamoleSocket.java:100` - guacd connection
- `ConfiguredGuacamoleSocket.java:220` - Protocol handshake

### guacd (C)
- `daemon.c:550` - Accept connections
- `connection.c:398` - Route connection
- `proc.c:338` - Load plugin

### SSH Plugin (C)
- `client.c:66` - Initialize SSH client
- `user.c:60` - User join handler
- `ssh.c:226` - **Main SSH thread** (connects to target)
- `terminal.c` - Terminal emulation

---

## Guacamole Protocol Example

**Actual protocol exchange for SSH connection:**

```
→ Java to guacd:
6.select,3.ssh;

← guacd to Java:
4.args,12.VERSION_1_5_0,8.hostname,4.port,8.username,8.password,9.font-name,9.font-size,12.color-scheme;

→ Java to guacd:
4.size,4.1920,4.1080,2.96;
5.audio,9.audio/L16,10.audio/L8;
5.video,10.video/webm;
5.image,9.image/png,10.image/jpeg;
7.connect,14.12.192.168.1.10,4.2.22,7.5.admin,8.6.secret,11.9.monospace,4.2.12,12.10.gray-black;

← guacd to Java:
5.ready,37.$a5b2c3d4-e5f6-7890-abcd-ef1234567890;

← guacd to Java (continuous stream):
4.size,1.0,4.1920,4.1080;
3.png,1.0,1.0,1.0,9.iVBORw0KG...;
3.png,1.0,1.0,2.10,9.iVBORw0KG...;
4.sync,10.1234567890,1.0;
```

Connection is now established and terminal is displayed!
