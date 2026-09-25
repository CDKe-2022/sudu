可以。既然你明确不要 NAT Linux，那这版我们就彻底只用 Cloudflare。

不过先把 V0.1 的定位说清楚：这一版是“Cloudflare Worker → Telegram DC 的 TCP 隧道验证版”，不是最终可以直接填进 Telegram 的 MTProto Proxy。它的目的只有一个：先证明

客户端
  ↓ WSS
Cloudflare Worker
  ↓ TCP
Telegram DC

这条链路稳定可用。

Cloudflare 目前的 Workers 已支持入站 WebSocket 和出站 TCP connect()；但仍不支持 Worker 的入站原生 TCP，所以 WSS → TCP 正是 V0.1 应该验证的架构。

另外，我核对了目前 GitHub 上 tg-ws-proxy-rs 的 Worker 实现，它采用的也是 /apiws → TCP 443 的思路。

⸻

1. V0.1 最终结构

我们先固定 Telegram DC2：

                     Cloudflare
┌──────────┐       ┌──────────────────┐
│ 测试客户端 │ WSS   │                  │
│           ├──────►│  Cloudflare      │
└──────────┘       │  Worker          │
                    │       │          │
                    │       │ TCP:443  │
                    └───────┼──────────┘
                            │
                            ▼
                     Telegram DC2

V0.1 不做：

* ❌ NAT Linux
* ❌ VPS
* ❌ VLESS
* ❌ Trojan
* ❌ SOCKS5
* ❌ D1
* ❌ KV
* ❌ Durable Objects
* ❌ MTProto Secret

先把最底层的 WSS → TCP 打通。

Telegram 官方定义的 WebSocket MTProto transport 本身就是一个长期双向字节流，WebSocket message 不能简单当成独立 MTProto packet；所以后面的 V0.2 才会把 Telegram MTProto transport 接进来。

⸻

2. 项目结构

新建：

cf-telegram-worker/
├── src/
│   └── index.js
├── wrangler.toml
├── package.json
└── .gitignore

⸻

3. src/index.js

这一版我不允许客户端随便指定任意 IP。

GitHub 上现有的 Worker 示例是：

/apiws?dst=<dc-ip>

然后 Worker 直接 connect(dst:443)。这虽然方便，但实际上会把 Worker 变成一个开放 TCP relay。

我们自己的版本先做成 Telegram DC IP 白名单，安全很多。

import { connect } from "cloudflare:sockets";
/**
 * V0.1
 *
 * Telegram DC2:
 *   WSS -> Cloudflare Worker -> TCP:443 -> Telegram DC2
 *
 * 目前只允许 Telegram DC IP。
 * 不允许用户把 Worker 当任意 TCP 代理使用。
 */
