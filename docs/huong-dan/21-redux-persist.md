# Bài 21 — redux-persist: giữ giỏ hàng và phiên đăng nhập

> **Phần 3 · Frontend — Quản lý state** — Thời lượng ước tính: **~55 phút**
> ⬅️ Bài trước: [20 — `createAsyncThunk` và `extraReducers`](20-async-thunk.md) · Bài sau: [22 — RTK Query: `createApi`, cache và `invalidatesTags`](22-rtk-query.md) ➡️
> 🏠 [Mục lục](README.md)

---

## 🎯 Sau bài này bạn sẽ

- Giải thích được vì sao state Redux **bốc hơi** mỗi lần bấm F5, còn giỏ hàng Yotea thì **không**.
- Phân biệt được `localStorage`, `sessionStorage`, `cookie` — biết chọn cái nào cho việc gì.
- Đọc hiểu từng dòng `yotea-fe/src/redux/store.js`: `persistConfig`, `persistReducer`, `persistStore`, `whitelist`.
- Biết `PersistGate` làm gì và vì sao thiếu nó thì F5 ở trang admin sẽ bị văng ra.
- Mở DevTools đọc được khoá `persist:root`, giải thích được vì sao `isAuthenticate()` phải `JSON.parse` **hai lần**.
- Tự quyết được **cái gì đáng persist** — và trả lời được: có nên đưa `topping` (slice bạn viết ở bài 19–20) vào `whitelist` không.
- Nhìn ra lỗ hổng thật của việc để JWT + `role` trong localStorage, hiểu chính xác kẻ xấu **làm được gì / không làm được gì**.

## 📋 Cần chuẩn bị

- Đã hoàn thành [Bài 20 — `createAsyncThunk` và `extraReducers`](20-async-thunk.md).
- Đã có `src/redux/toppingSlice.js` (bài 19) và thunk `getToppings` chạy được trên `ToppingPage` (bài 20).
- Backend đang chạy (`yotea-be` → `npm start`), frontend đang chạy (`yotea-fe` → `npm start`).
- Chrome hoặc Edge — bài này dùng tab **Application** của DevTools rất nhiều.

> Ở bài trước bạn đã cho `ToppingPage` **tự gọi API** bằng `createAsyncThunk` và đổ dữ liệu vào Redux.
> Bài này ta làm tiếp câu hỏi mà bài trước để ngỏ: **dữ liệu trong Redux sống được bao lâu?**
> Câu trả lời khá phũ — đúng bằng thời gian bạn chưa bấm F5.

---

## 1. Vấn đề: Redux sống trong RAM, F5 là chết

Redux store chỉ là **một object JavaScript** nằm trong bộ nhớ của tab. Bấm F5 = trình duyệt vứt toàn bộ
môi trường JS cũ đi và chạy lại `src/index.js` từ đầu. Mọi reducer quay về `initialState`.

Thử hình dung nếu Yotea **không có** redux-persist:

| Bạn làm gì | Điều gì xảy ra |
|---|---|
| Thêm 3 ly trà sữa vào giỏ | `state.cart.cart` có 3 phần tử — icon giỏ hàng hiện số **3** |
| Bấm F5 (hoặc mở lại link từ Zalo) | `cartSlice` về `{ cart: [] }` → **giỏ hàng trống trơn** |
| Đăng nhập, đang xem trang Đơn hàng | `state.auth.isLogged = true` |
| Bấm F5 | `authSlice` về `{ isLogged: false, value: {} }` → `PrivateRouter` đá bạn về `/login` |

Đây không phải bug của Redux — đây là **bản chất** của RAM. Muốn dữ liệu sống qua lần F5, phải **ghi
nó xuống ổ cứng** của trình duyệt rồi **đọc lại** lúc khởi động. `redux-persist` làm hộ bạn hai việc đó.

```
┌────────── PHIÊN 1 ──────────┐        ┌────────── PHIÊN 2 (sau F5) ──────────┐
 dispatch(addCart(...))                  index.js chạy lại
        │                                        │
        ▼  rootReducer đổi state                 ▼  persistReducer đọc localStorage
 redux-persist tự ghi                     dispatch(persist/REHYDRATE)
        │                                        │
        ▼                                        ▼
 localStorage["persist:root"] ──────────►  state.cart khôi phục y như cũ
```

---

## 2. Trình duyệt cất dữ liệu ở đâu được?

Có 3 chỗ phổ biến. Chọn sai chỗ là ăn bug hoặc ăn lỗ hổng bảo mật.

| | `localStorage` | `sessionStorage` | `cookie` |
|---|---|---|---|
| **Dung lượng** | ~5–10 MB | ~5 MB | **~4 KB** (rất bé) |
| **Sống bao lâu** | **Vĩnh viễn** cho tới khi bị xoá | Chỉ trong **một tab**; đóng tab là mất | Theo `Expires` / `Max-Age` |
| **Tự gửi kèm request?** | **KHÔNG** — phải tự gắn vào header | **KHÔNG** | **CÓ** — trình duyệt tự đính vào mọi request cùng domain |
| **JavaScript đọc được?** | Có | Có | Có — **trừ khi** cookie có cờ `httpOnly` |
| **Chia sẻ giữa các tab** | Có (cùng origin) | Không | Có |
| **Kiểu lưu được** | Chỉ **chuỗi** | Chỉ **chuỗi** | Chỉ chuỗi |

Ba câu để nhớ:

- **`localStorage`** — "tủ để đồ lâu dài của website này". Hợp với giỏ hàng, theme sáng/tối, ngôn ngữ.
- **`sessionStorage`** — "túi dùng một lần trong tab này". Hợp với bước đang dở của form dài.
- **`cookie`** — "tấm vé được đính tự động vào mọi request". Là chỗ **duy nhất** đặt được `httpOnly`
  để JavaScript **không** đọc được — nên là chỗ chuẩn nhất giữ token đăng nhập.

Yotea chọn `localStorage`: **tiện nhất**, nhưng không phải **an toàn nhất** — mục 8 sẽ mổ kỹ.

> 📖 **Thuật ngữ:** *rehydrate* (bù nước) — thao tác đọc dữ liệu đã lưu từ storage rồi "bơm" ngược
> vào Redux store lúc app khởi động. Chữ này xuất hiện liên tục trong redux-persist.

---

## 3. Soi code thật: `yotea-fe/src/redux/store.js`

File này chỉ **37 dòng** nhưng là trái tim của cả bài. Xem phần import trước.

`yotea-fe/src/redux/store.js:1-11`

