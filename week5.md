# 0910 미팅

# Part 1. 물리적 타당성 검증 방법론 구상

## 1. 기존 연구의 공백

### 기존 CFD 에이전트 연구의 공백

최근 2년간 LLM 기반 CFD 자동화 연구를 보면, 대부분 결과로서 보고하는 지표가 **실행 성공률**에 그침

| 연구 | 보고 지표 | 물리적 검증 |
| --- | --- | --- |
| MetaOpenFOAM (arXiv 2407.21320) | 에러 반복 교정 성공 | 없음 |
| OpenFOAMGPT (2501.06327) | 반복 교정 루프 수렴 | 없음 |
| ChatCFD (2506.02019) | 에러 없는 설정 30~40%, 운영 성공 60~80% | 없음 |
| NL2FOAM 파인튜닝 (2504.09602) | solution accuracy 88.7%, first-attempt 82.6% | 없음 |
| Foam-Agent (2509.18178) | Reviewer 에이전트 반복 디버깅 | 없음 |
| OpenFOAMGPT 2.0 (2504.19338) | 동일 케이스 10회 반복 실행의 일관성 | 재현성 ≠ 타당성 |
| TurboAgent (2604.06747) | 검증 에이전트 분리, 수렴 이력 실시간 모니터링 | 수렴 모니터링 수준 |

Agent를 이용한 CFD의 **실행 성공**과 **물리적 타당성**은 별개의 문제

**→ 물리적 타당성을 자동적으로 검증하고, 그 판정 결과를 최적화 루프에 반영**

---

## 2. 다른 분야에서의 물리적 타당성 검증 방법론

### 아날로그 회로 LLM 에이전트 - 검증을 여러 단계로 나누어 수행

Lai et al., *AnalogCoder: Analog Circuit Design via Training-Free Code Generation*, AAAI 2025

AnalogCoder(AAAI 2025)는 자연어 설명을 PySpice 라이브러리 기반 실행 가능한 Python 코드로 변환해 아날로그 회로를 설계함

**어떻게 검증했나:** SPICE 시뮬레이션 실행 중 발생하는 런타임 에러 외에, **설계의 정확성을 보장하려면 회로 관련 정보에 대한 추가 검증**이 필요하다는 점을 지적**,** 피드백 흐름을 4단계로 구성

1. **Requirement check** - 필요한 입력, 출력, 회로 구성 요소가 포함되어 있는지 확인
2. **Simulation and operating point check** - 시뮬레이션이 실제로 실행되는지, 동작점이 정상인지 확인
3. **DC sweep check** - 입력 변화에 대해 회로가 정상적인 거동을 보이는지 확인
4. **Function check** - 최종적으로 의도한 기능을 실제로 수행하는지 확인

그리고 설계가 실패하면 **런타임 에러인지 회로 특유의 테스트 에러인지를 구분해서** LLM에 되돌려줌

### Principle 1. Validation을 계층화

따라서 CFD 검증도 한 번에 pass/fail을 판정하기보다는 단계별로 나누는 것이 적절해 보임

예를 들면,

L1. Geometry / Input Valid

L2. Solver Runs

L3. Numerically Converged

L4. Physically Valid

L5. Performance Valid

와 같은 구조로 구성하고, 어디에서 왜 실패했는가를 피드백으로 전달

---

### 단백질 설계: **생성 결과를 다시 다른 모델을 이용해 확인**

단백질 생성 모델은 새로운 3차원 구조를 생성할 수 있지만, 모델이 어떤 구조를 생성했다는 사실만으로 실제로 그 구조를 갖는 단백질을 만들 수 있다는 보장은 없기에 단백질 설계 분야에서는 **생성 결과를 다시 다른 모델을 이용해 확인하는 self-consistency 방식**이 사용됨

```
Generated Backbone
       ↓
ProteinMPNN
       ↓
해당 backbone에 맞는 amino-acid sequence 여러 개 생성
       ↓
AlphaFold2 / ESMFold
       ↓
각 sequence의 3D 구조를 다시 예측
       ↓
Original Design ↔ Predicted Structure 비교
```

즉,

```
Structure A
   ↓
Sequence
   ↓
Structure A'
```

에서 **A ≈ A' 인가?** 를 확인하며, 처음 생성한 구조와 재예측된 구조가 비슷하다면, 서로 다른 방법을 거쳤음에도 동일한 구조가 재현되었다는 의미이므로 해당 설계의 신뢰도가 높다고 해석

Watson et al., *De novo design of protein structure and function with RFdiffusion*, Nature (2023).

**대표적인 정량 기준**

| Metric | Example threshold | 의미 |
| --- | --- | --- |
| scRMSD | `< 2 Å` | 원래 설계와 재예측 3D 구조의 거리 차이 |
| scTM | `> 0.5` | 전체적인 3D 구조의 유사성 |
| pLDDT | `> 70` 등 | 구조 예측 모델 자체의 confidence - 각각의 계산 자체가 제대로 수렴 |
| pAE | `< 5` | 예측된 구조 관계의 uncertainty -  |

