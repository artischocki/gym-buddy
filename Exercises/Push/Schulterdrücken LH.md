---
tags:
  - exercise
sets: "3"
reps: 4 - 6
---
```dataviewjs
const targetPath = dv.current().file.path;

// Notes finden, die diese Übung verlinken
const linkedPages = dv.pages()
  .where(p => (p.file.outlinks ?? []).some(l => l.path === targetPath));

function parseTodaySets(md) {
  const m = md.match(/^\|\s*Today\s*\|(.+?)\|\s*$/m);
  if (!m) return null;

  const cells = m[1].split("|").map(s => s.trim()).filter(s => s.length > 0);

  const sets = [];
  for (let i = 0; i < cells.length; i += 2) {
    sets.push({ kg: cells[i] ?? "", reps: cells[i + 1] ?? "" });
  }
  return sets;
}

function parseDateFromName(name) {
  // yyyy-mm-dd
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

let rows = [];
let maxSets = 0;

for (let p of linkedPages) {
  const md = await dv.io.load(p.file.path);
  const sets = parseTodaySets(md);
  if (!sets) continue;

  maxSets = Math.max(maxSets, sets.length);

  const sortKey =
    p.file.day ??
    parseDateFromName(p.file.name) ??
    new Date(0);

  rows.push({ note: p.file.link, sets, sortKey });
}

// Newest first
rows.sort((a, b) => b.sortKey - a.sortKey);

// Header dynamic
const headers = ["Workout"];
for (let i = 1; i <= maxSets; i++) headers.push(`${i} kg`, "reps");

// Body
const tableRows = rows.map(r => {
  const out = [r.note];
  for (let i = 0; i < maxSets; i++) out.push(r.sets[i]?.kg ?? "", r.sets[i]?.reps ?? "");
  return out;
});

dv.table(headers, tableRows);
```
