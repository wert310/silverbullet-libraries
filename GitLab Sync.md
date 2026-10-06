---
name: "Library/wert310/GitLab Sync"
tags: meta/library
ai: anthropic/claude-opus-5-5
---

# GitLab Sync

Synchronize the current SilverBullet page with one Markdown file in a GitLab repository.

The library uses GitLab's repository API for versioned reads and conditional writes. Sync state is kept in SilverBullet's **client-local store**, so it is not written into the page or shared with other users.

## Setup

Configure the GitLab API and token in SilverBullet configuration:

```lua
config.set("gitlabSync.instances", {
  ["gitlab.com"] = {
    apiUrl = "https://gitlab.com/api/v4",
    token = "YOUR_GITLAB_TOKEN",
  },
  ["secpriv"] = {
    apiUrl = "https://gitlab.example.com/api/v4",
    token = "YOUR_SECPRIV_TOKEN",
  },
})
```

Then add this to a page you want to synchronize:

```yaml
---
gitlabSync:
  instance: secpriv
  project: group/project
  file: docs/shared.md
  branch: main
---
```

Run **GitLab: Sync**. If the file does not exist yet it is created from the page.

On first use, if the local page and GitLab file differ, the command asks whether to push the local page or pull the GitLab version.

### Sync behavior

- file missing in GitLab → created from the page
- page and file already identical → no commit
- only local changed → conditional GitLab commit
- only remote changed → pull
- both changed → three-way merge
- clean merge → commit merged content
- conflict → commit ordinary Git conflict markers
- concurrent GitLab update → retry against the new remote version

Conflict markers are deliberately ordinary Git markers so SilverBullet's existing conflict UI can handle them.

Sync state is anchored on the file's `last_commit_id`, so commits touching other files in the repository do not count as remote changes.

The library intentionally does not use a local Git repository and does not store document contents as synchronization metadata.

### Status widget

Synced pages show a status bar at the top: whether the page has local changes, whether GitLab has moved on, or whether conflict markers are still waiting to be resolved. Local status is computed from the page itself; remote status costs one request and is cached for a minute. **Check GitLab** refreshes it immediately.

## Code

