# Bài 20 — `createAsyncThunk` và `extraReducers`

> **Phần 3 · Quản lý state với Redux** — Thời lượng ước tính: **~75 phút**
> ⬅️ Bài trước: [19 — Redux Toolkit: slice, action, reducer, selector](19-redux-toolkit-co-ban.md) · Bài sau: [21 — redux-persist: giữ giỏ hàng và phiên đăng nhập](21-redux-persist.md) ➡️
> 🏠 [Mục lục](README.md)

---

## 🎯 Sau bài này bạn sẽ

- Hiểu vì sao **không được** gọi API trong reducer, và middleware giải quyết chuyện đó thế nào.
- Giải thích được `redux-thunk` bằng lời: *dispatch một **hàm** thay vì một **object***.
- Đọc vanh vách 3 action `createAsyncThunk` tự sinh: `pending` / `fulfilled` / `rejected`.
- Phân biệt rạch ròi `reducers` và `extraReducers`.
- Dispatch được thunk trong component và **chờ kết quả thật** bằng `.unwrap()`.
- Chỉ ra được lỗ hổng lớn nhất của Redux trong Yotea: **không slice nào xử lý `pending`/`rejected`**.
- Tự viết `getToppings` đủ ba trạng thái, hiện spinner và báo lỗi trên `ToppingPage`.

## 📋 Cần chuẩn bị

- Đã hoàn thành [Bài 19](19-redux-toolkit-co-ban.md) — bạn đã có `src/redux/toppingSlice.js` với reducer đồng bộ, selector, và đã đăng ký vào `rootReducer`.
- Đã có `src/api/topping.js` từ [Bài 18](18-tang-api-axios.md) với `getAll / get / add / update / remove`.
- Backend chạy ở cổng **8080** với API Topping bạn tự xây ở [Bài 04](04-express-va-appjs.md)→[Bài 13](13-swagger-tai-lieu-api.md).
- Nắm `async/await` và Promise ở [Bài 03](03-kien-thuc-nen.md).

> Ở bài trước bạn đã dựng `toppingSlice` **đồng bộ** — dữ liệu phải tự tay nhét vào bằng
> `dispatch(setToppings([...]))`. Bài này ta làm tiếp phần còn thiếu: **để chính Redux đi gọi API**,
> tự bật cờ đang tải, tự bắt lỗi và tự đổ dữ liệu vào state.

---

## 1. Vấn đề: reducer không được phép gọi API

Nhắc lại luật sắt từ [Bài 19](19-redux-toolkit-co-ban.md): reducer phải là **hàm thuần** và **đồng bộ**.

"Thuần" nghĩa là cùng `(state, action)` thì **luôn** cho ra cùng kết quả, không đụng thế giới bên
ngoài — không gọi API, không `Math.random()`, không `new Date()`. Vì sao khắt khe vậy? Vì Redux
DevTools cho phép **tua ngược thời gian**, chạy lại y hệt chuỗi action cũ để tái dựng state. Nếu
reducer lén gọi API thì mỗi lần tua lại là một lần bắn request thật — hỗn loạn.

"Đồng bộ" nghĩa là reducer **phải trả về state ngay**, không có quyền nói "chờ tôi 300ms".

```js
// ❌ SAI HOÀN TOÀN — chỉ để minh hoạ cái sai, ĐỪNG chép vào dự án
reducers: {
  getToppings(state) {
    const { data } = await getAll();   // 💥 await trong reducer: không hợp lệ
    state.value = data;                // 💥 state đã bị trả về từ lâu rồi
  },
}
```

Vậy code gọi API đặt ở đâu?

| Đặt ở đâu | Ưu | Nhược |
|---|---|---|
| Trong component (`useEffect` + axios) | Dễ hiểu | Logic gọi API bị **lặp** ở mọi component cần dữ liệu |
| Trong reducer | — | **Bị cấm**, như vừa phân tích |
| Trong **middleware** | Viết một lần, dùng mọi nơi; component chỉ `dispatch` | Phải học thêm khái niệm middleware |

Redux chọn đường thứ ba. Middleware làm việc đó tên là **thunk**.

---

## 2. `redux-thunk` là gì — giải thích bằng lời

> 📖 **Thuật ngữ:** *thunk* — một đoạn việc được **gói lại trong một hàm để hoãn lại, chờ chạy sau**.
> Bạn không đưa **kết quả**, bạn đưa **cách lấy kết quả**.

Bình thường `dispatch` chỉ nhận **object**: `dispatch({ type: "cart/addCart", payload: {...} })`.
Middleware `redux-thunk` cắm thêm đúng một khả năng:

> **Nếu thứ bạn dispatch là một HÀM (không phải object), thunk chặn nó lại, không cho tới reducer,
> mà tự gọi hàm đó và đưa cho nó hai công cụ: `dispatch` và `getState`.**

Ví von: reducer là **thủ kho**, chỉ nhận **phiếu đã điền sẵn số liệu** (object). Thunk là **nhân viên
chạy việc**: bạn đưa **tờ hướng dẫn** ("ra chợ mua 5 ký trân châu, xong thì điền phiếu đưa thủ kho").
Anh ta đi bao lâu cũng được, xong việc mới quay về đưa phiếu. Thủ kho không bao giờ tự đi chợ.

```
 dispatch(x) ──▶ [ redux-thunk ]
                      │
            x là HÀM? ├── CÓ ──▶ gọi x(dispatch, getState) ──▶ x tự dispatch các object khác
                      │
                      └── KHÔNG ──▶ reducer ──▶ state mới ──▶ React vẽ lại
```

### Thunk được cắm vào store ở đâu trong Yotea?

`yotea-fe/src/redux/store.js:21-28`

```js
const middleware = [
  thunk,
  productApi.middleware,
  sliderApi.middleware,
  cateProductApi.middleware,
  userApi.middleware,
  newsApi.middleware,
];
```

Import ở `yotea-fe/src/redux/store.js:5`:

