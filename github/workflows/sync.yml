// Imports public web channels into a Firebase Realtime Database (channels + channelPosts) and sends pushes.
//   node sync.js            sync once
//   node sync.js --check    test the channels only (no Firebase needed)
// Env: FIREBASE_SERVICE_ACCOUNT, FIREBASE_DB_URL, ADMIN_UID, ONESIGNAL_APP_ID, ONESIGNAL_API_KEY, CHANNELS_JSON (the channel list)

const BASE = "https://t.me/s";
const ADMIN_UID = process.env.ADMIN_UID || "";
const OS_APP_ID = process.env.ONESIGNAL_APP_ID || "";
const OS_KEY = process.env.ONESIGNAL_API_KEY || "";
const FRESH_MS = 3 * 3600 * 1000;
const SEED_POSTS = 8;
const MAX_NEW_PER_RUN = 10;
const REFRESH_LAST = 20; // views / reactions of the newest N posts are refreshed on every run
const UA = "Mozilla/5.0 (compatible; ChannelSync/1.0)";

const selfRef = /telegram|t\.me\b|\btg\b/i;
// The channel list comes from the CHANNELS_JSON secret (so it is not visible in a public repo); channels.example.json is only a sample.
const allChannels = () => {
  const raw = (process.env.CHANNELS_JSON || "").trim();
  if (!raw) {
    console.log("CHANNELS_JSON is empty -> using the sample list (no real channels)");
    return require("./channels.example.json");
  }
  const j = JSON.parse(raw);
  console.log(`CHANNELS_JSON loaded: ${(j.channels || []).length} channels`);
  return j;
};
const config = () => allChannels().channels.filter((c) => c.enabled !== false);

let db = null;
function firebase() {
  if (db) return db;
  if (!ADMIN_UID) throw new Error("ADMIN_UID is not set");
  const admin = require("firebase-admin");
  admin.initializeApp({
    credential: admin.credential.cert(JSON.parse(process.env.FIREBASE_SERVICE_ACCOUNT || "{}")),
    databaseURL: process.env.FIREBASE_DB_URL,
  });
  db = admin.database();
  return db;
}

async function page(channel) {
  try {
    const r = await fetch(`${BASE}/${encodeURIComponent(channel)}`, { headers: { "User-Agent": UA, "Accept-Language": "en,am;q=0.8" }, signal: AbortSignal.timeout(20000), redirect: "follow" });
    return r.ok ? await r.text() : null;
  } catch (_) {
    return null;
  }
}

