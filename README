# PF-EBM-OR
## 기상 데이터 및 공간정보 기반 전력설비 인근 화재 위험도 분석

**대회**: 2026 날씨 빅데이터 콘테스트 (주제1, 재난안전)
**주관**: 기상청, 한국전력공사
**작성일**: 2026-05-28
**버전**: v0.1 (draft)

---

## 0. 한 줄 요약

KMA 기상 + KEPCO 전력설비 데이터를 융합하여, **glassbox 위험도 추정** (PF-EBM) → **공간·시간 보정** → **운영 의사결정 우선순위 산출** (OR layer) 까지 end-to-end 파이프라인을 구성한다. 핵심 통계 연산은 **Rust로 구현하여 PyO3로 바인딩**, Jupyter 환경에서 사용한다.

---

## 1. 프로젝트 개요

### 1.1 문제 정의

전력설비(송전선·송전탑·변전소) 인근에서 발생하는 화재는 두 방향의 위험을 동반한다:

- **방향 A (Power → Fire)**: 설비 측 점화원 — 아크, 단락, 수목 접촉, 노후 절연 파괴
- **방향 B (Fire → Power)**: 외부 화재의 설비 침범 — 화염 가교에 의한 절연 파괴, 연기·미세입자에 의한 절연강도 저하, 트립 (line tripping)

본 프로젝트는 두 방향 모두를 포괄하는 **시공간 위험도** $R(s, t)$를 추정하고, 이를 KEPCO의 의사결정(점검·PSPS·지중화 우선순위)에 직접 매핑한다.

### 1.2 목표 (산출물)

| # | 산출물 | 형태 |
|---|--------|------|
| O1 | 격자 단위 (1km × 1km, 일별) 화재 위험 확률맵 | GeoTIFF + 보정된 conformal 구간 |
| O2 | 전력설비 세그먼트별 위험 점수 | CSV + GeoJSON |
| O3 | 예산 제약 하 점검 우선순위 리스트 | CSV + 지도 시각화 |
| O4 | 해석가능성 리포트 (각 feature의 부분 의존성) | HTML/PDF |
| O5 | Rust 핵심 연산 라이브러리 + Python 바인딩 | PyPI 호환 wheel |

### 1.3 차별화 포인트

기존 문헌의 빈틈을 직접 공략한다.

1. **전력설비 관점**: 대부분 산불(wildfire) 관점이고 KEPCO 운영 의사결정과 분리되어 있음 → 설비 토폴로지를 1급 시민으로 취급
2. **본질적 해석성**: 기존 연구의 대다수가 SHAP 같은 post-hoc → EBM 기반 glassbox로 직접 설명
3. **OR 결정층**: 위험 추정에서 끝나지 않고 자원 배분 최적화까지 연결
4. **Conformal 보정**: 점추정이 아닌 검증 가능한 신뢰구간 제공
5. **고성능 통계 백엔드**: Rust로 핵심 연산 → 대규모 격자 처리 가능

### 1.4 평가 기준 가정

| 기준 | 가중치 추정 | 대응 |
|------|------------|------|
| 예측 정확도 | 25% | AUROC, AUPRC, calibration |
| 해석가능성 | 25% | EBM glassbox + 부분 의존성 시각화 |
| 실무 적용성 (KEPCO) | 30% | OR 결정층 + 우선순위 리스트 |
| 데이터 활용 깊이 | 10% | 다양한 외부 공공 데이터 융합 |
| 발표/제출 완성도 | 10% | 문서·시각화·재현성 |

---

## 2. 데이터 명세

### 2.1 1차 데이터 (콘테스트 제공)

#### 2.1.1 KMA 기상 데이터

| 항목 | 출처 | 해상도 | 비고 |
|------|------|--------|------|
| ASOS 종관관측 | data.kma.go.kr | 시간별 / 지점 | 약 100여 개 지점 |
| AWS 자동관측 | data.kma.go.kr | 1분~시간별 / 지점 | 약 600여 개 지점 |
| LDAPS/GDAPS 수치모델 | 날씨마루 | 1.5km / 1시간 | 공간 보간 대신 사용 가능 |
| 동네예보 격자자료 | 날씨마루 | 5km / 3시간 | 운영 시 예측 입력 |
| 기온통계·강수통계 | data.kma.go.kr | 평년값 | climatology baseline |

