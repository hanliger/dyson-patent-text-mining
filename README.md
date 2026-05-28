# Dyson Patent Text Mining

Dyson 특허 데이터(`발명의 명칭` + `요약`)에 대한 텍스트 마이닝과
LLM 기반 제품공학 요소 추출 파이프라인.

## Pipeline

1. xlsx 데이터 로드 (`data/TextDown_dyson_patent_20260526.xlsx`)
2. 텍스트 코퍼스 구성 (title + abstract)
3. 전처리 (소문자, 구두점/숫자 제거, 일반 특허 boilerplate stopword만 제거)
4. TF-IDF 기반 keyword 추출 (1–3 gram, 전체 + title only)
5. WordCloud (전체 / title only / cluster별)
6. KMeans clustering (k=3..25 sweep, k=12 pinned — silhouette은 단조 증가하므로 argmax는 noisy)
7. Cluster summary (distinctive term + representative title + example abstract)
8. 혼합 cluster(10, 11)에 대한 sub-clustering — 해당 cluster 문서만으로 TF-IDF 재학습 + KMeans `sub_k=2..8` sweep
9. **Optional** LLM 기반 cluster label 제안
10. **Optional** LLM 기반 제품공학 요소 추출
    - module candidates
    - functional elements (동사구)
    - physical elements (명사구)
    - function ↔ physical ↔ module mappings
    - notes (`uncertain_items`, `insufficient_evidence`)
11. 위 LLM 출력(JSONL)을 cluster별 평탄화 CSV 5종으로 변환 (Excel/Sheets에서 바로 필터링)

LLM 호출은 `ANTHROPIC_API_KEY` 가 설정된 경우에만 동작한다.
key 없이도 단계 1–8은 그대로 끝까지 실행된다. 단계 11은 LLM 출력이 있을 때만 실행된다.

## Repo 구조

```
.
├── dyson_patent_text_mining.ipynb        # 분석 노트북 (위에서 아래로 실행)
├── data/
│   └── TextDown_dyson_patent_20260526.xlsx
├── prompt/
│   ├── llm_cluster_label_prompt.txt      # cluster label 제안 prompt
│   └── llm_product_engineering_prompt.txt# module/function/physical 추출 prompt
└── outputs/                              # 노트북 실행 결과
    ├── corpus_top_keywords.csv
    ├── title_top_keywords.csv
    ├── document_top_keywords.csv
    ├── clustering_eval.csv
    ├── patent_clusters.csv
    ├── cluster_summary.csv
    ├── sub_clustering_eval.csv               # cluster 10, 11 sub-sweep
    ├── sub_cluster_summary.csv               # sub-cluster별 top term / 대표 title
    ├── patent_subclusters.csv                # cluster 10, 11 문서의 sub-cluster 매핑
    ├── llm_cluster_summary_input.jsonl
    ├── llm_cluster_labels.jsonl
    ├── llm_product_engineering.jsonl
    ├── cluster_product_engineering_overview.csv
    ├── cluster_modules.csv
    ├── cluster_functional_elements.csv
    ├── cluster_physical_elements.csv
    └── cluster_function_physical_mappings.csv
```

## 실행

```bash
# 의존성
pip install pandas numpy scikit-learn matplotlib openpyxl wordcloud tqdm \
            jupyter nbformat python-dotenv anthropic

# (선택) LLM 호출을 위해 .env 에 API key
echo "ANTHROPIC_API_KEY=..." > .env

# 노트북 실행
jupyter notebook dyson_patent_text_mining.ipynb
```

## 결과 요약 (현 데이터, k=12)

1,407건의 Dyson 특허가 12개 cluster로 묶였고, 각 cluster에서 LLM이
- 62 modules
- 86 functional elements (동사구)
- 111 physical elements (명사구)
- 83 function ↔ physical ↔ module mappings

을 evidence-grounded 로 추출했다. cluster별 정리된 결과는 `outputs/cluster_*.csv` 참고.

### 도출된 cluster (label은 LLM 제안값)

