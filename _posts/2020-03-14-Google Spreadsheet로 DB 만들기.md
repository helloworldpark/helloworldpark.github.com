---
layout: post
title:  "Google Spreadsheet로 DB 만들기"
date:   2020-03-14 00:00:00 +0900
categories: jekyll update
---

# DB값이 너무 아까운데요
개인적으로 돌리고 있는 서버가 있습니다. 혼자 봇을 돌리기 위해 쓰는 아주 작은 서버인데요, 그래도 은근 쓰는 기능이 많아서 다음과 같이 구성되어 있습니다.
 - **Google Cloud Compute Engine**
 - **Google Cloud SQL**
 - **Google Cloud Storage**
 - **Google Cloud Stackdrive**

이 중에서 **Google Cloud Storage**와 **Stackdrive**는 구글님이 주시는 무료 사용량으로 충분해서 불만이 없습니다. Compute Engine도 어떻게어떻게 잘 맞춰서 필요한 만큼은 괜찮은 성능을 잘 보여줍니다. 문제는 Google Cloud SQL. 사양은 다음과 같습니다.
 - vCPU: 1개
 - 메모리: 614MB
 - SSD 저장소: 10GB
     - 현재 사용량: 1.5GB

성능상 이슈 없이 잘 쓰고 있지만 문제는 가격. 이 정도 사양이면 매달 11,000원이 약간 안 되게 청구됩니다. 사실 더 불만이었던 것은 이 사양도 다 활용을 안 하고 있다는 것입니다. 많이 쓰지도 않는데 기본 사양이 저 정도라 어쩔 수 없이 매달 11,000원을 낭비하는 느낌입니다.
반면에 대체재로 찾아본 **Google Sheets API**는 무료에 그저 조금 불편하고 느린 정도? 였다면 얼마나 좋아겠습니까마는... 검색해보니 시트로 디비를 만들어본 사례가 없지도 않아서 할짓도 없는 차에 만들어보았습니다.

# Google Spreadsheet를 써보자!

## 어떤 DB를 만들어볼까
어차피 완벽한 DB는 만들 수 없습니다. 유명 DB 벤더들조차 다 조금씩 다른 판에 기능 몇 개 빠져도 상관없겠죠. 그래서 다음을 만족하면 스펙 완료!로 하겠습니다.
1. 필수 스펙
 - Upsert
 - Select
 - Delete
 - Create Table
 - Delete Table

2. 되면 좋은 것
 - (X) Primary Key: 고민 끝에 쓸모없어서 미구현
 - (O) Unique Constraint

