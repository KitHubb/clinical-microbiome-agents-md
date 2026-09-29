# AGENTS.md — 마이크로바이옴 HTML 보고서 작성 지침

기존 마이크로바이옴 분석 결과를 바탕으로 논문·연구용 HTML 보고서를 작성하거나 수정할 때 이 문서를 적용한다. 프로젝트 루트의 [AGENTS.md](AGENTS.md)와 해당 assay의 하위 지침도 함께 따른다.

# 목적과 적용 범위

- 분석 방법, 결과, 한계점을 간결하고 정확하게 전달한다.
- 보고서 편집 과정에서 승인된 분석 방법, 입력 데이터, 샘플 포함 기준 및 통계 결과를 변경하지 않는다.
- R Markdown(`.Rmd`) 수정은 공통 지침의 backup 및 부분 patch 절차를
  따르고, 수정 구간 검증 후 최종 HTML을 다시 렌더링하여 실제 표시
  상태까지 확인한다.
- 임시 render 산출물과 중간 분석 파일은 보관하지 않고 요청된 최종
  HTML 및 최종 그림·표만 저장한다.
- 기존 결과 폴더 아래에 추가 하위 폴더를 만들지 않는다.
- 명시적인 요청 없이 기존 분석 결과나 다른 보고서를 삭제하지 않는다.

# 변경 권한과 분석 보존

보고서 편집은 filtering, normalization, 통계 모형, covariate, reference level, 샘플 포함 여부 등 분석 결정을 변경할 권한을 포함하지 않는다. 보고서 작성 중 수치 불일치나 분석 오류로 의심되는 사항을 발견하면 그 내용을 먼저 제시하고, 기초 분석을 변경하기 전에 사용자 확인을 받는다.

요청된 부분만 수정한다. 사용자가 삭제를 요청한 분석이나 설명을 다시 삽입하지 않는다. 승인된 분석 내용이 변경된 경우에는 본문, 표, 그림, 결론 및 한계점에 포함된 관련 수치와 해석을 모두 함께 갱신한다.

# 보고서 문체

보고서 본문은 간결한 한국어 학술 보고서 문체로 작성한다.

- Microbiome 및 metagenome 분석 전문 용어는 영문으로 유지한다.
- 통계 용어와 표 column 이름은 영어로 유지한다.
- 분석 방법, 결론 및 한계점은 개조식으로 작성한다.
- 한국어 문장은 명사형 또는 `~함`, `~됨`으로 종결한다.
- 장문의 `~입니다`, `~하였습니다` 서술을 지양한다.
- 요약 앞에 `**요약:**`과 같은 표식을 붙이지 않는다.
- 구현 과정, 사용법 및 자명한 화면 기능에 대한 설명을 생략한다.
- 확인이 필요한 사항과 해석상 제약은 “6. 한계점 및 향후 분석 예정”에 통합한다.


다음과 같은 구체적인 표현이나 설명을 작성하지 않는다.

- “세 지수를 함께 제시했습니다”
- “표 제목을 누르면 정렬할 수 있습니다”
- significant 별표 기준을 풀어 쓴 안내
- “microeco 예제처럼 aplot으로 연결했습니다”
- PCoA marginal boxplot의 위치나 기능에 관한 장황한 설명
- taxa 색상과 `Others` 처리에 대한 반복 설명
- “본 조성 분석에는 가설검정·공변량 보정을 적용하지 않았습니다”
- “이는 기술적 비교이며…”와 같은 반복적인 단서
- 별도의 “교수님과 확인할 사항” 문단

# 출력 형식, 테마 및 폰트

- Self-contained HTML로 출력한다.
- R Markdown 테마는 `flatly`를 사용한다.
- 본문, 표 및 그림에 Noto Sans CJK KR 폰트를 사용한다.
- 최종 HTML이 사용자의 로컬 폰트 설치 여부에 의존하지 않도록 본문 폰트를 HTML에 포함한다.
- 목차 접기·펼치기 기능을 유지한다.
- source code, package loading message, 환경 정보 및 입력 처리 출력을 숨긴다.
- 요청하지 않은 디자인 변경을 하지 않는다.
- 동일 group의 색상과 순서를 모든 그림에서 유지한다.
- 축 제목에 abundance 단위와 taxonomic 또는 functional level을 표시한다.
- 결합 그림의 theme, font, 배경 및 panel 간격을 통일한다.
- 한글 폰트 구현 참고가 필요한 경우에만 <https://rpubs.com/websphere74/863351>을 참고한다.

# 표 표시 규칙

- `DT`의 기본 표 형태를 사용한다.
- 커스텀 표 색상, padding, compact style, 강제 column 너비 및 임의 CSS를 제거한다.
- Copy 및 CSV 버튼을 숨긴다.
- 전체 검색창과 column별 검색창을 숨긴다.
- column 정렬과 페이지 이동 기능은 유지한다.
- column 이름을 한국어로 번역하지 않는다.
- 샘플 수와 대상자 수는 소수점 없는 정수로 표시한다.
- Significant column이 요청된 경우 adjusted p-value를 기준으로 생성한다.
- Significant 표기는 `***` < 0.001, `**` < 0.01, `*` < 0.05, `ns` ≥ 0.05를 사용하되 본문에서 기준을 반복 설명하지 않는다.
- 표를 보기 좋게 만들기 위한 임의 재디자인을 하지 않는다.

# 보고서 필수 구성

다음 번호와 순서를 유지한다.

## 1. Dataset

