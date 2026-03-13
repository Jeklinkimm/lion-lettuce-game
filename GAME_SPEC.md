# Lion Lettuce — 게임 기획서 & 구현 명세

> **목적:** 이 문서만 보고 Claude Code가 동일한 게임을 처음부터 재현할 수 있도록 작성됨.
> **빌드 명령어:** 별도 빌드 없음. `index.html`을 브라우저에서 열거나 `python3 -m http.server 8000`으로 서빙.

---

## 1. 게임 개요

| 항목 | 내용 |
|------|------|
| 게임명 | Lion Lettuce (사자 양배추 게임) |
| 대상 | 자폐 아동 재활치료 |
| 장르 | 2D 캐주얼 모션인식 게임 |
| 입력 | MediaPipe Pose (웹캠) + 키보드/터치 fallback |
| 렌더링 | HTML5 Canvas 2D, 전체 뷰포트 |
| 사운드 | Web Audio API (합성음), SpeechSynthesis (한국어 TTS 칭찬) |
| 외부 라이브러리 | MediaPipe Pose (CDN), Google Fonts Jua |

---

## 2. 핵심 게임플레이 루프

```
[양배추 스폰] → [화면 내 바운싱] → [사자 입 범위 충돌] → [attracted → stuck]
→ [물기 3회 (health 3→2→1→0)] → [폭발 + 점수 +1] → [다음 양배추]
```

### 조작법
- **만세 (팔 올리기):** 사자 입 벌림 (jawOpen → 1.0)
- **팔 내리기:** 사자 입 닫힘 (스냅) → `tryBite()` 호출
- **키보드:** Space/Enter → 입 닫기 + 물기
- **터치:** 화면 터치 → 물기

### 레벨 시스템
- 레벨당 60초, 목표: 양배추 10개 먹기 (`clearGoal: 10`)
- 목표 달성 → 레벨 클리어, Space로 다음 레벨
- 시간 종료 → 게임 오버, Space로 처음부터

---

## 3. 파일 구조

```
project/
├── index.html          # UI, HUD, MediaPipe 연동, 게임 루프
├── engine.js           # 모든 게임 로직 (아래 클래스들)
└── assets/
    ├── head/
    │   ├── upper_jaw.png    # 상악 (349×306 원본)
    │   └── lower_jaw.png    # 하악 (253×151 원본)
    ├── lettuce/
    │   ├── lettuce_normal.png
    │   ├── lettuce_crush1.png
    │   └── lettuce_crush2.png
    ├── main screen/
    │   └── mainIcon.png     # 시작화면 아이콘
    └── startBt/
        └── Subject.png      # 시작 버튼 이미지
```

---

## 4. engine.js 클래스 상세 명세

### 4.1 Assets (싱글톤 객체)
```javascript
Assets = {
  lion: { upper: null, lower: null },
  lettuce: { normal: null, crush1: null, crush2: null },
  loaded: false,
  load(basePath = '../../assets') → Promise  // 5개 이미지 로드
}
```

### 4.2 SoundManager
Web Audio API 기반 합성음. 외부 오디오 파일 없음.

| 메서드 | 설명 | 핵심 파라미터 |
|--------|------|-------------|
| `playBite()` | 첫 물기 "아삭" | 3레이어: square어택(400→80Hz) + 크런치노이즈(HP1800/LP6000) + 서브(120→40Hz) |
| `playCrunchBite()` | stuck 양배추 물기 "와구작" | 4레이어: square더블탭 + bandpass노이즈(2800Hz) + sawtooth(1200→300) + 서브(90→25Hz) |
| `playEatComplete()` | 완식 징글 | square임팩트 + 상승 멜로디(523→1047Hz, 4음) |
| `playCrush()` | 으스러지는 소리 | sawtooth(300→100Hz) |
| `playFanfare()` | 게임 종료 팡파레 | 7음 멜로디(523→1318Hz) |
| `playPraise(text)` | 한국어 TTS | SpeechSynthesis, rate:1.1, pitch:1.3 |
| `startBGM()` | 루프 BGM | BPM 160, 코드: C→G→Am→F, 4파트(패드+베이스+멜로디+하이햇) |
| `stopBGM()` | BGM 정지 | clearTimeout |

### 4.3 Cabbage (양배추 상태머신)

**상수:**
- `CABBAGE_MAX_HEALTH = 3`
- `CABBAGE_STATES = { NORMAL: 0, CRUSH1: 1, CRUSH2: 2 }`

**생성자 파라미터:** `(x, y, vx, vy, size, canvasW, canvasH)`

