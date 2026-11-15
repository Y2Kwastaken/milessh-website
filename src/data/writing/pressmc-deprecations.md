---
title: 'Should Paper Remove Old Deprecated APIs?'
lead: 'A glance at static use of old deprecated APIs in Paper plugins.'
date: '2026-11-14'
tags: ['PaperMC', 'Static Analysis', 'Java']
source: 'https://github.com/Y2Kwastaken/PressMC/blob/research/ancient-api-usage/report/report.md'
---

## AI Disclosure

This report was helped along by AI. Below is a list of things AI worked on in this report for full transparency. A more sophisticated breakdown
of AI use will be reported at the end to keep integrity.

- [oldest-deprecations.md](/writing/pressmc-oldest-deprecations) was generated with the help of AI
- [deprecated-data.csv](/writing/pressmc/data/deprecated-data.csv) generated from AI analysis and search
- static-analyzer java project to read and analyze jars
- Bulk downloader (static-plugin-fetch) to sample jar files from Hangar and Modrinth
- the sample design (the selection split, jar choosing and sampling limitations) was drafted with AI
- the statistics tables in "Actual Data Analysis", which AI generated from the run
- the analysis data in [data/](https://github.com/Y2Kwastaken/PressMC/tree/research/ancient-api-usage/report/data) (`usage-*.csv`, `*-attributed.csv`, `xseries-plugins.csv`) and the scripts that produce it
  (`static-analyzer/summarize.py`, `static-analyzer/xseries_scan.py`)
- `static-analyzer/charts.py` and the eight figures were generated with AI assistance
- AI proofread this report and corrected numbers and wording.

## Opening

Paper's current ecosystem is bogged down and continues to maintain a long list of very old deprecated methods. The goal of this
report is an analysis of if paper truly needs to continue to maintain such old legacy systems, which can bog down developers and cause
unnecessary burden when expanding the paper API. This report has compiled a list of the oldest deprecation annotations added: the 50 oldest,
plus the remaining 86 members of the same 2013-08-19 "Deprecate magic values" commit, so 136 entries in total.
The dates of these deprecations start at "2012-01-27" and end at "2013-08-19." The goal of this report is to see if paper
truly needs to maintain such old APIs or whether they can be largely safely dropped in 2026 with little cause for concern.

## Sample Design and pipeline

In order to test this it is required that I have a sampling of the ecosystem. This required me to produce a bulk-fetch downloader
from both sources of [Modrinth](https://modrinth.com/) and [Hangar](https://hangar.papermc.io/). Hangar has only ~4,100
plugin projects whereas Modrinth contains around ~20,000 plugins. I am going to sample these proportionally meaning Modrinth
plugins will contain 80% of the share of plugins while Hangar will only represent 20% of that share.

### Selections

In this sample there exists 3 types of selections, Popular, Legacy, and Random tail

| Source   | Popular | Legacy | Random tail | Total |
|----------|--------:|-------:|------------:|------:|
| Modrinth |     400 |    160 |         240 |   800 |
| Hangar   |     100 |     40 |          60 |   200 |
| **Total**| **500** | **200**|     **300** |**1000**|

**Popular**: The most popular plugins on the respective platforms
**Legacy**: Projects whose newest jar is at least 2 years old, also taken in download order. A plugin can be both popular
and legacy; it then counts toward both quotas, so the number of distinct jars can come out slightly under 1,000.
**Random tail**: Projects are drawn at random from everything ranked below the last popular picked item. Popular and legacy are
inherently biased to the most well-known plugins. The random tail is meant to select a wide range of plugins in the ecosystem.

### Jar Choosing

For a given project, the sampler takes its best Paper/Bukkit build: release channels come first, then the build targeting the newest
Minecraft version, then the most recently published. Jars in this sample are **not required** to support the latest Minecraft version.
Plugins listed on both sources (Modrinth and Hangar) are taken once by matching name and SHA-256 of the downloaded jar.
The source quotas count jars actually kept, so if a download fails or duplicates occur they are discarded without incrementing the count.

### Sampling Choices and Limitations

The biggest drawback of this sampling is SpigotMC not being sampled. It is home to many legacy plugins that a lot of communities still use to this day.
I chose not to include Spigot in the sampling because of SpigotMC's rules itself. While I could have used Spiget to sample Spigot, Paper no longer
guarantees compatibility with Spigot plugins, as such much more analysis would be required for plugins in the Spigot marketplace than feels appropriate here.
I do acknowledge this may skew results in the direction of such deprecated methods not being used as often. However, there in my mind exists no
easy way to verify such a large set of plugins can run smoothly on a modern PaperMC server to begin with.

Projects that also ship Fabric/Forge builds rank higher than their plugin usage alone would justify. The jar taken is still the Paper/Bukkit build, but this
does inflate projects that also have mod versions.

Legacy operates on a 2-year cutoff as such "Legacy" itself is dynamic and changes upon each run.

### Run

In this project exists a static-plugin-fetch which was run with this command
```bash
python3 static-plugin-fetch/fetch.py \
  --popular modrinth=400,hangar=100 \
  --legacy modrinth=160,hangar=40 \
  --random-tail modrinth=240,hangar=60 \
  --seed 1 --scan 3000 --allow-external --game-version 26.3
```

### Actual Data Analysis

| Source    | Popular | Legacy | Random tail | Distinct jars | Size     |
|-----------|--------:|-------:|------------:|--------------:|---------:|
| Modrinth  |     400 |    160 |         240 |           774 | 1.61 GB  |
| Hangar    |     100 |     40 |          60 |           184 | 0.49 GB  |
| **Total** | **500** |**200** |     **300** |       **958** |**2.10 GB**|

42 plugins (26 Modrinth, 16 Hangar) are both popular and legacy, so 1,000 selections give 958 distinct jars.
Median jar size is 207 KB; the largest is 82.8 MB.

#### Selection By Bucket

| Bucket      |   n | Download rank range                  | Median downloads | Median jar age | Marked compatible with 26.3 |
|-------------|----:|--------------------------------------|-----------------:|---------------:|----------------------------:|
| Popular     | 500 | Modrinth 1–406, Hangar 18–217         |           59,136 |       0.2 years |                  284 (57%) |
| Legacy      | 200 | Modrinth 18–1,722, Hangar 27–396      |            5,792 |       2.7 years |                   17 (8%)  |
| Random tail | 300 | Modrinth 437–19,859, Hangar 462–4,137 |              144 |       0.6 years |                   60 (20%) |
| **All**     | 958 |                                      |            6,994 |       0.5 years |                  356 (37%) |

Our random tail did a good job sampling a wide breadth of plugins with a median of 144 downloads. While 144 isn't necessarily obscure,
it gives us a good idea of the less popular side of the ecosystem.

#### Jar Age

| Bucket      | < 1 year  | 1–2 years | 2–3 years | 3+ years |
|-------------|----------:|----------:|----------:|---------:|
| Popular     | 406 (81%) |  52 (10%) |   21 (4%) |  21 (4%) |
| Legacy      |      0    |      0    | 117 (58%) | 83 (42%) |
| Random tail | 204 (68%) |  65 (22%) |   21 (7%) |  10 (3%) |
| **All**     | 610 (64%) | 117 (12%) | 138 (14%) | 93 (10%) |

The oldest jar in the data set was published 2022-08-15, Modrinth and Hangar plugin hosting doesn't go back further than
that in this sample. As expected from not including Spigot our plugin age does skew newer in this sample.

#### Newest Minecraft version each jar is marked for

| Bucket      | 26.x      | 1.21.x    | 1.20.x   | 1.19.x or older |
|-------------|----------:|----------:|---------:|----------------:|
| Popular     | 391 (78%) |  87 (17%) |  18 (4%) |          4 (1%) |
| Legacy      |  27 (14%) |  73 (36%) | 74 (37%) |        26 (13%) |
| Random tail | 116 (39%) | 161 (54%) |  17 (6%) |          5 (2%) |
| **All**     | 528 (55%) | 304 (32%) | 94 (10%) |         31 (3%) |

#### Skipped and replaced during selection

| Reason                                         | Modrinth | Hangar | Total |
|------------------------------------------------|---------:|-------:|------:|
| Duplicate of a plugin already taken            |       48 |    140 |   188 |
| No downloadable jar (e.g. external page link)  |        9 |     42 |    51 |
| Download failed (HTTP 404)                     |        0 |      5 |     5 |

Skipped projects were replaced by the next eligible one, so every quota was filled.
All jars were checked against the source's published hash except 21 Hangar jars hosted externally.
Release channels: 874 release, 70 beta, 10 alpha, 4 other.

## Deprecated Method Use

Before getting into the charts, the main takeaway is that **107 of the 136 deprecated entries aren't referenced by a single plugin in the sample.**
Only 60 of the 958 plugins (6.3%) use any of these entries from their own code. Of course, no plugin in my sample using something doesn't mean
nobody uses it. There's a rule of thumb for this called the "rule of three": if none of n plugins use something, you can be about 95% confident
the real usage rate is at most 3/n. For those 107 entries that means less than ~0.6% of popular plugins (3/500), and going off the random tail,
less than ~1% of the wider ecosystem (3/300). Though I imagine my confidence is actually less than than this because of our ommission of spigot in
the data set.

### What Analysis tell us

The static-analyzer reads the bytecode of every class in a jar and records any reference to one of the 136 entries. That said there are a few
things it can't tell us, and I think it's important to go over them before getting into the numbers.

- Just because a plugin references a method doesn't mean it actually calls it. A lot of plugins put old calls behind a version check, so that code may
  never run on a 26.3 server, and the analyzer has no way of telling the difference.
- 14 of the 136 entries are `MapPalette` color constants, and these are invisible to the analyzer. The Java compiler copies the value of a constant
  straight into the plugin, so they never show up in a static scan. All 14 of them are among the 107 unreferenced entries, as such I think 93 is
  probably the more accurate number.
- Library attribution is a best guess. Libraries are found by matching package and class names against known libraries, so the split between a
  plugin's own code and the libraries it bundles isn't perfect.
- 4 jars (UPnP, Authy, Simple RTP and SmeltedOres) shipped a full copy of the Bukkit API inside their jar by mistake. References inside those copies
  aren't the plugin's own usage so I excluded them. Counting them would have inflated the result to 60 of 136 entries.

![Plugins referencing the 136 oldest deprecated APIs](/writing/pressmc/figures/plugins-by-bucket.svg)

We see that out of the most popular plugins 17.2% reference at least one of the deprecated entries, but only 10.4% do so from their own code.
Our less downloaded legacy and random tail plugins are much less likely to be including these deprecated entries. Our random tail fared best:
4.0% of random tail plugins reference any of the entries, and only 1.0% do so from their own code; the remaining 3 percentage points come from bundled libraries.

![The 29 deprecated entries referenced by any of 958 plugins](/writing/pressmc/figures/entries-attribution.svg)

As seen above, the most used entries are concentrated on a few legacy "magic value" methods, mostly the byte data on `Block`, `BlockState` and
`MaterialData`, plus `EntityType#getName()` and `InventoryClickEvent#setCursor`. As seen in the graphic, most of these usages go through specific
libraries. Let's take a look at what these libraries are.

![Bundled libraries whose code references the deprecated entries](/writing/pressmc/figures/libraries.svg)

There exists in our data library attribution detection. Plugins relocate the libraries they bundle under their own package, so this is done by matching
package and class names against known libraries. Lookup was done to ensure these libraries exist. XSeries is the main contributor to deprecated references:
for 16 plugins that bundle XSeries, it is their only source of deprecated references. Likewise for ACF, 10 plugins' only use of deprecated code comes from ACF
itself, and all of it is a single method, `EntityType#getName()`, in ACF's tab completion. The biggest offender on this list by far is XSeries as it relates
specifically to multi-version compatibility. This is interesting so let's take a closer look at XSeries specifically and count up how it uses each deprecated reference.

![How much of the deprecated-API usage comes from bundled XSeries](/writing/pressmc/figures/xseries-share.svg)

For most of these entries, the majority of plugins referencing them do so only through XSeries; for `TreeSpecies#getData()` it's all 19 of them.
Across all 136 entries, XSeries accounts for 479 of the 1,083 reference sites (44%). ACF is responsible for another 14, meaning 2 libraries account
for about 46% of all deprecated references. Now that we know libraries are largely responsible for these deprecated references we need to understand
who or what plugins use these libraries. Specifically here we will look at XSeries because it is a compatibility library.

![The 28 sampled plugins that bundle XSeries](/writing/pressmc/figures/xseries-plugins.svg)

![How the 28 sampled plugins that bundle XSeries depend on it](/writing/pressmc/figures/xseries-dependence.svg)

In our sample 28 of the 958 plugins (2.9%) bundle XSeries. For 16 of those, XSeries is the only place they reference any of the deprecated entries,
and that includes SkinsRestorer, which is the most downloaded plugin on this list with 1.8M downloads. Another 7 reference the entries through XSeries
as well as other code, and the last 5 bundle XSeries but their copy of it doesn't reference any of the entries at all.

## Extra Metrics

![Age of the sampled jars](/writing/pressmc/figures/sample-jar-age.svg)
![Newest Minecraft version each sampled jar is marked for](/writing/pressmc/figures/sample-minecraft-version.svg)

As you would expect, popular plugins are mostly up to date. 95% of the popular plugins in our sample are marked for 1.21.x or later, and 78% are
marked for 26.x. The random tail lags behind a bit more, while 93% are marked for 1.21.x or later only 39% are marked for 26.x. It's worth noting
that "marked for" is just what the author claims on Modrinth or Hangar, it doesn't mean the plugin was actually tested on that version.
Within Legacy there exist plugins that are marked for 26.x, 1.21.x and 1.20.x. This is possible via updating the metadata on the Hangar and Modrinth
page after the fact. All 27 legacy jars marked for 26.x were published between 2023-07 and 2024-11, while 26.1 only came out on 2026-03-24. As seen
from the age of the sampled jars, the jar files themselves were never updated, only the page on the download site was.

## Conclusion

Going off of this sample, most of Paper's oldest deprecated API is effectively worth removing. 107 of 136 entries aren't referenced by a single plugin in the sample of
958 plugins, and only 60 plugins roughly 6.3% reference any deprecations from their own code. Most of the remaining usage doesn't come from plugin authors themsevles,
it comes from a handful of bundled libraries, mainly XSeries and ACF. Removal would mostly require these few libraries to update rather than thousands of individual plugins.

That said this sample doesn't include SpigotMC where the oldest plugins go to die. Our static analysis can't cover these references or determine if these plugins
run on a modern (Paper) server as such they can't measure the impact of this large set of legacy plugins on this data. My thought on this matter is likely that our plugin
sources skewed the data to include less deprecations than the average plugin.

These findings should be taken with a grain of salt. I was truly just interested in who is still using the oldest
deprecated methods. In the end, my thought process is that Paper isn't necessarily required to remove these methods, 
but I think it is worth looking into the existing maintenance burden of each of them and considering who actually benefits from them. 
From this small study it seems like very few plugins leverage such old methods, only 60 of 958 (6.3%) from their own code, 
and there might be a benefit to moving on after an open deprecation and sunsetting announcement for these APIs, for example by 
marking them forRemoval so plugin authors get a warning when they compile.

## The 93 deprecated APIs no sampled plugin references

![The 93 deprecated APIs no sampled plugin references](/writing/pressmc/figures/unused-deprecations.svg)

## AI Usage in this project

I used Claude Code throughout this project. The research
question, the decisions about what to sample and how (the 80/20 Modrinth/Hangar split, the popular, legacy and random tail buckets,
dropping the requirement that jars support 26.3, and leaving SpigotMC out), and the direction of the analysis were mine alone. 
Most of the code was written by AI at my direction: the static-analyzer (I set up its structure and data classes, AI implemented most of the bytecode
analysis), the static-plugin-fetch downloader, and the scripts that summarize the results and draw the charts. AI also dated the
deprecations by digging through the Bukkit, Spigot and Paper git history to produce oldest-deprecations.md and deprecated-data.csv,
generated the statistics tables, and proofread this report, and  correcting numbers. 
I reviewed the results along the way and contested back where charts or numbers looked confusing or wrong, and
several corrections came out of checking the data, like excluding jars that bundled the whole Bukkit API and fixing how library code
was told apart from a plugin's own code.

You can find a full list of exported data in report/data and figures in report/figures
