# HIS 출결체크

AppSheet 출결체크 앱 → GitHub Pages + Firebase(his-main 공유) + Google Sheets 마이그레이션.

## 배포
1. 이 폴더를 새 GitHub repo에 push
2. repo Settings → Pages → Source를 main 브랜치(루트)로 설정
3. `firestore-rules-addition.txt` 내용을 기존 `his-main` 프로젝트의 `firestore.rules`에 병합 후:
   ```
   firebase deploy --only firestore:rules
   ```
   (규칙 배포 전에는 로그인은 되지만 학생 검색/저장이 막혀 있습니다.)

## 현재 구현 범위 (v1 – 1차)
- Google 로그인(@his.sc.kr, his-main과 동일 프로젝트 공유) + `staff` 컬렉션 존재 여부로 접근 제어
- 출결 입력: 학생 검색 → 기간/분류/상세/사유/교시선택 → 저장
- 저장 시 `attendanceReports`(원본) + `attendanceDaily`(일자별 펼침, 조회 전용) 동시 기록
- 최근 기록 탭: 본인이 입력한 최근 20건

## 스키마
```
attendanceReports/{reportId}
  studentEmail, studentName, grade, cls
  startDate, endDate, category, detail, reason, periods[]
  attendanceType            // detail+category 자동조합 (예: 질병결석)
  createdBy, createdByName, createdAt
  synced, syncedAt          // 시트 동기화 상태 (다음 단계에서 사용)

attendanceDaily/{reportId_date}
  reportId, date, studentEmail, studentName, grade, cls
  category, detail, reason, periods, attendanceType, createdBy, createdByName
```

## 중요 발견 — 교사 명단 재사용
새 컬렉션을 만들지 않고 **기존 `staff` 컬렉션을 그대로 사용**합니다 (his-main admin.html에서
이미 이메일/이름/부서/직위/담임 등을 관리 중). 단, 붙여주신 교사 데이터의 `최종이름`(표시명)
필드는 기존 staff import 매핑에는 없는 항목입니다. 지금은 `displayName` 필드가 있으면
우선 사용하고, 없으면 `nameKo`/`nameEn`으로 자동 대체하도록 짜뒀습니다 — 나중에 admin.html의
교직원 일괄 등록 매핑에 `최종이름 → displayName`만 추가하면 자동으로 반영됩니다 (백로그).

## 백로그 (v1 제외)
- Block B 파생 컬럼(급별 상대학년, 공휴일, 서류 등)
- 서류(첨부파일) 업로드
- Google Workspace Admin SDK Directory API 연동
- 오늘 출결 + 달력형 전체 조회 화면
- 구글시트 배치 동기화 (Apps Script Web App 호출)
- admin.html에 `최종이름 → displayName` 매핑 추가

## 다음 계획
| # | 작업 |
|---|---|
| 1 | 실제 배포 후 로그인 → 학생 검색 → 저장 → 최근기록 흐름 검증 |
| 2 | 오늘 출결 + 달력 뷰 화면 |
| 3 | 구글시트 배치 동기화(Apps Script Web App) 연결 |
| 4 | admin.html displayName 매핑 추가 |