### Principle 2. 생성기와 검증기를 분리

Optimization에 사용한 CFD 설정
↓
다른 mesh / scheme / solver를 이용한 recomputation

같은 계산 환경에서 한 번 더 돌리는 것이 아니라,
numerical assumption을 일부 바꿨을 때도 결과가 유지되는지를 확인

---

### A-Lab: 검증기 자체의 신뢰성 문제

A-Lab은 무기재료 합성을 자동화한 autonomous laboratory이며, 전체 흐름은 다음과 같음.

```
Target Material Selection
        ↓
ML-based Synthesis Recipe
        ↓
Robotic Experiment
        ↓
XRD Measurement
        ↓
Automated Phase Identification
+ Rietveld Refinement
        ↓
Success / Failure
        ↓
Next Experiment
```

Nature 2023 논문에서는 17일 동안 57개 target을 실험하고 그중 36개 합성에 성공했다고 보고함

Szymanski et al., *An autonomous laboratory for the accelerated synthesis of inorganic materials*, Nature 624, 86–91 (2023)

실험 결과를 “성공”이라고 판단하는 검증 단계까지 자동화했다는 점에서 후속 연구에서는 A-Lab의 XRD 해석과 자동 phase identification 과정에 문제가 있을 수 있다고 지적, Nature Author Correction에서 저자들이 diffraction pattern을 수동으로 재분석하여 기존 성공 판정 대부분은 유지되었지만 일부 target은 XRD만으로 결론을 내릴 수 없어 성공 목록에서 제외.

### Principle 3. Validator 자체도 검증

Generator가 틀릴 수 있음 + Validator도 틀릴 수 있음

따라서 CFD validation framework를 제안하려면, validation gate의 sensitivity / specificity 자체를 별도 benchmark로 평가할 필요가 있음

```
AI / Optimizer
     ↓
Simulation
     ↓
Validation Gate
     ↓
Trustworthy
```

의도적으로 다음 CFD case를 만들 수 있습니다.

```
Case A — 정상적으로 수렴한 simulation
Case B — insufficient mesh
Case C — residual이 충분히 감소하지 않은 simulation
Case D — mass conservation error가 큰 simulation
Case E — 비정상적인 flow field
```

그리고 validation gate가 각각을 올바르게 판정하는지 평가하여, 검증 규칙 자체의 검출 성능도 검증

## 3. 제안하는 CFD Validation Framework

```
AI Generated Geometry
            ↓
────────────────────────
GATE 1. Executability
────────────────────────
• Meshing success
• Solver initialization
• Solver completion

            ↓

────────────────────────
GATE 2. Numerical Reliability
────────────────────────
• Residual convergence
• Mass conservation
• Energy conservation

            ↓

────────────────────────
GATE 3. Self-Consistency
────────────────────────
Same geometry recomputed with:
• different mesh density
• different discretization scheme
• possibly different solver

Check consistency of:
• hotspot temperature
• pressure drop
• other QoIs

            ↓

────────────────────────
GATE 4. Physical Validity
────────────────────────
• Boundary-condition consistency
• Sign / magnitude checks
• Known physical constraints
• abnormal field detection

            ↓

      TRUSTWORTHY DESIGN

            ↓

────────────────────────
Performance Evaluation
────────────────────────
Only trustworthy designs are
used for optimization comparison
```

# Part 2. 베이지안 최적화와 검증 정보의 피드백

## 1. 왜 Bayesian Optimization인가

이번 연구에서 최적화 방법으로 Bayesian Optimization(BO)을 고려하는 가장 큰 이유는 CFD 평가 비용이 크기 때문임. 설계변수가 3~5개 정도이고, 하나의 설계에 대해 CFD를 수행하는 데 수분이 걸린다고 하면 모든 조합을 직접 탐색하기 어려움.

유전 알고리즘과 같은 population-based optimization도 많은 평가 횟수를 필요로 하기 때문에, CFD처럼 함수 평가가 비싼 문제에서는 계산 비용이 크게 증가하며, BO는 이런 상황에서 가능한 한 적은 함수 평가로 좋은 설계를 찾는 것을 목적으로 하는 방식임.

---

## 2. Bayesian Optimization의 기본 원리

설계변수 x와 hotspot temperature 사이의 관계 (x → T_max) 를 정확한 식으로 표현하기 어렵고, CFD를 직접 수행해야만 결과를 알 수 있는 함수 = expensive black-box function

BO는 지금까지 수행한 CFD 결과를 이용해 이 함수를 근사하는 surrogate model을 만들며, 대표적으로 Gaussian Process(GP)를 사용할 수 있음

**CFD 데이터 → GP 업데이트 → 다음 후보 선택 → CFD 실행 → GP 업데이트**

전체 과정 설명

