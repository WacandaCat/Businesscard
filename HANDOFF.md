# SLBS 명함 생성기 — 인수인계 (최종 갱신 2026-07-30)

## 프로젝트 개요
사내 명함 인쇄용 PDF 생성 웹앱. 단일 `index.html` (빌드 없음, 정적 배포).
디자인 템플릿은 고정, 인적 정보만 폼으로 입력 → 인쇄용 PDF 다운로드.
명단은 저장하지 않는 일회성 워크플로우.

- 원본 인쇄 파일: `260121_명함_SLBS_최종.pdf` (Illustrator 30.1) — **저장소에 넣지 말 것**
  (직원 개인정보 포함). 로컬 보관.
- 출력 규격: **86 × 52 mm** (243.78 × 147.40 pt) — 원본 파일은 88.5 × 58 mm
- 색상: **DeviceCMYK K100(먹1도)**, 도련 없음(배경 흰색)

## 배포 / 저장소
- GitHub: `WacandaCat/Businesscard` (**public**) — 브랜치 `main`
- Vercel 프로젝트: `slbs-namecard` (team `danny37park`), Git 연결 완료
- URL: https://slbs-namecard-danny37park.vercel.app (인증 없이 공개 접속)
- 배포 방식: **`main` push → Vercel 자동 배포**.
  커밋 author는 반드시 `Danny <daniel29park@gmail.com>` (자동배포 이슈 방지 규칙)
- 배포 보호: ssoProtection = `prod_deployment_urls_and_all_previews`
  (메인 주소는 공개, 과거 실데이터가 남은 개별 배포 URL만 보호)
- **주의**: 코드 수정·배포 창구는 Claude Code로 단일화. 다른 도구에서 직접 배포하면
  최신 작업을 덮어쓸 위험이 있음 (2026-07-30 실제 발생, preview URL이라 라이브는 무사)

## 아키텍처 (index.html 단일 파일)
핵심 원칙: **브랜드 벡터는 원본 유지, 가변 텍스트만 새로 생성**

1. **공용 뒷면**: 원본 1페이지 벡터를 base64 내장(`FRONT_PDF_B64`) → pdf-lib `embedPdf`.
   86×52 규격에서는 **95% 축소**(비율 유지, 좌측·하단 여백 고정 앵커)로 배치.
2. **정보면 로고 2종** (우상단 SLASH B SLASH® + 중단 SLBS®): 원본 2페이지에서 텍스트 제거 후
   벡터만 추출(`BACKLOGO_PDF_B64`). 규격이 줄어든 만큼 **상/하 분할 클립 배치**
   (`drawPageClipped`, `LOGO_SPLIT=60pt` 기준: 위는 상단 앵커, 아래는 하단 앵커).
3. **인적 정보 텍스트**: 캔버스 **1200dpi**(`BACK_DPI`) 렌더 → 알파 채널을 DeviceGray SMask로,
   본체는 1×1 **DeviceCMYK K100** 픽셀로 임베드 (`embedTextOverlayCMYK`).
   pdf-lib 저수준 API(flateStream/register/newXObject) 사용. 재단선도 `cmyk(0,0,0,1)`.
4. **미리보기**: 동일 소스 SVG(`FRONT_SVG`, `BACKLOGO_SVG`) + 캔버스 합성 (PDF와 동일 로직).
5. **외부 의존성 없음**: pdf-lib 1.17.1 → `assets/pdf-lib.min.js`,
   폰트 → `assets/fonts/*.woff2`. 빌드/서버/CDN 없음.

### 좌표 앵커 전략 (86×52 대응)
규격이 원본보다 작으므로 통째로 스케일하지 않고 요소별 앵커를 다르게 잡음:
- 상단 요소(이름 블록·주소·우상단 로고) = 상단 앵커 (원본 좌표 그대로)
- 하단 요소(SLBS® 로고·슬로건·사이트·뒷면 워드마크) = 하단 앵커 (`DY_BOT`만큼 위로)
- 결과: 위·아래 여백은 원본과 동일, 가운데 빈 공간만 축소. 로고·글자 크기는 원본 유지.

## 폰트
| 용도 | 폰트 | 파일 |
|---|---|---|
| 영문 이름 | Helvetica Now Display Bold | `assets/fonts/HNDisplayBold.woff2` |
| 영문 직함·팀 | Helvetica Now Text Bold | `assets/fonts/HNTextBold.woff2` |
| 연락처·주소·슬로건 | Helvetica Now Text Medium | `assets/fonts/HNTextMedium.woff2` |
| 국문 | Pretendard Variable (OFL) | `assets/fonts/PretendardVariable.woff2` |

- Helvetica Now는 **유료 폰트**. 저장소가 public이므로 **명함 사용 글리프만 남긴
  서브셋 woff2만 커밋**(14~18KB). 풀 폰트(otf/전체 woff2)는 절대 커밋 금지, 로컬 보관.
  웹 서빙 라이선스 범위는 회사 확인 사항.
- **캔버스 전용 폰트는 브라우저가 자동 로드하지 않음** → `ensureFonts()`로 명시적
  `document.fonts.load()` 후 렌더, `buildPDF()`도 로드 완료를 대기.
  (이 가드가 없으면 폴백 폰트(맑은 고딕 등)로 출력되는 버그 발생)