```js
import storage from "redux-persist/lib/storage";
import { persistStore, persistReducer } from "redux-persist";
import rootReducer from "./rootReducer";
import { applyMiddleware, createStore } from "redux";
import { thunk } from "redux-thunk";
import { setupListeners } from "@reduxjs/toolkit/query/react";
import { productApi } from "../api/product";
import { sliderApi } from "../api/slider";
import { cateProductApi } from "../api/category";
import { userApi } from "../api/user";
import { newsApi } from "../api/news";
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 1 | `import storage from "redux-persist/lib/storage"` | Đây chính là **`window.localStorage`** đã bọc thành Promise. Muốn đổi sang sessionStorage thì sửa đường dẫn thành `redux-persist/lib/storage/session` |
| 2 | `persistStore, persistReducer` | Hai hàm chính: một cái bọc reducer, một cái tạo "người quản lý ghi/đọc" |
| 3 | `rootReducer` | Reducer tổng đã gộp 16 reducer (11 slice + 5 RTK Query) ở [Bài 19](19-redux-toolkit-co-ban.md) |
| 4 | `applyMiddleware, createStore` từ `redux` | API Redux **đời cũ** — xem hộp cảnh báo dưới |
| 5 | `import { thunk } from "redux-thunk"` | redux-thunk **v3** export **có tên** (v2 là export mặc định) — sai ngoặc nhọn là crash |
| 6–11 | `setupListeners` + 5 `xxxApi` | Phần của RTK Query, bàn ở [Bài 22](22-rtk-query.md) |

Giờ tới phần cấu hình — đoạn quan trọng nhất bài.

`yotea-fe/src/redux/store.js:13-36`

```js
const persistConfig = {
  key: "root",
  storage,
  whitelist: ["auth", "cart"],
};

const persistedReducer = persistReducer(persistConfig, rootReducer);

const middleware = [
  thunk,
  productApi.middleware,
  sliderApi.middleware,
  cateProductApi.middleware,
  userApi.middleware,
  newsApi.middleware,
];

export const store = createStore(
  persistedReducer,
  applyMiddleware(...middleware)
);
export default persistStore(store);

setupListeners(store.dispatch);
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 13 | `const persistConfig = {` | Bản khai "lưu cái gì, lưu ở đâu, lưu dưới tên gì" |
| 14 | `key: "root"` | Tên khoá trong localStorage = **`persist:` + key** → **`persist:root`**. Đổi thành `key: "yotea"` thì khoá là `persist:yotea` |
| 15 | `storage,` | Object shorthand — viết đủ là `storage: storage`, tức localStorage ở dòng 1 |
| 16 | `whitelist: ["auth", "cart"]` | **Chỉ hai slice này được ghi xuống đĩa.** 14 key còn lại của root state bị bỏ qua hoàn toàn |
| 19 | `persistReducer(persistConfig, rootReducer)` | **Bọc** reducer tổng lại. Reducer mới biết thêm 2 việc: (a) tự ghi state xuống storage sau mỗi action, (b) xử lý action `persist/REHYDRATE` để nhét dữ liệu cũ vào state |
| 21–28 | mảng `middleware` | `thunk` (cho `createAsyncThunk` bài 20) + 5 middleware cache của RTK Query |
| 30–33 | `createStore(persistedReducer, applyMiddleware(...middleware))` | Tạo store từ reducer **đã bọc**, không phải `rootReducer` gốc. `...middleware` là **spread** — trải mảng thành 6 tham số rời |
| 34 | `export default persistStore(store)` | Tạo **persistor** — đối tượng theo dõi tiến trình ghi/đọc. **Đây là export mặc định của file, KHÔNG phải store** |
| 36 | `setupListeners(store.dispatch)` | Của RTK Query (refetch khi focus lại tab). Đặt **sau** `export default` — chạy được nhưng viết lộn xộn |

Hai hàm hay bị nhầm, nhớ bằng một bảng:

| Hàm | Trả về | Nhiệm vụ |
|---|---|---|
| `persistReducer(config, reducer)` | Một **reducer mới** | Phần **ghi**: sau mỗi action, chép các slice trong whitelist xuống storage — và hiểu action `persist/REHYDRATE` |
| `persistStore(store)` | Một **persistor** | Phần **đọc**: đọc `persist:root` rồi `dispatch({ type: "persist/REHYDRATE", payload })`. Cấp thêm `persistor.purge()`, `.flush()`, `.pause()` |

Thứ tự bắt buộc: **bọc reducer → tạo store → mới `persistStore(store)`**. Đảo là hỏng, vì persistor
cần một store có sẵn để dispatch vào.

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** dự án đã cài `@reduxjs/toolkit@2` (`yotea-fe/package.json:12`)
> nhưng vẫn dựng store bằng `createStore` + `applyMiddleware` — API bị đánh dấu `@deprecated` từ Redux v5.
> Hệ quả nặng nhất khi học: **mất Redux DevTools**, không xem được state bằng extension.
> Cách đúng là `configureStore` — ta refactor ở [Bài 34](34-refactor-du-an.md).

---

## 4. `whitelist` vs `blacklist` — chọn cái gì để lưu

`persistConfig` cho phép **một trong hai** (đừng dùng cả hai cùng lúc):

```js
// đoạn minh hoạ, KHÔNG có trong dự án — chỉ để so sánh
whitelist: ["auth", "cart"],    // CHỈ lưu 2 cái này, còn lại bỏ hết
blacklist: ["product", "news"], // lưu TẤT CẢ, TRỪ 2 cái này
```

| | `whitelist` | `blacklist` |
|---|---|---|
| Ý nghĩa | Danh sách **được phép** lưu | Danh sách **cấm** lưu |
| Slice mới thêm sau này | **Không** lưu | **Có** lưu (tự động!) |
| An toàn khi dự án lớn dần | ✅ | ❌ |

Giả sử tháng sau bạn thêm slice `paymentSlice` chứa số thẻ tạm. Dùng `whitelist` thì slice mới
**mặc định không được lưu**. Dùng `blacklist` thì nó **tự động bị ghi xuống đĩa** mà không ai để ý.
Nguyên tắc bảo mật: **mặc định là cấm, mở cửa từng cái một.** Yotea chọn `whitelist` — điểm này **đúng**.

### 4.1. Vì sao chỉ `auth` và `cart`, không phải `product`, `news`, `topping`?

Phân loại state theo **nguồn sự thật (source of truth)**:

| Loại state | Nguồn sự thật ở đâu | Ví dụ trong Yotea | Persist? |
|---|---|---|---|
| Dữ liệu do **client tạo ra** | Chỉ có ở trình duyệt, server không biết | `cart` — giỏ chưa đặt, `id` sinh bằng `uuidv4()` | ✅ **CÓ** — mất là mất luôn |
| **Phiên đăng nhập** | Token do server cấp, client phải giữ để gửi lại | `auth` — `{ token, user }` | ✅ **CÓ** — không giữ thì F5 là văng ra |
| **Dữ liệu server** | MongoDB. Client chỉ **mượn xem** | `product`, `news`, `slider`, **`topping`** | ❌ **KHÔNG** |
| **Cờ UI tạm** | Không ở đâu cả | `wishlist.showWishlist` | ❌ **KHÔNG** |
| **Cache RTK Query** | Thư viện tự quản vòng đời | `productApi`, `userApi`… | ❌ **KHÔNG** — persist vào là hỏng cơ chế cache |