**Phase 전이:**
```
bouncing → attracted → stuck → exploding → dead
```

| Phase | 동작 |
|-------|------|
| `bouncing` | 중력(350) + 바운스(0.72). 벽/바닥 반사. 속도 < 80이면 자동 점프 |
| `attracted` | `attractTo(x,y)` 호출 시. 0.25초간 lerp(t=dt*18)으로 목표 이동 |
| `stuck` | 사자 턱 사이에 고정 (위치는 GameEngine에서 외부 설정) |
| `exploding` | 0.08초간 scale 1.5→확대, opacity 급감 → dead |

**`bite()` 반환:** health가 0이 되면 `true` (완전히 먹힘)

**시각 효과:**
- `crushState CRUSH1`: 가로 1.15배, 세로 0.88배 + 금색 글로우
- `crushState CRUSH2`: 가로 1.3배, 세로 0.75배 + 주황 글로우
- 물릴 때: shakeTimer 0.35초, squishScale 0.5→1 복원, flashTimer 0.2초 흰색

**충돌 반경:** `size * 0.4 * scale`

### 4.4 Lion (분리 턱 사자)

**핵심 치수 (귀여운 비율):**
```javascript
UPPER_W = 340, UPPER_H = 340 * (306/349) ≈ 298.6
LOWER_W = 240, LOWER_H = 240 * (151/253) ≈ 143.2
jawMaxOffset = 60  // 하악 최대 열림 픽셀
```

**턱 스냅 속도:**
- 닫힐 때: speed = 35 (빠름)
- 열릴 때: speed = 20

**렌더링:**
- 상악: `drawImage(upper, -UPPER_W/2, -UPPER_H*0.72, UPPER_W, UPPER_H)` — y축 72% 위로 오프셋
- 하악: `drawImage(lower, -LOWER_W/2, 0, LOWER_W, LOWER_H)` — jawDown만큼 아래로
- 하악 아래에 bellyScale 타원 (먹으면 커짐)
- 상악에 maneGlow (물면 금색 섀도우)

**충돌 범위:** 상악 top ~ 하악 bottom 전체 사각형, 너비 = max(UPPER_W, LOWER_W) * baseScale

### 4.5 ParticleSystem

| 이미터 | 개수 | 특징 |
|--------|------|------|
| `emitEat(x,y,intensity)` | 18*intensity | 이모지60%+원형. 중력500. 강도에 따라 속도/크기/수명 스케일 |
| `emitCrush(x,y)` | 8 | 녹색 계열 작은 파티클 |
| `emitBiteSparkle(x,y)` | 6 | 흰색 작은 원, 방사형 |

**파티클 구조:** `{ x, y, vx, vy, life, age, size, color, emoji }`

### 4.6 FloatingText
- `add(text, x, y, color, size)` — life: 1.2초
- 매 프레임 y -= 40*dt, alpha = 1-age/life
- 바운스 스케일: `1 + sin(age*8)*0.06`
- 검은 stroke(3px) + 컬러 fill

### 4.7 ScreenShake
```javascript
trigger(intensity = 8, duration = 0.2)
// intensity는 Math.max로 스택 (기존보다 큰 값만 적용)
// duration은 Math.max로 연장
// update: timer 감소, t = timer/duration으로 decay
// offset = random * intensity * t
```

### 4.8 EndingSystem (게임 종료 연출)

**구성:**
- 초기 폭발 콘페티 3발 (각 40개, 총 120개) — 다양한 색상 rect/circle
- **떨어지는 이모티콘 쌓기 없음** (제거됨)
- 텍스트 오버레이 (캔버스에 직접 그림, DOM 아님)

**텍스트 연출 타임라인:**
| 시간 | 연출 |
|------|------|
| 0초~ | 반투명 배경 그라디언트 페이드인 + 제목 줌인 바운스 |
| 0.4초~ | 별 등급 팝업 (최대 3개, score/3 기준) |
| 0.5초~ | 점수 카운트업 애니메이션 + "N개 먹었어요" + "총 N번 물었어요" |
| 1.5초~ | 격려 메시지 페이드인 |
| 2.5초~ | "스페이스바를 눌러 다시 시작!" 깜빡이 |

### 4.9 GameEngine (메인 엔진)

**설정:**
```javascript
config = {
  cabbageSize: 100,
  fxIntensity: 1,
  maxCabbages: 5,
  spawnWaitSec: 10,
  clearGoal: 10
}
timeLeft = 60, totalTime = 60
```

