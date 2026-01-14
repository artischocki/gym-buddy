<%*
const DAILY_FOLDER = "src/resources/daily";
const DATE_FORMAT = "DD.MM.YY";
const LINE_PREFIX = "# Today's Workout: ";

let todaysWorkout = "";

// 1) Get candidate daily notes
const files = app.vault.getFiles()
  .filter(f => f.path.startsWith(DAILY_FOLDER) && f.extension === "md");

// 2) Pick the ones with a valid date in filename, sorted newest -> oldest
const dated = files
  .map(f => {
    const m = window.moment(f.basename, DATE_FORMAT, true);
    return m.isValid() ? { file: f, date: m } : null;
  })
  .filter(Boolean)
  .sort((a, b) => b.date.valueOf() - a.date.valueOf());

// If there is no previous daily (only the one just created), start with Push 1
if (dated.length <= 1) {
  todaysWorkout = "Push 1";
} else {
  // dated[0] is current/newest; dated[1] is previous
  const lastDaily = dated[1].file;

  const content = await app.vault.read(lastDaily);

  const rawLine = content
    .split(/\r?\n/)
    .find(l => l.trim().startsWith(LINE_PREFIX));

  let lastWorkout = "";
  if (rawLine) {
    lastWorkout = rawLine.slice(LINE_PREFIX.length).trim();
  }

  switch (lastWorkout) {
    case "Push 1": todaysWorkout = "Pull 1"; break;
    case "Pull 1": todaysWorkout = "Legs 1"; break;
    case "Legs 1": todaysWorkout = "Push 2"; break;
    case "Push 2": todaysWorkout = "Pull 2"; break;
    case "Pull 2": todaysWorkout = "Legs 2"; break;
    case "Legs 2": todaysWorkout = "Push 1"; break;
    default:       todaysWorkout = "Push 1";
  }
}

let workoutType = todaysWorkout.slice(0, -2);
let workoutIndex = todaysWorkout.slice(-1);
%>---
type: <% workoutType %>
index: <% workoutIndex %>
tags: workout
---
<%*
tR += `${LINE_PREFIX}${todaysWorkout}`;
%>
<%*
const WORKOUTS_FOLDER = "Workouts/";
const workoutPath = `${WORKOUTS_FOLDER}${todaysWorkout}.md`;

let workoutFile = app.vault.getAbstractFileByPath(workoutPath);


if (!workoutFile) {
  workoutFile = app.vault.getFiles().find(f =>
    f.path.startsWith(WORKOUTS_FOLDER) &&
    f.extension === "md" &&
    f.basename === todaysWorkout
  );
}

if (!workoutFile) {
  tR += `Workout-File for "${todaysWorkout}" not found. Expected: ${workoutPath}`;
  return;
}


const workoutMd = await app.vault.read(workoutFile);
%>
<%*
const dv = app.plugins?.plugins?.dataview?.api;
if (!dv) { tR += "Dataview ist nicht verfügbar."; return; }

const currentPath = tp.file.path(true);

/* ========= HELPERS ========= */

function extractWikiLinks(md) {
  const links = [];
  const re = /\[\[([^\]|]+)(?:\|[^\]]+)?\]\]/g;
  let m;
  while ((m = re.exec(md)) !== null) links.push(m[1].trim());
  return links;
}

function parseDateFromName(name) {
  let m = name.match(/(\d{4})[-.](\d{2})[-.](\d{2})/);
  if (m) return new Date(Number(m[1]), Number(m[2]) - 1, Number(m[3]));
  m = name.match(/(\d{2})[-.](\d{2})[-.](\d{2})/);
  if (m) {
    const yy = Number(m[3]);
    const yyyy = yy < 70 ? 2000 + yy : 1900 + yy;
    return new Date(yyyy, Number(m[2]) - 1, Number(m[1]));
  }
  return null;
}

function getSortKeyForPage(p) {
  if (p.file.day?.toJSDate) return p.file.day.toJSDate();
  if (p.file.day instanceof Date) return p.file.day;
  return parseDateFromName(p.file.name) ?? new Date(0);
}

function formatDDMMYY(d) {
  return window.moment(d).format("DD.MM.YY");
}

