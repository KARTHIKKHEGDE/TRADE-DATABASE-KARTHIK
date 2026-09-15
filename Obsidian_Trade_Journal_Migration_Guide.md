# Obsidian Trade Journal — Laptop Migration & Setup Guide

> Keep this guide with the vault so the setup can be recreated on a new laptop.

## 1. Vault Structure

```text
Trade Journal/
├── Trade Journals/
│   ├── 2026/
│   │   ├── Trades/
│   │   └── Trade Journal 2026.base
│   ├── 2027/
│   │   ├── Trades/
│   │   └── Trade Journal 2027.base
│   └── ...
├── Daily Playbook/
│   ├── 2026/
│   │   ├── January/
│   │   ├── February/
│   │   └── September/
│   │       ├── 09/15/2026.md
│   │       ├── 09/16/2026.md
│   │       └── ...
│   └── 2027/
├── Deep Dives/
├── Resources/
├── Images/
│   ├── Trades/
│   │   ├── 2026/
│   │   ├── 2027/
│   │   └── ...
│   └── Deep Dives/
│       ├── Parabolic Short/
│       ├── VWAP/
│       └── ...
└── Templates/
    ├── Daily Playbook.md
    ├── Trade Review.md
    └── Calculate Trade.md
```

## 2. Plugins

- **Templater** — templates and trade percentage calculation automation.
- **Attachment Management by trganda** — controls attachment paths and names.
- **Find Orphaned Images** — finds image files that are no longer linked and allows cleanup.
- **Bases** — used for Trade Journal `.base` files.
- If using Git synchronization, install/configure the **Obsidian Git** community plugin if needed.

## 3. Core Obsidian Attachment Setting

Go to:

**Settings → Files and links**

Set:

- **Default location for new attachments:** `In the folder specified below`
- **Attachment folder path:** `Images`

This is the global/default attachment location. Folder-specific Attachment Management overrides take precedence.

## 4. Attachment Management — Global Settings

Plugin: **Attachment Management by trganda**

Global settings:

- **Root path to save attachment:** `Copy Obsidian settings`
- **Attachment format:** `IMG-${date}`
- **Date format:** `YYYYMMDDHHmmssSSS`
- **Automatically rename attachment:** `ON`

Important:

The **Attachment Path** field still needs a valid path even when Root path is set to `Copy Obsidian settings`.

## 5. Trade Image Override

### Goal

```text
Trade Journals/2026/Trades/<trade>.md
        ↓
Images/Trades/2026/
```

For each year:

1. Right-click the year-specific `Trades` folder.
2. Choose **Override attachment setting**.
3. Set **Root path:** `Copy Obsidian settings`
4. Set **Attachment path:** `Trades/2026` for 2026.
5. Keep **Attachment format:** `IMG-${date}`.
6. For 2027, use `Trades/2027`, etc.

### Important

Do **not** create ticker/symbol subfolders.

Desired:

```text
Images/
└── Trades/
    └── 2026/
        ├── IMG-...
        ├── IMG-...
        └── IMG-...
```

Not:

```text
Images/Trades/2026/AAPL/
Images/Trades/2026/SMSPHARMA/
```

## 6. Deep Dive Image Override

### Goal

Each Deep Dive topic gets exactly one image folder:

```text
Deep Dives/<Topic>/<note>.md
        ↓
Images/Deep Dives/<Topic>/
```

Example:

```text
Deep Dives/
└── Parabolic Short/
    └── Ideas.md

Images/
└── Deep Dives/
    └── Parabolic Short/
        ├── IMG-...
        ├── IMG-...
        └── IMG-...
```

To configure a topic:

1. Right-click the **topic folder**, e.g. `Parabolic Short`.
2. Choose **Override attachment setting**.
3. Set **Root path:** `Copy Obsidian settings`.
4. Set the Attachment Path so the images go to `Images/Deep Dives/Parabolic Short/`.
5. Keep **Attachment format:** `IMG-${date}`.

### Important

Do **not** use:

```text
${notepath}/${notename}
```

for this goal because it can create an extra note-name folder such as:

```text
Images/Deep Dives/Parabolic Short/Ideas/
```

The desired structure is only:

```text
Images/Deep Dives/Parabolic Short/
```

## 7. Orphaned Image Cleanup

Use **Find Orphaned Images** specifically for image cleanup.

Purpose:

- Find image files that are no longer linked by any note.
- Review them.
- Delete/Trash them when confirmed unnecessary.

Recommended safety settings:

- **Move to Trash:** ON
- **Safety Scan Before Deleting:** ON, if available.

Important:

Removing an image from a note does **not necessarily delete the actual image file** from the vault.

Therefore:

```text
Remove image from note
        ↓
Image file may remain
        ↓
Find Orphaned Images
        ↓
Review
        ↓
Move unused image to Trash
```

Do not blindly delete files that may still be used elsewhere.

## 8. Trade Journal Columns

Each row represents one trade idea, including pyramiding within the same trade idea.

Columns:

1. Ticker
2. Date
3. Direction
4. Setup
5. Qty
6. Entry
7. Exit
8. P&L
9. % Gain/Loss
10. % Total Move

There is **NO Stop column**.

### Ticker

Use the built-in:

```text
file.name
```

Do **not** create a custom Ticker property.

The ticker should be clickable and open the trade note.

## 9. Trade Properties & Formulas

Trade notes use:

```text
Date
Direction
SetUp
Qty
Entry
Exit
P&L
% Gain/Loss
% Total Move
```

### % Gain/Loss

```text
(note["P&L"] * 100 / (note["Entry"] * note["Qty"])).round(2)
```

### % Total Move

```text
(((note["Exit"] - note["Entry"]) / note["Entry"]) * 100).abs().round(2)
```

