# Bài 29 — Tài khoản của tôi: sửa thông tin, đổi mật khẩu, lịch sử đơn

> **Phần 5 · Tính năng phía khách hàng** — Thời lượng ước tính: **~80 phút**
> ⬅️ Bài trước: [28 — Thanh toán: Order + OrderDetail, react-hook-form + yup](28-thanh-toan.md) · Bài sau: [30 — Bình luận, đánh giá sao và yêu thích sản phẩm](30-binh-luan-danh-gia-yeu-thich.md) ➡️
> 🏠 [Mục lục](README.md)

---

## 🎯 Sau bài này bạn sẽ

- Hiểu được cách một **layout con lồng nhau** (`MyAccountLayout` + `<Outlet/>`) dựng khu vực "Tài khoản của tôi" với 4 trang con.
- Đọc hiểu trọn vẹn form **sửa thông tin cá nhân** dùng `react-hook-form` + `yup`, và cách nó nạp dữ liệu từ Redux (`selectAuth`) rồi cập nhật qua thunk `updateMyAccount`.
- Nắm được luồng **đổi mật khẩu**: gọi `POST /api/checkPassword` để xác minh mật khẩu cũ, rồi cập nhật — và hiểu vì sao mật khẩu mới **tự động được băm lại** ở hook `pre("findOneAndUpdate")`.
- Đọc được trang **lịch sử đơn hàng** (`MyCartPage`) và **chi tiết một đơn** (`MyCartDetailPage`), gồm cả bảng chuyển đổi `status` (số) sang nhãn tiếng Việt.
- **Quan trọng nhất:** tự tay tái hiện và vá được **lỗ hổng leo thang admin** ẩn trong API `updateInfo` — lỗ hổng nghiêm trọng nhất của cả dự án.

## 📋 Cần chuẩn bị

- Đã hoàn thành [Bài 28 — Thanh toán](28-thanh-toan.md) và có ít nhất một đơn hàng trong database.
- Đã nắm `react-hook-form` + `yup` ([Bài 28](28-thanh-toan.md)), Redux Toolkit thunk ([Bài 20](20-async-thunk.md)), redux-persist ([Bài 21](21-redux-persist.md)) và phân quyền middleware ([Bài 12](12-phan-quyen-middleware.md)).
- Backend + frontend đang chạy, có Postman để thử API.
- Một tài khoản khách thường (`role = 0`) đã đăng nhập được.

> 💡 Bài này vừa là bài **đọc code nghiệp vụ**, vừa là bài **bảo mật thực chiến**. Nửa
> đầu ta soi 4 trang tài khoản; nửa sau ta mổ một lỗ hổng thật rồi tự tay vá. Đừng bỏ
> phần "Tự tay làm" — nó là bài học đắt giá nhất của bài này.

---

## 1. Bức tranh tổng: khu vực `/my-account/*`

Sau khi đăng nhập, khách bấm vào tên mình trên header sẽ vào `/my-account/`. Đây **không
phải một trang**, mà là một **cụm 4 trang** chia sẻ chung một khung sườn (sidebar bên
trái + vùng nội dung bên phải).

### 1.1. Sơ đồ route con

Trích từ bảng route, `yotea-fe/src/App.js:156-185`:

```jsx
{
  path: "my-account",
  element: (
    <PrivateRouter page="user">
      <MyAccountLayout />
    </PrivateRouter>
  ),
  children: [
    {
      path: "",
      element: <UpdateInfoPage />,
    },
    {
      path: "update-password",
      element: <UpdatePasswordPage />,
    },
    {
      path: "cart",
      element: <MyCartPage />,
    },
    {
      path: "cart/page/:page",
      element: <MyCartPage />,
    },
    {
      path: "cart/:id",
      element: <MyCartDetailPage />,
    },
  ],
},
```

Diễn giải bằng sơ đồ cây:

```
/my-account                     ← PrivateRouter(page="user") → MyAccountLayout (khung sườn)
├── ""                          → UpdateInfoPage        (sửa thông tin cá nhân)
├── update-password             → UpdatePasswordPage     (đổi mật khẩu)
├── cart                        → MyCartPage             (danh sách đơn hàng của tôi)
├── cart/page/:page             → MyCartPage             (danh sách, phân trang)
└── cart/:id                    → MyCartDetailPage        (chi tiết 1 đơn hàng)
```

Hai điểm cốt lõi cần nắm:

1. **Cả cụm được bọc trong `<PrivateRouter page="user">`** — chưa đăng nhập là bị đá về
   `/login` (xem [Bài 24](24-private-router.md)). Đây là điểm khác biệt với `/cart` và
   `/checkout` (hai trang đó **không** bọc `PrivateRouter`, vì dự án cho khách vãng lai
   đặt hàng).
2. **`MyAccountLayout` là layout cha, các trang con hiển thị qua `<Outlet/>`** — sidebar
   được vẽ một lần, còn phần bên phải đổi theo route con. Đây chính là kỹ thuật "layout
   lồng nhau" của React Router v6 bạn đã học ở [Bài 15](15-routing-v6.md), lặp lại y hệt
   mô hình `WebsiteLayout` và `AdminLayout`.

> 📖 **Thuật ngữ:** `<Outlet/>` — "lỗ khoét" trong layout cha, nơi React Router chèn
> component của route con đang khớp vào. Không có `<Outlet/>` thì các trang con không
> bao giờ hiện ra.

---

## 2. Khung sườn — `MyAccountLayout.js`

`yotea-fe/src/pages/layouts/MyAccountLayout.js:1-14`:

```js
import { useDispatch, useSelector } from "react-redux";
import { NavLink, Outlet } from "react-router-dom";
import { logout, selectAuth } from "../../redux/authSlice";
import { clearWishlist } from "../../redux/wishlistSlice";

const MyAccountLayout = () => {
  const dispatch = useDispatch();

  const { user } = useSelector(selectAuth);

  const handleLogout = () => {
    dispatch(logout());
    dispatch(clearWishlist());
  };
```