**필수 변수**: 기온(T), 상대습도(RH), 풍속(WS), 풍향(WD), 강수량(P), 일조, 토양수분(가능시)

#### 2.1.2 KEPCO 전력설비 데이터

콘테스트 제공 데이터 확인 전이라 가정 기반. 다음 항목이 포함될 것으로 예상:

| 항목 | 형태 | 활용 |
|------|------|------|
| 송전선로 GIS | LineString (위경도) | 인근 격자 식별, 거리 feature |
| 송전탑 위치 | Point | 점화원 위치 후보 |
| 변전소 위치 | Point | 거점 식별 |
| 전압 등급 | 154 / 345 / 765 kV | 화염 가교 임계거리 결정 |
| 가공/지중 구분 | Categorical | 위험 노출도 |
| 노후도 / 준공년도 | (제공 시) | 절연 열화 위험 |
| 최근 점검이력 | (제공 시) | 최근 점검일로부터의 기간 |

**불확실성**: 설비 노후도와 점검이력이 제공되지 않을 가능성. 이 경우 voltage class와 가공/지중 구분만으로 설비 측 feature 구성.

### 2.2 2차 데이터 (외부 공공, 필수)

이건 콘테스트 제공이 아니어도 무조건 끌어와야 합니다. **라벨이 산림청에서 옵니다**.

| 데이터 | 출처 | 용도 |
|--------|------|------|
| **산림청 산불통계** | forest.go.kr, 공공데이터포털 | **라벨 (양성 사건)** |
| 국가산불위험예보 | forestfire.nifos.go.kr | 비교 베이스라인 |
| DEM (수치표고모델) | 국토지리정보원 / SRTM | 고도, 슬로프, 향(aspect) |
| 토지피복도 | 환경부 EGIS | land cover 분류 |
| NDVI / 식생지수 | MODIS, Sentinel-2 (GEE) | 연료량 proxy |
| 임상도 (forest type) | 산림청 | 침엽수 비율 (위험 증폭) |
| 도로망 | 국토지리정보원 | 인위적 점화 접근성 |
| 인구 격자 | SGIS / 통계청 | 인위적 점화 압력 |
| 행정구역 | 통계청 SGIS | 집계 단위 |

### 2.3 라벨 정의

이게 모델링의 절반입니다. 신중하게 결정.

**양성 (positive) 정의**:
- 산림청 산불 발생일시 $t$, 위치 $(x, y)$에서
- 가장 가까운 전력설비(송전선/탑/변전소)까지 거리 $d \leq R$ 인 사건
- 동일 격자·동일 일자는 1회로 카운트

**거리 임계치 $R$의 다중 정의 (multi-task 또는 다중 모델)**:

| Scale | $R$ | 의미 | 예상 양성 수 |
|-------|-----|------|--------------|
| 직접 (direct) | 100 m | 화염 가교, 직접 접촉 위험 | 매우 희소 |
| 근접 (proximity) | 500 m | 연기·미세입자 절연강도 저하 | 희소 |
| 회랑 (corridor) | 1,000 m | 송전선 회랑 위협 (NBN 문헌 표준) | 보통 |

**1차 모델은 R=500m로 시작**하고, 다른 scale은 ablation으로 추가 검증.

**음성 (negative) 샘플링**:
- presence-only 문제. 단순 무작위 background는 편향 발생.
- 전략: **stratified background sampling**
  - 같은 시기 (계절 보존)
  - 같은 land cover 분포
  - 전력설비 인근 격자(같은 설비 노출도)에서 추출
  - 양성:음성 = 1:5 ~ 1:10

### 2.4 공간·시간 해상도

| 차원 | 해상도 | 근거 |
|------|--------|------|
| 공간 | **1 km × 1 km 격자** | KMA 격자자료 호환, 계산 가능성 |
| 시간 | **일별 (daily)** | 산불 데이터 시간 정밀도, 운영 의사결정 주기 |
| 분석기간 | **2017 ~ 2024 (8년)** | 라벨 충분성 + 최근 기후 반영 |
| 학습/검증 분할 | 2017~2022 train / 2023 val / 2024 test | 시간 분리 + spatial block CV |

