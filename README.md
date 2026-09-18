Darukaa BioIntel

An evidence-grounded biodiversity advisory engine. It diagnoses what is actually limiting a parcel of land, retrieves only the evidence that applies to those conditions, and traces each proposed action through a graph of coupled environmental metrics — so every recommendation arrives with a mechanism, a set of metrics it moves, a time horizon, a confidence figure, and a citation.

Live demo: see the submission document for the hosted URL.

The design argument

Most land-management advice fails not because it is wrong in general but because it treats the wrong problem well. Cover crops are excellent advice on a carbon-depleted soil and close to useless on a soil at 1.4% organic carbon that is losing birds because the landscape was cleared. A system that retrieves "biodiversity improvement" text and paraphrases it cannot tell those two sites apart.

Three structural decisions follow from that.

1. Diagnosis precedes retrieval. A rule layer reads the structured site profile and decides which constraints are binding, with a severity for each. Those constraints then drive query expansion, candidate scoring and sequencing. The engine also emits stop conditions — pressures such as annual residue burning or calendar pesticide spraying that will negate ecological investment regardless of what else is done — and puts them ahead of the plan.

2. The site profile is a hard gate on the corpus, not a ranking hint. A card whose applicability window excludes the site is refused with a stated reason that appears in the API response. On a pH 8.7 soil, the liming card is excluded and says why. This is the difference between a retrieval system and a search box.

3. The reasoning layer is deterministic and the language model only phrases. The engine produces a typed Analysis object; the model receives it as JSON and is forbidden from introducing any claim not present in it. Without an API key the built-in renderer produces the same content with no model at all — which is what the test suite asserts against. The consequence is that recommendations are stable, diffable and testable.

Architecture
                       ┌──────────────────────────────────────────────┐
  free text  ─────────▶│  conversation/state.py                       │
  JSON payload ───────▶│  slot extraction · merge · journal · memory  │
  geo-coords ─────────▶└───────────────────┬──────────────────────────┘
                                           │  SiteProfile (typed, partial)
                                           ▼
                       ┌──────────────────────────────────────────────┐
                       │  reasoning/diagnostics.py                    │
                       │  binding constraints + severity              │
                       │  stop conditions · recorded unknowns         │
                       └───────────────────┬──────────────────────────┘
                                           │  Diagnosis
                   ┌───────────────────────┴───────────────────────┐
                   ▼                                               │
   ┌───────────────────────────────┐                               │
   │  kb/retriever.py              │                               │
   │  query expansion per          │                               │
   │  constraint                   │                               │
   │    ├─ dense cosine  (kb/embeddings.py)                        │
   │    ├─ BM25          (kb/vector_store.py)                      │
   │    ├─ RRF fusion                                              │
   │    └─ applicability gate ──▶ excluded + reason                │
   └───────────────┬───────────────┘                               │
                   │  RetrievalHit[]                               │
                   ▼                                               ▼
   ┌───────────────────────────────────────────────────────────────────┐
   │  reasoning/engine.py                                              │
   │   severity-weighted leverage · keystone floors · diversity guard  │
   │   causal propagation through reasoning/ontology.py                │
   │   interaction + conflict detection · sequencing · confidence      │
   └───────────────┬───────────────────────────────────────────────────┘
                   │  Analysis
       ┌───────────┴────────────┐
       ▼                        ▼
 ┌──────────────┐    ┌────────────────────────────────┐
 │ llm/client   │    │ conversation/agent.py           │
 │ render_md    │    │ counterfactual question choice   │
 │ narrate(opt) │    └────────────────────────────────┘
 └──────────────┘
Knowledge system