**Đọc từng phần:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 9 | `const { user } = useSelector(selectAuth)` | Lấy thông tin người đăng nhập từ Redux. Nhớ lại `selectAuth` trả về `state.auth.value` = `{ token, user }`, nên phải bóc thêm một tầng `{ user }` |
| 11-14 | `handleLogout` | Đăng xuất: `dispatch(logout())` xóa phiên + `dispatch(clearWishlist())` dọn danh sách yêu thích của user cũ khỏi store |

Phần JSX (lược bớt class Tailwind dài) — sidebar hiển thị avatar, tên, và 3 `NavLink` +
nút đăng xuất, `yotea-fe/src/pages/layouts/MyAccountLayout.js:26-73`:

```jsx
<header className="flex items-center">
  <img src={user.avatar} /* ...class... */ alt="" />
  <div /* ...class... */>
    <span className="block text-gray-600">{user.fullName}</span>
    <span>{user.username}</span>
  </div>
</header>
<ul /* ...class... */>
  <li className="myAcc-nav__item">
    <NavLink to="/my-account/" /* ...class... */>Thông tin tài khoản</NavLink>
  </li>
  <li className="myAcc-nav__item">
    <NavLink to="/my-account/update-password" /* ...class... */>Đổi mật khẩu</NavLink>
  </li>
  <li className="myAcc-nav__item">
    <NavLink to="/my-account/cart" /* ...class... */>Đơn hàng</NavLink>
  </li>
  <li className="myAcc-nav__item">
    <div /* ...class... */ onClick={() => handleLogout()}>Đăng xuất</div>
  </li>
</ul>
{/* ... */}
<div className="col-span-12 lg:col-span-9">
  <Outlet />
</div>
```

Ba điều đáng chú ý:

1. **Không có `useState`/`useEffect`** — đây là layout thuần, chỉ đọc `user` và render.
2. **Sau `handleLogout` không có `navigate`.** Vậy làm sao rời trang? Nhờ `logout()` set
   `isLogged = false`, `PrivateRouter` re-render và trả `<Navigate to="/login" />`
   ([Bài 24](24-private-router.md)). May mắn là `PrivateRouter` kiểm `!isLogged` **trước**
   khi đọc `auth.user.active`, nên không nổ `TypeError` khi `value = {}`.

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** sidebar bị gắn class `hidden lg:block`
> (`MyAccountLayout.js:25`) → **trên điện thoại, toàn bộ menu tài khoản và nút Đăng xuất
> biến mất**. Khách dùng mobile vào `/my-account` sẽ chỉ thấy nội dung bên phải, không có
> cách nào chuyển sang trang khác hay đăng xuất. Cách đúng: làm menu dạng dropdown/tab
> hiển thị được cả trên mobile.

---

## 3. Trang sửa thông tin — `UpdateInfoPage.js`

Đây là trang mặc định của `/my-account/` (route `path: ""`). Nhiệm vụ: cho khách sửa họ
tên, số điện thoại, email, địa chỉ và ảnh đại diện.

### 3.1. Schema yup

`yotea-fe/src/pages/user/my-account/UpdateInfoPage.js:11-25`:

```js
const schema = yup.object().shape({
  fullName: yup.string().required("Vui lòng nhập họ tên"),
  email: yup
    .string()
    .required("Vui lòng nhập email")
    .matches(/^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$/, "Email không đúng định dạng"),
  address: yup.string().required("Vui lòng nhập địa chỉ chi tiết"),
  phone: yup
    .string()
    .required("Vui lòng nhập sdt")
    .matches(
      /(84|0[3|5|7|8|9])+([0-9]{8})\b/,
      "Số điện thoại không đúng định dạng"
    ),
});
```

Chỉ **4 trường** được kiểm tra: `fullName`, `email`, `address`, `phone`. (Ảnh `avatar` là
file, không nằm trong schema.)

### 3.2. Nạp dữ liệu ban đầu từ Redux vào form

`yotea-fe/src/pages/user/my-account/UpdateInfoPage.js:33-48`:

```js
  const { user } = useSelector(selectAuth);
  const dispatch = useDispatch();

  useEffect(() => {
    updateTitle("Cập nhật tài khoản");
    const start = async () => {
      setLoading(true);

      setPreview(user.avatar);
      reset({
        ...user,
      });
      setLoading(false);
    };
    start();
  }, []);
```

Điểm hay cần học: **`reset({ ...user })`**. `react-hook-form` có hàm `reset(values)` để
**đổ dữ liệu có sẵn vào toàn bộ các ô input** cùng lúc. Ở đây nó lấy nguyên object `user`
từ Redux (đã persist trong localStorage) và điền vào form → khách mở trang lên là thấy
sẵn thông tin cũ của mình, khỏi phải gõ lại.

### 3.3. Gửi cập nhật — `onSubmit`

`yotea-fe/src/pages/user/my-account/UpdateInfoPage.js:61-72`:

```js
  const onSubmit = async (dataInput) => {
    try {
      if (typeof dataInput.avatar === "object" && dataInput.avatar.length) {
        dataInput.avatar = await uploadFile(dataInput.avatar[0]);
      }

      dispatch(updateMyAccount(dataInput));
      toast.success("Cập nhật tài khoản thành công");
    } catch (error) {
      toast.error("Đã có lỗi xảy ra");
    }
  };
```

**Đọc từng bước:**

| Dòng | Ý nghĩa |
|---|---|
| 63 | Nếu người dùng có chọn file ảnh mới (`avatar` là object `FileList` có phần tử) thì upload lên Cloudinary trước, đổi `avatar` thành URL trả về. Nếu không chọn ảnh, `avatar` giữ nguyên URL cũ (do `reset({...user})` đã điền sẵn) |
| 67 | Dispatch thunk `updateMyAccount(dataInput)` — đây là nơi gọi API thật |
| 68 | Báo thành công **ngay lập tức** |

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** `dispatch(updateMyAccount(...))` **không có
> `await`**. Vì thế toast "Cập nhật thành công" hiện ra ngay cả khi API **chưa** xong,
> thậm chí khi API **thất bại**. Cả `try/catch` ở đây gần như vô dụng — lỗi trong thunk
> bị nuốt bởi Redux Toolkit chứ không rơi vào `catch` này. Cách đúng:
> `await dispatch(updateMyAccount(dataInput)).unwrap()` rồi mới toast, và bắt lỗi bằng
> `.catch`.

