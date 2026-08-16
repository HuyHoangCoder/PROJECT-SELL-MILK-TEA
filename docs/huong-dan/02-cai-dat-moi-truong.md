# Bài 02 — Cài đặt môi trường và chạy dự án lần đầu

> **Phần 0 · Khởi động** — Thời lượng ước tính: **~45 phút**
> ⬅️ Bài trước: [01 — Tổng quan dự án Yotea & kiến trúc full-stack](01-tong-quan-du-an.md) · Bài sau: [03 — Kiến thức nền tối thiểu](03-kien-thuc-nen.md) ➡️
> 🏠 [Mục lục](README.md)

---

## 🎯 Sau bài này bạn sẽ

- Cài xong **Node.js**, **MongoDB**, **MongoDB Compass**, **Postman** và **VS Code**.
- Chạy được backend ở cổng 8080 và frontend ở cổng 3000 **cùng lúc**.
- Nhìn thấy trang chủ Yotea hiện ra trong trình duyệt.
- Tự tạo được tài khoản **admin** đầu tiên và vào được trang quản trị.
- Thêm được dữ liệu mẫu (danh mục + sản phẩm) để những bài sau có cái mà xem.
- Biết cách đọc log trong terminal để đoán ra mình đang hỏng ở khâu nào.

## 📋 Cần chuẩn bị

- Máy Windows / macOS / Linux, còn trống ít nhất **2 GB** ổ cứng.
- Kết nối internet (bước cài thư viện tải khá nhiều dữ liệu).
- Đã đọc [Bài 01](01-tong-quan-du-an.md) để biết mình đang chạy cái gì.

---

## 1. Bộ đồ nghề cần cài

| Phần mềm | Dùng để làm gì | Bắt buộc? |
|---|---|---|
| **Node.js 18 LTS** | Chạy JavaScript ngoài trình duyệt — cả FE lẫn BE đều cần | ✅ Bắt buộc |
| **MongoDB Community Server** | Cơ sở dữ liệu lưu sản phẩm, đơn hàng, tài khoản | ✅ Bắt buộc |
| **MongoDB Compass** | Giao diện xem/sửa dữ liệu trong MongoDB bằng chuột | ✅ Rất nên có |
| **Visual Studio Code** | Soạn thảo code | ✅ Bắt buộc |
| **Postman** | Gọi thử API mà không cần frontend | ✅ Rất nên có (dùng nhiều từ [Bài 07](07-crud-category.md)) |
| **Git** | Quản lý phiên bản code | ⭕ Nên có |

> 💡 **Mẹo:** cài **Node.js 18 LTS**, đừng cài bản mới nhất. Dự án dùng một số thư viện
> khá cũ (`babel-cli` 6, `react-scripts` 5), bản Node quá mới dễ sinh lỗi lạ. Đồng thời
> backend có dùng `Object.hasOwn()` — hàm này chỉ có từ **Node 16.9 trở lên**, nên cũng
> đừng cài bản quá cũ. Vùng an toàn: **Node 16.9 – 20**.

### 1.1. Cài Node.js

Tải bản **LTS** tại <https://nodejs.org> rồi cài như phần mềm bình thường (bấm Next liên tục).

Kiểm tra sau khi cài — mở **PowerShell** (Windows) hoặc **Terminal** (macOS/Linux):

```bash
node -v
npm -v
```

Kết quả mong đợi (con số có thể khác chút):

```
v18.20.4
10.7.0
```

Nếu báo `'node' is not recognized...` → Node chưa được thêm vào PATH. Cách nhanh nhất
là **đóng hết cửa sổ terminal, mở lại**. Vẫn lỗi thì cài lại Node và nhớ tick
*"Add to PATH"*.

### 1.2. Cài MongoDB

Tải **MongoDB Community Server** tại
<https://www.mongodb.com/try/download/community>.

Khi cài trên Windows, ở màn hình cài đặt hãy **tick "Install MongoDB as a Service"** —
như vậy MongoDB tự chạy nền mỗi lần bật máy, bạn không phải khởi động thủ công.
Nhớ tick luôn **"Install MongoDB Compass"**.

Kiểm tra MongoDB đã chạy chưa:

```powershell
# Windows PowerShell
Get-Service MongoDB
```

Phải thấy `Status: Running`. Nếu là `Stopped`:

```powershell
net start MongoDB
```

Trên macOS (cài bằng Homebrew):

```bash
brew services start mongodb-community
```

