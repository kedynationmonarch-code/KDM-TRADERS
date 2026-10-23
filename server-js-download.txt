import express from 'express';
import helmet from 'helmet';
import rateLimit from 'express-rate-limit';
import dotenv from 'dotenv';

dotenv.config();
const app = express();
app.use(helmet({ contentSecurityPolicy: false }));
app.use(express.json({ limit: '20kb' }));
app.use('/api/', rateLimit({ windowMs: 60_000, limit: 20 }));
app.use(express.static('.'));

app.get('/api/health', (_req, res) => res.json({ ok: true, service: 'KDM TRADERS API', smsConfigured: Boolean(process.env.MEGASMS_API_TOKEN), provider: 'MegaSMS', senderConfigured: Boolean(process.env.MEGASMS_SENDER_ID) }));

app.post('/api/sms/receipt', async (req, res) => {
  const token = process.env.MEGASMS_API_TOKEN;
  if (!token) return res.status(503).json({ ok: false, error: 'SMS gateway is not configured on the server.' });
  const { phone, message, sender_id } = req.body ?? {};
  if (typeof phone !== 'string' || !/^\+?[0-9]{9,15}$/.test(phone.replace(/[\s-]/g, '')) || typeof message !== 'string' || !message.trim() || message.length > 1000) {
    return res.status(400).json({ ok: false, error: 'Provide a valid phone number and message (max 1000 characters).' });
  }
  const sender = typeof sender_id === 'string' && /^[A-Za-z0-9 ]{1,11}$/.test(sender_id) ? sender_id : (process.env.MEGASMS_SENDER_ID || 'KDM TRADERS');
  try {
    const upstream = await fetch(process.env.MEGASMS_API_URL || 'https://megasms.co.tz/api/v1/messages', {
      method: 'POST',
      headers: { Authorization: `Bearer ${token}`, 'Content-Type': 'application/json', Accept: 'application/json' },
      body: JSON.stringify({ to: phone.replace(/[\s-]/g, ''), message: message.trim(), sender_id: sender })
    });
    const raw = await upstream.text();
    let result;
    try { result = JSON.parse(raw); } catch { result = { response: raw.slice(0, 500) }; }
    if (!upstream.ok || result?.success === false) {
      const providerMessage = result?.error?.message || result?.message || 'SMS provider rejected the request.';
      return res.status(502).json({ ok: false, error: providerMessage, provider_status: upstream.status, provider_code: result?.error?.code || null });
    }
    return res.json({ ok: true, provider: result });
  } catch (_err) {
    return res.status(502).json({ ok: false, error: 'Could not reach the SMS provider.' });
  }
});

const port = Number(process.env.PORT || 3000);
app.listen(port, () => console.log(`KDM TRADERS starter listening on ${port}`));
