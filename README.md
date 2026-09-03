# Toronto Urban Mobility Analysis

## Overview

This project uses City of Toronto open transportation data to identify opportunities for improving urban mobility infrastructure and traffic operations. Instead of attempting to optimize the entire transportation network, the project focuses on two practical questions:

1. **Cycling Infrastructure:** Which intersections experience high bicycle traffic but lack nearby cycling infrastructure and may deserve further review?
2. **Traffic Signal Operations:** Do observed traffic patterns appear consistent with the logic of each intersection's existing signal control mode, and which unusual intersections deserve engineering review?

The project combines multimodal traffic counts, Toronto's cycling network, traffic signal data, street centreline geometry, geospatial analysis, and time-of-day demand patterns.

The objective is not to prescribe infrastructure or signal changes directly. Instead, the analysis develops **screening frameworks that can help prioritize locations for more detailed engineering assessment.**


---

# Part I: Cycling Infrastructure Screening

[Full Python Workflow](data_source/cycling_lane_analysis.ipynb)

## Question

> Which high-bicycle-demand intersections currently lack nearby cycling infrastructure?

A high number of cyclists at an intersection does not by itself establish that a new bicycle lane should be constructed. Road geometry, safety history, network connectivity, available right-of-way, parking, construction costs, and other factors would also need to be considered.

However, bicycle traffic counts can provide a useful **demand-based screening mechanism** for identifying locations where existing cycling infrastructure may not align with observed usage.

## Data

The analysis combines:

* **Multimodal Intersection Turning Movement Counts** 
* **Toronto Cycling Network**
* **Intersection Coordinates and Geospatial Information**

The traffic-count dataset contains 15-minute observations of vehicle, pedestrian, and bicycle movements through Toronto intersections.

Because intersections were observed a different number of times, ranging from relatively small samples to hundreds of observations, raw total bicycle counts would disproportionately favor heavily sampled intersections.

I therefore aggregated observations to create a comparable measure of bicycle demand for each location.

## Methodology

### 1. Estimate bicycle demand

For each intersection, bicycle movements were aggregated across directions and observation periods to estimate average observed bicycle traffic per 15 minutes, as this is the timeframe for each observation.

Intersections were then ranked based on bicycle demand.

The analysis focuses on approximately the **top 20% of observed bicycle-demand intersections** as an initial high-demand screening group.
<p align="center">
<img width="635" height="132" alt="image" src="https://github.com/user-attachments/assets/9495e7a4-9a31-4aa0-858a-11cbc21307f1" />
<p/>
  
### 2. Map the cycling network

Intersection coordinates and Toronto cycling-network geometries were converted into GeoDataFrames.

Because latitude/longitude coordinates are unsuitable for calculating distances directly, the spatial data were projected from geographic coordinates into an appropriate projected coordinate reference system before distance-based analysis.

<p align="center">
  <img 
    width="800" 
    alt="high_bike_intersections" 
    src="https://github.com/user-attachments/assets/14ecc330-d767-491d-92f4-4e15489073c3"
  />
</p>

### 3. Identify infrastructure gaps

For each high-bicycle-demand intersection, the analysis measures proximity to existing cycling infrastructure.

A **400-metre screening radius** was used to distinguish intersections with nearby cycling infrastructure from locations where the surrounding network appears relatively underserved.

This threshold is a screening assumption rather than an engineering standard.

<p align="center"><img width="800" alt="high_bike_intersections" src="https://github.com/user-attachments/assets/f17fb3f8-4796-4ba7-acb5-bd4f2e281c8a" />
</p>



## Results

43 high-bicycle-demand intersections were identified for further infrastructure review.

The strongest candidates combine:

* high observed bicycle demand
* relatively large distance from existing cycling infrastructure
* limited obvious physical or operational constraints to implementation

### Example Candidates

| Intersection     | Average 15-min Bicycle Demand| Distance to Cycling Infrastructure |
| ---------------- | -------------: | ---------------------------------: |
| Spadina Ave / Dundas St W |            49 |                              486 m | 
| Queen St E / Broadview Ave |            23 |                              516 m | 
| Isabella St / Church St |            22 |                              419 m | 


These locations should be interpreted as **candidates for infrastructure review, not automatic recommendations for bicycle-lane construction**.

