/**
 * Telegram Cloudflare WebSocket Relay
 * V0.3-A
 *
 * Domain:
 *   tg.cdfcc.kdns.fr
 *
 * Routes:
 *   /
 *   /health
 *   /apiws
 *   /mtws
 *   /mtproto
 *
 * Architecture:
 *
 * Browser MTProto
 *      ↓
 * WSS + binary + obfuscated2
 *      ↓
 * Cloudflare Worker
 *      ↓
 * outbound WebSocket
 *      ↓
 * Telegram WebSocket endpoint
 *
 * IMPORTANT:
 * /apiws
 *   保留 V0.2.2 的 TCP Relay
 *
 * /mtws
 *   新增真正的 Telegram WebSocket Relay
 *
 * Worker 本身不解析 MTProto。
 */

const TELEGRAM_WS = {
  1: "https://pluto.web.telegram.org/apiws",
  2: "https://venus.web.telegram.org/apiws",
  3: "https://aurora.web.telegram.org/apiws",
  4: "https://vesta.web.telegram.org/apiws",
  5: "https://flora.web.telegram.org/apiws"
};

const TELEGRAM_TCP = {
  1: [
    "149.154.175.50",
    "149.154.175.51"
  ],

  2: [
    "149.154.167.50",
    "149.154.167.41",
    "149.154.167.220"
  ],

  3: [
    "149.154.175.100",
    "149.154.167.91"
  ],

  4: [
    "149.154.167.92",
    "149.154.165.111"
  ],

  5: [
    "91.108.56.100",
    "91.108.56.101"
  ]
};

const DEFAULT_DC = 2;


/* =========================================================
   Worker
========================================================= */

export default {

  async fetch(request, env, ctx) {

    const url = new URL(request.url);


    /* -----------------------------------------------------
       Homepage
    ----------------------------------------------------- */

    if (url.pathname === "/") {

      return new Response(ROOT_HTML, {
        status: 200,

        headers: {
          "content-type":
            "text/html; charset=UTF-8",

          "cache-control":
            "no-store"
        }
      });
    }


    /* -----------------------------------------------------
       Health
    ----------------------------------------------------- */

    if (url.pathname === "/health") {

      return json({

        ok: true,

        service:
          "telegram-cloudflare-websocket-relay",

        version:
          "V0.3-A",

        architecture:
          "Browser WSS -> Worker WSS -> Telegram",

        defaultDC:
          DEFAULT_DC,

        routes: {

          websocket:
            "/mtws",

          legacyTCP:
            "/apiws",

          testPage:
            "/mtproto"
        },

        timestamp:
          new Date().toISOString()
      });
    }


    /* -----------------------------------------------------
       MTProto Test Page
    ----------------------------------------------------- */

    if (url.pathname === "/mtproto") {

      return new Response(
        MTPROTO_HTML,

        {
          status: 200,

          headers: {

            "content-type":
              "text/html; charset=UTF-8",

            "cache-control":
              "no-store"
          }
        }
      );
    }


    /* -----------------------------------------------------
       NEW:
       Telegram WebSocket Relay
    ----------------------------------------------------- */

    if (url.pathname === "/mtws") {

      return handleTelegramWebSocket(
        request
      );
    }


    /* -----------------------------------------------------
       LEGACY:
       V0.2.2 TCP Relay
    ----------------------------------------------------- */

    if (url.pathname === "/apiws") {

      return handleLegacyTCPRelay(
        request
      );
    }


    return new Response(
      "Not Found",
      {
        status: 404
      }
    );
  }
};


/* =========================================================
   Telegram WebSocket Relay
========================================================= */