Ba lý do cụ thể để **không** persist dữ liệu server:

1. **Dữ liệu cũ (stale).** Admin đổi giá trân châu từ 5.000 lên 7.000. Người đã persist danh sách cũ
   sẽ thấy 5.000 **mãi mãi** cho tới khi bạn viết thêm code làm mới — mà bạn sẽ quên.
2. **Phình localStorage.** Quota chỉ ~5 MB cho cả origin. Persist 500 sản phẩm + 200 tin tức là ăn
   `QuotaExceededError` — và khi đó **cả `auth` lẫn `cart` cũng ghi không được**, hỏng đúng cái cần.
3. **Vô nghĩa.** Bạn **vẫn phải gọi API** để có dữ liệu mới. Persist chỉ tiết kiệm ~200 ms lần render
   đầu, đổi lấy nguy cơ hiển thị **sai giá tiền**.

> 💡 **Quy tắc bỏ túi:** *"Cái gì server biết thì để server giữ. Chỉ persist cái mà mất đi thì không
> ai lấy lại được cho bạn."*

Áp dụng cho slice `topping` bạn viết ở [Bài 19](19-redux-toolkit-co-ban.md)–[Bài 20](20-async-thunk.md):
danh sách topping nằm trong MongoDB, thunk `getToppings` gọi lại là có ngay ⇒ **KHÔNG đưa `"topping"`
vào whitelist.** Ta sẽ chứng minh bằng thực nghiệm ở mục 7.

---

## 5. `PersistGate` — người gác cổng ở `index.js`

Có một vấn đề về **thời gian**: đọc storage là thao tác **bất đồng bộ**. Nếu React render ngay lập tức
thì trong vài mili-giây đầu state vẫn là `initialState` — tức `isLogged: false`. `PrivateRouter` thấy
vậy sẽ lập tức `<Navigate to="/login" />`, và bạn bị đá ra dù rõ ràng vẫn đang đăng nhập. Đây là bug
kinh điển "F5 ở trang admin thì bị văng ra". `PersistGate` sinh ra để chặn đúng khoảnh khắc đó.

`yotea-fe/src/index.js:6-19`

```jsx
import { Provider } from "react-redux";
import { PersistGate } from "redux-persist/integration/react";
import persistor, { store } from "./redux/store";

const root = ReactDOM.createRoot(document.getElementById("root"));
root.render(
  <React.StrictMode>
    <Provider store={store}>
      <PersistGate loading={null} persistor={persistor}>
        <App />
      </PersistGate>
    </Provider>
  </React.StrictMode>
);
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 8 | `import persistor, { store }` | **Default import** là *persistor* (`store.js:34`), **named import** mới là *store*. Chỗ dễ nhầm nhất |
| 13 | `<Provider store={store}>` | Bơm store vào cây React để `useSelector` / `useDispatch` dùng được |
| 14 | `<PersistGate ... persistor={persistor}>` | **Không render `<App />`** cho tới khi persistor báo "rehydrate xong" |
| 14 | `loading={null}` | Trong lúc chờ thì hiển thị **không gì cả** (màn trắng) |
| 15 | `<App />` | Chỉ được vẽ sau khi state đã đầy đủ |

Thứ tự lồng **bắt buộc**: `Provider` ngoài → `PersistGate` trong. Vì `PersistGate` cần dispatch action
`persist/REHYDRATE` vào store, mà muốn dispatch thì phải nằm trong `Provider`.

> 💡 **Mẹo:** `loading={null}` gây cảm giác "trang trắng chớp một cái". Dự án đã có sẵn
> `yotea-fe/src/components/Loading.js` — thay vào sẽ mượt hơn: `<PersistGate loading={<Loading />} ...>`.

---

## 6. 🔬 Thực hành quan sát: mở ruột `persist:root`

Phần này **không gõ code**, chỉ nhìn. Nhưng đây là 10 phút quan trọng nhất bài.

**Bước 1 — mở đúng chỗ.** Chạy frontend, vào `http://localhost:3000`, nhấn **F12** → tab **Application**
→ cột trái **Storage → Local Storage → `http://localhost:3000`**. Bạn sẽ thấy đúng **một dòng**, cột
Key là **`persist:root`**.

**Bước 2 — lúc chưa đăng nhập, giỏ trống.** Bấm vào dòng đó, ô Value hiện đại khái:

```json
{"cart":"{\"cart\":[]}","auth":"{\"isLogged\":false,\"value\":{}}","_persist":"{\"version\":-1,\"rehydrated\":true}"}
```

**Bước 3 — thêm món vào giỏ rồi đăng nhập, xem lại.** Vào một sản phẩm → chọn đá/đường → **Thêm vào
giỏ hàng** → rồi đăng nhập. Bấm lại `persist:root` (có thể phải bấm nút ⟳ Refresh của bảng):

```json
{
  "cart": "{\"cart\":[{\"id\":\"9f1c…\",\"productId\":\"663f…\",\"productName\":\"Trà sữa trân châu\",\"productPrice\":35000,\"quantity\":2,\"ice\":30,\"sugar\":50}]}",
  "auth": "{\"isLogged\":true,\"value\":{\"token\":\"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9…\",\"user\":{\"_id\":\"663f…\",\"fullName\":\"Nguyễn Văn A\",\"role\":0,\"active\":true}}}",
  "_persist": "{\"version\":-1,\"rehydrated\":true}"
}
```

*(DevTools hiển thị trên một dòng dài; mình xuống dòng cho dễ đọc.)*

**Bước 4 — câu hỏi cốt lõi: vì sao lại là CHUỖI nằm trong object?**

Nhìn kỹ: giá trị của `"auth"` **không phải** object JSON, mà là một **chuỗi** chứa JSON — nhận ra ngay
vì có `\"` (dấu ngoặc kép bị escape). Tức redux-persist **serialize hai tầng**:

```
tầng 1 — cả gói:     JSON.stringify({ auth: <chuỗi>, cart: <chuỗi>, _persist: <chuỗi> })
tầng 2 — từng slice: JSON.stringify(state.auth),  JSON.stringify(state.cart)
```

Vì sao phải phiền vậy? Vì redux-persist muốn **ghi/đọc từng slice độc lập**:

- Chỉ `cart` đổi thì **chỉ cần** stringify lại slice `cart`, không phải serialize cả cây state → nhanh hơn nhiều.
- Nếu một slice hỏng dữ liệu (JSON lỗi), chỉ slice đó rehydrate thất bại — các slice khác **vẫn sống**.
- Mỗi slice có thể có `persistConfig` riêng lồng bên trong (tính năng nested persist).

