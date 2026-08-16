# Bài 34 — Refactor: `.env`, `configureStore`, xử lý lỗi tập trung

> **Phần 6 · Nâng cao & hoàn thiện** — Thời lượng ước tính: **~120 phút**
> ⬅️ Bài trước: [33 — Rà soát bảo mật: dự án đang sai ở đâu](33-ra-soat-bao-mat.md) · Bài sau: [35 — 🎓 Đồ án cuối khoá: làm chức năng Voucher end-to-end](35-do-an-cuoi-voucher.md) ➡️
> 🏠 [Mục lục](README.md)

---

## 🎯 Sau bài này bạn sẽ

- Đưa được mọi **bí mật** (chuỗi kết nối MongoDB, `JWT_SECRET`, cổng) ra khỏi mã nguồn, cất vào file `.env` — cả backend lẫn frontend.
- Viết được `.gitignore` đúng và giải thích vì sao **không bao giờ** commit `node_modules` và `.env`.
- Thay `createStore` cũ kỹ trong `redux/store.js` bằng `configureStore` của Redux Toolkit, lấy lại Redux DevTools miễn phí.
- Thêm được **middleware xử lý lỗi tập trung** (4 tham số) + route 404, để `UnauthorizedError` trả về JSON gọn thay vì trang HTML lộ stack trace.
- Xoá sạch nạn copy-paste hàm `list()` ở 13 controller bằng một hàm dùng chung.
- Vá hai lỗ hổng nặng nhất từ [Bài 33](33-ra-soat-bao-mat.md): băm mật khẩu bằng **bcrypt** và khoá chặt API `updateInfo`.

## 📋 Cần chuẩn bị

- Đã đọc kỹ [Bài 33](33-ra-soat-bao-mat.md) và giữ lại **bảng báo cáo bảo mật** bạn tự lập ở phần "Tự tay làm".
- Dự án đang chạy được: backend tại `yotea-be` (cổng 8080), frontend tại `yotea-fe` (cổng 3000).
- Đã cài `git`. Bài này **sửa code thật** nên ta làm trên một nhánh riêng để lỡ hỏng thì quay lại được.

> ⚠️ **Khác mọi bài trước ở một điểm sống còn.** Từ Bài 04 đến Bài 33, quy tắc là *"không
> sửa code dự án, chỉ đọc và học"*. **Bài 34 là ngoại lệ duy nhất**: đây là bài thực hành
> *sửa*. Nhưng mọi đoạn "code sau khi sửa" dưới đây là **bản cải tiến do bạn (người học)
> tự áp dụng**, không phải code gốc của dự án. Dự án gốc vẫn còn nguyên các lỗi của Bài 33
> — chính vì thế ta mới ngồi đây vá.

---

## 1. Làm việc an toàn: một nhánh git riêng cho việc refactor

Refactor là *đổi cấu trúc bên trong mà không đổi hành vi bên ngoài*. Nghe thì hiền, nhưng
đụng vào `.env`, `store`, hook băm mật khẩu... là đụng vào tim của dự án. Sai một bước có
thể làm cả hệ thống không đăng nhập được nữa. Nguyên tắc: **tách ra một nhánh, sửa từng
phần nhỏ, test lại sau mỗi phần, rồi mới gộp.**

```bash
# đứng tại thư mục gốc D:/PROJECT-SELL-MILK-TEA/PROJECT-SELL-MILK-TEA
git checkout -b refactor/bai-34
```

Giờ bạn đang ở nhánh `refactor/bai-34`. Nhánh `main` vẫn nguyên vẹn. Sau mỗi mục dưới đây,
hãy commit một lần với thông điệp rõ ràng, ví dụ `git commit -am "mục 1: .env backend"`.
Nếu một mục làm hỏng, `git checkout -- .` để bỏ thay đổi chưa commit, hoặc `git reset --hard`
để về commit gần nhất.

> 💡 **Mẹo:** làm theo đúng thứ tự 7 mục dưới đây. Mục 1 (`.env` backend) là nền cho mục 6
> (bcrypt) và mục 7 (vá `updateInfo`), vì cả hai đều cần `JWT_SECRET` đã nằm trong `.env`.

**Bản đồ 7 việc sẽ làm — dán lên tường:**

```
  BACKEND                              FRONTEND
  ┌───────────────────────────┐       ┌───────────────────────────┐
  │ 1. .env + dotenv           │       │ 2. .env REACT_APP_API_URL │
  │    + .gitignore            │       │    (instance + createApi) │
  │ 4. error handler + 404     │       │ 3. configureStore         │
  │ 5. tách list() dùng chung  │       └───────────────────────────┘
  │ 6. bcrypt                  │
  │ 7. vá updateInfo           │
  └───────────────────────────┘
```

---

## 2. Mục 1 — Biến môi trường backend (`.env`, `dotenv`, `.gitignore`)

### 2.1. Hiện trạng: bí mật viết cứng khắp nơi

Ở [Bài 33](33-ra-soat-bao-mat.md) ta đã chỉ ra chuỗi `"TuongVy"` bị hardcode và dùng chung
cho **cả ba việc**: ký JWT, verify JWT, và băm mật khẩu. Chuỗi kết nối MongoDB cũng viết
thẳng trong code. Soi lại `yotea-be/src/app.js:44-52`:

