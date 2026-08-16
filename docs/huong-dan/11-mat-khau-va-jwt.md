# Bài 11 — Mã hoá mật khẩu và xác thực bằng JWT

> **Phần 1 · Backend** — Thời lượng ước tính: **~75 phút**
> ⬅️ Bài trước: [10 — Quan hệ dữ liệu & `populate`](10-quan-he-va-populate.md) · Bài sau: [12 — Phân quyền: `requireSignin`, `isAuth`, `isAdmin`](12-phan-quyen-middleware.md) ➡️
> 🏠 [Mục lục](README.md)

---

## 🎯 Sau bài này bạn sẽ

- Giải thích được **vì sao tuyệt đối không lưu mật khẩu dạng thô**, và phân biệt **mã hoá (encrypt)** với **băm (hash)**.
- Đọc hiểu từng dòng hai method `authenticate` / `encryptPassword` và hai hook `pre("save")` / `pre("findOneAndUpdate")` trong `models/user.js`.
- Biết **HMAC-SHA256** là gì và chuỗi `"TuongVy"` đóng vai trò gì trong dự án.
- Chỉ ra **3 lỗ hổng** trong cách băm mật khẩu của Yotea và biết cách làm đúng bằng **bcrypt**.
- Hiểu cấu trúc **3 phần của JWT**, tự giải mã payload, biết vì sao **không được** nhét dữ liệu nhạy cảm vào token.
- Mổ xẻ trọn vẹn `signin`, `signup`, `checkPassword` — kể cả **một bug làm sập server** trong `signup`.
- Tự viết endpoint `POST /api/changePassword` và test bằng Postman.

## 📋 Cần chuẩn bị

- Đã hoàn thành [Bài 10 — Quan hệ dữ liệu & `populate`](10-quan-he-va-populate.md).
- MongoDB đang chạy, backend chạy được bằng `npm start` ([Bài 02](02-cai-dat-moi-truong.md)); có Postman.
- Nhớ lại **destructuring lồng nhau** ở [Bài 03](03-kien-thuc-nen.md) — bài này dùng lại đúng kỹ thuật đó.

> 🧵 **Mạch Topping:** ở [Bài 10](10-quan-he-va-populate.md) bạn đã nối `Topping` với `OrderDetail` bằng
> `ObjectId` + `ref` + `populate`, và thấy tận mắt việc Mongoose **âm thầm vứt bỏ** `toppingId`/`sizeId`
> mà `CheckoutPage.js` gửi lên vì schema `orderDetail.js` không khai báo chúng. Bài này ta **tạm dừng
> Topping một nhịp** để học mật khẩu và token — vì sang [Bài 12](12-phan-quyen-middleware.md) bạn sẽ cần
> chính token này để **khoá các route ghi của Topping**.

---

## 1. Vì sao không được lưu mật khẩu dạng thô?

Nếu `users` lưu `"password": "matkhau123"` thì những người sau **đọc được** mật khẩu của khách: bạn (mở
Compass là thấy), mọi thành viên có quyền vào DB, ai lấy được bản backup, và hacker khai thác được một
lỗ hổng bất kỳ. Tệ nhất: người dùng hay xài **chung một mật khẩu cho nhiều nơi** — lộ ở web trà sữa là
lộ luôn Gmail, Facebook, ngân hàng.

> 🔒 **Ghi chú bảo mật:** Server **không cần biết** mật khẩu. Nó chỉ cần trả lời *"chuỗi vừa gõ có đúng
> là mật khẩu không?"* — và có cách trả lời mà **không lưu** mật khẩu.

### 1.1. Mã hoá ≠ Băm

| | **Mã hoá (encryption)** | **Băm (hashing)** |
|---|---|---|
| Chiều | **Hai chiều** — có khoá là giải ngược ra bản gốc | **Một chiều** — không có đường về |
| Ví von | Két sắt: khoá lại rồi mở ra được | Cối xay thịt: bò → thịt xay, không xay ngược |
| Đầu ra | Dài ngắn theo đầu vào | **Luôn cố định** (SHA256 → 64 ký tự hex) |
| Dùng cho | Dữ liệu cần đọc lại: số thẻ, tin nhắn | **Mật khẩu**, kiểm tra toàn vẹn file |
| Hàm tiêu biểu | AES, RSA | SHA256, **bcrypt**, argon2 |

Mật khẩu **phải băm, không mã hoá**: nếu mã hoá, ai lấy được cả DB **và** khoá là giải ngược ra toàn bộ
mật khẩu; còn băm thì chỉ có một mớ chuỗi hex vô nghĩa.

### 1.2. Luồng hoạt động

```
ĐĂNG KÝ (lưu)                          ĐĂNG NHẬP (kiểm tra)
─────────────────────────              ────────────────────────────────────
"matkhau123"                           người dùng gõ "matkhau123"
     │ băm                                    │ băm  (CÙNG một thuật toán)
     ▼                                        ▼
"a3f9c1...e2"  ──► lưu vào MongoDB      "a3f9c1...e2"
                                              │
                        chuỗi băm trong DB ───┤ SO SÁNH hai chuỗi băm
                                              ▼
                                     bằng nhau → đúng mật khẩu
```

Server **không bao giờ giải ngược** chuỗi băm — nó băm cái vừa gõ rồi so hai chuỗi băm. Nhớ kỹ sơ đồ
này, code dự án là bản dịch từng chữ của nó.

---

## 2. Soi code thật — `models/user.js`

Công cụ băm là module `crypto` **có sẵn của Node.js**, nhập ở `models/user.js:1`
(`import { createHmac } from "crypto";`).

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** trường `password` khai báo `min: 4` (`models/user.js:14`). Với
> kiểu `String`, Mongoose dùng **`minlength`** (còn `min` chỉ cho `Number`/`Date`) → dòng đó **vô tác
> dụng**, gửi mật khẩu `"a"` vẫn lưu được. Hiện việc kiểm tra độ dài do **frontend** làm (yup,
> `UpdatePasswordPage.js:39`) — mà kiểm tra ở frontend thì ai cũng bỏ qua được bằng Postman.

