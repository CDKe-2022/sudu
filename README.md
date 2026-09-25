import { connect } from "cloudflare:sockets";

const VERSION = "0.1.1";
const WS_PATH = "/apiws";

const TELEGRAM_DC2 = [
  "149.154.167.50",
  "149.154.167.41",
  "149.154.167.220",
];

function json(data, status = 200) {
  return new Response(JSON.stringify(data, null, 2), {
    status,
    headers: {
      "content-type": "application/json; charset=utf-8",
      "cache-control": "no-store",
    },
  });
}

function isWebSocket(request) {
  return (
    request.headers.get("Upgrade")?.toLowerCase() ===
    "websocket"
  );
}

function testPage(url) {
  const wsUrl =
    `${url.protocol === "https:" ? "wss:" : "ws:"}` +
    `//${url.host}${WS_PATH}?dc=2`;

  return new Response(
`<!doctype html>
<html lang="zh-CN">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>CF Telegram Worker Test</title>
<style>
body {
  font-family: -apple-system, BlinkMacSystemFont, sans-serif;
  padding: 24px;
  line-height: 1.6;
}
button {
  font-size: 17px;
  padding: 12px 18px;
  border-radius: 12px;
  border: 0;
}
pre {
  white-space: pre-wrap;
  word-break: break-word;
  background: #f5f5f5;
  padding: 16px;
  border-radius: 12px;
}
</style>
</head>

<body>

<h2>Cloudflare Telegram Worker</h2>

<p>V0.1 WebSocket 测试</p>

<button id="test">测试 WSS</button>

<pre id="log">等待测试...</pre>

<script>

const log = document.getElementById("log");

function write(text) {
  log.textContent += "\\n" + text;
}

document.getElementById("test").onclick = () => {

  log.textContent = "正在建立 WebSocket...";

  const ws = new WebSocket(
    ${JSON.stringify(wsUrl)}
  );

  ws.binaryType = "arraybuffer";

  ws.onopen = () => {
    write("✅ WebSocket 已连接");

    /*
     * 发送一个测试字节。
     *
     * 这不是完整 MTProto 数据。
     * 这里只测试 Worker ↔ Telegram TCP relay。
     */
    ws.send(
      new Uint8Array([0xef])
    );

    write("→ 已发送测试数据");
  };

  ws.onmessage = (event) => {

    let length = 0;

    if (event.data instanceof ArrayBuffer) {
      length = event.data.byteLength;
    } else if (event.data instanceof Blob) {
      length = event.data.size;
    }

    write(
      "← Telegram 返回数据：" +
      length +
      " bytes"
    );
  };

  ws.onerror = () => {
    write("❌ WebSocket 错误");
  };

  ws.onclose = (event) => {
    write(
      "WebSocket 已关闭：" +
      event.code +
      " " +
      event.reason
    );
  };
};

</script>

</body>
</html>`,
    {
      headers: {
        "content-type": "text/html; charset=utf-8",
        "cache-control": "no-store",
      },
    },
  );
}

export default {
  async fetch(request) {

    const url = new URL(request.url);

    /*
     * ==========================
     * 测试页面
     * ==========================
     */

    if (url.pathname === "/") {
      return testPage(url);
    }

    /*
     * ==========================
     * WebSocket
     * ==========================
     */

    if (url.pathname !== WS_PATH) {
      return new Response("Not Found", {
        status: 404,
      });
    }

    if (!isWebSocket(request)) {
      return new Response(
        "Expected WebSocket",
        {
          status: 426,
          headers: {
            Upgrade: "websocket",
          },
        },
      );
    }

    const dc =
      url.searchParams.get("dc") || "2";

    if (dc !== "2") {
      return json(
        {
          error: "unsupported_dc",
        },
        400,
      );
    }

    const target = TELEGRAM_DC2[0];

    const pair = new WebSocketPair();

    const client = pair[0];
    const server = pair[1];

    server.accept();

    let socket;
    let reader;
    let writer;

    try {

      /*
       * Worker → Telegram DC
       */

      socket = connect({
        hostname: target,
        port: 443,
        secureTransport: "off",
        allowHalfOpen: true,
      });

      await socket.opened;

      reader =
        socket.readable.getReader();

      writer =
        socket.writable.getWriter();

      /*
       * WebSocket → TCP
       */

      server.addEventListener(
        "message",
        async (event) => {

          try {

            let data;

            if (
              event.data instanceof ArrayBuffer
            ) {
              data =
                new Uint8Array(
                  event.data
                );

            } else if (
              event.data instanceof Blob
            ) {
              data =
                new Uint8Array(
                  await event.data.arrayBuffer()
                );

            } else {

              server.close(
                1003,
                "Binary data required"
              );

              return;
            }

            if (data.byteLength > 0) {
              await writer.write(data);
            }

          } catch (error) {

            console.error(
              "WS -> TCP:",
              error
            );

            try {
              server.close(
                1011,
                "TCP write failed"
              );
            } catch {}
          }
        },
      );

      /*
       * TCP → WebSocket
       */

      (async () => {

        try {

          while (true) {

            const {
              value,
              done
            } = await reader.read();

            if (done) {
              break;
            }

            if (
              value &&
              server.readyState ===
                WebSocket.OPEN
            ) {

              server.send(value);
            }
          }

        } catch (error) {

          console.error(
            "TCP -> WS:",
            error
          );

        } finally {

          try {
            reader.releaseLock();
          } catch {}

          try {
            await socket.close();
          } catch {}

          try {
            if (
              server.readyState !==
              WebSocket.CLOSED
            ) {
              server.close(
                1000,
                "TCP closed"
              );
            }
          } catch {}
        }

      })();

      /*
       * WebSocket 关闭
       */

      server.addEventListener(
        "close",
        async () => {

          try {
            await reader.cancel();
          } catch {}

          try {
            await writer.close();
          } catch {}

          try {
            await socket.close();
          } catch {}
        },
      );

      return new Response(null, {
        status: 101,
        webSocket: client,
      });

    } catch (error) {

      console.error(
        "TCP connection failed:",
        error
      );

      try {
        server.close(
          1011,
          "Telegram connection failed"
        );
      } catch {}

      try {
        await socket?.close();
      } catch {}

      return new Response(
        "Unable to connect to Telegram DC",
        {
          status: 502,
        },
      );
    }
  },
};