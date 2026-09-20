require('dotenv').config();

const express = require('express');
const path = require('path');
const axios = require('axios');
const TelegramBot = require('node-telegram-bot-api');

const app = express();
const PORT = process.env.PORT || 3000;
const API_BASE = 'https://smmcpan.com/api/v2';

app.use(express.json({ limit: '2mb' }));
app.use(express.urlencoded({ extended: true }));
app.use(express.static(path.join(__dirname, 'public')));

function parseJsonResponse(data) {
  if (typeof data === 'string') {
    try {
      return JSON.parse(data);
    } catch (error) {
      return data;
    }
  }

  return data;
}

function normalizeServices(data) {
  if (Array.isArray(data)) {
    return data;
  }

  if (data && Array.isArray(data.services)) {
    return data.services;
  }

  if (data && Array.isArray(data.data)) {
    return data.data;
  }

  if (data && typeof data === 'object') {
    return Object.values(data).flatMap((item) => {
      if (Array.isArray(item)) {
        return item;
      }
      return item ? [item] : [];
    });
  }

  return [];
}

function prepareServicePayload(rawService) {
  const service = rawService || {};
  return {
    id: service.service || service.id || service.order || '',
    name: service.name || 'Service',
    category: service.category || service.type || 'General',
    type: service.type || service.category || 'General',
    rate: service.rate || '0',
    min: service.min || 0,
    max: service.max || 0,
    refill: Boolean(service.refill),
    cancel: Boolean(service.cancel)
  };
}

async function sendSmmApiRequest(payload) {
  const apiKey = process.env.SMMCPAN_API_KEY;
  if (!apiKey) {
    throw new Error('Missing SMMCPAN_API_KEY in environment variables.');
  }

  const form = new URLSearchParams();
  form.append('key', apiKey);

  Object.entries(payload).forEach(([key, value]) => {
    if (value !== undefined && value !== null && value !== '') {
      form.append(key, String(value));
    }
  });

  const response = await axios.post(API_BASE, form.toString(), {
    headers: {
      'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'
    },
    timeout: 30000
  });

  return parseJsonResponse(response.data);
}

app.get('/health', (req, res) => {
  res.json({
    ok: true,
    app: 'telegram-smmcpan-miniapp',
    apiConfigured: Boolean(process.env.SMMCPAN_API_KEY),
    botConfigured: Boolean(process.env.TELEGRAM_BOT_TOKEN && process.env.APP_URL)
  });
});

app.get('/api/services', async (req, res) => {
  try {
    const data = await sendSmmApiRequest({ action: 'services' });
    const rawServices = normalizeServices(data);
    const services = rawServices.map(prepareServicePayload);
    res.json(services);
  } catch (error) {
    console.error('Failed to fetch services:', error.message);
    res.status(500).json({
      ok: false,
      message: 'Unable to fetch services from SMMCPAN API.',
      error: error.response?.data || error.message
    });
  }
});

app.post('/api/order', async (req, res) => {
  try {
    const { service, link, quantity } = req.body || {};

    if (!service || !link || !quantity) {
      return res.status(400).json({
        ok: false,
        message: 'Service, link and quantity are required.'
      });
    }

    const serviceId = Number(service);
    const qty = Number(quantity);

    if (!Number.isFinite(serviceId) || serviceId <= 0) {
      return res.status(400).json({
        ok: false,
        message: 'Service ID is invalid.'
      });
    }

    if (!Number.isFinite(qty) || qty <= 0) {
      return res.status(400).json({
        ok: false,
        message: 'Quantity must be greater than zero.'
      });
    }

    const orderResult = await sendSmmApiRequest({
      action: 'add',
      service: serviceId,
      link,
      quantity: qty
    });

    const processed = {
      ok: true,
      raw: orderResult,
      orderId: orderResult?.order || orderResult?.id || orderResult?.order_id || orderResult?.orderID || null,
      status: orderResult?.status || orderResult?.result || 'pending'
    };

    if (!processed.orderId) {
      return res.status(400).json({
        ok: false,
        message: 'The API did not return a valid order ID.',
        raw: orderResult
      });
    }

    return res.json(processed);
  } catch (error) {
    console.error('Failed to create order:', error.message);
    res.status(500).json({
      ok: false,
      message: 'Unable to create order.',
      error: error.response?.data || error.message
    });
  }
});

app.get('/api/order/:orderId/status', async (req, res) => {
  try {
    const { orderId } = req.params;
    const statusResult = await sendSmmApiRequest({
      action: 'status',
      order: orderId
    });

    const normalizedStatus = {
      ok: true,
      orderId: orderId,
      raw: statusResult,
      status: statusResult?.status || statusResult?.order_status || statusResult?.result || 'unknown'
    };

    res.json(normalizedStatus);
  } catch (error) {
    console.error('Failed to fetch order status:', error.message);
    res.status(500).json({
      ok: false,
      message: 'Unable to fetch order status.',
      error: error.response?.data || error.message
    });
  }
});

app.get('*', (req, res) => {
  res.sendFile(path.join(__dirname, 'public', 'index.html'));
});

let telegramBot = null;

if (process.env.TELEGRAM_BOT_TOKEN && process.env.APP_URL) {
  telegramBot = new TelegramBot(process.env.TELEGRAM_BOT_TOKEN, { polling: false });

  app.post('/telegram-webhook', async (req, res) => {
    const update = req.body || {};
    const message = update.message;

    if (!message) {
      res.sendStatus(200);
      return;
    }

    const chatId = message.chat && message.chat.id;
    const text = message.text || '';

    if (chatId && text === '/start') {
      await telegramBot.sendMessage(chatId, 'Welcome! Open the SMMCPAN Mini App to create orders.', {
        reply_markup: {
          inline_keyboard: [[{
            text: 'Open App',
            web_app: { url: process.env.APP_URL }
          }]]
        }
      });
    }

    res.sendStatus(200);
  });

  telegramBot.setWebHook(`${process.env.APP_URL}/telegram-webhook`)
    .then(() => {
      console.log('Telegram webhook is active.');
    })
    .catch((error) => {
      console.error('Telegram webhook setup failed:', error.message);
    });

  if (process.env.TELEGRAM_CHAT_ID) {
    telegramBot.sendMessage(process.env.TELEGRAM_CHAT_ID, 'Mini App ready. Open the app from Telegram using the button below.', {
      reply_markup: {
        inline_keyboard: [[{
          text: 'Open App',
          web_app: { url: process.env.APP_URL }
        }]]
      }
    }).catch((error) => {
      console.error('Unable to send startup message to Telegram chat:', error.message);
    });
  }
} else {
  console.log('Telegram bot is not configured. Set TELEGRAM_BOT_TOKEN and APP_URL to enable it.');
}

app.listen(PORT, () => {
  console.log(`Server is running on http://localhost:${PORT}`);
});