격자 총 셀: 한국 면적 ≈ 100,378 km² → 약 **100,000 격자**, 일별 ≈ 365 × 8 = 2,920일 → **~2.9억 셀**. 이 규모에서 Python 단독은 불가, Rust 가속이 정당화됩니다.

### 2.5 데이터 사전 (요약)

```
feature_name           type        unit       source        notes
─────────────────────  ──────────  ─────────  ────────────  ───────────────────
T_max                  float       °C         KMA           일 최고기온
T_min                  float       °C         KMA           일 최저기온
RH_min                 float       %          KMA           일 최저상대습도
WS_max                 float       m/s        KMA           일 최대풍속
WD_at_WSmax            float       degree     KMA           최대풍속 시점 풍향
precip_24h             float       mm         KMA           24시간 강수
days_since_rain        int        day        derived       강수 없음 연속일수
ffmc                   float       0-101      derived       Fine Fuel Moisture
dmc                    float       0-∞        derived       Duff Moisture
dc                     float       0-∞        derived       Drought Code
isi                    float       0-∞        derived       Initial Spread Index
bui                    float       0-∞        derived       Buildup Index
fwi                    float       0-∞        derived       Fire Weather Index
kbdi                   float       0-800      derived       Keetch-Byram Drought
elevation              float       m          DEM           고도
slope                  float       degree     DEM           기울기
aspect                 float       degree     DEM           향
ndvi                   float       -1~1       MODIS         식생지수
landcover              categorical -          환경부        토지피복 클래스
conifer_ratio          float       0~1        임상도        침엽수 면적비
dist_to_road           float       m          국토지리원    최근접 도로
dist_to_tline          float       m          KEPCO         최근접 송전선
nearest_voltage_kv     categorical kV         KEPCO         최근접 송전선 전압
overhead_ind           binary      -          KEPCO         가공/지중
n_facilities_1km       int         -          derived       1km 내 설비 수
pop_density            float       /km²       SGIS          인구밀도
hist_fire_kde          float       -          derived       과거 화재 KDE
season                 categorical -          derived       봄/여름/가을/겨울
dayofyear              int         1-366      derived       연중 일
```

---

## 3. 방법론: PF-EBM-OR

### 3.1 전체 구조

```mermaid
flowchart TB
    A[KMA 기상 데이터] --> F[Feature Pipeline<br/>Rust]
    B[KEPCO 설비 GIS] --> F
    C[외부 공공: DEM/NDVI/<br/>토지피복/임상도] --> F
    D[산림청 산불통계] --> L[Label 생성]
    F --> L1[Layer 1<br/>Climatology<br/>Spatial GAM]
    F --> L2[Layer 2<br/>Dynamic EBM<br/>with interactions]
    L --> L1
    L --> L2
    L1 --> L2
    L2 --> CC[Conformal<br/>Calibration]
    CC --> L3[Layer 3<br/>OR Decision<br/>Knapsack]
    L3 --> O1[격자 위험맵]
    L3 --> O2[설비 위험 점수]
    L3 --> O3[점검 우선순위]
```

### 3.2 Layer 1: Climatology Decomposition

장기 평균 위험을 데이터 기반으로 추정해 baseline으로 사용. 본질적으로 spatial GAM.

$$R_{\text{base}}(s, t) = \beta_0 + f_{\text{season}}(\text{doy}(t)) + f_{\text{geo}}(\text{lon}(s), \text{lat}(s)) + f_{\text{lc}}(\text{landcover}(s))$$

- $f_{\text{season}}$: 주기 spline (cyclic cubic spline, period=365)
- $f_{\text{geo}}$: tensor product spline on (lon, lat)
- $f_{\text{lc}}$: categorical effect (각 토지피복 클래스 효과)

**적합 방식**: penalized IRLS, smoothness parameter는 REML로 선택.
**왜 분해하는가**: Layer 2에서 잔차만 학습 → 학습 안정성 향상, 해석 시 "climatology vs anomaly" 분리 가능.

