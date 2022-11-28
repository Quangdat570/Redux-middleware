# Redux Middleware 

Là lớp nằm giữa Reducers và Dispatch Actions. Vị trí mà Middleware hoạt động là trước khi Reducers nhận được Actions và sau khi 1 Action được dispatch(). Middleware trong Redux được biết đến nhiều nhất trong việc xử lý ASYNC Action - đó là những Action không sẵn sàng ngay khi 1 Action Creator được gọi tới, thông thường ở đây là các API request

![markdown](https://res.cloudinary.com/practicaldev/image/fetch/s--BlRvc7W8--/c_imagga_scale,f_auto,fl_progressive,h_420,q_auto,w_1000/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/wdr2pgv8ax9iqilbgqj2.png)

# Attempt #1: Logging Manually
-  thêm trực tiếp lệnh log vào trước sau mỗi khi dispatch action
```php
store.dispatch(addTodo('Use Redux'))

const action = addTodo('Use Redux')

console.log('dispatching', action)
store.dispatch(action)
console.log('next state', store.getState())
```

# Attempt #2: Wrapping Dispatch
```php
function dispatchAndLog(store, action) {
  console.log('dispatching', action)
  store.dispatch(action)
  console.log('next state', store.getState())
}
```





# Attempt #3: Monkeypatching Dispatch
 là kỹ thuật mà ta sẽ chỉnh sửa, mở rộng 1 phần của ứng dụng mà không làm thay đổi hệ thống. Áp dụng kỹ thuật này vào đây ta sẽ "patching" lại hàm dispatch của Redux
 ```php
const next = store.dispatch
store.dispatch = function dispatchAndLog(action) {
  console.log('dispatching', action)
  let result = next(action)
  console.log('next state', store.getState())
  return result
}
```

# Problem: Crash Reporting
- trong Production, việc dựa vào sự kiện onError của Javascript là không đủ giá trị / tin tưởng để Debug cũng như tracking quá trình cua bugs. Sử dụng 1 dịch vụ third party như Sentry mang lại hiệu quả cao hơn nhiều. Dù vậy việc dựa vào external API/ external module như vậy thì cần thiết cho chúng ta viết những interface có khả năng tháo lắp, thay đổi dễ dàng - hãy nghĩ tới Adapter Design Pattern. Bằng không ta sẽ không thể xây dựng được 1 loose eco-system. Điều này cũng đúng với Logging Middleware ở phía trên, nghĩ theo hướng này ta cần thay đổi các middleware trở nên tách biệt với nhau:

```php
function patchStoreToAddLogging(store) {
  const next = store.dispatch
  store.dispatch = function dispatchAndLog(action) {
    console.log('dispatching', action)
    let result = next(action)
    console.log('next state', store.getState())
    return result
  }
}

function patchStoreToAddCrashReporting(store) {
  const next = store.dispatch
  store.dispatch = function dispatchAndReportErrors(action) {
    try {
      return next(action)
    } catch (err) {
      console.error('Caught an exception!', err)
      Raven.captureException(err, {
        extra: {
          action,
          state: store.getState()
        }
      })
      throw err
    }
  }
}
```
- Nếu các chức năng này được xuất bản dưới dạng các mô-đun riêng biệt, sau này chúng ta có thể sử dụng chúng để vá cửa hàng của mình:
```php

patchStoreToAddLogging(store)
patchStoreToAddCrashReporting(store)

```

# Attempt #4: Hiding Monkeypatching
- kỹ thuật monkeypatch mang lại nhiều rủi ro, bởi vậy thay vì trực tiếp thay thế hàm dispatch, ta sẽ trả về 1 hàm dispatch mới wrap bên ngoài hàm dispatch có sẵn của Redux.

```php
function logger(store) {
  const next = store.dispatch

  // Previously:
  // store.dispatch = function dispatchAndLog(action) {

  return function dispatchAndLog(action) {
    console.log('dispatching', action)
    let result = next(action)
    console.log('next state', store.getState())
    return result
  }
}
```

- Ta cần 1 helper để apply các hàm dispatch này vào lớp Middleware của Redux - thực tế thì Redux có helper này nhưng interface sẽ khác với ví dụ chúng ta làm

```php
function applyMiddlewareByMonkeypatching(store, middlewares) {
  middlewares = middlewares.slice()
  middlewares.reverse()

  // Transform dispatch function with each middleware.
  middlewares.forEach(middleware => (store.dispatch = middleware(store)))

  applyMiddlewareByMonkeypatching(store, [logger, crashReporter])
}
```

# Attempt #5: Removing Monkeypatching
-  các middleware thay vì đọc hàm dispatch trực tiếp từ store, thì ta sẽ truyền tham số next vào thể hiện cho hàm dispatch đã được middleware trước đó trong chuỗi wrap lại.

```php

function logger(store) {
  return function wrapDispatchToAddLogging(next) {
    return function dispatchAndLog(action) {
      console.log('dispatching', action)
      let result = next(action)
      console.log('next state', store.getState())
      return result
    }
  }
}
```

- Sử dụng ES6 cho code khô thoáng hơn bằng arrow function 
```php

const logger = store => next => action => {
  console.log('dispatching', action)
  let result = next(action)
  console.log('next state', store.getState())
  return result
}

const crashReporter = store => next => action => {
  try {
    return next(action)
  } catch (err) {
    console.error('Caught an exception!', err)
    Raven.captureException(err, {
      extra: {
        action,
        state: store.getState()
      }
    })
    throw err
  }
}
```

# Attempt #6: Naïvely Applying the Middleware
- Để apply middles vào Redux store, ta sẽ viết 1 helper nhận vào các Middlewares cũng như Store ban đầu, Kết quả trả về là store mới, với hàm dispatch đã được wrap lại bởi chuỗi Middleware

```php
function applyMiddleware(store, middlewares) {
  middlewares = middlewares.slice()
  middlewares.reverse()
  let dispatch = store.dispatch
  middlewares.forEach(middleware =>
    dispatch = middleware(store)(dispatch)
  )
  return Object.assign({}, store, { dispatch })
}

```

- Hàm applyMiddleware thật trong Redux API tương đương chức năng như hàm chúng ta vừa viết nhưng tất nhiên là toàn năng hơn:

- Nó đảm bảo chỉ cung cấp 1 số API cần thiết cho các Middleware truy cập ( hàm chúng ta vừa viết thì middleware toàn quyền truy cập vào store luôn )

- Cung cấp tính năng nếu bạn gọi trực tiếp store.dispatch() bên trong middleware thì action sẽ chạy lại từ đầu chuỗi Middleware (quan trọng để implement ASYNC middleware)

- Đảm bảo chỉ được cung cấp middleware 1 lần duy nhất


# Redux Thunk
- Redux Thunk là một Middleware cho phép bạn viết các Action trả về một function thay vì một plain javascript object bằng cách trì hoãn việc đưa action đến reducer.
- Redux Thunk được sử dụng để xử lý các logic bất đồng bộ phức tạp cần truy cập đến Store hoặc đơn giản là việc lấy dữ liệu như Ajax request.

## Writing Thunks
```php
// fetchTodoById is the "thunk action creator"
export function fetchTodoById(todoId) {
  // fetchTodoByIdThunk is the "thunk function"
  return async function fetchTodoByIdThunk(dispatch, getState) {
    const response = await client.get(`/fakeApi/todo/${todoId}`)
    dispatch(todosLoaded(response.todos))
  }
}
```

- Hàm Thunk và action có thể được viết bằng cách sử dụng từ khóa hàm hoặc hàm mũi tên - không có sự khác biệt có ý nghĩa nào ở đây. Cũng có thể viết thunk fetchTodoById tương tự bằng cách sử dụng các hàm mũi tên, như sau:

```php
export const fetchTodoById = todoId => async dispatch => {
  const response = await client.get(`/fakeApi/todo/${todoId}`)
  dispatch(todosLoaded(response.todos))
}
```

-Trong cả hai trường hợp, thunk được gửi đi bằng cách gọi action creator, giống như cách bạn gửi bất kỳ hành động Redux nào khác: 

```php
function TodoComponent({ todoId }) {
  const dispatch = useDispatch()

  const onFetchClicked = () => {
    // Calls the thunk action creator, and passes the thunk function to dispatch
    dispatch(fetchTodoById(todoId))
  }
}
```


# RTK Query
- là một addon trong bộ thư viện Redux Toolkit. Nó giúp chúng ta thực hiện data fetching một cách đơn giản hơn thay vì sử dụng createAsyncThunk để thực hiện async action. Chú ý RTK Query là dùng để query (kết nối API), chứ không phải dùng để code async trong Redux thay cho createAsyncThunk.

- RTK Query bao gồm : APIs, Bundle Size

1. APIs

 Truy vấn RTK bao gồm các API :
- `createApi()`

- `fetchBaseQuery():` Một trình bao bọc nhỏ xung quanh tìm nạp nhằm mục đích đơn giản hóa các yêu cầu

- `<ApiProvider />:` Có thể được sử dụng `Provider` nếu bạn chưa có `store` Redux.

- `setupListeners():` Một tiện ích được sử dụng để kích hoạt các hành vi `refetchOnMount` và `refetchOnReconnect`.

2. Basic Usage
- Create an API Slice

```php
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react'
import type { Pokemon } from './types'

// Define a service using a base URL and expected endpoints
export const pokemonApi = createApi({
  reducerPath: 'pokemonApi',
  baseQuery: fetchBaseQuery({ baseUrl: 'https://pokeapi.co/api/v2/' }),
  endpoints: (builder) => ({
    getPokemonByName: builder.query<Pokemon, string>({
      query: (name) => `pokemon/${name}`,
    }),
  }),
})

// Export hooks for usage in functional components, which are
// auto-generated based on the defined endpoints
export const { useGetPokemonByNameQuery } = pokemonApi
```

- Configure the Store

```php
import { configureStore } from '@reduxjs/toolkit'
// Or from '@reduxjs/toolkit/query/react'
import { setupListeners } from '@reduxjs/toolkit/query'
import { pokemonApi } from './services/pokemon'

export const store = configureStore({
  reducer: {
    // Add the generated reducer as a specific top-level slice
    [pokemonApi.reducerPath]: pokemonApi.reducer,
  },
  // Adding the api middleware enables caching, invalidation, polling,
  // and other useful features of `rtk-query`.
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware().concat(pokemonApi.middleware),
})

// optional, but required for refetchOnFocus/refetchOnReconnect behaviors
// see `setupListeners` docs - takes an optional callback as the 2nd arg for customization
setupListeners(store.dispatch)
```

- Use Hooks in Components

```php
import * as React from 'react'
import { useGetPokemonByNameQuery } from './services/pokemon'

export default function App() {
  // Using a query hook automatically fetches data and returns query values
  const { data, error, isLoading } = useGetPokemonByNameQuery('bulbasaur')
  // Individual hooks are also accessible under the generated endpoints:
  // const { data, error, isLoading } = pokemonApi.endpoints.getPokemonByName.useQuery('bulbasaur')
  
  // render UI based on data and loading state
}
```