### 3.4. Thunk `updateMyAccount` trong authSlice

Đây là trái tim của luồng. `yotea-fe/src/redux/authSlice.js:1-31`:

```js
import { createAsyncThunk, createSlice } from "@reduxjs/toolkit";
import { get, update, updateMyInfo } from "../api/user";

const initialState = {
  isLogged: false,
  value: {},
};

export const updateAuth = createAsyncThunk(
  "auth/updateAuth",
  async (dataAuth) => {
    return update(dataAuth).then(async () => {
      const {
        data: { password, ...rest },
      } = await get(dataAuth._id);
      return rest;
    });
  }
);

export const updateMyAccount = createAsyncThunk(
  "auth/updateMyAccount",
  async (dataAuth) => {
    return updateMyInfo(dataAuth).then(async () => {
      const {
        data: { password, ...rest },
      } = await get(dataAuth._id);
      return rest;
    });
  }
);
```

Và phần lưu kết quả vào state, `yotea-fe/src/redux/authSlice.js:46-60`:

```js
  extraReducers: (builder) => {
    builder.addCase(updateAuth.fulfilled, (state, { payload }) => {
      state.value.user = payload;
    });

    builder.addCase(updateMyAccount.fulfilled, (state, { payload }) => {
      state.value.user = payload;
    });
  },
});

export const { signin, logout } = authSlice.actions;
export const selectStatusLoggin = (state) => state.auth.isLogged;
export const selectAuth = (state) => state.auth.value;
export default authSlice.reducer;
```

**Mẫu "cập nhật xong → tải lại → loại password"** (rất đáng học thuộc):

1. `updateMyInfo(dataAuth)` — gọi `PUT /api/users/updateInfo/...` để ghi thay đổi xuống DB.
2. `.then(async () => ...)` — sau khi ghi xong, **gọi lại `get(dataAuth._id)`** để tải
   bản ghi user **mới nhất** từ server (thay vì tin dữ liệu client vừa gõ).
3. `const { data: { password, ...rest } } = ...` — bóc tách lồng nhau, **tách `password`
   ra khỏi dữ liệu**, chỉ giữ `rest` (mẹo "omit" bạn đã gặp ở [Bài 03](03-kien-thuc-nen.md)).
4. `return rest` → rơi vào `updateMyAccount.fulfilled` → `state.value.user = payload`.

Nhờ đó Redux (và localStorage qua redux-persist) luôn có bản user sạch, **không bao giờ
dính chuỗi băm mật khẩu**.

> 💡 **Để ý:** `updateAuth` và `updateMyAccount` gần như y hệt nhau, chỉ khác `updateAuth`
> gọi `update` (route admin `/users/:id/:userId`) còn `updateMyAccount` gọi `updateMyInfo`
> (route khách `/users/updateInfo/:myId/:userId`). Trang tài khoản của khách dùng cái thứ
> hai.

### 3.5. Kiểm tra thật: `wardsCode` / `districtCode` / `provinceCode` có được dùng không?

Model user khai báo 3 trường mã địa chỉ, `yotea-be/src/models/user.js:30-44`:

```js
    wardsCode: {
      type: Number,
      required: true,
      default: 0,
    },
    districtCode: {
      type: Number,
      required: true,
      default: 0,
    },
    provinceCode: {
      type: Number,
      required: true,
      default: 0,
    },
```

Nghe tên có vẻ dùng để lưu mã Phường/Quận/Tỉnh (chuẩn API địa giới hành chính Việt Nam).
Nhưng khi soi form `UpdateInfoPage` (mục 3.1) — nó **chỉ thu thập** `fullName`, `phone`,
`avatar`, `email`, `address`. Soi tiếp form đăng ký `RegisterPage.js` cũng **không** có ba
trường này. Grep toàn bộ frontend cũng không thấy nơi nào set chúng.

> ⚠️ **Trường chết:** `wardsCode`, `districtCode`, `provinceCode` **không có bất kỳ giao
> diện nào ghi giá trị**. Chúng luôn giữ `default: 0` cho mọi user, mãi mãi. Đây là di
> tích của một tính năng "chọn địa chỉ theo Phường/Quận/Tỉnh" bị bỏ dở — schema có, UI
> không. `address` (một ô text tự do) mới là thứ thật sự lưu địa chỉ. Đừng nhầm rằng dự
> án có hệ thống địa chỉ hoàn chỉnh.

---

## 4. Trang đổi mật khẩu — `UpdatePasswordPage.js`

Luồng đổi mật khẩu gồm 2 bước: **(1) xác minh mật khẩu cũ** ngay khi người dùng gõ, rồi
**(2) ghi mật khẩu mới** khi submit.

### 4.1. Xác minh mật khẩu cũ ngay trong schema yup

`yotea-fe/src/pages/user/my-account/UpdatePasswordPage.js:15-52`:

```js
  const schema = yup.object().shape({
    oldPassword: yup
      .string()
      .required("Vui lòng nhập mật khẩu hiện tại")
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
    newPassword: yup
      .string()
      .required("Vui lòng nhập mật khẩu mới")
      .min(4, "Vui lòng nhập mật khẩu tối thiểu 4 ký tự"),
    confirmPassword: yup
      .string()
      .required("Vui lòng xác nhận mật khẩu")
      .test(
        "is_confirm",
        "Mật khẩu xác nhận không chính xác",
        function (value) {
          const { newPassword } = this.parent;

          return newPassword === value;
        }
      ),
  });
```

Điểm rất hay: `oldPassword` dùng **`.test()` bất đồng bộ** (`async function`). Mỗi lần yup
kiểm tra trường này, nó gọi thẳng `checkPassword({ _id: user._id, password: value })` lên
backend. API trả `{ success: true }` thì test đậu; ngược lại hiện lỗi "Mật khẩu hiện tại
không chính xác". Nghĩa là **backend là trọng tài** cho việc "mật khẩu cũ có đúng không",
frontend không tự đoán.

`checkPassword` chỉ là wrapper axios, `yotea-fe/src/api/auth.js:13-16`:

