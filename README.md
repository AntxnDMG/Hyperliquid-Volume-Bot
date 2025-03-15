# Hyperliquid-Volume-Bot
# Made for PVP.Trade as an Example. 
# The Volume Bot itself is setup to be a Telegram Bot so that all the functions can be tested properly.

import { Telegraf } from 'telegraf';
import * as WebSocket from 'ws';
import * as winston from 'winston';
import * as fs from 'fs';
import * as path from 'path';

const TRACKING_DATA_FILE = path.join(__dirname, 'tracking_data.json'); 

const BOT_TOKEN = ''; 
const WEBSOCKET_URL = 'wss://api.hyperliquid.xyz/ws';

const bot = new Telegraf(BOT_TOKEN);
const tradesData: { [coin: string]: WsTrade[] } = {};
const userSubscriptions: Map<number, UserSubscriptions> = loadUserSubscriptions();

const logger = winston.createLogger({
    level: 'info',
    format: winston.format.combine(
      winston.format.timestamp(),
      winston.format.printf(({ timestamp, level, message }) => `[${timestamp}] ${level}: ${message}`)
    ),
    transports: [
      new winston.transports.File({ filename: 'app.log', level: 'info' }), 
      new winston.transports.File({ filename: 'error.log', level: 'error' }),
    ],
  });
  
  const wsLogger = winston.createLogger({
    level: 'info',
    format: winston.format.combine(
      winston.format.timestamp(),
      winston.format.printf(({ timestamp, level, message }) => `[${timestamp}] ${level}: ${message}`)
    ),
    transports: [
      new winston.transports.File({ filename: 'websocket.log', level: 'info' }),
    ],
  });

// Asset List to Track
const specificAssets = [
  'VAPOR','HFUN','FARM','PIP','OMNIX','CATBAL','SCHIZO','JEFF','ATEHUN','RAGE','POINTS',
];

// Asset indexes
const specificAssetIndexes = [
37, 1, 123, 85, 74, 59, 23, 4, 51, 49, 8
];

// Mapping index & asset name
const assetIndexToNameMap: { [index: number]: string } = specificAssetIndexes.reduce((map, index, i) => {
  map[index] = specificAssets[i];
  return map;
}, {} as { [index: number]: string });

function findTokenNameByIndex(coin: string): string {
  const index = parseInt(coin.slice(1)); 
  return assetIndexToNameMap[index] || coin.slice(1);
}

interface WsTrade {
  coin: string;
  side: string;
  px: string;
  sz: string;
  hash: string;
  time: number;
  tid: number;
  users: [string, string];
}

interface UserSubscriptions {
  trackAll: boolean;
  specificCoins: Set<string>;
}

let ws = connectWebSocket();

function connectWebSocket() {
  const ws = new WebSocket(WEBSOCKET_URL);

  ws.on('open', async () => {
      logger.info('WebSocket connected');
      wsLogger.info('WebSocket connected');
      await subscribeToSpotAssets(); 
      startHeartbeat(ws); 
      logger.info('Started heartbeat to keep WebSocket connection alive.');
  });

  ws.on('message', (data: string) => {
      try {
          const parsedData = JSON.parse(data);
          if (parsedData.channel === 'trades') {
              wsLogger.info(`Received trades data: ${JSON.stringify(parsedData.data)}`);
              const trades: WsTrade[] = parsedData.data;
              trades.forEach((trade) => {
                  if (!tradesData[trade.coin]) tradesData[trade.coin] = [];
                  tradesData[trade.coin].push(trade);

                  checkVolumeAlert(trade.coin, trade);
              });
          }
          if (parsedData.channel === 'pong') {
              wsLogger.info('Received pong, connection is alive.');
          }
      } catch (error) {
          const errorMsg = error instanceof Error ? error.message : JSON.stringify(error);
          logger.error(`Error parsing WS message: ${errorMsg}`);
          wsLogger.error(`Error parsing WS message: ${errorMsg}`);
      }
  });

  ws.on('close', () => {
      logger.warn('WS connection closed, attempting to reconnect...');
      wsLogger.warn('WS connection closed, attempting to reconnect...');
      
      setTimeout(() => {
          connectWebSocket();
      }, 3000);
  });

  return ws;
}

