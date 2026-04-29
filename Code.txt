/**
 * DomainFront Relay — Google Apps Script  [Optimized Build]
 *
 * Based on MasterHttpRelayVPN by @masterking32
 * Rust port: github.com/therealaleph/MasterHttpRelayVPN-RUST
 *
 * TWO modes:
 *   1. Single:  POST { k, m, u, h, b, ct, r }       → { s, h, b }
 *   2. Batch:   POST { k, q: [{m,u,h,b,ct,r}, ...] } → { q: [{s,h,b}, ...] }
 *      Uses UrlFetchApp.fetchAll() — all requests IN PARALLEL.
 *
 * DEPLOYMENT STEPS:
 *   1. Go to https://script.google.com → New project
 *   2. Delete the default code, paste THIS entire file
 *   3. Set AUTH_KEY below to a strong random secret (20+ chars, letters+digits)
 *   4. File → Save  (name it "mhrv-relay" or anything you like)
 *   5. Deploy → New deployment → gear icon → Web app
 *      Execute as: Me  |  Who has access: Anyone
 *   6. Click Deploy → copy the /exec URL into mhrv-rs (one URL per line)
 *
 * !! CHANGE AUTH_KEY BEFORE DEPLOYING !!
 *
 * TIP: Deploy 2–3 copies under different Google accounts and paste all
 *      /exec URLs into mhrv-rs (one per line). The client round-robins,
 *      so you stay well under the 20k/day per-script quota.
 */

// ═══════════════════════════════════════════════════════════════
//  CONFIGURATION  — edit this section only
// ═══════════════════════════════════════════════════════════════

/**
 * Your secret password.  Must match the AUTH_KEY you set in mhrv-rs.
 * Use at least 20 random characters (letters + digits).
 * Example generator: https://www.random.org/strings/
 */
const AUTH_KEY = "CHANGE_ME_TO_A_STRONG_SECRET_MIN_20_CHARS";

/**
 * DIAGNOSTIC_MODE
 *   false (default) → bad-auth and errors return a decoy HTML page.
 *                     Active scanners see a boring static site and move on.
 *   true            → errors return JSON { e: "..." } so you can debug setup.
 *
 * Set to true while configuring, then flip back to false before sharing the URL.
 */
const DIAGNOSTIC_MODE = false;

// ═══════════════════════════════════════════════════════════════
//  TUNABLES  — safe to leave at defaults
// ═══════════════════════════════════════════════════════════════

/** Hard cap on batch size. Apps Script free tier: 6 min execution limit. */
const MAX_BATCH = 20;

/** URL length guard (characters). */
const MAX_URL_LENGTH = 2048;

// ═══════════════════════════════════════════════════════════════
//  HEADER FILTERS
// ═══════════════════════════════════════════════════════════════

/**
 * Request headers NOT forwarded to the target server.
 * Keeps proxy-layer headers out and lets UrlFetchApp manage connection state.
 */
const SKIP_REQ_HEADERS = {
  host:                  1,
  connection:            1,
  "content-length":      1,
  "transfer-encoding":   1,
  "proxy-connection":    1,
  "proxy-authorization": 1,
  priority:              1,
  te:                    1,
  expect:                1,
  // We set accept-encoding ourselves so UrlFetchApp can decompress
  "accept-encoding":     1,
};

/**
 * Response headers NOT forwarded back to the client.
 * Strips hop-by-hop headers that are meaningless after relay.
 */
const SKIP_RESP_HEADERS = {
  connection:          1,
  "keep-alive":        1,
  "transfer-encoding": 1,
  "proxy-authenticate":1,
  "proxy-authorization":1,
  te:                  1,
  trailer:             1,
  upgrade:             1,
};

