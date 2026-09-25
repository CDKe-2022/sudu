/**
 * Telegram MTProto over WebSocket
 * Cloudflare Worker V0.3
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
 *   ↓ WSS
 * Cloudflare Worker
 *   ↓ TCP :443
 * Telegram DC2
 */

const TELEGRAM_DCS = {
  1: [
    "149.154.175.50",
    "149.154.175.51",
  ],

  2: [
    "149.154.167.50",
    "149.154.167.41",
    "149.154.167.220",
  ],

  3: [
    "149.154.175.100",
    "149.154.167.91",
  ],

  4: [
    "149.154.167.92",
    "149.154.165.111",
  ],

  5: [
    "91.108.56.100",
    "91.108.56.101",
  ],
};

const DEFAULT_DC = 2;

export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);

    if (url.pathname === "/") {
      return new Response(ROOT_HTML, {
        headers: {
          "content-type": "text/html; charset=UTF-8",
          "cache-control": "no-store",
        },
      });
    }

    if (url.pathname === "/health") {
      return json({
        ok: true,
        service: "telegram-mtproto-cloudflare",
        version: "V0.3",
        websocket: true,
        tcp: true,
        defaultDC: DEFAULT_DC,
        timestamp: new Date().toISOString(),
      });
    }

    if (url.pathname === "/mtproto") {
      return new Response(MTPROTO_HTML, {
        headers: {
          "content-type": "text/html; charset=UTF-8",
          "cache-control": "no-store",
        },
      });
    }

    if (url.pathname === "/apiws") {
      return handleWebSocket(request);
    }

    return new Response("Not Found", {
      status: 404,
      headers: {
        "content-type": "text/plain; charset=UTF-8",
      },
    });
  },
};


/* =========================================================
 * WebSocket → Telegram TCP
 * =======================================================*/

async function handleWebSocket(request) {
  if (request.headers.get("Upgrade")?.toLowerCase() !== "websocket") {
    return new Response("Expected WebSocket", {
      status: 426,
      headers: {
        "upgrade": "websocket",
      },
    });
  }

  const pair = new WebSocketPair();

  const client = pair[0];
  const server = pair[1];

  const protocol = request.headers.get(
    "Sec-WebSocket-Protocol"
  );

  const selectedProtocol =
    protocol
      ?.split(",")
      .map(x => x.trim())
      .find(x => x === "binary");

  server.accept({
    allowHalfOpen: true,
  });

  // Telegram DC
  const url = new URL(request.url);

  let dc = Number(
    url.searchParams.get("dc") || DEFAULT_DC
  );

  if (!TELEGRAM_DCS[dc]) {
    dc = DEFAULT_DC;
  }

  const hosts = TELEGRAM_DCS[dc];

  // 当前先固定第一个地址
  const hostname = hosts[0];

  let socket;

  try {
    socket = connectTelegram(hostname);
  } catch (error) {
    safeClose(server, 1011, "TCP connect failed");

    return new Response(null, {
      status: 101,
      webSocket: client,
    });
  }

  let closed = false;

  const closeAll = () => {
    if (closed) return;

    closed = true;

    try {
      socket.close();
    } catch {}

    try {
      server.close();
    } catch {}
  };

  server.addEventListener("message", async event => {
    if (closed) return;

    try {
      let data;

      if (typeof event.data === "string") {
        data = new TextEncoder().encode(event.data);
      } else {
        data = new Uint8Array(event.data);
      }

      if (!data.length) return;

      const writer = socket.writable.getWriter();

      try {
        await writer.write(data);
      } finally {
        writer.releaseLock();
      }
    } catch (error) {
      console.error("WS → TCP error:", error);

      safeClose(server, 1011, "TCP write failed");

      try {
        socket.close();
      } catch {}
    }
  });

  server.addEventListener("close", closeAll);
  server.addEventListener("error", closeAll);

  // TCP → WebSocket
  (async () => {
    try {
      const reader = socket.readable.getReader();

      while (!closed) {
        const { value, done } = await reader.read();

        if (done) break;

        if (value && value.length) {
          server.send(value);
        }
      }

      reader.releaseLock();

      if (!closed) {
        safeClose(server, 1000, "TCP closed");
      }
    } catch (error) {
      console.error("TCP → WS error:", error);

      safeClose(server, 1011, "TCP read failed");
    }
  })();

  return new Response(null, {
    status: 101,
    webSocket: client,

    headers: selectedProtocol
      ? {
          "Sec-WebSocket-Protocol": selectedProtocol,
        }
      : {},
  });
}


/* =========================================================
 * TCP connection
 * =======================================================*/