function startHeartbeat(ws: WebSocket) {
  setInterval(() => {
      if (ws.readyState === WebSocket.OPEN) {
          ws.send(JSON.stringify({ method: "ping" }));
          wsLogger.info('Sent ping to WS server.');
      }
  }, 50000);
}

async function subscribeToSpotAssets() {
  logger.info('Subscribing to specific spot assets');

  specificAssetIndexes.forEach((index) => {
      const coin = `@${index}`;
      logger.info(`Subscribing to trades for ${coin}`);
      ws.send(
          JSON.stringify({
              method: 'subscribe',
              subscription: { type: 'trades', coin },
          })
      );
  });
}

// Load user Subscriptions from file - just for now // Edit Later
function loadUserSubscriptions() {
    if (fs.existsSync(TRACKING_DATA_FILE)) {
        const data = fs.readFileSync(TRACKING_DATA_FILE, 'utf8');
        return new Map<number, UserSubscriptions>(JSON.parse(data));
    }
    return new Map<number, UserSubscriptions>();
}

// Save user Subscriptions to file just for now // Edit Later
function saveUserSubscriptions() {
    fs.writeFileSync(
        TRACKING_DATA_FILE,
        JSON.stringify(Array.from(userSubscriptions.entries()), null, 2),
        'utf8'
    );
}

const alertThresholds: { [coin: string]: number } = {};

function checkVolumeAlert(coin: string, trade: WsTrade) {
  const now = Date.now();

  if (!tradesData[coin]) tradesData[coin] = [];
  
  tradesData[coin].push(trade);

  const recentTrades = tradesData[coin].filter((t) => now - t.time < 5 * 60 * 1000);

  const buyVolume = recentTrades
      .filter((t) => t.side === 'B')
      .reduce((sum, t) => sum + parseFloat(t.px) * parseFloat(t.sz), 0);
  const sellVolume = recentTrades
      .filter((t) => t.side === 'A')
      .reduce((sum, t) => sum + parseFloat(t.px) * parseFloat(t.sz), 0);

  if (buyVolume > 0 || sellVolume > 0) {
      logger.info(
          `Volume calculation for ${coin}: Buy $${buyVolume.toFixed(2)}, Sell $${sellVolume.toFixed(2)}`
      );
  }

  if (!specificAssetIndexes.includes(parseInt(coin.slice(1)))) return;

  if (!(coin in alertThresholds)) {
      alertThresholds[coin] = 10000; // Volume Threshold for Alerts
  }

  const currentThreshold = alertThresholds[coin];

  if (buyVolume > currentThreshold && buyVolume > sellVolume) {
      notifyUsers(coin, buyVolume, sellVolume);

      alertThresholds[coin] *= 2;
      logger.info(`Threshold for ${coin} increased to $${alertThresholds[coin].toFixed(2)}`);
  }
}

function notifyUsers(coin: string, buyVolume: number, sellVolume: number) {
  const timestamp = new Date().toLocaleString();

  const tokenName = findTokenNameByIndex(coin);

  userSubscriptions.forEach((subscriptions, userId) => {
      if (subscriptions.trackAll || subscriptions.specificCoins.has(tokenName)) { // Edit Later
          bot.telegram.sendMessage( 
              userId,
              `🚨 *High Volume Alert* 🚨 

Spot Pair: ${tokenName}/USDC

Buy Volume: $${buyVolume.toFixed(2)}
Sell Volume: $${sellVolume.toFixed(2)}

Timestamp: ${timestamp}`,
              { parse_mode: 'Markdown' }
          ).then(() => {
              logger.info(`Alert sent to user ${userId} for ${tokenName}`);
          }).catch((error) => {
              logger.error(`Error sending alert to user ${userId}: ${error.message}`);
          });
      }
  });
}

