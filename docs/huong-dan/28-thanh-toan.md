# Bài 28 — Thanh toán: Order + OrderDetail, react-hook-form + yup

> **Phần 5 · Từng chức năng, từ đầu đến cuối** — Thời lượng ước tính: **~110 phút**
> ⬅️ Bài trước: [27 — Giỏ hàng](27-gio-hang.md) · Bài sau: [29 — Tài khoản của tôi: sửa thông tin, đổi mật khẩu, lịch sử đơn](29-tai-khoan-cua-toi.md) ➡️
> 🏠 [Mục lục](README.md)

---

## 🎯 Sau bài này bạn sẽ

- Đi trọn luồng **đặt hàng** từ cái nút "Đặt hàng" xuống tận collection `orders` và `orderdetails` trong MongoDB, rồi quay ngược trở lại trang `/thank-you`.
- Dùng thành thạo **react-hook-form**: `useForm`, `register`, `handleSubmit`, `formState.errors`, `reset` — và giải thích được vì sao nó render lại **ít hơn hẳn** so với form viết bằng `useState`.
- Đọc hiểu và tự viết được **schema yup**, kể cả bóc nghĩa từng mảnh của hai biểu thức chính quy (regex) số điện thoại và email trong dự án.
- Chỉ ra được **bug nghiêm trọng nhất của cả website**: `cart.forEach(async ...)` không chờ `await`, và tự tay sửa bằng hai cách khác nhau.
- Hiểu vì sao thiết kế "2 request tách rời, không transaction" có thể để lại **đơn hàng rỗng** trong database, và cách làm đúng.
- Kiểm chứng được bằng tab **Network** của trình duyệt và bằng **MongoDB Compass** rằng dữ liệu đã (hoặc chưa) xuống database.

## 📋 Cần chuẩn bị

- Đã hoàn thành [Bài 27 — Giỏ hàng](27-gio-hang.md): trong `localStorage` đang có sẵn vài món.
- Đã đọc [Bài 03](03-kien-thuc-nen.md) phần `async/await` và phần các phương thức mảng (đặc biệt bảng so sánh `forEach`).
- Backend chạy ở cổng **8080**, frontend ở **3000**, MongoDB đang bật.
- Mở sẵn **MongoDB Compass** trỏ vào `mongodb://localhost:27017/yotea`.
- Mở sẵn **DevTools → tab Network** của Chrome/Edge (F12).

> 💡 Đây là bài dài và quan trọng nhất phần 5. Bán hàng mà không lưu được đơn thì cả
> website vô nghĩa. Hãy làm chậm, làm kỹ, và **luôn mở Compass bên cạnh** để nhìn thấy
> dữ liệu thật sự chui xuống đâu.

---

## 1. Sơ đồ luồng — đường đi của một đơn hàng

Đây là bản đồ của cả bài. Mỗi ô là một lớp, mỗi mũi tên là một lần dữ liệu đổi tay.

```
   👤 NGƯỜI DÙNG gõ 4 ô + ghi chú, bấm nút "Đặt hàng"
   │
   ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│ 1. GIAO DIỆN — yotea-fe/src/pages/user/cart/CheckoutPage.js:119-289          │
│    <form onSubmit={handleSubmit(onSubmit)}>                                   │
│      4 <input> + 1 <textarea> + 1 <input type="checkbox">                      │
│      mỗi ô gắn {...register("tênTrường")}                                     │
└───────────────────────────────────────────────────────────────────────────────┘
   │ handleSubmit chặn reload trang, gom giá trị từ DOM
   ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│ 2. VALIDATE — yupResolver(schema)   CheckoutPage.js:16-30 + :44               │
│    ┌─ SAI  → nạp lỗi vào formState.errors → hiện chữ đỏ dưới ô → DỪNG HẲN    │
│    └─ ĐÚNG → gọi onSubmit(dataInput)                                          │
└───────────────────────────────────────────────────────────────────────────────┘
   │ dataInput = { fullName, phone, email, address, message, saveAddress }
   ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│ 3. STATE — Redux đọc giỏ hàng                                                  │
│    cart = useSelector(selectCart)      CheckoutPage.js:36                     │
│    totalPrice tính bằng reduce         CheckoutPage.js:100-107                │
│    user  = useSelector(selectAuth)     CheckoutPage.js:34  (có thể rỗng)      │
└───────────────────────────────────────────────────────────────────────────────┘
   │
   ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│ 4. TẦNG API (axios)  addOrder(orderData)                                       │
│    yotea-fe/src/api/order.js:23-26  →  instance.post("/orders", order)         │
│    baseURL "http://localhost:8080/api"  (api/instance.js:3-5)                  │
└───────────────────────────────────────────────────────────────────────────────┘
   │  HTTP  POST http://localhost:8080/api/orders     ❰ KHÔNG kèm token ❱
   ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│ 5. ROUTE       yotea-be/src/routes/order.js:8                                  │
│    router.post("/orders", create);            ← không middleware nào cả       │
├───────────────────────────────────────────────────────────────────────────────┤
│ 6. CONTROLLER  yotea-be/src/controllers/order.js:3-13                          │
│    new Order(req.body).save()                                                  │
├───────────────────────────────────────────────────────────────────────────────┤
│ 7. MODEL       yotea-be/src/models/order.js:3-43   (Mongoose Schema)           │
├───────────────────────────────────────────────────────────────────────────────┤
│ 8. MONGODB     yotea ▸ collection "orders"  ← 1 document mới ra đời           │
└───────────────────────────────────────────────────────────────────────────────┘
   │  ĐƯỜNG VỀ: res.json(order) → axios trả { data } → const orderId = data._id
   ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│ 9. LẶP GIỎ HÀNG — mỗi món 1 request riêng      CheckoutPage.js:64-89           │
│    cart.forEach(async (item) => { await addOrderDetail({ orderId, ...item }) })│
│         ▲▲▲  ĐÂY LÀ Ổ BUG CỦA CẢ BÀI, xem mục 6.1                            │
└───────────────────────────────────────────────────────────────────────────────┘
   │  N lần  POST http://localhost:8080/api/orderDetail
   ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│ 10. api/orderDetail.js:15-18 → routes/orderDetail.js:6 → controllers/         │
│     orderDetail.js:3-13 → models/orderDetail.js:3-31 → collection            │
│     "orderdetails"  ← N document mới, mỗi cái trỏ về orderId ở bước 8         │
└───────────────────────────────────────────────────────────────────────────────┘
   │
   ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│ 11. DỌN DẸP & ĐIỀU HƯỚNG          CheckoutPage.js:91-94                        │
│     dispatch(finishOrder())  → cartSlice.js:40-42 → state.cart = []           │
│     → redux-persist ghi đè localStorage["persist:root"]                        │
│     toast.success("Đặt hàng thành công")                                       │
│     navigate("/thank-you")  → App.js:152-155 → ThankPage.js                    │
└───────────────────────────────────────────────────────────────────────────────┘
```

Ba con số cần nhớ ngay từ bây giờ: **giỏ có N món ⇒ tổng cộng 1 + N request**. Giỏ 5 món
là 6 request HTTP riêng biệt, **không có gì ràng chúng lại với nhau**. Toàn bộ phần 6 của
bài này nói về hậu quả của con số đó.

---

## 2. react-hook-form — vì sao dự án không dùng `useState` cho form?

### 2.1. Hai trường phái làm form trong React

📖 **Thuật ngữ:**
- *Controlled component* (thành phần bị điều khiển): React **giữ giá trị** của ô input trong state. Mỗi ký tự gõ vào đều đi qua React.
- *Uncontrolled component* (thành phần tự do): giá trị **nằm trong chính thẻ DOM**, React chỉ cầm một cái "tay nắm" (`ref`) để lúc cần thì thò tay vào lấy.

Cùng một ô "Họ và tên", hai cách viết:

```jsx
// ❌ CÁCH 1 — controlled bằng useState. Đoạn này BẠN TỰ VIẾT để so sánh,
//    dự án Yotea KHÔNG dùng cách này ở trang thanh toán.
const [fullName, setFullName] = useState("");
const [errorName, setErrorName] = useState("");

<input value={fullName} onChange={(e) => setFullName(e.target.value)} />
```

```jsx
// ✅ CÁCH 2 — uncontrolled bằng react-hook-form. Đây là cách dự án dùng thật.
const { register } = useForm();

<input {...register("fullName")} />
```

### 2.2. Đếm số lần render — điểm ăn tiền của react-hook-form

Giả sử người dùng gõ họ tên "Nguyễn Văn An" (13 ký tự) vào form thanh toán:

| | Controlled (`useState`) | react-hook-form |
|---|---|---|
| Số lần `setState` | **13** | **0** |
| Số lần `CheckoutPage` render lại | **13** | **0** |
| Bảng "Đơn hàng của bạn" (dòng 248-284) vẽ lại | **13 lần** | 0 lần |
| Với 4 ô, mỗi ô 13 ký tự | **52 lần render** | **0 lần render** |
| Khi nào mới render? | mỗi phím | chỉ khi **danh sách lỗi thay đổi** hoặc lúc submit |

Vì sao react-hook-form không render? Vì `register("fullName")` trải ra 4 thứ:

```js
// register("fullName") trả về đại khái object này — minh hoạ, không phải code dự án
{
  name: "fullName",
  onChange: fn,   // RHF ghi giá trị vào một cái kho nội bộ, KHÔNG gọi setState
  onBlur: fn,
  ref: fn,        // gắn thẳng vào thẻ <input> thật của DOM
}
```

Giá trị người dùng gõ nằm im trong DOM. Chỉ đến khi bấm submit, react-hook-form mới đi
một vòng đọc hết các `ref` rồi gói thành object `dataInput`. Không state → không render.

> 💡 **Mẹo:** với form 2-3 ô đơn giản thì `useState` vẫn ổn. Nhưng form thanh toán này
> nằm cạnh một cái bảng liệt kê giỏ hàng — mỗi lần render lại là vẽ lại cả bảng. Đây
> đúng là chỗ react-hook-form toả sáng.

### 2.3. Bốn thứ lấy ra từ `useForm`

`yotea-fe/src/pages/user/cart/CheckoutPage.js:39-44`