```js
export const checkPassword = (data) => {
  const url = `/checkPassword`;
  return instance.post(url, data);
};
```

`confirmPassword` thì dùng `.test()` **đồng bộ** so `newPassword === value` — mẹo
`this.parent` để đọc trường anh em trong cùng schema (bạn đã gặp ở `RegisterPage`).

### 4.2. Ghi mật khẩu mới — `onSubmit`

`yotea-fe/src/pages/user/my-account/UpdatePasswordPage.js:61-69`:

```js
  const onSubmit = async (dataInput) => {
    try {
      await updateMyInfo({ _id: user._id, password: dataInput.newPassword });
      toast.success("Cập nhật mật khẩu thành công");
      reset();
    } catch (error) {
      toast.error("Đã có lỗi xảy ra, vui lòng thử lại");
    }
  };
```

Chỉ gửi đúng 2 trường: `_id` và `password` (mật khẩu mới). Gọi thẳng `updateMyInfo`
(không qua thunk `updateMyAccount`), vì đổi mật khẩu **không cần** cập nhật lại state auth
— mật khẩu chẳng bao giờ nằm trong Redux.

### 4.3. Mật khẩu mới được băm ở đâu?

Câu hỏi quan trọng: `onSubmit` gửi `password: dataInput.newPassword` — tức **mật khẩu
dạng chữ thô**. Vậy ai băm nó?

Không phải controller. Mà là **hook `pre("findOneAndUpdate")` của Mongoose**,
`yotea-be/src/models/user.js:87-94`:

```js
userSchema.pre("findOneAndUpdate", function (next) {
  if (this._update.password) {
    this._update.password = createHmac("SHA256", "TuongVy")
      .update(this._update.password)
      .digest("hex");
  }
  next();
});
```

Cơ chế:

1. `updateMyInfo` gửi `PUT /api/users/updateInfo/...` với body `{ _id, password }`.
2. Route đó gọi controller `update`, mà `update` dùng `findOneAndUpdate` (xem mục 5).
3. **Ngay trước khi `findOneAndUpdate` chạy**, Mongoose kích hoạt hook này. Nó thấy
   `this._update.password` có giá trị → băm lại bằng `createHmac("SHA256", "TuongVy")` →
   ghi đè giá trị đã băm vào `_update.password`.
4. Kết quả: DB lưu chuỗi băm, không bao giờ lưu mật khẩu thô.

> 🔒 **Ghi chú bảo mật:** cách băm này rất yếu — SHA256 **không salt**, khóa `"TuongVy"`
> viết cứng trong code và dùng chung cho cả việc ký JWT. Hai người đặt cùng mật khẩu
> `123456` sẽ ra **cùng một chuỗi băm**. Cách đúng là dùng **bcrypt/argon2** (tự sinh salt,
> cố tình chậm). Phân tích đầy đủ ở [Bài 33](33-ra-soat-bao-mat.md) và cách vá ở
> [Bài 34](34-refactor-du-an.md).

> ⚠️ **Chỗ này dự án làm chưa chuẩn (rủi ro lớn):** `POST /api/checkPassword` **không có
> middleware `requireSignin`**. Chỉ cần biết `_id` của một user (mà `GET /api/users` lại
> trả công khai toàn bộ danh sách user — xem mục 6), kẻ xấu có thể **dò mật khẩu (brute
> force)** của bất kỳ ai bằng cách gọi `checkPassword` liên tục, không token, không giới
> hạn số lần. Cách đúng: bắt buộc đăng nhập, chỉ cho kiểm mật khẩu của **chính mình**, và
> giới hạn tần suất (rate limit).

---

## 5. ⚠️ TRỌNG TÂM: lỗ hổng leo thang admin trong `updateInfo`

Đây là lỗ hổng **nghiêm trọng nhất của toàn dự án**. Ta sẽ ghép 3 mảnh code lại để thấy
vì sao.

### 5.1. Mảnh 1 — route thiếu `isAdmin`

`yotea-be/src/routes/users.js:8-13`:

```js
router.post("/users/:userId", requireSignin, isAuth, isAdmin, create);
router.get("/users/:id", read);
router.get("/users", list);
router.put("/users/:id/:userId", requireSignin, isAuth, isAdmin, update);
router.put("/users/updateInfo/:myId/:userId", requireSignin, isAuth, update);
router.delete("/users/:id/:userId", requireSignin, isAuth, isAdmin, remove);
```

Để ý dòng 12 (`updateInfo`): nó chỉ có `requireSignin, isAuth` — **thiếu `isAdmin`**.
Đây là chủ ý: khách thường (không phải admin) phải sửa được hồ sơ của **chính mình**. Cho
tới đây thì hợp lý.

### 5.2. Mảnh 2 — `isAuth` chỉ kiểm `:userId`, KHÔNG kiểm `:myId`

`yotea-be/src/middlewares/checkAuth.js:9-19`:

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

- `req.auth._id` = id người **thật sự** đang đăng nhập (giải mã từ token).
- `req.profile._id` = user nạp từ **`:userId`** trên URL (do `router.param("userId", userById)`).

Vậy `isAuth` chỉ xác nhận: *"`:userId` trên URL đúng là chủ nhân của token"*. Nhưng nhìn
lại URL: `/users/updateInfo/:myId/:userId`. **Người bị sửa là `:myId`, không phải
`:userId`.** Và **không có middleware nào đụng tới `:myId`**.

### 5.3. Mảnh 3 — controller nhét thẳng `req.body`, không lọc trường

`yotea-be/src/controllers/user.js:225-240`:

```js
export const update = async (req, res) => {
    const filter = { _id: req.params.id || req.params.myId };
    const update = req.body;

    const options = { new: true };

    try {
        const user = await User.findOneAndUpdate(filter, update, options).exec();
        res.json(user);
    } catch (error) {
        res.status(400).json({
            message: "Cập nhật user không thành công",
            error
        });
    }
};
```

- Dòng 226: bản ghi bị sửa được xác định bởi `req.params.id || req.params.myId`. Với route
  `updateInfo` thì `id` là `undefined`, nên nó dùng **`myId`** — chính là tham số **không
  ai kiểm tra**.