function connectTelegram(hostname) {
  return connect({
    hostname,
    port: 443,
    secureTransport: "off",
    allowHalfOpen: true,
  });
}


/* =========================================================
 * Helpers
 * =======================================================*/

function safeClose(ws, code = 1000, reason = "") {
  try {
    ws.close(code, reason);
  } catch {}
}

function json(data, status = 200) {
  return new Response(JSON.stringify(data, null, 2), {
    status,
    headers: {
      "content-type": "application/json; charset=UTF-8",
      "cache-control": "no-store",
    },
  });
}


/* =========================================================
 * Root diagnostic page
 * =======================================================*/

const ROOT_HTML = `
<!doctype html>
<html lang="zh-CN">
<head>
<meta charset="utf-8">
<meta name="viewport"
      content="width=device-width,
               initial-scale=1,
               viewport-fit=cover">

<title>Telegram Cloudflare Node</title>

<style>
* {
  box-sizing: border-box;
}

body {
  margin: 0;
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
  max-width: 680px;
  margin: auto;
  padding:
    calc(env(safe-area-inset-top) + 32px)
    20px
    calc(env(safe-area-inset-bottom) + 40px);
}

.card {
  background: white;
  border-radius: 24px;
  padding: 24px;
  box-shadow:
    0 10px 35px rgba(0,0,0,.06);
}

h1 {
  margin: 0 0 8px;
  font-size: 28px;
}

p {
  color: #666;
  line-height: 1.6;
}

a {
  display: block;
  margin-top: 12px;
  padding: 16px;
  background: #f1f2f4;
  color: #111;
  text-decoration: none;
  border-radius: 14px;
}

code {
  font-family: ui-monospace, SFMono-Regular, Menlo, monospace;
}
</style>
</head>

<body>

<main>

<div class="card">

<h1>Telegram Cloudflare Node</h1>

<p>
V0.3 · MTProto WebSocket
</p>

<a href="/health">
查看 /health
</a>

<a href="/mtproto">
打开 MTProto 测试
</a>

</div>

</main>

</body>
</html>
`;


/* =========================================================
 * MTProto test page
 *
 * 注意：
 * 这个页面目前用于验证 Worker WSS transport。
 * 真正的 MTProto Client 不直接放在 Worker 中。
 * =======================================================*/

