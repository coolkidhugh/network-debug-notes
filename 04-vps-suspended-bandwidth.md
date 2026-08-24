# 04 · The node dies completely, delay -1

**Conclusion**: not a client fighting over routes, not broken proxy software. The real cause was that the **VPS exceeded its monthly transfer cap (1001.4 of 1000 GB) and was suspended by the provider**. The server stopped answering on 443, so the client reported `The delay: -1 ms`.

Most likely trigger: TUN mode had been enabled, so the machine's entire traffic, including large domestic downloads, counted against that 1 TB.

---

## 1. Symptoms

| Where | What it showed |
|---|---|
| Client status bar | `delay: -1 ms, none` |
| Local test | `<VPS_IP>:443` TCP timeout after 5 seconds |
| Provider panel | Red banner `This service is currently suspended`, Reason: `Data transfer limit exceeded` |
| Usage | `1001.4 / 1000 GB`, Status: Suspended, SSH Unavailable |

---

## 2. The causal chain

```
TUN enabled, or heavy traffic through the self-hosted node
        |
        v
Provider's monthly counter crosses 1000 GB
        |
        v
Panel marks the service Suspended (443 stops answering)
        |
        v
Local client cannot reach the server, speed test reads -1
        |
        v
Every application pointed at that proxy port fails together
```

**Unrelated to the other TUN client.** It uses its own port while the self-hosted node uses another. It was still reaching the internet during this incident. What was down was the VPS itself.

---

## 3. Why TUN consumes so much traffic

| Mode | Who pays for the traffic |
|---|---|
| **System proxy with a bypass-domestic ruleset** (recommended) | Overseas traffic goes through the VPS, domestic traffic mostly goes direct. Economical |
| **TUN enabled** | Capture happens at the adapter level, so domestic downloads, video, and system updates often enter the tunnel too. Burning 1 TB within a month is easy |

Rough math: an hour of 1080p video is roughly 2 to 3 GB. Add system updates and cloud sync, and reaching 1000 GB in a month is unremarkable.

---

## 4. What to do

- **Wait for the plan's monthly reset date**, which the panel displays, and access returns automatically
- If you need it now, use the panel's "Restore access for 3 hours"
- While it is down, repoint anything that depends on that proxy port (a CLI tool's `HTTPS_PROXY` setting, for example) at a backup exit, otherwise those tools go down with it

## 5. Prevention

- **Do not run TUN day to day.** Use a system proxy with a bypass-domestic ruleset, unless you genuinely need network-layer capture (see case 02)
- Set a transfer alert in the provider's panel rather than finding out at suspension
- Keep the proxy configuration for your CLI tools switchable in one step, so an outage does not also take away the tools you need to debug it