### 1.3. Cài VS Code và Postman

- VS Code: <https://code.visualstudio.com>
- Postman: <https://www.postman.com/downloads/>

Vài extension VS Code rất đáng cài cho dự án này:

| Extension | Lợi ích |
|---|---|
| **ES7+ React/Redux snippets** | Gõ `rafce` + Tab là ra sẵn khung một component |
| **Tailwind CSS IntelliSense** | Gợi ý tên class Tailwind khi gõ |
| **Prettier** | Tự canh lề code cho đẹp |
| **MongoDB for VS Code** | Xem dữ liệu ngay trong editor |

---

## 2. 🛠️ Tự tay làm — chạy dự án

> Mục tiêu phần này: cuối phần bạn sẽ có **2 cửa sổ terminal chạy song song** và
> trang chủ Yotea hiện ra ở `http://localhost:3000`.

### Bước 1 — Mở dự án bằng VS Code

Mở VS Code → **File → Open Folder** → chọn thư mục gốc `PROJECT-SELL-MILK-TEA`
(thư mục chứa cả `yotea-be` và `yotea-fe`).

Bạn phải nhìn thấy hai thư mục con này ở khung bên trái. Nếu chỉ thấy một, bạn đã mở
nhầm cấp thư mục.

### Bước 2 — Cài thư viện cho backend

Mở terminal trong VS Code (`Ctrl + ~`), rồi:

```bash
cd yotea-be
npm install
```

Lệnh `npm install` đọc file `package.json` và tải toàn bộ thư viện về thư mục
`node_modules`. Lần đầu chạy mất khoảng **2–5 phút**.

> 💡 **Mẹo:** dự án này đã có sẵn `node_modules` được commit vào git (điều mà một dự án
> chuẩn **không nên** làm). Nếu thư mục đó đã tồn tại, bạn vẫn nên chạy `npm install`
> một lần cho chắc chắn — lệnh này an toàn, chạy lại bao nhiêu lần cũng được.

### Bước 3 — Bật backend

Vẫn đứng trong `yotea-be`:

```bash
npm start
```

Lệnh này thực chất chạy dòng script trong `yotea-be/package.json:7`:

```json
"start": "nodemon ./src/app.js --exec babel-node -e js"
```

Dịch ra tiếng Việt: *"dùng **nodemon** theo dõi mọi file `.js`, chạy `src/app.js` thông
qua **babel-node** (để hiểu được cú pháp `import`), và tự khởi động lại mỗi khi bạn lưu file."*

Terminal phải in ra:

```
[nodemon] starting `babel-node ./src/app.js`
App is running on port: 8080
Connected to MongoDB
```

