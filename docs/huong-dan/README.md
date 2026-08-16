# 🧋 Học lập trình Full-stack qua dự án Yotea

> Giáo trình 36 bài, đi từ con số 0 đến làm chủ toàn bộ website bán trà sữa **Yotea** —
> từ dòng `app.use()` đầu tiên cho tới lúc tự tay thêm một chức năng hoàn chỉnh và deploy.

---

## 👋 Giáo trình này dành cho ai?

| Bạn là… | Nên đọc |
|---|---|
| Mới học lập trình, chưa từng làm web | Bắt đầu từ **Bài 01**, làm tuần tự, không bỏ bài nào |
| Đã biết HTML/CSS/JS cơ bản | Đọc lướt Phần 0, làm kỹ từ **Bài 04** |
| Đã biết Node.js, muốn học React | Đọc lướt Phần 1–2, làm kỹ từ **Bài 14** |
| Đã biết React, muốn học backend | Làm Phần 1–2 (**Bài 04–13**), rồi nhảy sang Phần 5 |
| Cần hiểu nhanh dự án để bảo trì | Đọc **Bài 01**, **Bài 06**, **Bài 14**, rồi tra bài nghiệp vụ cần thiết |

**Yêu cầu tối thiểu:** biết dùng máy tính, biết mở terminal, biết HTML/CSS cơ bản.
Mọi thứ còn lại giáo trình sẽ dạy.

---

## 🏗️ Dự án Yotea là gì?

Một website bán trà sữa hoàn chỉnh gồm **2 phần chạy song song**:

```
┌─────────────────────────┐        HTTP/JSON        ┌──────────────────────────┐        ┌────────────┐
│      yotea-fe           │  ───────────────────▶   │       yotea-be           │  ───▶  │  MongoDB   │
│  React 18 + Redux       │                         │  Express + Mongoose      │        │  yotea     │
│  http://localhost:3000  │  ◀───────────────────   │  http://localhost:8080   │  ◀───  │  :27017    │
└─────────────────────────┘                         └──────────────────────────┘        └────────────┘
     Người dùng nhìn thấy                              Xử lý nghiệp vụ + bảo mật            Lưu dữ liệu
```

**Phía khách hàng:** xem thực đơn, lọc/tìm kiếm sản phẩm, xem chi tiết, chọn mức đá –
mức đường, thêm giỏ hàng, đặt hàng, bình luận, đánh giá sao, yêu thích sản phẩm,
đọc tin tức, xem hệ thống cửa hàng, gửi liên hệ, quản lý tài khoản và lịch sử đơn hàng.

**Phía quản trị:** dashboard, CRUD sản phẩm – danh mục – tin tức – slider – tài khoản,
duyệt đơn hàng, kiểm duyệt bình luận, xử lý liên hệ.

---

## 🗺️ Lộ trình 36 bài

### Phần 0 · Khởi động

Hiểu bức tranh tổng thể và dựng được môi trường chạy dự án.

| # | Bài học | Bạn sẽ làm được gì |
|---|---|---|
| 01 | [Tổng quan dự án Yotea & kiến trúc full-stack](01-tong-quan-du-an.md) | Vẽ được sơ đồ dự án, biết file nào nằm ở đâu |
| 02 | [Cài đặt môi trường và chạy dự án lần đầu](02-cai-dat-moi-truong.md) | Chạy được cả FE lẫn BE trên máy mình |
| 03 | [Kiến thức nền: ES6, async/await, HTTP/REST, MongoDB](03-kien-thuc-nen.md) | Đọc hiểu cú pháp xuất hiện khắp dự án |

### Phần 1 · Backend cơ bản

Xây được một API CRUD hoàn chỉnh bằng Express + Mongoose.

