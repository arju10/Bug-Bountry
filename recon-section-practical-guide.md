# Recon Section - Practical Walkthrough
### Matches your WebApp_Pentest_Checklist.xlsx "Recon" tab, all 6 sections, 52 items

---

# SECTION 1: Open Source Reconnaissance (OSINT) - Items 1-10

**What this section is:** finding information about the target that's
already public, without touching their servers at all. 100% passive,
100% safe.

### Item 1 - Google Dorks
```
site:target.com filetype:pdf
site:target.com inurl:admin
site:target.com ext:log
intitle:"index of" site:target.com
```
**Practical note:** just type these into Google's search bar directly.
Flag results with words like "internal," "draft," "confidential,"
"backup," or "config" in the filename.

### Item 2 - Shodan / Censys
Go to shodan.io or censys.io, search the company's domain or IP range.

**Practical note:** these show devices/services already scanned by
Shodan's own bots - you're not scanning anything yourself. Look for
exposed databases, open admin panels, or old software versions listed
in the results.

### Item 3 - theHarvester
```bash
theHarvester -d target.com -b all
```
**Practical note:** pulls emails, subdomains, and names from search
engines automatically. Useful mainly for finding employee emails (for
understanding org structure) and extra subdomains you might have missed.

### Item 4 - LinkedIn / social recon
**Practical note:** manually search "target company" on LinkedIn, look
at job postings (they often reveal exact tech stack: "must know React,
AWS, PostgreSQL") and employee titles. Not a tool - just browsing.

### Item 5 - GitHub/GitLab leaked credentials
```bash
truffleHog github --repo=https://github.com/targetcompany/reponame
```
Or manually search on github.com:
```
"target.com" password
"target.com" api_key
org:targetcompany filename:.env
```
**Practical note:** this is one of the highest-value checks in the
entire recon phase. Developers accidentally commit API keys and
passwords to public repos constantly.

### Item 6 - Pastebin search
**Practical note:** manually search pastebin.com and similar sites for
the target's domain name. Sometimes leaked credentials or internal data
gets posted there after a breach elsewhere.

### Item 7 - WHOIS lookup
```bash
whois target.com
```
**Practical note:** shows who registered the domain, when, and which
nameservers. Mostly low-value on its own, but sometimes reveals a
related internal domain name or hosting provider.

### Item 8 - ASN lookup
Visit bgp.he.net or ipinfo.io, search the target's IP.

**Practical note:** shows the full range of IP addresses the company
owns. Useful for finding servers that don't have a friendly subdomain
name pointing to them at all.

### Item 9 - Wayback Machine
Visit: `web.archive.org/web/*/target.com/*`

**Practical note:** you already know this one - shows old/removed pages
and endpoints. One of the best, most commonly skipped checks.

### Item 10 - Certificate Transparency logs
Visit: `crt.sh/?q=target.com`

**Practical note:** you already know this too - shows every subdomain
that's ever had an SSL certificate issued, often reveals subdomains
`subfinder` misses.

---

# SECTION 2: Fingerprinting Web Server - Items 11-14

**What this section is:** figuring out exactly what software runs the
server, so you can look up known weaknesses for that exact
software/version.

### Item 11 - Web server type/version
```bash
curl -I https://target.com
whatweb https://target.com
```
**Practical note:** you already know this - look at the `Server` header
for the software name and version number.

### Item 12 - OS fingerprint
```bash
nmap -O target.com
```
**Practical note:** guesses the operating system based on how the
server responds to network packets. Lower priority - useful mainly for
very advanced network-level attacks, not typical bug bounty work.

### Item 13 - Specific version headers
```bash
curl -I https://target.com
```
Look specifically for: `X-Powered-By`, `X-Generator`, `X-AspNet-Version`

**Practical note:** same idea as Item 11, just naming the exact header
fields to check. You already have this in your header checklist.

### Item 14 - CDN/WAF detection
```bash
wafw00f https://target.com
```
**Practical note:** tells you if a Web Application Firewall (like
Cloudflare or Akamai) sits in front of the real server. This matters
because a WAF might block or alter your test payloads before they even
reach the real app - if one's detected, you may need gentler,
obfuscated payloads later on.

---

# SECTION 3: Metafiles & Exposed Files - Items 15-23

**What this section is:** checking specific, predictable file paths
that companies often forget to lock down. All safe, all just visiting
a URL directly.

### Item 15 - robots.txt
```bash
curl https://target.com/robots.txt
```
You already know this one.

### Item 16 - sitemap.xml
```bash
curl https://target.com/sitemap.xml
curl https://target.com/sitemap_index.xml
```
**Practical note:** lists every page the site wants search engines to
index - basically a free map of the whole site structure.

### Item 17 - humans.txt
```bash
curl https://target.com/humans.txt
```
**Practical note:** low priority, sometimes lists developer names -
mildly useful for OSINT (Item 4), not a security issue itself.

### Item 18 - security.txt
```bash
curl https://target.com/.well-known/security.txt
```
**Practical note:** shows the official security contact email for
reporting bugs - useful to check if the company even has a
bounty/disclosure process set up.

### Item 19 - crossdomain.xml
```bash
curl https://target.com/crossdomain.xml
```
**Practical note:** an old Flash-era file. Mostly irrelevant on modern
sites, but if present and misconfigured (allows `*` origin), flag it
same as an open CORS policy.

### Item 20 - Exposed .git directory
```bash
curl https://target.com/.git/HEAD
curl https://target.com/.git/config
```
**Practical note:** if this loads (instead of a 404), the site's entire
source code history might be downloadable using a tool like
`git-dumper`. This is a Critical-severity finding - full source code
exposure.

### Item 21 - Exposed .env files
```bash
curl https://target.com/.env
```
**Practical note:** `.env` files often contain database passwords, API
keys, and secret tokens directly in plain text. If this loads, it's an
immediate Critical finding.

### Item 22 - .DS_Store / Thumbs.db
```bash
curl https://target.com/.DS_Store
```
**Practical note:** these are OS-generated files (Mac/Windows) that
sometimes leak the exact folder/file structure of the server. Lower
severity, but still worth a quick check.

### Item 23 - Exposed backup files
Try common backup naming patterns directly in browser or via ffuf:
```
target.com/index.php.bak
target.com/config.old
target.com/database.sql.swp
```
**Practical note:** developers sometimes leave `.bak`/`.old` copies of
files right next to the live ones. If the live file is `config.php`,
try `config.php.bak` - sometimes it's downloadable as plain text
instead of being executed as code, exposing everything inside it.

---

# SECTION 4: Enumeration & Discovery - Items 24-34

**What this section is:** actively mapping out everything on the
target - ports, subdomains, hidden pages, and parameters. Mostly light
active recon, a few heavier tools flagged.

### Item 24 - Nmap full port scan
```bash
nmap -sV -sC -p- -T4 target.com
```
**Check scope first** - this is a heavier active scan.
**Practical note:** `-p-` scans all 65535 ports (slow but thorough),
`-sV` grabs version info per port, `-sC` runs default safe scripts.

### Item 25 - Nikto scan
```bash
nikto -h target.com
```
**Check scope first.**
**Practical note:** an older but still useful scanner that checks for
thousands of known misconfigurations and outdated software signatures
in one run.

### Item 26 - DNS records
```bash
dig target.com ANY
nslookup target.com
```
**Practical note:** shows mail servers (MX), name servers (NS), text
records (TXT - sometimes contain verification tokens or SPF/DMARC
info). Mostly informational, occasionally reveals a related internal
service.

### Item 27 - DNS Zone Transfer attempt
```bash
dig axfr @ns1.target.com target.com
```
**Practical note:** in a badly configured DNS server, this command can
dump the ENTIRE internal DNS record list (every subdomain, every
internal hostname) in one request. Almost always fails on
well-configured servers, but free to check - and a serious finding if
it works.

### Item 28 - Subdomain enumeration
```bash
subfinder -d target.com -o subs.txt
```
You already know this one well.

### Item 29 - Directory/file brute-force
```bash
ffuf -u https://target.com/FUZZ -w wordlist.txt
```
You already know this one.

### Item 30 - Security headers check
```bash
curl -I https://target.com
```
Or visit securityheaders.com and paste the URL.

**Practical note:** you already have the full header checklist for
this.

### Item 31 - Web crawling/spidering
In Burp Suite: right-click the target in the site map → "Spider this
host" (or just browse the whole site manually while Burp's proxy is
running, which achieves the same map).

