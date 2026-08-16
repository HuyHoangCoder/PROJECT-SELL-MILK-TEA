# Bài 33 — Rà soát bảo mật: dự án đang sai ở đâu

> **Phần 6 · Nâng cao & hoàn thiện** — Thời lượng ước tính: **~90 phút**
> ⬅️ Bài trước: [32 — Trang quản trị: CRUD, phân trang, upload ảnh Cloudinary](32-trang-quan-tri.md) · Bài sau: [34 — Refactor: `.env`, `configureStore`, xử lý lỗi tập trung](34-refactor-du-an.md) ➡️
> 🏠 [Mục lục](README.md)

---

## 🎯 Sau bài này bạn sẽ

- Nhìn một dự án và tự đặt được câu hỏi "chỗ này có an toàn không?" thay vì chỉ hỏi "chỗ này có chạy không?".
- Chỉ ra và giải thích được **8 lỗ hổng thật** trong Yotea, xếp theo mức độ nguy hiểm.
- Tự tay **khai thác** vài lỗ hổng bằng Postman để hiểu vì sao chúng nguy hiểm (trên chính máy bạn — đây là bài học phòng thủ, không phải để tấn công ai).
- Biết nguyên tắc nền tảng: **không bao giờ tin dữ liệu từ client**.
- Nắm được hướng vá cho từng lỗi (chi tiết cách sửa để dành [Bài 34](34-refactor-du-an.md)).

## 📋 Cần chuẩn bị

- Đã học hết Phần 5, đặc biệt [Bài 11](11-mat-khau-va-jwt.md), [Bài 12](12-phan-quyen-middleware.md), [Bài 21](21-redux-persist.md), [Bài 29](29-tai-khoan-cua-toi.md).
- Dự án đang chạy được, có ít nhất 1 tài khoản admin và 1 tài khoản khách thường.
- Có Postman.

> ⚠️ **Lời nói đầu quan trọng.** Dự án Yotea là bài tập lớn của sinh viên. Việc nó có
> lỗ hổng là **hoàn toàn bình thường và không có gì đáng xấu hổ** — gần như mọi dự án
> đầu tay đều thế. Mục tiêu của bài này không phải chê bai, mà là biến chính những lỗi
> đó thành bài học đắt giá nhất bạn có được trong cả giáo trình. Biết một hệ thống sai
> ở đâu và vì sao, đó là lúc bạn thật sự trưởng thành nghề.

---

## 1. Một nguyên tắc, gói gọn mọi bài học

> **KHÔNG BAO GIỜ TIN DỮ LIỆU TỪ CLIENT.**

Frontend chạy trên máy người dùng. Mọi thứ ở đó — JavaScript, giá tiền, quyền admin,
`localStorage` — người dùng đều **xem được, sửa được, giả mạo được**. Vì thế:

- Kiểm tra ở frontend chỉ để **trải nghiệm mượt** (báo lỗi sớm, ẩn nút). Nó **không** bảo vệ được gì.
- Mọi quyết định quan trọng (bạn là ai, được làm gì, giá bao nhiêu) **phải** do backend
  tự tính lại từ nguồn nó tin được: **token đã ký** và **dữ liệu trong database**.

Bảy trong tám lỗ hổng dưới đây đều là biến thể của việc **quên mất nguyên tắc này**.

---

## 2. Bảng phân quyền thật của toàn bộ API

Trước khi soi lỗi, hãy nhìn bức tranh tổng. Bảng dưới đây được lập bằng cách đọc **tất
cả 14 file** trong `yotea-be/src/routes/`. Ký hiệu:

- 🌍 = công khai, ai cũng gọi được (không middleware)
- 🔑 = cần đăng nhập (`requireSignin` + `isAuth`)
- 👑 = cần là admin (`requireSignin` + `isAuth` + `isAdmin`)

| Resource | GET (đọc) | POST (tạo) | PUT (sửa) | DELETE (xoá) | Ghi chú |
|---|:---:|:---:|:---:|:---:|---|
| category | 🌍 | 👑 | 👑 | 👑 | chuẩn |
| product | 🌍 | 👑 | 👑 | 👑 | + `PATCH userUpdate` 🌍 |
| news, cateNews, slider, store | 🌍 | 👑 | 👑 | 👑 | chuẩn |
| contact | 🌍 | 🌍 | 👑 | 👑 | POST mở để khách gửi liên hệ |
| comment, rating, favoritesProduct | 🌍 | 🔑 | 🔑 | 🔑 | **thiếu `isAdmin`, thiếu kiểm chủ sở hữu** |
| **users** | **🌍** | 👑 | 👑 | 👑 | **GET công khai, trả cả mật khẩu** |
| users `/updateInfo` | — | — | 🔑 | — | **chỉ 🔑, không kiểm `:myId`** |
| **order** | **🌍** | **🌍** | 🔑 | **🌍** | **gần như mở toang** |
| **orderDetail** | **🌍** | **🌍** | **🌍** | **🌍** | **mở hoàn toàn** |

