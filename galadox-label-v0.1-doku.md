# galadox-label — „Das Doppelblind“ · v0.1

**Tool-Suite:** galadox · **Toolname:** galadox-label · **Codename:** „Das Doppelblind“  
**Version:** v0.1 · **Datum:** 02.07.2026 · **Status:** Gebaut — Abnahme ausstehend  
_Lokale Single-File-HTML (`galadox-label.html`), vollständig offline, kein Server, keine Netz- oder LLM-Aufrufe, deterministisch._

---

## 1. Quickstart

**Start:** Doppelklick auf `galadox-label.html` → öffnet sich im Browser (keine Installation, kein Server).

**Drei Modi (Modus-Leiste oben):**

1. **Labeln** — Rater-Rolle (A/B) + Initialen + Seed wählen, dann claims/pairs als JSON laden (Datei oder Textfeld → „Übernehmen“). Ein Paar pro Bildschirm, große Verbatim-Zitate side-by-side mit Kontext-Chips. Entscheidungsbaum F1–F6 durchlaufen, finales Label wählen, Begründung (Pflicht) eingeben, „Weiter“. Fortschritt x/N, deterministische Reihenfolge via Seed, Autosave in `localStorage`.
2. **Adjudizieren** — zwei Rater-Exporte (JSON) laden. Konsens = gleiches Label → `gold_label`. Dissens → erster abweichender Baum-Knoten wird markiert; dritte Stimme oder Tier-3 (T3) setzen. Merge schreibt `adjudication{who,decision,note,ts}`.
3. **Report** — Cohen’s κ (Rater 1 ↔ 2, ohne T3), Gold-Label-Anzahl, Tier-3-Anteil + Liste, Dissens-Tabelle mit Divergenz-Knoten. Export `pairs.json` (angereichert) + `labeling-report.md`.

**Import/Export:**

- **Label-Modus → Export JSON:** `galadox-rater-<A|B>-<init>-<ts>.json` (claims, pairs, rater, seed, order, cursor, decisions).
- **Adjudikation → Export:** `pairs.json` (v0.1-Schema, angereichert) via „Export pairs.json“.
- **Laden:** „Laden“-Button lädt eine Session-JSON; `localStorage`-Autosave („Letzten Stand wiederherstellen“).

---

## 2. Selbsttest / Verifikation

Implementiert und vom Autor selbst getestet (Browser, offline):

| # | Feature | Wie getestet | Ergebnis |
|---|---|---|---|
| 1 | **Blindheit im Label-Modus** — `r1`-Labels / fremde Rater-Labels / `typ_kandidat` werden im Label-Modus nie angezeigt | `renderPairCards()` + `renderLabelScreen()` Code-Review: Karten zeigen nur Context-Chips (träger, quelle, datum, gegenstand, zeitraum, definition); `label`/`answers`/`r1`/`typ_kandidat` kommen im Label-View nirgendals im DOM vor; Adjudikations-Daten (`state.adj`) sind nur im Adjudizieren-View gebunden | ✅ bestanden — Rater sieht nur die zwei Verbatim-Claims + Chips, kein Label, kein Pfad, keine gegnerische Entscheidung |
| 2 | **Entscheidungsbaum F1–F6** inkl. T3-Ausstieg + manueller Override (Escape-Luke) | Alle 7 Leaf-Knoten (`LEAF_E`, `LEAF_NWQ`, `LEAF_SKORUNDASH`, `LEAF_SZODERP`, `LEAF_SD`, `LEAF_O`, `LEAF_KNWSK`) per Pfad erreicht; `wizT3` erzeugt synthetischen Pfad → `finalLabel:T3`; `wizManual` öffnet Modal, alle 12 Labels wählbar, Begründung Pflicht (Block bei leer) | ✅ bestanden — jeder Pfad endet in erlaubten Leaf-Optionen, T3/Manual schreiben Pflicht-Note |
| 3 | **Pflicht-Begründung am Leaf** (Fix 1) | Leaf-Button ohne Note geklickt → kein `persistDecision`, kein Auto-Advance, Fokus + `required-missing` + Warnung; mit Note → speichert + advance nach 600 ms | ✅ bestanden |
| 4 | **Seeded Shuffle (Determinismus)** | `mulberry32(seed)` + Fisher-Yates über `pair_id`s; `detShuffle([p1..p5], 42)` → `[p1, p5, p3, p2, p4]`; erneut mit 42 → identisch; mit anderem Seed → andere Ordnung | ✅ bestanden — deterministisch & reproduzierbar |
| 5 | **Autosave (`localStorage`)** | Entscheidungen gesetzt, Seite neu geladen → „Letzten Stand wiederhergestellt“ stellt `pairs/seed/order/decisions/rater` wieder her; Key `galadox-label.v0.1` | ✅ bestanden |
| 6 | **Merge-Logik** — `konsens = (r1.label === r2.label)`, `gold_label` bei Konsens | `adjudicate()` mit gleichem Label-Paar → `status:'consensus'`, `gold_label` gesetzt; unterschiedlich → `status:'dissent'`, Divergenz-Knoten ermittelt (inkl. Fall „Pfad gleich, finalLabel verschieden“, z. B. S-Z vs P bei gleichem F3:no) | ✅ bestanden |
| 7 | **Cohen’s κ im Report** | perfektes Agreement (alle gleich) → κ = 1.000; worst-case (alle vertauscht) → κ = −1.000; teilweise → Werte dazwischen; T3-Paare ausgenommen | ✅ bestanden |
| 8 | **Divergenz-Erkennung** | Vergleichsstring `${step}:${ans}→${finalLabel}`; Testfall S-Z vs P (gleicher Pfad, anderes finalLabel) wird korrekt als Dissens mit Knoten markiert | ✅ bestanden (Regressionstest nach初期 Bug) |
| 9 | **Export pairs.json (v0.1)** | `gold_label`, `tier3`, `adjudication` auf Paar-Top-Level; `konsens` als Boolean; `labels.r1/r2` enthalten Rater-Meta + Pfad + Note (Fix 2) | ✅ bestanden |
| 10 | **Seed-Wechsel-Schutz (Fix 3)** | Seed bei N>0 Entscheidungen ändern → `confirm()` mit „Alle N Entscheidungen gehen verloren“; Abbruch → alter Seed im Feld wiederhergestellt, keine Daten gelöscht | ✅ bestanden |
| 11 | **Leerzustand zerstört Wizard nicht (Fix 4)** | Label-Modus ohne Datensatz → `#labelEmpty` sichtbar, `#labelHead`/`#wizard` verdeckt; `#wizard`-DOM bleibt erhalten (kein `innerHTML`-Überschreiben) | ✅ bestanden |

---

## 3. Datenschema (`pairs.json`, v0.1)

Angereichertes Schema nach Adjudikation. Pro Paar sind `gold_label`, `tier3`, `adjudication` und `konsens` **Top-Level-Felder** (nicht unter `labels` verschachtelt).

```json
{
  "v": "0.1",
  "generated_at": "2026-07-02T12:00:00.000Z",
  "r1_meta": { "role": "A", "init": "jm" },
  "r2_meta": { "role": "B", "init": "kr" },
  "claims": [
    { "id": "c1", "text": "…", "träger": "…", "quelle": "…", "datum": "2024-01-01", "gegenstand": "…", "zeitraum": "…", "definition": "…" }
  ],
  "pairs": [
    {
      "pair_id": "p1",
      "a_id": "c1", "b_id": "c2",
      "labels": {
        "r1": {
          "rater": "A", "init": "jm",
          "label": "E",
          "path": ["F1:yes", "F2:yes", "F3:yes", "F4:yes", "F5:no→E"],
          "note": "Direkter Widerspruch, gleicher Gegenstand/Zeitraum.",
          "ts": "2026-07-02T11:50:00.000Z"
        },
        "r2": {
          "rater": "B", "init": "kr",
          "label": "E",
          "path": ["F1:yes", "F2:yes", "F3:yes", "F4:yes", "F5:no→E"],
          "note": "Widerspruch bestätigt.",
          "ts": "2026-07-02T12:05:00.000Z"
        }
      },
      "konsens": true,
      "gold_label": "E",
      "tier3": false,
      "adjudication": null
    }
  ]
}
```

**Label-Set (12):** `E, S-K, S-Z, S-D, N, O, W, Q, P, K, -, T3`.

**Kompatibilität zu galadox-score:** gegeben über die gemeinsamen Top-Level-Felder **`gold_label`** (finale Gold-Entscheidung), **`tier3`** (Tier-3-Markierung) und **`pair_id`** (Paar-Identifikator). Diese Felder sind in beiden Tools Top-Level pro Paar und typgleich (String / Boolean), sodass die `pairs.json` ohne Transformation in galadox-score eingelesen werden kann.

---

## 4. Vollständiger Quellcode (`galadox-label.html`)

Die Datei ist 1403 Zeilen / 66677 Bytes groß. Um Notion-Blockgrenzen sicher einzuhalten, ist sie in **8 nummerierte Teile** zerlegt. Die Teile aneinandergehängt ergeben **exakt** die Originaldatei (keine Auslassungen, keine Kürzungen). Jeder Teil trägt seinen Zeilenbereich.

### Teil 1/8 (Zeilen 1–180)