bot.start((ctx) => {
    console.log(`User ${ctx.from.id} started the bot`); // Edit Later
    ctx.reply(
      `☠️ Hyperliquid Volume Alert Bot!
  
This bot monitors Hyperliquid on-chain data and alerts you about volume surges in spot trading pairs.
  
💀 Here are some of the commands you can run:
  
/track <ticker> - Tracks Volume Alerts for a specific ticker
/untrack <ticker> - Stop tracking a specific ticker
/track_all - Track Volume Alerts for all Tickers
/untrack_all - Untrack Volume Alerts for all Tickers
  
Made by @OxDMG for PVP.Trade 💀`
    );
    userSubscriptions.set(ctx.from.id, { trackAll: false, specificCoins: new Set() });
  });

// TG Bot Commands
bot.command('track', (ctx) => {
  const args = ctx.message.text.split(' ').slice(1);
  const ticker = args[0]?.toUpperCase();

  if (!ticker) {
      ctx.reply('☠️ Please specify a ticker. Example: /track HYPE');
      return;
  }

  const supportedTickers = [
      'VAPOR', 'HFUN', 'FARM',
      'PIP', 'OMNIX', 'CATBAL', 'SCHIZO', 'JEFF',
      'ATEHUN', 'RAGE', 'POINTS'
  ];

  if (!supportedTickers.includes(ticker)) {
      ctx.reply(`☠️ The ticker "${ticker}" is not supported. Supported tickers: ${supportedTickers.join(', ')}`);
      return;
  }

  const userId = ctx.from.id;
  const userSubscription = userSubscriptions.get(userId) || { trackAll: false, specificCoins: new Set() };

  if (userSubscription.trackAll) {
      ctx.reply('☠️ You are already tracking all tickers. Use /untrack_all if you want to modify specific subscriptions.');
      return;
  }

  if (userSubscription.specificCoins.has(ticker)) {
      ctx.reply(`☠️ You are already tracking ${ticker}.`);
      return;
  }

  userSubscription.specificCoins.add(ticker);
  userSubscriptions.set(userId, userSubscription);
  saveUserSubscriptions();

  ctx.reply(`☠️ You will now receive Volume Alerts for ${ticker}`);
  logger.info(`User ${userId} started tracking ${ticker}`);
});

bot.command('track_all', (ctx) => {
  const userId = ctx.from.id;
  const subscriptions = userSubscriptions.get(userId) || { trackAll: false, specificCoins: new Set() };
  subscriptions.trackAll = true;
  subscriptions.specificCoins.clear();
  userSubscriptions.set(userId, subscriptions);
  saveUserSubscriptions(); 

  ctx.reply('☠️ You will now receive Volume Alerts for all Spot Pairs');
  logger.info(`User ${userId} subscribed to all spot pairs.`);
});

bot.command('untrack_all', (ctx) => {
  const userId = ctx.from.id;
  const subscriptions = userSubscriptions.get(userId) || { trackAll: false, specificCoins: new Set() };
  subscriptions.trackAll = false;
  userSubscriptions.set(userId, subscriptions);
  saveUserSubscriptions();

  ctx.reply('☠️ You will no longer receive Volume Alerts for all Spot Pairs');
  logger.info(`User ${userId} unsubscribed from all spot pairs.`);
});

bot.command('untrack', (ctx) => {
  const userId = ctx.from.id;
  const [_, ticker] = ctx.message.text.split(' ');

  if (!ticker) {
      ctx.reply('☠️ Please specify a ticker to untrack. Example: /untrack HYPE');
      return;
  }

  const subscriptions = userSubscriptions.get(userId);
  if (!subscriptions) {
      ctx.reply(`☠️ You are not tracking any tickers. Use /track_all or /track <ticker> first.`);
      return;
  }

  if (subscriptions.trackAll) {
      ctx.reply('☠️ You are currently tracking all tickers. Use /untrack_all to stop tracking all tickers first.');
      return;
  }

  if (subscriptions.specificCoins.delete(ticker)) {
      userSubscriptions.set(userId, subscriptions);
      saveUserSubscriptions();
      ctx.reply(`☠️ You will no longer receive Volume Alerts for ${ticker}`);
      logger.info(`User ${userId} unsubscribed from ticker ${ticker}`);
  } else {
      ctx.reply(`☠️ You were not tracking ${ticker}.`);
  }
});

bot.launch().then(() => {
  console.log('Bot started');
});

process.once('SIGINT', () => bot.stop('SIGINT'));
process.once('SIGTERM', () => bot.stop('SIGTERM'));
