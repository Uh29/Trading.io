# Trading.iobackend/package.json
{
  "name": "ai-trader-backend",
  "version": "1.0.0",
  "main": "server.js",
  "license": "MIT",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "@alpacahq/alpaca-trade-api": "^3.1.3",
    "axios": "^1.4.0",
    "dotenv": "^16.0.0",
    "express": "^4.18.2",
    "cors": "^2.8.5"
  }
}
backend/.env.example
# Copy to .env and fill keys
ALPACA_API_KEY=your_alpaca_api_key
ALPACA_SECRET_KEY=your_alpaca_secret
# Use paper endpoint true/false (we'll only use Alpaca paper endpoint automatically)
ALPACA_PAPER=true
PORT=4000
SYMBOL=AAPL
backend/server.js
require('dotenv').config();
const express = require('express');
const cors = require('cors');
const TradeEngine = require('./tradeEngine');

const app = express();
app.use(cors());
app.use(express.json());

const port = process.env.PORT || 4000;

// Create single engine instance
const engine = new TradeEngine({
  key: process.env.ALPACA_API_KEY,
  secret: process.env.ALPACA_SECRET_KEY,
  paper: process.env.ALPACA_PAPER !== 'false',
  symbol: process.env.SYMBOL || 'AAPL'
});

// Simple REST endpoints

// Health
app.get('/api/health', (req, res) => res.json({ ok: true }));

// Start algo
app.post('/api/start', async (req, res) => {
  try {
    await engine.start();
    res.json({ status: 'started' });
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: String(err) });
  }
});

// Stop algo
app.post('/api/stop', (req, res) => {
  engine.stop();
  res.json({ status: 'stopped' });
});

// Get engine status
app.get('/api/status', async (req, res) => {
  const status = engine.getStatus();
  res.json(status);
});

// Force single decision step (for testing)
app.post('/api/step', async (req, res) => {
  try {
    const result = await engine.step();
    res.json(result);
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: String(err) });
  }
});

// Place manual order (safety: small qty)
app.post('/api/order', async (req, res) => {
  const { side, qty } = req.body;
  try {
    const result = await engine.placeMarketOrder(side, qty || 1);
    res.json(result);
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: String(err) });
  }
});

app.listen(port, () => {
  console.log(`Backend listening on ${port}`);
});
backend/tradeEngine.js
const Alpaca = require('@alpacahq/alpaca-trade-api');
const axios = require('axios');

class TradeEngine {
  constructor({ key, secret, paper = true, symbol = 'AAPL' }) {
    if (!key || !secret) {
      console.warn('ALPACA API keys not found. Place keys in environment to enable trading.');
    }
    this.symbol = symbol;
    this.running = false;
    // Alpaca client config
    this.alpaca = new Alpaca({
      keyId: key,
      secretKey: secret,
      paper: paper,
      usePolygon: false
    });

    // simple state
    this.prices = []; // latest close prices for SMA
    this.maxHistory = 200;
    this.shortWindow = 10;
    this.longWindow = 30;
    this.position = null; // 'long' or null
    this.loopIntervalMs = 15 * 1000; // every 15s for demo
    this._loop = null;
  }

  async fetchLatestBars(limit = 100) {
    // Use Alpaca data API to fetch recent bars (1m) — fallback to /v2/stocks... endpoints
    try {
      const bars = await this.alpaca.getBarsV2(
        this.symbol,
        { start: undefined, limit, timeframe: '1Min' }
      );
      const arr = [];
      for await (const b of bars) {
        arr.push(b);
      }
      // extract close prices
      return arr.map(x => x.closePrice);
    } catch (err) {
      console.warn('Error fetching bars, falling back to polygon or other source', err);
      return [];
    }
  }

  sma(arr, window) {
    if (arr.length < window) return null;
    const slice = arr.slice(arr.length - window);
    const sum = slice.reduce((s, v) => s + v, 0);
    return sum / window;
  }

  async evaluateSignal() {
    // fetch fresh bars and compute SMAs
    const latest = await this.fetchLatestBars(Math.max(this.longWindow + 10, 100));
    if (!latest || latest.length < this.longWindow) {
      return { ok: false, reason: 'not enough data' };
    }
    this.prices = latest;
    const shortSMA = this.sma(this.prices, this.shortWindow);
    const longSMA = this.sma(this.prices, this.longWindow);
    const lastPrice = this.prices[this.prices.length - 1];

    // signal logic: buy when shortSMA crosses above longSMA, sell when crosses below.
    let signal = 'hold';
    if (shortSMA && longSMA) {
      if (shortSMA > longSMA && this.position !== 'long') signal = 'buy';
      else if (shortSMA < longSMA && this.position === 'long') signal = 'sell';
    }

    return { ok: true, shortSMA, longSMA, lastPrice, signal };
  }

  async placeMarketOrder(side, qty = 1) {
    if (!this.alpaca) throw new Error('Alpaca client not configured.');
    const order = await this.alpaca.createOrder({
      symbol: this.symbol,
      qty,
      side,
      type: 'market',
      time_in_force: 'day'
    });
    // update position simplistically
    if (side === 'buy') this.position = 'long';
    if (side === 'sell') this.position = null;
    return order;
  }