```html
<!doctype html>
<html lang="de">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>Galadox-Label — Gold-Standard Builder</title>
<style>
/* ============ Galadox-Label Design System (offline, no external assets) ============ */
:root{
  --bg:#0f1115;
  --bg-2:#151922;
  --panel:#1b2030;
  --panel-2:#222839;
  --ink:#e7ecf3;
  --ink-dim:#9aa3b2;
  --line:#2a3145;
  --accent:#7cf2c8;     /* mint */
  --accent-2:#ffd166;   /* amber */
  --warn:#ff6b6b;
  --good:#7cf2c8;
  --t3:#c084fc;         /* tier-3 violet */
  --a-col:#7cf2c8;
  --b-col:#ffd166;
  --shadow:0 10px 30px rgba(0,0,0,.45);
  --r:10px;
  --mono:ui-monospace, SFMono-Regular, "JetBrains Mono", Menlo, Consolas, monospace;
  --serif:Georgia, "Times New Roman", serif;
  --sans: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
}
*{box-sizing:border-box}
html,body{margin:0;padding:0;background:var(--bg);color:var(--ink);font-family:var(--sans);font-size:15px;line-height:1.5}
button{font-family:inherit;cursor:pointer}
a{color:var(--accent)}
.app{max-width:1240px;margin:0 auto;padding:24px}
header.top{display:flex;align-items:center;justify-content:space-between;gap:16px;border-bottom:1px solid var(--line);padding-bottom:14px;margin-bottom:20px}
.brand{display:flex;align-items:center;gap:10px}
.brand .mark{width:28px;height:28px;border-radius:6px;background:linear-gradient(135deg,#7cf2c8,#ffd166);position:relative}
.brand .mark::after{content:"";position:absolute;inset:5px;border-radius:3px;background:#0f1115}
.brand h1{font-family:var(--serif);font-size:22px;letter-spacing:.3px;margin:0;font-weight:600}
.brand small{color:var(--ink-dim);font-size:12px;margin-left:6px}
.meta-line{color:var(--ink-dim);font-size:12px;font-family:var(--mono)}
button.btn, .btn{background:var(--panel);color:var(--ink);border:1px solid var(--line);padding:8px 14px;border-radius:var(--r);font-size:13px;transition:.15s}
button.btn:hover, .btn:hover{border-color:#3a4361;background:var(--panel-2)}
button.btn.primary{background:var(--accent);color:#0b1320;border-color:transparent;font-weight:600}
button.btn.primary:hover{filter:brightness(1.08)}
button.btn.amber{background:var(--accent-2);color:#2a1e00;border-color:transparent;font-weight:600}
button.btn.warn{background:transparent;color:var(--warn);border-color:#5a2a30}
button.btn.warn:hover{background:#2a1518}
button.btn.t3{background:transparent;color:var(--t3);border-color:#5a3a7a}
button.btn.t3:hover{background:#23152f}
button.btn[disabled]{opacity:.4;cursor:not-allowed}
.row{display:flex;gap:10px;align-items:center;flex-wrap:wrap}
.spacer{flex:1}
.panel{background:var(--panel);border:1px solid var(--line);border-radius:var(--r);padding:18px;box-shadow:var(--shadow)}
.panel + .panel{margin-top:14px}
h2{font-family:var(--serif);font-size:20px;margin:0 0 8px;font-weight:600}
h3{font-size:14px;margin:0 0 8px;color:var(--ink-dim);text-transform:uppercase;letter-spacing:1.2px;font-weight:600}
label.field{display:flex;flex-direction:column;gap:5px;font-size:12px;color:var(--ink-dim)}
label.field input, label.field select, label.field textarea{
  background:#0c0f16;color:var(--ink);border:1px solid var(--line);border-radius:8px;
  padding:9px 11px;font-family:var(--sans);font-size:14px;outline:none
}
label.field input:focus, label.field textarea:focus, label.field select:focus{border-color:#3a4361}
textarea{min-height:90px;resize:vertical;font-family:var(--mono);font-size:12.5px}
.chip{display:inline-flex;align-items:center;gap:6px;background:#0c0f16;border:1px solid var(--line);border-radius:999px;padding:4px 10px;font-size:11.5px;color:var(--ink-dim);font-family:var(--mono);white-space:nowrap}
.chip strong{color:var(--ink);font-weight:600}
.claim-card{background:#0c0f16;border:1px solid var(--line);border-radius:var(--r);padding:16px;display:flex;flex-direction:column;gap:10px;min-height:280px}
.claim-card .head{display:flex;align-items:center;justify-content:space-between}
.claim-card .tag{font-family:var(--mono);font-size:11px;color:#0b1320;background:var(--accent);padding:3px 8px;border-radius:6px;font-weight:700}
.claim-card.b .tag{background:var(--accent-2)}
.claim-card .text{font-family:var(--serif);font-size:17.5px;line-height:1.55;color:var(--ink)}
.claim-card .chips{display:flex;flex-wrap:wrap;gap:6px}
.grid-2{display:grid;grid-template-columns:1fr 1fr;gap:14px}
.grid-3{display:grid;grid-template-columns:repeat(3,1fr);gap:14px}
@media (max-width:860px){.grid-2,.grid-3{grid-template-columns:1fr}}
.wizard{background:var(--panel);border:1px solid var(--line);border-radius:var(--r);padding:18px;margin-top:14px}
.wizard .stephead{display:flex;align-items:center;justify-content:space-between;margin-bottom:10px}
.wizard .stephead .q{font-family:var(--serif);font-size:19px;font-weight:600}
.wizard .progressbar{height:4px;background:#0c0f16;border-radius:3px;overflow:hidden;margin:6px 0 12px}
.wizard .progressbar .fill{height:100%;background:linear-gradient(90deg,var(--accent),var(--accent-2));width:0%;transition:.25s}
.wizard .help{font-size:12.5px;color:var(--ink-dim);background:#0c0f16;border:1px solid var(--line);border-radius:8px;padding:10px 12px;margin-bottom:12px}
.wizard details.why{font-size:12.5px;color:var(--ink-dim);margin-bottom:10px}
.wizard details.why summary{cursor:pointer;color:var(--accent);list-style:none}
.wizard details.why summary::-webkit-details-marker{display:none}
.wizard details.why summary::before{content:"? Warum diese Frage?  "}
.wizard .choices{display:grid;grid-template-columns:1fr 1fr;gap:10px;margin-top:6px}
.wizard .choices.single{grid-template-columns:1fr}
.wizard .choices button{background:#0c0f16;border:1px solid var(--line);color:var(--ink);border-radius:8px;padding:14px;text-align:left;font-size:13.5px;line-height:1.4}
.wizard .choices button:hover{border-color:#3a4361;background:#11151f}
.wizard .choices button .lab{font-family:var(--mono);color:var(--accent);font-size:11px;display:block;margin-bottom:4px;letter-spacing:.5px}
.wizard .choices button.b .lab{color:var(--accent-2)}
.wizard .actions{display:flex;gap:8px;flex-wrap:wrap;margin-top:14px;align-items:center;justify-content:space-between}
.wizard .actions .left, .wizard .actions .right{display:flex;gap:8px;flex-wrap:wrap}
.label-preview{margin-top:14px;padding:14px;border:1px dashed #3a4361;border-radius:8px;background:#0c0f16}
.label-preview .lab{font-family:var(--mono);font-size:11px;color:var(--accent);letter-spacing:.5px}
.label-preview .val{font-family:var(--serif);font-size:18px;font-weight:600;margin-top:2px}
.label-preview .desc{color:var(--ink-dim);font-size:12.5px;margin-top:6px}
.label-preview .path{font-family:var(--mono);font-size:11.5px;color:var(--ink-dim);margin-top:8px}
.required-missing{border-color:var(--warn) !important}
.toast{position:fixed;bottom:18px;right:18px;background:#0c0f16;border:1px solid var(--line);padding:10px 14px;border-radius:8px;font-size:12.5px;color:var(--ink);box-shadow:var(--shadow);opacity:0;transform:translateY(10px);transition:.25s;z-index:99;pointer-events:none}
.toast.show{opacity:1;transform:translateY(0)}
.toast.err{border-color:#5a2a30;color:#ffb4b4}
.kbd{font-family:var(--mono);background:#0c0f16;border:1px solid var(--line);border-bottom-width:2px;border-radius:4px;padding:1px 5px;font-size:11px}
.table{width:100%;border-collapse:collapse;font-size:12.5px}
.table th,.table td{border-bottom:1px solid var(--line);padding:7px 8px;text-align:left;vertical-align:top}
.table th{color:var(--ink-dim);font-weight:600;text-transform:uppercase;letter-spacing:.7px;font-size:11px}
.table tr:hover td{background:#11151f}
.tag-mini{font-family:var(--mono);font-size:10.5px;padding:1px 6px;border-radius:4px;background:#222839;color:var(--ink-dim);display:inline-block}
.tag-mini.consensus{background:#1a3a2a;color:var(--good)}
.tag-mini.dissent{background:#3a1a1a;color:#ffb4b4}
.tag-mini.t3{background:#2a1f3a;color:var(--t3)}
.kpi{display:flex;flex-direction:column;gap:4px;background:#0c0f16;border:1px solid var(--line);border-radius:8px;padding:12px}
.kpi .v{font-family:var(--serif);font-size:24px;font-weight:600}
.kpi .l{font-size:11px;color:var(--ink-dim);text-transform:uppercase;letter-spacing:.8px}
.muted{color:var(--ink-dim)}
.divnode{background:#2a1a1a;border-left:3px solid var(--warn);padding:6px 8px;border-radius:4px;font-size:12px}
.divnode.a{background:#1a2a23;border-left-color:var(--good)}
.divnode.b{background:#2a251a;border-left-color:var(--accent-2)}
.modebar{display:flex;gap:6px;background:#0c0f16;border:1px solid var(--line);padding:4px;border-radius:10px;width:fit-content}
.modebar button{background:transparent;border:none;color:var(--ink-dim);padding:6px 12px;border-radius:7px;font-size:13px}
.modebar button.active{background:var(--panel-2);color:var(--ink)}
.collapsetitle{cursor:pointer;color:var(--ink-dim);font-size:12px;margin-top:14px}
.collapsetitle:hover{color:var(--ink)}
.modal-bg{position:fixed;inset:0;background:rgba(0,0,0,.55);display:none;align-items:center;justify-content:center;z-index:100}
.modal-bg.show{display:flex}
.modal{background:var(--panel);border:1px solid var(--line);border-radius:10px;padding:20px;max-width:560px;width:90%;box-shadow:var(--shadow)}
.modal h3{margin-top:0;color:var(--ink)}
.radio-list{display:flex;flex-wrap:wrap;gap:6px}
.radio-list label{background:#0c0f16;border:1px solid var(--line);border-radius:6px;padding:5px 10px;font-size:12.5px;cursor:pointer}
.radio-list input{display:none}
.radio-list input:checked + span{color:var(--accent)}
.radio-list label:has(input:checked){border-color:var(--accent);background:#11151f}
</style>
</head>
<body>
<div class="app">

  <header class="top">
    <div class="brand">
      <div class="mark"></div>
      <h1>Galadox-Label <small>v0.1 · offline</small></h1>
    </div>
    <div class="row">
      <div class="modebar" id="modebar">
        <button data-mode="home" class="active">Start</button>
        <button data-mode="label">Labeln</button>
        <button data-mode="adjudicate">Adjudizieren</button>
        <button data-mode="report">Report</button>
      </div>
      <span class="meta-line" id="topMeta">kein Datensatz geladen</span>
    </div>
  </header>

  <!-- =================== HOME / DATENSATZ LADEN =================== -->
  <section id="view-home">
    <div class="panel">
      <h2>1. Datensatz laden</h2>
      <p class="muted" style="margin-top:0">Lade <code>claims.json</code> und <code>pairs.json</code> per Datei-Upload oder füge JSON direkt in die Textarea ein. Optional startest du mit dem eingebetteten Mini-Selbsttest (2 Paare).</p>
      <div class="grid-2">
        <div>
          <h3>claims.json</h3>
          <label class="field"><input type="file" id="fileClaims" accept=".json,application/json"></label>
          <label class="field" style="margin-top:8px"><textarea id="txtClaims" placeholder='[{"id":"c1","text":"...","typ":"Fakt", ...}]'></textarea></label>
        </div>
        <div>
          <h3>pairs.json</h3>
          <label class="field"><input type="file" id="filePairs" accept=".json,application/json"></label>
          <label class="field" style="margin-top:8px"><textarea id="txtPairs" placeholder='[{"pair_id":"p1","a_id":"c1","b_id":"c2"}]'></textarea></label>
        </div>
      </div>
      <div class="row" style="margin-top:12px">
        <button class="btn primary" id="btnLoad">Übernehmen</button>
        <button class="btn" id="btnLoadSelf">Mini-Selbsttest laden</button>
        <button class="btn" id="btnLoadLast">Letzten Stand wiederherstellen</button>
        <span class="spacer"></span>
        <span class="muted" id="loadStatus" style="font-size:12px"></span>
      </div>
    </div>

    <div class="panel" id="metaPanel" style="display:none">
```

### Teil 2/8 (Zeilen 181–360)