// ═══════════════════════════════════════════════════════════════
//  DECOY PAGE  — shown to scanners / bad-auth requests
// ═══════════════════════════════════════════════════════════════
const DECOY_HTML = [
  "<!DOCTYPE html>",
  '<html lang="en">',
  "<head>",
  '  <meta charset="UTF-8">',
  '  <meta name="viewport" content="width=device-width,initial-scale=1">',
  "  <title>Personal Notes</title>",
  "  <style>",
  "    body{font-family:Georgia,serif;max-width:660px;margin:60px auto;",
  "         padding:0 24px;color:#333;line-height:1.75}",
  "    h1{font-size:1.7rem;border-bottom:1px solid #e0e0e0;padding-bottom:.4rem}",
  "    .meta{color:#aaa;font-size:.85rem;margin-top:-.5rem}",
  "    a{color:#0066cc;text-decoration:none}",
  "    footer{margin-top:3rem;padding-top:1rem;border-top:1px solid #eee;",
  "           font-size:.8rem;color:#bbb}",
  "  </style>",
  "</head>",
  "<body>",
  "  <h1>Notes &amp; Links</h1>",
  '  <p class="meta">Last updated — February 2024</p>',
  "  <p>This is a personal Apps Script project used for internal tooling.",
  "     There is nothing public here.</p>",
  "  <p>If you arrived here by mistake, please check the URL you were given.</p>",
  "  <footer>",
  "    <p>Hosted on Google Apps Script &middot; Not affiliated with any product</p>",
  "  </footer>",
  "</body>",
  "</html>"
].join("\n");

// ═══════════════════════════════════════════════════════════════
//  ENTRY POINTS
// ═══════════════════════════════════════════════════════════════

/** Handles all proxy traffic (mhrv-rs POSTs here). */
function doPost(e) {
  try {
    var body = e.postData && e.postData.contents;
    if (!body) return _decoyOrDiag("no body");

    var req;
    try { req = JSON.parse(body); }
    catch (_) { return _decoyOrDiag("json parse error"); }

    // Auth check — constant-time comparison to resist timing side-channel
    if (!_authOk(req.k)) return _decoy();

    // Dispatch
    if (Array.isArray(req.q)) return _doBatch(req.q);
    return _doSingle(req);

  } catch (err) {
    return _decoyOrDiag("internal: " + _errMsg(err));
  }
}

/** Handles browser GET visits — returns the decoy page. */
function doGet(_e) {
  return HtmlService.createHtmlOutput(DECOY_HTML);
}

// ═══════════════════════════════════════════════════════════════
//  AUTH
// ═══════════════════════════════════════════════════════════════

function _authOk(key) {
  if (typeof key !== "string") return false;
  // Length check first (avoids iterating different-length strings)
  if (key.length !== AUTH_KEY.length) return false;
  // Character-by-character without early exit (timing-safe in V8 JIT)
  var ok = true;
  for (var i = 0; i < AUTH_KEY.length; i++) {
    if (key.charCodeAt(i) !== AUTH_KEY.charCodeAt(i)) ok = false;
  }
  return ok;
}

// ═══════════════════════════════════════════════════════════════
//  SINGLE REQUEST
// ═══════════════════════════════════════════════════════════════

function _doSingle(req) {
  var err = _checkUrl(req.u);
  if (err) return _json({ e: err });

  try {
    var resp = UrlFetchApp.fetch(req.u, _buildOpts(req));
    return _json(_packResp(resp));
  } catch (ex) {
    return _json({ e: "fetch: " + _errMsg(ex) });
  }
}

// ═══════════════════════════════════════════════════════════════
//  BATCH REQUEST
// ═══════════════════════════════════════════════════════════════

function _doBatch(items) {
  if (!items.length) return _json({ q: [] });

  // Safety cap
  if (items.length > MAX_BATCH) items = items.slice(0, MAX_BATCH);

  var fetchList  = [];   // UrlFetchApp option objects for valid items
  var indexMap   = [];   // fetchList[i] corresponds to items[indexMap[i]]
  var results    = new Array(items.length);

  // Validate & build options
  for (var i = 0; i < items.length; i++) {
    var uErr = _checkUrl(items[i].u);
    if (uErr) {
      results[i] = { e: uErr };
    } else {
      var o = _buildOpts(items[i]);
      o.url = items[i].u;
      indexMap.push(i);
      fetchList.push(o);
    }
  }

  // Fire all valid requests in parallel
  if (fetchList.length > 0) {
    var responses;
    try {
      responses = UrlFetchApp.fetchAll(fetchList);
    } catch (ex) {
      // fetchAll itself failed — mark remaining slots as errors
      for (var fi = 0; fi < indexMap.length; fi++) {
        results[indexMap[fi]] = { e: "fetchAll: " + _errMsg(ex) };
      }
      return _json({ q: results });
    }

    for (var ri = 0; ri < responses.length; ri++) {
      var slot = indexMap[ri];
      try {
        results[slot] = _packResp(responses[ri]);
      } catch (ex2) {
        results[slot] = { e: "pack: " + _errMsg(ex2) };
      }
    }
  }

  return _json({ q: results });
}

