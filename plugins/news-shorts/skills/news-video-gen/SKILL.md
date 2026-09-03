---
name: news-video-gen
description: >
  [책임 경계] 뉴스 쇼츠 영상 생성 오케스트레이터. news-storyboard의 씬 표 + 샷 플랜을 받아 샷 단위 연속 생성(hook·context·evidence·cta·인물 씬) + 스틸 경로(data_card·caution 배경, demo 2스틸 교차컷) → 검수 → 조립 지시서 산출까지 생성 실행 체인을 담당한다. 프롬프트 설계는 news-storyboard, 생성 실행은 본 스킬. "영상 생성해줘", "이 스토리보드로 클립 뽑아줘", "AI로 씬 만들어줘" 요청 시 반드시 사용. 뉴스형 전용 규칙으로 **화면 글자·수치·로고를 AI에게 그리게 하지 않고**(자막 레이어로 이관), 실존 인물·언론사 지면 생성을 차단하며, 인물 씬은 앵커 스틸 start_image 고정 또는 extension 이어가기만 허용해 일관성을 보장한다. 비용 게이트 사다리(get_cost 실측 → 480p 드래프트 승인 → 1080p 본 생성) + 연결부(seam) 검수 + 플랜 제약 모델 폴백을 지원하고, 결과를 조립 지시서(JSON)로 산출한다.
version: "0.2.0"
---

# 뉴스 쇼츠 영상 생성 (Video Generation Orchestrator)

**책임 한 줄**: 확정된 스토리보드(씬 표 + 씬 시간표 + 샷 플랜) + 캐스팅 시트 → 샷 단위 연속 생성 + 스틸 경로 실행 → 검수된 클립 세트 + 조립 지시서 반환.

**뉴스형의 제1원칙**: 화면의 글자는 AI가 그리지 않는다. 수치·기관명·출처 크레딧·자막은 전부 조립 단계의 자막 레이어가 담당한다. AI가 그린 기관명은 **틀릴 수 있고, 틀렸는지 우리가 확인할 방법이 없다.** 확인 불가능한 기관명이 화면에 뜨는 것 자체가 가짜 자료 화면의 정의다.

**제2원칙 — 연속성은 생성 단계에서**: 씬마다 독립 스틸에 카메라 무빙만 입혀 이어붙이면 "사진을 붙인 영상"(슬라이드쇼)이 된다. `hook`·`context`·`evidence`·`cta`·인물 씬은 씬 2-3개를 **샷(8-15초)으로 묶어 연속 생성**하고, 샷 사이는 video_extension·start/end frame으로 잇는다 — 상세: **references/shot-continuity.md** (생성 실행 전 반드시 읽는다). 단 **`data_card`·`caution`은 자막이 주인공이라 정지 배경이 정답**이고(움직이면 글자가 안 읽힌다), **`demo`는 자세 정확도가 우선**이라 2스틸 교차컷을 유지한다 — 이 셋은 의도된 스틸 경로이지 슬라이드쇼가 아니다.

## 사전 조건 (모두 충족해야 시작)

