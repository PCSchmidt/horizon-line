# Horizon Line - Prior Art and Differentiation

**Status:** Draft v1. Limited web survey on 2026-10-30; store-page and forum material, not independently verified. Needs a deeper pass before any novelty claim.

**Correction to earlier planning notes:** the collection plan described this niche as "high whitespace." The survey shows the space is **not empty**: multiple long-range shooting trainers and sims already exist. The remaining opportunity is narrower: photoreal presentation, honest wind physics tied to visuals, and a competitive/ladder structure.

## 1. Existing products found
| Product | Type | Notes (from store/listing text) |
|---|---|---|
| [Yonder: Long Range](https://apps.apple.com/app/id6785105015) | Mobile trainer | Wind-call and DOPE drills; validated ballistics engine claimed |
| [Precision Academy](https://apps.apple.com/app/id6759255573) | Mobile simulator | "Certified ballistic engine" claimed |
| [Ridgeshot](https://apps.apple.com/qa/app/ridgeshot/id6774483008) | Mobile sim | Range, hold, wind and timing focus |
| [Shooter Ready](https://store.steampowered.com/app/5274250/) / other Steam listings | PC sim | Scenarios with varied wind and distance; some entries not yet released |
| [Ballistics Lab](https://rufin2003.itch.io/ballistics-lab) | PC | Drop/drift from published Hornady tables |
| Hardcore Sniper ([ModDB](https://www.moddb.com/games/hardcore-sniper)) | PC sim | Mil-ranging, wind correction, breath control, barrel heat drift |
| Expert Rifleman - Reloaded ([RAWG](https://rawg.io/games/expert-rifleman-reloaded)) | PC | Described as an early 3D ballistics simulator |
| Hornady 4DOF ([overview](https://www.hornady.com/team-hornady/ballistic-information/ballistic-resources/4dof-overview)) | Calculator | Uses Doppler-radar drag curves per projectile; proprietary data - **not usable here**; also supports plain G1/G7 BC mode |

Verify current availability, platforms and quality of each before relying on this table.

## 2. Real-sport context
Wind reading is a core skill in precision-rifle competition ([Wind Reading Basics, AccurateShooter](https://bulletin.accurateshooter.com/2026/01/wind-reading-basics-for-prs-and-nrl-competitors/)). Competition formats (PRS/NRL-style stages, ELR) inform the stage design.

## 3. Intended differentiation (to be validated)
1. **Honest wind**: visible cues and ballistics share one wind field, with automated tests (S3).
2. **Photoreal long-range optics**: scope rendering, mirage, haze and distance rendering as part of the skill.
3. **Competitive structure**: seeded stages, server-verified replays, bot-calibrated ratings.
4. **Learned wind reader and optional agents with published evaluation**: a research-flavoured contribution.
5. **Open specification, code and benchmark data.**

## 4. Constraints from the survey
- Use only **public drag data** (G1/G7 BCs and published Cd tables); do not use proprietary measured curves.
- Avoid claiming "certified" or "validated" accuracy; report validation against cited references.

## 5. Open research tasks
- Primary sources for external ballistics (published texts, standard drag functions) and test fixtures.
- Survey of sniper/shooting video games to see which already do mirage and wind rendering.
- Review PRS/NRL stage formats and scoring rules for authentic mode design.
- Platform and ratings considerations for weapons-themed titles on storefronts.