```space-lua
local GITLAB_SYNC_STATE = "gitlabSync.state"
local MAX_SYNC_RETRIES = 3
local REMOTE_TTL = 60

config.define("gitlabSync", {
  description = "GitLabSync configuration",
  type = "object",
  properties = {
    instances = {
      type = "object",
      description = "Named GitLab connections",
      additionalProperties = {
        type = "object",
        properties = {
          apiUrl = { type = "string", description = "GitLab API base URL" },
          token = { type = "string", description = "GitLab access token" },
        },
        required = { "apiUrl", "token" },
        additionalProperties = false,
      },
    },
    commitMessage = {
      type = "string",
      default = "Sync from SilverBullet",
      description = "Commit message used when pushing a page",
    },
  },
  required = { "instances" },
  additionalProperties = false,
})

-- Configuration ---------------------------------------------------------------

-- Resolves this page's frontmatter together with the connection it names,
-- reporting the first problem it finds unless `quiet` is set.
local function syncConfig(quiet)
  local meta = editor.getCurrentPageMeta()
  local pcfg = meta and meta.gitlabSync

  if type(pcfg) ~= "table" or not pcfg.instance or not pcfg.project or not pcfg.file then
    if not quiet then
      editor.flashNotification(
        "This page needs gitlabSync frontmatter with instance, project and file.",
        "error"
      )
    end
    return nil
  end

  local instance = config.get("gitlabSync.instances", {})[pcfg.instance]
  if not instance or not instance.apiUrl or not instance.token then
    if not quiet then
      editor.flashNotification(
        "GitLab connection '" .. tostring(pcfg.instance) .. "' is missing or incomplete.",
        "error"
      )
    end
    return nil
  end

  pcfg.branch = pcfg.branch or "main"

  return pcfg, {
    apiUrl = instance.apiUrl:gsub("/+$", ""),
    token = instance.token,
    commitMessage = config.get("gitlabSync.commitMessage", "Sync from SilverBullet"),
  }
end

local function stateKey(pcfg)
  return GITLAB_SYNC_STATE .. ":" .. crypto.sha256(
    pcfg.instance .. "\0" .. pcfg.project .. "\0" .. pcfg.file .. "\0" .. pcfg.branch
  )
end

-- Merging ---------------------------------------------------------------------

-- Lines keep their terminating newline, so joining is plain concatenation and
-- a missing final newline survives the round trip. No line is ever empty,
-- which makes an empty concatenated region unambiguous.
local function splitLines(text)
  local lines = {}
  local pos = 1
  while pos <= #text do
    local nl = text:find("\n", pos, true)
    table.insert(lines, text:sub(pos, nl))
    if not nl then
      break
    end
    pos = nl + 1
  end
  return lines
end

-- Maps each index of `a` to the index of the line it corresponds to in `b`,
-- leaving unmatched indices absent. The mapping is strictly increasing, which
-- the merge below relies on.
local function alignLines(a, b)
  local n, m = #a, #b
  local match = {}

  -- Anchoring on the common prefix and suffix keeps the table below
  -- proportional to the size of the edit rather than to the page.
  local lo = 1
  while lo <= n and lo <= m and a[lo] == b[lo] do
    match[lo] = lo
    lo = lo + 1
  end

  local hi = 0
  while hi < n - lo + 1 and hi < m - lo + 1 and a[n - hi] == b[m - hi] do
    match[n - hi] = m - hi
    hi = hi + 1
  end

  -- dp[x][y] is the longest common subsequence of a[lo + x ..] and b[lo + y ..].
  local p, q = n - hi - lo + 1, m - hi - lo + 1
  local dp = {}
  for x = 0, p do
    dp[x] = { [q] = 0 }
  end
  for y = 0, q do
    dp[p][y] = 0
  end
  for x = p - 1, 0, -1 do
    for y = q - 1, 0, -1 do
      if a[lo + x] == b[lo + y] then
        dp[x][y] = dp[x + 1][y + 1] + 1
      else
        dp[x][y] = math.max(dp[x + 1][y], dp[x][y + 1])
      end
    end
  end

  local x, y = 0, 0
  while x < p and y < q do
    if a[lo + x] == b[lo + y] then
      match[lo + x] = lo + y
      x, y = x + 1, y + 1
    elseif dp[x + 1][y] >= dp[x][y + 1] then
      x = x + 1
    else
      y = y + 1
    end
  end

  return match
end

-- A region only lacks a final newline at the very end of a file, but a
-- conflict marker has to start on its own line regardless.
local function endLine(region)
  if region == "" or region:sub(-1) == "\n" then
    return region
  end
  return region .. "\n"
end

-- Three-way line merge (the diff3 algorithm).
--
-- Aligning both sides against the base splits the three documents into
-- alternating stable regions (where all three agree) and unstable ones. Each
-- unstable region is resolved on its own, and is only a conflict when both
-- sides really changed it and changed it differently.
--
-- Conflicts use ordinary Git markers so SilverBullet's existing conflict UI
-- can recognize and resolve the result.
local function merge3(baseText, localText, remoteText)
  if localText == remoteText then
    return localText, false
  end
  if localText == baseText then
    return remoteText, false
  end
  if remoteText == baseText then
    return localText, false
  end

  local base = splitLines(baseText)
  local localLines = splitLines(localText)
  local remoteLines = splitLines(remoteText)

  local toLocal = alignLines(base, localLines)
  local toRemote = alignLines(base, remoteLines)

  local out = {}
  local conflict = false
  local i, j, k = 1, 1, 1

  local function resolve(baseTo, localTo, remoteTo)
    local b = table.concat(base, "", i, baseTo)
    local l = table.concat(localLines, "", j, localTo)
    local r = table.concat(remoteLines, "", k, remoteTo)

    if l == r or r == b then
      table.insert(out, l)
    elseif l == b then
      table.insert(out, r)
    else
      table.insert(out, "<<<<<<< local\n" .. endLine(l) ..
        "=======\n" .. endLine(r) .. ">>>>>>> remote\n")
      conflict = true
    end
  end

  while i <= #base or j <= #localLines or k <= #remoteLines do
    if i <= #base and toLocal[i] == j and toRemote[i] == k then
      table.insert(out, base[i])
      i, j, k = i + 1, j + 1, k + 1
    else
      -- The unstable region runs up to the next base line both sides kept.
      local sync = nil
      for o = i, #base do
        if toLocal[o] and toRemote[o] then
          sync = o
          break
        end
      end

      if not sync then
        resolve(#base, #localLines, #remoteLines)
        break
      end

      resolve(sync - 1, toLocal[sync] - 1, toRemote[sync] - 1)
      i, j, k = sync, toLocal[sync], toRemote[sync]
    end
  end

  return table.concat(out), conflict
end

-- GitLab ----------------------------------------------------------------------

local function urlEncode(s)
  return (tostring(s):gsub("[^%w%-%._~]", function(c)
    return string.format("%%%02X", string.byte(c))
  end))
end

-- SilverBullet's server proxy decodes the request path once before forwarding
-- it, which would turn the %2F GitLab requires in project and file paths back
-- into real slashes. Encoding path segments twice makes them arrive encoded
-- exactly once. The query string is passed through untouched.
local function pathEncode(s)
  return (urlEncode(s):gsub("%%", "%%25"))
end

local function fileUrl(gcfg, pcfg, suffix, ref)
  return string.format(
    "%s/projects/%s/repository/files/%s%s%s",
    gcfg.apiUrl,
    pathEncode(pcfg.project),
    pathEncode(pcfg.file),
    suffix,
    ref and ("?ref=" .. urlEncode(ref)) or ""
  )
end

local function gitlabRequest(gcfg, url, options)
  options = options or {}
  options.headers = options.headers or {}
  options.headers["PRIVATE-TOKEN"] = gcfg.token
  options.headers["Accept"] = "application/json"
  return net.proxyFetch(url, options)
end

-- Returns nil when the file does not exist at `ref`.
local function getRemoteFile(gcfg, pcfg, ref)
  local meta = gitlabRequest(gcfg, fileUrl(gcfg, pcfg, "", ref))
  if meta.status == 404 then
    return nil
  end
  if not meta.ok then
    error("GitLab metadata GET failed: HTTP " .. tostring(meta.status))
  end

  local raw = gitlabRequest(gcfg, fileUrl(gcfg, pcfg, "/raw", ref),
    { responseEncoding = "text/plain" })
  if not raw.ok then
    error("GitLab content GET failed: HTTP " .. tostring(raw.status))
  end

  local content = raw.body or ""
  return {
    content = content,
    contentSha256 = crypto.sha256(content),
    -- The commit that last touched this file, which is what GitLab's
    -- optimistic locking compares against. The response's commit_id is the
    -- head of the requested ref and moves on every unrelated commit.
    commit = meta.body.last_commit_id,
  }
end

-- Status of the GitLab file for the widget, cached per page. Only the file's
-- last commit is needed, so the raw content is never fetched here.
local remoteCache = {}

local function remoteStatus(pcfg, gcfg, force)
  local key = stateKey(pcfg)
  local cached = remoteCache[key]
  if not force and cached and os.time() - cached.at < REMOTE_TTL then
    return cached
  end

  local entry
  local ok, res = pcall(gitlabRequest, gcfg, fileUrl(gcfg, pcfg, "", pcfg.branch))
  if not ok then
    entry = { error = tostring(res) }
  elseif res.status == 404 then
    local msg = type(res.body) == "table" and res.body.message or nil
    entry = msg == "404 File Not Found" and { missing = true }
      or { error = "HTTP 404 " .. tostring(msg or "") }
  elseif not res.ok then
    entry = { error = "HTTP " .. tostring(res.status) }
  else
    entry = { commit = res.body.last_commit_id }
  end

  entry.at = os.time()
  remoteCache[key] = entry
  return entry
end

local function refreshWidgets()
  pcall(function() codeWidget.refreshAll() end)
end

-- Commits `content`. Given an `expectedCommit` the write is refused if the
-- file has moved on since; without one the file is created instead. Returns
-- the resulting remote state, or nil if the write raced with another commit
-- and the caller should start over.
local function writeRemoteFile(gcfg, pcfg, content, expectedCommit)
  local res = gitlabRequest(gcfg, fileUrl(gcfg, pcfg, ""), {
    method = expectedCommit and "PUT" or "POST",
    headers = { ["Content-Type"] = "application/json" },
    body = {
      branch = pcfg.branch,
      content = content,
      commit_message = gcfg.commitMessage,
      last_commit_id = expectedCommit,
    },
  })

  -- A missing file is indistinguishable from a missing project or branch, so
  -- the misconfiguration is reported here rather than on the read.
  if res.status == 404 then
    error(string.format("GitLab could not reach '%s' on branch '%s'.",
      pcfg.project, pcfg.branch))
  end

  -- GitLab reports a lost optimistic-locking race, and a file created
  -- underneath us, as a generic 400.
  if res.status == 400 or res.status == 409 or res.status == 412 then
    return nil
  end
  if not res.ok then
    error("GitLab write failed: HTTP " .. tostring(res.status))
  end

  -- The write response carries only file_path and branch, so the resulting
  -- commit has to be read back.
  local written = getRemoteFile(gcfg, pcfg, pcfg.branch)
  if not written or written.contentSha256 ~= crypto.sha256(content) then
    return nil
  end
  return written
end

-- Syncing ---------------------------------------------------------------------

-- Returns true when the page is in sync, false when the user cancelled, and
-- nil when GitLab moved underneath us and the caller should retry.
local function syncOnce(pcfg, gcfg)
  -- Save first so the editor content for this attempt is stable.
  editor.save()

  local localText = editor.getText()
  local state = clientStore.get(stateKey(pcfg))
  local remote = getRemoteFile(gcfg, pcfg, pcfg.branch)

  local function settle(text, commit, message, kind)
    if text ~= localText then
      editor.setText(text)
      editor.save()
    end
    clientStore.set(stateKey(pcfg), { commit = commit, hash = crypto.sha256(text) })
    remoteCache[stateKey(pcfg)] = { commit = commit, at = os.time() }
    editor.flashNotification(message, kind or "info")
    refreshWidgets()
    return true
  end

  local function write(text, message, kind)
    local written = writeRemoteFile(gcfg, pcfg, text, remote and remote.commit)
    if not written then
      return nil
    end
    return settle(text, written.commit, message, kind)
  end

  -- There is no file to reconcile with yet.
  if not remote then
    return write(localText, "Created the GitLab file.")
  end

  -- Nothing to reconcile. Re-anchoring here also keeps GitLab from rejecting
  -- an empty commit.
  if crypto.sha256(localText) == remote.contentSha256 then
    return settle(localText, remote.commit,
      state and "Already synchronized." or "GitLab sync initialized.")
  end

  if not state then
    local choice = editor.filterBox("Initialize GitLab sync", {
      { name = "Push local page", hint = "Replace the GitLab file with this page" },
      { name = "Pull GitLab file", hint = "Replace this page with the GitLab file" },
    }, "The page and GitLab file differ and there is no previous sync state.")

    if not choice then
      return false
    end
    if choice.name == "Pull GitLab file" then
      return settle(remote.content, remote.commit, "Pulled GitLab file.")
    end
    return write(localText, "Pushed page to GitLab.")
  end

  if state.commit == remote.commit then
    return write(localText, "Pushed local changes to GitLab.")
  end

  if state.hash == crypto.sha256(localText) then
    return settle(remote.content, remote.commit, "Pulled remote changes from GitLab.")
  end

  -- Both sides changed. GitLab history supplies the base document; if it has
  -- been rewritten away, a baseless merge conflicts rather than guessing.
  local base = getRemoteFile(gcfg, pcfg, state.commit)
  local merged, conflict = merge3(base and base.content or "", localText, remote.content)

  -- The local edits turned out to be contained in the remote version, so
  -- there is nothing to commit.
  if crypto.sha256(merged) == remote.contentSha256 then
    return settle(merged, remote.commit, "Pulled remote changes from GitLab.")
  end

  if conflict then
    return write(merged,
      "Sync conflict committed to GitLab. Resolve the conflict markers.", "error")
  end
  return write(merged, "Merged local and remote changes.")
end

command.define {
  name = "GitLab: Sync",
  run = function()
    local pcfg, gcfg = syncConfig()
    if not pcfg then
      return
    end

    for _ = 1, MAX_SYNC_RETRIES do
      local ok, result = pcall(syncOnce, pcfg, gcfg)
      if not ok then
        editor.flashNotification("GitLab sync failed: " .. tostring(result), "error")
        return
      end
      -- Anything but nil is a finished (or cancelled) sync.
      if result ~= nil then
        return
      end
      print("GitLabSync: remote changed during sync; retrying")
    end

    editor.flashNotification("GitLab changed repeatedly during sync; try again.", "error")
  end,
}

command.define {
  name = "GitLab: Forget Sync State",
  run = function()
    local pcfg = syncConfig()
    if pcfg and editor.confirm("Forget the local GitLab sync state for this page?") then
      clientStore.delete(stateKey(pcfg))
      editor.flashNotification("GitLab sync state forgotten.", "info")
      refreshWidgets()
    end
  end,
}

-- Status widget ---------------------------------------------------------------

local function hasConflictMarkers(text)
  return ("\n" .. text):find("\n<<<<<<< local\n", 1, true) ~= nil
end

-- Returns a CSS state name and a sentence describing where the page stands.
local function describeStatus(text, state, remote)
  local localChanged = not state or crypto.sha256(text) ~= state.hash
  local remoteChanged = state and remote.commit and remote.commit ~= state.commit

  if hasConflictMarkers(text) then
    return "conflict", "Conflict markers in page"
  elseif remote.error then
    return "unknown", localChanged and "Local changes, GitLab unreachable"
      or "No local changes, GitLab unreachable"
  elseif not state then
    return "new", remote.missing and "Not in GitLab yet" or "Not synced yet"
  elseif remote.missing then
    return "missing", "File missing in GitLab"
  elseif localChanged and remoteChanged then
    return "both", "Changed here and in GitLab"
  elseif localChanged then
    return "local", "Local changes not pushed"
  elseif remoteChanged then
    return "remote", "GitLab has newer changes"
  end
  return "ok", "In sync"
end

event.listen {
  name = "hooks:renderTopWidgets",
  run = function()
    local pcfg, gcfg = syncConfig(true)
    if not pcfg then
      return widget.new {}
    end

    local state = clientStore.get(stateKey(pcfg))
    local remote = remoteStatus(pcfg, gcfg)
    local kind, label = describeStatus(editor.getText(), state, remote)

    return widget.htmlBlock(dom.div {
      class = "gitlab-sync-status gitlab-sync-" .. kind,
      title = remote.error or ("GitLab instance: " .. pcfg.instance),
      dom.span { class = "gitlab-sync-dot" },
      dom.span { class = "gitlab-sync-label", label },
      dom.button {
        class = "gitlab-sync-action gitlab-sync-check",
        title = "Check GitLab for changes",
        ["aria-label"] = "Check GitLab for changes",
        onclick = function()
          remoteStatus(pcfg, gcfg, true)
          refreshWidgets()
        end,
      },
      dom.button {
        class = "gitlab-sync-action gitlab-sync-primary gitlab-sync-run",
        title = "Sync with GitLab",
        ["aria-label"] = "Sync with GitLab",
        onclick = function() system.invokeCommand("GitLab: Sync") end,
      },
      dom.span {
        class = "gitlab-sync-ref",
        pcfg.file .. " on " .. pcfg.project .. ":" .. pcfg.branch,
      },
    })
  end,
}

-- Re-evaluate local status on save in case top widgets aren't re-rendered on
-- every edit.
event.listen {
  name = "editor:pageSaved",
  run = function() refreshWidgets() end,
}
```

