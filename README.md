import { connect } from "cloudflare:sockets";

/*
 * CF Telegram Worker
 * Version: 0.1.0
 *
 * Architecture:
 *
 * Client
 *   ↓ WebSocket
 * Cloudflare Worker
 *   ↓ TCP
 * Telegram DC2
 *
 * V0.1:
 * - Only Telegram DC2
 * - Only port 443
 * - Only WebSocket
 * - No arbitrary TCP proxy
 * - No NAT Linux
 * - No VPS
 */

const VERSION = "0.1.0";
const WS_PATH = "/apiws";

/*
 * Telegram DC2 endpoints.
 *
 * V0.1 暂时只使用第一个地址。
 * 后续会加入自动 fallback。
 */
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
    request.headers.get("Upgrade")?.toLowerCase() === "websocket"
  );
}

export default {
  async fetch(request) {
    const url = new URL(request.url);

    /*
     * ==========================
     * 1. Health Check
     * ==========================
     */

    if (url.pathname === "/") {
      return json({
        name: "CF Telegram Worker",
        version: VERSION,
        status: "ok",
        cloudflare: true,
        target: "Telegram DC2",
        websocket: "/apiws?dc=2",
      });
    }

    /*
     * ==========================
     * 2. WebSocket Endpoint
     * ==========================
     */

    if (url.pathname !== WS_PATH) {
      return new Response("Not Found", {
        status: 404,
        headers: {
          "content-type": "text/plain; charset=utf-8",
        },
      });
    }

    /*
     * 必须是 WebSocket
     */
    if (!isWebSocket(request)) {
      return new Response("Expected WebSocket", {
        status: 426,
        headers: {
          Upgrade: "websocket",
        },
      });
    }

    /*
     * ==========================
     * 3. DC Selection
     * ==========================
     */

    const dc = url.searchParams.get("dc") || "2";

    if (dc !== "2") {
      return json(
        {
          error: "unsupported_dc",
          message: "V0.1 only supports Telegram DC2.",
        },
        400,
      );
    }

    /*
     * V0.1 固定 DC2 第一个地址
     */
    const target = TELEGRAM_DC2[0];

    /*
     * ==========================
     * 4. Create WebSocket Pair
     * ==========================
     */

    const pair = new WebSocketPair();

    const client = pair[0];
    const server = pair[1];

    server.accept();

    let socket = null;
    let tcpReader = null;
    let tcpWriter = null;

    try {
      /*
       * ==========================
       * 5. Worker → Telegram TCP
       * ==========================
       */

      socket = connect({
        hostname: target,
        port: 443,
        secureTransport: "off",
        allowHalfOpen: true,
      });

      await socket.opened;

      tcpReader = socket.readable.getReader();
      tcpWriter = socket.writable.getWriter();

      /*
       * ==========================
       * 6. WebSocket → TCP
       * ==========================
       */

      server.addEventListener("message", async (event) => {
        try {
          let data;

          /*
           * Telegram transport 使用 binary。
           */

          if (event.data instanceof ArrayBuffer) {
            data = new Uint8Array(event.data);
          } else if (event.data instanceof Blob) {
            data = new Uint8Array(
              await event.data.arrayBuffer(),
            );
          } else {
            server.close(
              1003,
              "Binary WebSocket data required",
            );
            return;
          }

          if (data.byteLength > 0) {
            await tcpWriter.write(data);
          }
        } catch (error) {
          console.error(
            "WebSocket -> TCP error:",
            error,
          );

          try {
            server.close(
              1011,
              "TCP write failed",
            );
          } catch {}
        }
      });

      /*
       * ==========================
       * 7. TCP → WebSocket
       * ==========================
       */

      (async () => {
        try {
          while (true) {
            const { value, done } =
              await tcpReader.read();

            if (done) {
              break;
            }

            if (!value) {
              continue;
            }

            if (
              server.readyState === WebSocket.OPEN
            ) {
              server.send(value);
            } else {
              break;
            }
          }
        } catch (error) {
          console.error(
            "TCP -> WebSocket error:",
            error,
          );
        } finally {
          try {
            tcpReader.releaseLock();
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
                "TCP connection closed",
              );
            }
          } catch {}
        }
      })();

      /*
       * ==========================
       * 8. Client Close
       * ==========================
       */

      server.addEventListener(
        "close",
        async () => {
          try {
            tcpReader?.cancel();
          } catch {}

          try {
            tcpWriter?.close();
          } catch {}

          try {
            await socket?.close();
          } catch {}
        },
      );

      /*
       * ==========================
       * 9. Return WebSocket
       * ==========================
       */

      return new Response(null, {
        status: 101,
        webSocket: client,
      });
    } catch (error) {
      console.error(
        "Telegram TCP connection failed:",
        error,
      );

      try {
        server.close(
          1011,
          "Telegram connection failed",
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