> GLAM 논문에서 GAM Stage I이 했던 역할과 동일. 그대로 응용.

### 3.3 Layer 2: Dynamic EBM with Domain Interactions

핵심 모델. EBM 구조로 잔차 위험을 학습:

$$\text{logit}(P(y=1 \mid x, s, t)) = R_{\text{base}}(s,t) + \sum_{j} f_j(x_j) + \sum_{(i,j) \in \mathcal{I}} f_{ij}(x_i, x_j)$$

**단일 feature 효과** $f_j$ — 다음 변수에 대해 학습:
- 동적 기상: FWI, FFMC, RH_min, WS_max, T_max, days_since_rain
- 정적 환경: elevation, slope, NDVI, conifer_ratio
- 설비: dist_to_tline, nearest_voltage_kv, n_facilities_1km
- 인위: dist_to_road, pop_density, hist_fire_kde

**도메인 상호작용** $\mathcal{I}$ — 사전 지식으로 선정:

| 상호작용 | 물리적 의미 |
|----------|-------------|
| (WS_max, dist_to_tline) | 풍하중 + 수목 접촉 |
| (RH_min, NDVI) | 연료 건조도 × 연료량 |
| (FWI, slope) | 점화 + 전파 위험 |
| (FWI, conifer_ratio) | 침엽수 가연성 |
| (T_max, RH_min) | 열·습도 결합 효과 |
| (nearest_voltage_kv, overhead_ind) | 설비 노출도 |

**학습 알고리즘**: EBM의 round-robin gradient boosting + bagging (interpretML 표준).

**왜 EBM인가**: 각 $f_j$, $f_{ij}$가 직접 시각화 가능 (2D heatmap 또는 1D curve). SHAP 없이도 "왜 위험한가"를 즉시 설명할 수 있음.

### 3.4 Layer 3: Conformal-Calibrated OR Decision

#### 3.4.1 Conformal calibration

EBM의 확률 출력에 conformal prediction을 씌워 calibrated interval 생성:

$$\hat{P}(y=1 \mid x) \in [\hat{p}_{\text{low}}, \hat{p}_{\text{high}}] \text{ with coverage } 1-\alpha$$

- **방법**: split conformal (jackknife+는 비용이 크므로 후순위)
- **calibration set**: 시간 분리 검증셋
- **coverage**: $1-\alpha = 0.9$ 기본

#### 3.4.2 설비 단위 집계

격자 위험을 송전선 세그먼트(타워 $i$ ~ 타워 $i+1$) 단위로 집계:

$$R_{\text{seg}}^k = \max_{s \in \text{buffer}(k, B)} \hat{P}(y=1 \mid x_s)$$

또는 가중평균 (긴 세그먼트는 max 보다 weighted mean이 안정적).

#### 3.4.3 자원 배분 최적화

$$\max_{z \in \{0,1\}^K} \sum_k w_k \cdot \Delta R_k \cdot z_k \quad \text{s.t.} \quad \sum_k c_k z_k \leq B$$

- $z_k$: 설비 $k$ 점검/조치 할당 여부
- $\Delta R_k$: 조치 시 기대 위험 감소 (전문가 입력 또는 데이터 기반 추정)
- $c_k$: 조치 비용
- $B$: 예산

**solver**: PuLP + CBC (오픈소스 MILP) 또는 OR-Tools.

#### 3.4.4 운영 KPI

| KPI | 정의 | 목표 |
|-----|------|------|
| Precision@10% | 상위 10% 우선순위 설비 중 실제 발생 비율 | ≥ baseline +10pp |
| Expected loss reduction | 예산 B 하에서 감소된 기대 위험량 | 최대화 |
| Coverage | conformal interval의 경험적 coverage | 89~91% (목표 90%) |

### 3.5 베이스라인 모델

| 모델 | 역할 |
|------|------|
| Logistic regression (raw features) | sanity check |
| Logistic + FWI only | 도메인 baseline |
| XGBoost + SHAP | "통상적 SOTA" 비교 |
| CatBoost + SHAP | tabular SOTA 비교 |
| PF-EBM-OR (ours) | 제안 모델 |