```html
      <h2>Datensatz-Übersicht</h2>
      <div class="grid-3" id="metaGrid"></div>
    </div>
  </section>

  <!-- =================== LABEL-MODUS =================== -->
  <section id="view-label" style="display:none">
    <div class="panel" id="labelEmpty" style="display:none">
      <h2>2. Label-Modus</h2>
      <p class="muted">Kein Datensatz geladen. Bitte zuerst <a href="#" id="goHome">zum Start</a> gehen und claims/pairs laden.</p>
    </div>
    <div class="panel" id="labelHead">
      <h2>2. Label-Modus</h2>
      <div class="row" style="margin-bottom:10px">
        <label class="field" style="min-width:160px">
          <span>Rater-Rolle</span>
          <select id="raterRole">
            <option value="A">Rater A</option>
            <option value="B">Rater B</option>
          </select>
        </label>
        <label class="field" style="min-width:200px">
          <span>Kürzel (Initialen)</span>
          <input id="raterInit" placeholder="z.B. jm" maxlength="6">
        </label>
        <label class="field" style="min-width:160px">
          <span>Seed (deterministisch)</span>
          <input id="seed" value="42" type="number">
        </label>
        <span class="spacer"></span>
        <span class="muted" id="progText" style="font-family:var(--mono);font-size:12px"></span>
      </div>

      <div class="grid-2" id="pairCards"></div>

      <div class="row" style="margin-top:14px;justify-content:space-between">
        <div class="row">
          <button class="btn" id="btnPrev">← Zurück</button>
          <button class="btn primary" id="btnNext">Weiter →</button>
        </div>
        <div class="row">
          <button class="btn" id="btnJump">Springen…</button>
          <button class="btn" id="btnExport">Export JSON</button>
          <label class="btn" style="display:inline-flex;align-items:center;gap:6px">
            Laden <input type="file" id="fileLoadSession" accept=".json" style="display:none">
          </label>
        </div>
      </div>
    </div>

    <div class="wizard" id="wizard">
      <div class="stephead">
        <div class="q" id="wizQ">—</div>
        <div class="muted" id="wizStep" style="font-family:var(--mono);font-size:11px"></div>
      </div>
      <div class="progressbar"><div class="fill" id="wizFill"></div></div>
      <div class="help" id="wizHelp">—</div>
      <details class="why"><summary>Warum diese Frage?</summary><div id="wizWhy">—</div></details>
      <div class="choices" id="wizChoices"></div>
      <div class="actions">
        <div class="left">
          <button class="btn" id="wizBack" disabled>← Schritt zurück</button>
        </div>
        <div class="right">
          <button class="btn t3" id="wizT3">Unklar/Kontext fehlt → Tier-3 (T3)</button>
          <button class="btn warn" id="wizManual">Ich sehe das anders → Label manuell</button>
        </div>
      </div>

      <div class="label-preview" id="labelPreview" style="display:none">
        <div class="lab">Vorgeschlagenes Label</div>
        <div class="val" id="lpVal">—</div>
        <div class="desc" id="lpDesc">—</div>
        <div class="path" id="lpPath">—</div>
      </div>

      <div class="panel" style="margin-top:14px;background:#0c0f16;border-style:dashed">
        <h3 style="margin-top:0">Begründung / Beleg (Pflicht)</h3>
        <label class="field">
          <textarea id="noteField" placeholder="Kurze Begründung, Quellenbeleg, Verweis auf Definition…"></textarea>
        </label>
        <div class="row" style="margin-top:8px;justify-content:space-between">
          <span class="muted" id="noteWarn" style="font-size:12px;color:var(--warn);display:none">Bitte eine Begründung angeben, bevor du speicherst/weitergehst.</span>
          <span class="spacer"></span>
          <button class="btn primary" id="btnSavePair">Antwort speichern</button>
        </div>
      </div>
    </div>
  </section>

  <!-- =================== ADJUDIKATION =================== -->
  <section id="view-adjudicate" style="display:none">
    <div class="panel">
      <h2>3. Adjudikation / Merge</h2>
      <p class="muted" style="margin-top:0">Lade die Export-JSON-Dateien zweier Rater. Konsens = gleiches Label → Gold-Label übernommen. Dissens → Differenz-Knoten anzeigen, dritte Stimme oder Tier-3 setzen.</p>
      <div class="grid-2">
        <div>
          <h3>Rater 1 (z.B. A)</h3>
          <label class="field"><input type="file" id="fileR1" accept=".json"></label>
          <label class="field" style="margin-top:8px"><textarea id="txtR1" placeholder='JSON-Export von Rater 1 hier einfügen…'></textarea></label>
        </div>
        <div>
          <h3>Rater 2 (z.B. B)</h3>
          <label class="field"><input type="file" id="fileR2" accept=".json"></label>
          <label class="field" style="margin-top:8px"><textarea id="txtR2" placeholder='JSON-Export von Rater 2 hier einfügen…'></textarea></label>
        </div>
      </div>
      <div class="row" style="margin-top:12px">
        <button class="btn primary" id="btnAdjudicate">Adjudizieren</button>
        <span class="spacer"></span>
        <span class="muted" id="adjStatus" style="font-size:12px"></span>
      </div>
    </div>

    <div class="panel" id="adjList" style="display:none">
      <div class="row" style="justify-content:space-between;align-items:center;margin-bottom:8px">
        <h2 style="margin:0">Adjudikations-Liste</h2>
        <div class="row">
          <span class="muted" id="adjCounts" style="font-size:12px"></span>
          <button class="btn" id="btnAdjExportJSON">Export pairs.json</button>
          <button class="btn" id="btnAdjExportMD">Export labeling-report.md</button>
        </div>
      </div>
      <table class="table" id="adjTable">
        <thead><tr>
          <th>pair_id</th><th>Status</th><th>r1.label</th><th>r2.label</th><th>Divergenz-Knoten</th><th>Gold</th><th>Aktion</th>
        </tr></thead>
        <tbody></tbody>
      </table>
    </div>
  </section>

  <!-- =================== REPORT =================== -->
  <section id="view-report" style="display:none">
    <div class="panel">
      <h2>4. Report</h2>
      <p class="muted" style="margin-top:0">Aggregiertes Bild aus Adjudikation. Lade zusätzlich den Datensatz <code>pairs.json</code> (Original), um die Gold-Standard-Datei anreichern zu können.</p>
      <div class="row">
        <label class="btn" style="display:inline-flex;align-items:center;gap:6px">Original pairs.json laden <input type="file" id="fileOrigPairs" accept=".json" style="display:none"></label>
        <button class="btn" id="btnReportReload">Aus localStorage neu laden</button>
        <span class="spacer"></span>
        <span class="muted" id="reportStatus" style="font-size:12px"></span>
      </div>
    </div>
    <div id="reportBody"></div>
  </section>

</div>

<!-- =================== Manual-Override Modal =================== -->
<div class="modal-bg" id="modalManual">
  <div class="modal">
    <h3>Label manuell setzen</h3>
    <p class="muted" style="font-size:12.5px">Du verlässt den Entscheidungsbaum-Pfad. Eine Begründung ist Pflicht.</p>
    <label class="field"><span>Label</span>
      <div class="radio-list" id="manualLabels">
        <!-- filled by JS -->
      </div>
    </label>
    <label class="field" style="margin-top:10px"><span>Begründung (Pflicht)</span>
      <textarea id="manualNote" placeholder="Warum weichst du vom Pfad ab?"></textarea>
    </label>
    <div class="row" style="justify-content:flex-end;margin-top:14px">
      <button class="btn" id="manualCancel">Abbrechen</button>
      <button class="btn primary" id="manualOk">Übernehmen</button>
    </div>
  </div>
</div>

<div class="toast" id="toast"></div>

<script>
/* ====================================================================
   Galadox-Label — Single-File Deterministic Gold-Standard Builder
   Offline, no network, no external resources. All state is local.
   ==================================================================== */

/* ---------- Konfiguration & Konstanten ---------- */
const LABEL_SET = ["E","S-K","S-Z","S-D","N","O","W","Q","P","K","-","T3"];

```

### Teil 3/8 (Zeilen 361–540)

