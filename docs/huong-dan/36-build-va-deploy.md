# Bài 36 — Build và deploy lên môi trường thật

> **Phần 6 · Nâng cao & hoàn thiện** — Thời lượng ước tính: **~120 phút**
> ⬅️ Bài trước: [35 — 🎓 Đồ án cuối khoá: làm chức năng Voucher end-to-end](35-do-an-cuoi-voucher.md) · Bài sau: 🎉 **Bạn đã hoàn thành giáo trình!**
> 🏠 [Mục lục](README.md)

---

## 🎯 Sau bài này bạn sẽ

- Phân biệt rạch ròi **development** và **production**, và hiểu vì sao `localhost` không ai ngoài máy bạn truy cập được.
- Chạy được `npm run build` cho frontend và hiểu bundle production khác gì server dev.
- Chuẩn bị backend cho production: **biến môi trường**, **MongoDB Atlas** (database đám mây miễn phí), chạy bằng `node`/`pm2` thay vì `nodemon`.
- Giới hạn **CORS** về đúng domain frontend thay vì mở cho mọi domain.
- Chọn một combo deploy miễn phí (**Vercel + Render + Atlas**) và bấm từng bước để đưa web lên internet.
- Chạy qua một **checklist trước khi deploy** gom lại mọi thứ đã học từ Bài 33 và 34.

## 📋 Cần chuẩn bị

- Đã hoàn thành [Bài 34](34-refactor-du-an.md) — dự án đã tách `.env`, đã đổi secret, đã có `configureStore` và error handler.
- Đã đọc [Bài 33](33-ra-soat-bao-mat.md) — biết 8 lỗ hổng cần vá trước khi đưa lên mạng.
- Một tài khoản **GitHub** (miễn phí), đã đẩy dự án lên một repo.
- Trình duyệt + Git đã cài trên máy.
- Khoảng 2 tiếng và một tách trà sữa. Đây là bài **cuối cùng** — không vội.

> 💡 **Mẹo:** Bài này dài vì deploy là chuỗi nhiều bước nhỏ, mỗi bước dễ sai một chỗ. Đừng
> đọc lướt. Làm tới đâu chắc tới đó, sai ở đâu quay lại mục **🐞 Lỗi thường gặp** ở cuối.

---

## 1. Development vs Production: hai thế giới khác nhau

Suốt 35 bài qua, bạn chạy dự án theo kiểu **development** (viết tắt: *dev*):

- Backend: `npm start` → `nodemon` + `babel-node`, cổng **8080**.
- Frontend: `npm start` → `react-scripts start`, cổng **3000**.
- Cả hai đều nằm trên **máy bạn**, gọi nhau qua `localhost`.

**Production** (viết tắt: *prod*) là khi web chạy **thật** trên một máy chủ ngoài internet
để **người lạ ở bất kỳ đâu** đều mở được. Bảng so sánh:

| | Development (đang làm) | Production (bài này) |
|---|---|---|
| Ai truy cập được | Chỉ máy bạn | Cả thế giới, qua một domain |
| Frontend chạy bằng | `react-scripts start` (server dev, tự reload) | File tĩnh đã **build** sẵn, đặt trên CDN |
| Backend chạy bằng | `nodemon` (tự restart khi sửa code) | `node`/`pm2` (chạy ổn định, tự hồi phục khi crash) |
| Database | MongoDB `localhost:27017` | MongoDB Atlas (đám mây) |
| Cấu hình | Hardcode / `.env` trên máy | Biến môi trường đặt trên dashboard hosting |
| Tốc độ frontend | Chậm, file to (chưa nén) | Nhanh, file nhỏ (đã minify + tách chunk) |
| Báo lỗi | Hiện stack trace chi tiết | Ẩn stack trace, chỉ báo lỗi chung |

### 1.1. Vì sao `localhost` không ai truy cập được từ ngoài?

`localhost` (và địa chỉ `127.0.0.1`) nghĩa là **"chính máy này"**. Khi trình duyệt của bạn
gọi `http://localhost:3000`, nó tìm web **ngay trên máy bạn** — không đi ra internet chút nào.

Nếu bạn nhắn cho bạn bè link `http://localhost:3000`, trên máy **họ** chữ `localhost` lại
trỏ về **máy họ** — nơi chẳng có Yotea nào cả. Họ sẽ thấy lỗi "không kết nối được".

> 📖 **Thuật ngữ:** *localhost* — tên gọi vòng-lặp-nội-bộ (loopback) chỉ chính thiết bị đang
> chạy. Muốn người khác vào được, web phải nằm trên một máy chủ có **địa chỉ công khai**
> (public IP) và thường gắn với một **domain** (ví dụ `yotea.vercel.app`).

Deploy, nói ngắn gọn, là **mang code từ máy bạn đặt lên một máy chủ công khai** rồi cho
nó một cái tên (domain) để mọi người gõ vào.

### 1.2. Sơ đồ luồng: dev vs prod

