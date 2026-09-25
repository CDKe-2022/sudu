import { connect } from "cloudflare:sockets";

const VERSION = "0.2.2";

const WS_PATH = "/apiws";

const TELEGRAM_DC2 = [
  "149.154.167.50",
  "149.154.167.41",
  "149.154.167.220"
];


// ==================================================
// 工具：HTML 转义
// ==================================================

function escapeHtml(value) {
  return String(value)
    .replaceAll("&", "&amp;")
    .replaceAll("<", "&lt;")
    .replaceAll(">", "&gt;")
    .replaceAll('"', "&quot;")
    .replaceAll("'", "&#039;");
}


// ==================================================
// 首页
// ==================================================

function homePage(request) {

  const url = new URL(request.url);

  const wsUrl =
    `wss://${url.host}${WS_PATH}?dc=2&diag=1`;

  return new Response(`<!doctype html>
<html lang="zh-CN">

<head>

<meta charset="utf-8">

<meta name="viewport"
      content="width=device-width,
               initial-scale=1,
               viewport-fit=cover">

<title>Telegram WSS Relay ${VERSION}</title>

<style>

* {
  box-sizing: border-box;
}

body {
  margin: 0;
  padding:
    max(24px, env(safe-area-inset-top))
    20px
    max(24px, env(safe-area-inset-bottom));

  background: #f5f5f7;

  color: #1d1d1f;

  font-family:
    -apple-system,
    BlinkMacSystemFont,
    "SF Pro Display",
    "Helvetica Neue",
    Arial,
    sans-serif;
}

.card {
  max-width: 680px;

  margin: 30px auto;

  padding: 22px;

  background: #fff;

  border-radius: 20px;

  box-shadow:
    0 10px 40px rgba(0,0,0,.08);
}

h1 {
  margin: 0 0 6px;

  font-size: 24px;

  letter-spacing: -0.3px;
}

.version {
  color: #86868b;

  font-size: 14px;

  margin-bottom: 22px;
}

button {
  width: 100%;

  padding: 15px;

  border: 0;

  border-radius: 14px;

  background: #007aff;

  color: white;

  font-size: 16px;

  font-weight: 600;

  -webkit-tap-highlight-color: transparent;
}

button:disabled {
  opacity: .5;
}

.status {
  margin-top: 18px;

  padding: 16px;

  background: #f5f5f7;

  border-radius: 14px;
}

.row {
  display: flex;

  justify-content: space-between;

  gap: 15px;

  padding: 7px 0;

  font-size: 14px;

  border-bottom:
    1px solid #e5e5e7;
}

.row:last-child {
  border-bottom: 0;
}

.label {
  color: #86868b;
}

.value {
  text-align: right;

  font-weight: 500;

  word-break: break-word;
}

pre {
  margin-top: 18px;

  padding: 15px;

  min-height: 180px;

  overflow: auto;

  background: #111;

  color: #d7ffd9;

  border-radius: 14px;

  font-size: 12px;

  line-height: 1.6;

  white-space: pre-wrap;

  word-break: break-word;
}

.good {
  color: #16803c;
}

.bad {
  color: #d70015;
}

.wait {
  color: #996600;
}

</style>

</head>


<body>

<div class="card">

<h1>Telegram WSS Relay</h1>

<div class="version">
Cloudflare Worker ${VERSION}
</div>


<button id="test">
开始完整诊断
</button>


<div class="status">

<div class="row">
<span class="label">WebSocket</span>
<span class="value" id="wsStatus">等待</span>
</div>

<div class="row">
<span class="label">协议</span>
<span class="value" id="protocol">—</span>
</div>

<div class="row">
<span class="label">Telegram DC</span>
<span class="value" id="dc">—</span>
</div>

<div class="row">
<span class="label">TCP</span>
<span class="value" id="tcpStatus">等待</span>
</div>

<div class="row">
<span class="label">测试数据</span>
<span class="value" id="dataStatus">—</span>
</div>

<div class="row">
<span class="label">服务器响应</span>
<span class="value" id="responseStatus">—</span>
</div>

</div>


<pre id="log">点击上面的按钮开始测试。</pre>

</div>


<script>

const button =
  document.getElementById("test");

const logBox =
  document.getElementById("log");

const wsStatus =
  document.getElementById("wsStatus");

const protocol =
  document.getElementById("protocol");

const dc =
  document.getElementById("dc");

const tcpStatus =
  document.getElementById("tcpStatus");

const dataStatus =
  document.getElementById("dataStatus");

const responseStatus =
  document.getElementById("responseStatus");


function log(message) {

  logBox.textContent +=
    "\\n" + message;
}


function setStatus(element, text, type) {

  element.textContent = text;

  element.className =
    "value " + (type || "");
}


function reset() {

  wsStatus.textContent = "等待";
  wsStatus.className = "value";

  protocol.textContent = "—";
  protocol.className = "value";

  dc.textContent = "—";
  dc.className = "value";

  tcpStatus.textContent = "等待";
  tcpStatus.className = "value";

  dataStatus.textContent = "—";
  dataStatus.className = "value";

  responseStatus.textContent = "—";
  responseStatus.className = "value";

  logBox.textContent =
    "正在启动诊断...";
}


button.onclick = () => {

  reset();

  button.disabled = true;


  log("① 建立 WSS...");
  log("目标：${wsUrl}");


  const ws =
    new WebSocket(
      "${wsUrl}",
      "binary"
    );


  ws.binaryType =
    "arraybuffer";


  ws.onopen = () => {

    setStatus(
      wsStatus,
      "✅ 已连接",
      "good"
    );


    protocol.textContent =
      ws.protocol || "(无)";


    protocol.className =
      ws.protocol === "binary"
        ? "value good"
        : "value bad";


    log(
      "② WSS 握手成功"
    );


    log(
      "③ 协商协议：" +
      (ws.protocol || "(无)")
    );


    log(
      "等待 Worker 建立 Telegram TCP..."
    );
  };


  ws.onmessage = event => {

    // --------------------------------------------
    // Worker 诊断消息
    // --------------------------------------------

    if (
      typeof event.data === "string"
    ) {

      try {

        const message =
          JSON.parse(event.data);


        if (
          message.type ===
          "diagnostic"
        ) {

          if (
            message.stage ===
            "tcp_connecting"
          ) {

            setStatus(
              tcpStatus,
              "连接中…",
              "wait"
            );

            dc.textContent =
              message.target || "DC2";

            log(
              "④ Worker 正在连接 Telegram："
              + message.target
            );
          }


          if (
            message.stage ===
            "tcp_connected"
          ) {

            setStatus(
              tcpStatus,
              "✅ 已连接",
              "good"
            );

            dc.textContent =
              message.target;

            log(
              "⑤ Telegram TCP 已连接："
              + message.target
            );
          }


          if (
            message.stage ===
            "tcp_failed"
          ) {

            setStatus(
              tcpStatus,
              "❌ 失败",
              "bad"
            );

            log(
              "⑤ Telegram TCP 连接失败："
              + message.error
            );
          }


          if (
            message.stage ===
            "tcp_closed"
          ) {

            setStatus(
              tcpStatus,
              "已关闭",
              "wait"
            );

            log(
              "⑥ Telegram TCP 已关闭"
            );
          }

        }

      } catch {

        log(
          "← 收到文本数据："
          + event.data
        );
      }

      return;
    }


    // --------------------------------------------
    // Telegram TCP 返回的二进制
    // --------------------------------------------

    if (
      event.data instanceof ArrayBuffer
    ) {

      const bytes =
        new Uint8Array(
          event.data
        );


      setStatus(
        responseStatus,
        "收到 " +
        bytes.length +
        " bytes",
        "good"
      );


      log(
        "⑦ ← Telegram 返回 " +
        bytes.length +
        " bytes"
      );


      const preview =
        Array.from(
          bytes.slice(0, 32)
        )
        .map(
          b =>
            b.toString(16)
             .padStart(2, "0")
        )
        .join(" ");


      log(
        "数据：" + preview
      );
    }

  };


  ws.onerror = () => {

    setStatus(
      wsStatus,
      "❌ Error",
      "bad"
    );


    log(
      "❌ WebSocket error"
    );

  };


  ws.onclose = event => {

    log(
      "WebSocket 已关闭：" +
      event.code +
      " " +
      (event.reason || "")
    );


    log(
      "wasClean: " +
      event.wasClean
    );


    button.disabled = false;

  };

};

</script>

</body>

</html>`, {

    headers: {

      "content-type":
        "text/html; charset=UTF-8",

      "cache-control":
        "no-store"

    }

  });

}


