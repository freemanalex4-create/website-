/**
 * Educational HTTP Proxy Server
 * ------------------------------
 * Demonstrates the core mechanics of a forward proxy:
 *   1. Receiving a client request
 *   2. Forwarding it to the target server
 *   3. Relaying headers correctly
 *   4. Caching responses to reduce repeat requests
 *   5. Logging traffic for inspection
 *
 * This is a teaching tool. It is intentionally simple and only
 * intended to be run against open, non-restricted sites for a
 * class demonstration of how proxies work at the protocol level.
 */

const express = require("express");
const fetch = require("node-fetch");

const app = express();
const PORT = 3000;

// --- Simple in-memory cache ---------------------------------------------
// Maps a URL to { body, contentType, timestamp }
const cache = new Map();
const CACHE_TTL_MS = 60 * 1000; // 1 minute

function getFromCache(url) {
  const entry = cache.get(url);
  if (!entry) return null;
  const isExpired = Date.now() - entry.timestamp > CACHE_TTL_MS;
  if (isExpired) {
    cache.delete(url);
    return null;
  }
  return entry;
}

function saveToCache(url, body, contentType) {
  cache.set(url, { body, contentType, timestamp: Date.now() });
}

// --- Logging middleware ---------------------------------------------------
app.use((req, res, next) => {
  const time = new Date().toISOString();
  console.log(`[${time}] ${req.method} ${req.url}`);
  next();
});

// --- Homepage: simple form to enter a URL to proxy ------------------------
app.get("/", (req, res) => {
  res.send(`
    <!DOCTYPE html>
    <html>
      <head><title>Proxy Demo</title></head>
      <body style="font-family: sans-serif; max-width: 600px; margin: 40px auto;">
        <h1>Educational Proxy Server</h1>
        <p>Enter a full URL (must start with http:// or https://) to fetch it through the proxy.</p>
        <form action="/fetch" method="get">
          <input type="text" name="url" placeholder="https://example.com" style="width: 400px;" />
          <button type="submit">Fetch</button>
        </form>
        <p><small>Cache entries: ${cache.size}</small></p>
      </body>
    </html>
  `);
});

// --- Core proxy route -------------------------------------------------------
app.get("/fetch", async (req, res) => {
  const targetUrl = req.query.url;

  if (!targetUrl || !/^https?:\/\//i.test(targetUrl)) {
    return res.status(400).send("Please provide a valid URL starting with http:// or https://");
  }

  // 1. Check cache first
  const cached = getFromCache(targetUrl);
  if (cached) {
    console.log(`  -> served from cache`);
    res.set("Content-Type", cached.contentType);
    res.set("X-Proxy-Cache", "HIT");
    return res.send(cached.body);
  }

  // 2. Forward the request to the real target
  try {
    const upstreamResponse = await fetch(targetUrl, {
      headers: {
        // Forward a normal-looking User-Agent; strip anything
        // that would identify the original client unnecessarily.
        "User-Agent": "EducationalProxyDemo/1.0",
      },
      redirect: "follow",
    });

    const contentType = upstreamResponse.headers.get("content-type") || "text/plain";
    const body = await upstreamResponse.text();

    // 3. Save to cache
    saveToCache(targetUrl, body, contentType);

    // 4. Relay response back to the client
    res.set("Content-Type", contentType);
    res.set("X-Proxy-Cache", "MISS");
    res.status(upstreamResponse.status).send(body);
  } catch (err) {
    console.error("Proxy fetch failed:", err.message);
    res.status(502).send(`Proxy error: could not reach ${targetUrl} (${err.message})`);
  }
});

app.listen(PORT, () => {
  console.log(`Educational proxy server running at http://localhost:${PORT}`);
});
