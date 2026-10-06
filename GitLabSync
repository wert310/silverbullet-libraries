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

```space-lua
local GITLAB_SYNC_STATE = "gitlabSync.state"
local MAX_SYNC_RETRIES = 3

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
-- reporting the first problem it finds.
local function syncConfig()
  local meta = editor.getCurrentPageMeta()
  local pcfg = meta and meta.gitlabSync

  if type(pcfg) ~= "table" or not pcfg.instance or not pcfg.project or not pcfg.file then
    editor.flashNotification(
      "This page needs gitlabSync frontmatter with instance, project and file.",
      "error"
    )
    return nil
  end

  local instance = config.get("gitlabSync.instances", {})[pcfg.instance]
  if not instance or not instance.apiUrl or not instance.token then
    editor.flashNotification(
      "GitLab connection '" .. tostring(pcfg.instance) .. "' is missing or incomplete.",
      "error"
    )
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
    editor.flashNotification(message, kind or "info")
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
    end
  end,
}
```
