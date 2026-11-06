<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Project KPI Dashboard</title>
    <style>
        :root {
            --bg: #f4f6fa;
            --card: #ffffff;
            --text: #1c2333;
            --muted: #6b7489;
            --border: #e1e6ef;
            --primary: #3d5afe;
            --good: #12a150;
            --warn: #e08a00;
            --bad: #d93a3a;
            --radius: 12px;
        }

        * {
            box-sizing: border-box;
            margin: 0;
            padding: 0;
        }

        body {
            font-family: "Segoe UI", system-ui, -apple-system, Roboto, Arial, sans-serif;
            background: var(--bg);
            color: var(--text);
            padding: 24px;
        }

        .wrap {
            max-width: 1200px;
            margin: 0 auto;
        }

        header {
            margin-bottom: 20px;
        }

        header h1 {
            font-size: 24px;
            font-weight: 700;
        }

        header p {
            color: var(--muted);
            font-size: 14px;
            margin-top: 4px;
        }

        /* Filter bar */
        .filters {
            background: var(--card);
            border: 1px solid var(--border);
            border-radius: var(--radius);
            padding: 16px 20px;
            display: flex;
            flex-wrap: wrap;
            gap: 16px;
            align-items: flex-end;
            margin-bottom: 20px;
        }

        .filter {
            display: flex;
            flex-direction: column;
            gap: 6px;
            min-width: 190px;
            flex: 1;
        }

        .filter .step {
            font-size: 11px;
            font-weight: 700;
            letter-spacing: .06em;
            text-transform: uppercase;
            color: var(--muted);
            display: flex;
            align-items: center;
            gap: 6px;
        }

        .filter .step span {
            background: var(--primary);
            color: #fff;
            width: 18px;
            height: 18px;
            border-radius: 50%;
            display: inline-flex;
            align-items: center;
            justify-content: center;
            font-size: 10px;
        }

        select,
        input[type="date"] {
            font: inherit;
            font-size: 14px;
            padding: 9px 10px;
            border: 1px solid var(--border);
            border-radius: 8px;
            background: #fff;
            color: var(--text);
            width: 100%;
        }

        input[type="date"] {
            cursor: pointer;
            caret-color: transparent;
        }

        select:focus,
        input:focus {
            outline: 2px solid var(--primary);
            outline-offset: 1px;
        }

        select:disabled {
            background: #eef1f6;
            color: #9aa3b5;
            cursor: not-allowed;
        }

        .date-row {
            display: flex;
            gap: 8px;
        }

        .btn {
            font: inherit;
            font-size: 14px;
            padding: 9px 16px;
            border-radius: 8px;
            border: 1px solid var(--border);
            background: #fff;
            color: var(--text);
            cursor: pointer;
            height: 40px;
        }

        .btn:hover {
            background: #f0f3fa;
        }

        .btn.primary {
            background: var(--primary);
            border-color: var(--primary);
            color: #fff;
            font-weight: 600;
        }

        .btn.primary:hover {
            background: #2f48d8;
        }

        .btn.primary.pending {
            box-shadow: 0 0 0 3px rgba(61, 90, 254, .25);
        }

        .summary {
            font-size: 13px;
            color: var(--muted);
            margin-bottom: 14px;
        }

        .summary b {
            color: var(--text);
        }

        /* KPI cards */
        .kpis {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
            gap: 16px;
            margin-bottom: 24px;
        }

        .kpi {
            background: var(--card);
            border: 1px solid var(--border);
            border-radius: var(--radius);
            padding: 18px;
            border-top: 4px solid var(--primary);
        }

        .kpi.good {
            border-top-color: var(--good);
        }

        .kpi.warn {
            border-top-color: var(--warn);
        }

        .kpi.bad {
            border-top-color: var(--bad);
        }

        .kpi .label {
            font-size: 13px;
            color: var(--muted);
        }

        .kpi .value {
            font-size: 30px;
            font-weight: 700;
            margin: 8px 0 4px;
        }

        .kpi .sub {
            font-size: 12px;
            color: var(--muted);
        }

        .bar {
            height: 6px;
            background: #e9edf5;
            border-radius: 3px;
            margin-top: 10px;
            overflow: hidden;
        }

        .bar i {
            display: block;
            height: 100%;
            background: var(--primary);
            border-radius: 3px;
            transition: width .4s;
        }

        /* Table */
        .panel {
            background: var(--card);
            border: 1px solid var(--border);
            border-radius: var(--radius);
            overflow: hidden;
        }

        .panel h2 {
            font-size: 16px;
            padding: 16px 20px;
            border-bottom: 1px solid var(--border);
        }

        .table-scroll {
            overflow-x: auto;
        }

        table {
            width: 100%;
            border-collapse: collapse;
            font-size: 14px;
        }

        th,
        td {
            text-align: left;
            padding: 12px 20px;
            white-space: nowrap;
        }

        th {
            background: #f8f9fc;
            color: var(--muted);
            font-size: 12px;
            text-transform: uppercase;
            letter-spacing: .04em;
        }

        tr+tr td {
            border-top: 1px solid var(--border);
        }

        .pill {
            padding: 3px 10px;
            border-radius: 999px;
            font-size: 12px;
            font-weight: 600;
        }

        .pill.good {
            background: #e3f6ec;
            color: var(--good);
        }

        .pill.warn {
            background: #fdf1dc;
            color: var(--warn);
        }

        .pill.bad {
            background: #fbe4e4;
            color: var(--bad);
        }

        .empty {
            padding: 30px;
            text-align: center;
            color: var(--muted);
        }

        @media (max-width: 600px) {
            body {
                padding: 14px;
            }

            .filter {
                min-width: 100%;
            }
        }
    </style>
