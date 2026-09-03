# shopping-shorts

유튜브 쇼츠 제작 풀세트 플러그인. 주제 한 줄로 스크립트 → 스토리보드 → AI 영상 생성 → 자동 조립 → 발행 키트까지 만들고, **완성된 9:16 MP4**를 돌려줍니다. 편집 앱을 열지 않는 것이 목표입니다.

**상품을 파는 판매형(`sales`)과 상품 없는 습관·정보형(`habit`)** 두 콘텐츠 유형을 모두 지원하며, 유형에 따라 구조·씬 규칙·검수 기준·고지 요건이 분기합니다.

## 스킬 체인 (정본)

```
shorts-script ─→ shorts-storyboard ─┬─→ shorts-video-gen ─→ shorts-assemble ─→ shorts-publish-kit
 (유형 판별·스크립트)  (씬 설계·프롬프트) │    (AI 생성·검수)      (자동 조립·렌더)     (발행 텍스트 자산)
                                    │                          ↑
                                    └──── 실촬영 경로 ──────────┘
```

- **AI 생성 / 혼합** → video-gen을 거쳐 assemble로
- **실촬영** → video-gen을 건너뛰고 스토리보드에서 바로 assemble로 (실촬영 클립도 조립 입력입니다)
- 어느 경로든 종착지는 **assemble의 완성 MP4**입니다. publish-kit은 그 뒤 마지막 단계입니다

| 스킬 | 하는 일 | 트리거 예시 |
|------|---------|------------|
| `shorts-script` | 콘텐츠 유형·판매 형태 판별 → 15-60초(기본 30-45초) 스크립트 + 문장 단위 나레이션 시간표. 기존 스크립트 100점 진단 모드 | "이 상품으로 쇼츠 대본 써줘", "무릎 습관 쇼츠 대본", "내 쇼츠 뭐가 문제야" |
| `shorts-storyboard` | 고정 8컬럼 씬 표(나레이션·person_group 포함) · **씬 시간표 확정** · **샷 플랜**(연속 생성 단위) · 샷별 AI 프롬프트 + 참조 매핑 | "스토리보드 만들어줘", "씬 나눠줘" |
| `shorts-video-gen` | 제품 실사진 직접 참조 + **샷 단위 연속 생성**(extension/전환 생성) → 원본 대조·연결부 검수 → 조립 지시서. 비용 게이트 드래프트 승인 · 인물 일관성 3중 장치 · 판매형/습관형 분기 | "영상 생성해줘", "클립 뽑아줘" |
| `shorts-assemble` | 클립·나레이션 수집 → 길이 정합 → 오디오 믹스 → 자막 번인 → 완성 MP4 | "조립해줘", "최종 영상 만들어줘" |
| `shorts-publish-kit` | 제목 3안 · 설명문 · 해시태그 · 고정댓글 · 업로드 체크리스트 | "업로드 준비해줘" |

각 스킬은 단독으로도 쓸 수 있고, 순서대로 체이닝하면 풀세트가 나옵니다.

## 에이전트 (2종)

- **`shorts-producer`** — "○○ 쇼츠 처음부터 끝까지 만들어줘" 한 마디로 5스킬 체인 전체 주행. 비용·스틸 승인 게이트는 유지합니다
- **`shorts-auditor`** — 발행 전 읽기 전용 검수: 스크립트 채점 · 스토리보드 정합 · 영상 규격 · 나레이션 · 인물/제품 일관성 · 법규와 고지를 증거 기반 PASS/FAIL로 판정. 파일은 수정하지 않습니다

## 체인이 나르는 값 4가지

스킬 간 계약입니다. 단계마다 다음 스킬로 반드시 넘어갑니다.

| 값 | 만드는 곳 | 쓰는 곳 |
|---|---|---|
| `content_type` (`sales`/`habit`) | shorts-script | 스토리보드 씬 규칙 · **video-gen 생성 경로** · 검수 기준 · publish-kit 고지 요건 |
| `sales_form` (`own`/`resell`/`affiliate`/`sponsored`) | shorts-script | video-gen의 제품 AI 재생성 제한 · publish-kit 고지 표 |
| **씬 시간표** (씬 단위: 예상 초→배정 초→실측 초) | shorts-storyboard (script의 문장 단위 시간표를 받아 확정) | video-gen 클립 길이 · assemble 길이 정합·자막 타이밍 |
| `person_group` (`P1`, `P2` …) | shorts-storyboard | video-gen 앵커 스틸 관리 · auditor 프레임 대조 |