Những ô **in đậm** là nơi có vấn đề. Ta đi qua từng cái.

---

## 3. Tám lỗ hổng, xếp theo mức nguy hiểm

### 🔴 Lỗ hổng #1 — Ai cũng tự phong mình làm admin

**Mức độ:** nghiêm trọng nhất. Chiếm được toàn quyền hệ thống.

Xem API cập nhật thông tin cá nhân trong `yotea-be/src/routes/users.js:12`:

```js
router.put("/users/updateInfo/:myId/:userId", requireSignin, isAuth, update);
```

Và controller nó gọi, `yotea-be/src/controllers/user.js:225-227`:

```js
export const update = async (req, res) => {
    const filter = { _id: req.params.id || req.params.myId };
    const update = req.body;
```

Ba sự thật ghép lại thành thảm hoạ:

1. Route này chỉ có `requireSignin, isAuth` — **không** có `isAdmin`. Bất kỳ khách nào đăng nhập đều gọi được.
2. `isAuth` (`middlewares/checkAuth.js:9-19`) chỉ kiểm `req.profile._id == req.auth._id`, tức chỉ so **`:userId`** với token. Nó **không hề đụng tới `:myId`** — mà `:myId` mới là người bị sửa.
3. Controller lấy nguyên `req.body` nhét thẳng vào `findOneAndUpdate`, **không lọc trường nào cả**.

Kết quả: một khách thường có thể gửi `{"role": 1}` vào chính hồ sơ mình và **trở thành admin**.

**🛠️ Tự tay khai thác (trên máy bạn):**

Đăng nhập bằng tài khoản khách thường, lấy `token` và `_id` của nó (xem trong
`localStorage` → `persist:root` → `auth`, hoặc gọi `POST /api/signin` bằng Postman).
Rồi trong Postman:

```
PUT http://localhost:8080/api/users/updateInfo/<_id-cua-ban>/<_id-cua-ban>
Headers:  Authorization: Bearer <token-cua-ban>
Body (JSON):
{
  "role": 1
}
```

Gọi xong, đăng xuất và đăng nhập lại trên web → bạn đã có menu admin. Tệ hơn, kẻ xấu có
thể đổi `:myId` thành id người khác để **chiếm hoặc phá hồ sơ bất kỳ ai**.

> 🔒 **Vì sao xảy ra:** vi phạm nguyên tắc vàng — backend tin `role` mà client gửi lên.
> **Hướng vá:** lấy id người dùng **từ token** (`req.auth._id`) chứ không từ URL; và chỉ
> cho phép cập nhật một **danh sách trắng** các trường an toàn (`fullName`, `phone`,
> `address`), tuyệt đối chặn `role`, `active`, `password`. Code sửa đầy đủ ở
> [Bài 34](34-refactor-du-an.md) và bạn đã thực hành ở phần "Tự tay làm" của
> [Bài 29](29-tai-khoan-cua-toi.md).

---

### 🔴 Lỗ hổng #2 — API trả về mật khẩu của mọi người

**Mức độ:** nghiêm trọng. Rò rỉ toàn bộ dữ liệu người dùng.

`yotea-be/src/routes/users.js:9-10`:

```js
router.get("/users/:id", read);
router.get("/users", list);
```

Cả hai đều **không có middleware** — hoàn toàn công khai. Tệ hơn, controller `list` và
`read` trả về document user **gần như nguyên vẹn**, chỉ bỏ `__v`. Nghĩa là trường
`password` (chuỗi băm) **cũng nằm trong kết quả**.

**🛠️ Tự tay kiểm chứng:** mở thẳng trình duyệt, gõ `http://localhost:8080/api/users`.
Bạn sẽ thấy JSON liệt kê **mọi tài khoản**, kèm email, số điện thoại, địa chỉ và chuỗi
băm mật khẩu — mà không cần đăng nhập gì cả.