## 원본 서식 스펙 (Acrobat 패널 + PDF 추출로 확정)
| 요소 | 폰트 | 크기 | 행간(배) | 자간 |
|---|---|---|---|---|
| 영문 이름 | HN Display Bold | 13.60 | 1.20 | -0.27 |
| 국문 이름 \| 직급 | Pretendard Variable | 8 | 1.20 | **+0.96** |
| 영문 직함·팀 | HN Text Bold | 6.08 | 1.20 | -0.12 |
| 연락처·주소·슬로건 | HN Text Medium | 6.08 | 1.28 | -0.12 |

원본 raw baseline(pt): 이름 23.0 / 국문 31.82 / 직함 49.38 / 휴대폰·이메일 68.54·76.54 /
주소A 49.42·57.42 / 주소B 68.74·76.75 / 슬로건 130.60·138.61 / 사이트 151.03

## 기능 (현재 UI 기준)
- **인적 정보**: 영문/국문 이름, 국문 직급, 휴대폰, **팀 이름(선택)**, 영문 직함, 이메일
- **레이아웃 A/B** (라디오, 반영하기 버튼 위):
  - A = 팀·직함이 국문 이름 바로 아래 (dy 9.56 / 17.56)
  - B = 팀·직함을 주소 B 블록과 하단 정렬(dy 36.92 / 44.93), 한 줄 여백 후 연락처(60.93 / 68.93)
  - 두 버전 모두 **팀 이름이 비면 직함이 윗줄로** 올라감
- **자동 맞춤**: 좌측 박스에 `fitR:117.5` — 긴 텍스트(예: 긴 이메일)는 해당 줄만
  폰트 크기·자간을 비례 축소해 우측 컬럼 침범 방지
- **예시확인하기 (클릭)**: 예시 1명(홍길동 | 매니저, `tpl:true`)이 폼에 로드된 상태로 시작.
  수정 후 **반영하기**를 누르면 덮어씀(추가 아님) + `(예시탬플릿)` 표시 사라짐
- **공통 정보**(카드형 접이식): 주소 A-1/A-2, 주소 B-1/B-2, 슬로건 1/2, 사이트.
  비워두면 `COMMON_DEFAULT` 값으로 폴백
- **출력**: 재단선 옵션(6mm 여백) / **PDF 다운받기** 버튼 하나.
  파일명 = `날짜_명함_영문이름.pdf` (2명 이상이면 `_외N명`)
- 미리보기 상단에 "반영하기를 눌러야 파일에 반영됩니다" 안내 문구

## 검증 방법 (좌표·폰트 수정 시 필수)
로컬에서 헤드리스 Chromium으로 실제 앱을 구동해 PDF를 생성하고 원본과 대조:
1. 앱을 정적 서빙 → Playwright(`/opt/pw-browsers/chromium`)로 열기
2. `document.fonts.ready` 대기 후 `roster=[...]; buildPDF(roster)` 실행해 PDF 저장
3. PyMuPDF로 200dpi 렌더 → 좌/우 컬럼별 흑픽셀 **세로 밴드** 추출해 원본과 y좌표 비교
   (목표 ±2px), 필요 시 줄별 **가로 폭**도 비교
4. CMYK 확인: `get_pixmap(colorspace=fitz.csCMYK)`로 획 내부가 C0 M0 Y0 K100인지

최근 검증 결과: 세로 밴드 전체 ±1px, 주요 줄 가로 폭 오차 0.00pt, 텍스트 K100 확인,
생성 PDF 페이지 크기 86.00 × 52.00 mm 실측.

## 결정사항
- **명단 영속화 안 함**: DB·localStorage 미사용. 사용 시마다 입력 → PDF 다운로드
- **저장소 public 유지**: roster는 가명 샘플만 커밋, 실직원 데이터 커밋 금지
- 위치 수동 조정(오브젝트 이동) 기능은 **넣지 않음** — 템플릿 좌표 고정이 설계 원칙.
  긴 텍스트는 자동 맞춤으로 대응

## 남은 과제
1. **재단선 옵션 인쇄소 확인**: crop marks 켠 출력을 첫 발주 때 1회 검증
2. 뒷면 워드마크 95% 축소가 인쇄 실물에서 적절한지 확인

## 파일 구성
```
index.html                      # 앱 전체 (배포 대상)
assets/pdf-lib.min.js           # pdf-lib 1.17.1 (MIT)
assets/fonts/*.woff2            # Helvetica Now 서브셋 3종 + Pretendard
assets/front_vector.pdf         # 공용면 벡터 (경량, HTML 내장분과 동일)
assets/backlogo_vector.pdf      # 정보면 로고 벡터 (경량)
assets/*_icc.pdf                # ICC 보존 원본 백업
assets/front.svg, backlogo.svg  # 미리보기용 SVG (HTML 내장분과 동일)
```

## 기타 주의사항
- pdf-lib은 object-streams 파싱 이슈 가능 → 내장 PDF 재생성 시 `qpdf --object-streams=disable`
- 좌표 단위는 전부 pt (mm 아님). 캔버스는 `setTransform(scale)`로 pt 좌표계 유지
- Acrobat 편집 모드에서 정보면 텍스트가 "맑은 고딕"으로 보이는 것은 **정상** —
  텍스트가 이미지로 들어가 Acrobat이 OCR로 추정한 결과(폰트명 앞 `*` = 대체 폰트).
  실제 인쇄 폰트와 무관하며, **OCR 상태로 저장하지 말 것**