// ── tiny HTML helpers (no dependency) ────────────────────────────────────────
const ENT = { "&amp;": "&", "&lt;": "<", "&gt;": ">", "&quot;": '"', "&#39;": "'", "&nbsp;": " " };
const decode = (s) =>
  s
    .replace(/&(amp|lt|gt|quot|nbsp);|&#39;/g, (m) => ENT[m] || m)
    .replace(/&#x([0-9a-f]+);/gi, (_, h) => String.fromCodePoint(parseInt(h, 16)))
    .replace(/&#(\d+);/g, (_, n) => String.fromCodePoint(+n));
const htmlToText = (h) => decode(h.replace(/<br\s*\/?>/gi, "\n").replace(/<\/(p|div)>/gi, "\n").replace(/<[^>]+>/g, ""));

/** "12.3K" / "1.2M" / "845" -> number */
function count(s) {
  const m = /([\d.,]+)\s*([KMB]?)/i.exec((s || "").replace(/\s/g, ""));
  if (!m) return 0;
  const n = parseFloat(m[1].replace(/,/g, "")) || 0;
  return Math.round(n * ({ K: 1e3, M: 1e6, B: 1e9 }[m[2].toUpperCase()] || 1));
}

/** Removes links and everything that points back to the source network. */
function scrub(t) {
  let lines = t.replace(/\r/g, "").split("\n");
  lines = lines.filter((l) => !selfRef.test(l)); // any line naming the network / t.me links
  while (lines.length && /^[\s@\w.\-|•·»>🔔📢👉🔗✅➡️⬇️🔥💥]*$/u.test(lines[lines.length - 1]) && /@\w/.test(lines[lines.length - 1])) lines.pop(); // footer like "@channel_name"
  return lines
    .join("\n")
    .replace(/(https?:\/\/|www\.)\S+/gi, "")
    .replace(/[ \t]+\n/g, "\n")
    .replace(/\n{3,}/g, "\n\n")
    .trim();
}
const clean = (s) => scrub(decode(s || "")).replace(/\s+/g, " ").trim();

/** Parses one channel page -> { name, avatar, description, followers, posts[] } */
function parse(html) {
  if (!html || !/tgme_channel_info|tgme_widget_message/.test(html)) return null;
  const title = /tgme_channel_info_header_title[^>]*>([\s\S]*?)<\/div>/.exec(html);
  const avatar = /tgme_page_photo_image[^>]*>\s*<img[^>]+src="([^"]+)"/.exec(html);
  const desc = /tgme_channel_info_description[^>]*>([\s\S]*?)<\/div>/.exec(html);
  let followers = 0;
  const re = /counter_value">([^<]+)<\/span>\s*<span class="counter_type">([^<]+)</g;
  for (let m; (m = re.exec(html)); ) if (/subscriber|member/i.test(m[2])) followers = count(m[1]);

  const posts = [];
  const parts = html.split(/<div class="tgme_widget_message_wrap/).slice(1);
  for (const part of parts) {
    const id = /data-post="[^"/]+\/(\d+)"/.exec(part);
    if (!id) continue;
    if (/tgme_widget_message_forwarded_from|tgme_widget_message_video|tgme_widget_message_roundvideo|tgme_widget_message_voice|tgme_widget_message_document|tgme_widget_message_poll|tgme_widget_message_sticker|tgme_widget_message_service/.test(part)) continue;
    const tm = /tgme_widget_message_text[^>]*>([\s\S]*?)<\/div>\s*(?:<div class="tgme_widget_message_(?:reactions|footer|inline)|<\/div>|$)/.exec(part);
    const text = tm ? scrub(htmlToText(tm[1])) : "";
    const images = [];
    const pr = /tgme_widget_message_photo_wrap[^>]*background-image:url\('([^']+)'\)/g;
    for (let m; (m = pr.exec(part)); ) images.push(decode(m[1]));
    if (!text && !images.length) continue;
    const time = /<time[^>]*datetime="([^"]+)"/.exec(part);
    const views = /tgme_widget_message_views">([^<]+)</.exec(part);
    let reactions = 0;
    const rr = /tgme_reaction[^>]*>[\s\S]*?<\/(?:span|i)>\s*([\d.,]+[KM]?)\s*<\/span>/g;
    const rblock = /tgme_widget_message_reactions[\s\S]*?<\/div>/.exec(part);
    if (rblock) for (let m; (m = rr.exec(rblock[0])); ) reactions += count(m[1]);
    posts.push({
      key: `x_${id[1]}`,
      n: +id[1],
      text,
      images,
      createdAt: Math.min(Date.parse(time ? time[1] : "") || Date.now(), Date.now()),
      views: views ? count(views[1]) : 0,
      reactions,
    });
  }
  return {
    name: title ? clean(htmlToText(title[1])) : "",
    avatar: avatar ? decode(avatar[1]) : "",
    description: desc ? scrub(htmlToText(desc[1])).slice(0, 200) : "",
    followers,
    posts,
  };
}

const preview = (text, n) => (text || (n > 1 ? "🖼 Album" : "🖼 Photo")).replace(/\s+/g, " ").slice(0, 80);

async function check() {
  let live = 0;
  for (const c of allChannels().channels) {
    const info = parse(await page(c.channel));
    const state = !info ? "DEAD " : info.posts.length ? "LIVE " : "EMPTY";
    if (state === "LIVE ") live++;
    console.log(`${state} ${(c.category || "").padEnd(14)} ${c.channel.padEnd(30)} ${info ? `"${info.name}" ${info.followers} followers ${info.posts.length} usable posts${c.enabled === false ? "  [disabled]" : ""}` : ""}`);
  }
  console.log(`\n${live} live channels`);
}

async function pushMembers(chId, name, avatar, body, now) {
  const d = firebase();
  const members = Object.keys((await d.ref(`channels/${chId}/members`).get()).val() || {}).filter((u) => u !== ADMIN_UID);
  if (!members.length) return 0;
  const upd = {};
  for (const m of members) upd[`channelPings/${m}/${d.ref().push().key}`] = { channelName: name, body, channelId: chId, senderUid: ADMIN_UID, ts: now };
  await d.ref().update(upd).catch(() => {});
  if (!OS_KEY || !OS_APP_ID) return members.length;
  for (let i = 0; i < members.length; i += 1000) {
    await fetch("https://api.onesignal.com/notifications", {
      method: "POST",
      headers: { "Content-Type": "application/json", Authorization: `Key ${OS_KEY}` },
      body: JSON.stringify({
        app_id: OS_APP_ID,
        include_aliases: { external_id: members.slice(i, i + 1000) },
        target_channel: "push",
        headings: { en: `📢 ${name}` },
        contents: { en: body.length > 100 ? `${body.slice(0, 100)}…` : body },
        data: { chatId: `channel_${chId}`, senderName: name },
        collapse_id: `channel_${chId}`,
        priority: 10,
        ttl: 86400,
        android_visibility: 1,
        ...(avatar ? { large_icon: avatar } : {}),
      }),
    }).catch(() => {});
  }
  return members.length;
}

