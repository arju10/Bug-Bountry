# Bug Bounty Full Reference Guide
### Step by Step - From Note-Taking to Reporting

---

# PART 1: THE CORE IDEA

Bug bounty hunting = finding a mistake a website's programmer made, and
telling the company about it (nicely, with proof) so you get paid.

Everything below is just learning to spot common mistakes, in an
organized, repeatable way.

---

# PART 2: HOW TO TAKE NOTES (do this from day one)

**The problem it solves:** stopping to write a "proper" note breaks your
focus, so most people skip it and forget what they tried an hour later.

**The fix:** one plain text file (Notes app, Google Doc, whatever's
fastest to open). One line per test. Write it the SECOND you try
something - not "later."

**Format:**
```
Tried X -> got Y
```

**Examples:**
```
Tried changing profile/123 to profile/124 - got my own data again, no bug
Tried the export link with a different account - IT WORKED, saw someone else's data!
Ran subfinder on taskflowhq.com, got 40 subdomains, saved to subs.txt
staging.taskflowhq.com flagged - test first
Checked robots.txt - found /internal-reports/ path, not seen before
```

Most lines will say "nothing found" - that's normal. The goal is to
never repeat a test you already tried, and to never lose the one line
that actually mattered.

When a test turns into a real finding, THEN you move it into a proper
report (see Part 8).

---

# PART 3: THE FULL METHODOLOGY (the 10-step loop, every target)

1. **Pick a target** - read the program's scope and rules carefully
   (what domains are allowed, what testing is/isn't allowed)
2. **Recon** - find everything that exists (subdomains, endpoints, files)
3. **Explore like a normal user** - make an account, click every feature
   once, capture traffic with Burp Suite while you do
4. **Map the requests** - look at what Burp captured; note anything with
   an ID number, file upload, admin action, or API call
5. **Create a second account** - essential for comparing what Account A
   can do vs what Account B can do
6. **Test the boundaries** - swap tokens between accounts, change IDs in
   URLs, try actions you shouldn't have permission for
7. **Note every test** - one line, immediately (Part 2)
8. **Confirm anything that breaks** - repeat it 2-3 times, screenshot it,
   figure out how bad it is (just your data? anyone's? can you change
   things, not just view them?)
9. **Write the report** - what's the bug, exact steps, why it matters,
   proof
10. **Submit it** - through the platform's report form, then wait

---

# PART 4: RECON - WHAT TO LOOK FOR

## The Simple Mental Filter (use first, always)

Ask two questions about anything you see:
1. Does the name suggest this wasn't meant for the public? (staging,
   internal, admin, backup...)
2. Does the response suggest something real exists but is being guarded,
   broken, or misconfigured? (401, 403, 500, error messages, old
   software versions)

If either answer is yes -> flag it and dig deeper.
If both are no -> boring, move on.

## 1. Subdomain Names

**Always flag:**
```
staging, stage, stg, dev, development, test, testing, qa, uat,
admin, administrator, internal, intranet, backup, bak, old,
beta, alpha, preview, demo, sandbox, api-v1, api-old, legacy,
vpn, remote, git, jenkins, jira, confluence, grafana, kibana,
console, dashboard, portal, mail, webmail
```

**Ignore (normal):**
```
www, blog, support, help, docs, status, cdn, static, assets, careers, jobs
```

## 2. HTTP Status Codes

| Code | Meaning | Flag it? |
|------|---------|----------|
| 200 | Loaded fine | Only if name looks suspicious |
| 301/302 | Redirects elsewhere | Follow it, judge the destination |
| 401 | Needs authentication | Always flag - something real exists |
| 403 | Forbidden/blocked | Always flag - something real exists |
| 404 | Doesn't exist | Ignore, dead end |
| 500 | Server error | Always flag - may leak info |
| 502/503 | Server down/overloaded | Note it, not usually a bug itself |

## 3. Tech Fingerprints (whatweb, Wappalyzer)

**Flag:** old version numbers (jQuery 1.x, PHP 5.x, WordPress 4.x),
server headers with exact versions, any CMS/plugin mentioned, debug mode
indicators.

**Ignore:** modern version numbers, standard cloud provider info (AWS
CloudFront, Cloudflare).

## 4. Google Dorking

**Flag names containing:**
```
internal, draft, confidential, private, backup, old, deprecated,
config, .env, credentials, secret, password, key, api_key, staging
```

