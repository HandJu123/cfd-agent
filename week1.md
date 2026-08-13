# 발표자료

## 연구 주제

<aside>
💡

**CFD surrogate model의 정확도와 신뢰성을 높여, 설계 최적화의 병목을 해소한다.**

**→ 난류 유동장에 대한 Neural Operator 대리모델의 학습 범위 밖 일반화 성능과 신뢰성을 정량화**

</aside>

### 연구 배경

- 설계 최적화를 진행하기 위해서는 형상 및 조건을 바꿔가며 수백~수천 번의 반복 평가가 필요한데, full CFD로는 **계산 비용이 매우 높고 시간적 제약이 큼.**
- 그래서 최근 산업에서도 **탐색 단계는 surrogate model로, 최종 검증만 CFD**라는 하이브리드 구조를 지향하고 있음.
- 문제는 Optimizer가 surrogate model의 학습 범위 밖을 파고들게 된다면 surrogate 모델의 평균 정확도가 높아도 정확하지 않은 최적점으로 수렴할 수 있음.

![image.png](image.png)

- 따라서 이를 해결하기 위해서는 ① surrogate가 예측과 함께 **불확실성**을 내놓고, ② Optimizer가 불확실성이 큰 영역으로는 함부로 안 나가게 제한하고(trust region), ③ 나가야 한다면 그 지점에서 **실제 CFD를 호출해 학습 데이터를 보강**하는 것(active learning)이 필요함.
- 즉, surrogate model의 **병목은 정확도가 아니라 신뢰성 문제라고 판단.**

**접근 아키텍처로 Neural Operator를 채택하는 이유**

- 설계 최적화용 surrogate는 **(a) 유동장 전체를 출력하고, (b) 조건이 바뀌어도 재학습 없이 대응**해야 함.
- 기존 Gaussian Process 기반 모델은 스칼라 출력에 한정되고, PINN은 케이스별 재학습이 필요해 반복 평가 루프에 부적합함.
- Neural Operator는 함수→함수 매핑을 학습하여 **두 요건을 동시에 만족하는 구조**임.
- 다만 NO가 내세우는 일반화 성능은 **학습 분포 밖에서 어디까지 유지되는지 정량화되어 있지 않으며**, 예측에 대한 불확실성 정보를 제공하지 않음.

### 선행 연구 조사

**Using Uncertainty Quantification to Characterize and Improve Out-of-Domain Learning for PDEs**https://arxiv.org/pdf/2403.10642

① 문제
****NO는 in-domain에서는 해를 잘 근사하지만, **PDE 파라미터가 학습 범위를 조금만 벗어나도 예측이 무너진다.** 기존 UQ 방법들이 in-domain에서는 잘 보정된 불확실성을 주는데, **OOD에서는 오차가 커져도 불확실성이 따라 커지지 않는다.**

③ 핵심 아이디어

- **EnsembleNO**
개별 모델이 서로 다른 posterior mode에 도달해 functional diversity를 가짐.
in-domain 입력에서는 마지막 층이 그 다양성을 없애고 모두 정답으로 수렴시키는데, OOD 입력에서는 다양성이 출력까지 살아남고 이렇게 남아있는 다양성이 epistemic uncertainty가 됨.
- **DiverseNO**
마지막 feed-forward 층만 M개(=10)의 예측 head로 나누고, 그 앞의 파라미터는 공유. 이런 방식을 사용하면 다양성이 감소하기에, head 간 가중치 L2 거리를 최대화하는 다양성 정규화 항을 손실에 추가.
head 가중치들이 서로 멀어지도록 최대화하며 단일 모델로 앙상블의 UQ 성질을 흉내내면서 비용은 크게 절감, in domain의 정확도는 조금 포기해도 OOD 불확실성의 품질은 개선하는 trade-off.
- **Operator-ProbConserv**
이렇게 얻은 오차-상관 불확실성을 기존 ProbConserv 프레임워크에 넣어, 분산이 큰 영역을 더 크게 보정하면서 보존법칙을 만족시킴.

② 기존 방법의 한계
BayesianNO, VarianceNO, MC-DropoutNO 셋 다 **OOD에서 불확실성과 실제 오차의 상관이 무너짐**. EnsembleNO(FNO 10개 독립 학습)는 잘 작동하지만 **학습/추론 비용이 10배**라, NO를 쓰는 이유가 약해짐

④ 검증과 한계

- **대상:** GPME 계열(heat=easy / PME=medium / Stefan=hard) + 선형 이류 + 2D Darcy. N=400 샘플, FNO 모델 사용
- **OOD 정의:** PDE 계수 범위를 small/medium/large로 이동 (ex: heat는 k_train∈[1,5] → k_test∈[5,6], [6,7], [7,8])
- **핵심 지표 n-MeRCI:** 불확실성이 실제 오차와 얼마나 상관되는지 측정, 0에 가까울수록 좋음
- **결과:** DiverseNO가 저비용 UQ 대비 n-MeRCI 2~70배 개선, 비싼 EnsembleNO와 대등하거나 더 나음.Operator-ProbConserv로 OOD 정확도 최대 34% 향상.
- **저자가 인정한 한계:** medium/hard PDE의 **large OOD shift에서는 여전히 오차가 큼(~10⁻²)**. 전역 보존 제약만으로는 부족하고 국소 물리 제약이 추가로 필요하다고 명시.

Mouli et al. 2024가 이 문제를 다뤘으나, 대상이 1D 스칼라 파라미터 PDE고 난류에 대한 내용은 다루지 않았음. 따라서 난류에서는 이게 어떻게 되는지 확인해보고 싶음.

Uncertainty quantification and stability of neural operators for prediction of three-dimensional turbulence

https://www.sciencedirect.com/science/article/pii/S0021999125009210

### 연구 목표

난류 유동장에 대한 NO 대리모델의 학습 범위 밖 일반화 한계를 정량화하고, 예측 불확실성 추정을 통해 신뢰 구역을 판별하는 방향으로 고민중임.

위와 같은 난류에서 이러한 문제를 다룬 논문들을 간단하게 확인해보니 현실적으로 감당 가능한 규모인지 판단이 서지 않는데, 실질적으로 핸들할 수 있는 범위?

스스로 생각했을 때 할 수 있는 가능성이 있는 과제

① 랩 기존 NO 코드로 작은 결과 재현하기

② Re나 rollout 길이 하나를 축으로 잡아 학습 범위 밖 성능 저하 곡선 뽑기

③ 모델 몇 개 앙상블해서 예측 분산이 실제 오차와 상관되는지 보기

[JFM](https://app.notion.com/p/JFM-39ed6111759680ffb650c04bd91f8e3b?pvs=21)

[NeurIPS ML4PS](https://app.notion.com/p/NeurIPS-ML4PS-39ed611175968083bf65d91537aa0a18?pvs=21)