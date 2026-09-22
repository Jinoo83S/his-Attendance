# HIS 출결

GitHub Pages + Firebase(`his-main`) + Google Sheets 백업으로 운영하는 출결 앱입니다.

## 현재 운영 기준

- Firebase가 출결 운영 원본입니다.
- Google Sheets는 열람·확인·장기 백업용입니다.
- 기존 AppSheet/Google Sheets → Firebase 이관은 완료된 일회성 작업입니다.
- 운영 화면에는 과거 시트 가져오기 기능을 표시하지 않습니다.
- 등록된 `@his.sc.kr` 교직원은 전체 학생의 출결 사유를 열람할 수 있습니다.
- 백업 실행과 설정 변경은 관리자만 할 수 있습니다.

## Firebase 저장 구조

```text
attendanceReports/{reportId}
  academicYear, monthKeys[]
  studentEmail, studentName, studentEngName, grade, cls
  startDate, endDate, category, detail, reason, periods[]
  attendanceType, createdBy, createdByName, createdAt
  synced, syncedAt
  documentSubmitted, documentCheckedAt
  documentCheckedBy, documentCheckedByName

attendanceDaily/{reportId_date}
  academicYear, monthKey
  reportId, date, studentEmail, studentName, studentEngName, grade, cls
  category, detail, reason, periods[], attendanceType
  createdBy, createdByName, createdAt
```

`attendanceReports`가 원본이고 `attendanceDaily`는 날짜별 조회를 위한 파생 자료입니다.
출결 1건 저장 시 Firestore 쓰기는 `1 + 출결 일수`입니다.

## Google Sheets 백업

백업 시트는 AppSheet 계산 열을 재현하지 않습니다. `attendanceReports` 한 문서가 시트 한 행이 됩니다.
`reportId`를 첫 열의 고유 키로 사용하므로 같은 자료를 다시 보내도 중복 행이 생기지 않습니다.

백업 열은 다음과 같습니다.

```text
reportId, 학년도, 시작일, 종료일, 학교학년, 학교반,
학생이름, 영문이름, 학생이메일, 분류, 상세, 출결구분,
사유, 교시, 등록자, 등록자이메일, 등록일시,
기존자료이관, 원본구분, 원본행, 시트저장일시
```

학년과 반은 학교 반 편성 기준입니다. NEIS 반으로 간주하거나 자동 변환하지 않습니다.

- `새 기록 백업`: `synced == false`인 기록만 조회하고 저장합니다.
- `현재 학년도 전체 백업`: 새 백업 시트의 최초 구성이나 복구에 사용합니다.
- 전체 백업도 시트에 이미 존재하는 `reportId`는 건너뜁니다.
- 시트에서 Firebase로 가져오는 API는 Apps Script v2에서 지원하지 않습니다.

## 출결상황보고서

`report.html`은 학년도·월·NEIS 학년·반별 출결 목록을 조회하는 전 교직원용 화면입니다.

- Firebase `attendanceReports.monthKeys`로 선택한 월의 보고서만 읽습니다.
- 같은 월에서 학년·반 조건을 바꿀 때는 메모리의 조회 결과를 재사용합니다.
- 학생 명단은 12시간 로컬 캐시를 사용합니다.
- 중등과 고등만 대상으로 하며 조회 기준은 NEIS로 고정합니다.
- 조회는 `neisClassGroup`, `neisCls`, `neisNum`을 사용합니다.
- 결과에는 비교 확인용 학교 반(`grade + cls`)도 함께 표시합니다.
- NEIS 값이 없으면 학교 반에서 추정하지 않고 제외 건수로 표시합니다.
- 여러 날짜에 걸친 출결은 선택한 월 안에서 날짜별 행으로 펼쳐 표시합니다.
- `서류` 상태는 모든 교직원이 볼 수 있고, 관리자 또는 `canManageAttendanceDocuments` 권한자만 변경합니다.
- 권한은 출결 앱의 `⚙ 관리자 → 교사 권한 관리` 또는 his-main 교직원 관리 화면에서 지정합니다.
- 출결 앱이 이미 읽은 NEIS 포함 학생명단을 보고서에 전달하여 재진입 시 명단 재조회를 줄입니다.
- 데스크톱·태블릿·모바일 조회와 A4 가로 인쇄/PDF를 지원합니다.

## 설정 위치

Apps Script 스크립트 속성:

```text
SYNC_TOKEN = 충분히 긴 임의 문자열
SPREADSHEET_ID = 1ZwJ6i0Vl_zSWpcQj7p6I96t_TCnCB5tMqJYUO1qVAhY
SHEET_NAME = 출결_백업
```

Firestore 관리자 전용 문서:

```text
adminSettings/attendanceSync
  token = Apps Script의 SYNC_TOKEN과 같은 값
  url   = Apps Script 웹 앱의 /exec 주소
```

웹 앱 주소와 토큰은 공개 `index.html`에 저장하지 않습니다.

## 무료 한도 고려

2026학년도 최초 전체 백업이 774건이라면 대략 다음 사용량이 발생합니다.

- Firestore 문서 읽기: 약 774건
- `settings/general` 쓰기: 1건
- `synced` 상태 쓰기: 아직 백업되지 않은 문서만 발생
- Google Sheets 쓰기: Apps Script가 한 번의 범위 쓰기로 처리

평상시에는 미백업 기록만 조회하므로 전체 자료를 매번 Firebase에서 읽지 않습니다.
장기 누적 자료의 보고서 조회는 학년도·월 단위 쿼리로 제한하는 후속 최적화가 필요합니다.

## 배포

정확한 적용 순서와 확인 방법은 `DEPLOYMENT.md`를 따릅니다.
