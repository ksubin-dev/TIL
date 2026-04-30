# 📝 [TIL] Paging 3와 Coroutines: 상태 관리와 스트림의 역할 분담

## 오늘의 핵심 요약

> 프레임워크가 제공하는 도구와 내가 만든 도구의 역할을 명확히 분리하자.  
> 무한 스크롤은 Paging 3에게 맡기고, 단발성 작업은 Custom Result 래퍼로 처리한다.

이번 프로젝트에서 Paging 3를 적용하면서 `suspend`, `Flow`, `PagingData`, `LoadResult`, 그리고 직접 만든 `DataResult`의 역할이 헷갈렸다.

특히 다음 두 가지가 가장 고민이었다.

- `Flow<PagingData<Book>>`를 반환하는 함수에도 `suspend`가 필요할까?
- Paging 3 내부에서도 내가 만든 `DataResult`를 사용해야 할까?

결론부터 말하면, **Flow를 반환하는 함수에는 보통 `suspend`가 필요하지 않고**, **Paging 3 내부에서는 Custom Result 래퍼보다 Paging 3가 제공하는 `LoadResult`를 사용하는 것이 더 적절하다.**

<br/>

## 1. Flow 반환 함수에서 `suspend`를 제거한 이유

기존에는 API에서 데이터를 가져올 때 `suspend` 함수를 사용했다.

```kotlin
suspend fun searchBooks(query: String): DataResult<List<Book>>
```

이런 형태는 단발성 API 요청에는 적절하다.  
함수를 호출하면 네트워크 요청을 수행하고 응답이 올 때까지 해당 코루틴을 일시 중단한 뒤 결과를 반환하기 때문이다.

하지만 Paging 3를 도입하면서 반환 타입이 다음과 같이 변경되었다.

```kotlin
fun searchBooks(query: String): Flow<PagingData<Book>>
```

여기서 중요한 점은 `Flow`는 **콜드 스트림(Cold Stream)** 이라는 것이다.

`Flow` 객체를 생성하고 반환하는 것 자체는 실제 데이터를 가져오는 작업이 아니다.  
즉, `Flow`를 반환하는 함수는 “데이터를 즉시 가져오는 함수”라기보다는 “데이터가 흘러갈 파이프라인을 만들어 반환하는 함수”에 가깝다.

실제 데이터 로드는 UI 계층에서 `collect`하거나, Paging 3의 경우 `collectAsLazyPagingItems()`를 호출하여 스트림을 구독하기 시작할 때 발생한다.

```kotlin
val books = viewModel.books.collectAsLazyPagingItems()
```

즉, 비동기 작업은 `Flow`를 만드는 순간이 아니라 **Flow를 수집하는 시점**에 시작된다.

<br/>

## 결론

`Flow`를 반환하는 함수는 보통 `suspend`일 필요가 없다.

```kotlin
fun getPagedBooks(query: String): Flow<PagingData<Book>>
```

이 형태가 더 자연스럽다.

비유하자면, `Flow`를 반환하는 함수는 수도관을 설치하는 작업이고, `collect`는 수도꼭지를 트는 작업이다.  
수도관을 설치한다고 바로 물이 흐르지는 않는다. 물은 수도꼭지를 틀었을 때 흐르기 시작한다.

<br/>

## 2. Custom DataResult와 Paging 3의 LoadResult

프로젝트에서는 안정적인 에러 핸들링을 위해 직접 만든 `DataResult`를 사용하고 있었다.

```kotlin
sealed interface DataResult<out T> {
    data class Success<T>(val data: T) : DataResult<T>
    data class Error(val exception: Throwable) : DataResult<Nothing>
}
```

이 구조는 단발성 API 요청이나 DB 작업에서 성공과 실패를 명확하게 나누는 데 유용하다.

그래서 Paging 3를 적용할 때도 처음에는 이런 고민을 했다.

> “무한 스크롤 API 응답도 `DataResult`로 감싸야 하지 않을까?”

하지만 결론적으로 Paging 3 내부에서는 Custom `DataResult`를 사용할 필요가 없었다.

<br/>

## 3. PagingSource는 이미 LoadResult를 사용한다

Paging 3의 핵심인 `PagingSource`는 `load()` 함수에서 반드시 `LoadResult`를 반환하도록 설계되어 있다.

```kotlin
override suspend fun load(params: LoadParams<Int>): LoadResult<Int, Book>
```

`LoadResult`는 Paging 3가 제공하는 자체 상태 래퍼다.

```kotlin
LoadResult.Page(
    data = books,
    prevKey = prevKey,
    nextKey = nextKey
)
```

```kotlin
LoadResult.Error(exception)
```

즉, Paging 3는 이미 다음 상태를 자체적으로 관리한다.

- 데이터 로드 성공
- 데이터 로드 실패
- 이전 페이지 키
- 다음 페이지 키
- 로딩 상태
- 에러 상태