const MTPROTO_HTML = `
<!doctype html>
<html lang="zh-CN">

<head>

<meta charset="utf-8">

<meta name="viewport"
      content="width=device-width,
               initial-scale=1,
               viewport-fit=cover">

<title>MTProto · Cloudflare Test</title>

<style>

* {
  box-sizing: border-box;
}

body {
  margin: 0;

  min-height: 100vh;

  background:
    linear-gradient(
      180deg,
      #f7f8fa 0%,
      #eef0f3 100%
    );

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
  width: min(680px, 100%);
  margin: auto;

  padding:
    calc(env(safe-area-inset-top) + 28px)
    18px
    calc(env(safe-area-inset-bottom) + 40px);
}

.header {
  margin-bottom: 22px;
}

.title {
  font-size: 30px;
  font-weight: 700;
  letter-spacing: -.6px;
}

.subtitle {
  margin-top: 7px;

  color: #777;

  font-size: 14px;
}

.card {
  background: rgba(255,255,255,.92);

  border-radius: 24px;

  padding: 20px;

  box-shadow:
    0 12px 40px rgba(0,0,0,.07);
}

.row {
  display: flex;

  align-items: center;

  gap: 14px;

  padding: 15px 4px;

  border-bottom:
    1px solid #eee;
}

.row:last-child {
  border-bottom: 0;
}

.icon {
  width: 38px;
  height: 38px;

  border-radius: 12px;

  display: flex;
  align-items: center;
  justify-content: center;

  background: #f1f2f4;

  font-size: 18px;
}

.info {
  flex: 1;
}

.name {
  font-size: 15px;
  font-weight: 600;
}

.detail {
  margin-top: 4px;

  font-size: 12px;

  color: #999;
}

.status {
  font-size: 13px;

  font-weight: 600;

  color: #999;
}

.ok {
  color: #18a058;
}

.fail {
  color: #e5484d;
}

.wait {
  color: #d28a00;
}

button {
  width: 100%;

  margin-top: 18px;

  border: 0;

  border-radius: 15px;

  padding: 15px;

  background: #111;

  color: white;

  font-size: 16px;

  font-weight: 600;
}

button:disabled {
  opacity: .5;
}

.log {
  margin-top: 18px;

  padding: 14px;

  border-radius: 15px;

  background: #111;

  color: #ddd;

  font-family:
    ui-monospace,
    SFMono-Regular,
    Menlo,
    monospace;

  font-size: 11px;

  line-height: 1.65;

  white-space: pre-wrap;

  word-break: break-word;

  min-height: 80px;
}

.note {
  margin-top: 18px;

  color: #888;

  font-size: 12px;

  line-height: 1.6;
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
Cloudflare WebSocket Transport Test · V0.3
</div>

</div>


<div class="card">

<div class="row">

<div class="icon">🌐</div>

<div class="info">

<div class="name">
Cloudflare WSS
</div>

<div class="detail">
WebSocket handshake
</div>

</div>

<div id="wss"
     class="status wait">
等待
</div>

</div>


<div class="row">

<div class="icon">◈</div>

<div class="info">

<div class="name">
WebSocket Protocol
</div>

<div class="detail">
Sec-WebSocket-Protocol
</div>

</div>

<div id="protocol"
     class="status wait">
等待
</div>

</div>


<div class="row">

<div class="icon">↕</div>

<div class="info">

<div class="name">
Binary Stream
</div>

<div class="detail">
WebSocket ↔ TCP
</div>

</div>

<div id="stream"
     class="status wait">
等待
</div>

</div>


<div class="row">

<div class="icon">🔐</div>

<div class="info">

<div class="name">
MTProto Transport
</div>

<div class="detail">
Obfuscated transport
</div>

</div>

<div id="mtproto"
     class="status wait">
未测试
</div>

</div>


<div class="row">

<div class="icon">📡</div>

<div class="info">

<div class="name">
Telegram DC2
</div>

<div class="detail">
149.154.167.50:443
</div>

</div>

<div id="telegram"
     class="status wait">
等待
</div>

</div>

</div>


<button id="start">
开始测试
</button>


<div id="log"
     class="log">
等待开始……
</div>


<div class="note">

当前 V0.3 页面先验证 Cloudflare WSS →
Telegram TCP 的透明字节流。
真正 MTProto 2.0 / DH / auth key
需要 MTProto Client 参与。

</div>

</main>


<script>

const $ = id =>
  document.getElementById(id);

const logEl = $("log");

function log(message) {

  const time =
    new Date().toLocaleTimeString();

  logEl.textContent +=
    "\\n[" + time + "] " + message;
}

function status(id, text, type) {

  const el = $(id);

  el.textContent = text;

  el.className =
    "status " + type;
}

async function run() {

  $("start").disabled = true;

  logEl.textContent = "";

  status("wss", "连接中", "wait");

  status("protocol", "等待", "wait");

  status("stream", "等待", "wait");

  status("telegram", "连接中", "wait");

  try {

    const url =
      location.origin.replace(
        /^http/,
        "ws"
      ) +
      "/apiws?dc=2";

    log("连接：");

    log(url);

    const ws =
      new WebSocket(
        url,
        "binary"
      );

    ws.binaryType =
      "arraybuffer";

    ws.onopen = () => {

      status(
        "wss",
        "成功",
        "ok"
      );

      status(
        "protocol",
        ws.protocol || "binary",
        "ok"
      );

      status(
        "stream",
        "正常",
        "ok"
      );

      status(
        "telegram",
        "TCP 已连接",
        "ok"
      );

      log(
        "WebSocket 已连接"
      );

      log(
        "协议：" +
        (ws.protocol || "binary")
      );

      /*
       * 这里只发送测试字节。
       *
       * 0xEF 是 Abridged transport
       * 的协议标识。
       *
       * 它不是完整 MTProto
       * authentication。
       */

      ws.send(
        new Uint8Array([0xef])
      );

      log(
        "→ 已发送 Abridged 0xEF"
      );

      status(
        "mtproto",
        "Transport 测试",
        "wait"
      );
    };


    ws.onmessage = event => {

      const data =
        new Uint8Array(
          event.data
        );

      log(
        "← Telegram 返回 " +
        data.length +
        " bytes"
      );

      if (data.length) {

        const hex =
          [...data]
            .slice(0, 32)
            .map(
              x =>
                x
                  .toString(16)
                  .padStart(2, "0")
            )
            .join(" ");

        log(
          "HEX: " + hex
        );
      }
    };


    ws.onerror = () => {

      status(
        "wss",
        "失败",
        "fail"
      );

      status(
        "telegram",
        "失败",
        "fail"
      );

      log(
        "WebSocket error"
      );
    };


    ws.onclose = event => {

      log(
        "WebSocket closed: " +
        event.code
      );

      $("start").disabled =
        false;
    };


  } catch (error) {

    status(
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
    run
  );

</script>

</body>

</html>
`;