**Và đây chính là lời giải cho đoạn code kỳ quái nhất dự án.**

`yotea-fe/src/utils/localStorage.js:1-4`

```js
export const isAuthenticate = () => {
  return JSON.parse(JSON.parse(localStorage.getItem("persist:root")).auth)
    .value;
};
```

**Bóc từng bước:**

| Bước | Biểu thức | Kết quả |
|---|---|---|
| 1 | `localStorage.getItem("persist:root")` | **Chuỗi** cả gói (tầng 1) |
| 2 | `JSON.parse(...)` — **lần 1** | Object `{ auth: "<chuỗi>", cart: "<chuỗi>", _persist: "<chuỗi>" }` |
| 3 | `.auth` | **Vẫn là chuỗi!** (tầng 2 chưa mở) |
| 4 | `JSON.parse(...)` — **lần 2** | Object `{ isLogged: true, value: { token, user } }` |
| 5 | `.value` | `{ token, user }` ← thứ hàm này trả về |

⇒ **Hai lần `JSON.parse` không phải do lập trình viên viết ẩu — nó là hệ quả trực tiếp của cách
redux-persist serialize hai tầng.** Giờ bạn đã hiểu tại sao.

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** tên hàm là `isAuthenticate` (nghe như trả về `true/false`)
> nhưng nó trả về **object `{ token, user }`**. Tên đúng phải là `getAuth()` hoặc `getCredentials()`.

### 6.1. Cầu nối với Bài 18: vì sao tầng `api/` đọc thẳng localStorage?

Ở [Bài 18](18-tang-api-axios.md) bạn đã gặp cú pháp lạ này khi tự viết `src/api/topping.js`:

`yotea-fe/src/api/category.js:18-25`

```js
export const add = (category, { token, user } = isAuthenticate()) => {
  const url = `/${DB_NAME}/${user._id}`;
  return instance.post(url, category, {
    headers: {
      Authorization: `Bearer ${token}`,
    },
  });
};
```

Ghép hai bài lại, bức tranh mới đầy đủ:

```
   Redux store (RAM)                 localStorage (đĩa)
   state.auth.value  ──ghi──►  persist:root → "auth" → value
        ▲                                  │
        │ useSelector(selectAuth)          │ isAuthenticate()
        │                                  ▼
   Component (JSX)                    src/api/*.js  ──► axios ──► Backend
```

| | Component | Tầng `api/` |
|---|---|---|
| Lấy auth bằng | `useSelector(selectAuth)` | `isAuthenticate()` |
| Đọc từ | **Redux store** (RAM) | **localStorage** (đĩa) |
| Tự cập nhật khi state đổi | ✅ có (re-render) | ❌ không |

Vì sao tác giả làm vậy? Vì file trong `src/api/` là **module JavaScript thuần**, không phải React
component — nó **không gọi được hook** `useSelector`. Muốn lấy token trong module thuần thì hoặc
`import { store }` rồi `store.getState()`, hoặc đọc thẳng localStorage. Dự án chọn cách hai.

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** đây là **hai nguồn sự thật song song**. Ngay sau
> `dispatch(signin(data))`, Redux đã có token mới nhưng redux-persist ghi xuống đĩa **bất đồng bộ**.
> Gọi API đúng khoảnh khắc đó, `isAuthenticate()` có thể đọc trúng **token cũ hoặc rỗng** → gửi lên
> `Authorization: Bearer undefined`. Race condition thật, rất khó debug. Cách đúng: **axios interceptor**
> đọc từ `store.getState().auth.value.token` — [Bài 34](34-refactor-du-an.md).
> Thêm bẫy nữa: nếu localStorage trống (chưa từng mở app), `getItem` trả `null` → `JSON.parse(null)`
> ra `null` → `.auth` trên `null` → **`TypeError: Cannot read properties of null (reading 'auth')`**,
> trắng màn hình. Hàm này không có `try/catch`, không có giá trị dự phòng.

---

## 7. 🔒 Phân tích bảo mật: JWT và `role` nằm trong localStorage

Phần này rất đáng đọc kỹ — nội dung được hỏi nhiều nhất khi bảo vệ đồ án.

### 7.1. Sự thật thứ nhất: mọi đoạn JS trên trang đều đọc được token

`localStorage` **không có** khái niệm "chỉ code của tôi mới đọc được". Mở Console gõ đúng một dòng:

```js
JSON.parse(JSON.parse(localStorage.getItem("persist:root")).auth).value.token
```

→ in ra nguyên vẹn chuỗi JWT. Bạn làm được thì **mọi script chạy trên trang đó cũng làm được**: một
thư viện npm bị chèn mã độc, một script quảng cáo, một bình luận chứa `<script>` mà backend quên lọc…
Đó là **XSS (Cross-Site Scripting)**, và với token trong localStorage thì hậu quả là **mất token** —
kẻ tấn công gửi nó về server của chúng và mạo danh bạn cho tới khi token hết hạn (3 giờ).

| | JWT trong `localStorage` (Yotea) | JWT trong cookie `httpOnly` |
|---|---|---|
| JavaScript đọc được | **Có** → XSS lấy được token | **Không** → XSS không lấy được token |
| Gửi lên server bằng | Tự gắn `Authorization: Bearer ...` từng request | Trình duyệt **tự** đính vào mọi request |
| Rủi ro CSRF | Thấp (phải cố ý gắn header) | Có — cần `SameSite` + CSRF token |
| Độ khó triển khai | Rất dễ | Phải sửa cả backend (`res.cookie`) lẫn CORS (`credentials: true`) |

Không phương án nào miễn phí: localStorage sợ XSS, cookie sợ CSRF. Nhưng **XSS phá được cả hai**, còn
`httpOnly` ít nhất **cứu được cái token**. Đó là lý do hệ thống thật thường chọn cookie `httpOnly` + refresh token.

### 7.2. Sự thật thứ hai: dự án lưu cả object `user`, trong đó có `role`

Response đăng nhập là `{ token, user }`, và `user` gồm cả `role` (0 = khách, 1 = admin) lẫn `active`.
Reducer `signin` nhét trọn gói vào state:

`yotea-fe/src/redux/authSlice.js:36-45`

```js
  reducers: {
    signin(state, { payload }) {
      state.isLogged = true;
      state.value = payload;
    },
    logout(state) {
      state.value = {};
      state.isLogged = false;
    },
  },
```

Mà `auth` nằm trong whitelist ⇒ **`role` bị ghi xuống localStorage**, nơi người dùng sửa được bằng hai
cú double-click trong DevTools.

### 7.3. Sửa `role` trong localStorage thì chiếm được quyền admin không?

**Câu trả lời có hai vế, phải nắm cả hai.**