### 3.6 평가 프로토콜

#### 3.6.1 평가지표

| 카테고리 | 지표 |
|----------|------|
| Discrimination | AUROC, AUPRC, F1@optimal |
| Calibration | Brier, ECE, reliability plot |
| Spatial | Moran's I on residuals (자기상관 잔존 여부) |
| Conformal | Empirical coverage, mean interval width |
| OR | Budget-utility curve, precision@k |
| Interpretability | EBM 부분 의존성 그래프 (정성 평가) |

#### 3.6.2 교차검증

**Spatial block CV** (Roberts et al. 2017 권장):
- 전국을 50km × 50km 블록으로 분할
- 5-fold spatial block CV → 공간 자기상관에 의한 leakage 방지

**Temporal forward chaining**:
- train: 2017-2020 / val: 2021 / test: 2022
- train: 2017-2021 / val: 2022 / test: 2023
- train: 2017-2022 / val: 2023 / test: 2024
- 3 rolling tests의 평균

#### 3.6.3 Ablation

- Layer 1 (climatology) 제거 시 성능 변화
- 도메인 interaction 제거 시 변화
- Conformal layer 제거 (점추정만) 시 의사결정 품질 변화
- $R$ (반경 라벨) 100/500/1000m 비교

---

## 4. 시스템 아키텍처

### 4.1 디렉토리 구조

```
pfire/
├── README.md
├── pyproject.toml                  # uv project, maturin build backend
├── notebooks/                      # Jupyter 워크플로우
│   ├── 01_data_eda.ipynb
│   ├── 02_feature_engineering.ipynb
│   ├── 03_layer1_climatology.ipynb
│   ├── 04_layer2_ebm.ipynb
│   ├── 05_layer3_or_decision.ipynb
│   ├── 06_evaluation.ipynb
│   └── 07_visualization.ipynb
├── pfire_py/                       # Python 패키지 (orchestration)
│   ├── __init__.py
│   ├── data/                       # 로더, 라벨 생성
│   │   ├── kma.py
│   │   ├── kepco.py
│   │   ├── forest_fire.py
│   │   └── labels.py
│   ├── pipeline/                   # 파이프라인 조립
│   │   ├── features.py
│   │   ├── train.py
│   │   └── inference.py
│   ├── models/                     # Python 모델 (EBM wrapper, OR)
│   │   ├── ebm_model.py
│   │   ├── or_decision.py
│   │   └── conformal.py
│   └── viz/                        # 시각화
│       ├── partial_dependence.py
│       └── risk_map.py
├── pfire_rs/                       # Rust workspace
│   ├── Cargo.toml                  # workspace root
│   ├── pfire_features/             # crate 1
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs              # PyO3 entry
│   │       ├── fwi.rs              # FWI/FFMC/DMC/DC/ISI/BUI
│   │       ├── kbdi.rs
│   │       ├── distance.rs         # KD-tree 기반 거리
│   │       ├── kde.rs              # 2D KDE
│   │       └── rolling.rs
│   ├── pfire_gam/                  # crate 2
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── spline.rs           # tensor product B-spline basis
│   │       ├── pirls.rs            # penalized IRLS
│   │       └── reml.rs             # smoothness selection
│   └── pfire_conformal/            # crate 3
│       ├── Cargo.toml
│       └── src/
│           ├── lib.rs
│           ├── split.rs            # split conformal
│           └── jackknife.rs        # jackknife+
└── tests/
    ├── python/
    └── rust/
```

### 4.2 Rust crates 명세

#### 4.2.1 `pfire_features` — 핵심 hot loop 가속

**Rust 측 API 예시**:

