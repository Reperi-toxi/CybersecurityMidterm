# Cybersecurity Fundamentals: Midterm Report

**Author:** Giorgi Cheishvili
**Course:** Cybersecurity Fundamentals
**Topic:** Network traffic analysis with Wireshark: TCP handshake, HTTP vs. HTTPS

---

## Overview

This project uses Wireshark to analyse network traffic in a controlled virtual lab. It has two parts:

- **Task A:** Capture and analyse traffic to a local Python web server, and break down the TCP three-way handshake.
- **Task B:** Compare a login over plain **HTTP** with a login over **HTTPS**, to show what an attacker on the network can and cannot see.

## Lab Environment

| Component | Details |
|---|---|
| Hypervisor | VMware Workstation |
| Server / analysis machine | Kali Linux VM (`kali-linux-2026.2-vmware-amd64`), IP `192.168.153.130` |
| Client machine | Windows host, VMware **VMnet8** virtual adapter, IP `192.168.153.1` |
| Capture tool | Wireshark (running on Kali) |
| Web server | Python 3 built-in `http.server` on port `8000` |

The IP addresses were obtained with `ip a` on Kali and `ipconfig` on Windows. The VMnet8 adapter is the one Windows uses to talk to the virtual machines.

---

## Task A: Local Server Traffic & TCP Three-Way Handshake

### Setup

1. Create a dummy website and start a Python server in the folder `~/CyberMidterm`:

   ```bash
   cd ~/CyberMidterm
   echo "<html><body><h1>Random Website<h1><body><html>" > index.html
   python3 -m http.server 8000
   ```

2. Get the IP addresses of both machines: Kali `192.168.153.130`, Windows VMnet8 `192.168.153.1`.
3. Start a Wireshark capture on Kali.
4. From Windows, send two requests to the server:
   - **Request 1 (curl):** `curl http://192.168.153.130:8000`
   - **Request 2 (browser):** visit `http://192.168.153.130:8000`

The server's terminal log shows the incoming `GET / HTTP/1.1` requests from `192.168.153.1`, with `200` and `304` responses. A `404` for `/favicon.ico` appears as well, because the browser asks for it automatically.

### Wireshark Filters Used

| # | Filter | Purpose |
|---|---|---|
| 1 | `tcp.port == 8000` | Isolate all traffic related to the local server |
| 2 | `tcp.flags.syn == 1` | Show every packet with the SYN (synchronize) flag set, i.e. every time the client tried to start a connection to the server |
| 3 | `http` | Display only the HTTP requests and responses |

Saved capture: `TaskA.pcapng`

*Screenshot to be added here: filter 1, `tcp.port == 8000`*

*Screenshot to be added here: filter 2, `tcp.flags.syn == 1`*

*Screenshot to be added here: filter 3, `http`*

Filter 2 shows each connection attempt clearly. The `http` filter shows the `GET / HTTP/1.1` requests, the `HTTP/1.0 200 OK (text/html)` response, and the `HTTP/1.0 304 Not Modified` response on the browser's repeat request.

### TCP Three-Way Handshake Breakdown

With filter `tcp.port == 8000` the handshake is visible twice:

- **Frames 9, 10, 11:** the `curl` request
- **Frames 31, 32, 33:** the browser request

**Participants**

- Client: `192.168.153.1` (Windows)
- Server: `192.168.153.130` (Kali)
- Client ephemeral port: `54852` (for the curl connection)
- Server port: `8000`

| Step | Frame (curl) | Flags | Direction | Description |
|---|---|---|---|---|
| 1 | 9 | `[SYN]` | Client → Server | The client requests a connection and announces its own **ISN** (Initial Sequence Number). Source IP is `192.168.153.1`, destination is `192.168.153.130`. |
| 2 | 10 | `[SYN, ACK]` | Server → Client | The server acknowledges (ACK) the client's ISN and replies with its own ISN. Source IP is `192.168.153.130`, destination is `192.168.153.1`. |
| 3 | 11 | `[ACK]` | Client → Server | The client sends the final acknowledgement. Source is again `192.168.153.1`, destination is `192.168.153.130`. |

After the handshake completes, the HTTP request (`GET / HTTP/1.1`) is sent over the established connection.

---

## Task B: HTTP vs. HTTPS Login Comparison

### Part 1: Login over HTTP (insecure)

Logged into a test HTTP website using these test credentials and captured the traffic with Wireshark:

- **Username:** `cybermidterm`
- **Password:** `cybermidterm123`

The captured `POST /login/login_results.asp` request (**packet 80**, content type `application/x-www-form-urlencoded`) exposes the form fields in plain text:

- `txtUsername` = `cybermidterm`
- `txtPassword` = `cybermidterm123`

*Screenshot to be added here: exposed username and password*

*Screenshot to be added here: full request details for packet 80*

**What an attacker can see:**

- Packet 80 shows the full interaction between the website and the client, fully readable, because HTTP does not encrypt application data.
- **Login credentials** (username and password).
- **Session cookie**, which would let an attacker **hijack the logged-in session without even needing the password**.
- **Visited site and page** (`Host` header, referrer, requested URL).
- **Browser and operating system** (`User-Agent`).

### Part 2: Login over HTTPS (secure)

Repeating the login on a production-level website that uses HTTPS makes retrieving the data significantly harder.

- The filter `http.request.method == "POST"` returns **nothing**, because the traffic is now hidden behind TLS.
- Filtering with `tls.record.content_type == 23` shows only **TLSv1.2 "Application Data"** records, whose payload is encrypted and unreadable.

*Screenshot to be added here: the POST filter returning no results*

*Screenshot to be added here: encrypted application data of the HTTPS login*

**Why it is protected:** HTTPS wraps all data in TLS encryption, which scrambles it using secret session keys that only the browser and the website know. Anyone capturing the traffic sees only unreadable bytes, because they do not have those keys.

---

## Key Takeaways

1. The **TCP three-way handshake** (SYN → SYN/ACK → ACK) can be observed directly in Wireshark and is the foundation of every TCP connection, including HTTP.
2. **HTTP sends everything in plain text.** Credentials, cookies, hostnames and browser details are readable by anyone who can capture the traffic.
3. **HTTPS (TLS) encrypts application data**, so a passive observer sees only encrypted bytes.
4. Sensitive actions like logins should **never** be done over plain HTTP.

## Repository Structure

```
.
├── README.md
├── CyberMidterm.pdf        # Full midterm report
└── TaskA.pcapng            # Task A capture (if included)
```

## Disclaimer

This project was carried out for educational purposes in an isolated virtual lab. The credentials shown are test credentials for a test website. Only capture traffic on networks and systems you own or have explicit permission to analyse.