**Vế 1 — KHÔNG chiếm được quyền thật.** Backend **không bao giờ** tin `role` do client gửi lên.

`yotea-be/src/middlewares/checkAuth.js:21-29`

```js
export const isAdmin = (req, res, next) => {
  if (!req.profile.role) {
    res.status(401).json({
      message: "Bạn không phải là Admin",
    });
  } else {
    next();
  }
};
```

`req.profile` từ đâu ra? Từ `router.param("userId", userById)`, và `userById` **truy vấn thẳng database**:

`yotea-be/src/controllers/user.js:288-299`

```js
export const userById = async (req, res, next, id) => {
    try {
        const user = await User.findById(id).exec();

        if (!user) {
            res.status(400).json({
                message: "Không tìm thấy User"
            });
        } else {
            req.profile = user;
            next();
        }
```

⇒ `req.profile.role` là `role` **trong MongoDB**, không liên quan gì tới localStorage.

| Kẻ xấu sửa gì trong localStorage | Backend phản ứng |
|---|---|
| `user.role` từ `0` → `1` | `isAdmin` đọc role **từ DB** vẫn là 0 → **401 "Bạn không phải là Admin"** |
| `user._id` → đổi thành `_id` của admin thật | `isAuth` (`checkAuth.js:9-19`) thấy id trên URL ≠ id giải mã từ token → **400 "Bạn không có quyền truy cập"** |
| Sửa chuỗi `token` | Chữ ký JWT sai → `requireSignin` chặn → **401 UnauthorizedError** |

**Vế 2 — NHƯNG qua mặt được `PrivateRouter` phía frontend.**

`yotea-fe/src/components/admin/PrivateRouter.js:5-22`

```jsx
const PrivateRouter = ({ children, page }) => {
  const isLogged = useSelector(selectStatusLoggin);
  const auth = useSelector(selectAuth);

  if (page === "admin") {
    if (!isLogged) {
      return <Navigate to="/login" />;
    } else if (!auth.user.role || !auth.user.active) {
      return <Navigate to="/" />;
    }
  } else {
    if (!isLogged || !auth.user.active) {
      return <Navigate to="/login" />;
    }
  }

  return children;
};
```

`auth.user.role` lấy từ Redux, mà Redux lấy từ localStorage lúc rehydrate. Sửa `\"role\":0` thành
`\"role\":1` rồi F5 ⇒ `PrivateRouter` cho qua ⇒ **toàn bộ giao diện `/admin` hiện ra**.

Bấm Lưu thì API trả 401, nhưng **giao diện đã lộ**. Và tệ hơn — nhiều API `GET` của dự án **không hề
kiểm tra quyền**:

`yotea-be/src/routes/users.js:8-13`

```js
router.post("/users/:userId", requireSignin, isAuth, isAdmin, create);
router.get("/users/:id", read);
router.get("/users", list);
router.put("/users/:id/:userId", requireSignin, isAuth, isAdmin, update);
router.put("/users/updateInfo/:myId/:userId", requireSignin, isAuth, update);
router.delete("/users/:id/:userId", requireSignin, isAuth, isAdmin, remove);
```

Dòng 10: `router.get("/users", list)` — **không một middleware nào**. Fake-admin vào `/admin/user` sẽ
**nhìn thấy toàn bộ danh sách người dùng thật** (email, số điện thoại, địa chỉ). Đây mới là lỗ hổng
nghiêm trọng, và nó nằm ở **backend**, không phải ở redux-persist.

> 🔒 **Ghi chú bảo mật — ba câu phải thuộc:**
> 1. **Frontend chỉ để "ẩn nút cho đỡ rối", không bao giờ là hàng rào bảo mật.** Ai cũng sửa được localStorage.
> 2. **Hàng rào thật luôn ở backend**, và phải kiểm tra quyền trên **dữ liệu trong DB**, không tin thứ client gửi lên.
> 3. Route `GET` cũng cần phân quyền nếu trả dữ liệu nhạy cảm — xem [Bài 33](33-ra-soat-bao-mat.md).

---

## 8. 🛠️ Tự tay làm

> Mục tiêu: bạn sẽ **tự chứng minh bằng thực nghiệm** vì sao không nên persist slice `topping`, tự viết
> một nút Đăng xuất và nhìn localStorage đổi theo thời gian thực, rồi tự tay khai thác lỗ hổng `role`
> để hiểu ranh giới frontend/backend.
>
> ⚠️ Bài 1 có **sửa tạm** `store.js` — nhớ **hoàn tác** ở bước cuối. Bài 3 chỉ sửa dữ liệu trong DevTools.

### Bài 1 — Thử persist `topping`, rồi tự tay gỡ ra

**Bước 1.1.** Mở `yotea-fe/src/redux/store.js`, sửa **tạm** dòng 16:

```js
// yotea-fe/src/redux/store.js — SỬA TẠM, sẽ hoàn tác ở bước 1.5
  whitelist: ["auth", "cart", "topping"],
```

**Bước 1.2.** Lưu file, đợi CRA reload. Vào trang `/topping` (route bạn thêm ở [Bài 15](15-routing-v6.md))
để thunk `getToppings` chạy và đổ dữ liệu vào `state.topping.value`.

**Bước 1.3.** DevTools → Application → Local Storage → `persist:root`. Bạn sẽ thấy xuất hiện thêm khoá
`"topping"` chứa **toàn bộ danh sách topping** nén thành chuỗi JSON. Chú ý độ dài giá trị tăng vọt.

**Bước 1.4 — thí nghiệm quyết định.** Không tắt trình duyệt, làm đúng thứ tự:

1. Mở Postman, đổi giá một topping: `PUT http://localhost:8080/api/topping/<id>/<adminId>` (API bạn tự
   viết ở [Bài 07](07-crud-category.md)–[Bài 12](12-phan-quyen-middleware.md)) — ví dụ `price` từ `5000` thành `9000`.
2. Quay lại trang `/topping`, bấm **F5**.
3. **Quan sát thật kỹ khoảnh khắc trang vừa hiện ra.**

Bạn sẽ thấy giá **5.000 ₫ (giá cũ)** nhấp nháy một nhịp rồi mới nhảy thành **9.000 ₫**. Vì `PersistGate`
bơm dữ liệu cũ vào trước, React render ngay, rồi mới tới lượt `getToppings` trả về ghi đè. Nếu mạng chậm
hoặc API lỗi, người dùng sẽ **đọc giá sai** — mà không hề biết.

**Bước 1.5 — hoàn tác.** Trả `store.js:16` về nguyên trạng:

```js
  whitelist: ["auth", "cart"],
```

Rồi **xoá rác còn sót**: DevTools → chuột phải `persist:root` → **Delete**, F5, đăng nhập lại.
(Bỏ tên khỏi whitelist **không** tự xoá dữ liệu cũ — nó nằm lì ở đó, xem mục 11.)