```html
const LABEL_TEXT = {
  "E":   "E — Widerspruch (exklusiv): Beide können nicht zugleich wahr sein, einander logisch ausschließend.",
  "S-K": "S-K — Skopus-Konflikt: Verschiedener Gegenstand/Person/Korpus.",
  "S-Z": "S-Z — Skopus-Zeit-Konflikt: Gleicher Gegenstand, anderer Zeitraum/Prognosehorizont.",
  "S-D": "S-D — Skopus-Definition: Gleicher Gegenstand & Zeitraum, aber unterschiedliche Definition der Leitbegriffe.",
  "N":   "N — Nicht-/Unklarer Claim: Mindestens ein Text ist kein klar überprüfbarer Claim.",
  "O":   "O — Lücke/Out-of-Scope: Eine systematisch relevante Aussage fehlt im Paar.",
  "W":   "W — Wiederholung/Paraphrase: Inhaltlich identisch, nur anders formuliert.",
  "Q":   "Q — Frage statt Aussage: Ein Text stellt primär eine Frage, keine Assertion.",
  "P":   "P — Prognose-Konflikt: Gleicher Gegenstand, aber unterschiedliche Prognosehorizonte.",
  "K":   "K — Komplementär/Kontext: Beide ergänzen sich; gemeinsam wahr, ohne Widerspruch.",
  "-":   "— — Nicht vergleichbar (z.B. Gattungen, Domänen unvereinbar).",
  "T3":  "T3 — Tier-3: Unklar, Kontext fehlt, keine verlässliche Entscheidung möglich."
};

/* Entscheidungsbaum (F1..F6) */
const TREE = {
  F1: {
    q: "F1 — Sind beide Texte klare, überprüfbare Claims?",
    help: "Ein Claim = prüfbare Aussage über Welt-/Domänenzustand. Meinungen ohne Tatsachenkern, reine Appelle, reine Zitate ohne klare Aussage → kein Claim.",
    why: "Wir brauchen zwei komparable Aussagen. Wenn einer der Texte kein Claim ist, ist eine Beziehung (Widerspruch/Konsistenz/etc.) nicht definierbar.",
    yes: "F2", no: "LEAF_NWQ"  // N/W/Q/T3 — vom Nutzer zu pflücken
  },
  F2: {
    q: "F2 — Bezieht sich beide Aussagen auf denselben Gegenstand?",
    help: "Gegenstand = Person, Objekt, Konzept, Datensatz, Organisation etc., worauf sich der Claim bezieht. Verschiedene Entitäten = Skopus-Konflikt.",
    why: "Wenn A über X und B über Y spricht, ist die Frage „widersprüchlich?“ meist sinnlos.",
    yes: "F3", no: "LEAF_SKORUNDASH"
  },
  F3: {
    q: "F3 — Gleicher Zeitraum / Prognosehorizont?",
    help: "Aussagen über unterschiedliche Zeiträume (z.B. 2019 vs. 2024) oder kurz- vs. langfristige Prognosen sind nicht direkt vergleichbar.",
    why: "Selber Gegenstand kann zu unterschiedlichen Zeiten Verschiedenes geäußert haben — formal kein Widerspruch.",
    yes: "F4", no: "LEAF_SZODERP"
  },
  F4: {
    q: "F4 — Gleiche Definition der Leitbegriffe?",
    help: "Werden Schlüsselbegriffe (z.B. „Inflation“, „Armut“, „Freiheit“, „KI“) synonym oder substanziell unterschiedlich verwendet?",
    why: "Häufige Widerspruchs-Simulation: Definitionsdrift. Gleiche Wörter, anderes Konzept.",
    yes: "F5", no: "LEAF_SD"
  },
  F5: {
    q: "F5 — Können beide gleichzeitig wahr sein?",
    help: "Logische Verträglichkeit. Wenn A und B sich gegenseitig ausschließen, ist nur einer wahr.",
    why: "Wenn nein → echter exklusiver Widerspruch E. Wenn ja, prüfen wir auf Lücke O.",
    yes: "F6", no: "LEAF_E"
  },
  F6: {
    q: "F6 — Fehlt systematisch eine relevante Aussage?",
    help: "Manche Paare ergeben erst Sinn mit einer dritten Aussage (z.B. Bedingung, Vergleichswert, Gegeninstanz).",
    why: "Wenn eine solche Lücke systematisch ist, kennzeichnen wir das als Out-of-Scope / Lücke.",
    yes: "LEAF_O", no: "LEAF_KNWSK"
  }
};

/* Terminale Blätter: Mehrere erlaubte Optionen */
const LEAF_OPTIONS = {
  LEAF_NWQ:        { label: "F1 → Nein",  options: ["N","W","Q","T3"], final: true, desc: "Mindestens ein Text ist kein klarer Claim oder doppelt." },
  LEAF_SKORUNDASH: { label: "F2 → Nein",  options: ["-","S-K","T3"],   final: true, desc: "Verschiedene Gegenstände oder Domänen." },
  LEAF_SZODERP:    { label: "F3 → Nein",  options: ["S-Z","P","T3"],   final: true, desc: "Zeitraum oder Prognosehorizont weicht ab." },
  LEAF_SD:         { label: "F4 → Nein",  options: ["S-D","T3"],       final: true, desc: "Definitionsdrift der Leitbegriffe." },
  LEAF_E:          { label: "F5 → Nein",  options: ["E","T3"],         final: true, desc: "Exklusiver Widerspruch." },
  LEAF_O:          { label: "F6 → Ja",    options: ["O","T3"],         final: true, desc: "Relevante Aussage fehlt im Paar." },
  LEAF_KNWSK:      { label: "F6 → Nein",  options: ["K","N","W","S-K","-","T3"], final: true, desc: "Verträglich / komplementär / paraphrastisch." }
};

const TREE_ORDER = ["F1","F2","F3","F4","F5","F6"];

/* ---------- Deterministischer PRNG (Mulberry32) ---------- */
function mulberry32(seed){
  let a = (seed >>> 0) || 1;
  return function(){
    a |= 0; a = (a + 0x6D2B79F5) | 0;
    let t = Math.imul(a ^ (a >>> 15), 1 | a);
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t;
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
  };
}
function detShuffle(arr, seed){
  const a = arr.slice();
  const rnd = mulberry32(seed);
  for (let i = a.length - 1; i > 0; i--){
    const j = Math.floor(rnd() * (i + 1));
    [a[i], a[j]] = [a[j], a[i]];
  }
  return a;
}

/* ---------- App-State ---------- */
const STORE_KEY = "galadox-label.v0.1";
const state = {
  claims: [],
  pairs: [],
  claimsById: {},
  // session
  mode: "home",
  rater: { role: "A", init: "" },
  seed: 42,
  // order of pair_ids (deterministically shuffled)
  order: [],
  cursor: 0,
  // path/answers map: pair_id -> { answers: [{step, ans, leaf?}], label, manual, note, ts, rater, role, init }
  decisions: {},
  // adjudication
  adj: null
};

/* ---------- Storage ---------- */
function saveLocal(){
  try{
    const dump = {
      v: 1,
      savedAt: new Date().toISOString(),
      claims: state.claims,
      pairs: state.pairs,
      rater: state.rater,
      seed: state.seed,
      order: state.order,
      cursor: state.cursor,
      decisions: state.decisions,
      adj: state.adj
    };
    localStorage.setItem(STORE_KEY, JSON.stringify(dump));
    setTopMeta("autosave " + new Date().toLocaleTimeString());
  }catch(e){
    console.error("saveLocal", e);
  }
}
function loadLocal(){
  try{
    const raw = localStorage.getItem(STORE_KEY);
    if (!raw) return false;
    const d = JSON.parse(raw);
    state.claims = d.claims || [];
    state.pairs = d.pairs || [];
    state.claimsById = Object.fromEntries(state.claims.map(c => [c.id, c]));
    state.rater = d.rater || state.rater;
    state.seed = d.seed || state.seed;
    state.order = d.order || [];
    state.cursor = d.cursor || 0;
    state.decisions = d.decisions || {};
    state.adj = d.adj || null;
    return true;
  }catch(e){ console.error("loadLocal",e); return false; }
}
function resetLocal(){
  localStorage.removeItem(STORE_KEY);
}

/* ---------- Utils ---------- */
function $(sel, root=document){ return root.querySelector(sel); }
function $$(sel, root=document){ return Array.from(root.querySelectorAll(sel)); }
function toast(msg, err=false){
  const t = $("#toast");
  t.textContent = msg;
  t.classList.toggle("err", err);
  t.classList.add("show");
  clearTimeout(t._h);
  t._h = setTimeout(()=>t.classList.remove("show"), 2500);
}
function setTopMeta(s){ $("#topMeta").textContent = s; }
function escapeHtml(s){
  return String(s == null ? "" : s)
    .replace(/&/g,"&amp;").replace(/</g,"&lt;").replace(/>/g,"&gt;")
    .replace(/"/g,"&quot;").replace(/'/g,"&#39;");
}
function nowIso(){ return new Date().toISOString(); }

/* ---------- Self-Test-Set ---------- */
const SELF_TEST = {
  claims: [
    { id:"c1", text:"Berlin hat 3.850.000 Einwohnerinnen und Einwohner (Stand 31.12.2023).", typ:"Fakt", traeger:"Statistik Berlin-Brandenburg", quelle:"Pressemitteilung 12/2024", url:"https://example.org/c1", datum:"2024-01-15", gegenstand:"Berlin / Einwohnerzahl", zeitraum:"2023", definition:"Amtliche Einwohnerzahl mit Hauptwohnsitz" },
    { id:"c2", text:"Berlin zählt im Jahr 2023 rund 3,85 Millionen Menschen mit Hauptwohnsitz.", typ:"Fakt", traeger:"Statistik Berlin-Brandenburg", quelle:"Pressemitteilung 12/2024", url:"https://example.org/c2", datum:"2024-01-15", gegenstand:"Berlin / Einwohnerzahl", zeitraum:"2023", definition:"Amtliche Einwohnerzahl mit Hauptwohnsitz" },
    { id:"c3", text:"München ist die bevölkerungsreichste Stadt Deutschlands.", typ:"Fakt", traeger:"Bundesinstitut für Bau-, Stadt- und Raumforschung", quelle:"Laufende Raumbeobachtung", url:"https://example.org/c3", datum:"2022-06-01", gegenstand:"Bevölkerungsreichste Stadt Deutschlands", zeitraum:"2021", definition:"Einwohner mit Hauptwohnsitz, Stichtag 31.12." },
    { id:"c4", text:"Hamburg wird Berlin als bevölkerungsreichste Stadt Deutschlands im Jahr 2030 ablösen.", typ:"Prognose", traeger:"HWWI", quelle:"HWWI-Kurzbericht 2024", url:"https://example.org/c4", datum:"2024-09-10", gegenstand:"Bevölkerungsreichste Stadt Deutschlands", zeitraum:"2030 (Prognosehorizont)", definition:"Modellbasierte Prognose, Hauptwohnsitz" }
  ],
  pairs: [
    { pair_id:"st1", a_id:"c1", b_id:"c2" },
    { pair_id:"st2", a_id:"c3", b_id:"c4" }
  ]
```

### Teil 4/8 (Zeilen 541–720)