**Flag file types:**
```
.env, .sql, .log, .bak, .config, .git, .zip, .tar, id_rsa
```

## 5. Error Messages / Server Responses

**Flag:** stack traces, file paths revealed, database error messages,
"debug/development" banners, internal IP addresses.

## 6. JavaScript Files (view page source, check .js files)

**Flag:** hardcoded API keys/tokens, hidden API endpoints referenced in
code, internal URLs mentioned, developer comments (// TODO, // FIXME).

## 7. robots.txt and sitemap.xml

Visit `website.com/robots.txt` and `website.com/sitemap.xml` directly -
always safe, plain text. Flag any path like `/admin/`, `/internal-api/`,
`/old-dashboard/`, `/backup-files/`.

## 8. HTTP Response Headers - see Part 6 for the full list

## 9. Cloud Storage Buckets

Try: `https://companyname-backup.s3.amazonaws.com`,
`-uploads`, `-assets`, `-dev`, `-files`.
Flag if you get a file listing instead of "Access Denied."

## 10. Wayback Machine (archive.org)

Search `web.archive.org/web/*/targetdomain.com/*`. Flag old admin panels
or old API endpoints that might still technically work.

## 11. Favicon Hash Matching (advanced, optional for later)

Fingerprints a site by its favicon to find other servers run by the same
company. Not needed as a beginner.

---

# PART 5: TOOLS - INSTALL, RUN, AND READ OUTPUT

Setup note (Ubuntu, after installing Go):
```bash
echo 'export PATH=$PATH:$(go env GOPATH)/bin' >> ~/.bashrc
source ~/.bashrc
```

## subfinder (finds subdomains) - PASSIVE
```bash
go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest
subfinder -d taskflowhq.com -o subs.txt
```
Output: plain list of subdomains, one per line.

## httpx (checks which are alive) - ACTIVE (light)
```bash
go install -v github.com/projectdiscovery/httpx/cmd/httpx@latest
httpx -l subs.txt -o alive.txt -status-code -title
```
Output: `https://app.taskflowhq.com [200] [TaskFlow - Project Management]`

## whatweb (tech fingerprinting) - ACTIVE (light)
```bash
sudo apt install whatweb
whatweb https://app.taskflowhq.com
```
Output: `Server[nginx/1.18.0], JQuery[1.8.2], X-Powered-By[Express]`

## ffuf (finds hidden folders) - ACTIVE (heavier, check scope first)
```bash
go install github.com/ffuf/ffuf/v2@latest
ffuf -u https://staging.taskflowhq.com/FUZZ -w /path/to/wordlist.txt
```
Output: `admin [Status: 403]`, `backup [Status: 200]`, `test [Status: 404]`
(ignore 404s, flag 200/403)

## nuclei (automated vuln scanner) - ACTIVE (heaviest, check program rules)
```bash
go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest
nuclei -u https://staging.taskflowhq.com
```
Output: `[exposed-panel] [medium] https://staging.taskflowhq.com/admin`

## Order to run tools

```
1. Google Dorking + crt.sh + subfinder          (passive, always safe)
2. robots.txt + sitemap.xml + Wayback Machine   (passive, always safe)
3. httpx                                        (active, light)
4. whatweb / Wappalyzer                         (active, light)
5. ffuf / gobuster                              (active, heavier - only
                                                  on flagged targets)
6. nuclei                                       (active, heaviest - only
                                                  if program allows it)
```
Narrow down at each step - don't run every tool on every subdomain.

---

# PART 6: FULL HEADER CHECKLIST

How to see them: F12 -> Network tab -> reload page -> click first
request -> Response Headers.

| Header | What it tells you | What to flag |
|--------|-------------------|---------------|
| `Server` | Web server software/version | Old version numbers |
| `X-Powered-By` | Backend framework/language | Old version numbers |
| `Access-Control-Allow-Origin` | Who can call this API | A `*` value |
| `Set-Cookie` | Session cookie setup | Missing `HttpOnly`/`Secure` |
| `Content-Security-Policy` | Restricts script loading | Missing entirely |
| `X-Frame-Options` | Blocks iframe loading | Missing entirely |
| `Strict-Transport-Security` | Forces HTTPS | Missing entirely |
| `X-Content-Type-Options` | Stops file-type guessing | Missing entirely |
| `WWW-Authenticate` | Auth method on 401s | Reveals old/basic auth |
| `Location` | Redirect destination | Reveals internal/unexpected URL |

**If short on time, check these 3 first:**
1. `Access-Control-Allow-Origin` - look for `*`
2. `Server` / `X-Powered-By` - look for old versions
3. `Set-Cookie` - look for missing `HttpOnly`/`Secure`

---

# PART 7: OLD VERSION -> REAL BUG (the full chain)

1. Find the exact software name + version (headers, whatweb, page
   source, error pages, CMS files like `/readme.html`)
2. Search: `"[software] [version]" vulnerability CVE`
3. Find the SPECIFIC known bug and its trigger/PoC (proof of concept)
4. Find WHERE on the real site that vulnerable function is actually used
   with user input (view source, search for the function name)
5. Test the exact trigger there yourself
6. If it works -> screenshot immediately
7. Confirm it's not a fluke (repeat it, check if it affects other users
   too - "stored" bugs are more serious than one-off)
