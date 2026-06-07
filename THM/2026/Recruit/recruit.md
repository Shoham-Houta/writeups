# Recruit — Challenge Write-Up

> **Category:** Web 
> **Difficulty:** Medium 
> **Author:** Shoham 
> **Date:** 2026-06-07

---

## Summary

Recruit is a vulnerable web application built around escalating from anonymous access to full admin-level compromise. The path involved chaining several vulnerabilities — directory enumeration that exposed an internal log file, a Local File Inclusion that leaked plaintext credentials, and a SQL injection that enabled database dumping and ultimate privilege escalation. Tools used included `nmap` for service discovery, `gobuster` for directory brute-forcing, and `sqlmap` for injection exploitation.

---

## Challenge Info

**Given:**

- IP: `10.112.190.205 --> recruit.thm`

**Tools used:** `nmap`, `gobuster`, `sqlmap`, `Burp Suite`

---

## The Vulnerability

Because the server didn't restrict access to internal directories and log files, an attacker could enumerate paths like `/mail/` and request sensitive files directly — even though they were never meant to be publicly accessible. This kind of exposure (information disclosure) often hands an attacker the exact details (usernames, file locations, internal structure) needed to plan their next move.

That leaked credentials information set up the next stage — a Local File Inclusion (LFI) vulnerability which allowed read access to internal files like `config.php` and to recover the password in plain text, which in turn gave the needed access to discover the SQL injection vulnerability that ultimately led to full compromise.

---

## Solution Walkthrough

### Step 1 — Reconnaissance

I ran an `nmap` scan and found ports 22, 53 and 80 open, with Apache serving PHP.

```bash
nmap -sC -sV -p- 10.112.190.205 -oA recruit
```

![](/Users/shoham/Library/Application%20Support/marktext/images/2026-06-07-06-13-01-image.png)

Confirmed the target was running PHP via an API link referenced on the homepage.

![](/Users/shoham/Library/Application%20Support/marktext/images/2026-06-07-06-24-23-image.png) 

From there I ran directory enumeration to discover additional PHP pages.

```bash
gobuster dir -u http://10.112.190.205 \
-w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x .php
```

![](/Users/shoham/Library/Application%20Support/marktext/images/2026-06-07-07-44-24-image.png)

This revealed several PHP endpoints – `phpmyadmin.php`, `config.php`, `api.php` and a `/mail/` directory, with `phpmyadmin.php` standing out as a potential login point and `/mail/` worth exploring further.

Also checked `sitemap.xml` as a cross-reference, but it largely overlapped with what the gobuster scan had already turned up.

### Step 2 — Analysis of the Log File Findings

Further inspection of `mail.log` revealed that the `hr` account's credentials were stored in `config.php`.

![](/Users/shoham/Library/Application%20Support/marktext/images/2026-06-07-10-05-37-image.png)

By targeting the `cv` parameter on the `file.php` endpoint with a `file://` wrapper (`file.php?cv=file:///var/www/html/config.php`) allowed arbitrary file reads from the filesystem – a Local File Inclusion (LFI) vulnerability. This let me read `config.php` directly, which stored the `hr` account's password in plain text.

![](/Users/shoham/Library/Application%20Support/marktext/images/2026-06-07-10-09-06-image.png)  

### Step 3 — Gaining Access

I logged in using the `hr` credentials recovered from `config.php` and captured the user flag.

![](/Users/shoham/Library/Application%20Support/marktext/images/2026-06-07-11-30-46-image.png)

### Step 4 — Exploration of the Dashboard

After logging in as `hr`, I was presented with a dashboard featuring a search bar. Capturing the search request in Burp Suite let me isolate the `search` parameter, and injecting a single quote (`'`) triggered a query syntax error, confirming a SQL injection vulnerability.

![](/Users/shoham/Library/Application%20Support/marktext/images/2026-06-07-11-36-53-image.png)

![](/Users/shoham/Library/Application%20Support/marktext/images/2026-06-07-11-40-30-image.png)

Using the captured request in combination with sqlmap I was able to discover the databases of the web application.

```bash
sqlmap -r req.txt -dbs
```

![](/Users/shoham/Library/Application%20Support/marktext/images/2026-06-07-11-52-02-image.png)

Of the six databases returned, `recruit_db` stood out as the application's primary database – the natural next target, while `phpmyadmin` suggests a potential secondary avenue worth investigating.

### Step 5 — Database enumeration

With access to `recruit_db` database confirmed, I enumerated its tables to identify anything worthy of investigating further.

```bash
sqlmap -r req.txt -D recruit_db --tables
```

![](/Users/shoham/Library/Application%20Support/marktext/images/2026-06-07-12-11-25-image.png)

Among the tables returned, `users` stood out as likely to contain account data, making it the natural next target.

```bash
sqlmap -r req.txt -D recruit_db -T users --dump
```

![](/Users/shoham/Library/Application%20Support/marktext/images/2026-06-07-12-12-29-image.png)

### Step 6 — Privilege Escalation

This revealed admin account's username and password, stored in plain text, which I used to log in and capture the final flag.

![](/Users/shoham/Library/Application%20Support/marktext/images/2026-06-07-12-13-00-image.png)  

---

## Dead Ends

- Checked `robots.txt` for hidden paths - no such file was found.

---

## References

- [OWASP SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [sqlmap documentation](https://sqlmap.org)
- [PortSwigger SQL Injection Labs](https://portswigger.net/web-security/sql-injection)
