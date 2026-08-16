  # Bài 18 — Tầng gọi API với axios

  > **Phần 2 · Frontend React** — Thời lượng ước tính: **~70 phút**
  > ⬅️ Bài trước: [17 — Tailwind CSS trong dự án](17-tailwind-css.md) · Bài sau: [19 — Redux Toolkit: slice, action, reducer, selector](19-redux-toolkit-co-ban.md) ➡️
  > 🏠 [Mục lục](README.md)

  ---

  Ở [Bài 17](17-tailwind-css.md) bạn đã trang trí xong `ToppingCard` bằng Tailwind — nhìn thì đẹp, nhưng dữ liệu vẫn là mảng "giả" bạn tự gõ tay trong `ToppingPage`. **Bài này ta làm tiếp việc quan trọng nhất: nối giao diện đó với API Topping mà chính bạn đã viết ở phần backend** ([Bài 07](07-crud-category.md) → [Bài 13](13-swagger-tai-lieu-api.md)).

  ## 🎯 Sau bài này bạn sẽ

  - Hiểu vì sao dự án tách hẳn thư mục `src/api/` thay vì gọi `axios` thẳng trong component.
  - Đọc hiểu `axios.create()` và `baseURL` — 7 dòng "nhỏ mà có võ" của Yotea.
  - Phân biệt được **axios** và **fetch**, biết vì sao dự án chọn axios.
  - Giải thích được vì sao mọi chỗ trong dự án đều viết `const { data } = await getAll()`.
  - Thuộc bộ quy ước `DB_NAME` / `getAll` / `get` / `getById` / `search` / `add` / `update` / `remove`.
  - Gắn được `Authorization: Bearer <token>` và hiểu chiêu `{ token, user } = isAuthenticate()`.
  - Mổ xẻ được `isAuthenticate()` — vì sao phải `JSON.parse` **hai lần**.
  - Chỉ ra 4 điểm yếu nghiêm trọng của tầng API này và biết cách làm chuẩn.
  - **Tự viết được `src/api/topping.js`** và gọi nó từ `ToppingPage`.

  ## 📋 Cần chuẩn bị

  - Đã hoàn thành [Bài 17](17-tailwind-css.md) — có sẵn `ToppingCard.js` và `ToppingPage.js`.
  - API Topping ở backend chạy được (thử Postman: `GET http://localhost:8080/api/toppings`).
  - Backend chạy cổng **8080**, frontend cổng **3000**.
  - Nhớ lại [Bài 03](03-kien-thuc-nen.md) phần *tham số mặc định* và *template literal*.

  ---

  ## 1. Vì sao phải có riêng thư mục `api/`?

  Hãy tưởng tượng bạn viết thẳng thế này trong component:

  ```jsx
  // ❌ Cách KHÔNG nên làm
  useEffect(() => {
    axios.get("http://localhost:8080/api/products/?_sort=createdAt&_order=desc")
      .then((res) => setProducts(res.data));
  }, []);
  ```

  Chạy được. Nhưng Yotea có **hơn 40 màn hình**. Nếu màn hình nào cũng tự gõ URL như vậy:

  | Vấn đề | Hậu quả thật |
  |---|---|
  | Deploy lên `https://api.yotea.vn` | Phải sửa **hàng chục file**, sót một chỗ là một màn hình trắng |
  | Cùng câu query lặp ở 5 màn hình | Sửa logic sắp xếp → phải nhớ sửa đủ 5 chỗ |
  | Gắn token | Copy-paste đoạn `headers` khắp nơi, dễ quên |
  | Viết test | Muốn test component phải giả lập cả mạng |

  Tách ra `src/api/` giải quyết cả bốn:

  ```
  Component / Redux slice     ← chỉ biết gọi hàm getAll(), add()
          ↓ import
    src/api/product.js       ← nơi DUY NHẤT biết URL và query string
          ↓ import
    src/api/instance.js      ← nơi DUY NHẤT biết địa chỉ server
          ↓
        axios  →  http://localhost:8080/api/...
  ```

  Ba lợi ích cụ thể:

  1. **Đổi `baseURL` một chỗ** — sửa `instance.js` là cả app đổi theo.
  2. **Tái sử dụng** — `getAll()` của `product.js` được dùng ở `ProductPage`, `HomeProducts` và cả `productSlice`: viết một lần, xài ba nơi.
  3. **Dễ test** — muốn test `ProductPage`, chỉ cần "giả" hàm `getAll` trả về mảng cứng, không cần server nào chạy.

  > 📖 **Thuật ngữ:** lớp này gọi là **data access layer** (tầng truy cập dữ liệu). Nguyên tắc: *component lo hiển thị, tầng api lo lấy dữ liệu, hai bên không lấn sân nhau.*

  ---

  ## 2. Soi code thật: `instance.js` — 7 dòng

  `yotea-fe/src/api/instance.js:1-7`

  ```js
  import axios from "axios";

  const instance = axios.create({
    baseURL: "http://localhost:8080/api",
  });

  export default instance;
  ```

  **Đọc từng dòng:**

  | Dòng | Code | Ý nghĩa |
  |---|---|---|
  | 1 | `import axios from "axios"` | Nạp thư viện axios (bản `^1.6.8` khai báo trong `package.json`) |
  | 3 | `const instance = axios.create({` | Tạo **bản sao axios có cấu hình riêng**, không đụng axios toàn cục |
  | 4 | `baseURL: "http://localhost:8080/api"` | Địa chỉ gốc — mọi URL sau này chỉ cần viết phần đuôi |
  | 7 | `export default instance` | Export mặc định → 13 file api khác viết `import instance from "./instance"` |

  `axios.create()` trả về một "axios con" mang sẵn cấu hình. Nhờ đó:

  ```js
  instance.get("/products");
  // axios tự ghép: http://localhost:8080/api + /products
  ```

  Vì `baseURL` **đã chứa `/api`**, bạn **không** viết `/api/products` — sẽ thành `/api/api/products` và nhận 404.

  > 💡 **Vì sao dùng `axios.create()` mà không set `axios.defaults`?** Cấu hình toàn cục sẽ áp lên **mọi** lời gọi, kể cả tới server khác. Dự án có một chỗ gọi thẳng `axios` lên Cloudinary (`yotea-fe/src/utils/index.js:7-10`) — dùng instance riêng là cách sạch để "mỗi server một instance".

  ---

  ## 3. axios và fetch khác nhau chỗ nào?

  `fetch` có sẵn trong trình duyệt, không cần cài. Vậy sao còn cài axios?

  | Tiêu chí | `fetch` (có sẵn) | `axios` (dự án dùng) |
  |---|---|---|
  | Tự parse JSON | ❌ Phải gọi `await res.json()` thêm bước | ✅ `res.data` đã là object |
  | Lỗi 4xx / 5xx | ❌ **Không** ném lỗi, phải tự kiểm `if (!res.ok)` | ✅ **Ném lỗi ngay**, rơi vào `catch` |
  | `baseURL` | ❌ Không có, phải tự nối chuỗi | ✅ `axios.create({ baseURL })` |
  | Interceptor | ❌ Không có | ✅ `instance.interceptors.request.use(...)` |
  | Gửi body JSON | Tự `JSON.stringify` + set `Content-Type` | Tự làm hết |
  | Kích thước | 0 KB | ~13 KB |

  Cái bẫy lớn nhất của `fetch`:

  ```js
  // ❌ fetch: server trả 404 nhưng KHÔNG rơi vào catch
  const res = await fetch("http://localhost:8080/api/khong-ton-tai");
  const data = await res.json();     // 💥 nổ ở đây, lỗi khó hiểu

  // ✅ axios: 404 rơi thẳng vào catch
  try {
    const { data } = await instance.get("/khong-ton-tai");
  } catch (e) {
    console.log(e.response.status);  // 404 — đúng như mong đợi
  }
  ```

  > 💡 **Chốt:** hai điểm khiến axios thắng là **tự parse JSON** và **ném lỗi khi 4xx/5xx** — chúng khiến `try...catch` hoạt động đúng trực giác.

  ---

  ## 4. Cấu trúc response — vì sao luôn viết `const { data } = ...`?

  Câu hỏi mà 100% người mới đều vấp. Khi gọi `const response = await instance.get("/products")`, `response` **không phải** mảng sản phẩm, mà là object mô tả **cả cuộc trao đổi HTTP**:

  ```js
  {
    data:       [ { _id: "...", name: "Trà sữa trân châu", price: 35000 }, ... ],  // ← thân phản hồi
    status:     200,
    statusText: "OK",
    headers:    { "content-type": "application/json; charset=utf-8", ... },
    config:     { url: "/products", method: "get", baseURL: "...", ... },
    request:    XMLHttpRequest { ... }
  }
  ```

  Dữ liệu thật nằm ở trường **`data`**. Nhờ destructuring ([Bài 03](03-kien-thuc-nen.md)) dự án viết gọn thành `const { data } = await getAll();`.

  Xem code thật `yotea-fe/src/pages/user/ProductByCate.js:13-20`:

  ```js
    useEffect(() => {
      const getIdCate = async () => {
        const { data } = await get(slug);
        setCateId(data._id);
        updateTitle(`${data.name}`);
      };
      getIdCate();
    }, [slug]);
  ```

  **Đọc từng dòng:**

  | Dòng | Code | Ý nghĩa |
  |---|---|---|
  | 13 | `useEffect(() => {` | Chạy sau khi component vẽ xong lần đầu |
  | 14 | `const getIdCate = async () => {` | Phải bọc hàm `async` riêng, vì callback của `useEffect` **không được** là `async` |
  | 15 | `const { data } = await get(slug)` | Gọi API, bóc lấy phần thân phản hồi |
  | 16 | `setCateId(data._id)` | Đưa vào state → React vẽ lại |
  | 18 | `getIdCate();` | **Gọi** hàm vừa định nghĩa (quên dòng này là không có gì xảy ra!) |
  | 20 | `}, [slug]);` | Chạy lại mỗi khi `slug` trên URL đổi |

  > ⚠️ **Ngoại lệ duy nhất:** `getAvgStar` (`yotea-fe/src/api/rating.js:34`) **không** trả về response mà trả về một con **số**. Chỗ gọi phải viết `const star = await getAvgStar(id)`; quen tay gõ `const { data } = ...` sẽ nhận `undefined`. Xem [Bài 30](30-binh-luan-danh-gia-yeu-thich.md).

  ---

  ## 5. Bộ quy ước tên lặp lại ở mọi file api

  Mở bất kỳ file nào trong `yotea-fe/src/api/` bạn sẽ thấy **cùng một khuôn**. Học thuộc khuôn này là đọc được cả 15 file.

  ### 5.1. Hằng số `DB_NAME`

  `yotea-fe/src/api/product.js:6`

  ```js
  const DB_NAME = "products";
  ```

  | File | `DB_NAME` | URL sinh ra |
  |---|---|---|
  | `product.js` | `"products"` | `/api/products` |
  | `category.js` | `"category"` | `/api/category` |
  | `comment.js` | `"comments"` | `/api/comments` |
  | `store.js` | `"store"` | `/api/store` |
  | `user.js` | `"users"` | `/api/users` |

  > ⚠️ **Chỗ này dự án đặt tên chưa chuẩn:** `DB_NAME` nghe như "tên database" nhưng thực ra là **tên tài nguyên trên URL** — đặt `RESOURCE` mới đúng nghĩa. Ngoài ra số ít/số nhiều lộn xộn (`products` nhưng `category`, `comments` nhưng `slider`) vì FE phải chạy theo route BE đã lỡ đặt.

  ### 5.2. Bộ hàm chuẩn

  | Tên hàm | Việc | HTTP | Dạng URL | Cần token? |
  |---|---|---|---|---|
  | `getAll(start, limit)` | Lấy danh sách | GET | `/{DB_NAME}/?_sort=...&_order=...` | Không |
  | `get(slug)` | Lấy 1 bản ghi | GET | `/{DB_NAME}/{slug}` | Không |
  | `getById(id)` | Lấy 1 bản ghi theo id | GET | `/{DB_NAME}/{id}` | Không |
  | `search(...)` | Tìm kiếm | GET | `?q=...` hoặc `?name_like=...` | Không |
  | `add(payload)` | Tạo mới | POST | `/{DB_NAME}/{user._id}` | **Có** |
  | `update(payload)` | Sửa | PUT | `/{DB_NAME}/{payload._id}/{user._id}` | **Có** |
  | `remove(id)` | Xoá | DELETE | `/{DB_NAME}/{id}/{user._id}` | **Có** |

  Nguyên tắc vàng: **đọc thì không cần token, ghi thì phải có token.**

  ### 5.3. Cách ghép query string

  `yotea-fe/src/api/category.js:7-16`

  ```js
  export const getAll = (start = 0, limit = 0) => {
    let url = `/${DB_NAME}/?_sort=createdAt&_order=desc`;
    if (limit) url += `&_start=${start}&_limit=${limit}`;
    return instance.get(url);
  };

  export const get = (slug) => {
    const url = `/${DB_NAME}/${slug}`;
    return instance.get(url);
  };
  ```

  **Đọc từng dòng:**

  | Dòng | Code | Ý nghĩa |
  |---|---|---|
  | 7 | `(start = 0, limit = 0)` | Tham số mặc định — gọi `getAll()` không truyền gì vẫn chạy |
  | 8 | `let url = ...` | Dùng `let` (không phải `const`) vì dòng dưới sẽ **nối thêm** |
  | 9 | `if (limit) url += ...` | **Chỉ khi** truyền `limit` mới nối phân trang; `limit = 0` là falsy → bỏ qua |
  | 10 | `return instance.get(url)` | Trả về **Promise**, không `await` ở đây — để nơi gọi tự `await` |
  | 13-16 | `get(slug)` | Không nối thêm gì → dùng `const` |

  Bản phức tạp hơn ở `yotea-fe/src/api/product.js:8-17` — thêm sắp xếp tuỳ biến và `_expand`:

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

  Gọi `getAll(0, 8)` sinh ra URL:

  ```
  http://localhost:8080/api/products/?_expand=categoryId&_sort=createdAt&_order=desc&_start=0&_limit=8
  ```

  `_sort`, `_order`, `_start`, `_limit`, `_expand` chính là **bộ lọc query** bạn đã tự cài ở backend trong [Bài 09](09-bo-loc-query.md). Frontend chỉ việc ghép chuỗi.

  Hàm tìm kiếm — `yotea-fe/src/api/product.js:37-47`:

  ```js
  export const search = (
    start = 0,
    limit = 0,
    sort = "createdAt",
    order = "desc",
    keyword
  ) => {
    let url = `/${DB_NAME}/?_sort=${sort}&_order=${order}&q=${keyword}`;
    if (limit) url += `&_start=${start}&_limit=${limit}`;
    return instance.get(url);
  };
  ```

  > ⚠️ **Hai chỗ dự án làm chưa chuẩn ở hàm này:**
  > 1. **Thứ tự tham số ngược đời:** `keyword` (bắt buộc) đứng **sau** 4 tham số có mặc định → người gọi **luôn phải** truyền đủ 5 đối số, giá trị mặc định thành vô dụng. Quy tắc: *tham số bắt buộc trước, tham số có mặc định sau.*
  > 2. **Không `encodeURIComponent(keyword)`:** gõ `trà & sữa` → URL thành `?q=trà & sữa&_sort=...`, dấu `&` cắt query làm đôi → `_sort` bị hiểu sai. Đúng phải là `` `&q=${encodeURIComponent(keyword)}` ``.

  Một biến thể tìm kiếm khác — `yotea-fe/src/api/store.js:12-15`:

  ```js
  export const search = (keyword) => {
    const url = `/${DB_NAME}/?name_like=${keyword}&_sort=createdAt&_order=desc`;
    return instance.get(url);
  };
  ```

  > ⚠️ Cùng tên `search` nhưng **hai cơ chế khác nhau**: `product.search` dùng `q=` (BE dịch thành `$text: { $search }`), `store.search` dùng `name_like=` (BE dịch thành regex). Hệ quả của việc không thống nhất quy ước từ đầu.

  ---

  ## 6. Gắn token: `Authorization: Bearer <token>`

  Với ba hàm ghi, backend đòi **tấm vé chứng minh đã đăng nhập** — chính là **JWT** bạn tạo ở [Bài 11](11-mat-khau-va-jwt.md), gửi lên qua header.

  `yotea-fe/src/api/product.js:59-66`

  ```js
  export const add = (product, { token, user } = isAuthenticate()) => {
    const url = `/${DB_NAME}/${user._id}`;
    return instance.post(url, product, {
      headers: {
        Authorization: `Bearer ${token}`,
      },
    });
  };
  ```

  **Đọc từng dòng:**

  | Dòng | Code | Ý nghĩa |
  |---|---|---|
  | 59 | `(product, { token, user } = isAuthenticate())` | Tham số 1 là dữ liệu gửi lên. Tham số 2: nếu người gọi **không truyền**, tự chạy `isAuthenticate()` rồi bóc `token` và `user` |
  | 60 | `` `/${DB_NAME}/${user._id}` `` | Nhét id người đăng nhập vào **cuối đường dẫn** — khớp route BE `router.post("/products/:userId", ...)` |
  | 61 | `instance.post(url, product, {...})` | `post` nhận **3 đối số**: URL, **body**, rồi mới tới config |
  | 63 | `` Authorization: `Bearer ${token}` `` | Chữ `Bearer` + **một dấu cách** + token; sai một ly là 401 |

  Bản `remove` — `yotea-fe/src/api/product.js:68-75`:

  ```js
  export const remove = (id, { token, user } = isAuthenticate()) => {
    const url = `/${DB_NAME}/${id}/${user._id}`;
    return instance.delete(url, {
      headers: {
        Authorization: `Bearer ${token}`,
      },
    });
  };
  ```

  Khác biệt duy nhất: `delete` **không có body**, nên config là **đối số thứ 2**. Đây là bẫy hay gặp — nhầm chỗ thì header không được gửi, server trả 401.

  | Method | Chữ ký axios | Config nằm ở |
  |---|---|---|
  | `instance.get(url, config)` / `instance.delete(url, config)` | 2 đối số | thứ **2** |
  | `instance.post/put/patch(url, body, config)` | 3 đối số | thứ **3** |

  ### 6.1. Vì sao viết `{ token, user } = isAuthenticate()` mà không gọi ở đầu file?

  Ba đặc tính của **tham số mặc định** giải thích tất cả:

  1. **Biểu thức mặc định được tính lại ở MỖI LẦN GỌI**, không phải một lần lúc import. Nếu viết `const { token } = isAuthenticate();` ở đầu file, nó chạy ngay lúc app khởi động — lúc đó chưa ai đăng nhập → **crash trắng màn hình**.
  2. Nhờ vậy token luôn là **token mới nhất**: đăng nhập tài khoản khác thì lần gọi sau tự lấy token mới.
  3. **Mặc định chỉ kích hoạt khi đối số là `undefined`.** `add(data)` ✅, nhưng `add(data, null)` ❌ → `TypeError: Cannot destructure property 'token' of 'null'`.

  > 🔒 **Ghi chú bảo mật:** `user._id` bị nhét lên URL ở **cả 3 hàm ghi** → id người dùng xuất hiện trong access log của server, proxy, CDN. Cách chuẩn là backend tự đọc id **từ token** (token đã chứa sẵn `_id`). Chi tiết ở [Bài 33](33-ra-soat-bao-mat.md).

  ---

  ## 7. Mổ xẻ `utils/localStorage.js` — hàm 3 dòng khó nhất dự án

  `yotea-fe/src/utils/localStorage.js:1-4`

  ```js
  export const isAuthenticate = () => {
    return JSON.parse(JSON.parse(localStorage.getItem("persist:root")).auth)
      .value;
  };
  ```

  Ba dòng, nhưng có tới **hai** lần `JSON.parse`. Ta bóc từng bước:

  | Bước | Biểu thức | Kết quả |
  |---|---|---|
  | 1 | `localStorage.getItem("persist:root")` | Một **chuỗi**: `{"auth":"{\"isLogged\":true,\"value\":{...}}","cart":"[]","_persist":"..."}`. Đám `\"` là dấu hiệu **chuỗi lồng trong chuỗi** |
  | 2 | `JSON.parse(...)` lần 1 | Object `{ auth: '...', cart: '[]', _persist: '...' }` — nhưng mỗi giá trị **vẫn là chuỗi** |
  | 3 | `.auth` | Lấy trường `auth` ra, **vẫn đang là chuỗi JSON**, chưa dùng được |
  | 4 | `JSON.parse(...)` lần 2 | Object state thật: `{ isLogged: true, value: { token: "eyJhbGci...", user: {...} } }` |
  | 5 | `.value` | Đúng thứ ta cần: `{ token, user }` |

  Đối chiếu với `yotea-fe/src/redux/authSlice.js:4-7`:

  ```js
  const initialState = {
    isLogged: false,
    value: {},
  };
  ```

  Đúng vậy — `value` chính là nơi `authSlice` cất `{ token, user }` sau khi đăng nhập thành công.

  > 💡 Muốn nhìn tận mắt: DevTools → tab **Application** → **Local Storage** → `http://localhost:3000` → khoá `persist:root`.

  ### 7.1. Vì sao phải parse HAI lần?

  Vì **redux-persist serialize từng reducer một cách độc lập** — nó không nén cả cây state thành một chuỗi, mà làm hai tầng:

  ```
  state Redux              redux-persist ghi xuống localStorage
  {                        {
    auth: { ... },   →       "auth": "<chuỗi JSON của auth>",
    cart: [ ... ],   →       "cart": "<chuỗi JSON của cart>"
  }                        }  ← rồi đóng gói cả object này thành 1 chuỗi nữa
  ```

  Thiết kế này có lý do: khi chỉ `cart` đổi, redux-persist chỉ serialize lại `cart`, không đụng `auth` — nhanh hơn nhiều. Cái giá là bạn **đọc tay** thì phải bóc hai tầng.

  Danh sách reducer được lưu nằm ở `yotea-fe/src/redux/store.js:13-17`:

  ```js
  const persistConfig = {
    key: "root",
    storage,
    whitelist: ["auth", "cart"],
  };
  ```

  `key: "root"` giải thích tên khoá `persist:root`; `whitelist` giải thích vì sao trong đó chỉ có hai trường. Chi tiết ở [Bài 21](21-redux-persist.md).

  ---

  ## 8. ⚠️ Bốn vấn đề nghiêm trọng của tầng API này

  > ⚠️ Bốn mục dưới đây đều đã kiểm chứng bằng cách đọc source. Bạn **không sửa** code dự án ở bài này — chỉ cần hiểu.

  ### 8.1. `isAuthenticate()` ném `TypeError` khi chưa đăng nhập

  Đọc lại hàm: không hề có một câu `if` nào kiểm tra dữ liệu tồn tại.

  | Tình huống | Chuyện gì xảy ra | Lỗi ném ra |
  |---|---|---|
  | Lần đầu vào web, redux-persist chưa kịp ghi | `getItem` trả `null` → `JSON.parse(null)` = `null` → `null.auth` | `TypeError: Cannot read properties of null (reading 'auth')` |
  | Đã vào web nhưng chưa login | `value` là `{}` → `user` là `undefined` → `user._id` | `TypeError: Cannot read properties of undefined (reading '_id')` |
  | Vừa bấm đăng xuất (`logout` set `value = {}`) | Y như trên | Y như trên |

  Điểm chết người: **đây là lỗi JavaScript đồng bộ, không phải lỗi HTTP 401.** Nó nổ ngay tại chữ ký hàm / dòng dựng URL — **trước cả khi request được gửi đi**. Hệ quả: `catch` báo chung chung không phân biệt được "chưa đăng nhập" với "server chết"; không có cơ chế chuyển hướng về `/login`; gọi ngoài `try/catch` → **màn hình trắng** (dự án không có Error Boundary).

  Cách viết đúng — *đoạn này bạn tự viết thêm, dự án chưa có, chỉ để tham khảo:*

  ```js
  export const isAuthenticate = () => {
    try {
      const root = localStorage.getItem("persist:root");
      if (!root) return { token: null, user: null };

      const auth = JSON.parse(JSON.parse(root).auth || "{}");
      const { token = null, user = null } = auth.value || {};
      return { token, user };
    } catch {
      return { token: null, user: null };
    }
  };
  ```

  > ⚠️ Còn một lỗi đặt tên: `isAuthenticate` nghe như trả về `true`/`false`, nhưng thực ra trả về **object** `{ token, user }`. Tên đúng phải là `getAuth()`. **Tên hàm nói dối là nguồn bug lớn nhất trong một codebase.**

  ### 8.2. `http://localhost:8080/api` bị hardcode ở **6 chỗ**

  Grep toàn bộ `yotea-fe/src` với từ khoá `http://localhost:8080` cho đúng **6 kết quả**:

  | # | Vị trí | Ngữ cảnh |
  |---|---|---|
  | 1 | `yotea-fe/src/api/instance.js:4` | `baseURL` của axios instance — phục vụ ~95% request |
  | 2 | `yotea-fe/src/api/category.js:48` | `fetchBaseQuery({ baseUrl })` của `cateProductApi` |
  | 3 | `yotea-fe/src/api/news.js:60` | `fetchBaseQuery({ baseUrl })` của `newsApi` |
  | 4 | `yotea-fe/src/api/product.js:99` | `fetchBaseQuery({ baseUrl })` của `productApi` |
  | 5 | `yotea-fe/src/api/slider.js:47` | `fetchBaseQuery({ baseUrl })` của `sliderApi` |
  | 6 | `yotea-fe/src/api/user.js:57` | `fetchBaseQuery({ baseUrl })` của `userApi` |

  Muốn deploy phải sửa tay **6 file**; sót một file là một phần trang chết lặng lẽ.

  Chi tiết dễ nhầm: `instance.js` viết `baseURL` (hoa cả ba chữ — cú pháp axios), 5 chỗ RTK Query viết `baseUrl` (chỉ hoa chữ **U** — cú pháp `fetchBaseQuery`). Gõ nhầm hoa/thường giữa hai thư viện **không hề báo lỗi**, chỉ khiến request bay sai địa chỉ. Gặp lại RTK Query ở [Bài 22](22-rtk-query.md).

  ### 8.3. Không có interceptor → không xử lý tập trung được token hết hạn

  `instance.js` chỉ 7 dòng, **không đăng ký interceptor nào**. Ba hệ quả:

  1. Đoạn `headers: { Authorization: ... }` bị **copy-paste 34 lần** khắp `api/`. Đổi cách gắn token → sửa 34 chỗ.
  2. Khi token hết hạn backend trả **401**, mỗi component tự `catch` kiểu riêng → người dùng kẹt ở màn hình lỗi mà không biết phải đăng nhập lại.
  3. Không refresh token, không retry, không có nơi log lỗi tập trung.

  ### 8.4. Cách làm chuẩn: biến môi trường + interceptor

  **Bước 1 — đưa địa chỉ server ra `.env`.** Create React App hỗ trợ sẵn: biến bắt đầu bằng `REACT_APP_` được nhúng vào bundle lúc build.

  ```bash
  # yotea-fe/.env  ← file MỚI, dự án chưa có
  REACT_APP_API_URL=http://localhost:8080/api
  ```

  **Bước 2 — viết lại `instance.js` có interceptor.** *Đoạn dưới bạn tự viết thêm, dự án chưa có — chỉ để tham khảo, đừng sửa file thật:*

  ```js
  import axios from "axios";
  import { isAuthenticate } from "../utils/localStorage";

  const instance = axios.create({
    baseURL: process.env.REACT_APP_API_URL || "http://localhost:8080/api",
    timeout: 10000,
  });

  // 1. Interceptor REQUEST — chạy TRƯỚC khi mỗi request bay đi
  instance.interceptors.request.use(
    (config) => {
      const { token } = isAuthenticate();
      if (token) {
        config.headers.Authorization = `Bearer ${token}`;  // gắn token MỘT chỗ duy nhất
      }
      return config;                                       // BẮT BUỘC phải return
    },
    (error) => Promise.reject(error)
  );

  // 2. Interceptor RESPONSE — chạy SAU khi nhận phản hồi
  instance.interceptors.response.use(
    (response) => response,                                // 2xx: cho đi tiếp
    (error) => {
      const status = error.response?.status;

      if (status === 401) {
        localStorage.removeItem("persist:root");           // token hỏng/hết hạn → dọn sạch
        window.location.href = "/login";                   // đá về trang đăng nhập
      }
      if (status === 403) window.location.href = "/";      // không đủ quyền

      return Promise.reject(error);                        // vẫn ném tiếp để component biết
    }
  );

  export default instance;
  ```

  Có interceptor rồi, mọi hàm ghi gọn lại còn một dòng:

  ```js
  export const add = (product) => instance.post(`/${DB_NAME}`, product);
  ```

  > 💡 **Ví von:** interceptor giống **anh bảo vệ ở cổng công ty** — ai đi ra cũng được dán thẻ (request interceptor), ai vào mà giấy tờ sai thì bị chặn và hướng dẫn về quầy lễ tân (response interceptor). Bạn không phải dặn từng người một.

  Ta sẽ refactor thật ở [Bài 34](34-refactor-du-an.md).

  ---

  ## 9. 🛠️ Tự tay làm — viết `src/api/topping.js`

  > Mục tiêu: cuối phần này bạn có file `src/api/topping.js` đúng chuẩn dự án, và `ToppingPage` hiển thị **dữ liệu Topping thật** từ MongoDB thay vì mảng giả.

  ### Bước 1 — Kiểm tra backend còn sống

  ```bash
  # đứng tại thư mục yotea-be
  npm start
  ```

  Mở Postman gọi `GET http://localhost:8080/api/toppings` → phải nhận được mảng:

  ```json
  [
    { "_id": "6650bb01c4e8b91234abcd11", "name": "Trân châu đen", "price": 5000, "slug": "tran-chau-den" },
    { "_id": "6650bb01c4e8b91234abcd12", "name": "Thạch dừa",     "price": 7000, "slug": "thach-dua" }
  ]
  ```

  Nếu chưa có dữ liệu, gọi `POST /api/toppings/:userId` thêm vài món trước.

  ### Bước 2 — Tạo file `yotea-fe/src/api/topping.js`

  *Toàn bộ file này bạn tự viết mới, dự án chưa có.* Bám **đúng khuôn** của `category.js` và `store.js` vừa mổ xẻ.

  ```js
  // yotea-fe/src/api/topping.js  ← file MỚI, bạn tự tạo
  import { isAuthenticate } from "../utils/localStorage";
  import instance from "./instance";

  const DB_NAME = "toppings";

  // ---- Nhóm ĐỌC: không cần token ----

  export const getAll = (start = 0, limit = 0) => {
    let url = `/${DB_NAME}/?_sort=createdAt&_order=desc`;
    if (limit) url += `&_start=${start}&_limit=${limit}`;
    return instance.get(url);
  };

  export const get = (slug) => {
    const url = `/${DB_NAME}/${slug}`;
    return instance.get(url);
  };

  // ---- Nhóm GHI: bắt buộc gắn token ----

  export const add = (topping, { token, user } = isAuthenticate()) => {
    const url = `/${DB_NAME}/${user._id}`;
    return instance.post(url, topping, {
      headers: {
        Authorization: `Bearer ${token}`,
      },
    });
  };

  export const update = (topping, { token, user } = isAuthenticate()) => {
    const url = `/${DB_NAME}/${topping._id}/${user._id}`;
    return instance.put(url, topping, {
      headers: {
        Authorization: `Bearer ${token}`,
      },
    });
  };

  export const remove = (id, { token, user } = isAuthenticate()) => {
    const url = `/${DB_NAME}/${id}/${user._id}`;
    return instance.delete(url, {
      headers: {
        Authorization: `Bearer ${token}`,
      },
    });
  };
  ```

  **Ba chỗ phải kiểm tra kỹ:**

  | Kiểm tra | Vì sao |
  |---|---|
  | `DB_NAME` khớp route backend | Nếu ở phần backend bạn đặt route `/topping` (số ít) thì sửa `DB_NAME = "topping"` |
  | `post`/`put` có **3** đối số, `delete` có **2** | Đặt config sai chỗ → header không được gửi → 401 |
  | `update` dùng `topping._id`, `remove` dùng `id` | `update` nhận cả object, `remove` chỉ nhận id |

  ### Bước 3 — Gọi `getAll` trong `ToppingPage`

  Mở `yotea-fe/src/pages/user/ToppingPage.js` (tạo ở [Bài 16](16-layout-va-component.md)), xoá mảng giả, thay bằng `useEffect` + `useState`.

  ```jsx
  // yotea-fe/src/pages/user/ToppingPage.js  ← file BẠN đã tạo ở bài 16, giờ sửa lại
  import { useEffect, useState } from "react";
  import { getAll } from "../../api/topping";
  import ToppingCard from "../../components/user/ToppingCard";
  import { updateTitle } from "../../utils";

  const ToppingPage = () => {
    const [toppings, setToppings] = useState([]);   // mảng rỗng, KHÔNG để undefined
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);

    useEffect(() => {
      updateTitle("Topping");

      const fetchToppings = async () => {
        try {
          const { data } = await getAll();          // bóc trường data ra khỏi response
          setToppings(data);
        } catch (err) {
          setError("Không tải được danh sách topping");
          console.log(err);
        } finally {
          setLoading(false);
        }
      };

      fetchToppings();
    }, []);                                          // [] → chỉ chạy MỘT lần khi vào trang

    if (loading) return <p className="text-center py-10">Đang tải...</p>;
    if (error) return <p className="text-center py-10 text-red-500">{error}</p>;

    return (
      <section className="container max-w-6xl mx-auto px-3 py-8">
        <h1 className="text-[#D9A953] font-semibold text-3xl text-center mb-6">Topping</h1>

        <div className="grid grid-cols-12 gap-6">
          {toppings.map((topping) => (
            <ToppingCard key={topping._id} topping={topping} />
          ))}
        </div>
      </section>
    );
  };

  export default ToppingPage;
  ```

  **Bốn điểm cần nhớ:**

  | Điểm | Giải thích |
  |---|---|
  | `useState([])` | Khởi tạo **mảng rỗng**; để `useState()` thì lần vẽ đầu `toppings` là `undefined` → `.map` nổ ngay |
  | Hàm `async` **bên trong** `useEffect` | Callback của `useEffect` không được là `async` (React cần nó trả về hàm dọn dẹp, không phải Promise) |
  | `finally { setLoading(false) }` | Tắt loading dù thành công hay thất bại |
  | `key={topping._id}` | React cần key duy nhất cho mỗi phần tử danh sách |

  ---

  ## 10. ✅ Kiểm chứng kết quả

  ```bash
  # terminal 1 — đứng tại thư mục yotea-be
  npm start

  # terminal 2 — đứng tại thư mục yotea-fe
  npm start
  ```

  Mở `http://localhost:3000/topping` (route bạn thêm ở [Bài 15](15-routing-v6.md)). Bạn sẽ thấy chữ "Đang tải...", rồi lưới các thẻ `ToppingCard` với đúng những topping đã tạo bằng Postman.

  **Kiểm chứng bằng DevTools — bước quan trọng nhất của bài:**

  1. `F12` → tab **Network** → lọc **Fetch/XHR** → nhấn `F5`.
  2. Phải thấy đúng **một** dòng request:

  | Cột | Giá trị mong đợi |
  |---|---|
  | Name | `toppings/?_sort=createdAt&_order=desc` |
  | Status | `200` |
  | Method | `GET` |
  | Type | `xhr` |

  3. Bấm vào dòng đó → tab **Headers** → **Request URL** phải đúng:

  ```
  http://localhost:8080/api/toppings/?_sort=createdAt&_order=desc
  ```

  4. Sang tab **Response** → phải là mảng JSON:

  ```json
  [
    { "_id": "6650bb01c4e8b91234abcd11", "name": "Trân châu đen", "price": 5000, "slug": "tran-chau-den" },
    { "_id": "6650bb01c4e8b91234abcd12", "name": "Thạch dừa",     "price": 7000, "slug": "thach-dua" }
  ]
  ```

  5. **Đối chiếu:** số phần tử phải **đúng bằng** số topping bạn tạo ở phần backend, và thứ tự là **mới nhất trước** (vì `_sort=createdAt&_order=desc`).

  > 💡 Nếu tab Network không thấy request nào bay đi, gần như chắc chắn bạn quên **gọi** hàm `fetchToppings()` sau khi định nghĩa nó.

  ---

  ## 11. 🐞 Lỗi thường gặp

  | Thông báo lỗi | Nguyên nhân | Cách sửa |
  |---|---|---|
  | `Network Error` | Backend chưa chạy | `cd yotea-be` rồi `npm start` |
  | `has been blocked by CORS policy` | Backend chưa bật CORS cho cổng 3000 | Kiểm tra `app.use(cors())` — xem [Bài 04](04-express-va-appjs.md) |
  | Request bay tới `/api/api/toppings` | Viết `instance.get("/api/toppings")` | Bỏ `/api` đi, `baseURL` đã có sẵn |
  | `404 Not Found` | `DB_NAME` không khớp route backend | So lại với `yotea-be/src/routes/topping.js` |
  | `toppings.map is not a function` | `data` không phải mảng | `console.log(data)` xem thật sự nhận được gì |
  | `Cannot read properties of undefined (reading 'map')` | Khởi tạo `useState()` thay vì `useState([])` | Luôn khởi tạo đúng kiểu dữ liệu |
  | `Cannot read properties of null (reading 'auth')` | Gọi `add`/`update`/`remove` khi **chưa đăng nhập** | Đăng nhập trước — đúng lỗi mục 8.1 |
  | `401 Unauthorized` khi gọi `add` | Thiếu header, hoặc đặt config sai vị trí ở `delete` | Xem bảng "config nằm ở đối số thứ mấy" mục 6 |
  | `403` / `Bạn không có quyền truy cập` | Tài khoản không phải admin | Đăng nhập bằng tài khoản `role: 1` |
  | Response 200 nhưng mảng rỗng `[]` | Database chưa có topping nào | Tạo vài bản ghi bằng Postman |
  | Request bay **hai lần** | `React.StrictMode` cố tình chạy effect 2 lần ở môi trường dev | Bình thường; bản production chỉ chạy 1 lần |

  ---

  ## 12. 📝 Bài tập

  **Bài 1.** Viết thêm hàm `search(keyword)` vào `src/api/topping.js`, tìm topping theo tên (dùng `name_like` giống `store.js`). Nhớ tránh cái bẫy `encodeURIComponent` ở mục 5.3.

  <details>
  <summary>💡 Xem gợi ý & lời giải</summary>

  *Đoạn này bạn tự viết thêm, dự án chưa có:*

  ```js
  // yotea-fe/src/api/topping.js  ← thêm vào cuối file
  export const search = (keyword) => {
    const url = `/${DB_NAME}/?name_like=${encodeURIComponent(
      keyword
    )}&_sort=createdAt&_order=desc`;
    return instance.get(url);
  };
  ```

  So với `yotea-fe/src/api/store.js:12-15`, bản của bạn **tốt hơn** nhờ `encodeURIComponent`. Thử `search("trân châu & thạch")`:

  - Bản dự án → `?name_like=trân châu & thạch&_sort=...` → dấu `&` cắt query, `_sort` mất.
  - Bản của bạn → `?name_like=tr%C3%A2n%20ch%C3%A2u%20%26%20th%E1%BA%A1ch&_sort=...` ✅

  Điều kiện để chạy: backend phải xử lý được `name_like` — nếu ở [Bài 09](09-bo-loc-query.md) bạn đã cài bộ lọc query cho controller topping thì đã có sẵn.

  </details>

  **Bài 2.** Đoạn code sau bị **lỗi**, chạy sẽ trả về 401. Chỉ ra lỗi và sửa lại.

  ```js
  export const remove = (id, { token, user } = isAuthenticate()) => {
    const url = `/${DB_NAME}/${id}/${user._id}`;
    return instance.delete(url, {}, {
      headers: {
        Authorization: `Bearer ${token}`,
      },
    });
  };
  ```

  <details>
  <summary>💡 Xem gợi ý & lời giải</summary>

  **Lỗi:** người viết tưởng `delete` cũng có 3 đối số như `post`, nên chèn thêm `{}` làm "body". Thực tế `instance.delete(url, config)` chỉ nhận **2** đối số → `{}` bị axios hiểu là **config**, còn object chứa `headers` thật bị **bỏ qua hoàn toàn**. Request bay đi **không có `Authorization`** → 401.

  Điểm nguy hiểm: axios **không hề báo lỗi**, code vẫn chạy, chỉ thiếu header. Kiểu bug tốn cả buổi để tìm.

  **Sửa** — bỏ `{}` đi, giống hệt `yotea-fe/src/api/product.js:68-75`:

  ```js
  export const remove = (id, { token, user } = isAuthenticate()) => {
    const url = `/${DB_NAME}/${id}/${user._id}`;
    return instance.delete(url, {
      headers: {
        Authorization: `Bearer ${token}`,
      },
    });
  };
  ```

  **Kiểm chứng:** DevTools → Network → bấm vào request → tab **Headers** → mục **Request Headers** phải thấy dòng `Authorization: Bearer eyJhbGci...`.

  </details>

  **Bài 3.** Với người dùng **chưa đăng nhập**, dòng nào trong hàm `add` (`product.js:59-66`) nổ **đầu tiên**, lỗi cụ thể là gì, và vì sao khối `try/catch` bọc quanh lời gọi API **không** phân biệt được lỗi này với lỗi mất mạng?

  <details>
  <summary>💡 Xem gợi ý & lời giải</summary>

  **Nổ ở dòng 59 — ngay tại chữ ký hàm**, chưa tới dòng dựng URL. Vì tham số mặc định được tính **trước** khi thân hàm chạy: `isAuthenticate()` gọi `JSON.parse(localStorage.getItem("persist:root"))`; nếu khoá chưa tồn tại thì `JSON.parse(null)` cho ra `null`, rồi `null.auth` ném:

  ```
  TypeError: Cannot read properties of null (reading 'auth')
  ```

  Nếu đã vào web nhưng chưa login (`value = {}`), `isAuthenticate()` trả `{}` → `user = undefined` → lúc này **dòng 60** mới nổ với `TypeError: Cannot read properties of undefined (reading '_id')`.

  **Vì sao `try/catch` không phân biệt được?** Vì cả ba nguyên nhân rơi vào cùng một `catch`:

  - Chưa đăng nhập → `TypeError` (lỗi JS đồng bộ, **request chưa hề được gửi**).
  - Mất mạng → `AxiosError` với `error.response === undefined`.
  - Token hết hạn → `AxiosError` với `error.response.status === 401`.

  Ba cách xử lý khác nhau (đá về `/login` / bảo thử lại / đăng nhập lại), nhưng người dùng chỉ nhận đúng một câu `"Có lỗi xảy ra"` vô nghĩa. Cách phân biệt đúng:

  ```js
  catch (error) {
    if (!error.response) toast.error("Không kết nối được máy chủ");
    else if (error.response.status === 401) navigate("/login");
    else toast.error(error.response.data?.message || "Có lỗi xảy ra");
  }
  ```

  Nhưng gốc rễ vẫn là hai thứ ở mục 8.4: **sửa `isAuthenticate` để trả `null` thay vì ném lỗi**, và **thêm response interceptor** để 401 được xử lý một chỗ.

  </details>

  **Bài 4.** Ngày mai bạn deploy Yotea, backend chuyển sang `https://api.yotea.vn/api`. Liệt kê **chính xác** những file phải sửa với code hiện tại, rồi viết lại `instance.js` dùng biến môi trường để lần sau chỉ sửa **một** chỗ.

  <details>
  <summary>💡 Xem gợi ý & lời giải</summary>

  **Với code hiện tại phải sửa 6 file** (grep `http://localhost:8080` cho đúng 6 kết quả): `api/instance.js:4`, `api/category.js:48`, `api/news.js:60`, `api/product.js:99`, `api/slider.js:47`, `api/user.js:57`.

  **Cách làm chuẩn** — *đoạn này bạn tự viết thêm, dự án chưa có:*

  ```bash
  # yotea-fe/.env.development
  REACT_APP_API_URL=http://localhost:8080/api

  # yotea-fe/.env.production
  REACT_APP_API_URL=https://api.yotea.vn/api
  ```

  ```js
  // yotea-fe/src/api/instance.js
  import axios from "axios";

  export const API_URL = process.env.REACT_APP_API_URL || "http://localhost:8080/api";

  const instance = axios.create({ baseURL: API_URL });

  export default instance;
  ```

  Rồi 5 file RTK Query đổi thành `baseQuery: fetchBaseQuery({ baseUrl: API_URL })` (nhớ `import { API_URL } from "./instance";`).

  Ba lưu ý:

  1. Biến **bắt buộc** có tiền tố `REACT_APP_` (quy ước của CRA), nếu không sẽ là `undefined`.
  2. Phải **khởi động lại** `npm start` sau khi sửa `.env` — CRA chỉ đọc lúc khởi động.
  3. `.env` được nhúng thẳng vào bundle JavaScript ai cũng đọc được → **tuyệt đối không** để secret key, mật khẩu database trong đó.

  Ta sẽ làm thật việc này ở [Bài 34](34-refactor-du-an.md).

  </details>

  ---

  ## 📌 Tóm tắt

  - Tách riêng `src/api/` cho ba lợi ích: **đổi `baseURL` một chỗ**, **tái sử dụng**, **dễ test**.
  - `axios.create({ baseURL })` tạo một axios riêng cho server Yotea — chỉ 7 dòng trong `yotea-fe/src/api/instance.js`.
  - axios hơn `fetch` ở ba điểm quyết định: **tự parse JSON**, **ném lỗi khi 4xx/5xx**, **có interceptor**.
  - Response của axios là object `{ data, status, headers, config, request }` — dữ liệu thật nằm ở `data`, nên dự án luôn viết `const { data } = await getAll()`.
  - Quy ước xuyên suốt: `DB_NAME` + bộ hàm `getAll` / `get` / `getById` / `search` / `add` / `update` / `remove`. **Đọc không cần token, ghi phải có token.**
  - Ba hàm ghi gắn header `Authorization: Bearer <token>` và dùng chiêu tham số mặc định `{ token, user } = isAuthenticate()` — biểu thức này được tính **lại ở mỗi lần gọi**.
  - `isAuthenticate()` phải `JSON.parse` **hai lần** vì redux-persist serialize **từng reducer** thành một chuỗi JSON riêng, rồi mới gói cả object thành một chuỗi nữa.
  - Bốn điểm yếu cần nhớ: **`isAuthenticate()` ném `TypeError` khi chưa đăng nhập**, **6 chỗ hardcode `http://localhost:8080/api`**, **không có interceptor** nên không xử lý tập trung được token hết hạn, và **34 chỗ copy-paste header**. Cách chuẩn: `REACT_APP_API_URL` + axios interceptor.

  **Từ khoá tra cứu thêm:** `axios create instance`, `axios interceptors`, `axios vs fetch`, `Bearer token authorization header`, `default parameters javascript`, `redux-persist storage format`, `REACT_APP environment variables`, `useEffect fetch data`

  ➡️ **Bài tiếp theo:** [19 — Redux Toolkit: slice, action, reducer, selector](19-redux-toolkit-co-ban.md) — `useState` trong `ToppingPage` chỉ dùng được cho **một** component; bài sau ta đưa danh sách topping lên **kho chung** để mọi màn hình cùng đọc được, bằng cách tự viết `toppingSlice.js`.
