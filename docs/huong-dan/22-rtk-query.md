# Bài 22 — RTK Query: `createApi`, cache và `invalidatesTags`

> **Phần 3 · Frontend React** — Thời lượng ước tính: **~75 phút**
> ⬅️ Bài trước: [21 — redux-persist: giữ giỏ hàng và phiên đăng nhập](21-redux-persist.md) · Bài sau: [23 — Chức năng Đăng ký / Đăng nhập / Đăng xuất](23-dang-ky-dang-nhap.md) ➡️
> 🏠 [Mục lục](README.md)

---

## 🎯 Sau bài này bạn sẽ

- Hiểu **RTK Query** xoá bỏ được đúng những đoạn code nào bạn đã viết ở bài 18–20.
- Đọc hiểu trọn vẹn khối `createApi` thật trong `yotea-fe/src/api/product.js` — từng thuộc tính một.
- Phân biệt `builder.query` với `builder.mutation`, thuộc lòng **quy tắc đặt tên hook tự sinh**.
- Giải thích được cơ chế **cache theo tag**: vì sao thêm sản phẩm xong danh sách **tự** tải lại.
- Nối được một `createApi` mới vào `rootReducer` + `store`, và hiểu `setupListeners` làm gì.
- Chỉ ra **lỗi hiểu sai API** ở 3 mutation sản phẩm và viết lại đúng bằng `prepareHeaders`.
- Tự viết lại chức năng **Topping** bằng RTK Query, so sánh trực tiếp với cách axios + thunk.

## 📋 Cần chuẩn bị

- Đã hoàn thành [Bài 18](18-tang-api-axios.md), [Bài 19](19-redux-toolkit-co-ban.md), [Bài 20](20-async-thunk.md), [Bài 21](21-redux-persist.md).
- Đã có sẵn: `src/api/topping.js` (axios), `src/redux/toppingSlice.js` (slice + thunk `getToppings`), trang `src/pages/user/ToppingPage.js`.
- Backend chạy ở cổng **8080** với API Topping bạn tự xây ở bài 04–13.

> Ở **bài trước** bạn đã quyết định **không** đưa `topping` vào `whitelist` của redux-persist, vì đó là
> dữ liệu server chứ không phải trạng thái người dùng. **Bài này** ta đi tiếp: nếu dữ liệu server đã có
> nơi chuyên trách lo cache, thì cái `toppingSlice` bạn viết ở bài 19–20 còn cần nữa không?

---

## 1. Bối cảnh: dự án đang chạy **hai đường ống lấy dữ liệu song song**

Nắm điều này trước, không thì mở source ra sẽ rất hoang mang:

```
              Backend Express :8080
                       │
      ┌────────────────┴─────────────────┐
 ĐƯỜNG ỐNG 1 (bài 18-20)          ĐƯỜNG ỐNG 2 (bài này)
 axios instance                   createApi + fetchBaseQuery
 src/api/instance.js              src/api/product.js:96
       │                                 │
 createAsyncThunk                  endpoints { query, mutation }
 src/redux/*Slice.js                     │
       │                                 │
 useDispatch + useSelector        useGetProductsQuery(...)
       │                                 │
   Component                        Component
```

- **Đường ống 1** phục vụ ~95% màn hình: 14 file trong `src/api/` dùng axios, 10 slice bọc bằng `createAsyncThunk`.
- **Đường ống 2** chỉ có ở vài chỗ: danh sách sản phẩm admin, slider trang chủ, danh mục trang chủ.
- Hai đường ống này **không biết nhau** — nguồn gốc một loạt bug ở mục 8.

> 📖 **Thuật ngữ:** *RTK Query* — bộ công cụ lấy dữ liệu (data fetching) nằm **sẵn** trong
> `@reduxjs/toolkit` (`yotea-fe/package.json:12`, bản `^2.2.3`). Không phải thư viện cài thêm.

---

## 2. RTK Query giải quyết vấn đề gì?

Nhìn lại đúng thứ bạn viết ở bài 20 — `yotea-fe/src/redux/productSlice.js:9-19`:

```js
export const getProducts = createAsyncThunk(
  "product/getProducts",
  async ({ start, limit }) => {
    const { data } = await getAll();
    const totalProduct = data.length;

    const { data: productsData } = await getAll(start, limit);

    return { totalProduct, productsData };
  }
);
```

Đếm xem để có **một** danh sách sản phẩm bạn phải viết bao nhiêu thứ:

| Việc phải làm bằng tay | Ở đâu |
|---|---|
| Hàm axios `getAll` | `api/product.js:8-17` |
| 4 thunk (get/add/update/delete) | `productSlice.js:9-42` |
| `initialState` | `productSlice.js:4-7` |
| 4 `addCase` nhét dữ liệu vào state | `productSlice.js:48-67` |
| Selector | `productSlice.js:70-71` |
| `useEffect` + `dispatch(...)` trong component | file page |
| Cờ `loading` / `error` | **dự án không hề có** |

~70 dòng cho một resource, và **lặp y hệt** cho news, slider, contact, store, user… — chính là 11 file
slice trong `src/redux/`.

**RTK Query nói: "để tôi lo."** Bạn chỉ khai *"resource này lấy ở URL nào"*, nó tự sinh:

- Hàm gọi HTTP (dùng `fetch` sẵn của trình duyệt), nơi lưu **cache** trong Redux store.
- Cờ `isLoading`, `isFetching`, `isSuccess`, `isError`, `error`.
- **Gộp request trùng** (2 component cùng cần → chỉ 1 request đi ra).
- **Tự tải lại** khi dữ liệu bị đánh dấu là cũ.
- Một **React hook** cho component dùng thẳng.

Đổi lại tất cả: khoảng **25 dòng**.

---

## 3. Soi code thật: `productApi` trong `yotea-fe/src/api/product.js`

### 3.1. Dòng import

`yotea-fe/src/api/product.js:4`

```js
import { createApi, fetchBaseQuery } from "@reduxjs/toolkit/query/react";
```

> ⚠️ Chú ý đuôi **`/react`**. `@reduxjs/toolkit/query` chỉ có phần lõi, **không sinh hook**;
> `@reduxjs/toolkit/query/react` mới sinh hook React. Import nhầm thì `useGetProductsQuery` là
> `undefined` và bạn sẽ ngồi tìm bug rất lâu.

### 3.2. Toàn bộ khối `createApi` — trích nguyên văn

`yotea-fe/src/api/product.js:96-147`

