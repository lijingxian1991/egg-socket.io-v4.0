# egg-socket.io

[![NPM version][npm-image]][npm-url]
[![build status][travis-image]][travis-url]
[![Test coverage][codecov-image]][codecov-url]
[![David deps][david-image]][david-url]
[![Known Vulnerabilities][snyk-image]][snyk-url]
[![npm download][download-image]][download-url]

[npm-image]: https://img.shields.io/npm/v/egg-socket.io.svg?style=flat-square
[npm-url]: https://npmjs.org/package/egg-socket.io
[travis-image]: https://img.shields.io/travis/eggjs/egg-socket.io.svg?style=flat-square
[travis-url]: https://travis-ci.org/eggjs/egg-socket.io
[codecov-image]: https://img.shields.io/codecov/c/github/eggjs/egg-socket.io.svg?style=flat-square
[codecov-url]: https://codecov.io/github/eggjs/egg-socket.io?branch=master
[david-image]: https://img.shields.io/david/eggjs/egg-socket.io.svg?style=flat-square
[david-url]: https://david-dm.org/eggjs/egg-socket.io
[snyk-image]: https://snyk.io/test/npm/egg-socket.io/badge.svg?style=flat-square
[snyk-url]: https://snyk.io/test/npm/egg-socket.io
[download-image]: https://img.shields.io/npm/dm/egg-socket.io.svg?style=flat-square
[download-url]: https://npmjs.org/package/egg-socket.io

egg plugin for socket.io

## Install

```bash
$ npm i egg-socket.io-v4.0 --save
```

## Requirements

- Node.js >= 8.0
- Egg.js >= 2.0

## Configuration

Change `${app_root}/config/plugin.js` to enable Socket.IO plugin:

```js
// {app_root}/config/plugin.js
exports.io = {
  enable: true,
  package: "egg-socket.io",
};
```

Configure Socket.IO in `${app_root}/config/config.default.js`:

```js
exports.io = {
  // init: { },  support socket.io v4 breaking change! not init config
  options: {
    // support all socket.io v4 options. ref: https://socket.io/docs/v4/server-options/
  },
  namespace: {
    "/": {
      connectionMiddleware: [],
      packetMiddleware: [],
    },
  },
  redis: {
    host: "127.0.0.1",
    port: 6379,
  },
};
```

### generateId

**Note:** This function is left on purpose to override and generate a unique ID according to your own rule:

```js
exports.io = {
  generateId: (request) => {
    // Something like UUID.
    return "This should be a random unique ID";
  },
};
```

## Deployment

### Node Conf

Because of socket.io's design, the multi process socket.io server must work at `sticky` mode.

So, you must start cluster server with `sticky` set to true, otherwise it will cause handshake exception.

```bash
$ # modify your package.json - npm scripts
$ egg-bin dev --sticky
$ egg-scripts start --sticky
```

which will start egg cluster with:

```js
startCluster({
  sticky: true,
  ...
});
```

### Nginx Conf

if you use a nginx proxy server:

```
location / {
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $host;
    proxy_pass   http://127.0.0.1:{ your node server port };
}
```

## Usage

### Directory Structure

```
app
├── io
│   ├── controller
│   │   └── chat.js
│   └── middleware
│       ├── auth.js
│       ├── filter.js
├── router.js
config
 ├── config.default.js
 └── plugin.js
```

### Middleware

middleware are functions which every connection or packet will be processed by.

#### Connection Middleware

- Write your connection middleware
  `app/io/middleware/auth.js`

```js
module.exports = (app) => {
  return async (ctx, next) => {
    ctx.socket.emit("res", "connected!");
    await next();
    // execute when disconnect.
    console.log("disconnection!");
  };
};
```

- then config this middleware to make it works.

`config/config.default.js`

```js
exports.io = {
  namespace: {
    "/": {
      connectionMiddleware: ["auth"],
    },
  },
};
```

pay attention to the namespace, the config will only work for a specific namespace.

#### Packet Middleware

- Write your packet middleware
  `app/io/middleware/filter.js`

```js
module.exports = (app) => {
  return async (ctx, next) => {
    ctx.socket.emit("res", "packet received!");
    console.log("packet:", this.packet);
    await next();
  };
};
```

- then config this middleware to make it works.

`config/config.default.js`

```js
exports.io = {
  namespace: {
    "/": {
      packetMiddleware: ["filter"],
    },
  },
};
```

pay attention to the namespace, the config will only work for a specific namespace.

### Controller

controller is designed to handle the `emit` event from the client.

example:

`app/io/controller/chat.js`

```js
module.exports = (app) => {
  class Controller extends app.Controller {
    async ping() {
      const message = this.ctx.args[0];
      await this.ctx.socket.emit(
        "res",
        `Hi! I've got your message: ${message}`
      );
    }
  }
  return Controller;
};

// or async functions
exports.ping = async function () {
  const message = this.args[0];
  await this.socket.emit("res", `Hi! I've got your message: ${message}`);
};
```

next, config the router at `app/router.js`

```js
module.exports = (app) => {
  // or app.io.of('/')
  app.io.route("chat", app.io.controller.chat.ping);
};
```

### Router

A router is mainly responsible for distributing different events corresponding to a controller on a specific socket connection.

It should be configured at `app/router.js` refer to the last chapter.

Besides that, there are several system Event:

- `disconnecting` doing the disconnect
- `disconnect` connection has disconnected
- `error` Error occured

Example：

`app/router.js`

```js
app.io.route("disconnect", app.io.controller.chat.disconnect);
```

`app/io/controller/chat.js`

```js
module.exports = (app) => {
  class Controller extends app.Controller {
    async disconnect() {
      const message = this.ctx.args[0];
      console.log(message);
    }
  }
  return Controller;
};
```

### Session

The session is supported by `egg-socket.io`. It's behaviour just like the normal HTTP session.

Session creates or check just happens at the handshake period. Session can be accessed by `ctx.session` in packetMiddleware and controller, but it's only created at the handshake period.

The feature is powered by [egg-session](https://github.com/eggjs/egg-session), make sure it has been enabled.

## Cluster

If your Socket.IO service is powered by mutil server, you must think about cluster solution. It can't work without cluster like broadcast, rooms and so on.

It's very easy to implement sharing source and event dispatch with [socket.io-redis](https://github.com/socketio/socket.io-redis) built in.

config at `config/config.${env}.js` ：

```js
exports.io = {
  redis: {
    host: { redis server host },
    port: { redis server prot },
    auth_pass: { redis server password },
    db: 0,
  }
};
```

```js
// Vue side example code
import { io } from "socket.io-client";

const socket = io("ws://127.0.0.1:7001", { transports: ["websocket"] });
socket.on("connect", function () {
  socket.emit("chat", "hello world!");
});
socket.on("res", (msg) => {
  console.log("res from server: %s!", msg);
});
```

```js
// Example code for mini program end
import io from "vendor/wxapp-socket-io.js";

const socket = io("ws://127.0.0.1:7001");
socket.on("connect", function () {
  socket.emit("chat", "hello world!");
});
socket.on("res", (msg) => {
  console.log("res from server: %s!", msg);
});
```

Application will try to connect the redis server when booting.

## Questions & Suggestions

Please open an issue [here](https://github.com/lijingxian1991/egg-socket.io-v4.0/issues).

## License

[MIT](LICENSE)
