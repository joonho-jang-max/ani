# APNG 최적화 작업 히스토리

집 컴퓨터에서 이어서 작업할 때 이 파일을 Claude에게 읽으라고 하면 컨텍스트 복원됩니다.

## 원본 정보

- 파일명: `tests.png` (현재 이 Mac의 `~/Desktop/tests.png`)
- 포맷: APNG, 120×130, RGBA
- 용량: 1054976 bytes (~1030KB)
- 고유 프레임 수: 49 (ffmpeg `vsync 0` 기준)
- 콘텐츠: 검은 고양이가 밤톨(?) 먹는 애니메이션 + 초록색 배경 모양 (말풍선 비슷한 모양)

## 원본 프레임 타이밍 (중요)

- 프레임 1: **300ms** (시작 hold)
- 프레임 2~31: **30ms** 각각 (애니메이션 1 — 약 1.2초간)
- 프레임 32: **4000ms** (중간 정지, 이 4초짜리 pause는 반드시 유지해야 함)
- 프레임 33~49: **30ms** 각각 (애니메이션 2 — 마지막 0.5초)

## 목표 / 제약조건

- 용량 예산: 처음 30KB → 50KB → 55KB → 최종 77KB (타협됨)
- 해상도: 120×130 유지 (원본 그대로)
- 컬러: **최소 32색** 필요 (24색은 부족, 11색은 색 뭉개짐)
- pause 프레임(4초) 유지 필수
- 프레임 너무 적으면 움직임이 끊겨 보임 (유저가 민감함)

## 최종 선택 버전

**[`ani.png`](./ani.png)** (= 이 Mac의 `tests_77kb_18f_32color.png`)

- 용량: **77KB**
- 프레임 수: **18** (원본 49 → 약 1/3)
- 컬러: **32색** (pngquant)
- 해상도: 120×130 (원본 유지)
- pause 프레임 유지됨

## 시도한 조합들 (참고용)

| step | colors | frames | size | 비고 |
|---|---|---|---|---|
| 2 | 8 | 25 | 45KB | 프레임 많지만 색 거침 |
| 2 | 10 | 25 | 49.5KB | |
| 2 | 11 | 25 | 50.7KB | 11색이 sweet spot |
| 2 | 12 | 25 | 70KB | 12색부터 palette 점프 |
| 2 | 16 | 25 | 82KB | |
| 2 | 32 | 25 | 112KB | 25프레임 + 32색은 절대 55KB 불가 |
| 3 | 16 | 18 | 43KB | |
| 3 | 32 | 18 | **77KB** | ← 최종 채택 |
| 4 | 24 | 14 | 54KB | |
| 4 | 32 | 14 | 60KB | |
| 5 | 32 | 12 | 52KB | |

**pngquant palette 계단 현상:** 컬러 수가 특정 경계(8→12, 16→18 등)를 넘으면 파일 사이즈가 확 점프함. 이유: 4-bit → 5-bit palette 인코딩 전환.

## 현재 상태 / 문제

- `ani.png` (77KB, 18프레임, 32색)은 퀄리티 OK
- 근데 GitHub Pages에서 렌더링 했을 때 **버벅임 발생** (브라우저의 APNG 렌더링 성능 문제로 추정)
- 해결 방향: **고양이와 초록 배경을 분리**하는 쪽으로 전환

## 다음 단계 (집에서 이어갈 작업)

### 아이디어: 배경과 고양이 분리

초록 배경 모양을 APNG에서 빼고:
1. 고양이만 투명 배경 APNG로 저장 (용량 확 줄어듬)
2. 초록 배경 모양은 정적(애니메이션 없음) 이미지 or SVG로 분리
3. HTML에서 두 개 겹쳐서 렌더링 → 브라우저 부담 감소, 퀄리티 유지

### AE 작업 필요 (유저가 할 일)

유저가 After Effects에서 **Track Matte** 방식으로 초록 Shape Layer를 고양이 마스크로 사용 중. 단순히 Shape Layer 눈알 끄면 고양이도 사라짐.

**해결책:**
1. 모든 레이어 선택 → 우클릭 → **Pre-compose** (move all attributes into new comp)
2. Pre-comp 안으로 진입
3. `Shape Layer 1` → Contents → Rectangle 1 → Gradient Fill 1 → **Opacity 0%** 로 설정
   - 레이어 가시성(눈알)이 아니라 **Fill의 Opacity**를 내려야 함
   - 이래야 alpha 채널은 유지되면서 초록 색만 사라져서 track matte는 정상 동작
4. Render Queue → PNG Sequence, RGB+Alpha, Millions of Colors+
5. 만약 안 되면 (Luma matte일 경우) → 해당 고양이 레이어의 Track Matte 설정을 Alpha Matte로 변경

### 그 다음 Claude가 할 작업

유저가 투명 PNG Sequence 주면:
1. `ffmpeg -vsync 0`으로 고유 프레임 추출
2. `pngquant`로 컬러 팔레트 축소 (배경 없어서 고양이 디테일에 32색 전부 배정 가능)
3. `apngasm`으로 원본 타이밍(300ms / 30ms×N / 4000ms / 30ms×N) 복원해서 재조립
4. 초록 배경 모양:
   - **옵션 A:** 단일 프레임 PNG로 export해서 정적 배경으로 사용 (1~2KB)
   - **옵션 B:** SVG path로 변환해서 CSS 배경에 사용
5. HTML에서 배경 이미지/SVG 위에 투명 APNG 올림

## 환경 정보 (참고)

- 이번 작업 Mac: darwin 24.6.0
- 설치된 도구: `pngquant`, `apngasm`, `ffmpeg` (brew로 설치됨)
- 집 Mac에도 없으면 `brew install pngquant apngasm ffmpeg` 필요
- 로컬 작업 폴더 (이 Mac): `/tmp/apng_work/` (집에서는 새로 만들면 됨)
- 빌드 스크립트 (이 Mac에만 있음, 참고용):
  ```bash
  # /tmp/apng_work/build.sh 가 있었음
  # 사용법: ./build.sh <step> <colors> <output.png>
  # step = 몇 프레임마다 하나씩 샘플링 할지 (2 = 절반, 3 = 1/3)
  # colors = pngquant 컬러 수
  # 원본 타이밍 (300/30×N/4000/30×N)을 재구성해서 apngasm에 per-frame delay로 전달
  ```

## 관련 링크

- GitHub 레포: https://github.com/joonho-jang-max/ani
- GitHub Pages: https://joonho-jang-max.github.io/ani/
- 이 레포에는 `ani.png` (최종 선택 버전 77KB) + `index.html` (60×65로 half-size 렌더링) 이 들어있음
