# 2차 미팅

## <연구 후보 제시>

## 1. 열관리 시스템 최적화를 위한 CFD Agentic Workflow

> **"열유체/냉각 해석에 특화된 Closed-loop 자율 최적화 파이프라인 구축"**
> 

### 1.1. 추진 배경 및 목표

- **배경:** 범용 CFD Agent는 열 경계조건, 복잡한 물리 솔버(`buoyantSimpleFoam`, `chtMultiRegionFoam`), 3D 형상 파라미터 제어 등 도메인 특화 영역에서 성능 저하 및 수렴 실패율이 높음.
- **목표:** 열관리 시스템(전자부품 냉각, 배터리 열관리, 열교환기 등)을 타겟으로, '형상/경계조건 파라미터화 → OpenFOAM 열유체 해석 → 결과 분석(최대 온도, Pressure Drop, 무차원 수 등) → 격자 수렴성 검증(GCI) → 파라미터 자율 재설정'의 Closed-loop 구동.

### 1.2. 핵심 Workflow 파이프라인

1. **Geometry & Meshing Agent:** Target 시스템(히트싱크 핀 배열, 배터리 냉각 채널, 열교환기 튜브 등)의 주요 치수를 파라미터화하여 메시 생성
2. **Thermal-CFD Solver Agent:** 열 경계조건(발열량, 대류열전달계수 등) 및 특화 솔버 옵션을 자율 설정하고 실행
3. **Evaluation & Grid Validation Agent:**
    - GCI(Grid Convergence Index)를 자율 계산하여 메시 정밀도 보장
    - 최고 온도, 온도 균일성, 압력 강하(Pressure Drop) 데이터를 추출 및 평가
4. **Optimization Loop:** 평가 결과를 바탕으로 다음 형상 파라미터 및 격자 밀도를 결정하는 Self-Correction / Optimization 루프 구동

### 1.3. 주요 타겟 응용 분야

- **전자부품/칩셋 냉각 :** 핀 두께, 간격, 배열에 따른 열저항 및 압력강하 최적화
- **전기차 배터리 열관리 :** 배터리 모듈/팩 Cold Plate의 유로 구조 최적화를 통한 온도 균일성 확보
- **열교환기 튜브/핀 구조 :** 열전달 효율 극대화 및 유동 마찰 손실 최소화

### 1.4. 학술적 차별성

- '열유체/열관리 분야'에 특화된 Agentic Workflow 구축
- 단순히 메시만 만드는 수준을 넘어, **GCI 계산 모듈을 Evaluation 함수로 내재화**하여 메시 품질과 해석 결과의 물리적 타당성을 스스로 검증
- 동일한 Framework 구조 내에서 파라미터화 모듈 교체만으로 다양한 열관리 대상(히트싱크, 배터리 냉각판 등)에 유연하게 적용 가능

## 2. OpenFOAM 버전 간 Dictionary 자율 변환 및 에러 복구 Agent

> **"OpenFOAM 버전 파편화 문제를 해결하는 Transpiler & Self-Correction Agent"**
> 

### 2.1. 추진 배경 및 목표

- **배경:** OpenFOAM은 버전별(ESI v2312 vs. Foundation v11 등) Dictionary 문법, 파라미터 키워드, 파일 구조가 달라 구버전 케이스 실행 시 `Fatal Error`가 발생하며 파이프라인이 정지됨
- **목표:** LLM의 코드 리팩토링 및 문맥 이해 능력을 활용해, **구버전 OpenFOAM Case를 목표 버전 표준 사양으로 자율 변환하고 실행 에러를 자율 복구하는 Framework** 구현

### 2.2. 핵심 Workflow 파이프라인

1. **Dictionary Parser & Transpiler:** Source 버전의 `system/` 및 `constant/` 파일 문법을 분석하여 Target 버전 규격에 맞게 자동 리팩토링
2. **Execution & Log Parsing:** CLI 환경에서 `blockMesh` 및 솔버를 구동하고 Terminal 출력 로그 파싱
3. **Self-Correction Loop:**
    - 문법 오류(`Fatal Error`) 발생 시, Terminal 로그에 출력되는 `Valid keywords are: [ ... ]` 정보를 Agent가 해석
    - 올바른 키워드로 Dictionary를 자동 수정하고 재실행하는 Closed-loop 제어

