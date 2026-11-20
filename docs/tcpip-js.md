# Connect browser JavaScript to a v86 guest with tcpip.js

Use tcpip.js when an application that embeds v86 needs to call a service inside
its guest. JavaScript and the guest share a virtual Ethernet link in the same
page. This setup needs no relay server and provides no guest Internet access.

This example requests a file served by a Linux guest. The browser-side stack
uses `192.168.77.1` and the guest uses `192.168.77.2` on the same subnet.

## Attach the virtual network

Start with a working [v86 embed](../examples/basic.html) and a Linux guest
with a driver for its emulated NIC. In a project that uses a JavaScript
bundler, install the networking packages:

```sh
npm install tcpip@0.4.1 @tcpip/v86@0.2.4 @tcpip/http@0.2.2
```

If you use Vite, add `tcpip` and `@tcpip/http` to
`optimizeDeps.exclude` so their WebAssembly files can load during development.

Remove any relay URL from your v86 configuration. Set `autostart: false` so
you can attach the virtual network before starting the guest. Keep the NIC type
that your guest supports. Add this to the module that creates the emulator:

```js
import { createStack } from "tcpip";
import { createV86NetworkStream } from "@tcpip/v86";
import { createHttp } from "@tcpip/http";

const stack = await createStack();
const tap = await stack.interfaces.createTap({ ip: "192.168.77.1/24" });
const nic = createV86NetworkStream(emulator);

tap.readable.pipeTo(nic.writable).catch(console.error);
nic.readable.pipeTo(tap.writable).catch(console.error);

const { fetch: guestFetch } = await createHttp(stack.tcp);
await emulator.run();
```

The two pipes carry Ethernet frames in each direction. Start both pipes
without awaiting them. They stay active while the streams are connected.

## Start an HTTP service in the guest

In the guest shell, configure the interface as root. Use `ip link` to check
the name and replace `eth0` if yours differs.

```sh
ip link set eth0 up
ip addr add 192.168.77.2/24 dev eth0

mkdir -p /tmp/v86-http-demo
printf 'Hello from v86\n' > /tmp/v86-http-demo/index.html
cd /tmp/v86-http-demo
python3 -m http.server 8080 --bind 192.168.77.2
```

Use a Linux image with Python 3 for these commands. The virtual link uses a
static address, so it needs no DHCP server, DNS resolver, or default gateway.

## Read the response from JavaScript

Once the guest prints that the HTTP server is listening, run this from your
application, for example in a button handler:

```js
const response = await guestFetch("http://192.168.77.2:8080/");
if (!response.ok) {
    throw new Error(`Guest returned HTTP ${response.status}`);
}
console.log(await response.text()); // Hello from v86
```

`guestFetch` sends plain HTTP over the virtual stack. Use this function for
guest URLs in your application or pass it to an SDK that accepts a custom
`fetch` implementation. If the request fails, check the guest interface name,
its IP address, and the address and port the server listens on.

For other TCP protocols, call
[`stack.tcp.connect()`](https://github.com/chipmk/tcpip.js#stacktcpconnect)
with the guest address and port. For multiple guests, see
[tcpip.js bridge interfaces](https://github.com/chipmk/tcpip.js#bridge-interface).
