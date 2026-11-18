---
layout: post
title: "I Don't Want a Lesson in Computer Science!"
authors:
  - Hannes Mühleisen
thumb: /images/blog/thumbs/star-trek-enterprise.svg
image: /images/blog/thumbs/star-trek-enterprise.png
excerpt: "We built a DuckDB database of Star Trek ship logs and gave Claude a briefing. It wrote and ran 70 queries to answer 49 lines of dialogue from the show—matching how Picard's Enterprise computer solved hard data problems without explaining the computer science."
tag: using-duckdb
---

In ["The Battle"](https://www.st-minutiae.com/resources/scripts/110.txt), Picard's old ship, the Stargazer, is handed back to him with a personal log aboard in which he appears to confess to an unprovoked attack. Data compares it with the ship's main computer log and reports to the first officer.

| On the bridge | What the computer ran |
|---|---|
| **DATA:** By comparing the main Stargazer computer log with Captain Picard's personal log, I have found "checksum" discrepancies, sir.<br><br>**RIKER:** What does that mean?<br><br>**DATA:** All information is time-coded by entry, and the bits when totaled produce an aggregate amount which...<br><br>**RIKER:** I don't want a lesson in computer science! Bottom line?<br><br>**DATA:** One of these two logs is a forgery. | ```sql SELECT log_type,        count(*) AS entries,        count(*) FILTER (WHERE checksum = md5(entry))          AS checksum_valid,        count(*) FILTER (WHERE checksum <> md5(entry))          AS checksum_mismatch,        list(id ORDER BY id)          FILTER (WHERE checksum <> md5(entry))          AS mismatched_ids FROM ship_logs WHERE ship='USS Stargazer' GROUP BY log_type ORDER BY log_type ``` |

<div class="monospace_table"></div>

| log_type | entries | checksum_valid | checksum_mismatch | mismatched_ids |
|---|---|---|---|---|
| captain's personal log | 5 | 2 | 3 | [9, 10, 11] |
| main computer log | 7 | 7 | 0 | NULL |

Riker gets his bottom line. Behind it is a `GROUP BY` that recomputes the checksum of every log entry, compares it with the one stored when the entry was recorded, and counts mismatches per log. Three of the five entries in the captain's personal log fail. The main computer log is clean.

Riker speaks for most people who ask a question about their data. Nobody wants the lesson. They want the answer.

## Now We Have the Enterprise Computer

For seven seasons we watched the crew of the Enterprise talk to the computer and solve hard data problems. Now we have DuckDB and Claude, so we have the Enterprise computer.

That exchange is from 1987. The query next to it was not written by us. We gave Claude a DuckDB database and the crew's lines, one at a time, and it wrote and ran the queries.

Here is exactly what we did.

**The database.** We picked eleven scenes and built one DuckDB file, `enterprise_computer.duckdb`, to stand in for the ship's records: 28 tables and about 27,000 rows. The data is invented, but it is built so that the dialogue is true. If the computer says there are 15,525 undetectable substances, the table has 15,525 of them. It also holds believable rows that each condition has to filter out: "Darmok" entries in other sectors and in non-linguistic databases, substances that alter glass but show up on scans, log entries from another ship. The ship's own date and position sit in a one-row `ship_status` table, which is how "this sector" and "the past six months" get resolved.

**The session.** A fresh Claude session in Claude Code, in a folder containing only that file, allowed to run nothing but the `duckdb` command line. Its whole briefing was that it is the main computer of the USS Enterprise, the ship's records are in that database, and it should answer each crew member's line from it. Then we fed it 49 lines from the eleven scenes, 44 of them word for word from the scripts.

**The result.** We ran it twice and compared each answer with what the dialogue says, by reading them.

| Run | Extra instruction | Queries | Answers matching the dialogue |
|---|---|---|---|
| A | none | 29 queries on 17 of the lines | 44 of 49 |
| B | "answer every line with a new query for exactly what was asked" | 70 queries, at least one per line | 40 of 49 |

No query failed in either run. The queries in this post are from run B.

## More From the Ship's Records

The data problems are easy to miss, because the show never calls them that. We went through all 176 episode scripts and found 235 scenes where someone searches, filters, counts or cross-references records. Some are one-liners. Some carry the plot. Here are two more, again with the queries Claude ran.

Most of the time nobody is even offered the lesson; the crew just asks. In ["Darmok"](https://www.st-minutiae.com/resources/scripts/202.txt), the Enterprise meets a people who speak only in references to their own stories. Troi and Data have two words to go on, and they do what anyone with a search box would do.

| On the bridge | What the computer ran |
|---|---|
| **TROI:** Computer, search for the term "Darmok" in all linguistic databases compiled for this sector.<br><br>**COMPUTER:** Searching... "Darmok" is the name of a seventh dynasty emperor on Kanda Four... A mytho-historical hunter on Shantil Three... A colony on Malindi Seven... A frozen dessert on Razna Five... A -- | ```sql SELECT l.entry_id, l.term, l.description,        l.planet, l.sector, l.database_name FROM library l WHERE l.category = 'linguistic'   AND l.term ILIKE '%Darmok%'   AND l.sector = (SELECT sector FROM ship_status                   WHERE ship = 'USS Enterprise'                   ORDER BY ship_date DESC LIMIT 1) ORDER BY l.database_name, l.planet, l.entry_id ``` |

| entry_id | term | description | planet | sector | database_name |
|---|---|---|---|---|---|
| 202010 | Darmok | river | Beludan II | El-Adrel | Federation Linguistic Survey |
| 202127 | Darmok | clan title | Belugath V | El-Adrel | Federation Linguistic Survey |
| 202091 | Darmok | harvest festival | Corarin IX | El-Adrel | Federation Linguistic Survey |
| 202055 | Darmok | spice | Drensel V | El-Adrel | Federation Linguistic Survey |
| 202019 | Darmok | unit of weight | Ethamar IX | El-Adrel | Federation Linguistic Survey |

First 5 of 47 rows.

| On the bridge | What the computer ran |
|---|---|
| **TROI:** -- Stop search. Computer, how many entries for "Darmok"?<br><br>**COMPUTER:** Forty-seven. | ```sql SELECT count(*) AS darmok_entries FROM library l WHERE l.category = 'linguistic'   AND l.term ILIKE '%Darmok%'   AND l.sector = (SELECT sector FROM ship_status                   WHERE ship = 'USS Enterprise'                   ORDER BY ship_date DESC LIMIT 1) ``` |

| darmok_entries |
|---|
| 47 |

| On the bridge | What the computer ran |
|---|---|
| **DATA:** Computer, search for the term "Tanagra." All databases.<br><br>**COMPUTER:** Searching... Tanagra is the name of the ruling family on Gallos Two... A ceremonial drink on Lerishi Four... An island-continent on Shantil Three --<br><br>**DATA:** -- Stop. | ```sql SELECT entry_id, term, description, planet,        sector, database_name, category FROM library WHERE term ILIKE '%Tanagra%' ORDER BY database_name, planet, entry_id ``` |

| entry_id | term | description | planet | sector | database_name | category |
|---|---|---|---|---|---|---|
| 202205 | Tanagra | wedding dance | Ethadan V | El-Adrel | Federation Cultural Survey | cultural |
| 202209 | Tanagra | winter solstice feast | Ivomar IV | Ostrel | Federation Cultural Survey | cultural |
| 202202 | Tanagra | ceremonial drink | Lerishi IV | El-Adrel | Federation Cultural Survey | cultural |
| 202201 | Tanagra | ruling family | Gallos II | El-Adrel | Federation Historical Archive | historical |
| 202208 | Tanagra | guild of weavers | Halithos VII | Mendar | Federation Linguistic Survey | linguistic |

First 5 of 11 rows.

| On the bridge | What the computer ran |
|---|---|
| **TROI:** Shantil Three? Computer, cross-reference the last entry with the previous search index.<br><br>**COMPUTER:** "Darmok" is the name of a mytho-historical hunter on Shantil Three. | ```sql SELECT t.planet, t.sector,        d.term AS darmok_term,        d.description AS darmok_meaning,        d.database_name AS darmok_database,        t.term AS tanagra_term,        t.description AS tanagra_meaning,        t.database_name AS tanagra_database FROM library t JOIN library d ON d.planet = t.planet              AND d.term ILIKE '%Darmok%' WHERE t.term ILIKE '%Tanagra%'   AND t.planet = 'Shantil III' ``` |

| planet | sector | darmok_term | darmok_meaning | darmok_database | tanagra_term | tanagra_meaning | tanagra_database |
|---|---|---|---|---|---|---|---|
| Shantil III | El-Adrel | Darmok | mytho-historical hunter | Starfleet Xenolinguistics Archive | Tanagra | island-continent | Stellar Cartography Gazetteer |

A search with too many hits, a count, a second search, and a join that takes 47 rows down to one. Nobody on the bridge wrote any of the right-hand column. They asked, the computer answered, and they asked a sharper question.

In ["Hollow Pursuits"](https://www.st-minutiae.com/resources/scripts/169.txt) the ship is about to tear itself apart and Geordi narrows the cause one sentence at a time.

| In Engineering | What the computer ran |
|---|---|
| **GEORDI:** Computer, list all physical substances that wouldn't normally be picked up by internal scans.<br><br>**COMPUTER:** There are fifteen thousand five hundred twenty-five known substances that cannot be detected by standard scans. | ```sql SELECT name, stable_in_oxygen, alters_materials,        half_life_s, toxicity, last_used_year FROM substances WHERE NOT detectable_by_internal_scan ORDER BY name ``` |

| name | stable_in_oxygen | alters_materials | half_life_s | toxicity | last_used_year |
|---|---|---|---|---|---|
| Bryakbenate | false | [polymer] | 309.0 | high | NULL |
| Bryakbenide | false | [duranium, polymer] | inf | low | 2187 |
| Bryakbenon | false | [] | 263026.8 | low | 2343 |
| Bryakboride | false | [tritanium, ceramic] | 186208.7 | moderate | 2239 |
| Bryakborite | false | [] | inf | moderate | 2222 |

First 5 of 15,525 rows.

| In Engineering | What the computer ran |
|---|---|
| **GEORDI:** Great. How many of those can exist in an oxygen atmosphere?<br><br>**COMPUTER:** Five hundred thirty-two. | ```sql SELECT count(*) AS undetectable_and_stable_in_oxygen FROM substances WHERE NOT detectable_by_internal_scan   AND stable_in_oxygen ``` |

| undetectable_and_stable_in_oxygen |
|---|
| 532 |

| In Engineering | What the computer ran |
|---|---|
| **GEORDI:** And could alter molecular structure when it comes in contact with... with glass.<br><br>**COMPUTER:** Five. | ```sql SELECT name, alters_materials, half_life_s,        toxicity, last_used_year, molecular_structure FROM substances WHERE NOT detectable_by_internal_scan   AND stable_in_oxygen   AND list_contains(alters_materials, 'glass') ORDER BY name ``` |

| name | alters_materials | half_life_s | toxicity | last_used_year | molecular_structure |
|---|---|---|---|---|---|
| Invidium | [glass] | inf | low | 2259 | MSG-16900005 |
| Jakmanite | [glass] | 15.0 | moderate | NULL | MSG-16900001 |
| Lucrovexitrin | [glass] | inf | extreme | NULL | MSG-16900003 |
| Saltzgadum | [glass] | inf | low | 2331 | MSG-16900004 |
| Selgninaem | [glass] | inf | extreme | NULL | MSG-16900002 |

One question, a list. Another question, a count. A third refinement, a smaller filtered set. Each new constraint narrows the search. Claude built this the same way Geordi described it.

## The Questions We Asked

We've gone through all 176 episodes of The Next Generation, identified 235 scenes where someone asks the computer for data, and sketched the queries behind them. Here are the categories:

* **Finding things (110 scenes).** Searches, filters, "show me all X in this sector," range queries, pattern matches in text. The `WHERE` clause rules the ship.
* **Counting and aggregating (65 scenes).** "How many," "what's the average," rank by condition, top-N results. Most need a `GROUP BY`.
* **Joining and correlating (43 scenes).** "Check this against that," "what do we know about X," cross-referencing records, finding overlaps.
* **Time-based queries (17 scenes).** "Since when," "in the past six months," comparison with baseline.

The dialogue shows why SQL is the working language of data. Every problem on the ship is a `SELECT` statement, and every one is different.

---

Give DuckDB a dataset. Give Claude a goal. Point Claude at your database with a simple briefing and a command line, and watch what happens. It writes queries the way the Enterprise computer did: not to teach you computer science, but to answer your question.

Learn more about using DuckDB with Claude and other LLMs in the [documentation](https://duckdb.org/docs/).
