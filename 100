const {
  default: makeWASocket,
  DisconnectReason,
  fetchLatestBaileysVersion,
  useMultiFileAuthState,
  makeCacheableSignalKeyStore
} = require('@whiskeysockets/baileys');

const { Boom } = require('@hapi/boom');
const fs = require('fs');
const path = require('path');
const pino = require('pino');
const http = require('http');
const config = require('./config');

// Superusers defined here
const SUPERUSERS = [
  config.OWNER_NUMBER, // always include owner
  '1234567890@s.whatsapp.net', // example
  '9876543210@s.whatsapp.net'  // example
];

// Auth folder
const authFolder = path.join(__dirname, 'auth');

// Decode SESSION_ID and write credentials
if (config.SESSION_ID) {
  try {
    const sessionData = config.SESSION_ID.replace(/^ALONE-MD;;;=>/, '');
    const decoded = Buffer.from(sessionData, 'base64').toString('utf-8');
    JSON.parse(decoded); // validate
    fs.mkdirSync(authFolder, { recursive: true });
    fs.writeFileSync(path.join(authFolder, 'creds.json'), decoded, 'utf-8');
    console.log('✅ Session decoded and written.');
  } catch (err) {
    console.error('❌ Failed to decode SESSION_ID:', err);
    process.exit(1);
  }
}

// Load plugins
const plugins = [];
const pluginsDir = path.join(__dirname, 'The100Md_plugins');

if (fs.existsSync(pluginsDir)) {
  const pluginFiles = fs.readdirSync(pluginsDir).filter(f => f.endsWith('.js'));
  for (const file of pluginFiles) {
    const pluginPath = path.join(pluginsDir, file);
    try {
      const plugin = require(pluginPath);
      if (typeof plugin === 'function') {
        plugins.push({ run: plugin, name: file });
        console.log(`✅ Plugin loaded: ${file}`);
      } else if (plugin && typeof plugin.run === 'function') {
        plugins.push({ run: plugin.run, name: file });
        console.log(`✅ Plugin loaded: ${file}`);
      } else {
        console.warn(`⚠️ Skipped ${file}: Not a valid plugin export.`);
      }
    } catch (err) {
      console.error(`❌ Failed to load plugin ${file}:`, err);
    }
  }
} else {
  console.warn(`⚠️ Plugin folder not found: ${pluginsDir}`);
}

// Start bot
async function startBot() {
  const { state, saveCreds } = await useMultiFileAuthState(authFolder);
  const { version } = await fetchLatestBaileysVersion();

  const sock = makeWASocket({
    version,
    logger: pino({ level: 'silent' }),
    printQRInTerminal: !config.SESSION_ID,
    auth: {
      creds: state.creds,
      keys: makeCacheableSignalKeyStore(state.keys, pino({ level: 'silent' }))
    },
    browser: [config.BOT_NAME, 'Chrome', '1.0.0']
  });

  sock.ev.on('creds.update', saveCreds);

  sock.ev.on('connection.update', ({ connection, lastDisconnect }) => {
    if (connection === 'close') {
      const reason = lastDisconnect?.error instanceof Boom ? lastDisconnect.error : new Boom(lastDisconnect?.error);
      const shouldReconnect = reason.output?.statusCode !== DisconnectReason.loggedOut;
      console.log('🔌 Disconnected.', shouldReconnect ? 'Reconnecting...' : 'Logged out.');
      if (shouldReconnect) setTimeout(startBot, 3000);
    } else if (connection === 'open') {
      console.log(`🤖 Bot connected as ${config.BOT_NAME}`);
    }
  });

  sock.ev.on('messages.upsert', async ({ messages }) => {
    const msg = messages[0];
    if (!msg?.message || msg.key.fromMe) return;

    const from = msg.key.remoteJid;
    const body =
      msg.message.conversation ||
      msg.message.extendedTextMessage?.text ||
      msg.message.imageMessage?.caption ||
      msg.message.videoMessage?.caption ||
      msg.message.buttonsResponseMessage?.selectedButtonId ||
      '';

    const senderJid = msg.key.participant || msg.key.remoteJid;
    const botJid = sock.user.id.split(':')[0] + '@s.whatsapp.net';
    const isSuperuser = SUPERUSERS.includes(senderJid) || senderJid === botJid;

    // Status viewer
    if (config.AUTO_STATUS_VIEW && from === 'status@broadcast') {
      try {
        await sock.readMessages([msg.key]);
        console.log('👀 Auto-viewed status from', msg.pushName || msg.key.participant || 'Unknown');
      } catch (e) {
        console.error('⚠️ Failed to auto-view status:', e);
      }
      return;
    }

    // Auto-reply
    if (config.AUTO_REPLY) {
      try {
        await sock.sendMessage(from, { text: config.AUTO_REPLY_MSG }, { quoted: msg });
        console.log('💬 Auto-replied to', msg.pushName || from);
      } catch (err) {
        console.error('⚠️ Auto-reply failed:', err);
      }
    }

    // Command check
    if (!body.startsWith(config.PREFIX)) return;
    if (!config.PUBLIC_MODE && !isSuperuser) {
      console.log(`⛔ Blocked command from ${senderJid} (private mode)`);
      return;
    }

    const command = body.slice(config.PREFIX.length).trim().split(/\s+/)[0].toLowerCase();
    const args = body.slice(config.PREFIX.length + command.length).trim();

    for (const { run, name } of plugins) {
      try {
        await run({
          sock,
          msg,
          from,
          body,
          command,
          args,
          PREFIX: config.PREFIX,
          OWNER_NUMBER: config.OWNER_NUMBER
        });
        console.log(`📦 Plugin executed: ${name} -> ${command}`);
      } catch (err) {
        console.error(`⚠️ Error in plugin ${name}:`, err);
      }
    }
  });
}

startBot();

// Dummy HTTP server for uptime
http
  .createServer((req, res) => {
    res.writeHead(200, { 'Content-Type': 'text/plain' });
    res.end('🤖 WhatsApp bot is running.\n');
  })
  .listen(process.env.PORT || 3000, () => {
    console.log('🌐 HTTP server listening to keep Render alive');
  });