```rust
// src/lib.rs
use pyo3::prelude::*;
use numpy::{PyArray2, PyReadonlyArray2, IntoPyArray};

#[pyfunction]
fn compute_fwi(
    py: Python<'_>,
    temp: PyReadonlyArray2<f64>,    // [N_grid, N_time]
    rh:   PyReadonlyArray2<f64>,
    ws:   PyReadonlyArray2<f64>,
    rain: PyReadonlyArray2<f64>,
) -> PyResult<PyObject> {
    let temp = temp.as_array();
    let rh   = rh.as_array();
    let ws   = ws.as_array();
    let rain = rain.as_array();
    let fwi  = pfire_features::fwi::compute(&temp, &rh, &ws, &rain);
    Ok(fwi.into_pyarray(py).to_object(py))
}

#[pyfunction]
fn nearest_facility_distance(
    py: Python<'_>,
    grid_points: PyReadonlyArray2<f64>,        // [N, 2]
    facility_segments: PyReadonlyArray2<f64>,  // [M, 4] = (x1,y1,x2,y2)
) -> PyResult<PyObject> {
    let d = pfire_features::distance::nearest_segment(
        &grid_points.as_array(),
        &facility_segments.as_array(),
    );
    Ok(d.into_pyarray(py).to_object(py))
}

#[pymodule]
fn pfire_features(_py: Python<'_>, m: &PyModule) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(compute_fwi, m)?)?;
    m.add_function(wrap_pyfunction!(nearest_facility_distance, m)?)?;
    Ok(())
}
```

**핵심 함수**:
- `compute_fwi`: Van Wagner (1987) FWI 시스템 6 지수 일괄 계산. rayon으로 격자 병렬.
- `kbdi`: Keetch-Byram 일별 누적. 시간 순차적이라 격자 차원만 병렬.
- `nearest_facility_distance`: 송전선 line segments에 대한 격자 점 최단거리. R*-tree (rstar crate) 사용.
- `kernel_density_2d`: 과거 화재 KDE. Gaussian kernel, grid-based, 격자 병렬.
- `rolling_stats`: 시계열 rolling mean/std/quantile. 격자 병렬.

#### 4.2.2 `pfire_gam` — Spatial GAM 핵심

**핵심 컴포넌트**:

```rust
pub struct TensorProductSpline {
    knots_x: Vec<f64>,
    knots_y: Vec<f64>,
    degree: usize,
    penalty_x: na::DMatrix<f64>,  // smoothness penalty
    penalty_y: na::DMatrix<f64>,
}

impl TensorProductSpline {
    pub fn basis(&self, x: &Array1<f64>, y: &Array1<f64>) -> Array2<f64> { ... }
    pub fn fit_pirls(
        &self,
        X: &Array2<f64>,
        y: &Array1<f64>,
        family: Family,
        lambda: f64,
    ) -> PirlsResult { ... }
    pub fn select_lambda_reml(&self, X: &Array2<f64>, y: &Array1<f64>) -> f64 { ... }
}
```

- **B-spline basis**: 1D B-spline tensor product.
- **PIRLS**: penalized iteratively reweighted least squares. SARIMAX의 multi-stage MLE와 구조 유사.
- **REML**: smoothness parameter $\lambda$ 자동 선택. Wood (2011) 알고리즘.
- **희소 행렬**: rustima에서 가져온 sparse linear algebra 재사용 가능.

> **rustima 코드 재활용**: sparse Kalman의 희소 행렬 처리, REML 최적화 루틴은 거의 그대로 PIRLS에 옮길 수 있습니다.

#### 4.2.3 `pfire_conformal` — Conformal wrapper

```rust
#[pyfunction]
fn split_conformal_intervals(
    cal_predictions: PyReadonlyArray1<f64>,   // calibration set predictions
    cal_labels: PyReadonlyArray1<f64>,
    test_predictions: PyReadonlyArray1<f64>,  // test predictions
    alpha: f64,                                // 1 - coverage
) -> PyResult<(Py<PyArray1<f64>>, Py<PyArray1<f64>>)> { ... }
```

- 모델 자체는 Python에서 학습 (EBM은 interpretML).
- Rust는 calibration score 계산, quantile, interval 산출만.
- jackknife+는 parallel CV fold 처리 (rayon).

### 4.3 빌드/배포 (maturin)

**Cargo.toml** (workspace root):
```toml
[workspace]
members = ["pfire_features", "pfire_gam", "pfire_conformal"]
resolver = "2"

[workspace.dependencies]
pyo3 = { version = "0.21", features = ["extension-module"] }
numpy = "0.21"
ndarray = { version = "0.16", features = ["rayon"] }
rayon = "1.10"
rstar = "0.12"
```

