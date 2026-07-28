# District Expansion Methodology

This note documents how the Hervanta, Annala, Kaukajarvi, Hervantajarvi, Hallila, Lukonmaki, Vuores, Turtola, Peltolammi, Multisilta, Vehmainen, Atala, Viiala, Lentavanniemi, and Messukyla housing-company manager map was built, and how to repeat the same approach for nearby districts.

The goal is to produce one map row per relevant apartment-building / housing-company relationship, with:

- district
- canonical address and coordinates
- housing-company name
- business ID
- manager company
- building year and age in 2026
- apartment count
- source links for the housing company, year, and count

The publishable output is `docs/index.html`. The data behind it is `data/maps/hervanta_manager_groups_plot.csv`.

## Current Outputs

- `docs/index.html`: dynamic OpenStreetMap map for Hervanta + Annala + Kaukajarvi + Hervantajarvi + Hallila + Lukonmaki + Vuores + Peltolammi + one manual Multisilta row + Vehmainen + Atala + Lentavanniemi + Messukyla, with Turtola and Viiala present as zero-row district filters after the latest evidence pass.
- `docs/offline.html`: offline raster snapshot. At the moment this is safest for Hervanta because the saved basemap tiles were originally prepared around Hervanta.
- `data/maps/hervanta_manager_groups_plot.csv`: merged map dataset used by the HTML.
- `data/interim/etuovi_taloyhtio_apartment_counts.csv`: company-level apartment-count cache.

## High-Level Pipeline

1. Build the apartment-building universe for a district from OpenStreetMap / Overpass.
2. Convert the raw OSM address rows into the canonical city-map format.
3. Find housing-company evidence for each canonical building.
4. Resolve split-building cases where one OSM base address actually contains multiple housing companies.
5. Fetch or infer manager company data from Virre / registry evidence.
6. Attach building years and source URLs.
7. Attach apartment counts and source URLs.
8. Regenerate the combined map and verify coverage.

The important principle is that the map should not assume "one address equals one housing company". Annala showed that base buildings can split into A/B/C or otherwise separate housing-company rows. Those split rows must be represented explicitly.

## Step 1: Fetch District Addresses From OSM

For Annala, this was done with:

```bash
python scripts/fetch_district_addresses_osm.py annala
```

Inputs / code:

- `scripts/fetch_district_addresses_osm.py`
- district bounding boxes in the script's `DISTRICTS` constant

Outputs:

- `data/raw/osm/annala_addresses_overpass.json`
- `data/canonical/annala_addresses_osm.csv`

For a new district, add a new entry to `DISTRICTS`:

```python
"kaukajarvi": {
    "label": "Kaukajarvi",
    "south": ...,
    "north": ...,
    "west": ...,
    "east": ...,
},
```

Then run:

```bash
python scripts/fetch_district_addresses_osm.py kaukajarvi
```

Use `--all-addresses` only for auditing. The map pipeline wants `building=apartments` rows.

## Step 2: Build The Canonical Building Universe

For Annala, this was done with:

```bash
python scripts/build_district_building_universe.py annala
```

Output:

- `data/canonical/annala_apartment_buildings_city_map.csv`

This gives one canonical row per OSM apartment-building address, with empty company / business-ID / manager fields ready to be filled.

Hervanta originally used the older district-specific script:

```bash
python scripts/build_hervanta_building_universe.py
```

For future districts, prefer the generic script:

```bash
python scripts/build_district_building_universe.py <district>
```

## Step 3: Discover Housing Companies

The company evidence comes from several sources:

- Etuovi listing pages, which can expose housing-company names.
- Etuovi `taloyhtiot` company pages, which can expose business IDs, building years, and apartment counts.
- Oikotie listing / talo pages when Etuovi does not expose a company page.
- Manual Google/browser discovery for missing cases.
- Manual correction CSVs for edge cases.

Relevant files used in the current repo:

- `data/interim/etuovi_listing_company_evidence.csv`
- `data/interim/etuovi_taloyhtiot_building_evidence.csv`
- `data/interim/google_etuovi_taloyhtiot_company_evidence.csv`
- `data/interim/oikotie_rental_building_company_evidence.csv`
- `data/interim/manual_building_company_corrections.csv`
- `data/interim/annala_company_evidence_pilot.csv`

For Hervanta, the canonical matched dataset is:

- `data/canonical/hervanta_apartment_buildings_company_matched.csv`

For Annala, the pilot evidence was consolidated into:

- `data/interim/annala_company_evidence_pilot.csv`