**Practical note:** this just means "visit every page/link on the site
so you have a complete map," same as what you've been doing manually.

### Item 32 - JavaScript file analysis
```bash
# Download all JS files linked from the page, then search them:
grep -r "api" *.js
grep -r "endpoint" *.js
```
Or use a tool: **LinkFinder**
```bash
python3 linkfinder.py -i https://target.com/app.js -o cli
```
**Practical note:** you already know the manual version of this (View
Page Source → search .js files) - this just automates finding
hidden API paths inside JavaScript code.

### Item 33 - Parameter discovery
```bash
arjun -u https://target.com/api/search
```
**Practical note:** guesses hidden URL parameters the app accepts but
doesn't advertise anywhere (e.g. discovering `?debug=true` even though
it's not in any visible form).

### Item 34 - Virtual host enumeration
```bash
ffuf -u https://target.com -H 'Host: FUZZ.target.com' -w subdomains-wordlist.txt
```
**Practical note:** some servers host multiple different sites/apps
behind the same IP address, distinguished only by the `Host` header.
This finds "hidden" sites that don't show up in normal subdomain scans.

---

# SECTION 5: Web Application Framework Fingerprinting - Items 35-40

**What this section is:** narrowing down exactly what CMS/framework/
libraries power the site - all passive, mostly just looking closer at
what you already loaded in your browser.

### Item 35 - Wappalyzer / BuiltWith
Install the Wappalyzer browser extension, or visit builtwith.com and
paste the URL.

