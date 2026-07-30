# SLBS 명함 생성기 — Claude Code 인수인계 (2026-07-30)

## 프로젝트 개요
사내 명함 인쇄용 PDF 생성 웹앱. 단일 `index.html` (빌드 없음, 정적 배포).
디자인 템플릿은 고정, 인적 정보만 폼으로 입력 → 원본 인쇄 파일과 동일한 구성의 PDF 출력.

- 원본 인쇄 파일: `260121_명함_SLBS_최종.pdf` (Illustrator 30.1, 10p = 앞면 1 + 인원별 정보면 9)
- 규격: **88.5 × 58 mm** (250.866 × 164.41 pt), 도련 없음(배경 흰색), 색상 K100 계열

## 현재 배포 상태
- Vercel 프로젝트: `slbs-namecard` (danny37park / wacandacat)
- URL: https://slbs-namecard-danny37park.vercel.app
- **Vercel Authentication: 전체 배포에 활성화됨** (직원 연락처 내장 → 해제 금지)
- 최초 배포는 Vercel MCP 직접 배포. **이후 운영은 표준 워크플로우로 전환할 것:**
  1. GitHub 저장소 생성 (예: `WacandaCat/slbs-namecard`) 후 이 폴더 push
  2. Vercel 대시보드 → slbs-namecard → Settings → Git 에서 저장소 연결
  3. 이후 모든 배포는 `git push` (커밋 author: `Danny <daniel29park@gmail.com>` — Vercel 자동배포 이슈 방지 규칙)

## 아키텍처 (index.html 단일 파일)
핵심 원칙: **브랜드 벡터는 원본 유지, 가변 텍스트만 새로 생성**

1. **앞면(공용)**: 원본 1페이지를 벡터 PDF로 추출·경량화하여 base64 내장
   (`FRONT_PDF_B64`) → pdf-lib `embedPdf`로 출력 PDF에 그대로 삽입.
   원본 대비 렌더 차이 0.0/255 검증 완료.
2. **정보면 로고 2종** (우상단 SLASH B SLASH® + 중단 SLBS®): 원본 2페이지에서
   텍스트(BT..ET) 제거 후 벡터만 추출 (`BACKLOGO_PDF_B64`) → 동일하게 벡터 삽입.
3. **인적 정보 텍스트**: 캔버스에 600dpi 투명 PNG로 렌더 → 벡터 로고 페이지 위에 오버레이.
4. **미리보기**: 동일 소스의 SVG(`FRONT_SVG`, `BACKLOGO_SVG`) + 캔버스 합성.
5. 외부 의존성 없음 (2026-07-30 셀프호스팅 전환): pdf-lib 1.17.1 → `assets/pdf-lib.min.js`,
   Pretendard Variable(OFL) → `assets/fonts/PretendardVariable.woff2`. 빌드/서버 없음.

### 경량화 이력 (재작업 시 참고)
- 원본 페이지의 `/PieceInfo` (AI 편집 데이터, ~770KB) 제거
- ICCBased CMYK 프로파일(557KB) → `/DeviceCMYK` 치환 (CMYK 수치 보존, 렌더 차이 ≤0.34/255)
- 결과: 앞면 2.3KB / 로고 3.7KB. ICC 보존 원본은 `assets/*_icc.pdf`에 백업

## 텍스트 좌표 (원본 pdfplumber 추출 + 대체폰트 보정, 단위 pt, baseline 기준)
| 요소 | 폰트/크기 | x | baseline |
|---|---|---|---|
| 영문 이름 | EN 800 / 13.6 | 13.62 | 23.2 |
| 국문 이름 \| 직급 | KR 500 / 8 | 13.7 | 31.8 |
| 영문 직함 | EN 700 / 6.08 | 13.7 | 49.0 |
| 휴대폰 / 이메일 | EN 500 / 6.08 | 14.18 | 68.35 / 76.55 |
| 서울 주소 1·2 | 〃 | 121.98 | 49.1 / 57.1 |
| 부산 주소 1·2 | 〃 | 121.98 | 68.35 / 76.55 |
| 슬로건 1·2 / 사이트 | 〃 | 122.15 | 130.15 / 138.15 / 150.65 |

