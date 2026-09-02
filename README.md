# Toronto Urban Mobility Analysis

## Overview

This project uses City of Toronto open data to answer two questions:

1. Which intersections have high observed bicycle traffic but no cycling infrastructure nearby?
2. Do observed vehicle and pedestrian demand patterns appear consistent with each intersection's existing traffic-signal control mode?

I built two screening tools rather than attempting to prescribe infrastructure or signal changes. The outputs identify locations that may deserve closer engineering review.


## Project at a glance

| Analysis | Starting point | Screening output |
|---|---:|---:|
| Cycling infrastructure | 3,979 traffic-count intersections | 43 high-bike-demand intersections without cycling infrastructure within 400 m |
| Signal operations | Matched traffic counts, signal records, and street geometry | A small set of unusual FT, SA1, SA2, and SAP intersections for engineering review |

> The results are screening candidates, not recommendations to construct a bike lane or convert a signal.

---

## Part 1: Cycling infrastructure screening

### Question

> Where does observed bicycle demand appear high relative to the surrounding cycling infrastructure?

### Data and demand measure

The Multimodal Intersection Turning Movement Counts dataset records bicycle, pedestrian, and vehicle movements in 15-minute intervals. All 335,566 cleaned observations used here represent 15-minute count intervals, but the number of observations per intersection varies substantially.

To avoid favoring intersections that were counted more often, I calculated total bicycle approaches in each interval and then took the mean for each location:

$$
\text{Average 15-minute bicycle demand}_i
=
\operatorname{mean}_t(N_{it}+S_{it}+E_{it}+W_{it})
$$

Across 3,979 intersections, the distribution was strongly right-skewed. The 80th-percentile value was **6.78125 bicycles per 15-minute interval**, which I used to define the initial high-demand group.

This percentile is a practical screening choice, not an engineering standard.

### Geospatial screening

I converted the intersection coordinates and Toronto Cycling Network into GeoDataFrames. The layers were projected from EPSG:4326 to EPSG:3857 before distance calculations so that buffers and nearest-network distances could be measured in metres.

For each high-demand intersection, I then:

1. created a 400 m buffer;
2. tested whether any cycling-network geometry intersected that buffer; and
3. calculated the distance to the nearest cycling-network segment for uncovered locations.

The **400 m radius is a screening assumption**, not a claim that every intersection beyond this distance requires new infrastructure.

[Insert citywide map: high-bike intersections and cycling network]

### Results

The screen identified **43 high-bike-demand intersections without cycling infrastructure within 400 m**.

The most useful priority view combines two dimensions:

- marker size: average observed bicycle demand;
- marker colour: distance to the nearest cycling-network segment.

[Insert map: High-Bike Intersections by Demand and Distance to Nearest Bike Lane]

Examples from the high-demand group include:

| Intersection | Avg. bicycles per 15 min | Nearest cycling infrastructure |
|---|---:|---:|
| Spadina Ave / Dundas St W | 48.98 | 485.59 m |
| Spadina Ave / Baldwin St (North) | 37.41 | 442.60 m |
| Spadina Ave / St Andrew St | 36.59 | 486.53 m |
| Queen St W / Lisgar St | 23.25 | 407.81 m |
| Queen St E / Broadview Ave | 23.10 | 516.12 m |

Distance reveals a different set of potential network gaps. For example, Kingston Rd / Scarborough Rd was approximately **937.82 m** from the nearest mapped cycling infrastructure, although its observed bicycle demand was much lower than the Spadina candidates.

This is why the framework retains both demand and distance instead of collapsing them into a single opaque score.

### Interpretation

High bicycle demand and limited nearby infrastructure can help the City decide where to look first. They cannot establish whether a bike lane is feasible or desirable. A site-level review would also need to consider road width, network continuity, parking and loading, collision history, planned projects, land use, and construction constraints.

Full implementation: [cycling infrastructure notebook](notebooks/01_cycling_infrastructure.ipynb)

---

## Part 2: Traffic-signal operations screening

### Question

> Does the observed time-of-day demand profile appear consistent with the logic of the intersection's existing signal-control mode?

Toronto uses several control modes, including Fixed Time (FT), Semi-Actuated Type 1 (SA1), Semi-Actuated Type 2 (SA2), and Semi-Actuated Pedestrian (SAP). These modes provide different forms of regular, vehicle-actuated, and pedestrian-actuated service.

This analysis does **not** calculate an optimal signal mode. It screens for unusual demand patterns that may justify closer operational review.

### Data preparation

The signal analysis combines:

- multimodal turning-movement counts;
- traffic-signal locations and control modes;
- intersection coordinates; and
- Toronto Centreline street geometry.

These datasets were not designed to join cleanly. I standardized street names, matched signal side-street names to centreline records, and excluded unresolved cases rather than forcing uncertain matches. After cleaning, only about 3% of the street-name matches remained unresolved.

### Inferring main- and side-street demand