### Key Takeaways

* **Spadina Ave** shows as a potential cycling infrastructure gap. Spadina Ave / Dundas St W, along with several other Spadina intersections, records relatively high bicycle demand despite being approximately 500 m from the nearest mapped cycling infrastructure. However, the presence of streetcar infrastructure may hinder potential cycling improvements. Further feasibility assessment is recommended.

* **Queen St E / Broadview Ave** demonstrates a similar combination of high bicycle demand and limited nearby cycling infrastructure. Like Spadina, current streetcar operations may complicate the implementation of dedicated cycling facilities.

* **Isabella St / Church St** also observes high bicycle demand without nearby dedicated cycling infrastructure, but does not face the same streetcar constraint. This makes it a potentially stronger candidate for further assessment of bike-lane implementation.

* Among intersections in the **top 20% of bicycle demand**, locations along **Kingston Rd** and **St Clair Ave W** are among the furthest from existing cycling infrastructure. While bike-lane implementation is limited by street car infrastructure on St Clair Ave W, Kingston Rd does not share this constraint, making it worthy of further investigation.

Overall, the analysis identifies several high-demand intersections where observed bicycle use is not matched by nearby dedicated infrastructure. These findings provide a data-driven checklist for further safety, feasibility, and network-connectivity assessment rather than definitive recommendations for bike-lane construction.

# Part II: Traffic Signal Operations

[Full Python Workflow](data_source/signal_change.ipynb)

## Question

> Does the observed demand pattern at an intersection appear consistent with the logic of its existing signal control mode?



Toronto operates intersections under several control modes, including **Fixed Time (FT), Semi-Actuated Type 1 (SA1), Semi-Actuated Type 2 (SA2), and Semi-Actuated Pedestrian (SAP)**. **Semi-Actuated Vehicle (SAV) and Pedestrian Actuated (PED)** will be excluded from the scope of this study due to lack of joint pedestrian/vehicle representation.

### Toronto Traffic Signal Types

| Type | How it works | Vehicle Detection | Pedestrian Activation |
|------|--------------|------------------|----------------------|
| **FT** | Runs on a fixed cycle regardless of demand | ❌ | ❌ |
| **SAP** | Side street activates when either a vehicle or pedestrian is detected | ✅ | ✅ |
| **SA1** | Vehicle phases may be actuated, while pedestrian phases remain on recall | ✅ | ❌ |
| **SA2** | Side-street timing responds independently to vehicle and pedestrian demand | ✅ | ✅ |
| **SAV** | No pedestriann crossings, side street is activated and extended by vehicle demand only | - | - |
| **PED** | Midblock pedestrian signal, crossing is activated by a pedestrian pushbutton | - | - |

Different modes provide different levels of demand responsiveness.

This analysis asks whether observed multimodal demand appears broadly consistent with the operational logic of the existing mode. Hence, the resulting framework is therefore a **citywide screening tool**, and not a traffic-engineering optimization model.

---

## Data Preparation

The combination of several datasets that were not originally designed to work together are required for this analysis:

* **Multimodal Intersection Turning Movement Counts**
* **Traffic Signal Locations and Control Modes**
* **Intersection Coordinates**
* **Toronto Centreline Street Geometries**


Approximately **3,979 unique traffic-count intersections** were initially identified.

Traffic-count and signal datasets could not be matched perfectly because of differences in geographic coverage, coordinates, and street naming. Street names were cleaned and standardized where practical, while a small number of unresolved cases were excluded.

At intersections with two differently named side streets, both approaches were treated as part of the same side-street axis when they shared the same directional orientation.

### Determining Main- and Side-Street Direction

The traffic-count dataset reports movements geographically as north, south, east, and west, while signal operation at intersections depends on **main-street versus side-street demand**.

To bridge these interpretations, I developed a **geometric orientation procedure**:

1. Match each signal's side-street name to Toronto Centreline data.
2. Identify the centreline segment closest to the intersection.
3. Calculate whether that local segment primarily runs north-south or east-west.
4. Use that orientation to classify traffic-count movements as main- or side-street demand.
5. Merge the resulting orientation lookup back into the full traffic-count dataset.


Approximately 140 ambiguous or incorrectly classified cases were identified during validation by comparing directional traffic patterns and were corrected before the final analysis.

