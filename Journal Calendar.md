---
name: "Library/wert310/Journal Calendar"
tags: meta/library
ai: aqueduct/deepseek-v4-flash-284b
---

# Journal Calendar

A minimal, pretty, interactive calendar widget for SilverBullet.

Place `${journalCalendar()}` on any page (e.g. your Dashboard or Index page) to render it:

${journalCalendar()}

## What it does

- **Journal link in the day** — a day whose journal page exists (`Journal/2026-08-24`) is shown as a filled accent circle.
- **Click a day** — navigates to that day's journal page; if it doesn't exist yet it is created (just start typing).
- **Due-dot** — a small dot under a day means there are *open* tasks with a `deadline` on that date:
  ```
  * [ ] Pay rent [deadline: "2026-08-24"]
  ```
  (or `deadline: 2026-08-24` in a page's frontmatter, which applies to every task on that page)
- **Today** is outlined. The **•** button jumps back to the current month (and refreshes the data).
- **Live** — the calendar re-renders automatically whenever anything in the space changes (new journal page, a task completed, a new `deadline`, …), and remembers which month you were viewing across refreshes.
- Prev / next month navigation, adapts to mobile and desktop, works in light & dark theme.

## Configuration

Edit the `calConfig` table below (`prefix` sets where your journal pages live — default `Journal/`).

## The widget

```space-lua
-- priority: 10
--- Interactive journal calendar. Add ${journalCalendar()} to a page to render it.

calConfig = {
  prefix = "Journal/",   -- path prefix of your daily journal pages
}

-- {array of existing journal page keys} (defensive)
local function calExistingJournals()
  local out = {}
  local okPages, pages = pcall(space.listPages)
  if not okPages or type(pages) ~= "table" then return out end
  local pfx = calConfig.prefix
  for _, p in ipairs(pages) do
    local okName, name = pcall(function() return p.name end)
    if okName and type(name) == "string" then
      local y, m, d = name:match("^" .. pfx .. "(%d%d%d%d)-(%d%d)-(%d%d)$")
      if y then out[#out + 1] = pfx .. y .. "-" .. m .. "-" .. d end
    end
  end
  return out
end

-- Normalize any deadline value to a "YYYY-MM-DD" key (or nil)
local function calDateKey(v)
  if type(v) == "number" then
    local s = v > 1e12 and v / 1000 or v -- ms or s
    return os.date("%Y-%m-%d", s)
  end
  return tostring(v):match("(%d%d%d%d%-%d%d%-%d%d)")
end

-- {array of due date keys that have open tasks with a deadline}
local function calDueDays()
  local out = {}
  pcall(function()
    local tasks = query[[from t = index.tasks() where not t.done and t.deadline select t.deadline]]
    for i = 1, #tasks do
      local k = calDateKey(tasks[i])
      if k then out[#out + 1] = k end
    end
  end)
  return out
end

-- Serialize an array of keys into a JS object literal
local function calToJson(arr)
  if type(arr) ~= "table" then return "{}" end
  local parts = {}
  for i = 1, #arr do
    parts[#parts + 1] = '"' .. tostring(arr[i]) .. '":true'
  end
  return "{" .. table.concat(parts, ",") .. "}"
end

local function calStep (label, fn)
  local ok, res = pcall(fn)
  if not ok then error("[" .. label .. "] " .. tostring(res)) end
  return res
end

-- Build the interactive (sandboxed) calendar widget
function journalCalendar()
  local okFn, result = pcall(function()
    return calJournalCalendar()
  end)
  if not okFn then
    -- Surface the real error inline so it's easy to debug
    return widget.markdownBlock("⚠️ Journal calendar error: " .. tostring(result))
  end
  return result
end

local function calJournalCalendar()
  local journals = calStep("journals", calExistingJournals)
  local due      = calStep("due",      calDueDays)

  return calStep("render", function()
  -- detect the editor theme so the widget matches it (light / dark)
  local theme = "light"
  pcall(function()
    if editor.getUiOption("darkMode") then theme = "dark" end
  end)

  -- month to show; leave empty to default to the current month (picked in JS)
  local view = ""
  local okV, savedView = pcall(clientStore.get, "journalCalendar.view")
  if okV and type(savedView) == "string" and savedView:match("^%d+%-%d+$") then
    view = savedView
  end

  -- serialize the data BEFORE building the markup (plain concatenation, no gsub)
  local j = calStep("jsonj", function() return calToJson(journals) end)
  local d = calStep("jsond", function() return calToJson(due) end)

  local html = [=[
  <style>
    .cal { --accent:#4f7cff; --ink:#1c1f26; --muted:#8b919c; --fill:#ffffff;
           --line:#e7e9ee; --hover:#f2f4f8; --dot:#ff9f43;
           font-family:system-ui,-apple-system,"Segoe UI",Roboto,sans-serif;
           width:100%; max-width:100%; box-sizing:border-box; color:var(--ink); }
    .cal-dark .cal { --ink:#e8eaf0; --muted:#9aa0ac; --line:#2c303a;
                     --fill:#1c1f27; --hover:#262a34; --dot:#ffb454; }
    .cal-inner { width:100%; margin:0 auto; box-sizing:border-box; }
    .cal-nav { display:flex; align-items:center; justify-content:flex-start; gap:8px; margin-bottom:6px; }
    .cal-title { font-weight:700; font-size:1rem; letter-spacing:.2px;
                 text-transform:capitalize; }
    .cal-year { color:var(--muted); font-weight:600; font-size:.85em; margin-left:4px; }
    .cal-btn { border:none; background:var(--hover); color:var(--ink); width:30px;
               height:30px; border-radius:50%; font-size:1.1rem; line-height:1;
               display:flex; align-items:center; justify-content:center;
               cursor:pointer; transition:background .15s, transform .1s; }
    .cal-btn:hover { background:var(--line); }
    .cal-btn:active { transform:scale(.92); }
    .cal-grid { display:grid; grid-template-columns:repeat(7,1fr); gap:2px; }
    .cal-dow { color:var(--muted); font-size:.62rem; font-weight:700;
               text-transform:uppercase; letter-spacing:.6px;
               text-align:center; padding:2px 0 4px; }
    .cal-day, .cal-empty { height:30px; display:flex; flex-direction:column;
               align-items:center; justify-content:center; position:relative; }
    .cal-day { cursor:pointer; border-radius:10px; transition:background .12s; }
    .cal-day:hover { background:var(--hover); }
    .cal-num { display:flex; align-items:center; justify-content:center;
               width:clamp(22px,6vw,28px); height:clamp(22px,6vw,28px);
               border-radius:50%; font-size:clamp(.7rem,2.8vw,.8rem); font-weight:600; }
    .cal-day.has-journal .cal-num { background:var(--accent); color:#fff;
               box-shadow:0 2px 8px rgba(79,124,255,.35); }
    .cal-day.today .cal-num { border: 1px solid var(--accent); }
    .cal-dot { position:absolute; bottom:14%; width:5px; height:5px; border-radius:50%;
               background:var(--dot); opacity:0; transition:opacity .12s; }
    .cal-day.due .cal-dot { opacity:1; }
    @media (min-width:560px){
      .cal-dow { font-size:.66rem; }
    }
  </style>
  <div class="cal" id="cal" data-view=']=] .. view .. [=[' data-journals=']=] .. j .. [=[' data-due=']=] .. d .. [=[' data-theme=']=] .. theme .. [=['></div>
  ]=]

  local script = [=[
    (function(){
      const wrap = document.getElementById("cal");
      const prefix = "]=] .. calConfig.prefix .. [=[";
      const journals = JSON.parse(wrap.dataset.journals || "{}");
      const due      = JSON.parse(wrap.dataset.due      || "{}");
      document.documentElement.classList.add("cal-dark");
      if (wrap.dataset.theme !== "dark") document.documentElement.classList.remove("cal-dark");

      const PAD = n => String(n).padStart(2, "0");
      const MONTHS = ["January","February","March","April","May","June",
                      "July","August","September","October","November","December"];
      const DOW = ["Mon","Tue","Wed","Thu","Fri","Sat","Sun"];

      // restore the month you last looked at (or fall back to the current month)
      const rawView = (wrap.dataset.view || "").split("-").map(Number);
      let viewY = rawView.length === 2 && rawView[0] >= 0 ? rawView[0] : new Date().getFullYear();
      let viewM = rawView.length === 2 && rawView[1] >= 0 ? rawView[1] : new Date().getMonth();
      function saveView(){
        try { syscall("clientStore.set", "journalCalendar.view", viewY + "-" + viewM); } catch(e){}
      }

      function navigate(key){ syscall("editor.navigate", prefix + key); }

      function render(){
        const now = new Date();
        const first = new Date(viewY, viewM, 1);
        const start = (first.getDay() + 6) % 7;            // Monday-first
        const days  = new Date(viewY, viewM + 1, 0).getDate();
        const todayKey = now.getFullYear()+"-"+PAD(now.getMonth()+1)+"-"+PAD(now.getDate());

        wrap.innerHTML =
          '<div class="cal-inner">' +
          '<div class="cal-nav">' +
            '<button class="cal-btn" id="cal-prev" aria-label="Previous">&#8249;</button>' +
            '<div class="cal-title">'+MONTHS[viewM]+'<span class="cal-year">'+viewY+'</span></div>' +
            '<button class="cal-btn" id="cal-now" title="Today / refresh">&#8226;</button>' +
            '<button class="cal-btn" id="cal-next" aria-label="Next">&#8250;</button>' +
          '</div>' +
          '<div class="cal-grid cal-dow">'+DOW.map(d=>'<div>'+d+'</div>').join("")+'</div>' +
          '<div class="cal-grid" id="cal-cells"></div>' +
          '</div>';

        const cells = document.getElementById("cal-cells");
        // always render a full 6 weeks so the widget height stays stable
        for (let i = 0; i < 42; i++){
          const d = i - start + 1;
          if (d < 1 || d > days){ cells.appendChild(el("div","cal-empty")); continue; }
          const key  = viewY+"-"+PAD(viewM+1)+"-"+PAD(d);
          const page = prefix + key;
          const cls = "cal-day"
            + (journals[page] ? " has-journal" : "")
            + (due[key]       ? " due" : "")
            + (key === todayKey ? " today" : "");
          const cell = el("div", cls);
          cell.innerHTML = '<div class="cal-num">'+d+'</div><div class="cal-dot"></div>';
          cell.addEventListener("click", ()=>navigate(key));
          cells.appendChild(cell);
        }

        document.getElementById("cal-prev").onclick = ()=>{
          if (viewM === 0){ viewM = 11; viewY--; } else viewM--;
          saveView(); render(); };
        document.getElementById("cal-next").onclick = ()=>{
          if (viewM === 11){ viewM = 0; viewY++; } else viewM++;
          saveView(); render(); };
        document.getElementById("cal-now").onclick = ()=>{
          const t = new Date(); viewY = t.getFullYear(); viewM = t.getMonth();
          saveView(); render();
          try { syscall("system.invokeFunction", "index.refreshWidgets"); } catch(e){} // re-pull data
        };
      }

      function el(tag, cls){
        const e = document.createElement(tag);
        if (cls) e.className = cls;
        return e;
      }

      render();
    })();
  ]=]

  return calStep("widget", function()
    return widget.new {
      html = html,
      script = script,
      markdown = "Journal calendar (interactive widget)",
      sandbox = true,
      display = "block",
    }
  end)
  end) -- /render
end

-- Live updates: whenever any page in the space changes (new journal entry,
-- a task completed, a deadline added...) re-render the widgets on the
-- current page, so the calendar always reflects the latest state.
-- (The global guard keeps this from registering a second listener on reload.)
if not calLiveHooked then
  calLiveHooked = true
  event.listen {
    name = "file:changed",
    run = function()
      pcall(system.invokeFunction, "index.refreshWidgets")
    end,
  }
end
```

## Notes

- After editing this page run **System: Reload** (Ctrl-Alt-r) so the new space-lua definition is picked up.
- **Live scope:** SilverBullet only refreshes widgets rendered on the *current* page. So while you're looking at the page with the calendar, it updates on every change (including changes made in other tabs). Navigating to any page re-renders it anyway, so it's always current when you arrive.
- The **•** button re-runs the widget's data gathering (`index.refreshWidgets`), so new journal pages / deadlines appear immediately without a full reload.
- The deadline marker uses SilverBullet's built-in `deadline` task attribute — see the docs: `[deadline: "2026-08-24"]` on a task, or `deadline:` in a page's frontmatter.