```js
  const {
    register,
    handleSubmit,
    formState: { errors },
    reset,
  } = useForm({ resolver: yupResolver(schema) });
```

| Thứ lấy ra | Dòng | Nhiệm vụ |
|---|---|---|
| `register` | `:40` | Hàm "đăng ký" một ô input với form. Trải bằng `{...register("tên")}` |
| `handleSubmit` | `:41` | Bọc quanh hàm xử lý của bạn: chặn reload trang → chạy validate → chỉ khi hợp lệ mới gọi `onSubmit` |
| `formState: { errors }` | `:42` | Bóc tách lồng nhau: lấy trường `errors` bên trong object `formState`. `errors.phone?.message` là câu tiếng Việt sẽ hiện dưới ô |
| `reset` | `:43` | Xoá trắng form / nạp lại giá trị mặc định |
| `resolver` | `:44` | Cây cầu nối yup vào react-hook-form (xem mục 3.3) |

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** `reset` được bóc ra ở dòng `:43` nhưng **không
> dùng ở đâu cả** trong `CheckoutPage.js` — đây là biến chết. Cách đúng: hoặc xoá đi,
> hoặc dùng nó để tự điền sẵn thông tin người đã đăng nhập (xem Bài tập 2 cuối bài).

### 2.4. `handleSubmit` làm gì bên trong

`yotea-fe/src/pages/user/cart/CheckoutPage.js:119-124`

```jsx
          <form
            action=""
            onSubmit={handleSubmit(onSubmit)}
            method="POST"
            className="container max-w-6xl mx-auto px-3 mt-10 mb-9 grid grid-cols-12 gap-5"
          >
```

Chuỗi sự kiện khi bấm nút "Đặt hàng":

```
bấm <button> (mặc định type="submit")
        │
        ▼
form phát sự kiện submit  →  handleSubmit(onSubmit) nhận sự kiện
        │
        ├── e.preventDefault()        ← chặn trình duyệt reload trang
        ├── gom giá trị từ các ref     ← { fullName: "...", phone: "...", ... }
        ├── đưa cả đống cho yup kiểm    ← qua resolver
        │       ├─ có lỗi  → set formState.errors → render 1 lần → hiện chữ đỏ → HẾT
        │       └─ sạch sẽ → onSubmit(dataInput)  ← hàm của bạn mới được chạy
```

> 💡 Chú ý `action=""` và `method="POST"` ở dòng `:120` và `:122` — hai thuộc tính này
> **hoàn toàn vô dụng** vì `handleSubmit` đã chặn hành vi mặc định của trình duyệt.
> Chúng là di tích từ HTML thuần, để lại cũng không sao nhưng hơi gây hiểu nhầm.

---

## 3. yup — mô tả "dữ liệu hợp lệ là thế nào"

### 3.1. Schema nguyên văn của trang thanh toán

`yotea-fe/src/pages/user/cart/CheckoutPage.js:16-30`

```js
const schema = yup.object().shape({
  fullName: yup.string().required("Vui lòng nhập họ tên"),
  phone: yup
    .string()
    .required("Vui lòng nhập sdt")
    .matches(
      /(84|0[3|5|7|8|9])+([0-9]{8})\b/,
      "Số điện thoại không đúng định dạng"
    ),
  email: yup
    .string()
    .required("Vui lòng nhập email")
    .matches(/^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$/, "Email không đúng định dạng"),
  address: yup.string().required("Vui lòng nhập địa chỉ chi tiết"),
});
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 16 | `yup.object().shape({...})` | "Dữ liệu form là một **object**, hình dáng của nó như sau" |
| 17 | `fullName: yup.string().required(...)` | Họ tên phải là chuỗi, **không được rỗng**. Chuỗi trong `required()` chính là câu báo lỗi tiếng Việt |
| 18-24 | `phone` | Chuỗi + bắt buộc + **khớp regex** số điện thoại |
| 25-28 | `email` | Chuỗi + bắt buộc + **khớp regex** email |
| 29 | `address` | Chuỗi + bắt buộc |

Để ý cách viết **nối chuỗi** (chaining): `yup.string().required(...).matches(...)`. Mỗi
lời gọi trả về chính schema đó nên có thể nối tiếp mãi. Đọc như một câu tiếng Việt:
*"là chuỗi, bắt buộc phải có, và phải khớp mẫu này."*

> ⚠️ **Hai trường KHÔNG có trong schema:** `message` (ghi chú) và `saveAddress`
> (ô tích "Lưu thông tin thanh toán"). yup **không cấm** trường lạ — nó chỉ bỏ qua.
> Nghĩa là hai ô đó **tuỳ chọn**, gõ gì cũng được, kể cả 100.000 ký tự. Ta sẽ vá chỗ
> `message` ở phần "Tự tay làm".

### 3.2. Bóc nghĩa hai biểu thức chính quy

Regex là chỗ người mới sợ nhất. Ta bóc từng mảnh.

#### Regex số điện thoại — `CheckoutPage.js:22`

```
/(84|0[3|5|7|8|9])+([0-9]{8})\b/
 │└──────┬───────┘│└────┬────┘└┬┘
 │       │        │     │      └── \b : ranh giới từ
 │       │        │     └───────── đúng 8 chữ số
 │       │        └─────────────── dấu + : nhóm bên trái lặp 1 lần trở lên
 │       └──────────────────────── nhóm 1: "84"  HOẶC  "0" + 1 ký tự đầu số
 └──────────────────────────────── dấu / mở đầu regex
```

| Mảnh | Nghĩa đen | Ví dụ khớp |
|---|---|---|
| `(...\|...)` | Dấu `\|` là **hoặc**. Nhóm này khớp `84` hoặc `0X` | `84`, `09` |
| `0[3\|5\|7\|8\|9]` | Số `0` rồi **một ký tự** nằm trong lớp `[...]` | `03`, `05`, `09` |
| `+` | Nhóm phía trước lặp **1 lần trở lên** | `0909` (lặp 2 lần) cũng lọt |
| `[0-9]{8}` | Đúng **8** ký tự từ 0 đến 9 | `87654321` |
| `\b` | *Word boundary* — vị trí giao giữa ký tự chữ/số và ký tự không phải chữ/số (hoặc hết chuỗi) | cuối `...4321` |

Ghép lại: `0987654321` → `09` (nhóm 1) + `87654321` (8 số) + hết chuỗi (`\b`) → ✅ hợp lệ.

> ⚠️ **Chỗ này dự án làm chưa chuẩn — regex điện thoại có 3 lỗi:**
>
> 1. **Dấu `|` bên trong `[ ]` là ký tự thường, không phải "hoặc".** Lớp `[3|5|7|8|9]`
>    thật ra là tập hợp **6 ký tự**: `3 5 7 8 9 |`. Nghĩa là `0|12345678` cũng được coi
>    là số điện thoại hợp lệ! Viết đúng phải là `[35789]`.
> 2. **Thiếu `^` và `$`** (neo đầu chuỗi / cuối chuỗi). Regex chỉ cần tìm thấy **một
>    đoạn khớp ở bất kỳ đâu** là pass. Vì vậy `"sđt của tôi là 0987654321"` **vẫn lọt**,
>    và cả `"abc0987654321"` cũng lọt.
> 3. **Dấu `+` là thừa** — số điện thoại đâu có lặp đầu số.
>
> Cách viết đúng (bạn tự viết thêm, dự án chưa có):
> ```js
> .matches(/^(0|\+84)(3|5|7|8|9)[0-9]{8}$/, "Số điện thoại không đúng định dạng")
> ```
> Thử ngay tại <https://regex101.com> với chuỗi `abc0987654321` để thấy sự khác biệt.

#### Regex email — `CheckoutPage.js:28`

```
/^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$/
 │└──┬───┘│└────┬───┘ └────┬────┘│
 │   │    │     │          │     └── $ : phải kết thúc chuỗi ở đây
 │   │    │     │          └──────── đuôi tên miền, dài 2 đến 4 ký tự
 │   │    │     └─────────────────── một hoặc nhiều cụm "chữ + dấu chấm"
 │   │    └───────────────────────── bắt buộc có @
 │   └────────────────────────────── phần tên, gồm chữ/số/_/-/.
 └────────────────────────────────── ^ : phải bắt đầu từ đầu chuỗi
```

| Mảnh | Nghĩa đen |
|---|---|
| `^` | Neo **đầu** chuỗi — không cho phép rác đứng trước |
| `[\w-\.]+` | `\w` = chữ cái, chữ số, dấu `_`. Cộng thêm `-` và `.`. Dấu `+` = một hoặc nhiều |
| `@` | Ký tự `@` bắt buộc |
| `([\w-]+\.)+` | Cụm "một hoặc nhiều ký tự chữ/số/gạch ngang, rồi một dấu chấm", **lặp 1 lần trở lên** → chấp nhận tên miền nhiều cấp như `mail.google.` |
| `[\w-]{2,4}` | Phần đuôi cùng, dài **từ 2 đến 4** ký tự |
| `$` | Neo **cuối** chuỗi |

Thử: `an.nguyen@gmail.com` → `an.nguyen` + `@` + `gmail.` + `com` (3 ký tự) → ✅

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** `[\w-]{2,4}` chặn oan các tên miền hiện đại.
> `ten@congty.online` (6 ký tự), `.travel`, `.museum`, `.technology` đều **bị từ chối**.
> Regex y hệt này bị **copy-paste ở 5 file**: `LoginPage.js`, `RegisterPage.js`,
> `ContactPage.js`, `UpdateInfoPage.js` và `CheckoutPage.js:28`. Sửa thì phải sửa cả 5 —
> đây chính là lý do nên gom regex vào một file `src/utils/validation.js` dùng chung
> (ta sẽ làm ở [Bài 34](34-refactor-du-an.md)).
>
> Bản dễ thở hơn: `/^[\w.-]+@([\w-]+\.)+[a-zA-Z]{2,}$/`

### 3.3. `yupResolver` — cây cầu nối hai thư viện

react-hook-form **không biết** yup là gì. yup **không biết** react-hook-form là gì. Gói
`@hookform/resolvers` đứng giữa làm phiên dịch:

```
       yup schema                    yupResolver                react-hook-form
  ┌─────────────────┐          ┌──────────────────┐        ┌────────────────────┐
  │ phone: chuỗi,   │  ──────► │ nhận dataInput,  │ ─────► │ formState.errors = │
  │ bắt buộc, khớp  │          │ chạy schema,     │        │ { phone: {         │
  │ regex...        │          │ đổi lỗi của yup  │        │     message: "Số   │
  └─────────────────┘          │ sang định dạng   │        │     điện thoại..." │
                               │ mà RHF hiểu      │        │ }}                 │
                               └──────────────────┘        └────────────────────┘