- Dòng 227: `const update = req.body` — lấy **nguyên** body client gửi, **không lọc trường
  nào cả**. Đây là lỗi **mass assignment**: client gửi trường gì, DB ghi trường đó.

### 5.4. Ghép lại — kịch bản khai thác bằng Postman

Ba mảnh trên cộng lại thành thảm họa. Một khách thường đã đăng nhập có thể làm **hai
việc**:

**Kịch bản A — tự phong mình làm admin:**

```
PUT http://localhost:8080/api/users/updateInfo/<_id-cua-ban>/<_id-cua-ban>
Headers:
  Authorization: Bearer <token-cua-ban>
  Content-Type: application/json
Body (JSON):
{
  "role": 1
}
```

`isAuth` đậu (vì `:userId` = id của bạn, khớp token). Controller ghi `role: 1` vào chính
hồ sơ bạn. Đăng xuất, đăng nhập lại → bạn **có menu admin thật** (khác với việc chỉ sửa
`role` trong localStorage — cái đó chỉ qua mặt giao diện, còn đây sửa **thẳng vào
database**).

**Kịch bản B — chiếm/phá hồ sơ người khác:**

```
PUT http://localhost:8080/api/users/updateInfo/<_id-nan-nhan>/<_id-cua-ban>
Headers:
  Authorization: Bearer <token-cua-ban>
Body (JSON):
{
  "password": "toi-doi-mat-khau-cua-ban",
  "phone": "0000000000"
}
```

`:userId` vẫn là id của bạn nên `isAuth` đậu, nhưng `:myId` là **nạn nhân** → bạn vừa đổi
mật khẩu người khác (hook `pre("findOneAndUpdate")` sẽ băm lại giúp bạn luôn), chiếm được
tài khoản của họ.

> 🔒 **Vì sao xảy ra:** vi phạm nguyên tắc vàng — **backend tin dữ liệu (id + role) do
> client gửi qua URL và body**. Hai sai lầm ghép lại: (1) xác định "sửa ai" bằng tham số
> URL không được kiểm; (2) không có danh sách trắng trường được sửa.
>
> **Hướng vá (làm ngay ở mục 7):**
> 1. Lấy id người dùng **từ token** (`req.auth._id`), **không** từ URL.
> 2. Chỉ cho cập nhật một **danh sách trắng** trường an toàn (`fullName`, `phone`,
>    `address`), chặn tuyệt đối `role`, `active`, `password` (đổi mật khẩu phải là luồng
>    riêng có xác minh mật khẩu cũ).
>
> Đây là lỗ hổng #1 trong [Bài 33 — Rà soát bảo mật](33-ra-soat-bao-mat.md).

---

## 6. Trang lịch sử đơn hàng — `MyCartPage.js` & `MyCartDetailPage.js`

### 6.1. Danh sách đơn của tôi + phân trang

`yotea-fe/src/pages/user/my-account/MyCartPage.js:9-35`:

```js
const MyCartPage = () => {
  const [orders, setOrders] = useState();
  const [totalOrder, setTotalOrder] = useState(0);
  const { user } = useSelector(selectAuth);

  const { page } = useParams();

  const limit = 10;
  const totalPage = Math.ceil(totalOrder / limit);
  let currentPage = Number(page) || 1;
  currentPage =
    currentPage < 1 ? 1 : currentPage > totalPage ? totalPage : currentPage;
  const start = (currentPage - 1) * limit > 0 ? (currentPage - 1) * limit : 0;

  useEffect(() => {
    const getOrders = async () => {
      const { data } = await getByUserId(user._id);
      setTotalOrder(data.length);
      const { data: ordersData } = await getByUserId(user._id, start, limit);
      setOrders(ordersData);
    };
    getOrders();
  }, [currentPage]);

  useEffect(() => {
    updateTitle("Đơn hàng của tôi");
  }, []);
```

- Công thức phân trang **giống hệt** `ProductContent` bạn học ở [Bài 25](25-danh-sach-san-pham.md):
  `limit = 10`, `totalPage = ceil(totalOrder/limit)`, kẹp `currentPage` vào `[1, totalPage]`,
  tính `start`.
- Gọi API **2 lượt**: lượt 1 `getByUserId(user._id)` (không limit) chỉ để **đếm tổng**;
  lượt 2 `getByUserId(user._id, start, limit)` để lấy đúng một trang.
- `getByUserId` build query `?userId=...&_sort=createdAt&_order=desc` — tức lấy đơn **của
  user hiện tại**, mới nhất trước (`yotea-fe/src/api/order.js:17-21`).

### 6.2. Bảng chuyển `status` (số) sang nhãn tiếng Việt

Trong dự án, `status` của đơn là **một con số** (`0`–`4`). Trang danh sách chuyển nó thành
chữ bằng chuỗi tam phân, `yotea-fe/src/pages/user/my-account/MyCartPage.js:94-102`:

```jsx
{!item.status
  ? "Đơn hàng mới"
  : item.status === 1
  ? "Đã xác nhận"
  : item.status === 2
  ? "Đang giao hàng"
  : item.status === 3
  ? "Đã giao hàng"
  : "Đã hủy"}
```

Lấy đúng từ code, bảng nghĩa là:

| `status` | Nhãn (MyCartPage) | Màu badge |
|---|---|---|
| 0 | Đơn hàng mới | Xanh (`status !== 4`) |
| 1 | Đã xác nhận | Xanh |
| 2 | Đang giao hàng | Xanh |
| 3 | Đã giao hàng | Xanh |
| 4 | Đã hủy | Đỏ (`bg-[#FFE2E5]`) |

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** nhãn cho cùng một `status` **không nhất quán** giữa
> các nơi. Với `status = 0`: bảng danh sách ghi "Đơn hàng mới", nhưng ô `<select>` lọc phía
> trên (`MyCartPage.js:51`) ghi "Chờ xác nhận", còn trang chi tiết (mục 6.3) ghi "Đang chờ
> xác nhận". Ba tên cho một trạng thái. Ngoài ra `<option value={3}>` còn gõ sai thành "Đã
> giao **hoàng**" (`MyCartPage.js:54`). Cách đúng: định nghĩa **một** bảng trạng thái dùng
> chung (ví dụ một object `ORDER_STATUS`) rồi tra ở mọi nơi.

