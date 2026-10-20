# Prompt: Migrate Arc Browser Spaces to Google Chrome (Arcify)

Copy everything below the line into Claude Code (or a similar AI coding agent) on your Mac.

**Before you start:**
- Install the **Arcify** extension in Chrome: search "Arcify: Arc like vertical tab spaces" in the Chrome Web Store. Open its sidebar once so it creates its bookmark folder.
- Keep Arc installed. Its files are only read, never changed.

---

I'm moving from the Arc browser to Google Chrome, where I've installed the **Arcify** extension. I want my Arc spaces copied into Chrome exactly as they are: the same spaces, pinned tabs, folders (including nested ones), order and titles. I also want my Arc Favorites. Please do the migration on my Mac by following the steps below.

## Ground rules
- **Read only.** Only read Arc's files. Do not edit Chrome's `Bookmarks` file directly: with Chrome Sync on, direct edits can be overwritten or duplicated. Instead, create a standard Netscape bookmarks HTML file that I import myself.
- Run any Python that reads browser files with `python3 -I`. Keep the scripts in a new folder, e.g. `~/arc-migration/`.
- Use typed, small, pure functions in the scripts.

## Step 1: Find the data
- Arc's sidebar data is in `~/Library/Application Support/Arc/StorableSidebar.json`. Ignore the dated backup copies.
- Chrome's profiles are in `~/Library/Application Support/Google/Chrome/`, usually `Default` or `Profile N`. Find the profile where Arcify is installed: look in `<profile>/Extensions/*/<version>/manifest.json` for the name "Arcify".
- Read that profile's `Bookmarks` file (read only). Report:
  - whether a folder named `Arcify` exists, normally under **Other Bookmarks**
  - which space folders it already contains

## Step 2: Parse Arc (format notes)
- Spaces and items are in `sidebar.containers[1]`. Use the entry that has the `spaces`, `items` and `topAppsContainerIDs` keys.
- `spaces` and `items` are **flat alternating arrays**: `[id, object, id, object, ...]`. Convert each to a dictionary keyed by ID, and keep the order the spaces appear in.
- Each space has:
  - `title`
  - an optional emoji at `customInfo.iconType.emoji_v2`
  - `containerIDs`, also a flat array, e.g. `["pinned", <id>, "unpinned", <id>]`
- Each item has `childrenIds`, `title`, and `data`:
  - If `data.tab` exists, the item is a link. Use `data.tab.savedURL`, and `title` or `data.tab.savedTitle` as the name.
  - Otherwise the item is a folder (or a split view). Recurse into `childrenIds`, keeping their order.
- Arc's top-level **Favorites** are the children of the string IDs in `topAppsContainerIDs`.
- For each space, build the tree from the `pinned` container. Collect the `unpinned` container separately as "open tabs".
- Save the result to `arc_spaces.json`.
- Print a summary of each space: number of links, number of folders, maximum folder depth and number of open tabs. Also list the Favorites and any empty folders.

## Step 3: How Arcify stores spaces
- Each Arcify space is a bookmark folder at **Other Bookmarks › Arcify › ‹Space name›**.
- The bookmarks inside that folder, and its subfolders, are the space's pinned tabs.
- If a space folder exists but no tab group is active for it, Arcify lists it as a space you can activate.
- Arcify **Favorites** are Chrome **pinned tabs**, shared by all spaces. They cannot be created through bookmarks.
- Arcify's sidebar only blocks *creating* subfolders more than 3 levels deep. Deeper folders that already exist in bookmarks still load, so keep the full depth.
- Check these facts against the installed extension's `sidebar.js`, since the version may differ. For example, search it for `findArcifyFolder` and `getOrCreateSpaceFolder`.

## Step 4: Generate the import file
Create `arc_spaces_bookmarks.html` with this structure. Escape every title and URL as HTML.

```
Arc Spaces (import)/
  <Space 1>/ ...pinned tree...
  <Space 2>/ ...
  _Arc Favorites (pin these)/
  _Arc open tabs (unpinned)/<Space>/...   (only if there are any)
```

- Do **not** name the top folder `Arcify`. Arcify searches bookmarks for a folder with that title and could pick up the wrong one.
- Check the counts: the number of `<A HREF` lines must equal links + favorites + open tabs, and the number of `<H3>` lines must equal the expected number of folders.

## Step 5: Give me import instructions
Tell me to:
1. Open `chrome://bookmarks`, click ⋮ and choose **Import bookmarks**, then select the HTML file.
2. Drag each space folder from **Arc Spaces (import)** into **Other Bookmarks › Arcify**. Each one will appear in the Arcify sidebar; clicking it opens that space.
3. Open **_Arc Favorites (pin these)**, right-click it and choose **Open all**, then right-click each tab and choose **Add to Favorites** / **Pin**.
4. Delete the leftover import folder. Optionally delete any unused default spaces in Arcify.
5. Make Chrome the default browser at `chrome://settings/defaultBrowser`, or in System Settings › Desktop & Dock › Default web browser.

Also tell me what does **not** transfer:
- Arc space emoji and colours: I can set colours in Arcify's space options, or put the emoji in the space name.
- Split views: they become plain tabs.
- Passwords and history: export passwords from Arc as a CSV, then import them at `chrome://password-manager/settings`.

Finish with a table of each space, its link count and its folder count.