| cid | n | label |
|----:|----:|---|
| 0 | 107 | Thin-film solid-state battery fabrication via sputter deposition |
| 1 | 115 | Haircare/Styling Appliances with Airflow Attachments |
| 2 |  92 | Handheld Appliance Fluid Flow Architecture (Hairdryer-class) |
| 3 |  91 | Sensor-driven smart vacuum cleaners and mobile cleaning robots |
| 4 | 125 | Vacuum Cleaner Head with Agitator/Brush Bar |
| 5 | 166 | Surface treating appliance main body & support assembly architecture |
| 6 | 130 | Multi-stage cyclonic dirt/dust separation for vacuum cleaners |
| 7 |  98 | Bladeless nozzle-based fan assemblies |
| 8 |  89 | Brushless (permanent-magnet) motor control via winding excitation |
| 9 |  58 | Brushless Motor / Compressor Rotor-Stator Assemblies |
| 10 | 130 | Cleaning appliances: dental fluid-jet devices and vacuum floor tools |
| 11 | 206 | Wearable air purification and air-treatment appliances |

### k 탐색 범위와 CHOSEN_K

전체 corpus에 대한 KMeans sweep을 `k=3..25` 로 확장해 두었다.
silhouette score는 k에 따라 단조 증가하지만 절대값이 매우 작아 (k=3 ≈ 0.013, k=12 ≈ 0.028, k=25 ≈ 0.043) argmax는 noisy하다.
그래서 노트북은 `CHOSEN_K = 12`로 명시적으로 고정해, 기존 LLM 라벨링 결과 및 cluster 10·11에 대한 sub-clustering ID 매핑과 호환되도록 했다.

### Mixed cluster sub-clustering (cluster 10, 11)

cluster 10, 11은 LLM notes 단계에서 "여러 product family가 한 cluster에 섞였다"고 명확히 지적된 mixed cluster다.
전체 k를 더 키우는 대신, 두 cluster의 문서만 분리해 **TF-IDF를 다시 학습 → KMeans `sub_k=2..8` sweep → silhouette 기준 best sub_k**를 잡았다.

| parent | n | best sub_k | sub-cluster silhouette (best) |
|----:|----:|----:|----:|
| 10 | 130 | 7 | 0.083 |
| 11 | 206 | 8 | 0.061 |

sub_k에서 silhouette가 parent silhouette(≈0.028)보다 2~3배 높아진 점이, 이 두 cluster가 sub-sweep 단위에서 실제로 더 잘 분리됨을 뒷받침한다.

**Cluster 10 → 7 sub-clusters** (top distinctive terms 일부):

| comp id | n | 주요 term |
|---|---:|---|
| 10.0 | 28 | working fluid · teeth · dental · fluid reservoir · dental cleaning |
| 10.1 | 26 | floor tool · vacuum cleaning · suction nozzle |
| 10.2 | 31 | bristle · attachment · brush · diffuser |
| 10.3 | 23 | receptacle · longitudinal axis · surface cleaning |
| 10.4 |  4 | canister · steering · rolling assembly |
| 10.5 |  9 | floor care · decontamination · refrigerant · heat exchanger |
| 10.6 |  9 | floor cleaner · dock · receiving unit · liquid |

**Cluster 11 → 8 sub-clusters** (top distinctive terms 일부):

| comp id | n | 주요 term |
|---|---:|---|
| 11.0 | 23 | water tank · humidifying · ultraviolet · moisture |
| 11.1 | 38 | air purifier · head wearable · speaker assembly · headgear |
| 11.2 | 24 | casing · cavity · drying · sleeve |
| 11.3 | 28 | hand dryer · air-knife · sink · basin |
| 11.4 | 13 | air delivery · mask · wearable air |
| 11.5 | 19 | light source · domestic appliance · illuminate |
| 11.6 | 27 | fan assembly · base body · stand · air outlets |
| 11.7 | 34 | filter · filter medium · vacuum cleaner |

전체 sub-cluster summary는 `outputs/sub_cluster_summary.csv`, 문서→sub-cluster 매핑은 `outputs/patent_subclusters.csv` 참고.

## Cluster별 추출 결과 (LLM)