```js
export const productApi = createApi({
  reducerPath: "productApi",
  baseQuery: fetchBaseQuery({
    baseUrl: "http://localhost:8080/api",
  }),
  tagTypes: ["Product"],
  endpoints: (builder) => ({
    getProducts: builder.query({
      query: ({ start = 0, limit = 0, sort = "createdAt", order = "desc" }) => {
        let url = `/${DB_NAME}/?_expand=categoryId&_sort=${sort}&_order=${order}`;
        if (limit) url += `&_start=${start}&_limit=${limit}`;
        return url;
      },
      providesTags: ["Product"],
    }),

    addProduct: builder.mutation({
      query: (data, { token, user } = isAuthenticate()) => ({
        url: `${DB_NAME}/${user._id}`,
        method: "POST",
        body: data,
        headers: {
          Authorization: `Bearer ${token}`,
        },
      }),
      invalidatesTags: ["Product"],
    }),

    deleteProduct: builder.mutation({
      query: (id, { token, user } = isAuthenticate()) => ({
        url: `${DB_NAME}/${id}/${user._id}`,
        method: "DELETE",
        headers: {
          Authorization: `Bearer ${token}`,
        },
      }),
      invalidatesTags: ["Product"],
    }),

    updateProduct: builder.mutation({
      query: (data, { token, user } = isAuthenticate()) => ({
        url: `${DB_NAME}/${data._id}/${user._id}`,
        method: "PUT",
        body: data,
        headers: {
          Authorization: `Bearer ${token}`,
        },
      }),
      invalidatesTags: ["Product"],
    }),
  }),
});
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 96 | `createApi({` | Tạo một "api slice". Kết quả chứa `reducer`, `middleware` và các hook. |
| 97 | `reducerPath: "productApi"` | **Tên ngăn** trong store để cất cache → state sẽ có `state.productApi`. Phải **duy nhất** trong app. |
| 98-100 | `baseQuery: fetchBaseQuery({ baseUrl })` | Hàm gọi HTTP dùng chung cho mọi endpoint. `fetchBaseQuery` bọc mỏng quanh `fetch`: tự `JSON.stringify` body, tự `res.json()`, tự set `Content-Type`. |
| 101 | `tagTypes: ["Product"]` | **Danh sách nhãn** api này được phép dùng — như khai trước "tôi sẽ dán loại tem nào". Dùng tem chưa khai ở đây → RTK Query báo lỗi. |
| 102 | `endpoints: (builder) => ({` | Hàm nhận `builder`, trả về **object các endpoint**. Chú ý `({` — ngoặc tròn ôm ngoặc nhọn = arrow function trả về object. |
| 103 | `getProducts: builder.query({` | Endpoint **đọc** dữ liệu. Tên `getProducts` quyết định tên hook (mục 3.3). |
| 104 | `query: ({ start, limit, sort, order }) =>` | Hàm **dựng request**, nhận đúng **một** đối số — chính là thứ bạn truyền vào hook. |
| 105-107 | dựng `url` rồi `return url` | `query` trả về **một chuỗi** → RTK Query hiểu là `GET` tới chuỗi đó. |
| 109 | `providesTags: ["Product"]` | *"Kết quả này **mang** tem `Product`."* Nửa đầu cơ chế cache. |
| 112 | `addProduct: builder.mutation({` | Endpoint **ghi** (thêm/sửa/xoá). Khác `query` ở chỗ **không tự chạy**, phải bấm mới chạy. |
| 113 | `query: (data, { token, user } = isAuthenticate())` | **Chỗ này dự án viết sai — mục 7 mổ kỹ.** |
| 114 | `url: \`${DB_NAME}/${user._id}\`` | `products/<id-người-đăng-nhập>` (backend Yotea nhét `userId` vào URL). |
| 115-119 | `method` / `body` / `headers` | Khi `query` trả về **object** thay vì chuỗi, bạn được khai đủ ba thứ này. `body` tự được `JSON.stringify`. |
| 121 | `invalidatesTags: ["Product"]` | *"Chạy xong thì **huỷ hiệu lực** mọi cache mang tem `Product`."* Nửa sau cơ chế cache. |
| 124-133 | `deleteProduct` | Giống `addProduct`, `method: "DELETE"`, không có `body`. |
| 135-145 | `updateProduct` | Giống `addProduct`, `method: "PUT"`, URL kèm `data._id`. |

> 💡 **Hai dạng trả về của `query`:** trả **chuỗi** → mặc định `GET` (dùng cho query);
> trả **object** `{ url, method, body, headers, params }` → tuỳ chỉnh đầy đủ (dùng cho mutation).

### 3.3. Hook tự sinh và quy tắc đặt tên

`yotea-fe/src/api/product.js:149-154`

```js
export const {
  useGetProductsQuery,
  useAddProductMutation,
  useDeleteProductMutation,
  useUpdateProductMutation,
} = productApi;
```

Bạn **không tự viết** 4 hook này — RTK Query sinh sẵn, bạn chỉ bóc ra bằng destructuring rồi export.

**Quy tắc đặt tên — học thuộc:**

```
use + <TênEndpoint viết hoa chữ đầu> + Query      (nếu là builder.query)
use + <TênEndpoint viết hoa chữ đầu> + Mutation   (nếu là builder.mutation)
```

| Tên endpoint | Loại | Hook sinh ra | Khai báo tại |
|---|---|---|---|
| `getProducts` | `builder.query` | `useGetProductsQuery` | `api/product.js:103` |
| `addProduct` | `builder.mutation` | `useAddProductMutation` | `api/product.js:112` |
| `deleteProduct` | `builder.mutation` | `useDeleteProductMutation` | `api/product.js:124` |
| `updateProduct` | `builder.mutation` | `useUpdateProductMutation` | `api/product.js:135` |
| `getSliders` | `builder.query` | `useGetSlidersQuery` | `api/slider.js:51` |
| `getCatesProduct` | `builder.query` | `useGetCatesProductQuery` | `api/category.js:52` |

> ⚠️ Gõ thiếu chữ `s` → `TypeError: useGetProductQuery is not a function`. Cứ soi tên endpoint rồi ghép theo công thức.

### 3.4. Component dùng hook thế nào

`yotea-fe/src/pages/admin/product/ProductListPage.js:6-8`

```js
const ProductListPage = () => {
  const { data: products } = useGetProductsQuery({});
  const totalItem = products?.length || 0;
```

`yotea-fe/src/components/admin/ListProduct.js:8-10`

```js
const ListProduct = ({ start, limit }) => {
  const { data: products } = useGetProductsQuery({ start, limit });
  const [deleteProduct] = useDeleteProductMutation();
```

So với cách cũ:

| Cách cũ — axios + thunk | Cách RTK Query |
|---|---|
| `const products = useSelector(selectProducts);` | `const { data: products } = useGetProductsQuery({...});` |
| `const dispatch = useDispatch();` | *(không cần)* |
| `useEffect(() => { dispatch(getProducts({start,limit})) }, [start, limit]);` | *(không cần — hook tự gọi, đối số đổi thì tự gọi lại)* |

Đối số truyền vào hook (`{ start, limit }`) **chính là** đối số hàm `query` ở dòng 104 nhận được.

Mutation thì trả về **mảng**: phần tử đầu là hàm để bắn, phần tử sau là object trạng thái.

`yotea-fe/src/pages/admin/product/AddProductPage.js:54-65`

```js
      addProduct({
        ...data,
        image: url,
        price: +data.price,
        status: +data.status,
      })
        .unwrap()
        .then(() => {
          toast.success("Thêm sản phẩm thành công");
          setPreview("");
          reset();
        });
```

> 📖 **Thuật ngữ:** `.unwrap()` — mutation mặc định **không bao giờ reject**, nó trả về `{ data }`
> hoặc `{ error }`. `.unwrap()` biến nó thành Promise bình thường: thành công thì resolve dữ liệu,
> thất bại thì **throw** để `try/catch` bắt được.

---

## 4. Cơ chế cache theo tag — phần hay nhất của RTK Query

### 4.1. Kịch bản cụ thể

Bạn đang ở `/admin/product` (đang chạy `useGetProductsQuery({})`). Bạn bấm "Thêm SP", lưu — và danh
sách **đã có sản phẩm mới**, dù không ai `dispatch` gì cả. Vì sao?

```
BƯỚC 1 — Trang danh sách chạy hook query
  useGetProductsQuery({})  →  GET /api/products/?_sort=createdAt&_order=desc
                           →  cất vào cache, DÁN TEM ["Product"]   (providesTags, dòng 109)

     ┌──── cache productApi ─────────────────┐
     │  getProducts({}) → [SP1, SP2, SP3]    │  🏷️ Product
     └───────────────────────────────────────┘

BƯỚC 2 — Bấm nút Thêm
  addProduct({...}) → POST /api/products/<userId> → OK
                    → invalidatesTags: ["Product"]   (dòng 121)

BƯỚC 3 — Middleware của productApi nghe thấy tem "Product" bị huỷ
  → duyệt cache, thấy entry getProducts({}) MANG TEM đó → đánh dấu "cũ rồi"
  → vì trang danh sách vẫn còn mở (còn subscribe) → TỰ ĐỘNG bắn lại GET

BƯỚC 4 — Dữ liệu mới về → React re-render → danh sách có sản phẩm mới. XONG.
```

Bước 3 và 4 **bạn không viết một dòng code nào**.

### 4.2. So với cách cũ

`productSlice.js:54-56` phải tự tay nhét vào state:

```js
    builder.addCase(addProduct.fulfilled, (state, { payload }) => {
      state.value = [...state.value, payload];
    });
```

Ba vấn đề: (1) phải tin backend trả về object đầy đủ — nếu không `populate` `categoryId` thì
`item.categoryId?.name` (`ListProduct.js:121`) hiện trống; (2) sản phẩm mới bị đẩy **xuống cuối**
trong khi API sắp `createdAt desc` → thứ tự khác sau khi F5; (3) `totalProduct` không cập nhật →
phân trang sai. RTK Query né cả ba, vì nó **hỏi lại server** thay vì đoán.

### 4.3. Tem phải "khớp cặp" mới có tác dụng

| API slice | `tagTypes` | `providesTags`? | Có mutation `invalidatesTags`? | Kết luận |
|---|---|---|---|---|
| `productApi` (`api/product.js:96`) | `["Product"]` | ✅ dòng 109 | ✅ dòng 121, 132, 144 | **Hoạt động đúng** |
| `sliderApi` (`api/slider.js:44`) | `["Slider"]` | ✅ dòng 53 | ❌ không có mutation nào | Tem **vô dụng** |
| `cateProductApi` (`api/category.js:45`) | `["CateProduct"]` | ✅ dòng 54 | ❌ không có mutation nào | Tem **vô dụng** |
| `newsApi` (`api/news.js:57`) | *không khai* | ❌ | ❌ | Không dùng tag |
| `userApi` (`api/user.js:54`) | *không khai* | ❌ | ❌ | Không dùng tag |

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** `sliderApi` và `cateProductApi` khai `tagTypes` + `providesTags`
> đầy đủ nhưng **không mutation nào huỷ tem đó**, vì việc sửa slider/danh mục ở admin đi qua **đường ống
> axios** (`sliderSlice`, `categoryProductSlice`). Hậu quả rất cụ thể: admin sửa banner xong, mở trang chủ →
> **vẫn thấy banner cũ** cho tới khi F5.

---

## 5. Nối một `createApi` vào store

Cần **hai** thứ được đăng ký, thiếu một là hỏng.

### 5.1. Reducer — nơi cất cache

`yotea-fe/src/redux/rootReducer.js:31-35`

```js
  [productApi.reducerPath]: productApi.reducer,
  [sliderApi.reducerPath]: sliderApi.reducer,
  [cateProductApi.reducerPath]: cateProductApi.reducer,
  [userApi.reducerPath]: userApi.reducer,
  [newsApi.reducerPath]: newsApi.reducer,
```

Đây là **computed key** ([Bài 03](03-kien-thuc-nen.md), mục 1.8). Vì `productApi.reducerPath` có giá trị
`"productApi"`, dòng đầu tương đương `productApi: productApi.reducer`. Viết kiểu này để đổi
`reducerPath` ở `api/product.js:97` thì chỗ này **tự đúng theo**, không phải sửa hai nơi.

### 5.2. Middleware — bộ não điều phối

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

`productApi.middleware` mới là thứ **thật sự gửi request, quản lý vòng đời cache, xử lý tag, gộp
request trùng và dọn cache khi không còn ai dùng**. Reducer chỉ là cái tủ chứa.

> ⚠️ **Quên `middleware`** là lỗi kinh điển: app không crash, nhưng mọi hook query đứng yên ở
> `isLoading: true` mãi mãi, kèm cảnh báo *"Middleware for RTK-Query API at reducerPath 'productApi'
> has not been added to the store."*

### 5.3. `setupListeners`

`yotea-fe/src/redux/store.js:30-36`

```js
export const store = createStore(
  persistedReducer,
  applyMiddleware(...middleware)
);
export default persistStore(store);

setupListeners(store.dispatch);
```

Nó gắn hai trình lắng nghe của trình duyệt vào store: `window` lấy lại focus (bật được
`refetchOnFocus`) và sự kiện `online` (bật được `refetchOnReconnect`).

> 💡 `setupListeners` chỉ **bật khả năng**. Muốn dữ liệu thật sự tải lại còn phải khai
> `refetchOnFocus: true` ở `createApi` hoặc ở từng hook — dự án Yotea **không khai chỗ nào**, nên dòng
> `store.js:36` hiện chạy mà chưa mang lại tác dụng gì.

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** `setupListeners(...)` nằm **sau** `export default persistStore(store)`
> (dòng 34). Vẫn chạy, nhưng đặt lệnh khởi tạo sau `export default` là cách viết lộn xộn. Ngoài ra dự án
> dùng `createStore` cũ thay vì `configureStore` — ta sửa ở [Bài 34](34-refactor-du-an.md).

---

## 6. Kiểm kê: RTK Query thực sự nằm ở đâu?

`grep "createApi"` trên `yotea-fe/src` cho **5 khai báo**:

| # | File | Api slice | Dòng | `tagTypes` | Số endpoint |
|---|---|---|---|---|---|
| 1 | `yotea-fe/src/api/product.js` | `productApi` | 96 | `["Product"]` | 4 (1 query + 3 mutation) |
| 2 | `yotea-fe/src/api/slider.js` | `sliderApi` | 44 | `["Slider"]` | 1 query |
| 3 | `yotea-fe/src/api/user.js` | `userApi` | 54 | *không có* | 1 query |
| 4 | `yotea-fe/src/api/category.js` | `cateProductApi` | 45 | `["CateProduct"]` | 1 query |
| 5 | `yotea-fe/src/api/news.js` | `newsApi` | 57 | *không có* | 1 query |

**10/15 file** còn lại trong `src/api/` hoàn toàn dùng axios. Grep các hook trong `pages/` + `components/`:

| Hook | Khai báo tại | Được dùng ở |
|---|---|---|
| `useGetProductsQuery` | `api/product.js:150` | `components/admin/ListProduct.js:9`, `pages/admin/product/ProductListPage.js:7` |
| `useAddProductMutation` | `api/product.js:151` | `pages/admin/product/AddProductPage.js:32` |
| `useDeleteProductMutation` | `api/product.js:152` | `components/admin/ListProduct.js:10` |
| `useUpdateProductMutation` | `api/product.js:153` | `pages/admin/product/EditProductPage.js:30` |
| `useGetSlidersQuery` | `api/slider.js:58` | `components/user/home/HomeBanner.js:7` |
| `useGetCatesProductQuery` | `api/category.js:59` | `components/user/home/HomeCategory.js:5` |
| `useGetNewsQuery` | `api/news.js:69` | ❌ **không dùng ở đâu cả** |
| `useGetUsersQuery` | `api/user.js:66` | ❌ **không dùng ở đâu cả** |

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** `newsApi` và `userApi` là **code chết** — hook không ai gọi,
> nhưng reducer vẫn nạp (`rootReducer.js:34-35`) và middleware vẫn chạy (`store.js:26-27`). Chúng
> phình bundle và khiến người đọc tưởng tin tức/người dùng đã chuyển sang RTK Query (thực tế vẫn đi
> qua `newsSlice`, `userSlice`).

> ⚠️ Chiều ngược lại cũng phải nói thẳng: **`yotea-fe/src/redux/productSlice.js` cũng là code chết.**
> Grep xác nhận nó **chỉ** được import ở `rootReducer.js:7` để lấy reducer; 4 thunk của nó
> **không được `dispatch` ở bất kỳ đâu**. Trang admin sản phẩm đã chuyển hết sang RTK Query mà không ai
> xoá slice cũ — dấu vết của một cuộc di cư dở dang.

---

## 7. ⚠️ PHÂN TÍCH LỖI QUAN TRỌNG: `query: (data, { token, user } = isAuthenticate())`

Phần đáng giá nhất của bài. Nhìn lại `yotea-fe/src/api/product.js:113` (y hệt ở `:125` và `:136`):

```js
      query: (data, { token, user } = isAuthenticate()) => ({
```

### 7.1. Vì sao đây là **hiểu sai API**

Hợp đồng của RTK Query rất rõ: hàm `query` của `builder.query` / `builder.mutation`
**chỉ được gọi với ĐÚNG MỘT đối số** — chính là `arg` bạn truyền vào hook.

```
addProduct(payload)  →  RTK Query gọi:  query(payload)
                                         └── và DỪNG. Không có đối số thứ hai.
```

Vì vậy tham số thứ hai **luôn luôn là `undefined`** → biểu thức mặc định `isAuthenticate()`
**luôn luôn chạy**. Đối số `{ token, user }` **không bao giờ** nhận được giá trị từ bên ngoài.
Cái vẻ ngoài "có thể truyền token vào để test" là **giả** — không có cách nào truyền.

### 7.2. "May mà vẫn chạy" — nhưng may thôi

Code này chạy đúng trong kịch bản bình thường vì: default parameter được **đánh giá lại ở mỗi lần gọi**
(không phải một lần lúc import), nên `isAuthenticate()` luôn đọc token mới nhất từ `localStorage`; và
admin thì bao giờ cũng đã đăng nhập khi vào `/admin/product/add`. Nhưng cái sai đó vẫn có giá:

| # | Hậu quả | Giải thích |
|---|---|---|
| 1 | **Đánh lừa người đọc** | Chữ ký hàm gợi ý một khả năng không tồn tại. Người mới thử `addProduct(data, { token })` → đối số thứ hai bị **im lặng bỏ qua**, không lỗi, không cảnh báo. |
| 2 | **Crash khi chưa đăng nhập** | `isAuthenticate()` (`utils/localStorage.js:1-4`) đọc `localStorage` **không try/catch**. Chưa có `persist:root` → `TypeError: Cannot read properties of null (reading 'auth')`. Đã boot nhưng chưa login → `user` là `undefined` → dòng `url: \`${DB_NAME}/${user._id}\`` ném `TypeError`. |
| 3 | **Lỗi ném ở chỗ khó bắt** | Lỗi trên xảy ra **bên trong middleware RTK Query**, không phải trong component. `.unwrap()` reject bằng lỗi JavaScript chứ không phải lỗi HTTP → thông báo cho người dùng thành vô nghĩa. |
| 4 | **Sai nguồn sự thật** | Token đọc từ `localStorage` (bản sao đã serialize) thay vì Redux store (bản gốc). redux-persist ghi **bất đồng bộ** → vừa đăng nhập xong mà gọi mutation ngay có thể đọc phải token cũ. |
| 5 | **Lặp code** | `headers: { Authorization: ... }` chép lại ở **cả ba** mutation. |

Hàm nó phụ thuộc — `yotea-fe/src/utils/localStorage.js:1-4`:

```js
export const isAuthenticate = () => {
  return JSON.parse(JSON.parse(localStorage.getItem("persist:root")).auth)
    .value;
};
```

Không guard, không fallback, không try/catch. Xem thêm [Bài 21](21-redux-persist.md) và [Bài 33](33-ra-soat-bao-mat.md).

### 7.3. Cách viết ĐÚNG: `prepareHeaders`

RTK Query có sẵn chỗ dành riêng cho việc gắn header: tham số `prepareHeaders` của `fetchBaseQuery`.
Nó chạy **trước mỗi request** và nhận `getState` — tức đọc thẳng từ **Redux store**, đúng nguồn sự thật.

```js
// CODE MẪU do bạn tự viết — dự án hiện KHÔNG có đoạn này.
export const productApi = createApi({
  reducerPath: "productApi",
  baseQuery: fetchBaseQuery({
    baseUrl: "http://localhost:8080/api",
    prepareHeaders: (headers, { getState }) => {
      const token = getState().auth?.value?.token;
      if (token) headers.set("authorization", `Bearer ${token}`);
      return headers;
    },
  }),
  tagTypes: ["Product"],
  endpoints: (builder) => ({
    addProduct: builder.mutation({
      query: (data) => ({            // chỉ MỘT tham số — đúng hợp đồng
        url: `products/${data.userId}`,
        method: "POST",
        body: data,                  // KHÔNG cần headers ở đây nữa
      }),
      invalidatesTags: ["Product"],
    }),
  }),
});
```

`getState().auth.value.token` khớp shape thật của `authSlice`: `signin` gán `state.value = payload`
(`yotea-fe/src/redux/authSlice.js:37-40`) và `selectAuth = (state) => state.auth.value` (`authSlice.js:59`).

**Bốn cái lợi:** (1) viết **một lần**, mọi endpoint đều có token; (2) đọc từ Redux store, cùng nguồn với
`useSelector`, không lệch; (3) có `if (token)` → chưa đăng nhập thì không gửi `Bearer undefined` và
**không crash**; (4) hàm `query` chỉ còn một tham số → khớp hợp đồng, không đánh lừa ai.

Nếu vẫn cần `user._id` để dựng URL (backend Yotea yêu cầu), hãy **truyền qua đối số** thay vì đọc lén:

```js
// component — code bạn tự viết thêm
const auth = useSelector(selectAuth);
addProduct({ ...formData, userId: auth.user._id });
```

---

## 8. ⚠️ Hai vấn đề kiến trúc còn lại

### 8.1. `baseUrl` bị hardcode lại ở **từng** `createApi`

Grep `localhost:8080` trong `yotea-fe/src` cho **6 kết quả**: `api/instance.js:4` (`baseURL` của axios),
`api/product.js:99`, `api/slider.js:47`, `api/category.js:48`, `api/user.js:57`, `api/news.js:60`
(`baseUrl` của 5 `fetchBaseQuery`).

Sáu chuỗi giống hệt nhau. Muốn deploy phải sửa tay **6 file**; sót một file thì một phần trang trắng mà
không có lỗi rõ ràng. CRA hỗ trợ sẵn `process.env.REACT_APP_API_URL` nhưng dự án không dùng — gom lại ở
[Bài 34](34-refactor-du-an.md).

> 💡 **Bẫy chính tả:** axios viết `baseURL` (**URL** hoa cả ba chữ), `fetchBaseQuery` viết `baseUrl`
> (chỉ hoa chữ **U**). Viết nhầm giữa hai thư viện **không hề báo lỗi**, chỉ khiến request bay tới địa chỉ sai.

### 8.2. Hai hệ thống lấy cùng một dữ liệu

| Resource | Đường ống axios + thunk | Đường ống RTK Query | Hậu quả |
|---|---|---|---|
| Sản phẩm | `api/product.js:8` + `redux/productSlice.js` | `productApi.getProducts` (`product.js:103`) | Slice là **code chết** nhưng vẫn nằm trong store |
| Slider | `api/slider.js:7` + `redux/sliderSlice.js` | `sliderApi.getSliders` (`slider.js:51`) | Admin sửa slider → **trang chủ vẫn hiện cái cũ** tới khi F5 |
| Danh mục | `api/category.js:7` + `redux/categoryProductSlice.js` | `cateProductApi.getCatesProduct` (`category.js:52`) | Như slider; ngoài ra bản RTK Query **thiếu `_sort`** → thứ tự danh mục trang chủ khác trang admin |

**Bài học:** chọn **một** cách rồi làm cho trót. Trộn hai cách trong cùng một resource là công thức tạo
ra bug "dữ liệu cũ" mà không ai giải thích nổi.

---

## 9. 🛠️ Tự tay làm — viết lại Topping bằng RTK Query

> Mục tiêu: cuối phần bạn có `toppingApi` chạy được, `ToppingPage` không còn `useDispatch`/`useSelector`,
> và khi thêm topping mới thì danh sách **tự cập nhật**.

> **Toàn bộ code mục 9 là code BẠN TỰ VIẾT THÊM. Dự án Yotea gốc không có đoạn nào như dưới đây.**

### Bước 1 — Thêm `toppingApi` vào `src/api/topping.js`

Mở file `yotea-fe/src/api/topping.js` viết ở bài 18, **giữ nguyên** các hàm axios cũ, chỉ **thêm** vào cuối:

```js
// yotea-fe/src/api/topping.js  ← THÊM vào file bạn đã có từ Bài 18
// (thêm dòng import này lên đầu file, cạnh các import cũ)
import { createApi, fetchBaseQuery } from "@reduxjs/toolkit/query/react";

// ... giữ nguyên DB_NAME và getAll/get/add/update/remove từ Bài 18 ...

export const toppingApi = createApi({
  reducerPath: "toppingApi",
  baseQuery: fetchBaseQuery({
    baseUrl: "http://localhost:8080/api",
    // ✅ cách ĐÚNG: gắn token một lần cho mọi endpoint, đọc từ Redux store
    prepareHeaders: (headers, { getState }) => {
      const token = getState().auth?.value?.token;
      if (token) headers.set("authorization", `Bearer ${token}`);
      return headers;
    },
  }),
  tagTypes: ["Topping"],
  endpoints: (builder) => ({
    getToppings: builder.query({
      query: ({ start = 0, limit = 0 } = {}) => {
        let url = `/${DB_NAME}/?_sort=createdAt&_order=desc`;
        if (limit) url += `&_start=${start}&_limit=${limit}`;
        return url;
      },
      providesTags: ["Topping"],
    }),

    addTopping: builder.mutation({
      query: (data) => ({          // CHỈ MỘT tham số — đúng hợp đồng RTK Query
        url: `/${DB_NAME}/${data.userId}`,
        method: "POST",
        body: data,
      }),
      invalidatesTags: ["Topping"],
    }),
  }),
});

export const { useGetToppingsQuery, useAddToppingMutation } = toppingApi;
```

Ba điểm cần để ý:

1. File trong `src/api/` **không bao giờ** được import hook React (`useSelector`…). Token lấy qua `getState`.
2. `DB_NAME` giữ nguyên giá trị bạn đặt ở bài 18 (phải khớp route backend viết ở bài 06–07).
3. `{ start = 0, limit = 0 } = {}` — thêm `= {}` để gọi `useGetToppingsQuery()` không đối số cũng không nổ.

### Bước 2 — Đăng ký reducer vào `rootReducer.js`

```js
// yotea-fe/src/redux/rootReducer.js  ← BẠN SỬA, thêm 2 chỗ
import { toppingApi } from "../api/topping";            // cạnh các import api khác

const store = combineReducers({
  // ... giữ nguyên toàn bộ key cũ ...
  [newsApi.reducerPath]: newsApi.reducer,
  [toppingApi.reducerPath]: toppingApi.reducer,         // ← thêm dòng này
});
```

### Bước 3 — Đăng ký middleware vào `store.js`

```js
// yotea-fe/src/redux/store.js  ← BẠN SỬA, thêm 2 chỗ
import { toppingApi } from "../api/topping";            // cạnh các import api khác

const middleware = [
  thunk,
  productApi.middleware,
  sliderApi.middleware,
  cateProductApi.middleware,
  userApi.middleware,
  newsApi.middleware,
  toppingApi.middleware,                                // ← thêm dòng này
];
```

> ⚠️ **Đừng** thêm `"toppingApi"` vào `whitelist` của `persistConfig` (`store.js:16`). Lý do đã bàn ở
> [Bài 21](21-redux-persist.md): cache RTK Query là **dữ liệu server**, persist nó chỉ tạo ra dữ liệu cũ
> mắc kẹt trong localStorage.

### Bước 4 — Thay `useSelector`/`useDispatch` trong `ToppingPage.js`

**Trước** (phiên bản bài 20, mã bạn đã viết): `useDispatch` + `useSelector(selectToppings)` +
`useEffect(() => dispatch(getToppings({ start: 0, limit: 12 })), [dispatch])`.

**Sau** (phiên bản RTK Query, bạn tự viết):

```jsx
// yotea-fe/src/pages/user/ToppingPage.js  ← PHIÊN BẢN MỚI
import { useGetToppingsQuery } from "../../api/topping";
import ToppingCard from "../../components/user/ToppingCard";
import Loading from "../../components/Loading";

const ToppingPage = () => {
  const {
    data: toppings,
    isLoading,
    isError,
    error,
  } = useGetToppingsQuery({ start: 0, limit: 12 });

  if (isLoading) return <Loading />;
  if (isError) return <p className="text-red-500">Lỗi: {error?.status}</p>;

  return (
    <div className="grid grid-cols-12 gap-5">
      {toppings?.map((item) => (
        <ToppingCard key={item._id} topping={item} />
      ))}
    </div>
  );
};

export default ToppingPage;
```

**Biến mất:** `useDispatch`, `useSelector`, `useEffect`, import từ `toppingSlice`.
**Được thêm miễn phí:** `isLoading`, `isError`, `error` — thứ `toppingSlice` chưa từng có.

### Bước 5 — Dùng mutation ở trang admin

```jsx
// code bạn tự viết thêm
import { useSelector } from "react-redux";
import { selectAuth } from "../../../redux/authSlice";
import { useAddToppingMutation } from "../../../api/topping";

const AddToppingPage = () => {
  const [addTopping, { isLoading }] = useAddToppingMutation();
  const auth = useSelector(selectAuth);

  const onSubmit = async (data) => {
    try {
      await addTopping({ ...data, userId: auth.user._id }).unwrap();
      toast.success("Thêm topping thành công");
    } catch (error) {
      toast.error("Có lỗi xảy ra, vui lòng thử lại");
    }
  };
  // ... JSX form giữ nguyên, nút submit thêm disabled={isLoading} ...
};
```

`userId` được **truyền vào** qua đối số thay vì đọc lén localStorage; token thì `prepareHeaders` đã lo
ở Bước 1 — bạn không phải viết `headers` ở endpoint nào cả.

---

## 10. ✅ Kiểm chứng kết quả

```bash
# terminal 1 — đứng tại thư mục yotea-be
npm start

# terminal 2 — đứng tại thư mục yotea-fe
npm start
```

**Kiểm chứng 1 — hook chạy.** Mở `http://localhost:3000/topping`, bật DevTools tab **Network**, lọc
`Fetch/XHR`. Phải thấy **đúng một** request:

```
GET http://localhost:8080/api/toppings/?_sort=createdAt&_order=desc&_start=0&_limit=12   200
```

trả về mảng JSON dạng:

```json
[
  { "_id": "6650a1f2c4e8b91234abcd01", "name": "Trân châu đen", "price": 5000, "slug": "tran-chau-den" }
]
```

**Kiểm chứng 2 — cache có thật.** Bấm sang trang khác rồi quay lại `/topping`: **không có request mới**
trong 60 giây đầu (mặc định `keepUnusedDataFor` là 60 giây), nhưng danh sách vẫn hiện ngay lập tức.

**Kiểm chứng 3 — `invalidatesTags` (quan trọng nhất).** Trong **cùng một tab**: mở `/topping`, đi tới
trang admin thêm topping, thêm "Trân châu trắng", bấm lưu, rồi quay lại `/topping`. Tab Network sẽ có
**hai** request nối nhau:

```
POST http://localhost:8080/api/toppings/6650...    201   ← mutation
GET  http://localhost:8080/api/toppings/?_sort=...  200   ← TỰ ĐỘNG chạy lại!
```

Request `GET` thứ hai **bạn không hề gọi** — nó xảy ra vì `invalidatesTags: ["Topping"]` đã huỷ hiệu lực
cache mang tem `Topping`. Danh sách hiện "Trân châu trắng" ngay, không cần F5.

> Nếu bạn thêm topping bằng **Postman** thay vì qua mutation thì danh sách **sẽ không** tự cập nhật —
> vì Postman không đi qua middleware của `toppingApi`. Đây chính xác là bài học ở mục 8.2.

**Kiểm chứng 4 — token đi đúng chỗ.** Trong request `POST`, tab **Headers** → **Request Headers** phải có:

```
authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

Thấy `Bearer undefined` → `prepareHeaders` đọc sai đường dẫn state; kiểm tra lại `getState().auth.value.token`.

---

## 11. 🐞 Lỗi thường gặp

| Thông báo lỗi | Nguyên nhân | Cách sửa |
|---|---|---|
| `Middleware for RTK-Query API at reducerPath 'toppingApi' has not been added to the store` | Quên `toppingApi.middleware` | Thêm vào mảng `middleware` (Bước 3) |
| Hook đứng mãi ở `isLoading: true`, Network không có request nào | Như trên | Như trên |
| `useGetToppingsQuery is not a function` | Import từ `@reduxjs/toolkit/query` (thiếu `/react`), hoặc gõ sai tên hook | Import đúng đường; đối chiếu tên theo công thức mục 3.3 |
| `Cannot destructure property 'start' of 'undefined'` | Gọi `useGetToppingsQuery()` không đối số mà `query` lại destructuring object | Thêm `= {}`: `query: ({ start = 0, limit = 0 } = {}) => ...` |
| `reducerPath 'toppingApi' already exists` | Hai `createApi` trùng `reducerPath` | Đặt `reducerPath` duy nhất cho từng api slice |
| Thêm xong nhưng danh sách **không** tự cập nhật | Thiếu `providesTags`, thiếu `invalidatesTags`, hoặc tên tem hai bên không khớp (`"Topping"` vs `"topping"`) | Đủ ba: khai trong `tagTypes`, dán ở query, huỷ ở mutation — **cùng chuỗi, đúng hoa thường** |
| Request bay tới `http://localhost:3000/toppings` thay vì cổng 8080 | Viết nhầm `baseURL` (kiểu axios) trong `fetchBaseQuery` | Đổi thành `baseUrl` |
| Header là `Bearer undefined` | Chưa đăng nhập, hoặc `getState()` trỏ sai key | Thêm `if (token)`; kiểm tra shape `state.auth.value.token` |
| 401 dù đã đăng nhập | `prepareHeaders` đặt sai chỗ (phải nằm **trong** `fetchBaseQuery`, không phải trong `createApi`) | Xem lại Bước 1 |
| `data` là `undefined` ở lần render đầu | Bình thường — request chưa xong | Dùng `toppings?.map(...)` hoặc chặn bằng `if (isLoading)` |

---

## 12. 📝 Bài tập

**Bài 1.** Nhìn khối `sliderApi` — `yotea-fe/src/api/slider.js:44-56`:

```js
export const sliderApi = createApi({
  reducerPath: "sliderApi",
  baseQuery: fetchBaseQuery({
    baseUrl: "http://localhost:8080/api",
  }),
  tagTypes: ["Slider"],
  endpoints: (builder) => ({
    getSliders: builder.query({
      query: () => `${DB_NAME}/?_sort=createdAt&_order=desc`,
      providesTags: ["Slider"],
    }),
  }),
});
```

(a) Hook sinh ra tên là gì? (b) `tagTypes` + `providesTags` ở đây có tác dụng không, vì sao?
(c) Admin vừa sửa banner ở `/admin/slider` xong, mở trang chủ thì thấy gì?

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

(a) Endpoint `getSliders` là `builder.query` → hook **`useGetSlidersQuery`** (khớp `slider.js:58`,
dùng ở `components/user/home/HomeBanner.js:7`).

(b) **Không có tác dụng gì.** Tem chỉ phát huy khi có **cặp**: một bên `providesTags`, một bên
`invalidatesTags`. `sliderApi` chỉ có **một** endpoint query, **không mutation nào** → không ai huỷ tem
`"Slider"` → dán tem xong để đó. Code trang trí.

(c) **Vẫn thấy banner cũ** tới khi F5 (hoặc tới khi cache hết 60 giây và component mount lại). Vì thao
tác sửa slider ở admin đi qua `redux/sliderSlice.js` → axios → **không chạm vào middleware của `sliderApi`**.

**Cách sửa đúng:** chuyển luôn phần thêm/sửa/xoá slider sang mutation của `sliderApi` với
`invalidatesTags: ["Slider"]`. Tức là: **đừng để một resource có hai đường ống.**

</details>

**Bài 2.** Đây là mutation thật — `yotea-fe/src/api/product.js:124-133`:

```js
    deleteProduct: builder.mutation({
      query: (id, { token, user } = isAuthenticate()) => ({
        url: `${DB_NAME}/${id}/${user._id}`,
        method: "DELETE",
        headers: {
          Authorization: `Bearer ${token}`,
        },
      }),
      invalidatesTags: ["Product"],
    }),
```

Một bạn đọc code này rồi viết trong component:

```js
const [deleteProduct] = useDeleteProductMutation();
deleteProduct(productId, { token: myToken, user: myUser });   // ???
```

Giải thích chuyện thực sự xảy ra, rồi viết lại endpoint cho đúng chuẩn.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

**Chuyện thực sự xảy ra:** đối số thứ hai bị **im lặng bỏ qua**. Hook mutation chỉ nhận **một** đối số,
và RTK Query cũng chỉ gọi `query` với **một** đối số → tham số thứ hai luôn `undefined` → biểu thức mặc
định `isAuthenticate()` **luôn** chạy → token/user vẫn đọc từ `localStorage`, không phải `myToken`/`myUser`.

Nguy hiểm ở chỗ **không lỗi, không cảnh báo**. Người viết tưởng đã truyền token khác, thực tế thì không —
nếu đang test bằng tài khoản khác, kết quả sẽ sai một cách khó hiểu.

**Viết lại đúng chuẩn** (code bạn tự viết):

```js
baseQuery: fetchBaseQuery({
  baseUrl: "http://localhost:8080/api",
  prepareHeaders: (headers, { getState }) => {
    const token = getState().auth?.value?.token;
    if (token) headers.set("authorization", `Bearer ${token}`);
    return headers;
  },
}),

deleteProduct: builder.mutation({
  query: ({ id, userId }) => ({          // MỘT đối số
    url: `products/${id}/${userId}`,
    method: "DELETE",
  }),
  invalidatesTags: ["Product"],
}),
```

Component gọi: `deleteProduct({ id: productId, userId: auth.user._id }).unwrap();`

Giờ chữ ký hàm nói đúng sự thật, token đọc từ Redux store, và hết lặp `headers` ở ba nơi.

</details>

**Bài 3.** Thêm **phân trang** cho `/topping` bằng RTK Query: component nhận `page` từ URL
(`useParams`), tính `start`, truyền vào hook. Kèm câu hỏi: khi người dùng bấm qua lại giữa trang 1 và
trang 2, có bao nhiêu request được gửi đi?

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

```jsx
// yotea-fe/src/pages/user/ToppingPage.js — code bạn tự viết thêm
import { useParams } from "react-router-dom";
import { useGetToppingsQuery } from "../../api/topping";

const ToppingPage = () => {
  const { page } = useParams();
  const limit = 12;
  const currentPage = Number(page) || 1;
  const start = (currentPage - 1) * limit;

  const { data: toppings, isFetching } = useGetToppingsQuery({ start, limit });

  return (
    <div className={isFetching ? "opacity-50 transition" : "transition"}>
      <div className="grid grid-cols-12 gap-5">
        {toppings?.map((item) => (
          <ToppingCard key={item._id} topping={item} />
        ))}
      </div>
    </div>
  );
};
```

**Trả lời:** chỉ **2 request** cho cả phiên (trong vòng 60 giây). RTK Query cache **theo đối số**:
`{ start: 0, limit: 12 }` và `{ start: 12, limit: 12 }` là **hai entry khác nhau**. Vào trang 1 → 1
request; qua trang 2 → 1 request; quay lại trang 1 → **lấy từ cache, không request**. Với cách cũ
(`useEffect` + `dispatch`), **mỗi lần** đổi trang là **một** request mới.

**Mẹo `isFetching` vs `isLoading`:** `isLoading` chỉ `true` ở **lần tải đầu tiên** của entry đó;
`isFetching` `true` **mỗi lần** có request đang bay, kể cả khi đã có dữ liệu cũ để hiện. Dùng
`isFetching` để làm mờ danh sách cũ trong lúc chờ → mượt hơn hẳn so với chớp sang màn hình trắng.

</details>

**Bài 4.** Dự án có hai `createApi` mà hook **không được dùng ở đâu**. Chỉ ra chúng và liệt kê chính
xác những dòng cần xoá nếu muốn dọn sạch.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Hai chỗ đó là **`newsApi`** và **`userApi`**.

| Cần xoá | Vị trí |
|---|---|
| Khối `newsApi` + dòng export hook | `yotea-fe/src/api/news.js:57-69` |
| Khối `userApi` + dòng export hook | `yotea-fe/src/api/user.js:54-66` |
| `import { userApi } ...` / `import { newsApi } ...` | `yotea-fe/src/redux/rootReducer.js:16-17` |
| `[userApi.reducerPath]` / `[newsApi.reducerPath]` | `yotea-fe/src/redux/rootReducer.js:34-35` |
| Hai dòng import tương ứng | `yotea-fe/src/redux/store.js:10-11` |
| `userApi.middleware,` / `newsApi.middleware,` | `yotea-fe/src/redux/store.js:26-27` |

Sau khi xoá, tin tức vẫn chạy qua `redux/newsSlice.js`, người dùng vẫn chạy qua `redux/userSlice.js` —
vì hai hook kia **chưa từng được gọi**.

Muốn dọn luôn `redux/productSlice.js` (code chết, mục 6) thì nhớ xoá cả key `product: productReducer`
(`rootReducer.js:25`) và ba hàm axios `add`/`remove`/`update` (`api/product.js:59-84`) vốn chỉ phục vụ
nó. **Nhưng đừng làm bây giờ** — để dành cho [Bài 34](34-refactor-du-an.md), nơi được phép sửa code dự án.

</details>

---

## 13. BẢNG SO SÁNH: axios + `createAsyncThunk` vs RTK Query

| Tiêu chí | axios + `createAsyncThunk` (bài 18–20) | RTK Query (bài này) |
|---|---|---|
| **Số dòng cho 1 resource CRUD** | ~70 (hàm api + 4 thunk + initialState + 4 `addCase` + selector) | ~25 (1 khối `createApi`) |
| **Số file phải đụng vào** | 3 (`api/x.js`, `redux/xSlice.js`, component) | 2 (`api/x.js`, component) |
| **Loading / error** | **Tự viết** — Yotea không viết → không màn hình chờ, không báo lỗi | Có sẵn `isLoading`, `isFetching`, `isSuccess`, `isError`, `error` |
| **Cache** | Không có. Mount lại là gọi API lại | Có sẵn, **cache theo đối số**, giữ mặc định 60 giây |
| **Gộp request trùng** | Không — 2 component cùng cần → 2 request | Có — nhiều component subscribe 1 entry → **1 request** |
| **Tự refetch sau khi ghi** | Phải tự `dispatch` lại, hoặc tự sửa mảng trong `extraReducers` (dễ lệch server) | `invalidatesTags` lo hết, **không viết dòng nào** |
| **Refetch khi quay lại tab / có mạng lại** | Không có | `setupListeners` + `refetchOnFocus`/`refetchOnReconnect` |
| **Gắn token** | Lặp `headers: { Authorization }` từng hàm (dự án lặp **34 lần**) | `prepareHeaders` — **một lần** cho cả api slice |
| **Huỷ request khi unmount** | Phải tự `AbortController` (dự án không làm) | Tự động |
| **Độ khó ban đầu** | Dễ hiểu — mọi thứ hiện rõ trước mắt | Khó hơn — nhiều thứ chạy ngầm, phải tin cơ chế tem |
| **Chèn logic tuỳ ý** | Thoải mái: trong thunk gọi mấy API cũng được, biến đổi tuỳ thích | Gò hơn: cần `transformResponse`, `queryFn`, `onQueryStarted` |
| **Upload file có progress** | Dễ (axios có `onUploadProgress`) | Khó — `fetch` không hỗ trợ progress |
| **👉 Nên chọn khi** | Logic nhiều bước, trộn nhiều API, upload có progress, hoặc **state không đến từ server** (giỏ hàng, form nhiều bước, bộ lọc UI) | **Đọc/ghi dữ liệu server thuần tuý** — danh sách, chi tiết, CRUD admin. Tức **đa số** màn hình |

**Kết luận thực dụng cho Yotea:** `cartSlice`, `authSlice` → **giữ nguyên** `createSlice` (state của
client). `productSlice`, `newsSlice`, `sliderSlice`, `categoryProductSlice`, `userSlice`, `contactSlice`,
`storeSlice`, `cateNewsSlice`, `wishlistSlice` → về lý thuyết **nên** chuyển hết sang RTK Query.
Điều tệ nhất là **làm nửa chừng** — đúng thứ dự án đang mắc phải.

---

## 📌 Tóm tắt

- **RTK Query** nằm sẵn trong `@reduxjs/toolkit`, import từ `@reduxjs/toolkit/query/react` (nhớ đuôi `/react`).
- Một `createApi` cần 4 thứ: **`reducerPath`** (tên ngăn cache), **`baseQuery`** (`fetchBaseQuery` + `baseUrl`),
  **`tagTypes`** (danh sách tem), **`endpoints`** (`builder.query` đọc, `builder.mutation` ghi).
- Hàm `query` trả **chuỗi** = `GET`; trả **object** `{ url, method, body, headers }` = tuỳ chỉnh đầy đủ.
- Hook sinh tự động theo công thức **`use` + `TênEndpoint` + `Query`/`Mutation`**.
- Cặp **`providesTags` ↔ `invalidatesTags`** là trái tim của cache: query dán tem, mutation xé tem →
  danh sách **tự tải lại**. Thiếu một vế thì tem vô dụng (đúng như `sliderApi`, `cateProductApi`).
- Đăng ký **hai** chỗ: reducer vào `rootReducer.js:31-35`, middleware vào `store.js:21-28`.
  Thiếu middleware là hook đứng im ở `isLoading`.
- ⚠️ Ba mutation ở `api/product.js:113,125,136` viết `query: (data, { token, user } = isAuthenticate())` —
  **sai hợp đồng**: RTK Query chỉ truyền **một** đối số nên tham số thứ hai luôn rơi vào giá trị mặc định.
  Cách đúng là **`prepareHeaders`** đọc token từ Redux store.
- ⚠️ `baseUrl` hardcode ở **6 chỗ**; **3 resource** (sản phẩm, slider, danh mục) tồn tại song song hai
  đường ống lấy dữ liệu → cache lệch, code chết.
- Chọn RTK Query cho **dữ liệu server**; giữ `createSlice` cho **state của client** (giỏ hàng, phiên đăng nhập).

**Từ khoá tra cứu thêm:** `RTK Query createApi`, `fetchBaseQuery prepareHeaders`, `providesTags invalidatesTags`,
`keepUnusedDataFor`, `setupListeners refetchOnFocus`, `RTK Query vs createAsyncThunk`, `isFetching vs isLoading`

➡️ **Bài tiếp theo:** [23 — Chức năng Đăng ký / Đăng nhập / Đăng xuất](23-dang-ky-dang-nhap.md) —
hết phần công cụ, bắt đầu nghiệp vụ thật: từ form đăng ký, qua `bcrypt` ở backend, tới JWT vào Redux và
cái `persist:root` bạn đã gặp suốt bốn bài vừa rồi.