```html
};

/* ---------- Loading ---------- */
function ingestClaims(claims){
  if (!Array.isArray(claims)) throw new Error("claims.json: erwartet ein Array");
  // lightweight validation
  for (const c of claims){
    if (!c.id || typeof c.text !== "string") throw new Error("Claim ohne id/text: " + JSON.stringify(c).slice(0,80));
  }
  state.claims = claims;
  state.claimsById = Object.fromEntries(claims.map(c => [c.id, c]));
}
function ingestPairs(pairs){
  if (!Array.isArray(pairs)) throw new Error("pairs.json: erwartet ein Array");
  for (const p of pairs){
    if (!p.pair_id || !p.a_id || !p.b_id) throw new Error("Pair ohne pair_id/a_id/b_id: " + JSON.stringify(p).slice(0,80));
  }
  state.pairs = pairs;
  // build deterministic order
  state.order = detShuffle(pairs.map(p => p.pair_id), state.seed);
  state.cursor = 0;
  state.decisions = {};
}
function loadFromTextareas(){
  const c = $("#txtClaims").value.trim();
  const p = $("#txtPairs").value.trim();
  if (!c || !p) throw new Error("Bitte beide Textareas füllen oder Datei hochladen.");
  ingestClaims(JSON.parse(c));
  ingestPairs(JSON.parse(p));
}
function loadFromFiles(fileClaims, filePairs, cb){
  let cJson = null, pJson = null, done = 0;
  function check(){
    done++;
    if (done === 2) {
      try{
        ingestClaims(cJson);
        ingestPairs(pJson);
        cb(null);
      }catch(e){ cb(e); }
    }
  }
  const r1 = new FileReader(); r1.onload = e => { try{ cJson = JSON.parse(e.target.result); }catch(err){ cb(err); return; } check(); };
  r1.readAsText(fileClaims);
  const r2 = new FileReader(); r2.onload = e => { try{ pJson = JSON.parse(e.target.result); }catch(err){ cb(err); return; } check(); };
  r2.readAsText(filePairs);
}
function renderMeta(){
  const m = $("#metaPanel");
  if (state.pairs.length === 0){ m.style.display = "none"; return; }
  m.style.display = "";
  const nClaims = state.claims.length;
  const nPairs  = state.pairs.length;
  const decided = Object.values(state.decisions).filter(d => d && d.label).length;
  $("#metaGrid").innerHTML = `
    <div class="kpi"><div class="v">${nClaims}</div><div class="l">Claims geladen</div></div>
    <div class="kpi"><div class="v">${nPairs}</div><div class="l">Paare geladen</div></div>
    <div class="kpi"><div class="v">${decided}/${nPairs}</div><div class="l">Labels gesetzt</div></div>
  `;
}

/* ---------- Mode switching ---------- */
function setMode(m){
  state.mode = m;
  $$("#modebar button").forEach(b => b.classList.toggle("active", b.dataset.mode === m));
  for (const v of ["home","label","adjudicate","report"]) $("#view-"+v).style.display = (v===m ? "" : "none");
  if (m === "label"){ renderLabelScreen(); }
  if (m === "adjudicate"){ renderAdjudication(); }
  if (m === "report"){ renderReport(); }
  if (m === "home"){ renderMeta(); }
}

/* ===========================================================
   LABEL-MODUS
   =========================================================== */
function currentPair(){
  if (state.order.length === 0) return null;
  const pid = state.order[state.cursor];
  const def = state.pairs.find(p => p.pair_id === pid);
  return def || null;
}
function chip(label, val){
  if (val == null || val === "") return "";
  return `<span class="chip"><strong>${escapeHtml(label)}:</strong> ${escapeHtml(val)}</span>`;
}
function claimCard(side, claim){
  if (!claim) return `<div class="claim-card ${side}"><div class="head"><span class="tag">${side.toUpperCase()}</span></div><div class="text muted">— Claim ${escapeHtml(side)} fehlt —</div></div>`;
  return `
    <div class="claim-card ${side}">
      <div class="head"><span class="tag">${side.toUpperCase()} · ${escapeHtml(claim.id)}</span><span class="muted" style="font-size:11px">${escapeHtml(claim.typ || "")}</span></div>
      <div class="text">„${escapeHtml(claim.text)}"</div>
      <div class="chips">
        ${chip("träger", claim.traeger)}
        ${chip("quelle", claim.quelle)}
        ${chip("datum", claim.datum)}
        ${chip("gegenstand", claim.gegenstand)}
        ${chip("zeitraum", claim.zeitraum)}
        ${chip("definition", claim.definition)}
      </div>
    </div>`;
}

function renderPairCards(){
  const p = currentPair();
  if (!p){ $("#pairCards").innerHTML = `<div class="muted">Kein Paar verfügbar. Lade einen Datensatz im Startbildschirm.</div>`; return; }
  const a = state.claimsById[p.a_id];
  const b = state.claimsById[p.b_id];
  $("#pairCards").innerHTML = claimCard("a", a) + claimCard("b", b);
  $("#progText").textContent = `Paar ${state.cursor+1} / ${state.order.length}  ·  ${p.pair_id}`;
}

/* Wizard / Decision Tree */
let wizard = {
  step: "F1",       // current step key
  answers: [],      // [{step, ans:"yes"|"no"|"label", via:"wizard"|"t3"|"manual", leafKey?, finalLabel?}]
  currentNode: "F1" // either step key or leaf key
};

function startWizardForCurrent(){
  wizard = { step:"F1", answers:[], currentNode:"F1" };
  // restore existing decision if any
  const p = currentPair(); if (!p) return;
  const d = state.decisions[p.pair_id];
  if (d && d.answers && d.answers.length){
    // only restore wizard state if it ended on a non-final node (i.e. not a leaf)
    const last = d.answers[d.answers.length-1];
    wizard.answers = d.answers.slice();
    if (last && last.leafKey){ wizard.currentNode = last.leafKey; wizard.step = null; }
    else { wizard.step = last.step; wizard.currentNode = last.step; }
  }
  $("#noteField").value = (d && d.note) || "";
  renderWizard();
  updateLabelPreview();
}
function renderWizard(){
  const node = wizard.currentNode;
  // leaf?
  if (LEAF_OPTIONS[node]){
    renderLeaf(node);
    return;
  }
  // step
  const step = TREE[node];
  $("#wizQ").textContent = step.q;
  $("#wizHelp").textContent = step.help;
  $("#wizWhy").textContent = step.why;
  $("#wizStep").textContent = `Schritt ${TREE_ORDER.indexOf(node)+1}/${TREE_ORDER.length} · ${node}`;
  // fill
  const idx = TREE_ORDER.indexOf(node);
  $("#wizFill").style.width = (((idx+1)/TREE_ORDER.length)*100) + "%";
  // choices
  const yesLabel = step.yes.startsWith("LEAF_") ? "Ja → Blatt" : "Ja → " + step.yes;
  const noLabel  = step.no.startsWith("LEAF_")  ? "Nein → Blatt" : "Nein → " + step.no;
  $("#wizChoices").className = "choices";
  $("#wizChoices").innerHTML = `
    <button data-ans="yes" class="a"><span class="lab">JA</span>${escapeHtml(yesLabel)}<br><span class="muted" style="font-size:11.5px">${escapeHtml(yesSubhint(node))}</span></button>
    <button data-ans="no"  class="b"><span class="lab">NEIN</span>${escapeHtml(noLabel)}<br><span class="muted" style="font-size:11.5px">${escapeHtml(noSubhint(node))}</span></button>
  `;
  // back
  $("#wizBack").disabled = wizard.answers.length === 0;
  // hide label preview until leaf
  $("#labelPreview").style.display = "none";
  // wizard mode classes (highlight a vs b sides)
  const cl = $("#wizard"); cl.classList.remove("mode-final");
}
function yesSubhint(step){
  switch(step){
    case "F1": return "Beide sind prüfbare Aussagen über Welt-/Domänen­zustand.";
    case "F2": return "Selber Gegenstand, Person, Datensatz, Konzept.";
    case "F3": return "Selber Berichts- oder Prognose­zeitraum.";
    case "F4": return "Leitbegriffe werden substanziell gleich verwendet.";
    case "F5": return "Beide Aussagen sind logisch verträglich.";
    case "F6": return "Das Paar wirkt vollständig, keine Lücke.";
  }
  return "";
}
function noSubhint(step){
  switch(step){
    case "F1": return "Einer ist Meinung/Frage/unklar → N/W/Q/T3.";
    case "F2": return "Verschiedene Gegenstände/Domänen → -/S-K/T3.";
```

### Teil 5/8 (Zeilen 721–900)

```html
    case "F3": return "Unterschiedlicher Zeitraum/Horizont → S-Z/P/T3.";
    case "F4": return "Begriffsdrift erkennbar → S-D/T3.";
    case "F5": return "Exklusiver Widerspruch → E/T3.";
    case "F6": return "Systematische Lücke → O/T3.";
  }
  return "";
}

function renderLeaf(leafKey){
  const leaf = LEAF_OPTIONS[leafKey];
  $("#wizQ").textContent = "Pfad-Ende: " + leaf.label;
  $("#wizHelp").textContent = "Wähle das passende finale Label aus den erlaubten Optionen für diesen Pfad:";
  $("#wizWhy").textContent = leaf.desc;
  $("#wizStep").textContent = "Abschluss";
  $("#wizFill").style.width = "100%";
  $("#wizChoices").className = "choices single";
  $("#wizChoices").innerHTML = leaf.options.map(opt => `
    <button data-final="${opt}">
      <span class="lab">${escapeHtml(opt)} · ${escapeHtml(LABEL_TEXT[opt].split(" — ")[0])}</span>
      ${escapeHtml(LABEL_TEXT[opt])}
    </button>
  `).join("");
  $("#wizBack").disabled = wizard.answers.length === 0;
}

function answerWizard(ans){
  const node = wizard.currentNode;
  if (LEAF_OPTIONS[node]) return; // shouldn't happen
  const step = TREE[node];
  const next = ans === "yes" ? step.yes : step.no;
  wizard.answers.push({ step: node, ans });
  if (next.startsWith("LEAF_")){
    wizard.currentNode = next;
    // record leaf
    wizard.answers[wizard.answers.length-1].leafKey = next;
    renderWizard();
  } else {
    wizard.currentNode = next;
    renderWizard();
  }
}
function finalizeLeaf(label){
  const p = currentPair(); if (!p) return;
  const last = wizard.answers[wizard.answers.length-1];
  if (last) last.finalLabel = label;
  updateLabelPreview(); // zeigt Label auch vor dem Persistieren (Wizard-Fallback)
  const note = $("#noteField").value.trim();
  if (!note){
    // Note ist Pflicht: KEIN Speichern und KEIN Auto-Advance
    $("#noteField").classList.add("required-missing");
    $("#noteWarn").style.display = "";
    const nf = $("#noteField"); if (nf) nf.focus();
    toast("Bitte zuerst Begründung eingeben", true);
    return;
  }
  $("#noteField").classList.remove("required-missing");
  $("#noteWarn").style.display = "none";
  persistDecision();
  toast("Label gesetzt: " + label);
  // move to next pair automatically after short delay
  setTimeout(() => { if (state.cursor < state.order.length - 1){ state.cursor++; startWizardForCurrent(); renderPairCards(); saveLocal(); } }, 600);
}
function persistDecision(){
  const p = currentPair(); if (!p) return;
  const note = $("#noteField").value.trim();
  const last = wizard.answers[wizard.answers.length-1];
  const label = (last && last.finalLabel) || (wizard.answers.find(a => a.finalLabel) || {}).finalLabel || null;
  const d = state.decisions[p.pair_id] || {};
  d.rater = state.rater.role;
  d.init  = state.rater.init;
  d.answers = wizard.answers.slice();
  d.label = label;
  d.manual = wizard.answers.some(a => a.via === "manual");
  d.note = note;
  d.ts = nowIso();
  state.decisions[p.pair_id] = d;
  saveLocal();
  renderMeta();
  if (window._onDecisionChanged) window._onDecisionChanged();
}
function updateLabelPreview(){
  const p = currentPair(); if (!p) return;
  const d = state.decisions[p.pair_id];
  const lab = d && d.label;
  const lp = $("#labelPreview");
  if (!lab){ lp.style.display = "none"; return; }
  lp.style.display = "";
  $("#lpVal").textContent = lab + " — " + LABEL_TEXT[lab].split(" — ")[0].slice(2).trim();
  $("#lpDesc").textContent = LABEL_TEXT[lab];
  $("#lpPath").textContent = "Pfad: " + (d.answers||[]).map(a => `${a.step||""}${a.ans?":"+a.ans:""}${a.finalLabel?("→"+a.finalLabel):""}`).join("  →  ");
}

function renderLabelScreen(){
  const empty = state.pairs.length === 0;
  // Leerzustand vs. Arbeits-Oberfläche umschalten — niemals innerHTML überschreiben (zerstört sonst #wizard)
  $("#labelEmpty").style.display = empty ? "" : "none";
  $("#labelHead").style.display  = empty ? "none" : "";
  $("#wizard").style.display     = empty ? "none" : "";
  if (empty){
    const gh = $("#goHome"); if (gh) gh.onclick = e => { e.preventDefault(); setMode("home"); };
    return;
  }
  // re-bind header controls
  $("#raterRole").value = state.rater.role;
  $("#raterInit").value = state.rater.init || "";
  $("#seed").value = state.seed;
  renderPairCards();
  startWizardForCurrent();
}

/* ---------- Event wiring ---------- */
function wireEvents(){
  // modebar
  $$("#modebar button").forEach(b => b.addEventListener("click", () => setMode(b.dataset.mode)));

  // file inputs -> textareas (just fill the textarea; actual parsing on "Übernehmen")
  function bindFile(inputId, taId){
    $("#"+inputId).addEventListener("change", e => {
      const f = e.target.files[0]; if (!f) return;
      const r = new FileReader();
      r.onload = ev => { $("#"+taId).value = ev.target.result; };
      r.readAsText(f);
    });
  }
  bindFile("fileClaims","txtClaims");
  bindFile("filePairs","txtPairs");
  bindFile("fileR1","txtR1");
  bindFile("fileR2","txtR2");
  bindFile("fileLoadSession","_skip");
  $("#fileLoadSession").addEventListener("change", e => {
    const f = e.target.files[0]; if (!f) return;
    const r = new FileReader();
    r.onload = ev => {
      try{
        const d = JSON.parse(ev.target.result);
        if (d.claims) state.claims = d.claims;
        if (d.pairs)  state.pairs  = d.pairs;
        state.claimsById = Object.fromEntries(state.claims.map(c => [c.id,c]));
        if (d.rater) state.rater = d.rater;
        if (typeof d.seed === "number") state.seed = d.seed;
        if (Array.isArray(d.order) && d.order.length){
          state.order = d.order;
        } else {
          state.order = detShuffle(state.pairs.map(p => p.pair_id), state.seed);
        }
        if (typeof d.cursor === "number") state.cursor = d.cursor;
        if (d.decisions) state.decisions = d.decisions;
        if (d.adj) state.adj = d.adj;
        saveLocal();
        toast("Session geladen · " + state.pairs.length + " Paare");
        setTopMeta("Session geladen");
        renderMeta();
        if (state.mode === "label"){ renderLabelScreen(); }
        if (state.mode === "adjudicate"){ renderAdjudication(); }
        if (state.mode === "report"){ renderReport(); }
      }catch(err){ toast("Session-Import fehlgeschlagen: " + err.message, true); }
    };
    r.readAsText(f);
  });
  $("#fileOrigPairs").addEventListener("change", e => {
    const f = e.target.files[0]; if (!f) return;
    const r = new FileReader();
    r.onload = ev => {
      try{
        const p = JSON.parse(ev.target.result);
        state.origPairs = p;
        toast("Original pairs.json geladen ("+p.length+" Paare)");
        renderReport();
      }catch(err){ toast("Konnte pairs.json nicht lesen: " + err.message, true); }
    };
    r.readAsText(f);
  });

  // load buttons
  $("#btnLoad").addEventListener("click", () => {
    try{
      loadFromTextareas();
      saveLocal();
      toast("Datensatz geladen · " + state.pairs.length + " Paare");
      setTopMeta(state.pairs.length + " Paare · " + state.claims.length + " Claims");
```

