# Labs · Session 4 · Networking

Acronyms used: IP = Internet Protocol · DNS = Domain Name System (turns names into IP addresses) · TCP = Transmission Control Protocol (reliable, ordered) · UDP = User Datagram Protocol (fast, no delivery guarantee) · ICMP = Internet Control Message Protocol (what `ping` uses) · HTTP/HTTPS = HyperText Transfer Protocol (Secure = encrypted with TLS, Transport Layer Security) · CIDR = Classless Inter-Domain Routing (the `/24` notation) · TTL = Time To Live · PID = Process ID.

---

## Lab 1 · Show the IP address, subnet, default route and listening ports

**Answer (what to write in the exam):**
```bash
ip addr show            # short form: ip a   → IP address + subnet prefix of each interface
ip -4 addr show eth0    # only the IPv4 address of one interface (name may be ens33 / enp0s3)
ip route show           # routing table; the "default" line is the default gateway
ip route show default   # only the default route
sudo ss -tulpn          # all listening TCP/UDP ports + which program owns each one
```

**How to read the output (write this as your "interpretation"):**
```text
$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> ...
    inet 127.0.0.1/8 scope host lo
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    inet 192.168.1.25/24 brd 192.168.1.255 scope global dynamic eth0
```
- `eth0` = the network card; `UP` = it is switched on. `lo` = loopback (the machine talking to itself, 127.0.0.1).
- `inet 192.168.1.25/24` → **IP address** = 192.168.1.25; **/24 prefix** = subnet mask 255.255.255.0, so the first 24 bits (192.168.1) are the **network part**. Network = 192.168.1.0/24 → 254 usable host addresses (.1 to .254).
- `brd 192.168.1.255` = broadcast address (sends to every host in the subnet). `dynamic` = address came from DHCP (Dynamic Host Configuration Protocol).

```text
$ ip route
default via 192.168.1.1 dev eth0 proto dhcp metric 100
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.25
```
- `default via 192.168.1.1` → **default gateway** = 192.168.1.1 (the router). Any traffic not for our own subnet is sent there.
- Second line: hosts in 192.168.1.0/24 are reached **directly** on eth0, no router needed.

```text
$ sudo ss -tulpn
Netid State   Local Address:Port   Peer Address:Port  Process
tcp   LISTEN  0.0.0.0:22           0.0.0.0:*          users:(("sshd",pid=812,fd=3))
tcp   LISTEN  127.0.0.1:5432       0.0.0.0:*          users:(("postgres",pid=990,fd=5))
udp   UNCONN  127.0.0.53%lo:53     0.0.0.0:*          users:(("systemd-resolve",pid=601,fd=12))
```
- Flags: `-t` TCP, `-u` UDP, `-l` only listening sockets, `-p` show process name/PID (needs `sudo`), `-n` show numbers (port 22) not names (ssh).
- `0.0.0.0:22` → SSH (Secure Shell) is listening on **all** interfaces, so other machines can connect. `127.0.0.1:5432` → PostgreSQL accepts **local-only** connections. `udp ... :53` → local DNS resolver.

**Verify:** `ip a` shows an `inet x.x.x.x/NN` line on your main interface; `ip route` shows exactly one `default via …` line; `ss -tulpn` shows `LISTEN` rows.
(Mac: `ifconfig`, `netstat -rn`, `lsof -iTCP -sTCP:LISTEN -n -P`.)

**🧠 In simple words:** `ip a` tells you "who am I on the network" (your IP and how big your subnet is), `ip route` tells you "where do I send traffic that is not local" (the default gateway), and `ss -tulpn` tells you "which programs are waiting for connections, on which ports". These three are the first things you check on any server. In short: **ip a = my address + subnet, ip route = my gateway, ss -tulpn = my open ports.**

---

## Lab 2 · Troubleshoot a web service with ping, ss and curl (reachability vs closed port vs HTTP error)

**Answer (what to write in the exam):**
```bash
# Setup: a simple web server on the target machine (192.168.1.50), port 8080
python3 -m http.server 8080 &

# Step 1 – Is the host reachable at all? (network layer)
ping -c 4 192.168.1.50
# Step 2 – Is anything listening on the port? (run ON the server)
sudo ss -tulpn | grep :8080
# Step 3 – Can I connect, and what HTTP status comes back? (application layer)
curl -v http://192.168.1.50:8080/
curl -I http://192.168.1.50:8080/
curl -s -o /dev/null -w "%{http_code}\n" http://192.168.1.50:8080/missing.html

# Simulate the 3 failures:
#  a) wrong/unreachable IP:     ping -c 4 192.168.1.99
#  b) closed port:              kill %1   (stop the server), then curl again
#  c) HTTP error:               curl -I http://192.168.1.50:8080/missing.html  → 404
```

**Decision table (the key part of the answer):**

| What you see | Diagnosis | Where the problem is |
|---|---|---|
| `ping`: `100% packet loss` or `Destination Host Unreachable`; `curl`: `Connection timed out` / `No route to host` | **Reachability problem** (host down, wrong IP, bad route, firewall dropping packets) | Network (IP/routing) |
| `ping` OK (`0% packet loss`), but `curl`: `Failed to connect … Connection refused`, and `ss` shows **nothing** `LISTEN` on :8080 | **Closed port** (service stopped / crashed / listening on a different port or only on 127.0.0.1) | Transport (TCP port) |
| `curl` connects (`* Connected to 192.168.1.50 port 8080`) but status is **4xx** (e.g. 404 Not Found, 403 Forbidden) or **5xx** (500 Internal Server Error, 503 Service Unavailable) | **HTTP-level problem** (wrong path, permissions, app bug, backend down) | Application (HTTP) |
| `curl` gets `HTTP/1.0 200 OK` | Service is healthy | none |

**What each line does:**
- `python3 -m http.server 8080 &`: starts a tiny web server on port 8080 in the background (`&`), so we have something to test.
- `ping -c 4 <ip>`: sends 4 ICMP echo requests (`-c` = count; without it Linux pings forever). Read `4 received, 0% packet loss` and `time=… ms`.
- `ss -tulpn | grep :8080`: shows if a process is in `LISTEN` state on port 8080 (`grep` filters the line).
- `curl -v URL`: verbose; shows each step: DNS lookup, `Connected to … port 8080` (TCP worked), the request sent (`>`), and the response status (`< HTTP/1.0 200 OK`).
- `curl -I URL`: sends a HEAD request, prints **only the response headers** (first line = status code).
- `curl -s -o /dev/null -w "%{http_code}\n"`: silent, throws the body away, prints just the status number (e.g. `404`).

**Verify:** healthy case → `ping` shows `0% packet loss`, `ss` shows `LISTEN … :8080 … python3`, `curl -I` shows `HTTP/1.0 200 OK`. Note: some networks block ICMP, so a failed `ping` with a working `curl` means "ping blocked", not "host down".

**🧠 In simple words:** you test from the bottom up: first "can I reach the machine?" (ping), then "is the program actually listening on that port?" (ss + curl connect), then "does the program answer correctly?" (HTTP status code). Each failure type gives a different message, so the first step that fails tells you the layer to fix. In short: **ping fails = network, connection refused = closed port, 4xx/5xx = the web app itself.**

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