// ═══════════════════════════════════════════════════════════════
//  SHARED HELPERS
// ═══════════════════════════════════════════════════════════════

/**
 * Validate a URL before forwarding.
 * Returns null if OK, or an error string if rejected.
 */
function _checkUrl(u) {
  if (!u || typeof u !== "string")          return "missing url";
  if (u.length > MAX_URL_LENGTH)            return "url too long";
  if (!u.match(/^https?:\/\//i))            return "bad url scheme";
  // SSRF guard: block loopback & RFC-1918 ranges
  if (u.match(/^https?:\/\/(localhost|127\.\d+\.\d+\.\d+|0\.0\.0\.0|::1|10\.\d+\.\d+\.\d+|192\.168\.\d+\.\d+|172\.(1[6-9]|2\d|3[01])\.\d+\.\d+)([:\/]|$)/i)) {
    return "private address blocked";
  }
  return null;
}

/**
 * Build UrlFetchApp options from a relay request object.
 * Adds accept-encoding so UrlFetchApp (which auto-decompresses) benefits
 * from smaller wire transfers without any extra work on our end.
 */
function _buildOpts(req) {
  var opts = {
    method:                   (req.m || "GET").toLowerCase(),
    muteHttpExceptions:       true,    // don't throw on 4xx/5xx
    followRedirects:          req.r !== false,
    validateHttpsCertificates:true,
    escaping:                 false,   // don't percent-encode the URL again
  };

  // Merge forwarded headers, adding gzip hint for faster target fetches
  var headers = { "accept-encoding": "gzip, deflate, br" };
  if (req.h && typeof req.h === "object") {
    for (var k in req.h) {
      if (!req.h.hasOwnProperty(k)) continue;
      if (!SKIP_REQ_HEADERS[k.toLowerCase()]) headers[k] = req.h[k];
    }
  }
  opts.headers = headers;

  // Body
  if (req.b) {
    try   { opts.payload = Utilities.base64Decode(req.b); }
    catch (_) { /* malformed base64 — send empty body */ }
    if (req.ct) opts.contentType = req.ct;
  }

  return opts;
}

/**
 * Pack an HTTPResponse into our wire format { s, h, b }.
 * Strips hop-by-hop response headers before forwarding.
 */
function _packResp(resp) {
  var rawHeaders = {};
  try {
    rawHeaders = (typeof resp.getAllHeaders === "function")
      ? resp.getAllHeaders()
      : resp.getHeaders();
  } catch (_) { /* ignore */ }

  var outHeaders = {};
  for (var k in rawHeaders) {
    if (rawHeaders.hasOwnProperty(k) && !SKIP_RESP_HEADERS[k.toLowerCase()]) {
      outHeaders[k] = rawHeaders[k];
    }
  }

  return {
    s: resp.getResponseCode(),
    h: outHeaders,
    b: Utilities.base64Encode(resp.getContent()),
  };
}

// ─── Output helpers ────────────────────────────────────────────

function _json(obj) {
  return ContentService
    .createTextOutput(JSON.stringify(obj))
    .setMimeType(ContentService.MimeType.JSON);
}

function _decoy() {
  return HtmlService.createHtmlOutput(DECOY_HTML);
}

/** In DIAGNOSTIC_MODE return JSON error; otherwise return the decoy page. */
function _decoyOrDiag(msg) {
  return DIAGNOSTIC_MODE ? _json({ e: msg }) : _decoy();
}

/** Return error message only if DIAGNOSTIC_MODE is on. */
function _errMsg(err) {
  return DIAGNOSTIC_MODE ? String(err) : "error";
}