**pyproject.toml** (root, 각 crate별로):
```toml
[build-system]
requires = ["maturin>=1.5"]
build-backend = "maturin"

[project]
name = "pfire-features"
requires-python = ">=3.10"

[tool.maturin]
features = ["pyo3/extension-module"]
```

### 4.4 Jupyter 워크플로우

#### 개발 단계
```bash
cd pfire_rs/pfire_features
maturin develop --release       # 빌드 + 현재 venv에 설치
```
Jupyter에서:
```python
import pfire_features as pf
fwi = pf.compute_fwi(temp, rh, ws, rain)
```

**핫 리로드 주의**: PyO3 compiled module은 `importlib.reload`로 갱신 안 됨. Rust 코드 수정 후엔 **`maturin develop` 재실행 + 커널 재시작** 필요. rustima에서 겪은 패턴 그대로.

#### 안정화 후
```bash
maturin build --release         # wheel 생성
pip install target/wheels/pfire_features-*.whl
```

### 4.5 Python ↔ Rust 데이터 흐름

- **numpy ndarray ↔ Rust ndarray**: zero-copy 가능 (PyReadonlyArray). 큰 격자 데이터는 반드시 zero-copy.
- **DataFrame**: pandas → numpy로 변환 후 전달 (polars-arrow 통합도 옵션이지만 복잡도 증가).
- **GIS**: GeoPandas → 좌표 array로 변환 후 전달.

---

## 5. 실험 설계 상세

### 5.1 데이터 분할

```
2017 ─── 2018 ─── 2019 ─── 2020 ─── 2021 ─── 2022 ─── 2023 ─── 2024
                                                      ↑           ↑
                                                      val         test (hold-out)
└──────────── train ──────────────────────────────┘
```

- **공간**: spatial block CV (50km block, 5-fold)
- **시간**: 2024년은 최종 hold-out
- **Calibration set**: 학습셋의 마지막 10%를 conformal calibration에 할당

### 5.2 Ablation 매트릭스

| 구성 | Layer 1 | Layer 2 단일 | Layer 2 상호작용 | Layer 3 OR | Conformal |
|------|---------|--------------|------------------|------------|-----------|
| A0 (Logistic) | - | linear only | - | - | - |
| A1 (EBM-no-base) | ✗ | ✓ | ✓ | ✗ | ✗ |
| A2 (EBM+base) | ✓ | ✓ | ✓ | ✗ | ✗ |
| A3 (EBM+base+conf) | ✓ | ✓ | ✓ | ✗ | ✓ |
| **A4 (full PF-EBM-OR)** | ✓ | ✓ | ✓ | ✓ | ✓ |

### 5.3 정성 평가

EBM 부분 의존성 그래프를 도메인 전문가(또는 산림청 가이드라인)와 대조:
- $f(\text{RH\_min})$가 단조 감소해야 함 (습도 낮을수록 위험↑)
- $f(\text{FWI})$가 단조 증가
- $f(\text{dist\_to\_tline})$가 거리 증가에 따라 감소
- $f_{12}(\text{WS}, \text{dist\_to\_tline})$가 풍속↑·거리↓일 때 최댓값

도메인 부합성 위배 시 monotonic constraint 추가 검토.

---

## 6. 마일스톤

| 주차 | 작업 | 산출물 | 책임 |
|------|------|--------|------|
| W1 | 데이터 수집·EDA | EDA 노트북, 데이터 카탈로그 | - |
| W2 | 라벨 생성, negative sampling | label 데이터셋, 양/음성 분포 분석 | - |
| W3 | Rust `pfire_features` 구현 | FWI/KBDI/distance/KDE crate v0.1 | - |
| W4 | Feature pipeline 통합 | features.parquet, feature 분포 리포트 | - |
| W5 | Layer 1 (Climatology GAM) | Rust `pfire_gam` + Python wrapper | - |
| W6 | Layer 2 (EBM) | EBM 학습·평가 결과 | - |
| W7 | Layer 3 (Conformal + OR) | Rust `pfire_conformal` + 우선순위 산출 | - |
| W8 | Ablation, baseline 비교 | 비교 테이블, 그래프 | - |
| W9 | 시각화, 해석 리포트 | 부분 의존성 시각화, 위험 지도 | - |
| W10 | 발표자료, 최종 제출 | PPT, 코드 저장소, README | - |