---

# Signal Screening Metrics

Four complementary metrics describe the observed demand pattern.

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


## 3. Vehicle-Pedestrian Temporal Correlation

Pearson correlation is calculated between side-street vehicle and pedestrian demand across observed time-of-day bins.

* **High positive correlation:** vehicle and pedestrian demand tend to increase and decrease together.
* **Near zero:** little linear temporal relationship.
* **Negative correlation:** vehicle and pedestrian demand tend to occur at different times.

This metric helps distinguish situations where vehicle and pedestrian calls may be reasonably coupled from situations where independent detection could potentially provide more operational value.

---

## 4. Main-Side Vehicle Temporal Correlation

Pearson correlation between main- and side-street vehicle demand, this measures whether both approaches follow similar time-of-day patterns.

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
| FT / SA1     | High K + strong vehicle/pedestrian synchronization      | SAP may deserve review                                                                          |
| FT / SA1     | High K + weak/negative vehicle/pedestrian relationship  | SA2 may deserve review                                                                          |
| SAP          | Low K                                                  | Side service may no longer behave as strongly secondary; FT/SA1 review                          |
| SAP          | High K + strong vehicle/pedestrian synchronization     | Observed demand is broadly consistent with SAP                                                  |
| SAP          | High K + weak/negative vehicle/pedestrian relationship | Independent calls under SA2 may deserve review                                                  |

These rules intentionally leave many intersections without a recommendation and identify **strong exceptions** without forcing every intersection into a different operating mode.

---

# Results

## Existing Operations Often Appear Reasonable

One of the most important findings is that **most intersections do not produce strong evidence for a control-mode review**.

Many SA2 intersections show clear main-street dominance, while many FT/SA1 intersections show relatively balanced demand patterns.

This is useful evidence in itself: the framework is capable of validating existing operations rather than simply searching for anomalies.

| Location | Main Street | Side Street | Control Mode | K | Persistence | Side Street Vehicle-Pedestrian Correlation | Main–Side Street Vehicle Correlation |
|---|---|---|:---:|---:|---:|---:|---:|
| Rusholme Rd / Lisgar St / Dundas St W | Dundas St W | Lisgar St | SA2 | 0.998 | 1.000  | -0.170 | 0.053 |
| Victoria Park Ave / McNicoll Ave | VICTORIA PARK AVE | MCNICOLL AVE | FT | 0.044 | 0.000 | 0.396 | 0.953 |
| Burnhamthorpe Rd / Kipling Ave | KIPLING AVE | BURNHAMTHORPE RD | SA1 | 0.008 | 0.000 | 0.422 | 0.862 |

---

## SA2 Review Candidates

Low-K SA2 intersections were treated as the primary review group because comparable main- and side-street demand weakens the observed demand-based rationale for highly responsive side-street service.

The initial screening identified **56 SA2 intersections with K ≤ 0.5**, with only **7 intersections below approximately K = 0.2**.

K < 0.2 was selected as an empirical screening threshold to identify SA2 intersections with unusually balanced main- and side-street demand. It acts as a conservative criterion for identifying intersections warranting further review. Supplementary time-of-day plots support this threshold visually, showing a noticeable decline in main-street dominance as K approaches 0.2

The graph below captures average main-side street vehicle demand of **Finch Ave E / Middlefield Rd** throughout the day. We confirm both numerically and visually that main street exhibits no dominance and mains-side vehicle demand shares high correlation, making it a strong candidate for recall signal mode review (**FT/SA1**).

<p align="center"><img width="800" alt="Finch_Middlefield" src="https://github.com/user-attachments/assets/edbf9fb2-4ef9-4eac-9b7b-9a3de5d50a00" /></p>

Other review candidates include:

* Wintermute Blvd / Bamburgh Crcl
* Finch Ave E / Middlefield Rd
* Lonsdale Rd / Avenue Rd
* Gardiner Expy E Park Lawn Rd Ramp / Mimico Creek Trl / Park Lawn Rd
* Kipling Ave / Belfield Rd
* Fairview Mall / Fairview Mall Dr / Hwy 404 S Fairview Mall Ramp / Hwy 404 S Ramp
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
* Eglinton Ave W / Hwy 27 / Hwy 401 / Hwy 427 Eglinton E Ramp
* Keele St / Canarctic Dr