```

Trong dự án chỉ tốn đúng một dòng — `CheckoutPage.js:44`:

```js
  } = useForm({ resolver: yupResolver(schema) });
```

và một dòng import — `CheckoutPage.js:2`:

```js
import { yupResolver } from "@hookform/resolvers/yup";
```

### 3.4. Lỗi hiện ra màn hình ở đâu

`yotea-fe/src/pages/user/cart/CheckoutPage.js:158-167`

```jsx
                  <input
                    type="text"
                    {...register("phone")}
                    id="cart__checkout-form-phone"
                    className="/* ...class Tailwind... */"
                    placeholder="Số điện thoại người nhận hàng"
                  />
                  <div className="text-sm mt-0.5 text-red-500">
                    {errors.phone?.message}
                  </div>
```

| Phần | Ý nghĩa |
|---|---|
| `{...register("phone")}` | Trải `name`, `onChange`, `onBlur`, `ref` vào thẻ input |
| `errors.phone?.message` | Dấu `?.` cực quan trọng: khi **chưa có lỗi** thì `errors.phone` là `undefined`; không có `?.` sẽ nổ `Cannot read properties of undefined` và **trắng cả trang** |
| `text-red-500` | Class Tailwind cho chữ đỏ |

Bốn ô còn lại lặp lại đúng khuôn này: `fullName` (`:140-149`), `email` (`:176-185`),
`address` (`:194-203`).

---

## 4. Soi code thật — hàm `onSubmit`

### 4.1. Toàn văn

`yotea-fe/src/pages/user/cart/CheckoutPage.js:48-95`

```js
  const onSubmit = async (dataInput) => {
    setLoading(true);
    // save order
    const orderData = {
      userId: (user && user._id) || "",
      customerName: dataInput.fullName,
      address: dataInput.address,
      phone: dataInput.phone,
      email: dataInput.email,
      totalPrice,
      message: dataInput.message,
    };
    const { data } = await addOrder(orderData);
    const orderId = data._id;

    // save order detail
    cart.forEach(
      async ({
        productId,
        productPrice,
        sizeId,
        sizePrice,
        quantity,
        ice,
        sugar,
        toppingId,
        toppingPrice,
      }) => {
        await addOrderDetail({
          orderId,
          productId,
          productPrice,
          sizeId,
          sizePrice,
          quantity,
          ice,
          sugar,
          toppingId,
          toppingPrice,
        });
      }
    );

    dispatch(finishOrder());
    setLoading(false);
    toast.success("Đặt hàng thành công");
    navigate("/thank-you");
  };