Then converted into:

```bash
python scripts/build_annala_company_candidates.py
```

Outputs:

- `data/canonical/annala_apartment_buildings_company_candidates.csv`
- `data/canonical/annala_apartment_buildings_company_matched.csv`
- `data/canonical/annala_apartment_buildings_company_unresolved.csv`

For the next district, use the generic candidate builder where possible:

```bash
python scripts/build_kaukajarvi_company_candidates.py <district>
```

Despite the historical filename, it now has district configs for Kaukajarvi, Hervantajarvi, Hallila, Lukonmaki, Turtola, Vuores, Peltolammi, Multisilta, Vehmainen, Atala, Viiala, Lentavanniemi, and Messukyla.

## Step 4: Handle Split Buildings

Annala revealed an important matching issue: OSM may contain only one base address, but the real estate evidence may show separate housing companies by stairwell or subaddress.

Example pattern:

- OSM row: `Ojavainionkatu 4`
- Evidence rows: `Ojavainionkatu 4 A`, `Ojavainionkatu 4 B`, `Ojavainionkatu 4 C`

The correct result is to keep the base OSM row as a split marker / audit row and create supplemental subaddress rows for the actual housing companies. This is implemented in `scripts/build_annala_company_candidates.py`:

- base rows get `company_match_status = split_into_subaddress_rows`
- supplemental rows get `city_map_source = manual_supplemental_subaddress_from_listing_or_virre`

For new districts, always audit cases where:

- one address has multiple company candidates
- a company source address contains a letter suffix missing from OSM
- Etuovi/Oikotie lists several taloyhtiot under one physical building footprint

## Step 5: Infer Managers

Managers were inferred from Virre-derived registry details, using:

```bash
python scripts/infer_virre_managers.py
```

Relevant outputs:

- `data/interim/virre/hervanta_private_housing_company_virre_manager_inferred.csv`
- `data/interim/virre/annala_pilot_virre_manager_inferred.csv`

The inference logic uses:

- postal address care-of text
- known website domains
- known email domains
- a small set of manual manager overrides

The important output columns are:

- `inferred_manager_company`
- `inferred_manager_source`
- `inferred_manager_evidence`
- consistency columns for postal/email/website conflicts

For new districts, build a Virre queue with company names / business IDs, fetch or add registry details, then run the inference step. If a company has no manager after this step, keep it in the dataset but audit it separately.

## Step 6: Attach Building Years

Hervanta years are loaded from:

- `data/analysis/hervanta_building_age_analysis_2026.csv`

Annala years are loaded from:

- `data/interim/annala_company_evidence_pilot.csv`

The current map generator reads both in `scripts/map_hervanta_manager_groups.py`.

For Annala, the important fields are:

- `building_year`
- `evidence_status`
- `etuovi_taloyhtio_url`

The map generator converts `building_year` into:

- `age_year`
- `age_in_2026`
- `age_source`
- `building_year_source_urls`

Ranges are allowed. For example, `2021 - 2022` becomes age `4-5` in 2026.

For future districts, use the same fields and source conventions so the map template does not need new per-district display logic.

## Step 7: Attach Apartment Counts And Links

The shared count cache is:

- `data/interim/etuovi_taloyhtio_apartment_counts.csv`

The main fetcher is:

```bash
python scripts/fetch_etuovi_taloyhtio_apartment_counts.py
```

It reads map rows, follows Etuovi `taloyhtiot` links, parses `apartmentCount` from cached or fetched HTML, and writes the count cache.

Important behavior added during Annala/Hervanta repair:

- if a current map row has no direct Etuovi URL, the script preserves an existing count/source row instead of overwriting it with a blank placeholder
- this prevents losing older evidence for companies such as Satoreino and Satosanteri

The cache can also be restored from raw saved Etuovi company pages:

```bash
python scripts/restore_apartment_counts_from_cache.py
```

That script scans:

- `data/raw/etuovi_taloyhtiot/pages/*.html`

and rebuilds count/source rows from cached HTML.

Annala-specific supplemental count/link handling is in:

```bash
python scripts/build_annala_apartment_links_counts.py
```

This fills Annala count gaps from:

- cached Etuovi taloyhtiot pages
- NCC project page for Annalan Tahti
- Oikotie rental listing for Satoriekko

For a new district, use Etuovi company pages first, then add clearly labeled supplemental rows for non-Etuovi evidence.

## Step 8: Regenerate The Map

Run:

