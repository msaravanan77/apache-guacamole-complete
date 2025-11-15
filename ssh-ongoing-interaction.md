# Apache Guacamole - SSH Ongoing Interaction Flow

**Scope:** This document covers keystroke and data interaction AFTER the initial connection has been established.

**Prerequisites:** Connection is already established (see `ssh-connection-establishment.md`)

---

## Overview

Once the SSH terminal is displayed, two continuous data flows operate:

1. **Input Flow:** Browser keyboard/mouse → guacd → SSH server
2. **Output Flow:** SSH server → guacd → Browser display

These flows operate independently and asynchronously.

---

## Flow 1: User Keystroke → SSH Server

### Step 1: User Presses Key in Browser

**Location:** Browser

**Event:** User presses a key (e.g., 'l' for `ls` command)

**JavaScript Event Handler:**
- Browser DOM captures `keydown` event
- Guacamole Keyboard module converts to keysym

---

### Step 2: Send Key Event via Guacamole Protocol

**File:** `guacamole-client/guacamole-common-js/src/main/webapp/modules/Client.js:349`

**Function:** `sendKeyEvent(pressed, keysym)`

```javascript
this.sendKeyEvent = function(pressed, keysym) {
    if (!isConnected()) return;

    // Send "key" instruction
    tunnel.sendMessage("key", keysym, pressed);
};
```

**Guacamole Protocol Instruction Sent:**
```
3.key,5.65027,1.1;    // key,keysym,pressed (1=down, 0=up)
```

- `keysym 65027` = lowercase 'l'
- `pressed 1` = key down

---

### Step 3: WebSocket Transmits to Java Backend

**File:** `Tunnel.js:958`

**Function:** `sendMessage(elements)`

```javascript
this.sendMessage = function(elements) {
    if (!tunnel.isConnected()) return;
    socket.send(Guacamole.Parser.toInstruction(arguments));
};
```

**Transmission:**
- WebSocket frame contains: `3.key,5.65027,1.1;`
- Sent to Java WebSocket endpoint

---

### Step 4: Java WebSocket Receives and Forwards

**File:** `guacamole-client/guacamole/src/main/java/org/apache/guacamole/websocket/GuacamoleWebSocketTunnelEndpoint.java`

**Process:**
1. Receives WebSocket message
2. Parses Guacamole instruction
3. Writes instruction to `GuacamoleSocket` (TCP socket to guacd)

**Code Flow:**
```java
// WebSocket onMessage handler receives "3.key,5.65027,1.1;"
// Writes directly to InetGuacamoleSocket
writer.writeInstruction(instruction);  // Sends to guacd
```

---

### Step 5: guacd Receives Key Instruction

**File:** `guacamole-server/src/libguac/user.c`

**Function:** User instruction handler receives key instruction

**Dispatcher:** `guacamole-server/src/libguac/user-handlers.c:__guac_user_key_handler`

**Code:**
```c
// Parse instruction: opcode="key", args=["65027", "1"]
int keysym = atoi(args[0]);  // 65027
int pressed = atoi(args[1]); // 1

// Call SSH-specific key handler
if (user->key_handler)
    return user->key_handler(user, keysym, pressed);
```

---

### Step 6: SSH Key Handler Processes Key

**File:** `guacamole-server/src/protocols/ssh/input.c:36`

**Function:** `guac_ssh_user_key_handler(guac_user* user, int keysym, int pressed)`

```c
int guac_ssh_user_key_handler(guac_user* user, int keysym, int pressed) {
    guac_client* client = user->client;
    guac_ssh_client* ssh_client = (guac_ssh_client*) client->data;

    // Send key to terminal emulator
    guac_terminal_send_key(ssh_client->term, keysym, pressed);

    return 0;
}
```

---

### Step 7: Terminal Processes Keystroke

**File:** `guacamole-server/src/terminal/terminal.c`

**Function:** `guac_terminal_send_key(guac_terminal* term, int keysym, int pressed)`

**Actions:**
1. Converts keysym to character or escape sequence
2. Writes to terminal's STDIN buffer
3. Terminal emulator handles special keys (arrows, function keys, etc.)

**Example Conversions:**
- `keysym 65027` → character `'l'`
- `keysym 65293` (Enter) → `"\r"` or `"\n"`
- `keysym 65361` (Left arrow) → `"\x1B[D"` (VT escape sequence)
- `keysym 65470` (F1) → `"\x1BOP"`

---

### Step 8: SSH Input Thread Reads from Terminal

**File:** `guacamole-server/src/protocols/ssh/ssh.c:201`