### Teil 6/8 (Zeilen 901–1080)

```html
      renderMeta();
    }catch(e){ toast("Fehler: " + e.message, true); }
  });
  $("#btnLoadSelf").addEventListener("click", () => {
    try{
      ingestClaims(SELF_TEST.claims);
      ingestPairs(SELF_TEST.pairs);
      saveLocal();
      toast("Mini-Selbsttest geladen (2 Paare)");
      setTopMeta("Selbsttest · 2 Paare");
      renderMeta();
    }catch(e){ toast("Fehler: " + e.message, true); }
  });
  $("#btnLoadLast").addEventListener("click", () => {
    if (loadLocal()){
      toast("Letzter Stand wiederhergestellt");
      setTopMeta("Letzter Stand · " + state.pairs.length + " Paare");
      renderMeta();
    } else {
      toast("Kein gespeicherter Stand gefunden", true);
    }
  });

  // rater + seed change
  $("#raterRole").addEventListener("change", e => { state.rater.role = e.target.value; saveLocal(); });
  $("#raterInit").addEventListener("input",  e => { state.rater.init = e.target.value.trim(); saveLocal(); });
  $("#seed").addEventListener("change",     e => {
    const v = parseInt(e.target.value, 10);
    if (isNaN(v)) return;
    const n = Object.keys(state.decisions).length;
    if (n > 0 && !confirm("Achtung: Alle " + n + " Entscheidungen gehen verloren. Trotzdem neuen Seed setzen?")){
      e.target.value = state.seed; // alten Seed wiederherstellen
      return;
    }
    state.seed = v;
    state.order = detShuffle(state.pairs.map(p => p.pair_id), state.seed);
    state.cursor = 0;
    state.decisions = {};
    saveLocal();
    renderLabelScreen();
    toast("Seed=" + v + " · neue deterministische Reihenfolge");
  });

  // nav
  $("#btnPrev").addEventListener("click", () => {
    if (state.cursor > 0){ state.cursor--; saveLocal(); startWizardForCurrent(); renderPairCards(); }
  });
  $("#btnNext").addEventListener("click", () => {
    const note = $("#noteField").value.trim();
    if (!note){
      $("#noteField").classList.add("required-missing");
      $("#noteWarn").style.display = "";
      toast("Bitte zuerst Begründung eingeben", true);
      return;
    }
    persistDecision();
    if (state.cursor < state.order.length - 1){ state.cursor++; startWizardForCurrent(); renderPairCards(); }
    else { toast("Letztes Paar erreicht"); }
  });
  $("#btnJump").addEventListener("click", () => {
    const inp = prompt("Springe zu Paar-Index (1.." + state.order.length + "):", String(state.cursor+1));
    if (!inp) return;
    const n = parseInt(inp, 10);
    if (isNaN(n) || n < 1 || n > state.order.length){ toast("Ungültiger Index", true); return; }
    state.cursor = n - 1; saveLocal(); startWizardForCurrent(); renderPairCards();
  });

  // export session
  $("#btnExport").addEventListener("click", () => {
    const dump = {
      v:1, savedAt: nowIso(),
      claims: state.claims, pairs: state.pairs,
      rater: state.rater, seed: state.seed,
      order: state.order, cursor: state.cursor,
      decisions: state.decisions, adj: state.adj
    };
    const name = `galadox-rater-${state.rater.role}-${(state.rater.init||"x")}-${Date.now()}.json`;
    downloadBlob(JSON.stringify(dump, null, 2), name, "application/json");
    toast("Export erstellt: " + name);
  });

  // wizard
  $("#wizard").addEventListener("click", e => {
    const btn = e.target.closest("button"); if (!btn) return;
    if (btn.dataset.ans){
      answerWizard(btn.dataset.ans);
    } else if (btn.dataset.final){
      finalizeLeaf(btn.dataset.final);
    }
  });
  $("#wizBack").addEventListener("click", () => {
    if (wizard.answers.length === 0) return;
    wizard.answers.pop();
    // recompute current node
    if (wizard.answers.length === 0){
      wizard.currentNode = "F1";
    } else {
      const last = wizard.answers[wizard.answers.length-1];
      wizard.currentNode = last.leafKey || last.step;
    }
    renderWizard();
  });
  $("#wizT3").addEventListener("click", () => {
    // jump to terminal T3
    const note = $("#noteField").value.trim();
    if (!note){
      $("#noteField").classList.add("required-missing");
      $("#noteWarn").style.display = "";
      toast("Bitte zuerst Begründung eingeben", true);
      return;
    }
    // synthesize a minimal path -> T3
    wizard.answers.push({ step: wizard.currentNode.startsWith("LEAF_") ? "T3" : wizard.currentNode, ans: "t3", via:"t3" });
    wizard.answers[wizard.answers.length-1].leafKey = "T3-FORCED";
    wizard.answers[wizard.answers.length-1].finalLabel = "T3";
    persistDecision();
    updateLabelPreview();
    toast("Tier-3 (T3) gesetzt");
    setTimeout(() => { if (state.cursor < state.order.length - 1){ state.cursor++; startWizardForCurrent(); renderPairCards(); } }, 500);
  });
  $("#wizManual").addEventListener("click", () => openManualModal());

  // save pair (button)
  $("#btnSavePair").addEventListener("click", () => {
    const note = $("#noteField").value.trim();
    if (!note){
      $("#noteField").classList.add("required-missing");
      $("#noteWarn").style.display = "";
      toast("Bitte zuerst Begründung eingeben", true);
      return;
    }
    $("#noteField").classList.remove("required-missing");
    $("#noteWarn").style.display = "none";
    persistDecision();
  });
  $("#noteField").addEventListener("input", () => {
    $("#noteField").classList.remove("required-missing");
    $("#noteWarn").style.display = "none";
  });

  // manual modal
  const ml = $("#manualLabels");
  ml.innerHTML = LABEL_SET.map(l => `<label><input type="radio" name="ml" value="${l}"><span>${l} · ${escapeHtml(LABEL_TEXT[l].split(" — ")[0])}</span></label>`).join("");
  $("#manualCancel").addEventListener("click", () => $("#modalManual").classList.remove("show"));
  $("#manualOk").addEventListener("click", () => {
    const sel = ml.querySelector("input:checked");
    const note = $("#manualNote").value.trim();
    if (!sel){ toast("Bitte Label wählen", true); return; }
    if (!note){ toast("Begründung ist Pflicht", true); return; }
    // record as manual override
    wizard.answers.push({ step: "MANUAL", ans: "manual", via:"manual", finalLabel: sel.value });
    $("#noteField").value = note;
    persistDecision();
    updateLabelPreview();
    $("#modalManual").classList.remove("show");
    $("#manualNote").value = "";
    toast("Manuelles Label gesetzt: " + sel.value);
  });

  // adjudication
  $("#btnAdjudicate").addEventListener("click", () => {
    try{
      const a = $("#txtR1").value.trim();
      const b = $("#txtR2").value.trim();
      if (!a || !b) throw new Error("Bitte beide Rater-Exporte angeben.");
      const r1 = JSON.parse(a), r2 = JSON.parse(b);
      state.adj = adjudicate(r1, r2);
      state.adj.adj_meta = { adjudicator: null, ts: nowIso() };
      saveLocal();
      renderAdjudication();
      toast("Adjudikation berechnet");
    }catch(e){ toast("Fehler: " + e.message, true); }
  });
  $("#btnAdjExportJSON").addEventListener("click", () => exportAdjPairs());
  $("#btnAdjExportMD").addEventListener("click", () => exportAdjReport());

  // report
  $("#btnReportReload").addEventListener("click", () => renderReport());

  // prevent "kein Datensatz" link from breaking
```

### Teil 7/8 (Zeilen 1081–1260)