Traffic counts are reported by compass direction, while the signal data identify main and side streets by name. I therefore developed a geometry-based orientation procedure:

1. match the named side street to Toronto Centreline;
2. identify the centreline segment closest to the intersection;
3. classify the local segment as mainly north-south or east-west; and
4. use that orientation to aggregate directional counts into main- and side-street demand.

Validation against north-south and east-west traffic volumes exposed approximately 140 ambiguous or incorrect classifications, which were corrected before the final analysis.

[Insert diagram or map: side-street orientation procedure]

Full implementation: [signal operations notebook](notebooks/02_signal_operations.ipynb)

---

## Signal screening metrics

The metrics are interpreted together. They are not a mechanical classifier.

### Main-Street Dominance Index (K)

For each intersection, I first calculated directional imbalance in each observed time-of-day bin and then averaged across bins:

$$
K=operatorname{mean}_t\left(\frac{Main_t-Side_t}{Main_t+Side_t}\right)
$$

| K value | Approximate interpretation |
|---:|---|
| 0 | Main and side demand are comparable |
| 0.2 | Approximately 1.5:1 main-to-side demand |
| 0.5 | Approximately 3:1 main-to-side demand |
| High positive K | Strong main-street dominance |
| Negative K | Side-street demand exceeds main-street demand |

K is the backbone of the framework because it has an interpretable relationship to actual directional demand. Values such as 0.2 and 0.5 define useful screening regions; they are not presented as scientifically optimal conversion thresholds.

### Dominance persistence

An interval is treated as meaningfully main-street dominant when:

$$
Main_t \geq 1.5 \times Side_t
$$

Dominance persistence is the fraction of observed time bins satisfying this condition.

In short:

- **K** measures the magnitude of overall imbalance;
- **persistence** measures how consistently meaningful dominance occurs during the observed period.

### Side-street vehicle variability

Side-street variability is measured with the coefficient of variation:

$$
CV_{side}=\frac{\sigma_{side}}{\mu_{side}}
$$

A higher CV indicates greater within-day variation relative to average side-street demand. It is supporting evidence for whether actuation could potentially respond to variable demand; it is not a standalone conversion rule.

### Vehicle-pedestrian temporal correlation

Pearson correlation between side-street vehicle and pedestrian demand describes whether the two forms of side service rise and fall together across the observed time profile.

- high positive correlation: vehicle and pedestrian demand move together;
- near zero: weak linear co-movement;
- negative correlation: the two demands tend to occur at different times.

This is a **temporal correlation**, not proof of operational overlap or causality.

### Main-side vehicle temporal correlation

Correlation between main- and side-street vehicle demand measures whether the two approaches follow similar time-of-day patterns. Low K combined with high main-side correlation provides additional evidence that regular service such as FT or SA1 may be reasonable.

---

## Screening logic by current mode

| Current mode | Observed pattern | Screening interpretation |
|---|---|---|
| SA2 | Low K and little dominance persistence | Main and side demand are comparable; review whether SA2 provides meaningful incremental benefit |
| SA2 | High K | Strong main-street dominance is broadly consistent with SA2 |
| FT / SA1 | Low K | Comparable approach demand supports regular service |
| FT / SA1 | High, persistent main dominance | Actuation may deserve investigation |
| FT / SA1 | Above + strong vehicle-pedestrian synchronization | SAP may deserve review if calls can remain coupled |
| FT / SA1 | Above + weak or negative vehicle-pedestrian correlation | SA2 may deserve review if independent calls could add value |
| SAP | Low K | Side demand may no longer behave as strongly secondary; FT/SA1 review |
| SAP | High K + strong vehicle-pedestrian synchronization | Observed demand is broadly consistent with SAP |
| SAP | High K + weak or negative vehicle-pedestrian correlation | Independent calls under SA2 may deserve review |

The framework deliberately leaves the middle ambiguous. Its purpose is to identify strong exceptions, not force every intersection into a new mode.

---

## Signal results

### Most existing operations appeared plausible

The most important result was not a large list of proposed changes. Most analyzed intersections did not produce strong, coherent evidence for review.

Many SA2 intersections showed clear main-street dominance, while many FT and SA1 intersections showed balanced volumes or continuously present side-street demand. Showing these cases is important because it demonstrates that the framework can validate existing operation rather than merely hunt for problems.

[Insert example graph: FT/SA1 operation that appears reasonable]

[Insert example graph: SA2 operation that appears reasonable]

### SA2 review group

An initial screen identified **56 SA2 intersections with K below 0.5**. Visual review showed that the separation between main- and side-street demand generally increased with K, which supported using K as the primary citywide screening measure.

Only **7 SA2 intersections had K below 0.2**. These became the strongest FT/SA1 review group because their main- and side-street demand was comparatively balanced:

- Wintermute Blvd / Bamburgh Crcl
- Finch Ave E / Middlefield Rd
- Lonsdale Rd / Avenue Rd
- Gardiner Expy E Park Lawn Rd Ramp / Mimico Cr...
- Kipling Ave / Belfield Rd
- Fairview Mall / Fairview Mall Dr / Hwy 404 S F...
- DVP S Wynford Dr Ramp / Wynford Dr

These cases require caution: the available counts cover only one to three observation dates at several locations.

[Insert SA2 K distribution]

[Insert low-K SA2 candidate graph]

### FT and SA1 review candidates

Low-K FT and SA1 intersections generally supported regular service. High K alone did not justify conversion because persistent side demand, corridor coordination, and implementation costs can make existing operation reasonable.

The strongest SA2 review candidates combined persistent main-street dominance with weak or negative side-street vehicle-pedestrian temporal correlation.

**FT examples**

- King St E / Jarvis St
- Lake Shore Blvd W / Long Branch Ave

**SA1 examples**

- Sorauren Ave / Dundas St W
- Eglinton Ave W / Hwy 27 / Hwy 401 / Hwy 427...
- Keele St / Canarctic Dr

[Insert representative FT or SA1 candidate graph]

### SAP review candidates

Pharmacy Ave / Nancy Ave combined strong main-street dominance with strongly synchronized side-street vehicle and pedestrian demand, providing an example where existing SAP operation appeared consistent with the observed profile.

The following high-K SAP intersections had weak or negative vehicle-pedestrian relationships and were identified for possible SA2 review:

- Danforth Ave / Glebemount Ave
- College St / Borden St
- Pharmacy Ave / Gatineau Hydro Corridor Trl

One unusually low-K SAP case—Gardiner Expy Express W Sherway Gardens Ramp / ...—was identified for FT/SA1 review.

[Insert SAP-validating example and one SAP review candidate]

---

## Limitations

### Traffic counts are snapshots

Many intersections have multiple intraday bins but only one or a small number of count dates. The metrics therefore describe the **observed within-day demand pattern**, not typical long-run traffic behaviour.

The analysis cannot establish whether the patterns persist across weekdays, seasons, weather conditions, construction periods, or long-term changes in travel behaviour.

### Counts do not describe the full signal system

The available data do not directly observe:

- corridor coordination;
- delay, queues, or level of service;
- cycle lengths and splits;
- detector condition or skipped-phase frequency;
- transit priority or emergency-vehicle preemption;
- collision history and safety constraints;
- school, hospital, or accessibility requirements; or
- implementation and maintenance costs.

These omissions are why every flagged location is described as a **candidate for engineering review**, not a prescribed conversion.

### Cycling gaps require feasibility review

The cycling screen measures observed demand and proximity to mapped infrastructure. It does not evaluate right-of-way, network design, planned projects, parking and loading needs, safety history, or construction feasibility.

---

## Potential monitoring KPIs

| KPI | Purpose |
|---|---|
| High-bike-demand intersections without nearby cycling infrastructure | Monitor possible cycling-network gaps |
| Average observed 15-minute bicycle demand | Compare bicycle activity across unevenly sampled intersections |
| Distance to nearest cycling infrastructure | Describe the scale of a potential network gap |
| Main-Street Dominance Index (K) | Measure directional vehicle imbalance |
| Dominance persistence | Measure how consistently meaningful dominance occurs |
| Side-street vehicle CV | Describe relative within-day variability |
| Vehicle-pedestrian temporal correlation | Describe synchronization of side-service demand |
| Main-side vehicle temporal correlation | Describe similarity between approach demand profiles |
| Intersections flagged for review | Prioritize detailed engineering investigation |

These are screening KPIs, not engineering design standards.

---

## Tools and data

**Python:** pandas, NumPy, GeoPandas, Matplotlib

**Methods:** data cleaning, record linkage, spatial joins, buffers, nearest-neighbour distance, local street-orientation inference, time-of-day aggregation, coefficients of variation, and temporal correlations

**City of Toronto datasets:**

- Multimodal Intersection Turning Movement Counts
- Cycling Network
- Traffic Signal Tabular Data
- Toronto Centreline

[Add exact dataset links]

---

## Repository structure

```text
Toronto-Urban-Mobility/
├── README.md
├── notebooks/
│   ├── 01_cycling_infrastructure.ipynb
│   └── 02_signal_operations.ipynb
├── figures/
│   ├── cycling/
│   └── signals/
└── data/
    └── README.md
```

Large raw datasets can be excluded from the repository and documented in `data/README.md` with download links and expected filenames.

## Conclusion

The two analyses answer the same practical question at different levels of Toronto's transportation system:

> Given thousands of locations and limited engineering resources, where should the City look first?

The cycling analysis narrows the network to intersections where high observed bicycle demand is not matched by nearby mapped infrastructure. The signal analysis identifies intersections whose observed vehicle and pedestrian demand patterns appear unusual relative to their current operating mode.

Neither tool replaces engineering judgment. Their value is in turning disconnected public datasets into transparent, interpretable shortlists for more detailed review.