**Function:** `ssh_input_thread(void* data)`

**Main Loop:**
```c
void* ssh_input_thread(void* data) {
    guac_client* client = (guac_client*) data;
    guac_ssh_client* ssh_client = (guac_ssh_client*) client->data;

    char buffer[8192];
    int bytes_read;

    // Continuously read from terminal's STDIN
    while ((bytes_read = guac_terminal_read_stdin(ssh_client->term, buffer, sizeof(buffer))) > 0) {

        // Lock channel (thread safety)
        pthread_mutex_lock(&(ssh_client->term_channel_lock));

        // Write to SSH channel → Target SSH server
        libssh2_channel_write(ssh_client->term_channel, buffer, bytes_read);

        pthread_mutex_unlock(&(ssh_client->term_channel_lock));

        // Check if client is stopping
        if (client->state == GUAC_CLIENT_STOPPING)
            break;
    }

    return NULL;
}
```

**What Happens:**
- Reads `'l'` from terminal STDIN
- Calls `libssh2_channel_write()` to send to SSH server
- Character `'l'` transmitted over SSH connection

---

### Step 9: Target SSH Server Receives Keystroke

**Location:** Target SSH server (e.g., 192.168.1.10:22)

**Process:**
1. libssh2 on guacd sends SSH protocol packet
2. Target SSH server receives character `'l'`
3. Shell (bash/zsh) processes character
4. Adds to command line buffer
5. May echo back to client (depends on terminal settings)

---

## Flow 2: SSH Server Output → Browser Display

### Step 1: SSH Server Sends Data

**Example:** Server sends prompt or command output

**Data Sent:**
```
admin@server:~$ l_
```

**SSH Protocol:** Data sent as SSH channel data packet

---

### Step 2: guacd Receives SSH Data

**File:** `guacamole-server/src/protocols/ssh/ssh.c:226`

**Function:** `ssh_client_thread(void* data)` - Main loop

```c
void* ssh_client_thread(void* data) {
    // ... initialization code ...

    char buffer[8192];
    int bytes_read;

    // MAIN OUTPUT LOOP
    // Continuously read from SSH channel
    while ((bytes_read = libssh2_channel_read(ssh_client->term_channel, buffer, sizeof(buffer))) > 0) {

        // Write received data to terminal emulator
        guac_terminal_write(ssh_client->term, buffer, bytes_read);

        // Check for client stop
        if (client->state == GUAC_CLIENT_STOPPING)
            break;
    }

    return NULL;
}
```

**Process:**
- `libssh2_channel_read()` blocks until data available
- Receives raw bytes from SSH server
- Passes to terminal emulator

---

### Step 3: Terminal Emulator Processes Data

**File:** `guacamole-server/src/terminal/terminal.c`

**Function:** `guac_terminal_write(guac_terminal* term, const char* buffer, int length)`

**What It Does:**
1. **Parses VT escape sequences** (CSI, OSC, DCS, etc.)
2. **Updates internal buffer** (character grid)
3. **Renders changed regions** to images
4. **Generates Guacamole protocol instructions**

**Example VT Sequences Handled:**
- `\x1B[H` - Move cursor to home
- `\x1B[2J` - Clear screen
- `\x1B[31m` - Set foreground color to red
- `\x1B[1;5H` - Move cursor to row 1, column 5
- Regular text - Display characters

---

### Step 4: Terminal Renders to Images

**File:** `guacamole-server/src/terminal/display.c`

**Function:** `guac_terminal_display_flush()`

**Process:**
1. Identifies changed regions (dirty rectangles)
2. Renders text using **Pango** (text layout) and **Cairo** (graphics)
3. Encodes rendered regions as **PNG** (or JPEG/WebP)
4. Base64 encodes image data

**Example Rendering:**
```
Text: "admin@server:~$ ls"
↓
Cairo renders with font "monospace 12pt"
↓
PNG image: 400x16 pixels
↓
Base64: iVBORw0KGgoAAAANSUhEUgAA...
```

---

### Step 5: Generate Guacamole Protocol Instructions

**File:** `guacamole-server/src/libguac/protocol.c`

**Instructions Generated:**