| # | Bài học | Bạn sẽ làm được gì |
|---|---|---|
| 04 | [Mổ xẻ `app.js`: Express, middleware, CORS, morgan](04-express-va-appjs.md) | Hiểu server khởi động ra sao |
| 05 | [Mongoose Model: Schema, kiểu dữ liệu, timestamps, index](05-mongoose-model.md) | Tự thiết kế được một model mới |
| 06 | [Vòng đời một request: Router → Controller → Model](06-vong-doi-mot-request.md) | Lần theo được đường đi của mọi request |
| 07 | [CRUD trọn vẹn với Category + test bằng Postman](07-crud-category.md) | Viết đủ 5 API và tự test |
| 08 | [Slug thân thiện SEO với slugify](08-slug-slugify.md) | Sinh URL đẹp từ tên tiếng Việt |

### Phần 2 · Backend nâng cao

Những kỹ thuật khiến API trở nên "thật": lọc dữ liệu, quan hệ, bảo mật, tài liệu.

| # | Bài học | Bạn sẽ làm được gì |
|---|---|---|
| 09 | [Bộ lọc kiểu json-server: `_sort`, `_limit`, `_expand`, `_like`, `q`](09-bo-loc-query.md) | Hiểu và mở rộng bộ lọc dùng chung |
| 10 | [Quan hệ dữ liệu & `populate`](10-quan-he-va-populate.md) | Nối Product ↔ Category, Order ↔ OrderDetail |
| 11 | [Mã hoá mật khẩu và xác thực bằng JWT](11-mat-khau-va-jwt.md) | Tự làm chức năng đăng ký / đăng nhập |
| 12 | [Phân quyền: `requireSignin`, `isAuth`, `isAdmin`](12-phan-quyen-middleware.md) | Chặn API chỉ cho admin |
| 13 | [Viết tài liệu API tự động với Swagger](13-swagger-tai-lieu-api.md) | Bật được trang `/api-docs` |

### Phần 3 · Frontend cơ bản

Hiểu bộ khung React của dự án và cách nó nói chuyện với backend.

| # | Bài học | Bạn sẽ làm được gì |
|---|---|---|
| 14 | [Cấu trúc dự án React & luồng khởi động](14-cau-truc-react-app.md) | Biết code chạy từ đâu tới đâu |
| 15 | [React Router v6: `createBrowserRouter`, layout lồng nhau](15-routing-v6.md) | Thêm một trang mới vào web |
| 16 | [Layout và component tái sử dụng](16-layout-va-component.md) | Tách UI thành component gọn gàng |
| 17 | [Tailwind CSS trong dự án](17-tailwind-css.md) | Đọc và sửa được giao diện |
| 18 | [Tầng gọi API với axios](18-tang-api-axios.md) | Viết hàm gọi API đúng chuẩn dự án |

### Phần 4 · Quản lý state

Redux — nơi dữ liệu dùng chung của toàn ứng dụng sinh sống.

| # | Bài học | Bạn sẽ làm được gì |
|---|---|---|
| 19 | [Redux Toolkit: slice, action, reducer, selector](19-redux-toolkit-co-ban.md) | Tự tạo một slice mới |
| 20 | [`createAsyncThunk` và `extraReducers`](20-async-thunk.md) | Gọi API bất đồng bộ qua Redux |
| 21 | [redux-persist: giữ giỏ hàng và phiên đăng nhập](21-redux-persist.md) | F5 không mất giỏ hàng |
| 22 | [RTK Query: `createApi`, cache và `invalidatesTags`](22-rtk-query.md) | Thay 50 dòng thunk bằng 1 hook |

### Phần 5 · Từng chức năng, từ đầu đến cuối

Mỗi bài đi trọn một chức năng: giao diện → Redux → API → controller → database.

