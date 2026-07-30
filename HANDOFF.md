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
3. **인적 정보 텍스트**: 캔버스에 1200dpi로 렌더 → 알파 채널을 DeviceGray SMask로,
   본체는 1×1 DeviceCMYK **K100** 픽셀로 임베드 (먹1도 인쇄용, 재단선도 cmyk K100).
   pdf-lib 저수준 API(flateStream/register/newXObject) 사용 — `embedTextOverlayCMYK()` 참고.
4. **미리보기**: 동일 소스의 SVG(`FRONT_SVG`, `BACKLOGO_SVG`) + 캔버스 합성.
5. 외부 의존성 없음 (2026-07-30 셀프호스팅 전환): pdf-lib 1.17.1 → `assets/pdf-lib.min.js`,
   Pretendard Variable(OFL) → `assets/fonts/PretendardVariable.woff2`. 빌드/서버 없음.

### 경량화 이력 (재작업 시 참고)
- 원본 페이지의 `/PieceInfo` (AI 편집 데이터, ~770KB) 제거
- ICCBased CMYK 프로파일(557KB) → `/DeviceCMYK` 치환 (CMYK 수치 보존, 렌더 차이 ≤0.34/255)
- 결과: 앞면 2.3KB / 로고 3.7KB. ICC 보존 원본은 `assets/*_icc.pdf`에 백업

## 원본 서식 전체 스펙 (2026-07-30, Acrobat 패널 + PDF 추출로 확정)
| 요소 | 폰트 | 크기 | 행간(배) | 자간 | 단락간격 |
|---|---|---|---|---|---|
| 영문 이름 | Helvetica Now Display **Bold** | 13.60 | 1.20 | -0.27 | 0 |
| 국문 이름 \| 직급 | Pretendard Variable | 8 | 1.20 | **+0.96** | 10.18 |
| 영문 직함 | Helvetica Now Text **Bold** | 6.08 | 1.20 | -0.12 | 11.56 |
| 휴대폰 / 이메일 | Helvetica Now Text **Medium** | 6.08 | 1.28 | -0.12 | 0 |
| 주소 (서울/부산) | 〃 | 6.08 | 1.28 | -0.12 | 3.53 |
| 슬로건 / 사이트 | 〃 | 6.08 | 1.28 | -0.12 | — |

- 원본 raw baseline: 이름 23.0 / 국문 31.82 / 직함 49.38 / 휴대폰·이메일 68.54·76.54 /
  서울 49.42·57.42 / 부산 68.74·76.75 / 슬로건 130.60·138.61 / 사이트 151.03
- **실폰트 전환 완료 (2026-07-30)**: Helvetica Now Display Bold / Text Bold / Text Medium
  woff2를 `assets/fonts/`에 셀프호스팅. 코드 좌표는 원본 raw baseline, 자간도 전 요소 적용.
  검증 결과: 세로 밴드 전체 ±1px, 가로 폭 8개 요소 중 6개 0.00pt·나머지 ±0.36pt.
  (대체폰트 보정값은 git 히스토리 c213cd7 참고)

## 검증 완료 항목
- 앞면 벡터: 원본 200dpi 렌더와 픽셀 diff 0.0
- 정보면: 텍스트 세로 밴드 10개 전부 원본 ±2px(@200dpi) 이내
- 9명 전체 출력(10p PDF) Chromium 실환경 생성 성공, pdffonts/pdfinfo 정상

## 남은 과제
1. **재단선 옵션 검증**: crop marks 켠 출력을 인쇄소에 1회 확인

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
- **저장소 공개(public) 운영 (2026-07-30 확정)**: roster는 가명 샘플만 커밋(비저장 워크플로우).
  Helvetica Now는 유료 폰트이므로 **사용 글리프만 남긴 서브셋 woff2만 커밋**(14~18KB) —
  풀 폰트 파일(otf/전체 woff2)은 절대 커밋 금지, 로컬 보관. 웹 서빙 라이선스 범위는 회사 확인 사항.
- pdf-lib은 object-streams 형식 파싱 이슈 가능 → 내장 PDF 재생성 시
  `qpdf --object-streams=disable` 사용
- 좌표 단위는 전부 pt (mm 아님). 캔버스는 `setTransform(scale)`로 pt 좌표계 유지