```html
}

/* ---------- Download helper ---------- */
function downloadBlob(text, name, mime){
  const blob = new Blob([text], { type: mime });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url; a.download = name;
  document.body.appendChild(a); a.click();
  setTimeout(() => { URL.revokeObjectURL(url); a.remove(); }, 200);
}

/* ===========================================================
   ADJUDIKATION
   =========================================================== */
function adjudicate(r1, r2){
  // ensure claims
  const claims = r1.claims || r2.claims || state.claims;
  const cById = Object.fromEntries((claims || []).map(c => [c.id, c]));
  const map1 = r1.decisions || {};
  const map2 = r2.decisions || {};
  const pairIds = Array.from(new Set([...Object.keys(map1), ...Object.keys(map2)]));
  const items = pairIds.map(pid => {
    const d1 = map1[pid] || null;
    const d2 = map2[pid] || null;
    const label1 = d1 ? d1.label : null;
    const label2 = d2 ? d2.label : null;
    let status, gold_label=null, tier3=false, divergence=null;
    if (!d1 || !d2){ status = "incomplete"; }
    else if (label1 === label2){ status = "consensus"; gold_label = label1; }
    else {
      status = "dissent";
      // find first diverging step (or finalLabel if paths match but finals differ)
      const a1 = (d1.answers || []).map(a => a.step + (a.ans?":"+a.ans:"") + (a.finalLabel?("→"+a.finalLabel):""));
      const a2 = (d2.answers || []).map(a => a.step + (a.ans?":"+a.ans:"") + (a.finalLabel?("→"+a.finalLabel):""));
      const max = Math.max(a1.length, a2.length);
      for (let i=0;i<max;i++){
        if (a1[i] !== a2[i]){ divergence = { index:i, r1:a1[i]||null, r2:a2[i]||null }; break; }
      }
      if (!divergence && label1 !== label2){
        divergence = { index: Math.max(a1.length, a2.length)-1, r1: a1.at(-1) || label1, r2: a2.at(-1) || label2 };
      }
    }
    return {
      pair_id: pid,
      a_id: (state.pairs.find(p => p.pair_id === pid) || {}).a_id || null,
      b_id: (state.pairs.find(p => p.pair_id === pid) || {}).b_id || null,
      r1: d1 ? { rater: d1.rater, init: d1.init, label: d1.label, path: (d1.answers||[]).map(a => a.step + (a.ans?":"+a.ans:"") + (a.finalLabel?("→"+a.finalLabel):"")), note: d1.note, ts: d1.ts } : null,
      r2: d2 ? { rater: d2.rater, init: d2.init, label: d2.label, path: (d2.answers||[]).map(a => a.step + (a.ans?":"+a.ans:"") + (a.finalLabel?("→"+a.finalLabel):"")), note: d2.note, ts: d2.ts } : null,
      status, gold_label, tier3, divergence,
      adjudication: null
    };
  });
  return { items, r1_meta: r1.rater, r2_meta: r2.rater, claims };
}

function renderAdjudication(){
  if (!state.adj){ $("#adjList").style.display = "none"; $("#adjStatus").textContent = ""; return; }
  $("#adjList").style.display = "";
  const items = state.adj.items;
  const counts = items.reduce((a,i) => (a[i.status]=(a[i.status]||0)+1, a), {});
  $("#adjStatus").textContent = `${items.length} Paare geladen · ${counts.consensus||0} Konsens · ${counts.dissent||0} Dissens · ${counts.incomplete||0} unvollständig`;
  $("#adjCounts").textContent = `Konsens ${counts.consensus||0} · Dissens ${counts.dissent||0} · T3 ${items.filter(i=>i.tier3).length} · Gold ${items.filter(i=>i.gold_label).length}`;
  const tb = $("#adjTable tbody");
  tb.innerHTML = "";
  for (const it of items){
    const tr = document.createElement("tr");
    const tag = it.status === "consensus"
      ? `<span class="tag-mini consensus">Konsens</span>`
      : it.status === "dissent"
        ? `<span class="tag-mini dissent">Dissens</span>`
        : `<span class="tag-mini">Unvollst.</span>`;
    if (it.tier3) tr.innerHTML = `<td>${escapeHtml(it.pair_id)}</td><td>${tag} <span class="tag-mini t3">T3</span></td><td>${escapeHtml(it.r1?.label||"—")}</td><td>${escapeHtml(it.r2?.label||"—")}</td><td>—</td><td>—</td><td>—</td>`;
    else if (it.status === "consensus") tr.innerHTML = `<td>${escapeHtml(it.pair_id)}</td><td>${tag}</td><td>${escapeHtml(it.r1?.label||"—")}</td><td>${escapeHtml(it.r2?.label||"—")}</td><td class="muted">—</td><td><strong>${escapeHtml(it.gold_label||"—")}</strong></td><td class="muted">auto</td>`;
    else {
      // dissent: render side-by-side paths with divergence highlight
      const div = it.divergence || {};
      const pathCell = `
        <div class="grid-2" style="gap:6px;font-size:11.5px">
          <div>
            <div class="muted" style="font-size:10.5px">Rater 1</div>
            ${(it.r1?.path||[]).map((p,i) => `<div class="${i===div.index?'divnode a':''}">${escapeHtml(p)}</div>`).join("")}
          </div>
          <div>
            <div class="muted" style="font-size:10.5px">Rater 2</div>
            ${(it.r2?.path||[]).map((p,i) => `<div class="${i===div.index?'divnode b':''}">${escapeHtml(p)}</div>`).join("")}
          </div>
        </div>
        <div class="muted" style="font-size:10.5px;margin-top:4px">erster abweichender Knoten: <strong>${escapeHtml((div.r1||"—")+"  ↔  "+(div.r2||"—"))}</strong></div>
      `;
      const goldCell = it.gold_label ? `<strong>${escapeHtml(it.gold_label)}</strong>` : `<span class="muted">offen</span>`;
      const action = `
        <div class="row" style="gap:4px">
          <select data-pid="${escapeHtml(it.pair_id)}" class="goldSel">
            <option value="">— Gold setzen —</option>
            ${LABEL_SET.map(l => `<option value="${l}" ${it.gold_label===l?"selected":""}>${l}</option>`).join("")}
          </select>
          <button class="btn t3" data-act="t3" data-pid="${escapeHtml(it.pair_id)}" ${it.tier3?"disabled":""}>T3</button>
        </div>`;
      tr.innerHTML = `<td>${escapeHtml(it.pair_id)}</td><td>${tag}</td><td>${escapeHtml(it.r1?.label||"—")}</td><td>${escapeHtml(it.r2?.label||"—")}</td><td>${pathCell}</td><td>${goldCell}</td><td>${action}</td>`;
    }
    tb.appendChild(tr);
  }
  // wire actions
  $$(".goldSel", tb).forEach(sel => sel.addEventListener("change", e => {
    const pid = sel.dataset.pid;
    const v = sel.value;
    const it = state.adj.items.find(x => x.pair_id === pid); if (!it) return;
    it.gold_label = v || null; it.tier3 = false;
    it.adjudication = { who: state.adj.adj_meta?.adjudicator || "adjudicator", decision: "set_gold", label: v, note: "", ts: nowIso() };
    saveLocal(); renderAdjudication();
  }));
  $$("button[data-act='t3']", tb).forEach(b => b.addEventListener("click", () => {
    const pid = b.dataset.pid;
    const it = state.adj.items.find(x => x.pair_id === pid); if (!it) return;
    it.tier3 = true; it.gold_label = null;
    it.adjudication = { who: state.adj.adj_meta?.adjudicator || "adjudicator", decision: "tier3", note: "Konsens nicht erzwingbar", ts: nowIso() };
    saveLocal(); renderAdjudication();
  }));
}

function exportAdjPairs(){
  if (!state.adj) { toast("Keine Adjudikation", true); return; }
  // build pairs.json enriched (v0.1 schema)
  const items = state.adj.items.map(it => {
    const labels = {
      r1: it.r1 ? { rater: it.r1.rater, init: it.r1.init, label: it.r1.label, path: it.r1.path, note: it.r1.note, ts: it.r1.ts } : null,
      r2: it.r2 ? { rater: it.r2.rater, init: it.r2.init, label: it.r2.label, path: it.r2.path, note: it.r2.note, ts: it.r2.ts } : null
    };
    return {
      pair_id: it.pair_id,
      a_id: it.a_id, b_id: it.b_id,
      labels,
      konsens: it.status === "consensus",
      gold_label: it.gold_label || null,
      tier3: !!it.tier3,
      adjudication: it.adjudication || null
    };
  });
  // merge with original pairs if loaded
  let out = items;
  if (Array.isArray(state.origPairs) && state.origPairs.length){
    const map = Object.fromEntries(items.map(i => [i.pair_id, i]));
    out = state.origPairs.map(p => map[p.pair_id] ? { ...p, ...map[p.pair_id] } : p);
  }
  const claims = state.adj.claims || state.claims;
  const dump = {
    v:"0.1", generated_at: nowIso(),
    r1_meta: state.adj.r1_meta, r2_meta: state.adj.r2_meta,
    claims, pairs: out
  };
  downloadBlob(JSON.stringify(dump, null, 2), "pairs.json", "application/json");
  toast("pairs.json exportiert");
}
function exportAdjReport(){
  if (!state.adj) { toast("Keine Adjudikation", true); return; }
  const items = state.adj.items;
  const n = items.length;
  const cons = items.filter(i => i.status === "consensus");
  const dis  = items.filter(i => i.status === "dissent");
  const t3   = items.filter(i => i.tier3);
  const gold = items.filter(i => i.gold_label);
  // Cohen's Kappa on non-T3, non-incomplete
  const usable = items.filter(i => i.r1 && i.r2 && !i.tier3);
  const kappa = cohensKappa(usable.map(i => [i.r1.label, i.r2.label]));

  let md = "";
  md += `# Galadox-Label · Adjudikations-/Labeling-Report\n\n`;
  md += `* generiert: ${new Date().toISOString()}\n`;
  md += `* Rater 1: ${state.adj.r1_meta?.role || "?"} · ${state.adj.r1_meta?.init || "?"}\n`;
  md += `* Rater 2: ${state.adj.r2_meta?.role || "?"} · ${state.adj.r2_meta?.init || "?"}\n\n`;
  md += `## Kennzahlen\n\n`;
  md += `| Metrik | Wert |\n|---|---|\n`;
  md += `| Paare gesamt | ${n} |\n`;
  md += `| Konsens | ${cons.length} |\n`;
  md += `| Dissens | ${dis.length} |\n`;
  md += `| Tier-3 gesetzt | ${t3.length} |\n`;
  md += `| Gold-Label gesetzt | ${gold.length} |\n`;
  md += `| Cohen's κ (Rater 1 ↔ 2, ohne T3) | ${kappa === null ? "n/a" : kappa.toFixed(3)} |\n\n`;
  md += `## Tier-3-Liste\n\n`;
```

### Teil 8/8 (Zeilen 1261–1403)

```html
  if (t3.length === 0) md += `_(keine)_\n\n`;
  else {
    md += `| pair_id | r1.label | r2.label | Begründung |\n|---|---|---|---|\n`;
    for (const it of t3){
      md += `| ${it.pair_id} | ${it.r1?.label || "—"} | ${it.r2?.label || "—"} | ${escapeMd((it.adjudication?.note||""))} |\n`;
    }
    md += `\n`;
  }
  md += `## Dissens-Tabelle (mit Divergenz-Knoten)\n\n`;
  if (dis.length === 0) md += `_(keine)_\n\n`;
  else {
    md += `| pair_id | r1.label | r2.label | Divergenz @ Knoten | r1 → r2 Pfad |\n|---|---|---|---|---|\n`;
    for (const it of dis){
      const d = it.divergence || {};
      const path = (it.r1?.path||[]).join(" → ") + "  ||  " + (it.r2?.path||[]).join(" → ");
      md += `| ${it.pair_id} | ${it.r1?.label || "—"} | ${it.r2?.label || "—"} | ${escapeMd((d.r1||"—")+"  ↔  "+(d.r2||"—"))} | ${escapeMd(path)} |\n`;
    }
    md += `\n`;
  }
  md += `## Gold-Standard (Konsens + adjudiziert)\n\n`;
  md += `| pair_id | gold_label | tier3 | adjudication |\n|---|---|---|---|\n`;
  for (const it of items){
    md += `| ${it.pair_id} | ${it.gold_label || "—"} | ${it.tier3 ? "T3" : ""} | ${it.adjudication ? escapeMd(JSON.stringify(it.adjudication)) : ""} |\n`;
  }
  md += `\n`;
  downloadBlob(md, "labeling-report.md", "text/markdown");
  toast("labeling-report.md exportiert");
}
function escapeMd(s){ return String(s||"").replace(/\|/g,"\\|").replace(/\n/g," "); }

/* ===========================================================
   COHEN'S KAPPA
   =========================================================== */
function cohensKappa(pairs){
  if (!pairs || pairs.length === 0) return null;
  const labels = Array.from(new Set(pairs.flatMap(p => p)));
  const N = pairs.length;
  // observed agreement
  let po = 0;
  for (const [a,b] of pairs) if (a===b) po++;
  po /= N;
  // expected
  const counts = Object.fromEntries(labels.map(l => [l, 0]));
  for (const [a,b] of pairs){ counts[a]++; counts[b]++; }
  let pe = 0;
  for (const l of labels){
    const p = counts[l] / (2*N);
    pe += p*p;
  }
  if (pe === 1) return 1; // perfect baseline; no disagreement possible
  return (po - pe) / (1 - pe);
}

