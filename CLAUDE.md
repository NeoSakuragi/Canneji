# CLAUDE.md

## Project Overview

**Canneji** is a joyo kanji study system — a searchable web browser + Anki deck covering all 2,136 常用漢字 with vocabulary, radical decomposition, and mnemonics.

- **Web app**: https://canneji.duckdns.org
- **Repo**: https://github.com/NeoSakuragi/Canneji

## Architecture

Single-page static web app (`index.html`) + JSON data file (`data/kanji_export.json`) + Anki deck (`Canneji.apkg`). No backend.

### Data Pipeline

All source data lives in `/home/bruno/CLProjects/AnkiRefactor/data/`. The web app and Anki deck are generated from `kanji_export.json`.

| Data | Source | Path |
|------|--------|------|
| Kanji (grade, readings, meanings) | KANJIDIC2 | `kanjidic2-en-3.6.2.json` |
| Radical decomposition | KanjiVG (recursive, stop at known kanji) | `kanjivg_decomp.json` |
| Radical names | Kanji Alive | `radical_names.json` |
| Vocabulary | Jitendex (score >= 100) | `jitendex.db` (SQLite) |
| Vocab frequency | JITEN_GLOBAL from RIRIKKU corpus (anime, drama, novels, games) | `japanese-word-ranks/data/V2/` |
| JLPT vocab tags | PikaPikaGems | `japanese-word-ranks/data/jlpt/word_jlpt.json` |
| Mnemonics | Heisig RTK + Koohii community (separate fields) | `koohii_heisig.csv` |
| Heisig components | sdcr/heisig-kanjis | `heisig_kanjis.csv` |
| Unicode radical mapping | EquivalentUnifiedIdeograph.txt | `equivalent-unified.txt` |

### Vocab Selection Rules

- Jitendex entry score >= 100 (primary kanji writings only)
- Include if: has JITEN_GLOBAL frequency, OR is JMdict common (score >= 200), OR has JLPT level
- Max 15 per kanji, sorted by frequency
- JITEN_GLOBAL is the frequency source (drama, anime, manga, novels, video games, visual novels, web novels)

### Radical Decomposition Rules

- Use KanjiVG's hierarchical tree
- Stop at any component that exists in KANJIDIC (known kanji) or radical names table
- Recurse deeper only for obscure components not in KANJIDIC
- Each component gets: character + reading (from radical names or kanjidic) + English meaning
- Format in Anki field: `今[コン] now<br>貝[かい] shell` — Anki's furigana filter renders the ruby

### Anki Card Template

- **Front**: kanji in 6 fonts (KosugiMaru, YujiSyuku, NotoSansJP, NotoSerifJP, SawarabiGothic, MPlus1p)
- **Back**: kanji + meaning, radicals in pills, grade badge, on/kun readings, vocab with toggle (click to reveal furigana + definition), Heisig + Koohii mnemonics, confusion notes
- **Font**: Shippori Mincho (closest to JLPT exam style)
- **Pattern**: all dynamic field content rendered via hidden `<div>` → JS reads `innerHTML` → reformats → injects into display div. This avoids Anki's auto-furigana and JS string quote issues.
- **Note type name in Anki**: `canneji`

### Anki Fields

`kanji`, `meaning`, `ja_on`, `ja_kun`, `grade`, `radicals`, `examples`, `heisig_story`, `koohii_story`, `heisig_keyword`, `confusion`, `mnemonic`

### Progress Migration

Scheduling data (interval, reps, lapses, due date, factor) copied from existing N5/N4/N3/N2 Kanji decks. The `.apkg` uses `crt = 1693245600` (original collection creation timestamp) so due dates are compatible.

## Deployment

### Web App

- **Server**: Hetzner Cloud at `195.201.91.211`
- **Domain**: `canneji.duckdns.org` (DuckDNS free DNS)
- **SSL**: Let's Encrypt via Certbot (auto-renewing)
- **Nginx config**: `/etc/nginx/sites-available/kanji`
- **Web root**: `/var/www/kanji/`

### Deploy Commands

```bash
# Deploy web app
rsync -az /home/bruno/Canneji/index.html root@195.201.91.211:/var/www/kanji/
rsync -az /home/bruno/Canneji/data/kanji_export.json root@195.201.91.211:/var/www/kanji/data/
rsync -az /home/bruno/Canneji/Canneji.apkg root@195.201.91.211:/var/www/kanji/
```

### Updating the Anki Deck

1. Make changes to data in `/home/bruno/CLProjects/AnkiRefactor/data/`
2. Rebuild `kanji_export.json`
3. Push to Anki via AnkiConnect (model name: `canneji`)
4. Export: `anki("exportPackage", deck="Canneji", path="...", includeSched=True)`
5. Copy files to `/home/bruno/Canneji/`, commit, push, deploy

### Updating Card Template via AnkiConnect

The imported `.apkg` created its own model. When updating templates, target the model name `canneji` (lowercase, user renamed it). Check `modelNamesAndIds` if unsure — cards may be on a different model ID than expected.

## Kanji Ordering

- **Grades 1-6**: official 教育漢字 list (1,026 kanji), sorted by KANJIDIC2 newspaper frequency within each grade
- **Junior High (grade 8)**: remaining 1,110 joyo kanji, sorted by KANJIDIC2 frequency (no official per-year breakdown exists)

## Known Limitations

- 41 kanji have zero vocab (mostly prefecture name kanji and rare formal kanji)
- Mnemonic formatting: `**bold**` and `_italic_` converted to `<b>`/`<i>` in data
- JLPT level removed from kanji (no official list exists) — kept only on vocab words
- Some KanjiVG components have non-renderable Unicode chars (CJK Extension B) — filtered out