> 🔒 **Vì sao nguy hiểm:** email + sđt + địa chỉ là dữ liệu cá nhân. Chuỗi băm mật khẩu
> tuy đã băm nhưng vì dự án dùng **SHA256 không salt** (xem lỗ hổng #4), kẻ xấu có thể
> tra bảng có sẵn để tìm ra mật khẩu gốc của các mật khẩu yếu.
> **Hướng vá:** bắt buộc `isAdmin` cho `GET /users`; và ở tầng model/controller luôn
> `.select("-password")` để mật khẩu **không bao giờ** rời khỏi server.

---

### 🔴 Lỗ hổng #3 — Đơn hàng của mọi khách bị phơi bày và xoá được

**Mức độ:** nghiêm trọng. Rò rỉ dữ liệu đơn hàng + phá hoại.

`yotea-be/src/routes/order.js:8-12`:

```js
router.post("/orders", create);
router.get("/orders/:id", read);
router.get("/orders", list);
router.put("/orders/:id/:userId", requireSignin, isAuth, update);
router.delete("/orders/:id", remove);
```

Chỉ mỗi `PUT` được bảo vệ. `GET /orders` công khai → ai cũng liệt kê được **mọi đơn
hàng** kèm tên, số điện thoại, email, địa chỉ khách. `DELETE /orders/:id` công khai →
ai cũng **xoá đơn bất kỳ**. Toàn bộ `orderDetail` (`routes/orderDetail.js`) còn mở hơn:
**cả 5 thao tác** đều không có middleware.

> 🔒 **Hướng vá:** `GET /orders` (xem tất cả) phải là 👑; khách chỉ được xem **đơn của
> chính mình** (lọc theo `req.auth._id` lấy từ token, không phải theo `userId` client
> gửi); `DELETE` phải là 👑.

---

### 🟠 Lỗ hổng #4 — Băm mật khẩu yếu và bí mật viết cứng trong code

**Mức độ:** cao.

`yotea-be/src/models/user.js:70-79`:

```js
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
```

Ba vấn đề trong một đoạn ngắn:

1. **Không có salt riêng cho từng người.** Hai người đặt cùng mật khẩu `123456` sẽ có
   **cùng một chuỗi băm**. Kẻ xấu chỉ cần tính trước bảng băm của các mật khẩu phổ biến
   (rainbow table) là dò ngược ra hàng loạt.
2. **SHA256 quá nhanh.** Nó sinh ra để băm nhanh, nên máy tấn công thử được hàng tỷ mật
   khẩu mỗi giây. Băm mật khẩu cần thuật toán **cố tình chậm** như **bcrypt** hoặc **argon2**.
3. **Khoá bí mật `"TuongVy"` viết cứng trong mã nguồn** — và trùng y hệt khoá dùng để ký
   JWT (`controllers/auth.js:52`) lẫn khoá verify token (`middlewares/checkAuth.js:5`).
   Bất kỳ ai đọc được source (dự án lại còn commit cả `node_modules` lên git) đều biết
   khoá này, và có thể **tự ký một token admin giả**.

**Hướng vá (dùng bcrypt):**

```js
// code minh hoạ hướng sửa — sẽ làm ở Bài 34
import bcrypt from "bcryptjs";

// lúc đăng ký:
this.password = await bcrypt.hash(this.password, 10); // 10 = độ "chậm", tự sinh salt

// lúc đăng nhập:
const ok = await bcrypt.compare(passwordNguoiDungGo, this.password);
```

Và đưa khoá bí mật vào biến môi trường `process.env.JWT_SECRET` (xem [Bài 34](34-refactor-du-an.md)).

---

### 🟠 Lỗ hổng #5 — Giá tiền do client quyết định

**Mức độ:** cao. Gây thiệt hại tài chính trực tiếp.

Nhớ lại luồng đặt hàng ở [Bài 28](28-thanh-toan.md): frontend tự tính `totalPrice` rồi
gửi lên, và tự gửi `productPrice` của từng dòng trong `orderDetail`. Controller
`order.create` và `orderDetail.create` **lưu y nguyên** những con số đó, **không hề tra
lại giá thật trong bảng `products`**.

Vì giỏ hàng nằm hoàn toàn trong `localStorage` (xem [Bài 27](27-gio-hang.md)), người
dùng chỉ cần sửa `productPrice` từ `35000` thành `1000` trước khi bấm đặt hàng là mua
được ly trà sữa giá 1.000₫ — và hệ thống ghi nhận đơn hàng hợp lệ.

> 🔒 **Vì sao xảy ra:** lại là "tin client". **Hướng vá:** backend nhận `productId` và
> `quantity` thôi; **tự truy vấn giá từ database** theo `productId`, rồi tự tính
> `totalPrice`. Con số client gửi lên chỉ dùng để hiển thị, không bao giờ để tính tiền.

---

### 🟡 Lỗ hổng #6 — Token nằm trong localStorage (rủi ro XSS)

**Mức độ:** trung bình (phụ thuộc có lỗ XSS khác không).

Dự án lưu JWT trong `localStorage` (qua redux-persist, xem [Bài 21](21-redux-persist.md)).
Mọi đoạn JavaScript chạy trên trang đều đọc được `localStorage`. Nếu ở đâu đó có lỗ
**XSS** (chèn được script lạ — ví dụ qua một bình luận không được làm sạch), script đó
đọc trộm token và mạo danh người dùng.

> 🔒 **Hướng vá:** lưu token trong **cookie `httpOnly`** — loại cookie mà JavaScript
> không đọc được, chỉ trình duyệt tự đính vào request. Đồng thời làm sạch (sanitize)
> mọi nội dung người dùng nhập trước khi hiển thị.

> 💡 **Phân biệt cho rõ (nối [Bài 21](21-redux-persist.md) và [Bài 24](24-private-router.md)):**
> việc **sửa `role` trong localStorage** chỉ qua mặt được `PrivateRouter` phía frontend
> — bạn *nhìn thấy* giao diện admin, nhưng **không** gọi được các API 👑 vì backend vẫn
> kiểm `role` thật trong database. Đó là lý do lỗ hổng #6 này chỉ ở mức trung bình, còn
> lỗ hổng #1 (sửa `role` **trong database** qua API `updateInfo`) mới là nghiêm trọng.

---

### 🟡 Lỗ hổng #7 — Không kiểm chủ sở hữu với bình luận / đánh giá / yêu thích

**Mức độ:** trung bình.

`yotea-be/src/routes/comment.js:11-12` (rating và favoritesProduct y hệt):

```js
router.put("/comments/:id/:userId", requireSignin, isAuth, update);
router.delete("/comments/:id/:userId", requireSignin, isAuth, remove);
```

`isAuth` chỉ xác nhận "`:userId` đúng là bạn". Nhưng controller `update`/`remove` sửa/xoá
bình luận theo **`:id` của bình luận**, mà **không kiểm bình luận đó có phải của bạn
không**. Vậy khách A đăng nhập hợp lệ vẫn có thể **sửa hoặc xoá bình luận của khách B**,
miễn đoán được `:id`.

> 🔒 **Hướng vá:** trong controller, nạp bình luận theo `:id`, so `comment.userId` với
> `req.auth._id`; khác nhau thì trả 403.

---

### 🟡 Lỗ hổng #8 — Rò rỉ email tồn tại + lộ chi tiết lỗi

**Mức độ:** thấp–trung bình.

`yotea-be/src/controllers/auth.js` trả về **hai thông báo khác nhau**: "Email không tồn
tại" và "Mật khẩu không chính xác". Điều này giúp kẻ xấu **dò xem email nào có đăng ký**
trong hệ thống (thử email, thấy báo "sai mật khẩu" tức là email đó tồn tại). Ngoài ra,
nhiều `catch` trả thẳng cả object `error` về client — có thể lộ cấu trúc bên trong.

Cuối cùng, vì `app.js` **không có middleware xử lý lỗi tập trung**, khi token sai/hết
hạn, `express-jwt` ném `UnauthorizedError` và Express trả về **trang HTML kèm stack
trace** — lộ đường dẫn file và cấu trúc dự án.

> 🔒 **Hướng vá:** dùng một thông báo chung "Email hoặc mật khẩu không đúng"; không trả
> object `error` thô; thêm error handler tập trung ([Bài 34](34-refactor-du-an.md)).

---

## 4. 🛠️ Tự tay làm — lập báo cáo bảo mật của riêng bạn

> Mục tiêu: luyện con mắt rà soát. Đây là kỹ năng dùng được cho **mọi** dự án về sau.

### Bước 1 — Tự kiểm chứng 3 lỗ hổng

Dùng Postman, tự tay tái hiện (trên máy mình) lỗ hổng **#1**, **#2**, **#3** ở trên. Với
mỗi cái, chụp lại: request đã gửi, response nhận về, và viết một câu "kẻ xấu làm được gì".

### Bước 2 — Tự đi tìm lỗ hổng thứ 9

Mở `yotea-be/src/controllers/product.js` hàm `list`, và nhớ lại nhánh xử lý `q` (tìm
kiếm) cùng nhánh `*_like` ở [Bài 09](09-bo-loc-query.md):

```js
filter[objectKey] = { $in: [new RegExp(req.query[item], "i")] };
```

Câu hỏi: khi ghép thẳng chuỗi người dùng gõ vào `new RegExp(...)` mà không làm sạch, điều
gì có thể xảy ra? (Gợi ý: tra cụm từ **ReDoS — Regular expression Denial of Service**.)

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Người dùng có thể gửi một biểu thức chính quy **ác ý** (ví dụ một chuỗi khiến regex
engine phải thử số tổ hợp tăng theo cấp số nhân). Một request duy nhất có thể khiến
Node.js — vốn **đơn luồng** — bận xử lý regex và **treo cả server**, không phục vụ được
ai. Đó là một dạng tấn công từ chối dịch vụ (DoS).

Hướng vá: **escape** ký tự đặc biệt trong chuỗi tìm kiếm trước khi tạo RegExp, hoặc dùng
`$text` search (đã có text index) thay cho RegExp tự ghép, hoặc giới hạn độ dài từ khoá.

</details>

### Bước 3 — Viết bảng báo cáo

Lập một bảng theo mẫu dưới đây (đây là **định dạng chuẩn** của một báo cáo bảo mật thật):

| # | Lỗ hổng | Vị trí (`file:line`) | Mức độ | Kẻ xấu làm được gì | Hướng vá |
|---|---|---|---|---|---|
| 1 | Tự phong admin | `routes/users.js:12` | 🔴 | Chiếm quyền admin | Lấy id từ token + whitelist trường |
| … | … | … | … | … | … |

Điền đủ 8–9 lỗ hổng. Giữ lại bảng này — [Bài 34](34-refactor-du-an.md) sẽ lần lượt vá chúng.

---

## 5. ✅ Kiểm chứng kết quả

Bạn nắm được bài khi trả lời được, **không nhìn tài liệu**:

| Câu hỏi | Đáp án cốt lõi |
|---|---|
| Nguyên tắc bảo mật quan trọng nhất là gì? | Không bao giờ tin dữ liệu từ client |
| Vì sao kiểm tra `role` ở `PrivateRouter` không đủ để bảo mật? | Nó chạy ở client, người dùng sửa localStorage là qua mặt được; chỉ backend mới bảo vệ thật |
| Lỗ hổng nghiêm trọng nhất của Yotea? | `updateInfo` cho phép tự gửi `role: 1` để thành admin |
| Vì sao SHA256 không salt là tệ khi băm mật khẩu? | Nhanh (dễ brute-force) + không salt (dễ tra rainbow table); nên dùng bcrypt/argon2 |
| Vì sao không được tin `totalPrice` client gửi lên? | Client sửa được; backend phải tự tra giá từ database |

---

## 6. 🐞 Lỗi thường gặp (khi tự kiểm thử)

| Vấn đề | Nguyên nhân | Cách xử lý |
|---|---|---|
| Gọi `updateInfo` báo 401 | Token sai hoặc `:userId` không khớp id trong token | Dùng đúng token và đúng `_id` của **chính** tài khoản đang đăng nhập |
| Sửa `role` xong vào `/admin` vẫn bị đá ra | Chưa đăng xuất/đăng nhập lại | Quyền lưu ở localStorage lúc đăng nhập — phải đăng nhập lại (xem [Bài 02](02-cai-dat-moi-truong.md)) |
| `GET /api/users` báo lỗi | Backend chưa chạy hoặc chưa có user nào | Bật backend, đăng ký ít nhất 1 tài khoản |
| Không tìm thấy token trong localStorage | Chưa đăng nhập, hoặc tìm sai khoá | Khoá là `persist:root`, bên trong có chuỗi JSON `auth` (xem [Bài 21](21-redux-persist.md)) |

---

## 7. 📝 Bài tập

**Bài 1.** Trong 8 lỗ hổng, hãy chọn ra 3 cái mà **cùng một cách sửa** vá được, và giải
thích cách sửa chung đó.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Lỗ hổng **#1 (tự phong admin)**, **#3 (đơn hàng của người khác)** và **#7 (sửa bình luận
người khác)** đều vá được bằng **cùng một nguyên tắc**: *lấy danh tính người dùng từ
token (`req.auth._id`), và chỉ cho thao tác trên tài nguyên thuộc về chính họ* — thay vì
tin `:userId`/`:myId` trên URL hay tin `userId` trong body.