### 2.1. Hai method: `authenticate` và `encryptPassword`

`yotea-be/src/models/user.js:66-80`

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
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 66 | `userSchema.methods = {` | Gắn thêm hàm cho **mỗi document** `User` → sau này viết `user.authenticate(...)` |
| 67 | `authenticate(password) {` | Nhận mật khẩu **thô** người dùng vừa gõ |
| 68 | `this.password === this.encryptPassword(password)` | `this.password` là **chuỗi băm trong DB**; vế phải là **chuỗi băm của cái vừa gõ**. So sánh hai chuỗi băm → `true`/`false` |
| 71 | `if (!password) return;` | Không truyền gì thì trả `undefined`, tránh nổ |
| 73 | `createHmac("SHA256", "TuongVy")` | Bộ băm HMAC, thuật toán **SHA256**, khoá bí mật là chuỗi `"TuongVy"` |
| 74 | `.update(password)` | Nạp mật khẩu vào bộ băm |
| 75 | `.digest("hex")` | "Xay" ra kết quả cuối dạng hex → chuỗi 64 ký tự |
| 76-78 | `catch` → `console.log` | Nuốt lỗi; nếu lỗi hàm trả `undefined` |

> 📖 **Thuật ngữ:** `methods` là hàm gắn vào **một document** (`user.authenticate(...)`), khác `statics`
> là hàm gắn vào **cả model** (`User.timGiDo(...)`). Trong `methods`, `this` là document đó.

### 2.2. HMAC là gì, `"TuongVy"` là gì?

**Hash thường:** `SHA256("matkhau123")` — ai cũng tính được, chỉ cần Google "sha256 online".
**HMAC** (Hash-based Message Authentication Code) = hash **cộng thêm một khoá bí mật**:

```
SHA256("admin")                      → ai cũng tính được
HMAC-SHA256(khoá="TuongVy", "admin") → chỉ ai BIẾT KHOÁ mới tính được
```

Vậy `"TuongVy"` chính là **khoá bí mật** của cả hệ thống. Tự kiểm chứng bằng file nháp `thu.js` **ngoài**
thư mục dự án (*code này bạn tự viết để thử, dự án không có*):

```js
const { createHmac } = require("crypto");
const bam = (p) => createHmac("SHA256", "TuongVy").update(p).digest("hex");

console.log(bam("admin"));   // fc75cb391799939c094a290c29dadaf06c0e040b189ad5fef7ec4e80b43246e9
console.log(bam("admin"));   // chạy lại vẫn ra Y HỆT
console.log(bam("Admin"));   // đổi 1 chữ hoa → khác hoàn toàn
```

Hai đặc điểm cần rút ra: **cùng đầu vào luôn ra cùng đầu ra** (không thì không so sánh được), và **đổi
một ký tự thì kết quả khác sạch** (hiệu ứng thác đổ).

### 2.3. Hai hook: băm lúc nào?

Điều thú vị là **controller không hề gọi `encryptPassword`**. Việc băm giao cho hai **hook** (móc) chạy
tự động ngay trước khi ghi xuống DB:

`yotea-be/src/models/user.js:82-94`