```js
// connect db
mongoose
  .connect("mongodb://localhost:27017/yotea")
  .then(() => console.log("Connected to MongoDB"))
  .catch((error) => console.log(error));

// connect
const PORT = process.env.PORT || 8080;
app.listen(PORT, () => console.log(`App is running on port: ${PORT}`));
```

`yotea-be/src/middlewares/checkAuth.js:3-7`:

```js
export const requireSignin = expressJWT({
  algorithms: ["HS256"],
  secret: "TuongVy",
  requestProperty: "auth",
});
```

`yotea-be/src/controllers/auth.js:50-54`:

```js
      const token = jwt.sign(
        { _id: user._id, email: user.email },
        "TuongVy",
        { expiresIn: "3h" }
      );
```

Và trong `yotea-be/src/models/user.js:73` (băm khi lưu) lẫn dòng `89` (băm khi cập nhật),
lại một chuỗi `"TuongVy"` nữa.

### 2.2. Vì sao phải sửa

- **Bí mật lộ theo mã nguồn.** Bất kỳ ai đọc được source (dự án gốc lại còn commit cả
  `node_modules` lên git) đều biết khoá ký JWT, và **tự ký được một token admin giả**.
- **Không đổi được theo môi trường.** Máy dev, máy staging, máy production cần khoá và
  database khác nhau. Viết cứng thì mỗi lần deploy phải sửa code.
- **`process.env.PORT` đã có sẵn nhưng lẻ loi** — dòng `51` đã biết đọc biến môi trường,
  nghĩa là ý tưởng đúng đã manh nha, chỉ chưa làm tới nơi.

> 📖 **Thuật ngữ:** *biến môi trường* (environment variable) — cặp `KEY=value` do hệ điều
> hành/tiến trình cung cấp, đọc trong Node bằng `process.env.KEY`. Thư viện **dotenv** đọc
> một file tên `.env` rồi bơm các cặp đó vào `process.env` lúc khởi động.

### 2.3. Sau khi sửa (bản cải tiến bạn tự áp dụng)

**Bước a — cài dotenv.** Đứng tại `yotea-be`:

```bash
cd yotea-be
npm install dotenv
```

**Bước b — tạo file `yotea-be/.env`** (file MỚI, bạn tự tạo):

```bash
# yotea-be/.env  ← KHÔNG commit file này lên git
PORT=8080
MONGO_URI=mongodb://localhost:27017/yotea
JWT_SECRET=hay-doi-thanh-mot-chuoi-ngau-nhien-that-dai-va-kho-doan
```

> 🔒 **Ghi chú bảo mật:** `JWT_SECRET` thật phải là chuỗi ngẫu nhiên dài (32+ ký tự). Sinh
> nhanh bằng `node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"`.

**Bước c — nạp dotenv thật sớm** trong `yotea-be/src/app.js`. Thêm **ngay dòng đầu tiên**,
trước mọi import khác:

```js
// yotea-be/src/app.js  ← thêm dòng này lên TRÊN CÙNG (bản sửa của bạn)
import "dotenv/config";
```

Vì sao phải là dòng đầu? Vì các file được import bên dưới (như `checkAuth.js`) sẽ đọc
`process.env.JWT_SECRET` ngay lúc nạp — nếu dotenv chạy sau, biến còn `undefined`.

**Bước d — thay chuỗi kết nối MongoDB.** Sửa `yotea-be/src/app.js`:

```js
// TRƯỚC
.connect("mongodb://localhost:27017/yotea")

// SAU (bản sửa của bạn)
.connect(process.env.MONGO_URI)
```

**Bước e — thay `"TuongVy"` ở cả 4 chỗ** thành `process.env.JWT_SECRET`:

| File | Chỗ cần sửa | Trước | Sau |
|---|---|---|---|
| `middlewares/checkAuth.js` | dòng `5` | `secret: "TuongVy"` | `secret: process.env.JWT_SECRET` |
| `controllers/auth.js` | dòng `52` | `"TuongVy"` (tham số 2 của `jwt.sign`) | `process.env.JWT_SECRET` |
| `models/user.js` | dòng `73` | `createHmac("SHA256", "TuongVy")` | *(sẽ thay hẳn bằng bcrypt ở mục 6)* |
| `models/user.js` | dòng `89` | `createHmac("SHA256", "TuongVy")` | *(mục 6)* |

> ⚠️ **Bẫy thứ tự nạp:** `checkAuth.js` gọi `expressJWT({ secret: process.env.JWT_SECRET })`
> **ngay lúc file được import** (không phải lúc có request). Nếu `import "dotenv/config"`
> không nằm trước `import ... routes` trong `app.js`, `secret` sẽ là `undefined` và mọi API
> cần đăng nhập báo lỗi khó hiểu. Đây là lý do bước c bắt buộc là **dòng đầu tiên**.

**Bước f — tạo `yotea-be/.gitignore`** (file MỚI):