Cụ thể, viết một middleware `isOwner(Model)` nạp tài nguyên theo `:id`, so trường chủ sở
hữu với `req.auth._id`, khác thì trả 403. Đây chính là bài refactor ở [Bài 34](34-refactor-du-an.md).

</details>

**Bài 2.** Lỗ hổng #5 (sửa giá) và lỗ hổng #1 (sửa role) thoạt nhìn khác nhau, nhưng
chung một "gốc bệnh". Gốc bệnh đó là gì?

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Cả hai đều là **mass assignment** — backend lấy nguyên `req.body` do client gửi và nhét
thẳng vào database (`const update = req.body` ở `controllers/user.js:227`; và
`new Order(req.body)` ở `controllers/order.js`). Client gửi trường gì, backend lưu trường
đó, kể cả những trường lẽ ra nó không được phép đụng tới (`role`, `totalPrice`,
`productPrice`).

Cách chữa gốc: **luôn dùng danh sách trắng** — chỉ đọc đúng những trường được phép từ
`req.body`, mọi trường khác bỏ qua; và với các giá trị nhạy cảm (giá, quyền) thì backend
**tự tính lại** từ nguồn nó tin.

</details>

**Bài 3.** (suy ngẫm) `PATCH /api/products/userUpdate/:id` cố tình để công khai cho khách
tăng lượt xem. Nhưng nó nhận cả `view` lẫn `favorites` từ client. Hãy chỉ ra hai cách
lạm dụng, và đề xuất thiết kế an toàn mà vẫn cho khách tăng lượt xem được.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

