# Bug Bounty Recon Cheat Sheet
### What to Actually Look For in Recon Output

---

## The Simple Mental Filter (use this first, always)

Ask two questions about anything you see:

1. **Does the name suggest this wasn't meant for the public?**
   (staging, internal, admin, backup...)
2. **Does the response suggest something real exists but is being
   guarded, broken, or misconfigured?**
   (401, 403, 500, error messages, old software versions)

If either answer is **yes** → flag it and dig deeper.
If both are **no** → boring, move on.

---

## 1. Subdomain Names (subfinder, crt.sh, amass, etc.)

**Always flag these words if you see them in a subdomain:**
```
staging, stage, stg
dev, development
test, testing, qa, uat
admin, administrator
internal, intranet
backup, bak, old
beta, alpha, preview
demo, sandbox
api-v1, api-old, legacy
vpn, remote
git, jenkins, jira, confluence, grafana, kibana
console, dashboard, portal
mail, webmail (mildly - worth a quick check)
```

**Ignore (normal, expected):**
```
www, blog, support, help, docs, status, cdn, static, assets, careers, jobs
```

---

## 2. HTTP Status Codes (httpx, ffuf, browser)

| Code | Meaning | Flag it? |
|------|---------|----------|
| 200  | Loaded fine | Only if name looks suspicious |
| 301/302 | Redirects elsewhere | Follow it, then judge the destination |
| 401  | Needs authentication | **Always flag** - something real exists |
| 403  | Forbidden/blocked | **Always flag** - something real exists |
| 404  | Doesn't exist | Ignore, dead end |
| 500  | Server error | **Always flag** - may leak info |
| 502/503 | Server down/overloaded | Note it, not usually a bug itself |

---

## 3. Tech Fingerprints (whatweb, Wappalyzer)

**Flag:**
- Old version numbers (jQuery 1.x, PHP 5.x, WordPress 4.x, outdated frameworks)
- Server headers revealing exact versions (Apache/2.2.15, nginx/1.10.0)
- Any CMS or plugin mentioned (WordPress plugins = frequent bug source)
- Debug mode indicators (X-Debug headers, "DEBUG=True" responses)

**Ignore:**
- Modern, current version numbers
- Standard cloud provider info (AWS CloudFront, Cloudflare) - expected everywhere

---

## 4. Google Dorking Results

**Flag file/page names containing:**
```
internal, draft, confidential, private, backup, old, deprecated,
config, .env, credentials, secret, password, key, api_key, staging
```

**Flag file types that shouldn't be public:**
```
.env, .sql, .log, .bak, .config, .git, .zip, .tar, id_rsa (SSH keys)
```

---

## 5. Error Messages / Server Responses

**Flag if you see:**
- Stack traces (raw code shown in the error page)
- File paths revealed (e.g. /var/www/html/app/config.php)
- Database error messages (SQL, MySQL, Postgres syntax visible)
- "Debug" or "Development" mode banners
- Internal IP addresses mentioned

---

## 6. JavaScript Files (view page source, check .js files)

**Flag if you find:**
- Hardcoded API keys or tokens
- Hidden/undocumented API endpoints referenced in code
- Internal URLs or subdomain names mentioned
- Developer comments (// TODO, // FIXME, // remove before prod)

---

## 7. robots.txt and sitemap.xml

Visit `website.com/robots.txt` and `website.com/sitemap.xml` directly in
a browser - plain text, always safe to check.

**Flag any path mentioned like:**
```
/admin/
/internal-api/
/old-dashboard/
/backup-files/
```

These are pages the company itself doesn't want indexed - written by them,
handed to you for free.

---

## 8. HTTP Response Headers (beyond just status code)

**Flag:**
- Missing security headers (no Content-Security-Policy, no X-Frame-Options)
  - signals a less mature security setup, not a bug by itself
- `Access-Control-Allow-Origin: *`
  - means ANY website can call this API - often a real finding
    (CORS misconfiguration)
- Server header revealing exact software+version (Apache/2.4.6, IIS/8.5)
- `Set-Cookie` without `Secure` or `HttpOnly` flags - session security issue

---

## 9. Cloud Storage Buckets (S3, Google Cloud, Azure)

Try guessing/searching for predictable bucket names:
```
companyname-uploads
companyname-backup
companyname-assets
companyname-dev
```

Try visiting directly: `https://companyname-backup.s3.amazonaws.com`

**Flag if you get a file listing instead of "Access Denied."** Misconfigured
public buckets are a common, very real bug category.

---

## 10. Wayback Machine (archive.org) - Old Snapshots

Search: `web.archive.org/web/*/targetdomain.com/*`

Shows every URL archive.org has ever seen for that domain, including pages
that no longer exist on the live site.

**Flag:**
- Old admin panels that used to exist
- Old API endpoints no longer linked from the current site, but maybe
  still technically working

One of the highest-value, most commonly skipped passive recon steps.

---

## 11. Favicon Hash Matching (advanced - optional for now)

Fingerprinting a site by its favicon (browser tab icon) to find other
servers/panels run by the same company that don't show up in normal
subdomain scans. Not needed as a beginner - just know it exists.

---

## 12. Full Header Checklist (how to see them: F12 -> Network tab ->
reload page -> click first request -> Response Headers)