async function handleTelegramWebSocket(
  request
) {

  const upgrade =
    request.headers.get("Upgrade");


  if (
    !upgrade ||
    upgrade.toLowerCase() !==
      "websocket"
  ) {

    return new Response(
      "Expected WebSocket",
      {
        status: 426,

        headers: {
          "Upgrade":
            "websocket"
        }
      }
    );
  }


  const url =
    new URL(request.url);


  let dc =
    Number(
      url.searchParams.get("dc") ||
      DEFAULT_DC
    );


  if (!TELEGRAM_WS[dc]) {

    dc =
      DEFAULT_DC;
  }


  const telegramURL =
    TELEGRAM_WS[dc];


  /*
   * -------------------------------------------------------
   * Create incoming WebSocket pair
   * -------------------------------------------------------
   */

  const pair =
    new WebSocketPair();


  const client =
    pair[0];

  const server =
    pair[1];


  /*
   * Half-open is important for proxying.
   */

  server.accept({
    allowHalfOpen: true
  });


  /*
   * Make binary messages predictable.
   *
   * Cloudflare changed the default binaryType
   * to Blob in 2026.
   */

  server.binaryType =
    "arraybuffer";


  let upstream;


  try {

    /*
     * -----------------------------------------------------
     * Cloudflare Worker -> Telegram WebSocket
     * -----------------------------------------------------
     *
     * Workers supports outbound WebSocket
     * via fetch() + Upgrade.
     *
     * Telegram requires:
     *
     * Sec-WebSocket-Protocol: binary
     */

    const upstreamResponse =
      await fetch(
        telegramURL,
        {

          headers: {

            "Upgrade":
              "websocket",

            "Sec-WebSocket-Protocol":
              "binary"
          }
        }
      );


    /*
     * Telegram must return 101
     * and expose response.webSocket.
     */

    if (
      upstreamResponse.status !==
        101 ||
      !upstreamResponse.webSocket
    ) {

      const body =
        await safeText(
          upstreamResponse
        );


      throw new Error(
        `Telegram WebSocket upgrade failed: ` +
        `HTTP ${upstreamResponse.status}` +
        (body
          ? ` ${body.slice(0, 200)}`
          : "")
      );
    }


    upstream =
      upstreamResponse.webSocket;


    /*
     * Accept backend WebSocket.
     */

    upstream.accept({
      allowHalfOpen: true
    });


    upstream.binaryType =
      "arraybuffer";


    console.log(
      "Telegram WebSocket connected:",
      telegramURL
    );


  } catch (error) {

    console.error(
      "Telegram WebSocket connection failed:",
      formatError(error)
    );


    try {

      server.close(
        1011,
        "Telegram WebSocket failed"
      );

    } catch (_) {}


    return new Response(
      null,
      {
        status: 101,
        webSocket: client
      }
    );
  }


  let closed =
    false;


  function closeBoth(
    code = 1000,
    reason = "closed"
  ) {

    if (closed) {
      return;
    }


    closed = true;


    try {

      upstream.close(
        code,
        reason
      );

    } catch (_) {}


    try {

      server.close(
        code,
        reason
      );

    } catch (_) {}
  }


  /* =======================================================
     Browser -> Telegram
  ======================================================= */

  server.addEventListener(
    "message",
    async event => {

      if (closed) {
        return;
      }


      try {

        /*
         * MTProto WebSocket transport
         * is binary-only.
         */

        if (
          typeof event.data ===
            "string"
        ) {

          console.warn(
            "Received unexpected text frame"
          );


          return;
        }


        let data;


        if (
          event.data instanceof
            ArrayBuffer
        ) {

          data =
            event.data;

        }

        else if (
          ArrayBuffer.isView(
            event.data
          )
        ) {

          data =
            event.data.buffer;

        }

        else if (
          typeof Blob !==
            "undefined" &&
          event.data instanceof Blob
        ) {

          data =
            await event.data.arrayBuffer();

        }

        else {

          throw new Error(
            "Unsupported WebSocket message type"
          );
        }


        if (
          !data ||
          data.byteLength === 0
        ) {

          return;
        }


        /*
         * IMPORTANT:
         *
         * Do NOT parse MTProto.
         *
         * Do NOT modify bytes.
         *
         * Do NOT add framing.
         */

        upstream.send(
          data
        );


      } catch (error) {

        console.error(
          "Client -> Telegram failed:",
          formatError(error)
        );


        closeBoth(
          1011,
          "Relay write failed"
        );
      }
    }
  );


  /* =======================================================
     Telegram -> Browser
  ======================================================= */

  upstream.addEventListener(
    "message",
    async event => {

      if (closed) {
        return;
      }


      try {

        if (
          typeof event.data ===
            "string"
        ) {

          /*
           * Telegram MTProto WebSocket
           * should be binary.
           */

          console.warn(
            "Telegram returned text frame"
          );


          return;
        }


        let data;


        if (
          event.data instanceof
            ArrayBuffer
        ) {

          data =
            event.data;

        }

        else if (
          ArrayBuffer.isView(
            event.data
          )
        ) {

          data =
            event.data.buffer;

        }

        else if (
          typeof Blob !==
            "undefined" &&
          event.data instanceof Blob
        ) {

          data =
            await event.data.arrayBuffer();

        }

        else {

          throw new Error(
            "Unsupported Telegram message type"
          );
        }


        if (
          !data ||
          data.byteLength === 0
        ) {

          return;
        }


        /*
         * Transparent relay.
         */

        server.send(
          data
        );


      } catch (error) {

        console.error(
          "Telegram -> Client failed:",
          formatError(error)
        );


        closeBoth(
          1011,
          "Relay read failed"
        );
      }
    }
  );


  /* =======================================================
     Client close
  ======================================================= */

  server.addEventListener(
    "close",
    event => {

      console.log(
        "Client closed:",
        event.code,
        event.reason
      );


      if (!closed) {

        closed = true;


        try {

          upstream.close(
            event.code || 1000,
            "client closed"
          );

        } catch (_) {}
      }
    }
  );


  /* =======================================================
     Telegram close
  ======================================================= */

  upstream.addEventListener(
    "close",
    event => {

      console.log(
        "Telegram closed:",
        event.code,
        event.reason
      );


      if (!closed) {

        closed = true;


        try {

          server.close(
            1000,
            "telegram closed"
          );

        } catch (_) {}
      }
    }
  );


  /* =======================================================
     Errors
  ======================================================= */

  server.addEventListener(
    "error",
    error => {

      console.error(
        "Client WebSocket error:",
        formatError(error)
      );


      closeBoth(
        1011,
        "client websocket error"
      );
    }
  );


  upstream.addEventListener(
    "error",
    error => {

      console.error(
        "Telegram WebSocket error:",
        formatError(error)
      );


      closeBoth(
        1011,
        "telegram websocket error"
      );
    }
  );


  /*
   * Return the client side of our
   * WebSocketPair to the browser.
   */

  return new Response(
    null,
    {
      status: 101,

      webSocket:
        client
    }
  );
}


