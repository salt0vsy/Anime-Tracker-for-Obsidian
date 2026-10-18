# Anime Tracker for Obsidian

An anime watchlist for [Obsidian](https://obsidian.md) built on top of the **Media DB** plugin.

- Add an anime through Media DB and it lands in **To watch** automatically.
- Sequels and prequels (other seasons) are detected and added automatically.
- Track `to watch` / `watching` / `watched` / `dropped` and your current episode right inside each note.
- Browse everything in a Base with cover images.

## Features

| Feature | Provided by |
| --- | --- |
| Search and add anime (MyAnimeList data) | Media DB |
| Auto-add `status` and `currentlyAtEpisode` to new notes | Templater |
| Auto-add other seasons (sequels / prequels) | Templater script + AniList API |
| Status dropdown and episode slider inside the note | Meta Bind |
| Tabs with cover images (To watch / Watching / Watched / Dropped) | Obsidian Bases |

## Requirements

- Obsidian with **Bases** support (a recent version)
- Community plugins:
  - [Media DB](https://github.com/mProjectsCode/obsidian-media-db-plugin)
  - [Templater](https://github.com/SilentVoid13/Templater)
  - [Meta Bind](https://github.com/mProjectsCode/obsidian-meta-bind-plugin)
- Network access from Obsidian to `graphql.anilist.co`

## Folder structure

```
Vault/
├── Anime.base                  # the tracker view
├── Media DB/
│   ├── series/                 # anime notes (created by Media DB and the script)
│   └── images/                 # cover images
└── Templates/
    └── anime-status.md         # Templater template (script below)
```

## Setup

### 1. Media DB

Set the **series folder** to `Media DB/series` and the **images folder** to `Media DB/images`. Keep anime in their own folder so the Templater folder template does not touch other media (movies, games, etc.).

### 2. Templater

1. Open Templater settings:
   - Enable **Trigger Templater on new file creation**.
   - Under **Folder templates**, add `Media DB/series` → `Templates/anime-status.md`.

### 3. Meta Bind

Install and enable it. No configuration is needed.

## How it works

1. You add an anime with the Media DB search command.
2. Templater fires on the new file. The script waits until Media DB has finished writing the note.
3. It adds `status: to watch` and `currentlyAtEpisode: 0` if they are missing, and appends the status dropdown and episode slider to the note body.
4. It looks the anime up on AniList by its MAL id and follows `SEQUEL` / `PREQUEL` relations.
5. For every related TV / TV_SHORT / ONA entry that does not exist in your vault yet, it creates a note with the same fields as a Media DB note and downloads the cover image.
6. A notice lists the seasons that were added.

Existing notes are detected by the `id` property (MAL id), so no duplicates are created.

### Configuration

Constants at the top of the script:

| Constant | Default | Meaning |
| --- | --- | --- |
| `SERIES_FOLDER` | `Media DB/series` | Where anime notes live |
| `IMAGE_FOLDER` | `Media DB/images` | Where cover images are saved |
| `WANTED_FORMATS` | `TV`, `TV_SHORT`, `ONA` | Which related entries become notes. Add `MOVIE` or `OVA` to include them |
| `MAX_VISITS` | `15` | Safety cap on how many related titles are followed per run |

## Usage

1. Run **Media DB: Search** (or any `open-media-db-search-modal-with-series` command) and add an anime.
2. The note appears under **To watch** in the Base. Other seasons are added a few seconds later.
3. Open a note and use the dropdown to switch the status to `watching`, `watched` or `dropped`, and the slider to set your current episode.

## Troubleshooting

**No seasons are added.** Open the developer console (`Ctrl+Shift+I`) and look for lines starting with `[anime]`.

- `anilist status 200` but no seasons: the title may simply have no sequels or prequels of a wanted format.
- `ERR_CONNECTION_TIMED_OUT`: Obsidian cannot reach the API. Check your network, firewall, antivirus or VPN. The script uses AniList instead of Jikan for this reason, since Jikan was unreachable in the original setup.
- Nothing is logged at all: the script exited early. Check that the note has `type: series` and `dataSource: MALAPI`, and that it does not already contain `currentlyAtEpisode` (already processed notes are skipped).

**Meta Bind error: `minValue` is only applicable to slider, progressBar.** An old note uses `INPUT[number(minValue(...))]`. Replace `number(` with `slider(` in that note.

**Fields are missing on a new note.** Templater and Media DB both react to file creation. Restart Obsidian or toggle *Trigger Templater on new file creation* off and on. A missing `status` still counts as `to watch` in the Base.

**`ENOENT: no such file or directory` in the console.** Media DB may create a temporary file and rename it. The script retries reading the file, so this is usually harmless.

**Testing again on the same anime.** Delete the note and its cover image first. The script ignores notes that already have `currentlyAtEpisode`.

## Limitations

- Season detection depends on AniList relations and requires an AniList entry with a MAL id. Titles without a MAL id are skipped.
- Titles come from AniList (romaji), so a season's file name may differ slightly from what Media DB would generate. Duplicates are still avoided through the MAL `id`.
- The `url` slug is generated locally. MyAnimeList redirects by id, so the link works even if the slug differs.
- Notes created before the script was updated are not changed. Delete and re-add them if you want the new format.
- Media DB has no public API for programmatic search, so adding the first anime is always done through its search modal.

## License

Personal project. Use and modify freely.