**Bước 1.6.** Viết ra câu trả lời:

> Không persist `topping` vì đó là **dữ liệu server** — nguồn sự thật ở MongoDB. Persist nó chỉ tiết
> kiệm ~200 ms lần render đầu nhưng đổi lấy: hiển thị **giá cũ sai lệch**, chiếm quota localStorage
> vốn chỉ ~5 MB, và tạo thêm một nguồn sự thật thứ hai phải đồng bộ. Chỉ persist `auth` (mất thì phải
> đăng nhập lại) và `cart` (mất thì mất luôn, server không giữ hộ).

### Bài 2 — Tự viết nút Đăng xuất và nhìn localStorage đổi

Tạo file **MỚI** — dự án chưa có file này:

```jsx
// yotea-fe/src/components/user/LogoutButton.js  ← file MỚI, bạn tự tạo
import { useDispatch } from "react-redux";
import { logout } from "../../redux/authSlice";
import { clearWishlist } from "../../redux/wishlistSlice";

const LogoutButton = () => {
  const dispatch = useDispatch();

  const handleLogout = () => {
    dispatch(logout());
    dispatch(clearWishlist());
    console.log(
      "persist:root sau khi logout:",
      localStorage.getItem("persist:root")
    );
  };

  return (
    <button
      onClick={handleLogout}
      className="px-4 py-2 rounded bg-red-500 text-white text-sm"
    >
      Đăng xuất
    </button>
  );
};

export default LogoutButton;
```

Nhúng tạm vào `ToppingPage` (trang bạn tự viết ở [Bài 16](16-layout-va-component.md)):

```jsx
// yotea-fe/src/pages/user/ToppingPage.js — thêm 2 dòng, bạn tự viết
import LogoutButton from "../../components/user/LogoutButton";
// ... rồi đặt <LogoutButton /> vào đâu đó trong JSX
```

**Quan sát:** mở sẵn bảng Local Storage trong DevTools rồi bấm nút. Ngay lập tức giá trị `"auth"` đổi
từ `"{\"isLogged\":true,\"value\":{...}}"` thành `"{\"isLogged\":false,\"value\":{}}"`.

**Ba điều rút ra:**

1. Bạn **không** viết dòng `localStorage.setItem` nào — `persistReducer` tự ghi sau mỗi action.
2. `logout` **không xoá** khoá `persist:root`, chỉ ghi đè bằng state rỗng. Muốn xoá sạch thật sự phải
   gọi `persistor.purge()`.
3. `dispatch(clearWishlist())` là bắt buộc vì `wishlist` **không** được persist nhưng vẫn nằm trong RAM.
   Không dọn thì tài khoản sau vẫn thấy wishlist của tài khoản trước. *(Dự án làm đúng ở
   `MyAccountLayout.js:11-14` nhưng **quên** ở `AdminLayout.js:22-24` — bug thật, xem [Bài 23](23-dang-ky-dang-nhap.md).)*

### Bài 3 — Tự tay khai thác lỗ hổng `role` (rồi tự tay thấy nó bị chặn)

> 🔒 Chỉ làm trên **máy của bạn**, với **dữ liệu của bạn**. Đây là bài học phòng thủ.

**Bước 3.1.** Đăng nhập bằng tài khoản **khách thường** (`role: 0`). Vào `http://localhost:3000/admin`
→ bị `PrivateRouter` đá về `/`. Đúng như thiết kế.

**Bước 3.2.** DevTools → Application → Local Storage → double-click vào **Value** của `persist:root`.
Tìm `\"role\":0` (trong phần `auth`), sửa thành `\"role\":1`. Enter, rồi **F5**.

**Bước 3.3.** Vào lại `http://localhost:3000/admin`. Lần này **vào được**: sidebar admin, dashboard,
danh sách người dùng… hiện đầy đủ. Đây chính là điều mục 7.3 đã nói: **`PrivateRouter` bị qua mặt.**

**Bước 3.4 — kiểm chứng backend.** Trong giao diện admin vừa lọt vào, thử **thêm một danh mục mới**.
Mở tab **Network** xem request đó:

```
POST http://localhost:8080/api/category/663f...   →  401 Unauthorized
{ "message": "Bạn không phải là Admin" }
```

**Bước 3.5.** Trả `role` về `0` và F5 (hoặc xoá `persist:root` rồi đăng nhập lại).

**Kết luận phải tự nói được thành lời:**

| Câu hỏi | Trả lời |
|---|---|
| Sửa `role` có chiếm được quyền admin không? | **Không.** `isAdmin` đọc role từ MongoDB qua `userById`, không đọc từ client. |
| Vậy sửa `role` được gì? | **Thấy được giao diện admin**, và đọc được mọi API `GET` không có middleware (ví dụ `GET /api/users`). |
| Sai lầm nằm ở đâu? | Ở chỗ coi `PrivateRouter` là bảo mật, và ở chỗ backend để `GET /api/users` công khai. |

---

## 9. ✅ Kiểm chứng kết quả

| # | Việc cần làm | Kết quả phải nhìn thấy |
|---|---|---|
| 1 | Thêm 2 món vào giỏ → F5 | Icon giỏ hàng ở header vẫn hiện số **2** |
| 2 | Đăng nhập → vào `/my-account` → F5 | **Không** bị đá về `/login` |
| 3 | Vào `/topping` → F5 | Danh sách **biến mất một nhịp** rồi mới hiện lại (phải gọi API) — chứng tỏ `topping` KHÔNG được persist |
| 4 | DevTools → Application → Local Storage | Đúng **1 khoá** `persist:root`, bên trong có `auth`, `cart`, `_persist`; **không có** `topping`, `product`, `news` |
| 5 | Console gõ `JSON.parse(localStorage.getItem("persist:root")).auth` | In ra một **chuỗi** (có `\"` escape), **không phải object** — bằng chứng serialize hai tầng |

Nếu mục 4 vẫn còn `topping` sau khi đã gỡ khỏi whitelist ⇒ đó là **dữ liệu cũ còn sót**, xoá
`persist:root` rồi F5.

---

## 10. 🔧 Khi cấu trúc state đổi làm dữ liệu cũ gây lỗi

Tình huống **chắc chắn sẽ gặp** khi làm đồ án. Hôm nay `cartSlice` có `initialState = { cart: [] }`.
Tuần sau bạn refactor thành `initialState = []` cho gọn. F5 → app **trắng màn hình**:

```
TypeError: state.cart.filter is not a function
```

Vì `persistReducer` bơm dữ liệu **cũ** (`{ cart: [...] }`) đè lên `initialState` **mới** (`[]`).
Reducer mới tưởng state là mảng, nhận được object → nổ.

**Cách 1 — xoá tay (khi đang học/dev).**

