# Toronto Urban Mobility Analysis

## Overview

This project uses City of Toronto open transportation data to identify opportunities for improving urban mobility infrastructure and traffic operations. Instead of attempting to optimize the entire transportation network, the project focuses on two practical questions:

1. **Cycling Infrastructure:** Which intersections experience high bicycle traffic but lack nearby cycling infrastructure and may deserve further review?
2. **Traffic Signal Operations:** Do observed traffic patterns appear consistent with the logic of each intersection's existing signal control mode, and which unusual intersections deserve engineering review?

The project combines multimodal traffic counts, Toronto's cycling network, traffic signal data, street centreline geometry, geospatial analysis, and time-of-day demand patterns.

The objective is not to prescribe infrastructure or signal changes directly. Instead, the analysis develops **screening frameworks that can help prioritize locations for more detailed engineering assessment.**


---

# Part I: Cycling Infrastructure Screening

## Question

> Which high-bicycle-demand intersections currently lack nearby cycling infrastructure?

A high number of cyclists at an intersection does not by itself establish that a new bicycle lane should be constructed. Road geometry, safety history, network connectivity, available right-of-way, parking, construction costs, and other factors would also need to be considered.

However, bicycle traffic counts can provide a useful **demand-based screening mechanism** for identifying locations where existing cycling infrastructure may not align with observed usage.

## Data

The analysis combines:

* **Multimodal Intersection Turning Movement Counts** 
* **Toronto Cycling Network**
* Intersection coordinates and geospatial information

The traffic-count dataset contains 15-minute observations of vehicle, pedestrian, and bicycle movements through Toronto intersections.

Because intersections were observed a different number of times, ranging from relatively small samples to hundreds of observations, raw total bicycle counts would disproportionately favor heavily sampled intersections.

I therefore aggregated observations to create a comparable measure of bicycle demand for each location.

## Methodology

### 1. Estimate bicycle demand

For each intersection, bicycle movements were aggregated across directions and observation periods to estimate average observed bicycle traffic.

```python
intersection_d["bike_count"] = intersection_d["n_appr_bike"]+intersection_d["s_appr_bike"]+intersection_d["e_appr_bike"]+intersection_d["w_appr_bike"]

intersection_bike = (
    intersection_d
    .groupby("location_name")["bike_count"]
    .mean()
    .reset_index(name="avg_15min_bike")
)
```

Intersections were then ranked based on bicycle demand.

The analysis focuses on approximately the **top 20% of observed bicycle-demand intersections** as an initial high-demand screening group.

### 2. Map the cycling network

Intersection coordinates and Toronto cycling-network geometries were converted into GeoDataFrames.

Because latitude/longitude coordinates are unsuitable for calculating distances directly, the spatial data were projected from geographic coordinates into an appropriate projected coordinate reference system before distance-based analysis.

<img width="600" alt="high_bike_intersections" src="https://github.com/user-attachments/assets/14ecc330-d767-491d-92f4-4e15489073c3" />

### 3. Identify infrastructure gaps

For each high-bicycle-demand intersection, the analysis measures proximity to existing cycling infrastructure.

A **400-metre screening radius** was used to distinguish intersections with nearby cycling infrastructure from locations where the surrounding network appears relatively underserved.

This threshold is a screening assumption rather than an engineering standard.

[INSERT MAP OF HIGH-BIKE INTERSECTIONS WITHOUT NEARBY INFRASTRUCTURE]

## Results

[INSERT NUMBER] high-bicycle-demand intersections were identified for further infrastructure review.

The strongest candidates combine:

* high observed bicycle demand;
* relatively large distance from existing cycling infrastructure; and
* potential gaps in surrounding network connectivity.

### Example Candidates

| Intersection     | Bicycle Demand | Distance to Cycling Infrastructure | Interpretation   |
| ---------------- | -------------: | ---------------------------------: | ---------------- |
| [Intersection 1] |            [X] |                              [X m] | [Interpretation] |
| [Intersection 2] |            [X] |                              [X m] | [Interpretation] |
| [Intersection 3] |            [X] |                              [X m] | [Interpretation] |

[INSERT 1–2 DETAILED EXAMPLE MAPS]

These locations should be interpreted as **candidates for infrastructure review, not automatic recommendations for bicycle-lane construction**.

---

# Part II: Traffic Signal Operations

## Question

