/**

 * Telegram Cloudflare WebSocket Relay

 *

 * V0.3.2

 *

 * Domain:

 *   tg.cdfcc.kdns.fr

 *

 * Routes:

 *   /

 *   /health

 *   /apiws

 *   /mtproto

 *

 * Architecture:

 *

 * Browser

 *    ↓ WSS

 * Cloudflare Worker

 *    ↓ TCP :443

 * Telegram DC2

 *

 * IMPORTANT:

 * This version is a transparent WebSocket ↔ TCP relay.

 * It does NOT implement MTProto encryption/authentication.

 */

/* =========================================================

 * Telegram DC

 * ======================================================= */

const TELEGRAM_DCS = {

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

 * Worker

 * ======================================================= */

export default {

  async fetch(request, env, ctx) {

    const url =

      new URL(request.url);

    /* -----------------------------------------------------

     * /

     * --------------------------------------------------- */

    if (url.pathname === "/") {

      return new Response(

        ROOT_HTML,

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

     * /health

     * --------------------------------------------------- */

    if (url.pathname === "/health") {

      return json({

        ok: true,

        service:

          "telegram-cloudflare-relay",

        version:

          "V0.3.2",

        websocket:

          true,

        tcp:

          true,

        defaultDC:

          DEFAULT_DC,

        timestamp:

          new Date().toISOString()

      });

    }

    /* -----------------------------------------------------

     * /mtproto

     * --------------------------------------------------- */

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

     * /apiws

     * --------------------------------------------------- */

    if (url.pathname === "/apiws") {

      return handleWebSocket(

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

 * WebSocket Relay

 * ======================================================= */

async function handleWebSocket(request) {

  /* -------------------------------------------------------

   * Check Upgrade

   * ----------------------------------------------------- */

  const upgrade =

    request.headers.get(

      "Upgrade"

    );

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

  /* -------------------------------------------------------

   * URL / DC

   * ----------------------------------------------------- */

  const url =

    new URL(request.url);

  let dc =

    Number(

      url.searchParams.get(

        "dc"

      ) || DEFAULT_DC

    );

  if (

    !TELEGRAM_DCS[dc]

  ) {

    dc =

      DEFAULT_DC;

  }

  const hostname =

    TELEGRAM_DCS[dc][0];

  /* -------------------------------------------------------

   * WebSocket pair

   * ----------------------------------------------------- */

  const pair =

    new WebSocketPair();

  const client =

    pair[0];

  const server =

    pair[1];

  /* -------------------------------------------------------

   * Accept WebSocket

   * ----------------------------------------------------- */

  server.accept({

    allowHalfOpen: true

  });

  /*

   * IMPORTANT

   *

   * connect() 是异步 API。

   *

   * 必须 await。

   */

  let socket;

  try {

    console.log(

      "Connecting Telegram:",

      hostname,

      443

    );

    socket =

      await connect({

        hostname:

          hostname,

        port:

          443,

        secureTransport:

          "off",

        allowHalfOpen:

          true

      });

    console.log(

      "Telegram TCP connected:",

      hostname

    );

  } catch (error) {

    console.error(

      "Telegram TCP connection failed:",

      error

    );

    try {

      server.close(

        1011,

        "TCP connection failed"

      );

    } catch {}

    return new Response(

      null,

      {

        status: 101,

        webSocket:

          client

      }

    );

  }

  /* -------------------------------------------------------

   * Relay state

   * ----------------------------------------------------- */

  let stopped =

    false;

  /* -------------------------------------------------------

   * Close everything

   * ----------------------------------------------------- */

  function stop(

    code = 1000,

    reason = "closed"

  ) {

    if (stopped) {

      return;

    }

    stopped =

      true;

    try {

      socket.close();

    } catch {}

    try {

      server.close(

        code,

        reason

      );

    } catch {}

  }

  /* =======================================================

   * WebSocket → TCP

   * ===================================================== */

  server.addEventListener(

    "message",

    async event => {

      if (stopped) {

        return;

      }

      try {

        let data;

        /* -----------------------------------------------

         * String

         * --------------------------------------------- */

        if (

          typeof event.data ===

          "string"

        ) {

          data =

            new TextEncoder()

              .encode(

                event.data

              );

        }

        /* -----------------------------------------------

         * Blob

         * --------------------------------------------- */

        else if (

          typeof Blob !==

            "undefined" &&

          event.data instanceof

            Blob

        ) {

          data =

            new Uint8Array(

              await event.data

                .arrayBuffer()

            );

        }

        /* -----------------------------------------------

         * ArrayBuffer / TypedArray

         * --------------------------------------------- */

        else {

          data =

            new Uint8Array(

              event.data

            );

        }

        if (

          !data ||

          data.byteLength === 0

        ) {

          return;

        }

        const writer =

          socket.writable

            .getWriter();

        try {

          await writer.write(

            data

          );

        } finally {

          writer.releaseLock();

        }

      } catch (error) {

        console.error(

          "WS → TCP failed:",

          error

        );

        stop(

          1011,

          "TCP write failed"

        );

      }

    }

  );

  /* =======================================================

   * TCP → WebSocket

   * ===================================================== */

  (async () => {

    try {

      const reader =

        socket.readable

          .getReader();

      while (!stopped) {

        const result =

          await reader.read();

        if (

          result.done

        ) {

          break;

        }

        const data =

          result.value;

        if (

          data &&

          data.byteLength > 0

        ) {

          server.send(

            data

          );

        }

      }

      reader.releaseLock();

      if (!stopped) {

        stop(

          1000,

          "Telegram TCP closed"

        );

      }

    } catch (error) {

      console.error(

        "TCP → WS failed:",

        error

      );

      stop(

        1011,

        "TCP read failed"

      );

    }

  })();

  /* =======================================================

   * WebSocket close

   * ===================================================== */

  server.addEventListener(

    "close",

    event => {

      console.log(

        "Client WebSocket closed:",

        event.code,

        event.reason

      );

      stopped =

        true;

      try {

        socket.close();

      } catch {}

    }

  );

  /* =======================================================

   * WebSocket error

   * ===================================================== */

  server.addEventListener(

    "error",

    error => {

      console.error(

        "Client WebSocket error:",

        error

      );

      stopped =

        true;

      try {

        socket.close();

      } catch {}

    }

  );

  /* =======================================================

   * Response

   * ===================================================== */

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

 * JSON

 * ======================================================= */

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

 * Root HTML

 * ======================================================= */

const ROOT_HTML = `

<!DOCTYPE html>

<html lang="zh-CN">

<head>

<meta charset="UTF-8">

<meta

  name="viewport"

  content="width=device-width,

           initial-scale=1,

           viewport-fit=cover"

>

<title>

Telegram Cloudflare

</title>

<style>

* {

  box-sizing: border-box;

}

body {

  margin: 0;

  min-height: 100vh;

  background: #f5f6f8;

  color: #111;

  font-family:

    -apple-system,

    BlinkMacSystemFont,

    "SF Pro Display",

    "Helvetica Neue",

    Arial,

    sans-serif;

}

main {

  width:

    min(680px,100%);

  margin:

    auto;

  padding:

    calc(

      env(safe-area-inset-top)

      + 32px

    )

    20px

    calc(

      env(safe-area-inset-bottom)

      + 40px

    );

}

.card {

  background:

    #fff;

  border-radius:

    24px;

  padding:

    24px;

  box-shadow:

    0 12px 40px

    rgba(0,0,0,.06);

}

h1 {

  margin:

    0;

  font-size:

    28px;

}

.subtitle {

  margin-top:

    8px;

  color:

    #777;

  font-size:

    14px;

}

.link {

  display:

    block;

  margin-top:

    14px;

  padding:

    16px;

  border-radius:

    15px;

  background:

    #f1f2f4;

  color:

    #111;

  text-decoration:

    none;

  font-weight:

    600;

}

</style>

</head>

<body>

<main>

<div class="card">

<h1>

Telegram Cloudflare

</h1>

<div class="subtitle">

V0.3.2 · WebSocket → TCP

</div>

<a

  class="link"

  href="/health"

>

健康检查

</a>

<a

  class="link"

  href="/mtproto"

>

MTProto 测试

</a>

</div>

</main>

</body>

</html>

`;

/* =========================================================

 * MTProto test HTML

 * ======================================================= */

const MTPROTO_HTML = `

<!DOCTYPE html>

<html lang="zh-CN">

<head>

<meta charset="UTF-8">

<meta

  name="viewport"

  content="width=device-width,

           initial-scale=1,

           viewport-fit=cover"

>

<title>

MTProto Test

</title>

<style>

* {

  box-sizing:

    border-box;

}

body {

  margin:

    0;

  min-height:

    100vh;

  background:

    linear-gradient(

      180deg,

      #f7f8fa,

      #eef0f3

    );

  color:

    #111;

  font-family:

    -apple-system,

    BlinkMacSystemFont,

    "SF Pro Display",

    "Helvetica Neue",

    Arial,

    sans-serif;

}

main {

  width:

    min(680px,100%);

  margin:

    auto;

  padding:

    calc(

      env(safe-area-inset-top)

      + 28px

    )

    18px

    calc(

      env(safe-area-inset-bottom)

      + 40px

    );

}

.header {

  margin-bottom:

    20px;

}

.title {

  font-size:

    30px;

  font-weight:

    700;

  letter-spacing:

    -.7px;

}

.subtitle {

  margin-top:

    7px;

  color:

    #777;

  font-size:

    14px;

}

.card {

  background:

    rgba(

      255,

      255,

      255,

      .94

    );

  border-radius:

    24px;

  padding:

    18px 20px;

  box-shadow:

    0 12px 40px

    rgba(0,0,0,.07);

}

.row {

  display:

    flex;

  align-items:

    center;

  gap:

    13px;

  padding:

    15px 2px;

  border-bottom:

    1px solid #eee;

}

.row:last-child {

  border-bottom:

    0;

}

.icon {

  width:

    40px;

  height:

    40px;

  flex:

    0 0 40px;

  border-radius:

    12px;

  display:

    flex;

  align-items:

    center;

  justify-content:

    center;

  background:

    #f1f2f4;

  font-size:

    18px;

}

.info {

  flex:

    1;

}

.name {

  font-size:

    15px;

  font-weight:

    600;

}

.detail {

  margin-top:

    4px;

  color:

    #999;

  font-size:

    12px;

}

.status {

  font-size:

    13px;

  font-weight:

    600;

  color:

    #999;

}

.ok {

  color:

    #18a058;

}

.fail {

  color:

    #e5484d;

}

.wait {

  color:

    #c18400;

}

button {

  width:

    100%;

  margin-top:

    18px;

  border:

    0;

  border-radius:

    15px;

  padding:

    16px;

  background:

    #111;

  color:

    #fff;

  font-size:

    16px;

  font-weight:

    600;

}

button:disabled {

  opacity:

    .5;

}

.log {

  margin-top:

    18px;

  min-height:

    110px;

  padding:

    15px;

  border-radius:

    16px;

  background:

    #111;

  color:

    #d8d8d8;

  font-family:

    ui-monospace,

    SFMono-Regular,

    Menlo,

    monospace;

  font-size:

    11px;

  line-height:

    1.65;

  white-space:

    pre-wrap;

  word-break:

    break-word;

}

.note {

  margin-top:

    16px;

  color:

    #888;

  font-size:

    12px;

  line-height:

    1.6;

}

</style>

</head>

<body>

<main>

<div class="header">

<div class="title">

MTProto

</div>

<div class="subtitle">

Cloudflare WebSocket Relay · V0.3.2

</div>

</div>

<div class="card">

<div class="row">

<div class="icon">

🌐

</div>

<div class="info">

<div class="name">

Cloudflare WSS

</div>

<div class="detail">

WebSocket handshake

</div>

</div>

<div

  id="wss"

  class="status wait"

>

等待

</div>

</div>

<div class="row">

<div class="icon">

↕

</div>

<div class="info">

<div class="name">

Binary Stream

</div>

<div class="detail">

WebSocket ↔ TCP

</div>

</div>

<div

  id="stream"

  class="status wait"

>

等待

</div>

</div>

<div class="row">

<div class="icon">

📡

</div>

<div class="info">

<div class="name">

Telegram DC2

</div>

<div class="detail">

149.154.167.50:443

</div>

</div>

<div

  id="telegram"

  class="status wait"

>

等待

</div>

</div>

<div class="row">

<div class="icon">

🔐

</div>

<div class="info">

<div class="name">

MTProto

</div>

<div class="detail">

Transport test

</div>

</div>

<div

  id="mtproto"

  class="status wait"

>

未测试

</div>

</div>

</div>

<button id="start">

开始测试

</button>

<div

  id="log"

  class="log"

>等待开始……</div>

<div class="note">

V0.3.2 当前验证：

浏览器

→ WSS

→ Cloudflare Worker

→ Telegram TCP

0xEF 只是 Abridged Transport

协议标识，不代表已经完成

MTProto 2.0 authentication。

</div>

</main>

<script>

const $ =

  id =>

    document.getElementById(id);

const logEl =

  $("log");

function log(

  message

) {

  const time =

    new Date()

      .toLocaleTimeString();

  logEl.textContent +=

    "\\n[" +

    time +

    "] " +

    message;

}

function setStatus(

  id,

  text,

  type

) {

  const el =

    $(id);

  el.textContent =

    text;

  el.className =

    "status " +

    type;

}

async function runTest() {

  $("start").disabled =

    true;

  logEl.textContent =

    "";

  setStatus(

    "wss",

    "连接中",

    "wait"

  );

  setStatus(

    "stream",

    "等待",

    "wait"

  );

  setStatus(

    "telegram",

    "等待",

    "wait"

  );

  setStatus(

    "mtproto",

    "测试中",

    "wait"

  );

  try {

    /* -----------------------------------------------

     * WebSocket URL

     * --------------------------------------------- */

    const protocol =

      location.protocol ===

      "https:"

        ? "wss:"

        : "ws:";

    const url =

      protocol +

      "//" +

      location.host +

      "/apiws?dc=2";

    log(

      "连接："

    );

    log(

      url

    );

    /* -----------------------------------------------

     * WebSocket

     *

     * 暂时不指定 binary subprotocol。

     * --------------------------------------------- */

    const ws =

      new WebSocket(

        url

      );

    ws.binaryType =

      "arraybuffer";

    /* -----------------------------------------------

     * Open

     * --------------------------------------------- */

    ws.onopen = () => {

      setStatus(

        "wss",

        "成功",

        "ok"

      );

      setStatus(

        "stream",

        "正常",

        "ok"

      );

      setStatus(

        "telegram",

        "TCP 已连接",

        "ok"

      );

      log(

        "WebSocket 已连接"

      );

      log(

        "protocol = " +

        (

          ws.protocol ||

          "none"

        )

      );

      /*

       * Abridged transport marker

       */

      ws.send(

        new Uint8Array([

          0xef

        ])

      );

      log(

        "→ 已发送 0xEF"

      );

      setStatus(

        "mtproto",

        "字节已发送",

        "wait"

      );

    };

    /* -----------------------------------------------

     * Message

     * --------------------------------------------- */

    ws.onmessage =

      async event => {

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

          event.data instanceof

            Blob

        ) {

          data =

            new Uint8Array(

              await event.data

                .arrayBuffer()

            );

        }

        else {

          log(

            "← 收到未知数据类型"

          );

          return;

        }

        log(

          "← Telegram 返回 " +

          data.byteLength +

          " bytes"

        );

        if (

          data.byteLength

          > 0

        ) {

          const hex =

            Array.from(

              data.slice(

                0,

                32

              )

            )

            .map(

              x =>

                x

                  .toString(16)

                  .padStart(

                    2,

                    "0"

                  )

            )

            .join(" ");

          log(

            "HEX: " +

            hex

          );

        }

      };

    /* -----------------------------------------------

     * Error

     * --------------------------------------------- */

    ws.onerror =

      () => {

        setStatus(

          "wss",

          "失败",

          "fail"

        );

        log(

          "WebSocket error"

        );

      };

    /* -----------------------------------------------

     * Close

     * --------------------------------------------- */

    ws.onclose =

      event => {

        log(

          "WebSocket closed: " +

          event.code +

          " / " +

          (

            event.reason ||

            "no reason"

          )

        );

        $("start").disabled =

          false;

      };

  } catch (error) {

    setStatus(

      "wss",

      "失败",

      "fail"

    );

    log(

      "ERROR: " +

      error.message

    );

    $("start").disabled =

      false;

  }

}

$("start")

  .addEventListener(

    "click",

    runTest

  );

</script>

</body>

</html>

`;