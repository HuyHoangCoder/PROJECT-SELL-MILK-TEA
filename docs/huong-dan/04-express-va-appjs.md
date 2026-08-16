  # Bài 04 — Mổ xẻ `app.js`: Express, middleware, CORS, morgan

  > **Phần 1 · Backend** — Thời lượng ước tính: **~75 phút**
  > ⬅️ Bài trước: [03 — Kiến thức nền tối thiểu](03-kien-thuc-nen.md) · Bài sau: [05 — Mongoose Model: Schema, kiểu dữ liệu, timestamps, index](05-mongoose-model.md) ➡️
  > 🏠 [Mục lục](README.md)

  ---

  ## 🎯 Sau bài này bạn sẽ

  - Nói được **Express là gì** và vì sao không ai viết server bằng module `http` thuần.
  - Hiểu **middleware**: chữ ký `(req, res, next)`, vai trò của `next()`, và chuyện gì xảy ra khi quên gọi nó.
  - Đọc hiểu **từng dòng** của `yotea-be/src/app.js` — file khởi động toàn bộ backend.
  - Giải thích được vì sao **thứ tự** khai báo middleware quyết định sống chết của `req.body`.
  - Hiểu **CORS** và Same-Origin Policy: vì sao `localhost:3000` gọi `localhost:8080` bị trình duyệt chặn.
  - Đọc được log của **morgan**, và biết `npm start` thật ra chạy cái gì.
  - Tự tay viết được middleware đầu tiên và một route `GET /api/health`.

  ## 📋 Cần chuẩn bị

  - Đã chạy được backend ở [Bài 02](02-cai-dat-moi-truong.md) (thấy dòng `App is running on port: 8080`).
  - Đã đọc [Bài 03](03-kien-thuc-nen.md) — phần `import/export`, arrow function, HTTP method, status code.
  - Có Postman (trình duyệt cũng đủ dùng cho bài này).
  - Mở sẵn `yotea-be/src/app.js` — cả bài xoay quanh đúng **52 dòng** của nó.

  ---

  ## 1. Express là gì, và vì sao cần nó?

  Backend Yotea giống một **toà nhà văn phòng**. Khách (trình duyệt) đi vào, cầm tờ phiếu ghi *"tôi muốn `GET /api/products`"*.

  **Express chính là cô lễ tân ở sảnh.** Cô ấy đọc phiếu xem khách muốn **hành động gì** (`GET`) và **tới bộ phận nào** (`/api/products`), cho khách qua vài **chốt kiểm tra** (bảo vệ soi thẻ, ghi sổ khách ra vào), dẫn tới **đúng phòng ban** xử lý (controller), rồi mang kết quả trả lại.

  Không có lễ tân thì khách phải tự mò khắp toà nhà — đó chính là viết server bằng module `http` thuần của Node.js:

  ```js
  // code minh hoạ — dự án KHÔNG viết như thế này
  http.createServer((req, res) => {
    if (req.method === "GET" && req.url === "/api/products") { /* ... */ }
    else if (req.method === "POST" && req.url === "/api/products") { /* ... */ }
    else if (req.url.startsWith("/api/products/")) {
      const slug = req.url.split("/")[3];   // tự tay cắt chuỗi lấy tham số 😱
    }
    // ... còn ~60 route nữa trong Yotea
  }).listen(8080);
  ```

  | Việc cần làm | `http` thuần | Express |
  |---|---|---|
  | Định tuyến | `if/else` lồng nhau vô tận | `router.get("/products/:slug", read)` |
  | Lấy tham số trong URL | Tự cắt chuỗi bằng `split` | `req.params.slug` có sẵn |
  | Đọc body JSON | Tự gom từng mẩu byte rồi `JSON.parse` | `express.json()` làm hộ ⇒ `req.body` |
  | Tái sử dụng logic (check quyền, log…) | Copy-paste khắp nơi | **Middleware** — khai báo một lần |

  > 📖 **Thuật ngữ:** *framework* — bộ khung dựng sẵn. Bạn không viết lại phần "nhận request, phân tuyến, trả response"; bạn chỉ điền phần **nghiệp vụ** riêng của mình vào. Yotea dùng **Express 4** (`yotea-be/package.json:18` khai `"express": "^4.17.3"`).

  ---

  ## 2. Middleware — dây chuyền các chốt kiểm tra

  Middleware là những **chốt** mà request phải đi qua, xếp thành hàng dọc theo đúng thứ tự bạn khai báo:

  ```
  Request  →  [1] express.json()  →  [2] cors()  →  [3] morgan("tiny")  →  [4] Router /api  →  Response
                gắn req.body         dán header      ghi log                controller
                    │                    │              │                      │
                  next()               next()         next()               res.json()
  ```

  Điểm cốt lõi: **mọi chốt đều mở cửa cho chốt sau bằng cùng một hàm tên là `next()`.**

  ```js
  // code minh hoạ chuẩn Express — không trích từ dự án
  const middlewareCuaBan = (req, res, next) => {
    // 1. đọc / gắn thêm dữ liệu vào req
    // 2. hoặc trả luôn response và DỪNG dây chuyền
    // 3. hoặc gọi next() để nhường cho chốt kế tiếp
    next();
  };
  ```

  | Tham số | Là gì | Dùng để |
  |---|---|---|
  | `req` | Đối tượng **request** — tờ phiếu của khách | Đọc `req.body`, `req.params`, `req.query`, `req.headers`; **gắn thêm** dữ liệu cho chốt sau (`req.profile`, `req.auth`) |
  | `res` | Đối tượng **response** — phong bì trả lời | `res.json(...)`, `res.status(400).json(...)` |
  | `next` | **Một hàm** do Express đưa vào | Gọi `next()` = "chốt tôi xong, mời chốt sau" |

  Một middleware chỉ được làm **đúng một trong hai** việc kết thúc: **trả response** (`res.json`, `res.send`… ⇒ dây chuyền dừng tại đây, chốt sau không chạy), hoặc **gọi `next()`** (nhường quyền cho chốt kế tiếp).

  Bạn đã gặp middleware thật ở [Bài 03](03-kien-thuc-nen.md) — `yotea-be/src/middlewares/checkAuth.js:9-19`:

  ```js
  export const isAuth = (req, res, next) => {
    const status = req.profile._id == req.auth._id;

    if (!status) {
      res.status(400).json({
        message: "Bạn không có quyền truy cập",
      });
    } else {
      next();
    }
  };
  ```

  Đọc lại bằng con mắt mới: *"không đúng chủ tài khoản → **trả lỗi và dừng**; đúng → `next()`, cho đi tiếp tới controller."* Đây là dạng middleware kinh điển: **cái chốt canh cửa**.

  **Quên gọi `next()` thì sao?** Không có gì "nổ". Không có lỗi đỏ. Đó mới là chỗ đáng sợ: request **đứng im tại chốt đó mãi mãi**, trình duyệt quay vòng vòng đến khi timeout, còn terminal backend **im lặng tuyệt đối** (vì morgan chỉ ghi log khi response kết thúc).

  > ⚠️ Bug này rất khó tìm vì nó **không tạo ra thông báo lỗi nào**. Ở phần 🛠️ Tự tay làm bên dưới, bạn sẽ **cố ý gây ra nó một lần** cho nhớ đời.

  ---

  ## 3. Soi code thật: mổ xẻ `app.js` từng dòng

  Toàn bộ file chỉ 52 dòng, chia làm 6 khối.

  ### 3.1. Khối 1 — Import 4 thư viện lõi

  `yotea-be/src/app.js:1-4`

  ```js
  import express from "express";
  import cors from "cors";
  import morgan from "morgan";
  import mongoose from "mongoose";
  ```

  **Đọc từng dòng:**

  | Dòng | Code | Ý nghĩa |
  |---|---|---|
  | 1 | `import express from "express"` | Framework web. Không có ngoặc nhọn ⇒ **export mặc định** |
  | 2 | `import cors from "cors"` | Middleware mở khoá cho trình duyệt gọi khác cổng (mục 4.2) |
  | 3 | `import morgan from "morgan"` | Middleware ghi log mọi request ra terminal (mục 4.3) |
  | 4 | `import mongoose from "mongoose"` | Thư viện nói chuyện với MongoDB |

  ### 3.2. Khối 2 — Import 14 router

  `yotea-be/src/app.js:6-20`

  ```js
  // routes
  import categoryRouter from "./routes/category";
  import productRouter from "./routes/product";
  import sliderRouter from "./routes/slider";
  import storeRouter from "./routes/store";
  import cateNewsRouter from "./routes/cateNews";
  import newsRouter from "./routes/news";
  import contactRouter from "./routes/contact";
  import userRouter from "./routes/users";
  import authRouter from "./routes/auth";
  import orderRouter from "./routes/order";
  import orderDetailRouter from "./routes/orderDetail";
  import cmtRouter from "./routes/comment";
  import ratingRouter from "./routes/rating";
  import favoritesRouter from "./routes/favoritesProduct";
  ```

  Mười bốn dòng, mười bốn nhóm chức năng. Ba điểm cần để ý: (1) **không có đuôi `.js`** — Node tự thêm khi tìm file; (2) **dòng 14 là `./routes/users` — số nhiều**, khác 13 file kia, rất dễ gõ nhầm thành `user` ⇒ `Cannot find module`; (3) **tên biến do bạn đặt**, vì router export mặc định nên `cmtRouter` hay `commentRouter` đều được.

  > 💡 Mỗi file trong `routes/` là một **cụm route con** (dùng `Router()` của Express). Nhờ tách file mà `app.js` không phình lên hàng nghìn dòng. Cách router hoạt động sẽ mổ kỹ ở [Bài 06](06-vong-doi-mot-request.md).

  ### 3.3. Khối 3 — Tạo app và gắn 3 middleware toàn cục

  `yotea-be/src/app.js:22-27`

  ```js
  const app = express();

  // middleware
  app.use(express.json());
  app.use(cors());
  app.use(morgan("tiny"));
  ```

  **Đọc từng dòng:**

  | Dòng | Code | Ý nghĩa |
  |---|---|---|
  | 22 | `const app = express()` | Gọi hàm `express()` để **tạo ứng dụng**. Từ đây `app` là "toà nhà" của ta |
  | 24 | `// middleware` | Comment phân khối |
  | 25 | `app.use(express.json())` | Chốt 1: đọc body JSON, đổ vào `req.body` |
  | 26 | `app.use(cors())` | Chốt 2: dán header cho phép trình duyệt khác origin gọi vào |
  | 27 | `app.use(morgan("tiny"))` | Chốt 3: ghi một dòng log cho mỗi request |

  `app.use(x)` nghĩa là: *"xếp `x` vào cuối dây chuyền, áp dụng cho **mọi** request."*

  **`express.json()` làm gì cụ thể?** Khi client gửi `POST /api/signin` với body `{ "email": "a@gmail.com", "password": "123456" }`, dữ liệu đó tới server dưới dạng **luồng byte thô**, không phải object. `express.json()` gom luồng đó lại, `JSON.parse` rồi gán vào `req.body`. Nhờ vậy controller mới viết được — `yotea-be/src/controllers/category.js:5-6`:

  ```js
  export const create = async (req, res) => {
      req.body.slug = slugify(req.body.name);
  ```

  > ⚠️ **Chỗ này dự án làm chưa chuẩn:** `express.json()` gọi **không tham số** ⇒ giới hạn body mặc định **100 KB**. Trang quản trị Yotea có upload ảnh; gửi ảnh base64 lớn hơn 100 KB sẽ dính `PayloadTooLargeError`. Sửa: `app.use(express.json({ limit: "5mb" }))` — xem [Bài 32](32-trang-quan-tri.md). Ngoài ra dự án **không có** `express.urlencoded()`, nên form HTML truyền thống gửi lên sẽ cho `req.body` rỗng (may là Yotea chỉ gửi JSON qua axios).

  ### 3.4. Khối 4 — Mount 14 router lên tiền tố `/api`

  `yotea-be/src/app.js:29-42`

  ```js
  app.use("/api", categoryRouter);
  app.use("/api", productRouter);
  app.use("/api", sliderRouter);
  app.use("/api", storeRouter);
  app.use("/api", cateNewsRouter);
  app.use("/api", newsRouter);
  app.use("/api", contactRouter);
  app.use("/api", userRouter);
  app.use("/api", authRouter);
  app.use("/api", orderRouter);
  app.use("/api", orderDetailRouter);
  app.use("/api", cmtRouter);
  app.use("/api", ratingRouter);
  app.use("/api", favoritesRouter);
  ```

  Đây là dạng `app.use(<đường dẫn>, <middleware>)` — chỉ áp dụng cho request có URL bắt đầu bằng `/api`. **Ghép đường dẫn thế nào?** Lấy `app.js:29` cộng với `yotea-be/src/routes/category.js:10` (`router.get("/category", list);`) ⇒ `/api` + `/category` = **`GET http://localhost:8080/api/category`**, chính là URL mà frontend gọi.

  | Dòng | URL thật sự | | Dòng | URL thật sự |
  |---|---|---|---|---|
  | 29 `categoryRouter` | `/api/category…` | | 36 `userRouter` | `/api/users…` |
  | 30 `productRouter` | `/api/products…` | | 37 `authRouter` | `/api/signin`, `/api/signup`… |
  | 31 `sliderRouter` | `/api/slider…` | | 38 `orderRouter` | `/api/orders…` |
  | 32 `storeRouter` | `/api/store…` | | 39 `orderDetailRouter` | `/api/orderDetail…` |
  | 33 `cateNewsRouter` | `/api/cateNews…` | | 40 `cmtRouter` | `/api/comments…` |
  | 34 `newsRouter` | `/api/news…` | | 41 `ratingRouter` | `/api/ratings…` |
  | 35 `contactRouter` | `/api/contact…` | | 42 `favoritesRouter` | `/api/favoritesProduct…` |

  **Vì sao 14 router cùng một tiền tố mà không đá nhau?** Vì Express duyệt **tuần tự** từ dòng 29 xuống 42. Router nào không khớp URL thì lặng lẽ gọi `next()` nhường router sau. Ví dụ `GET /api/products`: `categoryRouter` (dòng 29) xem qua, thấy trong nó chỉ có `/category…` nên nhường; `productRouter` (dòng 30) khớp `/products` nên xử lý.

  > 💡 **Hệ quả cần nhớ:** nếu hai router định nghĩa **trùng path**, router có số dòng nhỏ hơn **luôn thắng**, router dưới bị che vĩnh viễn — loại bug rất khó nhìn ra bằng mắt. Đây cũng là lý do khi bạn tự thêm router mới (bài sau: `toppingRouter`), bạn **phải nhớ thêm một dòng `app.use("/api", toppingRouter)`** — quên là API mới im lặng trả 404.

  ### 3.5. Khối 5 — Kết nối MongoDB

  `yotea-be/src/app.js:44-48`

  ```js
  // connect db
  mongoose
    .connect("mongodb://localhost:27017/yotea")
    .then(() => console.log("Connected to MongoDB"))
    .catch((error) => console.log(error));
  ```

  | Dòng | Code | Ý nghĩa |
  |---|---|---|
  | 45-46 | `mongoose.connect("mongodb://localhost:27017/yotea")` | Mở kết nối tới MongoDB trên **máy này** (`localhost`), cổng **27017**, database **`yotea`** |
  | 47 | `.then(() => console.log(...))` | Kết nối xong thì in `Connected to MongoDB` |
  | 48 | `.catch((error) => console.log(error))` | Kết nối hỏng thì **chỉ in lỗi ra** |

  > ⚠️ **Chỗ này dự án làm chưa chuẩn — hai lỗi cùng lúc:**
  > 1. **Chuỗi kết nối bị hardcode.** Muốn đổi sang MongoDB Atlas hay đổi tên database, bạn phải sửa **source code** rồi commit lại. Cách chuẩn: để trong `.env` và viết `mongoose.connect(process.env.MONGO_URI)`. Backend Yotea **không có `.env`** — sẽ sửa ở [Bài 34](34-refactor-du-an.md).
  > 2. **`.catch` chỉ `console.log`.** Kết nối DB chết nhưng server **vẫn lên cổng 8080** như không có chuyện gì; người dùng gọi API bị **treo tới timeout** thay vì nhận lỗi rõ ràng. Chuẩn hơn: `.catch((error) => { console.log(error); process.exit(1); })` — thà chết hẳn còn hơn chết dở.

  ### 3.6. Khối 6 — Mở cổng lắng nghe

  `yotea-be/src/app.js:50-52`

  ```js
  // connect
  const PORT = process.env.PORT || 8080;
  app.listen(PORT, () => console.log(`App is running on port: ${PORT}`));
  ```

  | Dòng | Code | Ý nghĩa |
  |---|---|---|
  | 50 | `// connect` | Comment đặt sai nghĩa — đúng ra là `// start server` |
  | 51 | `process.env.PORT \|\| 8080` | Lấy cổng từ **biến môi trường**; không có thì dùng 8080 |
  | 52 | `app.listen(PORT, callback)` | Bắt đầu lắng nghe; callback chạy khi server sẵn sàng |

  Dòng 51 là **chỗ duy nhất trong toàn bộ `src/` dùng `process.env`**. Nhờ nó mà khi deploy lên Heroku/Render (những nơi tự cấp cổng) server vẫn chạy đúng — xem [Bài 36](36-build-va-deploy.md).

  > 💡 **Chi tiết tinh tế:** `app.listen` (dòng 52) chạy **đồng bộ**, còn `.then()` của mongoose (dòng 47) là **bất đồng bộ**. Vì vậy terminal gần như luôn in `App is running on port: 8080` **trước** `Connected to MongoDB` — có một khoảnh khắc ngắn server đã nhận request nhưng database chưa sẵn sàng.

  ---

  ## 4. Ba thứ cần hiểu sâu

  ### 4.1. Vì sao THỨ TỰ middleware quan trọng đến vậy?

  Đây là phần quan trọng nhất của bài. Hãy nhớ: **`app.use` là xếp hàng, không phải khai báo.** Dòng nào viết trước thì chạy trước, không có ngoại lệ. Giả sử ai đó "dọn dẹp" `app.js` cho đẹp và đảo thành:

  ```js
  // ❌ CODE SAI — chỉ để minh hoạ, dự án KHÔNG viết như thế này
  app.use("/api", categoryRouter);   // router đứng TRƯỚC
  app.use(express.json());           // body parser đứng SAU
  ```

  Gọi `POST /api/category/:userId` để thêm danh mục mới sẽ diễn ra như sau:

  ```
  Request tới
    → categoryRouter khớp path → chạy thẳng vào controller create
        → controller đọc req.body.name
          → nhưng express.json() CHƯA hề chạy
            → req.body là undefined
              → 💥 TypeError: Cannot read properties of undefined (reading 'name')
    → express.json() không bao giờ tới lượt, vì response lỗi đã trả rồi
  ```

  Dòng chết chính xác là `yotea-be/src/controllers/category.js:6` — `req.body.slug = slugify(req.body.name);`.

  > ⚠️ **Quy tắc vàng cần thuộc lòng:** mọi middleware **chuẩn bị dữ liệu** (`express.json`, `cors`, `morgan`, xác thực) phải đứng **TRƯỚC** router. Mọi middleware **dọn dẹp cuối** (404 handler, error handler) phải đứng **SAU** router.

  | Thứ tự | Dòng | Middleware | Nếu đặt sai chỗ thì sao? |
  |---|---|---|---|
  | 1 | `:25` | `express.json()` | Đặt sau router ⇒ `req.body` undefined ở mọi API thêm/sửa |
  | 2 | `:26` | `cors()` | Đặt sau router ⇒ response thiếu header CORS, trình duyệt chặn |
  | 3 | `:27` | `morgan("tiny")` | Đặt sau router ⇒ chỉ log được request **không** khớp route nào |
  | 4 | `:29-42` | 14 router | — |
  | 5 | *(thiếu)* | 404 handler | mục 6.2 |
  | 6 | *(thiếu)* | error handler | mục 6.3 |

  > ⚠️ **Chỗ này dự án làm chưa chuẩn:** `express.json()` (dòng 25) đứng **trước** `cors()` (dòng 26); đúng ra `cors()` nên là chốt **đầu tiên**. Lý do: nếu client lỡ gửi JSON hỏng, `express.json()` ném lỗi 400 **trước khi** header CORS kịp được dán vào response. Trình duyệt lúc đó chỉ báo một dòng "CORS error" mù mờ, còn lỗi 400 thật thì bạn không bao giờ thấy.

  Nguyên tắc tương tự cũng đúng **bên trong một route** — `yotea-be/src/routes/product.js:8`:

  ```js
  router.post("/products/:userId", requireSignin, isAuth, isAdmin, create);
  ```

  Bốn hàm xếp hàng: `requireSignin` → `isAuth` → `isAdmin` → `create`. Đảo `isAuth` lên trước `requireSignin` thì `req.auth` chưa tồn tại, `req.auth._id` ném `TypeError`, server trả **500** thay vì **401** lịch sự. Chi tiết ở [Bài 12](12-phan-quyen-middleware.md).

  ### 4.2. CORS — vì sao `localhost:3000` không gọi được `localhost:8080`?

  **Same-Origin Policy (chính sách cùng nguồn gốc)** là luật an ninh của **trình duyệt** (không phải của server): *"JavaScript trên trang A chỉ được đọc dữ liệu từ trang A."* Hai URL cùng **origin** khi trùng **cả ba**: giao thức, tên miền, cổng.

  | Trang gọi | Gọi tới | Cùng origin? | Vì sao |
  |---|---|---|---|
  | `http://localhost:3000` | `http://localhost:3000/x` | ✅ | Trùng cả 3 |
  | `http://localhost:3000` | `http://localhost:8080/api` | ❌ | **Khác cổng** |
  | `http://localhost:3000` | `https://localhost:3000` | ❌ | Khác giao thức |
  | `http://localhost:3000` | `http://api.yotea.vn` | ❌ | Khác tên miền |

  Vì sao cần luật khắt khe vậy? Vì nếu không, một trang web độc hại bạn vô tình mở có thể chạy JavaScript ngầm gọi tới `https://internet-banking.com/api/chuyen-tien` bằng chính phiên đăng nhập của bạn — và đọc luôn kết quả.

  **Vấn đề của Yotea:** frontend cổng **3000**, backend cổng **8080** — `yotea-fe/src/api/instance.js:3-5`:

  ```js
  const instance = axios.create({
    baseURL: "http://localhost:8080/api",
  });
  ```

  Khác cổng ⇒ khác origin ⇒ trình duyệt chặn, với thông báo quen thuộc:

  ```
  Access to XMLHttpRequest at 'http://localhost:8080/api/products'
  from origin 'http://localhost:3000' has been blocked by CORS policy:
  No 'Access-Control-Allow-Origin' header is present on the requested resource.
  ```

  **CORS (Cross-Origin Resource Sharing)** là cơ chế để **server nói với trình duyệt**: *"tôi cho phép trang đó đọc dữ liệu của tôi"* — nói bằng cách gắn thêm **header** vào response. Đó chính là việc `app.js:26` (`app.use(cors())`) làm: mọi response có thêm `Access-Control-Allow-Origin: *`, tức **"mọi origin trên đời đều được"**. Ngoài ra `cors()` còn tự trả lời **request preflight** — request `OPTIONS` mà trình duyệt tự bắn ra trước các request "không đơn giản" (có `PUT`, `DELETE`, hoặc có header `Authorization`) để hỏi trước cho chắc.

  > 🔒 **Ghi chú bảo mật:** `cors()` **không tham số** = mở cửa cho **mọi domain**. Khi học thì tiện, nhưng lên production đây là rủi ro thật: bất kỳ website nào cũng gọi được API Yotea từ trình duyệt của người dùng. Cách chuẩn là khai danh sách trắng — *đoạn dưới đây bạn tự viết thêm, dự án hiện chưa có*:
  >
  > ```js
  > app.use(cors({
  >   origin: ["http://localhost:3000", "https://yotea.vn"],
  >   credentials: true,
  > }));
  > ```
  >
  > Quay lại ở [Bài 33 — Rà soát bảo mật](33-ra-soat-bao-mat.md).

  > 💡 **Hai điều hay bị hiểu nhầm:** (1) **CORS không bảo vệ server** — Postman, `curl`, hay một script Node.js đều gọi được API bất chấp CORS, vì chúng không phải trình duyệt. (2) **Lỗi CORS không phải lỗi của frontend** — cách sửa duy nhất là **server** gắn header. Thấy "CORS error" thì đi sửa `app.js`, đừng loay hoay trong React.

  ### 4.3. morgan — cuốn sổ ghi khách ra vào

  `app.js:27` gọi `app.use(morgan("tiny"))`. `"tiny"` là tên một **định dạng có sẵn**, gọn nhất: `:method :url :status :res[content-length] - :response-time ms`. Chạy backend rồi mở frontend, terminal sẽ trôi những dòng như:

  ```
  GET /api/products 200 4821 - 18.512 ms
  POST /api/signin 200 289 - 121.907 ms
  POST /api/orders 400 63 - 9.887 ms
  ```

  | Mảnh | Ví dụ | Nghĩa |
  |---|---|---|
  | `:method` | `POST` | Method HTTP |
  | `:url` | `/api/signin` | Đường dẫn (đã gồm `/api`) |
  | `:status` | `200` | Status code. Thấy `400`/`401`/`500` là có chuyện |
  | `:res[content-length]` | `289` | Kích thước response (byte) |
  | `- :response-time ms` | `- 121.907 ms` | Xử lý mất bao lâu |

  > 💡 **Dùng morgan để debug:** frontend báo lỗi mà không biết lỗi ở đâu ⇒ nhìn terminal backend trước. **Không có dòng log nào** ⇒ request **chưa hề tới** backend (sai URL, sai cổng, backend chưa chạy, hoặc bị CORS chặn từ trước). **Có log, status `404`** ⇒ request tới rồi nhưng **sai đường dẫn**. **Có log, status `400`/`500`** ⇒ đúng chỗ rồi, lỗi nằm **trong controller**. **Response-time > 1000 ms** ⇒ nghi truy vấn database chậm.

  Chi tiết thú vị: morgan in log **khi response đã gửi xong**, không phải lúc nhận request. Nên nếu bạn thêm một middleware log ở **sau** morgan, log của bạn vẫn xuất hiện **trước** — bạn sẽ tận mắt thấy ở phần Tự tay làm.

  ---

  ## 5. Vì sao backend chạy được `import`? — `.babelrc` và `npm start`

  Node.js đời cũ chỉ hiểu `require(...)`. Viết `import express from "express"` rồi chạy `node src/app.js` sẽ nhận ngay `SyntaxError: Unexpected token 'import'`. Yotea giải quyết bằng **Babel** — trình biên dịch chuyển cú pháp mới thành cú pháp cũ.

  `yotea-be/.babelrc:1-3`

  ```json
  {
      "presets": ["env", "stage-0"]
  }
  ```

  | Preset | Là gì | Cho phép viết gì |
  |---|---|---|
  | `env` | Gói `babel-preset-env` (Babel **6**) | `import/export`, arrow function, `async/await`, template literal… |
  | `stage-0` | Gói `babel-preset-stage-0` (Babel **6**) | Object rest/spread — chính là `{ password, ...rest }` bạn học ở [Bài 03](03-kien-thuc-nen.md) |

  Câu lệnh khởi động — `yotea-be/package.json:7`:

  ```json
      "start": "nodemon ./src/app.js --exec babel-node -e js",
  ```

  | Mảnh | Nghĩa |
  |---|---|
  | `nodemon` | Chương trình theo dõi file: bạn **sửa & lưu** là nó **tự khởi động lại** server. Không có nó, mỗi lần sửa code phải `Ctrl + C` rồi chạy lại |
  | `./src/app.js` | File điểm vào — chính là file ta vừa mổ xẻ |
  | `--exec babel-node` | Thay vì chạy bằng `node`, chạy bằng `babel-node`: biên dịch **ngay trong bộ nhớ** rồi mới chạy, không sinh thư mục `dist/` |
  | `-e js` | Viết tắt của `--ext js`: nodemon chỉ theo dõi file đuôi `.js` |

  Ghép lại:

  ```
  npm start
  └─> nodemon gọi babel-node ./src/app.js
        └─> babel-node đọc .babelrc → preset env + stage-0
            └─> dịch app.js và mọi file được import sang CommonJS (trong RAM)
                  └─> chạy app.js:  :22 tạo app · :25-27 middleware · :29-42 mount router
                                    :45-48 mongoose.connect · :52 app.listen(8080)
  └─> terminal in:  App is running on port: 8080  /  Connected to MongoDB
  └─> nodemon ngồi canh; bạn lưu file .js nào là nó restart
  ```

  > ⚠️ **Chỗ này dự án làm chưa chuẩn:** `package.json:5` khai `"main": "index.js"` nhưng file `index.js` **không hề tồn tại** (chạy `node .` sẽ lỗi). Và **không có script `build`**, nghĩa là dự án chỉ chạy được ở chế độ dev; muốn deploy thật phải tự thêm `babel src -d dist` rồi `node dist/app.js` ([Bài 36](36-build-va-deploy.md)).

  ---

  ## 6. Những gì `app.js` còn thiếu

  Đọc hết 52 dòng rồi, giờ hãy chú ý tới **những dòng KHÔNG có** trong file — kỹ năng đọc code quan trọng không kém.

  **6.1. Không mount Swagger.** `yotea-be/package.json:24-25` khai `"swagger-jsdoc": "^6.2.0"` và `"swagger-ui-express": "^4.3.0"` — hai thư viện làm tài liệu API **đã được cài**. Nhưng hãy **tự kiểm chứng**: `Ctrl + F` trong `app.js` tìm chữ `swagger` → **0 kết quả**. Không có `import swaggerJsdoc`, không có `import swaggerUi`, không có `app.use("/api-docs", ...)`. Hệ quả: mở `http://localhost:8080/api-docs` sẽ nhận `Cannot GET /api-docs`. Trớ trêu hơn: trong các controller **đã có sẵn 13 khối comment `@swagger`** (`product.js` 6 khối, `user.js` 5 khối, `auth.js` 2 khối — riêng `yotea-be/src/controllers/product.js` nhiều nhất). Người viết đã bỏ công viết tài liệu, nhưng vì quên mount nên **không ai đọc được** — "code chết" dạng tài liệu. Ta sẽ tự tay bật `/api-docs` ở [Bài 13](13-swagger-tai-lieu-api.md).

  **6.2. Không có route 404.** Sau dòng 42 là đi thẳng tới `mongoose.connect`, không có middleware nào hứng URL không khớp. Gọi `GET /api/khong-ton-tai` sẽ nhận về một **trang HTML** của Express (`Cannot GET /api/khong-ton-tai`). Frontend dùng axios đang chờ JSON, nhận HTML ⇒ lỗi parse khó hiểu thay vì thông báo rõ ràng.

  **6.3. Không có middleware xử lý lỗi tập trung.** Express có một loại middleware đặc biệt: **error handler**, nhận **bốn** tham số — *đoạn dưới bạn tự viết thêm, dự án chưa có*:

  ```js
  app.use((err, req, res, next) => {
    res.status(500).json({ message: "Lỗi server", error: err.message });
  });
  ```

  Dấu hiệu nhận biết duy nhất là **số tham số = 4**. Thiếu một tham số, Express coi nó là middleware thường và không bao giờ gọi tới khi có lỗi. Yotea **không có** hàm nào như vậy. Hệ quả nặng nhất: khi `express-jwt` ném `UnauthorizedError` (token sai / hết hạn / thiếu header), Express dùng error handler mặc định, trả HTML kèm **stack trace đầy đủ** — vừa lộ cấu trúc thư mục server, vừa khiến frontend không đọc nổi `message`.

  > ⚠️ **Tóm lại `app.js` thiếu 3 thứ:** mount Swagger, 404 handler, error handler. Cả ba đều chỉ là "thêm vài dòng ở cuối file", nhưng thiếu chúng thì trải nghiệm debug tệ đi rất nhiều. Ta sẽ bổ sung đầy đủ ở [Bài 34](34-refactor-du-an.md).

  ---

  ## 7. 🛠️ Tự tay làm

  > Mục tiêu: cuối phần này bạn sẽ có **middleware đầu tiên do chính mình viết** đang chạy trong dự án thật, một endpoint `GET /api/health` để kiểm tra server còn sống, và **cảm giác tận tay** về chuyện quên `next()`.

  > 💡 Ở [Bài 03](03-kien-thuc-nen.md) bạn mới chỉ **đọc** code; từ bài này bạn bắt đầu **gõ** vào dự án thật. Và từ [Bài 05](05-mongoose-model.md), bạn sẽ xây một chức năng hoàn toàn mới của riêng mình — **Topping** (trân châu, thạch, pudding) — chạy song song suốt phần backend. Bài này là bước tập tay trước khi vào việc.

  Mở terminal tại `yotea-be`, chạy `npm start` và để đó — nodemon sẽ tự restart mỗi lần bạn lưu file.

  ### Bước 1 — Viết middleware log của riêng bạn

  Mở `yotea-be/src/app.js`, chèn đoạn sau **ngay dưới dòng 27** (`app.use(morgan("tiny"));`) và **trên dòng 29** (`app.use("/api", categoryRouter);`):

  ```js
  // yotea-be/src/app.js  ← đoạn này BẠN TỰ VIẾT THÊM, dự án chưa có
  app.use((req, res, next) => {
    const gio = new Date().toLocaleTimeString("vi-VN");
    console.log(`>>> [${gio}] ${req.method} ${req.originalUrl}`);
    next();
  });
  ```

  | Phần | Ý nghĩa |
  |---|---|
  | `app.use((req, res, next) => {` | Xếp một hàm vào dây chuyền, áp dụng cho **mọi** request |
  | `new Date().toLocaleTimeString("vi-VN")` | Lấy giờ hiện tại dạng `14:32:07` |
  | `req.originalUrl` | URL **đầy đủ** kể cả query string (khác `req.url` đã bị Express cắt prefix bên trong router) |
  | `next()` | **Bắt buộc** — nhường quyền cho router phía dưới |

  Vì sao đặt ở đây? Phải **trước router** thì mới log được mọi request; còn việc đặt **sau morgan** là để bạn nhìn thấy hiện tượng ở Bước 3.

  ### Bước 2 — Thêm route `GET /api/health`

  Chèn đoạn sau **ngay dưới dòng 42** (`app.use("/api", favoritesRouter);`), tức **trên** comment `// connect db`:

  ```js
  // yotea-be/src/app.js  ← đoạn này BẠN TỰ VIẾT THÊM, dự án chưa có
  app.get("/api/health", (req, res) => {
    res.json({ status: "ok" });
  });
  ```

  Ba điều cần nhớ: (1) `app.get(...)` khác `app.use(...)` — nó chỉ khớp đúng **method `GET`** và đúng **path `/api/health`**; (2) ở đây viết thẳng `/api/health` đủ cả tiền tố, vì đang gắn vào `app` chứ không phải vào router; (3) handler này **không gọi `next()`** — và như thế là **đúng**, vì nó đã `res.json(...)` để kết thúc request rồi.

  Lưu file. Terminal sẽ hiện `[nodemon] restarting due to changes...` rồi in lại 2 dòng khởi động.

  ### Bước 3 — Gọi thử

  Mở trình duyệt vào `http://localhost:8080/api/health`, phải thấy `{ "status": "ok" }`. Nhìn sang terminal backend:

  ```
  >>> [14:32:07] GET /api/health
  GET /api/health 200 15 - 2.418 ms
  ```

  Hai dòng, hai người ghi khác nhau. Để ý **dòng của bạn in TRƯỚC dòng của morgan**, dù morgan khai báo ở dòng 27 còn middleware của bạn ở dòng 28 — lý do đã nói ở mục 4.3: morgan chờ **response gửi xong** mới ghi sổ.

  Thử thêm vài URL để thấy middleware của bạn bắt được **tất cả**: `http://localhost:8080/api/category` → log `>>> [..] GET /api/category`; `http://localhost:8080/api/gi-do-khong-co` → **vẫn** log, rồi trình duyệt hiện `Cannot GET /api/gi-do-khong-co` (chính là chỗ thiếu 404 handler ở mục 6.2).

  ### Bước 4 — Cố ý bỏ `next()` để thấy request bị treo

  Phần đáng nhớ nhất. Trong middleware ở Bước 1, hãy **comment dòng `next()`**:

  ```js
  // yotea-be/src/app.js  ← thí nghiệm TẠM THỜI, nhớ khôi phục ở cuối bước
  app.use((req, res, next) => {
    const gio = new Date().toLocaleTimeString("vi-VN");
    console.log(`>>> [${gio}] ${req.method} ${req.originalUrl}`);
    // next();   ← cố tình bỏ đi
  });
  ```

  Lưu file, gọi lại `http://localhost:8080/api/health`:

  | Nơi | Hiện tượng |
  |---|---|
  | Trình duyệt | Tab quay vòng vòng mãi, cuối cùng báo timeout |
  | Terminal | **Chỉ có** dòng `>>> [14:35:12] GET /api/health` của bạn |
  | Terminal | **KHÔNG có** dòng log của morgan, cũng **không có** dòng lỗi đỏ nào |

  Vì sao morgan im lặng? Vì morgan chỉ ghi khi **response kết thúc** — mà response này không bao giờ kết thúc. Thử luôn `http://localhost:8080/api/products` — cũng treo. **Toàn bộ backend đã tê liệt**, chỉ vì thiếu một dòng bảy ký tự.

  > ⚠️ **Bài học:** không có thông báo lỗi nào cả. Sau này gặp hiện tượng "API tự nhiên treo, terminal không báo gì", việc đầu tiên là **rà lại xem middleware nào quên `next()`** (hoặc quên trả response).

  **Khôi phục:** bỏ comment dòng `next();`, lưu lại, gọi lại `/api/health` → mọi thứ bình thường.

  ### Bước 5 — Dọn dẹp

  Middleware log và route `/api/health` là **code bạn tự thêm**, không có trong dự án gốc. Chúng vô hại, bạn có thể **giữ lại** để tiện debug các bài sau. Muốn trả `app.js` về đúng 52 dòng nguyên bản, chạy trong `yotea-be`: `git diff src/app.js` (xem mình đã thêm gì) rồi `git checkout src/app.js`.

  ---

  ## 8. ✅ Kiểm chứng kết quả

  ```bash
  # đứng tại thư mục yotea-be
  npm start
  ```

  Terminal phải hiện đúng hai dòng này, theo đúng thứ tự:

  ```
  App is running on port: 8080
  Connected to MongoDB
  ```

  Sau đó kiểm 4 thứ:

  **1.** Route bạn vừa viết — `GET http://localhost:8080/api/health` → `{ "status": "ok" }`.

  **2.** Một route có sẵn của dự án — `GET http://localhost:8080/api/category`:

  ```json
  [
    { "_id": "6650a1f2c4e8b91234abcd99", "name": "Trà sữa", "slug": "tra-sua" }
  ]
  ```

  (Database trống thì nhận `[]` — vẫn là **thành công**, chỉ là chưa có dữ liệu.)

  **3.** Header CORS thật sự tồn tại — trong Postman, bấm tab **Headers** của phần response, phải thấy `Access-Control-Allow-Origin: *`.

  **4.** Trang Swagger CHƯA hoạt động (đúng như phân tích ở mục 6.1) — `GET http://localhost:8080/api-docs` → `Cannot GET /api-docs`.

  Đủ 4 kết quả trên là bạn đã hiểu và kiểm chứng được toàn bộ `app.js`.

  ---

  ## 9. 🐞 Lỗi thường gặp

  | Thông báo lỗi | Nguyên nhân | Cách sửa |
  |---|---|---|
  | `SyntaxError: Unexpected token 'import'` | Chạy `node src/app.js` thay vì `npm start` | Luôn dùng `npm start` để đi qua `babel-node` |
  | `Error: listen EADDRINUSE :::8080` | Cổng 8080 đang bị chiếm (thường là một `npm start` cũ chưa tắt) | Tắt terminal cũ, hoặc `netstat -ano \| findstr :8080` rồi `taskkill /PID <pid> /F` |
  | `MongooseServerSelectionError: connect ECONNREFUSED ::1:27017` | MongoDB chưa chạy | Bật service (`net start MongoDB`). Chú ý server **vẫn** lên cổng 8080 nên dễ tưởng nhầm là ổn |
  | `Cannot read properties of undefined (reading 'name')` trong controller | `req.body` là `undefined` — thiếu hoặc đặt sai chỗ `express.json()` | Đảm bảo `app.use(express.json())` nằm **trước** các dòng `app.use("/api", ...)` |
  | `Cannot GET /api/xxx` | Sai URL, hoặc quên `app.use("/api", xxxRouter)` | Đối chiếu path trong `routes/*.js`; nhớ tiền tố `/api` |
  | `has been blocked by CORS policy` | Response thiếu header CORS | Kiểm tra `app.use(cors())` có tồn tại và nằm **trước** router không |
  | Request treo mãi, terminal không báo gì | Một middleware quên `next()` hoặc quên trả response | Rà từng middleware: hàm nào không `res.*` thì **phải** có `next()` |
  | `Cannot find module './routes/user'` | Gõ thiếu chữ `s` — file thật tên `users.js` | Sửa thành `./routes/users` |
  | `PayloadTooLargeError: request entity too large` | Body vượt 100 KB mặc định của `express.json()` | `app.use(express.json({ limit: "5mb" }))` |

  ---

  ## 10. 📝 Bài tập

  **Bài 1.** Một bạn cùng nhóm "dọn dẹp" `app.js` và nộp phiên bản sau. Hãy chỉ ra **hai lỗi** và giải thích hậu quả cụ thể của từng lỗi.

  ```js
  // phiên bản SAI do bạn cùng nhóm viết — không phải code của dự án
  const app = express();

  app.use("/api", categoryRouter);
  app.use("/api", productRouter);

  app.use(express.json());
  app.use(cors());

  app.use((req, res, next) => {
    console.log(req.method, req.originalUrl);
  });

  app.listen(8080);
  ```

  <details>
  <summary>💡 Xem gợi ý & lời giải</summary>

  **Lỗi 1 — `express.json()` và `cors()` đứng SAU router.** Với `express.json()`: mọi `POST`/`PUT` vào `/api/category` hay `/api/products` chạy thẳng vào controller khi `req.body` còn `undefined`; dòng `yotea-be/src/controllers/category.js:6` (`req.body.slug = slugify(req.body.name)`) ném `TypeError: Cannot read properties of undefined (reading 'name')` → server trả 500. Với `cors()`: response do 2 router phía trên tạo ra **không có** header `Access-Control-Allow-Origin`, frontend cổng 3000 bị trình duyệt chặn — dù Postman gọi vẫn ngon lành (Postman không áp Same-Origin Policy).

  **Lỗi 2 — middleware log ở cuối quên `next()`.** Ở đây nó **vô hại một cách tình cờ**: mọi request khớp `/api` đã được router phía trên trả response rồi nên không bao giờ chạy tới. Nhưng request **không** khớp `/api` (ví dụ `GET /`) sẽ rơi xuống đây, log ra rồi **treo vĩnh viễn**. Đây là loại bug "lúc chạy lúc không" — kinh khủng nhất khi debug.

  **Phiên bản đúng:**

  ```js
  const app = express();

  app.use(cors());            // dán tem CORS trước tiên
  app.use(express.json());    // rồi mới parse body
  app.use((req, res, next) => {
    console.log(req.method, req.originalUrl);
    next();                   // ĐỪNG QUÊN
  });

  app.use("/api", categoryRouter);
  app.use("/api", productRouter);

  app.listen(8080);
  ```

  </details>

  **Bài 2.** Viết một middleware **chỉ chạy cho URL bắt đầu bằng `/api/products`**, đếm số lượt kể từ lúc server khởi động và in ra `[sản phẩm] lượt thứ 7`. Cho biết phải đặt nó ở đâu trong `app.js` và vì sao.

  <details>
  <summary>💡 Xem gợi ý & lời giải</summary>

  **Gợi ý:** `app.use()` nhận thêm đường dẫn ở tham số đầu — `app.use("/api/products", fn)`. Express chỉ gọi `fn` khi URL bắt đầu bằng chuỗi đó.

  ```js
  // yotea-be/src/app.js  ← đoạn này BẠN TỰ VIẾT THÊM, dự án chưa có
  let luotXemSanPham = 0;

  app.use("/api/products", (req, res, next) => {
    luotXemSanPham++;
    console.log(`[sản phẩm] lượt thứ ${luotXemSanPham}`);
    next();
  });
  ```

  **Đặt ở đâu:** phải **trước** dòng `app.use("/api", productRouter);` (dòng 30). Đặt sau thì `productRouter` đã trả response và kết thúc dây chuyền, middleware đếm không bao giờ tới lượt.

  **Ba lưu ý:** (1) biến `luotXemSanPham` nằm trong bộ nhớ tiến trình ⇒ **restart server là về 0**, muốn giữ lâu dài phải lưu xuống database; (2) nó khớp **mọi** method và **mọi** URL con (`GET /api/products`, `GET /api/products/tra-sua-tran-chau`, `POST /api/products/:userId` đều được đếm) — muốn chỉ đếm `GET` thì thêm `if (req.method !== "GET") return next();`; (3) dự án đã có cơ chế đếm lượt xem **thật**, lưu vào database qua `PATCH /api/products/userUpdate/:id` (xem `yotea-be/src/routes/product.js:12`) — mổ ở [Bài 26](26-chi-tiet-san-pham.md).

  </details>

  **Bài 3.** Viết hai middleware còn thiếu của `app.js` (mục 6.2 và 6.3): một 404 handler trả về **JSON** thay vì HTML, và một error handler bắt `UnauthorizedError` của `express-jwt` để trả 401 kèm thông báo tiếng Việt. Nói rõ chúng phải đặt ở đâu.

  <details>
  <summary>💡 Xem gợi ý & lời giải</summary>

  Cả hai đặt **sau dòng 42** (sau khi mount đủ 14 router), **trước** khối `mongoose.connect`. Thứ tự: 404 trước, error handler sau cùng.

  ```js
  // yotea-be/src/app.js  ← đoạn này BẠN TỰ VIẾT THÊM, dự án chưa có

  // 1) Không router nào khớp → chắc chắn là 404
  app.use("/api", (req, res) => {
    res.status(404).json({ message: "Route không tồn tại" });
  });

  // 2) Error handler — DẤU HIỆU NHẬN BIẾT: đúng 4 tham số
  app.use((err, req, res, next) => {
    if (err.name === "UnauthorizedError") {
      return res.status(401).json({
        message: "Token không hợp lệ hoặc đã hết hạn",
      });
    }

    console.error(err);
    res.status(500).json({ message: "Lỗi server", error: err.message });
  });
  ```

  - 404 handler dùng `app.use` **không kèm method**, path `/api` ⇒ hứng mọi request `/api/...` còn sót lại. Nó phải nằm **sau** 14 router, vì đặt trước thì nó nuốt hết mọi request.
  - Error handler nhận **4 tham số** `(err, req, res, next)` — đây là **cách duy nhất** Express phân biệt nó với middleware thường. Tham số `next` không dùng nhưng **không được xoá**: xoá đi còn 3 tham số, Express lại hiểu nhầm thành middleware thường.
  - Thử: gọi một API cần quyền admin mà **không gửi token**, ví dụ `POST http://localhost:8080/api/category/123`. Trước khi thêm: HTML kèm stack trace. Sau khi thêm: `{"message":"Token không hợp lệ hoặc đã hết hạn"}`.

  Xử lý lỗi tập trung sẽ làm bài bản ở [Bài 34](34-refactor-du-an.md).

  </details>

  ---

  ## 📌 Tóm tắt

  - **Express** lo hộ phần định tuyến, đọc body, quản lý middleware — thứ mà module `http` thuần bắt bạn tự viết bằng `if/else` và cắt chuỗi.
  - **Middleware** là hàm `(req, res, next)` xếp thành dây chuyền. Mỗi hàm hoặc **trả response** (dừng dây chuyền), hoặc gọi **`next()`** (nhường chốt sau). Quên cả hai ⇒ **request treo, không báo lỗi**.
  - `app.js` gồm 6 khối: import thư viện (`:1-4`), import 14 router (`:6-20`), tạo app (`:22`), 3 middleware toàn cục (`:25-27`), mount 14 router lên `/api` (`:29-42`), kết nối MongoDB (`:45-48`), lắng nghe cổng 8080 (`:51-52`).
  - **Thứ tự là tất cả:** middleware chuẩn bị dữ liệu đứng **trước** router; 404/error handler đứng **sau** router. Đặt `express.json()` sau router ⇒ `req.body` là `undefined` ở mọi API ghi.
  - **CORS** sinh ra vì Same-Origin Policy của trình duyệt: cổng 3000 gọi cổng 8080 là khác origin. `cors()` gắn `Access-Control-Allow-Origin: *` — tiện khi học, **rủi ro khi lên production** vì mở cho mọi domain.
  - **morgan `"tiny"`** in `method url status size - time ms` khi response xong. Nhìn terminal backend là bước debug đầu tiên: không log ⇒ request chưa tới; log 404 ⇒ sai URL; log 400/500 ⇒ lỗi trong controller.
  - Backend chạy được `import` nhờ **`.babelrc`** (preset `env` + `stage-0`) và `npm start` = `nodemon ./src/app.js --exec babel-node -e js`.
  - Dự án **thiếu 3 thứ ở `app.js`**: mount Swagger `/api-docs` (dù đã cài 2 thư viện), route 404, error handler 4 tham số — cộng thêm chuỗi kết nối MongoDB bị **hardcode**.

  **Từ khoá tra cứu thêm:** `express middleware`, `express.json body parser`, `Same-Origin Policy`, `CORS preflight OPTIONS`, `morgan tiny format`, `express error handling middleware`, `babel-node nodemon`

  ➡️ **Bài tiếp theo:** [05 — Mongoose Model: Schema, kiểu dữ liệu, timestamps, index](05-mongoose-model.md) — bạn đã hiểu cách request đi vào server; giờ xuống tầng dữ liệu, và **bắt đầu tự tay xây chức năng Topping của riêng bạn** bằng file model đầu tiên.