  async step() {
    const data = await this.evaluateSignal();
    if (!data.ok) return data;
    if (data.signal === 'buy') {
      console.log('Signal BUY at', data.lastPrice);
      // place small order (demo)
      const order = await this.placeMarketOrder('buy', 1);
      return { action: 'bought', order, data };
    }
    if (data.signal === 'sell') {
      console.log('Signal SELL at', data.lastPrice);
      const order = await this.placeMarketOrder('sell', 1);
      return { action: 'sold', order, data };
    }
    return { action: 'hold', data };
  }

  async start() {
    if (this.running) return;
    this.running = true;
    // loop
    this._loop = setInterval(async () => {
      try {
        const result = await this.step();
        console.log('Engine step result', result.action || result);
      } catch (err) {
        console.error('Engine error', err);
      }
    }, this.loopIntervalMs);
    // run one immediately
    await this.step();
  }

  stop() {
    this.running = false;
    if (this._loop) clearInterval(this._loop);
  }

  getStatus() {
    return {
      running: this.running,
      symbol: this.symbol,
      position: this.position,
      latestPrice: this.prices.length ? this.prices[this.prices.length - 1] : null,
      shortWindow: this.shortWindow,
      longWindow: this.longWindow
    };
  }
}

module.exports = TradeEngine;{
  "name": "ai-trader-frontend",
  "version": "1.0.0",
  "private": true,
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "axios": "^1.4.0"
  },
  "scripts": {
    "start": "parcel public/index.html --port 3000"
  },
  "devDependencies": {
    "parcel": "^2.9.3"
  }
}
frontend/public/index.html
<!doctype html>
<html>
  <head>
    <meta charset="utf-8" />
    <title>AI Trader</title>
    <meta name="viewport" content="width=device-width,initial-scale=1"/>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="../src/App.jsx"></script>
  </body>
</html>
frontend/src/App.jsx
import React, { useEffect, useState } from 'react';
import { createRoot } from 'react-dom/client';
import axios from 'axios';

const API = 'http://localhost:4000/api';

function App() {
  const [status, setStatus] = useState(null);
  const [logs, setLogs] = useState([]);

  async function refresh() {
    try {
      const res = await axios.get(`${API}/status`);
      setStatus(res.data);
    } catch (e) {
      setStatus({ error: e.message });
    }
  }

  useEffect(() => {
    refresh();
    const t = setInterval(refresh, 5000);
    return () => clearInterval(t);
  }, []);

  async function start() {
    await axios.post(`${API}/start`);
    setLogs(l => [`started ${new Date().toLocaleTimeString()}`, ...l]);
    refresh();
  }

  async function stop() {
    await axios.post(`${API}/stop`);
    setLogs(l => [`stopped ${new Date().toLocaleTimeString()}`, ...l]);
    refresh();
  }

  async function step() {
    const r = await axios.post(`${API}/step`);
    setLogs(l => [JSON.stringify(r.data).slice(0,200), ...l]);
    refresh();
  }

  async function buy() {
    const r = await axios.post(`${API}/order`, { side: 'buy', qty: 1 });
    setLogs(l => [`buy ${new Date().toLocaleTimeString()} ${JSON.stringify(r.data).slice(0,200)}`, ...l]);
    refresh();
  }

  async function sell() {
    const r = await axios.post(`${API}/order`, { side: 'sell', qty: 1 });
    setLogs(l => [`sell ${new Date().toLocaleTimeString()} ${JSON.stringify(r.data).slice(0,200)}`, ...l]);
    refresh();
  }

  return (
    <div style={{ padding: 24, fontFamily: 'Arial, Helvetica, sans-serif' }}>
      <h1>AI Trader — demo</h1>
      <div style={{ marginBottom: 12 }}>
        <strong>Symbol:</strong> {status?.symbol ?? '—'} &nbsp;
        <strong>Running:</strong> {String(status?.running ?? 'false')} &nbsp;
        <strong>Position:</strong> {status?.position ?? '—'} &nbsp;
        <strong>Latest:</strong> {status?.latestPrice ?? '—'}
      </div>

      <div style={{ marginBottom: 12 }}>
        <button onClick={start}>Start</button>
        <button onClick={stop} style={{ marginLeft: 8 }}>Stop</button>
        <button onClick={step} style={{ marginLeft: 8 }}>Run Step</button>
        <button onClick={buy} style={{ marginLeft: 8 }}>Manual Buy</button>
        <button onClick={sell} style={{ marginLeft: 8 }}>Manual Sell</button>
      </div>

      <h3>Logs</h3>
      <div style={{ maxHeight: 300, overflow: 'auto', background: '#f8f8f8', padding: 12 }}>
        {logs.map((l, i) => <div key={i} style={{ fontFamily: 'monospace', marginBottom: 6 }}>{l}</div>)}
      </div>

      <footer style={{ marginTop: 18, color: '#666' }}>
        Demo SMA crossover. Use paper keys only until fully tested.
      </footer>
    </div>
  );
}

createRoot(document.getElementById('root')).render(<App />);