> ⚠️ **UI chết:** ô tìm kiếm và `<select>` trạng thái (`MyCartPage.js:39-57`) **không có
> `onChange`, không có handler nào** — chúng chỉ là trang trí, gõ/chọn không lọc được gì.

### 6.3. Chi tiết một đơn — `_expand` để "nở" sản phẩm

`yotea-fe/src/pages/user/my-account/MyCartDetailPage.js:14-26`:

```js
  useEffect(() => {
    const getOrder = async () => {
      const { data } = await get(id);
      setOrder(data);
    };
    getOrder();

    const getOrderDetail = async () => {
      const { data } = await getOrderById(id);
      setOrderDetail(data);
    };
    getOrderDetail();
  }, []);
```

Trang này gọi **2 API**:

1. `get(id)` → `GET /api/orders/:id` — lấy thông tin đơn (tên, sđt, địa chỉ, tổng tiền...).
2. `getOrderById(id)` — thực chất là `get` từ `api/orderDetail`, `yotea-fe/src/api/orderDetail.js:10-13`:

```js
export const get = (orderId) => {
  const url = `/${DB_NAME}/?orderId=${orderId}&_expand=productId`;
  return instance.get(url);
};
```

Chú ý `_expand=productId`. Nhớ lại [Bài 09](09-bo-loc-query.md) và [Bài 10](10-quan-he-va-populate.md):
`_expand` bảo backend `populate` khóa ngoại `productId` thành **object sản phẩm đầy đủ**.
Nhờ đó trong JSX ta truy cập được `item.productId.image`, `item.productId.name`,
`item.productId.slug`, `yotea-fe/src/pages/user/my-account/MyCartDetailPage.js:98-124`:

```jsx
{orderDetail?.map((item, index) => (
  <tr className="border-b" key={index}>
    <td>{++index}</td>
    <td className="py-2 flex items-center">
      <img src={item.productId.image} /* ...class... */ alt="" />
      <div className="pl-3">
        <Link to={`/san-pham/${item.productId.slug}`} className="text-blue-500">
          {item.productId.name}
        </Link>
        <div className="text-sm">
          <p>Đá: {item.ice}%</p>
          <p>Đường: {item.sugar}%</p>
        </div>
      </div>
    </td>
    <td className="py-2">{formatCurrency(item.productPrice)}</td>
    <td className="py-2">{item.quantity}</td>
    <td className="py-2 text-right text-black font-medium">
      {formatCurrency(item.productPrice * item.quantity)}
    </td>
  </tr>
))}
```

Đơn còn có nút **Hủy đơn** (chỉ hiện khi `status` là 0 hoặc 1), gọi `update({...order, status: 4})`
qua `PUT /api/orders/:id/:userId` (kèm token).

> ⚠️ **Nhắc lại lỗi ở [Bài 28](28-thanh-toan.md):** chi tiết đơn chỉ hiện Đá/Đường, **không
> có size và topping** — vì model `OrderDetail` không khai báo `sizeId`/`toppingId`, nên dù
> `CheckoutPage` có gửi lên thì Mongoose cũng âm thầm bỏ đi. Đây chính là lý do mạch thực
> hành Topping của bạn cần sửa cả model `orderDetail` ở phía backend.

### 6.4. ⚠️ GET /orders công khai → xem được đơn người khác

`getByUserId` ở mục 6.1 gọi `GET /api/orders/?userId=<id-cua-toi>`. Nhưng route này
**hoàn toàn công khai**, `yotea-be/src/routes/order.js:8-12`:

```js
router.post("/orders", create);
router.get("/orders/:id", read);
router.get("/orders", list);
router.put("/orders/:id/:userId", requireSignin, isAuth, update);
router.delete("/orders/:id", remove);
```

Chỉ mỗi `PUT` được bảo vệ. `GET /orders` không có middleware nào cả. Hệ quả: chỉ cần
**đổi `userId` trong query**, ai cũng xem được đơn của người khác:

```
GET http://localhost:8080/api/orders/?userId=<_id-nguoi-khac>
```

→ nhận về danh sách đơn kèm **tên, số điện thoại, email, địa chỉ** của nạn nhân. Tệ hơn,
`GET /api/orders` (không kèm `userId`) trả về **toàn bộ đơn của mọi khách**, và
`DELETE /api/orders/:id` công khai cho phép **xóa đơn bất kỳ**.

> 🔒 **Hướng vá:** `GET /orders` (xem tất cả) phải là quyền admin; khách chỉ được xem đơn
> **của chính mình** — backend tự lọc theo `req.auth._id` lấy từ token, **không** tin
> `userId` trong query. Xem lỗ hổng #3 ở [Bài 33](33-ra-soat-bao-mat.md).

---

## 7. 🛠️ Tự tay làm — viết endpoint cập nhật hồ sơ AN TOÀN

> Mục tiêu: cuối phần này bạn có một route `PUT /api/users/me` vá được lỗ hổng ở mục 5 —
> lấy danh tính từ token, chỉ nhận đúng 3 trường an toàn, chặn `role`/`active`/`password`.

> 🔒 **Nhắc lại quy tắc của giáo trình:** **không sửa file trong `yotea-be/`**. Hãy chép
> đoạn code dưới đây ra một **dự án nháp riêng** (hoặc một nhánh git thử nghiệm của bạn) để
> chạy thử. Đây là code **bạn tự viết thêm — dự án gốc chưa có**.

### Bước 1 — Viết controller an toàn

Tạo (hoặc thêm) hàm `updateMe` bên cạnh controller user:

```js
// yotea-be/src/controllers/user.js  ← hàm MỚI bạn tự viết thêm (chép ra dự án nháp)
export const updateMe = async (req, res) => {
  // 1. Danh tính LẤY TỪ TOKEN, không lấy từ URL
  const myId = req.auth._id;

  // 2. DANH SÁCH TRẮNG: chỉ nhận đúng 3 trường an toàn
  const allow = ["fullName", "phone", "address"];
  const safeUpdate = {};
  allow.forEach((key) => {
    if (req.body[key] !== undefined) safeUpdate[key] = req.body[key];
  });

  try {
    const user = await User.findByIdAndUpdate(myId, safeUpdate, { new: true })
      .select("-password -__v")
      .exec();

    res.json(user);
  } catch (error) {
    res.status(400).json({ message: "Cập nhật thất bại" });
  }
};
```