8. Write the report connecting all the pieces (see Part 8)

**Important:** an old version number is a CLUE, not a bug. Reporting
"you're using an old version" alone usually gets rejected. You must
prove the specific vulnerability actually works on their site.

---

# PART 8: MISCONFIGURATION CHECKLIST (5 checks, try in order)

**Check 1 - Default password**
```
Username: admin   Password: admin
Username: admin   Password: password
```

**Check 2 - Directory listing**
Visit a folder-style URL directly (`site.com/uploads/`).
- Normal page/error -> nothing found
- Plain clickable file list -> found something

**Check 3 - Open CORS**
F12 -> Network -> reload -> click an `/api/` request -> Response Headers
-> search `access-control-allow-origin`.
- `*` -> found something
- Specific domain or missing -> nothing found

**Check 4 - Public cloud bucket**
```
https://companyname-backup.s3.amazonaws.com
https://companyname-uploads.s3.amazonaws.com
```
- "Access Denied" -> nothing found
- List of real files -> found something

**Check 5 - Forced error message**
Type `'` or `///???` into any input box.
- Generic "invalid input" -> nothing found
- Detailed technical error with file paths/code -> found something

**The pattern behind all 5:** try one simple thing -> look at what
happens -> decide normal vs unexpected.

---

# PART 9: WRITING THE REPORT

Template:
```markdown
# Title: [short, specific description of the bug]

## Summary
[1-2 sentences: what's wrong and why]

## Steps to Reproduce
1. ...
2. ...
3. ...

## Impact
[what an attacker could actually do with this]

## Suggested Fix
[optional, but appreciated]

## Proof of Concept
[screenshot(s) attached]
```

Real example (IDOR):
```markdown
# Title: IDOR on /api/v1/export/:reportId allows access to any user's
project reports

## Summary
The export endpoint does not verify that the requesting user has access
to the project associated with the requested reportId.

## Steps to Reproduce
1. Log in as User A, create a project, generate a report (note reportId)
2. Log in as User B (unrelated account)
3. Send GET /api/v1/export/{User A's reportId} using User B's token
4. Observe 200 OK with User A's full report contents

## Impact
Confidential project data exposed to any authenticated user. reportId
appears sequential, allowing mass enumeration.

## Suggested Fix
Verify server-side that the user is a member of the project tied to
reportId before returning data.

## Proof of Concept
[screenshots attached]
```

---

# PART 10: THE VULN-PATTERN HABIT (compounding value)

Keep ONE running file where, every time you learn a new pattern, you add
one entry - not per target, across ALL targets:

```
- Export/report endpoints are a recurring IDOR source. Devs treat
  "export" as a side feature and forget the same ownership checks
  applied to the main resource. Always test export URLs with a second,
  unrelated account first.
```

Months later, seeing a similar endpoint on a totally different target
will jog this memory instantly - this is what separates hunters who get
faster over time from those who start from zero every time.

---

# QUICK REFERENCE: THE WHOLE LOOP, COMPRESSED

```
Look around -> Map what exists -> See how normal use works ->
Try to break the rules -> If it breaks, document it -> Report it
```

Two questions to ask about anything you find during recon:
```
1. Does the name suggest this wasn't meant for the public?
2. Does the response suggest something guarded, broken, or misconfigured?
```