### Subject characteristics
- 별도로 첨부한 metadata 를 기반으로 수행한다. 없다면 Sample 정보를 기반으로 중복된 값을 제거하여 사용한다. 이때는 샘플 정보 기반한 인구학적 정보라는 명시를 추가한다.
- 질환군과 대조군 또는 다그룹 연구의 모든 관련 그룹에 따라 인구학적 정보를 정리한다.
- 동일 대상자의 중복을 제거하여 대상자별 한 행만 사용한다.
- 동일 대상자에서 Age 값이 일치하지 않으면 기록된 최솟값을 사용한다.
- age, sex 등 최소 변수를 포함하여 `moonBook` 기반으로 요약한다.

### Sample info

- 수집된  original 샘플 정보가 있다면 다음과 같이 정리한다. 없다면 NGS 이후 샘플 결과를 기반으로 한다.
- Swab, Pluck 등 `Sample Collection Method`에 따라 분리하여 요약한다.
- `Group`과 `Site`를 별도 column으로 표시한다.
- 각 `Group × Site` 조합별 선택, 분석 및 제외 샘플 수를 제시한다.
- 모든 개수를 정수로 표시한다.

### NGS info
- sequencing 이후 얻어진 샘플 정보를 기반으로 한다. 따로 주어지지 않았다면 샘플 수를 기반으로 정리한다.
- 만약 original 샘플 수와 NGS 이후 샘플 수가 따로 있다면, NGS 에 따른 drop 수를 정리한다.
- 각 group 에 따른 read depth의 Min, Max, Median, Mean, SD를 정리한다. 없다면 생략한다.
- 각 group에 따른 ASV (amplicon 분석의 경우), Kingdom (Shotgun 의 경우), Species, Genus, Phylum의 개수를 정리한다. 이때 표는 taxa rank 에 맞게 column 을 배치한다.


## 2. Alpha diversity

서두에 다음 내용을 개조식으로 제시한다.

- 지수: Observed species, Shannon, Simpson
- 비교군: Swab, Pluck 등 각 collection method 내 HV, CA_U, CA_A
- Wilcoxon 검정 방법 및 paired 여부
- Linear mixed-effects model의 covariate와 random effect
- 보정 후 pairwise comparison 방법
- BH correction을 적용한 검정 범위

결과는 다음 순서로 제시한다.

1. 세 지수를 포함한 patchwork 그림
2. Wilcoxon 결과
3. 공변량 보정 후 세 쌍의 pairwise comparison
4. `Group`, `age`, `sex` 및 추가적으로 넣은 공변량에 따른  전체 효과
5. 간결한 결과 요약

다음 분석과 출력은 제외한다.

- 무작위로 선택한 상태 또는 방문 시점에 대한 Kruskal–Wallis 검정
- Mixed model effect size heatmap 또는 표
- 유의한 효과에 대한 scatter plot

## 3. Beta diversity

서두에 다음 내용을 개조식으로 제시한다.

- Bray–Curtis dissimilarity, Jaccard, UniFrac 등 사용한 distance
- PCoA, PCA, NMDS 등 적용한 차원 축소 방법
- PERMANOVA의 permutation 설정과 covariate
- `subject_id`와 permutation `strata`를 이용한 반복측정 처리
- `betadisper` 수행 방법
- BH correction을 적용한 검정 범위

다음 결과를 제시한다.

- Ordination plot과 marginal boxplot
- 그림 전체에 동일한 흰 배경, theme 및 font 적용
- PERMANOVA와 `betadisper` 요약표
- 간결한 결과 요약

## 4. Taxonomic composition

- 시각화에 사용한 taxonomic level을 명시한다.
- Mean relative abundance 1% 이상 또는 top 14 taxa 등 legend에 표시할 taxa 선정 기준을 명시한다.
- 그룹별 composition 표의 taxa와 그림 legend의 taxa를 일치시킨다.

## 5. 결론

- 한국 과학기술논문에 적합한 간결한 개조식 문장으로 작성한다.
- Alpha diversity, beta diversity 및 taxonomic composition의 주요 결과를 요약한다.
- 명사형 또는 `~함`, `~됨`으로 문장을 종결한다.
- 분석이 변경되면 관련 수치와 결론을 함께 갱신한다.

## 6. 한계점 및 향후 분석 예정

- 해석상 제약과 해결되지 않은 확인 사항을 포함하여 실질적인 한계점 4–10개를 정리한다.
- 향후 분석 약 3개를 우선순위에 따라 제시하고, 바로 수행할 다음 단계와 그 이후 단계를 구분한다.

# 보고서 전체 일관성

- 각 분석의 section 순서와 표·그림 형식을 통일한다.
- 그룹별 composition 표의 taxa와 그림 legend의 taxa를 일치시킨다.
- 본문, 표, 그림 및 결론에 제시된 수치를 서로 일치시킨다.
- 삭제 요청을 받은 설명이나 분석을 다시 삽입하지 않는다.

# 렌더링 및 최종 검증

보고서를 수정할 때마다 다음 절차를 수행한다.

1. `.Rmd`를 HTML로 다시 렌더링한다.
2. 렌더링이 오류 없이 완료되었는지 확인한다.
3. 렌더링 종료 상태만 확인하지 말고 HTML을 직접 열거나 이미지로 렌더링하여 시각적으로 검사한다.
4. 한글 폰트, 표, 그림 및 접기·펼치기 목차가 올바르게 표시되는지 확인한다.
5. 코드, 환경 출력, package loading message, 입력 처리 과정, 검색창 및 Copy·CSV 버튼이 숨겨졌는지 확인한다.
6. 샘플 수와 대상자 수가 정수로 표시되는지 확인한다.
7. 본문, 표, 그림 및 결론의 모든 수치를 교차 확인한다.
8. 최종 HTML 파일의 링크를 제공한다.

HTML 렌더링과 시각 검증이 모두 통과하기 전에는 작업 완료로 보고하지 않는다.
