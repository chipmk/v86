# Connect browser JavaScript to a v86 guest with tcpip.js

Use [tcpip.js](https://github.com/chipmk/tcpip.js) when you need to communicate
with a service inside the guest.
In this example, JavaScript makes an HTTP request to the guest.
The request and response stay in the same browser page. No relay server or
internet connection is needed for that exchange.

![JavaScript calls tcpip.js, which shares an in-browser virtual network with the v86 guest.](tcpip-js-network.svg)

## Attach the virtual network

Start with a working [v86 embed](../examples/basic.html). To follow along,
download [Buildroot Linux 6.8](https://i.copy.sh/buildroot-bzimage68.bin).
Remove the default `cdrom` option and set these options:

```js
bzimage: { url: "../images/buildroot-bzimage68.bin" },
memory_size: 64 * 1024 * 1024,
cmdline: "tsc=reliable mitigations=off random.trust_cpu=on",
```

Serve the image from the URL in `bzimage`. In a project that uses a JavaScript
bundler, install the networking packages:

```sh
npm install tcpip@0.4.1 @tcpip/v86@0.2.4 @tcpip/http@0.2.2 @tcpip/transport@0.1.0
```

If you use Vite, add `tcpip` and `@tcpip/http` to
`optimizeDeps.exclude` so their WebAssembly files can load during development.

Keep `autostart: true` from the basic embed and remove any relay URL. Add this
to the module that creates the emulator:

```js
import { createStack } from "tcpip";
import { createV86NetworkStream } from "@tcpip/v86";
import { createHttp } from "@tcpip/http";
import { connectStreams } from "@tcpip/transport";

const stack = await createStack();
const tap = await stack.interfaces.createTap({ ip: "192.168.1.1/24" });
const nic = createV86NetworkStream(emulator);

connectStreams(tap, nic);

const { fetch: virtualFetch } = await createHttp(stack.tcp);
```

`connectStreams()` uses standard `ReadableStream` and `WritableStream` objects
to forward Ethernet frames in both directions.

## Start an HTTP service in the guest

In your app, you might run Nginx or a Python web app in the guest and start it
at boot. This example uses `inetd` from the stock Buildroot image to return a
sample HTTP response.

After Linux boots, paste these commands into the guest shell as root. Use
`ip link` to check the interface name and replace `eth0` if yours differs.

```sh
ip link set eth0 up
ip addr add 192.168.1.2/24 dev eth0

cat >/tmp/demo-http <<'SERVER'
#!/bin/sh
cr=$(printf '\r')
while IFS= read -r line; do
    [ "$line" = "$cr" ] && break
done
printf 'HTTP/1.0 200 OK\r\nContent-Length: 15\r\n\r\nHello from v86\n'
SERVER
chmod +x /tmp/demo-http
echo '8080 stream tcp nowait nobody /tmp/demo-http demo-http' | inetd -f - &
```

The virtual link uses a static address, so it needs no DHCP server, DNS
resolver, or default gateway. To send guest commands from JavaScript, see the
[v86 serial example](../examples/serial.html).

## Read the response from JavaScript

In the guest, run `netstat -ln` to confirm port `8080` is listening. Then run
this from your application:

```js
const response = await virtualFetch("http://192.168.1.2:8080/");
if (!response.ok) {
    throw new Error(`Guest returned HTTP ${response.status}`);
}
console.log(await response.text()); // Hello from v86
```

If the request fails, check the guest interface name, its IP address, and the
address and port the server listens on.

`virtualFetch` follows the [Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Window/fetch)
signature. It accepts `(input, init)` and returns a promise for a standard
`Response`. It sends plain HTTP over the virtual network.

If your library or SDK accepts a custom `fetch` function, pass it `virtualFetch`
to reach services on that network. For example, Apollo Client is a GraphQL
client that accepts a custom `fetch` through its `HttpLink`.

```js
import { HttpLink } from "@apollo/client/link/http";

const link = new HttpLink({
    uri: "http://192.168.1.2:8080/graphql",
    fetch: virtualFetch,
});
```

For other TCP protocols, call
[`stack.tcp.connect()`](https://github.com/chipmk/tcpip.js#stacktcpconnect)
with the guest address and port. For multiple guests, see
[tcpip.js bridge interfaces](https://github.com/chipmk/tcpip.js#bridge-interface).