아래 표는 각 cluster에 대해 LLM이 추출한 **module / functional element / physical element** 를 정리한 것이다.
근거(evidence)·confidence 까지 포함된 전체 표는 `outputs/cluster_*.csv` 참고.

<details><summary><b>Cluster 0</b> &nbsp;|&nbsp; 107 patents &nbsp;|&nbsp; Thin-film solid-state battery fabrication via sputter deposition</summary>

**Modules (6)**
  - **electrode layer module**
  - **electrolyte layer module**
  - **substrate module**
  - **stack assembly module**
  - **sputter deposition module**  _(confidence: medium)_
  - **current collector module**  _(confidence: low)_

**Functional elements (8)** — 동사구
  - **store energy**
  - **deposit electrolyte material**
  - **deposit electrode material on substrate**
  - **provide electrode layer on substrate**
  - **separate adjacent stacks with a groove**
  - **conduct ions between electrodes**  _(confidence: medium)_
  - **sputter deposit layers**  _(confidence: medium)_
  - **form solid-state electrochemical cell**

**Physical elements (10)** — 명사구
  - **substrate**
  - **first electrode layer**
  - **second electrode layer**
  - **electrolyte layer**
  - **stack**
  - **groove**
  - **cathode**  _(confidence: medium)_
  - **sputter target**  _(confidence: medium)_
  - **current collector**  _(confidence: low)_
  - **solid-state electrochemical cell component**

**Notes**
  - _uncertain_: Existence of a dedicated sputter deposition module is inferred from top terms rather than explicit abstract text.
  - _uncertain_: Current collector is referenced only by a truncated abstract fragment.
  - _insufficient evidence_: Specific battery chemistry (e.g., lithium) is not explicitly stated in provided abstracts.
  - _insufficient evidence_: Packaging, encapsulation, or thermal management modules are not described in the provided evidence.

</details>

<details><summary><b>Cluster 1</b> &nbsp;|&nbsp; 115 patents &nbsp;|&nbsp; Haircare/Styling Appliances with Airflow Attachments</summary>

**Modules (3)**
  - **airflow generation module**
  - **hair styling attachment module**
  - **airflow shaping/guiding surface module**  _(confidence: medium)_

**Functional elements (7)** — 동사구
  - **generate airflow**
  - **receive air**
  - **emit airflow**
  - **attach accessory to haircare appliance**
  - **style hair**
  - **guide airflow along curved surface**  _(confidence: medium)_
  - **diffuse airflow**  _(confidence: medium)_

**Physical elements (7)** — 명사구
  - **air inlet**
  - **air outlet**
  - **airflow generator**
  - **attachment**
  - **curved surface**
  - **flat surface**  _(confidence: medium)_
  - **diffuser**  _(confidence: medium)_

**Notes**
  - _uncertain_: Diffuser is listed in top title terms but not explicitly described in provided abstracts; its specific function is inferred only at a high level.
  - _uncertain_: Flat surface role is partially described and its specific function is not fully detailed.
  - _insufficient evidence_: No explicit evidence of heating, temperature control, or motor-specific components in the provided text.
  - _insufficient evidence_: No explicit evidence of power/battery module in the provided text.

</details>

<details><summary><b>Cluster 2</b> &nbsp;|&nbsp; 92 patents &nbsp;|&nbsp; Handheld Appliance Fluid Flow Architecture (Hairdryer-class)</summary>

**Modules (5)**
  - **fluid flow path module**
  - **appliance body module**
  - **duct module**
  - **heater assembly module**  _(confidence: medium)_
  - **attachment module**  _(confidence: medium)_

**Functional elements (8)** — 동사구
  - **admit fluid into the appliance**
  - **emit fluid flow from the appliance**
  - **convey fluid from inlet to outlet**
  - **line the duct with a material**
  - **guide fluid flow in an axial direction**
  - **heat fluid**  _(confidence: medium)_
  - **attach accessory to appliance**  _(confidence: medium)_
  - **clean**  _(confidence: low)_

**Physical elements (9)** — 명사구
  - **body**
  - **duct**
  - **fluid inlet**
  - **fluid outlet**
  - **primary fluid flow path**
  - **duct lining material**
  - **wall**  _(confidence: medium)_
  - **heater assembly**  _(confidence: medium)_
  - **attachment**  _(confidence: medium)_