Ba điểm khác biệt so với `update` gốc:

| Vấn đề của bản gốc | Bản an toàn |
|---|---|
| `filter = { _id: req.params.id \|\| req.params.myId }` — lấy id từ URL, không kiểm | `myId = req.auth._id` — lấy từ **token đã ký**, không giả mạo được |
| `const update = req.body` — nhận mọi trường (mass assignment) | Vòng lặp `allow` — chỉ giữ `fullName`, `phone`, `address` |
| Trả nguyên document (có `password`) | `.select("-password -__v")` — không bao giờ lộ hash |

### Bước 2 — Đăng ký route (không cần `:myId`, không cần `:userId`)

```js
// yotea-be/src/routes/users.js  ← dòng MỚI bạn tự thêm
import { updateMe } from "../controllers/user";

router.put("/users/me", requireSignin, updateMe);
```

Chỉ cần `requireSignin` — vì `requireSignin` (express-jwt) đã gán `req.auth` sau khi
verify token. Không còn `:userId` trên URL nghĩa là **không còn gì để giả mạo**.

### Bước 3 — Sửa hàm gọi API phía frontend

```js
// yotea-fe/src/api/user.js  ← hàm MỚI bạn tự viết thêm
export const updateMe = (userData, { token } = isAuthenticate()) => {
  return instance.put(`/${DB_NAME}/me`, userData, {
    headers: { Authorization: `Bearer ${token}` },
  });
};
```

Để ý: **không** còn ghép id vào URL nữa. Backend tự biết bạn là ai qua token.

---

## 8. ✅ Kiểm chứng kết quả

### 8.1. Kiểm chứng lỗ hổng gốc (trên máy bạn)

Đăng nhập bằng tài khoản khách thường, lấy `token` và `_id` (trong Postman gọi
`POST /api/signin`, hoặc đọc `localStorage → persist:root → auth`). Gửi:

```
PUT http://localhost:8080/api/users/updateInfo/<_id-cua-ban>/<_id-cua-ban>
Authorization: Bearer <token-cua-ban>
Body: { "role": 1 }
```

Response trả về document user với `"role": 1`. Đăng nhập lại trên web → bạn thấy menu
admin. **Đây là bằng chứng lỗ hổng.**

### 8.2. Kiểm chứng bản vá

Trỏ frontend (hoặc Postman) sang endpoint mới `PUT /api/users/me`, cố tình gửi kèm `role`:

```
PUT http://localhost:8080/api/users/me
Authorization: Bearer <token-cua-ban>
Body:
{
  "fullName": "Nguyễn Văn An",
  "role": 1,
  "password": "hack"
}
```

Kết quả **phải là**: trường `fullName` được cập nhật, còn `role` giữ nguyên `0` và
`password` không đổi (vì cả hai không nằm trong danh sách trắng nên bị bỏ qua). Response
cũng **không** chứa trường `password`. Nếu đúng như vậy, bạn đã vá thành công.

```json
{
  "_id": "6650...",
  "email": "an@gmail.com",
  "fullName": "Nguyễn Văn An",
  "phone": "0912345678",
  "role": 0,
  "active": 1
}
```

### 8.3. Kiểm chứng các trang tài khoản

- Vào `/my-account/` → form đã điền sẵn thông tin của bạn (nhờ `reset({...user})`).
- Đổi họ tên → bấm Cập nhật → header hiện tên mới (state Redux đã đổi).
- Vào `/my-account/update-password` → gõ sai mật khẩu cũ, ô "Mật khẩu cũ" phải báo "Mật
  khẩu hiện tại không chính xác" (nhờ `.test()` gọi `checkPassword`).
- Vào `/my-account/cart` → thấy danh sách đơn của bạn, bấm **View** → sang trang chi tiết
  có đủ sản phẩm (nhờ `_expand=productId`).

---

## 9. 🐞 Lỗi thường gặp

| Thông báo / hiện tượng | Nguyên nhân | Cách xử lý |
|---|---|---|
| Vào `/my-account` bị đá về `/login` | Chưa đăng nhập (bị `PrivateRouter` chặn) | Đăng nhập trước |
| `Cannot read properties of undefined (reading 'avatar')` ở layout | `selectAuth` trả `{}` khi chưa login, `user` là `undefined` | Đảm bảo route được bọc `PrivateRouter`; không mở trực tiếp component |
| Đổi mật khẩu xong, gõ lại mật khẩu **cũ** vẫn đăng nhập được | Hook `pre("findOneAndUpdate")` chỉ băm khi `this._update.password` tồn tại — nếu bạn gửi sai tên trường thì password không đổi | Kiểm tra body gửi lên đúng khóa `password` |
| Form cập nhật báo "thành công" nhưng dữ liệu không đổi | `dispatch(updateMyAccount())` không `await`, toast chạy trước; hoặc API 400 bị nuốt lỗi | Dùng `.unwrap()` và kiểm tra Network tab |
| `MyCartDetailPage` nổ `Cannot read properties of ... (reading 'image')` | Đơn cũ có `productId` trỏ tới sản phẩm đã bị xóa → `_expand` trả `null` | Thêm optional chaining `item.productId?.image` |
| Gọi `PUT /users/me` báo 401 | Token sai/hết hạn (JWT hạn 3h) | Đăng nhập lại lấy token mới |

---

## 10. 📝 Bài tập

**Bài 1.** Trong `authSlice.js`, thunk `updateMyAccount` gọi lại `get(dataAuth._id)` sau
khi cập nhật, rồi loại `password` trước khi lưu vào state. Vì sao **không** dùng luôn
`dataAuth` (dữ liệu client vừa gõ) mà phải tải lại từ server?

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Vì `dataAuth` là **những gì client gửi lên**, chưa chắc trùng với những gì server thật sự
lưu. Ví dụ: server có thể áp default, bỏ qua trường không hợp lệ, hoặc (trong bản vá của
bạn) **lọc bỏ các trường không nằm trong danh sách trắng**. Tải lại bằng `get()` đảm bảo
Redux phản ánh **đúng trạng thái thật của database** — đúng tinh thần "không tin dữ liệu
client". Việc loại `password` là để chuỗi băm mật khẩu **không bao giờ** lọt vào Redux/
localStorage.