> Does the observed demand pattern at an intersection appear consistent with the logic of its existing signal control mode?

Toronto operates intersections under several control modes, including **Fixed Time (FT), Semi-Actuated Type 1 (SA1), Semi-Actuated Type 2 (SA2), and Semi-Actuated Pedestrian (SAP)**.

Different modes provide different levels of demand responsiveness.

Instead of attempting to calculate an "optimal" control mode, this analysis asks whether observed multimodal demand appears broadly consistent with the operational logic of the existing mode.

The resulting framework is therefore a **citywide screening tool**, not a traffic-engineering optimization model.

---

## Data Preparation

This analysis required combining several datasets that were not originally designed to work together:

* multimodal intersection turning-movement counts;
* traffic signal locations and control modes;
* intersection coordinates; and
* Toronto Centreline street geometries.

Approximately **3,979 unique traffic-count intersections** were initially identified.

Traffic-count and signal datasets could not be matched perfectly because of differences in geographic coverage, coordinates, and street naming. Street names were cleaned and standardized where practical, while a small number of unresolved cases were excluded rather than forcing uncertain matches.

### Determining Main- and Side-Street Direction

The traffic-count dataset reports movements geographically as north, south, east, and west, while signal operation depends on **main-street versus side-street demand**.

To bridge these representations, I developed a geometric orientation procedure:

1. Match each signal's side-street name to Toronto Centreline data.
2. Identify the centreline segment closest to the intersection.
3. Calculate whether that local segment primarily runs north-south or east-west.
4. Use that orientation to classify traffic-count movements as main- or side-street demand.
5. Merge the resulting orientation lookup back into the full traffic-count dataset.

[INSERT ORIENTATION DIAGRAM / MAP]

Approximately 140 ambiguous or incorrectly classified cases were identified during validation by comparing directional traffic patterns and were corrected before the final analysis.

---

# Signal Screening Metrics

Five complementary metrics describe the observed demand pattern.

## 1. Main-Street Dominance Index (K)

The primary metric is:

```text
K = mean_t[(Main_t - Side_t) / (Main_t + Side_t)]
```

where `Main_t` and `Side_t` represent observed main- and side-street vehicle demand during time interval `t`.

Interpretation:

|               K | Approximate Demand Relationship               |
| --------------: | --------------------------------------------- |
|               0 | Main ≈ Side                                   |
|             0.2 | ≈ 1.5:1                                       |
|             0.5 | ≈ 3:1                                         |
| High positive K | Strong main-street dominance                  |
|      Negative K | Side-street demand exceeds main-street demand |

K measures the **magnitude of directional imbalance**.

Rather than treating arbitrary percentiles as engineering thresholds, the analysis uses interpretable regions such as `K ≈ 0.2` and `K ≈ 0.5` to screen unusual demand relationships.

---

## 2. Dominance Persistence

K describes average imbalance, but not whether that imbalance persists throughout the observed period.

An interval is classified as meaningfully main-street dominant when:

```text
Main Vehicle Demand >= 1.5 × Side Vehicle Demand
```

Dominance persistence is the proportion of observed time bins satisfying this condition.

Therefore:

**K = magnitude of imbalance**

**Persistence = consistency of that imbalance**

---

## 3. Side-Street Vehicle Variability

Side-street demand variability is measured using the coefficient of variation:

```text
CV = Standard Deviation of Side Demand / Mean Side Demand
```

Higher CV indicates greater variation in side-street traffic relative to its average level.

This metric provides supporting evidence when considering whether demand-responsive control could exploit variations in side-street demand. It is not treated as a standalone conversion criterion.

---

## 4. Vehicle-Pedestrian Temporal Correlation

Pearson correlation is calculated between side-street vehicle and pedestrian demand across observed time-of-day bins.

* **High positive correlation:** vehicle and pedestrian demand tend to increase and decrease together.
* **Near zero:** little linear temporal relationship.
* **Negative correlation:** vehicle and pedestrian demand tend to occur at different times.

This metric helps distinguish situations where vehicle and pedestrian calls may reasonably remain coupled from situations where independent detection could potentially provide operational value.

---

## 5. Main-Side Vehicle Temporal Correlation

Correlation between main- and side-street vehicle demand measures whether both approaches follow similar time-of-day patterns.

For example:

> Low K + high main-side correlation