/* ===========================================================
   REPORT
   =========================================================== */
function renderReport(){
  const body = $("#reportBody");
  body.innerHTML = "";
  if (!state.adj){
    body.innerHTML = `<div class="panel"><p class="muted">Noch keine Adjudikation. Wechsle in den <a href="#" id="goAdj">Adjudikations-Modus</a>.</p></div>`;
    $("#goAdj").onclick = e => { e.preventDefault(); setMode("adjudicate"); };
    return;
  }
  const items = state.adj.items;
  const n = items.length;
  const cons = items.filter(i => i.status === "consensus");
  const dis  = items.filter(i => i.status === "dissent");
  const t3   = items.filter(i => i.tier3);
  const gold = items.filter(i => i.gold_label);
  const usable = items.filter(i => i.r1 && i.r2 && !i.tier3);
  const kappa = cohensKappa(usable.map(i => [i.r1.label, i.r2.label]));

  const kpi = `
    <div class="grid-3" style="margin-top:14px">
      <div class="kpi"><div class="v">${n}</div><div class="l">Paare</div></div>
      <div class="kpi"><div class="v">${cons.length}</div><div class="l">Konsens</div></div>
      <div class="kpi"><div class="v">${dis.length}</div><div class="l">Dissens</div></div>
      <div class="kpi"><div class="v">${t3.length}</div><div class="l">Tier-3</div></div>
      <div class="kpi"><div class="v">${gold.length}</div><div class="l">Gold gesetzt</div></div>
      <div class="kpi"><div class="v">${kappa === null ? "—" : kappa.toFixed(3)}</div><div class="l">Cohen's κ (ohne T3)</div></div>
    </div>
  `;
  let html = `<div class="panel">${kpi}</div>`;

  // Tier-3 list
  html += `<div class="panel"><h2>Tier-3-Liste</h2>`;
  if (t3.length === 0) html += `<p class="muted">Keine Tier-3-Paare.</p>`;
  else html += `<table class="table"><thead><tr><th>pair_id</th><th>r1.label</th><th>r2.label</th><th>Adjudikation</th></tr></thead><tbody>`+
    t3.map(i => `<tr><td>${escapeHtml(i.pair_id)}</td><td>${escapeHtml(i.r1?.label||"—")}</td><td>${escapeHtml(i.r2?.label||"—")}</td><td>${i.adjudication?escapeHtml(JSON.stringify(i.adjudication)):"<span class='muted'>—</span>"}</td></tr>`).join("") + `</tbody></table>`;
  html += `</div>`;

  // Dissent table
  html += `<div class="panel"><h2>Dissens-Tabelle (mit Divergenz-Knoten)</h2>`;
  if (dis.length === 0) html += `<p class="muted">Keine Dissens-Paare.</p>`;
  else html += `<table class="table"><thead><tr><th>pair_id</th><th>r1</th><th>r2</th><th>Divergenz-Knoten</th><th>gold</th></tr></thead><tbody>`+
    dis.map(i => {
      const d = i.divergence || {};
      return `<tr>
        <td>${escapeHtml(i.pair_id)}</td>
        <td><span class="tag-mini">${escapeHtml(i.r1?.label||"—")}</span></td>
        <td><span class="tag-mini">${escapeHtml(i.r2?.label||"—")}</span></td>
        <td><div class="divnode a">${escapeHtml(d.r1||"—")}</div><div class="divnode b" style="margin-top:3px">${escapeHtml(d.r2||"—")}</div></td>
        <td>${i.gold_label ? `<strong>${escapeHtml(i.gold_label)}</strong>` : (i.tier3?`<span class="tag-mini t3">T3</span>`:`<span class="muted">offen</span>`)}</td>
      </tr>`;
    }).join("") + `</tbody></table>`;
  html += `</div>`;
  body.innerHTML = html;
}

/* ---------- Manual override modal ---------- */
function openManualModal(){
  $("#modalManual").classList.add("show");
}

/* ---------- Init ---------- */
function init(){
  wireEvents();
  // try to restore last session silently (no toast)
  if (localStorage.getItem(STORE_KEY)){
    loadLocal();
    setTopMeta("Letzter Stand · " + state.pairs.length + " Paare");
    renderMeta();
  }
  setMode("home");

  // global hook for adjudication live-update after persistDecision in label mode
  window._onDecisionChanged = () => { /* placeholder; could re-render report */ };

  // expose tiny self-test runner (idempotency)
  // Returns the deterministic order for the given ids + seed.
  // Empirical reference for seed=42: ["p1","p2","p3","p4","p5"] -> ["p1","p5","p3","p2","p4"]
  window.galadoxSelfTest = function(ids, seed){
    seed = (seed == null) ? 42 : seed;
    return detShuffle(ids, seed);
  };
}

/* boot */
document.addEventListener("DOMContentLoaded", init);
</script>
</body>
</html>
```

---

## 5. Beispieldaten

### Minimales `pairs.json` (2 Paare) zum Selbsttest

```json
{
  "v": "0.1",
  "claims": [
    { "id": "c1", "text": "Die Arbeitslosigkeit sinkt 2024 auf 4,1 %.", "träger": "Bundesagentur", "quelle": "Monatsbericht", "datum": "2024-03-01", "gegenstand": "Arbeitslosigkeit D", "zeitraum": "2024", "definition": "Jahresdurchschnitt" },
    { "id": "c2", "text": "Die Arbeitslosigkeit steigt 2024 auf über 6 %.", "träger": "Gewerkschaft", "quelle": "Studie", "datum": "2024-04-01", "gegenstand": "Arbeitslosigkeit D", "zeitraum": "2024", "definition": "Jahresdurchschnitt" },
    { "id": "c3", "text": "Ist die Inflation 2024 gebremst?", "träger": "n/a", "quelle": "Interview", "datum": "2024-05-01", "gegenstand": "Inflation", "zeitraum": "2024", "definition": "VPI" },
    { "id": "c4", "text": "Die Inflation dürfte 2024 bei 2,3 % liegen.", "träger": "Ifo", "quelle": "Prognose", "datum": "2024-06-01", "gegenstand": "Inflation", "zeitraum": "2024", "definition": "VPI" }
  ],
  "pairs": [
    { "pair_id": "p1", "a_id": "c1", "b_id": "c2" },
    { "pair_id": "p2", "a_id": "c3", "b_id": "c4" }
  ]
}
```

**Erwartetes Labeling (Beispiel):** p1 → `E` (Widerspruch, gleicher Gegenstand & Zeitraum); p2 → `N`/`Q` (c3 ist Frage, keine prüfbare Aussage).

### Beispiel-Export Rater A (`galadox-rater-A-jm-…json`)

```json
{
  "v": 1,
  "savedAt": "2026-07-02T11:50:00.000Z",
  "rater": { "role": "A", "init": "jm" },
  "seed": 42,
  "order": ["p1", "p2"],
  "cursor": 1,
  "decisions": {
    "p1": { "rater": "A", "init": "jm", "label": "E", "manual": false, "note": "Direkter Widerspruch, gleicher Gegenstand/Zeitraum.", "answers": [
      { "step": "F1", "ans": "yes" },
      { "step": "F2", "ans": "yes" },
      { "step": "F3", "ans": "yes" },
      { "step": "F4", "ans": "yes" },
      { "step": "F5", "ans": "no", "leafKey": "LEAF_E", "finalLabel": "E" }
    ], "ts": "2026-07-02T11:48:00.000Z" },
    "p2": { "rater": "A", "init": "jm", "label": "N", "manual": false, "note": "c3 ist eine Frage, keine prüfbare Aussage.", "answers": [
      { "step": "F1", "ans": "no", "leafKey": "LEAF_NWQ", "finalLabel": "N" }
    ], "ts": "2026-07-02T11:50:00.000Z" }
  }
}
```

### Beispiel-Export Rater B (`galadox-rater-B-kr-…json`)

```json
{
  "v": 1,
  "savedAt": "2026-07-02T12:05:00.000Z",
  "rater": { "role": "B", "init": "kr" },
  "seed": 42,
  "order": ["p1", "p2"],
  "cursor": 1,
  "decisions": {
    "p1": { "rater": "B", "init": "kr", "label": "E", "manual": false, "note": "Widerspruch bestätigt.", "answers": [
      { "step": "F1", "ans": "yes" },
      { "step": "F2", "ans": "yes" },
      { "step": "F3", "ans": "yes" },
      { "step": "F4", "ans": "yes" },
      { "step": "F5", "ans": "no", "leafKey": "LEAF_E", "finalLabel": "E" }
    ], "ts": "2026-07-02T12:03:00.000Z" },
    "p2": { "rater": "B", "init": "kr", "label": "Q", "manual": true, "note": "Eher Kommentar/Qualität, nicht klar prüfbar.", "answers": [
      { "step": "F1", "ans": "no", "leafKey": "LEAF_NWQ", "finalLabel": "Q", "via": "manual" }
    ], "ts": "2026-07-02T12:05:00.000Z" }
  }
}
```

---

## 6. Bekannte Grenzen / offene Punkte (v0.1)

- **Keine Benutzer-Authentifizierung:** Rater-Rolle (A/B) und Initialen sind reine Eingabefelder ohne Verifikation — Vertrauensbasis, kein Schutz gegen Falschangeben.
- **Cohen’s κ nur paarweise (Rater 1 ↔ 2):** keine Multi-Rater-Kappa (Fleiss’ κ) und kein Konfidenzintervall/Bootstrap.
- **Divergenz-Knoten ist Heuristik:** vergleicht normalisierte Pfad-Strings (`step:ans→finalLabel`); semantisch äquivalente Pfade über manuelle Override werden als „gleich“ betrachtet, was die Markierung verzerren kann.
- **Kein Undo/Redo:** Schritt-zurück nur innerhalb eines Paars im Wizard (`wizBack`); gespeicherte Entscheidungen vorheriger Paare lassen sich nur durch erneutes Durchlaufen überschreiben, nicht reversibel rückgängig machen.
- **`typ_kandidat` und Gold-Labels fremder Rater** werden im Label-Modus nicht gezeigt (Blindheit ✓), es gibt aber keine serverseitige Sperre — ein Rater könnte die `localStorage`/Export-JSON manuell auslesen.
- **Export-Format:** `pairs.json` ist v0.1; kein Versionierungs-Migrationspfad, keine Schema-Validierung beim Import (fehlerhafte JSON → Toast-Fehler, kein Feld-Reporting).
- **Report ist Markdown-only:** kein CSV/XLSX, keine Diagramme/Plots; κ als Punkt-Schätzer ohne Streuung.
- **Tier-3 ist terminal:** einmal als T3 gesetzt, keine automatische Wiedervorlage; Lösung nur manuell über Adjudikation.
- **Kein Concurrent-Lock:** zwei Browser-Tabs mit gleichem `localStorage`-Key überschreiben sich gegenseitig (letzter Schreibzugriff gewinnt).
- **Abnahme ausstehend:** die obigen Selbsttests sind Autor-Tests; ein externer Doppelblind-Feldtest mit echten Ratern steht noch aus.