### 2.3. 학술적 차별성

- 기존 정적 변환 스크립트의 한계를 넘어, LLM 기반 **Transpilation + Self-Correction** 루프를 결합하여 높은 실행 성공률 달성
- OpenFOAM 공식 튜토리얼 케이스를 버전별로 구동해보며 '버전 변환 성공률', 'Self-Correction 시도 횟수' 등의 정량적 Evaluation Metric 제시 가능

## <데이터셋 정리>

| 프레임워크 | 데이터셋 규모 | 입력 | 출력 | 특징 |
| --- | --- | --- | --- | --- |
| **MetaOpenFOAM** | **OpenFOAM 튜토리얼 케이스** 중심 | 자연어(`usr_requirement`) | OpenFOAM
케이스 폴더 | YAML 설정에 `case_name`, `max_loop`, `temperature` 등 하이퍼파라미터 |
| **NL2FOAM** (FoamGPT의 파인튜닝용 데이터셋) | **OPENFOAM 케이스 16개 → 28,716쌍** | 자연어 문제 설명 | OpenFOAM 설정 + CoT(chain-of-thought) 주석 | 16개 검증된 원본 OpenFOAM 케이스에서 수치해석 설정 파일을 바꿔가고, 명령의 방식을 바꿔가며 28,716쌍 생성 |
| **OpenFOAMGPT** | 명시적 벤치마크 수 불명, **6개 대표 케이스로 시연** | 자연어 (zero-shot/few-shot) | OpenFOAM 설정 파일 | Cavity, PitzDaily, Hotroom(자연대류), Dambreak, Particle column, Mixed vessel. 자체 벤치마크 논문은 없고 시연 위주 |
| **ChatCFD** | **OpenFOAM 기본 테스트 315개 케이스** | 자연어 + **PDF 논문 업로드 + 메시 파일** | OpenFOAM 케이스 | 실행 성공률(82%)/물리적 타당성(59%)
논문 PDF에서 케이스를 추출하는 독특한 입력 방식 |
| **Foam-Agent / Foam-Agent 2.0** | **110개 케이스, 11개 물리 카테고리** | 자연어 (문제 설명 + 물리 시나리오 + 형상 + 솔버 요구사항 + 경계조건 + 파라미터) | OpenFOAM 케이스 전체(메시+설정) + 시각화 | 단순 형상~복잡 3D까지, forwardStep, CounterFlowFlame, wedge 등 |
| **SwarmFoam** | **25개 테스트 케이스** (자연어 10 + 멀티모달 15) | 자연어 또는 이미지+자연어 | OpenFOAM 케이스 | 단상/이상 유동, MHD, 연소까지 포함. solver별 세분화(simpleFoam 1, pisoFoam 2, reactingFoam 2 등) |

### 공통 패턴 및 시사점

1. 입출력 포맷은 자연어 “문제 기술 → OpenFOAM 케이스 폴더”로 통일
2. 규모는 20~200개 사이가 표준
3. 열유체 케이스는 다들 "일부"로만 들어있음 → MetaOpenFOAM의 buoyantCavity(자연대류), OpenFOAMGPT의 Hotroom이 열 관련인데, 열유체를 중심축으로 삼은 데이터셋은 없음.
4. 튜토리얼 근처에서 벗어나지 못한다는 게 공통 한계
5. 데이터셋을 만들 때 참고할 최소 규모: SwarmFoam(25개)를 참고하면, 20~30개 열유체 케이스 또는 파라미터를 변경해감으로서 데이터셋 개수 증가하여 자연어-설정 쌍과 같은 식으로 구성
6. Foam-Agent의 110개 데이터셋을 기반 삼아, 그중 열 관련 케이스만 추출해서 시작하는 것도 방법