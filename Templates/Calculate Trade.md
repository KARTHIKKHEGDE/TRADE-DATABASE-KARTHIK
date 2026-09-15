<%*
const vault = app.vault;
const processing = new Set();
const reviewProcessing = new Set();
const reviewTemplatePath = "Templates/Trade Review.md";

function sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
}

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

async function insertTradeReview(file) {
    if (!file || file.extension !== "md") return;
    if (!file.path.includes("/Trades/")) return;
    if (reviewProcessing.has(file.path)) return;

    reviewProcessing.add(file.path);

    try {
        // Give Bases time to finish creating its properties.
        await sleep(500);

        const templateFile =
            vault.getAbstractFileByPath(reviewTemplatePath);

        if (!templateFile || templateFile.extension !== "md") return;

        const template =
            (await vault.read(templateFile)).trim();

        if (!template) return;

        // Try a few times because Bases may still be writing frontmatter.
        for (let attempt = 0; attempt < 10; attempt++) {
            const current = await vault.read(file);

            // Find YAML frontmatter.
            const match = current.match(
                /^---\r?\n[\s\S]*?\r?\n---\r?\n?/
            );

            const frontmatter = match ? match[0] : "";
            const body = match
                ? current.slice(match[0].length)
                : current;

            // If Trade Review is already there, stop.
            if (body.includes("# Trade Review")) return;

            // We only insert into an empty body.
            if (body.trim() === "") {
                const newContent =
                    frontmatter +
                    (frontmatter ? "\n" : "") +
                    template +
                    "\n";

                await vault.modify(file, newContent);
                return;
            }

            // Bases may still be finishing its creation.
            await sleep(300);
        }
    } finally {
        reviewProcessing.delete(file.path);
    }
}

app.workspace.onLayoutReady(() => {
    // Existing percentage calculator.
    vault.on("modify", async (file) => {
        await updateTrade(file);
        await insertTradeReview(file);
    });

    // Handle notes created by Bases + New.
    vault.on("create", insertTradeReview);
});
%>