# Connect browser JavaScript to a v86 guest with tcpip.js

Use tcpip.js when you need to communicate with a service inside the guest.
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
autostart: false,
```

Serve the image from the URL in `bzimage`. In a project that uses a JavaScript
bundler, install the networking packages:

```sh
npm install tcpip@0.4.1 @tcpip/v86@0.2.4 @tcpip/http@0.2.2 @tcpip/transport@0.1.0
```

If you use Vite, add `tcpip` and `@tcpip/http` to
`optimizeDeps.exclude` so their WebAssembly files can load during development.

Remove any relay URL from your v86 configuration. Add this to the module
that creates the emulator, registering `emulator-ready` immediately after
creating it:

```js
import { createStack } from "tcpip";
import { createV86NetworkStream } from "@tcpip/v86";
import { createHttp } from "@tcpip/http";
import { connectStreams } from "@tcpip/transport";

const ready = new Promise(resolve => {
    emulator.add_listener("emulator-ready", resolve);
});
const stack = await createStack();
const tap = await stack.interfaces.createTap({ ip: "192.168.1.1/24" });
const nic = createV86NetworkStream(emulator);

connectStreams(tap, nic);

const { fetch: virtualFetch } = await createHttp(stack.tcp);
await ready;
await emulator.run();
```

`connectStreams()` forwards Ethernet frames between the stack and the guest.

## Start an HTTP service in the guest

If you build the guest image for your app, you can configure the network and
start your service at boot. To try it by hand, use Buildroot’s built-in
`inetd` to serve a short HTTP response.

Run these commands as root in the guest. Use `ip link` to check the interface
name and replace `eth0` if yours differs.

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
resolver, or default gateway.

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

`virtualFetch` sends plain HTTP to services reachable on the virtual network.
You can also pass it to an SDK that accepts a custom `fetch` implementation.
If the request fails, check the guest interface name, its IP address, and the
address and port the server listens on.

For other TCP protocols, call
[`stack.tcp.connect()`](https://github.com/chipmk/tcpip.js#stacktcpconnect)
with the guest address and port. For multiple guests, see
[tcpip.js bridge interfaces](https://github.com/chipmk/tcpip.js#bridge-interface).
