<%*
const { requestUrl, Notice } = tp.obsidian;
const file = tp.config.target_file;

const SERIES_FOLDER = "Media DB/series";
const IMAGE_FOLDER = "Media DB/images";
const WANTED_FORMATS = ["TV", "TV_SHORT", "ONA"];
const MAX_VISITS = 15;
const sleep = ms => new Promise(r => setTimeout(r, ms));
const q = v => JSON.stringify(v);

const FIELDS = `id idMal format episodes status
  title { romaji english native }
  startDate { year month day } endDate { year month day }
  genres studios(isMain: true) { nodes { name } }
  coverImage { extraLarge }
  relations { edges { relationType node { id idMal type format } } }`;

async function anilist(by, value) {
  const query = by === "mal"
    ? `query($v:Int){ Media(idMal:$v, type:ANIME){ ${FIELDS} } }`
    : `query($v:Int){ Media(id:$v, type:ANIME){ ${FIELDS} } }`;
  for (let i = 0; i < 4; i++) {
    await sleep(700 * (i + 1));
    try {
      const r = await requestUrl({
        url: "https://graphql.anilist.co",
        method: "POST",
        contentType: "application/json",
        body: JSON.stringify({ query, variables: { v: Number(value) } }),
        throw: false
      });
      console.log("[anime] anilist", by, value, "status", r.status);
      if (r.status === 200) return r.json?.data?.Media ?? null;
      if (r.status === 404) return null;
    } catch (e) {
      console.log("[anime] anilist error", value, e);
    }
  }
  return null;
}

const safeName = s => s.replace(/\[/g, "(").replace(/\]/g, ")").replace(/[\\/:*?"<>|#^]/g, "").trim();

const fmtDate = d => {
  if (!d?.year) return "unknown";
  return String(d.month ?? 1).padStart(2, "0") + "/" +
         String(d.day ?? 1).padStart(2, "0") + "/" + d.year;
};

const list = (k, arr) => arr.length ? `${k}:\n` + arr.map(x => `  - ${q(x)}`).join("\n") : `${k}: []`;

function block(episodes) {
  const status = "INPUT[inlineSelect(option(to watch), option(watching), option(watched), option(dropped)):status]";
  const ep = episodes
    ? "INPUT[slider(minValue(0), maxValue(" + episodes + "), addLabels(true)):currentlyAtEpisode]"
    : "INPUT[number:currentlyAtEpisode]";
  return "Статус: `" + status + "`\n\nСерія: `" + ep + "`" + (episodes ? " / " + episodes : "");
}

function exists(malId) {
  return app.vault.getMarkdownFiles().some(f =>
    f.path.startsWith(SERIES_FOLDER + "/") &&
    String(app.metadataCache.getFileCache(f)?.frontmatter?.id) === String(malId));
}

async function createNote(a) {
  const title = a.title.romaji ?? a.title.english ?? a.title.native;
  const year = String(a.startDate?.year ?? "");
  const base = safeName(`${title} (${year})`);
  const path = `${SERIES_FOLDER}/${base}.md`;
  if (app.vault.getAbstractFileByPath(path)) return null;

  let image = "";
  try {
    const url = a.coverImage?.extraLarge;
    if (url) {
      const imgPath = `${IMAGE_FOLDER}/series_${base}.jpg`;
      if (!app.vault.getAbstractFileByPath(imgPath)) {
        const r = await requestUrl({ url });
        await app.vault.createBinary(imgPath, r.arrayBuffer);
      }
      image = `[[${imgPath}]]`;
    }
  } catch (e) { console.log("[anime] image error", e); }

  const slug = title.replace(/[^A-Za-z0-9]+/g, "_").replace(/^_+|_+$/g, "");
  const url = `https://myanimelist.net/anime/${a.idMal}/${slug}`;

  const yaml = [
    "---",
    "type: series",
    `title: ${q(title)}`,
    `englishTitle: ${q(a.title.english ?? title)}`,
    `year: ${q(year)}`,
    "dataSource: MALAPI",
    `url: ${url}`,
    `id: ${q(String(a.idMal))}`,
    `image: ${q(image)}`,
    list("genres", a.genres ?? []),
    list("studio", (a.studios?.nodes ?? []).map(s => s.name)),
    `episodes: ${a.episodes ?? ""}`,
    "actors: []",
    `airedFrom: ${fmtDate(a.startDate)}`,
    `airedTo: ${fmtDate(a.endDate)}`,
    "status: to watch",
    "currentlyAtEpisode: 0",
    "---",
    "",
    block(a.episodes),
    ""
  ].join("\n");

  await app.vault.create(path, yaml);
  return base;
}

async function main() {
  window.animeBusy ??= new Set();
  if (window.animeBusy.has(file.path)) return;
  window.animeBusy.add(file.path);
  try {
    let text = "";
    for (let i = 0; i < 30; i++) {
      await sleep(500);
      try { text = await app.vault.read(file); } catch (e) { continue; }
      if (/^id:/m.test(text) && /^dataSource:/m.test(text)) break;
    }
    if (!/^id:/m.test(text)) return;
    if (/^currentlyAtEpisode:/m.test(text)) return;

    let seedMal, ds, eps;
    await app.fileManager.processFrontMatter(file, fm => {
      if (fm.type !== "series") return;
      fm.status ??= "to watch";
      fm.currentlyAtEpisode ??= 0;
      seedMal = fm.id; ds = fm.dataSource; eps = fm.episodes;
    });
    if (!seedMal) return;

    await app.vault.process(file, t =>
      t.includes("INPUT[") ? t : t.trimEnd() + "\n\n" + block(eps) + "\n");

    if (ds !== "MALAPI") return;

    const seed = await anilist("mal", seedMal);
    if (!seed) { new Notice("AniList can't find MAL id " + seedMal); return; }

    const visited = new Set([seed.id]);
    const queue = [seed];
    const added = [];
    let n = 0;
    while (queue.length && n < MAX_VISITS) {
      const a = queue.shift(); n++;
      for (const e of a.relations?.edges ?? []) {
        if (!["SEQUEL", "PREQUEL"].includes(e.relationType)) continue;
        if (e.node.type !== "ANIME" || visited.has(e.node.id)) continue;
        visited.add(e.node.id);
        const full = await anilist("id", e.node.id);
        if (!full) continue;
        console.log("[anime]", full.title.romaji, full.format, "malId", full.idMal);
        queue.push(full);
        if (WANTED_FORMATS.includes(full.format) && full.idMal && !exists(full.idMal)) {
          const name = await createNote(full);
          if (name) added.push(name);
        }
      }
    }
    new Notice(added.length ? "Added seasons:\n" + added.join("\n") : "Not found");
  } catch (e) {
    console.error(e);
    new Notice("Помилка пошуку сезонів: " + e.message);
  } finally {
    window.animeBusy.delete(file.path);
  }
}

setTimeout(main, 0);
-%>