```gitignore
# yotea-be/.gitignore  ← file MỚI, bạn tự tạo
node_modules/
.env
```

Vì sao **không** commit `node_modules`?

- **Nặng và vô nghĩa.** Nó là hàng ngàn file tải về từ npm — dựng lại y hệt bất cứ lúc nào
  bằng `npm install` dựa trên `package.json` + `package-lock.json`. Dự án Yotea gốc lỡ
  commit hơn 8000 file `node_modules` vào git ([Bài 33](33-ra-soat-bao-mat.md), mục B-20).
- **Có thể chứa mã biên dịch theo hệ điều hành** — commit từ Windows, người khác `git pull`
  trên máy Mac có thể gặp lỗi.

Còn `.env` không commit vì nó chứa **bí mật**. Thay vào đó, người ta commit một file mẫu
`.env.example` (chỉ ghi tên khoá, không ghi giá trị thật) để đồng đội biết cần khai báo gì.

---

## 3. Mục 2 — Biến môi trường frontend (`REACT_APP_API_URL`)

### 3.1. Hiện trạng: `baseURL` lặp lại khắp nơi

`yotea-fe/src/api/instance.js:1-7`:

```js
import axios from "axios";

const instance = axios.create({
  baseURL: "http://localhost:8080/api",
});

export default instance;
```

Và chuỗi đó **lặp lại** trong từng khối `createApi` của RTK Query, ví dụ
`yotea-fe/src/api/product.js:96-100`:

```js
export const productApi = createApi({
  reducerPath: "productApi",
  baseQuery: fetchBaseQuery({
    baseUrl: "http://localhost:8080/api",
  }),
```

Các file `api/slider.js`, `api/category.js`, `api/user.js`, `api/news.js` cũng lặp y hệt.
Đến ngày deploy (Bài 36), địa chỉ backend đổi thành `https://api.yotea.com/api`, bạn phải
đi sửa **6 chỗ** — quên một chỗ là gọi API hỏng mà không biết vì sao.

### 3.2. Sau khi sửa (bản cải tiến bạn tự áp dụng)

> 📖 **Thuật ngữ:** Create React App (CRA) chỉ nạp biến môi trường có tiền tố
> **`REACT_APP_`**. Biến không có tiền tố này bị bỏ qua (để tránh lỡ tay lộ bí mật server ra
> trình duyệt). Biến được "nướng" vào bundle lúc `npm run build`, nên đổi `.env` phải **khởi
> động lại** `npm start`.

**Bước a — tạo `yotea-fe/.env`** (file MỚI):

```bash
# yotea-fe/.env  ← file MỚI, bạn tự tạo
REACT_APP_API_URL=http://localhost:8080/api
```

**Bước b — sửa `yotea-fe/src/api/instance.js`:**

```js
// yotea-fe/src/api/instance.js  ← bản sửa của bạn
import axios from "axios";

const instance = axios.create({
  baseURL: process.env.REACT_APP_API_URL,
});

export default instance;
```

**Bước c — sửa mọi `fetchBaseQuery`.** Trong cả 5 file `createApi`, thay:

```js
// TRƯỚC
baseUrl: "http://localhost:8080/api",

// SAU (bản sửa của bạn)
baseUrl: process.env.REACT_APP_API_URL,
```

**Bước d — thêm `.env` vào `.gitignore` của frontend.** CRA tạo sẵn `.gitignore` có dòng
`.env*` rồi — kiểm tra lại cho chắc; nếu chưa có thì thêm `.env`.

> 💡 **Mẹo:** nếu sau khi sửa mà trang trắng, mở Console trình duyệt xem `process.env.REACT_APP_API_URL`
> có `undefined` không. `undefined` = bạn quên khởi động lại `npm start` sau khi tạo `.env`.

---

## 4. Mục 3 — Thay `createStore` bằng `configureStore`

### 4.1. Hiện trạng

`yotea-fe/src/redux/store.js:1-37`:

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

### 4.2. Vì sao phải sửa

`createStore` + `applyMiddleware` là **API Redux đời cũ**, đã bị chính đội Redux đánh dấu
**deprecated** (không khuyến khích dùng). Dự án đang dùng nó nên **mất trắng** những thứ mà
`configureStore` cho không:

| Thứ được cho | Với `createStore` | Với `configureStore` |
|---|---|---|
| Redux DevTools | ❌ phải tự cấu hình | ✅ tự bật ở chế độ dev |
| Kiểm tra state có bị mutate không | ❌ | ✅ cảnh báo lúc dev |
| Kiểm tra giá trị có serialize được không | ❌ | ✅ |
| `thunk` middleware | phải tự thêm | ✅ có sẵn |

### 4.3. Sau khi sửa (bản cải tiến bạn tự áp dụng)

