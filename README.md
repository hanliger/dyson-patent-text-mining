# Dyson Patent Text Mining

Dyson 특허 데이터(`발명의 명칭` + `요약`)에 대한 텍스트 마이닝과
LLM 기반 제품공학 요소 추출 파이프라인.

## Pipeline

1. xlsx 데이터 로드 (`data/TextDown_dyson_patent_20260526.xlsx`)
2. 텍스트 코퍼스 구성 (title + abstract)
3. 전처리 (소문자, 구두점/숫자 제거, 일반 특허 boilerplate stopword만 제거)
4. TF-IDF 기반 keyword 추출 (1–3 gram, 전체 + title only)
5. WordCloud (전체 / title only / cluster별)
6. KMeans clustering (k=3..12 sweep, silhouette 기준 자동 선택)
7. Cluster summary (distinctive term + representative title + example abstract)
8. **Optional** LLM 기반 cluster label 제안
9. **Optional** LLM 기반 module / functional element / physical element 추출

LLM 호출은 `ANTHROPIC_API_KEY` 가 설정된 경우에만 동작한다.
key 없이도 단계 1–7은 그대로 끝까지 실행된다.

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