```js
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

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 82 | `userSchema.pre("save", function (next) {` | *"Trước khi document User được `.save()`, chạy hàm này."* Dùng `function` **chứ không phải arrow function** — vì cần `this` trỏ vào document |
| 83 | `this.password = this.encryptPassword(this.password)` | Ghi đè mật khẩu thô bằng chuỗi băm |
| 84 | `next()` | Báo "xong rồi, cho lưu tiếp" |
| 87 | `pre("findOneAndUpdate", ...)` | Hook cho **query** `findOneAndUpdate` (dùng khi đổi mật khẩu) |
| 88 | `if (this._update.password)` | Chỉ băm khi request **có gửi** `password`; sửa tên/địa chỉ thì bỏ qua |
| 89-91 | `createHmac(...)` | Ở đây `this` là **query** chứ không phải document nên **không gọi được** `this.encryptPassword` → phải chép lại nguyên đoạn băm |

> 💡 **Vì sao cần hai hook?** Mongoose có hai đường ghi: `new User(...).save()` (qua document → hook
> `save`) và `User.findOneAndUpdate(...)` (thẳng xuống DB → hook `findOneAndUpdate`). Hook `save`
> **không** chạy khi dùng `findOneAndUpdate`. Thiếu một trong hai là mật khẩu bị lưu thô ở một luồng.

Đây là lý do hàm `update` của user (`yotea-be/src/controllers/user.js:225-240`) chỉ đẩy nguyên `req.body`
vào `User.findOneAndUpdate(filter, update, options)` mà **không biết gì về mật khẩu** — hook lo hết.

> ⚠️ **Bẫy trong hook `pre("save")`:** nó **không kiểm tra** mật khẩu có thay đổi hay không. Gọi `.save()`
> lần thứ hai trên cùng document (ví dụ chỉ để đổi `avatar`) là chuỗi **đã băm bị băm thêm lần nữa** →
> người dùng không đăng nhập được nữa. Cách viết đúng: thêm dòng cứu mạng
> `if (!this.isModified("password")) return next();` ngay đầu hook. Dự án **may mắn chưa nổ** vì không
> chỗ nào gọi `.save()` lần hai trên document User — nhưng mục 6 bạn sắp viết endpoint đổi mật khẩu,
> nhớ cái bẫy này.

---

## 3. ⚠️ Phân tích bảo mật: 3 vấn đề

Cách làm trên **chạy đúng**, nhưng theo chuẩn ngành thì có ba lỗi nặng.

### 3.1. Khoá bí mật hardcode, và dùng chung cho hai việc

Chuỗi `"TuongVy"` xuất hiện ở bốn nơi: `models/user.js:73` và `:89` (khoá **băm mật khẩu**),
`controllers/auth.js:52` (khoá **ký JWT**), `middlewares/checkAuth.js:5` (khoá **xác minh JWT**).

1. **Hardcode trong source** → nằm luôn trong Git, ai clone repo cũng thấy. Mà biết khoá là **tự băm
   được mật khẩu bất kỳ để đối chiếu với DB**. Khoá phải nằm trong `.env` — làm ở [Bài 34](34-refactor-du-an.md).
2. **Một khoá cho hai mục đích** vi phạm nguyên tắc *key separation*: lộ khoá là **mất cả hai lớp bảo
   vệ cùng lúc** — vừa dò được mật khẩu, vừa **tự ký được token giả** để đóng vai bất kỳ ai.

### 3.2. Không có salt riêng cho từng người

Băm chữ `"admin"` **lúc nào cũng ra đúng một chuỗi**, nên trong collection `users`:

```json
{ "email": "an@gmail.com",   "password": "fc75cb391799939c094a290c29dadaf06c0e040b189ad5fef7ec4e80b43246e9" }
{ "email": "binh@gmail.com", "password": "fc75cb391799939c094a290c29dadaf06c0e040b189ad5fef7ec4e80b43246e9" }
```

Hai chuỗi băm **giống hệt nhau** → kẻ tấn công nhìn phát biết ngay **An và Bình dùng chung một mật
khẩu**. Tệ hơn, vì mọi user chung một khoá, chỉ cần băm sẵn **một** danh sách mật khẩu phổ biến là có
ngay **bảng tra sẵn (rainbow table)** dùng cho **toàn bộ** hệ thống.

**Cách đúng: salt** — chuỗi **ngẫu nhiên, khác nhau từng user**, trộn vào trước khi băm và lưu kèm:

```
An:   băm("matkhau123" + "x7Kq2p") → "9a1c..."   ← salt riêng
Bình: băm("matkhau123" + "Lm0Zv4") → "e42f..."   ← salt riêng
```

Cùng mật khẩu nhưng **hai chuỗi băm khác nhau** → bảng tra sẵn vô dụng.

### 3.3. SHA256 quá... nhanh

Với mật khẩu thì **nhanh là dở**. SHA256 sinh ra để băm file vài GB trong tích tắc; một GPU chơi game
băm được **hàng tỉ chuỗi mỗi giây**. **bcrypt**/**argon2** giải quyết bằng cách **cố tình chậm**: tham
số `saltRounds` tăng 1 là chậm gấp đôi.

| | HMAC-SHA256 (dự án đang dùng) | bcrypt (`saltRounds = 10`) |
|---|---|---|
| Băm 1 mật khẩu | ~0,000002 giây | ~0,1 giây |
| Người dùng đăng nhập | Không cảm nhận được | Không cảm nhận được |
| **Kẻ tấn công thử 1 tỉ mật khẩu** | **vài phút** | **hơn 3 năm** |
| Salt riêng từng user | ❌ Không | ✅ Tự sinh, tự lưu trong chuỗi băm |

### 3.4. Viết lại bằng bcrypt

> Đoạn dưới **bạn tự viết thêm để so sánh — dự án Yotea KHÔNG có code này**, và `bcryptjs` cũng **chưa**
> có trong `yotea-be/package.json` (muốn thử phải `npm i bcryptjs` tại thư mục `yotea-be`).

```js
// code tham khảo, bạn tự viết — dự án chưa có
import bcrypt from "bcryptjs";

userSchema.methods = {
  async authenticate(password) {          // async vì bcrypt chậm có chủ đích
    return bcrypt.compare(password, this.password);
  },
};

userSchema.pre("save", async function (next) {
  if (!this.isModified("password")) return next();       // tránh băm hai lần
  this.password = await bcrypt.hash(this.password, 10);  // 10 = saltRounds
  next();
});
```

1. **Không thấy salt đâu** — bcrypt **tự sinh salt ngẫu nhiên** và nhét luôn vào kết quả:
   `$2a$10$N9qo8uLOickgx2ZMRZoMy.MH/rniOgU8u...` (`$2a$` = phiên bản, `10` = saltRounds, 22 ký tự tiếp
   là salt, phần còn lại mới là hash).
2. **Không so bằng `===`** mà dùng `bcrypt.compare()` — vì mỗi lần băm ra một chuỗi khác nhau.
3. Mọi thứ thành **`async`** → chỗ gọi `user.authenticate(password)` phải thêm `await`.

> 🔒 **Chốt lại:** Yotea dùng **HMAC-SHA256, không salt riêng, khoá hardcode dùng chung với JWT**. Bài
> tập lớn sinh viên thì chấp nhận được, code chạy thật **bắt buộc** dùng bcrypt/argon2. Tổng kết ở
> [Bài 33](33-ra-soat-bao-mat.md).

---

## 4. JWT — tấm vé vào cửa

HTTP **không nhớ gì cả** (stateless), xử lý xong một request là quên sạch. Vậy làm sao request thứ hai
biết *"người này vừa đăng nhập rồi"*? Sau khi đăng nhập đúng, server phát cho client một **tấm vé có
chữ ký của server** — **JWT (JSON Web Token)**.

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2NjUwYTFmMmM0ZThiOTEyMzRhYmNkMDEiLCJlbWFpbCI6ImFkbWluQGdtYWlsLmNvbSIsImlhdCI6MTc1NTAwMDAwMCwiZXhwIjoxNzU1MDEwODAwfQ.hZ9dmDcxJSFD6-CBVD9nxV8yU5WOnIPq2UP96NtlSZE
└────────── header ──────────┘ └───────────────────────── payload ─────────────────────────┘ └──────────── signature ────────────┘
```

| Phần | Nội dung | Bảo vệ kiểu gì |
|---|---|---|
| **Header** | `{"alg":"HS256","typ":"JWT"}` | **base64** — ai cũng đọc được |
| **Payload** | `{"_id":"...","email":"...","iat":...,"exp":...}` | **base64** — ai cũng đọc được |
| **Signature** | Tính từ `header + payload + khoá bí mật` | HMAC-SHA256 — **không giả được** |

`iat` = *issued at*, `exp` = *expires* — thư viện tự thêm.

### 4.1. ⚠️ Payload KHÔNG được mã hoá!

Đây là hiểu nhầm phổ biến nhất về JWT. **base64 không phải mã hoá** — nó chỉ là cách viết lại dữ liệu
cho an toàn khi truyền qua URL, ai cũng giải ngược trong 1 giây. Thử ngay: mở trình duyệt, **F12** →
tab **Console**, dán vào:

```js
atob("eyJfaWQiOiI2NjUwYTFmMmM0ZThiOTEyMzRhYmNkMDEiLCJlbWFpbCI6ImFkbWluQGdtYWlsLmNvbSIsImlhdCI6MTc1NTAwMDAwMCwiZXhwIjoxNzU1MDEwODAwfQ")
// → {"_id":"6650a1f2c4e8b91234abcd01","email":"admin@gmail.com","iat":1755000000,"exp":1755010800}
```

**Không cần khoá, không cần mật khẩu.** Hoặc dán cả token vào **https://jwt.io** để xem giao diện đẹp
hơn — hãy làm thử với **token thật** của bạn ở mục 7.

> 🔒 **Ghi chú bảo mật:** vì payload ai cũng đọc được, **tuyệt đối không nhét vào token**: mật khẩu (dù
> đã băm), số CCCD, số thẻ, địa chỉ nhà, khoá API...

JWT bảo vệ **tính toàn vẹn**, không phải tính bí mật: kẻ tấn công **đọc được** payload (JWT chấp nhận
điều đó), nhưng nếu **sửa** payload (đổi `_id` thành id admin chẳng hạn) thì signature không còn khớp;
muốn ký lại phải có khoá `"TuongVy"`. Server sẽ trả `invalid signature`.

---

## 5. Soi code thật — `controllers/auth.js`

Ba endpoint auth khai báo ở `yotea-be/src/routes/auth.js:6-8`:

```js
router.post("/signin", signin);
router.post("/signup", signup);
router.post("/checkPassword", checkPassword);
```

Router mount ở `yotea-be/src/app.js:37` (`app.use("/api", authRouter)`) nên đường dẫn đầy đủ là
`POST http://localhost:8080/api/signin`.

### 5.1. `signin` — nơi phát ra tấm vé

`yotea-be/src/controllers/auth.js:32-66`

```js
export const signin = async (req, res) => {
  const { email, password } = req.body;

  try {
    const user = await User.findOne({ email }).exec();

    if (!user) {
      res.status(400).json({
        message: "Email không tồn tại",
      });
    } else if (!user.authenticate(password)) {
      res.status(400).json({
        message: "Mật khẩu không chính xác",
      });
    } else {
      const {
        _doc: { password: hashed_password, __v, ...rest },
      } = user;
      const token = jwt.sign(
        { _id: user._id, email: user.email },
        "TuongVy",
        { expiresIn: "3h" }
      );

      res.json({
        token,
        user: rest,
      });
    }
  } catch (error) {
    res.status(400).json({
      message: "Lỗi",
    });
  }
};
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 33 | `const { email, password } = req.body` | Bóc hai trường client gửi lên |
| 36 | `User.findOne({ email }).exec()` | Tìm user **theo email** (không phải username) |
| 38-41 | `if (!user)` → 400 | Không có email này trong DB |
| 42 | `!user.authenticate(password)` | Gọi method ở `models/user.js:67-69`: băm cái vừa gõ rồi so với chuỗi băm trong DB |
| 47-49 | `const { _doc: { password: hashed_password, __v, ...rest } } = user` | **Bóc `password` và `__v` ra khỏi dữ liệu**, phần còn lại vào `rest` |
| 50-54 | `jwt.sign(...)` | Ký token — xem bảng dưới |
| 56-59 | `res.json({ token, user: rest })` | Trả **tấm vé** + **hồ sơ đã sạch mật khẩu** |
| 61-65 | `catch` → 400 `"Lỗi"` | Nuốt lỗi, không log gì |

**Vì sao phải đi qua `_doc`?** `user` là Mongoose Document, ngoài dữ liệu thật còn mang cả đống thuộc
tính nội bộ (`$__`, `$isNew`, các hàm...); dữ liệu thật nằm ở `user._doc`. Còn `password: hashed_password`
và `__v` là **mẹo loại trường**: hai biến này khai báo ra rồi **không dùng ở đâu cả**, mục đích duy nhất
là để chúng **không lọt vào** `rest` (kỹ thuật đã gặp ở [Bài 03, mục 1.3](03-kien-thuc-nen.md)).

**Lệnh ký token** — `yotea-be/src/controllers/auth.js:50-54`

```js
      const token = jwt.sign(
        { _id: user._id, email: user.email },
        "TuongVy",
        { expiresIn: "3h" }
      );
```

| Tham số | Giá trị trong Yotea | Ý nghĩa |
|---|---|---|
| 1 — **payload** | `{ _id: user._id, email: user.email }` | Dữ liệu nhét vào vé, đúng 2 trường |
| 2 — **secret** | `"TuongVy"` | Khoá bí mật để **ký**; phải trùng khoá **xác minh** ở `checkAuth.js:5` |
| 3 — **options** | `{ expiresIn: "3h" }` | Vé hết hạn sau **3 giờ**; thư viện tự tính `exp` |

Phía xác minh nằm ở `yotea-be/src/middlewares/checkAuth.js:3-7`: `expressJWT({ algorithms: ["HS256"],
secret: "TuongVy", requestProperty: "auth" })` — đọc header `Authorization: Bearer <token>`, xác minh
chữ ký, rồi gắn payload đã giải mã vào `req.auth`. Chi tiết ở bài sau.

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** payload **không có `role`** → backend không thể biết người gọi có
> phải admin **chỉ từ token**, nên buộc phải bắt client nhét thêm `:userId` vào URL rồi truy vấn DB lại
> để lấy `role` — chính là những đường dẫn kỳ dị kiểu `PUT /api/products/:id/:userId`. Toàn bộ hệ quả là
> nội dung [Bài 12](12-phan-quyen-middleware.md).

### 5.2. `signup` — và một bug làm sập server

`yotea-be/src/controllers/auth.js:87-116`

```js
export const signup = async (req, res) => {
  try {
    const exitsEmail = await User.findOne({ email: req.body.email }).exec();

    if (exitsEmail) {
      res.status(400).json({
        message: "Email đã tồn tại trên hệ thống",
      });
    }

    const { _id, email, fullName, username, phone, role, active, avatar } =
      await new User(req.body).save();

    res.json({
      _id,
      email,
      fullName,
      username,
      phone,
      role,
      active,
      avatar,
    });
  } catch (error) {
    res.status(400).json({
      message: "Đăng ký tài khoản không thành công",
      error,
    });
  }
};
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 89 | `const exitsEmail = await User.findOne({ email: req.body.email })` | Kiểm tra email đã ai dùng chưa (tên biến sai chính tả, đúng là `existsEmail`) |
| 91-95 | `if (exitsEmail) { res.status(400).json(...) }` | Báo lỗi "Email đã tồn tại" — **nhưng thiếu `return`!** |
| 97-98 | `await new User(req.body).save()` | Tạo user mới. `.save()` kích hoạt hook `pre("save")` → **mật khẩu được băm ở đây** |
| 100-109 | `res.json({ ... })` | Trả đúng 8 trường, **cố ý không có `password`** và **không có `token`** |
| 110-115 | `catch` → 400 | Trả nguyên object `error` của Mongo về cho client |

> ⚠️ **BUG THẬT — dòng 91-95 thiếu `return`.** Trong Express, `res.status(400).json(...)` **chỉ gửi
> response đi, KHÔNG dừng hàm**. Khi email trùng:
>
> 1. Dòng 92 gửi `400 "Email đã tồn tại trên hệ thống"` — client nhận đúng thông báo.
> 2. Hàm **vẫn chạy tiếp** xuống dòng 97: `new User(req.body).save()`.
> 3. MongoDB từ chối vì `email` có `unique: true` (`models/user.js:9`) → ném **`E11000 duplicate key error`**.
> 4. Lỗi rơi vào `catch` dòng 110 → gọi `res.status(400).json(...)` **lần thứ hai trên response đã gửi**.
> 5. Node ném `ERR_HTTP_HEADERS_SENT` — *"Cannot set headers after they are sent to the client"*. Vì đây
>    là exception trong hàm `async` không được `next(err)`, nó thành **unhandled promise rejection**; với
>    Node phiên bản mới, mặc định là **kill tiến trình** → **server chết**, `nodemon` phải restart.
>
> **Cách sửa — thêm đúng một từ** ở dòng 92:
>
> ```js
> if (exitsEmail) {
>   return res.status(400).json({ message: "Email đã tồn tại trên hệ thống" });
> }
> ```
>
> Hoặc bọc phần còn lại vào `else { ... }` — đúng cách mà `signin` (dòng 38-60) đã làm.
> **Nguyên tắc vàng: sau mỗi `res.` ở nhánh lỗi, LUÔN có `return`.** Cùng bug này còn ở
> `yotea-be/src/controllers/user.js:39-43` (hàm `create`).

### 5.3. `checkPassword` — kiểm tra mật khẩu cũ

`yotea-be/src/controllers/auth.js:118-142`

```js
export const checkPassword = async (req, res) => {
  const { _id, password } = req.body;

  try {
    const user = await User.findById(_id).exec();
    if (!user) {
      res.status(400).json({
        message: "Không tìm thấy User",
      });
    } else if (!user.authenticate(password)) {
      res.status(400).json({
        message: "Mật khẩu không chính xác",
      });
    } else {
      res.json({
        message: "Mật khẩu chính xác",
        success: true,
      });
    }
  } catch (error) {
    res.status(400).json({
      message: "Có lỗi xảy ra",
    });
  }
};
```

Hàm này **không cập nhật gì cả** — chỉ trả lời *"mật khẩu này có đúng không?"*. Dùng để làm gì? Nhìn
sang trang **Đổi mật khẩu** — `yotea-fe/src/pages/user/my-account/UpdatePasswordPage.js:19-35`:

```js
      .test(
        "is_confirm",
        "Mật khẩu hiện tại không chính xác",
        async function (value) {
          try {
            const { data } = await checkPassword({
              _id: user._id,
              password: value,
            });
            if (data.success) return true;
          } catch (error) {
            console.log({ error });
          }

          return false;
        }
      ),
```

Đây là luật **validate bất đồng bộ** của yup: người dùng gõ "mật khẩu hiện tại" → frontend gọi
`POST /api/checkPassword` hỏi backend; sai thì hiện chữ đỏ và **không cho submit**. Qua được vòng đó,
`onSubmit` (`UpdatePasswordPage.js:61-69`) mới gọi `updateMyInfo({ _id: user._id, password: dataInput.newPassword })`
→ `PUT /api/users/updateInfo/:myId/:userId` → controller `update` → `findOneAndUpdate` → **hook
`pre("findOneAndUpdate")` băm mật khẩu mới**.

> ⚠️ **Chỗ này dự án làm chưa chuẩn — rất nặng:** `/api/checkPassword` (`routes/auth.js:8`) **không có
> `requireSignin`**. Kết hợp với `GET /api/users` cũng public (trả `_id` của mọi người), bất kỳ ai cũng
> **dò được mật khẩu của bất kỳ ai** bằng cách bắn liên tục `POST /api/checkPassword` với `_id` nạn nhân
> — không token, không giới hạn số lần thử. Đúng ra việc kiểm tra mật khẩu cũ **phải nằm bên trong**
> chính API đổi mật khẩu — đó là thứ bạn sắp tự tay làm.

---

## 6. 🛠️ Tự tay làm — endpoint `POST /api/changePassword`

> **Mục tiêu:** cuối phần này bạn có endpoint **gộp một bước**: nhận `_id`, mật khẩu cũ, mật khẩu mới →
> tự kiểm tra mật khẩu cũ bằng `authenticate` → đúng thì đổi.
> 📌 Toàn bộ code dưới đây là **code bạn tự viết thêm** — dự án Yotea **chưa có** endpoint này.

### Bước 1 — Viết controller

Mở `yotea-be/src/controllers/auth.js`, **thêm vào cuối file** (giữ nguyên mọi thứ đang có):

```js
// ===== code MỚI, bạn tự viết thêm vào cuối yotea-be/src/controllers/auth.js =====
export const changePassword = async (req, res) => {
  const { _id, oldPassword, newPassword } = req.body;

  if (!_id || !oldPassword || !newPassword) {
    return res.status(400).json({ message: "Thiếu _id, oldPassword hoặc newPassword" });
  }
  if (newPassword.length < 4) {
    return res.status(400).json({ message: "Mật khẩu mới phải có tối thiểu 4 ký tự" });
  }

  try {
    const user = await User.findById(_id).exec();
    if (!user) {
      return res.status(400).json({ message: "Không tìm thấy User" });
    }

    // Xác minh mật khẩu CŨ bằng method có sẵn của model
    if (!user.authenticate(oldPassword)) {
      return res.status(400).json({ message: "Mật khẩu cũ không chính xác" });
    }

    // Cập nhật: dùng findOneAndUpdate để kích hoạt hook pre("findOneAndUpdate")
    // (models/user.js:87-94) — hook sẽ tự băm giúp ta.
    await User.findOneAndUpdate({ _id }, { password: newPassword }, { new: true }).exec();

    return res.json({ message: "Đổi mật khẩu thành công", success: true });
  } catch (error) {
    return res.status(400).json({ message: "Đổi mật khẩu không thành công", error });
  }
};
```

1. **Mọi `res.` đều có `return`** — đúng bài học từ bug ở mục 5.2.
2. Ta **không tự băm** `newPassword`: việc đó là của hook. Tự băm rồi gửi vào `findOneAndUpdate` thì hook
   băm **lần thứ hai** → mật khẩu hỏng, không đăng nhập lại được.
3. Dùng `findOneAndUpdate` chứ không phải `user.password = newPassword; await user.save()` — cách sau đi
   qua hook `pre("save")` vốn **không có** `isModified` (bẫy ở mục 2.3); ở đây vẫn đúng, nhưng
   `findOneAndUpdate` an toàn hơn về lâu dài.

### Bước 2 — Khai báo route

Mở `yotea-be/src/routes/auth.js`, sửa 2 chỗ:

```js
// yotea-be/src/routes/auth.js — bạn tự sửa
import { Router } from "express";
import { changePassword, checkPassword, signin, signup } from "../controllers/auth"; // ← thêm changePassword

const router = Router();

router.post("/signin", signin);
router.post("/signup", signup);
router.post("/checkPassword", checkPassword);
router.post("/changePassword", changePassword);   // ← THÊM

export default router;
```

Không cần đụng `app.js`: `authRouter` đã mount sẵn ở `yotea-be/src/app.js:37` với prefix `/api`, nên
route mới **tự động** có đường dẫn `POST /api/changePassword`.

> 💡 **Mẹo:** dùng `POST` chứ không phải `GET`, vì mật khẩu phải nằm trong **body**. Đưa lên query string
> (`?password=123`) là mật khẩu bị ghi vào log của `morgan` (`app.js:27`), vào lịch sử trình duyệt và
> vào header `Referer`.

---

## 7. ✅ Kiểm chứng kết quả

```bash
# đứng tại thư mục yotea-be
npm start
```

Terminal phải hiện `App is running on port: 8080` và `Connected to MongoDB`.

**Bước 1 — tạo tài khoản thử.** `POST http://localhost:8080/api/signup`, Body → raw → JSON:

```json
{
  "email": "hocvien@gmail.com",
  "password": "1234",
  "username": "hocvien",
  "fullName": "Học Viên Yotea",
  "phone": "0912345678"
}
```

Response `200` trả 8 trường (`_id`, `email`, `fullName`, `username`, `phone`, `role`, `active`, `avatar`)
— **không có `password`**. Mở Compass → database `yotea` → collection `users`: trường `password` phải là
chuỗi 64 ký tự hex, cụ thể với mật khẩu `1234` bạn sẽ thấy đúng chuỗi:

```
c17122b239c00c2ce60760402c239a5c0c460da94b030fe4521a53cc5edce504
```

Chuỗi này **giống hệt trên máy bạn và máy tôi** — vì khoá `"TuongVy"` cố định và **không có salt riêng**.
Vấn đề 3.2 vừa học, giờ nhìn thấy tận mắt.

**Bước 2 — đăng nhập lấy token.** `POST http://localhost:8080/api/signin` với
`{ "email": "hocvien@gmail.com", "password": "1234" }` → nhận `{ "token": "eyJhbGciOi...", "user": { ... } }`
(trong `user` **không có** `password` và `__v`). Copy chuỗi `token` → mở https://jwt.io → dán vào ô
*Encoded*: payload hiện ra rõ ràng **mà không cần nhập khoá bí mật** — bằng chứng sống cho mục 4.1.

**Bước 3 — gọi endpoint mới.** `POST http://localhost:8080/api/changePassword`:

```json
{ "_id": "66c1a3f7c4e8b91234abcd77", "oldPassword": "1234", "newPassword": "yotea2026" }
```

→ `{ "message": "Đổi mật khẩu thành công", "success": true }`. Gửi lại **đúng request đó lần nữa** →
phải nhận `400 { "message": "Mật khẩu cũ không chính xác" }` (vì mật khẩu cũ giờ là `yotea2026`).

**Bước 4 — chứng minh mật khẩu mới đã băm đúng.** Gọi `POST /api/signin` với `yotea2026` → nhận token
mới; với mật khẩu cũ `1234` → `400 "Mật khẩu không chính xác"`. Vào Compass xem lại: `password` là chuỗi
64 ký tự hex **khác** chuỗi cũ → hook `pre("findOneAndUpdate")` đã làm việc.

---

## 8. 🐞 Lỗi thường gặp

| Thông báo lỗi | Nguyên nhân | Cách sửa |
|---|---|---|
| `Cannot set headers after they are sent to the client` | Gọi `res.` hai lần trong một request — thiếu `return` ở nhánh lỗi | Thêm `return` trước mọi `res.status(...)` ở nhánh lỗi (mục 5.2) |
| `E11000 duplicate key error collection: yotea.users` | Đăng ký trùng `email` hoặc `username` (cả hai đều `unique`) | Đổi email/username khác |
| `UnauthorizedError: No authorization token was found` | Quên header `Authorization: Bearer <token>` | Postman → tab **Authorization** → **Bearer Token** → dán token |
| `invalid signature` | Ký bằng khoá này, verify bằng khoá khác | Khoá ở `auth.js:52` phải **trùng** khoá ở `checkAuth.js:5` |
| `jwt expired` | Token quá **3 giờ** (`expiresIn: "3h"`) | Đăng nhập lại lấy token mới |
| `jwt malformed` | Copy thiếu token, hoặc gửi chữ `Bearer` hai lần | Token phải có đúng **2 dấu chấm** |
| Báo "Mật khẩu không chính xác" dù gõ đúng | Mật khẩu bị **băm hai lần** (gọi `.save()` lần nữa trên document đã băm) | Thêm `if (!this.isModified("password")) return next();` vào hook `pre("save")` |
| Sửa hồ sơ xong không đăng nhập được | Frontend gửi kèm `password` (đã băm) trong body `PUT /users/...` → hook băm lại | Không gửi `password` khi chỉ sửa thông tin |
| `Cannot read properties of undefined (reading '_id')` ở `isAuth` | Route có `isAuth` nhưng URL **không có** `:userId` → `req.profile` là `undefined` | Xem [Bài 12](12-phan-quyen-middleware.md) |

---

## 9. 📝 Bài tập

**Bài 1.** Lấy token thật của bạn ở mục 7 rồi trả lời: (a) phần nào trong 3 phần chứa `email` của bạn?
(b) Nếu tự sửa một ký tự trong payload rồi gửi lên server thì chuyện gì xảy ra? (c) Vì sao **không nên**
để frontend tin vào `role` đọc từ payload, dù nghe rất hợp lý?

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

(a) **Phần thứ hai — payload.** Giải bằng `atob("<phần giữa>")` trong Console sẽ thấy
`{"_id":"...","email":"...","iat":...,"exp":...}`.

(b) Server trả **`invalid signature`**. Signature tính từ `header + payload + secret`; đổi payload thì
signature cũ không khớp, mà muốn ký lại phải biết `"TuongVy"`. Đây đúng là thứ JWT bảo vệ: **không bảo
mật nội dung, nhưng bảo đảm nội dung không bị sửa**.

(c) Thêm `role` vào payload **không sai** — thậm chí phổ biến, và sẽ giúp Yotea bỏ được cái `:userId` xấu
xí trên URL. Cái sai là **frontend tin vào nó để quyết định quyền**: payload ai cũng đọc, ai cũng tự bịa
một chuỗi base64 được. Dùng `role` trong token để **ẩn/hiện menu** cho đẹp thì được; nhưng **cho phép xoá
sản phẩm** thì backend **vẫn phải tự xác minh chữ ký** rồi mới tin.

</details>

**Bài 2.** Tái hiện bug ở mục 5.2: gọi `POST /api/signup` **hai lần liên tiếp** cùng một email. Mô tả
chính xác những gì bạn thấy ở (a) Postman và (b) terminal chạy backend. Sau đó sửa bug bằng một từ.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

(a) **Postman:** lần 1 trả `200` kèm thông tin user; lần 2 trả `400 { "message": "Email đã tồn tại trên
hệ thống" }` — nhìn thì tưởng đúng.

(b) **Terminal** mới là chỗ lộ bug: `Error [ERR_HTTP_HEADERS_SENT]: Cannot set headers after they are
sent to the client`, và tuỳ phiên bản Node, tiến trình có thể **chết hẳn** rồi `nodemon` restart
(`[nodemon] app crashed - waiting for file changes before starting...`).

Diễn biến: `res.status(400).json(...)` dòng 92 **không dừng hàm** → dòng 97-98 vẫn chạy
`new User(req.body).save()` → Mongo ném `E11000` (vì `email` `unique`, `models/user.js:9`) → nhảy vào
`catch` dòng 110 → gọi `res.` lần hai trên response đã gửi → nổ.

**Cách sửa** (`yotea-be/src/controllers/auth.js:92`): thêm `return` trước `res.status(400)`. Sau khi sửa,
gọi lần 2 vẫn trả `400` giống hệt nhưng **terminal sạch bong**, server không chết. Bài học rộng hơn:
`res.json()` chỉ *"gửi thư đi"*, không phải *"kết thúc hàm"* — cùng bug này còn ở
`yotea-be/src/controllers/user.js:39-43`.

</details>

**Bài 3.** Chuyển `models/user.js` sang **bcryptjs** (dùng code tham khảo ở mục 3.4). Liệt kê những chỗ
**khác** buộc phải sửa theo, và giải thích vì sao user cũ trong DB sẽ không đăng nhập được nữa.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

| File | Sửa gì |
|---|---|
| `yotea-be/src/controllers/auth.js:42` | `!user.authenticate(password)` → `!(await user.authenticate(password))` |
| `yotea-be/src/controllers/auth.js:127` | Y hệt, trong `checkPassword` |
| Endpoint `changePassword` bạn viết ở mục 6 | Y hệt, thêm `await` |
| Hook `pre("findOneAndUpdate")` (`models/user.js:87-94`) | Thêm `async` và đổi sang `await bcrypt.hash(this._update.password, 10)` |

**Bẫy lớn nhất:** quên `await` thì `user.authenticate(...)` trả về một **Promise** — mà Promise luôn
**truthy** → `!promise` luôn `false` → **mọi mật khẩu đều được chấp nhận**. Lỗi bảo mật kinh điển: code
không báo lỗi gì, chỉ là hệ thống mất sạch tác dụng bảo vệ.

**Vì sao user cũ không đăng nhập được?** Mật khẩu của họ đang là chuỗi HMAC-SHA256 64 ký tự hex, trong
khi `bcrypt.compare()` chờ định dạng `$2a$10$...` → luôn trả `false`. Băm là một chiều nên **không có
cách nào "chuyển đổi"**. Ngoài đời người ta hoặc bắt đặt lại mật khẩu, hoặc thêm trường `hashVersion` để
kiểm tra bằng thuật toán cũ rồi **băm lại bằng bcrypt ngay lần đăng nhập kế tiếp**.

</details>

**Bài 4.** *(nối mạch Topping)* Nhìn lại `yotea-be/src/routes/topping.js` bạn đã viết ở Bài 06-07: trong
5 route CRUD của Topping, route nào **bắt buộc** phải có token, route nào để public? Vì sao?

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

| Route | Cần token? | Lý do |
|---|---|---|
| `GET /api/toppings` | ❌ Public | Khách chưa đăng nhập vẫn phải xem được để chọn khi mua |
| `GET /api/toppings/:slug` | ❌ Public | Như trên |
| `POST /api/toppings/:userId` | ✅ **Admin** | Chỉ quản trị viên mới được thêm topping |
| `PUT /api/toppings/:id/:userId` | ✅ **Admin** | Sửa giá topping = sửa tiền → cực nhạy cảm |
| `DELETE /api/toppings/:id/:userId` | ✅ **Admin** | Xoá dữ liệu |

Nguyên tắc chung: **route đọc thì mở, route ghi thì khoá** — `routes/product.js` theo đúng khuôn mẫu này.
Đáng buồn là `routes/orderDetail.js` **không khoá gì cả**, kể cả `PUT` và `DELETE`, nên ai cũng sửa/xoá
được chi tiết đơn hàng của người khác. Cách gắn `requireSignin`, `isAuth`, `isAdmin` vào từng route
Topping chính là việc ở [Bài 12](12-phan-quyen-middleware.md).

</details>

---

## 📌 Tóm tắt

- **Băm (một chiều)** chứ không phải **mã hoá (hai chiều)** mới là thứ dùng cho mật khẩu. Server không
  cần biết mật khẩu, nó chỉ **so sánh hai chuỗi băm**.
- Yotea băm bằng `createHmac("SHA256", "TuongVy")` (`models/user.js:73`), so sánh trong `authenticate`
  (`models/user.js:67-69`), băm **tự động** qua hai hook `pre("save")` + `pre("findOneAndUpdate")`
  (`models/user.js:82-94`) — controller không phải làm gì cả.
- **Ba lỗi bảo mật:** khoá `"TuongVy"` **hardcode và dùng chung với JWT**; **không có salt riêng** nên
  hai người cùng mật khẩu ra cùng chuỗi băm; **SHA256 quá nhanh** nên dễ brute-force. Chuẩn công nghiệp
  là **bcrypt/argon2** — cố tình chậm và tự sinh salt.
- **JWT** gồm 3 phần `header.payload.signature`. Payload chỉ **base64**, **ai cũng đọc được** (thử
  `atob(...)` hoặc jwt.io) → **không bao giờ** nhét dữ liệu nhạy cảm. JWT bảo vệ *tính toàn vẹn*, không
  bảo vệ *tính bí mật*.
- `signin` (`controllers/auth.js:32-66`): tìm theo email → `authenticate` → **bóc `_doc` để loại
  `password` và `__v`** → `jwt.sign({ _id, email }, "TuongVy", { expiresIn: "3h" })` → trả `{ token, user }`.
- **Bug thật trong `signup`** (`controllers/auth.js:91-95`): thiếu `return` sau `res.status(400)` → code
  chạy tiếp, Mongo ném `E11000`, `catch` gọi `res.` lần hai → `Cannot set headers after they are sent` →
  **server sập**. Sửa bằng đúng một từ: `return`.
- `checkPassword` (`controllers/auth.js:118-142`) chỉ **xác minh mật khẩu cũ**, không cập nhật gì —
  frontend gọi nó trong luật yup của trang đổi mật khẩu; và nó **thiếu `requireSignin`**.

**Từ khoá tra cứu thêm:** `password hashing vs encryption`, `HMAC SHA256`, `bcrypt salt rounds`, `argon2`,
`rainbow table attack`, `JWT structure`, `jwt.io`, `base64 is not encryption`, `mongoose pre save hook`,
`ERR_HTTP_HEADERS_SENT`

➡️ **Bài tiếp theo:** [12 — Phân quyền: `requireSignin`, `isAuth`, `isAdmin`](12-phan-quyen-middleware.md)
— bạn đã có tấm vé, giờ tới lúc dựng ba lớp chốt chặn ở cửa: vé thật hay giả, đúng chủ vé không, và có
phải vé hạng VIP không. Xong bài đó, các route ghi của **Topping** sẽ được khoá lại.