</details>

**Bài 2.** Trang `MyCartPage` chuyển `status` số sang chữ bằng chuỗi tam phân lồng nhau,
và ta thấy nhãn không nhất quán giữa 3 nơi. Hãy viết một hàm dùng chung để mọi nơi tra cùng
một nguồn.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Code bạn tự viết thêm (dự án chưa có):

```js
// yotea-fe/src/utils/orderStatus.js  ← file MỚI bạn tự tạo
export const ORDER_STATUS = {
  0: "Chờ xác nhận",
  1: "Đã xác nhận",
  2: "Đang giao hàng",
  3: "Đã giao hàng",
  4: "Đã hủy",
};

export const getStatusLabel = (status) => ORDER_STATUS[status] ?? "Không rõ";
```

Rồi ở `MyCartPage`, `MyCartDetailPage` và ô `<select>` đều dùng `getStatusLabel(item.status)`
hoặc lặp `Object.entries(ORDER_STATUS)`. Một nguồn duy nhất → sửa một chỗ, đúng mọi nơi,
hết cảnh "Đã giao hoàng".

</details>

**Bài 3.** (mạch Topping) Ở [Bài 28](28-thanh-toan.md) bạn đã nhúng topping vào giỏ và
đơn hàng. Hãy làm cho trang **chi tiết đơn** (`MyCartDetailPage`) hiển thị topping của
từng dòng. Cần sửa những gì?

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Ba việc, theo thứ tự từ backend ra frontend:

1. **Model `orderDetail` (backend):** thêm `toppingId` (`ObjectId`, `ref: "Topping"`) và
   `toppingPrice` (Number). Không có bước này thì Mongoose vẫn **âm thầm loại** dữ liệu
   topping như đã phân tích ở mục 6.3.
2. **API `getOrderById`:** đổi query thành `?orderId=...&_expand=productId&_expand=toppingId`
   để "nở" cả topping (lưu ý bộ lọc của dự án nhận nhiều `_expand`; nếu không, populate ở
   controller).
3. **JSX (frontend):** trong `orderDetail?.map(...)`, thêm dòng hiển thị topping, ví dụ:

```jsx
{item.toppingId && (
  <p>Topping: {item.toppingId.name} (+{formatCurrency(item.toppingPrice)})</p>
)}
```

Và cộng `toppingPrice` vào cột "Thành tiền":
`(item.productPrice + (item.toppingPrice || 0)) * item.quantity`.

Đây là code bạn tự viết thêm — dự án gốc chưa có topping trong đơn.

</details>

**Bài 4.** Bản vá `updateMe` ở mục 7 lấy id từ `req.auth._id`. Nhưng nếu ai đó tự **ký một
token giả** với `_id` của người khác thì sao? Bản vá còn an toàn không?

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Token là **JWT có chữ ký**. Muốn ký được token hợp lệ, kẻ xấu phải biết **khóa bí mật**.
Với dự án này, khóa là chuỗi hardcode `"TuongVy"` — mà nó lại lộ trong source (dự án còn
commit cả `node_modules`). Vậy **về lý thuyết** kẻ xấu biết khóa thì ký được token giả với
`_id` bất kỳ → vượt qua cả bản vá.

Kết luận: bản vá `updateMe` chặn được lỗ hổng mass assignment/IDOR, nhưng **gốc rễ vẫn là
khóa bí mật yếu và lộ**. Phải kết hợp: đưa khóa vào biến môi trường `process.env.JWT_SECRET`
(chuỗi ngẫu nhiên đủ dài) và **không** commit nó. Đây là nội dung [Bài 34](34-refactor-du-an.md).
Bài học: bảo mật là **nhiều lớp**, vá một chỗ chưa đủ.

</details>

---

## 📌 Tóm tắt

- Khu vực `/my-account/*` là **layout lồng nhau**: `MyAccountLayout` (sidebar + `<Outlet/>`)
  bọc 4 trang con, cả cụm được bảo vệ bởi `<PrivateRouter page="user">`.
- `UpdateInfoPage` nạp dữ liệu bằng `reset({...user})`, cập nhật qua thunk `updateMyAccount`
  theo mẫu **"cập nhật → tải lại từ server → loại `password`"** trong `authSlice`.
- `wardsCode`/`districtCode`/`provinceCode` là **trường chết** — model có, không UI nào ghi,
  luôn giữ default `0`.
- Đổi mật khẩu: `.test()` bất đồng bộ gọi `POST /api/checkPassword` xác minh mật khẩu cũ;
  mật khẩu mới được **băm lại tự động** ở hook `pre("findOneAndUpdate")` của Mongoose.
- **Lỗ hổng nghiêm trọng nhất:** `PUT /users/updateInfo/:myId/:userId` thiếu `isAdmin`,
  `isAuth` chỉ kiểm `:userId` chứ không kiểm `:myId`, controller nhét thẳng `req.body` →
  khách thường gửi `{"role":1}` là thành admin, hoặc chiếm hồ sơ người khác.
- `GET /api/orders` công khai → đổi `userId` là xem được đơn (kèm PII) của người khác.
- Cách vá gốc: **lấy danh tính từ token**, **danh sách trắng trường được sửa**, luôn
  `.select("-password")`.

**Từ khoá tra cứu thêm:** `nested routes Outlet`, `react-hook-form reset defaultValues`,
`yup async test`, `mongoose pre findOneAndUpdate hook`, `mass assignment`, `broken access control IDOR`

➡️ **Bài tiếp theo:** [30 — Bình luận, đánh giá sao và yêu thích sản phẩm](30-binh-luan-danh-gia-yeu-thich.md) — cùng một "gốc bệnh" thiếu kiểm chủ sở hữu lại xuất hiện, lần này với bình luận và đánh giá.
