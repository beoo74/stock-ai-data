# Stock AI 자동 분석 알림 시스템

Google Sheets의 `forGhatGPT` 시트를 원본으로 사용해 GitHub `forGhatGPT.csv`를 자동 갱신하고, ChatGPT가 평일 오전 9:01에 해당 CSV를 읽어 주식 분석 알림을 보내는 구조입니다.

---

## 1. 전체 구조

```text
Google Sheets forGhatGPT
→ Apps Script가 CSV 생성
→ GitHub forGhatGPT.csv 갱신
→ ChatGPT가 GitHub raw CSV 읽기
→ 최신 시세/뉴스/공시/수급 기반 분석 알림
```

---

## 2. GitHub 저장소 생성

GitHub에서 새 저장소를 만듭니다.

권장 설정:

| 항목 | 값 |
|---|---|
| Repository name | `stock-ai-data` |
| Visibility | Public |
| Add README | 선택 |
| .gitignore | 없음 |
| License | 없음 |

Public으로 만드는 이유는 ChatGPT가 별도 권한 없이 raw CSV를 읽을 수 있게 하기 위해서입니다.

생성 후 저장소 주소:

```text
https://github.com/beoo74/stock-ai-data
```

---

## 3. GitHub CSV 파일 생성

GitHub 저장소에서 아래 파일을 만듭니다.

```text
forGhatGPT.csv
```

초기 내용:

```csv
ticker,name,avg_price,position_weight,holding_type
```

파일 생성 경로:

```text
GitHub 저장소 → Add file → Create new file
→ 파일명: forGhatGPT.csv
→ 내용 입력
→ Commit changes
```

ChatGPT가 읽는 raw CSV 주소:

```text
https://raw.githubusercontent.com/beoo74/stock-ai-data/main/forGhatGPT.csv
```

GitHub 파일 화면에서 `Raw` 버튼을 누르면 raw 주소를 확인할 수 있습니다.

---

## 4. GitHub Fine-grained Token 생성

Apps Script가 GitHub의 `forGhatGPT.csv`를 자동 갱신하려면 GitHub Token이 필요합니다.

GitHub에서는 가능하면 classic token보다 fine-grained personal access token을 쓰는 것이 안전합니다. fine-grained token은 특정 저장소와 필요한 권한만 제한해서 줄 수 있습니다.

### 생성 경로

```text
GitHub 오른쪽 위 프로필 아이콘
→ Settings
→ Developer settings
→ Personal access tokens
→ Fine-grained tokens
→ Generate new token
```

### 기본 설정

| 항목 | 값 |
|---|---|
| Token name | `forGhatGPT-csv-uploader` |
| Expiration | 90일 또는 1년 |
| Resource owner | `beoo74` |
| Repository access | Only select repositories |
| Selected repository | `stock-ai-data` |

### Repository permissions

아래 권한만 설정합니다.

| 권한 | 값 |
|---|---|
| Contents | Read and write |
| Metadata | Read-only |

필요 없는 권한:

```text
Actions
Administration
Issues
Pull requests
Workflows
Secrets
```

이 자동화는 GitHub 파일 하나를 읽고 업데이트하는 작업만 하므로 `Contents: Read and write`면 충분합니다.

### 토큰 생성 후 주의

토큰은 생성 직후 한 번만 보입니다.

```text
토큰을 복사해서 Apps Script Script Properties에 저장
채팅이나 README에 토큰을 붙여넣지 않기
코드 안에 직접 넣지 않기
```

---

## 5. Apps Script에 GitHub Token 저장

Google Sheets에서:

```text
확장 프로그램 → Apps Script
```

Apps Script 왼쪽 메뉴에서:

```text
프로젝트 설정 ⚙️
→ 스크립트 속성
→ 스크립트 속성 추가
```

아래처럼 저장합니다.

| 속성 | 값 |
|---|---|
| `GITHUB_TOKEN` | GitHub에서 만든 fine-grained token |

이렇게 해야 코드에 토큰이 노출되지 않습니다.

---

## 6. Google Sheets 구조

시트 이름:

```text
forGhatGPT
```

첫 행 컬럼:

```csv
ticker,name,avg_price,position_weight,holding_type
```

예시:

```csv
ticker,name,avg_price,position_weight,holding_type
000660,SK하이닉스,1971825,5.97,core
005930,삼성전자,291434,11.02,core
063080,컴투스홀딩스,34249,18.14,trade
329750,TIGER 미국달러단기채권액티브,13902,1.89,hedge
```