/* =========================================================
   Legacy V0.2.2 TCP Relay
   保留，不作为 V0.3 测试链路
========================================================= */

async function handleLegacyTCPRelay(
  request
) {

  const upgrade =
    request.headers.get("Upgrade");


  if (
    !upgrade ||
    upgrade.toLowerCase() !==
      "websocket"
  ) {

    return new Response(
      "Expected WebSocket",
      {
        status: 426,

        headers: {
          "Upgrade":
            "websocket"
        }
      }
    );
  }


  const url =
    new URL(request.url);


  let dc =
    Number(
      url.searchParams.get("dc") ||
      DEFAULT_DC
    );


  if (!TELEGRAM_TCP[dc]) {

    dc =
      DEFAULT_DC;
  }


  const hostname =
    TELEGRAM_TCP[dc][0];


  const pair =
    new WebSocketPair();


  const client =
    pair[0];

  const server =
    pair[1];


  server.accept({
    allowHalfOpen: true
  });


  server.binaryType =
    "arraybuffer";


  let socket;


  try {

    /*
     * V0.2.2 verified TCP path.
     *
     * This route is intentionally preserved.
     */

    const {
      connect
    } =
      await import(
        "cloudflare:sockets"
      );


    socket =
      connect({

        hostname,

        port: 443,

        secureTransport:
          "off",

        allowHalfOpen:
          true
      });


    await socket.opened;


  } catch (error) {

    console.error(
      "Legacy TCP failed:",
      formatError(error)
    );


    try {

      server.close(
        1011,
        "TCP connection failed"
      );

    } catch (_) {}


    return new Response(
      null,
      {
        status: 101,
        webSocket: client
      }
    );
  }


  let stopped =
    false;


  function stop(
    code = 1000,
    reason = "closed"
  ) {

    if (stopped) {
      return;
    }


    stopped = true;


    try {
      socket.close();
    } catch (_) {}


    try {
      server.close(
        code,
        reason
      );
    } catch (_) {}
  }


  server.addEventListener(
    "message",
    async event => {

      if (stopped) {
        return;
      }


      try {

        let data;


        if (
          event.data instanceof
            ArrayBuffer
        ) {

          data =
            new Uint8Array(
              event.data
            );

        }

        else if (
          typeof Blob !==
            "undefined" &&
          event.data instanceof Blob
        ) {

          data =
            new Uint8Array(
              await event.data.arrayBuffer()
            );

        }

        else {

          return;
        }


        if (
          data.byteLength === 0
        ) {

          return;
        }


        const writer =
          socket.writable.getWriter();


        try {

          await writer.write(
            data
          );

        } finally {

          writer.releaseLock();
        }


      } catch (error) {

        console.error(
          "Legacy WS -> TCP failed:",
          formatError(error)
        );


        stop(
          1011,
          "TCP write failed"
        );
      }
    }
  );


  (async () => {

    try {

      const reader =
        socket.readable.getReader();


      while (!stopped) {

        const result =
          await reader.read();


        if (result.done) {
          break;
        }


        if (
          result.value &&
          result.value.byteLength
        ) {

          server.send(
            result.value
          );
        }
      }


      reader.releaseLock();


      if (!stopped) {

        stop(
          1000,
          "TCP closed"
        );
      }


    } catch (error) {

      console.error(
        "Legacy TCP -> WS failed:",
        formatError(error)
      );


      stop(
        1011,
        "TCP read failed"
      );
    }

  })();


  server.addEventListener(
    "close",
    () => {

      stopped = true;

      try {
        socket.close();
      } catch (_) {}
    }
  );


  return new Response(
    null,
    {
      status: 101,
      webSocket: client
    }
  );
}