```
DEVELOPMENT (trên máy bạn)
  Trình duyệt ──> localhost:3000 (React dev server)
                        │  gọi API
                        ▼
                  localhost:8080 (Express + nodemon)
                        │
                        ▼
                  MongoDB localhost:27017


PRODUCTION (trên internet)
  Trình duyệt của khách ──> yotea.vercel.app        (frontend tĩnh, CDN)
   (bất kỳ đâu)                   │  gọi API
                                  ▼
                        yotea-api.onrender.com       (Express + pm2/node)
                                  │
                                  ▼
                        MongoDB Atlas (cluster đám mây)
```

Ba mảnh — **frontend**, **backend**, **database** — lúc dev nằm chung một máy, lúc prod
tách ra ba nơi khác nhau. Phần lớn lỗi deploy là do **ba mảnh này gọi nhầm địa chỉ của nhau**.

---

## 2. Build frontend: `npm run build`

### 2.1. `npm start` khác `npm run build` thế nào?

Nhìn lại script của frontend, `yotea-fe/package.json:36-42`:

```json
  "scripts": {

    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test",
    "eject": "react-scripts eject"
  },
```

| Lệnh | Làm gì | Dùng khi nào |
|---|---|---|
| `npm start` | Bật một **server dev** ở cổng 3000, tự reload khi bạn sửa code, giữ file ở dạng dễ đọc | Lúc đang code |
| `npm run build` | Biên dịch toàn bộ React thành **file tĩnh** (HTML + CSS + JS đã nén) trong thư mục `build/`, **không** chạy server nào | Trước khi deploy |

Chạy lệnh build (đứng tại thư mục `yotea-fe`):

```bash
# đứng tại thư mục yotea-fe
npm run build
```

Sau khoảng 30–60 giây, xuất hiện thư mục mới:

```
yotea-fe/build/
├── index.html
├── asset-manifest.json
└── static/
    ├── css/main.<hash>.css      ← toàn bộ CSS đã gộp + nén
    └── js/main.<hash>.js        ← toàn bộ React đã gộp + minify
```

Terminal in ra kích thước từng file, đại loại:

```
File sizes after gzip:
  148.2 kB  build/static/js/main.a1b2c3d4.js
  6.4 kB    build/static/css/main.e5f6a7b8.css
```

### 2.2. "Bundle", "minify" là gì?

- **Bundle (gộp gói):** hàng trăm file `.js` bạn viết được gộp lại thành một (vài) file lớn,
  để trình duyệt tải ít lượt hơn → nhanh hơn.
- **Minify (rút gọn):** xoá mọi khoảng trắng, đổi tên biến `formatCurrency` thành `t`, bỏ
  comment... File nhỏ đi nhiều lần. Con người đọc không nổi, nhưng máy chạy y hệt.
- **Hash trong tên file** (`main.a1b2c3d4.js`): mỗi lần build lại mà nội dung đổi thì hash đổi
  theo, buộc trình duyệt tải bản mới thay vì dùng bản cũ trong cache.

Thư mục `build/` này chính là thứ ta sẽ đẩy lên Vercel/Netlify — nó **không cần Node.js để
chạy**, chỉ là file tĩnh.

### 2.3. ⚠️ Biến `REACT_APP_API_URL` bị "nướng cứng" vào bundle

Ở [Bài 34](34-refactor-du-an.md) bạn đã đổi `baseURL` hardcode thành biến môi trường. Nhắc
lại tình trạng gốc của dự án, `yotea-fe/src/api/instance.js:3-5`:

```js
const instance = axios.create({
  baseURL: "http://localhost:8080/api",
});
```

Và sau khi refactor ở Bài 34, dòng này trở thành:

```js
// đã sửa ở Bài 34 — đọc từ biến môi trường
baseURL: process.env.REACT_APP_API_URL,
```

> ⚠️ **Chỗ cực kỳ dễ sai khi deploy:** Với Create React App, mọi biến `REACT_APP_*` được
> **thay bằng giá trị chuỗi ngay lúc `npm run build`** rồi *đóng băng* vào file JS. Trình
> duyệt của khách **không** đọc `.env` lúc chạy — nó chỉ chạy chuỗi đã bị "nướng cứng" vào
> bundle. Hệ quả: **bạn phải đặt đúng `REACT_APP_API_URL` trỏ tới backend thật TRƯỚC KHI
> build.** Build xong mới đổi biến thì phải **build lại**, sửa `.env` không có tác dụng.

Cụ thể, trước khi build cho production, file `yotea-fe/.env` (hoặc `.env.production`) phải là:

```bash
# yotea-fe/.env.production  ← file bạn tự tạo
REACT_APP_API_URL=https://yotea-api.onrender.com/api
```

Chứ **không** phải `http://localhost:8080/api`. Nếu quên, web deploy xong sẽ trắng trang vì
nó cố gọi `localhost` (máy khách chẳng có backend nào) — đây là lỗi #1 trong mục 🐞 cuối bài.