```js
// gõ trong Console của trình duyệt — nhanh nhất
localStorage.removeItem("persist:root");
```

Hoặc DevTools → Application → chuột phải `persist:root` → **Delete**, hoặc **Clear site data**.
Nhược điểm: **chỉ sửa được máy của bạn**; người dùng thật vẫn ôm dữ liệu cũ và vẫn thấy màn hình trắng.

**Cách 2 — `version` + `migrate` (cách chuyên nghiệp).** Đoạn dưới **bạn tự viết thêm, dự án chưa có**:

```js
// yotea-fe/src/redux/store.js — phiên bản CẢI TIẾN, bạn tự viết
import { createMigrate } from "redux-persist";

const migrations = {
  // key = version đích. Chạy khi dữ liệu cũ có version < 1
  1: (state) => ({
    ...state,
    cart: Array.isArray(state.cart) ? state.cart : state.cart.cart,
  }),
};

const persistConfig = {
  key: "root",
  storage,
  whitelist: ["auth", "cart"],
  version: 1,                                   // ← tăng mỗi lần đổi cấu trúc state
  migrate: createMigrate(migrations, { debug: false }),
};
```

Cơ chế: redux-persist lưu `_persist: { version: -1 }` trong localStorage (bạn đã thấy ở mục 6). Lúc
rehydrate nó so version đã lưu với `version` trong config:

- **Bằng nhau** → bơm thẳng.
- **Đã lưu < config** → chạy lần lượt các hàm migrate còn thiếu để **nâng cấp** dữ liệu cũ, rồi mới bơm.
- **Không có hàm migrate phù hợp** → **bỏ luôn** dữ liệu cũ, dùng `initialState`. An toàn, không nổ.

**Cách 3 — xoá bằng code khi đăng xuất triệt để** *(bạn tự viết thêm, dự án chưa có)*:

```js
import persistor from "./redux/store";

const handleLogoutTriet = async () => {
  dispatch(logout());
  await persistor.purge();   // xoá sạch persist:root khỏi localStorage
};
```

Hợp với máy công cộng — nhưng lưu ý `purge()` xoá **cả giỏ hàng**.

---

## 11. 🐞 Lỗi thường gặp

| Thông báo lỗi / Hiện tượng | Nguyên nhân | Cách sửa |
|---|---|---|
| F5 ở `/admin` hoặc `/my-account` là bị đá về `/login` | Thiếu `PersistGate`, hoặc `PrivateRouter` chạy trước khi rehydrate xong | Bọc `<App />` bằng `<PersistGate>` như `index.js:14` |
| `TypeError: Cannot read properties of null (reading 'auth')` | `isAuthenticate()` chạy khi localStorage chưa có `persist:root` | Thêm `try/catch` + fallback `{}` — xem [Bài 33](33-ra-soat-bao-mat.md) |
| Gỡ tên khỏi `whitelist` rồi mà DevTools vẫn thấy | redux-persist **không tự dọn** khoá cũ, chỉ ngừng ghi thêm | Xoá `persist:root` bằng tay, hoặc `persistor.purge()` |
| `state.xxx.map is not a function` sau khi refactor slice | Dữ liệu cũ lệch cấu trúc mới | Xoá localStorage, hoặc dùng `version` + `migrate` (mục 10) |
| `QuotaExceededError: Setting the value exceeded the quota` | Persist quá nhiều, vượt ~5 MB | Rút gọn `whitelist`, chỉ giữ dữ liệu client thật sự cần |
| API trả `Bearer undefined` ngay sau khi đăng nhập | Race: `isAuthenticate()` đọc đĩa **đồng bộ**, redux-persist ghi **bất đồng bộ** | Đọc token từ `store.getState()` hoặc dùng axios interceptor ([Bài 34](34-refactor-du-an.md)) |
| Persist rồi mà vẫn mất khi F5 | Đang mở **cửa sổ ẩn danh**, hoặc trình duyệt chặn storage cho site | Thử ở cửa sổ thường; kiểm tra thiết lập cookie/dữ liệu site |
| Cảnh báo `A non-serializable value was detected` | State chứa `Date`, `File`, `Map`… | Chỉ lưu dữ liệu thuần JSON *(dự án dùng `createStore` nên **không** hiện cảnh báo này — mất luôn lớp bảo vệ)* |

---

## 12. 📝 Bài tập

**Bài 1.** Không mở DevTools, hãy dự đoán: sau khi người dùng đăng nhập rồi thêm 1 món vào giỏ,
localStorage có **bao nhiêu khoá**, tên là gì, và bên trong khoá đó có **bao nhiêu** trường cấp 1?

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

**Đúng 1 khoá**, tên **`persist:root`** (vì `persistConfig.key = "root"` ở `store.js:14`).

Bên trong có **3 trường cấp 1**:

| Trường | Vì sao có | Giá trị là gì |
|---|---|---|
| `auth` | Nằm trong `whitelist` (`store.js:16`) | **Chuỗi** JSON của `{ isLogged, value }` |
| `cart` | Nằm trong `whitelist` | **Chuỗi** JSON của `{ cart: [...] }` — chú ý **lồng 2 tầng**, vì `cartSlice.js:3-5` khai `initialState = { cart: [] }` |
| `_persist` | redux-persist tự thêm | **Chuỗi** JSON của `{ version: -1, rehydrated: true }` |

**Không có** `product`, `news`, `wishlist`, `topping`, cũng không có 5 cache RTK Query.

Bẫy hay sai: nhiều bạn tưởng mỗi slice là một khoá riêng trong localStorage. Không phải — **cả gói chỉ
có một khoá**, `key` trong config quyết định tên khoá đó.

</details>

**Bài 2.** Đồng nghiệp thêm slice `themeSlice` (chế độ sáng/tối do người dùng chọn) và
`orderHistorySlice` (lịch sử đơn lấy từ `GET /api/order`). Nên đưa cái nào vào `whitelist`?
Giải thích, kèm `persistConfig` mới.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

- **`theme` → CÓ persist.** Đó là **lựa chọn của người dùng**, chỉ tồn tại ở client, server không biết.
  Không persist thì mỗi lần F5 lại nhảy về giao diện sáng. Dữ liệu lại cực nhẹ (một chuỗi `"dark"`).
- **`orderHistory` → KHÔNG persist.** **Dữ liệu server**, nguồn sự thật ở MongoDB. Persist vào thì: đơn
  vừa được admin đổi từ "Chờ xác nhận" sang "Đã giao" vẫn hiện trạng thái cũ; lịch sử đơn có thể rất
  nặng; và tệ nhất là **thông tin đơn hàng của tài khoản cũ còn nằm trên máy** sau khi đăng xuất
  (vì `logout` chỉ dọn `auth`).

```js
// yotea-fe/src/redux/store.js — bạn tự viết
const persistConfig = {
  key: "root",
  storage,
  whitelist: ["auth", "cart", "theme"],   // thêm theme, KHÔNG thêm orderHistory
};
```