---

## 7. 위험 요소 및 대응

| 위험 | 발생 가능성 | 영향 | 대응 |
|------|-------------|------|------|
| KEPCO 데이터에 설비 노후도/점검이력 없음 | 높음 | 중 | voltage class + overhead/underground만으로 진행, 노후도 변수 ablation 제거 |
| 양성 사건 너무 희소 (특히 R=100m) | 높음 | 높음 | R=500m을 1차 모델로, R=100m은 보조 분석. focal loss / class weight 적용 |
| Spatial leakage로 인한 환상적 점수 | 중 | 높음 | spatial block CV 엄격 적용, Moran's I 확인 |
| Rust GAM 구현 일정 초과 | 중 | 중 | fallback으로 pyGAM 또는 mgcv-via-rpy2. `pfire_gam`은 v0.1만 우선 (univariate spline + tensor product 2D), nonlinear는 v0.2로 미룸 |
| FWI 한국 적용성 (캐나다 시스템) | 낮음 | 중 | 한국 산림청 KFWI 또는 자체 calibration |
| 콘테스트 마감 대비 일정 부족 | 중 | 높음 | 마일스톤 우선순위: 데이터 → Layer 2(EBM) → 평가 순. Layer 1·3은 시간 부족 시 단순화 |

---

## 8. 다음 행동 (Immediate Next Actions)

1. **콘테스트 데이터 정확 명세 확인** — 한전 제공 데이터의 실제 컬럼·해상도 파악. 가정 보정.
2. **산림청 산불통계 다운로드** — 라벨 데이터 확보가 모든 것의 출발점.
3. **Rust workspace 초기화** — `cargo new --lib pfire_features`, `maturin init`, hello-world 바인딩 확인.
4. **FWI Rust 구현 시작** — 가장 작고 자기완결적인 crate. 검증 데이터: 산림청 KFWI 또는 캐나다 cffdrs R 패키지 출력과 대조.
5. **EDA 노트북 v0** — 라벨 시공간 분포, 기상-화재 상관, 설비 인근 화재 비율 등.

---

## 부록 A. FWI 시스템 (Van Wagner 1987) 요약

6개 지수:
- **FFMC** (Fine Fuel Moisture Code): 표층 미세 연료 (낙엽 등)
- **DMC** (Duff Moisture Code): 중층 부식토
- **DC** (Drought Code): 심층 유기물
- **ISI** (Initial Spread Index): 초기 확산 속도 = FFMC × wind
- **BUI** (Buildup Index): 가용 연료 = f(DMC, DC)
- **FWI** (Fire Weather Index): 종합 = f(ISI, BUI)

Rust 구현 시 일별 순차 업데이트 (격자 차원만 병렬).

## 부록 B. 참고 문헌

- Wang et al. (2025). *Susceptibility assessment of wildfire-induced transmission line tripping using a physical-Bayesian modeling approach*. Sci. Rep.
- Liang et al. (2024). *A flame combustion model-based wildfire-induced tripping risk assessment*. Frontiers in Energy Research.
- Wang et al. (2021). *Wildfire risk assessment of transmission-line corridors based on naïve Bayes network*. (PMC)
- Lou et al. (2013). *Accurate intelligible models with pairwise interactions* (EBM 원 논문).
- Wood (2011). *Fast stable REML for GAMs*. JRSS-B.
- Roberts et al. (2017). *Cross-validation strategies for data with temporal, spatial, hierarchical, or phylogenetic structure*. Ecography.
- Angelopoulos & Bates (2023). *A gentle introduction to conformal prediction*. Found. Trends ML.
- Van Wagner (1987). *Development and structure of the Canadian Forest Fire Weather Index System*. Canadian Forestry Service.

---

*Document version: v0.1 (draft)*
*Next revision after: 콘테스트 데이터 실물 확인*