그리고 이 상태들은 UI 계층에서 `LoadState`로 전달된다.

```kotlin
books.loadState.refresh
books.loadState.append
```

따라서 Paging 3 내부에서 굳이 `DataResult`로 한 번 감싸고, 다시 `LoadResult`로 변환할 필요가 없다.

<br/>

## 4. DataResult를 PagingSource 안에서 쓰지 않은 이유

만약 PagingSource 내부에서 API 응답을 `DataResult`로 감싼다면 다음과 같은 구조가 된다.

```kotlin
when (val result = repository.searchBooks(query)) {
    is DataResult.Success -> {
        LoadResult.Page(...)
    }

    is DataResult.Error -> {
        LoadResult.Error(result.exception)
    }
}
```

물론 이렇게 구현할 수는 있다.  
하지만 Paging 3에서는 어차피 최종적으로 `LoadResult.Page` 또는 `LoadResult.Error`를 반환해야 한다.

즉, 다음과 같은 이중 래핑 구조가 된다.

```text
API Result
   ↓
DataResult
   ↓
LoadResult
   ↓
PagingData
   ↓
LoadState
   ↓
UI
```

이렇게 되면 코드가 더 안전해진다기보다는, 오히려 중간 변환 코드만 늘어나고 역할이 모호해진다.

프레임워크가 이미 상태 관리 방식을 제공하는 영역에서는 그 규칙을 따르는 편이 더 깔끔하다.

<br/>

## 5. 역할 분담 기준

이번 고민을 통해 데이터 처리 기준을 다음처럼 정리할 수 있었다.

<br/>

## 🌊 스트림 데이터

연속적으로 데이터가 흘러오거나, 상태가 계속 바뀌는 데이터는 `Flow`를 사용한다.

### 사용 형태

```kotlin
fun getBooks(query: String): Flow<PagingData<Book>>
```

### 사용 상황

- 도서 검색 결과 무한 스크롤
- 로컬 DB 데이터 실시간 관찰
- 북마크 목록 변화 감지
- 검색어 상태 변화에 따른 결과 갱신

### 특징

- 데이터를 한 번만 가져오는 것이 아니라 계속 관찰한다.
- collect 시점에 실행된다.
- Paging 3와 함께 사용할 경우 로딩, 에러, 성공 상태를 `LoadState`로 관리할 수 있다.

<br/>

## 🎯 단발성 데이터

한 번 요청하고 결과를 받는 작업은 `suspend`와 Custom `DataResult`를 사용한다.

### 사용 형태

```kotlin
suspend fun getBookDetail(bookId: String): DataResult<Book>
```

### 사용 상황

- 특정 도서 상세 정보 1건 조회
- 북마크 추가
- 북마크 삭제
- 서버에 특정 요청을 한 번 보내는 작업

### 특징

- 호출 시점에 비동기 작업이 실행된다.
- 성공과 실패를 `DataResult`로 명확하게 분기할 수 있다.
- UI에서 토스트, 스낵바, 에러 메시지 등을 처리하기 좋다.

<br/>

## 6. 최종 정리

이번에 정리한 기준은 다음과 같다.

| 상황 | 사용 방식 | 상태 처리 |
| --- | --- | --- |
| 무한 스크롤 | `Flow<PagingData<T>>` | Paging 3의 `LoadResult`, `LoadState` |
| 로컬 DB 실시간 관찰 | `Flow<T>` | Flow collect로 상태 반영 |
| 단발성 API 요청 | `suspend fun` | Custom `DataResult` |
| 단발성 DB 작업 | `suspend fun` | 성공/실패 필요 시 `DataResult` |
| PagingSource 내부 로딩 | `suspend load()` | `LoadResult.Page`, `LoadResult.Error` |

<br/>

## 오늘 배운 점

- `suspend`는 “비동기 작업의 결과를 기다리는 함수”에 적합하다.
- `Flow`는 데이터를 즉시 가져오는 것이 아니라, 데이터 흐름을 표현하는 스트림이다.
- `Flow`를 반환하는 함수는 보통 `suspend`일 필요가 없다.
- Paging 3는 자체적으로 `LoadResult`와 `LoadState`를 제공한다.
- 프레임워크가 이미 상태 관리 방식을 제공한다면, Custom Wrapper를 억지로 끼워 넣지 않는 편이 낫다.
- 내가 만든 도구와 프레임워크가 제공하는 도구의 역할을 구분하는 것이 중요하다.

<br/>

## 한 줄 회고

Paging 3를 사용하면서 중요한 것은 단순히 데이터를 페이지 단위로 불러오는 것이 아니라,  
**어떤 상태 관리는 프레임워크에 맡기고, 어떤 상태 관리는 직접 설계할지 구분하는 것**이었다.
