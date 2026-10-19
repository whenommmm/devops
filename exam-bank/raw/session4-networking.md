# Labs · Session 4 · Networking (answers use macOS commands)

Acronyms used: IP = Internet Protocol · DNS = Domain Name System (turns names into IP addresses) · TCP = Transmission Control Protocol (reliable, ordered) · UDP = User Datagram Protocol (fast, no delivery guarantee) · ICMP = Internet Control Message Protocol (what `ping` uses) · HTTP/HTTPS = HyperText Transfer Protocol (Secure = encrypted with TLS, Transport Layer Security) · CIDR = Classless Inter-Domain Routing (the `/24` notation) · TTL = Time To Live · PID = Process ID.

---

## Lab 1 · Show the IP address, subnet, default route and listening ports

**Answer (macOS):**
```bash
ipconfig getifaddr en0
ifconfig en0
ipconfig getoption en0 subnet_mask
route -n get default
netstat -rn -f inet
lsof -iTCP -sTCP:LISTEN -P -n
```
**What each line does:**
- `ipconfig getifaddr en0`: prints just your IP address on `en0` (on a MacBook, `en0` = Wi-Fi).
- `ifconfig en0`: full details of that network card: IP address, netmask, broadcast address, `status: active`.
- `ipconfig getoption en0 subnet_mask`: prints the subnet mask in dotted form.
- `route -n get default`: shows the **default gateway** (the router) and which card it uses; `-n` = show numbers, don't look up names.
- `netstat -rn -f inet`: the whole IPv4 routing table (`-r` = routes, `-n` = numbers, `-f inet` = IPv4 only); the `default` row is the gateway.
- `lsof -iTCP -sTCP:LISTEN -P -n`: every program **listening** for TCP connections, with its port. `lsof` = "list open files"; `-iTCP` = TCP network connections, `-sTCP:LISTEN` = only listening ones, `-P` = port numbers not names, `-n` = IPs not names.

**How to read the output (your "interpretation"), using your own Mac's output:**
```text
$ ipconfig getifaddr en0
100.128.161.247
$ ifconfig en0
	inet 100.128.161.247 netmask 0xfffff000 broadcast 100.128.175.255
	status: active
$ ipconfig getoption en0 subnet_mask
255.255.240.0
```
- **IP address** = 100.128.161.247. `status: active` = the Wi-Fi card is connected.
- **Subnet**: netmask `0xfffff000` = 255.255.240.0 = **/20** (20 network bits). So the network is **100.128.160.0/20**: 2^12 − 2 = **4,094 usable host addresses**. `broadcast 100.128.175.255` is the last address of that range (it sends to every host in the subnet).

```text
$ route -n get default
    gateway: 100.128.160.1
  interface: en0
$ netstat -rn -f inet
Destination        Gateway            Flags     Netif
default            100.128.160.1      UGScg     en0
```
- **Default gateway** = 100.128.160.1 (the router): anything not inside our own subnet is sent there. `UG` flags = the route is Up and goes through a Gateway.