The corpus is 34 evidence cards in data/knowledge/*.yaml, each one a retrievable unit with a mechanism, quantified metric effects, an applicability window, prerequisites, trade-offs, synergy and conflict links, cost, reversibility and citations.

Card kinds:

kind	role
intervention	an action the engine may recommend
mechanism	why a coupling exists; retrievable but never recommended
context	diagnostic thresholds and classifications
caution	actions that look like restoration and are not

Sources are published meta-analyses and assessments — FAO, IPCC AR6, IPBES 2019, Poeplau & Don (2015), Powlson et al. (2014), Haddad et al. (2015), Damschen et al. (2019), Albrecht et al. (2020), Garibaldi et al. (2013), Minasny & McBratney (2018), Delgado-Baquerizo et al. (2016), Tscharntke et al. (2012), Jeffery et al. (2017), Davis et al. (2018), Reij et al. (2009), Veldman et al. (2015), and others. Several cards exist specifically to prevent plausible bad advice: the limits of no-till carbon sequestration, the order-of-magnitude overstatement of the SOC-to-available-water relationship, and afforestation of grassy biomes being recorded as restoration.

Retrieval pipeline (all three stages visible in POST /kb/search):

Recall. Dense cosine over card embeddings and Okapi BM25 over the same text, run independently. Lexical recall is not optional here: queries carry exact terms — sodic, Faidherbia, neonicotinoid — that a small dense encoder smooths away.
Fusion. Reciprocal Rank Fusion, chosen over score normalisation because BM25 and cosine are not on comparable scales and RRF is robust to that without tuning.
Gating. The SiteProfile is applied as a hard filter plus a soft fit score. Exclusions carry reasons. Unknown values never exclude — they simply earn no fit credit, which caps confidence downstream.

Encoder is all-MiniLM-L6-v2 when sentence-transformers is installed, and a deterministic hashed n-gram encoder otherwise, so CI and offline use run the real pipeline rather than skipping it. A ChromaVectorStore adapter is included for when the corpus outgrows exact search; the retriever contract does not change.

Multi-metric reasoning

data/ontology/metrics.yaml defines 21 metrics and 37 signed causal edges, each with a coupling weight, a lag in seasons, and the ecological reason for the link. Signs are expressed in terms of metric value, not of "good" or "bad", so they compose correctly along a path: residue retention lowers erosion, a negative edge carries that to water quality, and water quality rises — three hops from an action nobody described as a water-quality measure.

Propagation is breadth-first to depth 3 with attenuation and a minimum-weight cutoff, keeps the strongest path to each node, and refuses to revisit a node already on the path. Every propagated delta reports the path it travelled and the citation on the final edge.

Ranking combines severity-weighted constraint alignment, alignment with the primary constraint, domain breadth, evidence strength, cost and site fit. Two overrides sit on top:

Keystone floors. Some actions are a necessary condition for relieving a constraint — other measures under-deliver until they are in place. These get a leverage floor rather than losing to a broader, cheaper practice on aggregate score.
Diversity guard. A fifth carbon practice adds less than a first water practice, so a candidate covering no new metric is dropped — unless its leverage is within 12% of the best on the list.

The engine then detects synergies, declared conflicts, and metric-level antagonism (two recommendations pushing the same metric in opposite directions), and surfaces all of them instead of hiding them.

Conversational intelligence

Clarifying questions are chosen by counterfactual information gain. For each unknown slot the agent fills it with two plausible extreme values, re-runs the entire analysis under both, and measures the Jaccard shift in the recommendation set and in the diagnosis. Slots that do not change the answer are not asked about — which is why the agent asks about pH on an unclassified soil but not about slope on a paddy.

The agent answers before it asks: it gives a provisional analysis from what it has, states that confidence is capped by the missing data, then asks. Questions are never repeated within a session.

Memory is the profile. Free text and structured JSON write into the same SiteProfile, so a user can type "SOC is 0.3%" in turn one, paste JSON in turn three, and correct "actually it's 0.9%" in turn five and the engine sees one coherent site. Every write is journalled with its source, so a correction is visibly a correction.

Repository layout
data/knowledge/*.yaml        34 evidence cards (the corpus; single source of truth)
data/ontology/metrics.yaml   21 metrics, 37 signed causal edges, declared antagonisms
src/biointel/
  schemas.py                 typed contracts crossing every module boundary
  config.py                  env-driven configuration
  kb/loader.py               YAML → EvidenceCard, with referential-integrity checks
  kb/embeddings.py           sentence-transformers + deterministic fallback encoder
  kb/vector_store.py         persistent numpy vector store, BM25, Chroma adapter
  kb/retriever.py            hybrid recall → RRF → site gating
  reasoning/ontology.py      causal graph and signed propagation
  reasoning/diagnostics.py   constraint rules, severities, stop conditions
  reasoning/engine.py        leverage, deltas, interactions, sequencing, confidence
  conversation/state.py      slot extraction, merge, journal, session store
  conversation/agent.py      counterfactual question selection, turn handling
  llm/client.py              deterministic renderer + optional model narration
  api/main.py                FastAPI surface
  cli.py                     interactive and one-shot terminal client
tests/test_system.py         26 behavioural tests
eval/run_eval.py             golden-case harness (CI gate at 0.80)
eval/golden_cases.json       8 scored scenarios incl. the brief's worked example
scripts/export_web_kb.py     regenerates the browser demo's corpus from the YAML
web/index.html               self-contained demo (engine ported to JS, corpus inlined)
Data model

EvidenceCard — id, title, kind, domains, summary, mechanism, effects[] (metric, direction, magnitude, horizon, confidence), applicability (land_use[], climate[], and numeric windows on soc_pct, ph, rainfall_mm, slope_pct, semi_natural_pct), prerequisites[], tradeoffs[], synergies[], conflicts[], cost, reversibility, sources[], tags[].

SiteProfile — every field optional: land use, climate zone, region, lat/lon, area, rainfall, temperature, SOC, pH, texture, slope, semi-natural share, tree density, crop, cropping pattern, irrigation source, livestock, species observations, pressures, concerns.

Recommendation — action, rationale, metric_deltas[] (direct and propagated, each with horizon, confidence, path and reason), horizon, confidence, confidence_basis[], prerequisites[], tradeoffs[], sources[], leverage_score, sequence_stage, monitoring[].

There is no relational database. The corpus is version-controlled YAML — reviewable in a pull request by a domain scientist, which a row in Postgres is not — and the vector index is a persisted numpy matrix in .index/, rebuilt automatically when the encoder changes. Session memory is in-process behind a SessionStore interface with two methods, so Redis or Postgres is a drop-in replacement when horizontal scaling is needed.

Local setup
bash
git clone <repo-url> && cd darukaa-biointel
python -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt

# optional: real encoder, Chroma backend, model narration
pip install -r requirements-optional.txt
export ANTHROPIC_API_KEY=sk-...          # optional; renderer is used without it

pytest -q                                 # 26 behavioural tests
python eval/run_eval.py                   # golden-case scores
uvicorn biointel.api.main:app --reload    # http://localhost:8000/docs

CLI:

bash
python -m biointel.cli                                  # conversational
python -m biointel.cli --json site.json --question "biodiversity is declining"
python -m biointel.cli --search "sodic soil infiltration"   # inspect retrieval

Docker:

bash
docker compose up --build      # http://localhost:8000

The image warms the vector index at build time so the first request is not the slow one.

Environment variables
variable	default	effect
ANTHROPIC_API_KEY	unset	enables model narration; renderer used when absent
BIOINTEL_USE_ST	auto	no forces the deterministic encoder (used in CI)
BIOINTEL_EMBED_MODEL	all-MiniLM-L6-v2	dense encoder
BIOINTEL_INDEX_DIR	.index	vector index location
BIOINTEL_MAX_RECS	5	recommendations per analysis
API
method	path	purpose
GET	/health	status, card count, active encoder
POST	/chat	conversational turn: reply, analysis, updated profile, questions
POST	/analyze	one-shot structured analysis, optionally rendered to markdown
POST	/kb/search	raw retrieval with ranks, fit scores and exclusion reasons
GET	/kb/cards	the full corpus
GET	/session/{id}	profile, write journal, asked slots, turn history
DELETE	/session/{id}	reset memory
bash
curl -X POST localhost:8000/chat -H 'content-type: application/json' -d '{
  "session_id": "demo",
  "message": "Biodiversity is declining on my land",
  "structured": {"land_use":"cropland","climate_zone":"semi_arid","soc_pct":0.3,
                 "rainfall_mm":420,"cropping_pattern":"monoculture","semi_natural_pct":8,
                 "pressures":["residue_burning"]}
}'
Testing and evaluation

pytest asserts on reasoning properties rather than strings — the things that would make the system wrong in a way a user could not detect:

liming is never recommended on an alkaline soil, and the refusal carries a reason
dryland planting-pit advice never reaches a humid paddy
exact technical terms are recoverable (Faidherbia albida, neonicotinoid)
a healthy-carbon site is not diagnosed as a carbon problem
causal signs compose correctly along multi-hop paths
every recommendation spans at least two domains and includes a propagated effect
stop conditions outrank optimisation in the sequence
corrections update memory and are journalled as corrections
information gain is lower for slots that do not change the answer

eval/run_eval.py scores 8 scenarios on inclusion, exclusion, diagnosis, domain breadth and citation coverage. Cases include the brief's worked example plus adversarial ones: an acidic site where the user's proposed legume strategy would fail until pH is corrected, a groundwater overdraft where the user asks about drip irrigation and the answer is crop choice, and a site with healthy soil that is losing birds for landscape reasons.

Current status: 26/26 tests passing, golden-case mean 1.00, ruff clean.

CI/CD

.github/workflows/ci.yml runs on every push and pull request:

test — matrix over Python 3.11 and 3.12: ruff check, then pytest, then python eval/run_eval.py, which exits non-zero below a 0.80 mean. A change to the corpus or the scoring function that degrades reasoning quality fails the build, not just a syntax error.
docker — builds the image, starts it, polls /health, and issues a real /chat request against the running container.

CI pins BIOINTEL_USE_ST=no so the build does not depend on a model download and remains deterministic.

Notes and limitations
The hosted demo runs the engine in the browser with the corpus inlined by scripts/export_web_kb.py, so the demo and the API cannot drift apart on content. The browser build's fallback encoder uses a different hash function from the Python one, so candidate ordering can differ slightly between the two. Gating, diagnosis, causal propagation and scoring logic are identical.
Effect sizes are indicative ranges from published syntheses, not local predictions. They belong in a plan as expectations to be measured against, which is why every recommendation carries a monitoring line.
The corpus is weighted toward South Asian and dryland smallholder systems. Temperate intensive and humid tropical forest systems are represented but thinner; adding coverage is a YAML pull request, not a code change.
Geo-coordinates are parsed and stored but not yet joined to gridded soil, climate or land cover rasters. That join — SoilGrids, CHIRPS, ESA WorldCover — is the natural next step and would let the system populate most of the profile from a point instead of asking for it.