## Google Sheets API 사용하기
쓰기 더럽게 나쁩니다. 제가 뭔가 제대로 초기화하지 않아서인지 access token을 http 헤더에 넣어주는 로직을 직접 구현해야 해서 매우 귀찮았습니다. Google에서 개발한 [Go 클라이언트](https://developers.google.com/sheets/api/quickstart/go)를 써도 제대로 작동하지 않는 경우도 있었고, 그 클라이언트가 모든 기능을 제공해주는 것도 아니었습니다. 공식 문서도 설명이 부실해서 답답했었습니다. 

### 스프레드시트 파일을 DB처럼
여기에서 사용 중인 API는 [Google Sheets API v4](https://developers.google.com/sheets/api), [Google Drive API v3](https://developers.google.com/drive/api/v3/about-sdk)입니다.
열(Column)은 A열~Z열까지 26개를 쓸 수 있습니다.
1. 헤더
메타데이터를 보관하는 영역입니다.
 - 1행: 테이블의 Column name을 기록합니다.
 - 2행: 테이블의 Column의 데이터타입을 기록합니다. reflect.Kind 를 string으로 표현했을 때의 이름입니다.
 - 3행
     - 1열: 유효한 데이터의 총 갯수입니다. 삭제의 효율성을 위해 데이터를 다 지우지는 않고, '어디까지가 데이터!' 이런 식으로 표기합니다.
     - 2열: 테이블의 Column의 갯수입니다.
     - 3열: (선택) Unique Constraint의 JSON 표현입니다.
2. 데이터
데이터를 기록하는 영역입니다. ```A1```셀 값만큼의 행이 데이터입니다(예를 들어, ```A4:G100```).

### Access Token 집어넣기 
예제에 나온 것처럼 구글에서 제공해준 Go client를 사용했는데 자꾸 40X error가 발생하더군요. 하라는 대로 했더니 니잘못 하는 인성에도 불구하고 비슷한 사례를 찾아보니 Access Token을 HTTP 헤더에 직접 넣어줘야 하더군요. 아래 코드는 특정 Range(```A4:G10``` 이런 거요)의 값을 시트로부터 읽어오는 코드입니다.
```go
type httpValueRangeRequest struct {
	manager       *SheetManager
	ranges        string
	spreadsheetID string
}
func (r *httpValueRangeRequest) Do() *sheets.ValueRange {
	r.manager.refreshToken()
	req := r.manager.service.Spreadsheets.Values.Get(r.spreadsheetID, r.ranges)
	req.Header().Add("Authorization", "Bearer "+r.manager.token.AccessToken)
	valueRange, err := req.Do()
	if err != nil {
		panic(err)
	}
	return valueRange
}
```

### 스프레드시트 파일 삭제하기 
스프레드시트 내의 내용을 삭제하는 것이 아니라, 스프레드시트 자체를 삭제하는 방법입니다. API v4에서 시트 관련해 소개하는 함수들을 보면 create는 있어도 delete나 remove 같은 삭제 함수는 없습니다. 
그렇다고 삭제 방법이 없냐 하면 그건 또 아닙니다. Google Drive API로 우회해서 파일 자체를 삭제할 수 있습니다. 코드를 보시면 알겠지만 파일명은 스프레드시트의 ID와 같습니다.
```go
// deleteSpreadsheet deletes spreadsheet file with `spreadsheetId`
// Returns true if deleted(status code 20X)
//         false if else
// https://stackoverflow.com/questions/46836393/how-do-i-delete-a-spreadsheet-file-using-google-spreadsheets-api
// https://stackoverflow.com/questions/46310113/consume-a-delete-endpoint-from-golang
// api count: 1
func (m *SheetManager) deleteSpreadsheet(spreadsheetID string) bool {
	resp := newURLRequest(m, delete, fmt.Sprintf("https://www.googleapis.com/drive/v3/files/%s", spreadsheetID)).Do()
	return resp.StatusCode/100 == 2
}
```

## 필수 스펙

### Upsert
Insert문 돌리다가 duplicated keys라며 꽥!하고 죽어버리는 게 너무 싫어서 처음부터 Upsert로 개발했습니다. 배열을 통째로 덮어쓰는 방식으로 구현한 후에, 그 배열을 필터링하는 로직을 넣고, 나중에 인덱싱까지 추가하다보니 함수가 너무 지저분해져서 UpsertIf 라는 함수 하나로 통일합니다.

주의할 점은, 스프레드시트에 Request를 날려서 업로드할 때 ```[][]interface{}```로 준비해서 날려야 한다는 것입니다. 즉, ```[]mystruct```가 있다면 리플렉션으로 ```mystruct```의 각 필드를 ```interface{}```로 나눠줘야 한다는 것이죠.
```go
r.manager.refreshToken()
batchRequest := &sheets.BatchUpdateValuesRequest{}
batchRequest.IncludeValuesInResponse = true
batchRequest.ValueInputOption = "RAW"
batchRequest.Data = make([]*sheets.ValueRange, 2)
rangeValues := &sheets.ValueRange{}
rangeValues.Range = r.rangeValues
rangeValues.Values = r.updatingValues

rangeRows := &sheets.ValueRange{}
rangeRows.Range = r.rangeRows
rangeRows.Values = r.updatingRows

batchRequest.Data[0] = rangeValues
batchRequest.Data[1] = rangeRows

req := r.manager.service.Spreadsheets.Values.BatchUpdate(r.spreadsheetID, batchRequest)
req.Header().Add("Authorization", "Bearer "+r.manager.token.AccessToken)
updatedRange, err := req.Do()
```

### Select
API에 AddFilterRequest가 있길래 설렜습니다만 네 그건 그냥 필터를 추가하는 겁니다. 필터를 적용한 결과를 받는 게 아닙니다. 아무리 찾아봐도 방법이 없길래 그냥 DB 전체를 다운받고 서버에서 필터링하는 식으로 구현했습니다.
```go
r.manager.refreshToken()
req := r.manager.service.Spreadsheets.Values.Get(r.spreadsheetID, r.ranges)
req.Header().Add("Authorization", "Bearer "+r.manager.token.AccessToken)
valueRange, err := req.Do()
if err != nil {
    panic(err)
}
return valueRange
```

### Delete
역시나 조건문 걸어서 삭제하는 API같은 건 존재하지 않습니다. 그냥 DB 전체를 다운받고 남겨둘 애들만 필터링한 후에 덮어쓰는 식으로 구성했습니다.
```go
// download all data
data, scheme := table.selectData(-1)
if data == nil {
	return nil
}

// delete if predicate==true
newData := make([]interface{}, 0)
deletedIndex := make([]int64, 0)
for i, values := range data {
	if deleteThis(values) {
		// add to deleted index
		deletedIndex = append(deletedIndex, int64(i))
	} else {
		newData = append(newData, values)
	}
}

if len(deletedIndex) == 0 {
	return nil
}

// update deleted data
table.upsertIf(newData, false)
```

# 아니 구글양반, 1003행밖에 못쓴다고요?
결론적으로 이 계획은 실패합니다. 아주 작은 DB 정도로는 쓸 수 있을지 몰라도 하루에 800개씩 레코드를 싸지를 계획이었다면 인생을 너무 꽁으로 먹으려던 것이지요.

## 테이블 하나당 1000개만
![1000딸라](/images/2020-03-14-sheet01.png)
이런 중요한 정보를 VSCode 검은창으로 알게 될 줄은 몰랐습니다. 원래는 1003행까지 무료인데, 첫 3행은 메타데이터용으로 따로 빼두었으니 실질적으로는 1000행만 무료로 쓸 수 있습니다. 하루에 나오는 데이터가 800행인데, 저에게는 전혀 맞지 않는 것입니다.
제가 **Cloud SQL**을 **Sheets**로 대체하는 걸 포기한 결정적인 이유입니다. 이 제한에 관해서는 어디서도 본 적이 없고, 어디에 얘기하면 이 제한을 풀 수 있는지, 그 가격이 얼마인지 현재는 전혀 알 수 없기 때문입니다. 

## 100초당 Quota는 1인당 100개
[Usage Limits](https://developers.google.com/sheets/api/limits)에 대해 설명한 글을 보면 다음과 같습니다.
 - 프로젝트당: 리퀘스트 500회 / 100초
 - 유저당: 리퀘스트 100회 / 100초

100초라는 저 애매한 숫자도 그렇지만, 어떻게 저 지표를 계산하는지도 명확하지 않아서 참 궁금합니다. 이 제한을 넘기면 429에러가 뜨며 제한을 넘겼다고 돌아옵니다.
이를 회피하는 방법으로 API를 호출하는 함수가 **Sheets API**를 몇 개나 호출하는지 다 세놓고, 이를 래핑하는 함수에서 API Usage를 체크하는 방식을 쓸 수는 있습니다(어차피 안 쓸 거지만).
```go
func (m *SheetManager) enqueueAPIUsage(task int64, blockIfQuota bool) {
    // https://hakurei.tistory.com/193 [Reimu's Development Blog])
    // Google uses Pacific Time Zone
	now := time.Now().In(time.FixedZone("GMT-7", -7*60*60)).Unix()
	if m.lastQuotaTime == 0 {
		m.lastQuotaTime = (now / 100) * 100
	} else if m.lastQuotaTime+100 <= now {
		m.lastQuotaTime = (now / 100) * 100
	}
	// check if api usage is enough
	if m.apiUsage >= 90 {
		if blockIfQuota {
			timeToWait := time.Second * time.Duration(m.lastQuotaTime+100-now)
			fmt.Printf("[%v] Pending %v seconds, api usage: %v\n", now, m.lastQuotaTime+100-now, m.apiUsage)
			<-time.NewTimer(timeToWait).C
		}

        // Reset quota
		now = time.Now().In(time.FixedZone("GMT-7", -7*60*60)).Unix()
		fmt.Printf("[%v] Resetting quota\n", now)
		m.lastQuotaTime += 100
		m.apiUsage = 0
	}
	m.apiUsage += task
}
```
```go
// Select Selects all the rows from the table
func (table *Table) Select(rows int64) ([][]interface{}, *TableScheme) {
	table.manager.enqueueAPIUsage(1, true)
	return table.selectData(rows)
}
```

## 처참한 속도
속도도 처참합니다. 아무리 비교대상인 DB가 빠르다지만 좀 심한 편입니다. Quota 때문에 테스트 시간도 오래 걸려서 함수당 10번밖에 못했습니다. 구현상 더 극복할 수도 있겠지만, Cloud SQL과 너무 심하게 비교가 납니다.
```go
func BenchmarkSelect(t *testing.B) {
	// create table
	t.StopTimer()
	manager := NewSheetManager(jsonPath)
	db := manager.FindDatabase("testdb")
	if db == nil {
		t.Fatalf("Sheet %s is nil", "testdb")
	}

	// Find or make table
	table := db.FindTable(TestStructMeme{})
	if table == nil {
		table = db.createTable(TestStructMeme{})
	} 
	deleted := table.Drop()

	table = db.createTable(TestStructMeme{})
	
	values, _ := createRandomDataMeme()
	table.UpsertIf(values, true)
	for i := 0; i < 10; i++ {
		table.manager.enqueueAPIUsage(1, true)
		t.StartTimer()
		table.selectData(-1)
		t.StopTimer()
	}
	now := time.Now().In(time.FixedZone("GMT-7", -7*60*60)).Unix()
	next := ((now / 100) + 1) * 100
	<-time.NewTimer(time.Duration(next-now+1) * time.Second).C
}
```
```
// Benchmark
// 6 columns, 100 rows
// 20 times
goos: darwin
goarch: amd64
pkg: github.com/helloworldpark/gsheet-db-go
Spreadsheet-Select   	       1	4717255702 ns/op	 1290888 B/op	   18005 allocs/op
CloudSQL-Select                1	 587846045 ns/op	  646080 B/op	   16600 allocs/op
Spreadsheet-Upsert   	       1	15953589274 ns/op	225378560 B/op	 2257679 allocs/op
CloudSQL-Upsert                1	1355422415 ns/op	  950720 B/op	    6433 allocs/op
```

# 결론
돈 받는데는 다 이유가 있으니 그냥 쓰는 게 좋겠습니다.