1. 초기 설계점 몇 개를 선택하고 CFD를 수행
2. 얻어진 (설계변수, CFD 결과) 데이터를 이용해 GP를 학습
3. GP가 아직 계산하지 않은 영역의 성능을 예측
4. Acquisition function을 이용하여 다음으로 계산할 설계를 선택
5. 새로운 CFD 결과를 데이터에 추가하고 같은 과정을 반복

GP를 사용하는 이유는 단순히 목적함수의 예측값만 제공하는 것이 아니라, 예측의 불확실성도 함께 제공하기 때문이며 BO는 이 두 방향을 함께 고려함

- **Exploitation:** 현재까지 좋은 성능이 예상되는 영역을 더 탐색
- **Exploration:** 아직 잘 모르지만 좋은 해가 존재할 가능성이 있는 영역을 탐색

Expected Improvement(EI)와 같은 acquisition function은 이 두 요소를 동시에 고려하여 다음 CFD 평가 지점을 결정

---

## 3. Part 1과 연결되는 문제

일반적인 BO에서는 CFD가 결과값을 반환하면 그 값을 그대로 surrogate model에 추가 → BO가 새로운 형상을 제안하고 CFD 결과가 T_max = 65°C로 나오면, BO는 이 값을 좋은 성능 데이터로 받아들임.

하지만 Part 1에서 확인한 것처럼 CFD가 정상적으로 종료되었다고 해서 모든 계산 결과가 신뢰할 수 있는 것은 아님.

예를 들어 65°C라는 결과가 나왔지만,

- mesh quality가 좋지 않거나
- residual이 충분히 수렴하지 않았거나
- mass 또는 energy conservation error가 크거나
- mesh를 변경했을 때 결과가 크게 달라진다면

65°C라는 수치를 그대로 objective data로 사용하는 것은 문제가 될 수 있음. BO는 이 값을 실제 성능 개선으로 학습하게 되고, 이후 비슷한 설계 영역을 반복해서 탐색할 수 있기 때문에 Part 1의 validation 결과를 BO에 다시 반영하는 구조가 필요함.

---

## 4. Constrained Bayesian Optimization

목적함수 모델과 별도로 제약 조건을 만족할 확률을 모델링하며, 두 가지 질문을 따로 학습

1. 이 설계의 성능은 얼마나 좋은가? - hotspot temperature를 예측하는 GP가 이에 해당
2. 이 설계에서 신뢰할 수 있는 CFD 결과를 얻을 가능성이 얼마나 높은가? - probability of feasibility

**Acquisition = Expected Improvement × Probability of Feasibility**

즉, 단순히 성능이 좋아 보이는 설계를 선택하는 것이 아니라, 성능이 좋을 가능성이 높으면서 동시에 신뢰할 수 있는 CFD 평가가 가능할 가능성이 높은 설계를 우선적으로 선택

즉, 기존 BO가 어디가 성능이 좋을 것인가? 를 중심으로 판단한다면,

constrained BO에서는 어디가 성능도 좋고 정상적인 평가도 가능할 것인가? 를 함께 고려함.

---

## 5. 제안하는 구조

먼저 BO가 새로운 형상을 제안

그 형상에 대해 CFD를 수행

이후 Part 1에서 정의한 validation gate를 적용

### Validation을 통과한 경우

해당 CFD 결과를 trustworthy한 결과로 간주하고,

- hotspot temperature
- pressure drop

등의 성능 값을 objective model에 추가

### Validation에 실패한 경우

해당 성능값을 그대로 objective model에 넣지 않고,

- mesh failure
- divergence
- conservation failure
- self-consistency failure
- physical validity failure

등의 정보를 constraint model에 추가하여 이후 BO는 두 종류의 정보를 모두 이용해 다음 설계를 결정

즉,

`설계 제안 → CFD → Validation → 성능 또는 실패 정보 학습 → 다음 설계 제안`

의 closed-loop 구조를 제안

---

---

> CFD simulation의 hierarchical validation에서 발생하는 heterogeneous failure modes를 구조적으로 분류하고, 이 정보를 constrained Bayesian optimization에 피드백하여 신뢰할 수 있는 설계 영역을 효율적으로 탐색한다.
> 

---

BO가 무엇인지에 대한 구체적으로, 실제 수식적으로 정확한 정의가 무엇인지 알아야함

그래야지 현재의 변수를 가지고 BO를 학습할 수 있나?를 확인할 수 있음

목표 T_max값 → BO를 통해 학습하여 → 설계변수 x를 얻어내고 → agentic CFD를 돌려서 → T_max를 얻어내어 역설계 검증

→ 실제 역설계 학습 모델이 ‘빠르다’를 넘어, 그 결과의 타당성을 보여줄 수 있는 구조

이 방식이 지금까지 해온 agentic CFD + 설계 최적화 BO를 모두 사용 가능

단, 이 구조는

BO가 학습을 할 수 있어야 한다

agent CFD가 Tmax 결과를 실행할 수 있어야 한다

는 2개의 전제가 가정되어야함
