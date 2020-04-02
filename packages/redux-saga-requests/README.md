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

Redux-Saga addon to simplify handling of AJAX requests. It supports Axios, Fetch API and GraphQL, but different
integrations could be added, as they are implemented in a plugin fashion.

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

With `redux-saga-requests`, assuming you use `axios` you could refactor a code in the following way:
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
So basically you end up writing just actions to manage your whole remote state, so no more famous boilerplate in your Redux apps!

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

To install the package, just run:
```
$ npm install redux-saga-requests
```
or you can just use CDN: `https://unpkg.com/redux-saga-requests`.

Also, you need to install a driver:
- if you use Axios, install `axios` and `redux-saga-requests-axios`:

  ```
  $ npm install axios redux-saga-requests-axios
  ```
  or CDN: `https://unpkg.com/redux-saga-requests-axios`.
- if you use Fetch API, install `isomorphic-fetch` (or a different Fetch polyfill) and `redux-saga-requests-fetch`:

  ```
  $ npm install isomorphic-fetch redux-saga-requests-fetch
  ```
  or CDN: `https://unpkg.com/redux-saga-requests-fetch`.

Of course, because this is Redux-Saga addon, you also need to install `redux-saga`.
Also, it requires to install `reselect`.

## Usage [:arrow_up:](#table-of-content)