> ⚠️ Thấy dòng `Connected to MongoDB` mới là **thật sự thành công**. Nếu chỉ thấy
> `App is running on port: 8080` mà không có dòng kia, MongoDB của bạn chưa chạy —
> xem mục [Lỗi thường gặp](#5--lỗi-thường-gặp).

**Để nguyên cửa sổ terminal này, đừng tắt.** Backend phải chạy suốt.

### Bước 4 — Bật frontend ở cửa sổ terminal MỚI

Mở terminal thứ hai (trong VS Code bấm dấu `+` ở góc phải khung terminal):

```bash
cd yotea-fe
npm install
npm start
```

`npm install` của frontend lâu hơn backend, tầm **3–8 phút**. Kiên nhẫn.

Sau `npm start`, terminal in ra:

```
Compiled successfully!

You can now view reactjs-v2 in the browser.

  Local:            http://localhost:3000
```

Trình duyệt sẽ **tự mở** `http://localhost:3000`.

### Bước 5 — Kiểm tra 2 tiến trình đang chạy song song

Bây giờ bạn phải có:

```
Terminal 1 (yotea-be)   →  App is running on port: 8080     ← ĐỪNG TẮT
Terminal 2 (yotea-fe)   →  Compiled successfully!            ← ĐỪNG TẮT
```

Mỗi lần muốn code, bạn sẽ luôn phải bật cả hai cửa sổ này.

---

## 3. 🛠️ Tự tay làm — tạo dữ liệu đầu tiên

Vào lúc này trang chủ mở được nhưng **trống trơn**: chưa có sản phẩm, chưa có danh mục,
chưa có tài khoản nào. Vì dự án **không kèm dữ liệu mẫu**, ta phải tự tạo.

### Bước 1 — Đăng ký tài khoản

Vào `http://localhost:3000/register`, điền form và bấm đăng ký.

Điền gì cũng được, nhưng hãy nhớ kỹ **email và mật khẩu** — ví dụ:

```
Email:     admin@gmail.com
Mật khẩu:  admin123
```

### Bước 2 — Xem dữ liệu vừa tạo trong MongoDB Compass

Mở **MongoDB Compass** → ở ô connection string gõ:

```
mongodb://localhost:27017
```

→ bấm **Connect**. Bạn sẽ thấy database tên **`yotea`** (nó được tạo tự động lúc
backend ghi bản ghi đầu tiên), bên trong có collection **`users`** chứa tài khoản
bạn vừa đăng ký.

Bấm vào bản ghi đó, để ý hai trường:

```json
{
  "role": 0,
  "active": 1
}
```

### Bước 3 — Tự phong mình làm admin

Trong dự án này:

| Trường | Giá trị | Nghĩa |
|---|---|---|
| `role` | `0` | Khách hàng bình thường |
| `role` | `1` | Quản trị viên |
| `active` | `1` | Tài khoản đang hoạt động |
| `active` | `0` | Tài khoản bị khoá |

Trong Compass, bấm biểu tượng **bút chì (Edit document)** ở bản ghi của bạn, sửa
`role` từ `0` thành `1`, rồi bấm **UPDATE**.

### Bước 4 — Đăng nhập lại và vào trang quản trị

Quay lại trình duyệt, **đăng xuất rồi đăng nhập lại** (bắt buộc — thông tin quyền được
lưu vào trình duyệt ngay lúc đăng nhập, không tự cập nhật).

Bây giờ vào `http://localhost:3000/admin` — bạn phải thấy trang quản trị.

> 💡 **Vì sao phải đăng nhập lại?** Vì quyền admin được lưu trong Redux và ghi xuống
> `localStorage` của trình duyệt tại thời điểm đăng nhập. Bạn sửa dưới database nhưng
> bản sao trên trình duyệt vẫn là bản cũ. Cơ chế này sẽ được mổ ở
> [Bài 21 — redux-persist](21-redux-persist.md).

### Bước 5 — Thêm danh mục và sản phẩm

Trong trang quản trị, làm **đúng thứ tự** này:

1. **Vào `Danh mục` → `Thêm danh mục`** — tạo trước ít nhất 2 danh mục, ví dụ
   `Trà sữa` và `Trà trái cây`.
2. **Vào `Sản phẩm` → `Thêm sản phẩm`** — tạo vài sản phẩm, mỗi sản phẩm phải **chọn
   một danh mục**.

Bắt buộc phải theo thứ tự này vì trong `yotea-be/src/models/product.js` trường
`categoryId` được khai báo `required: true` — không có danh mục thì không thể tạo sản phẩm.

Quay lại `http://localhost:3000/thuc-don`, sản phẩm bạn vừa thêm phải hiện ra.

🎉 Đến đây bạn đã có một dự án chạy được với dữ liệu thật.

---

## 4. ✅ Kiểm chứng kết quả

Đánh dấu từng mục, đủ hết mới sang bài sau:

- [ ] `node -v` in ra phiên bản từ v16.9 trở lên.
- [ ] MongoDB đang chạy (`Get-Service MongoDB` → Running).
- [ ] Terminal backend in đủ **cả hai dòng** `App is running on port: 8080` và `Connected to MongoDB`.
- [ ] Terminal frontend in `Compiled successfully!`.
- [ ] Mở `http://localhost:3000` thấy giao diện Yotea (header, banner, footer).
- [ ] Mở `http://localhost:8080/api/products` trên trình duyệt thấy JSON (có thể là `[]` nếu chưa có sản phẩm — thế là **đúng**, không phải lỗi).
- [ ] Trong Compass thấy database `yotea` với collection `users`.
- [ ] Vào được `http://localhost:3000/admin`.
- [ ] Trang `/thuc-don` hiển thị sản phẩm bạn vừa thêm.

### Bài test API đầu tiên bằng Postman

Mở Postman → **New → HTTP Request**:

```
Method: GET
URL:    http://localhost:8080/api/products
```

Bấm **Send**. Khung dưới phải trả về JSON dạng:

```json
[
  {
    "_id": "6650a1f2c4e8b91234abcd01",
    "name": "Trà sữa trân châu đường đen",
    "price": 35000,
    "slug": "tra-sua-tran-chau-duong-den",
    "categoryId": { "_id": "...", "name": "Trà sữa" },
    "createdAt": "2026-08-15T09:12:00.000Z"
  }
]
```

Đây chính xác là dữ liệu mà frontend nhận được. Từ giờ, mỗi khi giao diện hiển thị sai,
việc đầu tiên nên làm là **gọi API bằng Postman** để biết lỗi nằm ở FE hay BE.

---

## 5. 🐞 Lỗi thường gặp

| Thông báo lỗi | Nguyên nhân | Cách sửa |
|---|---|---|
| `MongooseServerSelectionError: connect ECONNREFUSED ::1:27017` | MongoDB chưa chạy | Windows: `net start MongoDB` · macOS: `brew services start mongodb-community` |
| `Error: listen EADDRINUSE: address already in use :::8080` | Cổng 8080 đang bị chiếm (thường do backend cũ chưa tắt) | Tìm và tắt: `netstat -ano \| findstr :8080` rồi `taskkill /PID <số> /F` |
| `Something is already running on port 3000` | Frontend cũ chưa tắt | Gõ `Y` để React tự đổi sang cổng 3001, hoặc tắt tiến trình cũ |
| Trang chủ trắng, Console báo `ERR_CONNECTION_REFUSED` | Backend chưa chạy | Bật lại terminal `yotea-be` bằng `npm start` |
| `Access-Control-Allow-Origin` / lỗi CORS | Backend chạy nhưng thiếu `app.use(cors())` | Kiểm tra `yotea-be/src/app.js:26` còn dòng `app.use(cors())` không |
| `'nodemon' is not recognized` | Chưa chạy `npm install` trong `yotea-be` | `cd yotea-be` rồi `npm install` |
| `Cannot find module 'babel-preset-env'` | Cài thư viện dở dang | Xoá `node_modules` + `package-lock.json` rồi `npm install` lại |
| `Object.hasOwn is not a function` | Node.js quá cũ (< 16.9) | Nâng Node lên bản 18 LTS |
| Vào `/admin` bị đá về trang chủ | `role` vẫn đang là `0`, hoặc chưa đăng nhập lại sau khi sửa | Sửa `role = 1` trong Compass rồi **đăng xuất → đăng nhập lại** |
| Thêm sản phẩm báo lỗi | Chưa có danh mục nào | Tạo danh mục trước, rồi mới tạo sản phẩm |
| Ảnh sản phẩm không lên được | Chức năng upload dùng tài khoản Cloudinary của tác giả gốc, có thể đã hết hạn | Tạm dán thẳng một URL ảnh bất kỳ vào ô ảnh. Chi tiết ở [Bài 32](32-trang-quan-tri.md) |
| Sửa code mà giao diện không đổi | Trình duyệt cache | `Ctrl + Shift + R` để tải lại và bỏ cache |

### Cách đọc log để đoán bệnh

Nhờ có `morgan` (`yotea-be/src/app.js:27`), mỗi request tới backend đều được in ra terminal:

```
GET /api/products 200 45.123 ms - 1234
POST /api/signin 400 12.456 ms - 41
```

Cách đọc:

| Phần | Ý nghĩa |
|---|---|
| `GET` / `POST` | Loại hành động |
| `/api/products` | Đường dẫn được gọi |
| `200` | **Mã trạng thái** — 200 là OK, 400 là request sai, 401 là chưa đăng nhập, 404 là không tìm thấy, 500 là backend nổ |
| `45.123 ms` | Thời gian xử lý |

> 💡 **Mẹo vàng khi debug:** bấm nút trên giao diện mà **terminal backend không in
> thêm dòng nào** → request chưa hề bay tới backend, lỗi nằm ở **frontend**.
> Còn nếu có in dòng mới với mã `400`/`500` → lỗi nằm ở **backend**. Chỉ một mẹo này
> đã giúp bạn khoanh vùng được 90% sự cố.

---

## 6. 📝 Bài tập

**Bài 1.** Cố tình tắt MongoDB rồi khởi động lại backend. Ghi lại **nguyên văn** thông
báo lỗi. Sau đó bật lại MongoDB và khởi động backend lần nữa.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

```powershell
net stop MongoDB
# sang terminal yotea-be, Ctrl+C rồi npm start
```

Terminal sẽ in `App is running on port: 8080` **nhưng không có** dòng
`Connected to MongoDB`, kèm một khối lỗi dài bắt đầu bằng
`MongooseServerSelectionError: connect ECONNREFUSED`.

Điều đáng chú ý: **server Express vẫn chạy bình thường**, chỉ database là chết. Lý do
nằm ở `yotea-be/src/app.js:45-48` — phần kết nối Mongoose dùng `.catch()` chỉ để
`console.log(error)` chứ không dừng chương trình. Kết quả là server "sống dở": nhận
request nhưng mọi truy vấn đều thất bại. Một dự án chuẩn nên **thoát hẳn** khi không
kết nối được database. Ta sẽ sửa ở [Bài 34](34-refactor-du-an.md).

```powershell
net start MongoDB
```

</details>

**Bài 2.** Dùng Postman gọi `POST http://localhost:8080/api/signin` với body JSON là
email và mật khẩu bạn vừa đăng ký. Ghi lại kết quả trả về.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Trong Postman: tab **Body** → chọn **raw** → chọn kiểu **JSON**, dán vào:

```json
{
  "email": "admin@gmail.com",
  "password": "admin123"
}
```

Kết quả (mã 200):

```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2...",
  "user": {
    "_id": "6650a1f2c4e8b91234abcd01",
    "email": "admin@gmail.com",
    "fullName": "Quản trị viên",
    "role": 1,
    "active": 1
  }
}
```

Chuỗi `token` dài ngoằng đó là **JWT** — tấm vé thông hành chứng minh bạn đã đăng nhập.
Frontend sẽ đính kèm nó vào mọi request cần quyền admin. Toàn bộ cơ chế này ở
[Bài 11](11-mat-khau-va-jwt.md).

Thử gõ sai mật khẩu → nhận mã **400** kèm `{ "message": "Mật khẩu không chính xác" }`.
Để ý là **thông báo lỗi nói rõ sai mật khẩu chứ không phải sai email** — tiện cho người
dùng nhưng lại giúp kẻ xấu dò xem email nào có tồn tại trong hệ thống. Chi tiết ở
[Bài 33 — Rà soát bảo mật](33-ra-soat-bao-mat.md).

</details>

**Bài 3.** Sửa file `yotea-be/src/app.js`, đổi dòng `console.log("Connected to MongoDB")`
thành `console.log("Đã kết nối MongoDB thành công!")`, rồi **lưu file mà không tắt terminal**.
Quan sát điều gì xảy ra.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Ngay khi bạn nhấn `Ctrl + S`, terminal tự in ra:

```
[nodemon] restarting due to changes...
[nodemon] starting `babel-node ./src/app.js`
App is running on port: 8080
Đã kết nối MongoDB thành công!
```

Đó là công của **nodemon** — nó theo dõi file và tự khởi động lại server. Nhờ vậy bạn
không phải tắt/bật tay sau mỗi lần sửa code backend.

Frontend cũng có cơ chế tương tự nhưng còn "xịn" hơn, gọi là **Hot Reload**: sửa code
xong trình duyệt tự cập nhật mà **không mất** dữ liệu đang có trên trang.

Nhớ đổi lại dòng log về như cũ để đồng bộ với phần còn lại của giáo trình.

</details>

---

## 📌 Tóm tắt

- Cần 4 thứ: **Node 16.9–20**, **MongoDB**, **VS Code**, **Postman**. Thêm **Compass** để xem dữ liệu bằng chuột.
- Luôn phải mở **2 terminal**: một cho `yotea-be` (cổng 8080), một cho `yotea-fe` (cổng 3000).
- `npm install` chỉ cần chạy lần đầu (hoặc khi thêm thư viện mới); `npm start` chạy mỗi lần code.
- Backend khởi động thành công khi có **đủ 2 dòng** log: `App is running on port: 8080` **và** `Connected to MongoDB`.
- Dự án **không có dữ liệu mẫu** — phải tự đăng ký tài khoản, sửa `role = 1` trong Compass để thành admin, rồi tạo danh mục **trước**, sản phẩm **sau**.
- Mẹo debug quan trọng nhất: nhìn log của `morgan` trong terminal backend. Không có dòng mới → lỗi ở FE. Có dòng mới với mã 400/500 → lỗi ở BE.

**Từ khoá tra cứu thêm:** `npm install`, `nodemon`, `babel-node`, `localhost port`, `MongoDB Compass`, `HTTP status code`

➡️ **Bài tiếp theo:** [03 — Kiến thức nền tối thiểu](03-kien-thuc-nen.md) — những cú pháp JavaScript xuất hiện dày đặc trong dự án mà nếu không biết bạn sẽ đọc code như đọc chữ tượng hình.
