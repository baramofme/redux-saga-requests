# redux-saga-requests

[![npm version](https://badge.fury.io/js/redux-saga-requests.svg)](https://badge.fury.io/js/redux-saga-requests)
[![gzip size](http://img.badgesize.io/https://unpkg.com/redux-saga-requests/dist/redux-saga-requests.min.js?compression=gzip)](https://unpkg.com/redux-saga-requests)
[![dependencies](https://david-dm.org/klis87/redux-saga-requests.svg?path=packages/redux-saga-requests)](https://david-dm.org/klis87/redux-saga-requests?path=packages/redux-saga-requests)
[![dev dependencies](https://david-dm.org/klis87/redux-saga-requests/dev-status.svg?path=packages/redux-saga-requests)](https://david-dm.org/klis87/redux-saga-requests?path=packages/redux-saga-requests&type=dev)
[![peer dependencies](https://david-dm.org/klis87/redux-saga-requests/peer-status.svg?path=packages/redux-saga-requests)](https://david-dm.org/klis87/redux-saga-requests?path=packages/redux-saga-requests&type=peer)
[![Build Status](https://travis-ci.org/klis87/redux-saga-requests.svg?branch=master)](https://travis-ci.org/klis87/redux-saga-requests)
[![Coverage Status](https://coveralls.io/repos/github/klis87/redux-saga-requests/badge.svg?branch=master)](https://coveralls.io/github/klis87/redux-saga-requests?branch=master)
[![Known Vulnerabilities](https://snyk.io/test/github/klis87/redux-saga-requests/badge.svg)](https://snyk.io/test/github/klis87/redux-saga-requests)
[![lerna](https://img.shields.io/badge/maintained%20with-lerna-cc00ff.svg)](https://lernajs.io/)
[![code style: prettier](https://img.shields.io/badge/code_style-prettier-ff69b4.svg?style=flat-square)](https://github.com/prettier/prettier)

AJAX 요청 처리를 단순화하는 Redux-Saga 애드온. Axios, Fetch API 및 GraphQL을 지원하지만 플러그인 방식으로 구현되므로 다른 통합을 추가 할 수 있습니다.

## Table of content

- [동기](#motivation-arrow_up)
- [설치](#installation-arrow_up)
- [사용](#usage-arrow_up)
- [Actions](#actions-arrow_up)
- [Selectors](#selectors-arrow_up)
- [Reducers](#reducers-arrow_up)
- [Middleware](#middleware-arrow_up)
- [handleRequests](#handleRequests-arrow_up)
- [sendRequest](#sendRequest-arrow_up)
- [Interceptors](#interceptors-arrow_up)
- [FSA](#fsa-arrow_up)
- [Usage with Fetch API](#usage-with-fetch-api-arrow_up)
- [Usage with GraphQL](#usage-with-graphql-arrow_up)
- [Mocking](#mocking-arrow_up)
- [Multiple drivers](#multiple-drivers-arrow_up)
- [Normalisation](#normalisation-arrow_up)
- [React bindings](#react-bindings-arrow_up)
- [예시](#examples-arrow_up)

## Motivation [:arrow_up:](#table-of-content)

`redux-saga-requests '에서`axios'를 사용한다고 가정하면 다음과 같은 방식으로 코드를 리팩터링 할 수 있습니다:
```diff
  import axios from 'axios';
- import { takeEvery, put, call } from 'redux-saga/effects';
+ import { all } from 'redux-saga/effects';
+ import { handleRequests } from 'redux-saga-requests';
+ import { createDriver } from 'redux-saga-requests-axios'; // or another driver

  const FETCH_BOOKS = 'FETCH_BOOKS';
- const FETCH_BOOKS_SUCCESS = 'FETCH_BOOKS_SUCCESS';
- const FETCH_BOOKS_ERROR = 'FETCH_BOOKS_ERROR';

- const fetchBooks = () => ({ type: FETCH_BOOKS });
- const fetchBooksSuccess = data => ({ type: FETCH_BOOKS_SUCCESS, data });
- const fetchBooksError = error => ({ type: FETCH_BOOKS_ERROR, error });
+ const fetchBooks = () => ({
+   type: FETCH_BOOKS,
+   request: {
+     url: '/books',
+     // you can put here other Axios config attributes, like method, data, headers etc.
+   },
+ });

- const defaultState = {
-   data: null,
-   pending: 0, // number of pending FETCH_BOOKS requests
-   error: null,
- };
-
- const booksReducer = (state = defaultState, action) => {
-   switch (action.type) {
-     case FETCH_BOOKS:
-       return { ...defaultState, pending: state.pending + 1 };
-     case FETCH_BOOKS_SUCCESS:
-       return { ...defaultState, data: action.data, pending: state.pending - 1 };
-     case FETCH_BOOKS_ERROR:
-       return { ...defaultState, error: action.error, pending: state.pending - 1 };
-     default:
-       return state;
-   }
- };

- const fetchBooksApi = () => axios.get('/books');
-
- function* fetchBooksSaga() {
-   try {
-     const response = yield call(fetchBooksApi);
-     yield put(fetchBooksSuccess(response.data));
-   } catch (e) {
-     yield put(fetchBooksError(e));
-   }
- }
-
  const configureStore = () => {
+   const { requestsReducer, requestsSagas } = handleRequests({
+     driver: createDriver(axios),
+   });
+
    const reducers = combineReducers({
-     books: booksReducer,
+     requests: requestsReducer,
    });

    const sagaMiddleware = createSagaMiddleware();
    const store = createStore(
      reducers,
      applyMiddleware(sagaMiddleware),
    );

    function* rootSaga() {
-     yield takeEvery(FETCH_BOOKS, fetchBooksSaga);
+     yield all(requestsSagas);
    }

    sagaMiddleware.run(rootSaga);
    return store;
  };
```
`redux-saga-requests`를 사용하면, 더 이상 오류 처리 또는로드 스피너 표시와 같은 작업을 수행하기 위해 오류 및 성공 작업을 정의 할 필요가 없습니다. 
반복적 인 사가 및 감속기와 관련된 요청을 작성할 필요도 없습니다. selector 작성에 대해 걱정할 필요조차 없습니다, 이 라이브러리는 최적화 된 선택기를 즉시 제공하므로.
`redux-actions` 같은 액션 헬퍼 라이브러리를 쓰면, 상수를 쓸 필요조차 없습니다.!
따라서 기본적으로 전체 원격 상태를 관리하는 작업을 작성하므로 Redux 앱에서 더 이상 유명한 보일러플레이트가 없습니다!

여기에서이 라이브러리가 제공하는 기능 목록을 볼 수 있습니다:
- AJAX 요청을 `{ type: FETCH_BOOKS, request: { url: '/books' } }` 와 같이 간단한 액션으로 정의 가능, 그리고 `success`, `error` (`abort` 또한 지원됩니다, 아래를 참조하십시오) 액션은 자동으로 디스패치 됩니다
- `success`, `error`, `abort` 함수는, 요청 작업 유형에 정확하고 일관된 접미사를 추가하므로 리듀서/사가/미들웨어의 응답 작업에 쉽게 대응할 수 있습니다.
- `handleRequests` 함수는, 이 라이브러리가 작동하는 데 필요한 모든 것을 제공합니다.
- 자동 및 구성 가능한 요청 abort, 성능을 높이고 경쟁 조건 버그가 발생하기도 전에 방지합니다.
- 한 번의 작업으로 여러 요청 보내기 - `{ type: FETCH_BOOKS_AND_AUTHORS, request: [{ url: '/books' }, { url: '/authors}'] }`
두 개의 요청을 보내고 `Promise.all` 로 감쌀 것입니다.
- 선언적 프로그래밍 - 이 라이브러리의 아이디어는 모든 요청 논리를 조치 내부에 캡슐화하는 것입니다, 따라서 액션, 리듀서, 사가 및 미들웨어 사이에 더 이상 흩어진 로직이 없습니다.
- Axios, Fetch API 및 GraphQL 지원 - 추가 고객을 추가 할 수 있습니다,하나의 앱 내에서 모두 사용할 수 있습니다, 자신의 클라이언트 통합을 `드라이버`로 작성할 수도 있습니다. (see [./packages/redux-saga-requests-axios/src/axios-driver.js](https://github.com/klis87/redux-saga-requests/blob/master/packages/redux-saga-requests-axios/src/axios-driver.js)
for the example)
- optimistic updates 지원, 따라서 요청이 완료되기 전에도 뷰를 업데이트 할 수 있습니다, optimistic updates를 되돌려 오류가 발생하더라도 일관성을 유지합니다.
- TTL을 통한 캐시 지원, 항상 서버에서 가져올 필요가없는 데이터에 대한 반복적 인 요청을 피할 수 있습니다.
- mocking -테스트 목적으로 또는 아직 구현되지 않은 API와 통합하려는 경우 사용할 수있는 모의 드라이버 (API가 완료되면 드라이버를 Axios 또는 Fetch로 변경하면 모든 것이 작동합니다.!)
- 여러 드라이버 지원 - 예를 들어 요청의 한 부분에는 Axios를 사용하고 다른 부분에는 Fetch Api를 사용할 수 있습니다
- compatible with FSA, `redux-act` 와 `redux-actions` 라이브러리 (see [redux-act example](https://github.com/klis87/redux-saga-requests/tree/master/examples/redux-act-integration))
- 서버 측 렌더링과 함께 사용하기 간단 - 하나의 추가 옵션을`handleRequests '에 전달하면 앱이 SSR을 준비합니다!
- `onRequest`, `onSuccess`, `onError`, `onAbort` 인터셉터, 당신은 주어진 이벤트 타입을 위핸 전역 행위를 정의하기 위해 당신의 sagas(또는 간단한 함수)를 그들에게 부착 할 수 있습니다 
- 선택적인 `requestsPromiseMiddleware`, 요청 액션 디스패치를 프로미스화 시킵니다, 따라서 리액트 컴포넌트에서 기다렸다가 요청 응답을 얻을 수 있습니다. `redux-thunk`로 하는 것과 같은 방법으로
- 원격 상태를 검색하는 고도로 최적화 된 셀렉터
- 자동 (그러나 선택적인) 데이터 정규화, 수동 데이터 업데이트를 잊을 수 있습니다, graphql 세계에서와 마찬가지로 보편적으로 사용 가능!
- `redux-saga-requests-react` 패키지의 리액트 바인딩

## Installation [:arrow_up:](#table-of-content)

패키지를 설치하려면 다음을 실행하십시오:
```
$ npm install redux-saga-requests
```
또는 당신은 CDN을 사용할 수 있습니다: `https://unpkg.com/redux-saga-requests`.

또한 드라이버를 설치해야합니다:
- Axios를 사용한다면`axios`와`redux-saga-requests-axios`를 설치하십시오:

  ```
  $ npm install axios redux-saga-requests-axios
  ```
  or CDN: `https://unpkg.com/redux-saga-requests-axios`.
- Fetch API를 사용하는 경우, install `isomorphic-fetch` (or a different Fetch polyfill) and `redux-saga-requests-fetch`:

  ```
  $ npm install isomorphic-fetch redux-saga-requests-fetch
  ```
  or CDN: `https://unpkg.com/redux-saga-requests-fetch`.

물론 이것은 Redux-Saga 애드온이기 때문에`redux-saga`도 설치해야합니다.
또한 '재 선택'을 설치해야합니다.

## Usage [:arrow_up:](#table-of-content)

작동 방식에 대한 빠른 소개는 [동기 부여](#motivation-arrow_up) 단락을 참조하십시오.

더 나아 가기 전에,이 라이브러리 뒤에있는 명명 규칙 설명과 아이디어로 시작해 봅시다.

아마도 '동기 부여'섹션에서 알 수 있듯이, 사용 된 내용 일부 중 하나는 '요청'키가있는 동작입니다..
지금부터 요청을 액션이라합니다. 이러한 액션이 전달되면 AJAX 요청이 자동으로 시작됩니다. 그런 다음 결과에 따라 해당 성공, 오류 또는 중단 조치가 발송됩니다.
다음 단락에서는 요청 조치에 대한 자세한 정보가 있습니다., 하지만 지금은, 그 요청 동작은 드라이버라고 불리는 것에 의해 구동됩니다. 
당신은 `handleRequest` 기능에서 드라이버를 설정했습니다. 공식적으로 지원되는 Axios, Fetch API, GraphQL 및 모의 드라이버가 있습니다, 그러나 자신의 드라이버를 작성하는 것은 매우 쉽습니다.
원하는 것을 골라주세요. 이해하는 열쇠는 Fetch API를 사용하는 방법을 알고 있다면, Fetch API 드라이버 사용법을 알고 있습니다. `fetch` 함수에 전달할 설정 객체가 있다면,
이제 요청 액션의 `request` 키에 첨부합니다. 나중에 설명 할 또 다른 정보는`request` 옆에 `meta` 키를 사용할 수 있다는 것입니다, 이것은 몇 가지 추가 옵션을 전달하는 방법입니다.
예제 중 하나는`meta.driver` 옵션 일 수 있습니다, 요청 액션 당 드라이버를 정의 할 수 있습니다., 그게 다야,
하나의 응용 프로그램 내에서 여러 드라이버를 사용할 수 있습니다. 후술한다, 지금은 핵심 개념에 집중하자.

또 다른 중요한 점은 `요청`을 `쿼리`와 `뮤테이션`로 나눌 수 있다는 것입니다.
이것은 이 라이브러리에서 사용되는 명명 규칙이며 graphql에서 빌 렸습니다.
기억해라, `쿼리`는 일부 데이터를 얻기 위해 실행 된 요청이고, 반면에 `mutation`은 데이터 업데이트를 포함하여 부작용을 일으키는 요청입니다. 
아니면 다른 관점에서 생각할 수도 있습니다, REST를 사용하는 경우 일반적으로 쿼리는`GET`,`HEAD`,`OPTIONS` 요청입니다.,
뮤테이션은 `POST`,`PUT`,`PATCH`,`DELETE` 요청이 될 것입니다. 물론 graphql을 사용하면 설명이 필요하지 않습니다.

이제 명명 규칙이 명확 해짐에 따라 지금 액션은 내려놓고 리듀서에 초점을 맞추겠습니다.
As shown in `Motivation` example, `handleRequests`는 모든 원격 상태를 한곳에서 관리하는 `requestsReducer`를 사용할 준비가되었습니다.
그말은, 당신이 당신 자신이 만든 리듀서의 요청에 대해서 반응 할 수 없다는 것을 의미하지는 않습니다. 그리고 대부분 그럴 필요도 없습니다.

`state` 내부의 `request` 키에 첨부된 하나의 커다란 객체인 하나의 리듀서에 모든 원격 상태는 보관됩니다.
그러나 애플리케이션에서이 상태를 직접 읽지 말고이 라이브러리에서 제공하는 셀렉터를 사용해야합니다. 왜? 이미 `reselect`를 사용하여 최적화되어 있기 때문입니다.
더해서, 요청 리듀서의 상태에는 내부 구현 세부 사항으로 취급되어야하는 일부 정보가 포함됩니다.not needed to be understood or used by users of this library. 
Selectors will be explained in a dedicated chapter,
지금은 셀렉터 `getQuery`,`getMutation`뿐만 아니라 셀렉터 생성기`getQuerySelector` 및`getMutationSelector`가 있음을 알고 있습니다.

또한 아마도 당신은 사가를 눈치챘겠지만. 실제로 응용 프로그램에서 sagas를 알거나 사용할 필요가 없습니다! You only need to do
what is shown in `Motivation` part. 그러나 이 라이브러리는 완전히 호환 가능합니다. 실제로 sagas를 사용하여 일부 기능을 강화합니다. 
다음 릴리스 중 하나가 'redux-saga'의존성을 제거하기 위해 다시 작성 될 수 있지만 이 라이브러리 API를 변경해서는 안되며 호기심으로 알고 있어야합니다.

## Actions [:arrow_up:](#table-of-content)

위에서 설명한 것처럼이 라이브러리의 핵심은 액션입니다. 알림은 요청을 쿼리와 뮤테이션으로 나눌 수 있습니다. Axios 드라이버를 사용한다고 가정 할 때 이러한 동작을 작성하는 방법을 예를 들어 보겠습니다.
우리는 하나의 쿼리와 하나의 뮤테이션을 쓸 것입니다:
```js
// query
const fetchBooks = () => ({
  type: 'FETCH_BOOKS',
  request: {
    url: '/books',
  },
});

// mutation
const deleteBook = id => ({
  type: 'DELETE_BOOK',
  request: {
    url: `/books/${id}`,
    method: 'delete'
  },
  meta: {
    mutations: {
      FETCH_BOOKS: data => data.filter(book => book.id !== id),
    },
  },
});
```

이제 위의 작업 중 하나가 디스패치된 후 어떻게됩니까? `dispatch(deleteBook('1'))` 에 의해서 `DELETE_BOOK` 울 디스패치 한다고 상상합시다. 우리는 Axios를 사용하기 때문에, `axios.delete('/books/1')` 촉발 될 것이다.
그런 다음 AJAX 요청이 완료된 후, 성공, 오류 또는 중단 조치가 디스패치됩니다. 
따라서 우리의 경우 아래 작업 중 하나:
```js
// success
{
  type: 'DELETE_BOOK_SUCCESS',
  response: {
    data: {
      id: '1',
      name: 'deleted book',
    },
  },
  meta: {
    mutations: {
      FETCH_BOOKS: data => data.filter(book => book.id !== '1'),
    },
    requestAction: {
      type: 'DELETE_BOOK',
      request: {
        url: '/books/1',
        method: 'delete',
      },
      meta: {
        mutations: {
          FETCH_BOOKS: data => data.filter(book => book.id !== '1'),
        },
      },
    },
  },
}

// error
{
  type: 'DELETE_BOOK_ERROR',
  error: 'a server error',
  meta: {
    mutations: {
      FETCH_BOOKS: data => data.filter(book => book.id !== '1'),
    },
    requestAction: {
      type: 'DELETE_BOOK',
      request: {
        url: '/books/1',
        method: 'delete',
      },
      meta: {
        mutations: {
          FETCH_BOOKS: data => data.filter(book => book.id !== '1'),
        },
      },
    },
  },
}

// abort
{
  type: 'DELETE_BOOK_ABORT',
  meta: {
    mutations: {
      FETCH_BOOKS: data => data.filter(book => book.id !== '1'),
    },
    requestAction: {
      type: 'DELETE_BOOK',
      request: {
        url: '/books/1',
        method: 'delete',
      },
      meta: {
        mutations: {
          FETCH_BOOKS: data => data.filter(book => book.id !== '1'),
        },
      },
    },
  },
}
```

따라서 요청 조치를 디스패치 한 후 요청이 완료된 후 응답 조치가 디스패치됩니다. 결과에 따라, `_SUCCESS`, `_ERROR` 혹은 `_ABORT`가 요청 액션 타입에 접미사로 추가됩니다.

Now, you probably noticed `meta` key next to `request` in `DELETE_BOOK` action. What's that?
`request`가 정보를 당신이 사용하는 드라이버(우리의 경우 Axios)에게 전달하는데 사용되는 반면에, `메타`는 몇 가지 추가 옵션을 전달할 수있는 곳입니다,
all of which will be explained later. 우리의 경우에는 `meta.mutations`가 있습니다.
아마 당신이 짐작했듯이, 이것은 뮤테이션 성공에 대한 쿼리 데이터를 업데이트 할 수있는 곳입니다- in our case
사용 가능한 책을 필터링하여 ID가 `'1'`인 책을 제거합니다. 또 다른`meta` 특성은 모든 응답 조치에 관련 요청 조치에 대한 참조를 제공하는 추가`meta.requestAction` 키가 있다는 것입니다.
또한 요청 액션의 모든 메타 키가 응답 액션에 추가됩니다., 그렇기 때문에 응답 액션에서도 `meta.mutations`를 사용할 수 있습니다..

### Meta options

이제`meta`에서 사용 가능한 모든 옵션에 대해 이야기 해 봅시다:
- `getData: data => transformedData`: 요청 성공시 호출되는 함수, 서버에서받은 데이터를 변환 할 수 있습니다
- `getError: error => transformedError`: 요청 오류시 호출되는 함수, 서버에서받은 오류를 변환 할 수 있습니다
- `asPromise: boolean`: `true` or `false`, 요청 액션를 프로미스화 하는 데 사용할 수있는, 그래서 `dispatch(fetchBooks()).then().catch()` 해준다. 자세한 내용은 '미들웨어'장에서 찾을 수 있습니다
- `asMutation: boolean`: `true` 일 때 요청 조치를 돌연변이로 강제 처리하는 데 사용할 수 있습니다. 또는 'false'일 때 쿼리
- `driver: string`: 여러 드라이버를 사용하는 경우에만 '여러 드라이버'장에서 자세한 내용을 확인하십시오
- `takeLatest: boolean`:'true'일 때, 주어진 유형의 요청이 보류 중이고 다른 유형의 요청이 발생하면 첫 번째 요청이 자동으로 중단되어 경쟁 조건 버그를 방지하고 성능을 향상시킬 수 있습니다. 기본적으로 쿼리의 경우 `true`, 돌연변이는 `false`.
- `abortOn: string | string[] | action => boolean`: 예를 들면 `'LOGOUT'`, `['LOGOUT']` 혹은 `action => action.type === 'LOGOUT'`,  
당신은 자동으로 요청을 중단하는 데 사용할 수 있습니다
- `requestKey: string` - 기본적으로 주어진 요청 유형에 대해 정보를 한 번만 저장하면된다고 가정합니다.
데이터, 오류 또는로드 상태와 같은, `fetchBook ( '2')`는 이전 책의 데이터를 무시합니다, like with `id` `'1'`,이 속성으로 동작을 변경할 수 있습니다, like `requestKey: id`
- `requestsCapacity: number` - `requestKey`와 함께 사용, 메모리 누수 방지, 1000 개 이상의 다른`requestKey`로 요청을 발송한다고 상상해보십시오., `requestsCapacity : 2`를 전달하면 세 번째가 해결 된 후 첫 번째 요청의 상태가 제거됩니다., FIFO 규칙이 여기에 적용됩니다
- `normalize: boolean` - 응답 성공시 '데이터'자동 정규화, '정규화'장의 자세한 정보
- `cache: boolean | number` - 쿼리를 캐시하는 데 사용할 수 있습니다, 'true'일 때 영원히 또는 몇 초 동안 , 미들웨어 장에서 더 많은 정보
- `dependentRequestsNumber: number` - 이 요청 이후에 실행될 요청 수, SSR 목적으로 만, 미들웨어 장에서 더 많은 정보
- `isDependentRequest: boolean`: `dependentRequestsNumber`와 함께 사용, SSR에도 유사하게 사용됩니다.
- `optimisticData`: an object which will be normalized on request as an optimistic update
- `revertedData`: an object which will be normalized on response error so if optimistic update failed
- `localData`: 그것은 어떤 행동에도 붙일 수 있습니다, 액션을 요청하지 않더라도, 요청 없이 데이터를 정규화 한다
- `mutations`: 쿼리 데이터를 업데이트하는 개체는 아래에서 설명합니다.

### Mutations and data updates

아시다시피, 돌연변이는 데이터를 얻는 데 사용되는 쿼리 계약에서 사이드 이펙트를 일으키는 요청입니다. 
그러나 쿼리에서 수신 한 데이터를 업데이트하는 방법? 우리는 보통 리듀서에서 그런 일을하지만이 라이브러리를 사용하면 다른 방법 인 'meta.mutations'가 있습니다.

`DELETE_BOOK` 돌연변이에서 보았 듯이`meta.mutations` 객체는 다음과 같이 선언되었습니다:
```js
mutations: {
  FETCH_BOOKS: data => data.filter(book => book.id !== '1'),
}
```

위 함수는 'FETCH_BOOKS'쿼리의 현재 상태 인 'data'로 'DELETE_BOOK'성공시 호출됩니다.

반환되는 것은 무엇이든 데이터를 업데이트하기 위해 'FETCH_BOOKS'를 담당하는 리듀서에 사용합니다.

물론 하나의 돌연변이 동작으로 여러 쿼리를 업데이트 할 수 있습니다. 다른 키만 추가하면됩니다..

삭제 된 책 등에 대한 정보와 같은 돌연변이 액션 자체에서 데이터 응답에 액세스해야하는 경우 전달 된 두 번째 인수에서 액세스 할 수 있습니다. 
- 즉,이 함수에는 실제로 서명 `(data, mutationData) => updatedData`가 있습니다.

### Updating data of queries with requestType

이제 `requestKey`로 쿼리 데이터를 업데이트하려면 어떻게해야합니까? `queryType`에`requestKey`를 추가하십시오.:
```js
mutations: {
  [FETCH_BOOK_DETAIL + id]: data => data.id === id ? null : data,
}
```

`type : FETCH_BOOK_DETAIL` 및`meta.requestKey : id`의 쿼리 작업이 있다고 가정합니다.,
보시다시피`id`를`FETCH_BOOK_DETAIL`에 추가하면됩니다. 업데이트 함수는 id가 일치하거나 다른 경우에 데이터가 있으면 `null`을 반환합니다., typically what you would do in a reducer.

### Local updates

뮤테이션 요청을하지 않고도 쿼리 데이터를 로컬로 업데이트 할 수도 있습니다.
어떻게? 'meta.mutations'를 다음과 같은 액션에 첨부하십시오.:
```js
const deleteBookLocally = id => ({
  type: 'DELETE_BOOK_LOCALLY',
  mutations: {
    [FETCH_BOOK_DETAIL + id]: {
      updateData: data => data.id === id ? null : data,
      local: true,
    },
  },
});
```
보시다시피, 데이터 업데이트는 update 함수를 직접 사용하는 것과 동일한`updateData` 키를 사용하여 객체로 정의 할 수도 있습니다.
그러나 객체를 사용하면 몇 가지 추가 옵션을 전달할 수 있습니다. 여기서 로컬 변수를 사용하려는 라이브러리에 알리기 위해 'local : true'를 사용합니다.

### Optimistic updates

때로는 데이터를 업데이트하기 위해 돌연변이 응답을 기다리지 않고 때로는 한번에 업데이트하기를 원할 수도 있습니다.
그 때 실제로 돌연변이 요청이 실패한 경우 데이터 업데이트를 되돌릴 수있는 방법을 원합니다.
이것을 소위 낙관적 업데이트라고하며 객체 표기법은 지역 뮤테이션이과 같이 유용합니다..
여기에 예가 있습니다:
```js
const deleteBookOptimistic = book => ({
  type: 'DELETE_BOOK_OPTIMISTIC',
  request: {
    url: `/book/${book.id}`,
    method: 'delete',
  },
  meta: {
    mutations: {
      [FETCH_BOOKS]: {
        updateDataOptimistic: data => data.filter(v => v.id !== book.id),
        revertData: data => [book, ...data],
      },
    },
  },
});
```

따라서 위의 FETCH_BOOKS 쿼리에 대한 낙관적 업데이트로 돌연변이 동작이 있습니다..
`DELETE_BOOK_OPTIMISTIC` 액션이 디스패치 된 직후 'updateDataOptimistic'이 호출됩니다.,
`updateData`의 경우와 같이 성공하지 못하면 `revertData`는`DELETE_BOOK_OPTIMISTIC_ERROR`에서 호출됩니다.
예기치 않은 오류 발생시 데이터를 수정하고 삭제를 되돌릴 수 있습니다..
동시에`updateData`를 사용하여`DELETE_BOOK_OPTIMISTIC_SUCCESS`의 데이터를 추가로 업데이트 할 수 있습니다.

### resetRequests

때로는 쿼리와 뮤테이션을 포함하여 요청의 데이터와 오류를 지워야 할 수도 있습니다.
`resetRequests` 액션을 사용하면됩니다. 예를 들면 다음과 같습니다:
```js
import { resetRequests } from 'redux-saga-requests';

dispatch(resetRequests()); // clear everything
dispatch(resetRequests([FETCH_BOOKS])); // clear errors and data for FETCH_BOOKS query
dispatch(resetRequests([DELETE_BOOKS])); // clear errors if any for for DELETE_BOOKS mutation
dispatch(resetRequests([FETCH_BOOKS, { requestType: FETCH_BOOK, requestKey: '1' }]));
// clear errors and data for FETCH_BOOKS and FETCH_BOOK with 1 request key
```

## Selectors [:arrow_up:](#table-of-content)

스스로 원격 상태를 얻을 수는 있지만 아래 선택기를 사용하는 것이 좋습니다.
우선 캐시를 재사용하고 필요한 경우 캐시를 정리하여 이미 최적화되어 있습니다. 
또 다른 이유는 애플리케이션에 필요한 정보 만 반환하고 'requestsReducer'에 보관 된 상태에는 라이브러리 자체에 필요한 더 많은 데이터가 포함되어 있기 때문입니다. 
자동 정규화를 사용할 때의 상황은 말할 것도 없습니다.
리듀서의 데이터는 정규화 된 상태로 유지되지만 앱에서는 비정규 화 된 상태가 필요합니다. 
선택기는 이미 자동으로 빠르게 비정규 화하는 방법을 알고 있으므로 걱정할 필요가 없습니다.

### getQuery

`getQuery`는 주어진 쿼리에 대한 상태를 반환하는 선택기입니다. props 가 필요한 셀렉터입니다.
이전에 사용한 FETCH_BOOKS 쿼리의 상태를 얻고 싶다고 상상해보십시오. 이렇게 사용할 수 있습니다:
```js
import { getQuery } from 'redux-saga-requests';

const booksQuery = getQuery(state, { type: 'FETCH_BOOKS' });
/* for example {
  data: [{ id: '1', name: 'Some book title' }],
  loading: false,
  error: null,
} */
```

숙련 된 Redux 개발자라면`getQuery`의 메모이제이션이 걱정 될 수 있습니다.
두려워 말라! 다른 props로 호출 할 수 있으며 메모이제이션이 손실되지 않습니다.:
```js
const booksQuery = getQuery(state, { type: 'FETCH_BOOKS' });
getQuery(state, { type: 'FETCH_STH_ELSE' });
booksQuery === getQuery(state, { type: 'FETCH_BOOKS' })
// returns true (unless state for FETCH_BOOKS query really changed in the meantime)
```

우리는`type` prop에 대한 예제만을 제공했습니다, 그러나 여기에 모든 가능성의 목록이 있습니다:
- `type: string`: action creator 라이브러리를 사용할 때 쿼리 액션 유형 또는 액션 자체를 전달하십시오.
- `requestKey: string`: 쿼리 동작에서`meta.requestKey`를 사용한 경우 사용하십시오
- `multiple`: 데이터가 비어있는 경우`data`를 `null` 대신`[]`로 설정하려면`true`로 설정하십시오., 기본적으로`false`
- `defaultData`: `null` 대신 'data'를 임의의 객체로 나타내는 데 사용하십시오., 그래도 최상위 객체를 사용하십시오.,
 셀렉터 메모이제이션을 중단하지 않도록 여러 번 다시 만들지 마십시오.

### getQuerySelector

`getQuery`와 거의 동일하다, 차이점은 `getQuery`가 셀렉터라는 것입니다.,
`getQuerySelector`는 셀렉터 생성자입니다. - `getQuery` 만 반환합니다.

어딘가에 props없이 셀렉터를 제공해야 할 때 유용합니다 (`useSelector` React hook에서와 같이).
따라서`useSelector (state => getQuery (state, {type : 'FETCH_BOOKS'}))`대신`useSelector (getQuerySelector ({type : 'FETCH_BOOKS'}))`만하면됩니다.

### getMutation

`getQuery`와 거의 동일하며 뮤테이션에만 사용됩니다.
```js
import { getMutation } from 'redux-saga-requests';

const deleteBookMutation = getMutation(state, { type: 'DELETE_BOOK' });
/* for example {
  loading: false,
  error: null,
} */
```

쿼리처럼 `type`과 선택적으로`requestKey` props을 받아들입니다.

### getMutationSelector

`getQuerySelector`와 마찬가지로 `getMutation` 셀렉터를 반환합니다.

## Reducers [:arrow_up:](#table-of-content)

`handleRequests`에 의해 리턴 된`requestsReducer`에 의해 이미 수행되기 때문에 원격 상태를 관리하기 위해 리듀서를 작성할 필요가 없습니다.
쿼리 및 돌연변이에 추가 상태가 필요한 경우 셀렉터 수준에서 수행하는 것이 훨씬 좋습니다., 예를 들어 책에 속성을 추가한다고 가정합니다:
```js
import { createSelector } from 'reselect';
import { getQuerySelector } from 'redux-saga-requests;

const bookQuerySelector = createSelector(
  getQuerySelector({ type: 'FETCH_BOOKS', multiple: true }),
  booksQuery => ({
    ...booksQuery,
    data: booksQuery.data.map(book => ({ ...book, extraProp: 'extraValue' })),
  }),
);
```
위처럼 하지 않으면 책 상태를 복제해야 되며, 이는 매우 나쁜 습관입니다.

로컬 상태를 관리하는 리듀서의 요청 및 응답 액션에 반응하는 것이 좋습니다., 예를 들면:
```js
import { success, error, abort } from 'redux-saga-requests';

const FETCH_BOOKS = 'FETCH_BOOKS';

const localReducer = (state, action) => {
  switch (action.type) {
    case FETCH_BOOKS:
      return updateStateForRequestAction(state);
    case success(FETCH_BOOKS):
      return updateStateForSuccessResponseAction(state);
    case error(FETCH_BOOKS):
      return updateStateForErrorResponseAction(state);
    case abort(FETCH_BOOKS):
      return updateStateForAbortResponseAction(state);
    default:
      return state;
  }
};
```

'성공', '오류'및 '취소'도우미에 주목하십시오. 편의를 위해 조치를 요청하기 위해 적절한 접미사 만 추가하면됩니다., 
따라서이 경우에는 각각 'FETCH_BOOKS_SUCCESS', 'FETCH_BOOKS_ERROR'및`FETCH_BOOKS_ABORT '를 반환합니다.

## Middleware [:arrow_up:](#table-of-content)

`handleRequests`에 전달 된 일부 옵션은 추가 키인`requestsMiddleware`를 반환합니다.
이러한 옵션은`promisify`,`cache` 및`ssr`입니다., 이들 모두는 임의의 조합으로 독립적으로 사용될 수있다.
단지, 미들웨어 목록 중, 사가 미들웨어 이전에 `requestsMiddleware` 를 추가하면 됩니다. 
따라서 모든 요청 미들웨어 (아래 설명)를 사용한다고 가정하면 다음과 같이 `Modivation`에서 코드를 조정합니다.
```js
const configureStore = () => {
  const { requestsReducer, requestsSagas, requestsMiddleware } = handleRequests({
    driver: createDriver(axios),
    promisify: true,
    cache: true,
    ssr: 'client',
  });

  const reducers = combineReducers({
    requests: requestsReducer,
  });

  const sagaMiddleware = createSagaMiddleware();
  const middleware = [...requestsMiddleware, sagaMiddleware];
  const store = createStore(
    reducers,
    applyMiddleware(middleware),
  );

  function* rootSaga() {
    yield takeEvery(FETCH_BOOKS, fetchBooksSaga);
    yield all(requestsSagas);
  }

  sagaMiddleware.run(rootSaga);
  return store;
};
```

### Promise middleware

어딘가에 요청 액션을 전달하고 동일한 위치에서 응답을 받으려면 어떻게해야합니까?
기본적으로 액션 디스패치 행위는 작업 자체를 반환합니다., promise 미들웨어를 사용하여이 동작을 변경할 수 있습니다. 
`promisify : true`를`handleRequests '로 전달하면`requestsMiddleware`에 promise 미들웨어가 포함됩니다.

이제 액션 메타를 요청하려면`asPromise : true`를 추가하면됩니다.:
```js
const fetchBooks = () => ({
  type: FETCH_BOOKS,
  request: { url: '/books'},
  meta: {
    asPromise: true,
  },
});
```

그런 다음 예를 들어 컴포넌트에서 액션을 디스패치하고 응답을 기다릴 수 있습니다.:
```js
class Books extends Component {
  fetch = () => {
    this.props.fetchBooks().then(successAction => {
      // handle successful response
    }).catch(errorOrAbortAction => {
      // handle error or aborted request
    })
  }

  render() {
    // ...
  }
}
```

또한 선택적인`autoPromisify : true` 플래그를`handleRequests`에 전달하면 모든 요청을 promise화 할 수 있습니다. - 더 이상`meta.asPromise : true`를 사용할 필요가 없습니다..

### Cache middleware

때로는 응답을 일정 시간 동안 또는 영원히 캐시하기를 원할 수도 있습니다 (최소한 페이지가 다시로드 될 때까지).
또는 다른 방법으로, 주어진 요청을 일정 시간 동안 한 번만 보내려고합니다. 선택적인 캐시 미들웨어로 쉽게 달성 할 수 있습니다.`cache : true`를`handleRequests`에 전달하면됩니다.

그런 다음`meta.cache`를 사용할 수 있습니다:
```js
const fetchBooks = () => ({
  type: FETCH_BOOKS,
  request: { url: '/books'},
  meta: {
    cache: 10, // in seconds, or true to cache forever
  },
});
```

이제 일어날 성공적인 책 가져 오기 ( `FETCH_BOOKS_SUCCESS`가 전달 된 후 특정) 후 '10'초 동안 `FETCH_BOOKS` 동작이 AJAX 호출을 트리거하지 않으며 다음 `FETCH_BOOKS_SUCCESS` 에 캐시 된 내용이 포함됩니다. 이전 서버 응답. 
`cache : true`를 사용하여 영원히 캐싱 할 수도 있습니다.


다른 유스 케이스는 캐시 키를 기반으로 동일한 요청 조치에 대해 별도의 캐시를 유지하려는 경우입니다. 예를 들어:
```js
const fetchBook = id => ({
  type: FETCH_BOOK,
  request: { url: `/books/${id}`},
  meta: {
    cache: true,
    requestKey: id,
    requestsCapacity: 2 // 선택 사항,이 수를 초과하는 오래된 캐시를 지우려면
  },
});

/* 다음과 같은 동작을 달성합니다:
- GET /books/1 - make request, cache /books/1
- GET /books/1 - cache hit
- GET /books/2 - make request, /books/2
- GET /books/2 - cache hit
- GET /books/1 - cache hit
- GET /books/3 - make request, cache /books/3, invalidate /books/1 cache
- GET /books/1 - make request, cache /books/1, invalidate /books/2 cache
*/
```

어떤 이유로 캐시를 수동으로 지워야한다면`clearRequestsCache` 액션을 사용할 수 있습니다:
```js
import { clearRequestsCache } from 'redux-saga-requests';

dispatch(clearRequestsCache()) // clear the whole cache
dispatch(clearRequestsCache(FETCH_BOOKS)) // clear only FETCH_BOOKS cache
dispatch(clearRequestsCache(FETCH_BOOKS, FETCH_AUTHORS)) // clear only FETCH_BOOKS and FETCH_AUTHORS cache
```

그러나 'clearRequestsCache'는 쿼리 상태를 제거하지 않습니다., 단지 캐시 만료만 지웁니다 그래서 다음번에 주어진 요청 타입이 디스패치될 때, AJAX 요청이 서버에 닿습니다.
따라서 캐시 무효화 작업과 같습니다.

또한 캐시는 기본적으로 SSR과 호환됩니다., 서버에서 메타 캐시와 함께 요청 액션을 전달하면, 이 정보는 클라이언트 내부 상태로 전달됩니다.

### Server side rendering middleware

서버 측 렌더링은 매우 복잡한 주제이며 이를 해결하는 방법에는 여러 가지가 있습니다..
많은 사람들이 React 컴포넌트를 중심으로 전략을 사용합니다, 예를 들어 응답을 통해 요청을하고 프로미스를 반환하는 컴포넌트 요소에 정적 메서드를 연결합니다.
, 그런 다음`Promise.all`로 감쌉니다. Redux를 사용할 때이 전략을 권장하지 않습니다, 서버에 추가 코드와 이중 렌더링이 필요할 수 도 있기 때문입니다, 하지만 정말로하고 싶다면 프로미스 미들웨어로 가능합니다.

그러나 다른 접근법을 사용하는 것이 좋습니다.. See [server-side-rendering-example](https://github.com/klis87/redux-saga-requests/tree/master/examples/server-side-rendering) with the complete setup, but in a nutshell you can write universal code like you would
normally write it without SSR, with just only minor additions. Here is how:

1. 시작하기 전에, 이 전략은 Redux 수준에서 요청을 발송해야 함을 확실히 합니다, 최소한 응용 프로그램로드에서 실행되어야하는 것들은 말이죠. 
예를 들어`componentDidMount` 안에서 디스패치 할 수 없습니다. 그것들을 디스패치 할 명백한 장소는`yield put (fetchBooks ())`와 같은 당신의 sagas에 있습니다. 
그러나 앱에 여러 경로가 있고 각 경로가 다른 요청을 보내야하는 경우에는 어떻게됩니까? 글쎄, 당신은 Redux가 현재 경로를 인식하도록해야합니다. 
Redux에 대한 일류 지원 라우터를 사용하는 것이 좋습니다, 즉 [redux-first-router](https://github.com/faceyspacey/redux-first-router). 
그래도`react-router`를 사용하면 괜찮습니다., [connected-react-router](https://github.com/supasate/connected-react-router)를 사용하여 Redux와 통합하면됩니다.
그런 다음`take` 이펙트를 사용하여 경로 변경을 듣고`select` 이펙트로 현재 위치를 얻을 수 있습니다. 
이것은 어떤 요청이 디스패치되는 지 알기 위해 어떤 경로가 활성화되어 있는지에 대한 정보를 제공합니다.

2. 서버에서, `handleRequests`에 `ssr: 'server'` (`ssr: 'client'` 는 클라이언트에서, 다음 스텝에 설명) 옵션을 넣습니다 , 
여기에는`requestsMiddleware`에 SSR 미들웨어가 포함되며, 추가로 모든 요청이 완료되면 해결되는 `requestsPromise`가 반환됩니다,  

여기서 가능한 구현을 볼 수 있습니다:
    ```js
    import { createStore, applyMiddleware, combineReducers } from 'redux';
    import createSagaMiddleware from 'redux-saga';
    import { all, put, call } from 'redux-saga/effects';
    import axios from 'axios';
    import { handleRequests } from 'redux-saga-requests';
    import { createDriver } from 'redux-saga-requests-axios';

    import { fetchBooks } from './actions';

    function* bookSaga() {
      yield put(fetchBooks());
    }

    export const configureStore = (initialState = undefined) => {
      const ssr = !initialState; // if initialState is not passed, it means we run it on server

      const {
        requestsReducer,
        requestsMiddleware,
        requestsSagas,
        requestsPromise,
      } = handleRequests({
        driver: createDriver(
          axios.create({
            baseURL: 'http://localhost:3000',
          }),
        ),
        ssr: ssr ? 'server' : 'client',
      });

      const reducers = combineReducers({
        requests: requestsReducer,
      });

      const sagaMiddleware = createSagaMiddleware();
      const middleware = [...requestsMiddleware, sagaMiddleware];

      const store = createStore(
        reducers,
        initialState,
        applyMiddleware(...middleware),
      );

      function* rootSaga() {
        yield all([...requestsSagas, call(bookSaga)]);
      }

      sagaMiddleware.run(rootSaga, requestsSagas);

      return { store, requestsPromise };
    };

    // on the server
    import React from 'react';
    import { renderToString } from 'react-dom/server';
    import { Provider } from 'react-redux';

    // in an express/another server handler
    const { store, requestsPromise } = configureStore();

    requestsPromise
      .then(() => {
        const html = renderToString(
          <Provider store={store}>
            <App />
          </Provider>,
        );

        res.render('index', {
          html,
          initialState: JSON.stringify(store.getState()),
        });
      })
      .catch(e => {
        console.log('error', e);
        res.status(400).send('something went wrong');
      });
    ```
보시다시피, redux 앱을 위해 SSR에서 일반적으로 수행하는 것과 비교하여 추가`ssr` 옵션을`handleRequests '에 전달하고`requestsPromise`가 해결 될 때까지 기다려야합니다..

그러나 어떻게 작동합니까? 논리는 내부 카운터를 기반으로합니다.. 초기에는 `0`으로 설정되며 각 요청이 초기화 된 후 `1`씩 증가합니다. 
그런 다음 각 응답 후`1` 씩 감소합니다. 
따라서 첫 번째 요청 후 처음에는 양수가 되고 모든 요청이 완료된 후에는 값이 다시 '0'으로 다시 설정됩니다.
그리고 이때가 모든 요청이 완료되고 'requestsPromise'가 리졸브됩니다 (모든 성공 액션 포함).
또한 특별한 `redux-saga` `END` 액션이 모든 사가를 막기 위해 전달됩니다.

요청 오류가 발생하면 응답 오류 조치와 함께 'requestsPromise'가 거부됩니다..

더 복잡한 경우도 있습니다. `x`라는 요청이 있고 다른`y`를 디스패치하고 싶다고 상상해보십시오. 
`y`에는`x` 응답의 정보가 필요하기 때문에 즉시 할 수 없습니다.
위의 알고리즘은 `x`응답 카운터가 이미 `0`으로 재설정되기 때문에 `y`가 완료 될 때까지 기다리지 않습니다. 
여기에 도움이되는 두 가지`action.meta` 속성이 있습니다:
- `dependentRequestsNumber` - 양의 정수,이 요청 이후에 발생하는 많은 요청,
위의 예에서`y`만이`x`에 의존하기 때문에`dependentRequestsNumber : 1`을`x` 액션에 넣습니다.
- `isDependentRequest` - 다른 요청에 따라 요청을 `isDependentRequest : true`로 표시
이 예에서는`x`에 의존하기 때문에 `isDependentRequest : true`를`y`에 넣습니다

You could even have a more complicated situation, in which you would need to dispatch `z` after `y`. 
Then you would also add `dependentRequestsNumber: 1` to `y` and `isDependentRequest: true` to `z`. 
Yes, a request can have both of those attibutes at the same time! 
Anyway, how does it work? Easy, just a request with `dependentRequestsNumber: 2` would increase counter by `3` on request and decrease by `1` on response,
while an action with `isDependentRequest: true` would increase counter on request by `1` as usual but decrease it on response by `2`. 
So, the counter will be reset to `0` after all requests are finished, also dependent ones.

3. The last thing you need to do is to pass `ssr: 'client` to `handleRequests`,
like you noticed in previous step on the client side, What does it do? 
Well, it will ignore request actions which match those already dispatched during SSR. 
Why? Because otherwise with universal code the same job done on the server would be repeated on the client.

## handleRequests [:arrow_up:](#table-of-content)

As you probably noticed in other chapters, `handleRequests` is a function which gets some options
and returns object with the following keys:
- `requestsReducer`: ready to use reducer managing the whole remote state, you need to attach it
to `requests` key in `combineReducers`
- `requestsSagas`: list of sagas you have to use like in examples
- `requestsMiddleware`: list of optional middleware you should put before sagaMiddleware, only applicable
when option `cache`, `promisify` or `ssr` is used
- `requestsPromise`: promise which is resolved after all requests are finished, only with `ssr: 'server'` option

Below you can see all available options for `handleRequests`:
- `driver`: the only option which is required, a driver or object of drivers if you use multiple drivers
- `onRequest`: see interceptors
- `onSuccess`: see interceptors
- `onError`: see interceptors
- `onAbort`: see interceptors
- `cache`: see Cache middleware
- `promisify`: see Promise middleware
- `autoPromisify`: see Promise middleware
- `ssr`: see Server side rendering middleware
- `isRequestAction: (action: AnyAction) => boolean`: here you can adjust which actions are treated
as request actions, usually you don't need to worry about it, might be useful for custom drivers
- `isRequestActionQuery: (requestAction: RequestAction) => boolean`: if this function returns true,
request action is treated as query, if false, as mutation, probably only useful for custom drivers
- `takeLatest: boolean || (action: requestAction) => boolean`: if true, pending requests of a given type
are automatically cancelled if a new request of a given type is fired, by default queries are run as `takeLatest: true`
and mutations as `takeLatest: false`
- `abortOn: string | string[] | action => boolean`: for example `[LOGOUT]`, or `action => action.type === LOGOUT`,
you can use it to automatically abort all requests for a given action
- `normalize`: by default `false`, see normalisation
- `getNormalisationObjectKey`: see normalisation
- `shouldObjectBeNormalized`: see normalisation

## sendRequest [:arrow_up:](#table-of-content)

When you dispatch a request action, under the hood `sendRequest` saga is called.
Typically you don't need to use, as dispatching Redux action as usual is enough.
However, `sendRequest` is useful in [Interceptors](#interceptors-arrow_up).
This is how you can use it:
```js
import { takeLatest } from 'redux-saga/effects';
import { sendRequest } from 'redux-saga-requests';

const FETCH_BOOKS = 'FETCH_BOOKS';

const fetchBooks = () => ({
  type: FETCH_BOOKS,
  request: { url: '/books' },
});

function* booksSaga() {
  const { response, error } = yield call(sendRequest, fetchBooks());
}
```
Above is actually the same as `yield put(fetchBooks)`, or `yield putResolve(fetchBooks)`
together with promise middleware, if you want to get response in this place.
In Redux thunk or React component you would do `dispatch(fetchBooks())`.

Optionally you can pass config to `sendRequest`, like:
```js
function* booksSaga() {
  yield call(sendRequest, fetchBooks(), { dispatchRequestAction: false });
}
```

The following options are possible:
- `dispatchRequestAction`: useful if you use `sendRequest` to react on already dispatched request action not to duplicate it, default as `true`
- `silent: boolean;`: passing `false` can disable dispatching all Redux actions for this request, default as `false`
- `runOnRequest: boolean;`: passing `false` can block `onRequest` interceptor, more in the next chapter, default as `true`
- `runOnSuccess: boolean;`: passing `false` can block `onResponse` interceptor, default as `true`
- `runOnError: boolean;`: passing `false` can block `onError` interceptor, default as `true`
- `runOnAbort: boolean;`: passing `false` can block `onAbort` interceptor, default as `true`

## Interceptors [:arrow_up:](#table-of-content)

You can add global handlers to `onRequest`, `onSuccess`, `onError` add `onAbort`,
just pass them to `handleRequests`, like so:
```js
import axios from 'axios';
import { sendRequest, handleRequests } from 'redux-saga-requests';

function* onRequestSaga(request, action) {
  // do sth with you request, like add token to header, or dispatch some action etc.
  return request;
}

function* onSuccessSaga(response, action) {
  // do sth with the response, dispatch some action etc
  return response;
}

function* onErrorSaga(error, action) {
  // do sth here, like dispatch some action

  // you must return { error } in case you dont want to catch error
  // or { error: anotherError }
  // or { response: someRequestResponse } if you want to recover from error

  if (tokenExpired(error)) {
    // get driver instance, in our case Axios to make a request without Redux
    const requestInstance = yield getRequestInstance();

    try {
      // trying to get a new token, we use axios directly not to touch redux
      const { data } = yield call(
        axios.post,
        '/refreshToken',
      );

      saveNewToken(data.token); // for example to localStorage

      // we fire the same request again:
      // - with silent: true not to dispatch duplicated actions
      return yield call(sendRequest, action, { silent: true });

      /* above is a handy shortcut of doing
      const { response, error } = yield call(
        sendRequest,
        action,
        { silent: true },
      );

      if (response) {
        return { response };
      } else {
        return { error };
      } */
    } catch(e) {
      // we didnt manage to get a new token
      return { error: e }
    }
  }

  // not related token error, we pass it like nothing happened
  return { error };
}

function* onAbortSaga(action) {
  // do sth, for example an action dispatch
}

handleRequests({
  driver: createDriver(axios),
  onRequest: onRequestSaga,
  onSuccess: onSuccessSaga,
  onError: onErrorSaga,
  onAbort: onAbortSaga,
);
```

If you need to use `sendRequest` in an interceptor, be aware of an additional options you
can pass to it:
```js
yield call(sendRequest, action, {
  silent: true,
  runOnRequest: false,
  runOnSuccess: false,
  runOnError: false,
  runOnAbort: false,
});
```
Generally, use `silent` if you don't want to dispatch actions for a given request.
The rest options is to disable given interceptors for a given request. By default `silent` is `false`,
which simply means that `sendRequests` will dispatch Redux actions. The rest is slightly more dynamic:
- if a request is sent not from an interceptor, all interceptors will be run
- if you use `sendRequest` in `onRequest` interceptor, `runOnRequest` is set to `false`
- if you use `sendRequest` in `onSuccess` interceptor, `runOnSuccess` and `runOnError` are set to `false`
- if you use `sendRequest` in `onError` interceptor, `runOnError` is set to `false`
- if you use `sendRequest` in `onAbort` interceptor, `runOnAbort` is set to `false`

Those defaults are set to meet most use cases without the need to worry about disabling proper interceptors manually.
For example, if you use `sendRequest` in `onRequest` interceptor, you might end up with inifinite loop when `runOnRequest` was true.
If your use case vary though, you can always overwrite this behaviour by `runOn...` options.

## FSA [:arrow_up:](#table-of-content)

If you like your actions to be compatible with
[Flux Standard Action](https://github.com/acdlite/flux-standard-action#flux-standard-action),
that's totally fine, you can define your request actions like:
```js
const fetchBooks = () => ({
  type: 'FETCH_BOOKS',
  payload: {
    request: {
      url: '/books',
    },
  },
  meta: { // optional
    someKey: 'someValue',
  },
});
```
Then, success, error and abort actions will also be FSA compliant.
For details, see [redux-act example](https://github.com/klis87/redux-saga-requests/tree/master/examples/redux-act-integration).



## Usage with Fetch API [:arrow_up:](#table-of-content)

All of the above examples show Axios usage, in order to use Fetch API, just pass Fetch driver to `handleRequests`:
```js
import 'isomorphic-fetch'; // or a different fetch polyfill
import { handleRequests } from 'redux-saga-requests';
import { createDriver } from 'redux-saga-requests-fetch';

handleRequests({
  driver: createDriver(
    window.fetch,
    {
      baseURL: 'https://my-domain.com' // optional - it works like axios baseURL, prepending all relative urls
      AbortController: window.AbortController, // optional, if your browser supports AbortController or you use a polyfill like https://github.com/mo/abortcontroller-polyfill
    }
  ),
});
```

And in order to create Fetch API requests, below:
```js
fetch('/users', {
  method: 'POST',
  body: JSON.stringify(data),
  headers: {
    'Content-Type': 'application/json',
  },
});
```
should be translated to this:
```js
const fetchUsers = () => ({
  type: 'FETCH_USERS',
  request: {
    url: '/users/',
    method: 'POST',
    body: JSON.stringify(data),
    headers: {
      'Content-Type': 'application/json',
    },
  }
});
```
The point is, you can use the same request config like you do with pure Fetch API, but you need to pass `url` in the
config itself. Also, one additional parameter you could provide in the config is `responseType`, which is set as `json`
as the default. Available response types are: `'arraybuffer'`, `'blob'`, `'formData'`, `'json'`, `'text'`, or `null`
(if you don't want a response stream to be read for the given response).

Also, this driver reads response streams automatically for you (depending on `responseType` you choose)
and sets it as `response.data`, so instead of doing `response.json()`, just read `response.data`.

## Usage with GraphQL [:arrow_up:](#table-of-content)

Just install `redux-saga-requests-graphql` driver. See
[docs](https://github.com/klis87/redux-saga-requests/tree/master/packages/redux-saga-requests-graphql)
for more info.

## Mocking [:arrow_up:](#table-of-content)

Probably you are sometimes in a situation when you would like to start working on a feature which needs some integration with
an API. What you can do then? Probably you just wait or start writing some prototype which then you will polish once API is finished. You can do better with `redux-saga-requests-mock`, especially with multi driver support, which you can read about in the
next paragraph. With this driver, you can define expected responses and errors which you would get from server and write your app
normally. Then, after API is finished, you will just need to replace the driver with a real one, like Axios or Fetch API, without
any additional refactoring necessary, which could save you a lot of time!

You can use it like this:
```js
import { handleRequests } from 'redux-saga-requests';
import { createDriver } from 'redux-saga-requests-mock';

const FETCH_PHOTO = 'FETCH_PHOTO';

const fetchPhoto = id => ({
  type: FETCH_PHOTO,
  request: { url: `/photos/${id}` },
});

handleRequests({
  driver: createDriver(
    {
      [FETCH_PHOTO]: (requestConfig, requestAction) => {
        // mock normal response for id 1 and 404 error fot the rest
        const id = requestConfig.url.split('/')[2];

        if (id === '1') {
          return {
            data: {
              albumId: 1,
              id: 1,
              title: 'accusamus beatae ad facilis cum similique qui sunt',
            },
          };
        }

        throw { status: 404 };
      },
    },
    {
      timeout: 1000, // optional, in ms, defining how much time mock request would take, useful for testing spinners
      getDataFromResponse: response => response.data // optional, if you mock Axios or Fetch API, you dont need to worry about it
    },
  ),
})
```

## Multiple drivers [:arrow_up:](#table-of-content)

You can use multiple drivers if you need it. For example, if you want to use Axios by default, but also Fetch API
sometimes, you can do it like this:
```js
import axios from 'axios';
import 'isomorphic-fetch';
import { handleRequests } from 'redux-saga-requests';
import { createDriver as createAxiosDriver } from 'redux-saga-requests-axios';
import { createDriver as createFetchDriver } from 'redux-saga-requests-fetch';

handleRequests({
  driver: {
    default: createAxiosDriver(axios),
    fetch: createFetchDriver(
      window.fetch,
      {
        baseURL: 'https://my-domain.com',
        AbortController: window.AbortController,
      },
    ),
  },
});
```

As you can see, the default driver is Axios, so how to mark a request to be run by Fetch driver?
Just pass the key you assigned Fetch driver to (`fetch` in our case) in `action.meta.driver`, for instance:
```js
const fetchUsers = () => ({
  type: 'FETCH_USERS',
  request: {
    url: '/users/',
    method: 'POST',
    body: JSON.stringify(data),
    headers: {
      'Content-Type': 'application/json',
    },
  },
  meta: {
    driver: 'fetch',
  },
});
```

## Normalisation [:arrow_up:](#table-of-content)

Normalisation is a process of keeping data in such a way that information is not duplicated.
So for instance if you have a book with id `1` in multiple queries, it should be stored only in
one place anyway. This way state is better organized plus updates are easy - book has to be updated
only in one place no matter in how many queries it is present.

Typically `normalizr` is used in Redux world to normalize data, it has big disadvantage though, namely
you need to do everything manually. This library suggests 100% automatic approach, which you might
already see in GraphQL world like in Apollo library.

This library does similar thing, but not only for GraphQL, but for anything, including REST!

How does it work? By default nothing is normalized. You can pass `normalize: true` to `handleRequests`
to normalize everything, or you can use request action `meta.normalize: true` to activate it
per request type.

Now, lets say you have two queries:
```js
const fetchBooks = () => ({
  type: 'FETCH_BOOKS',
  request: { url: '/books' },
  meta: { normalize: true },
});

const fetchBook = id => ({
  type: 'FETCH_BOOK',
  request: { url: `/books/${id}` },
  meta: { normalize: true },
})
```
and `getQuery` returns the following data:
```js
import { getQuery } from 'redux-saga-requests';

const booksQuery = getQuery(state, { type: 'FETCH_BOOKS' });
// booksQuery.data is [{ id: '1', title: 'title 1'}, { id: '2', title: 'title 2'}]

const bookDetailQuery = getQuery(state, { type: 'FETCH_BOOK' });
// bookDetailQuery.data is { id: '1', title: 'title 1'}
```

Now, imagine you have a mutation to update book title. Normally you would need to do
something like that:
```js
const updateBookTitle = (id, newTitle) => ({
  type: 'UPDATE_BOOK_TITLE',
  request: { url: `books/${id}`, method: 'PATCH', data: { newTitle } },
  meta: {
    mutations: {
      FETCH_BOOKS: (data, mutationData) => data.map(v => v.id === id ? mutationData : v),
      FETCH_BOOK: (data, mutationData) => data.id === id ? mutationData : data,
    },
  },
})
```
assuming `mutationData` is equal to book with updated title.

Now, because we have queries normalized, we can also use normalization in mutation:
```js
const updateBookTitle = (id, newTitle) => ({
  type: 'UPDATE_BOOK_TITLE',
  request: { url: `books/${id}`, method: 'PATCH', data: { newTitle } },
  meta: { normalize: true },
})
```

No manual mutations! How does it work? By default all objects with `id` key are
organized by their ids. Now, if you use `normalize: true`, any object with key `id`
will be normalized, which simply means stored by id. If there is already a matching object
with the same id, new one will be deeply merged with the one already in state.
So, if only server response data from `UPDATE_BOOK_TITLE` is `{ id: '1', title: 'new title' }`,
this library will automatically figure it out to update `title` for object with `id: '1'`.

It also works with nested objects with ids, no matter how deep. If an object with id has other objects
with ids, then those will be normalized separately and parent object will have just reference to those nested
objects.

Anyway, to make those things work, the following conditions has to be meet:
1) you have to have a standardized way to identify your objects, usually this is just `id` key
2) ids have to be unique across the whole app, if not, you will need to append something to them,
the same has to be done in GraphQL world, usually adding `_typename`
3) objects with the same ids should have consistent structure, if an object like book in one
query has `title` key, it should be `title` in others, not `name` out of a sudden

Two functions which can be passed to `handleRequest` can help to meet those requirements,
`shouldObjectBeNormalized` and `getNormalisationObjectKey`.

`shouldObjectBeNormalized` can help you with 1) point, if for instance you identify
objects differently, for instance by `_id` key, then you can pass
`shouldObjectBeNormalized: obj => obj._id !== undefined` to `handleRequest`.

`getNormalisationObjectKey` allows you to pass 2nd requirement. For example, if your ids
are unique, but not across the whole app, but within object types, you could use
`getNormalisationObjectKey: obj => obj.id + obj.type` or something similar.
If that is not possible, then you could just compute a suffix yourself, for example:
```js
const getType = obj => {
  if (obj.bookTitle) {
    return 'book';
  }

  if (obj.surname) {
    return 'user';
  }

  throw 'we support only book and user object';
}

{
  getNormalisationObjectKey: obj => obj.id + getType(obj),
}
```

Point 3 should always be met, if not, your really should ask your backend developers
to keep things standardized and consistent. As a last resort, you can amend response in
`meta.getData`.

Unfortunately it does not mean you will never use `meta.mutations`. Some updates still need
to be done like usually, namely adding and removing items from array. Why? Imagine `REMOVE_BOOK`
mutation. This book could be present in many queries, library cannot know from which query
you would like to remove it. The same for `ADD_BOOK`, library cannot know to which query a book should be added,
or even as which array index. The same thing for action like `SORT_BOOKS`. This problem is only for top
level arrays though. For instance, if you have a book with some id and another key like `likedByUsers`,
then if you return new book with updated list in `likedByUsers`, this will work again automatically.

There are also 3 action meta options dedicated for normalized data:
- `optimisticData`: cousin of meta.mutation `updateDataOptimistic`,
- `revertedData`: cousin of meta.mutation `revertData`
- `localData`: cousing of meta.mutation `updateData` with `local: true`

Just attached an object or objects with ids there to update data, for example:
```js
const likeBooks = ids => ({
  type: 'LIKE_BOOKS',
  meta: {
    localData: ids.map(id => ({ id, liked: true })),
  },
})
```

## React bindings [:arrow_up:](#table-of-content)

`redux-saga-requests-react`를 설치하십시오. See
[docs](https://github.com/klis87/redux-saga-requests/tree/master/packages/redux-saga-requests-react)
for more info.


## Examples [:arrow_up:](#table-of-content)

I highly recommend to try examples how this package could be used in real applications. You could play with those demos
and see what actions are being sent with [redux-devtools](https://github.com/zalmoxisus/redux-devtools-extension).

There are following examples currently:
- [basic](https://github.com/klis87/redux-saga-requests/tree/master/examples/basic)
- [advanced](https://github.com/klis87/redux-saga-requests/tree/master/examples/advanced)
- [mutations](https://github.com/klis87/redux-saga-requests/tree/master/examples/mutations)
- [normalisation](https://github.com/klis87/redux-saga-requests/tree/master/examples/normalisation)
- [Fetch API](https://github.com/klis87/redux-saga-requests/tree/master/examples/fetch-api)
- [GraphQL](https://github.com/klis87/redux-saga-requests/tree/master/examples/graphql)
- [redux-act integration](https://github.com/klis87/redux-saga-requests/tree/master/examples/redux-act-integration)
- [mock-and-multiple-drivers](https://github.com/klis87/redux-saga-requests/tree/master/examples/mock-and-multiple-drivers)
- [server-side-rendering](https://github.com/klis87/redux-saga-requests/tree/master/examples/server-side-rendering)

## Licence [:arrow_up:](#table-of-content)

MIT