suggests not only comparable traffic volumes but also similar temporal demand patterns, providing additional evidence that regular service such as FT or SA1 may be reasonable.

---

# Signal-Control Screening Logic

The metrics are interpreted together rather than as a mechanical classification algorithm.

| Current Mode | Observed Pattern                                       | Screening Interpretation                                                                        |
| ------------ | ------------------------------------------------------ | ----------------------------------------------------------------------------------------------- |
| SA2          | Low K                                                  | Main and side demand are comparable; review whether SA2 provides meaningful incremental benefit |
| SA2          | High K                                                 | Strong main-street dominance is broadly consistent with SA2                                     |
| FT / SA1     | Low K                                                  | Comparable demand supports regular service                                                      |
| FT / SA1     | High K + persistent dominance                          | Actuation may deserve review                                                                    |
| FT / SA1     | Above + strong vehicle/pedestrian synchronization      | SAP may deserve review                                                                          |
| FT / SA1     | Above + weak/negative vehicle/pedestrian relationship  | SA2 may deserve review                                                                          |
| SAP          | Low K                                                  | Side service may no longer behave as strongly secondary; FT/SA1 review                          |
| SAP          | High K + strong vehicle/pedestrian synchronization     | Observed demand is broadly consistent with SAP                                                  |
| SAP          | High K + weak/negative vehicle/pedestrian relationship | Independent calls under SA2 may deserve review                                                  |

These rules intentionally leave many intersections without a recommendation.

The purpose is to identify **strong exceptions**, not force every intersection into a different operating mode.

---

# Results

## Existing Operations Often Appear Reasonable

One of the most important findings is that **most intersections do not produce strong evidence for a control-mode review**.

Many SA2 intersections show clear main-street dominance, while many FT/SA1 intersections show relatively balanced demand patterns.

This is useful evidence in itself: the framework is capable of validating existing operations rather than simply searching for anomalies.

[INSERT EXAMPLE: FT/SA1 THAT CLEARLY MAKES SENSE]

[INSERT EXAMPLE: SA2 THAT CLEARLY MAKES SENSE]

---

## SA2 Review Candidates

Low-K SA2 intersections were treated as the primary review group because comparable main- and side-street demand weakens the observed demand-based rationale for highly responsive side-street service.

The initial screening identified **56 SA2 intersections with K ≤ [THRESHOLD]**, with only **7 intersections below approximately K = 0.2**.

[INSERT SA2 K DISTRIBUTION]

[INSERT GOOD SA2 GRAPH]

[INSERT LOW-K SA2 REVIEW CANDIDATE GRAPH]

Example review candidates include:

* Wintermute Blvd / Bamburgh Crcl
* Finch Ave E / Middlefield Rd
* Lonsdale Rd / Avenue Rd
* Gardiner Expy E Park Lawn Rd Ramp / Mimico Cr...
* Kipling Ave / Belfield Rd
* Fairview Mall / Fairview Mall Dr / Hwy 404...
* DVP S Wynford Dr Ramp / Wynford Dr

These results require particular caution because several candidates contain traffic counts from only one to three observation dates.

---

## FT / SA1 Review Candidates

Low-K FT and SA1 intersections generally provide reassuring evidence for regular service.

For high-K intersections, additional metrics were used to distinguish cases where actuation might deserve investigation.

Examples identified for SA2 review include:

**FT**

* King St E / Jarvis St
* Lake Shore Blvd W / Long Branch Ave

**SA1**

* Sorauren Ave / Dundas St W
* Eglinton Ave W / Hwy 27 / Hwy 401 / Hwy 427...
* Keele St / Canarctic Dr

[INSERT REPRESENTATIVE GRAPH]

Several additional high-K intersections showed strongly synchronized side-street vehicle and pedestrian patterns and were therefore identified as possible SAP review candidates.

---

## SAP Review Candidates

Most high-K SAP intersections showed demand relationships broadly consistent with their existing operation.

One unusually low-K case was identified for FT/SA1 review:

* Gardiner Expy Express W Sherway Gardens Ramp / [...]

Several high-K intersections with weak or negative side-street vehicle-pedestrian temporal relationships were identified for SA2 review, including:

* Danforth Ave / Glebemount Ave
* College St / Borden St
* Pharmacy Ave / Gatineau Hydro Corridor Trl

