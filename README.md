# bot.js
import makeWASocket, { useMultiFileAuthState } from "@whiskeysockets/baileys";
import axios from "axios";
import fs from "fs";
import path from "path";

async function startBot() {
  const { state, saveCreds } = await useMultiFileAuthState("auth_info");

  const sock = makeWASocket({
    printQRInTerminal: true,
    auth: state,
  });

  sock.ev.on("creds.update", saveCreds);

  sock.ev.on("messages.upsert", async ({ messages }) => {
    const msg = messages[0];
    if (!msg.message || !msg.key.remoteJid) return;

    const from = msg.key.remoteJid;
    let text = "";

    if (msg.message.conversation) text = msg.message.conversation;
    if (msg.message.extendedTextMessage)
      text = msg.message.extendedTextMessage.text;

    if (text.startsWith("http")) {
      await sock.sendMessage(from, { text: "⏳ جاري التحميل بدون علامة مائية..." });

      try {
        const apiUrl = `https://api.tikmate.app/api/lookup?url=${encodeURIComponent(text)}`;
        const res = await axios.get(apiUrl);
        const downloadUrl = res.data.videoUrl;

        const videoBuffer = await axios.get(downloadUrl, {
          responseType: "arraybuffer",
        });

        const filePath = path.join("./", "video.mp4");
        fs.writeFileSync(filePath, videoBuffer.data);

        await sock.sendMessage(from, {
          video: { url: filePath },
          caption: "✅ تم التحميل بدون علامة مائية",
        });

        fs.unlinkSync(filePath);
      } catch (error) {
        await sock.sendMessage(from, { text: "❌ حدث خطأ أثناء التحميل. حاول رابط آخر." });
      }
    }
  });
}

startBot();


# package.json
{
  "name": "whatsapp-downloader-bot",
  "version": "1.0.0",
  "main": "bot.js",
  "type": "module",
  "scripts": {

