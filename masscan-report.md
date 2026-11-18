# Masscan — Fast Port Scanner

**Student:** Steve
**Target:** Metasploitable 3 (metasploitable3-ub1404), 192.168.10.12
**Scanner:** Kali Linux, masscan 1.3.2
**Scope:** Lab only — my own virtual machines

## What Masscan is

Masscan is a port scanner like Nmap, but much faster. It can sweep thousands of ports in seconds. It only tells you which ports are open — it does not tell you the service name or version. That part is still Nmap's job. The normal workflow is: Masscan first for a fast, wide sweep, then Nmap on the open ports for detail.

## Step 1 — Check Masscan is installed

Command:

    masscan --version

Result:

    Masscan version 1.3.2 ( https://github.com/robertdavidgraham/masscan )
    Compiled on: Jul 25 2024 16:44:22
    Compiler: gcc 14.1.0
    OS: Linux
    CPU: unknown (64 bits)
    GIT version: unknown

Masscan was already installed on Kali. (One attempt failed with "Command 'mascan' not found" — that was a typing mistake, one "s" missing. The correct command is `masscan`.)

## Step 2 — Find the target's IP

Metasploitable 3 gets its IP from DHCP, so it changes. It was 192.168.10.91, then 192.168.10.162. Before scanning, the target was checked with a ping sweep:

    nmap -sn 192.168.10.0/24

The host with MAC address 00:0C:29:08:C5:97 was Metasploitable 3, now at **192.168.10.12**. Always verify the target IP first — scanning an old IP finds nothing.

## Step 3 — First Masscan scan (1–10000)

Command:

    sudo masscan -p1-10000 192.168.10.12 --rate 1000 -oL masscan-first-scan.txt

The screen showed no open ports. This was confusing, because Nmap finds 7 open ports on this machine. The reason was found in the next steps: the `-oL` option writes results to a file and prints nothing on screen. The results were in the file all along.

Lesson: with `-oL`, read the file. The screen stays quiet.

Note: pasting the command into the terminal added stray characters (`^[[200~`) and failed with `zsh: bad pattern`. Type commands by hand and check for stray characters before pressing Enter.

## Step 4 — Small test: one port, slow rate

To check whether Masscan itself worked, one port was scanned slowly:

    sudo masscan -p80 192.168.10.12 --rate 100

Result:

    Discovered open port 80/tcp on 192.168.10.12

Masscan works. Port 80 was found.

## Step 5 — Full scan again, slow rate

Command:

    sudo masscan -p1-10000 192.168.10.12 --rate 100 -oL masscan-first-scan.txt

Time: about 2 minutes (10,000 ports at 100 packets per second). The screen again showed nothing, but the file had the results:

    cat masscan-first-scan.txt

    #masscan
    open tcp 3306 192.168.10.12 1795011643
    open tcp 445 192.168.10.12 1795011648
    open tcp 3500 192.168.10.12 1795011658
    open tcp 21 192.168.10.12 1795011660
    open tcp 8080 192.168.10.12 1795011663
    open tcp 631 192.168.10.12 1795011669
    open tcp 22 192.168.10.12 1795011687
    open tcp 6697 192.168.10.12 1795011711
    open tcp 80 192.168.10.12 1795011729
    # end

Masscan found **9 open ports** — the 7 Nmap already knew, plus 2 extra.

## Step 6 — Identify the two extra ports

Nmap's default scan only checks the top 1000 ports, so it never probed 3500 and 6697. Their services were identified with:

    nmap -sV -p 3500,6697 192.168.10.12

Result:

    PORT     STATE SERVICE VERSION
    3500/tcp open  http    WEBrick httpd 1.3.1 (Ruby 2.3.8 (2018-10-18))
    6697/tcp open  irc     UnrealIRCd

- **3500/tcp** — a small Ruby web server (WEBrick).
- **6697/tcp** — an IRC chat server (UnrealIRCd, host irc.TestIRC.net).

## Results table

| Port | Service | Found by |
|------|---------|----------|
| 21/tcp | FTP | Nmap + Masscan |
| 22/tcp | SSH | Nmap + Masscan |
| 80/tcp | HTTP | Nmap + Masscan |
| 445/tcp | SMB (microsoft-ds) | Nmap + Masscan |
| 631/tcp | IPP (printing) | Nmap + Masscan |
| 3306/tcp | MySQL | Nmap + Masscan |
| 8080/tcp | HTTP proxy | Nmap + Masscan |
| 3500/tcp | HTTP (WEBrick / Ruby) | Masscan only |
| 6697/tcp | IRC (UnrealIRCd) | Masscan only |

## Masscan vs Nmap

| | Nmap (default scan) | Masscan (1–10000) |
|---|---|---|
| Ports checked | Top 1000 | 1–10000 |
| Open ports found | 7 | 9 |
| Time | ~5 seconds | ~2 minutes (rate 100) |
| Service names/versions | Yes | No |

Masscan's wide sweep caught 2 ports that Nmap's default scan misses. Nmap is still needed to name the services. Use them together: Masscan first to find everything fast, Nmap second for detail.

## Lessons learned

1. Verify the target IP before scanning — DHCP changes it (here .91 → .162 → .12).
2. With `-oL`, Masscan writes results to the file only. An empty screen does not mean an empty result — read the file.
3. Type commands by hand. Pasting adds stray characters (`^[[200~`) and fails.
4. Masscan does not resend a lost probe, so on a lossy path a slower rate finds more. (In this lab the rate was not the cause — the file held the results.)
5. A fast, wide sweep plus a detailed follow-up finds more than either tool alone.

## Conclusion

Masscan works as a fast first sweep: in about 2 minutes it covered 10,000 ports and found 9 open ones, including 2 that Nmap's default scan had never checked. The two extra services (a Ruby web server on 3500 and an IRC server on 6697) are real Metasploitable 3 services. For a full assessment, follow a Masscan sweep with Nmap service detection on everything found.