```

### 4.2. Mổ từng bước

| Dòng | Code | Chuyện gì xảy ra |
|---|---|---|
| 48 | `async (dataInput) => {` | `dataInput` là object react-hook-form gom được, **đã qua yup** |
| 49 | `setLoading(true)` | Bật overlay xoay xoay (`components/Loading.js`) |
| 51-59 | `const orderData = {...}` | **Dịch** tên trường của form sang tên trường của model Order |
| 52 | `userId: (user && user._id) \|\| ""` | Đã đăng nhập → lấy `_id`; khách vãng lai → chuỗi rỗng. Đây là mấu chốt cho phép **mua không cần tài khoản** |
| 53 | `customerName: dataInput.fullName` | Form gọi là `fullName`, database gọi là `customerName` |
| 57 | `totalPrice,` | Viết tắt của `totalPrice: totalPrice` — lấy từ state ở `:37` |
| 60 | `const { data } = await addOrder(orderData)` | **Request thứ nhất.** Bóc `data` khỏi response axios |
| 61 | `const orderId = data._id` | **Đây là mắt xích quan trọng nhất**: `_id` MongoDB vừa sinh, dùng làm khoá ngoại cho mọi dòng chi tiết |
| 64-89 | `cart.forEach(async (...) => {...})` | Lặp giỏ hàng, mỗi món bắn **một request** `POST /api/orderDetail` |
| 65-75 | destructuring trong tham số | Bóc thẳng 9 trường ra khỏi mỗi món trong giỏ |
| 91 | `dispatch(finishOrder())` | `cartSlice.js:40-42` → `state.cart = []` → redux-persist xoá giỏ trong localStorage |
| 92 | `setLoading(false)` | Tắt overlay |
| 93 | `toast.success(...)` | Thông báo xanh góc màn hình |
| 94 | `navigate("/thank-you")` | Đổi URL, React Router vẽ `ThankPage` |

### 4.3. Tầng api — hai hàm gọn lỏn

`yotea-fe/src/api/order.js:23-26`

```js
export const add = (order) => {
  const url = `/${DB_NAME}`;
  return instance.post(url, order);
};
```

`yotea-fe/src/api/orderDetail.js:15-18`

```js
export const add = (orderDetail) => {
  const url = `/${DB_NAME}`;
  return instance.post(url, orderDetail);
};
```

Cả hai file đều khai `DB_NAME` ở đầu: `"orders"` (`api/order.js:4`) và `"orderDetail"`
(`api/orderDetail.js:3`). Ghép với `baseURL` ở `api/instance.js:3-5`:

```js
const instance = axios.create({
  baseURL: "http://localhost:8080/api",
});
```

→ URL cuối cùng là `http://localhost:8080/api/orders` và `http://localhost:8080/api/orderDetail`.

Vì hai hàm `add` trùng tên nhau, `CheckoutPage.js:6-7` phải **đổi tên khi import**:

```js
import { add as addOrder } from "../../../api/order";
import { add as addOrderDetail } from "../../../api/orderDetail";
```

> 💡 **Nhớ nhé:** cú pháp `import { X as Y }` = *"lấy `X` ra nhưng trong file này gọi nó
> là `Y`"*. Rất hay dùng khi hai module xuất trùng tên.

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** đường dẫn `/orderDetail` viết **số ít, camelCase**
> trong khi mọi resource khác dùng số nhiều (`/orders`, `/users`, `/products`). REST
> chuẩn phải là `/order-details` hoặc `/orderDetails`. Bất nhất kiểu này khiến người mới
> vào dự án gõ nhầm liên tục.

### 4.4. Route → Controller → Model — phía backend

`yotea-be/src/routes/order.js:8`

```js
router.post("/orders", create);
```

`yotea-be/src/controllers/order.js:3-13`

```js
export const create = async (req, res) => {
    try {
        const order = await new Order(req.body).save();
        res.json(order);
    } catch (error) {
        res.status(400).json({
            message: "Thêm đơn hàng thất bại",
            error
        });
    }
};
```

`yotea-be/src/models/order.js:3-43` (rút gọn phần lặp)

```js
const orderSchema = new Schema({
    userId: {
        type: String,
    },
    customerName: {
        type: String,
        required: true,
    },
    address: {
        type: String,
        required: true,
    },
    phone: {
        type: String,
        required: true,
    },
    email: {
        type: String,
        required: true,
    },
    totalPrice: {
        type: Number,
        required: true,
    },
    priceDecrease: {
        type: Number,
        default: 0
    },
    message: {
        type: String,
        default: ""
    },
    status: {
        type: Number,
        default: 0
    },
    voucher: {
        type: Array,
        default: []
    },
}, { timestamps: true });
```

**Ba điều đáng chú ý ở model này:**

| Trường | Dòng | Ghi chú |
|---|---|---|
| `userId` | `:4-6` | Kiểu **`String`**, **không có `ref`** — khác hẳn 11 quan hệ còn lại của dự án vốn dùng `ObjectId` + `ref`. Hệ quả: **không `populate()` được** sang User. Xem lại [Bài 10](10-quan-he-va-populate.md) |
| `status` | `:35-38` | `Number`, mặc định `0` = "chờ xác nhận". Trang admin sẽ đổi số này |
| `priceDecrease`, `voucher` | `:27-30`, `:39-42` | **Không bao giờ được ghi giá trị** — `models/voucher.js` tồn tại nhưng chưa có API nào, frontend cũng không gửi. Code chết |

Phía OrderDetail, `yotea-be/src/routes/orderDetail.js:6`:

```js
router.post("/orderDetail", create);
```

`yotea-be/src/controllers/orderDetail.js:3-13`

```js
export const create = async (req, res) => {
    try {
        const orderDetail = await new OrderDetail(req.body).save();
        res.json(orderDetail);
    } catch (error) {
        res.status(400).json({
            message: "Thêm đơn hàng thất bại",
            error
        });
    }
};
```

> 💡 Để ý câu lỗi vẫn là **"Thêm đơn hàng thất bại"** — copy nguyên từ `order.js` sang.
> Khi debug bạn sẽ không phân biệt được lỗi ở bảng nào. Đổi thành "Thêm chi tiết đơn
> hàng thất bại" là việc 5 giây mà tiết kiệm 30 phút cho người sau.

`yotea-be/src/models/orderDetail.js:3-31`

```js
const orderSchema = new Schema(
  {
    orderId: {
      type: ObjectId,
      ref: "Order",
    },
    productId: {
      type: ObjectId,
      ref: "Product",
    },
    productPrice: {
      type: Number,
      required: true,
    },
    quantity: {
      type: Number,
      required: true,
    },
    ice: {
      type: Number,
      required: true,
    },
    sugar: {
      type: Number,
      required: true,
    },
  },
  { timestamps: true }
);
```

**Đếm giúp mình:** schema có đúng **6 trường**. Còn `CheckoutPage.js:76-87` gửi lên
**10 trường**. Bốn trường thừa đi đâu? Mục 6.3 trả lời.

---

## 5. Điều gì xảy ra khi giỏ trống, và trang cảm ơn

### 5.1. Giỏ trống → đá về `/cart`

`yotea-fe/src/pages/user/cart/CheckoutPage.js:113-118` và `:291-296`

```jsx
  return (
    <>
      {cart.length ? (
        <>
          <CartNav page="checkout" />
```

```jsx
      ) : (
        navigate("/cart")
      )}

      <Loading active={loading} />
```

> ⚠️ **Chỗ này dự án làm chưa chuẩn — gọi `navigate()` ngay trong JSX:**
> Đây là **side-effect trong lúc render**. React Router v6 sẽ in cảnh báo đỏ trong
> console: *"You should call navigate() in a React.useEffect(), not when your component
> is first rendered"*.
>
> Tệ hơn nữa: ở dòng `:91` ta vừa `dispatch(finishOrder())` làm `cart` rỗng ⇒ component
> render lại ⇒ nhánh `navigate("/cart")` chạy **cùng lúc** với `navigate("/thank-you")`
> ở dòng `:94`. Hai lệnh điều hướng **đua nhau**, kết quả phụ thuộc thứ tự gom batch của
> React. Có máy nhảy đúng `/thank-you`, có máy nhảy về `/cart` — bug "lúc được lúc không"
> kinh điển.
>
> **Cách đúng** (bạn tự viết thêm):
> ```jsx
> {cart.length ? ( ... ) : <Navigate to="/cart" replace />}
> ```
> với `import { Navigate } from "react-router-dom"`.

### 5.2. `ThankPage.js` — điểm cuối của luồng

`yotea-fe/src/pages/user/cart/ThankPage.js:11-14`

```js
const ThankPage = () => {
  useEffect(() => {
    updateTitle("Đặt hàng thành công");
  }, []);
```

`yotea-fe/src/pages/user/cart/ThankPage.js:16-29`

```jsx
  return (
    <div className="mb-32">
      <CartNav page="thank-you" />

      <section className="container max-w-6xl mx-auto">
        <h1 className="text-center mt-4 font-semibold text-2xl uppercase">
          Đặt hàng thành công
        </h1>

        <p className="text-center mt-2">
          Cảm ơn bạn đã đặt hàng của Tea House. Nhân viên sẽ gọi điện từ số điện
          thoại bạn đã cung cấp để Confirm (Xác nhận) lại với bạn trong thời
          gian sớm nhất để xác nhận đơn hàng.
        </p>
```

Cả file chỉ có **50 dòng** và **không có một dòng logic nào**: không state, không API,
không nhận dữ liệu từ trang trước. Nó chỉ là một trang tĩnh + 2 nút:

| Nút | Dòng | Đi đâu |
|---|---|---|
| "Tiếp tục mua hàng" | `ThankPage.js:32` | `/thuc-don` |
| "Kiểm tra đơn hàng" | `ThankPage.js:38` | `/my-account/cart` |

`CartNav` (`components/user/CartNav.js:5-47`) là thanh 3 bước **SHOPPING CART → CHECKOUT
DETAILS → ORDER COMPLETE**; prop `page` quyết định bước nào được tô đen
(`CartNav.js:13`, `:26`, `:38`).

> ⚠️ **Bốn điểm chưa chuẩn của `ThankPage`:**
> 1. **Không hiển thị mã đơn hàng.** `CheckoutPage` đã có `orderId` trong tay
>    (`CheckoutPage.js:61`) nhưng không truyền sang. Cách đúng:
>    `navigate("/thank-you", { state: { orderId } })` rồi bên kia đọc bằng `useLocation()`.
> 2. **Không có guard.** Gõ thẳng `localhost:3000/thank-you` là vào được, dù chưa mua gì.
> 3. **Sai tên thương hiệu** — dòng `:26` ghi *"Tea House"* trong khi dự án tên **Yotea**.
>    Dấu vết copy từ template khác.
> 4. Nút "Kiểm tra đơn hàng" trỏ `/my-account/cart` — khách vãng lai bấm vào sẽ bị
>    `PrivateRouter` đá thẳng về `/login` (xem [Bài 24](24-private-router.md)).

---

## 6. 🔴 Những chỗ dự án làm sai — mổ xẻ kỹ

### 6.1. BUG TRỌNG TÂM: `forEach` không chờ `await`

Đây là bug nghiêm trọng nhất của cả website. Đọc thật chậm phần này.

`yotea-fe/src/pages/user/cart/CheckoutPage.js:64-89` (trích phần lõi)

```js
    cart.forEach(
      async ({ productId, productPrice, /* ...các trường khác... */ }) => {
        await addOrderDetail({
          orderId,
          productId,
          productPrice,
          /* ...các trường khác... */
        });
      }
    );
```

#### Vì sao `forEach` không chờ?

Hãy nhìn vào định nghĩa (đơn giản hoá) của chính `Array.prototype.forEach`:

```js
// Đây là mô phỏng do BẠN TỰ VIẾT để hiểu bản chất, không phải code dự án
Array.prototype.forEachMoPhong = function (callback) {
  for (let i = 0; i < this.length; i++) {
    callback(this[i], i, this);   // ← gọi xong là đi tiếp NGAY
    //  ▲ giá trị trả về của callback bị VỨT ĐI, không ai nhận
  }
};
```

Mấu chốt nằm ở chỗ `callback(...)` được gọi mà **không có `await`, không ai giữ lại giá
trị trả về**. Callback của ta là hàm `async` ⇒ nó trả về một **Promise**. Promise đó rơi
xuống đất, không ai chờ.

Chữ `await` bên trong callback vẫn hoạt động — nhưng nó chỉ khiến **callback đó** dừng
lại. Nó **không** khiến `forEach` dừng lại.

#### Dòng thời gian thật sự

Giả sử giỏ có 3 món, mỗi request mất 300ms:

```
t=0ms    forEach bắn callback #1  →  addOrderDetail(món 1) đang bay ✈
t=0ms    forEach bắn callback #2  →  addOrderDetail(món 2) đang bay ✈
t=0ms    forEach bắn callback #3  →  addOrderDetail(món 3) đang bay ✈
t=0ms    forEach KẾT THÚC ✔ (nó tưởng xong việc rồi)
t=0ms    dispatch(finishOrder())      ← XOÁ SẠCH GIỎ HÀNG
t=0ms    setLoading(false)            ← tắt overlay dù việc chưa xong
t=0ms    toast.success("Đặt hàng thành công")   ← NÓI DỐI
t=0ms    navigate("/thank-you")       ← ĐỔI TRANG
   ...
t=300ms  3 request kia mới thật sự về đích (nếu người dùng còn ở lại)
```

Nghĩa là: **màn hình báo thành công trước khi dữ liệu kịp lưu.**

#### Hậu quả cụ thể

| Tình huống | Chuyện gì xảy ra |
|---|---|
| Mạng ổn, người dùng ngồi yên | May mắn, 3 request về đích. Không ai biết có bug |
| Người dùng **đóng tab** ngay sau khi thấy chữ "Đặt hàng thành công" | Trình duyệt huỷ các request đang bay ⇒ database có **Order nhưng không có OrderDetail** ⇒ **đơn hàng rỗng** |
| Wifi rớt giữa chừng | Một vài món lưu được, vài món không ⇒ **đơn hàng thiếu món**, mà `totalPrice` thì vẫn tính đủ tiền |
| Một request trả về lỗi 400 | Promise bị reject, **không ai bắt** (`onSubmit` không có `try/catch`, callback cũng không có `.catch`) ⇒ console hiện "Unhandled promise rejection", còn người dùng vẫn thấy chữ xanh "Đặt hàng thành công" |
| Giỏ 20 món | 20 request bắn ra **cùng một lúc** — dội bom server, không kiểm soát |

Thêm một chi tiết: giỏ hàng đã bị `dispatch(finishOrder())` xoá ở dòng `:91`. Nếu request
lỗi, **dữ liệu giỏ hàng đã mất luôn**, người dùng không thể bấm "thử lại".

#### Cách sửa 1 — `for...of` + `await` (tuần tự)

```js
// Đoạn này BẠN TỰ VIẾT — dự án chưa có
for (const item of cart) {
  await addOrderDetail({
    orderId,
    productId: item.productId,
    productPrice: item.productPrice,
    quantity: item.quantity,
    ice: item.ice,
    sugar: item.sugar,
  });
}
```

Vì sao `for...of` chờ được còn `forEach` thì không? Vì `for...of` là **cú pháp của ngôn
ngữ**, `await` bên trong nó dừng chính vòng lặp lại. Còn `forEach` chỉ là **một hàm bình
thường** gọi callback của bạn — nó không hiểu gì về `async`.

- ✅ Ưu: đơn giản, dễ đọc, thứ tự các dòng chi tiết đúng như trong giỏ.
- ❌ Nhược: 5 món × 300ms = **1,5 giây** (chạy nối tiếp).

#### Cách sửa 2 — `Promise.all` (song song nhưng vẫn chờ đủ)

```js
// Đoạn này BẠN TỰ VIẾT — dự án chưa có
await Promise.all(
  cart.map((item) =>
    addOrderDetail({
      orderId,
      productId: item.productId,
      productPrice: item.productPrice,
      quantity: item.quantity,
      ice: item.ice,
      sugar: item.sugar,
    })
  )
);
```

- `cart.map(...)` tạo ra một **mảng Promise** (khác `forEach` ở chỗ `map` **có trả về**).
- `Promise.all([...])` gộp cả mảng thành **một Promise duy nhất**, chỉ hoàn thành khi
  **tất cả** xong; **một cái lỗi là cả cụm reject** ngay.
- ✅ Ưu: 5 món vẫn chỉ ~300ms.
- ❌ Nhược: khi một cái lỗi, những cái kia **vẫn đã lưu rồi** → đơn hàng thiếu món (nhưng
  ít nhất bạn **biết** có lỗi).

> 💡 **Mẹo nhớ:** `forEach` không trả về gì (`undefined`) ⇒ không có gì để `await`.
> `map` trả về mảng ⇒ có thứ để đưa cho `Promise.all`. Khi cần `await` trong vòng lặp,
> **luôn nghĩ tới `for...of` hoặc `map` + `Promise.all`, đừng bao giờ dùng `forEach`.**

#### Bản vá đầy đủ cho `onSubmit` (bạn sẽ gõ ở phần "Tự tay làm")

```js
// Đoạn này BẠN TỰ VIẾT — bản vá đề xuất cho CheckoutPage.js:48-95
const onSubmit = async (dataInput) => {
  setLoading(true);

  try {
    const orderData = {
      userId: (user && user._id) || "",
      customerName: dataInput.fullName,
      address: dataInput.address,
      phone: dataInput.phone,
      email: dataInput.email,
      totalPrice,
      message: dataInput.message,
    };

    const { data } = await addOrder(orderData);
    const orderId = data._id;

    // CHỜ ĐỦ tất cả dòng chi tiết rồi mới đi tiếp
    for (const item of cart) {
      await addOrderDetail({
        orderId,
        productId: item.productId,
        productPrice: item.productPrice,
        quantity: item.quantity,
        ice: item.ice,
        sugar: item.sugar,
      });
    }

    dispatch(finishOrder());          // chỉ xoá giỏ khi CHẮC CHẮN đã lưu xong
    toast.success("Đặt hàng thành công");
    navigate("/thank-you", { state: { orderId } });
  } catch (error) {
    toast.error("Đặt hàng thất bại, vui lòng thử lại");
    // KHÔNG dispatch(finishOrder()) → giữ nguyên giỏ để người dùng bấm lại
  } finally {
    setLoading(false);                // dù thành công hay lỗi cũng phải tắt overlay
  }
};
```

Ba thay đổi cốt lõi: **`try/catch/finally`**, **`for...of` + `await`**, và **chỉ xoá giỏ
sau khi mọi thứ đã xong**.

### 6.2. Không có transaction — đơn hàng rỗng nằm lại vĩnh viễn

Ngay cả khi bạn đã sửa `forEach`, vẫn còn một vấn đề kiến trúc:

```
POST /api/orders        ✅ thành công  →  document Order đã nằm trong DB
POST /api/orderDetail   ❌ lỗi         →  không có dòng chi tiết nào

Kết quả: DB có một Order với totalPrice 120.000đ nhưng KHÔNG CÓ MÓN NÀO.
Không có ai dọn nó đi. Nó nằm đó mãi mãi, làm bẩn báo cáo doanh thu.
```

📖 **Thuật ngữ:** *transaction* (giao dịch) — một nhóm thao tác database được coi là
**một khối không thể chia**: hoặc tất cả cùng thành công, hoặc tất cả cùng bị huỷ bỏ
(rollback). Giống chuyển khoản ngân hàng: không thể trừ tiền tài khoản A mà không cộng
vào tài khoản B.

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** grep toàn bộ `yotea-be/src` không có một chữ
> `startSession` hay `startTransaction` nào. Thiết kế "client tự gọi 2 loại API rồi tự
> ghép" là **sai về nguyên tắc** với nghiệp vụ đặt hàng.

**Hai hướng làm đúng:**

**Hướng A — một endpoint duy nhất (khuyên dùng cho người mới).** Frontend gửi **một**
request chứa cả đơn lẫn danh sách món; backend tự lo phần còn lại:

```js
// Đoạn này BẠN TỰ VIẾT — gợi ý cho yotea-be/src/controllers/order.js
export const createFull = async (req, res) => {
  const { items, ...orderInfo } = req.body;
  try {
    const order = await new Order(orderInfo).save();
    const details = items.map((item) => ({ ...item, orderId: order._id }));
    await OrderDetail.insertMany(details);
    res.json({ order, details });
  } catch (error) {
    res.status(400).json({ message: "Đặt hàng thất bại", error });
  }
};
```

Lợi ích ngay lập tức: **1 request thay vì 1+N**, và client không còn cầm `orderId` để
nghịch. Nhưng nếu `insertMany` lỗi thì `Order` vẫn còn — nên cần thêm hướng B.

**Hướng B — transaction thật của MongoDB:**

```js
// Đoạn này BẠN TỰ VIẾT — cần MongoDB chạy chế độ replica set
const session = await mongoose.startSession();
session.startTransaction();
try {
  const [order] = await Order.create([orderInfo], { session });
  await OrderDetail.insertMany(details, { session });
  await session.commitTransaction();   // chốt sổ: mọi thứ có hiệu lực
} catch (error) {
  await session.abortTransaction();    // huỷ sạch, DB như chưa có gì xảy ra
  throw error;
} finally {
  session.endSession();
}
```

> 💡 MongoDB **chỉ hỗ trợ transaction khi chạy ở chế độ replica set**, còn bản cài đặt
> mặc định trên máy bạn là standalone. Đây là lý do rất nhiều đồ án sinh viên bỏ qua
> transaction. Biết là một chuyện, làm được là chuyện khác — nhưng ít nhất bạn phải
> **biết mình đang thiếu gì**.

### 6.3. Bốn trường bị Mongoose âm thầm nuốt

Nhớ bảng đếm ở mục 4.4 chứ? Frontend gửi 10 trường, schema khai 6.

| Trường FE gửi (`CheckoutPage.js:76-87`) | Có trong `models/orderDetail.js`? | Số phận |
|---|---|---|
| `orderId` | ✅ `:5-8` | Lưu |
| `productId` | ✅ `:9-12` | Lưu |
| `productPrice` | ✅ `:13-16` | Lưu |
| `quantity` | ✅ `:17-20` | Lưu |
| `ice` | ✅ `:21-24` | Lưu |
| `sugar` | ✅ `:25-28` | Lưu |
| `sizeId` | ❌ | **Bị vứt, không báo lỗi** |
| `sizePrice` | ❌ | **Bị vứt, không báo lỗi** |
| `toppingId` | ❌ | **Bị vứt, không báo lỗi** |
| `toppingPrice` | ❌ | **Bị vứt, không báo lỗi** |

> ⚠️ **Chỗ này dự án làm chưa chuẩn — `strict` mode của Mongoose:**
> Mặc định Mongoose bật `strict: true`, nghĩa là **mọi trường không khai trong Schema đều
> bị loại bỏ trước khi ghi xuống MongoDB — im lặng, không cảnh báo, không lỗi**. Request
> vẫn trả về **200 OK**. Đây là loại bug khó chịu nhất: mọi thứ trông như đang chạy đúng.
>
> Đáng nói hơn: cart item tạo ra ở `ProductDetailPage.js` (xem [Bài 26](26-chi-tiet-san-pham.md))
> **vốn dĩ không có** `sizeId`/`sizePrice`/`toppingId`/`toppingPrice` ⇒ chúng luôn là
> `undefined`. Đây là **di tích của một bản thiết kế cũ có size và topping** mà nhóm làm
> dở rồi bỏ.
>
> **Cách phòng tránh:** đặt `{ strict: "throw" }` khi tạo Schema để Mongoose **ném lỗi**
> thay vì im lặng khi gặp trường lạ.

**Kiểm chứng bằng MongoDB Compass — làm ngay 3 phút:**

1. Mở Compass, kết nối `mongodb://localhost:27017`, chọn database `yotea`.
2. Mở Postman, gửi request (cố tình nhét 2 trường lạ):
   ```
   POST http://localhost:8080/api/orderDetail
   Content-Type: application/json

   {
     "orderId": "66b0c1aa2f9d4e0012ab34cd",
     "productId": "66b0c1aa2f9d4e0012ab34ce",
     "productPrice": 35000,
     "quantity": 2,
     "ice": 50,
     "sugar": 70,
     "toppingId": "66b0c1aa2f9d4e0012ab34cf",
     "toppingPrice": 10000
   }
   ```
3. Nhìn response: trả về **200**, không có chữ `toppingId` nào.
4. Vào Compass → collection `orderdetails` → sort theo `createdAt` giảm dần → mở document
   vừa tạo. Bạn thấy đúng 9 trường: `_id`, `orderId`, `productId`, `productPrice`,
   `quantity`, `ice`, `sugar`, `createdAt`, `updatedAt` (+ `__v`). **Hai trường topping
   đã bốc hơi.**

> 💡 **Nối với [Bài 10](10-quan-he-va-populate.md):** ở bài đó bạn đã **tự tay thêm**
> `toppingId` (`type: ObjectId, ref: "Topping"`) và `toppingPrice` vào
> `models/orderDetail.js`. Nếu bạn đã làm, bước 4 ở trên sẽ cho kết quả **khác**:
> `toppingId` xuất hiện đàng hoàng dưới dạng `ObjectId('...')`. Hãy chạy lại thử để tự
> chứng minh: **schema quyết định tất cả, không phải request.**

### 6.4. `totalPrice` do client tính — sửa được giá

`yotea-fe/src/pages/user/cart/CheckoutPage.js:97-111`

```js
  useEffect(() => {
    setLoading(true);

    const getTotalPrice = () => {
      setTotalPrice(() => {
        return cart.reduce((total, cart) => {
          return total + cart.productPrice * cart.quantity;
        }, 0);
      });
    };
    getTotalPrice();

    updateTitle("Thanh toán");
    setLoading(false);
  }, []);
```

Con số này được gửi thẳng lên ở `CheckoutPage.js:57` (`totalPrice,`), và backend
(`controllers/order.js:5`) **nhận nguyên xi, không kiểm tra một chữ nào**.

> 🔒 **Ghi chú bảo mật:** bất kỳ ai mở DevTools → tab Network → chuột phải request
> `POST /api/orders` → *Copy as fetch* → sửa `"totalPrice": 250000` thành `"totalPrice": 0`
> → dán vào Console → **đặt hàng 0 đồng thành công**. Không cần biết lập trình, chỉ cần
> biết bấm chuột phải.
>
> Tệ hơn, `productPrice` trong từng OrderDetail cũng do client gửi ⇒ hoá đơn nội bộ cũng
> khớp với con số giả.
>
> **Nguyên tắc vàng:** *"Never trust the client"* — **mọi con số liên quan tới tiền phải
> được tính lại ở server**. Backend chỉ nên nhận `productId` + `quantity`, rồi tự tra giá
> trong collection `products` và tự cộng. Ta sẽ làm việc này ở
> [Bài 33](33-ra-soat-bao-mat.md).

**Hai lỗi phụ trong chính `useEffect` này:**

1. **`setLoading(true)` ở `:98` và `setLoading(false)` ở `:110` chạy đồng bộ trong cùng
   một effect.** React gom (batch) hai lần set state này lại, kết quả cuối vẫn là `false`
   ⇒ overlay `Loading` **không bao giờ hiện ra**. Đoạn code loading này hoàn toàn vô dụng.
   (Xem `components/Loading.js:3-19` và luật CSS `index.css:78-86` — overlay chỉ hiện khi
   phần tử có class `active`.)
2. **`deps` là `[]` nhưng bên trong dùng `cart`.** `totalPrice` chỉ được tính **một lần
   lúc mount**. Trang này không cho sửa giỏ nên tạm chưa vỡ, nhưng chỉ cần ai đó thêm nút
   "+/-" vào bảng tổng kết là tổng tiền sẽ **đứng im**. Đúng ra phải để `[cart]`, hoặc tốt
   hơn là dùng `useMemo` thay vì `useState` + `useEffect`.

### 6.5. Ô "Lưu thông tin thanh toán?" — chỉ là trang trí

Ta đọc trực tiếp file thật để khỏi phải đoán.

`yotea-fe/src/pages/user/cart/CheckoutPage.js:206-220`

```jsx
                {user && (
                  <div className="col-span-12 mb-3 flex items-center">
                    <input
                      type="checkbox"
                      id="cart__checkout-save-address"
                      {...register("saveAddress")}
                    />
                    <label
                      htmlFor="cart__checkout-save-address"
                      className="ml-1 block text-md"
                    >
                      Lưu thông tin thanh toán?
                    </label>
                  </div>
                )}
```

Ô này **có** `register("saveAddress")` ⇒ giá trị `true`/`false` **có** nằm trong
`dataInput` khi submit. Nhưng hãy đọc lại `onSubmit` (`:48-95`) một lần nữa:
`orderData` (`:51-59`) chỉ lấy `fullName`, `address`, `phone`, `email`, `message`.
**Không có một dòng nào nhắc tới `saveAddress`.**

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** tích hay không tích, kết quả **y hệt nhau**. Đây
> là chức năng "làm giao diện trước, quên code sau" — cùng loại với ô "Ghi nhớ mật khẩu"
> ở `LoginPage.js:115-121` (cái đó thậm chí còn không `register`).
>
> **Cách làm đúng** (bạn tự viết thêm): trong `onSubmit`, sau khi tạo đơn thành công:
> ```js
> if (dataInput.saveAddress && user) {
>   dispatch(updateMyAccount({
>     _id: user._id,
>     phone: dataInput.phone,
>     address: dataInput.address,
>   }));
> }
> ```
> (Thunk `updateMyAccount` nằm ở `redux/authSlice.js`, ta sẽ mổ ở
> [Bài 29](29-tai-khoan-cua-toi.md).)
>
> Và ngược lại, khi form mở ra thì **tự điền sẵn** thông tin người đã đăng nhập bằng
> `reset()` — biến `reset` đang bị bỏ không ở `:43` chính là để làm việc đó.

### 6.6. Bảng tổng hợp các lỗi của trang thanh toán

| # | Lỗi | Vị trí | Mức độ |
|---|---|---|---|
| 1 | `forEach` + `async` không chờ được | `CheckoutPage.js:64-89` | 🔴 Nghiêm trọng |
| 2 | Không transaction, đơn rỗng nằm lại DB | Toàn luồng | 🔴 Nghiêm trọng |
| 3 | `totalPrice`/`productPrice` do client gửi | `:57`, `:79` | 🔴 Nghiêm trọng |
| 4 | Không `try/catch` quanh `addOrder` | `:60` | 🟠 Cao |
| 5 | `navigate()` gọi trong JSX → đua điều hướng | `:291-293` | 🟠 Cao |
| 6 | 4 trường bị Mongoose nuốt im lặng | `:68-74`, `:80-86` | 🟠 Cao |
| 7 | `POST /orders` và `POST /orderDetail` **không cần token** | `routes/order.js:8`, `routes/orderDetail.js:6` | 🟠 Cao |
| 8 | Ô "Lưu thông tin" không xử lý | `:206-220` | 🟡 Trung bình |
| 9 | Loading giả (batch state) | `:98`, `:110` | 🟡 Trung bình |
| 10 | Nút "Đặt hàng" không `disabled` → bấm 2 lần ra 2 đơn | `:285-287` | 🟡 Trung bình |
| 11 | `reset` khai mà không dùng; form không tự điền | `:43` | 🟢 Nhỏ |
| 12 | Regex điện thoại/email sai | `:22`, `:28` | 🟢 Nhỏ |

---

## 7. 🛠️ Tự tay làm

> Mục tiêu phần này: cuối phần bạn sẽ (1) sửa xong bug `forEach` và **nhìn thấy** sự khác
> biệt trên tab Network, (2) đưa được `toppingId` xuống tận MongoDB, (3) chặn được ghi
> chú dài quá 500 ký tự.

> 💡 **Trước khi bắt đầu:** đứng ở thư mục gốc repo, tạo nhánh riêng để lỡ hỏng còn quay
> lại được:
> ```bash
> git checkout -b thuc-hanh-thanh-toan
> ```

### Bước 1 — Sửa `forEach` thành `for...of` + `await`

Mở `yotea-fe/src/pages/user/cart/CheckoutPage.js`, tìm khối `cart.forEach(...)` từ dòng
`64` tới dòng `89`. **Xoá toàn bộ khối đó**, thay bằng:

```js
// yotea-fe/src/pages/user/cart/CheckoutPage.js  ← BẠN SỬA (thay khối :64-89)
    for (const item of cart) {
      await addOrderDetail({
        orderId,
        productId: item.productId,
        productPrice: item.productPrice,
        quantity: item.quantity,
        ice: item.ice,
        sugar: item.sugar,
      });
    }
```

Lưu file. CRA sẽ tự nạp lại trang.

**Kiểm chứng bằng tab Network — đây mới là phần thú vị:**

1. F12 → tab **Network** → tích ô **Preserve log** (để log không bị xoá khi đổi trang).
2. Ở ô lọc, gõ `order` để chỉ hiện các request liên quan.
3. Trong ô **Throttling** (thường ghi "No throttling"), chọn **Slow 3G** — làm mạng chậm
   lại để nhìn rõ.
4. Thêm **3 món** vào giỏ, vào `/checkout`, điền form, bấm "Đặt hàng".
5. Nhìn cột **Waterfall** (biểu đồ thanh ngang bên phải):

```
   TRƯỚC KHI SỬA (forEach) — 3 thanh XẾP CHỒNG, bắt đầu cùng lúc:
   orders       ████
   orderDetail       ██████
   orderDetail       ██████     ← cả 3 khởi hành cùng lúc
   orderDetail       ██████
   (và trang đã nhảy sang /thank-you từ lâu)

   SAU KHI SỬA (for...of) — 4 thanh XẾP BẬC THANG, nối đuôi nhau:
   orders       ████
   orderDetail      ██████
   orderDetail            ██████
   orderDetail                  ██████
                                      ▲ giờ mới nhảy sang /thank-you
```

Bậc thang = **chờ đúng**. Xếp chồng = **không chờ**.

6. Thử nghiệm quyết định: với bản **cũ**, bấm "Đặt hàng" rồi **đóng tab ngay lập tức**.
   Vào Compass, tìm đơn vừa tạo trong `orders`, copy `_id`, rồi lọc `orderdetails` theo
   `{ "orderId": ObjectId("...") }` → bạn sẽ thấy **0 hoặc 1 dòng** thay vì 3.
   Làm lại với bản **mới** → luôn đủ 3 dòng (hoặc chưa tạo đơn nào, chứ không nửa vời).

### Bước 2 — Đưa `toppingId` đã chọn xuống `OrderDetail`

Nhắc lại mạch thực hành: [Bài 10](10-quan-he-va-populate.md) bạn đã thêm `toppingId` +
`toppingPrice` vào `models/orderDetail.js`; [Bài 26](26-chi-tiet-san-pham.md) bạn đã thêm
phần chọn topping vào trang chi tiết sản phẩm, và nhét `toppingId`/`toppingPrice` vào món
trong giỏ; [Bài 27](27-gio-hang.md) bạn đã hiển thị topping trong bảng giỏ hàng. Giờ là
chặng cuối: **đẩy nó xuống database**.

**2a. Kiểm tra món trong giỏ đã có `toppingId` chưa.** Mở DevTools → tab
**Application** → **Local Storage** → `http://localhost:3000` → khoá `persist:root` →
tìm phần `cart`. Mỗi món phải có dạng:

```json
{
  "id": "a3f1...",
  "productId": "66b0...",
  "productName": "Trà sữa trân châu đường đen",
  "productPrice": 35000,
  "quantity": 2,
  "ice": 50,
  "sugar": 70,
  "toppingId": "66b0c1aa2f9d4e0012ab34cf",
  "toppingPrice": 10000
}
```

Nếu chưa có 2 trường cuối, quay lại [Bài 26](26-chi-tiet-san-pham.md) làm nốt.

**2b. Gửi 2 trường đó lên trong vòng lặp.** Sửa lại đoạn vừa viết ở Bước 1:

```js
// yotea-fe/src/pages/user/cart/CheckoutPage.js  ← BẠN SỬA tiếp
    for (const item of cart) {
      await addOrderDetail({
        orderId,
        productId: item.productId,
        productPrice: item.productPrice,
        quantity: item.quantity,
        ice: item.ice,
        sugar: item.sugar,
        toppingId: item.toppingId || null,      // ← MỚI
        toppingPrice: item.toppingPrice || 0,   // ← MỚI
      });
    }
```

| Chi tiết | Vì sao |
|---|---|
| `item.toppingId \|\| null` | Khách không chọn topping → `undefined`. Gửi `undefined` thì axios **bỏ luôn trường đó** khỏi JSON; gửi `null` thì Mongoose ghi `null` rõ ràng — dễ truy vấn hơn |
| `item.toppingPrice \|\| 0` | `toppingPrice` bạn đã đặt `default: 0` ở Bài 10, nhưng gửi hẳn `0` cho tường minh |

**2c. Cộng tiền topping vào tổng.** Công thức hiện tại ở `:100-107` bỏ quên phụ thu
topping. Sửa lại:

```js
// yotea-fe/src/pages/user/cart/CheckoutPage.js  ← BẠN SỬA (thay khối :100-107)
    const getTotalPrice = () => {
      setTotalPrice(
        cart.reduce(
          (total, item) =>
            total + (item.productPrice + (item.toppingPrice || 0)) * item.quantity,
          0
        )
      );
    };
```

> 💡 Nhân tiện, mình cũng bỏ luôn cái `setTotalPrice(() => {...})` lồng hàm ở bản gốc —
> chỉ cần truyền thẳng giá trị. Và đổi tên tham số `cart` trong `reduce` thành `item`
> để không **che mất** (shadow) biến `cart` bên ngoài.

**2d. Hiện topping trong bảng "Đơn hàng của bạn".** Trong khối `cart?.map` ở `:260-274`,
thêm một dòng ngay dưới dòng hiện đá/đường:

```jsx
{/* yotea-fe/src/pages/user/cart/CheckoutPage.js — BẠN THÊM, dưới dòng :268 */}
{item.toppingName && (
  <p className="uppercase">Topping: {item.toppingName}</p>
)}
```

### Bước 3 — Giới hạn ghi chú tối đa 500 ký tự

Mở `CheckoutPage.js`, thêm một dòng vào schema (`:16-30`), ngay sau `address`:

```js
// yotea-fe/src/pages/user/cart/CheckoutPage.js  ← BẠN THÊM vào schema
const schema = yup.object().shape({
  fullName: yup.string().required("Vui lòng nhập họ tên"),
  // ...phone, email giữ nguyên...
  address: yup.string().required("Vui lòng nhập địa chỉ chi tiết"),
  message: yup
    .string()
    .max(500, "Ghi chú tối đa 500 ký tự"),          // ← MỚI
});
```

| Điểm cần hiểu | Giải thích |
|---|---|
| **Không** có `.required()` | Ghi chú là tuỳ chọn — đúng như nhãn "(tuỳ chọn)" ở `:231` |
| `.max(500, "...")` | Với `yup.string()`, `.max` đếm **số ký tự**. (Với `yup.number()` thì `.max` là giá trị lớn nhất — đừng nhầm) |

Rồi thêm chỗ hiện lỗi dưới `<textarea>` (khối `:226-241`):

```jsx
{/* yotea-fe/src/pages/user/cart/CheckoutPage.js — BẠN THÊM, ngay dưới thẻ </textarea> */}
<div className="text-sm mt-0.5 text-red-500">
  {errors.message?.message}
</div>
```

> 💡 `errors.message?.message` trông buồn cười nhưng đúng: `errors.message` là *object
> lỗi của trường tên `message`*, còn `.message` bên trong là *câu thông báo*. Đây là hệ
> quả của việc dự án đặt tên trường ghi chú là `message` — trùng với từ khoá của yup.

---

## 8. ✅ Kiểm chứng kết quả

### 8.1. Chạy dự án

```bash
# Terminal 1 — đứng tại thư mục yotea-be
npm start
```

```bash
# Terminal 2 — đứng tại thư mục yotea-fe
npm start
```

### 8.2. Kịch bản kiểm thử đầy đủ

1. Vào `http://localhost:3000/thuc-don`, thêm **2 sản phẩm khác nhau** vào giỏ, mỗi cái
   chọn topping khác nhau, số lượng khác nhau.
2. Vào `/cart` xem lại → bấm **"Tiến hành thanh toán"**.
3. Ở `/checkout`, **bấm thẳng "Đặt hàng" khi form còn trống** → phải thấy **4 dòng chữ đỏ**:
   ```
   Vui lòng nhập họ tên
   Vui lòng nhập sdt
   Vui lòng nhập email
   Vui lòng nhập địa chỉ chi tiết
   ```
   Mở tab Network → **không có request nào được bắn đi**. Đây là bằng chứng
   `handleSubmit` đã chặn đúng.
4. Gõ số điện thoại `12345` → chữ đỏ đổi thành `Số điện thoại không đúng định dạng`.
5. Gõ email `abc@xyz` (thiếu đuôi) → `Email không đúng định dạng`.
6. Gõ ghi chú dài hơn 500 ký tự (copy dán một đoạn văn) → `Ghi chú tối đa 500 ký tự`.
7. Điền đúng hết: `Nguyễn Văn An` / `0987654321` / `an@gmail.com` / `Số 1, Cầu Giấy` →
   bấm **Đặt hàng**.

### 8.3. Nhìn trên tab Network

Phải thấy đúng **3 request** (1 order + 2 orderDetail), xếp **bậc thang**:

```
POST http://localhost:8080/api/orders            → 200
POST http://localhost:8080/api/orderDetail       → 200
POST http://localhost:8080/api/orderDetail       → 200
```

Bấm vào request `orders` → tab **Payload** phải thấy:

```json
{
  "userId": "",
  "customerName": "Nguyễn Văn An",
  "address": "Số 1, Cầu Giấy",
  "phone": "0987654321",
  "email": "an@gmail.com",
  "totalPrice": 90000,
  "message": ""
}
```

Tab **Response** trả về document đã lưu, **có thêm** `_id`, `status: 0`, `priceDecrease: 0`,
`voucher: []`, `createdAt`, `updatedAt`:

```json
{
  "userId": "",
  "customerName": "Nguyễn Văn An",
  "address": "Số 1, Cầu Giấy",
  "phone": "0987654321",
  "email": "an@gmail.com",
  "totalPrice": 90000,
  "priceDecrease": 0,
  "message": "",
  "status": 0,
  "voucher": [],
  "_id": "66b0c1aa2f9d4e0012ab34cd",
  "createdAt": "2026-08-15T09:12:00.000Z",
  "updatedAt": "2026-08-15T09:12:00.000Z",
  "__v": 0
}
```

Chuỗi `_id` này chính là `orderId` được nhét vào 2 request tiếp theo — hãy **so bằng mắt**
với trường `orderId` trong Payload của request `orderDetail`. Chúng phải **giống hệt nhau**.

### 8.4. Kiểm chứng trong MongoDB Compass

1. Mở Compass → `yotea` → collection **`orders`** → sort `{ createdAt: -1 }` → document
   đầu tiên chính là đơn bạn vừa đặt.
2. Copy giá trị `_id` (Compass hiển thị `ObjectId('66b0...')` — chỉ lấy phần trong nháy).
3. Sang collection **`orderdetails`**, dán vào ô FILTER:
   ```js
   { "orderId": ObjectId("66b0c1aa2f9d4e0012ab34cd") }
   ```
4. Kết quả phải là **đúng 2 document**, mỗi cái có:
   - `productId` kiểu `ObjectId`
   - `quantity`, `ice`, `sugar` đúng như bạn chọn
   - **`toppingId` kiểu `ObjectId` và `toppingPrice` là số** ← thành quả của Bước 2

Nếu `toppingId` **không xuất hiện**: schema `models/orderDetail.js` chưa có trường đó
(quay lại [Bài 10](10-quan-he-va-populate.md)) hoặc bạn quên **khởi động lại backend**
sau khi sửa model.

5. Cuối cùng, kiểm tra giỏ hàng đã sạch: DevTools → Application → Local Storage →
   `persist:root` → `cart` phải là `{"cart":[]}`.

### 8.5. Kiểm chứng bằng API (không cần giao diện)

```
GET http://localhost:8080/api/orderDetail?orderId=66b0c1aa2f9d4e0012ab34cd&_expand=productId
```

Bộ lọc `_expand=productId` (xem [Bài 09](09-bo-loc-query.md)) khiến `productId` **nở**
thành object sản phẩm đầy đủ. Muốn nở cả topping thì thêm khoảng trắng đã mã hoá:

```
GET http://localhost:8080/api/orderDetail?orderId=66b0...&_expand=productId%20toppingId
```

---

## 9. 🐞 Lỗi thường gặp

| Thông báo lỗi | Nguyên nhân | Cách sửa |
|---|---|---|
| Bấm "Đặt hàng" nhưng **trang reload trắng**, URL thêm `?fullName=...` | Quên bọc `handleSubmit`, viết `onSubmit={onSubmit}` | Phải là `onSubmit={handleSubmit(onSubmit)}` (`CheckoutPage.js:121`) |
| Chữ đỏ **không bao giờ hiện** dù bỏ trống ô | Quên `resolver: yupResolver(schema)` trong `useForm` | Xem `CheckoutPage.js:44` |
| `Cannot read properties of undefined (reading 'message')` | Viết `errors.phone.message` thiếu dấu `?.` | Dùng `errors.phone?.message` |
| Ô input gõ được nhưng `dataInput` **thiếu trường đó** | Quên trải `{...register("tên")}`, hoặc gõ sai tên so với schema | Đối chiếu chính tả tên trường giữa `register(...)` và schema |
| `Cannot read properties of undefined (reading '_id')` ở dòng `const orderId = data._id` | `addOrder` thất bại (backend chết / MongoDB chưa bật) nên `data` là `undefined` | Bọc `try/catch` như mục 6.1; kiểm tra terminal backend |
| Overlay loading **xoay mãi không tắt** | `addOrder` ném lỗi, `setLoading(false)` ở `:92` không bao giờ chạy tới | Dùng `finally { setLoading(false) }` |
| `Order validation failed: customerName: Path 'customerName' is required` | Gửi thiếu trường bắt buộc của `models/order.js` | Kiểm tra Payload trong tab Network; nhớ form gọi `fullName` còn model gọi `customerName` |
| `Cast to Number failed for value "abc" at path "totalPrice"` | `totalPrice` không phải số (thường do `cart` rỗng hoặc `productPrice` là chuỗi) | Kiểm tra dữ liệu giỏ trong `persist:root` |
| Trường mới gửi lên **biến mất** trong Compass, không báo lỗi | Mongoose `strict` nuốt trường không có trong Schema | Thêm trường vào `models/orderDetail.js` rồi **restart backend** |
| `Cast to ObjectId failed for value "" at path "toppingId"` | Gửi chuỗi rỗng `""` cho một trường `ObjectId` | Gửi `null` thay vì `""` (xem Bước 2b) |
| `MongooseServerSelectionError` | Chưa bật MongoDB | Chạy `net start MongoDB` (PowerShell chạy bằng quyền Administrator) |
| Đặt hàng xong nhảy về `/cart` thay vì `/thank-you` | Đua điều hướng ở `:291-293` (mục 5.1) | Thay bằng `<Navigate to="/cart" replace />` |
| Bấm nút 2 lần ra **2 đơn hàng** | Nút không `disabled` khi đang gửi | Thêm `disabled={loading}` vào `<button>` ở `:285` |

---

## 10. 📝 Bài tập

**Bài 1.** Đoạn code dưới đây in ra gì, và theo thứ tự nào? Giải thích tại sao.

```js
const ids = [1, 2, 3];

const luu = (id) =>
  new Promise((resolve) => setTimeout(() => { console.log("lưu", id); resolve(); }, 100));

const chay = async () => {
  ids.forEach(async (id) => {
    await luu(id);
  });
  console.log("XONG");
};

chay();
```

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Kết quả:

```
XONG
lưu 1
lưu 2
lưu 3
```

**"XONG" in ra TRƯỚC.** Vì `forEach` gọi 3 callback rồi kết thúc ngay, không chờ Promise
nào cả. Ba dòng `lưu` xuất hiện sau đó khoảng 100ms, gần như cùng lúc (chúng chạy song
song, không phải 100 → 200 → 300ms).

Đây **chính xác** là bug ở `CheckoutPage.js:64-89`: `console.log("XONG")` đóng vai
`dispatch(finishOrder())` + `navigate("/thank-you")`.

Sửa lại cho đúng:

```js
const chay = async () => {
  for (const id of ids) {
    await luu(id);
  }
  console.log("XONG");
};
// → lưu 1 → lưu 2 → lưu 3 → XONG   (mất ~300ms)
```

hoặc:

```js
const chay = async () => {
  await Promise.all(ids.map((id) => luu(id)));
  console.log("XONG");
};
// → lưu 1, lưu 2, lưu 3 (gần như cùng lúc) → XONG   (mất ~100ms)
```

</details>

**Bài 2.** Người dùng **đã đăng nhập** phải gõ lại họ tên, điện thoại, email, địa chỉ mỗi
lần mua hàng — rất phiền. Hãy dùng `reset` (biến đang bị bỏ không ở `CheckoutPage.js:43`)
để tự điền sẵn thông tin từ Redux.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Thêm một `useEffect` mới (đoạn này **bạn tự viết**, dự án chưa có):

```js
// yotea-fe/src/pages/user/cart/CheckoutPage.js — BẠN THÊM, đặt sau useEffect ở :97-111
  useEffect(() => {
    if (user) {
      reset({
        fullName: user.fullName || "",
        phone: user.phone || "",
        email: user.email || "",
        address: user.address || "",
      });
    }
  }, [user, reset]);
```

Ba điểm cần lưu ý:

1. **Liệt kê rõ 4 trường**, đừng viết `reset({ ...user })`. Nếu trải cả object `user` thì
   `_id`, `role`, `active`, `createdAt`… cũng lọt vào form và **được gửi lên server** khi
   submit. Đây chính là lỗ hổng *mass assignment* mà `UpdateInfoPage.js` đang mắc phải
   (xem [Bài 29](29-tai-khoan-cua-toi.md) và [Bài 33](33-ra-soat-bao-mat.md)).
2. **`deps` là `[user, reset]`** chứ không phải `[]` — vì `user` có thể xuất hiện muộn
   (redux-persist cần một nhịp để nạp lại từ localStorage).
3. `|| ""` phòng trường hợp user chưa có địa chỉ — react-hook-form không thích
   `undefined`, nó sẽ tưởng ô đó là *uncontrolled → controlled*.

</details>

**Bài 3.** (Khó) Backend hiện tin tuyệt đối vào `totalPrice` client gửi. Hãy viết lại hàm
`create` của `controllers/order.js` để **tự tính lại tổng tiền ở server**. Giả sử frontend
đã gửi kèm một mảng `items: [{ productId, quantity }]`.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Đoạn này **bạn tự viết**, dự án chưa có. Ý tưởng: **không nhận** `totalPrice` từ client,
mà tra giá thật trong collection `products`.

```js
// yotea-be/src/controllers/order.js  ← phiên bản BẠN TỰ VIẾT
import Order from "../models/order";
import OrderDetail from "../models/orderDetail";
import Product from "../models/product";

export const createSecure = async (req, res) => {
  const { items, ...orderInfo } = req.body;

  try {
    if (!Array.isArray(items) || !items.length) {
      return res.status(400).json({ message: "Đơn hàng không có sản phẩm nào" });
    }

    // 1. Lấy giá THẬT từ database, không tin giá client gửi
    const ids = items.map((item) => item.productId);
    const products = await Product.find({ _id: { $in: ids } }).exec();

    // 2. Dựng bảng tra: productId -> giá
    const bangGia = {};
    products.forEach((p) => { bangGia[p._id.toString()] = p.price; });

    // 3. Tự cộng tổng
    let totalPrice = 0;
    const details = items.map((item) => {
      const gia = bangGia[item.productId];
      if (gia === undefined) throw new Error("Sản phẩm không tồn tại");

      const qnt = Number(item.quantity);
      if (!Number.isInteger(qnt) || qnt <= 0) throw new Error("Số lượng không hợp lệ");

      totalPrice += gia * qnt;
      return { ...item, productPrice: gia, quantity: qnt };
    });

    // 4. Lưu — CHỈ dùng totalPrice do server tính
    const order = await new Order({ ...orderInfo, totalPrice }).save();
    await OrderDetail.insertMany(
      details.map((d) => ({ ...d, orderId: order._id }))
    );

    res.json({ order, details });
  } catch (error) {
    res.status(400).json({ message: "Đặt hàng thất bại", error: error.message });
  }
};
```

Bốn cái lợi cùng lúc:

| Lợi ích | Giải thích |
|---|---|
| **Không sửa được giá** | `totalPrice` và `productPrice` đều do server tra ra |
| **Chỉ 1 request** | Client không còn phải lặp `POST /orderDetail` ⇒ bug `forEach` biến mất theo |
| **Chỉ 1 truy vấn giá** | `$in` lấy tất cả sản phẩm trong một lần, không N+1 |
| **Kiểm tra số lượng** | Chặn `quantity` âm, `0`, hoặc số thập phân |

Còn thiếu gì? **Transaction** (mục 6.2) và **kiểm tra tồn kho**. Nếu `insertMany` lỗi thì
`Order` vẫn nằm lại — muốn kín kẽ hẳn thì phải bọc `session`.

</details>

---

## 📌 Tóm tắt

- Đặt hàng đi qua **8 lớp**: form → yup → `onSubmit` → axios → route → controller → model
  → MongoDB, rồi quay về qua `data._id`, `finishOrder()` và `navigate("/thank-you")`.
- **react-hook-form** giữ giá trị trong DOM (uncontrolled) nên gõ phím **không gây render
  lại** — khác hẳn form viết bằng `useState`. `register` gắn ô input, `handleSubmit` chặn
  reload + chạy validate, `errors` chứa câu báo lỗi, `reset` nạp lại giá trị.
- **yup** mô tả "dữ liệu hợp lệ là thế nào" bằng cách nối chuỗi
  `.string().required().matches()`; **`yupResolver`** là cây cầu nối nó vào react-hook-form.
- Hai regex của dự án đều có lỗi: điện thoại **thiếu neo `^...$`** và hiểu sai dấu `|`
  trong `[ ]`; email **chặn oan** tên miền dài hơn 4 ký tự (`.online`, `.travel`).
- 🔴 **`cart.forEach(async ...)` không chờ `await`** — `forEach` vứt bỏ Promise mà callback
  trả về. Hậu quả: giỏ bị xoá và trang bị chuyển **trước khi** dữ liệu kịp lưu. Sửa bằng
  **`for...of` + `await`** (tuần tự) hoặc **`Promise.all(cart.map(...))`** (song song).
- 🔴 **Không có transaction**: tạo Order xong mà OrderDetail lỗi thì **đơn rỗng nằm lại
  database vĩnh viễn**. Hướng đúng: một endpoint duy nhất tạo cả đơn, hoặc `session` +
  `startTransaction` của MongoDB.
- 🔴 **`totalPrice` do client tính** và backend tin tuyệt đối ⇒ sửa được giá bằng DevTools.
  Nguyên tắc: **mọi con số tiền phải tính lại ở server**.
- **Mongoose `strict` nuốt im lặng** mọi trường không có trong Schema — `sizeId`,
  `sizePrice`, `toppingId`, `toppingPrice` gửi lên nhưng không bao giờ tới MongoDB (trừ
  khi bạn đã tự thêm ở Bài 10). Kiểm chứng bằng Compass, không đoán.
- Ô **"Lưu thông tin thanh toán?"** có `register("saveAddress")` nhưng **không ai đọc giá
  trị** — chức năng chưa làm.
- **`ThankPage.js`** chỉ là trang tĩnh 50 dòng: không mã đơn hàng, không guard, và còn
  ghi nhầm tên thương hiệu "Tea House".

**Từ khoá tra cứu thêm:** `react-hook-form useForm`, `yup schema validation`,
`@hookform/resolvers yup`, `controlled vs uncontrolled components react`,
`async await in forEach loop`, `Promise.all`, `mongoose transaction session`,
`mongoose strict mode`, `never trust the client`

➡️ **Bài tiếp theo:** [29 — Tài khoản của tôi: sửa thông tin, đổi mật khẩu, lịch sử đơn](29-tai-khoan-cua-toi.md) — đơn hàng vừa tạo sẽ hiện ở đâu? Ta đi tìm nó trong `/my-account/cart`, và phát hiện một lỗ hổng cho phép **bất kỳ ai cũng sửa được hồ sơ người khác**.
