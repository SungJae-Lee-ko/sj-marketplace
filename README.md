# sj-marketplace

SJ의 Claude 플러그인 마켓플레이스입니다.

## 설치

```bash
# 1. 마켓플레이스 등록
claude plugin marketplace add SungJae-Lee-ko/sj-marketplace

# 2. 플러그인 설치
claude plugin install shopping-shorts@sj-marketplace
claude plugin install news-shorts@sj-marketplace
```

## 플러그인 목록

| 플러그인 | 버전 | 설명 |
|---|---|---|
| [shopping-shorts](plugins/shopping-shorts/) | 0.7.1 | **상품을 파는 쇼츠** — 스크립트 → 스토리보드 → AI 영상 생성(인물 일관성 체인) → 자동 조립(완성 MP4) → 발행 키트 5스킬 체인 + 에이전트 2종. 판매형·습관정보형 지원 |
| [news-shorts](plugins/news-shorts/) | 0.1.0 | **사실을 전하는 쇼츠** — 소스 수집 → 검증(팩트 카드) → 대본 → 스토리보드(출처 크레딧) → AI 영상 생성 → 자동 조립(자막 2층) → 발행 키트(출처 목록) 7스킬 체인 + 에이전트 2종. 이슈 해설·통념 검증·근거 실행법 지원 |

두 플러그인은 독립적으로 동작하며 서로를 필요로 하지 않습니다. 같은 채널에서 상품 편과 정보 편을 번갈아 낼 때 각각을 씁니다.

---

### shopping-shorts 구성

**스킬 체인 (5종)**

```
shorts-script ─→ shorts-storyboard ─┬─→ shorts-video-gen ─→ shorts-assemble ─→ shorts-publish-kit
 (유형 판별·스크립트)  (씬 설계·프롬프트) │    (AI 생성·검수)      (자동 조립·렌더)     (발행 텍스트 자산)
                                    │                          ↑
                                    └──── 실촬영 경로 ──────────┘
```

**에이전트 (2종)**

- `shorts-producer` — "○○ 쇼츠 처음부터 끝까지 만들어줘" 한 마디로 5스킬 체인 전체 주행
- `shorts-auditor` — 발행 전 읽기 전용 검수: 스크립트 채점 · 정합성 · 영상 규격 · 나레이션 · 인물/제품 일관성 · 법규와 고지 PASS/FAIL

**핵심 설계**

- 콘텐츠 유형(`sales`/`habit`)과 판매 형태(`sales_form`)를 스크립트 단계에서 판별해 체인 전체가 나른다
- 제품 실사진은 재생성 없이 영상 모델에 직접 참조로 전달, 씬 2-3개를 샷으로 묶어 연속 생성 (제품 왜곡·슬라이드쇼 방지)
- 인물 등장 샷은 앵커 스틸 고정 → 영상 변환 필수 체인, `person_group`별 앵커 스틸 관리 (identity switching 방지)
- 나레이션이 시간을 정한다 — 스크립트가 문장별 예상 초, 스토리보드가 씬 시간표로 확정, 조립의 배속 보정은 0.95~1.05
- 조립은 ffmpeg 자동 렌더 — 편집 앱 없이 완성 MP4 산출

변경 이력: [plugins/shopping-shorts/CHANGELOG.md](plugins/shopping-shorts/CHANGELOG.md)

---

### news-shorts 구성

**스킬 체인 (7종)**

```
news-research ─→ news-verify ─→ news-script ─→ news-storyboard ─┬─→ news-video-gen ─→ news-assemble ─→ news-publish-kit
 (소스 수집·이슈)   (검증·팩트카드)   (대본·시간표)    (씬 설계·크레딧)  │    (AI 생성·검수)     (조립·자막 2층)    (발행 자산·출처 목록)
                                                              │                          ↑
                                                              └──── 실촬영 경로 ──────────┘
```

**에이전트 (2종)**

- `news-producer` — 키워드 한 마디로 7스킬 체인 전체 주행. 양보 가능한 것(길이·앵글·경로)과 양보 불가한 것(근거 강도·출처·경고 카드·만료)을 분리
- `news-auditor` — 읽기 전용 검수 9영역(시의성·출처 추적성·근거 강도 정합·수치 해석·안전/법규·정합성·영상 규격과 크레딧·인물 일관성·발행 키트). 제작 이후 나온 정정 보도를 직접 재검색하며 게시 후 7일·30일 점검에도 재사용

**핵심 설계**

- 신뢰도 등급 T1~T4 · 1차 출처 필수 · 2차 전달 수치는 독립 2출처 · 정정·철회 확인 절차
- 주장 강도 E1~E4 사다리가 대본에서 쓸 수 있는 어미를 고정 — 제목이 본문보다 셀 수 없다
- **수치는 유형까지 검증한다** — 상대위험 단독 서술 금지(절대값 병기), 관찰연구 인과 서술 금지
- 출처 크레딧이 스토리보드 컬럼 → 조립 자막 레이어 → 발행 키트 출처 목록으로 이어진다 (끊기면 다음 단계가 멈춘다)
- `expiry_date`(업로드 마감)와 `content_expiry`(게시본 유효 종료) + 게시 후 7일·30일 정정 점검
- 화면의 글자·수치·로고는 AI에게 그리게 하지 않고 전부 자막 레이어로 이관

변경 이력: [plugins/news-shorts/CHANGELOG.md](plugins/news-shorts/CHANGELOG.md)

---

## 사용 시작

설치 후 Claude Code 또는 Cowork에서:

```
> 무선 미니 가습기로 쇼핑쇼츠 만들어줘. 타깃은 사무실 직장인      # shopping-shorts
> 무릎건강으로 뉴스 쇼츠 만들어줘                                  # news-shorts
```

> 플러그인을 설치·재설치한 뒤에는 **새 세션을 열어야** 갱신된 스킬이 로드됩니다. 실행 중인 세션은 시작 시점 스냅샷을 계속 씁니다.