- 원본 폰트 (2026-07-30 새 기준 파일에서 확정): 영문 이름 = Helvetica Now **Display Bold**,
  영문 직함 = Helvetica Now **Text Bold**, 연락처·주소·슬로건 = Helvetica Now **Text Medium**,
  국문 = Pretendard Variable. 원본 raw baseline: 이름 23.0 / 국문 31.82 / 직함 49.38 /
  휴대폰·이메일 68.54·76.54 / 서울 49.42·57.42 / 부산 68.74·76.75 / 슬로건 130.60·138.61 / 사이트 151.03
- 현재 대체: Helvetica Neue/Arial + Pretendard(CDN). 위 baseline은 대체폰트 기준으로
  원본 렌더와 밴드 비교하여 ±0.5pt 이내로 보정된 값 (원본 raw 값과 다름 주의)

## 검증 완료 항목
- 앞면 벡터: 원본 200dpi 렌더와 픽셀 diff 0.0
- 정보면: 텍스트 세로 밴드 10개 전부 원본 ±2px(@200dpi) 이내
- 9명 전체 출력(10p PDF) Chromium 실환경 생성 성공, pdffonts/pdfinfo 정상

## 남은 과제 (우선순위순)
1. **폰트 정합**: 회사 보유 Helvetica Now otf/ttf 확보 시 `@font-face`로 교체하고
   baseline 재보정 (보정값이 대체폰트 기준이므로 재측정 필요. 검증 스크립트는 아래 참고)
2. **재단선 옵션 검증**: crop marks 켠 출력을 인쇄소에 1회 확인

## 결정사항 (2026-07-30)
- **명단 영속화 안 함**: DB(Supabase)·localStorage 연동하지 않음. 사용 시마다 폼에 입력
  → PDF 다운로드하는 일회성 워크플로우로 확정. 기본 roster는 입력 형식 예시용 샘플 1명(홍길동)만 유지.
- 코드·배포본에 실직원 데이터가 전혀 없으므로 접근 정책(Vercel Authentication)은
  유지 여부를 자유롭게 선택 가능 (해제해도 개인정보 노출 없음).

## 검증 스크립트 패턴 (좌표 수정 시 반드시 사용)
```bash
# 원본과 생성본 렌더 비교
pdftoppm -png -r 200 원본.pdf ref
pdftoppm -png -r 200 생성본.pdf out
# PIL로 흑픽셀 세로 밴드 추출하여 라인별 y 비교 (±2px 이내 목표)
```

## 파일 구성
```
index.html                      # 앱 전체 (배포 대상)
assets/front_vector.pdf         # 앞면 벡터 (경량, HTML 내장분과 동일)
assets/backlogo_vector.pdf      # 정보면 로고 벡터 (경량)
assets/front_vector_icc.pdf     # ICC 보존 원본 백업
assets/backlogo_vector_icc.pdf  # ICC 보존 원본 백업
assets/front.svg, backlogo.svg  # 미리보기용 SVG (HTML 내장분과 동일)
```
원본 `260121_명함_SLBS_최종.pdf`는 저장소에 넣지 말 것(직원 개인정보). 로컬 보관.

## 주의사항
- **저장소 공개(public) 운영 정책 (2026-07-30 변경)**: 이 저장소(`WacandaCat/Businesscard`)는
  public이므로 roster에는 **가명 샘플 데이터만** 커밋할 것 (현재 홍길동 등 9명 더미).
  실제 직원 명단(휴대폰·이메일)은 절대 커밋 금지 — 로컬 보관 또는 남은 과제 3번(Supabase 분리) 참고.
  Vercel 배포본에 실데이터를 쓰려면 데이터를 코드에서 분리한 뒤 비공개 저장소/DB에서 주입할 것.
- pdf-lib은 object-streams 형식 파싱 이슈 가능 → 내장 PDF 재생성 시
  `qpdf --object-streams=disable` 사용
- 좌표 단위는 전부 pt (mm 아님). 캔버스는 `setTransform(scale)`로 pt 좌표계 유지