```bash
python scripts/map_hervanta_manager_groups.py
```

Outputs:

- `data/maps/kiinteistotahkola_hervanta.html`
- `data/maps/kiinteistotahkola_hervanta_dynamic.html`
- `data/maps/hervanta_manager_groups_plot.csv`
- `docs/index.html`
- `docs/offline.html`

The current map generator:

- combines Hervanta + Annala + Kaukajarvi + Hervantajarvi + Hallila + Lukonmaki + Vuores + Peltolammi + Multisilta + Vehmainen + Atala + Lentavanniemi + Messukyla
- keeps Turtola and Viiala available in the district selector even when the current matched row count is zero
- includes manager groups with 1 or more mapped addresses
- shows building year and age in 2026
- shows apartment counts
- links to source pages
- omits popup-only `Count source` text

## Verification Checklist

After regenerating, check:

```bash
python -m compileall scripts/map_hervanta_manager_groups.py
python scripts/map_hervanta_manager_groups.py
```

Then verify the output CSV:

- every expected district row is present
- no expected Annala/Hervanta row lost manager data
- no expected row lost `apartment_count`
- no expected row lost `apartment_count_source_url`
- no expected row lost `age_year`
- split subaddress rows are present as separate map rows
- `docs/index.html` no longer contains unwanted labels such as `Count source`

Useful checks:

```bash
rg -n "Count source" docs/index.html docs/offline.html data/maps/*.html scripts/map_hervanta_manager_groups.py
```

For Annala specifically, the last verified state was:

- 30 Annala map rows
- 30 / 30 with manager companies in the generated map CSV
- 30 / 30 with building years
- 30 / 30 with apartment counts and source URLs

## What To Generalize Before Adding Many Districts

The current code works, but several files still have Hervanta/Annala names baked in. Before adding multiple areas, it would be cleaner to generalize:

- `scripts/build_annala_company_candidates.py` into `scripts/build_district_company_candidates.py`
- `scripts/build_annala_apartment_links_counts.py` into `scripts/build_district_apartment_links_counts.py`
- `scripts/map_hervanta_manager_groups.py` into a district-list driven map builder
- constants for evidence files, matched files, and year-source files into one config table

Suggested config shape:

```python
DISTRICT_INPUTS = {
    "Hervanta": {
        "matched_csv": "data/canonical/hervanta_apartment_buildings_company_matched.csv",
        "year_csv": "data/analysis/hervanta_building_age_analysis_2026.csv",
    },
    "Annala": {
        "matched_csv": "data/canonical/annala_apartment_buildings_company_matched.csv",
        "year_csv": "data/interim/annala_company_evidence_pilot.csv",
    },
}
```

## Nearby Areas To Add Next

Kaukajarvi, an initial Hervantajarvi pass, Hallila, Lukonmaki, Vuores, Peltolammi, one conservative Multisilta row, Vehmainen, Atala, Lentavanniemi, and Messukyla have now been added using the same approach. Turtola has also been prepared through the OSM address-universe step, but the current Etuovi pass did not produce manager-matched apartment-company rows. Viiala produced no manager-matched apartment-company rows in the current Etuovi filter pass. The latest Lentavanniemi pass reached 34 apartment-building listing records before Etuovi 429 rate limits, yielding 24 manager-matched map rows. The latest Messukyla pass collected 12 apartment-listing records, collapsed them into 2 housing-company candidates, and added 1 manager-matched map row; the other candidate is a 2028 building whose Etuovi evidence says the manager will be selected later.

- Taatala / Koivistonkyla: west of Kaukajarvi/Turtola and dense enough to be worth a pass.
- Nekala / Hatanpaa: a larger comparison area with older housing-company stock.

Recommended order:

1. Taatala / Koivistonkyla
2. Nekala / Hatanpaa

## Minimal New-District Runbook

For a new district named `<district>`:

1. Add its OSM bounding box to `scripts/fetch_district_addresses_osm.py`.
2. Run `python scripts/fetch_district_addresses_osm.py <district>`.
3. Run `python scripts/build_district_building_universe.py <district>`.
4. Discover Etuovi/Oikotie/company evidence and write `data/interim/<district>_company_evidence_pilot.csv`.
5. Build matched/unresolved canonical company rows.
6. Build/fetch Virre manager details and run manager inference.
7. Fetch or restore apartment counts.
8. Add the district's matched/year inputs to the map generator.
9. Run `python scripts/map_hervanta_manager_groups.py`.
10. Audit the output CSV before trusting the HTML.