**스폰 로직:**
- 양배추 0개 → 즉시 스폰
- 가장 오래된 것과 가장 새로운 것 모두 10초 이상 → +1 스폰 (최대 5개)
- 스폰 위치: 화면 왼쪽/오른쪽 밖, y = CH*(0.1~0.4)
- 속도: vx = ±(180~400), vy = -(120~300)

**tryBite() 플로우:**
1. **stuck 양배추 있으면** → `bite()` + 와구작 크런치 + 스파클
   - 완전히 먹힘: score++, 대폭발 파티클(intensity=5), 금색 플래시(0.7), 쉐이크(35, 0.6초) + 120ms 후 2차 쉐이크(18, 0.3초), 칭찬 TTS
   - 아직 남음: 쉐이크(16, 0.3초), 흰색 플래시(0.3), 크러시 파티클
2. **bouncing 양배추 충돌** → attracted → stuck 전이, 물기 사운드, 쉐이크(10, 0.2초)
3. **빈 물기** → 작은 쉐이크(4, 0.15초)

**stuck 양배추 위치 (핵심 공식):**
```javascript
const sc = lion.baseScale;
const jawDown = lion.jawOpen * lion.jawMaxOffset * sc;
const upperBottom = lion.UPPER_H * 0.28 * sc;
c.y = lion.y + upperBottom + jawDown * 0.35;
```

**tick() 업데이트 순서:**
1. dt 계산 (max 0.05초)
2. 엔딩 업데이트
3. lion/particles/floatingText/screenShake/flash 업데이트 (항상, freeze 방지)
4. if (!running) return
5. 시간 감소 → 종료 체크
6. 스폰 로직
7. stuck 양배추 위치 갱신 + 양배추 업데이트

---

## 5. index.html 상세 명세