```text
$ lsof -iTCP -sTCP:LISTEN -P -n
COMMAND     PID  USER  ... NAME
ControlCe   608  vansh ... TCP *:7000 (LISTEN)
ControlCe   608  vansh ... TCP *:5000 (LISTEN)
```
- `*:5000` / `*:7000` = ControlCenter (macOS's AirPlay Receiver) is listening on ports 5000 and 7000 on **all** interfaces (`*` = 0.0.0.0, reachable from other machines). A program shown as `127.0.0.1:PORT` would be **local-only**.

**Verify:** you see an `inet x.x.x.x netmask …` line, exactly one `default` route with a gateway, and `(LISTEN)` rows in `lsof`.

*(Linux version, if the exam asks for Linux: `ip a` (IP + `/prefix`) · `ip route` (`default via <gateway>`) · `sudo ss -tulpn` (listening TCP/UDP ports + process).)*

**🧠 In simple words:** `ipconfig`/`ifconfig` tell you "who am I on the network" (your IP and how big your subnet is), `route -n get default` tells you "where do I send traffic that isn't local" (the gateway), and `lsof … LISTEN` tells you "which programs are waiting for connections, on which ports". In short: **ifconfig = my address + subnet, route get default = my gateway, lsof LISTEN = my open ports.**

---

## Lab 2 · Troubleshoot a web service with ping, a port check and curl (reachability vs closed port vs HTTP error)

**Answer (macOS):**
```bash
python3 -m http.server 9090 &
ping -c 4 127.0.0.1
lsof -nP -iTCP:9090 -sTCP:LISTEN
nc -zv 127.0.0.1 9090
curl -v http://localhost:9090/
curl -I http://localhost:9090/
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:9090/missing.html
```
**Simulate the three failures:**
```bash
ping -c 4 -t 5 192.0.2.1
kill %1
nc -zv 127.0.0.1 9090
curl http://localhost:9090/
```
(and the HTTP error is the `missing.html` line above: `404`). Use the server machine's IP instead of `127.0.0.1`/`localhost` when testing another computer.

**What each line does:**
- `python3 -m http.server 9090 &`: starts a tiny web server on port 9090 in the background (`&`), so there is something to test. (Not 8080: on your Mac that port is taken by your other cluster.)
- `ping -c 4 <ip>`: sends 4 ICMP (Internet Control Message Protocol) echo requests (`-c` = count). Read `4 packets received, 0.0% packet loss`.
- `lsof -nP -iTCP:9090 -sTCP:LISTEN`: is any program listening on port 9090? It shows `Python … TCP *:9090 (LISTEN)`. (The Mac's version of `ss -tulpn | grep :9090`.)
- `nc -zv 127.0.0.1 9090`: `nc` (netcat) tries to open a TCP connection only (`-z` = don't send data, `-v` = say what happened): `Connection to 127.0.0.1 port 9090 … succeeded!`.
- `curl -v URL`: verbose: shows `Connected to … port 9090` (TCP worked), the request, and the response status.
- `curl -I URL`: only the response headers; the first line is the status (`HTTP/1.0 200 OK`).
- `curl -s -o /dev/null -w "%{http_code}\n" URL`: prints only the status number (here `404`).
- `ping -c 4 -t 5 192.0.2.1`: pings an address that doesn't exist (`-t 5` = give up after 5 seconds) → `100.0% packet loss`.
- `kill %1`: stops the background server (`%1` = background job number 1), so the port is now closed.

**Decision table (the key part of the answer):**

| What you see | Diagnosis | Layer |
|---|---|---|
| `ping`: `100.0% packet loss`; `curl`: `timed out` / `No route to host` | **Reachability problem** (host down, wrong IP, bad route, firewall dropping packets) | Network (IP) |
| `ping` OK, but `nc`: `Connection refused`, `curl: (7) Failed to connect … Couldn't connect to server`, and `lsof` shows **nothing** on the port | **Closed port** (service stopped/crashed, wrong port, or listening only on 127.0.0.1) | Transport (TCP port) |
| `curl` connects but the status is **4xx** (404 Not Found, 403 Forbidden) or **5xx** (500, 503) | **HTTP-level problem** (wrong path, permissions, app bug) | Application (HTTP) |
| `curl -I` → `HTTP/1.0 200 OK` | Healthy | none |

**Verify (tested on your Mac):** healthy → `0.0% packet loss`, `Python … TCP *:9090 (LISTEN)`, `nc … succeeded!`, `HTTP/1.0 200 OK`; missing page → `404`; after `kill %1` → `Connection refused` and `curl: (7) … Couldn't connect to server`; bad IP → `100.0% packet loss`.

*(Linux version, if the exam asks for Linux: same `ping` / `curl`, but `sudo ss -tulpn | grep :9090` instead of `lsof`.)*

**🧠 In simple words:** you test from the bottom up: first "can I reach the machine?" (ping), then "is a program actually listening on that port?" (lsof / nc), then "does the program answer correctly?" (curl's status code). The first step that fails tells you which layer is broken. In short: **ping fails = network, connection refused = closed port, 4xx/5xx = the web app itself.**

---

## Lab 3 · Resolve a domain with nslookup/dig, then check HTTP/HTTPS with curl

**Answer (what to write in the exam):**
```bash
nslookup example.com                 # DNS lookup: which IP(s) does the name map to?
dig example.com                      # full DNS answer (record type, TTL)
dig +short example.com               # only the IP address(es)
curl -I http://example.com           # HTTP status over port 80
curl -I https://example.com          # HTTPS status over port 443 (TLS)
curl -IL http://google.com           # -L follows redirects (301 → final 200)
curl -sv -o /dev/null https://example.com 2>&1 | grep -E "Connected to|SSL|HTTP/"
```

**How to read the output (record these as your results):**
```text
$ nslookup example.com
Server:   127.0.0.53              ← the DNS server that answered
Address:  127.0.0.53#53           ← its IP, port 53
Non-authoritative answer:         ← came from a cache, not the domain's own DNS server
Name:     example.com
Address:  104.20.23.154           ← the A record (IPv4 address) = DNS RESULT

$ dig example.com     (ANSWER SECTION)
example.com.   300   IN   A   104.20.23.154
   name        TTL  class type  value     ← TTL 300 = cache this answer for 300 seconds

$ dig +short example.com
104.20.23.154

$ curl -I https://example.com
HTTP/2 200                                ← HTTP STATUS = 200 OK, HTTPS works
$ curl -I http://google.com
HTTP/1.1 301 Moved Permanently            ← 3xx = redirect; Location: header says where
```
(IP addresses change over time and by location; write down whatever your machine shows.)

**What each line does:**
- `nslookup <name>` / `dig <name>`: ask the DNS server for the name's A record (IPv4); `dig` shows more detail (TTL, record type, which server answered, query time).
- `dig +short`: prints only the answer, handy for scripts.
- `curl -I http://…` vs `https://…`: tests port 80 (plain) vs port 443 (encrypted). `-I` = headers only.
- `curl -L`: follow redirects until the final page.
- `curl -sv … | grep`: shows `Connected to example.com (104.20.23.154) port 443`, the TLS/SSL handshake lines, and the HTTP status line, which proves DNS → TCP → TLS → HTTP all worked.

**Status code families:** 2xx success (200 OK) · 3xx redirect (301, 302) · 4xx client error (400, 401, 403, 404) · 5xx server error (500, 502, 503).

**Verify:** `dig +short` returns at least one IP (empty output = DNS failure, e.g. `NXDOMAIN` = name does not exist), and `curl -I https://…` first line is `HTTP/… 200`.

**🧠 In simple words:** before your computer can talk to a website it must turn the name into an IP address (DNS); `nslookup`/`dig` show you that step on its own. Then `curl` makes a real web request and the status code tells you if the website answered correctly over HTTP and over HTTPS. In short: **dig/nslookup = name → IP, curl -I = is the website answering, and with which status code.**
