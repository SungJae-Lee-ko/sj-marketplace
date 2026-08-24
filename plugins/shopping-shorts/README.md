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
| `shorts-script` | 콘텐츠 유형 판별 → 30-45초 스크립트 + 나레이션 실측 기준 씬 시간표. 기존 스크립트 100점 진단 모드 | "이 상품으로 쇼츠 대본 써줘", "내 쇼츠 뭐가 문제야" |
| `shorts-storyboard` | 고정 8컬럼 씬 표(나레이션·person_group 포함) · 앵글 · 전환 · AI 생성 프롬프트 | "스토리보드 만들어줘", "씬 나눠줘" |
| `shorts-video-gen` | 씬 스틸 생성 → image-to-video → 클립 검수 → 조립 지시서. 인물 일관성 3중 장치 | "영상 생성해줘", "클립 뽑아줘" |
| `shorts-assemble` | 클립·나레이션 수집 → 길이 정합 → 오디오 믹스 → 자막 번인 → 완성 MP4 | "조립해줘", "최종 영상 만들어줘" |
| `shorts-publish-kit` | 제목 3안 · 설명문 · 해시태그 · 고정댓글 · 업로드 체크리스트 | "업로드 준비해줘" |

각 스킬은 단독으로도 쓸 수 있고, 순서대로 체이닝하면 풀세트가 나옵니다.

## 에이전트 (2종)

- **`shorts-producer`** — "○○ 쇼츠 처음부터 끝까지 만들어줘" 한 마디로 5스킬 체인 전체 주행. 비용·스틸 승인 게이트는 유지합니다
- **`shorts-auditor`** — 발행 전 읽기 전용 검수: 스크립트 채점 · 스토리보드 정합 · 영상 규격 · 나레이션 · 인물/제품 일관성 · 법규와 고지를 증거 기반 PASS/FAIL로 판정. 파일은 수정하지 않습니다

## 체인이 나르는 값 3가지

스킬 간 계약입니다. 단계마다 다음 스킬로 반드시 넘어갑니다.

| 값 | 만드는 곳 | 쓰는 곳 |
|---|---|---|
| `content_type` (`sales`/`habit`) | shorts-script | 스토리보드 씬 규칙 · 검수 기준 · publish-kit 고지 요건 |
| **씬 시간표** (음절수→예상 초→배정 초) | shorts-script | 스토리보드 씬 시간 · assemble 길이 정합 |
| `person_group` (`P1`, `P2` …) | shorts-storyboard | video-gen 앵커 스틸 관리 · auditor 프레임 대조 |

## 핵심 설계

- **인물 일관성** — 인물 등장 씬은 스틸 고정 → image-to-video 필수 체인. `person_group`별로 앵커 스틸을 따로 관리해 identity switching을 막습니다 (실제 실패 사례 기반)
- **나레이션이 씬 시간을 정한다** — 스크립트 단계에서 나레이션 음절 수로 씬 시간을 확정하고, 조립은 그 값을 검증만 합니다. 배속(atempo) 보정은 ±5% 이내로 제한합니다
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