const DC_TARGETS = {
  "2": [
    "149.154.167.50",
    "149.154.167.41",
    "149.154.167.220",
  ],
};
const PATH = "/apiws";
function getTarget(dc) {
  const targets = DC_TARGETS[dc];
  if (!targets || targets.length === 0) {
    return null;
  }
  // V0.1 固定第一个，后面再做故障切换。
  return targets[0];
}
function isWebSocket(request) {
  return (
    (request.headers.get("Upgrade") || "").toLowerCase() === "websocket"
  );
}
function json(data, status = 200) {
  return new Response(JSON.stringify(data, null, 2), {
    status,
    headers: {
      "content-type": "application/json; charset=utf-8",
      "cache-control": "no-store",
    },
  });
}
export default {
  async fetch(request) {
    const url = new URL(request.url);
    /*
     * 健康检查
     */
    if (url.pathname === "/") {
      return json({
        name: "CF Telegram Worker",
        version: "0.1.0",
        status: "ok",
        mode: "telegram-dc2-tcp-tunnel",
        websocket: "/apiws?dc=2",
      });
    }
    /*
     * 只允许 WebSocket
     */
    if (url.pathname !== PATH) {
      return new Response("Not Found", {
        status: 404,
        headers: {
          "content-type": "text/plain; charset=utf-8",
        },
      });
    }
    if (!isWebSocket(request)) {
      return new Response("Expected WebSocket", {
        status: 426,
        headers: {
          "Upgrade": "websocket",
        },
      });
    }
    /*
     * V0.1 只支持 DC2
     */
    const dc = url.searchParams.get("dc") || "2";
    const target = getTarget(dc);
    if (!target) {
      return json(
        {
          error: "unsupported_dc",
          message: "V0.1 only supports Telegram DC2.",
        },
        400,
      );
    }
    /*
     * 创建 WebSocket
     */
    const pair = new WebSocketPair();
    const client = pair[0];
    const server = pair[1];
    server.accept({
      allowHalfOpen: true,
    });
    let socket;
    let tcpReader;
    let tcpWriter;
    try {
      /*
       * Cloudflare Worker -> Telegram DC
       */
      socket = connect({
        hostname: target,
        port: 443,
      });
      await socket.opened;
      tcpReader = socket.readable.getReader();
      tcpWriter = socket.writable.getWriter();
      /*
       * WebSocket -> TCP
       */
      server.addEventListener("message", async (event) => {
        try {
          let data;
          if (event.data instanceof ArrayBuffer) {
            data = new Uint8Array(event.data);
          } else if (event.data instanceof Blob) {
            data = new Uint8Array(await event.data.arrayBuffer());
          } else if (typeof event.data === "string") {
            /*
             * Telegram MTProto transport 应该使用 binary。
             * V0.1 不接受字符串数据。
             */
            server.close(1003, "Binary WebSocket frames required");
            return;
          } else {
            server.close(1003, "Unsupported WebSocket data");
            return;
          }
          if (data.byteLength > 0) {
            await tcpWriter.write(data);
          }
        } catch (error) {
          console.error("WebSocket -> TCP failed:", error);
          try {
            server.close(1011, "TCP write failed");
          } catch {}
        }
      });
      /*
       * WebSocket 关闭 -> TCP 关闭
       */
      server.addEventListener("close", async () => {
        try {
          await tcpWriter.close();
        } catch {}
        try {
          await socket.close();
        } catch {}
      });
      /*
       * TCP -> WebSocket
       */
      (async () => {
        try {
          while (true) {
            const { value, done } = await tcpReader.read();
            if (done) {
              break;
            }
            if (!value) {
              continue;
            }
            if (server.readyState === WebSocket.OPEN) {
              server.send(value);
            } else {
              break;
            }
          }
        } catch (error) {
          console.error("TCP -> WebSocket failed:", error);
        } finally {
          try {
            tcpReader.releaseLock();
          } catch {}
          try {
            await socket.close();
          } catch {}
          try {
            if (server.readyState !== WebSocket.CLOSED) {
              server.close(1000, "TCP connection closed");
            }
          } catch {}
        }
      })();
      /*
       * 101 Switching Protocols
       */
      return new Response(null, {
        status: 101,
        webSocket: client,
      });
    } catch (error) {
      console.error("TCP connection failed:", error);
      try {
        server.close(1011, "Telegram DC connection failed");
      } catch {}
      try {
        await socket?.close();
      } catch {}
      return new Response("Unable to connect to Telegram DC", {
        status: 502,
      });
    }
  },
};

这里使用的 Telegram DC2 地址是目前公开资料和现有 Telegram WS proxy 实现中使用的地址之一；Telegram 的实际 DC 配置可能发生变化，所以后续正式版不应该永久硬编码这一组 IP。现有项目的日志也显示 DC2 使用过 149.154.167.50、149.154.167.41 等地址。

⸻

4. wrangler.toml

name = "cf-telegram-worker"
main = "src/index.js"
compatibility_date = "2026-09-25"
workers_dev = true
[observability]
enabled = true

Cloudflare 当前推荐新项目使用 wrangler.jsonc，但 wrangler.toml 仍然支持；name、main 和 compatibility_date 是部署配置的核心字段。

这里暂时使用：

workers_dev = true

所以不用购买域名、不需要 DNS、不需要 Cloudflare Tunnel。

部署以后直接得到：

https://cf-telegram-worker.<你的Cloudflare用户名>.workers.dev

⸻

5. package.json

{
  "name": "cf-telegram-worker",
  "private": true,
  "version": "0.1.0",
  "description": "Cloudflare Worker TCP tunnel for Telegram DC",
  "scripts": {
    "dev": "wrangler dev",
    "deploy": "wrangler deploy",
    "tail": "wrangler tail"
  },
  "devDependencies": {
    "wrangler": "^4.40.0"
  }
}

这里不需要任何运行时 npm 依赖。

因为：

import { connect } from "cloudflare:sockets";

是 Cloudflare Workers Runtime API，不需要自己安装 socket 库。

⸻

6. .gitignore

node_modules/
.wrangler/
.dev.vars
.env
.DS_Store

⸻

7. 本地创建项目

如果你使用 Mac / Windows / Linux，终端执行：

mkdir cf-telegram-worker
cd cf-telegram-worker
mkdir src

然后：

npm init -y
npm install -D wrangler

接着创建：

src/index.js
wrangler.toml
package.json
.gitignore

把上面的内容分别放进去。

⸻

8. 登录 Cloudflare