```js
// yotea-fe/src/redux/store.js  ← bản sửa của bạn
import storage from "redux-persist/lib/storage";
import {
  persistStore,
  persistReducer,
  FLUSH,
  REHYDRATE,
  PAUSE,
  PERSIST,
  PURGE,
  REGISTER,
} from "redux-persist";
import { configureStore } from "@reduxjs/toolkit";
import { setupListeners } from "@reduxjs/toolkit/query/react";
import rootReducer from "./rootReducer";
import { productApi } from "../api/product";
import { sliderApi } from "../api/slider";
import { cateProductApi } from "../api/category";
import { userApi } from "../api/user";
import { newsApi } from "../api/news";

const persistConfig = {
  key: "root",
  storage,
  whitelist: ["auth", "cart"],
};

const persistedReducer = persistReducer(persistConfig, rootReducer);

export const store = configureStore({
  reducer: persistedReducer,
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({
      // redux-persist phát ra vài action mang giá trị không-serialize-được;
      // bỏ qua chúng để tắt cảnh báo giả
      serializableCheck: {
        ignoredActions: [FLUSH, REHYDRATE, PAUSE, PERSIST, PURGE, REGISTER],
      },
    }).concat(
      productApi.middleware,
      sliderApi.middleware,
      cateProductApi.middleware,
      userApi.middleware,
      newsApi.middleware
    ),
});

export default persistStore(store);

setupListeners(store.dispatch);
```

**Đọc những chỗ quan trọng:**

| Đoạn | Ý nghĩa |
|---|---|
| `getDefaultMiddleware()` | trả về bộ middleware mặc định (đã có sẵn `thunk` + hai bộ kiểm tra dev) → **không cần import `redux-thunk` nữa** |
| `.concat(productApi.middleware, ...)` | nối thêm 5 middleware của RTK Query vào **cuối** — bắt buộc, nếu thiếu thì cache/refetch của RTK Query không chạy |
| `ignoredActions: [...]` | redux-persist bắn ra action `persist/REHYDRATE`... chứa giá trị không serialize được; khai báo bỏ qua để tránh cảnh báo đỏ giả trong console |

> 💡 **Kết quả nhìn thấy được:** cài extension **Redux DevTools** cho Chrome, mở tab
> "Redux". Trước khi sửa, tab này báo *"No store found"*. Sau khi sửa, bạn thấy toàn bộ
> state cây (`auth`, `cart`, `productApi`...) và mọi action chạy qua — vàng ròng để debug.

---

## 5. Mục 4 — Error handler tập trung + route 404

### 5.1. Hiện trạng: không có lưới an toàn cuối cùng