### Critical rule

**P&L must NOT be modified by the calculation automation.**

The automation should update only:

- `% Gain/Loss`
- `% Total Move`

## 10. Working Trade Calculation Script

This script updates only `% Gain/Loss` and `% Total Move`.

```javascript
<%*
const vault = app.vault;
const processing = new Set();

async function updateTrade(file) {
    if (!file || file.extension !== "md") return;
    if (!file.path.includes("/Trades/")) return;
    if (processing.has(file.path)) return;

    processing.add(file.path);

    try {
        await app.fileManager.processFrontMatter(file, (fm) => {
            const pnl = Number(fm["P&L"]);
            const entry = Number(fm["Entry"]);
            const qty = Number(fm["Qty"]);
            const exit = Number(fm["Exit"]);

            if (!Number.isFinite(pnl) ||
                !Number.isFinite(entry) ||
                !Number.isFinite(qty) ||
                !Number.isFinite(exit) ||
                entry === 0 ||
                qty === 0) {
                return;
            }

            // % Gain/Loss
            fm["% Gain/Loss"] =
                Math.round((pnl * 100 / (entry * qty)) * 100) / 100;

            // % Total Move
            fm["% Total Move"] =
                Math.round((Math.abs((exit - entry) / entry) * 100) * 100) / 100;
        });
    } finally {
        processing.delete(file.path);
    }
}

app.workspace.onLayoutReady(() => {
    vault.on("modify", updateTrade);
});
%>
```

**Do not casually modify this working script.**

Do not add P&L calculations or changes to other properties.

## 11. Trade Review Template

```markdown
# Trade Review

#### 1 Day Timeframe

#### Why I Entered

#### 5 Minute Timeframe

#### 1 Minute Timeframe

#### Why I Exited

#### Lessons

#### After Some Days
```

Purpose:

- **1 Day Timeframe** — screenshot before trade
- **Why I Entered** — reasoning
- **5 Minute Timeframe** — screenshot
- **1 Minute Timeframe** — screenshot
- **Why I Exited** — reasoning
- **Lessons** — what was learned
- **After Some Days** — later add a 1D chart showing what happened after exit

## 12. Daily Playbook Template

```markdown
# Daily Playbook

## Today's Process Plan

Write what I plan to do today and how I want to trade.

## Execution

Write what I actually did today. Did I follow the process I planned? Where did I deviate?

## Reflection

Write what I learned from today's execution and what I want to improve tomorrow.
```

Daily Playbook is a **diary-style process journal**, not a checklist/table-heavy form.

## 13. Moving the Vault to a New Laptop

Recommended approach: keep the complete vault in a Git repository or another reliable sync/backup system.

### On the old laptop

1. Save all changes.
2. Commit the latest vault changes.
3. Push to the Git repository.
4. Make sure important folders/files are included.
5. Do not delete the old copy yet.

### On the new laptop

1. Install **Obsidian**.
2. Install **Git** if using Git.
3. Clone the repository.
4. Open Obsidian.
5. Choose **Open folder as vault**.
6. Select the cloned `Trade Journal` folder.
7. Install and enable the required community plugins.
8. Check the global attachment settings.
9. Check Attachment Management settings.
10. Recreate/check year-specific Trade folder overrides.
11. Recreate/check Deep Dive topic overrides.
12. Test one trade screenshot.
13. Test one Deep Dive screenshot.
14. Test the percentage calculation automation with a test trade.
15. Only after everything works, consider the new laptop fully migrated.

## 14. Git / Sync Notes

- If `.obsidian` is excluded from Git, plugin settings and some configuration must be recreated manually.
- If `.obsidian` is included, review what is being synced because some settings can be machine-specific.
- Never commit API keys, passwords, access tokens, or other secrets.
- Keep the old laptop/vault intact until the new laptop is verified.

## 15. New Laptop Final Checklist

- [ ] Vault opens correctly.
- [ ] Trade Journals exist.
- [ ] Daily Playbook exists.
- [ ] Deep Dives exist.
- [ ] Images exists.
- [ ] Resources exists.
- [ ] Templates exist.
- [ ] Trade Journal `.base` files open correctly.
- [ ] Templater is installed/enabled.
- [ ] Attachment Management is installed/enabled.
- [ ] Find Orphaned Images is installed/enabled.
- [ ] Global attachment folder = `Images`.
- [ ] Trade override = `Images/Trades/<YEAR>`.
- [ ] Deep Dive topic override = `Images/Deep Dives/<Topic>/`.
- [ ] Trade percentage calculations work.
- [ ] P&L remains untouched.
- [ ] Test trade screenshot lands in the correct folder.
- [ ] Test Deep Dive screenshot lands in the correct folder.

## 16. Important Rules

### Rule 1 — Do not break working automation

The calculation automation is only for:

```text
% Gain/Loss
% Total Move
```

It must not modify:

```text
P&L
Entry
Exit
Qty
Date
Direction
SetUp
```

### Rule 2 — Attachment organization

Trade screenshots:

```text
Images/Trades/<YEAR>/
```

Deep Dive screenshots:

```text
Images/Deep Dives/<Topic>/
```

No ticker folders.

No extra note-name folders.

### Rule 3 — New years

When creating a new year, create:

```text
Trade Journals/<YEAR>/
├── Trades/
└── Trade Journal <YEAR>.base
```

and:

```text
Images/Trades/<YEAR>/
```

Then create/check the corresponding Attachment Management override for that year's `Trades` folder.

### Rule 4 — Deep Dive topics

When creating a new Deep Dive topic, create:

```text
Deep Dives/<Topic>/
```

and:

```text
Images/Deep Dives/<Topic>/
```

Then create/check the Attachment Management override for that topic folder.