// ==================================================
// 连接 Telegram DC2
// ==================================================

async function connectTelegram(
  report
) {

  let lastError = null;


  for (
    const hostname of
    TELEGRAM_DC2
  ) {

    report({
      type: "diagnostic",
      stage: "tcp_connecting",
      target: hostname
    });


    try {

      const socket =
        connect({

          hostname,

          port: 443,

          secureTransport: "off",

          allowHalfOpen: true

        });


      await socket.opened;


      report({

        type: "diagnostic",

        stage: "tcp_connected",

        target: hostname

      });


      return {
        socket,
        hostname
      };


    } catch (error) {

      lastError = error;


      report({

        type: "diagnostic",

        stage: "tcp_failed",

        target: hostname,

        error:
          error?.message ||
          String(error)

      });

    }

  }


  throw (
    lastError ||
    new Error(
      "All Telegram DC2 addresses failed"
    )
  );

}


// ==================================================
// WebSocket Relay
// ==================================================

async function handleWebSocket(
  request
) {

  const url =
    new URL(request.url);


  const diagnostic =
    url.searchParams.get(
      "diag"
    ) === "1";


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
        status: 426
      }
    );

  }


  const requestedProtocol =
    request.headers.get(
      "Sec-WebSocket-Protocol"
    );


  console.log(
    "Requested protocol:",
    requestedProtocol || "(none)"
  );


  const pair =
    new WebSocketPair();


  const client =
    pair[0];

  const server =
    pair[1];


  server.accept({
    allowHalfOpen: true
  });


  let socket = null;

  let tcpWriter = null;

  let tcpReader = null;

  let closed = false;


  function sendDiagnostic(
    data
  ) {

    if (!diagnostic) {
      return;
    }


    try {

      if (
        server.readyState ===
        WebSocket.OPEN
      ) {

        server.send(
          JSON.stringify(data)
        );

      }

    } catch {}

  }


  function cleanup() {

    if (closed) {
      return;
    }


    closed = true;


    try {

      if (tcpWriter) {
        tcpWriter.releaseLock();
      }

    } catch {}


    try {

      if (tcpReader) {
        tcpReader.releaseLock();
      }

    } catch {}


    try {

      if (socket) {
        socket.close();
      }

    } catch {}

  }


  // =================================================
  // WebSocket → Telegram TCP
  // =================================================

  server.addEventListener(
    "message",
    async event => {

      if (
        closed ||
        !tcpWriter
      ) {

        return;
      }


      try {

        let data =
          event.data;


        if (
          typeof data ===
          "string"
        ) {

          /*
           * 诊断模式下：
           * 浏览器不会主动发送文本，
           * 因此这里只处理普通 relay。
           */

          data =
            new TextEncoder()
              .encode(data);

        }


        else if (
          data instanceof
          ArrayBuffer
        ) {

          data =
            new Uint8Array(
              data
            );

        }


        else if (
          ArrayBuffer.isView(
            data
          )
        ) {

          data =
            new Uint8Array(
              data.buffer,
              data.byteOffset,
              data.byteLength
            );

        }


        else {

          return;
        }


        await tcpWriter.write(
          data
        );


        console.log(
          "WS -> TCP:",
          data.byteLength,
          "bytes"
        );


      } catch (error) {

        console.log(
          "WS -> TCP error:",
          error
        );


        cleanup();

      }

    }
  );


  // =================================================
  // WebSocket close
  // =================================================

  server.addEventListener(
    "close",
    event => {

      console.log(
        "WebSocket closed:",
        event.code,
        event.reason || ""
      );


      cleanup();

    }
  );


  // =================================================
  // 建立 Telegram TCP
  // =================================================

  try {

    const result =
      await connectTelegram(
        sendDiagnostic
      );


    socket =
      result.socket;


    tcpWriter =
      socket.writable
        .getWriter();


    tcpReader =
      socket.readable
        .getReader();


    // =================================================
    // Telegram TCP → WebSocket
    // =================================================

    (async () => {

      try {

        while (!closed) {

          const result =
            await tcpReader.read();


          if (result.done) {

            sendDiagnostic({

              type:
                "diagnostic",

              stage:
                "tcp_closed"

            });


            break;
          }


          const data =
            result.value;


          if (
            data &&
            server.readyState ===
              WebSocket.OPEN
          ) {

            server.send(
              data
            );


            console.log(
              "TCP -> WS:",
              data.byteLength,
              "bytes"
            );

          }

        }

      } catch (error) {

        console.log(
          "TCP -> WS error:",
          error
        );

      } finally {

        cleanup();

      }

    })();


  } catch (error) {

    console.log(
      "Telegram TCP failed:",
      error
    );


    sendDiagnostic({

      type:
        "diagnostic",

      stage:
        "tcp_failed",

      error:
        error?.message ||
        String(error)

    });


    try {

      server.close(
        1011,
        "Telegram TCP failed"
      );

    } catch {}


    cleanup();

  }


  // =================================================
  // WebSocket handshake response
  // =================================================

  const headers =
    new Headers();


  const protocols =
    (requestedProtocol || "")
      .split(",")
      .map(
        x => x.trim()
      )
      .filter(Boolean);


  if (
    protocols.includes(
      "binary"
    )
  ) {

    headers.set(
      "Sec-WebSocket-Protocol",
      "binary"
    );

  }


  return new Response(
    null,
    {

      status: 101,

      headers,

      webSocket:
        client

    }
  );

}


// ==================================================
// Worker 主入口
// ==================================================

export default {

  async fetch(
    request
  ) {

    const url =
      new URL(request.url);


    // ----------------------------------------------
    // 首页
    // ----------------------------------------------

    if (
      request.method === "GET" &&
      url.pathname === "/"
    ) {

      return homePage(
        request
      );

    }


    // ----------------------------------------------
    // WebSocket
    // ----------------------------------------------

    if (
      url.pathname ===
      WS_PATH
    ) {

      return handleWebSocket(
        request
      );

    }


    // ----------------------------------------------
    // Health
    // ----------------------------------------------

    if (
      url.pathname ===
      "/health"
    ) {

      return Response.json({

        ok: true,

        service:
          "telegram-wss-relay",

        version:
          VERSION,

        dc:
          2,

        websocket:
          true,

        tcp:
          true

      });

    }


    return new Response(
      "Not Found",
      {
        status: 404
      }
    );

  }

};