**Practical note:** gives you an instant summary box of every
technology detected - CMS, analytics, frameworks, hosting.

### Item 36 - whatweb CLI
```bash
whatweb target.com
```
Already covered - same tool, command-line version of Item 35.

### Item 37 - URL pattern/file extension analysis
**Practical note:** just look at the URLs as you browse. `.php` =
PHP backend, `.aspx`/`.do` = ASP.NET/Java. This alone tells you which
vulnerability classes are even possible (e.g. no point trying PHP-
specific tricks on a Java site).

### Item 38 - Page source generator tags
```bash
curl -s https://target.com | grep -i generator
```
Or Ctrl+U → Ctrl+F → search "generator"

**Practical note:** you already learned this - `<meta name="generator"
content="WordPress 5.2.1">` type tags reveal exact CMS versions.

### Item 39 - Cookie name analysis
Check cookies in Burp Suite or DevTools → Application tab.

**Practical note:** cookie names like `PHPSESSID`, `JSESSIONID`, or
`ASP.NET_SessionId` reveal the backend language even without checking
headers - each framework has a signature default cookie name.

### Item 40 - Framework-specific headers
```bash
curl -I https://target.com
```
**Practical note:** same header-checking skill as before, just looking
for less common framework-specific header names (e.g.
`X-Powered-By-Plesk`).

---

# SECTION 6: Configuration & Deployment Testing - Items 41-52

**What this section is:** this is your misconfiguration checklist,
expanded. Testing whether things were SET UP correctly, not whether the
code itself is flawed.

### Item 41 - Default credentials
```
admin/admin, admin/password, root/root, guest/guest
```
You already know this one.

### Item 42 - Enumerate admin interfaces
```bash
ffuf -u https://target.com/FUZZ -w admin-wordlist.txt
```
Try manually too: `/admin`, `/administrator`, `/wp-admin`, `/manager`,
`/phpmyadmin`

**Practical note:** same idea as directory brute-forcing, just focused
specifically on common admin panel paths.

### Item 43 - Test HTTP methods
```bash
curl -X OPTIONS https://target.com -i
```
**Practical note:** the response lists which HTTP methods the server
allows (GET, POST, PUT, DELETE, etc.). Methods beyond GET/POST being
open is worth flagging.

### Item 44 - HTTP TRACE method
```bash
curl -X TRACE https://target.com -i
```
**Practical note:** if TRACE is enabled, it can be abused (called
"Cross-Site Tracing") to read cookies that are otherwise protected by
HttpOnly, in combination with other bugs. Should normally be disabled.

### Item 45 - PUT method test
```bash
curl -X PUT -d 'test content' https://target.com/testfile.txt -i
```
**Practical note:** if this succeeds instead of being rejected, it may
mean you can upload arbitrary files directly to the server just by
sending a PUT request - a serious finding if true.

### Item 46 - Verbose error messages
Type something broken into any input field, or visit a broken URL.
**Practical note:** you already know this - look for stack traces, file
paths, or database errors in the response.

### Item 47 - 4xx/5xx error behavior
```bash
curl https://target.com/thispagedoesnotexist123 -i
```
**Practical note:** check if the 404 page itself leaks information
(software version, internal paths) beyond just "page not found."

### Item 48 - File extension execution test
**Practical note:** relevant during file upload testing (covered later
in the Business Logic tab) - checking that uploaded `.php`/`.asp`/`.jsp`
files don't get executed as code by the server.

### Item 49 - Directory listing
```bash
curl https://target.com/uploads/
```
You already know this one.

### Item 50 - File permission check
**Practical note:** hard to test directly from outside without another
bug already giving you file access (like LFI) - mostly relevant once
you find a separate vulnerability that lets you read files.

### Item 51 - Subdomain takeover
```bash
subjack -w subs.txt -t 100 -o results.txt -ssl
```
**Practical note:** happens when a subdomain's DNS record still points
to a third-party service (like an old Heroku/AWS/GitHub Pages app) that
was deleted - you can sometimes claim that service yourself and take
over the subdomain. Check any subdomain with a CNAME pointing to a
third-party service you can register on.

### Item 52 - NS record takeover
```bash
dig NS target.com
```
**Practical note:** rare, advanced case - if a nameserver domain itself
expired and is available for registration, you could register it and
control DNS for the target. Low priority for beginners, good to know
it exists.

---

# PRACTICAL ORDER TO ACTUALLY RUN SECTION 1-6

```
1. Section 1 (OSINT) first        - 100% passive, always safe, do first
2. Section 3 (Metafiles) next     - 100% passive, quick wins
3. Section 2 (Fingerprinting)     - light active
4. Section 5 (Framework ID)       - light active, mostly passive
5. Section 4 (Enumeration)        - active, get scope approval for
                                     heavier items (nmap full scan,
                                     nikto)
6. Section 6 (Misconfig)          - active, do last since it depends
                                     on knowing what admin panels/
                                     endpoints exist from earlier steps
```

Note down every test using the same "Tried X -> got Y" format you
already use, one line per item, moving through this checklist top to
bottom on each new target.
