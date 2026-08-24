# 01 · Self-hosted Reality node unreachable, one full day of debugging

**Conclusion**: two causes stacked on top of each other. A type error on the server's `short_id` was the first layer. The second was that **I read the Reality keys off a scaled-down screenshot by eye and mistyped characters**, so the public and private keys were never a matching pair. The dest domain and the local TUN client, both of which I accused, were innocent.

---

## 1. Two error strings that are not the same thing

This table is the most useful artifact from the whole session. Confuse these two error strings and you will debug in entirely the wrong direction.

| Client error | Meaning | Actual cause |
|---|---|---|
| `REALITY: received real certificate (potential MITM or redirection)` | The server **did not recognize** the client, treated it as a stranger, fell back to proxying the real site, and so the client received a genuine certificate | The server's `short_id` was never registered, or something on the path is intercepting |
| `reality verification failed` | The server **did recognize** the client and completed a normal handshake, but the client failed to verify the signature with its public_key | private_key and public_key are not a matching pair |

> After fixing `short_id`, the error changed from the first to the second. **That was progress**: the server had stopped falling back and started handshaking properly. At the time I did not realize this was good news and panicked for another round.

---

## 2. First layer: `short_id` must be an array on the server

```jsonc
// Wrong. But sing-box still reports active and still listens on 443, which is thoroughly misleading
"short_id": "0123456789abcdef"

// Right
"short_id": ["0123456789abcdef"]
```

```bash
sed -i 's#"short_id": "[^"]*"#"short_id": ["<SHORT_ID>"]#' /etc/sing-box/config.json
```

⚠️ On the client side `short_id` **is a string**. Do not change it to an array to match. Array on the server, string on the client, is correct.

---

## 3. Second layer: never transcribe a key by hand

This is what actually consumed the day. Every public_key I entered differed from the server's private_key by a character or two. In a scaled screenshot rendered in a monospace font, `O` and `0`, or `l` and `I` and `1`, are nearly indistinguishable.

**The fix is to keep the key away from human eyes and hands entirely**:

```bash
# Generate a pair and write it to disk
sing-box generate reality-keypair > /root/kp.txt

# Substitute it straight into the config. Nothing is typed by hand
sed -i "s#\"private_key\": \"[^\"]*\"#\"private_key\": \"$(awk '/PrivateKey/{print $2}' /root/kp.txt)\"#" /etc/sing-box/config.json

# Have the server generate the share link too, then diff what I read back against it
echo '<the link as I transcribed it>' | diff - /root/link.txt
```

Only send the link if `diff` prints nothing. This one step eliminated an entire class of failure.

---

## 4. A trap when changing keys with sed

```bash
# Does nothing. The old private key starts with a dash, the pattern never matches,
# and sed neither errors nor warns. It fails silently.
sed -i 's#-OLD_KEY_STARTING_WITH_DASH#<NEW>#' /etc/sing-box/config.json

# Match on the field name instead, regardless of what the old value looks like
sed -i 's#"private_key": "[^"]*"#"private_key": "<NEW>"#' /etc/sing-box/config.json
```

**Always `grep` the file back afterwards.** I did not, and wasted two rounds of testing on a change that had never applied.

---

## 5. The golden self-test: connect the server to itself

This is what I should have done first and did not do until last. Start a client on the server and connect it to `127.0.0.1:443`. In one move this rules out the client machine, the local VPN, the firewall, and the national firewall.

```bash
# Write the config in chunks
printf '%s' '{"log":{"level":"error"},"inbounds":[{"type":"socks","listen":"127.0.0.1","listen_port":10999}],"outbounds":[{"type":"vless","server":"127.0.0.1","server_port":443,' > /tmp/c.json
printf '%s' '"uuid":"<UUID>","flow":"xtls-rprx-vision","tls":{"enabled":true,"server_name":"<SNI>","utls":{"enabled":true,"fingerprint":"chrome"},' >> /tmp/c.json
printf '%s' '"reality":{"enabled":true,"public_key":"<PUBLIC_KEY>","short_id":"<SHORT_ID>"}}}]}' >> /tmp/c.json
sing-box check -c /tmp/c.json

# Run the whole self-test as a single command
pkill -f c.json; sleep 1; setsid sing-box run -c /tmp/c.json >/tmp/c.log 2>&1 & sleep 8; curl -s --socks5-hostname 127.0.0.1:10999 -m 20 https://api.ipify.org; echo ' <=IP'; echo LOG:; cat /tmp/c.log; pkill -f c.json
```

It only counts as working if your VPS IP comes back. An empty IP means read the error string in the log and locate it against the table in section 1.

---

## 6. Hard limits of the BandwagonHost KiwiVM "Root shell - basic" console

The web console is not a shell. It is a **stateless command executor**. It took me a whole evening to understand this:

- **Stateless**: every press of enter is a brand new shell. Environment variables and `cd` do not persist across commands.
- **Heredocs do not work at all, and they are dangerous.** A heredoc hits end-of-file immediately, so its body executes as separate commands, and **the `>` has already truncated the config file to zero bytes**. I wiped my config this way once. **Back it up with `cp` before touching it.**
- **Single commands are capped at roughly 255 characters.** Past that it reports `Command transmission error`. Long JSON has to be appended in chunks with `printf '%s' '...' >> file`. The long command that was supposed to write the config had been failing silently for exactly this reason.
- **Background processes do not survive across commands**, so a self-test has to pack `setsid ... &`, `sleep` and `curl` into one single command.
- The console times out after a minute or two of inactivity, and will always be gone by the next day.

---

## 7. dest / SNI test results

| dest | Result |
|---|---|
| `www.cloudflare.com` | Failed during this session, but **later proved to be a misdiagnosis caused by the misread key** |
| `www.microsoft.com` | Same |
| `www.amazon.com` | Works. `curl -sI` returns 503, which is HTTP-layer rate limiting. The TLS handshake succeeds, which is all Reality needs |
| `www.apple.com` | What I settled on |

**Correction**: looking back after the keys were fixed, dest was never the problem. At the time I treated this entire table as a set of conclusions about dest compatibility. In reality it was one key error reproducing itself under different dest values. **A single unresolved variable will contaminate your conclusions on every other dimension you test.**

---

## 8. The TUN client I wrongly accused

The machine also ran a commercial TUN client that started with Windows, which is why the tunnel was already up before I had touched anything after a reboot. It does intercept at the process and WFP level, and static routes do not get around it: after adding a `/32` route for the VPS IP pointing at the physical gateway, `Find-NetRoute` confirmed traffic was taking the physical adapter, and the connection was still captured.

**The decisive evidence**: after killing its service and all its processes completely, the node still showed delay -1 and still returned `received real certificate`. That is what proved it was not the root cause.

Once the keys were fixed, both can run at the same time. A static route letting the node connection bypass the TUN tunnel is sufficient. The earlier claim that it "has to be turned off" was another conclusion the misread key had led me to.