## Style

```space-style
/* Drop SilverBullet's default frame around this one top widget. */
#sb-main .cm-editor .sb-lua-top-widget:has(.gitlab-sync-status) {
  border: none;
  background: none;
  padding: 0;
}

.gitlab-sync-status {
  --gls: #8a8f98;
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 0.4em 0.6em;
  margin: 0.25em 0 0.75em;
  padding: 0.35em 0.55em 0.35em 0.85em;
  border-left: 3px solid var(--gls);
  border-radius: 0 6px 6px 0;
  background: color-mix(in srgb, var(--gls) 9%, transparent);
  font-size: 0.85em;
  line-height: 1.4;
  transition: background-color 0.3s, border-color 0.3s;
}

.gitlab-sync-ok       { --gls: #2f9e6b; }
.gitlab-sync-local    { --gls: #b07d0c; }
.gitlab-sync-remote   { --gls: #2f74c8; }
.gitlab-sync-both     { --gls: #c8601c; }
.gitlab-sync-conflict { --gls: #cf3a4c; }
.gitlab-sync-missing  { --gls: #a8569a; }
.gitlab-sync-new,
.gitlab-sync-unknown  { --gls: #8a8f98; }

.gitlab-sync-dot {
  flex: none;
  width: 0.6em;
  height: 0.6em;
  border-radius: 50%;
  background: var(--gls);
}

/* The only animated state: something needs a person to look at it. */
.gitlab-sync-conflict .gitlab-sync-dot {
  animation: gitlab-sync-pulse 1.6s ease-out infinite;
}

@keyframes gitlab-sync-pulse {
  0%   { box-shadow: 0 0 0 0 color-mix(in srgb, var(--gls) 55%, transparent); }
  100% { box-shadow: 0 0 0 0.55em transparent; }
}

@media (prefers-reduced-motion: reduce) {
  .gitlab-sync-status { transition: none; }
  .gitlab-sync-conflict .gitlab-sync-dot { animation: none; }
}

.gitlab-sync-label {
  font-weight: 600;
}

.gitlab-sync-ref {
  /* Grows into the leftover space and truncates rather than wrapping; only on
     very narrow screens does it move to a line of its own. */
  flex: 1 1 0;
  min-width: 8em;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  opacity: 0.6;
  font-family: var(--editor-code-font-family, ui-monospace, "SF Mono", Menlo, Consolas, monospace);
}

/* Icon buttons. The icons are Feather icons (MIT) drawn as CSS masks, so
   they take the button's text color and need no extra DOM. */
.gitlab-sync-action {
  flex: none;
  display: inline-grid;
  place-items: center;
  width: 1.9em;
  height: 1.9em;
  padding: 0;
  border: 1px solid color-mix(in srgb, var(--gls) 45%, transparent);
  border-radius: 5px;
  background: transparent;
  color: inherit;
  cursor: pointer;
}

.gitlab-sync-action::before {
  content: "";
  width: 1.05em;
  height: 1.05em;
  background: currentColor;
  -webkit-mask: var(--gls-icon) center / contain no-repeat;
  mask: var(--gls-icon) center / contain no-repeat;
}

/* feather: search */
.gitlab-sync-check {
  --gls-icon: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24' fill='none' stroke='black' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'%3E%3Ccircle cx='11' cy='11' r='8'/%3E%3Cline x1='21' y1='21' x2='16.65' y2='16.65'/%3E%3C/svg%3E");
}

/* feather: repeat */
.gitlab-sync-run {
  --gls-icon: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24' fill='none' stroke='black' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'%3E%3Cpolyline points='17 1 21 5 17 9'/%3E%3Cpath d='M3 11V9a4 4 0 0 1 4-4h14'/%3E%3Cpolyline points='7 23 3 19 7 15'/%3E%3Cpath d='M21 13v2a4 4 0 0 1-4 4H3'/%3E%3C/svg%3E");
}

.gitlab-sync-action:hover {
  background: color-mix(in srgb, var(--gls) 16%, transparent);
}

.gitlab-sync-action:focus-visible {
  outline: 2px solid var(--gls);
  outline-offset: 1px;
}

.gitlab-sync-primary {
  border-color: var(--gls);
  background: var(--gls);
  color: #fff;
}

.gitlab-sync-primary:hover {
  background: color-mix(in srgb, var(--gls) 82%, black);
}

/* Nothing to do: keep the bar quiet. */
.gitlab-sync-ok .gitlab-sync-primary {
  background: transparent;
  border-color: color-mix(in srgb, var(--gls) 45%, transparent);
  color: inherit;
}
```