| Header | What it tells you | What to flag |
|--------|-------------------|--------------|
| `Server` | Web server software/version | Old version numbers |
| `X-Powered-By` | Backend framework/language | Old version numbers |
| `Access-Control-Allow-Origin` | Who can call this API | A `*` value (means "anyone") |
| `Set-Cookie` | How session cookies are set up | Missing `HttpOnly` or `Secure` flags |
| `Content-Security-Policy` | Restricts what scripts can load | Header missing entirely |
| `X-Frame-Options` | Blocks page from loading in an iframe | Header missing entirely |
| `Strict-Transport-Security` | Forces HTTPS only | Header missing entirely |
| `X-Content-Type-Options` | Stops browser mis-guessing file types | Header missing entirely |
| `WWW-Authenticate` | Shows auth method on 401 responses | Reveals old/basic auth method |
| `Location` | Shows redirect destination | Reveals an internal/unexpected URL |

**If this list feels like too much, check these 3 first:**
1. `Access-Control-Allow-Origin` - look for `*`
2. `Server` / `X-Powered-By` - look for old version numbers
3. `Set-Cookie` - look for missing `HttpOnly`/`Secure`

---

## 13. Misconfiguration Checklist (5 simple checks, try in order)

**Check 1 - Default password**
Find any login page. Try:
```
Username: admin   Password: admin
Username: admin   Password: password
```

**Check 2 - Directory listing**
Visit any folder-style URL directly (e.g. `site.com/uploads/`).
- Normal page/error -> nothing found
- Plain clickable list of files -> found something

**Check 3 - Open CORS**
F12 -> Network tab -> reload -> click an `/api/` request -> Response
Headers -> search for `access-control-allow-origin`.
- Value is `*` -> found something
- Value is a specific domain, or missing -> nothing found

**Check 4 - Public cloud storage bucket**
Try these directly in the browser:
```
https://companyname-backup.s3.amazonaws.com
https://companyname-uploads.s3.amazonaws.com
https://companyname-files.s3.amazonaws.com
```
- "Access Denied" -> nothing found
- A list of real files -> found something

**Check 5 - Forced error message**
Type something broken into any input box, e.g. `'` or `///???`.
- Generic "invalid input" message -> nothing found
- Detailed technical error with file paths/code -> found something

**The pattern behind all 5 checks:**
```
1. Try one specific, simple thing
2. Look at what happens
3. Decide: normal behavior (nothing found) OR unexpected behavior (found something)
```

---

## Order to Actually Run Things

```
1. Google Dorking + crt.sh + subfinder     (passive, always safe, do first)
2. robots.txt + sitemap.xml + Wayback Machine  (passive, always safe)
3. httpx                                    (active but light - alive check)
4. whatweb / Wappalyzer                     (active, light - tech fingerprint)
5. ffuf / gobuster                          (active, heavier - only on
                                              flagged targets, check scope)
6. nuclei                                   (active, heaviest - only if
                                              program explicitly allows
                                              automated scanning)
```

You don't run every tool on every subdomain. Narrow down at each step -
only dig deeper (whatweb/ffuf/nuclei) on what already looked interesting
by name or status code.

---

## Notes Format Reminder

One line per test, the moment you try it:
```
Tried X -> got Y
```
Example:
```
Ran subfinder on taskflowhq.com, got 40 subdomains, saved to subs.txt
staging.taskflowhq.com and dev-api.taskflowhq.com flagged - test first
Checked robots.txt - found /internal-reports/ path, not seen before
```