| Location | Main Street | Side Street | Control Mode | K | Persistence | Side Street Vehicle-Pedestrian Correlation | Main–Side Street Vehicle Correlation |
|---|---|---|:---:|---:|---:|---:|---:|
| King St E / Jarvis St | King St E | Jarvis St | FT | 0.685 | 1.000  | -0.122 | -0.525 |
| Sorauren Ave / Dundas St W | Sorauren Ave | Dundas St W | SA1 | 0.753 | 1.000 | -0.087 | 0.435 |

In the example chart above, both intersections show strong and persistent main-street dominance, while side-street vehicle and pedestrian demand are negatively correlated. This suggests side-street vehicle and pedestrian demand may occur at different times, making them good candidates for SA2 review. However, the 15-minute count intervals are not fine-grained enough to confirm whether these movements overlap within individual signal cycles; cycle-level data would be required for a more definitive assessment.

Several additional high-K intersections showed strongly synchronized side-street vehicle and pedestrian patterns and were therefore identified as possible SAP review candidates.

---

## SAP Review Candidates

Most high-K SAP intersections showed demand relationships broadly consistent with their existing operation. Intersections such as **Pharmacy Ave / Nancy Ave** exhibited strong main-street dominance combined with synchronized side-street vehicle and pedestrian demand, providing an example where existing SAP operation appears consistent with the observed demand pattern.

<p align="center"><img width="800" alt="Pharmacy_Nancy" src="https://github.com/user-attachments/assets/077ee36c-1f8b-42b1-a412-d5df8aa93ea2" /></p>


One unusually low-K case was identified for FT/SA1 review:

* Gardiner Expy Express W Sherway Gardens Ramp / Sherway Gardens Rd

Several high-K intersections with weak or negative side-street vehicle-pedestrian temporal relationships were identified for SA2 review, including:

* Danforth Ave / Glebemount Ave
* College St / Borden St
* Pharmacy Ave / Gatineau Hydro Corridor Trl

---

# Limitations

## Cycling recommendations require additional feasibility analysis

High bicycle demand and limited nearby infrastructure do not establish that a bicycle lane is feasible or desirable.

Future analysis should incorporate road width, collision history, parking demand, network connectivity, land use, planned infrastructure, and construction constraints.

## Traffic counts are observed snapshots

Many intersections contain multiple 15-minute observations but only one or a small number of observation dates.

The analysis therefore describes:

> **observed within-day demand patterns**

rather than long-run or typical traffic behavior.

The data cannot establish whether these patterns persist across weekdays, seasons, weather conditions, construction periods, or longer-term changes in travel behavior.

## Signal operations depend on factors not captured here

Traffic counts alone cannot observe several important operational considerations, including:

* corridor coordination
* actual delay and queue lengths
* cycle lengths and signal splits
* detector performance
* skipped-phase frequency
* transit priority
* emergency-vehicle preemption
* collision history
* school or hospital operations
* accessibility requirements
* implementation and maintenance costs

For this reason, every output should only be interpreted as a **candidate for engineering review**, not a prescribed signal conversion.

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
Source: [City of Toronto Open Data](https://open.toronto.ca/catalogue/?topics=Transportation)

* [Multimodal Intersection Turning Movement Counts](data_source/tmc_raw_data_2020_2029.csv.zip)
* [Traffic Signal Tabular Data](data_source/Traffic_Signal_4326.csv)
* [Toronto Cycling Network](data_source/cycling-network-4326.csv)
* [Toronto Centreline](data_source/Centreline.csv.zip)


---

# Conclusion

This project demonstrates how public transportation data can be transformed from disconnected datasets into practical **decision-support screening tools**.

The cycling analysis identifies locations where high observed bicycle demand may not be matched by nearby cycling infrastructure.

The traffic-signal analysis evaluates whether observed main-street, side-street, vehicle, and pedestrian demand patterns appear broadly consistent with existing control modes.

Neither framework attempts to replace transportation engineering judgment.

Instead, they answer a more realistic data-analytics question:

> **Given thousands of locations and limited engineering resources, where should the City look at first?**
  [Full Signal Screening Results](results/signal_recommendations.csv)
