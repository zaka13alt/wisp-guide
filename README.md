

---

### Step 1: download and configs

download it
to get a WireGuard config you can get one from your VPN provider or from using Cloudflare WARP and this generator for free
https://lanrat.github.io/wireguard-warp-generator/



**run this command:**
```bash
curl -L "https://github.com/windtf/wireproxy/releases/download/v1.1.3/wireproxy_linux_amd64.tar.gz" | tar -xz && sudo mv wireproxy /usr/local/bin/
```

Run `wireproxy --version` to make sure it installed
---

### step 2



Create your config file(s)

nano wireproxy.conf

fill out this replace key and IP with stuff from your WireGuard config

this will be called wireproxy.conf
```
WGConfig = ./wg0.conf

[Socks5]
BindAddress = 127.0.0.1:35344

[http]
BindAddress = 127.0.0.1:35345
```
now paste your WireGuard config you got from step 1 in wg0.conf using nano wg0.conf

### step 3

to test

```bash
sudo wireproxy -c wireproxy.conf > wireproxy.log 2>&1 &
```

now to test the proxy

**To test the SOCKS5 proxy:**
```bash
curl --socks5-hostname 127.0.0.1:35344 https://ifconfig.me
```

**To test the HTTP proxy:**
```bash
curl -x http://127.0.0.1:35345 https://ifconfig.me
```

The command will instantly output your WireGuard IP 

---

### step 4

DO THIS IF YOU WANT IT TO STAY

1. **Create the system service file:**
   ```bash
   sudo nano /etc/systemd/system/wireproxy.service
   ```

2. **Paste this entire block inside:**
   ```
   [Unit]
   Description=wireproxy
   After=network.target

   [Service]
   Type=simple
   WorkingDirectory=/home/ubuntu
   ExecStart=/usr/local/bin/wireproxy -c /home/ubuntu/wireproxy.conf
   Restart=always
   RestartSec=5
   StandardOutput=append:/home/ubuntu/wireproxy.log
   StandardError=append:/home/ubuntu/wireproxy.log

   [Install]
   WantedBy=multi-user.target
   ```
   Note: If your file is located somewhere other than `/home/ubuntu`, make sure to swap out the file paths above.

3. **final steps:**
   ```bash
   # kill all stray wireproxys
   sudo pkill -9 -f wireproxy

   # start the new process
   sudo systemctl daemon-reload

   # auto start it
   sudo systemctl enable wireproxy

   # run right now
   sudo systemctl start wireproxy
   ```
now for the Wisp server

so if you are running wisp-js or wisp-server-node this will not work, i recommend using wispurr and swapping your package.json to something like this for example (add any more packages and stuff as you need)
```
{
  "name": "wisp-server",
  "version": "1.0.0",
  "description": "wisp server",
  "main": "index.js",
  "type": "module",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "wispurr": "^4.2.0"
  }
}
```
and a index.js like this 
```

import { wispurr } from "wispurr";
import { createServer } from "node:http";
import { readFileSync } from "node:fs";

const userConfig = JSON.parse(
  readFileSync(new URL("./config.json", import.meta.url), "utf8")
);

const wisp = new wispurr(userConfig);
console.log(wisp.config); // sanity check: your values should show up here
await wisp.start(4);

const server = createServer();
server.on("upgrade", (req, socket, head) => wisp.route(req, socket, head));
server.listen(3000);
```
and the final file the config.json (this is my recommend config, change as you wish)
```
{
	"port": 6001,
	"allowTCP": true,
	"allowUDP": true,
	"allowDirectIP": true,
	"allowPrivateIPs": false,
	"allowLoopbackIPs": false,
	"tcpBufferSize": 1048576,
	"socketBufferSize": 33554432,
	"pendingQueueSize": 268435456,
	"bufferRemainingLength": 32768,
	"tcpNoDelay": true,
	"blacklist": { "hostnames": {}, "ports": {} },
	"whitelist": { "hostnames": {}, "ports": {} },
	"websocketPermessageDeflate": false,
	"dnsServers": [],
	"dnsMethod": "resolve",
	"dnsResultOrder": "ipv4first",
	"enableTwisp": false,
	"enableV2": true,
	"handshakeTimeoutSeconds": 10,
	"motd": "",
	"passwordAuth": false,
	"passwordAuthRequired": false,
	"passwordUsers": {},
	"parseRealIP": true,
	"trustedProxies": ["127.0.0.1"],
	"trustedHeaders": ["CF-Connecting-IP", "X-Forwarded-For"],
	"nonWSResponse": "404 not found",
	"logLevel": "error",
	"proxy": "socks5://127.0.0.1:35344",
	"maxMessageSize": 1048581,
	"staticDir": "",
	"bandwidthLimitKbps": 0,
	"connectionsLimitPerIP": 0,
	"connectionWindowSeconds": 0,
	"floodProtection": {
		"enabled": false,
		"maxConnectsPerSourceIPPerSecond": 0,
		"maxConnectsPerDestPerSecond": 0,
		"maxConnectsPerDestPerMinute": 0,
		"maxInFlightSyns": 0,
		"maxConcurrentStreamsPerConnection": 0,
		"maxConcurrentConnections": 0,
		"synFloodSignature": {
			"enabled": false,
			"windowMs": 2000,
			"minSamples": 32,
			"failedHandshakeRatio": 0.75
		},
		"wsCloseAfterViolations": 0,
		"logBlockedDials": false
	},
	"reputation": {
		"enabled": false,
		"storePath": "./data/wispurr-reputation.json",
		"saveIntervalSeconds": 30,
		"scoreDecayPerHour": 1,
		"evictAfterDays": 7,
		"thresholds": { "warn": 21, "throttle": 51, "strict": 81 },
		"weights": {},
		"destinationWeights": {}
	}
}
```


