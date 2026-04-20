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

**Bias Identification:**

**Bias Mitigation:**







