# 02 · Desktop app cannot log in, while its own CLI works fine

**Conclusion**: not an account problem, not an exit IP blocked by Cloudflare, not a broken node. The real cause was that **the Windows system proxy was switched off (`ProxyEnable=0`), and Electron applications read only the system proxy**. The app had been surviving on a second TUN client that was catching traffic at the network layer.

---

## 1. First, overturning the previous conclusion

The earlier verdict was:

> The exit IP is blocked by Cloudflare, the site returns a 403 challenge page, so the desktop app never receives the login JSON. Recommend switching nodes.

The evidence was a bare `curl` against the target returning 403 with a Cloudflare challenge page.

**That conclusion was wrong.** Bare curl has no browser user agent and no browser TLS fingerprint. Getting a 403 from Cloudflare is the normal outcome, and a textbook false positive. Retest with a normal browser UA:

```powershell
$ua = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36"
curl.exe --socks5-hostname 127.0.0.1:10808 -s -m 25 -D - -o NUL -A $ua https://<target>/
```

Result:

```
HTTP/1.1 302 Found
Location: https://<target>/login
Server: cloudflare
```

A 302 to the login page is entirely normal, and there is no `cf-mitigated: challenge` header in the response. The exit IP was never blocked.

---

## 2. The actual causal chain

| Test | Result | Meaning |
|---|---|---|
| Node exit, `curl --socks5-hostname 127.0.0.1:10808 https://api.ipify.org` | Returns the VPS IP | The node itself is completely fine |
| API domain | `{"message":"hello"}` | Which is why the CLI kept working |
| Target site, bare curl | 403 | False positive, see section 1 |
| Target site, with browser UA | 302 to /login | The real answer: it works |
| **System proxy registry key** | **`ProxyEnable = 0`** | The culprit |
| Leftover system proxy address | `127.0.0.1:7892` | Belongs to the other client, not the 10808 in current use |
| TUN adapter | Up, default route at metric 0 | Catching all traffic, which masked the problem |

```
Electron desktop app
      |
      +-- reads only the Windows system proxy (WinINET)
                |
        ProxyEnable=0  <-- switched off, so 10808 is never used
                |
                v
          default route
                |
      +---------+---------+
   TUN on              TUN off
      |                    |
  caught at the        direct connection,
  adapter layer        blocked
  (regardless of       |
  app proxy support)   v
      |             login fails
      v
   works
```

**Why did the CLI keep working?** Its config file sets `HTTPS_PROXY=http://127.0.0.1:10808` as an explicit environment variable, so it never depends on the system proxy.

That fully explains the seemingly bizarre split: the CLI is fine, the desktop app cannot log in.

---

## 3. The fix

Turn on the client's "Set system proxy" toggle. The equivalent registry state:

```powershell
$k = "HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings"
Set-ItemProperty $k -Name ProxyServer -Value "127.0.0.1:10808"
Set-ItemProperty $k -Name ProxyEnable -Value 1 -Type DWord
# then P/Invoke wininet.dll InternetSetOption(0, 39, 0, 0) and (0, 37, 0, 0) to broadcast the change
```

Chromium picks up proxy changes live, so the desktop app does not need a restart.

⚠️ **But after editing the registry you still have to click the toggle in the GUI.** The client's own state machine still believes the toggle is off, and it may reset the registry when it exits or restarts. Only when both agree is the change persistent.

### Acceptance checks

1. Shut down the other TUN client
2. Confirm the TUN adapter is Down and the only default route is the physical adapter
3. `curl.exe --socks5-hostname 127.0.0.1:10808 -s https://api.ipify.org` returns the VPS IP
4. The desktop app logs in normally
5. A browser opens the target site and reaches the login page

### Fallback

Some Electron builds bypass the system proxy and connect directly. In that case enable the client's own TUN mode (requires administrator), which captures at the network layer. ⚠️ Two TUN clients cannot run at once. They will fight over the default route.

---

## 4. Lessons

**1. The test tool is a variable too.** When testing a Cloudflare-fronted site with curl, always send a browser UA and read the `cf-mitigated` header. Never judge by status code alone.

**2. "The story that explains everything" is still the most dangerous.** "The IP is blocked by CF" explained the failed login, explained why turning on TUN fixed it, and explained why changing nodes might help. Everything lined up, and it was wrong.

**3. A split symptom is a gold mine.** A works and B does not: ask first whether A and B acquire their proxy the same way.

**4. Configuration drifts. Read it back periodically.** My own written setup guide explicitly said "Set system proxy: on". Reality was off. **Diffing the current state against your own documentation beats guessing by a factor of ten.**

**5. Write the notes down before running a test that can cut your own connection.** Verifying this fix required shutting down the TUN client that the debugging session itself was running on. If it failed, the session would drop. So the notes got written first, then the change.