/* =========================================================
   Helpers
========================================================= */

async function safeText(
  response
) {

  try {

    return await response.text();

  } catch (_) {

    return "";
  }
}


function formatError(
  error
) {

  if (!error) {
    return "Unknown error";
  }


  if (
    typeof error ===
      "string"
  ) {

    return error;
  }


  const result =
    [];


  if (error.name) {

    result.push(
      error.name
    );
  }


  if (error.message) {

    result.push(
      error.message
    );
  }


  if (
    error.code !==
      undefined
  ) {

    result.push(
      `code=${error.code}`
    );
  }


  return (
    result.join(" | ") ||
    String(error)
  );
}


function json(
  data,
  status = 200
) {

  return new Response(

    JSON.stringify(
      data,
      null,
      2
    ),

    {
      status,

      headers: {

        "content-type":
          "application/json; charset=UTF-8",

        "cache-control":
          "no-store"
      }
    }
  );
}


/* =========================================================
   Root HTML
========================================================= */

const ROOT_HTML = `<!DOCTYPE html>

<html lang="zh-CN">

<head>

<meta charset="UTF-8">

<meta
name="viewport"
content="width=device-width,initial-scale=1"
/>

<title>
Telegram Cloudflare Relay
</title>

<style>

* {
  box-sizing: border-box;
}

body {

  margin: 0;

  padding: 24px;

  background: #f5f5f7;

  color: #1d1d1f;

  font-family:
    -apple-system,
    BlinkMacSystemFont,
    "SF Pro Display",
    sans-serif;
}

.card {

  max-width: 720px;

  margin: 0 auto 16px;

  padding: 22px;

  background: #fff;

  border-radius: 20px;

  box-shadow:
    0 10px 30px
    rgba(0,0,0,.06);
}

h1 {

  margin: 0 0 8px;

  font-size: 24px;
}

a {

  color: #007aff;

  text-decoration: none;
}

.grid {

  display: grid;

  grid-template-columns:
    repeat(2, 1fr);

  gap: 10px;

  margin-top: 18px;
}

.item {

  padding: 14px;

  background: #f5f5f7;

  border-radius: 14px;
}

.label {

  font-size: 12px;

  color: #86868b;
}

.value {

  margin-top: 5px;

  font-weight: 600;
}

</style>

</head>

<body>

<div class="card">

<h1>
Telegram Cloudflare Relay
</h1>

<div>
V0.3-A
</div>

<div class="grid">

<div class="item">

<div class="label">
MTProto
</div>

<div class="value">
WebSocket
</div>

</div>

<div class="item">

<div class="label">
Obfuscation
</div>

<div class="value">
Client-side
</div>

</div>

<div class="item">

<div class="label">
Worker
</div>

<div class="value">
WSS Relay
</div>

</div>

<div class="item">

<div class="label">
Telegram
</div>

<div class="value">
DC2
</div>

</div>

</div>

</div>


<div class="card">

<h2>
MTProto 测试
</h2>

<p>
测试完整 Telegram MTProto WebSocket
链路。
</p>

<p>

<a href="/mtproto">
打开 MTProto 测试页面
</a>

</p>

</div>


<div class="card">

<h2>
Diagnostics
</h2>

<p>

<a href="/health">
/health
</a>

</p>

<p>

<a href="/apiws?dc=2">
/apiws
</a>

</p>

<p>

<a href="/mtws?dc=2">
/mtws
</a>

</p>

</div>

</body>

</html>`;