async function syncOne(c) {
  const d = firebase();
  const chId = `ext_${c.slug}`;
  const info = parse(await page(c.channel));
  if (!info) return { c: c.channel, error: "unreachable, private or not found" };
  const posts = info.posts.sort((a, b) => a.n - b.n);

  const chRef = d.ref(`channels/${chId}`);
  const meta = (await chRef.get()).val();
  const existing = (await d.ref(`channelPosts/${chId}`).orderByKey().limitToLast(80).get()).val() || {};
  const isNew = !meta;

  let name = (meta && meta.name) || clean(info.name) || c.channel;
  let avatar = (meta && meta.avatar) || info.avatar || "";
  const now = Date.now();
  if (isNew) {
    await chRef.set({
      name,
      description: clean(info.description),
      isPublic: true,
      isVerified: c.verified === true, // blue badge only when you set "verified": true for that channel
      channelType: "super",
      avatar,
      ownerId: ADMIN_UID,
      ownerUid: ADMIN_UID,
      createdAt: now,
      followers: info.followers,
      realStats: true, // the app shows the stored numbers as they are (no simulated growth)
      likes: 0,
      postCount: 0,
      members: { [ADMIN_UID]: { role: "owner", joinedAt: now } },
      admins: { [ADMIN_UID]: true },
    });
  } else if (info.followers && info.followers !== meta.followers) {
    await chRef.update({ followers: info.followers });
  }

  // refresh the real numbers of posts we already have
  const upd = {};
  for (const p of posts.slice(-REFRESH_LAST)) {
    const e = existing[p.key];
    if (!e) continue;
    if (e.views !== p.views) upd[`channelPosts/${chId}/${p.key}/views`] = p.views;
    if (e.likes !== p.reactions) upd[`channelPosts/${chId}/${p.key}/likes`] = p.reactions;
  }
  if (Object.keys(upd).length) await d.ref().update(upd).catch(() => {});

  let fresh = posts.filter((p) => !existing[p.key]);
  fresh = isNew ? fresh.slice(-SEED_POSTS) : fresh.slice(-MAX_NEW_PER_RUN);
  if (!fresh.length) return { c: c.channel, added: 0 };

  for (const p of fresh) {
    await d.ref(`channelPosts/${chId}/${p.key}`).set({
      text: p.text,
      attachUrl: p.images.length === 1 ? p.images[0] : "",
      attachUrls: p.images.length > 1 ? p.images : [],
      attachType: p.images.length === 1 ? "image" : p.images.length > 1 ? "album" : "",
      attachFileName: "",
      authorId: ADMIN_UID,
      authorName: name,
      authorAvatar: avatar,
      createdAt: p.createdAt,
      likes: p.reactions,
      views: p.views,
      reactions: {},
      commentCount: 0,
    });
  }
  const last = fresh[fresh.length - 1];
  await chRef.update({
    lastPostAt: Math.max(last.createdAt, (meta && meta.lastPostAt) || 0),
    lastPostText: preview(last.text, last.images.length),
    postCount: ((meta && meta.postCount) || 0) + fresh.length,
  });

  let pushed = 0;
  if (!isNew && now - last.createdAt < FRESH_MS) pushed = await pushMembers(chId, name, avatar, preview(last.text, last.images.length), now);
  return { c: c.channel, added: fresh.length, pushed };
}

async function syncAll() {
  const list = config();
  const out = [];
  let i = 0;
  const worker = async () => {
    while (i < list.length) {
      const c = list[i++];
      try {
        out.push(await syncOne(c));
      } catch (e) {
        out.push({ c: c.channel, error: String(e.message || e) });
      }
    }
  };
  await Promise.all([worker(), worker()]); // gentle: 2 at a time
  return out;
}

module.exports = (app) => {
  app.post("/api/sync", async (req, res) => {
    const key = process.env.ADMIN_KEY;
    if (!key || req.headers["x-admin-key"] !== key) return res.status(401).json({ error: "no" });
    res.json(await syncAll());
  });
};
module.exports.syncAll = syncAll;
module.exports._parse = parse;
module.exports._scrub = scrub;

if (require.main === module) {
  (process.argv.includes("--check")
    ? check()
    : syncAll().then((r) => {
        console.log(JSON.stringify(r, null, 1));
        const failed = r.filter((x) => x.error);
        const added = r.reduce((n, x) => n + (x.added || 0), 0);
        console.log(`\nSUMMARY: ${r.length} channels, ${added} new posts, ${failed.length} failed`);
        // a failed channel makes the run RED (and GitHub emails you) instead of a misleading green check
        if (failed.length) {
          failed.forEach((x) => console.error(`FAILED ${x.c}: ${x.error}`));
          process.exitCode = 1;
        }
      })
  ).then(() => process.exit(), (e) => {
    console.error(e);
    process.exit(1);
  });
}