执行：

npx wrangler login

浏览器会自动打开 Cloudflare 授权页面。

登录你准备部署 Worker 的 Cloudflare 账号。

成功后检查：

npx wrangler whoami

应该看到你的 Cloudflare 账号信息。

⸻

9. 本地测试

先不要部署。

运行：

npm run dev

Wrangler 会给你类似：

http://localhost:8787

打开：

http://localhost:8787/

应该得到：

{
  "name": "CF Telegram Worker",
  "version": "0.1.0",
  "status": "ok",
  "mode": "telegram-dc2-tcp-tunnel",
  "websocket": "/apiws?dc=2"
}

⸻

10. 部署到 Cloudflare

直接：

npm run deploy

或者：

npx wrangler deploy

Cloudflare 会输出类似：

Uploaded cf-telegram-worker
Published cf-telegram-worker
https://cf-telegram-worker.xxxxx.workers.dev

把这个地址记下来。

⸻

11. Cloudflare 后台怎么设置

如果使用上面的：

workers_dev = true

其实不需要配置 DNS。

进入 Cloudflare：

Workers & Pages → cf-telegram-worker

然后确认：

Settings
   ↓
Domains & Routes

应该能看到：

*.workers.dev

例如：

cf-telegram-worker.xxxxx.workers.dev

Cloudflare 官方的 Worker 创建流程也是从 Workers & Pages 创建 Worker，然后部署代码。现有 tg-ws-proxy-rs 的 Cloudflare Worker 文档同样采用 workers.dev，不要求自己购买域名。

⸻

12. 第一项测试：HTTP

浏览器访问：

https://cf-telegram-worker.xxxxx.workers.dev/

应该返回：

{
  "name": "CF Telegram Worker",
  "version": "0.1.0",
  "status": "ok",
  "mode": "telegram-dc2-tcp-tunnel",
  "websocket": "/apiws?dc=2"
}

这证明：

你
 ↓
Cloudflare Edge
 ↓
Worker

正常。

⸻

13. 第二项测试：WebSocket

真正重要的是：

wss://cf-telegram-worker.xxxxx.workers.dev/apiws?dc=2

注意这里是：

wss://

不是：

https://

因为我们要求：

Upgrade: websocket

Cloudflare Workers 原生支持 WebSocket，并且可以使用 WebSocketPair 接受连接。

⸻

14. 怎么测试 TCP 是否真的连接 Telegram？

这里不要仅仅测试：

HTTP 101

因为：

WebSocket 101

只证明 Worker 接受了 WebSocket。

不一定证明 Worker → Telegram DC 成功。

这一点 GitHub 上现有的 tg-ws-proxy-rs 文档也特别强调了：单纯 WebSocket upgrade 成功并不能证明 Worker 能访问 Telegram；它的 --check 会进一步通过 tunnel 发送真实 MTProto init 来验证。

所以我们 V0.1 最好用一个简单的 WebSocket 测试客户端。

⸻

15. 推荐你用 Node 测试

安装：

npm install ws

新建：

test-ws.js

内容：

const WebSocket = require("ws");
const WORKER =
  "wss://cf-telegram-worker.xxxxx.workers.dev/apiws?dc=2";
console.log("Connecting:", WORKER);
const ws = new WebSocket(WORKER);
ws.binaryType = "arraybuffer";
ws.on("open", () => {
  console.log("WebSocket connected");
  /*
   * 这里只发送测试二进制。
   *
   * 注意：
   * 这不是合法 MTProto 数据。
   * 只是为了确认：
   *
   * Client
   *   ↓
   * Worker
   *   ↓
   * Telegram TCP
   *
   * 这条双向链路确实建立了。
   */
  const test = new Uint8Array([
    0xef,
  ]);
  ws.send(test);
  console.log("Test bytes sent");
});
ws.on("message", (data) => {
  console.log(
    "Received from Telegram:",
    data.length,
    "bytes",
  );
  console.log(
    Buffer.from(data).toString("hex").slice(0, 128),
  );
});
ws.on("close", (code, reason) => {
  console.log(
    "Closed:",
    code,
    reason.toString(),
  );
});
ws.on("error", (error) => {
  console.error("WebSocket error:", error);
});

运行：

node test-ws.js

⸻

16. 不过这里有一个重要现象

你可能看到：

WebSocket connected
Test bytes sent
Closed

这不一定代表 Worker 有问题。

因为：

0xef

只是我们故意发送的测试字节。

它不是一个完整合法的 MTProto 会话。

Telegram 官方的 MTProto transport 有自己的 framing，例如 Abridged transport 首字节可以是 0xef，但后续必须按照 MTProto envelope 继续传输；WebSocket transport 还要求 transport obfuscation。