Quy tắc: **persist cái mà mất đi thì không ai lấy lại được cho bạn**. Cái nào gọi một request là có
lại thì để nó chết cùng RAM.

</details>

**Bài 3.** Đọc lại `yotea-fe/src/utils/localStorage.js:1-4`: nếu đổi `persistConfig.key` từ `"root"`
thành `"yotea"` thì chuyện gì xảy ra? Liệt kê **đủ** các chỗ hỏng và cách sửa.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Khoá đổi thành **`persist:yotea`**. Hàm `isAuthenticate()` vẫn đi tìm `persist:root` → `getItem` trả
**`null`** → `JSON.parse(null)` ra `null` → `.auth` trên `null` →

```
TypeError: Cannot read properties of null (reading 'auth')
```

**Chỗ hỏng:** hàm này được import ở **12 file** trong `src/api/` (mọi hàm `add` / `update` / `remove`
đều dùng nó làm giá trị mặc định của tham số thứ hai — xem `api/category.js:18, 27, 36`).
⇒ Mọi thao tác cần token đều crash. Trang chỉ xem (`getAll`, `get`) vẫn chạy, nên bug này **rất dễ lọt
qua khâu test qua loa**.

**Sửa đúng — không phải sửa 13 file, mà là bỏ chuỗi hard-code đi:**

```js
// yotea-fe/src/utils/localStorage.js — bạn tự viết
import { store } from "../redux/store";

export const isAuthenticate = () => {
  return store.getState().auth.value || {};
};
```

Lấy thẳng từ Redux store — **một nguồn sự thật duy nhất**, không phụ thuộc `key`, không phụ thuộc cách
serialize, và không dính race condition ghi-đĩa-chậm ở mục 6.1.

**Bài học lớn hơn:** `"persist:root"` là một **chuỗi ma thuật (magic string)** bị hard-code ở nơi cách
xa chỗ định nghĩa nó. Mọi magic string đều là bom hẹn giờ.

</details>

**Bài 4.** *(nâng cao)* Yotea persist `cart` ở **client**. Nhưng người dùng thêm hàng trên điện thoại
rồi mở máy tính thì giỏ **trống**. Nêu 2 phương án khắc phục và đánh giá ưu/nhược.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

**Phương án A — giỏ hàng lưu ở server.** Tạo model `Cart` ở backend, mỗi lần `addCart` gọi API lưu luôn;
khi đăng nhập thì thunk kéo giỏ về.
✅ Đồng bộ mọi thiết bị; admin thống kê được "giỏ hàng bị bỏ quên" — chỉ số vàng trong thương mại điện tử.
❌ Mỗi thao tác `+/-` là một request; **khách chưa đăng nhập thì không có giỏ**; phải giải bài toán gộp
giỏ local vào giỏ server ngay sau khi đăng nhập.

**Phương án B — lai (hybrid), thường dùng nhất.** Giữ redux-persist cho khách vãng lai; khi đăng nhập
thì đẩy giỏ local lên server và **gộp** với giỏ đã có, từ đó đồng bộ hai chiều.
✅ Khách vãng lai vẫn mua được ngay (không ép đăng nhập — cực kỳ quan trọng cho tỷ lệ chốt đơn).
❌ Code phức tạp nhất: phải viết logic gộp (cùng `productId` + `ice` + `sugar` thì cộng dồn `quantity`,
giống hệt `cartSlice.js:11-24`) và xử lý xung đột.

**Ghi chú thực tế:** Yotea đang ở mức đơn giản nhất (chỉ client) — hoàn toàn chấp nhận được cho đồ án,
miễn là khi bảo vệ bạn **nói rõ được** đây là lựa chọn có ý thức và biết hướng nâng cấp.

</details>

---

## 📌 Tóm tắt

- Redux store nằm trong **RAM** → F5 là mất sạch. `redux-persist` ghi state xuống **localStorage** và
  bơm ngược lại lúc khởi động (**rehydrate**).
- Ba chỗ lưu: **localStorage** (lâu dài, JS đọc được), **sessionStorage** (chết theo tab), **cookie**
  (bé, tự gửi kèm request, đặt được `httpOnly` để JS **không** đọc được).
- Bộ ba API: `persistConfig` (lưu gì, ở đâu, tên gì) → `persistReducer` (**ghi**) → `persistStore`
  (**đọc** + tạo persistor). Thứ tự không được đảo.
- `key: "root"` (`store.js:14`) → khoá localStorage là **`persist:root`**.
  `whitelist: ["auth", "cart"]` (`store.js:16`) → chỉ 2 slice này xuống đĩa.
- **`whitelist` an toàn hơn `blacklist`**: slice mới thêm mặc định **không** bị lưu.
- **Chỉ persist dữ liệu client** (giỏ hàng, phiên đăng nhập, tuỳ chọn giao diện). **Không persist dữ
  liệu server** — nên `topping` (bài 19–20) **không** vào whitelist.
- `PersistGate` (`index.js:14`) chặn render tới khi đọc xong storage — thiếu nó thì F5 ở trang cần
  đăng nhập sẽ bị đá về `/login`; `loading` là thứ hiển thị trong lúc chờ.
- redux-persist serialize **hai tầng** (cả gói một lần, từng slice một lần) — đó là lý do
  `isAuthenticate()` (`utils/localStorage.js:1-4`) phải `JSON.parse` **hai lần**.
- 🔒 JWT trong localStorage thì **mọi script trên trang đều đọc được** (rủi ro XSS); cookie `httpOnly`
  an toàn hơn cho token.
- 🔒 Sửa `role` trong localStorage **qua mặt được `PrivateRouter`** (thấy giao diện admin) nhưng
  **không chiếm được quyền thật** — `isAdmin` (`checkAuth.js:21-29`) đọc role **từ database**.
  **Frontend không bao giờ là hàng rào bảo mật.**
- Đổi cấu trúc state → xoá `persist:root` khi đang dev; dùng `version` + `migrate` khi đã có người dùng thật.

**Từ khoá tra cứu thêm:** `redux-persist whitelist blacklist`, `PersistGate rehydrate`,
`persistStore purge`, `redux-persist migrate version`, `localStorage vs cookie httpOnly`,
`JWT storage XSS`, `client-side authorization bypass`

➡️ **Bài tiếp theo:** [22 — RTK Query: `createApi`, cache và `invalidatesTags`](22-rtk-query.md) —
bạn đã viết `src/api/topping.js` + `toppingSlice` + thunk + extraReducers, gần trăm dòng chỉ để lấy
một danh sách. Bài sau làm lại **toàn bộ** việc đó trong khoảng 15 dòng, kèm cache và tự động refetch —
rồi ta so sánh trực diện hai cách.