```c
// If terminal size changes
guac_protocol_send_size(socket, layer, width, height);
// Sent: "4.size,1.0,4.1920,4.1080;"

// Draw image at position
guac_protocol_send_png(socket, GUAC_COMP_OVER, layer, x, y, surface);
// Sent: "3.png,1.0,1.0,2.10,<base64_length>.<base64_data>;"

// Copy region (optimization)
guac_protocol_send_copy(socket, src_layer, sx, sy, w, h, GUAC_COMP_OVER, dst_layer, dx, dy);
// Sent: "4.copy,1.0,1.0,1.0,3.400,2.16,3.255,1.0,1.0,2.10;"

// Update cursor position
guac_protocol_send_cursor(socket, x, y, src_layer, sx, sy, w, h);
// Sent: "6.cursor,2.15,1.5,1.0,1.0,1.0,1.8,2.16;"

// Synchronization point
guac_protocol_send_sync(socket, timestamp);
// Sent: "4.sync,10.1234567890;"
```

---

### Step 6: guacd Sends to Java Backend

**File:** `guacamole-server/src/libguac/socket.c`

**Function:** Socket write/flush

**Transmission:**
- Instructions written to `GuacamoleSocket`
- Buffered and flushed to TCP socket
- Java `InetGuacamoleSocket` receives on other end

---

### Step 7: Java Forwards to WebSocket

**File:** `guacamole-client/guacamole/src/main/java/org/apache/guacamole/websocket/GuacamoleWebSocketTunnelEndpoint.java`

**Process:**
1. Reads instructions from `GuacamoleSocket`
2. Writes to WebSocket session
3. Sends to browser

**Code:**
```java
// Read from guacd socket
GuacamoleInstruction instruction = reader.readInstruction();

// Send to WebSocket
session.getBasicRemote().sendText(instruction.toString());
```

---

### Step 8: Browser Receives Instructions

**File:** `guacamole-client/guacamole-common-js/src/main/webapp/modules/Tunnel.js:1014`

**Function:** `socket.onmessage`

```javascript
socket.onmessage = function(event) {
    resetTimers();  // Reset keep-alive

    try {
        // Parse Guacamole protocol
        parser.receive(event.data);
    }
    catch (e) {
        close_tunnel(new Guacamole.Status(...));
    }
};
```

---

### Step 9: Parser Extracts Instructions

**File:** `guacamole-client/guacamole-common-js/src/main/webapp/modules/Parser.js`

**Function:** `receive(text)`

**Process:**
1. Parses length-prefixed format
2. Extracts opcode and arguments
3. Calls `oninstruction` callback

**Example:**
```javascript
// Receives: "3.png,1.0,1.0,2.10,9.iVBORw0KG...;"
// Parses to:
opcode = "png"
args = ["0", "0", "10", "iVBORw0KG..."]
```

---

### Step 10: Client Dispatches to Instruction Handler

**File:** `guacamole-client/guacamole-common-js/src/main/webapp/modules/Client.js:1818`

**Function:** `tunnel.oninstruction = function(opcode, parameters)`

```javascript
tunnel.oninstruction = function(opcode, parameters) {

    // Dispatch to handler
    var handler = instructionHandlers[opcode];
    if (handler)
        handler(parameters);

    // Reset keep-alive
    scheduleKeepAlive();
};
```

---

### Step 11: Instruction Handler Updates Display

**File:** `Client.js:1555` (PNG instruction handler)

```javascript
instructionHandlers["png"] = function(parameters) {
    var channelMask = parseInt(parameters[0]);
    var layer = getLayer(parseInt(parameters[1]));
    var x = parseInt(parameters[2]);
    var y = parseInt(parameters[3]);
    var data = parameters[4];

    // Draw base64-encoded PNG to layer
    display.setChannelMask(layer, channelMask);
    display.draw(layer, x, y, "data:image/png;base64," + data);
};
```

---

### Step 12: Display Renders to Canvas

**File:** `guacamole-client/guacamole-common-js/src/main/webapp/modules/Display.js`

**Function:** `draw(layer, x, y, url)`

**Process:**
1. Creates Image object from data URL
2. When loaded, draws to canvas layer
3. Schedules display flush

```javascript
this.draw = function(layer, x, y, url) {
    var image = new Image();
    image.onload = function() {
        // Draw to canvas context
        layer.getCanvas().getContext("2d").drawImage(image, x, y);
        scheduleFlush();
    };
    image.src = url;  // "data:image/png;base64,iVBORw0KG..."
};
```

---

### Step 13: User Sees Updated Terminal!

**Result:** The character 'l' appears on screen, cursor moves forward

**Latency:** Typically 10-50ms end-to-end for local connections

---

## Special Case: Mouse Events

### Mouse Click/Movement

**Browser Event → guacd:**

```javascript
// Client.js:369
this.sendMouseState = function(mouseState) {
    tunnel.sendMessage("mouse", x, y, buttonMask);
};
```