```js
import { thunk } from "redux-thunk";
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 5 | `import { thunk } from "redux-thunk"` | Có ngoặc nhọn vì đây là named export (đúng chuẩn `redux-thunk` v3) |
| 22 | `thunk,` | Phần tử **đầu tiên** trong chuỗi — mọi action đi qua nó trước |
| 23-27 | `productApi.middleware`, … | 5 middleware của RTK Query, nói ở [Bài 22](22-rtk-query.md) |

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** phải **tự cài và tự ghép** `redux-thunk` là hệ quả của việc
> dùng `createStore` cũ. Nếu dùng `configureStore` của Redux Toolkit thì **thunk đã có sẵn**, không
> cần package `redux-thunk` (`yotea-fe/package.json:28`) và không cần dòng 22. Xem [Bài 34](34-refactor-du-an.md).

---

## 3. `createAsyncThunk` — máy sinh 3 action tự động

Viết thunk tay khá dài: phải tự dispatch "bắt đầu tải", tự `try/catch` để dispatch "lỗi", tự dispatch
"xong". Redux Toolkit đóng gói khuôn mẫu đó thành `createAsyncThunk`. Bạn chỉ cần **2 thứ**:

```js
export const getToppings = createAsyncThunk(
  "topping/getToppings",   // ① tiền tố cho tên action
  async (thamSo) => {      // ② hàm bất đồng bộ, trả về DỮ LIỆU
    const { data } = await getAll();
    return data;
  }
);
```

Đổi lại, bạn được **ba action type**:

| Action type | Bắn ra khi nào | Ý nghĩa với giao diện | Đọc dữ liệu ở đâu |
|---|---|---|---|
| `topping/getToppings/pending` | **Ngay lập tức**, trước khi request bay đi | Hiện **spinner / skeleton**, khoá nút | `action.meta.arg` = tham số truyền vào |
| `topping/getToppings/fulfilled` | Hàm async **return** thành công | Tắt spinner, **hiện dữ liệu** | `action.payload` = giá trị `return` |
| `topping/getToppings/rejected` | Hàm async **ném lỗi** (API 4xx/5xx, mất mạng) | Tắt spinner, **hiện lỗi** + nút "Thử lại" | `action.error.message` |

Ba action này **luôn** được bắn ra dù bạn có xử lý hay không. Muốn "nghe" cái nào thì khai báo cái
đó trong `extraReducers`; không khai báo thì action vẫn đi qua reducer nhưng không ai phản ứng.

```
 t=0ms                          t=250ms
   ├── pending ──────────────────┼── fulfilled → state.value = [...]
   │   loading = true            └── rejected  → state.error = "..."
   │                                 loading = false ở CẢ HAI nhánh
```

> 💡 Tên action type là `"<tiền tố>/<pending|fulfilled|rejected>"`. Đặt tiền tố theo mẫu
> `"<tên slice>/<tên thunk>"` — cả 10 slice có thunk của Yotea đều theo quy ước này.

---

## 4. Soi code thật: `productSlice.js`

File mẫu mực nhất để học, vì chứa **đủ 4 thao tác CRUD** dưới dạng thunk. Trích **nguyên văn toàn bộ**:

`yotea-fe/src/redux/productSlice.js:1-72`

```js
import { createAsyncThunk, createSlice } from "@reduxjs/toolkit";
import { add, getAll, remove, update } from "../api/product";

const initialState = {
  value: [],
  totalProduct: 0,
};

export const getProducts = createAsyncThunk(
  "product/getProducts",
  async ({ start, limit }) => {
    const { data } = await getAll();
    const totalProduct = data.length;

    const { data: productsData } = await getAll(start, limit);

    return { totalProduct, productsData };
  }
);

export const addProduct = createAsyncThunk(
  "product/addProduct",
  async (dataProduct) => {
    const { data } = await add(dataProduct);
    return data;
  }
);

export const updateProduct = createAsyncThunk(
  "product/updateProduct",
  async (dataProduct) => {
    const { data } = await update(dataProduct);
    return data;
  }
);

export const deleteProduct = createAsyncThunk(
  "product/deleteProduct",
  async (id) => {
    return remove(id).then(() => id);
  }
);

const productSlice = createSlice({
  name: "product",
  initialState,
  reducers: {},
  extraReducers: (builder) => {
    builder.addCase(getProducts.fulfilled, (state, { payload }) => {
      state.value = payload.productsData;
      state.totalProduct = payload.totalProduct;
    });

    builder.addCase(addProduct.fulfilled, (state, { payload }) => {
      state.value = [...state.value, payload];
    });

    builder.addCase(updateProduct.fulfilled, (state, { payload }) => {
      state.value = state.value.map((item) =>
        item._id === payload._id ? payload : item
      );
    });

    builder.addCase(deleteProduct.fulfilled, (state, { payload }) => {
      state.value = state.value.filter((item) => item._id !== payload);
    });
  },
});