</head>

<body>
    <div class="wrap">
        <header>
            <h1>Project KPI Dashboard</h1>
            <p>Select filters in order: Date → Project Type → Project Name</p>
        </header>

        <section class="filters">
            <!-- Step 1 -->
            <div class="filter">
                <label class="step" for="fromDate"><span>1</span> Date</label>
                <div class="date-row">
                    <input type="date" id="fromDate" aria-label="From date">
                    <input type="date" id="toDate" aria-label="To date">
                </div>
            </div>

            <!-- Step 2 -->
            <div class="filter">
                <label class="step" for="typeSel"><span>2</span> Project Type</label>
                <select id="typeSel"></select>
            </div>

            <!-- Step 3 -->
            <div class="filter">
                <label class="step" for="nameSel"><span>3</span> Project Name</label>
                <select id="nameSel"></select>
            </div>

            <button class="btn primary" id="applyBtn" type="button">Apply</button>
            <button class="btn" id="resetBtn" type="button">Reset</button>
        </section>

        <p class="summary" id="summary"></p>

        <section class="kpis" id="kpis"></section>

        <section class="panel">
            <h2>Project details</h2>
            <div class="table-scroll">
                <table>
                    <thead>
                        <tr>
                            <th>Project</th>
                            <th>Type</th>
                            <th>Start date</th>
                            <th>Budget</th>
                            <th>Spent</th>
                            <th>Progress</th>
                            <th>Open issues</th>
                            <th>Status</th>
                        </tr>
                    </thead>
                    <tbody id="rows"></tbody>
                </table>
            </div>
        </section>
    </div>

    <script>
        /* ---------- Sample data (replace with your API / database results) ---------- */
        const PROJECTS = [
            { name: "Skyline Tower", type: "Construction", start: "2026-01-12", budget: 5200000, spent: 3100000, progress: 62, issues: 7, onTime: true },
            { name: "Riverside Mall", type: "Construction", start: "2026-03-05", budget: 3800000, spent: 2900000, progress: 71, issues: 12, onTime: false },
            { name: "Green Park Housing", type: "Construction", start: "2026-08-18", budget: 2600000, spent: 600000, progress: 22, issues: 3, onTime: true },
            { name: "ERP Rollout", type: "IT", start: "2026-02-01", budget: 900000, spent: 810000, progress: 88, issues: 9, onTime: false },
            { name: "Mobile Banking App", type: "IT", start: "2026-04-20", budget: 650000, spent: 340000, progress: 55, issues: 5, onTime: true },
            { name: "Cloud Migration", type: "IT", start: "2026-09-10", budget: 480000, spent: 90000, progress: 18, issues: 2, onTime: true },
            { name: "Highway Bypass NH-44", type: "Infrastructure", start: "2026-01-25", budget: 7400000, spent: 5200000, progress: 68, issues: 15, onTime: false },
            { name: "Water Treatment Plant", type: "Infrastructure", start: "2026-06-02", budget: 4100000, spent: 1900000, progress: 44, issues: 6, onTime: true },
            { name: "Metro Line Survey", type: "Infrastructure", start: "2026-10-05", budget: 1200000, spent: 150000, progress: 12, issues: 1, onTime: true },
            { name: "Solar Efficiency Study", type: "Research", start: "2026-02-14", budget: 320000, spent: 300000, progress: 95, issues: 2, onTime: true },
            { name: "AI Quality Inspection", type: "Research", start: "2026-05-11", budget: 540000, spent: 260000, progress: 49, issues: 4, onTime: false },
            { name: "Soil Health Mapping", type: "Research", start: "2025-11-20", budget: 210000, spent: 205000, progress: 100, issues: 0, onTime: true },
            { name: "Legacy Bridge Repair", type: "Infrastructure", start: "2025-09-15", budget: 1900000, spent: 1850000, progress: 100, issues: 1, onTime: true }
        ];

        /* ---------- Helpers ---------- */
        const $ = id => document.getElementById(id);
        const fmtMoney = n => n >= 1e6 ? "₹" + (n / 1e6).toFixed(2) + "M" : "₹" + (n / 1e3).toFixed(0) + "K";
        const iso = d => d.toISOString().slice(0, 10);
        const ALL = "__all__";

        function getDateRange() {
            return [$("fromDate").value || "0000-01-01", $("toDate").value || "9999-12-31"];
        }

        function setDefaultDates() {
            const now = new Date();
            $("fromDate").value = now.getFullYear() + "-01-01";   // start of this year
            $("toDate").value = iso(now);                          // today
        }

        function fillSelect(sel, values, allLabel, keep) {
            sel.innerHTML = "";
            sel.add(new Option(allLabel, ALL));
            values.forEach(v => sel.add(new Option(v, v)));
            sel.value = values.includes(keep) ? keep : ALL;   // keep selection if still valid
            sel.disabled = values.length === 0;
        }

        /* ---------- Cascading filter logic ---------- */
        function inDateRange() {
            const [from, to] = getDateRange();
            return PROJECTS.filter(p => p.start >= from && p.start <= to);
        }

        function refreshTypeOptions() {
            const types = [...new Set(inDateRange().map(p => p.type))].sort();
            fillSelect($("typeSel"), types, "All project types", $("typeSel").value);
        }

        function refreshNameOptions() {
            const t = $("typeSel").value;
            const names = inDateRange()
                .filter(p => t === ALL || p.type === t)
                .map(p => p.name).sort();
            fillSelect($("nameSel"), names, "All projects", $("nameSel").value);
        }

        /* Filters actually applied to the dashboard (set only when Apply is clicked) */
        let applied = { from: "", to: "", type: ALL, name: ALL };

        function getFiltered() {
            return PROJECTS.filter(p =>
                p.start >= applied.from && p.start <= applied.to &&
                (applied.type === ALL || p.type === applied.type) &&
                (applied.name === ALL || p.name === applied.name));
        }

        function markPending() { $("applyBtn").classList.add("pending"); }

        function applyFilters() {
            const [from, to] = getDateRange();
            applied = { from, to, type: $("typeSel").value, name: $("nameSel").value };
            $("applyBtn").classList.remove("pending");
            render();
        }

        /* ---------- Render ---------- */
        function render() {
            const data = getFiltered();
            const range = `${applied.from.replace("0000-01-01", "…")} to ${applied.to.replace("9999-12-31", "…")}`;
            $("summary").innerHTML = `Showing <b>${data.length}</b> project(s) · Date: <b>${range}</b> · Type: <b>${applied.type === ALL ? "All" : applied.type}</b> · Project: <b>${applied.name === ALL ? "All" : applied.name}</b>`;

            const n = data.length;
            const budget = data.reduce((s, p) => s + p.budget, 0);
            const spent = data.reduce((s, p) => s + p.spent, 0);
            const util = budget ? spent / budget * 100 : 0;
            const avgProg = n ? data.reduce((s, p) => s + p.progress, 0) / n : 0;
            const onTime = n ? data.filter(p => p.onTime).length / n * 100 : 0;
            const issues = data.reduce((s, p) => s + p.issues, 0);

            const cls = (v, good, warn) => v >= good ? "good" : v >= warn ? "warn" : "bad";

            const cards = [
                { label: "Total projects", value: n, sub: "in selection", cls: "" },
                { label: "Total budget", value: fmtMoney(budget), sub: "allocated", cls: "" },
                { label: "Amount spent", value: fmtMoney(spent), sub: `${util.toFixed(1)}% of budget`, cls: util > 90 ? "bad" : util > 75 ? "warn" : "good", bar: util },
                { label: "Avg. progress", value: avgProg.toFixed(0) + "%", sub: "completion", cls: cls(avgProg, 70, 40), bar: avgProg },
                { label: "On-time delivery", value: onTime.toFixed(0) + "%", sub: `${data.filter(p => p.onTime).length} of ${n} on schedule`, cls: cls(onTime, 80, 60), bar: onTime },
                { label: "Open issues", value: issues, sub: n ? (issues / n).toFixed(1) + " per project" : "—", cls: issues > 30 ? "bad" : issues > 15 ? "warn" : "good" }
            ];

            $("kpis").innerHTML = cards.map(c => `
    <div class="kpi ${c.cls}">
      <div class="label">${c.label}</div>
      <div class="value">${c.value}</div>
      <div class="sub">${c.sub}</div>
      ${c.bar !== undefined ? `<div class="bar"><i style="width:${Math.min(c.bar, 100)}%"></i></div>` : ""}
    </div>`).join("");

            $("rows").innerHTML = n ? data.map(p => {
                const st = p.onTime ? ["good", "On track"] : ["bad", "Delayed"];
                return `<tr>
      <td><b>${p.name}</b></td><td>${p.type}</td><td>${p.start}</td>
      <td>${fmtMoney(p.budget)}</td><td>${fmtMoney(p.spent)}</td>
      <td>${p.progress}%</td><td>${p.issues}</td>
      <td><span class="pill ${st[0]}">${st[1]}</span></td></tr>`;
            }).join("") : `<tr><td colspan="8" class="empty">No projects match the selected filters.</td></tr>`;
        }

        /* ---------- Date inputs: pick from calendar only, no typing ---------- */
        /* (The readonly attribute is avoided on purpose: it stops the calendar from opening.) */
        ["fromDate", "toDate"].forEach(id => {
            const el = $(id);
            const openPicker = () => { try { el.showPicker(); } catch (e) { } };
            el.addEventListener("click", openPicker);
            el.addEventListener("keydown", e => {
                if (e.key === "Tab") return;                       // keep keyboard navigation
                if (e.key === "Enter" || e.key === " ") openPicker(); // open calendar from keyboard
                e.preventDefault();                                // block typing, Delete, arrows
            });
            el.addEventListener("paste", e => e.preventDefault());
            el.addEventListener("drop", e => e.preventDefault());
        });

        /* ---------- Events (sequence: date → type → name) ---------- */
        function onDateChange() {
            refreshTypeOptions();   // type options depend on date
            refreshNameOptions();   // name options depend on date + type
            markPending();
        }
        function onTypeChange() { refreshNameOptions(); markPending(); }

        $("fromDate").addEventListener("change", onDateChange);
        $("toDate").addEventListener("change", onDateChange);
        $("typeSel").addEventListener("change", onTypeChange);
        $("nameSel").addEventListener("change", markPending);
        $("applyBtn").addEventListener("click", applyFilters);
        $("resetBtn").addEventListener("click", () => {
            setDefaultDates();
            $("typeSel").value = $("nameSel").value = ALL;
            onDateChange();
            applyFilters();
        });

        setDefaultDates();
        onDateChange();
        applyFilters();   // initial load
    </script>
</body>

</html>
