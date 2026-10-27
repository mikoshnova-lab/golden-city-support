const { Bot, InlineKeyboard, Keyboard } = require("grammy");

const TOKEN = process.env.BOT_TOKEN;
const ADMIN_ID = Number(process.env.ADMIN_ID || 0);

if (!TOKEN) throw new Error("BOT_TOKEN environment variable is missing.");
if (!Number.isSafeInteger(ADMIN_ID) || ADMIN_ID <= 0) {
  throw new Error("ADMIN_ID must be your numeric Telegram user ID.");
}

const bot = new Bot(TOKEN);

// These maps are kept in memory. Pending requests are lost if the process restarts.
const awaitingUserMessage = new Set();
const awaitingAdminReply = new Map(); // admin ID -> target user ID
const requestCounts = new Map();

function mainMenu() {
  return new InlineKeyboard()
    .text("🛠 Техқолдау", "support")
    .row()
    .text("📩 Админге жазу", "contact_admin")
    .row()
    .text("ℹ️ Көмек", "help");
}

function isAdmin(ctx) {
  return ctx.from && ctx.from.id === ADMIN_ID;
}

bot.command("start", async (ctx) => {
  await ctx.reply(
    "🌆 GOLDEN CITY — TECH SUPPORT\n\n" +
    "Сәлем! Серверге қатысты сұрағыңды жіберу үшін төмендегі мәзірді таңда.\n\n" +
    "🛠 Техқолдау — өтініш жіберу\n" +
    "📩 Админге жазу — админге хабарлама жіберу",
    { reply_markup: mainMenu() }
  );
});

bot.command("help", async (ctx) => {
  await ctx.reply(
    "📌 Қолдану нұсқаулығы\n\n" +
    "/start — басты мәзір\n" +
    "/help — көмек\n\n" +
    "Өтініш жіберу үшін 🛠 Техқолдау батырмасын басып, мәселеңді мәтінмен жаз."
  );
});

bot.command("admin", async (ctx) => {
  if (!isAdmin(ctx)) {
    return ctx.reply("⛔ Бұл команда тек админге арналған.");
  }
  const total = [...requestCounts.values()].reduce((sum, n) => sum + n, 0);
  await ctx.reply(
    "👑 GOLDEN CITY — АДМИН ПАНЕЛІ\n\n" +
    `📨 Осы іске қосылғаннан бергі өтініштер: ${total}\n` +
    `👤 Хабарласқан пайдаланушылар: ${requestCounts.size}\n\n` +
    "Жаңа өтініштер осы чатқа автоматты түрде келеді. Оларға боттың жіберген өтініш хабарламасындағы «✍️ Жауап беру» батырмасын бас."
  );
});

bot.callbackQuery("help", async (ctx) => {
  await ctx.answerCallbackQuery();
  await ctx.reply(
    "📌 Көмек\n\n" +
    "🛠 Техқолдау батырмасын басып, мәселеңді жаз. Хабарлама админге жіберіледі."
  );
});

bot.callbackQuery("support", async (ctx) => {
  awaitingUserMessage.add(ctx.from.id);
  await ctx.answerCallbackQuery();
  await ctx.reply("📝 Мәселеңді бір хабарламада жазып жібер. Болдырмау үшін /cancel жібер.");
});

bot.callbackQuery("contact_admin", async (ctx) => {
  awaitingUserMessage.add(ctx.from.id);
  await ctx.answerCallbackQuery();
  await ctx.reply("📩 Админге жіберетін хабарламаңды жаз. Болдырмау үшін /cancel жібер.");
});

bot.command("cancel", async (ctx) => {
  const uid = ctx.from.id;
  const cancelled = awaitingUserMessage.delete(uid);
  if (cancelled) {
    return ctx.reply("❎ Өтініш жіберу тоқтатылды.", { reply_markup: mainMenu() });
  }
  return ctx.reply("Қазір тоқтатылатын әрекет жоқ.", { reply_markup: mainMenu() });
});

// Admin starts a reply by pressing a button on a request notification.
bot.callbackQuery(/^reply:(\d+)$/, async (ctx) => {
  if (!isAdmin(ctx)) {
    return ctx.answerCallbackQuery({ text: "Бұл батырма тек админге арналған.", show_alert: true });
  }
  const userId = Number(ctx.match[1]);
  awaitingAdminReply.set(ADMIN_ID, userId);
  await ctx.answerCallbackQuery();
  await ctx.reply(`✍️ ${userId} пайдаланушысына жіберетін жауабыңды мәтінмен жаз. Болдырмау: /cancelreply`);
});

bot.command("cancelreply", async (ctx) => {
  if (!isAdmin(ctx)) return ctx.reply("⛔ Бұл команда тек админге арналған.");
  if (awaitingAdminReply.delete(ADMIN_ID)) return ctx.reply("❎ Жауап жазу тоқтатылды.");
  return ctx.reply("Қазір жауап жазу режимі қосылмаған.");
});

// Handle plain text messages: admin replies first, then user support requests.
bot.on("message:text", async (ctx) => {
  const userId = ctx.from.id;
  const messageText = ctx.message.text;

  // Do not process commands as support messages.
  if (messageText.startsWith("/")) return;

  if (isAdmin(ctx) && awaitingAdminReply.has(ADMIN_ID)) {
    const targetUserId = awaitingAdminReply.get(ADMIN_ID);
    awaitingAdminReply.delete(ADMIN_ID);
    try {
      await bot.api.sendMessage(
        targetUserId,
        "👑 GOLDEN CITY админінің жауабы:\n\n" + messageText
      );
      await ctx.reply("✅ Жауабың пайдаланушыға жіберілді.");
    } catch (error) {
      console.error("Could not deliver admin reply:", error);
      await ctx.reply("⚠️ Хабарлама жіберілмеді. Пайдаланушы ботты ашып, /start басқанын тексер.");
    }
    return;
  }

  if (!awaitingUserMessage.has(userId)) return;
  awaitingUserMessage.delete(userId);

  const username = ctx.from.username ? `@${ctx.from.username}` : "(username жоқ)";
  const displayName = [ctx.from.first_name, ctx.from.last_name].filter(Boolean).join(" ") || "Аты көрсетілмеген";
  const previous = requestCounts.get(userId) || 0;
  requestCounts.set(userId, previous + 1);

  const adminKeyboard = new InlineKeyboard().text("✍️ Жауап беру", `reply:${userId}`);

  try {
    await bot.api.sendMessage(
      ADMIN_ID,
      "📩 GOLDEN CITY — ЖАҢА ӨТІНІШ\n\n" +
      `👤 Аты: ${displayName}\n` +
      `🔗 Username: ${username}\n` +
      `🆔 User ID: ${userId}\n\n` +
      "📝 Хабарлама:\n" + messageText,
      { reply_markup: adminKeyboard }
    );
    await ctx.reply("✅ Өтініш админге жіберілді. Жауап келгенде осы бот арқылы хабарлама аласың.", {
      reply_markup: mainMenu()
    });
  } catch (error) {
    console.error("Could not send request to admin:", error);
    await ctx.reply("⚠️ Өтінішті жіберу мүмкін болмады. Кейінірек қайталап көр.");
  }
});

bot.catch((error) => {
  console.error("Telegram bot error:", error);
});

bot.start({
  onStart: (botInfo) => {
    console.log(`GOLDEN CITY support bot started: @${botInfo.username}`);
  }
});