/* =========================================================
   MTProto Test Page
========================================================= */

const MTPROTO_HTML = `<!DOCTYPE html>

<html lang="zh-CN">

<head>

<meta charset="UTF-8">

<meta
name="viewport"
content="width=device-width,initial-scale=1"
/>

<title>
MTProto WebSocket Test
</title>

<style>

* {
  box-sizing: border-box;
}

body {

  margin: 0;

  padding:
    20px 16px 40px;

  background: #f5f5f7;

  color: #1d1d1f;

  font-family:
    -apple-system,
    BlinkMacSystemFont,
    "SF Pro Display",
    sans-serif;
}

.card {

  max-width: 720px;

  margin: 0 auto 16px;

  padding: 20px;

  background: #fff;

  border-radius: 20px;

  box-shadow:
    0 8px 30px
    rgba(0,0,0,.06);
}

h1 {

  margin:
    0 0 8px;

  font-size: 24px;
}

h2 {

  font-size: 18px;

  margin-top: 0;
}

input {

  width: 100%;

  padding: 13px 14px;

  margin:
    6px 0 10px;

  border:
    1px solid #ddd;

  border-radius: 12px;

  font-size: 16px;

  background: #fff;
}

button {

  width: 100%;

  padding: 14px;

  border: 0;

  border-radius: 13px;

  background: #007aff;

  color: white;

  font-size: 16px;

  font-weight: 600;
}

button:disabled {

  opacity: .45;
}

.status {

  padding: 14px;

  border-radius: 14px;

  background: #f5f5f7;

  margin:
    10px 0;
}

.log {

  background: #111;

  color: #eee;

  border-radius: 14px;

  padding: 14px;

  font-family:
    ui-monospace,
    SFMono-Regular,
    Menlo,
    monospace;

  font-size: 12px;

  line-height: 1.6;

  white-space: pre-wrap;

  word-break: break-word;

  min-height: 180px;

  max-height: 420px;

  overflow: auto;
}

small {

  color: #86868b;

  line-height: 1.5;
}

.success {

  color: #168a43;

  font-weight: 600;
}

.error {

  color: #d93025;

  font-weight: 600;
}

</style>

</head>

<body>


<div class="card">

<h1>
Telegram MTProto
</h1>

<div>
Cloudflare WebSocket Relay V0.3-A
</div>

</div>


<div class="card">

<h2>
Telegram API
</h2>

<label>
API ID
</label>

<input
id="apiId"
type="number"
placeholder="例如 12345678"
/>


<label>
API Hash
</label>

<input
id="apiHash"
type="password"
placeholder="Telegram API Hash"
/>


<label>
Bot Token
</label>

<input
id="botToken"
type="password"
placeholder="例如 123456:ABC..."
/>


<small>

API ID / API Hash / Bot Token
只在当前浏览器内存中使用，
不会提交给本 Worker。

</small>

</div>


<div class="card">

<h2>
Connection
</h2>

<div
id="status"
class="status"
>
未连接
</div>

<button
id="connect"
>
连接 Telegram
</button>

</div>


<div class="card">

<h2>
日志
</h2>

<div
id="log"
class="log"
></div>

</div>


<script>

/* =========================================================
   Global WebSocket Hook
========================================================= */

const NativeWebSocket =
  window.WebSocket;


/*
 * Telegram DC -> Worker
 */

const DC_MAP = {

  "pluto.web.telegram.org": 1,

  "venus.web.telegram.org": 2,

  "aurora.web.telegram.org": 3,

  "vesta.web.telegram.org": 4,

  "flora.web.telegram.org": 5

};


/*
 * The mtgo WASM client creates the browser
 * WebSocket itself.
 *
 * We intercept ONLY Telegram's WebSocket
 * URLs and rewrite them to our Worker.
 *
 * MTProto itself remains inside WASM.
 */

window.WebSocket =
  new Proxy(
    NativeWebSocket,
    {

      construct(
        Target,
        args
      ) {

        let url =
          args[0];


        if (
          typeof url ===
            "string"
        ) {

          try {

            const parsed =
              new URL(url);


            const hostname =
              parsed.hostname;


            const dc =
              DC_MAP[
                hostname
              ];


            if (
              dc &&
              parsed.pathname
                .startsWith(
                  "/apiws"
                )
            ) {

              const workerURL =
                new URL(
                  "/mtws",
                  location.origin
                );


              workerURL.searchParams
                .set(
                  "dc",
                  String(dc)
                );


              url =
                workerURL.toString();


              /*
               * Browser WebSocket constructor
               * expects ws:// or wss://.
               */

              workerURL.protocol =
                location.protocol ===
                  "https:"
                    ? "wss:"
                    : "ws:";


              url =
                workerURL.toString();


              args[0] =
                url;


              log(
                "↪ Telegram DC" +
                dc +
                " → " +
                url
              );
            }

          } catch (_) {}
        }


        return Reflect.construct(
          Target,
          args
        );
      }
    }
  );


/* =========================================================
   UI
========================================================= */

const $ =
  id =>
    document.getElementById(id);


function log(
  message
) {

  const box =
    $("log");


  const time =
    new Date()
      .toLocaleTimeString();


  box.textContent +=
    "[" +
    time +
    "] " +
    message +
    "\\n";


  box.scrollTop =
    box.scrollHeight;
}


function status(
  text,
  type = ""
) {

  const el =
    $("status");


  el.textContent =
    text;


  el.className =
    "status " +
    type;
}


/* =========================================================
   Load MTGo WASM
========================================================= */

let mtgo = null;

let client = null;


async function loadMTGo() {

  if (mtgo) {
    return mtgo;
  }


  log(
    "正在加载 @mtgo-labs/wasm..."
  );


  /*
   * Official plain-browser loading method
   * documented by the project.
   */

  mtgo =
    await import(
      "https://unpkg.com/@mtgo-labs/wasm/browser"
    )
      .then(
        module =>
          module.load(
            "https://unpkg.com/@mtgo-labs/wasm/mtgo-wasm.wasm.gz"
          )
      );


  log(
    "✅ MTProto WASM 已加载"
  );


  return mtgo;
}


/* =========================================================
   Connect
========================================================= */

$("connect")
  .addEventListener(
    "click",
    async () => {

      const apiId =
        Number(
          $("apiId").value
        );


      const apiHash =
        $("apiHash").value
          .trim();


      const botToken =
        $("botToken").value
          .trim();


      if (
        !apiId ||
        !apiHash ||
        !botToken
      ) {

        alert(
          "请填写 API ID、API Hash 和 Bot Token"
        );


        return;
      }


      $("connect")
        .disabled = true;


      $("log")
        .textContent = "";


      try {

        status(
          "正在加载 MTProto..."
        );


        const engine =
          await loadMTGo();


        log(
          "创建 MTProto Client..."
        );


        client =
          engine.createClient({

            apiID:
              apiId,

            apiHash:
              apiHash,

            botToken:
              botToken
          });


        status(
          "正在连接 Telegram..."
        );


        log(
          "开始 MTProto connect()"
        );


        await client.connect();


        status(
          "✅ Telegram MTProto 连接成功",
          "success"
        );


        log(
          "✅ MTProto connect() 成功"
        );


        const me =
          client.me();


        if (me) {

          log(
            "账号：" +
            JSON.stringify(
              me
            )
          );
        }


        /*
         * -------------------------------------------------
         * 真正的 Telegram API 调用
         * -------------------------------------------------
         *
         * help.getNearestDc
         *
         * 如果这里成功：
         *
         * Browser
         *   ↓
         * MTProto
         *   ↓
         * obfuscated2
         *   ↓
         * WSS
         *   ↓
         * Cloudflare Worker
         *   ↓
         * WSS
         *   ↓
         * Telegram
         *
         * 就真正打通了。
         */

        log(
          "调用 help.getNearestDc()..."
        );


        const result =
          await client.invoke(
            "help.getNearestDc",
            {}
          );


        log(
          "🎉 Telegram API 响应："
        );


        log(
          JSON.stringify(
            result,
            null,
            2
          )
        );


        status(
          "🎉 MTProto + WSS + Cloudflare + Telegram 全链路成功",
          "success"
        );


      } catch (error) {

        console.error(
          error
        );


        status(
          "❌ 连接失败",
          "error"
        );


        log(
          "❌ " +
          (
            error?.message ||
            String(error)
          )
        );


      } finally {

        $("connect")
          .disabled = false;
      }
    }
  );


log(
  "页面初始化完成"
);

log(
  "Worker: " +
  location.origin
);

log(
  "MTProto Relay: /mtws"
);

log(
  "DC2 → venus.web.telegram.org"
);

</script>


</body>

</html>`;