> 💡 **Mẹo kiểm tra nhanh:** build xong, mở `build/static/js/main.<hash>.js`, bấm `Ctrl+F` gõ
> `onrender.com` (hoặc domain backend của bạn). Nếu tìm thấy → đã nướng đúng. Nếu chỉ thấy
> `localhost:8080` → bạn quên đổi biến trước khi build.

---

## 3. Chuẩn bị backend cho production

### 3.1. Biến môi trường (đã làm ở Bài 34)

Bài 33 đã chỉ ra: secret `"TuongVy"` bị hardcode ở 4 nơi, connection string MongoDB hardcode
trong `app.js`. Bài 34 đã gom chúng vào `.env`. Nhắc lại các biến backend cần có khi lên prod:

| Biến | Ví dụ giá trị | Thay cho chỗ hardcode nào |
|---|---|---|
| `PORT` | do hosting tự cấp | `app.js:51` fallback `8080` |
| `MONGO_URI` | `mongodb+srv://...` (chuỗi Atlas, mục 3.2) | `app.js:46` `mongodb://localhost:27017/yotea` |
| `JWT_SECRET` | chuỗi ngẫu nhiên dài | `"TuongVy"` (ký + verify token) |
| `CLIENT_URL` | `https://yotea.vercel.app` | dùng để giới hạn CORS (mục 3.4) |

> 🔒 **Ghi chú bảo mật:** `.env` **không bao giờ** được commit lên GitHub (phải nằm trong
> `.gitignore`). Trên hosting, bạn nhập các biến này vào dashboard chứ không đẩy file `.env` lên.

### 3.2. MongoDB Atlas — database đám mây miễn phí

MongoDB local (`localhost:27017`) chỉ sống trên máy bạn. Máy chủ backend ngoài internet không
với tới nó được. Giải pháp: dùng **MongoDB Atlas** — dịch vụ database đám mây của chính hãng
MongoDB, có **bậc miễn phí M0** (512 MB, quá đủ cho đồ án).

**Các bước tạo cluster:**