**Lạm dụng:** (1) gửi `view: 999999` để bơm khống lượt xem; (2) gửi `favorites` tuỳ ý để
làm sai lệch bảng xếp hạng "sản phẩm được yêu thích".

**Thiết kế an toàn:** endpoint **không** nhận số từ client. Nó chỉ nhận `productId`, còn
backend tự tăng bằng toán tử `$inc`:

```js
// hướng sửa
await Product.findByIdAndUpdate(id, { $inc: { view: 1 } });
```

Như vậy mỗi lần gọi chỉ cộng đúng 1, client không quyết định được con số. Muốn chặt hơn
nữa thì chống spam bằng cách chỉ tăng một lần mỗi phiên/mỗi IP trong khoảng thời gian.

</details>

---

## 📌 Tóm tắt

- Một câu bao trùm tất cả: **không bao giờ tin dữ liệu từ client** — kiểm tra ở frontend chỉ để trải nghiệm, backend mới là nơi bảo vệ thật.
- Lỗ hổng nghiêm trọng nhất của Yotea: API `updateInfo` cho phép **tự gửi `role: 1`** để thành admin (`routes/users.js:12` + `controllers/user.js:225`).
- `GET /api/users` công khai và **trả về cả mật khẩu**; `order`/`orderDetail` gần như không được bảo vệ.
- Băm mật khẩu bằng **SHA256 không salt** và **bí mật `"TuongVy"` hardcode** dùng chung cho cả hash lẫn JWT — nên dùng **bcrypt** + biến môi trường.
- **Giá tiền** và **quyền** tuyệt đối không được để client quyết định; backend phải tự tính lại từ database và token.
- "Gốc bệnh" chung của nhiều lỗi là **mass assignment** (`const update = req.body`) — chữa bằng **danh sách trắng** các trường được phép.
- Kỹ năng rèn được: đọc route + controller và tự hỏi "ai gọi được, tin cái gì, sửa được cái gì" — dùng cho mọi dự án về sau.

**Từ khoá tra cứu thêm:** `OWASP Top 10`, `broken access control`, `mass assignment`, `bcrypt vs sha256`, `JWT httpOnly cookie`, `IDOR insecure direct object reference`, `ReDoS`

➡️ **Bài tiếp theo:** [34 — Refactor: `.env`, `configureStore`, xử lý lỗi tập trung](34-refactor-du-an.md) — xắn tay vá lần lượt những lỗ hổng vừa tìm ra và nâng dự án lên chuẩn hiện đại.