1. news-storyboard 산출물이 확정됨. **씬 표의 10개 고정 컬럼(#·시간·화면 구성·앵글/무빙·role·나레이션·자막·출처 크레딧·오디오·person_group)과 씬 시간표, 그리고 AI 생성 경로면 샷 플랜(샷 ID·씬 범위·연결 방식·공통 스타일 프리픽스)**이 있어야 한다
   - 샷 플랜이 없는 구버전 스토리보드면 shot-continuity.md의 그룹핑 규칙으로 샷 플랜을 만들어 사용자 확인 후 진행한다
   - `person_group` 반려 조건은 **인물이 등장하는 씬이 있는데 그 씬의 ID가 비어 있을 때**다. 인물이 아예 없는 기획(전 씬 `—`)은 정상이며 무인물 모드로 진행한다
   - **출처 크레딧 반려 조건**: 씬 시간표의 `근거 카드`가 채워진 씬인데 출처 크레딧이 `—`이면 **스토리보드로 되돌린다**
   - **role 반려 조건**: role 컬럼이 비어 있으면 되돌린다 (모션 프리셋·자막 스타일·검수 기준이 전부 이 값으로 갈린다)
2. **`content_type` 확인** (`issue` | `myth` | `guide`) — 없으면 되돌린다
3. **`expiry_date` 확인** — 오늘 날짜가 만료를 지났으면 **생성을 시작하지 않는다.** news-verify 재실행을 제안한다. **`expiry_date`가 아예 없어도 시작하지 않는다** (없음은 무기한이 아니다)
4. **검증 산출물 묶음 확인** — 출처 원장·팩트 카드 세트·탈락 목록이 함께 왔는가. 없으면 요청한다. **이 스킬은 묶음을 쓰지 않지만 조립·발행이 쓰므로 그대로 나른다**
5. **영상 생성 MCP(Higgsfield 등) 자동 연동 확인** — 시작 시 가벼운 호출(잔액 조회 등)로 연결을 스스로 검증한다. 미연결이면 커넥터 연결을 제안하고, 연결 불가면 씬별 프롬프트만 정리해 주고 종료
6. **비용 게이트 사다리 준수** (아래 섹션) — `get_cost` 실측 고지·승인 없이 생성 호출 금지, **드래프트(G2) 승인 전 본 생성 해상도(1080p) 호출 금지**
7. **입력 소재**
   - 인물 등장 기획이면 **캐스팅 시트가 필수 입력**이다
   - 뉴스형은 판매 제품이 없으므로 **제품 사진 업로드 게이트가 없다.** 자료 업로드가 필요 없으면 업로드 절차 전체를 건너뛴다
   - 선택 입력: 톤 참고 이미지. 프롬프트의 mood·lighting·wardrobe에만 반영하고 **소스 이미지로 쓰지 않는다**
   - 업로드가 필요한 경우(사용자 실촬영 스틸 등)에만: 직접 업로드(presigned URL PUT)를 1회 시도하고, 차단되면 재시도 없이 업로드 위젯으로 전환한다. 연결된 로컬 폴더가 있으면 먼저 스캔해 파일명·위치를 짚어 안내한다

## 생성 파이프라인

```
[Step 0] 인물 일관성 결정 (인물 등장 시 필수 — 아래 의사결정 트리)
   ↓
[Step 1] 샷 플랜 확정
   └─ 스토리보드의 샷 플랜 검토: hook·context·evidence·cta·인물 씬을 샷당 2-3씬
      (8-15초)으로 묶음, 훅은 단독 샷 — 규칙은 shot-continuity.md
   └─ data_card·caution·demo 씬은 샷으로 묶지 않는다 (아래 스틸 경로)
   ↓
[Step 2] 스틸 생성 (스틸 경로 씬 + 인물 앵커만) → 파일명: still_scene{N}.png (인물 앵커는 still_{P그룹}_anchor.png)
   └─ 인물 앵커: person_group별 1장 (캐스팅 시트 기반) — 승인 게이트 대상 (규칙 c)
   └─ data_card·caution 씬: **배경만.** 글자·숫자·그래프를 그리지 않는다.
      정지 배경 + 조립 켄번스가 기본 (클립 비용 없음)
   └─ demo 씬(guide): 시작·정점 2스틸
   └─ 샷 묶음에 들어간 씬은 스틸을 만들지 않는다 — 샷 생성이 직접 그린다
   ↓
[Step 3] 샷 생성 (비용 게이트 G2 드래프트 → 승인 → G3 본 생성, 앵커 샷 → 체인 순서)
   └─ 앵커 샷(첫 샷): 참조 생성 — 프롬프트 = 공통 스타일 프리픽스(no text, no logos,
      no on-screen numbers 포함) + 샷 비트
   └─ 후속 샷 — 연결 방식별 (shot-continuity.md의 모드 트리):
      · extension → video_extension으로 직전 샷 job_id 이어가기 (같은 공간·연속 동작)
      · cut → 새 참조 생성 + 동일 스타일 프리픽스·인물 앵커 (공간 전환)
      · transition → start_image(직전 끝 프레임) + end_image(다음 첫 프레임)로 전환 생성
   └─ 인물 포함 샷: 해당 person_group 앵커 스틸을 start_image로 고정 + 연속 테이크
      문구 필수 — 참조만 넣은 text-to-video 직행 금지 (identity switching).
      extension 이어가기는 실제 픽셀을 연장하므로 허용
   └─ 스틸 경로 씬(demo)의 i2v: 스틸을 start_image로, 모션은 카메라 연출 표대로
   └─ 공통: aspect_ratio "9:16" 명시, 샷 duration = 묶인 씬 배정 초 합계.
      **배정 초보다 짧은 클립을 만들지 않는다** — 조립에서 빈 구간이 생긴다.
      generate_audio 기본 false (오디오는 assemble의 믹스 담당)
   ↓
[Step 4] 검수
   └─ 공통: 손가락·관절 왜곡 / 씬 목적 부합 / 자막·크레딧 얹힐 하단 여백 확보
   └─ 뉴스형 필수: **화면에 글자·숫자·로고가 생성되지 않았는가** / 실존 인물처럼 보이는 얼굴이 없는가
   └─ 인물 씬: 클립 중간 인물 교체 / 같은 person_group 씬·샷 간 동일 인물 유지
   └─ 연결부(seam) 검사: 샷 경계의 끝/첫 프레임 연속성 (공간·조명·인물) — shot-continuity.md
   └─ 실행 환경이 클립 파일을 직접 열 수 없으면 검수 항목을 사용자에게 명시적으로 제시하고 확인받는다
   └─ 불합격 시 재생성 1회 (프롬프트 수정 후) — 2회 실패면 그 씬을 스틸 경로(정지 배경·다이어그램)
      또는 실촬영으로 전환 제안. 글자·로고 생성 불합격은 프리픽스에 no-text 지시를 강화해 재시도
   └─ 판정을 `review`에 기록: pass / fail(조립 금지) / manual(사용자 확인 대체)
   ↓
[Step 5] 조립 지시서 산출 (샷 클립은 씬 단위로 분할 — 같은 url + 씬별 trim_to)
```

## 데이터 카드 씬 (`role: data_card`)

수치·기관명·인용문이 화면에 뜨는 씬이다. **여기서 AI에게 글자를 그리게 하는 것이 뉴스형 실패의 최다 원인이다.**

1. **배경만 생성한다** — 프롬프트에 `no text, no numbers, no charts, no logos, no signage`를 반드시 넣고, 저채도·저대비·중앙 여백이 있는 배경으로 지시한다. 글자가 얹힐 중앙~중하단 영역을 비운다
2. **수치·문구는 조립 자막 레이어** — 스토리보드 자막 컬럼과 출처 크레딧 컬럼이 그대로 번인된다. `note`에 `data_card`임을 남겨 조립이 강조형 자막 스타일을 쓰게 한다
3. **그래프는 코드로** — AI에게 차트를 그리게 하지 않는다(축·눈금·숫자가 전부 틀린다). matplotlib 등으로 1080×1920 투명 배경 PNG를 만들어 조립에서 오버레이하거나 수치 자막으로 대체하고, **출처 크레딧을 함께 얹는다**
4. **인용문 카드** — 직접 인용은 따옴표 + 발화자·소속을 자막으로. 인물 사진을 만들지 않는다

**비용 절감안**: data_card 씬은 **정지 배경 1장 + 조립 단계 켄번스(느린 줌)**로 대체하면 클립 생성 비용이 들지 않는다. 비용 추정에서 이 대안을 함께 제시한다.

## 동작 시연 (`content_type: guide`)

동작 1개당 **최소 2장의 스틸**을 만들고 각각 짧은 클립으로 변환해 교차컷한다:

| 스틸 | 내용 | 클립 |
|---|---|---|
| 시작 자세 | 동작의 출발 프레임 (정면) | 1.5~2초, slow_zoom |
| 정점 자세 | 힘이 들어간 순간 (측면 — 관절 각도가 보이는 각도) | 1.5~2초, slow_zoom 또는 static |

- 두 스틸은 **같은 앵커·같은 의상·같은 배경**으로 생성한다
- 배정 4초 이상이면 [시작 → 정점 → 시작] 3컷
- **횟수·시간은 화면 텍스트로만 전달된다** — 스토리보드 자막에 카운트 자막이 없으면 되돌려 넣는다
- 동작 시연은 **실촬영이 1순위**다. 생성 시작 전 1회 고지한다. 거부하면 2스틸 교차컷이 표준이며, 그래도 동작이 읽히지 않으면 마지막 폴백은 **정지 스틸 + 화살표·궤적 오버레이 + 카운트 자막**(다이어그램형)이다
- 어떤 경로를 썼는지 `clips[].note`에 남긴다

## 멀티컷 씬 표기 (한 씬 = 클립 2개 이상 — 필수)

- `clips[]`에 **같은 `scene` 값의 항목을 여러 개** 싣고 각 항목에 `"cut": 1`, `"cut": 2` …를 붙인다 (단일 컷 씬도 `"cut": 1`)
- 씬의 배정 초를 컷 수만큼 나눠 `"cut_seconds"`에 적는다 — 합이 그 씬의 배정 초와 정확히 같아야 한다
- `assembly.sequence`는 씬 번호만 적고, 같은 씬 안의 순서는 `cut` 오름차순이 정본이다
- 파일명은 `scene{씬}-{컷}` 규칙 (`scene4-1.mp4`)
- 나레이션은 **그 씬의 첫 컷(`cut: 1`)에만** 싣는다. 나머지 컷의 `narration`·`narration_file`은 `null`
  - **`cut: 1`의 나레이션은 씬 구간 전체에 걸쳐 재생된다.** cut 2 이후가 `null`인 것은 "파일이 없다"는 뜻이지 "무음 구간"이라는 뜻이 아니다 — 6초 한 문장이 3초씩 2컷으로 쪼개져도 문장은 잘리지 않는다
- **출처 크레딧은 씬 단위다** — 멀티컷이어도 크레딧은 그 씬 구간 전체에 통으로 유지된다 (컷마다 깜빡이지 않게)

## 카메라 연출 (스토리보드 `role` 기준 자동 매핑)

`role`은 **스토리보드가 소유하는 값**이다. 여기서 새로 만들지 않고 받아서 매핑만 한다. 샷 묶음 씬은 이 값을 **샷 프롬프트의 카메라 지시**로, 스틸 경로 씬은 **i2v 모션 프리셋**으로 쓴다 (프리셋 이름은 연결된 MCP가 지원하는 실제 명칭으로 치환).

| `role` | 연출 | 이유 |
|---|---|---|
| `hook` | fast push-in 또는 reveal | 첫 프레임에서 시선 고정 |
| `context` 상황·b-roll·브리지 | pan 또는 handheld follow | 생활감. 샷 내 연속 동작으로 생동감 확보 (스틸+줌으로 흉내내지 않는다) |
| `data_card` 수치·인용 카드 | static 또는 very_slow_zoom | **자막이 주인공 — 배경이 움직이면 글자가 안 읽힌다** |
| `evidence` 근거 제시 | static + 화면 분할 여지 | 정보 전달은 무빙 최소화 |
| `demo` 동작 시연 | slow_zoom 또는 static | 관절 각도가 흔들리지 않게 |
| `caution` 경고 카드 | static | 안전 문구는 무조건 정지 화면 |
| `cta` | static 또는 slow_zoom_out | 자막·행동 지시가 주인공 |

사용자가 지정하면 그 값이 우선.

**인물 샷 프롬프트 제한 (필수)**
- 프롬프트에 항상 명시: "one single continuous take, no camera cuts, no scene changes, the same person(s) with consistent clothing and appearance from first frame to last"
- 복잡한 행위를 한 프롬프트로 연출하라고 요구하지 않는다 — 행위의 정점을 앵커 스틸에 고정하고, 샷은 그 장면의 자연스러운 전후 동작·카메라 무빙까지만 지시한다

## 생성 금지 소재 (뉴스형 절대 규칙)

프롬프트에 넣지 않고, 결과물에 나오면 재생성한다:

- **실존 인물의 얼굴** — 전문가·정치인·기자·연예인 모두. 특정인을 연상시키는 묘사도 금지
- **언론사 로고·신문 지면·기사 화면·방송 화면**
- **기관 로고·간판·엠블럼** (질병관리청·병원·학회 등) — 크레딧은 텍스트로만
- **실제 브랜드 제품·패키지·약통 라벨**
- **의료 시술·수술 장면·환부 클로즈업** — 시청자 불쾌감 + 플랫폼 제재 위험
- 진단서·처방전·검사 결과지처럼 보이는 문서

실존 인물 얼굴로 캐릭터를 만들려는 요청은 본인 동의 확인이 없으면 거절한다.

## 인물 일관성 의사결정 트리 (Step 0)

```
[Q1] 이 영상(또는 시리즈)에 사람이 등장하나?
   ├─ 아니오 → Step 0 skip (무인물 모드 — 뉴스형은 이 경우가 흔하다)
   └─ 예 → [Q2]
[Q2] 같은 인물이 반복 등장하나? (시리즈 간 반복 OR 단일 영상 내 2개 이상 씬)
   ├─ 아니오 (씬마다 다른 단역, `P1-a`·`P1-b`) → 씬별 스틸 고정만 (규칙 a). person_anchor는 null
   └─ 예 → [Q3]
[Q3] 캐릭터 등록 기능(character_id)을 현재 플랜에서 쓸 수 있나?
   ├─ 예 → 등록된 id를 모든 인물 씬 생성에 전달 (최초 1회 비용·시간 고지)
   └─ 아니오/불확실 → 앵커 스틸 방식 (규칙 b)
```

## 인물 일관성 규칙 (단일 영상 내)

- **a. 샷 내 일관성 — 앵커 고정**: 인물 등장 샷은 반드시 앵커 스틸 → start_image 체인으로 생성한다 (또는 직전 샷의 extension 이어가기 — 실제 픽셀을 연장하므로 인물이 유지된다). 참조 이미지만 넣은 text-to-video 직행은 금지
- **b. 씬 간 일관성 — 앵커 스틸 재사용**: **스토리보드의 `person_group` ID가 기준이다.** 같은 그룹의 앵커 스틸을 공유하고, 새 샷·스틸 생성 시 앵커를 인물 참조로 함께 전달해 "same person as the reference image, different pose/angle"로 지시한다. **`P1`의 앵커를 `P2` 씬에 쓰지 않는다**
- **c. 스틸 승인 게이트**: 인물 앵커 스틸은 영상 변환 전에 사용자에게 보여주고 확인받는다. 승인 뒤에만 i2v 크레딧을 쓴다
- **d. 검수 연동**: Step 3에서 "클립 중간 인물 교체 없음 + 같은 `person_group` 씬 간 동일 인물 유지"를 명시적으로 확인한다

## 모델 폴백 규칙

폴백으로 모델이 바뀌면 **비용을 재추정해 다시 승인**받는다.

| 오류 | 대응 |
|---|---|
| 플랜 제한 | 같은 계열 경량 모델 → 동일 참조 롤 지원 타 벤더 모델 순으로 폴백. 단가를 다시 preflight |
| video_extension 미지원 모델로 폴백됨 | extension 연결을 "직전 샷 끝 프레임 캡처 → start_image" 방식으로 대체하고 seam 검사를 강화 |
| 레이트 리밋 (429) | 60-90초 대기 후 실패분만 재제출. 즉시 연속 재시도 금지 |
| 프리셋 추천 개입 | 의도한 연출이면 수락, 아니면 declined_preset_id를 붙여 원래 요청 재제출 |
| 업로드 위젯 미지원 클라이언트 | 서비스 직접 업로드 안내 후 `show_medias`로 media_id 확인 |
| 생성 실패 (과금 없음 확인) | 프롬프트를 단순화해 1회 재생성 |
| 안전 필터 거부 (의료·신체 묘사) | 묘사를 추상화한다 (환부 → 실루엣·손 클로즈업). 2회 실패면 그 씬을 data_card로 대체 제안 |

## 비용 게이트 사다리 (승인 없이 생성 호출 금지)

영상 생성은 크레딧이 비싸다. 확신이 낮은 단계에서 비싼 호출을 하지 않도록, 게이트를 위에서부터 순서대로 통과한다:

| 게이트 | 비용 | 확인하는 것 |
|---|---|---|
| G0. 샷 플랜·프롬프트·(인물) 앵커 스틸 승인 | 0~스틸값 | 연출 방향, 샷 묶음, 인물 외형 (스틸 재생성은 클립보다 수 배 싸다) |
| G1. `get_cost:true` 실측 프리플라이트 | 0 | 드래프트·본 생성 각각의 실제 크레딧 (생성 없이 조회) — 합계 + 재생성 예비 +20% 고지 |
| G2. **드래프트 생성** — 앵커 샷 1개 | 저가 | 스타일·모션·no-text 준수가 원하는 방향인지. 본 생성과 **동일한 프롬프트**로, 해상도만 최저(480p)·fast/mini 변형 사용 |
| G3. 본 생성 (1080p, 전 샷 체인 + 스틸 경로 클립) | 고가 | G2에서 확정된 프롬프트 그대로 실행 |

- 드래프트 불합격이면 프롬프트를 수정해 **드래프트만** 재시도 — 본 생성 크레딧은 방향이 확정된 뒤에만 쓴다. 스틸 프리뷰로 대체 검증하지 않는다 (스틸은 영상 결과를 예고하지 못한다 — 인물 앵커 스틸은 예외로, 인물 외형 확정용이다)
- 무료 무제한 생성(unlim)을 지원하는 모델·잔여분이 있으면 드래프트에 우선 사용을 제안한다
- G1 계산 규칙:
  - **data_card·caution 씬은 정지 스틸 + 조립 켄번스가 기본** — 클립 비용 없음
  - (guide) 동작 시연 씬은 씬당 스틸 2장·클립 2개, 배정 4초 이상이면 3컷 + 그룹별 앵커 스틸 1장 추가
  - 샷 묶음 씬은 씬 수가 아니라 **샷 수 × 샷 duration** 기준으로 계산 (씬별 클립보다 통상 저렴하다)

## 출력 형식 (조립 지시서)

**이 지시서는 news-assemble의 입력 데이터다.** 필드명을 임의로 바꾸거나 빼지 않는다. **샷으로 연속 생성한 클립은 씬 단위로 분할해 싣는다** — 같은 샷의 인접 씬들은 **같은 `url`을 공유하고 `trim_to`로 각자 구간을 지정**한다 (연속 영상을 씬 경계에서 잘라 다시 붙이면 프레임이 동일해 이음새가 없다. assemble은 url 단위로 1회만 다운로드).

```json
{
  "storyboard_ref": "무릎소리-v1",
  "content_type": "myth",
  "freshness": "B",
  "expiry_date": "2026-10-24",
  "content_expiry": "2027-01-22",
  "verification_bundle": "./무릎소리/검증묶음.md",
  "caution_scene": 10,
  "caution_level": "W-visit",
  "character_id": null,
  "scene_timing": [
    {"scene": 1, "assigned": 4.0, "measured": 3.8, "fact_cards": []},
    {"scene": 2, "assigned": 2.0, "measured": null, "fact_cards": []},
    {"scene": 3, "assigned": 6.0, "measured": null, "fact_cards": ["F1"]}
  ],
  "shots": [
    {"shot": "S1", "scenes": [1, 2], "mode": "reference", "duration": 6, "url": "...",
     "seam_to_next": "cut", "seam_check": "pass"}
  ],
  "clips": [
    {"scene": 1, "cut": 1, "cut_seconds": 4.0, "shot": "S1",
     "role": "hook", "duration": 6, "trim_to": "0.0-4.0",
     "motion": "fast push-in", "source": "shot S1 (연속 생성)", "url": "...",
     "person_group": "P1", "person_anchor": "still_p1_anchor.png",
     "narration": "계단 내려갈 때 무릎 시큰한 분", "narration_file": null,
     "source_credit": null,
     "review": "pass", "note": null},
    {"scene": 2, "cut": 1, "cut_seconds": 2.0, "shot": "S1",
     "role": "context", "duration": 6, "trim_to": "4.0-6.0",
     "motion": "pan", "source": "shot S1 (연속 생성)", "url": "...(S1과 동일)",
     "person_group": null, "person_anchor": null,
     "narration": null, "narration_file": null,
     "source_credit": null,
     "review": "pass", "note": "브리지 씬 (나레이션 없음)"},
    {"scene": 3, "cut": 1, "cut_seconds": 6.0, "shot": null,
     "role": "data_card", "duration": 10, "trim_to": "0.0-6.0",
     "motion": "static", "source": "still_scene3_bg.png", "url": "...",
     "person_group": null, "person_anchor": null,
     "narration": "소리 자체는 대부분 문제가 아니라고 알려져 있습니다", "narration_file": "narr03.mp3",
     "source_credit": "출처: 대한정형외과학회(2025.11)",
     "review": "pass", "note": "data_card — 배경만 생성, 수치는 자막 레이어"}
  ],
  "assembly": {
    "sequence": [1,2,3],
    "transitions": [{"between": [2,3], "type": "xfade", "sec": 0.25}],
    "subtitles": "스토리보드 자막 컬럼 참조 (타이밍은 scene_timing의 씬 구간)",
    "source_credits": "clips[].source_credit — 별도 자막 레이어로 번인",
    "narration": "clips[].narration_file 참조 (없으면 assemble이 TTS 생성 여부를 확인)",
    "bgm_cue": "스토리보드 오디오 컬럼 참조",
    "export": "9:16 1080×1920, 씬별 trim_to 구간만 사용"
  },
  "ai_disclosure": "유튜브 업로드 시 '변경·합성된 콘텐츠' 공개 설정 필요"
}
```

**필드 계약 — 아래 표에 없는 필드를 새로 만들지 않는다.**

| 필드 | 생산 | 소비 |
|---|---|---|
| `storyboard_ref` | 이 스킬 (스토리보드 제목-버전) | assemble·auditor가 대조 대상 식별 |
| `content_type` | 스토리보드에서 그대로 전달 | assemble 자막 규칙 · publish-kit · auditor 판정 기준 |
| `freshness` / `expiry_date` / `content_expiry` | 검증 값을 그대로 전달 | assemble·publish-kit 시작 게이트 · 정정 점검 예약 · auditor |
| `verification_bundle` | 스토리보드에서 받은 **검증 산출물 묶음의 경로 또는 내용** | **publish-kit의 출처 목록·고정댓글 · auditor의 탈락 목록 대조** — 이 값이 없으면 발행 키트를 만들 수 없다 |
| `caution_scene` / `caution_level` | 스토리보드 체인 값 줄에서 옮김 | **assemble 렌더 전 게이트 3** · auditor 안전 검수 |
| `scene_timing` | 스토리보드 씬 시간표를 그대로 옮김 (`assigned`·`measured`·`fact_cards`). **예상 초는 옮기지 않는다 — 배정 초가 유일한 계약값이다** | **assemble의 길이 정합·자막 타이밍의 유일한 기준** · 크레딧 강제 검사 |
| `shots[]` | 이 스킬 (샷 ID·씬 범위·생성 모드·seam 검사 결과) | auditor의 연결부 대조 · 재생성 시 이 스킬 |
| `clips[].shot` | 그 클립이 나온 샷 ID (스틸 경로 클립은 `null`) | assemble의 url 중복 다운로드 방지 · 같은 shot 인접 씬 경계는 전환 효과 생략 |
| `clips[].scene` | 이 스킬 (씬 번호, 접두사 없는 정수) | assemble 매칭 |
| `cut` / `cut_seconds` | 이 스킬 (단일 컷 씬은 `1` / 씬 배정 초) | assemble의 컷 순서·씬 구간 분할 |
| `duration` | 생성 클립의 원본 길이(초) | **assemble이 `trim_to` 유효성 검사** — `trim_to` 길이 < `cut_seconds`면 되돌린다 |
| `role` | **스토리보드 컬럼 값을 그대로 옮김** | 이 스킬의 프리셋 매핑 · assemble 자막 스타일 · auditor 씬 비중 판정 |
| `motion` | 적용한 프리셋명 | assemble 리포트 · 재생성 시 이 스킬 |
| `trim_to` | 생성 클립에서 씬 길이만큼 잘라 쓸 구간 | assemble 트림 |
| `source` | 스틸 파일명 (`still_scene{N}.png`) | auditor 일관성 대조 |
| `url` | 생성 결과 원격 URL (만료 가능) | assemble의 클립 다운로드 |
| `person_group` | **스토리보드 씬 표와 같은 ID.** 인물 없으면 `null` | auditor 그룹별 프레임 대조 |
| `person_anchor` | 앵커 스틸 파일명(또는 character_id). **같은 `person_group`이 2개 이상 씬에 등장하면 반드시 채운다. 단역(`Pn-x`)은 `null` 허용** | auditor가 앵커 ↔ 클립 프레임 대조 |
| `narration` / `narration_file` | 스토리보드 나레이션 / TTS 파일명(미생성이면 `null`) | assemble 오디오 믹스 |
| `source_credit` | **스토리보드 출처 크레딧 컬럼의 값 그대로.** 없으면 `null` | **assemble의 크레딧 자막 레이어** · auditor 크레딧 검사 |
| `review` | `pass` \| `fail` \| `manual` | **assemble이 `pass`가 아닌 클립은 조립하지 않는다** |
| `note` | 특이 처리(data_card·다이어그램 폴백·2스틸 교차컷·브리지 씬 등). 없으면 `null` | assemble 자막 스타일·리포트 · auditor 참고 |
| `character_id` | Step 0 [Q3]에서 캐릭터를 등록했으면 그 ID, 아니면 `null` | auditor의 인물 일관성 판정 기준 |
| `assembly.sequence` | 씬 번호 순서 | assemble 연결 순서 (같은 씬 안은 `cut` 오름차순) |
| `assembly.transitions` | **스토리보드 전환 가이드를 그대로 옮김.** 없으면 빈 배열(= 전부 하드컷) | assemble의 xfade 적용 + 총 길이 보정 |
| `assembly.subtitles` / `source_credits` / `narration` / `bgm_cue` / `export` | 소스 위치 포인터 | assemble 설정 |
| `ai_disclosure` | 고정 문구 | assemble 리포트 → publish-kit 체크리스트 |

**정합 규칙 (지시서를 내기 전 자체 검사)**
- `scene_timing[].fact_cards`가 비어 있지 않은 씬의 `clips[].source_credit`은 `null`일 수 없다
- `assembly.sequence`의 모든 씬이 `scene_timing`과 `clips[]` 양쪽에 존재해야 한다 (브리지 씬 포함 — 나레이션이 없어도 화면은 필요하다)
- 씬별 `cut_seconds` 합 = 그 씬의 `assigned`
- 각 클립의 `trim_to` 길이 ≥ `cut_seconds`
- `caution_scene`이 `clips[]`에 존재하고 그 씬의 `role`이 `caution`인가
- 같은 `shot`의 씬들이 같은 `url`을 갖고, 그 `trim_to` 구간들이 겹치거나 비지 않고 연속인가
- `assembly.transitions`가 **같은 shot 내부의 씬 경계를 가리키지 않는가** (그 경계는 이어지는 영상이다 — 전환은 샷과 샷 사이에만)

하나라도 깨지면 지시서를 내지 말고 스토리보드로 되돌린다.

- 클립 URL은 만료될 수 있음을 지시서에 명기하고 즉시 다운로드를 권장한다

## 이 스킬을 사용하지 말아야 할 때

- **대본·콘티가 아직 없음** → news-script / news-storyboard 먼저
- **단일 이미지·단일 클립만 필요** → 영상 생성 MCP 직접 호출이 빠름
- **실촬영으로 확정된 프로젝트** → 본 스킬을 건너뛰고 스토리보드 → **news-assemble**로 간다
- **기사 화면·뉴스 클립을 만들어 달라는 요청** → 거절한다. data_card로 대체 제안

## 다음 단계 (체이닝)

체인 위상: **news-research → news-verify → news-script → news-storyboard → news-video-gen(AI 생성 시) → news-assemble → news-publish-kit**

클립 세트가 나오면 **자동 조립이 기본 경로**다: "클립을 연결 폴더에 넣어주시면 자막·출처 크레딧·오디오까지 자동 조립해 완성 MP4를 만들어 드릴까요?" → **news-assemble**. 조립 지시서와 **검증 산출물 묶음**을 함께 넘긴다.