**Guacamole Protocol:**
```
5.mouse,3.150,2.75,1.1;  // x=150, y=75, left_button=pressed
```

**SSH Handler:**
**File:** `input.c:67` - `guac_ssh_user_mouse_handler()`

```c
int guac_ssh_user_mouse_handler(guac_user* user, int x, int y, int mask) {
    guac_client* client = user->client;
    guac_ssh_client* ssh_client = (guac_ssh_client*) client->data;

    // Send mouse event to terminal
    guac_terminal_send_mouse(ssh_client->term, user, x, y, mask);

    return 0;
}
```

**Terminal processes:** Selection, scrolling, etc.

---

## Special Case: Clipboard (Copy/Paste)

### Copy from Terminal (Terminal → Browser)

**File:** `clipboard.c:30` - `guac_ssh_clipboard_handler()`

**Flow:**
```
User selects text in terminal
  ↓ (terminal.c - selection handling)
guac_common_clipboard_send() generates "clipboard" instruction
  ↓
Sent: "9.clipboard,9.text/plain,<length>.<clipboard_text>;"
  ↓
Browser receives, updates clipboard
```

### Paste to Terminal (Browser → Terminal)

**Browser sends:**
```
9.clipboard,9.text/plain,5.hello;
```

**SSH Handler:**
```c
// clipboard.c
// Receives clipboard data
// Writes to terminal as if user typed it
```

---

## Performance Optimizations

### 1. Display Frame Coalescing

**File:** `display.c`

**Optimization:**
- Multiple changes within ~16ms batched into single frame
- Reduces instruction count
- Prevents screen tearing

### 2. Copy Instructions

**Instead of sending full images:**
```
4.copy,1.0,1.0,2.10,3.400,2.16,3.255,1.0,1.0,3.100;
```

**Meaning:** Copy region from layer 0 to avoid re-encoding

### 3. Keep-Alive Pings

**File:** `Client.js:1778`

**Sent every 5 seconds:**
```
3.nop;  // No-operation, keeps connection alive
```

---

## Summary: Complete Keystroke Round-Trip

```
1. User presses 'l' key
   ↓ (Client.js:349)
2. Browser: tunnel.sendMessage("key", 65027, 1)
   ↓ (WebSocket)
3. Java: WebSocket → InetGuacamoleSocket → guacd
   ↓ (user-handlers.c)
4. guacd: Dispatches to SSH key handler
   ↓ (input.c:36)
5. SSH: guac_terminal_send_key(term, 65027, 1)
   ↓ (terminal.c)
6. Terminal: Converts keysym → 'l', writes to STDIN
   ↓ (ssh.c:210 ssh_input_thread)
7. SSH thread: libssh2_channel_write(channel, "l", 1)
   ↓ (SSH protocol over TCP)
8. Target SSH server receives 'l', echoes back
   ↓ (SSH protocol over TCP)
9. SSH thread: libssh2_channel_read() receives echo
   ↓ (ssh.c:226 main loop)
10. guac_terminal_write(term, "l", 1)
    ↓ (terminal.c + display.c)
11. Terminal renders 'l', generates PNG instruction
    ↓ (protocol.c)
12. guacd: send_png(socket, layer, x, y, png_data)
    ↓ (socket.c → InetGuacamoleSocket)
13. Java: Reads instruction, sends to WebSocket
    ↓ (WebSocket)
14. Browser: Receives "3.png,..."
    ↓ (Parser.js → Client.js:1555)
15. PNG handler: display.draw(layer, x, y, data_url)
    ↓ (Display.js)
16. Canvas draws image
    ↓
17. USER SEES 'l' ON SCREEN! ✅
```

**Total Time:** ~20-50ms for local guacd, longer for remote servers

---

## Key Files Reference

### Input Path (Browser → SSH Server)
1. `Client.js:349` - sendKeyEvent()
2. `Tunnel.js:958` - WebSocket send
3. `user-handlers.c` - Dispatch to handler
4. `input.c:36` - SSH key handler
5. `terminal.c` - Terminal processing
6. `ssh.c:210` - ssh_input_thread()

### Output Path (SSH Server → Browser)
1. `ssh.c:226` - ssh_client_thread() main loop
2. `terminal.c` - VT sequence parsing
3. `display.c` - Rendering
4. `protocol.c` - Instruction generation
5. `socket.c` - Send to Java
6. `Tunnel.js:1014` - WebSocket receive
7. `Client.js:1818` - Instruction dispatch
8. `Display.js` - Canvas rendering
