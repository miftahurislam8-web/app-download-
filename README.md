const express = require('express');
const rateLimit = require('express-rate-limit');
const path = require('path');
const https = require('https');
const http = require('http');

const app = express();
const PORT = process.env.PORT || 3000;
const MAX_FILE_SIZE = parseInt(process.env.MAX_FILE_SIZE || '209715200', 10);
const ALLOWED_HOSTS = (process.env.ALLOWED_HOSTS || 'github.com,raw.githubusercontent.com,objects.githubusercontent.com')
  .split(',')
  .map((h) => h.trim().toLowerCase())
  .filter(Boolean);

app.use(express.json());
app.use(express.static(path.join(__dirname, 'public')));

const limiter = rateLimit({
  windowMs: 60 * 1000,
  max: 20,
  message: { error: 'অনেক বেশি রিকোয়েস্ট! একটু পরে চেষ্টা করুন।' },
});
app.use('/api/', limiter);

function extractHost(urlStr) {
  try { return new URL(urlStr).hostname.toLowerCase(); } catch { return null; }
}

function isAllowedHost(host) {
  if (!host) return false;
  return ALLOWED_HOSTS.some((allowed) => host === allowed || host.endsWith('.' + allowed));
}

function safeFilename(name) {
  return (name || 'app.apk').replace(/[^\w.\-]/g, '_').slice(0, 100);
}

app.post('/api/validate', async (req, res) => {
  const { url } = req.body || {};
  if (!url) return res.status(400).json({ error: 'URL প্রয়োজন' });

  const host = extractHost(url);
  if (!host) return res.status(400).json({ error: 'সঠিক URL দিন' });
  if (!isAllowedHost(host)) return res.status(403).json({ error: 'এই ডোমেইন অনুমোদিত নয়: ' + host });

  try {
    const meta = await new Promise((resolve, reject) => {
      const client = url.startsWith('https') ? https : http;
      const req2 = client.request(url, { method: 'HEAD' }, (r) => {
        const size = parseInt(r.headers['content-length'] || '0', 10);
        const disp = r.headers['content-disposition'] || '';
        let filename = '';
        const match = /filename\*?=(?:UTF-8'')?["']?([^"';]+)/i.exec(disp);
        if (match) filename = decodeURIComponent(match[1]);
        resolve({ statusCode: r.statusCode, size, contentType: r.headers['content-type'] || '', filename });
      });
      req2.on('error', reject);
      req2.setTimeout(10000, () => { req2.destroy(); reject(new Error('রিকোয়েস্ট টাইমআউট')); });
      req2.end();
    });

    if (meta.size && meta.size > MAX_FILE_SIZE) return res.status(413).json({ error: 'ফাইল অনেক বড়' });

    res.json({
      ok: true, size: meta.size, sizeMB: (meta.size / 1024 / 1024).toFixed(2),
      contentType: meta.contentType, filename: meta.filename || 'app',
      downloadUrl: `/api/download?url=${encodeURIComponent(url)}`,
    });
  } catch (err) {
    res.status(500).json({ error: 'লিংক যাচাই ব্যর্থ: ' + err.message });
  }
});

app.get('/api/download', async (req, res) => {
  const fileUrl = req.query.url;
  if (!fileUrl) return res.status(400).send('URL প্রয়োজন');
  const host = extractHost(fileUrl);
  if (!isAllowedHost(host)) return res.status(403).send('অনুমোদিত ডোমেইন নয়');

  try {
    const client = fileUrl.startsWith('https') ? https : http;
    const proxyReq = client.get(fileUrl, (proxyRes) => {
      if ([301, 302, 303, 307, 308].includes(proxyRes.statusCode)) {
        const redirectUrl = proxyRes.headers.location;
        if (!redirectUrl) return res.status(500).send('রিডাইরেক্ট ব্যর্থ');
        return res.redirect(302, `/api/download?url=${encodeURIComponent(redirectUrl)}`);
      }
      if (proxyRes.statusCode !== 200) return res.status(proxyRes.statusCode).send('সোর্স সার্ভার এরর');

      let filename = 'app.apk';
      const disp = proxyRes.headers['content-disposition'] || '';
      const m = /filename\*?=(?:UTF-8'')?["']?([^"';]+)/i.exec(disp);
      if (m) filename = decodeURIComponent(m[1]);
      else {
        try {
          const urlPath = new URL(fileUrl).pathname;
          const guess = path.basename(urlPath);
          if (guess && guess.includes('.')) filename = guess;
        } catch {}
      }
      filename = safeFilename(filename);

      res.setHeader('Content-Type', proxyRes.headers['content-type'] || 'application/octet-stream');
      res.setHeader('Content-Disposition', `attachment; filename="${filename}"`);
      if (proxyRes.headers['content-length']) res.setHeader('Content-Length', proxyRes.headers['content-length']);
      proxyRes.pipe(res);
    });
    proxyReq.on('error', () => { if (!res.headersSent) res.status(500).send('ডাউনলোড ব্যর্থ'); });
  } catch (err) {
    res.status(500).send('সার্ভার এরর');
  }
});

app.listen(PORT, () => {
  console.log(`✅ সার্ভার চালু: http://localhost:${PORT}`);
});