| 컬럼 | 설명 |
|---|---|
| `ticker` | 종목 코드. 한국 종목은 앞자리 0 유지 |
| `name` | 종목명 |
| `avg_price` | 평균 매수 단가 |
| `position_weight` | 포트폴리오 내 비중, % 기준 |
| `holding_type` | 보유 목적 |

`holding_type` 추천 값:

| 값 | 의미 |
|---|---|
| `core` | 장기 핵심 보유 |
| `trade` | 단기/스윙 매매 |
| `hedge` | 방어/분산용 |
| `income` | 배당/분배금 목적 |
| `speculative` | 고위험 테마/코인성 자산 |

---

## 7. Apps Script 전체 코드

Google Sheets에서:

```text
확장 프로그램 → Apps Script
```

아래 코드로 교체합니다.

```javascript
const SHEET_ID = "구글 시트 ID";
const SHEET_NAME = "구글 시트 Name";

const GITHUB_OWNER = "GitHub 아이디";
const GITHUB_REPO = "stock-ai-data";
const GITHUB_BRANCH = "main";
const GITHUB_PATH = "forGhatGPT.csv";

function syncForGhatGPTToGitHub() {
  const csv = buildForGhatGPTCsv_();

  const token = PropertiesService
    .getScriptProperties()
    .getProperty("GITHUB_TOKEN");

  if (!token) {
    throw new Error("GITHUB_TOKEN script property is missing.");
  }

  const apiUrl =
    "https://api.github.com/repos/" +
    GITHUB_OWNER + "/" +
    GITHUB_REPO +
    "/contents/" +
    encodeURIComponent(GITHUB_PATH);

  const currentFile = getCurrentGitHubFile_(apiUrl, token);

  const payload = {
    message: "Update forGhatGPT.csv from Google Sheets",
    content: Utilities.base64Encode(csv, Utilities.Charset.UTF_8),
    branch: GITHUB_BRANCH
  };

  if (currentFile && currentFile.sha) {
    payload.sha = currentFile.sha;
  }

  const response = UrlFetchApp.fetch(apiUrl, {
    method: "put",
    contentType: "application/json",
    headers: {
      Authorization: "Bearer " + token,
      Accept: "application/vnd.github+json",
      "X-GitHub-Api-Version": "2022-11-28"
    },
    payload: JSON.stringify(payload),
    muteHttpExceptions: true
  });

  const code = response.getResponseCode();
  const body = response.getContentText();

  if (code < 200 || code >= 300) {
    throw new Error("GitHub update failed. HTTP " + code + "\n" + body);
  }

  Logger.log("GitHub CSV updated successfully.");
  Logger.log(body);

  return body;
}

function buildForGhatGPTCsv_() {
  const ss = SpreadsheetApp.openById(SHEET_ID);
  const sheet = ss.getSheetByName(SHEET_NAME);

  if (!sheet) {
    throw new Error("Sheet not found: " + SHEET_NAME);
  }

  const values = sheet.getDataRange().getDisplayValues();

  if (!values || values.length === 0) {
    throw new Error("Sheet is empty: " + SHEET_NAME);
  }

  const headers = values[0].map(h => String(h).trim());

  const requiredColumns = [
    "ticker",
    "name",
    "avg_price",
    "position_weight",
    "holding_type"
  ];

  const columnIndexes = requiredColumns.map(col => headers.indexOf(col));
  const missingColumns = requiredColumns.filter((col, i) => columnIndexes[i] === -1);

  if (missingColumns.length > 0) {
    throw new Error(
      "Required columns not found: " +
      missingColumns.join(", ") +
      ". Current headers: " +
      JSON.stringify(headers)
    );
  }

  const lines = [requiredColumns.join(",")];

  values.slice(1).forEach(row => {
    const ticker = normalizeTicker_(row[columnIndexes[0]]);
    const name = String(row[columnIndexes[1]]).trim();
    const avgPrice = String(row[columnIndexes[2]]).trim();
    const positionWeight = String(row[columnIndexes[3]]).trim();
    const holdingType = String(row[columnIndexes[4]]).trim();

    if (!ticker || !name) {
      return;
    }

    lines.push([
      csvEscape_(ticker),
      csvEscape_(name),
      csvEscape_(avgPrice),
      csvEscape_(positionWeight),
      csvEscape_(holdingType)
    ].join(","));
  });

  return lines.join("\n") + "\n";
}

function getCurrentGitHubFile_(apiUrl, token) {
  const url = apiUrl + "?ref=" + encodeURIComponent(GITHUB_BRANCH);

  const response = UrlFetchApp.fetch(url, {
    method: "get",
    headers: {
      Authorization: "Bearer " + token,
      Accept: "application/vnd.github+json",
      "X-GitHub-Api-Version": "2022-11-28"
    },
    muteHttpExceptions: true
  });

  const code = response.getResponseCode();
  const body = response.getContentText();

  if (code === 404) {
    return null;
  }

  if (code < 200 || code >= 300) {
    throw new Error("GitHub get file failed. HTTP " + code + "\n" + body);
  }

  return JSON.parse(body);
}

function normalizeTicker_(value) {
  const ticker = String(value).trim();

  if (!ticker) {
    return "";
  }

  if (/^\d+$/.test(ticker)) {
    return ticker.padStart(6, "0");
  }

  return ticker;
}

function csvEscape_(value) {
  const s = String(value);

  if (/[",\n\r]/.test(s)) {
    return '"' + s.replace(/"/g, '""') + '"';
  }

  return s;
}

function testBuildCsv() {
  const csv = buildForGhatGPTCsv_();
  Logger.log(csv);
}

function doGet() {
  const csv = buildForGhatGPTCsv_();

  return ContentService
    .createTextOutput(csv)
    .setMimeType(ContentService.MimeType.CSV);
}
```