所以 V0.1 的真正目标不是：

Telegram 接受 0xef

而是：

Cloudflare Worker 成功建立到 Telegram DC 的 TCP socket，并能够进行双向字节 relay。

⸻

17. Cloudflare 日志

部署后执行：

npm run tail

或者：

npx wrangler tail

然后重新运行测试。

你应该能够看到：

WebSocket request
TCP connection

如果出现：

proxy request failed

或者：

TCP Loop detected

就说明 Worker 的 connect() 被 Cloudflare 拒绝。

Cloudflare 官方文档也明确列出了这些 TCP socket 错误，包括禁止连接 Cloudflare IP、localhost/private IP，以及 Worker 自己回连自己等情况。

⸻

18. 一个非常重要的 Cloudflare 设置

进入：

Workers & Pages → cf-telegram-worker → Settings → Observability

打开：

Workers Logs

V0.1 调试期间建议开启。

我们代码里面已经有：

console.error(...)

所以发生：

TCP connect failed
TCP write failed
TCP read failed

时，可以直接从 Cloudflare Logs 看。

⸻

19. 暂时不要绑定自己的域名

你可能有自己的：

xxx.eu.org

之类域名。

V0.1 我反而建议先不要绑定。

先用：

xxxx.workers.dev

因为这样少了：

DNS
↓
Worker Route
↓
Custom Domain
↓
SSL

这些变量。

先证明：

workers.dev
 ↓
Worker
 ↓
Telegram

成功。

然后 V0.2 再改：

tg.xxx.eu.org

⸻

20. 目前我们和最终目标的距离

现在：

                 V0.1
测试客户端
    │
    │ WSS
    ▼
Cloudflare Worker
    │
    │ TCP :443
    ▼
Telegram DC2

下一步：

                 V0.2
Telegram
    │
    │ MTProto WebSocket
    ▼
Cloudflare Worker
    │
    │ TCP
    ▼
Telegram DC

最后：

                 V1.0
Telegram iPhone
      │
      │ tg://proxy
      │
      ▼
Cloudflare Worker
      │
      ├── DC1
      ├── DC2
      ├── DC3
      ├── DC4
      └── DC5

⸻

21. 为什么 V0.1 不直接写成 MTProto Proxy？

因为这一步非常容易走错。

Telegram 官方明确规定：

WebSocket
   ↓
transport obfuscation
   ↓
MTProto transport
   ↓
encrypted payload

其中 WebSocket 要求：

Sec-WebSocket-Protocol: binary

而且 WebSocket message 本身不能被当成独立 MTProto packet；它应该被视为 TCP 一样的连续 byte stream。

再往下还有：

64-byte initialization
        ↓
AES-256-CTR
        ↓
MTProto transport
        ↓
DC

Telegram 官方文档也明确描述了 WebSocket transport 必须启用 transport obfuscation，以及 64-byte initialization、key/IV 派生和 AES-256-CTR。

所以我们现在先把：

WebSocket
   ↓
TCP

这一层彻底验证。

⸻

22. 我还建议把 V0.1 的 Worker 稍微再升级一个点

现在：

const DC_TARGETS = {
  "2": [
    "149.154.167.50",
    "149.154.167.41",
    "149.154.167.220"
  ]
};

只使用第一个：

149.154.167.50

下一小版本可以变成：

DC2
 ├─ 149.154.167.50
 ├─ 149.154.167.41
 └─ 149.154.167.220
       ↓
   自动 fallback

例如：

第一 IP 失败
     ↓
第二 IP
     ↓
第三 IP

然后：

DC1
DC2
DC3
DC4
DC5

全部纳入。

现有 tg-ws-proxy-rs 的代码/日志也显示一个 DC 会有多个 Telegram endpoint，因此正式版不能只绑定一个 IP。

⸻

你现在实际需要做的只有 5 步

① 创建目录

mkdir cf-telegram-worker
cd cf-telegram-worker
mkdir src

② 放入上面的 index.js + wrangler.toml

③ 安装

npm install

④ 登录

npx wrangler login

⑤ 部署

npx wrangler deploy

然后把 Cloudflare 给你的：

https://xxxxx.workers.dev

发给我。

先不要把任何 Cloudflare API Token、密码、Secret 发给我。

我拿到你的 Worker 域名后，下一步就可以直接给你做 V0.2：把 Telegram 官方 MTProto WebSocket transport 接进这个 Worker，并开始测试 iPhone Telegram 是否能够直接通过这个 Cloudflare Worker 建立 MTProto 连接。