1. Vào [mongodb.com/cloud/atlas](https://www.mongodb.com/cloud/atlas), đăng ký tài khoản (miễn phí).
2. Bấm **Build a Database** → chọn gói **M0 Free** → chọn nhà cung cấp (AWS) và vùng gần bạn
   (ví dụ Singapore) → **Create**.
3. Màn hình **Security Quickstart** hiện ra:
   - **Create a database user:** đặt **username** và **password** (ghi lại password, sẽ nhét vào
     connection string). Tránh ký tự đặc biệt như `@`, `:`, `/` trong password vì chúng làm hỏng chuỗi URI.
   - **Add IP address (whitelist):** đây là bước hay bị quên. Atlas mặc định **chặn mọi kết nối**;
     bạn phải khai địa chỉ IP được phép vào. Vì backend hosting có IP hay đổi, chọn **Allow access
     from anywhere** (`0.0.0.0/0`) cho đồ án học tập.

> ⚠️ **`0.0.0.0/0` nghĩa là "cho mọi IP trên đời kết nối tới database".** Với đồ án học thì tiện,
> nhưng production thật **không nên** làm vậy — chỉ nên whitelist đúng dải IP của máy chủ backend.
> Bù lại, database vẫn được bảo vệ bằng username/password, nên đừng dùng mật khẩu yếu.

4. Lấy **connection string:** bấm **Connect** → **Drivers** → chọn Node.js. Bạn nhận được chuỗi kiểu:

```
mongodb+srv://yotea_user:<password>@cluster0.ab1cd.mongodb.net/?retryWrites=true&w=majority
```

Thay `<password>` bằng mật khẩu thật, và thêm tên database `yotea` ngay trước dấu `?`:

```
mongodb+srv://yotea_user:matkhauthat@cluster0.ab1cd.mongodb.net/yotea?retryWrites=true&w=majority
```

Chuỗi này chính là giá trị của biến `MONGO_URI`. Để ý: local dùng schema `mongodb://`, còn Atlas
dùng `mongodb+srv://` (có thêm `+srv`).

> 💡 **Mẹo:** Database trên Atlas ban đầu **rỗng** — không có sản phẩm, danh mục, topping nào.
> Bạn phải tự tạo lại dữ liệu (đăng ký admin, thêm sản phẩm qua trang admin), hoặc dùng công cụ
> `mongodump`/`mongorestore` để bê dữ liệu local lên. Đừng hoảng khi thấy web deploy xong trống trơn.

### 3.3. Chạy production bằng `node`/`pm2` thay vì `nodemon`

Nhớ lại script backend, `yotea-be/package.json:7`:

```json
"start": "nodemon ./src/app.js --exec babel-node -e js",
```

`nodemon` sinh ra để **theo dõi file và restart mỗi khi bạn sửa code** — tuyệt cho lúc dev,
nhưng vô nghĩa (và tốn tài nguyên) trên production vì code prod không đổi. `babel-node` cũng
biên dịch **lại mỗi lần chạy**, chậm.

Cách chuẩn cho production là **build trước bằng Babel** rồi chạy bằng `node` thuần:

```json
// hướng thêm vào yotea-be/package.json — bạn tự viết thêm
"scripts": {
  "start": "nodemon ./src/app.js --exec babel-node -e js",
  "build": "babel src -d dist",
  "serve": "node dist/app.js"
}
```

- `npm run build` → Babel dịch `src/` thành CommonJS trong thư mục `dist/`.
- `npm run serve` → `node dist/app.js` chạy bản đã dịch, nhanh và ổn định.

> ⚠️ **Chỗ dự án gốc còn thiếu:** `yotea-be/package.json` **không có script `build`** — nghĩa là
> dự án gốc chưa hề có đường deploy production chuẩn. Đây là việc bạn phải tự thêm.

**pm2** là một cấp cao hơn: nó là **trình quản lý tiến trình** giúp app **tự khởi động lại khi
crash**, chạy nền, và ghi log. Trên máy chủ VPS bạn sẽ gõ:

```bash
npm install -g pm2
pm2 start dist/app.js --name yotea-api
pm2 save
```

Còn trên các hosting như **Render/Railway** (mục 5), bạn **không cần đụng tới pm2** — nền tảng
tự lo việc giữ app sống. Bạn chỉ cần khai lệnh build và lệnh start trong dashboard.

### 3.4. ⚠️ CORS production: đừng để mở cho mọi domain

Nhìn lại middleware CORS của dự án, `yotea-be/src/app.js:25-26`:

```js
app.use(express.json());
app.use(cors());
```

`cors()` gọi **không kèm option** nghĩa là gắn header `Access-Control-Allow-Origin: *` —
**bất kỳ website nào trên internet** cũng gọi được API Yotea từ trình duyệt. Lúc dev thì tiện,
nhưng production nên **giới hạn** chỉ cho phép đúng domain frontend của bạn:

```js
// hướng sửa cho production — bạn tự viết thêm
app.use(
  cors({
    origin: process.env.CLIENT_URL, // ví dụ https://yotea.vercel.app
    credentials: true,
  })
);
```

> 🔒 **Vì sao quan trọng:** để `*` nghĩa là một trang web lừa đảo bất kỳ cũng có thể nhúng
> JavaScript gọi tới API của bạn thay mặt người dùng đang đăng nhập. Giới hạn `origin` về đúng
> domain frontend là hàng rào cơ bản chống loại lạm dụng đó.

> 💡 **Mẹo:** Nếu bạn có nhiều domain frontend (ví dụ cả `yotea.vercel.app` lẫn domain riêng),
> truyền một **mảng** origin, hoặc một **hàm** kiểm tra origin nằm trong danh sách trắng hay không.

---

## 4. Ba lựa chọn deploy (đều có bậc miễn phí)

Không cần thuê server đắt tiền. Ba mảnh của Yotea có ba nhóm dịch vụ miễn phí quen thuộc:

| Mảnh | Dịch vụ gợi ý | Bậc miễn phí | Ghi chú |
|---|---|---|---|
| **Frontend** (file tĩnh) | **Vercel** hoặc **Netlify** | Có, hào phóng | Kết nối repo GitHub, tự build & phát hành qua CDN |
| **Backend** (Node/Express) | **Render** hoặc **Railway** | Có (Render free tier ngủ sau 15 phút không dùng) | Chạy được server luôn-bật |
| **Database** (MongoDB) | **MongoDB Atlas** | Có (M0, 512 MB) | Đã làm ở mục 3.2 |

> 💡 **Về "ngủ đông" của Render free:** service miễn phí của Render **tự ngủ sau ~15 phút** không
> có request; request đầu tiên sau khi ngủ mất ~30–50 giây để "thức dậy". Với đồ án học thì chấp
> nhận được — chỉ cần biết để không tưởng là web bị lỗi khi lần đầu tải chậm.

Từ đây, ta chọn **một combo cụ thể** để hướng dẫn: **Vercel (frontend) + Render (backend) +
Atlas (database)**. Đây là combo phổ biến nhất cho người mới vì cả ba đều đăng nhập bằng
GitHub và thao tác gần như chỉ bấm nút.

### 4.1. Deploy backend lên Render

1. Đẩy dự án lên GitHub (nếu chưa). **Kiểm tra `node_modules/` KHÔNG bị commit** (xem checklist
   mục 6 — dự án gốc lỡ commit hơn 8000 file `node_modules`).
2. Vào [render.com](https://render.com), đăng nhập bằng GitHub.
3. Bấm **New +** → **Web Service** → chọn repo Yotea → chọn **Root Directory** là `yotea-be`.
4. Điền cấu hình:
   - **Build Command:** `npm install && npm run build` (chạy `babel src -d dist`).
   - **Start Command:** `npm run serve` (tức `node dist/app.js`).
5. Mục **Environment** → **Add Environment Variable**, thêm từng biến (không đẩy file `.env` lên):

```
MONGO_URI   = mongodb+srv://yotea_user:matkhauthat@cluster0.ab1cd.mongodb.net/yotea?retryWrites=true&w=majority
JWT_SECRET  = <chuỗi-ngẫu-nhiên-dài-của-bạn>
CLIENT_URL  = https://yotea.vercel.app
```

   (Không cần đặt `PORT` — Render tự cấp và app đọc qua `process.env.PORT`, đúng như `app.js:51`.)
6. Bấm **Create Web Service**. Render clone repo, chạy build, rồi start. Vài phút sau bạn có URL kiểu
   `https://yotea-api.onrender.com`. Mở `https://yotea-api.onrender.com/api/products` để kiểm tra.

### 4.2. Deploy frontend lên Vercel

1. Vào [vercel.com](https://vercel.com), đăng nhập bằng GitHub.
2. **Add New...** → **Project** → chọn repo Yotea → **Root Directory** là `yotea-fe`.
3. Vercel tự nhận diện Create React App (Framework Preset = *Create React App*), tự điền:
   - **Build Command:** `npm run build`
   - **Output Directory:** `build`
4. Mục **Environment Variables**, thêm:

```
REACT_APP_API_URL = https://yotea-api.onrender.com/api
```

   **Đặt biến này TRƯỚC khi bấm Deploy** — nhớ mục 2.3: biến bị nướng cứng vào bundle lúc build.
5. Bấm **Deploy**. Vercel chạy `npm run build`, phát hành thư mục `build/` lên CDN, cho bạn URL
   kiểu `https://yotea.vercel.app`.
6. Quay lại Render, cập nhật `CLIENT_URL` thành đúng URL Vercel vừa nhận, để CORS (mục 3.4) cho
   frontend gọi API. Render sẽ tự deploy lại backend.

> 💡 **Mẹo về React Router:** Yotea dùng `createBrowserRouter` (Bài 15). Với các trang con như
> `/thuc-don`, nếu người dùng **F5 (tải lại)** trên đó, một số host trả 404 vì không có file
> `/thuc-don`. Vercel xử lý sẵn cho Create React App (mọi đường dẫn trả về `index.html`). Nếu
> deploy nơi khác như Netlify, bạn cần thêm file cấu hình rewrite `/* → /index.html`.

---

## 5. Checklist trước khi deploy (rất quan trọng)

Đây là lúc gom **mọi thứ đã học** ở Bài 33 và 34 lại. Đừng bấm Deploy khi còn ô chưa tick.

| ✅ | Việc | Vì sao | Bài liên quan |
|---|---|---|---|
| ☐ | Đã tách `.env`, `.env` nằm trong `.gitignore` | Không lộ secret lên GitHub | [34](34-refactor-du-an.md) |
| ☐ | Đã đổi secret `"TuongVy"` thành chuỗi ngẫu nhiên dài, đặt qua `JWT_SECRET` | Ai đọc source cũ đều ký được token admin giả | [33](33-ra-soat-bao-mat.md) #4 |
| ☐ | Đã vá lỗ hổng phân quyền (updateInfo tự phong admin, users trả password, order mở toang) | Web thật sẽ bị khai thác thật | [33](33-ra-soat-bao-mat.md) #1,#2,#3 |
| ☐ | Đã **bỏ `node_modules/` khỏi git**, có `.gitignore` cho cả FE lẫn BE | Dự án gốc lỡ commit hơn 8000 file; repo phình, deploy chậm | [33](33-ra-soat-bao-mat.md) |
| ☐ | Đã giới hạn **CORS** về `CLIENT_URL` thay vì `cors()` mở `*` | Chặn domain lạ gọi API | mục 3.4 |
| ☐ | Đã đổi **baseURL** frontend sang `REACT_APP_API_URL` trỏ backend thật, và build lại | Nếu không, web trắng trang vì gọi `localhost` | mục 2.3 |
| ☐ | Ảnh Cloudinary đã dùng **tài khoản của mình** (đổi `levantuan`/`kkio3wiw`) | Dự án gốc hardcode tài khoản người khác, có thể bị xoá bất cứ lúc nào | [32](32-trang-quan-tri.md) |
| ☐ | Đã có script `build`/`serve` cho backend | Dự án gốc chỉ chạy được dev | mục 3.3 |
| ☐ | MongoDB Atlas đã tạo cluster, đã whitelist IP, đã có `MONGO_URI` | Backend không có DB thì mọi API treo | mục 3.2 |

> ⚠️ **Chỗ dự án gốc cực kỳ dễ bị bỏ sót — Cloudinary.** Nhắc lại từ Bài 32, hàm upload ảnh
> `yotea-fe/src/utils/index.js` hardcode cloud name `levantuan` và preset `kkio3wiw` — **không
> phải tài khoản của bạn**. Deploy production mà giữ nguyên thì: (1) ảnh có thể chết nếu chủ
> tài khoản xoá preset; (2) bạn đang bơm ảnh vào Cloudinary của người lạ. Hãy tạo tài khoản
> Cloudinary miễn phí của riêng bạn, tạo unsigned preset, và thay hai giá trị đó.

---

## 6. 🛠️ Tự tay làm — deploy database + frontend

> Mục tiêu phần này: cuối phần, bạn có một database Atlas chạy thật, và (tuỳ chọn) một trang
> Yotea sống trên internet mà bạn gửi link cho bạn bè mở được.

### Bước 1 — Đưa dự án lên GitHub (bỏ `node_modules`)

Tạo/kiểm tra file `.gitignore` ở gốc mỗi thư mục:

```bash
# yotea-be/.gitignore  ← file bạn tự tạo (dự án gốc thiếu)
node_modules/
dist/
.env
```

```bash
# yotea-fe/.gitignore  (dự án gốc đã có sẵn phần này)
node_modules/
build/
.env
.env.production
```

Nếu `node_modules` **đã lỡ bị commit** trước đó, gỡ nó khỏi git (không xoá file trên đĩa):

```bash
# đứng tại gốc repo
git rm -r --cached yotea-be/node_modules
git commit -m "Remove node_modules from git"
```

Rồi đẩy lên GitHub:

```bash
git push origin main
```

### Bước 2 — Tạo MongoDB Atlas (làm thật ngay bây giờ)

Làm theo **mục 3.2**:

1. Đăng ký [Atlas](https://www.mongodb.com/cloud/atlas), tạo cluster **M0 Free**.
2. Tạo **database user** (nhớ username + password).
3. **Network Access** → Add IP → **Allow access from anywhere** (`0.0.0.0/0`).
4. **Connect** → **Drivers** → copy connection string, thay `<password>`, thêm `/yotea` trước `?`.

Kết quả bạn có một chuỗi `MONGO_URI` dùng được. Test nhanh trên máy: tạm đặt biến này vào
`yotea-be/.env` rồi `npm start` — nếu terminal in `Connected to MongoDB` là Atlas đã thông.

### Bước 3 — Build frontend trỏ về backend thật

Tạo `yotea-fe/.env.production` (backend của bạn có thể chưa deploy — dùng URL Render dự kiến, hoặc
làm bước này lại sau khi có URL thật):

```bash
# yotea-fe/.env.production  ← file bạn tự tạo
REACT_APP_API_URL=https://yotea-api.onrender.com/api
```

Rồi build:

```bash
# đứng tại yotea-fe
npm run build
```

Kiểm tra bundle đã nướng đúng URL (mục 2.3): mở `build/static/js/main.<hash>.js`, tìm `onrender.com`.

### Bước 4 — Deploy frontend lên Vercel

Làm theo **mục 4.2**: Add Project → chọn repo → Root `yotea-fe` → thêm biến `REACT_APP_API_URL`
→ Deploy. Nhận URL `https://<tên-bạn>.vercel.app`.

> 💡 Nếu backend chưa lên Render, frontend vẫn deploy được nhưng gọi API sẽ lỗi. Bạn có thể làm
> **Bước 4 (backend, mục 4.1)** trước, lấy URL Render, rồi mới quay lại đặt `REACT_APP_API_URL`
> và deploy frontend. Thứ tự đúng: **Atlas → Render (backend) → Vercel (frontend)**.

### Bước 5 — Nối ba mảnh và thử thật

1. Trên Render, đặt `CLIENT_URL` = URL Vercel (cho CORS).
2. Mở URL Vercel trên **điện thoại** (dùng 4G, không dùng wifi nhà bạn) — nếu vào được nghĩa là
   web đã thật sự công khai, không còn phụ thuộc máy bạn.
3. Đăng ký một tài khoản trên web deploy → thử xem sản phẩm, thêm giỏ hàng, đặt đơn.

---

## 7. ✅ Kiểm chứng kết quả

Bạn deploy thành công khi **tất cả** các mục sau đúng:

1. Mở URL backend `https://yotea-api.onrender.com/api/products` trên trình duyệt → nhận về JSON
   (dù là mảng rỗng `[]` nếu database Atlas chưa có dữ liệu):

```json
[]
```

2. Mở URL frontend `https://yotea.vercel.app` → trang chủ Yotea hiện lên bình thường, **không**
   trắng trang, **không** báo lỗi đỏ trong Console (F12).
3. Mở tab **Network** (F12) → tải lại trang → các request API trỏ tới domain Render, **không** còn
   `localhost:8080`, và trả về status **200**.
4. Đăng ký / đăng nhập được; token lưu trong `localStorage` như lúc dev.
5. Gửi link cho một người bạn (hoặc mở bằng mạng 4G điện thoại) → họ vào được.

> 💡 **Mẹo debug tại chỗ:** mọi lỗi deploy đều để lại dấu vết ở một trong hai chỗ: **Console/Network
> của trình duyệt** (lỗi phía frontend/CORS) hoặc **Logs của Render** (lỗi phía backend/database).
> Mở cả hai lên trước khi đoán mò.

---

## 8. 🐞 Lỗi thường gặp

| Triệu chứng | Nguyên nhân | Cách sửa |
|---|---|---|
| **Trang trắng**, Console báo gọi `localhost:8080` thất bại | Quên đổi `REACT_APP_API_URL` trước khi build; bundle vẫn trỏ localhost | Đặt đúng biến rồi **build lại** & deploy lại (mục 2.3) |
| Console báo **`blocked by CORS policy`** | Backend chưa cho phép origin của frontend | Đặt `CLIENT_URL` = URL Vercel trong `cors({ origin })` và trên dashboard Render (mục 3.4) |
| Backend log **`MongooseServerSelectionError` / `Could not connect`** | Atlas chưa whitelist IP, hoặc `MONGO_URI` sai password | Network Access → thêm `0.0.0.0/0`; kiểm tra password trong chuỗi URI (mục 3.2) |
| Backend crash ngay khi start, log **`undefined is not a valid secret`** hoặc tương tự | Quên đặt biến môi trường trên dashboard | Vào Render → Environment → thêm đủ `MONGO_URI`, `JWT_SECRET`, `CLIENT_URL` |
| API trả JSON đúng nhưng web vẫn **trống dữ liệu** | Database Atlas rỗng (khác database local) | Tự tạo dữ liệu trên web deploy, hoặc `mongorestore` từ local lên Atlas (mục 3.2) |
| F5 trên `/thuc-don` ra **404** | Host chưa rewrite mọi route về `index.html` | Vercel + CRA xử lý sẵn; nơi khác cần thêm rule rewrite `/* → /index.html` (mục 4.2) |
| Lần đầu tải web **chậm ~30–50 giây** rồi sau đó nhanh | Render free tier vừa "ngủ dậy" | Bình thường với bậc miễn phí; nâng gói trả phí nếu cần luôn-bật (mục 4) |
| Ảnh không lên / lỗi upload | Vẫn dùng Cloudinary `levantuan`/`kkio3wiw` của người khác | Đổi sang tài khoản Cloudinary của bạn (mục 5) |
| Deploy build fail, log **`babel: command not found`** | Backend thiếu script `build`, hoặc babel nằm ngoài dependency được cài | Thêm script `build: "babel src -d dist"` và đảm bảo babel-cli được cài (mục 3.3) |

---

## 9. 📝 Bài tập

**Bài 1.** Bạn build frontend, deploy lên Vercel, nhưng web trắng trang và Console báo request tới
`http://localhost:8080/api/products` bị lỗi. Bạn vào `.env` trên máy sửa `REACT_APP_API_URL` thành
URL Render rồi refresh trình duyệt — vẫn trắng trang. Vì sao? Sửa đúng như thế nào?

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Vì với Create React App, biến `REACT_APP_*` được **thay bằng giá trị và đóng băng vào bundle ngay
lúc `npm run build`**. Sửa `.env` **sau khi đã build** không thay đổi được file JS đã phát hành —
trình duyệt vẫn chạy chuỗi `localhost:8080` cũ đã nướng cứng.

Sửa đúng: đổi biến **trước**, rồi **build lại** (`npm run build`) và deploy lại. Trên Vercel, bạn
đặt `REACT_APP_API_URL` trong **Environment Variables** của dự án rồi bấm **Redeploy** để Vercel
build lại với biến mới. Kiểm chứng bằng cách tìm chuỗi domain backend trong `build/static/js/main.<hash>.js`.

</details>

**Bài 2.** Backend đã deploy lên Render, mở `https://yotea-api.onrender.com/api/products` bằng trình
duyệt thì thấy JSON bình thường. Nhưng frontend gọi API lại bị chặn với lỗi CORS. Nghịch lý ở đâu?

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Không nghịch lý. Khi bạn **gõ thẳng URL vào thanh địa chỉ**, trình duyệt tải trực tiếp — **không có
khái niệm "origin khác"**, nên CORS không áp dụng. CORS chỉ kiểm khi **một trang web ở domain A**
(frontend Vercel) dùng JavaScript gọi tới **domain B** (backend Render).

Vì backend gốc để `cors()` mở `*` sẽ **không** bị lỗi này; lỗi CORS xuất hiện khi bạn đã giới hạn
`origin` (mục 3.4) nhưng đặt **sai** giá trị `CLIENT_URL` (ví dụ thiếu `https://`, thừa dấu `/` cuối,
hoặc chưa cập nhật sau khi Vercel đổi URL). Sửa: đặt `CLIENT_URL` **khớp chính xác** origin của
frontend (không kèm đường dẫn, không dấu `/` thừa) rồi để Render deploy lại.

</details>

**Bài 3.** (suy ngẫm) Liệt kê ba lỗ hổng từ [Bài 33](33-ra-soat-bao-mat.md) mà **bắt buộc** phải vá
trước khi deploy public, và giải thích vì sao lúc chạy `localhost` chúng "vô hại" nhưng lên internet
lại thành thảm hoạ.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Ba lỗ hổng nguy hiểm nhất cần vá trước:

1. **`updateInfo` cho tự phong admin** (`routes/users.js` thiếu `isAdmin` + `const update = req.body`):
   bất kỳ khách đăng nhập nào gửi `{"role": 1}` là thành admin.
2. **`GET /api/users` công khai trả cả `password`**: lộ toàn bộ email, sđt, địa chỉ, chuỗi băm mật khẩu.
3. **`order`/`orderDetail` gần như mở toang**: ai cũng liệt kê và xoá đơn hàng của mọi khách.

Lúc chạy `localhost`, **chỉ mình bạn** truy cập được máy → không ai khai thác. Khi lên internet, URL
công khai cho **cả thế giới** — bất kỳ ai cũng gọi được API, dò được endpoint, và tự động hoá tấn công.
Nói cách khác, deploy **không tạo ra** lỗ hổng, nó chỉ **mở cửa** cho người khác chạm vào những lỗ hổng
vốn đã tồn tại. Đó là lý do checklist bảo mật (mục 5) phải xong **trước** nút Deploy.

Cùng nhóm gốc bệnh — **mass assignment** (`const update = req.body`) và **không kiểm chủ sở hữu** —
đã bàn ở Bài 33 và vá ở Bài 34.

</details>

---

## 📌 Tóm tắt

- **`localhost` = chính máy bạn**; muốn cả thế giới vào phải đặt web lên máy chủ công khai có domain.
- Ba mảnh Yotea tách ra ba nơi khi deploy: **frontend (Vercel)** · **backend (Render)** · **database (Atlas)**.
- `npm run build` sinh thư mục `build/` gồm file tĩnh đã **bundle + minify**, khác hẳn server dev `npm start`.
- ⚠️ `REACT_APP_API_URL` bị **nướng cứng vào bundle lúc build** — phải đặt đúng backend thật **trước khi** build, đổi sau phải build lại.
- Backend production: đọc config từ **biến môi trường**, database dùng **MongoDB Atlas** (nhớ **whitelist IP**), chạy bằng `node dist` / `pm2` chứ không `nodemon`.
- ⚠️ CORS production phải **giới hạn `origin`** về đúng domain frontend, không để `cors()` mở `*`.
- **Checklist trước deploy** gom cả bảo mật (Bài 33) lẫn refactor (Bài 34): tách `.env`, đổi secret, vá lỗ hổng, bỏ `node_modules` khỏi git, giới hạn CORS, đổi baseURL, đổi Cloudinary.
- Lỗi deploy hay gặp: trắng trang do sai API URL · CORS bị chặn · quên đặt biến trên dashboard · Atlas chưa whitelist IP.

**Từ khoá tra cứu thêm:** `create react app build`, `environment variables react production`, `MongoDB Atlas free tier`, `deploy express render`, `deploy react vercel`, `CORS origin whitelist`, `pm2 process manager`

---

## 🎉 Bạn đã hoàn thành giáo trình!

Xin **chúc mừng bạn!** Từ Bài 01 đến đây, bạn đã đi trọn một vòng của một web full-stack thật:

- **Backend:** Express, Mongoose, CRUD, slug, bộ lọc query, quan hệ & populate, mã hoá mật khẩu, JWT, phân quyền, Swagger.
- **Frontend:** React, Router v6, layout, Tailwind, axios, Redux Toolkit, thunk, redux-persist, RTK Query.
- **Nghiệp vụ:** đăng nhập, danh sách & chi tiết sản phẩm, giỏ hàng, thanh toán, tài khoản, bình luận/đánh giá, trang quản trị.
- **Trưởng thành nghề:** bạn đã tự **rà soát bảo mật** một dự án thật, **refactor** nó, **tự tay xây một chức năng end-to-end** (Topping, rồi Voucher ở đồ án cuối), và giờ là **đưa nó lên internet**.

Quan trọng hơn cả kỹ thuật: bạn đã học cách **đọc một codebase có thật — kể cả những chỗ nó làm
sai — và biết vì sao sai, sửa thế nào**. Đó là kỹ năng theo bạn suốt sự nghiệp, ở bất kỳ dự án nào.

**Học tiếp gì bây giờ?** Vài hướng tự nhiên nối từ những gì bạn vừa làm:

- **TypeScript** — thêm kiểu tĩnh cho cả BE lẫn FE, bắt lỗi ngay lúc gõ code thay vì lúc chạy. Nhớ
  đống bug "undefined is not a function" trong giáo trình? TypeScript chặn phần lớn chúng.
- **Next.js** — framework React "đàn anh" của CRA: server-side rendering, routing theo file, tốt cho SEO.
- **Testing** — Jest + React Testing Library (frontend) và Supertest (API). Dự án Yotea gần như không có
  test; viết test là bước trưởng thành lớn tiếp theo.
- **Docker** — đóng gói backend + database vào container để "chạy giống nhau trên mọi máy", tiền đề của deploy chuyên nghiệp.

Hãy lấy chính Yotea làm sân tập: chuyển nó sang TypeScript, viết test cho vài API, hay Docker hoá nó.
Không có cách học nào nhanh bằng cải tạo một dự án mình đã hiểu tường tận.

Cảm ơn bạn đã đi cùng tới dòng cuối. Chúc bạn viết nên những sản phẩm thật của riêng mình. 🧋