Conversely, intersections such as **Pharmacy Ave / Nancy Ave** exhibited strong main-street dominance combined with synchronized side-street vehicle and pedestrian demand, providing an example where existing SAP operation appears consistent with the observed demand pattern.

---

# Key Findings

The two analyses demonstrate how Toronto's existing open data can support **citywide screening before expensive site-level investigation**.

### Cycling Infrastructure

High bicycle demand can be combined with cycling-network proximity to identify locations where observed cycling activity and existing infrastructure appear misaligned.

### Traffic Signal Operations

A relatively small set of interpretable demand metrics can identify intersections whose observed time-of-day patterns appear unusual relative to the logic of their current control mode.

Importantly, the analysis does **not** suggest widespread infrastructure or signal changes.

Most analyzed intersections appear broadly plausible under their existing configuration. The strongest value of the framework is therefore its ability to narrow thousands of locations into a much smaller set deserving detailed review.

---

# Limitations

## Traffic counts are observed snapshots

Many intersections contain multiple 15-minute observations but only one or a small number of observation dates.

The analysis therefore describes:

> **observed within-day demand patterns**

rather than long-run or typical traffic behavior.

The data cannot establish whether these patterns persist across weekdays, seasons, weather conditions, construction periods, or longer-term changes in travel behavior.

## Signal operations depend on factors not captured here

Traffic counts alone cannot observe several important operational considerations, including:

* corridor coordination;
* actual delay and queue lengths;
* cycle lengths and signal splits;
* detector performance;
* skipped-phase frequency;
* transit priority;
* emergency-vehicle preemption;
* collision history;
* school or hospital operations;
* accessibility requirements; and
* implementation and maintenance costs.

For this reason, every output should be interpreted as a **candidate for engineering review**, not a prescribed signal conversion.

## Cycling recommendations require additional feasibility analysis

High bicycle demand and limited nearby infrastructure do not establish that a bicycle lane is feasible or desirable.

Future analysis should incorporate road width, collision history, parking demand, network connectivity, land use, planned infrastructure, and construction constraints.

---

# Potential KPIs for the City

The frameworks could be updated as new traffic counts become available.

Potential monitoring KPIs include:

| KPI                                                                  | Purpose                                        |
| -------------------------------------------------------------------- | ---------------------------------------------- |
| High-bike-demand intersections without nearby cycling infrastructure | Track potential cycling-network gaps           |
| Main-Street Dominance Index (K)                                      | Track directional traffic imbalance            |
| Dominance Persistence                                                | Measure consistency of directional imbalance   |
| Side-Street Vehicle CV                                               | Track variation in side-street demand          |
| Vehicle-Pedestrian Temporal Correlation                              | Measure synchronization of side-service demand |
| Main-Side Vehicle Correlation                                        | Measure similarity of approach demand patterns |
| Number of intersections flagged for review                           | Prioritize engineering investigation           |

These indicators should be treated as **screening KPIs rather than engineering design standards**.

---

# Tools

**Python**

* pandas
* NumPy
* GeoPandas
* Matplotlib

**Geospatial Analysis**

* Coordinate reference system transformation
* Spatial joins
* Distance/buffer analysis
* Toronto Centreline geometry
* Local street-orientation inference

**Data Sources**

* City of Toronto Open Data
* Multimodal Intersection Turning Movement Counts
* Traffic Signal Tabular Data
* Toronto Cycling Network
* Toronto Centreline

[ADD EXACT DATASET LINKS]

---

# Project Structure

```text
Toronto-Urban-Mobility/
│
├── data/
│   └── [describe raw/processed data availability]
│
├── notebooks/
│   ├── 01_cycling_infrastructure.ipynb
│   └── 02_signal_optimization.ipynb
│
├── figures/
│   ├── cycling/
│   └── signals/
│
├── src/
│   └── [optional reusable functions]
│
└── README.md
```

---

# Conclusion

This project demonstrates how public transportation data can be transformed from disconnected datasets into practical **decision-support screening tools**.

The cycling analysis identifies locations where high observed bicycle demand may not be matched by nearby cycling infrastructure.

The traffic-signal analysis evaluates whether observed main-street, side-street, vehicle, and pedestrian demand patterns appear broadly consistent with existing control modes.

Neither framework attempts to replace transportation engineering judgment.

Instead, they answer a more realistic data-analytics question:

> **Given thousands of locations and limited engineering resources, where should the City look first?**