---

## 8. 실행 테스트

1. Apps Script 저장
2. 함수 선택에서 `testBuildCsv` 실행
3. 로그에서 CSV 출력 확인
4. 함수 선택에서 `syncForGhatGPTToGitHub` 실행
5. 로그에 아래 문구가 나오면 성공

```text
GitHub CSV updated successfully.
```

그 다음 GitHub의 `forGhatGPT.csv`가 갱신됐는지 확인합니다.

---

## 9. Apps Script 트리거

월요일~금요일 각각 1개씩, 총 5개 트리거를 설정합니다.

| 항목 | 값 |
|---|---|
| 실행 함수 | `syncForGhatGPTToGitHub` |
| 배포 | `Head` |
| 이벤트 | 시간 기반 |
| 유형 | 주 단위 타이머 |
| 요일 | 월~금 각각 |
| 시간 | 오전 8시~9시 |

Apps Script는 정확한 분 단위 지정이 어려우므로, ChatGPT 자동 분석보다 앞선 시간대에 실행합니다.

---

## 10. ChatGPT 자동 알림

| 항목 | 값 |
|---|---|
| 실행 주기 | 월~금 |
| 실행 시간 | 오전 9:01 |
| 입력 데이터 | GitHub raw `forGhatGPT.csv` |
| 분석 방식 | 계좌 기준 판단 |
| 알림 | ChatGPT 알림 |

분석에는 `avg_price`, `position_weight`, `holding_type`을 내부적으로 반영합니다.

---

## 11. 분석 결과 형식

표에서 제외:

```text
현재가/최근가
평균단가
비중
```

표에 포함:

```text
ticker
name
holding_type
손익률
판단
오늘 액션
신뢰도
과거 판단 검증
핵심 근거
리스크/무효화 조건
```

판단 라벨:

| 판단 | 의미 |
|---|---|
| 매수 | 신규 또는 추가 매수 가능 |
| 보유 | 유지 |
| 일부매도 | 수익 보호 또는 리스크 축소 |
| 매도 | 정리 또는 강한 비중 축소 |
| 관망 | 신규매수/매도 보류 |

---

## 12. 현재 완료 상태

| 항목 | 상태 |
|---|---|
| `forGhatGPT` 시트 | 완료 |
| GitHub `stock-ai-data` 저장소 | 완료 |
| `forGhatGPT.csv` 자동 갱신 | 완료 |
| Apps Script 월~금 트리거 | 완료 |
| ChatGPT 오전 9:01 자동 알림 | 완료 |
| 계좌 기준 컬럼 반영 | 완료 |
| 판단 한국어화 | 완료 |
| 표 간소화 | 완료 |
| 신뢰도 포함 | 완료 |
| 과거 판단 검증 반영 | 완료 |