**Notes**
  - _uncertain_: Heater assembly and attachment modules are inferred from top title terms but not explicitly described in the example abstracts.
  - _uncertain_: Cleaning function appears in title terms but is not supported by the provided example abstracts which focus on hairdryers.
  - _insufficient evidence_: Specific structure of heater assembly
  - _insufficient evidence_: Specific types of attachments and how they connect
  - _insufficient evidence_: Cleaning appliance configuration

</details>

<details><summary><b>Cluster 3</b> &nbsp;|&nbsp; 91 patents &nbsp;|&nbsp; Sensor-driven smart vacuum cleaners and mobile cleaning robots</summary>

**Modules (7)**
  - **airflow generation module**
  - **cleaner head module**
  - **sensing module**
  - **control and processing module**
  - **mobile robot platform module**  _(confidence: medium)_
  - **hairstyling attachment module**  _(confidence: low)_
  - **oral treatment module**  _(confidence: low)_

**Functional elements (7)** — 동사구
  - **generate airflow**
  - **agitate surface debris**
  - **sense motion and orientation**
  - **generate sensor signals**
  - **process sensor signals**
  - **determine surface type**
  - **perform diagnostics**  _(confidence: medium)_

**Physical elements (8)** — 명사구
  - **vacuum motor**
  - **cleaner head**
  - **agitator**
  - **motion and orientation sensor**
  - **diagnostic sensors**
  - **controller**
  - **first and second controller modules**
  - **surface type model**

**Notes**
  - _uncertain_: Whether 'mobile robot' refers to robotic vacuum cleaner variants within this cluster - title terms suggest yes but no abstract evidence shown.
  - _uncertain_: Presence of 'hairstyling' and 'oral treatment' title terms suggests heterogeneous cluster contents beyond vacuum cleaners.
  - _insufficient evidence_: No abstracts provided for hairstyling or oral treatment products, so corresponding modules and functions cannot be reliably extracted.
  - _insufficient evidence_: The role of 'light' (top tfidf term) is not explained in any provided abstract.

</details>

<details><summary><b>Cluster 4</b> &nbsp;|&nbsp; 125 patents &nbsp;|&nbsp; Vacuum Cleaner Head with Agitator/Brush Bar</summary>

**Modules (5)**
  - **cleaner head housing module**
  - **agitator module**
  - **suction airflow module**
  - **debris illumination module**  _(confidence: medium)_
  - **dirt separator module**  _(confidence: low)_

**Functional elements (9)** — 동사구
  - **define a suction chamber**
  - **engage a surface to be cleaned**
  - **provide fluid connection to a suction generator**
  - **illuminate debris on a work surface**
  - **agitate surface debris**
  - **drive the agitator about an axis**
  - **rotate the drive mechanism relative to the housing**  _(confidence: medium)_
  - **mount agitator elements on opposing sides of the housing**  _(confidence: medium)_
  - **separate dirt from airflow**  _(confidence: low)_

**Physical elements (13)** — 명사구
  - **cleaner head**
  - **housing**
  - **body**
  - **suction chamber**
  - **suction inlet**
  - **outlet**
  - **agitator**
  - **brush bar**
  - **drive mechanism**
  - **laser diode**
  - **first agitator element**
  - **second agitator element**  _(confidence: medium)_
  - **dirt separator**  _(confidence: low)_

**Notes**
  - _uncertain_: dirt separator module is suggested by title terms but no abstract evidence is provided
  - _uncertain_: second agitator element function is truncated in the abstract
  - _insufficient evidence_: specific separation mechanism for dirt separator
  - _insufficient evidence_: materials of housing, agitator, or brush bar

</details>

<details><summary><b>Cluster 5</b> &nbsp;|&nbsp; 166 patents &nbsp;|&nbsp; Surface treating appliance main body & support assembly architecture</summary>

**Modules (7)**
  - **main body**
  - **surface treating head**
  - **support assembly**
  - **hose and wand assembly**
  - **airflow generation module**
  - **handle**
  - **flow change-over valve**  _(confidence: medium)_