export const selectProducts = (state) => state.product.value;
export const selectTotalProduct = (state) => state.product.totalProduct;
export default productSlice.reducer;
```

### 4.1. Mổ từng thunk

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 2 | `import { add, getAll, remove, update } from "../api/product"` | Slice **không tự gọi axios**, nó gọi qua tầng API của [Bài 18](18-tang-api-axios.md) |
| 4-7 | `initialState` | `value` = sản phẩm của **trang hiện tại**; `totalProduct` = tổng số trong DB, để tính số nút phân trang |
| 9-19 | `getProducts` | Đọc danh sách — mổ riêng ở 4.2 |
| 21-27 | `addProduct` | Gọi `add()`, `return data` = sản phẩm vừa tạo (đã có `_id` do MongoDB sinh) |
| 29-35 | `updateProduct` | Gọi `update()`, trả về sản phẩm **sau khi sửa** |
| 37-42 | `deleteProduct` | `remove(id)` không trả gì hữu ích, nên `.then(() => id)` để **trả lại chính `id`** — reducer cần `id` mới biết xoá phần tử nào |

Ba thunk `add`/`update`/`delete` là **khuôn mẫu** lặp y hệt ở 8 slice khác (`userSlice`, `newsSlice`,
`sliderSlice`, `storeSlice`, `cateNewsSlice`, `categoryProductSlice`…). Nắm 3 cái này là đọc được cả bộ.

Chú ý dòng 40: nếu viết `return remove(id)` thôi thì `payload` sẽ là **cả object response của axios**,
reducer dòng 65 so sánh `item._id !== payload` luôn đúng ⇒ **không xoá được gì**. Chi tiết nhỏ mà chết người.

### 4.2. Vì sao `getProducts` gọi `getAll()` **hai lần**?

Muốn hiểu, phải nhìn tầng API — `yotea-fe/src/api/product.js:8-17`:

```js
export const getAll = (
  start = 0,
  limit = 0,
  sort = "createdAt",
  order = "desc"
) => {
  let url = `/${DB_NAME}/?_expand=categoryId&_sort=${sort}&_order=${order}`;
  if (limit) url += `&_start=${start}&_limit=${limit}`;
  return instance.get(url);
};
```

Mấu chốt ở dòng 15: **chỉ khi `limit` khác 0 thì URL mới có `_start`/`_limit`**. Do đó:

| Lời gọi | URL bắn đi | Trả về |
|---|---|---|
| `getAll()` — `productSlice.js:12` | `/products/?_expand=categoryId&_sort=createdAt&_order=desc` | **TOÀN BỘ** sản phẩm |
| `getAll(start, limit)` — `productSlice.js:15` | `…&_start=0&_limit=8` | Đúng **một trang** |

Ý đồ của tác giả: lần 1 tải **hết** rồi lấy `data.length` để biết **tổng số** (vẽ thanh phân trang);
lần 2 tải đúng 8 món để hiển thị. Nghĩa là để hiện 8 sản phẩm, trình duyệt tải về **137 + 8 = 145** bản ghi.

> ⚠️ **Chỗ này dự án làm chưa chuẩn — một trong những chỗ lãng phí nhất của cả frontend:**
>
> - **Tốn băng thông kinh khủng.** Mỗi sản phẩm kèm `description` dài và object `categoryId` đã
>   `_expand`. 500 sản phẩm ⇒ vài MB JSON tải về **chỉ để lấy `.length`** rồi vứt đi.
> - **Chậm gấp đôi.** Hai `await` chạy **nối tiếp**. Mỗi request 200ms ⇒ tổng 400ms.
> - **Có thể sai số liệu.** Giữa hai request, nếu admin máy khác vừa thêm/xoá thì `totalProduct` và
>   `productsData` **lệch nhau**.
> - **Không dùng được với dữ liệu lớn.** 100.000 bản ghi ⇒ treo trình duyệt.
>
> **Cách chuẩn:** để **backend** trả tổng số, frontend chỉ tải một trang:
>
> ```
> // Kiểu 1 — trả trong HEADER (json-server, GitHub API dùng kiểu này)
> GET /api/products?_start=0&_limit=8
> ← 200 OK   X-Total-Count: 137
>   [ …8 sản phẩm… ]
>
> // Kiểu 2 — trả trong BODY (dễ đọc hơn, khuyên dùng)
> ← 200 OK
>   { "data": [ …8 sản phẩm… ], "total": 137, "page": 1, "totalPages": 18 }
> ```
>
> Backend Mongoose làm việc này bằng **một dòng**: `const total = await Product.countDocuments(filter);`
> — MongoDB đếm ngay trong index, **không tải bản ghi nào lên RAM**. Ta dựng lại kiểu này ở
> [Bài 25](25-danh-sach-san-pham.md) và [Bài 34](34-refactor-du-an.md).
>
> Nếu buộc phải giữ 2 request, ít nhất cho chúng chạy **song song**:
> `const [all, page] = await Promise.all([getAll(), getAll(start, limit)]);` — cắt một nửa thời gian chờ.

Khuôn "gọi 2 lần" này bị copy nguyên xi sang các slice khác, ví dụ `yotea-fe/src/redux/userSlice.js:9-19`:

```js
export const getUsers = createAsyncThunk(
  "user/getUsers",
  async ({ start, limit }) => {
    const { data } = await getAll();
    const totalUser = data.length;

    const { data: dataUser } = await getAll(start, limit);

    return { totalUser, dataUser };
  }
);
```

Giống hệt, chỉ đổi tên biến. `newsSlice`, `storeSlice`, `contactSlice` cũng vậy.

### 4.3. Mổ từng `builder.addCase`

**Đọc từng dòng** (`yotea-fe/src/redux/productSlice.js:48-67`):

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 48 | `extraReducers: (builder) => {` | Nhận object `builder` để "đăng ký" các case |
| 49 | `builder.addCase(getProducts.fulfilled, …)` | `getProducts.fulfilled` **chính là action creator** RTK sinh ra; RTK tự lấy `.type` = `"product/getProducts/fulfilled"` |
| 50-51 | `state.value = payload.productsData` | `payload` là **đúng object bạn `return`** ở dòng 17, tách ra hai chỗ trong state |
| 54-56 | `state.value = [...state.value, payload]` | **Thêm** vào cuối mảng — không cần gọi lại API danh sách |
| 58-62 | `.map(… ? payload : item)` | **Thay thế** phần tử có `_id` trùng, giữ nguyên thứ tự |
| 64-66 | `.filter((item) => item._id !== payload)` | **Loại bỏ** phần tử vừa xoá; `payload` là **chuỗi id** nhờ mẹo `.then(() => id)` ở dòng 40 |

> 💡 `(state, { payload })` là destructuring tham số thứ hai. Viết đủ là `(state, action) => { … action.payload … }`.

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** cả `productSlice` là **code chết**. Không component nào import
> `selectProducts` hay `getProducts` — trang quản trị đã chuyển sang RTK Query (`useGetProductsQuery`
> ở `yotea-fe/src/components/admin/ListProduct.js:9`). File 72 dòng vẫn được mount vào store, tốn RAM
> và gây hoang mang. So sánh hai cách ở [Bài 22](22-rtk-query.md).

---

## 5. `reducers` vs `extraReducers`

| | `reducers` | `extraReducers` |
|---|---|---|
| Cú pháp | Object `{ tênAction(state, action) {} }` | Hàm nhận `builder`, gọi `builder.addCase(…)` |
| Có **sinh** action creator? | **CÓ** → `slice.actions.tênAction` | **KHÔNG** — chỉ *lắng nghe* |
| Action của ai? | **Của chính slice này** | Của **thunk**, hoặc của **slice khác** |
| Dùng cho | Việc **đồng bộ**: thêm giỏ hàng, bật/tắt drawer, logout | Kết quả **bất đồng bộ** |
| Ví dụ Yotea | `cartSlice`, `authSlice` (`signin`, `logout`) | Mọi slice có thunk |

> **Cách nhớ: `reducers` = action **do tôi đẻ ra**; `extraReducers` = action **của người khác** mà tôi muốn nghe.**

`authSlice` là ví dụ duy nhất dùng **cả hai** — `yotea-fe/src/redux/authSlice.js:36-54`:

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
  extraReducers: (builder) => {
    builder.addCase(updateAuth.fulfilled, (state, { payload }) => {
      state.value.user = payload;
    });

    builder.addCase(updateMyAccount.fulfilled, (state, { payload }) => {
      state.value.user = payload;
    });
  },
```

`signin`/`logout` là hành động tức thì → `reducers`. `updateAuth`/`updateMyAccount` là thunk gọi API
→ chỉ nghe được qua `extraReducers`. Còn 8 slice CRUD kia đều viết `reducers: {}` **rỗng**
(`productSlice.js:47`) vì mọi thay đổi đều đến từ server.

> 💡 **Công dụng khác mà dự án chưa dùng:** nghe action của **slice khác**. Khi `logout` chạy ta muốn
> `wishlistSlice` tự dọn sạch. Thay vì bắt component nhớ dispatch hai lần (`MyAccountLayout.js:12-13`
> làm vậy, còn `AdminLayout.js:23` **quên**), chỉ cần trong `wishlistSlice` viết:
> ```js
> // đoạn này bạn tự viết thêm, dự án chưa có
> builder.addCase(logout, (state) => { state.value = []; });
> ```

---

## 6. Dispatch một thunk trong component

### 6.1. Cách cơ bản

`yotea-fe/src/components/admin/UserList.js:8-15`

```js
const UserList = ({ start, limit }) => {
  const dispatch = useDispatch();

  const users = useSelector(selectUsers);

  useEffect(() => {
    dispatch(getUsers({ start, limit }));
  }, [start]);
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 9 | `useDispatch()` | Lấy hàm `dispatch` của store |
| 11 | `useSelector(selectUsers)` | **Đọc** state; state đổi thì component tự render lại |
| 13-15 | `useEffect(…, [start])` | Gọi thunk **sau khi** mount, gọi lại mỗi khi đổi trang |
| 14 | `dispatch(getUsers({ start, limit }))` | `getUsers({…})` **chưa gọi API** — nó chỉ *tạo ra* thunk; chính `dispatch` mới kích hoạt |

Mẫu này lặp ở 6 component: `UserList`, `NewsList`, `ContactList`, `StoreList`, `ListSlider`, `CateNewsList`.

> ⚠️ Dependency array `[start]` **thiếu `limit` và `dispatch`** → ESLint cảnh báo
> `react-hooks/exhaustive-deps`. Ở đây `limit` là hằng nên may mắn không sinh bug.

### 6.2. `.unwrap()` — cách **chờ kết quả thật**

> **`dispatch(thunk)` trả về một Promise gần như KHÔNG BAO GIỜ reject.**

`createAsyncThunk` **nuốt** lỗi: hàm async ném lỗi thì nó chuyển thành action `rejected`, còn Promise
vẫn **resolve** (resolve với một action object có `type` kết thúc bằng `/rejected`). Hệ quả:
`try/catch` quanh `dispatch(...)` là **vô dụng**. Đúng lỗi đó trong dự án —
`yotea-fe/src/pages/admin/profile/AdminUpdateInfoPage.js:43-54`:

```js
  const onSubmit = async (dataInput) => {
    try {
      if (typeof dataInput.avatar === "object" && dataInput.avatar.length) {
        dataInput.avatar = await uploadFile(dataInput.avatar[0]);
      }

      dispatch(updateAuth(dataInput));
      toast.success("Cập nhật tài khoản thành công");
    } catch (error) {
      toast.error("Cập nhật tài khoản không thành công");
    }
  };
```

| Vấn đề | Giải thích |
|---|---|
| Không `await` trước `dispatch` | Toast "thành công" hiện **ngay**, request còn đang bay |
| Không `.unwrap()` | Token hết hạn, server trả 400 → `catch` **không bao giờ chạy** |
| Kết quả | Người dùng **luôn** thấy "Cập nhật thành công", kể cả khi thất bại 100% |

`yotea-fe/src/pages/user/my-account/UpdateInfoPage.js:61-72` mắc y hệt với `updateMyAccount`.

**`.unwrap()` là thuốc chữa:** `fulfilled` → trả thẳng `payload`; `rejected` → **ném lỗi thật** để
`try/catch` bắt được. Dự án **có** dùng `.unwrap()` ở đúng 3 chỗ, nhưng cả 3 đều là **mutation của
RTK Query**, chưa từng dùng với `createAsyncThunk`. Ví dụ `yotea-fe/src/components/admin/ListProduct.js:22-28`:

```js
      if (result.isConfirmed) {
        deleteProduct(id)
          .unwrap()
          .then(() => {
            Swal.fire("Thành công!", "Đã xóa thành công.", "success");
          });
      }
```

(Hai chỗ còn lại: `AddProductPage.js:60` và `EditProductPage.js:50`.) Áp đúng công thức đó cho
`createAsyncThunk` — **đoạn dưới bạn tự viết thêm**, đây là bản sửa cho `AdminUpdateInfoPage`:

```js
// đoạn này bạn tự viết thêm, dự án chưa có
const onSubmit = async (dataInput) => {
  try {
    if (typeof dataInput.avatar === "object" && dataInput.avatar.length) {
      dataInput.avatar = await uploadFile(dataInput.avatar[0]);
    }

    await dispatch(updateAuth(dataInput)).unwrap();   // ← await + unwrap
    toast.success("Cập nhật tài khoản thành công");
  } catch (error) {
    toast.error("Cập nhật tài khoản không thành công");
  }
};
```

> 💡 **Quy tắc bỏ túi:** *cần biết thành/bại để báo cho người dùng* ⇒ **bắt buộc** `await … .unwrap()`.
> *Chỉ nạp dữ liệu nền* ⇒ `dispatch(…)` trần là đủ.

---

## 7. Soi `authSlice`: mẫu "cập nhật xong thì tải lại"

`yotea-fe/src/redux/authSlice.js:9-31`

```js
export const updateAuth = createAsyncThunk(
  "auth/updateAuth",
  async (dataAuth) => {
    return update(dataAuth).then(async () => {
      const {
        data: { password, ...rest },
      } = await get(dataAuth._id);
      return rest;
    });
  }
);

export const updateMyAccount = createAsyncThunk(
  "auth/updateMyAccount",
  async (dataAuth) => {
    return updateMyInfo(dataAuth).then(async () => {
      const {
        data: { password, ...rest },
      } = await get(dataAuth._id);
      return rest;
    });
  }
);
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 11 | `async (dataAuth) =>` | `dataAuth` là object user đầy đủ, **bắt buộc có `_id`** |
| 12 | `update(dataAuth)` | `yotea-fe/src/api/user.js:36-43` → `PUT /api/users/{userData._id}/{user._id}` kèm `Bearer token`; route backend có `isAdmin` |
| 14-15 | `const { data: { password, ...rest } } = await get(dataAuth._id)` | Gọi lại `GET /api/users/{id}` lấy **bản mới nhất từ server**, đồng thời **bóc bỏ `password`** |
| 16 | `return rest` | Chính là `payload` mà `extraReducers` dòng 48 nhận |
| 24 | `updateMyInfo(dataAuth)` | Khác biệt **duy nhất**: `yotea-fe/src/api/user.js:45-52` → `PUT /api/users/updateInfo/{id}/{userId}`, route backend **không có `isAdmin`** — dành cho user thường tự sửa hồ sơ |

**Vì sao gọi API hai lần (sửa xong lại đọc lại)?** Một là **lấy bản chuẩn từ server** (backend có thể
tự sinh `updatedAt`, `slug`… mà form không có). Hai là **đề phòng backend không trả về document mới**.

**Vì sao phải bóc `password`?** Vì backend `read` user **vẫn trả về chuỗi hash mật khẩu**. Nếu đưa
nguyên vào Redux, mà `auth` lại **được redux-persist ghi xuống localStorage** ([Bài 21](21-redux-persist.md)),
hash mật khẩu sẽ nằm chình ình trong trình duyệt. Đây là **miếng vá tạm ở frontend cho lỗi của
backend** — chỗ sửa đúng là backend `.select("-password")`. Xem [Bài 33](33-ra-soat-bao-mat.md).

> ⚠️ **Cách viết ở đây rối không cần thiết:** trộn `.then()` với `await` trong cùng một biểu thức.
> Viết lại bằng `async/await` thuần sẽ phẳng và dễ đọc hơn hẳn:
>
> ```js
> // đoạn này bạn tự viết thêm — bản gọn của updateAuth
> export const updateAuth = createAsyncThunk("auth/updateAuth", async (dataAuth) => {
>   await update(dataAuth);
>
>   const {
>     data: { password, ...rest },
>   } = await get(dataAuth._id);
>
>   return rest;
> });
> ```
>
> Ngắn hơn, không lồng callback, dễ đặt breakpoint. (Biến `password` khai báo mà không dùng sẽ bị
> ESLint kêu `no-unused-vars` — cái giá quen thuộc của mẹo omit-key này.)

---

## 8. ⚠️ Lỗ hổng lớn nhất: không slice nào xử lý `pending` và `rejected`

Tự kiểm chứng. Mở terminal **tại thư mục gốc repo**:

```bash
grep -rn "addCase" yotea-fe/src/redux
```

Kết quả: **35 dòng `addCase`**, và **cả 35 đều kết thúc bằng `.fulfilled`**. Không một `.pending`,
không một `.rejected`. Trên Windows/PowerShell:

```powershell
Select-String -Path yotea-fe\src\redux\*.js -Pattern "addCase" | Select-String -Pattern "pending|rejected"
```

→ trả về **rỗng**.

| Tình huống | Điều thực sự xảy ra | Người dùng thấy gì |
|---|---|---|
| Đang tải danh sách | 3 action bắn ra, chỉ `fulfilled` được nghe | **Màn hình trắng**, không biết đang tải hay không có dữ liệu |
| API trả 500 | `rejected` bắn ra, **không ai nghe** | Bảng rỗng **vĩnh viễn**, không một thông báo |
| Token hết hạn (3h) | `updateAuth` rejected, state không đổi | Toast **"Cập nhật thành công"** (mục 6.2) — thông tin sai trắng trợn |
| Mất mạng | rejected im lặng | Người dùng bấm lại nhiều lần, tưởng máy lag |

Vì slice không cung cấp cờ `loading`, mỗi trang phải **tự chế** bằng `useState`. Có **6 file** làm
vậy: `CheckoutPage.js:35`, `UpdateInfoPage.js:28`, `CartDetailPage.js:12`, `EditUserPage.js:46`,
`AddUserPage.js:48`, `AdminUpdateInfoPage.js:30`. Và cách dùng thì rất tuỳ hứng —
`yotea-fe/src/pages/user/my-account/UpdateInfoPage.js:36-48`:

```js
  useEffect(() => {
    updateTitle("Cập nhật tài khoản");
    const start = async () => {
      setLoading(true);

      setPreview(user.avatar);
      reset({
        ...user,
      });
      setLoading(false);
    };
    start();
  }, []);
```

`setLoading(true)` rồi `setLoading(false)` ngay, ở giữa **không có `await` nào** — spinner bật/tắt
trong cùng một tick, tức **không bao giờ nhìn thấy**. Còn `AdminUpdateInfoPage.js:56-59` thì bật
`setLoading(true)` mà **không bao giờ tắt**.

**Mẫu chuẩn xử lý đủ ba trạng thái** (đoạn dưới **bạn tự viết thêm**, dự án chưa có):

```js
// đoạn này bạn tự viết thêm, dự án chưa có
const initialState = { value: [], loading: false, error: null };

extraReducers: (builder) => {
  builder
    .addCase(getX.pending, (state) => {
      state.loading = true;
      state.error = null;          // xoá lỗi cũ khi thử lại
    })
    .addCase(getX.fulfilled, (state, { payload }) => {
      state.loading = false;
      state.value = payload;
    })
    .addCase(getX.rejected, (state, action) => {
      state.loading = false;
      state.error = action.error.message || "Không tải được dữ liệu";
    });
},
```

Nối chuỗi được vì mỗi `addCase` **trả về chính `builder`**. Nhớ hai điều: (1) `loading = false` phải
có ở **cả** `fulfilled` **và** `rejected` — quên một nhánh là spinner quay mãi; (2) `pending` phải
**xoá `error` cũ**, không thì lần thử lại thành công vẫn còn hiện lỗi cũ.

---

## 9. 🛠️ Tự tay làm — `getToppings` với đủ ba trạng thái

> Mục tiêu: cuối phần này `ToppingPage` sẽ **tự gọi API topping qua Redux**, hiện spinner khi tải,
> hiện danh sách khi xong, và hiện hộp lỗi kèm nút "Thử lại" khi backend chết.

Bạn đang có: `src/api/topping.js` (bài 18), `src/redux/toppingSlice.js` (bài 19),
`src/pages/user/ToppingPage.js` + `src/components/user/ToppingCard.js` (bài 16-17).

### Bước 1 — Thêm thunk `getToppings`

Mở `yotea-fe/src/redux/toppingSlice.js`, sửa phần đầu file. **Toàn bộ code trong mục 9 là code bạn
tự viết — dự án gốc không có file `toppingSlice.js`.**

```js
// yotea-fe/src/redux/toppingSlice.js  ← file bạn tạo từ Bài 19, giờ bổ sung
import { createAsyncThunk, createSlice } from "@reduxjs/toolkit";
import { getAll } from "../api/topping";

const initialState = {
  value: [],
  loading: false,
  error: null,
};

export const getToppings = createAsyncThunk(
  "topping/getToppings",
  async () => {
    const { data } = await getAll();
    return data;
  }
);
```

| Điểm | Vì sao |
|---|---|
| Tiền tố `"topping/getToppings"` | Khớp `name: "topping"` của slice → log trong DevTools đọc rất rõ |
| Hàm async **return `data`**, không return cả response axios | `payload` phải là **mảng topping**, không phải `{ data, status, headers }` |
| Không truyền tham số | Trang Topping chưa phân trang. Cần phân trang thì đổi thành `async ({ start, limit }) => …` như `getProducts` |

### Bước 2 — Viết `extraReducers` đủ 3 case

```js
// yotea-fe/src/redux/toppingSlice.js  ← code bạn tự viết
const toppingSlice = createSlice({
  name: "topping",
  initialState,
  reducers: {
    clearToppings(state) {
      state.value = [];
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(getToppings.pending, (state) => {
        state.loading = true;
        state.error = null;
      })
      .addCase(getToppings.fulfilled, (state, { payload }) => {
        state.loading = false;
        state.value = payload;
      })
      .addCase(getToppings.rejected, (state, action) => {
        state.loading = false;
        state.error = action.error.message || "Không tải được danh sách topping";
      });
  },
});

export const { clearToppings } = toppingSlice.actions;

export const selectToppings = (state) => state.topping.value;
export const selectToppingLoading = (state) => state.topping.loading;
export const selectToppingError = (state) => state.topping.error;

export default toppingSlice.reducer;
```

`clearToppings` nằm ở `reducers` (đồng bộ, slice tự đẻ), ba case kia nằm ở `extraReducers` (của
thunk) — đúng bảng phân biệt ở mục 5.

### Bước 3 — Nối vào `ToppingPage`

Mở `yotea-fe/src/pages/user/ToppingPage.js` (viết ở [Bài 16](16-layout-va-component.md)). Trước đây
nó tự `useState` + gọi API; giờ chuyển hẳn sang Redux. Phần JSX dưới đây đã **lược bớt class
Tailwind dài**, bạn giữ nguyên style đã trang trí ở [Bài 17](17-tailwind-css.md):

```jsx
// yotea-fe/src/pages/user/ToppingPage.js  ← code bạn tự viết
import { useEffect } from "react";
import { useDispatch, useSelector } from "react-redux";
import ToppingCard from "../../components/user/ToppingCard";
import {
  getToppings,
  selectToppingError,
  selectToppingLoading,
  selectToppings,
} from "../../redux/toppingSlice";
import { updateTitle } from "../../utils";

const ToppingPage = () => {
  const dispatch = useDispatch();

  const toppings = useSelector(selectToppings);
  const loading = useSelector(selectToppingLoading);
  const error = useSelector(selectToppingError);

  useEffect(() => {
    updateTitle("Topping");
    dispatch(getToppings());
  }, [dispatch]);

  return (
    <section className="container max-w-6xl mx-auto px-3 py-8">
      <h1 className="text-[#D9A953] font-semibold text-3xl mb-6 text-center">
        Danh sách topping
      </h1>

      {/* ① ĐANG TẢI */}
      {loading && (
        <div className="flex justify-center py-16">
          <div className="w-10 h-10 border-4 border-[#D9A953] border-t-transparent rounded-full animate-spin" />
        </div>
      )}

      {/* ② LỖI */}
      {!loading && error && (
        <div className="max-w-md mx-auto text-center bg-red-50 border border-red-200 rounded-lg p-6">
          <p className="text-red-600 font-semibold mb-3">{error}</p>
          <button type="button" onClick={() => dispatch(getToppings())} className="...">
            Thử lại
          </button>
        </div>
      )}

      {/* ③ THÀNH CÔNG */}
      {!loading && !error && (
        <div className="grid grid-cols-12 gap-6">
          {toppings.length === 0 ? (
            <p className="col-span-12 text-center text-gray-500">Chưa có topping nào.</p>
          ) : (
            toppings.map((item) => <ToppingCard key={item._id} topping={item} />)
          )}
        </div>
      )}
    </section>
  );
};

export default ToppingPage;
```

| Khối | Điều kiện | Ứng với action |
|---|---|---|
| ① | `loading` | `getToppings.pending` |
| ② | `!loading && error` | `getToppings.rejected` |
| ③ | `!loading && !error` | `getToppings.fulfilled` |

Ba điều kiện **loại trừ nhau hoàn toàn** — không bao giờ vừa hiện spinner vừa hiện lỗi. Bên trong ③
còn tách **trạng thái rỗng** (`length === 0`): "chưa có topping nào" khác hẳn "đang tải" và "lỗi".
Rất nhiều bug UI đến từ việc gộp ba thứ này làm một.

> 💡 Muốn overlay che toàn màn hình như trang thanh toán, dùng component có sẵn
> `yotea-fe/src/components/Loading.js:1-21` (`<Loading active={loading} />`, dùng `PuffLoader`).

---

## 10. ✅ Kiểm chứng kết quả

**Đường thành công.** Đứng tại `yotea-be` chạy `npm start`; đứng tại `yotea-fe` chạy `npm start`.
Mở `http://localhost:3000/topping`, phải thấy **theo thứ tự**: spinner vàng quay vài trăm mili-giây →
spinner biến mất → lưới `ToppingCard` hiện ra.

DevTools → tab **Network**, lọc `topping`, phải thấy đúng **một** request:

```
GET http://localhost:8080/api/topping?_sort=createdAt&_order=desc   200 OK
```

```json
[
  { "_id": "665...a1", "name": "Trân châu đen", "price": 5000, "slug": "tran-chau-den" },
  { "_id": "665...a2", "name": "Thạch dừa",     "price": 7000, "slug": "thach-dua" }
]
```

> 💡 Muốn nhìn spinner lâu hơn: DevTools → Network → chọn **Slow 3G** rồi F5.

**Đường thất bại — bước quan trọng nhất của bài.** Sang terminal backend bấm **Ctrl + C** để **tắt
backend**, rồi F5 trang `/topping`:

1. Spinner quay 1–2 giây.
2. Spinner tắt, hiện hộp đỏ **"Network Error"** kèm nút **Thử lại**.
3. Bấm "Thử lại" → spinner quay rồi lại lỗi (backend vẫn tắt).
4. Bật lại backend, bấm "Thử lại" → **dữ liệu hiện ra**, hộp lỗi biến mất.

Bước 4 chứng minh dòng `state.error = null` trong case `pending` hoạt động đúng.

> So sánh cho thấm: làm y hệt với trang `/thuc-don` (dùng slice của dự án) — bạn chỉ nhận được **trang
> trắng im lặng**, không một chữ giải thích. Đó là cái giá của việc bỏ qua `pending` và `rejected`.

---

## 11. 🐞 Lỗi thường gặp

| Thông báo lỗi / Hiện tượng | Nguyên nhân | Cách sửa |
|---|---|---|
| `Actions must be plain objects. Use custom middleware for async actions.` | Store **chưa cắm** thunk | Kiểm tra `store.js:22` có `thunk`; hoặc chuyển sang `configureStore` |
| Spinner quay mãi không dừng | Quên `state.loading = false` ở case `rejected` (hoặc `fulfilled`) | Tắt `loading` ở **cả hai** nhánh kết thúc |
| `x.map is not a function`, dữ liệu `undefined` | Thunk `return` cả response axios thay vì `data` | `const { data } = await getAll(); return data;` |
| `Cannot read properties of undefined (reading 'value')` | Slice chưa đăng ký vào `rootReducer`, hoặc mount key khác tên trong selector | Kiểm tra `rootReducer.js` có `topping: toppingReducer` và selector đọc `state.topping.value` |
| Toast "thành công" dù server lỗi / `try/catch` không chạy | `createAsyncThunk` nuốt lỗi | Thêm `await … .unwrap()` (mục 6.2) |
| Gọi API **2 lần** khi mount | `React.StrictMode` chạy effect hai lần ở môi trường dev | Bình thường ở dev, bản build không bị |
| `useEffect` gọi thunk lặp vô tận | Đưa object/mảng đổi mỗi render vào dependency array | Chỉ đưa giá trị nguyên thuỷ hoặc `dispatch` |
| Xoá item nhưng danh sách không đổi | Thunk delete quên `.then(() => id)` | Xem `productSlice.js:40` |
| Lỗi cũ vẫn hiện sau khi thử lại thành công | Quên `state.error = null` ở `pending` | Thêm dòng đó vào case `pending` |

---

## 12. 📝 Bài tập

**Bài 1.** Nếu ai đó sửa `yotea-fe/src/redux/productSlice.js:40` từ `return remove(id).then(() => id);`
thành `return remove(id);` thì chuyện gì xảy ra? `payload` lúc đó là gì, và reducer dòng 64-66 hành xử ra sao?

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

`remove(id)` là lời gọi axios, nó resolve về **object response đầy đủ**
`{ data, status, statusText, headers, config, request }`. Vậy `payload` là object đó, không phải chuỗi id.

Reducer `state.value = state.value.filter((item) => item._id !== payload);` so sánh một **chuỗi** với
một **object** → **luôn khác nhau** → điều kiện luôn `true` → `filter` giữ lại **toàn bộ** phần tử.

Kết quả: API xoá thành công (bản ghi biến mất khỏi MongoDB) nhưng **giao diện không đổi gì cả**.
Người dùng bấm xoá, thấy toast "Đã xoá thành công", mà dòng đó vẫn nằm nguyên trên bảng cho tới khi F5.
Đây là loại bug khó chịu nhất: **không có thông báo lỗi nào**, chỉ có hành vi sai.

Cách viết an toàn và rõ nghĩa hơn (bạn tự viết thêm):

```js
export const deleteProduct = createAsyncThunk("product/deleteProduct", async (id) => {
  await remove(id);
  return id;      // ← nói thẳng: payload chính là id vừa xoá
});
```

</details>

**Bài 2.** Viết lại `getProducts` sao cho **chỉ gọi API một lần**, giả sử backend đã được sửa để trả
về `{ data, total }` trong body. Kèm `extraReducers` đủ ba trạng thái.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Đoạn dưới **bạn tự viết thêm**, dự án chưa có. Backend (sửa ở [Bài 34](34-refactor-du-an.md)) trả về
`{ "data": [ … ], "total": 137, "page": 1, "totalPages": 18 }`:

```js
export const getProducts = createAsyncThunk(
  "product/getProducts",
  async ({ start, limit }) => {
    const { data } = await getAll(start, limit);
    return data;                       // { data: [...], total: 137, ... }
  }
);

const initialState = { value: [], totalProduct: 0, loading: false, error: null };

extraReducers: (builder) => {
  builder
    .addCase(getProducts.pending, (state) => {
      state.loading = true;
      state.error = null;
    })
    .addCase(getProducts.fulfilled, (state, { payload }) => {
      state.loading = false;
      state.value = payload.data;
      state.totalProduct = payload.total;
    })
    .addCase(getProducts.rejected, (state, action) => {
      state.loading = false;
      state.error = action.error.message || "Không tải được sản phẩm";
    });
},
```

Lợi ích: **một** request thay vì hai, tổng số luôn khớp dữ liệu trả về, và tải được dữ liệu lớn mà
không treo trình duyệt.

</details>

**Bài 3.** Thêm thunk `addTopping` vào `toppingSlice.js` của bạn, xử lý đủ ba case, rồi dùng ở một
form nhỏ. Yêu cầu: chỉ hiện toast "Thêm topping thành công" **khi server thật sự trả 2xx**.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Trong `yotea-fe/src/redux/toppingSlice.js` (code bạn tự viết):

```js
import { add, getAll } from "../api/topping";

export const addTopping = createAsyncThunk(
  "topping/addTopping",
  async (dataTopping) => {
    const { data } = await add(dataTopping);
    return data;
  }
);
```

Nối thêm vào chuỗi `extraReducers`:

```js
      .addCase(addTopping.pending, (state) => {
        state.loading = true;
        state.error = null;
      })
      .addCase(addTopping.fulfilled, (state, { payload }) => {
        state.loading = false;
        state.value = [...state.value, payload];   // khuôn mẫu giống productSlice.js:55
      })
      .addCase(addTopping.rejected, (state, action) => {
        state.loading = false;
        state.error = action.error.message || "Thêm topping thất bại";
      });
```

Trong component (code bạn tự viết):

```jsx
const onSubmit = async (dataInput) => {
  try {
    await dispatch(addTopping(dataInput)).unwrap();   // ← mấu chốt
    toast.success("Thêm topping thành công");
    reset();
  } catch (error) {
    toast.error(error.message || "Thêm topping thất bại");
  }
};
```

Bỏ `.unwrap()` đi thì `dispatch(...)` resolve kể cả khi thunk `rejected`, `catch` không bao giờ chạy,
người dùng luôn thấy "thành công" — đúng lỗi `AdminUpdateInfoPage.js:49-50` đang mắc.

**Kiểm chứng:** tắt backend rồi bấm Thêm → phải thấy toast **đỏ**, không phải toast xanh.

</details>

**Bài 4.** Vì sao `dispatch(getToppings())` trong `useEffect` với dependency `[dispatch]` là an toàn,
còn `[toppings]` sẽ gây **vòng lặp vô tận**?

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

`dispatch` là hàm **ổn định** — `react-redux` đảm bảo nó giữ nguyên tham chiếu suốt vòng đời
component, nên `[dispatch]` tương đương `[]`: effect chạy đúng **một lần** khi mount.

Với `[toppings]`:

```
mount → dispatch(getToppings())
      → fulfilled → state.value = mảng MỚI (payload từ API là object mới hoàn toàn)
      → useSelector thấy tham chiếu đổi → render lại
      → useEffect thấy `toppings` đổi → dispatch(getToppings()) LẦN NỮA
      → … lặp vô tận, bắn request liên tục cho tới khi treo trình duyệt
```

Điểm chết người: dependency array so sánh **tham chiếu** (`Object.is`), không so sánh nội dung. Hai
mảng nội dung y hệt nhưng khác tham chiếu vẫn bị coi là "đã đổi".

**Quy tắc:** dependency array chỉ nên chứa **giá trị nguyên thuỷ** (chuỗi, số, boolean) hoặc tham
chiếu ổn định như `dispatch`. Đừng bao giờ đưa vào đó chính cái state mà effect sẽ ghi vào.

Dự án có một chỗ suýt dính bẫy: `yotea-fe/src/pages/user/cart/CartPage.js:45` dùng
`useEffect(…, [cart, totalPrice])` trong khi bên trong lại `setTotalPrice(...)`. Nó chỉ không nổ vì
React bỏ qua `setState` khi giá trị mới bằng giá trị cũ.

</details>

---

## 📌 Tóm tắt

- Reducer phải **thuần** và **đồng bộ** → không được gọi API. Chỗ đúng để gọi API là **middleware**.
- **redux-thunk** cho phép `dispatch` một **hàm** thay vì object; nó chặn hàm đó lại và đưa cho
  `dispatch` + `getState`. Trong Yotea, thunk được cắm tay ở `store.js:22`.
- **`createAsyncThunk`** sinh 3 action: `pending` (hiện loading) / `fulfilled` (hiện dữ liệu,
  `payload` = giá trị `return`) / `rejected` (hiện lỗi, đọc `action.error.message`).
- **`reducers`** = action do slice tự đẻ (đồng bộ). **`extraReducers`** = nghe action của thunk hoặc
  slice khác. `authSlice` dùng cả hai.
- `getProducts` gọi `getAll()` **hai lần** — một lần lấy hết chỉ để `.length`. Rất lãng phí; cách
  chuẩn là backend trả tổng số trong **header `X-Total-Count`** hoặc trong **body**.
- `dispatch(thunk)` trả về Promise **không bao giờ reject** → `try/catch` vô dụng nếu thiếu
  **`.unwrap()`**. Dự án có dùng `.unwrap()` nhưng chỉ với RTK Query, chưa từng với thunk.
- **Cả 35 `addCase` trong `yotea-fe/src/redux/` đều chỉ xử lý `.fulfilled`** → màn hình trắng khi
  tải, im lặng khi lỗi, và toast "thành công" giả.
- Mẫu chuẩn: `{ value: [], loading: false, error: null }` + 3 `addCase`; tắt `loading` ở **cả**
  `fulfilled` lẫn `rejected`, xoá `error` ở `pending`.

**Từ khoá tra cứu thêm:** `createAsyncThunk`, `redux-thunk middleware`, `extraReducers builder callback`, `unwrapResult redux toolkit`, `thunk pending fulfilled rejected`, `X-Total-Count pagination header`, `countDocuments mongoose`

➡️ **Bài tiếp theo:** [21 — redux-persist: giữ giỏ hàng và phiên đăng nhập](21-redux-persist.md) — dữ liệu topping vừa tải về sẽ **bay sạch khi F5**. Có nên lưu nó xuống localStorage không? Câu trả lời sẽ khiến bạn bất ngờ.