For a quick introduction how things work, see [Motivation](#motivation-arrow_up) paragraph.

Before we go further, let's start with some naming conventions explanation and ideas behind this library.

As you probably noticed in `Motivation` section, 사용 된 내용 일부 중 하나는 '요청'키가있는 동작입니다..
지금부터 요청을 액션이라합니다. 이러한 액션이 전달되면 AJAX 요청이 자동으로 시작됩니다. 그런 다음 결과에 따라 해당 성공, 오류 또는 중단 조치가 발송됩니다.
다음 단락에서는 요청 조치에 대한 자세한 정보가 있습니다., but for now know, 그 요청 동작은 드라이버라고 불리는 것에 의해 구동됩니다. 
당신은 `handleRequest` 기능에서 드라이버를 설정했습니다. 공식적으로 지원되는 Axios, Fetch API, GraphQL 및 모의 드라이버가 있습니다, 그러나 자신의 드라이버를 작성하는 것은 매우 쉽습니다.
Just pick whatever you prefer. The key to understand is that if you know how to use Fetch API,
you know how to use Fetch API driver. `fetch` 함수에 전달할 설정 객체가 있다면,
이제 요청 액션의 `request` 키에 첨부합니다. Another information which will be explained later is that you can use `meta` key next to `request`, which is the way to pass some additional options.
One of examples can be `meta.driver` option, which allows you to define driver per request action, that's it,
you can use multiple drivers within one application. It will be described later, 지금은 핵심 개념에 집중하자.

또 다른 중요한 점은 `요청`을 `쿼리`와 `뮤테이션`로 나눌 수 있다는 것입니다.
이것은 이 라이브러리에서 사용되는 명명 규칙이며 graphql에서 빌 렸습니다.
기억해라, `쿼리`는 일부 데이터를 얻기 위해 실행 된 요청이고, 반면에 `mutation`은 데이터 업데이트를 포함하여 부작용을 일으키는 요청입니다. 
Or to think about it from different perspective,
REST를 사용하는 경우 일반적으로 쿼리는`GET`,`HEAD`,`OPTIONS` 요청입니다.,
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
what is shown in `Motivation` part. However, this library is completely compatible with it, actually it uses sagas
to power some of its functionalities. It might be possible though that one of the next releases will be rewritten to get rid
of `redux-saga` dependency, it shouldn't change this library API, just know this as a curiosity.

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

Like you probably remember, mutations are requests to cause a side effect, in the contracts to queries which
are used to get data. But how to update data received from queries? We usually do such things in reducers, but with this library
we have another way - `meta.mutations`.

As you saw in `DELETE_BOOK` mutation, there was declared `meta.mutations` object as:
```js
mutations: {
  FETCH_BOOKS: data => data.filter(book => book.id !== '1'),
}
```

Above function will be called on `DELETE_BOOK` success with `data` which is current state of `FETCH_BOOKS` query.
Whatever it returns will be used by reducer responsible for `FETCH_BOOKS` to update data.

Of course, one mutation action can update multiple queries, just add another key.

If you need to have access to data response from mutation action itself, like information about deleted book and so on,
you can access it from passed second argument - that's it, this function actually has signature `(data, mutationData) => updatedData`.

### Updating data of queries with requestType

Now, what if you want to update a data of a query with a `requestKey`? Just add `requestKey`
to queryType, for example:
```js
mutations: {
  [FETCH_BOOK_DETAIL + id]: data => data.id === id ? null : data,
}
```

Assuming we have a query action with `type: FETCH_BOOK_DETAIL` and `meta.requestKey: id`,
as you can see we just append `id` to `FETCH_BOOK_DETAIL`. The update function just returns `null`
when id is matched or data in another case, typically what you would do in a reducer.

### Local updates

You can also update your queries data locally, without even making any mutation request.
How? Just attach `meta.mutations` to any action, like this:
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
As you can see, data mutation can be defined also as object with `updateData` key, which is the same
as using update function directly. But with object we can pass some extra options, here we use
`local: true` to tell the library we want to use local mutation.

### Optimistic updates

Sometimes you don't want to wait for a mutation response to update your data, sometimes you want to update it at once.
At the same time you want to have a way to revert data update if actually mutation request failed.
This is so called optimistic update and object notation is useful for that too, like for local mutations.
Here is example:
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

So, above we have a mutation action with optimistic update for `FETCH_BOOKS` query.
`updateDataOptimistic` is called right away after `DELETE_BOOK_OPTIMISTIC` action is dispatched,
so not on success like in case for `updateData`, while `revertData` is called on `DELETE_BOOK_OPTIMISTIC_ERROR`,
so you can amend the data and revert deletion in case of an unpredicted error.
At the very same time you can still use `updateData` to further update data on `DELETE_BOOK_OPTIMISTIC_SUCCESS`.

### resetRequests

Sometimes you might need to clear data and errors of your requests, including both queries and mutations.
You can use `resetRequests` action to do it. For example:
```js
import { resetRequests } from 'redux-saga-requests';

dispatch(resetRequests()); // clear everything
dispatch(resetRequests([FETCH_BOOKS])); // clear errors and data for FETCH_BOOKS query
dispatch(resetRequests([DELETE_BOOKS])); // clear errors if any for for DELETE_BOOKS mutation
dispatch(resetRequests([FETCH_BOOKS, { requestType: FETCH_BOOK, requestKey: '1' }]));
// clear errors and data for FETCH_BOOKS and FETCH_BOOK with 1 request key
```

## Selectors [:arrow_up:](#table-of-content)

While it is possible to get a remote state on your own, it is recommented to use below selectors.
For one thing, they are already optimized, reusing cache and clearing it when necessary. Another reason is
that they return only information needed by applications, while state kept in `requestsReducer` contains
more data required by the library itself. Not to mention a situation when you use automatic normalisation.
Data in reducer is kept normalized, while you need it denormalized in your apps. Selectors already know how to denormalize it automatically and quickly, so that you don't even need to worry about it.

### getQuery

`getQuery` is a selector which returns a state for a given query. It is the selector which requires props.
Imagine you want to get a state for `FETCH_BOOKS` query which we played with earlier. You can use it like this:
```js
import { getQuery } from 'redux-saga-requests';

const booksQuery = getQuery(state, { type: 'FETCH_BOOKS' });
/* for example {
  data: [{ id: '1', name: 'Some book title' }],
  loading: false,
  error: null,
} */
```

If you are an experienced Redux developer, you might be worried about memoization of `getQuery`.
Fear not! You can call it with different props and memoization is not lost, for example:
```js
const booksQuery = getQuery(state, { type: 'FETCH_BOOKS' });
getQuery(state, { type: 'FETCH_STH_ELSE' });
booksQuery === getQuery(state, { type: 'FETCH_BOOKS' })
// returns true (unless state for FETCH_BOOKS query really changed in the meantime)
```

We only provided example for `type` prop, but here you have the list of all possibilities:
- `type: string`: just pass query action type or action itself when using action creator library
- `requestKey: string`: use it if you used `meta.requestKey` in query action
- `multiple`: set to `true` if you prefer `data` to be `[]` instead of `null` if data is empty, `false` by default
- `defaultData`: use it to represent `data` as an orbitrary object instead of `null`, use top level object though,
not recreate it multiple times not to break selector memoization

### getQuerySelector

It is almost the same as `getQuery`, the difference is that `getQuery` is the selector,
while `getQuerySelector` is the selector creator - it just returns `getQuery`.

It is helpful when you need to provide a selector without props somewhere (like in `useSelector` React hook).
So instead of doing `useSelector(state => getQuery(state, { type: 'FETCH_BOOKS' }))`
you could just `useSelector(getQuerySelector({ type: 'FETCH_BOOKS' }))`.

### getMutation

Almost the same as `getQuery`, it is just used for mutations:
```js
import { getMutation } from 'redux-saga-requests';

const deleteBookMutation = getMutation(state, { type: 'DELETE_BOOK' });
/* for example {
  loading: false,
  error: null,
} */
```

It accept `type` and optionally `requestKey` props, which work like for queries.

### getMutationSelector

Like `getQuerySelector`, it just returns `getMutation` selector.

## Reducers [:arrow_up:](#table-of-content)

You won't need to write reducers to manage remote state, because this is done already by `requestsReducer`
returned by `handleRequests`. If you need some extra state attached to queries and mutations, it is much better
just to do it on selectors level, for instance imagine you want to add a property to books:
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
Otherwise you would duplicate books state which is a very bad practice.

It is totally fine though to react on requests and responses actions in your reducers
managing a local state, for example:
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

Notice `success`, `error` and `abort` helpers, they just add proper suffixes to
request actions for convenience, so in our case they return `FETCH_BOOKS_SUCCESS`,
`FETCH_BOOKS_ERROR` and `FETCH_BOOKS_ABORT` respectively.

## Middleware [:arrow_up:](#table-of-content)

Some options passed to `handleRequests` will cause it to return an additional key - `requestsMiddleware`.
Those options are `promisify`, `cache` and `ssr`, all of which can be used independently in any
combination. All you need to do is to add `requestsMiddleware` to your middleware list, before
saga middleware. So, assuming you want to use all requests middleware (explained below), you would
adjust your code from `Motivation` like that:
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

What if you dispatch a request action somewhere and you would like to get a response in the same place?
Dispatching action by default just returns the dispatched action itself, but you can change this behaviour
by using promise middleware. All you need to is passing `promisify: true` to `handleRequests`,
which will include promise middleware in `requestsMiddleware`.

Now, you just need to add `asPromise: true` to request action meta like that:
```js
const fetchBooks = () => ({
  type: FETCH_BOOKS,
  request: { url: '/books'},
  meta: {
    asPromise: true,
  },
});
```

Then you can dispatch the action for example from a component and wait for a response:
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

Also, you can pass an optional `autoPromisify: true` flag to `handleRequests`, which will just
promisify all requests - so no need to use `meta.asPromise: true` anymore.

### Cache middleware

Sometimes you might want your responses to be cached for an amount of time or even forever (until the page is not reloaded at least).
Or, putting it another way, you would like to send a given request no more often than once for an amount of time. You can easily
achieve it with an optional cache middleware, just pass `cache: true` to `handleRequests`.

After this, you can use `meta.cache`:
```js
const fetchBooks = () => ({
  type: FETCH_BOOKS,
  request: { url: '/books'},
  meta: {
    cache: 10, // in seconds, or true to cache forever
  },
});
```

What will happen now, is that after a succesfull book fetch (to be specific after `FETCH_BOOKS_SUCCESS` is dispatched),
any `FETCH_BOOKS` actions for `10` seconds won't trigger any AJAX calls and the following `FETCH_BOOKS_SUCCESS` will contain
cached previous server response. You could also use `cache: true` to cache forever.

Another use case is that you might want to keep a separate cache for the same request action based on a cache key. For example:
```js
const fetchBook = id => ({
  type: FETCH_BOOK,
  request: { url: `/books/${id}`},
  meta: {
    cache: true,
    requestKey: id,
    requestsCapacity: 2 // optional, to clear old cache exceeding this number
  },
});

/* then, you will achieve the following behaviour:
- GET /books/1 - make request, cache /books/1
- GET /books/1 - cache hit
- GET /books/2 - make request, /books/2
- GET /books/2 - cache hit
- GET /books/1 - cache hit
- GET /books/3 - make request, cache /books/3, invalidate /books/1 cache
- GET /books/1 - make request, cache /books/1, invalidate /books/2 cache
*/
```

If you need to clear the cache manually for some reason, you can use `clearRequestsCache` action:
```js
import { clearRequestsCache } from 'redux-saga-requests';

dispatch(clearRequestsCache()) // clear the whole cache
dispatch(clearRequestsCache(FETCH_BOOKS)) // clear only FETCH_BOOKS cache
dispatch(clearRequestsCache(FETCH_BOOKS, FETCH_AUTHORS)) // clear only FETCH_BOOKS and FETCH_AUTHORS cache
```

Note however, that `clearRequestsCache` won't remove any query state, it will just remove cache timeout so that
the next time a request of a given type is dispatched, AJAX request will hit your server.
So it is like cache invalidation operation.

Also, cache is compatible with SSR by default, so if you dispatch a request action with meta cache
on your server, this information will be passed to client inside state.

### Server side rendering middleware

Server side rendering is a very complex topic and there are many ways how to go about it.
Many people use the strategy around React components, for instance they attach static methods to components which
make requests and return promises with responses, then they wrap them in `Promise.all`. I don't recommend this strategy
when using Redux, because this requires additional code and potentially double rendering on server, but if you really want
to do it, it is possible thanks to promise middleware.

However, I recommend using another approach. See [server-side-rendering-example](https://github.com/klis87/redux-saga-requests/tree/master/examples/server-side-rendering) with the complete setup, but in a nutshell you can write universal code like you would
normally write it without SSR, with just only minor additions. Here is how:

1. Before we begin, be advised that this strategy requires to dispatch requests on Redux level, at least those which have to be
fired on application load. So for instance you cannot dispatch them inside `componentDidMount`. The obvious place to dispatch them
is in your sagas, like `yield put(fetchBooks())`. However, what if your app has multiple routes, and each route has to send
different requests? Well, you need to make Redux aware of current route. I recommend to use a router with first class support for
Redux, namely [redux-first-router](https://github.com/faceyspacey/redux-first-router). If you use `react-router` though, it is
fine too, you just need to integrate it with Redux with
[connected-react-router](https://github.com/supasate/connected-react-router). Then, you can use `take` effect to listen to
routes changes and/or get current location with `select` effect. This would give you information which route is active to know
which requests to dispatch.
2. On the server you need to pass `ssr: 'server'` (`ssr: 'client'` on the client, more in next step)
option to `handleRequests`, which will include SSR middleware in `requestsMiddleware` and additionally return
`requestsPromise` which will be resolved once all requests are finished. Here you can see a possible implementation:
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
    As you can see, compared to what you would normally do in SSR for redux app, you only need to
    pass the extra `ssr` option to `handleRequests` and wait for `requestsPromise` to be resolved.

    But how does it work? The logic is based on an internal counter. Initially it is set to `0` and is
    increased by `1` after each request is initialized. Then, after each response it is decreased by `1`. So, initially after a first
    request it gets positive and after all requests are finished, its value is again set back to `0`. And this is the moment
    which means that all requests are finished and `requestsPromise` is resolved (with all success actions).
    Additionally a special `redux-saga` `END` action is dispatched to stop all of sagas.

    In case of any request error, `requestsPromise` will be rejected with response error action.

    There is also more complex case. Imagine you have a request `x`, after which you would like to dispatch
    another `y`. You cannot do it immediately because `y` requires some information from `x` response.
    Above algorythm would not wait for `y` to be finished, because on `x` response counter would be
    already reset to `0`. There are two `action.meta` attributes to help here:
    - `dependentRequestsNumber` - a positive integer, a number of requests which will be fired after this one,
    in above example we would put `dependentRequestsNumber: 1` to `x` action, because only `y` depends on `x`
    - `isDependentRequest` - mark a request as `isDependentRequest: true` when it depends on another request,
    in our example we would put `isDependentRequest: true` to `y`, because it depends on `x`

    You could even have a more complicated situation, in which you would need to dispatch `z` after `y`. Then
    you would also add `dependentRequestsNumber: 1` to `y` and `isDependentRequest: true` to `z`. Yes, a request
    can have both of those attibutes at the same time! Anyway, how does it work? Easy, just a request with
    `dependentRequestsNumber: 2` would increase counter by `3` on request and decrease by `1` on response,
    while an action with `isDependentRequest: true` would increase counter on request by `1` as usual but decrease
    it on response by `2`. So, the counter will be reset to `0` after all requests are finished, also dependent ones.

3. The last thing you need to do is to pass `ssr: 'client` to `handleRequests`, like you noticed in previous step on the client side, What does it do? Well, it will ignore request actions which match
those already dispatched during SSR. Why? Because otherwise with universal code the same job done on the server
would be repeated on the client.

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

Just install `redux-saga-requests-react`. See
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
