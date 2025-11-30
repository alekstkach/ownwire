# OwnWire – End-to-End Encrypted Messaging for Websites

Secure encrypted communication between your frontend and backend—even behind MITM proxies.

## Problem

Even though most websites use HTTPS, TLS private keys are often held by reverse proxies like Cloudflare, Fastly, AWS ALB/ELB, etc. These systems terminate TLS, meaning all traffic between your backend and the browser is visible to them.

OwnWire fixes this by implementing an additional layer of public-key session encryption on top of a websocket. Only your frontend and your internal WS client can decrypt messages — not the reverse proxy in the middle.

OwnWire exposes two websockets:
- a public websocket (for browsers / SDK / embeddable widget)
- an internal websocket (for your backend, bot, or LLM client)

## Download

- [Linux binary](https://link-to/)
- [macOS binary](https://link-to/)

## Local test / demo

**1. See available options:**
```bash
ownwire --help
```

**2. Start OwnWire:**
```bash
./ownwire --log-level=info
```

**3. Run [the example Ruby internal-WS client](https://link-to/internal_ws_client.tar.gz):**
```bash
a) Put your OpenAI API key into a file: .openai_key
b) Install Ruby + Bundler:
   bundle install
c) Start the WS client:
   bundle exec ruby internal_ws_client.rb --openai-api-key-fn=.openai_key
```

**4. Open the demo:**
```
http://localhost:8080/embed_example.html
```

## Setting up in production

### 1. Install binary

```bash
cp ownwire /usr/local/bin/
chmod +x /usr/local/bin/ownwire
```

### 2. systemd service

```ini
[Unit]
Description=OwnWire secure messaging service
After=network.target

[Service]
ExecStart=/usr/local/bin/ownwire --log-level=warn
Restart=always
User=www-data
Group=www-data
WorkingDirectory=/var/lib/ownwire
Environment=OWNWIRE_PORT=8080

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable ownwire
systemctl start ownwire
```

### 3. Nginx reverse proxy

```nginx
server {
    listen 443 ssl;
    server_name ownwire.yourwebsite.com;

    ssl_certificate     /path/to/fullchain.pem;
    ssl_certificate_key /path/to/privkey.pem;

    # Optional but nice for keep-alive with pings/pongs
    proxy_read_timeout 75s;
    proxy_send_timeout 75s;

    # WebSocket endpoint (public WS URL used by SDK / widget)
    location = /ws {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;

        # Preserve upgrade semantics
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Crucial: forward original Host (includes port) and Origin
        proxy_set_header Host   $http_host;
        proxy_set_header Origin $http_origin;

        # Usual forwards
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Do NOT manually set Sec-WebSocket-*; nginx forwards them automatically
        proxy_buffering off;
    }

    # Everything else (HTML, JS, widget iframe, etc.)
    location / {
        proxy_pass http://127.0.0.1:8080;

        # Keep original host + port and origin
        proxy_set_header Host   $http_host;
        proxy_set_header Origin $http_origin;

        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
ln -s /etc/nginx/sites-available/ownwire /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx
```

## Note

> **⚠️ Important:** The widget and SDK only work on secure origins `https://*` or on `http://localhost`. It does not work on insecure origins.

## Using the JavaScript SDK

**Include the SDK:**
```html
<script src="https://ownwire.yourwebsite.com/js/ownwire.js"></script>
```

**Use the API:**
```javascript
let ownwire = new Ownwire("wss://ownwire.yourwebsite.com/ws");
ownwire.onMessage = (msg) => console.log("received:", msg);
ownwire.connect();
ownwire.send("What's the weather in Brazil?");
```

## Adding the chat widget

Place this in your HTML:

```html
<script src="https://ownwire.yourwebsite.com/js/ownwire_widget.js"></script>
<script>
  document.addEventListener("DOMContentLoaded", () => {
      ownwireWidget({
          ws_url: "wss://ownwire.yourwebsite.com/ws",
          metadata: "username:user1",
          title: "Chat with our agent",
          widget_origin: "https://ownwire.yourwebsite.com",
          widget_path: "/"
      });
  });
</script>
```

The widget appears in the corner and communicates securely through OwnWire.

## Working with the internal websocket

The public websocket is used by browsers and widgets. The *internal* websocket is where your own backend client (bot, LLM worker, custom app) connects to receive decrypted user messages and send replies back.

Conceptually it works like this:

```
browser  →  OwnWire public WS  →  OwnWire internal WS  →  your client
browser  ←  OwnWire public WS  ←  OwnWire internal WS  ←  your client
```

### Connection

You can use any language and any websocket client library.

**Typical connection parameters:**

- **host**: the same host/port where OwnWire is running (or proxied)
- **path**: `/ws` (default in OwnWire examples)
- **scheme**:
  - `ws://...` when talking directly to OwnWire on localhost
  - `wss://...` when OwnWire is behind HTTPS / reverse proxy

**Example URLs:**
```
ws://127.0.0.1:8081/ws
wss://ownwire.yourwebsite.com/ws
```

The internal websocket uses standard websocket semantics: you connect once and keep the connection open. OwnWire will send messages as they arrive and expects your client to reply with JSON when it wants to respond to a user.

### Message format

OwnWire sends and receives text frames containing JSON objects with the following shape:

```json
{
  "content":    "... message text ...",
  "session_id": "UUID string",
  "metadata":   "optional metadata string"
}
```

**Fields:**

**content**
The decrypted message text.
- **Incoming** (from OwnWire to your client): text the user typed in the widget / SDK.
- **Outgoing** (from your client to OwnWire): reply text you want OwnWire to deliver back to that user.

**session_id**
A UUID that identifies the user session / conversation.
- **Incoming**: which browser session sent this message.
- **Outgoing**: you **must** send the same `session_id` back so OwnWire knows which user should receive the reply.

**metadata**
An optional string for arbitrary metadata.
- **Incoming**: whatever metadata you supplied on the frontend (username, tags, etc.).
- **Outgoing**: you can pass it through unchanged, or modify it if you want. OwnWire does not interpret its contents.

### Typical workflow

Your internal WS client typically does the following:

1. Build the websocket URL (host + port + path), for example:
   ```
   ws_url = "ws://127.0.0.1:8081/ws"
   ```

2. Connect using your language's websocket client library.

3. When the connection is open, start reading messages.

4. For each incoming message frame:
   ```
   - Parse the text as JSON.
   - Extract:
       content    = data["content"]
       session_id = data["session_id"]
       metadata   = data["metadata"]
   - Process the content (e.g. send it to an LLM, or your own logic).
   - Build a reply JSON object:
       {
         "content":    reply_text,
         "session_id": session_id,
         "metadata":   metadata
       }
   - Serialize to JSON string and send back as a text websocket frame.
   ```

5. If the websocket closes or errors, log it and reconnect as needed.

### Pseudocode (language agnostic)

The Ruby example uses OpenAI, Faye WebSocket, and EventMachine, but you can do the same in any language. High-level pseudocode:

```
function handle_incoming(raw_json):
  data       = parse_json(raw_json)
  content    = data["content"]
  session_id = data["session_id"]
  metadata   = data.get("metadata")

  if content is empty:
      log("empty message, ignoring")
      return

  # Do your own processing here (LLM, rules, etc.)
  reply_text = process_message(content, metadata)

  reply = {
    "content":    reply_text,
    "session_id": session_id,
    "metadata":   metadata   # or modified metadata if you want
  }

  ws.send(encode_json(reply))


main():
  ws_url = "ws://127.0.0.1:8081/ws"   # or wss://... in production

  ws = connect_websocket(ws_url)

  ws.on_open = function():
      log("internal websocket connected")

  ws.on_message = function(event):
      try:
          handle_incoming(event.data)
      except error as e:
          log("error processing message: " + e)

  ws.on_close = function(code, reason):
      log("internal websocket closed: " + code + " / " + reason)
      # optionally: reconnect here

  ws.on_error = function(err):
      log("websocket error: " + err)
```

### Notes

- OwnWire will send periodic websocket pings; any reasonable client library will automatically reply with pongs, so you typically do not need to handle that manually.
- All user's messages (for all sessions) arrive over the same internal websocket connection. Always use the provided `session_id` to route and reply to the correct conversation.
- Messages are simple JSON text frames; there is no extra binary protocol or handshake beyond the standard websocket upgrade.
- You are free to replace the OpenAI/LLM part with any logic you want: RAG agent, custom chatbot, ticketing system, database queries, or direct human operator.

## Styling the chat widget

The chat widget is fully styled through a single CSS file. OwnWire now supports two ways to customize the widget's appearance:

1. **Simple method (recommended):** provide a custom CSS file via the `--custom-css` flag. This appends your CSS to the built-in stylesheet automatically.
2. **Advanced method:** override `/css/style.css` at the nginx layer and serve your own complete stylesheet.

Both approaches allow you to re-theme the chat widget without modifying any JavaScript or HTML.

### 1. Simple method: using `--custom-css`

You can pass your own CSS file to OwnWire, which gets appended after the standard widget CSS. Because it loads last, it overrides the built-in styles.

**Example:**
```bash
./ownwire --custom-css=/path/to/custom.css
```

**Minimal example for turning the widget green-themed:**

```css
/* custom.css: green theme overrides */
.chat { background-color: #2f3b30 !important; }
.chat header { background: #3a4a3a !important; }
.chat .message:not(.reply) { background: #57d38c !important; color: #0d2214 !important; }
.chat .message.reply { background: #465646 !important; }
.chat .send-message { background: linear-gradient(#4fdc87,#3cbf72) !important; }
.chat .send-message:hover { background: linear-gradient(#5aed95,#43d67f) !important; }
.chat .typing-dot { background: #c4f5d6 !important; }
```

Because this is appended to the default stylesheet, you do not need to rewrite the rest of the widget CSS — only the parts you want to override.

### 2. Advanced method: full CSS replacement via nginx

If you want complete control over the widget styles (for example a total redesign), you can override the default `style.css` at the reverse-proxy layer.

Place your custom file somewhere nginx can serve it, such as: `/var/www/ownwire-custom/style.css`

Then add an override:

```nginx
# Force nginx to serve your version of style.css
location = /css/style.css {
    alias /var/www/ownwire-custom/style.css;
    add_header Cache-Control "no-cache";
}
```

With this configuration, OwnWire's built-in stylesheet is never served; your stylesheet fully replaces it.

### Which method should you use?

- **Use `--custom-css`** if you just want to tweak colors, spacing, or theme. This is the simplest and safest method.
- **Use nginx override** only if you want a completely custom UI or you want to remove/replace the default CSS entirely.

In both cases, the widget is isolated inside an iframe, so your custom styles will not affect the rest of your website.

## Serving Assets From a CDN

By default, OwnWire serves all static assets (JS, CSS) directly from the binary. If you want higher performance or wish to offload this traffic, you can host these assets on a CDN. Here's a step-by-step guide on how to do this:

### 1. Extract Assets

Use the `-a` / `--extract-assets` flag to dump all JS and CSS assets from the embedded filesystem:

```bash
./ownwire --extract-assets=./public
```

This produces a directory like:
```
./public/ownwire_assets/css/...
./public/ownwire_assets/js/...
```

After extracting, OwnWire will exit.

You may then append to `style.css` your own custom css, for example, or replace it completely to style your widget the way you like.

### 2. Uploading Assets to Your CDN

Upload the contents of `ownwire_assets/css/` and `ownwire_assets/js/` to your CDN.

For example, assume that after uploading the assets are available under:

```
https://mycdn.com/mycompany/ownwire/css/style.css
https://mycdn.com/mycompany/ownwire/js/application.js
https://mycdn.com/mycompany/ownwire/js/ownwire.js
https://mycdn.com/mycompany/ownwire/js/ownwire_widget.js
```

We will refer to this as your **assets root URL**:
```
https://mycdn.com/mycompany/ownwire/
```

### 3. Starting OwnWire With CDN Assets

To make OwnWire rewrite its HTML so that CSS and JS come from your CDN, pass the root URL:

```bash
./ownwire --assets-url="https://mycdn.com/mycompany/ownwire/"
```

When this is enabled, OwnWire rewrites:

```html
<link href="/css/style.css" rel="stylesheet">
<script src="/js/application.js" type="module"></script>
```

...into:

```html
<link href="https://mycdn.com/mycompany/ownwire/css/style.css" rel="stylesheet">
<script src="https://mycdn.com/mycompany/ownwire/js/application.js" type="module"></script>
```

### 4. Updating the Widget Embed Code

If you embed the widget on your website, you must also load the widget script from the CDN. Notice how we added CDN base URL in front of the js/ownwire_widget.js on the first line:

```html
<script src="https://mycdn.com/mycompany/ownwire/js/ownwire_widget.js"></script>
<script>
  ownwireWidget({
    ws_url: "https://ownwire.yourwebsite.com/ws",
    metadata: "username:user1",
    title: "Chat with our agent",
    widget_origin: "",
    widget_path: "/"
  });
</script>
```

After this, all CSS and JS assets (including the widget) will come from your CDN, while OwnWire handles WebSocket traffic only.

---

## Documentation

For the full interactive documentation, visit: [https://yourusername.github.io/ownwire/](https://yourusername.github.io/ownwire/)

## License

[Your License Here]
