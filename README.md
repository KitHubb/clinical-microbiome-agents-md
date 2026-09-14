# Clinical Microbiome AGENTS.md

임상 microbiome 논문 분석을 Codex, Claude Code 같은 코딩 에이전트한테 맡길 때 쓰려고
만든 `AGENTS.md`다. R(phyloseq + tidyverse) 기반이고, 프로젝트 루트에 이 파일만 두면
에이전트가 QC부터 decontam, 통계, 시각화까지 정해진 순서와 확인 절차를 따라간다.

## 왜 만들었나
LLM한테 R 분석을 시켜보면 매번 비슷한 문제가 반복된다. 이미 있는 패키지 함수를 두고
지저분하게 직접 짜거나, rarefaction depth나 filtering threshold를 물어보지도 않고
지가 정해버리거나, batch effect·contamination 점검을 그냥 건너뛰거나. phyloseq 
object의 문제점 (`subset_samples()`, `as.data.frame()`, 숫자로 시작하는 이름 앞에 `X` 붙는
문제)을 이해하지 못한다. 

그래서 이런 지점들을 에이전트가 알아서 판단하지 않고, 정해진 규칙을 따르거나 멈춰서
나한테 확인받도록 문서로 작성하였다

## 파이프라인

```
[1] Subject Information      [2] Sample Info → Decontam 결정      [3] Phyloseq 기초 분석
 (moonBook 기술통계)   →      (depth/batch 확인 → decontam)  →      (주효과/batch effect/confounding)
```

단계별로 `.rds`로 저장하고, 재실행할 땐 이미 있는 결과를 다시 계산하지 않는다.

## 연구자가 확인해야 할 부분은? 

에이전트가 값을 임의로 정하지 않고 멈춰서 확인받도록 정해둔 지점이 8개다.

| # | 위치 | 뭘 물어보나 |
|---|---|---|
| 1 | 문서 맨 위 원칙 | rarefaction depth·filtering threshold 확정, decontam 방법 선택, covariate 포함 여부, 기존 객체 구조 변경 |
| 2 | Subject Information | 임상변수 결측치 처리 방침 |
| 3 | Sample Info → Depth | rarefaction depth 값 (자동 산정 절대 금지) |
| 4 | Sample Info → Batch confounding table | batch가 group·institution이랑 안 갈릴 때 그걸 어떻게 다룰지 |
| 5 | Decontam decision | frequency/prevalence/combined 중 최종 선택, threshold 0.1 바꿀지 여부 |
| 6 | Gate (threshold sensitivity 실행 전) | batch balance 검토 후 batch별로 할지 pooled로 할지, 0.01~0.9 전체 sweep을 진짜 돌릴지 |
| 7 | Base phyloseq — batch effect | covariate 보정만 할지, ConQuR 같은 batch correction을 쓸지 |
| 8 | Base phyloseq — main effect | adjusted model에 어떤 confounder를 최종적으로 넣을지 |

6번 Gate가 유일하게 "무조건 멈추고 물어봐라"라고 못박아둔 자리다. threshold sweep은
계산량도 크고 결과 해석도 민감해서, 여기서만큼은 확인 없이 넘어가는 걸 막아뒀다.

## 어떤 내용을 다루는가?

- **phyloseq 객체 다룰 때 지켜야 하는 것들**: `subset_samples()` 대신 `prune_samples()`,
  `as.data.frame()` 대신 `data.frame()`, taxa가 행인지 열인지 매번 확인, 숫자로 시작하는
  이름 앞에 붙는 `X` 제거하기.
- **decontam 결정 트리**: negative control이랑 DNA 농도가 있는지 없는지에 따라 방법을
  나눠서 제안하고, 최종 선택은 내가 한다.
- **batch confounding table**: batch 레벨마다 표본 수, 그룹 수, 기관 수, 성비를 표로
  뽑고, batch가 group이나 institution이랑 분리가 안 되면 그냥 넘어가지 말고 경고하게
  해뒀다.
- **decontam threshold sensitivity**: `decontamSensitivity` 패키지로 0.01부터 0.9까지
  돌려서 alpha, beta(PERMANOVA), taxa(평균 상대풍부도 1% 이상인 Genus)를 전후 비교하고,
  결론이 뒤집히는 구간이 있으면 따로 표시하게 했다.
- **ScruB·MicrobIEM 교차검증**: 환경에 깔려 있으면 default 값으로 돌려서 메인 decontam
  결과랑 나란히 비교한다. 대체가 아니라 검증용이다.
- **단면 vs 종단**: 종단 데이터는 permutation을 SubjectID 안으로 제한하고, GDM처럼 방문
  마다 값이 안 바뀌는 변수는 종단 모형에서 검정 자체를 안 하게 해뒀다.
- **코딩 스타일**: tidyverse랑 phyloseq/microbiome/vegan 같은 도메인 패키지 우선,
  일회성 함수 만들지 말기, seed는 42로 고정, 다중검정은 BH-FDR로 고정.

## 쓰는 법

1. `AGENTS.md`를 분석 프로젝트 루트에 넣는다.
2. 분석 시작 전에 `Clinical_Microbiome_Data_Prep_Checklist.xlsx`를 채운다. 주
   비교변수, covariate, 반복측정 구조, batch/institution/sex 컬럼명, negative control
   식별 방법, decontam 기본값 같은 걸 미리 적어두면 나중에 매번 다시 안 물어봐도 된다.
3. Codex나 Claude Code한테 분석을 시키면, `AGENTS.md` 규칙대로 단계를 밟으면서 위 8개
   지점에서 확인을 요청한다.
4. 프로젝트마다 다른 것(negative control 샘플 ID, 실제 batch 변수명 등)은 `AGENTS.md`
   맨 아래 `Project-Specific Guidelines`에 추가한다. 본문은 건드리지 않는다.

## 파일

| 파일 | 내용 |
|---|---|
| `AGENTS.md` | 에이전트용 하네스 문서 |
| `Clinical_Microbiome_Data_Prep_Checklist.xlsx` | 분석 전에 채우는 사전 정보 체크리스트 |

## 참고

문서 구조(권한/자율성 구분하는 방식)는 공개된 코딩 에이전트 하네스 문서들 구성을 참고했고,
분석 내용(phyloseq 처리 규칙, decontam 결정, batch/confounding 체크)은 실제 임상
microbiome 분석하면서 겪은 문제들 기반으로 새로 썼다.