## 핵심 설계

- **제품은 재생성하지 않는다** — 제품 실사진(A)을 이미지 모델로 스틸 재생성하지 않고 영상 모델의 참조 입력으로 직접 전달해 생성 통과를 1회로 제한합니다. 왜곡 시 보존 대응 사다리(참조 생성 → 정지 앵커 → 무생성 컷 → 실촬영)로 국소 강등
- **연속성은 생성 단계에서** — 씬 2-3개를 샷(8-15초)으로 묶어 연속 생성하고 샷 사이는 video_extension·start/end frame으로 잇습니다. 씬별 스틸 이어붙이기(슬라이드쇼)로 후퇴하지 않습니다
- **비용 게이트** — get_cost 실측 고지 후, 본 생성(1080p) 전에 앵커 샷 1개를 480p 드래프트로 승인받습니다
- **인물 일관성** — 인물 등장 샷은 앵커 스틸 고정(start_image) 또는 extension 이어가기만 허용. `person_group`별로 앵커 스틸을 따로 관리해 identity switching을 막습니다 (실제 실패 사례 기반)
- **나레이션이 시간을 정한다** — 스크립트가 문장별 음절 수로 예상 시간을 내고, 스토리보드가 이를 **씬 단위 씬 시간표**로 확정합니다. 조립은 그 값을 검증만 하며 배속(atempo) 보정은 0.95~1.05로 제한합니다
- **습관형도 AI로 만든다** — 상품이 없으면 캐스팅 시트로 인물 앵커 스틸을 만들고, 동작은 시작·정점 2스틸 교차컷 + 카운트 자막으로 전달합니다 (실촬영이 1순위인 것은 그대로)
- **조립은 코드로** — ffmpeg 자동 렌더. 편집 앱 없이 완성 MP4를 산출합니다
- Higgsfield 등 영상 생성 MCP 자동 연동, 직접 전송 우선 (egress 허용 시 완전 자동)
- 타인 영상은 분석 전용, 자사 소재만 생성·조립 소스로 사용

## 설치 (Claude Code)

```bash
# 마켓플레이스 등록
claude plugin marketplace add SungJae-Lee-ko/sj-marketplace

# 플러그인 설치
claude plugin install shopping-shorts@sj-marketplace
```

> 플러그인을 재설치한 뒤에는 **새 세션을 열어야** 갱신된 스킬이 로드됩니다. 실행 중인 세션은 시작 시점 스냅샷을 계속 씁니다.

## 사용 예시

```
> 무선 미니 가습기로 쇼핑쇼츠 만들어줘. 타깃은 사무실 직장인
> 무릎 지키는 3분 습관으로 쇼츠 만들어줘. 상품은 없어
```

레퍼런스 확인부터 완성 MP4·발행 키트까지 체인으로 진행됩니다.

## 구조

```
shopping-shorts/
├── .claude-plugin/plugin.json
├── agents/
│   ├── shorts-producer.md
│   └── shorts-auditor.md
├── skills/
│   ├── shorts-script/
│   │   ├── SKILL.md
│   │   ├── references/{hook-patterns,category-briefs,reference-analysis}.md
│   │   └── tests/test-cases.yaml
│   ├── shorts-storyboard/
│   │   ├── SKILL.md
│   │   ├── references/youtube-shorts-specs.md
│   │   └── tests/test-cases.yaml
│   ├── shorts-video-gen/{SKILL.md, tests/}
│   ├── shorts-assemble/{SKILL.md, tests/}
│   └── shorts-publish-kit/{SKILL.md, tests/}
├── CHANGELOG.md
└── README.md
```

변경 이력: [CHANGELOG.md](CHANGELOG.md)

## 라이선스

MIT