### 5.1 시작 화면 (Start Overlay)
- 풀스크린 오버레이, 배경 `linear-gradient(160deg, #FFF6E8, #F0FFF6)`
- mainIcon.png (160px)
- 제목 "Lion Lettuce" (52px, #3a6e28)
- 부제 "사자 양배추 게임" (22px, #8aaa7a)
- Subject.png 시작 버튼 (320px, 호버 1.08배)
- 설명 "만세하면 입 벌림 · 팔 내리면 와구!"
- **배경 이모지 파티클:** 40개 이모지 (🥬⭐💚🌿✨🎉🦁🌟💫🎈🎀🏆) + 25개 반짝이 도트
  - 4종 애니메이션: float, orbit, pulse, drift
  - 나선형 배치 (각도 + 링 거리)

### 5.2 인게임 HUD (Casual Game Style)
폰트: Google Fonts "Jua"

| 요소 | 위치 | 스타일 |
|------|------|--------|
| 레벨 뱃지 | 좌상단 | 금색 그라디언트(#FFD700→#FFA500), ⭐ + "Lv.N", 테두리 #c8860a |
| 먹은 양배추 패널 | 레벨 뱃지 아래 | blur(8px) 반투명, lettuce_normal.png 아이콘 28px, pop-in 애니메이션, "N/10" |
| 원형 타이머 | 우상단 | SVG circle, r=34, circumference=213.6, stroke #7dd85a, 10초 이하 #ff5555 + pulse |
| HP 하트 | 하단 중앙 | 💚(full) / 🖤(empty) × 3, blur(6px) 배경, 현재 stuck/edible 양배추 기준 |
| 힌트 | HP 위 | "🦁 만세 → 입 벌림 · 팔 내리면 와구! (3번 물면 터짐!)" |
| 포즈 상태 | 상단 중앙 90px | idle/ready/detected/noperson 4상태 |
| 디버그 버튼 | 우하단 | 🔧 디버그, 🥬 추가 |
| 하악 슬라이더 | 우측 중앙 | vertical range, jawFromPose 조절 |

### 5.3 MediaPipe Pose 연동

**CDN:**
```
@mediapipe/camera_utils/camera_utils.js
@mediapipe/pose/pose.js
```

**설정:** `modelComplexity:1, smoothLandmarks:true, minDetectionConfidence:0.5, minTrackingConfidence:0.5`

**포즈 → 사자 위치:**
```javascript
// 랜드마크: 11=왼어깨, 12=오른어깨, 15=왼손목, 16=오른손목
lion.x = (1 - (ls.x + rs.x) / 2) * CW;  // 좌우 미러링
const shoulderY = (ls.y + rs.y) / 2 * CH;
const shoulderW = Math.abs(rs.x - ls.x);
lion.y = shoulderY + shoulderW * CH * 0.15;  // 턱 아래에 위치
lion.baseScale = clamp(0.3, shoulderW * 3, 0.8);
```

**입 열기/닫기:**
```javascript
const avgUp = ((ls.y - lw.y) + (rs.y - rw.y)) / 2;  // 어깨-손목 높이차
const jawAmount = avgUp > 0.12 ? min(1, avgUp / 0.2) : 0;
// 만세(avgUp > 0.12) = 입 벌림
// 팔 내림(prevJaw >= 0.3 → jawAmount <= 0.05) = tryBite() (400ms 쿨다운)
```

### 5.4 게임 루프 (렌더링 순서)
```
1. engine.tick(ts)
2. ctx.clearRect
3. ctx.save + screenShake translate
4. 카메라 배경 (미러링) 또는 그라디언트
5. 스켈레톤 (디버그 모드)
6. 하악 drawLowerJaw
7. 상악 drawUpperJaw
8. 양배추 draw
9. 파티클 draw
10. 플로팅텍스트 draw
11. 플래시 오버레이
12. 디버그: 충돌 박스
13. 엔딩 draw (콘페티 + 텍스트)
14. ctx.restore (쉐이크 해제)
15. HUD 업데이트 (DOM, running일 때만)
```

### 5.5 입력 처리

| 입력 | 게임 진행 중 | 게임 종료 (레벨 클리어) | 게임 종료 (시간 종료) |
|------|-------------|----------------------|---------------------|
| Space/Enter | tryBite() | 다음 레벨 | 레벨1부터 재시작 |
| Touch | tryBite() | 다음 레벨 | 레벨1부터 재시작 |
| 포즈: 팔내림 | tryBite() | — | — |

---

## 6. 시각 이펙트 상수 요약

| 이벤트 | screenShake | flash | particles |
|--------|-------------|-------|-----------|
| 빈 물기 | (4, 0.15) | — | — |
| 첫 물기 (attracted) | (10, 0.2) | — | — |
| 와구작 크런치 | (16, 0.3) | white 0.3 | biteSparkle(6) + crush(8) |
| 완식 폭발 | (35, 0.6) + 120ms후 (18, 0.3) | gold 0.7 + 120ms후 white 0.25 | emitEat(intensity=5, 90개) |
| 게임 종료 | (15, 0.4) | gold 0.6 | — |

---

## 7. CSS 핵심 값

```css
body { background: #111; font-family: 'Jua', 'Apple SD Gothic Neo', system-ui; }
.hud-level-badge { gradient: #FFD700→#FFA500; border: 3px #c8860a; radius: 14px; }
.hud-eaten-panel { bg: rgba(255,255,255,0.15); blur: 8px; radius: 16px; }
.timer-ring-fill { stroke: #7dd85a; urgent: #ff5555 + pulse animation; }
.hud-hp { bg: rgba(0,0,0,0.3); blur: 6px; radius: 20px; }
.ctrl button { bg: rgba(0,0,0,0.35); blur: 6px; radius: 20px; }
.pose-status { 4 states: idle(white), ready(green), detected(bright green scale 1.05), noperson(red) }
.start-overlay { gradient: 160deg #FFF6E8→#F0FFF6 }
```

---

## 8. 배포 (GitHub Pages)

- 리포: `Jeklinkimm/lion-lettuce`
- URL: `https://jeklinkimm.github.io/lion-lettuce/`
- 플랫 구조: index.html, engine.js, assets/ (basePath를 `assets`로 변경)
- `.github/workflows/deploy.yml`로 GitHub Actions 자동 배포

---

## 9. 재현 시 주의사항

1. **에셋 경로**: 로컬 개발은 `../../assets`, 배포는 `assets` — `Assets.load(basePath)` 파라미터로 조절
2. **SVG stroke-dashoffset**: `.style.strokeDashoffset`이 아닌 `.setAttribute('stroke-dashoffset', value)` 사용
3. **카메라 미러링**: drawImage 전 `ctx.translate(CW, 0); ctx.scale(-1, 1)` + 포즈 x좌표도 `1 - x`
4. **턱 위치 공식**: `lion.y`는 어깨 y + shoulderW * CH * 0.15 — 사용자 턱 바로 아래
5. **상악 오프셋**: `-UPPER_H * 0.72` (상악의 72%가 y 위에 위치)
6. **양배추 stuck 위치**: `lion.y + UPPER_H * 0.28 * sc + jawDown * 0.35` — 상악 하단에서 하악 방향으로 35%
7. **게임 종료 후 freeze 방지**: lion/particles/shake는 `!running`이어도 계속 update
8. **터치 이벤트**: `{ passive: true }` 필수
