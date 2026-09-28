# 임상 마이크로바이옴 분석 에이전트 지침

임상 마이크로바이옴 연구에서 Codex, Claude Code 같은 코딩 에이전트가 분석 기준을 임의로 바꾸지 않고, 정해진 절차에 따라 R 분석과 HTML 보고서 작성을 수행하도록 만든 개인용 지침 모음이다.

공통 원칙과 분석 유형별 규칙을 분리했다. Amplicon 분석에는 공통 지침과 `amplicon/AGENTS.md`를 함께 적용한다. Shotgun은 `microeco` 기반 core downstream 분석 지침을 별도로 적용한다. 이 문서는 read preprocessing, assembly, MAG reconstruction 등 shotgun 전체 과정을 포괄하지 않는다.

## 문서 구성

| 파일 | 용도 |
|---|---|
| [AGENTS.md](AGENTS.md) | 임상정보, 연구설계, batch 점검, 코딩, 검증 및 캐싱에 관한 공통 지침 |
| [amplicon/AGENTS.md](amplicon/AGENTS.md) | `phyloseq` 기반 amplicon QC, decontam, diversity 및 통계 분석 지침 |
| [shotgun/CORE_DOWNSTREAM_ANALYSIS.md](shotgun/CORE_DOWNSTREAM_ANALYSIS.md) | `microeco` 기반 shotgun core downstream 분석 지침 |
| [HTML_REPORT_AGENTS.md](HTML_REPORT_AGENTS.md) | 한국어 HTML 보고서의 문체, 구성, 표·그림 표시, 렌더링 및 최종 검증 지침 |
| `Clinical_Microbiome_Data_Prep_Checklist.xlsx` | 분석 전에 연구설계와 변수 정보를 정리하는 체크리스트 |

## 만든 이유

에이전트에게 R 분석을 맡기면 다음 문제가 반복되기 쉽다.

- 기존 패키지 함수 대신 불필요하게 복잡한 코드를 작성함
- rarefaction depth나 filtering threshold를 임의로 선택함
- batch effect, contamination 및 confounding 점검을 생략함
- subject 수와 sample 수를 구분하지 않음
- `phyloseq` 객체의 orientation, NSE 및 이름 변환 문제를 놓침
- 보고서 편집 중 분석 방법이나 수치를 의도치 않게 변경함
- HTML 렌더링 성공 여부만 확인하고 실제 표시 상태를 검사하지 않음

이 문서들은 에이전트가 임의로 결정해도 되는 작업과 연구자 확인이 필요한 결정을 구분하고, 분석 결과가 본문·표·그림·결론에서 일관되게 유지되도록 하기 위해 작성했다.

## Amplicon 분석 흐름

```text
[1] Subject Information      [2] Sample Info → Decontam 결정      [3] Phyloseq 기초 분석
    moonBook 기술통계       →     depth·batch 점검 → decontam    →     주효과·batch effect·confounding
```

각 단계의 결과는 `.rds`로 저장한다. 재분석할 때는 검증된 이전 단계의 산출물을 재사용하고, 변경된 단계부터 다시 계산한다.

## 연구자 확인이 필요한 주요 결정

에이전트는 다음 항목을 실제 데이터와 선택지의 장단점 없이 임의로 확정하지 않는다.

1. Rarefaction depth와 abundance-filtering threshold
2. 임상변수 결측치 처리 방법
3. Negative control 및 DNA 정량값 유무에 따른 decontam 방법
4. Batch별 decontam과 pooled decontam 중 선택
5. Decontam threshold 변경 및 전체 sensitivity sweep 실행 여부
6. Batch와 group 또는 institution이 분리되지 않을 때의 처리
7. Adjusted model에 포함할 covariate
8. Batch correction 적용 여부

특히 decontam threshold sensitivity를 실행하기 전에는 batch balance 검토 결과와 분석 범위를 먼저 확인한다.

## Amplicon 지침의 핵심 내용

- `subset_samples()` 대신 `prune_samples()` 사용
- `otu_table()`의 taxa orientation 확인
- S4 객체를 `as.data.frame()`으로 변환하지 않음
- 숫자로 시작하는 이름에 붙는 `X` prefix 복원
- Negative control과 DNA 농도에 따른 decontam 방법 결정
- Batch별 sample 수, 비교군 수, institution 수 및 성비 확인
- PERMANOVA와 `betadisper()`를 함께 검토
- Cross-sectional 분석과 longitudinal 분석 구분
- 반복측정 분석에서 SubjectID 내 permutation 제한
- `set.seed(42)`와 BH-FDR 적용
- 분석 단계별 sample·subject·taxa 수와 전후 변화 검증

## Shotgun core downstream 지침의 핵심 내용

- `microeco::microtable`을 기본 taxonomic analysis 객체로 사용
- Kraken2 report와 Bracken abundance estimate를 구분
- MetaPhlAn 4 relative abundance를 ASV·OTU·read count로 취급하지 않음
- HUMAnN gene family, pathway abundance 및 pathway coverage를 별도 자료로 유지
- Unstratified total과 taxon-stratified contribution의 중복 합산 방지
- 실제 read count 또는 QC 지표만 sequencing depth로 사용
- Count, relative abundance, transformed abundance 및 coverage 단위 구분
- Independent, paired 및 repeated design에 맞는 통계와 permutation 적용
- Taxonomic profile과 functional profile을 별도로 분석한 뒤 필요한 경우 연결
- Differential abundance와 profiling-tool comparison은 분석 계획에 포함된 경우에만 수행
- 분석 산출물의 내용·단위는 Shotgun 지침에서, 문체·표 UI·theme·font·layout은 HTML 보고서 지침에서 관리
## 사용 방법

1. 분석 프로젝트의 루트에 공통 `AGENTS.md`를 둔다.
2. `amplicon/AGENTS.md`, `shotgun/CORE_DOWNSTREAM_ANALYSIS.md` 및 `HTML_REPORT_AGENTS.md`의 상대 경로를 유지한다.
3. 분석 전에 `Clinical_Microbiome_Data_Prep_Checklist.xlsx`에 주요 비교변수, covariate, 반복측정 구조, batch 변수, institution, sex, negative control 식별 방법 등을 기록한다.
4. Amplicon 분석에는 공통 지침과 amplicon 지침을, shotgun core downstream 분석에는 공통 지침과 shotgun 지침을 함께 적용한다.
5. HTML 보고서 작성 또는 수정에는 분석 지침과 HTML 보고서 지침을 함께 적용한다.
6. 프로젝트별 예외는 해당 지침 하단의 `Project-Specific Guidelines`에 추가한다.

## 현재 상태

- 공통 임상 마이크로바이옴 분석 지침: 작성됨
- Amplicon 분석 지침: 작성됨
- Shotgun core downstream 분석 지침: 작성됨
- 한국어 HTML 보고서 작성 지침: 작성됨

분석 규칙은 실제 임상 마이크로바이옴 연구에서 반복적으로 발생한 QC, batch, contamination, confounding 및 재현성 문제를 바탕으로 정리했다.