| # | Bài học | Bạn sẽ làm được gì |
|---|---|---|
| 23 | [Đăng ký / Đăng nhập / Đăng xuất](23-dang-ky-dang-nhap.md) | Nối trọn luồng xác thực FE ↔ BE |
| 24 | [`PrivateRouter` — chặn route theo quyền](24-private-router.md) | Khách vãng lai không vào được `/admin` |
| 25 | [Danh sách sản phẩm: lọc, sắp xếp, phân trang, tìm kiếm](25-danh-sach-san-pham.md) | Làm trang thực đơn hoàn chỉnh |
| 26 | [Chi tiết sản phẩm: lượt xem, đá/đường, sản phẩm liên quan](26-chi-tiet-san-pham.md) | Trang chi tiết đầy đủ tính năng |
| 27 | [Giỏ hàng](27-gio-hang.md) | Thêm / sửa / xoá / tính tiền |
| 28 | [Thanh toán: Order + OrderDetail, react-hook-form + yup](28-thanh-toan.md) | Đặt hàng thành công và lưu DB |
| 29 | [Tài khoản của tôi: sửa thông tin, đổi mật khẩu, lịch sử đơn](29-tai-khoan-cua-toi.md) | Khu vực cá nhân của khách |
| 30 | [Bình luận, đánh giá sao và yêu thích sản phẩm](30-binh-luan-danh-gia-yeu-thich.md) | 3 chức năng tương tác |
| 31 | [Tin tức, liên hệ, cửa hàng và slider trang chủ](31-tin-tuc-lien-he-cua-hang.md) | Các trang nội dung |
| 32 | [Trang quản trị: CRUD, phân trang, upload ảnh Cloudinary](32-trang-quan-tri.md) | Toàn bộ khu vực admin |

### Phần 6 · Nâng cao & hoàn thiện

Từ "chạy được" lên "làm nghề được".

| # | Bài học | Bạn sẽ làm được gì |
|---|---|---|
| 33 | [Rà soát bảo mật: dự án đang sai ở đâu](33-ra-soat-bao-mat.md) | Nhìn ra lỗ hổng và biết cách vá |
| 34 | [Refactor: `.env`, `configureStore`, xử lý lỗi tập trung](34-refactor-du-an.md) | Nâng cấp code lên chuẩn hiện đại |
| 35 | [🎓 Đồ án cuối khoá: làm chức năng Voucher end-to-end](35-do-an-cuoi-voucher.md) | Tự làm 1 chức năng từ số 0 |
| 36 | [Build và deploy lên môi trường thật](36-build-va-deploy.md) | Đưa web lên internet |

---

## 📐 Mỗi bài học có gì?

```
🎯 Sau bài này bạn sẽ    →  mục tiêu rõ ràng, đo được
📋 Cần chuẩn bị          →  kiến thức/công cụ tiên quyết
1. Lý thuyết             →  giải thích bằng ví dụ đời thường trước
2. Soi code thật         →  trích nguyên văn code dự án, kèm đường dẫn:số dòng
3. 🛠️ Tự tay làm          →  từng bước gõ code, có file mới để bạn tự viết
4. ✅ Kiểm chứng          →  chạy thế nào, phải thấy kết quả gì
5. 🐞 Lỗi thường gặp      →  bảng lỗi ↔ nguyên nhân ↔ cách sửa
6. 📝 Bài tập             →  có lời giải gấp lại, đừng mở vội
📌 Tóm tắt               →  chốt kiến thức trong 5 gạch đầu dòng
```

---

## 🚀 Bắt đầu ngay

Chưa cài gì cả? Nhảy thẳng vào **[Bài 02 — Cài đặt môi trường](02-cai-dat-moi-truong.md)**
để có dự án chạy được trên máy trong ~20 phút, rồi quay lại
**[Bài 01](01-tong-quan-du-an.md)** để hiểu mình vừa chạy cái gì.

Muốn hiểu trước rồi mới làm? Bắt đầu từ
**[Bài 01 — Tổng quan dự án Yotea](01-tong-quan-du-an.md)**.

---

## 💡 Lời khuyên để học hiệu quả

1. **Gõ lại, đừng copy-paste.** Ngón tay nhớ lâu hơn con mắt.
2. **Làm hết phần "Tự tay làm"** trước khi sang bài mới — kiến thức các bài xếp chồng lên nhau.
3. **Bí quá 15 phút thì đọc mục "Lỗi thường gặp"**, chưa ra thì mở lời giải, nhưng phải gõ lại.
4. **Mở song song 2 cửa sổ:** một bên giáo trình, một bên VS Code.
5. **Đừng bỏ qua các hộp ⚠️.** Dự án này là bài tập lớn của sinh viên nên có nhiều chỗ làm
   chưa chuẩn — biết chỗ sai và biết vì sao sai chính là lúc bạn lên trình.
