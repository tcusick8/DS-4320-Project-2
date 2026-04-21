# DS-4320-Project-2: Humans Dont Learn 30 Years of Wildfire Data Reveals Preventable Crisis

> [!NOTE]
> EXECUTIVE SUMMARY

---

| Spec | Value |
|---|---|
| Name | Thomas Cusick |
| NetID | tpg6hu |
| DOI | [https://doi.org/10.5281/zenodo.19667633](https://doi.org/10.5281/zenodo.19667633) |
| Press Release | [XXXXXXX](Press_Release.md) |
| Pipeline | [XXX](XXX) |
| License | [MIT](License.md) |

---

## Problem Definition

**General Problem:**

Predicting wildfire risk

**Specific refined statement:**

Predicting the size class and cause cateogry of US wildfire incidents using historical fire records (1992-2020) enriched with geographic and seasonal context to help land managers and fire agencies prioritize prevention resourced by identifying which conditions and locations produce the most destructive fires.

**Motivation:**

Wildfires have become one of the most costly and deadly natural disasters in the United States, burning millions of acres annually and displacing hundreds of thousands of people. Despite significant investment in firefighting resources, prevention and early intervention remain underfunded relative to suppression, largely because land managers lack reliable, data-driven tools to identify where and why the most destructive fires originate. The 2020 fire season alone burned over 10 million acres nationally, and climate projections suggest that fire seasons will continue to grow longer and more severe. A model that can predict fire size and likely cause from early incident characteristics gives agencies a meaningful head start in allocating prevention resources before fires grow out of control.

**Rationale for refinement:**

The general problem of predicting wildfire risk is broad enough to encompass everything from satellite image classification to atmospheric modeling. This project narrows the scope to a tabular, incident-level prediction task, specifically size class and cause, because these two outcomes are directly actionable for land managers: knowing a fire is likely to grow large changes suppression priority, and knowing its probable cause (lightning vs. debris burning vs. arson) changes prevention strategy. The 1992–2020 USFS Fire Occurrence Database was chosen because it is the most comprehensive, publicly available, and well-documented wildfire record in existence, with consistent schema across nearly three decades. The document model is a natural fit because each fire incident bundles heterogeneous attributes like spatial coordinates, discovery timing, jurisdictional metadata, cause codes, and containment outcomes, which vary in completeness across records, exactly the kind of semi-structured, entity-centric data MongoDB handles better than rigid relational tables.

[XXXXXX](Press_Release.md)

---

## Domain Exposition

**Terminology:**

| Term | Definition |
|---|---|
| **Fire Size Class** | USFS classification of a fire by acres burned: A (<0.25 ac), B (0.26–9.9), C (10–99.9), D (100–299), E (300–999), F (1000–4999), G (5000+) |
| **Cause Code** | Categorical code for fire ignition source (e.g., lightning, campfire, debris burning, arson, equipment use) |
| **Containment Date** | The date a fire is declared fully controlled — used to derive burn duration |
| **WUI (Wildland-Urban Interface)** | The zone where human development meets undeveloped wildland vegetation — highest risk area for structure loss |
| **Burn Probability** | The modeled likelihood that a given location will experience fire in any given year |
| **RAWS** | Remote Automated Weather Stations — ground-based sensors providing real-time fire weather data |
| **Fire Weather** | Atmospheric conditions (low humidity, high temp, high wind) that promote rapid fire spread |
| **Suppression Cost** | Total expenditure to control a fire — often used as a proxy for fire severity |
| **Discovery Date** | The date a fire was first reported — used to derive seasonality features |
| **POO (Point of Origin)** | The geographic coordinates where a fire ignited |
| **NWCG** | National Wildfire Coordinating Group — the federal body that standardizes wildfire data and terminology |
| **Fire Year** | The calendar year in which a fire was discovered — key temporal unit for trend analysis |

**Project Domain:**

This project lives at the intersection of environmental data science, natural resource management, and public safety. Wildfire management in the US is coordinated across a complex web of federal, state, and local agencies, including the USDA Forest Service, Bureau of Land Management, and state fire departments, each maintaining their own incident records that feed into national databases like the one used here. The core analytical challenge is that wildfire behavior is driven by a highly nonlinear combination of fuel load, weather, terrain, and ignition source, making prediction difficult but not intractable when sufficient historical data is available. Analysts in this domain work closely with GIS tools, satellite remote sensing, and increasingly machine learning to move from reactive suppression toward proactive risk modeling. The document model is particularly well-suited here because wildfire records are naturally entity-centric — each incident is a self-contained event with its own attributes, and the data is semi-structured, with varying completeness across fields depending on jurisdiction and fire size.

> [!TIP]
> Background reading materials are available in the [Background Reading Folder](https://1drv.ms/f/c/9e42f755abca0340/IgAh0G0FX_4ISIzBAolqJb_oATZCmaogTj36Xhq7btXstMk?e=pB9Wp0)

| # | Title | Description | Link |
|---|---|---|---|
| 1 | *Spatial Database of Wildfires in the US 1992–2020* | The methodology paper behind your actual dataset — essential provenance reading | [Link](https://1drv.ms/b/c/9e42f755abca0340/IQDbE0Ndmkc8Ro0sAsCT2hh2AQO8ujHilsZkHd5DJgk9jDc?e=kTf8pM) |
| 2 | *Climate Warming Increases Extreme Daily Wildfire Growth Risk in California* (Nature, 2023) | Peer-reviewed study linking climate trends to fire behavior — gives scientific backbone | [Link](https://1drv.ms/b/c/9e42f755abca0340/IQAY8rkr7oNdR7Cc-1X8UrEFAYGjRp1hHuaD00m0BvdApk0?e=UGVBvS) |
| 3 | *Wildfire Risk to Communities* (USFS/wildfirerisk.org) | The federal framework for community-level risk scoring — key for framing KPIs | [Link](https://1drv.ms/b/c/9e42f755abca0340/IQCL8J3x4jjcTI3Pb046LqH3ARDcd889cmQRjVX_c2Sw2cI?e=Up2GOy) |
| 4 | *The Rising Cost of Wildfire Operations* (USFS Policy Report) | Explains the suppression cost crisis and why prediction matters financially | [Link](https://1drv.ms/b/c/9e42f755abca0340/IQA0dh7ThlQuTI4VMfvDMtpZAVO4grxpWonZaMKdtSj2pvw?e=GyOe7v) |
| 5 | *Modeling initial attack success of wildfire suppression in Catalonia, Spain* (ScienceDirect, 2023) | Academic ML approach to wildfire prediction — directly relevant to methodology | [Link](https://1drv.ms/b/c/9e42f755abca0340/IQDfcB7ty4uWQp8gnrJDt5DVAROmWCOzMgPbRY-3Pe_6ySg?e=OM9FsH) |

---

## Data Creation

**Raw Data Acquisition:**

The dataset used in this project is the Spatial Wildfire Occurrence Data for the United States, 1992–2020 (FPA_FOD_20221014, 6th Edition), published by Karen C. Short through the USDA Forest Service Research Data Archive in 2022. It is publicly available at no cost and was accessed directly via the [Forest Service Research Data Archive](https://www.fs.usda.gov/rds/archive/catalog/RDS-2013-0009.6). The data was downloaded as a GeoPackage (.gpkg) file and loaded into a Google Colab environment using GeoPandas, targeting the Fires layer which contains the full incident-level records.

The database contains 2.3 million geo-referenced wildfire records compiled from the reporting systems of federal, state, and local fire organizations across all 50 states. Records were required to include a discovery date, final fire size, and a point location precise to at least a one-square-mile PLSS section in order to be included. The data was standardized to conform to National Wildfire Coordinating Group (NWCG) conventions, including an updated cause classification standard approved in August 2020, and underwent basic error-checking and deduplication prior to publication. In total, the dataset spans 29 years and represents approximately 180 million acres burned.

**Code:**

| File | Description | Link |
|---|---|---|
| `data_creation.ipynb` | Downloads the raw FPA-FOD GeoPackage, loads the Fires layer, selects and slims columns, shapes each row into a nested MongoDB document, and inserts 300,000 records into MongoDB Atlas | [Data Creation Code](data_creation.ipynb) |

**Rationale:**

**Column selection:** The 15 columns retained were chosen to cover the four dimensions most relevant to the prediction task — ignition (cause classification, general cause), temporal context (fire year, discovery date, discovery day of year), outcome (fire size, size class, containment date, containment day of year), and spatial context (latitude, longitude, state, county, owner). Columns excluded include identifiers with no predictive value (FPA_ID, LOCAL_FIRE_REPORT_ID), join keys for external tables not used in this project (ICS_209_PLUS_INCIDENT_JOIN_ID, MTBS_ID), and fields with extremely high missingness that would introduce more noise than signal (DISCOVERY_TIME, CONT_TIME, FIRE_CODE). This is a judgment call: retaining more columns increases information but also increases sparsity and storage cost.

**Document structure:** Each fire incident is stored as a self-contained nested document with six sub-objects: `cause`, `discovery`, `containment`, `size`, and `location`. This structure was chosen because it mirrors the natural conceptual groupings of the data, makes queries against logical groups (e.g., all location fields) more readable, and satisfies the document model requirement of the assignment without artificially inflating nesting depth. Flat insertion of all 15 fields as top-level keys was rejected because it provides no structural benefit and makes the implicit schema harder to communicate.

**Record count:** The full 2.3 million record dataset exceeds the 512 MB free tier storage limit of MongoDB Atlas. Rather than upgrading to a paid tier or aggressively dropping additional columns, 300000 records were inserted — well above the 1,000-record threshold for full rubric credit. This is a deliberate trade-off that accepts reduced scale in exchange for a zero-cost deployment.

**Date handling:** `DISCOVERY_DATE` and `CONT_DATE` were cast to strings prior to insertion. The raw values from the GeoPackage are Julian date floats, which are not human-readable and do not serialize cleanly to BSON. Casting to string preserves the value without loss and avoids timezone conversion issues that arise when using Python datetime objects with PyMongo.

**County field:** The `COUNTY` field in the raw data stores FIPS numeric codes rather than county names. This is noted as a known limitation. FIPS codes are retained as-is because joining to a FIPS lookup table was judged out of scope for this deliverable, but this introduces interpretability risk for any model or visualization that treats county as a categorical label.

**Bias Identification:**

Several sources of bias are present in this dataset. Reporting bias is the most significant: smaller fires in rural or less-resourced jurisdictions are systematically underreported compared to fires on federal land, which has more robust incident tracking infrastructure. Temporal bias exists because reporting standards, cause classification codes, and data collection practices changed materially over the 29-year span, meaning fires from the early 1990s are not directly comparable to those from 2020. Geographic bias is introduced because the western United States, particularly California, Oregon, and Idaho, contributes disproportionately to the record count, both because those states have more fires and because their agencies have more mature reporting systems. Finally, cause classification bias exists because a significant portion of records carry unknown or unspecified causes, which could skew any model trained to predict cause category.

**Bias Mitigation:**

Reporting and geographic bias can be partially mitigated by including `STATE` and `OWNER_DESCR` as features in the model, allowing it to learn jurisdiction-specific patterns rather than treating all records as drawn from the same population. Temporal bias can be addressed by including `FIRE_YEAR` as a feature and, where appropriate, stratifying train/test splits by year rather than sampling randomly. For cause classification bias, records with null or unknown cause values can be excluded from the cause-prediction task specifically, or treated as a separate class, so that missingness does not propagate as a spurious signal. Uncertainty quantification in the data dictionary further documents where numerical features carry high measurement error so that downstream analysis can weight or discount those fields accordingly.

---

## Metadata

**Implicit Schema:**
Every document in the `ds4320.hw10` collection conforms to the following structure. All top-level fields are required. Within sub-objects, fields may be null where the source data did not record a value (most commonly `containment.date` and `containment.day_of_year` for fires with no recorded containment).

```json
{
  "fod_id":     <integer>,         // Unique record identifier — never null
  "fire_year":  <integer>,         // Calendar year of discovery — never null

  "cause": {
    "classification": <string>,    // Broad NWCG cause class — may be null
    "general_cause":  <string>     // Specific NWCG cause — may be null
  },

  "discovery": {
    "date":        <string>,       // Discovery date as string (from Julian float) — never null
    "day_of_year": <integer>       // Day of year (1–366) — never null
  },

  "containment": {
    "date":        <string>,       // Containment date as string — may be null
    "day_of_year": <float>         // Day of year — may be null
  },

  "size": {
    "acres":      <float>,         // Estimated acres burned — never null
    "size_class": <string>         // USFS size class A–G — never null
  },

  "location": {
    "latitude":  <float>,          // NAD83 decimal degrees — never null
    "longitude": <float>,          // NAD83 decimal degrees — never null
    "state":     <string>,         // Two-letter state code — never null
    "county":    <string>,         // FIPS county code (numeric string) — may be null
    "owner":     <string>          // Primary land owner description — may be null
  }
}
```

**Conventions:**
- String dates are sourced directly from the GeoPackage and are not guaranteed to follow a consistent format across all records.
- `county` stores FIPS codes, not county names.
- No additional top-level fields should be added without updating this schema document.

**Data Summary:**
| Table | Description | Link |
|---|---|---|
| `Fires` (FPA_FOD_20221014.gpkg) | Primary incident-level wildfire records, 2.3M rows, sourced from the USDA Forest Service Research Data Archive (6th Edition, 2022). Contains geographic, temporal, cause, and size attributes for each fire. | [https://www.fs.usda.gov/rds/archive/catalog/RDS-2013-0009.6](https://www.fs.usda.gov/rds/archive/catalog/RDS-2013-0009.6) |
| `_variable_descriptions.csv` | Metadata table describing every field in the Fires layer and the supplementary NWCG unit table. One row per variable. | [Link to metadata](_variable_descriptions.csv) |

**Data Dictionary:**
| Field (MongoDB path) | Data Type | Description | Example |
|---|---|---|---|
| `fod_id` | Integer | Unique numeric record identifier assigned by the FPA system | `1` |
| `fire_year` | Integer | Calendar year in which the fire was discovered or confirmed | `2005` |
| `cause.classification` | String | Broad NWCG classification of ignition cause | `"Human"` |
| `cause.general_cause` | String | Specific NWCG general cause category | `"Power generation/transmission/distribution"` |
| `discovery.date` | String | Date the fire was discovered or confirmed, cast from Julian float | `"2/2/2005"` |
| `discovery.day_of_year` | Integer | Day of year (1–366) on which the fire was discovered | `33` |
| `containment.date` | String | Date the fire was declared contained; null if not recorded | `"2/2/2005"` |
| `containment.day_of_year` | Float | Day of year on which containment was declared; null if not recorded | `33.0` |
| `size.acres` | Float | Estimated acres within the final fire perimeter | `0.1` |
| `size.size_class` | String | USFS size class code: A (<0.25 ac), B (0.26–9.9), C (10–99.9), D (100–299), E (300–999), F (1000–4999), G (5000+) | `"A"` |
| `location.latitude` | Float | NAD83 decimal degree latitude of the fire point of origin | `40.037` |
| `location.longitude` | Float | NAD83 decimal degree longitude of the fire point of origin | `-121.006` |
| `location.state` | String | Two-letter USPS state code for the state in which the fire burned | `"CA"` |
| `location.county` | String | FIPS numeric county code for the county in which the fire burned | `"63"` |
| `location.owner` | String | Name of the primary land owner or managing entity | `"USFS"` |

**Data Dict Quantificaiton:**
| Field | Range in Dataset | Source of Uncertainty | Quantification |
|---|---|---|---|
| `size.acres` | 0.0 – 662,702 | Fire perimeter estimates are manually mapped; smaller fires have lower mapping precision. Fires under 1 acre may have size recorded as nominal (e.g., exactly 0.1). | High uncertainty for Class A fires (<0.25 ac); low-to-moderate for Class D+ where perimeter mapping is more rigorous. Treat sub-1-acre values as approximate. |
| `location.latitude` | 17.9 – 71.3 | Point location is precise to the centroid of the PLSS section (approximately 1 square mile), not the actual ignition point. | Positional uncertainty of up to ~0.7 miles radially for all records. |
| `location.longitude` | -178.3 – -65.3 | Same PLSS centroid limitation as latitude. | Same ~0.7 mile radial uncertainty as latitude. |
| `discovery.day_of_year` | 1 – 366 | Derived from discovery date; uncertainty depends on date recording accuracy at the originating agency. Smaller agencies may record date as the report date rather than actual discovery date. | Low uncertainty for federal records; moderate uncertainty for local/state agency records where reporting lag is common. |
| `containment.day_of_year` | 1 – 366 (nullable) | Same agency-reporting uncertainty as discovery DOY; additionally, approximately 15–20% of records have null containment dates, introducing missingness that is not random (smaller fires and older records are more likely to be missing). | Null rate is a structural uncertainty. Present in analysis as a binary missingness indicator where relevant. |
| `fire_year` | 1992 – 2020 | Reporting completeness and cause classification standards changed over this period; earlier years are less complete. | Treat pre300000 records with higher skepticism for cause and size fields. Include `fire_year` as a covariate to allow the model to account for temporal drift. |