**Functional elements (7)** — 동사구
  - **generate flow of fluid**
  - **treat a surface**
  - **roll main body along a surface**
  - **support the main body**
  - **operate appliance by user**
  - **convey air between components**  _(confidence: medium)_
  - **switch airflow path**  _(confidence: medium)_

**Physical elements (10)** — 명사구
  - **main body**
  - **surface treating head**
  - **support assembly**
  - **handle**
  - **hose**
  - **wand assembly**
  - **motor**
  - **fan**
  - **rotary change over valve**
  - **dome-shaped wheels**  _(confidence: medium)_

**Notes**
  - _uncertain_: The function of the rotary change over valve is inferred from its name; abstracts do not explicitly describe what paths it switches.
  - _uncertain_: Robotic and domestic appliance variants appear in title terms but are not detailed in abstracts.
  - _insufficient evidence_: Filtration, dust collection, or cyclone separation modules are not described in the provided abstracts even though typical for vacuum cleaners.
  - _insufficient evidence_: Battery/power module not mentioned in provided text.

</details>

<details><summary><b>Cluster 6</b> &nbsp;|&nbsp; 130 patents &nbsp;|&nbsp; Multi-stage cyclonic dirt/dust separation for vacuum cleaners</summary>

**Modules (5)**
  - **cyclonic separating module**
  - **multistage cyclonic separation module**
  - **dust collection module**
  - **handheld cleaning appliance module**
  - **airflow path module**

**Functional elements (7)** — 동사구
  - **separate dirt and dust from airflow**
  - **perform cyclonic separation**
  - **perform multistage separation**
  - **collect dust**
  - **admit dirty air**
  - **discharge clean air**
  - **guide airflow from inlet to outlet**

**Physical elements (9)** — 명사구
  - **first cyclonic separating unit**
  - **second cyclonic separating unit**
  - **first cyclone**
  - **second cyclone**
  - **annular dust collecting chamber**
  - **dirty air inlet**
  - **clean air outlet**
  - **airflow path**
  - **handheld cleaning appliance body**  _(confidence: medium)_

**Notes**
  - _uncertain_: Whether the handheld cleaning appliance body is a distinct physical element or only the overall product housing is not explicitly described.
  - _insufficient evidence_: No explicit mention of motor, impeller, or filter within the provided abstracts, although such components are typical of vacuum cleaners.

</details>

<details><summary><b>Cluster 7</b> &nbsp;|&nbsp; 98 patents &nbsp;|&nbsp; Bladeless nozzle-based fan assemblies</summary>

**Modules (4)**
  - **airflow generation module**
  - **nozzle air emission module**
  - **base housing module**
  - **humidifying module**  _(confidence: low)_

**Functional elements (6)** — 동사구
  - **create an air current**
  - **rotate the impeller**
  - **generate air flow from inlet to outlet**
  - **receive air flow from interior passage**
  - **emit air flow**
  - **humidify air**  _(confidence: low)_

**Physical elements (8)** — 명사구
  - **impeller**
  - **motor**
  - **air inlet**
  - **air outlet**
  - **nozzle**
  - **interior passage**
  - **mouth**
  - **base**

**Notes**
  - _uncertain_: humidifying module is suggested only by a title term and lacks abstract-level evidence in the provided examples
  - _insufficient evidence_: specific control electronics, power source, or oscillation mechanisms are not described in provided text

</details>

<details><summary><b>Cluster 8</b> &nbsp;|&nbsp; 89 patents &nbsp;|&nbsp; Brushless (permanent-magnet) motor control via winding excitation</summary>

**Modules (5)**
  - **brushless motor control module**
  - **motor winding excitation module**
  - **AC-to-DC rectification module**
  - **permanent-magnet rotor module**  _(confidence: medium)_
  - **current sensing and threshold module**  _(confidence: medium)_

**Functional elements (8)** — 동사구
  - **control a brushless motor**
  - **excite a motor winding with a voltage**
  - **freewheel the winding**
  - **sense winding current**
  - **rectify an alternating voltage**
  - **continue exciting winding for an overrun period**
  - **terminate excitation after a timeout period**
  - **provide rectified voltage with ripple**