function getExerciseSetsAndReps(exName) {
  let f = app.metadataCache.getFirstLinkpathDest(exName, "");
  if (!f) {
    f = app.vault.getAbstractFileByPath(`Excercises/${exName}.md`)
      ?? app.vault.getAbstractFileByPath(`Exercises/${exName}.md`);
  }
  if (!f) return { file: null, sets: null };

  const cache = app.metadataCache.getFileCache(f);
  const setsRaw = cache?.frontmatter?.sets;
  const repsRaw = cache?.frontmatter?.reps;
  const sets = setsRaw == null ? null : Number(setsRaw);
  return { file: f, sets: Number.isFinite(sets) ? sets : null, reps: repsRaw};
}

function escapeRegExp(s) {
  return s.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
}

function pageLinksToExercise(p, exercisePath) {
  const out = p.file.outlinks ?? [];
  for (const l of out) {
    const dest = app.metadataCache.getFirstLinkpathDest(l.path, p.file.path);
    if (dest?.path === exercisePath) return true;
  }
  return false;
}

function parseTodaySetsFromExerciseSection(md, exerciseName) {
  const lines = md.split(/\r?\n/);
  const esc = escapeRegExp(exerciseName);
  const linkRe = new RegExp(String.raw`\[\[${esc}(?:\|[^\]]+)?\]\]`);

  let start = lines.findIndex(l => /^#{2,}\s/.test(l) && linkRe.test(l));
  if (start === -1) start = lines.findIndex(l => linkRe.test(l));
  if (start === -1) return null;

  let end = lines.length;
  for (let i = start + 1; i < lines.length; i++) {
    if (/^#{2,}\s/.test(lines[i])) { end = i; break; }
  }

  const section = lines.slice(start, end);

  const todayLine = section.find(l => /^\s*\|\s*Today\s*\|/i.test(l));
  if (!todayLine) return null;

  const m = todayLine.match(/^\s*\|\s*Today\s*\|(.*)\|\s*$/i);
  if (!m) return null;

  let cells = m[1].split("|").map(s => s.trim());
  while (cells.length && cells[0] === "") cells.shift();
  while (cells.length && cells[cells.length - 1] === "") cells.pop();

  const sets = [];
  for (let i = 0; i < cells.length; i += 2) {
    sets.push({ kg: cells[i] ?? "", reps: cells[i + 1] ?? "" });
  }
  return sets;
}

function buildTable(setCount, refLabel, refSets) {
  const x = Math.max(1, Number(setCount) || 1);

  const header = ["Workout"];
  for (let i = 1; i <= x; i++) header.push(`${i} kg`, "reps");

  const sep = header.map((_, idx) => (idx === 0 ? "--------" : "----"));

  // Referenzzeile nur, wenn vorhanden
  let refRow = null;
  if (refLabel && Array.isArray(refSets) && refSets.length) {
    refRow = [refLabel];
    for (let i = 0; i < x; i++) {
      refRow.push(refSets?.[i]?.kg ?? "", refSets?.[i]?.reps ?? "");
    }
  }

  const todayRow = ["Today", ...Array(x * 2).fill("")];

  const line = (cells) => `| ${cells.join(" | ")} |`;

  const out = [line(header), line(sep)];
  if (refRow) out.push(line(refRow));
  out.push(line(todayRow));

  return out.join("\n");
}

/* ========= DAILY PAGES ========= */

let allDailyPages = Array.from(dv.pages(`"${DAILY_FOLDER.replace(/\/$/, "")}"`))
  .filter(p => p?.file?.path)
  .filter(p => p.file.path !== currentPath);

/* ========= OUTPUT ========= */

const exerciseLinks = extractWikiLinks(workoutMd);

for (let i = 0; i < exerciseLinks.length; i++) {
  const exName = exerciseLinks[i];

  const { file: exFile, sets, reps } = getExerciseSetsAndReps(exName);
  const setCount = sets ?? 4;

  let refLabel = null;
  let refSets = null;

  if (exFile?.path) {
    const linked = allDailyPages
      .filter(p => pageLinksToExercise(p, exFile.path))
      .sort((a, b) => getSortKeyForPage(b) - getSortKeyForPage(a));

    for (const p of linked) {
      const md = await dv.io.load(p.file.path);
      const parsed = parseTodaySetsFromExerciseSection(md, exName);
      if (parsed && parsed.length) {
        refLabel = formatDDMMYY(getSortKeyForPage(p));
        refSets = parsed;
        break;
      }
    }
  }

  tR += `## ${i + 1}. [[${exName}]]\n---\n`;
  tR += `Rep Range: ${reps}\n\n`
  tR += buildTable(setCount, refLabel, refSets);
  tR += `\n\n`;
}
%>
