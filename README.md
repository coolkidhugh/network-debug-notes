# Network Debugging Notes

Four real failures, traced to root cause. Every one of them shares a pattern: **I first invented a story that explained all the symptoms, spent a long time fixing according to that story, and eventually found the real cause in a completely different layer.**

What makes these worth keeping is not the conclusions. It is the record of how I went wrong, and which specific action pulled me back.

| # | Symptom | What I thought caused it | Actual cause |
|---|---|---|---|
| [01](01-reality-node-wont-connect.md) | Self-hosted Reality node unreachable, client shows delay -1 | Incompatible dest domain, or a local TUN client intercepting | A type error on `short_id` in the config, plus **I misread key characters off a screenshot** |
| [02](02-electron-app-ignores-proxy.md) | Desktop app cannot log in, while its own CLI works fine | Exit IP blocked by Cloudflare | The Windows system proxy toggle was off, and Electron reads only the system proxy |
| [03](03-captcha-403-referer.md) | CAPTCHA image fails to load, page says "try a different network" | Overseas exit IP, or the domain not routed direct | A browser extension rewrote `Referer` globally, triggering CDN hotlink protection |
| [04](04-vps-suspended-bandwidth.md) | Node dies completely, no warning | Another client fighting over routes | VPS suspended for exceeding the monthly transfer cap. TUN mode had been counting domestic traffic too |

---

## Five things that carried over

**1. The story that explains everything is the dangerous one.**

"The exit IP is blocked by Cloudflare" simultaneously explained: the desktop app cannot log in, switching to the other client fixes it, and changing nodes might help. All three symptoms lined up. It was still wrong. How well a story holds together tells you nothing about whether it is true.

**2. The test tool is a variable too.**

Before using `curl` to test whether something is being blocked by Cloudflare, you have to rule out that `curl` gets blocked by Cloudflare on its own. Bare curl has no browser user agent and no browser TLS fingerprint, so a 403 is the normal outcome. Add a UA and retest, and the 403 becomes a 302.

```bash
UA="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36"
curl -s -D - -o /dev/null -A "$UA" https://example.com/
```

Read the `cf-mitigated` response header. Do not judge by status code alone.

**3. A split symptom is a gold mine.**

"The CLI works, the desktop app does not" points directly at one thing: the two acquire their proxy through different paths. Check the system proxy and the case solves itself.

> When A works and B does not, ask first: do A and B get their proxy the same way?

**4. Eliminate variables physically, not by reasoning.**

In case 01, the single most time-saving action was **starting a client on the server itself and connecting to `127.0.0.1:443`**. Thirty seconds, and it rules out the client machine, the local VPN, the firewall, and the national firewall all at once. Instead I spent a full day on the client side and never once ran that self-test.

**5. Read back after every change. Never assume the write succeeded.**

`sed` does not complain when its pattern matches nothing. It exits zero. Read the config back with `grep` after every edit. It costs seconds and saves an entire round of testing against a change that never applied.

---

## About the example values

All IP addresses, keys, UUIDs and account paths have been replaced with placeholders (`<VPS_IP>`, `<PRIVATE_KEY>`, and so on). The commands are otherwise verbatim. Substitute your own values and they will run.

---

## Author

**H** ([@odpanda](https://x.com/odpanda)) · [写春秋管理咨询](https://www.xchqzx.com/?utm_source=github&utm_medium=readme&utm_campaign=network-debug-notes)（人效提升、组织诊断、薪酬绩效）— practical notes on debugging production networks.