Nhìn lại `yotea-be/src/app.js:22-52` — sau khối `app.use("/api", ...)` là nhảy thẳng tới
`mongoose.connect` rồi `app.listen`. **Không có** middleware xử lý lỗi, **không có** nhánh
bắt "đường dẫn không tồn tại". Hậu quả (đã nêu ở [Bài 33](33-ra-soat-bao-mat.md), lỗ hổng #8):

- Token sai/hết hạn → `express-jwt` ném `UnauthorizedError` → Express mặc định trả về
  **trang HTML kèm stack trace**, lộ đường dẫn file và cấu trúc thư mục server.
- Gõ sai URL API → cũng ra trang HTML khó hiểu thay vì JSON `404`.

### 5.2. Vì sao phải sửa

Express nhận ra một middleware là **error handler** khi nó có **đúng 4 tham số**:
`(err, req, res, next)`. Bất cứ lỗi nào ném ra (hoặc `next(err)`) trong toàn app sẽ rơi
vào đây. Đặt nó **ở cuối cùng** thì nó thành "lưới đỡ" gom mọi lỗi về một chỗ, trả JSON
sạch sẽ, và **giấu stack trace** khỏi client.

### 5.3. Sau khi sửa (bản cải tiến bạn tự áp dụng)

Mở `yotea-be/src/app.js`, thêm khối sau **ngay sau** dòng `app.use("/api", favoritesRouter);`
(dòng `42`) và **trước** `mongoose.connect` (dòng `45`):

```js
// yotea-be/src/app.js  ← chèn giữa các route và mongoose.connect (bản sửa của bạn)

// 1) Không route nào khớp → 404 JSON (đặt SAU tất cả route)
app.use((req, res) => {
  res.status(404).json({ message: "Không tìm thấy đường dẫn yêu cầu" });
});

// 2) Error handler tập trung — PHẢI có đủ 4 tham số (err, req, res, next)
app.use((err, req, res, next) => {
  // token sai / hết hạn: express-jwt ném UnauthorizedError
  if (err.name === "UnauthorizedError") {
    return res.status(401).json({ message: "Token không hợp lệ hoặc đã hết hạn" });
  }

  // các lỗi còn lại: ghi log ra terminal server, trả thông báo gọn cho client
  console.error(err);
  res.status(err.status || 500).json({
    message: err.message || "Lỗi máy chủ",
  });
});
```

> ⚠️ **Chỗ cực dễ sai:** thứ tự đăng ký quyết định tất cả. Route 404 phải đứng **sau** mọi
> `app.use("/api", ...)` (nếu đứng trước, nó nuốt hết request). Error handler phải là
> **cuối cùng**, và **bắt buộc viết đủ 4 tham số** — bỏ `next` đi (còn 3 tham số) thì
> Express hiểu nhầm nó là middleware thường, lỗi sẽ **không** rơi vào đây.

---

## 6. Mục 5 — Tách hàm `list()` dùng chung, hết copy-paste 13 chỗ

Ở [Bài 09](09-bo-loc-query.md) bạn đã phát hiện: khối dựng `filter` trong hàm `list()` bị
**copy-paste y hệt ở 13/14 controller**, và đã tự viết hàm dùng chung
`yotea-be/src/utils/buildQuery.js` để áp dụng cho controller Topping. Bài này ta **nhân
rộng** thành tựu đó ra toàn dự án.

### 6.1. Hiện trạng: mỗi controller một bản sao

Ví dụ `yotea-be/src/controllers/user.js:112-186` là **~50 dòng** dựng filter + query, giống
gần như từng ký tự với `controllers/product.js`, `controllers/news.js`... Sửa một lỗi (ví
dụ lỗi `_start`/`_limit` lọt vào filter ở [Bài 33](33-ra-soat-bao-mat.md)) là phải sửa **13
lần**, và chắc chắn sẽ bỏ sót.

### 6.2. Sau khi sửa (bản cải tiến bạn tự áp dụng)

Bạn đã có sẵn `yotea-be/src/utils/buildQuery.js` từ [Bài 09](09-bo-loc-query.md). Giờ chỉ
việc thay hàm `list` trong **từng** controller còn lại theo đúng khuôn đã dùng cho Topping:

```js
// yotea-be/src/controllers/user.js  ← thay TOÀN BỘ hàm list (bản sửa của bạn)
import { buildQuery } from "../utils/buildQuery";

export const list = async (req, res) => {
  const { filter, sortOpt, start, limit, populate } = buildQuery(req.query);

  try {
    const users = await User.find(filter)
      .select("-password -__v") // tiện tay bỏ luôn password (vá lỗ hổng #2 Bài 33)
      .populate(populate)
      .skip(start)
      .limit(limit)
      .sort(sortOpt)
      .exec();
    res.json(users);
  } catch (error) {
    res.status(400).json({ message: "Không tìm thấy user", error });
  }
};
```

Làm lần lượt cho: `product`, `category`, `news`, `cateNews`, `slider`, `store`, `contact`,
`comment`, `rating`, `favoritesProduct`, `order`, `orderDetail`. Mỗi hàm rút từ ~50 dòng
xuống ~15 dòng, và **toàn bộ 13 controller chia sẻ đúng một bộ logic** — sửa một lần, đúng
mọi nơi.

> 💡 **Lưu ý riêng cho `controllers/category.js`:** nó có thêm vòng lặp gộp `products` cho
> mỗi danh mục (vòng lặp N+1 nhắc ở [Bài 09](09-bo-loc-query.md)). Giữ nguyên phần đặc thù
> đó, chỉ thay khối dựng `filter` bằng `buildQuery(req.query)`.

---

## 7. Mục 6 — Băm mật khẩu bằng `bcryptjs`

Nối tiếp [Bài 11](11-mat-khau-va-jwt.md) và lỗ hổng #4 của [Bài 33](33-ra-soat-bao-mat.md).

### 7.1. Hiện trạng

`yotea-be/src/models/user.js:66-94`:

```js
userSchema.methods = {
  authenticate(password) {
    return this.password === this.encryptPassword(password);
  },
  encryptPassword(password) {
    if (!password) return;
    try {
      return createHmac("SHA256", "TuongVy")
        .update(password)
        .digest("hex");
    } catch (error) {
      console.log(error);
    }
  },
};

userSchema.pre("save", function (next) {
  this.password = this.encryptPassword(this.password);
  next();
});

userSchema.pre("findOneAndUpdate", function (next) {
  if (this._update.password) {
    this._update.password = createHmac("SHA256", "TuongVy")
      .update(this._update.password)
      .digest("hex");
  }
  next();
});
```

Ba bệnh (Bài 33): **SHA256 không salt** (hai người cùng mật khẩu ra cùng chuỗi băm),
**quá nhanh** (dễ brute-force), **khoá cứng `"TuongVy"`**. Thêm một bệnh nữa: hook
`pre("save")` **không kiểm `isModified("password")`** — mỗi lần `.save()` (kể cả chỉ sửa
`fullName`) lại băm đè lên chuỗi băm cũ, làm hỏng mật khẩu.

### 7.2. Sau khi sửa (bản cải tiến bạn tự áp dụng)

**Bước a — cài bcryptjs.** Tại `yotea-be`:

```bash
npm install bcryptjs
```

> 💡 Dùng `bcryptjs` (thuần JavaScript) thay `bcrypt` (cần biên dịch native) để tránh rắc
> rối cài đặt trên Windows.

**Bước b — sửa `yotea-be/src/models/user.js`:**

```js
// yotea-be/src/models/user.js  ← bản sửa của bạn
import bcrypt from "bcryptjs";
// ...phần Schema giữ nguyên...

// so mật khẩu người gõ với chuỗi băm đã lưu — trả về Promise nên là async
userSchema.methods.authenticate = async function (password) {
  return await bcrypt.compare(password, this.password);
};

// băm khi tạo mới / khi password thực sự thay đổi
userSchema.pre("save", async function (next) {
  if (!this.isModified("password")) return next(); // ← vá lỗi băm đè
  this.password = await bcrypt.hash(this.password, 10); // 10 = "độ chậm", tự sinh salt
  next();
});

// băm khi cập nhật mật khẩu qua findOneAndUpdate
userSchema.pre("findOneAndUpdate", async function (next) {
  if (this._update.password) {
    this._update.password = await bcrypt.hash(this._update.password, 10);
  }
  next();
});
```

**Bước c — sửa nơi gọi `authenticate`.** Vì `authenticate` giờ là `async` (trả Promise),
`controllers/auth.js` phải `await` nó. Sửa dòng `42` (trong `signin`) và dòng `127` (trong
`checkPassword`):

```js
// TRƯỚC
} else if (!user.authenticate(password)) {

// SAU (bản sửa của bạn)
} else if (!(await user.authenticate(password))) {
```

> ⚠️ **Bẫy quên `await`:** nếu để nguyên `!user.authenticate(password)`, biểu thức là
> `!Promise` → luôn `false` → **ai gõ mật khẩu gì cũng đăng nhập được**. Đây là lỗi bảo
> mật còn tệ hơn ban đầu, nên bước c là bắt buộc.

**Bước d — dữ liệu cũ.** Mọi user tạo trước khi đổi có mật khẩu băm kiểu SHA256, không
`compare` được với bcrypt. Cách gọn nhất khi đang học: **tạo lại tài khoản** (đăng ký mới),
hoặc xoá collection `users` trong MongoDB Compass rồi đăng ký lại admin.

---

## 8. Mục 7 — Vá lỗ hổng `updateInfo`

Đây là lỗ hổng **#1 nghiêm trọng nhất** của [Bài 33](33-ra-soat-bao-mat.md), cũng nối tiếp
phần "Tự tay làm" của [Bài 29](29-tai-khoan-cua-toi.md).

### 8.1. Hiện trạng

`yotea-be/src/routes/users.js:12`:

```js
router.put("/users/updateInfo/:myId/:userId", requireSignin, isAuth, update);
```

`yotea-be/src/controllers/user.js:225-227`:

```js
export const update = async (req, res) => {
    const filter = { _id: req.params.id || req.params.myId };
    const update = req.body;
```

`isAuth` chỉ kiểm `:userId` khớp token, **không** kiểm `:myId` (người thật sự bị sửa), và
`const update = req.body` nhét **nguyên** dữ liệu client gửi vào database. Khách thường gửi
`{"role": 1}` là **tự phong admin**; đổi `:myId` thành id người khác là **chiếm hồ sơ bất
kỳ ai**.

### 8.2. Sau khi sửa (bản cải tiến bạn tự áp dụng)

Hai chìa khoá, đúng nguyên tắc vàng của [Bài 33](33-ra-soat-bao-mat.md): **lấy id từ token,
không từ URL** và **whitelist (danh sách trắng) các trường được sửa**.

**Bước a — thêm một hàm riêng `updateInfo`** trong `yotea-be/src/controllers/user.js` (đừng
dùng chung hàm `update` với admin nữa):

```js
// yotea-be/src/controllers/user.js  ← hàm MỚI, bản sửa của bạn
export const updateInfo = async (req, res) => {
  // 1) id lấy từ TOKEN, không lấy từ URL
  const filter = { _id: req.auth._id };

  // 2) chỉ nhận đúng các trường an toàn — role/active/password bị chặn tuyệt đối
  const { fullName, phone, address, wardsCode, districtCode, provinceCode } = req.body;
  const update = { fullName, phone, address, wardsCode, districtCode, provinceCode };

  try {
    const user = await User.findOneAndUpdate(filter, update, { new: true })
      .select("-password -__v")
      .exec();
    res.json(user);
  } catch (error) {
    res.status(400).json({ message: "Cập nhật thông tin không thành công", error });
  }
};
```

**Bước b — sửa route** `yotea-be/src/routes/users.js`. Bỏ `:myId`/`:userId` khỏi URL vì id
đã lấy từ token; chỉ cần đăng nhập:

```js
// TRƯỚC
router.put("/users/updateInfo/:myId/:userId", requireSignin, isAuth, update);

// SAU (bản sửa của bạn)
router.put("/users/updateInfo", requireSignin, updateInfo);
```

**Bước c — cập nhật frontend.** Hàm gọi API trong `yotea-fe/src/api/user.js` phải đổi URL
tương ứng (bỏ hai id trên đường dẫn) — id giờ do backend tự lấy từ token trong header.

> 🔒 **Vì sao giờ an toàn:** dù client có cố gửi `{"role": 1, "_id": "id-nguoi-khac"}`,
> backend **bỏ qua** `role` (không nằm trong whitelist) và **bỏ qua** `_id` client gửi (chỉ
> tin `req.auth._id` từ token đã ký). Lỗ hổng #1 bị bịt hoàn toàn.

---

## 9. 🛠️ Tự tay làm

> Mục tiêu: tự tay hoàn thành **mục 1** (`.env` backend), **mục 3** (`configureStore`) và
> **mục 4** (error handler), rồi kiểm chứng dự án **vẫn chạy y như trước** — refactor đúng
> nghĩa là "đổi bên trong, giữ hành vi bên ngoài".

### Bước 1 — Làm mục 1 và commit

Làm theo mục 2 của bài này (cài `dotenv`, tạo `.env`, sửa `app.js`, thay 2 chỗ secret ở
`checkAuth.js` và `auth.js`, tạo `.gitignore`). Chạy lại backend:

```bash
cd yotea-be
npm start
```

Terminal phải in `Connected to MongoDB` và `App is running on port: 8080` **y như cũ**. Nếu
in `MongooseError` với `undefined`, tức `process.env.MONGO_URI` chưa đọc được → kiểm tra
`import "dotenv/config"` đã ở **dòng đầu** `app.js` chưa. Xong thì:

```bash
git add -A && git commit -m "mục 1: .env backend + dotenv + .gitignore"
```

### Bước 2 — Làm mục 3 và mở Redux DevTools

Sửa `redux/store.js` theo mục 4. Chạy lại frontend (`npm start` tại `yotea-fe`), mở
extension Redux DevTools. Bấm thêm một món vào giỏ hàng → bạn phải thấy action
`cart/addCart` chạy qua và nhánh `cart` trong state đổi. Đó là bằng chứng store mới hoạt
động.

### Bước 3 — Làm mục 4 và thử token sai

Thêm khối error handler + 404 vào `app.js`. Test bằng Postman:

```
GET http://localhost:8080/api/duong-dan-khong-ton-tai
```

Phải nhận về **JSON** `{"message":"Không tìm thấy đường dẫn yêu cầu"}` với status 404 (chứ
không phải trang HTML). Rồi gọi một API cần đăng nhập với header
`Authorization: Bearer token-bay-ba` → phải nhận `401` + JSON
`{"message":"Token không hợp lệ hoặc đã hết hạn"}`.

---

## 10. ✅ Kiểm chứng kết quả

Sau cả bài, chạy lại **cả hai** đầu và đối chiếu checklist:

| Hạng mục | Cách kiểm | Đạt khi |
|---|---|---|
| 1. `.env` BE | `npm start` tại `yotea-be` | vẫn `Connected to MongoDB`, cổng 8080 |
| 2. `.env` FE | mở web, xem sản phẩm | danh sách sản phẩm hiện bình thường |
| 3. `configureStore` | mở Redux DevTools | thấy cây state, action chạy qua |
| 4. error handler | gọi URL sai bằng Postman | nhận JSON 404, không phải HTML |
| 5. `buildQuery` | `GET /api/users?_sort=fullName&_limit=2` | trả đúng 2 user, đã sắp xếp |
| 6. bcrypt | đăng ký user mới rồi đăng nhập | đăng nhập được; xem DB thấy hash bắt đầu bằng `$2a$` |
| 7. `updateInfo` | gửi `{"role":1}` vào `PUT /api/users/updateInfo` | user trả về vẫn `role: 0` |

> 💡 **Phép thử "hành vi không đổi":** trước khi refactor, ghi lại kết quả một vài API quen
> thuộc (danh sách sản phẩm, đăng nhập admin). Sau refactor, gọi lại đúng các API đó — kết
> quả phải **giống hệt**. Khác đi nghĩa là bạn đã vô tình đổi hành vi, không còn là refactor.

---

## 11. 🐞 Lỗi thường gặp

| Thông báo / hiện tượng | Nguyên nhân | Cách sửa |
|---|---|---|
| BE: `The "uri" ... must be of type string. Received undefined` | `process.env.MONGO_URI` rỗng | `import "dotenv/config"` chưa ở dòng đầu `app.js`, hoặc `.env` sai chỗ (phải ở gốc `yotea-be/`) |
| BE: mọi API auth báo `secret should be set` | `JWT_SECRET` `undefined` lúc `checkAuth.js` được nạp | như trên — dotenv phải nạp trước mọi import |
| FE: `process.env.REACT_APP_API_URL` là `undefined` | quên khởi động lại `npm start` sau khi tạo `.env` | Ctrl+C rồi `npm start` lại |
| FE: gọi API ra `undefined/products` | biến không có tiền tố `REACT_APP_` | đổi tên khoá trong `.env` cho đúng tiền tố |
| Redux: cảnh báo đỏ `non-serializable value` | thiếu `ignoredActions` của redux-persist | thêm đủ 6 action `FLUSH, REHYDRATE, ...` |
| RTK Query không refetch/cache | quên `.concat(...api.middleware)` | nối đủ 5 api middleware trong `configureStore` |
| Lỗi vẫn ra trang HTML stack trace | error handler viết thiếu tham số | phải đủ **4** tham số `(err, req, res, next)` và đặt **cuối cùng** |
| Đăng nhập bằng tài khoản cũ báo sai mật khẩu | user cũ còn hash SHA256, không hợp bcrypt | đăng ký lại, hoặc xoá user cũ trong DB |
| Ai gõ mật khẩu gì cũng vào được | quên `await` trước `user.authenticate()` | sửa `!(await user.authenticate(password))` |

---

## 12. 📝 Bài tập

**Bài 1.** Tạo file `yotea-be/.env.example` để đồng đội biết cần khai báo những khoá nào,
mà không lộ giá trị thật. File này **có** commit lên git. Nội dung nên như thế nào, và vì
sao nó không phá vỡ nguyên tắc "không commit bí mật"?

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

```bash
# yotea-be/.env.example  ← file này CÓ commit lên git
PORT=8080
MONGO_URI=mongodb://localhost:27017/ten-database-cua-ban
JWT_SECRET=chuoi-bi-mat-ngau-nhien-tu-sinh
```

Nó chỉ liệt kê **tên khoá** và giá trị mẫu/rỗng, **không** chứa bí mật thật. Đồng đội
`git clone` về, copy `.env.example` thành `.env`, rồi điền giá trị thật của máy họ. Đây là
quy ước chuẩn: `.env` (bí mật, không commit) + `.env.example` (khung, có commit).

</details>

**Bài 2.** Sau khi thêm error handler, một số `catch` trong controller vẫn tự trả
`res.status(400).json({ message, error })`. Với error handler tập trung, có cách viết
controller gọn hơn không? Viết lại hàm `create` của một controller theo hướng đó.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Thay vì tự bắt lỗi trong mỗi controller, ta có thể **đẩy lỗi ra error handler** bằng
`next(error)` (hoặc bọc bằng một wrapper `asyncHandler`). Ví dụ:

```js
// bản viết thêm của bạn — dồn mọi lỗi về error handler tập trung
export const create = async (req, res, next) => {
  try {
    const category = await new Category(req.body).save();
    res.json(category);
  } catch (error) {
    next(error); // ← chuyển cho error handler ở cuối app.js
  }
};
```

Lợi: thông báo lỗi và status được quyết định **một chỗ duy nhất** (error handler), không
còn rải rác `res.status(400)` mỗi nơi một kiểu. Nâng cao hơn nữa là viết hàm bọc
`const asyncHandler = (fn) => (req, res, next) => Promise.resolve(fn(req, res, next)).catch(next);`
rồi bỏ luôn `try/catch` khỏi controller.

</details>

**Bài 3.** (suy ngẫm) Sau khi đưa `JWT_SECRET` vào `.env`, giả sử bí mật cũ `"TuongVy"` đã
từng bị lộ. Việc đổi sang khoá mới có làm **các token cũ** (đã phát cho người đang đăng
nhập) hết hiệu lực không? Điều đó tốt hay xấu?

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

**Có** — mọi token đã ký bằng `"TuongVy"` sẽ **không verify được** bằng khoá mới, nên tất
cả phiên đăng nhập hiện tại bị "đá ra", buộc đăng nhập lại. Trong tình huống **khoá đã lộ**,
đây là điều **tốt và cần thiết**: nó vô hiệu hoá cả những token mà kẻ xấu có thể đã tự ký
bằng khoá lộ. Đánh đổi là chút bất tiện cho người dùng thật (phải đăng nhập lại một lần).
Đây chính là lý do khoá bí mật phải ở `.env` — để **xoay khoá** (rotate) được mà không phải
sửa code.

</details>

---

## 📌 Tóm tắt

- **Bí mật ra khỏi code:** `dotenv` + `.env` cho backend (`MONGO_URI`, `JWT_SECRET`, `PORT`),
  `REACT_APP_API_URL` cho frontend. `.gitignore` phải chặn `node_modules` và `.env`.
- **`node_modules` không commit** vì dựng lại được từ `package.json`; `.env` không commit vì
  chứa bí mật — thay bằng `.env.example`.
- **`configureStore`** thay `createStore` cũ: có sẵn `thunk`, kiểm tra dev, và **Redux
  DevTools** miễn phí; nhớ `.concat(...api.middleware)` và bỏ qua action của redux-persist.
- **Error handler 4 tham số + route 404** đặt cuối `app.js`: biến stack-trace HTML thành
  JSON gọn, `UnauthorizedError` trả 401 sạch sẽ.
- **`buildQuery()` dùng chung** xoá nạn copy-paste `list()` ở 13 controller — sửa một lần,
  đúng mọi nơi.
- **bcrypt** thay SHA256 (tự sinh salt, cố tình chậm) + hook kiểm `isModified("password")`;
  nhớ `await user.authenticate()`.
- **`updateInfo`** lấy id **từ token** (`req.auth._id`) và **whitelist** trường — bịt lỗ
  hổng tự phong admin.
- Refactor là **đổi bên trong, giữ hành vi bên ngoài**: làm trên nhánh git riêng, từng phần,
  test lại sau mỗi phần.

**Từ khoá tra cứu thêm:** `dotenv`, `12-factor app config`, `Redux Toolkit configureStore`,
`express error handling middleware`, `bcrypt salt rounds`, `mass assignment whitelist`,
`gitignore node_modules`

➡️ **Bài tiếp theo:** [35 — 🎓 Đồ án cuối khoá: làm chức năng Voucher end-to-end](35-do-an-cuoi-voucher.md) — gom mọi thứ đã học (kể cả nền `.env` sạch của bài này) để tự tay dựng một chức năng hoàn chỉnh từ model tới giao diện.