**Physical elements (7)** — 명사구
  - **brushless motor**
  - **permanent-magnet motor**
  - **phase winding**
  - **rectifier**
  - **alternating voltage supply**  _(confidence: medium)_
  - **current sensor**  _(confidence: medium)_
  - **motor controller**  _(confidence: medium)_

**Notes**
  - _uncertain_: The presence of a dedicated current sensor and motor controller is inferred from control actions but not explicitly named as components in the abstracts.
  - _uncertain_: Permanent-magnet rotor module is supported only by the term 'permanent-magnet motor' without explicit rotor structure details.
  - _insufficient evidence_: No explicit evidence of inverter/switching bridge topology, gate drivers, or specific control IC.
  - _insufficient evidence_: No explicit evidence of position/Hall sensors despite control of brushless motors typically requiring them.

</details>

<details><summary><b>Cluster 9</b> &nbsp;|&nbsp; 58 patents &nbsp;|&nbsp; Brushless Motor / Compressor Rotor-Stator Assemblies</summary>

**Modules (6)**
  - **rotor assembly module**
  - **stator assembly module**
  - **bearing assembly module**
  - **motor frame / housing module**
  - **impeller module**  _(confidence: medium)_
  - **heat sink module**  _(confidence: medium)_

**Functional elements (8)** — 동사구
  - **rotate relative to the stator**
  - **house the stator and rotor assemblies**
  - **support the rotor shaft**
  - **mount the bearing assembly to the frame**
  - **move air via impeller rotation**  _(confidence: medium)_
  - **dissipate heat**  _(confidence: medium)_
  - **compress fluid**  _(confidence: medium)_
  - **bond components with adhesive**  _(confidence: low)_

**Physical elements (15)** — 명사구
  - **rotor assembly**
  - **stator assembly**
  - **shaft**
  - **rotor core**
  - **stator core**
  - **bearing assembly**
  - **bearings**
  - **frame**
  - **frame outer portion**
  - **frame support portion**
  - **impeller**
  - **heat sink assembly**  _(confidence: medium)_
  - **magnet**  _(confidence: medium)_
  - **adhesive**  _(confidence: low)_
  - **C-shaped stator core**

**Notes**
  - _uncertain_: Role of 'adhesive' is unclear from abstracts; possibly used to bond stator core or bearings.
  - _uncertain_: 'Magnet' appears in tfidf terms but not in shown abstracts; likely part of rotor core.
  - _insufficient evidence_: Specific control electronics or sensor modules are not described in the provided excerpts.
  - _insufficient evidence_: Detailed cooling airflow path beyond heat sink is not specified.

</details>

<details><summary><b>Cluster 10</b> &nbsp;|&nbsp; 130 patents &nbsp;|&nbsp; Cleaning appliances: dental fluid-jet devices and vacuum floor tools</summary>

**Modules (5)**
  - **handle module**
  - **cleaning tool attachment module**
  - **fluid delivery module**
  - **fluid reservoir module**
  - **floor tool module**  _(confidence: medium)_

**Functional elements (6)** — 동사구
  - **deliver a burst of working fluid to the teeth**
  - **store working fluid**
  - **receive working fluid from the fluid reservoir**
  - **detachably connect the cleaning tool to the handle**
  - **move fluid conduit about an axis**  _(confidence: medium)_
  - **clean floors via suction**  _(confidence: medium)_

**Physical elements (8)** — 명사구
  - **handle**
  - **cleaning tool**
  - **nozzle**
  - **stem**
  - **fluid reservoir**
  - **fluid delivery system**
  - **moveable fluid conduit**
  - **floor tool**  _(confidence: medium)_

**Notes**
  - _uncertain_: The cluster mixes dental cleaning appliances (handle, nozzle, fluid reservoir) with floor/vacuum cleaning appliances (floor tool, suction); these likely represent two distinct sub-domains under generic title 'CLEANING APPLIANCE'.
  - _insufficient evidence_: Detail on motor/pump generating the burst of working fluid is not present in the provided abstracts.
  - _insufficient evidence_: Detail on the floor tool's internal components (brush bar, etc.) is not visible in the provided abstracts.

