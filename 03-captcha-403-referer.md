# 03 · CAPTCHA image fails to load: "please refresh or try a different network"

**Conclusion**: not the proxy, not the node, not the exit IP. The real cause was **a browser extension using `declarativeNetRequest` to globally rewrite the `Referer` header on requests to a particular CDN**. The CAPTCHA image happened to live on that same CDN, which triggered hotlink protection, returned `referer-acl-deny`, 403, and a broken image.

---

## 1. Led astray by four words, wrong three times in a row

The symptom was a slider CAPTCHA on a login page showing `image failed to load, please refresh or try a different network [5202]`. The machine runs a proxy full time, so every guess went to the network layer:

1. **"The exit IP is overseas, the CAPTCHA CDN is refusing it"** so I had the user disable the system proxy. **Wrong**: this Chrome is launched from a shortcut that hard-binds `--proxy-server=127.0.0.1:10808`. It never consults the Windows system proxy, so disabling it changed nothing.
2. **"The CAPTCHA domain is not in the direct-connect list and is leaking through the overseas node"** so I added a batch of direct rules to the routing config. **No effect.** Still broken.
3. **"The page loaded the international CAPTCHA variant, and the international variant cannot pull domestic images"** so I had the user switch to a direct connection and a clean Chrome profile. **Still nothing.**

All three were self-consistent, unverified network-layer stories. The truth was not in the network layer at all.

---

## 2. One look at the request headers and it was obvious

F12 → Network → click refresh on the CAPTCHA → grab the red 403 image request → read the headers:

```
Request URL:  https://<cdn-host>/tos-cn-i-xxxxxx/....jpeg
Status:       403 Forbidden
Ks-Deny-Reason: referer-acl-deny          <- the server is refusing, reason: hotlink protection
--- Request Headers ---
Origin:       https://www.<other-site>.com    <- why is this page sending another site's origin?
Referer:      https://www.<other-site>.com/   <- this is what triggers referer-acl-deny
Sec-Fetch-Site: cross-site
```

Reading it:

- The `tos-cn-i` prefix marks a domestic image, while the `Referer` is an international site. The CDN's hotlink protection does not accept that referrer, so 403.
- **`Remote Address: 127.0.0.1:10808` is a red herring.** This Chrome hard-binds a proxy, so every request shows 10808 as its remote address. **You cannot tell from it whether the underlying route went direct or through the proxy.** Do not let it mislead you.
- That domain had already been set to direct in step 2, and it was still 403. That is the proof: this is not a routing problem, the server is refusing based on `Referer`.
- `Referer` is set automatically by the browser from page context. There is only one explanation for it carrying another domain: **something is rewriting the header.**

---

## 3. Finding the extension that rewrites headers

Scan the permissions of every extension in the Chrome profile, looking for one that has both host access to that CDN domain and the ability to modify headers:

```powershell
$ext = "$env:LOCALAPPDATA\Google\Chrome\User Data\Default\Extensions"
Get-ChildItem $ext -Directory | ForEach-Object {
  $v = Get-ChildItem $_.FullName -Directory | Sort-Object Name -Descending | Select-Object -First 1
  $j = Get-Content (Join-Path $v.FullName manifest.json) -Raw | ConvertFrom-Json
  $perms = (@($j.permissions)+@($j.host_permissions)) -join ','
  if($perms -match '<cdn-domain-keyword>'){ "$($_.Name)  ||  $perms" }
}
```

One hit: a social media asset downloader extension, holding `declarativeNetRequestWithHostAccess` plus host permissions across that CDN's domains.

To download hotlink-protected media, it disguises the `Referer` on requests to those domains as the corresponding international site. That is standard practice for a downloader. The CAPTCHA image happens to sit on the same CDN, gets rewritten along with everything else, and becomes collateral damage.

This also explains why **toggling the proxy, going direct, using the node, and changing IP all had no effect**: the problem occurs inside the browser at the `declarativeNetRequest` stage, which is upstream of the network layer entirely.

---

## 4. The fix

1. Address bar → `chrome://extensions`
2. Find the extension → toggle it off
3. Back to the login page → `Ctrl+Shift+R` for a hard refresh → the CAPTCHA image loads
4. Turn it back on when you next need to download assets

You only need to disable it temporarily when logging into a site backed by that same CDN and hitting a CAPTCHA.

---

## 5. Lessons

- **"Try a different network" does not mean it is a network problem.** Get ground truth from the F12 request and response headers first, then talk about the network. The single header `Ks-Deny-Reason: referer-acl-deny` swung the investigation from "proxy" to "Referer".
- **Header-rewriting browser extensions are silent killers.** Media downloaders and scraping extensions routinely rewrite `Referer` and `Origin` globally, and take out unrelated services on the same CDN as collateral. When you hit a CAPTCHA failure or a hotlink 403, check `chrome://extensions` for header-rewriting extensions early.
- **A Chrome launched with a hard-bound proxy always reports the proxy port as `Remote Address`**, so you cannot use it to infer the underlying direct-versus-proxy decision.
- 🔴 **Do not kill proxy processes bare-handed.** While editing the routing database I ran `Stop-Process` on the proxy but did not immediately restart it. Nothing was listening on the port while the system proxy still pointed at it, and the whole machine lost connectivity. Any such change has to be one continuous motion: back up, stop, edit, **restart immediately**. Or just use the GUI, which reloads gracefully when saving routes and never drops the connection.
