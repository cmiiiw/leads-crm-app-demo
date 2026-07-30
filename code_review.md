## AI Security Review

## Code Review: Admin Endpoints (`server.ts`)

### Context
This change adds three new admin endpoints to [server.ts](file:///home/silaban_james13/leads-crm-app-demo/server.ts):
- `POST /api/admin/import`: Bulk imports leads.
- `GET /api/admin/export`: Exports leads to a CSV file and downloads it.
- `GET /api/admin/stats`: Returns summary statistics about leads and meetings.

---

### Findings by Severity

#### **Critical: Security Vulnerabilities**
1. **Plaintext & Hardcoded Password Authentication**
   - **File:** [server.ts](file:///home/silaban_james13/leads-crm-app-demo/server.ts#L231)
   - **Description:** `const ADMIN_PASSWORD = "admin123";` is hardcoded in source code and plaintext authentication is used via query parameters (`/api/admin/export?password=...`) or JSON body (`/api/admin/import`). Hardcoded credentials pose a high security risk, and passing passwords via `req.query` exposes them in server access logs, browser history, and proxy logs.
   - **Proposed Fix:** Use environment variables (e.g. `process.env.ADMIN_PASSWORD`) with secure hashing (e.g. bcrypt/argon2) or proper session-based/token-based authentication (JWT/OAuth), and ensure credentials are transmitted via request headers (e.g. `Authorization: Bearer ...` or `X-Admin-Password`).

2. **Path Traversal / Arbitrary File Write & Leak in Export**
   - **File:** [server.ts](file:///home/silaban_james13/leads-crm-app-demo/server.ts#L295-L318)
   - **Description:** The filename is taken directly from user input (`req.query.filename`), and the file is written to `path.join(__dirname, filename)` and subsequently downloaded via `res.download(filePath, filename)`. An attacker with admin access (or if auth is bypassed/leaked) could supply a malicious filename like `../../etc/passwd` or overwrite critical application files.
   - **Proposed Fix:** Sanitize the filename or disallow custom filenames from client query parameters, or use a fixed name / securely generate a temporary file name using `crypto.randomUUID()` within a designated temp directory.

3. **CSV Injection (Formula Injection)**
   - **File:** [server.ts](file:///home/silaban_james13/leads-crm-app-demo/server.ts#L302-L308)
   - **Description:** Database fields (`l.name`, `l.company`, `l.email`, `l.phone`, `l.notes`) are directly interpolated into the CSV output without sanitization. If any lead name or company starts with formula trigger characters (`=`, `+`, `-`, `@`), spreadsheet applications (Excel, LibreOffice Calc) will execute them when the CSV is opened.
   - **Proposed Fix:** Sanitize CSV fields by prefixing strings starting with `=`, `+`, `-`, or `@` with a single quote or wrapping/escaping properly.

---

#### **Required: Correctness & Robustness**
4. **Lack of Cleanup for Exported CSV Files on Disk**
   - **File:** [server.ts](file:///home/silaban_james13/leads-crm-app-demo/server.ts#L311)
   - **Description:** `fs.writeFileSync(filePath, headers + rows);` writes temporary export files directly into the application directory (`__dirname`), and they are never deleted. Over time, this will accumulate files or potentially leak sensitive lead data on the filesystem.
   - **Proposed Fix:** Use Node's `os.tmpdir()` or delete the file after sending via `res.download(filePath, filename, (err) => { fs.unlinkSync(filePath); })`.

5. **Unbounded Input in Bulk Import**
   - **File:** [server.ts](file:///home/silaban_james13/leads-crm-app-demo/server.ts#L233-L270)
   - **Description:** `POST /api/admin/import` accepts an unconstrained array of leads (`leads`) and executes a transaction over all of them. Passing an extremely large array could cause excessive memory consumption, database locking, or denial of service.
   - **Proposed Fix:** Implement pagination or slice the array into reasonable batches (e.g., max 1000 items per request).

---

#### **Nit / Optional**
6. **Verbose Debug Logging of Sensitive Data**
   - **File:** [server.ts](file:///home/silaban_james13/leads-crm-app-demo/server.ts#L261,L283,L313,L330)
   - **Description:** `console.debug` logs full lead objects, imported data, and exported records, which may log Personally Identifiable Information (PII) such as emails and phone numbers into application log streams.
   - **Proposed Fix:** Reduce logging to counts or non-sensitive identifiers instead of dumping full JSON objects containing PII.

---

### Verdict
- [ ] **Approve** — Ready to merge
- [x] **Request changes** — Critical security vulnerabilities (hardcoded password, path traversal via export filename, CSV injection) must be addressed before merge.