**Sub-clusters** (best sub_k = 7, silhouette 0.083 — `outputs/sub_cluster_summary.csv`)

| comp id | n | 추정 sub-domain | 대표 title / 주요 term |
|---|---:|---|---|
| 10.0 | 28 | Dental cleaning appliance | CLEANING APPLIANCE · working fluid · teeth · dental · fluid reservoir |
| 10.1 | 26 | Floor tool for vacuum cleaner | FLOOR TOOL FOR A VACUUM CLEANING APPLIANCE · suction nozzle |
| 10.2 | 31 | Brush/bristle attachment | ATTACHMENT FOR A VACUUM CLEANING APPLIANCE · bristle · carrier · diffuser |
| 10.3 | 23 | Domestic surface cleaning / receptacle | DOMESTIC APPLIANCE · receptacle · longitudinal axis · side wall |
| 10.4 |  4 | Canister vacuum + steering mechanism | CANISTER VACUUM CLEANER · steering · rolling assembly |
| 10.5 |  9 | Self-cleaning vacuum (decontamination) | SELF-CLEANING VACUUM CLEANER · decontamination · refrigerant · heat exchanger |
| 10.6 |  9 | Floor cleaner dock / liquid reservoir | FLOOR CLEANER DOCK · receiving unit · liquid · reservoir |

</details>

<details><summary><b>Cluster 11</b> &nbsp;|&nbsp; 206 patents &nbsp;|&nbsp; Wearable air purification and air-treatment appliances</summary>

**Modules (4)**
  - **air filtration module**
  - **airflow generation module**
  - **head wearable ear/speaker assembly module**
  - **hand dryer / drying module**  _(confidence: low)_

**Functional elements (5)** — 동사구
  - **create airflow**
  - **filter air**
  - **be worn over an ear of a user**
  - **output sound to user**  _(confidence: medium)_
  - **dry hands**  _(confidence: low)_

**Physical elements (7)** — 명사구
  - **filter assembly**
  - **motor-driven impeller**
  - **ear assembly**
  - **first speaker assembly**
  - **second speaker assembly**
  - **fan assembly**  _(confidence: medium)_
  - **hand dryer**  _(confidence: low)_

**Notes**
  - _uncertain_: Relationship between hand dryer/drying terms and the head wearable air purifier representative titles is unclear; the cluster may mix multiple Dyson air-treatment product families.
  - _uncertain_: Role of 'water' and 'light' top terms is not supported by the example abstracts.
  - _insufficient evidence_: hand dryer module - no abstract evidence in provided examples
  - _insufficient evidence_: specific water-related or light-related components

**Sub-clusters** (best sub_k = 8, silhouette 0.061 — `outputs/sub_cluster_summary.csv`)

| comp id | n | 추정 sub-domain | 대표 title / 주요 term |
|---|---:|---|---|
| 11.0 | 23 | Humidifier / UV air treatment | AIR TREATMENT APPARATUS · water tank · humidifying · ultraviolet · moisture |
| 11.1 | 38 | Wearable air purifier (headgear) | WEARABLE AIR PURIFIER · head wearable · speaker assembly · headgear |
| 11.2 | 24 | Drying apparatus (casing/cavity) | DRYING APPARATUS · casing · cavity · sleeve · slot-like opening |
| 11.3 | 28 | Hand dryer (basin/air-knife) | HAND DRYER · air-knife · sink · basin · spout |
| 11.4 | 13 | Wearable air purifier (delivery mask) | WEARABLE AIR PURIFIER · delivery mask · air purification |
| 11.5 | 19 | Self-cleaning domestic appliance (light/heat) | SELF-CLEANING DOMESTIC APPLIANCE · light source · illuminate · heating |
| 11.6 | 27 | Stand fan assembly | FAN ASSEMBLY · base body · stand · air outlets |
| 11.7 | 34 | Filter assembly (vacuum cleaner) | FILTER ASSEMBLY · filter medium · vacuum cleaner · biodegradable filter |

</details>
