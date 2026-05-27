# Example 05 — TCP vs UDP: when each one is correct

Both run on top of IP. The difference is what guarantees they give you, and what those guarantees cost.

## TCP — connection-oriented, reliable, ordered

- **3-way handshake** before any data flows (`SYN`/`SYN-ACK`/`ACK`).
- Every byte is acknowledged. Lost packets are retransmitted.
- Packets arrive **in order**, even if the network reorders them.
- **Flow control** (don't drown the receiver) + **congestion control** (don't drown the network).
- Connection is stateful — both sides remember sequence numbers, windows, RTT estimates.

**Cost:** 1 RTT for handshake, head-of-line blocking (if packet #5 is lost, #6 and #7 wait), more overhead per packet.

## UDP — connectionless, best-effort, unordered

- No handshake. Just send.
- Packets may be lost. May arrive out of order. May arrive twice.
- No flow control, no congestion control (unless the app implements it).
- Stateless — the OS just slings packets at the destination.

**Cost:** the app must handle loss, ordering, deduplication, congestion — **if it cares**.

## When to use TCP

If you need **every byte, in order, exactly once**:

- HTTP, HTTPS, REST, gRPC (over HTTP/2).
- SSH.
- Database connections (Postgres, MySQL).
- Email (SMTP).
- File transfer (FTP, SFTP).
- Anything where losing one byte breaks correctness.

## When to use UDP

If you need **speed** and can **tolerate loss**, or you need **broadcast/multicast**:

- **Voice/video calls** (Zoom audio, WebRTC) — if you drop one 20ms audio frame, the user hears a tiny glitch. Better than waiting for retransmit and getting 200ms of silence.
- **Online games** (FPS, real-time multiplayer) — if a player position update is lost, the next one (50ms later) makes it irrelevant. Retransmitting stale data is worse than dropping it.
- **DNS queries** — single small request, single small response. TCP handshake would double the cost. (DNS falls back to TCP for large responses.)
- **DHCP, mDNS, NTP** — discovery and time sync protocols.
- **Streaming telemetry** — IoT sensors firing readings at high rate; losing 1% is fine.
- **QUIC** (used by HTTP/3) — built on UDP, implements reliability at the app layer with better behavior than TCP (no head-of-line blocking across streams).

## The architect's decision framework

Ask:
1. **Is correctness violated by losing a packet?** → TCP.
2. **Is freshness more important than completeness?** → UDP.
3. **Do I need ordering across messages?** → TCP.
4. **Is connection setup latency a problem?** (e.g., one-shot DNS query) → UDP.
5. **Will I send to many receivers at once?** (multicast/broadcast) → UDP (TCP is point-to-point only).

## Why this comes up in system design

**WhatsApp / Telegram messaging:** TCP for the chat text (must arrive, must be ordered). UDP-based protocols for voice calls.

**Stock trading APIs:** market-data feeds are often UDP multicast (broadcast prices to thousands of subscribers, drop tolerable). Order submission is TCP (lose an order = lose money).

**Online games:** UDP for position updates, TCP for chat and matchmaking.

**Modern web:** HTTP/3 is UDP-based (QUIC), and it's becoming dominant. The "always use TCP for web" rule is changing.

## The protocol stack picture

```
HTTP/1, HTTP/2, gRPC, SMTP, SSH, MySQL
              ↓
             TCP
              ↓
             IP
              ↑
             UDP
              ↑
DNS, DHCP, NTP, RTP (voice), QUIC/HTTP/3, games
```

## Architect's takeaway

- TCP is the default for **anything stateful and reliable**.
- UDP is the right choice when **stale data is worse than no data**.
- The boundary isn't sharp anymore — QUIC (HTTP/3) shows you can build TCP-grade reliability on UDP and beat TCP.
- The real question is *"what does my app do when a packet is lost?"*. The answer drives the choice.
