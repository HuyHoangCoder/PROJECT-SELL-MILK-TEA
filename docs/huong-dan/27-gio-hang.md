# Bài 27 — Giỏ hàng

> **Phần 5 · Các chức năng phía khách hàng** — Thời lượng ước tính: **~75 phút**
> ⬅️ Bài trước: [26 — Chi tiết sản phẩm: lượt xem, đá/đường, sản phẩm liên quan](26-chi-tiet-san-pham.md) · Bài sau: [28 — Thanh toán: Order + OrderDetail, react-hook-form + yup](28-thanh-toan.md) ➡️
> 🏠 [Mục lục](README.md)

---

## 🎯 Sau bài này bạn sẽ

- Hiểu được điều quan trọng nhất của bài: **giỏ hàng Yotea sống hoàn toàn ở trình duyệt**, không có một dòng nào trong database.
- Đọc vanh vách toàn bộ `cartSlice.js` — bốn reducer `addCart`, `removeItemCart`, `updateQuantity`, `finishOrder`.
- Chỉ ra được một lỗi thiết kế nghiêm trọng: giỏ hàng gộp món theo một khoá nhưng lại xoá/sửa theo một khoá **khác**.
- Mổ xẻ `CartPage.js`: bảng giỏ, ô tăng/giảm số lượng, tính tạm tính và tổng, nút cập nhật/xoá có xác nhận SweetAlert2.
- Hiểu mẹo định dạng tiền Việt bằng `toLocaleString("it-IT", ...)`.
- Nhận ra vì sao "giá do client giữ" là một lỗ hổng bảo mật thật (nối [Bài 33](33-ra-soat-bao-mat.md)).
- Tự tay: vá lỗi gộp topping, thêm nút "Xoá toàn bộ giỏ", và làm badge đếm tổng số món trên header.

## 📋 Cần chuẩn bị

- Đã hoàn thành [Bài 26 — Chi tiết sản phẩm](26-chi-tiet-san-pham.md) — nơi phát sinh dữ liệu ném vào giỏ.
- Nắm chắc [Bài 19 — Redux Toolkit](19-redux-toolkit-co-ban.md) (slice, reducer, action) và [Bài 21 — redux-persist](21-redux-persist.md).
- Hiểu các phương thức mảng `find`, `filter`, `map`, `reduce` ([Bài 03](03-kien-thuc-nen.md)).
- Dự án chạy được cả backend (8080) lẫn frontend (3000), đã có vài sản phẩm để bỏ vào giỏ.

---

## 1. Điều phải nhớ trước tiên: giỏ hàng KHÔNG nằm trong database

Đây là điều dễ hiểu sai nhất của cả bài, nên ta nói thẳng ngay từ đầu.

Với hầu hết các web bán hàng lớn, khi bạn bấm "Thêm vào giỏ", một dòng dữ liệu được ghi
xuống **server** (bảng `cart` trong database), để bạn đăng nhập từ máy khác vẫn thấy giỏ.

**Yotea KHÔNG làm vậy.** Giỏ hàng Yotea sống **100% trong trình duyệt của bạn**:

- Không có `models/cart.js` trong backend.
- Không có `routes/cart.js`, không có `controllers/cart.js`.
- Không có API nào tên `/api/cart`.

Toàn bộ giỏ hàng chỉ là **một mảng JavaScript** nằm trong Redux store, và được
redux-persist sao chép xuống `localStorage` để F5 (tải lại trang) vẫn còn.

```
Bạn bấm "Thêm vào giỏ" ở ProductDetailPage
        │  dispatch(addCart(cartData))
        ▼
   ┌──────────────┐
   │  cartSlice   │  reducer addCart cập nhật state.cart  (RAM, mất khi tắt tab)
   └──────┬───────┘
          │  redux-persist tự động ghi lại
          ▼
   localStorage["persist:root"]   ← đĩa cứng trình duyệt, còn mãi
          │
          │  F5 / mở lại tab
          ▼
   redux-persist đọc ngược lên  →  state.cart được khôi phục  →  giỏ vẫn còn
```

> 💡 **Nhắc lại từ [Bài 21](21-redux-persist.md):** trong `redux/store.js`,
> `persistConfig.whitelist = ["auth", "cart"]` — chỉ đúng hai reducer này được lưu
> xuống `localStorage`. Nhờ vậy giỏ hàng và phiên đăng nhập sống sót qua F5, còn các
> state khác (wishlist, ...) thì không.

**Hệ quả cần ghi nhớ ngay** (ta sẽ đào sâu ở mục 6):

- Vì giỏ ở client nên **mọi giá tiền trong giỏ cũng do client giữ**. Người dùng mở
  DevTools sửa `localStorage` là đổi được giá — và backend hiện **không kiểm lại**.
- Mỗi trình duyệt / mỗi máy có một giỏ riêng. Đăng nhập ở máy khác sẽ **không** thấy giỏ cũ.

---

## 2. Trái tim của giỏ hàng: `cartSlice.js`

Toàn bộ logic giỏ hàng gói gọn trong đúng một file 49 dòng. Ta đọc **nguyên văn**, rồi mổ từng reducer.

`yotea-fe/src/redux/cartSlice.js:1-49`

```js
import { createSlice } from "@reduxjs/toolkit";

const initialState = {
  cart: [],
};

const cartSlice = createSlice({
  name: "cart",
  initialState,
  reducers: {
    addCart({ cart }, { payload: newProduct }) {
      const exitsProduct = cart.find(
        (item) =>
          item.productId === newProduct.productId &&
          item.ice === newProduct.ice &&
          item.sugar === newProduct.sugar
      );

      if (!exitsProduct) {
        cart.push(newProduct);
      } else {
        exitsProduct.quantity += +newProduct.quantity;
      }
    },
    removeItemCart(state, { payload }) {
      state.cart = state.cart.filter((item) => item.id !== payload);
    },
    updateQuantity(state, { payload: listQuantity }) {
      listQuantity.forEach((cartItem) => {
        if (!cartItem.quantity) {
          state.cart = state.cart.filter((item) => item.id !== cartItem.id);
        } else {
          const currentProduct = state.cart.find(
            (item) => item.id === cartItem.id
          );
          currentProduct.quantity = cartItem.quantity;
        }
      });
    },
    finishOrder(state) {
      state.cart = [];
    },
  },
});

export const selectCart = (state) => state.cart.cart;
export const { addCart, removeItemCart, updateQuantity, finishOrder } =
  cartSlice.actions;
export default cartSlice.reducer;
```

> 📖 **Thuật ngữ:** *reducer* — một hàm nhận `state` hiện tại + `action` rồi quyết định
> state mới. Trong Redux Toolkit, nhờ thư viện **Immer** ở bên dưới, bạn được phép viết
> kiểu "sửa trực tiếp" như `cart.push(...)` mà không phải tự tạo mảng mới. Trông như
> mutate nhưng Immer âm thầm tạo bản sao bất biến giúp bạn.

### 2.1. `addCart` — thêm món, gộp nếu trùng

`yotea-fe/src/redux/cartSlice.js:11-24`

```js
addCart({ cart }, { payload: newProduct }) {
  const exitsProduct = cart.find(
    (item) =>
      item.productId === newProduct.productId &&
      item.ice === newProduct.ice &&
      item.sugar === newProduct.sugar
  );

  if (!exitsProduct) {
    cart.push(newProduct);
  } else {
    exitsProduct.quantity += +newProduct.quantity;
  }
},
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 11 | `addCart({ cart }, { payload: newProduct })` | Tham số 1 là `state`, bóc luôn `cart` ra; tham số 2 là `action`, đổi tên `payload` thành `newProduct` |
| 12-17 | `cart.find(...)` | Tìm trong giỏ xem **đã có món giống hệt chưa** — giống theo bộ 3: cùng `productId`, cùng `ice`, cùng `sugar` |
| 19-20 | `if (!exitsProduct) cart.push(newProduct)` | Chưa có → đẩy nguyên món mới vào giỏ |
| 21-22 | `exitsProduct.quantity += +newProduct.quantity` | Đã có → chỉ **cộng dồn số lượng** vào món cũ, không tạo dòng mới |

Vì sao gộp theo bộ 3 `productId + ice + sugar`? Vì với Yotea, "Trà sữa đá 50% đường 30%"
và "Trà sữa đá 100% đường 0%" là **hai món khác nhau** dù cùng một sản phẩm — không được
gộp chung. Ý tưởng này đúng.

> 💡 **Chú ý dấu `+` trước `newProduct.quantity`** (dòng 22): đó là toán tử **ép về số**.
> Nếu `quantity` lỡ là chuỗi `"2"`, `"2" + 1` sẽ ra `"21"` (nối chuỗi); còn `+"2" + 1`
> mới ra `3`. Đây là mẹo phòng thủ nho nhỏ.

### 2.2. `removeItemCart` — xoá một dòng khỏi giỏ

`yotea-fe/src/redux/cartSlice.js:25-27`

```js
removeItemCart(state, { payload }) {
  state.cart = state.cart.filter((item) => item.id !== payload);
},
```

Nhận vào một `payload` là **`item.id`** (một chuỗi uuid, xem mục dưới), rồi `filter` giữ
lại mọi món **trừ** món có `id` trùng. Kết quả là mảng ngắn đi một phần tử.

### 2.3. `updateQuantity` — cập nhật số lượng, số 0 thì xoá luôn

`yotea-fe/src/redux/cartSlice.js:28-39`

```js
updateQuantity(state, { payload: listQuantity }) {
  listQuantity.forEach((cartItem) => {
    if (!cartItem.quantity) {
      state.cart = state.cart.filter((item) => item.id !== cartItem.id);
    } else {
      const currentProduct = state.cart.find(
        (item) => item.id === cartItem.id
      );
      currentProduct.quantity = cartItem.quantity;
    }
  });
},
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 28 | `updateQuantity(state, { payload: listQuantity })` | `payload` là **một mảng** `[{ id, quantity }, ...]` — cả giỏ một lượt |
| 29 | `listQuantity.forEach((cartItem) => {` | Duyệt từng phần tử `{ id, quantity }` |
| 30-31 | `if (!cartItem.quantity) ... filter(...)` | Số lượng **falsy** (0, undefined, "") → **xoá thẳng** món đó khỏi giỏ |
| 33-36 | `find` theo `item.id` rồi gán `currentProduct.quantity = cartItem.quantity` | Ngược lại → tìm đúng món và cập nhật số lượng mới |

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** dòng 30 dùng `if (!cartItem.quantity)` để hiểu
> ngầm "số lượng 0 tức là muốn xoá". Nhưng số lượng 0 **cũng có thể là do người dùng lỡ
> xoá trắng ô input**. Kết quả: gõ nhầm một cái là món biến mất **không kịp hỏi han gì**
> — mất dữ liệu thầm lặng. Cách đúng: chặn số ≤ 0 ngay ở tầng nhập liệu, và việc xoá
> phải là một hành động riêng, có xác nhận (như nút thùng rác), không suy ra từ con số 0.

### 2.4. `finishOrder` — dọn sạch giỏ sau khi đặt hàng

`yotea-fe/src/redux/cartSlice.js:40-42`

```js
finishOrder(state) {
  state.cart = [];
},
```

Đặt lại giỏ về mảng rỗng. Reducer này được [Bài 28 — Thanh toán](28-thanh-toan.md) gọi
(`dispatch(finishOrder())`) ngay sau khi tạo đơn thành công. redux-persist thấy state
đổi sẽ ghi lại `localStorage`, nên giỏ rỗng cũng được lưu — F5 vẫn rỗng.

### 2.5. Selector và export

`yotea-fe/src/redux/cartSlice.js:46-49`

```js
export const selectCart = (state) => state.cart.cart;
export const { addCart, removeItemCart, updateQuantity, finishOrder } =
  cartSlice.actions;
export default cartSlice.reducer;
```

- `selectCart` — hàm để component lấy mảng giỏ: `const cart = useSelector(selectCart)`.
  Chú ý `state.cart.cart` (hai chữ `cart`): chữ đầu là **tên slice**, chữ sau là **trường
  `cart` bên trong `initialState`**.
- Dòng 47-48 xuất bốn action creator để nơi khác `dispatch`.

---

## 3. ⚠️ Lỗi thiết kế nghiêm trọng: gộp một khoá, xoá/sửa một khoá khác

Đây là **trọng tâm bài học**. Hãy để ý thật kỹ.

Món trong giỏ được sinh ra ở `ProductDetailPage.js` (Bài 26), mỗi dòng có cấu trúc đại loại:

```js
// dữ liệu một dòng giỏ hàng — trích ý từ ProductDetailPage.js
const cartData = {
  id: uuidv4(),          // ← id CỤC BỘ của dòng giỏ, sinh ngẫu nhiên mỗi lần thêm
  productId: product._id, // ← id THẬT của sản phẩm trong database
  productName: product.name,
  productPrice: product.price,
  productImage: product.image,
  productSlug: product.slug,
  quantity,
  ice: +ice,
  sugar: +sugar,
};
```

Vậy mỗi món có **hai loại khoá**:

| Khoá | Nguồn | Đặc tính |
|---|---|---|
| `id` | `uuidv4()` — sinh **mỗi lần** bấm "Thêm vào giỏ" | Luôn khác nhau, kể cả cùng sản phẩm |
| `productId` | `product._id` từ database | Giống nhau nếu cùng sản phẩm |

Bây giờ nhìn lại `cartSlice`:

- `addCart` **gộp trùng theo** `productId + ice + sugar` (dòng 12-17).
- `removeItemCart` và `updateQuantity` **lọc/tìm theo** `item.id` (dòng 26, 31, 34).

**Hai bên dùng hai khoá khác nhau.** Ta lần theo một kịch bản để thấy hậu quả:

1. Bạn thêm "Trà sữa, đá 50%, đường 50%", số lượng 1.
   → Giỏ có 1 dòng: `{ id: "aaa", productId: "P1", ice: 50, sugar: 50, quantity: 1 }`.
2. Bạn quay lại thêm **đúng món đó** (cùng sản phẩm, cùng đá, cùng đường) lần nữa.
   → `addCart` thấy trùng bộ 3 → **cộng dồn**, giỏ vẫn 1 dòng, `quantity` thành 2.
   → Nhưng dòng này vẫn giữ `id` cũ là `"aaa"`. **uuid mới không bao giờ được ghi vào.**

Vấn đề nảy sinh ở chỗ: **món thứ hai bạn định thêm mang một `id` uuid mới** (giả sử
`"bbb"`), nhưng vì bị gộp nên `"bbb"` bị vứt đi. Trong giỏ chỉ tồn tại `id = "aaa"`.

Giờ nếu ở đâu đó (một phiên bản UI khác, hoặc chính bug) code cầm nhầm `"bbb"` để gọi
`removeItemCart("bbb")` → `filter` không tìm thấy `id` nào bằng `"bbb"` → **không xoá được
gì cả**. Ngược lại, nếu hai dòng lẽ ra khác nhau nhưng lỡ trùng `id` (uuid tuy hiếm khi
trùng nhưng logic không hề đảm bảo tính duy nhất sau khi gộp), một thao tác xoá có thể
"quét trúng" nhiều dòng.

> ⚠️ **Bản chất cái sai:** một slice mà **khoá định danh không nhất quán**. `addCart`
> coi "trùng" nghĩa là cùng `productId + ice + sugar`; `removeItemCart`/`updateQuantity`
> coi "một dòng" là cùng `id`. Khi hệ thống lớn lên, hai định nghĩa lệch nhau chắc chắn
> sinh bug: xoá không trúng, cập nhật nhầm dòng, hoặc số lượng nhảy sai.
>
> **Cách sửa đúng:** chọn **một** khoá định danh duy nhất cho cả slice. Đơn giản nhất là
> để `addCart` cũng thao tác theo `id`: khi gộp, giữ nguyên `id` của dòng đã có (như hiện
> tại) — điều này thật ra đang đúng; cái sai là **định nghĩa "trùng" của `addCart` không
> khớp với dữ liệu định danh mà UI dùng để xoá/sửa**. Giải pháp gọn: sinh `id` **tất định**
> từ chính bộ khoá gộp, ví dụ `id = \`${productId}-${ice}-${sugar}\`` (và có topping thì
> thêm topping vào — xem mục 5). Như vậy "trùng để gộp" và "trùng để xoá/sửa" trở thành
> **cùng một khái niệm**, hết mâu thuẫn.

### Và còn một lỗ nữa: gộp KHÔNG xét topping

Ở [Bài 26](26-chi-tiet-san-pham.md) bạn đã tự nhúng **topping** (trân châu, thạch,
pudding) vào món. Nhưng nhìn lại điều kiện gộp của `addCart`:

```js
item.productId === newProduct.productId &&
item.ice === newProduct.ice &&
item.sugar === newProduct.sugar
```

**Không có `toppingId`.** Nghĩa là "Trà sữa + trân châu" và "Trà sữa + thạch" (cùng đá,
cùng đường) sẽ bị `addCart` coi là **cùng một món** và gộp lại — trong khi giá và nội
dung hai món hoàn toàn khác. Đây là lỗi gộp sai, ta sẽ vá ngay ở phần "Tự tay làm".

---

## 4. Mổ `CartPage.js` — trang giỏ hàng

`CartPage` là route `/cart` (`App.js:144-147`), **không** bọc `PrivateRouter` — khách
vãng lai xem giỏ được. Trang khá dài (313 dòng) nên ta chia theo chức năng.

### 4.1. Import và state

`yotea-fe/src/pages/user/cart/CartPage.js:1-21`

```js
import { faLongArrowAltLeft, faTimes } from "@fortawesome/free-solid-svg-icons";
import { FontAwesomeIcon } from "@fortawesome/react-fontawesome";
import { useEffect, useState } from "react";
import { Link } from "react-router-dom";
import { formatCurrency, updateTitle } from "../../../utils";
import Swal from "sweetalert2";
import CartNav from "../../../components/user/CartNav";
import { toast } from "react-toastify";
import { useDispatch, useSelector } from "react-redux";
import {
  removeItemCart,
  selectCart,
  updateQuantity,
} from "../../../redux/cartSlice";

const CartPage = () => {
  const dispatch = useDispatch();
  const cart = useSelector(selectCart);
  const [cartQnt, setCartQnt] = useState([]);
  const [disableBtnUpdate, setDisableBtnUpdate] = useState(true);
  const [totalPrice, setTotalPrice] = useState(0);
```

| Biến | Vai trò |
|---|---|
| `cart` | Mảng giỏ **thật** lấy từ Redux qua `selectCart` |
| `cartQnt` | **Bản sao cục bộ** `[{ id, quantity }]` — số lượng người dùng đang chỉnh, **chưa** lưu vào Redux |
| `disableBtnUpdate` | Cờ khoá nút "Cập nhật giỏ hàng"; mặc định `true` (chưa chỉnh gì thì không cho bấm) |
| `totalPrice` | Tổng tiền tạm tính |

> 💡 **Vì sao lại có hai nguồn số lượng (`cart` và `cartQnt`)?** Ý đồ của tác giả: cho
> người dùng bấm +/- thoải mái ở `cartQnt` (local), chỉ khi bấm "Cập nhật giỏ hàng" mới
> ghi một lần xuống Redux. Nhược điểm: cột "Tạm tính" của từng dòng đọc số lượng từ
> `cart` (Redux), nên khi bạn tăng số lượng, **tiền chưa đổi** cho đến khi bấm cập nhật —
> gây bối rối. Đây là một anti-pattern "hai nguồn sự thật" (two sources of truth).

### 4.2. Tính tổng tiền và đồng bộ `cartQnt`

`yotea-fe/src/pages/user/cart/CartPage.js:23-45`

```js
useEffect(() => {
  // get id + quantity
  const getCartQuantity = () => {
    const listIdQnt = cart.map((item) => {
      return {
        id: item.id,
        quantity: item.quantity,
      };
    });

    setCartQnt(listIdQnt);
  };
  getCartQuantity();

  const getTotalPrice = () => {
    setTotalPrice(() => {
      return cart.reduce((total, cart) => {
        return total + cart.productPrice * cart.quantity;
      }, 0);
    });
  };
  getTotalPrice();
}, [cart, totalPrice]);
```

- `getCartQuantity` copy `{ id, quantity }` của từng món từ Redux sang `cartQnt` local.
- `getTotalPrice` dùng `reduce` cộng dồn `productPrice × quantity` của mọi món, bắt đầu
  từ `0` — đúng công thức tính tổng đã học ở [Bài 03](03-kien-thuc-nen.md).

**Công thức tổng tiền:** `Σ (productPrice × quantity)` — không phí ship, không giảm giá.
Vì thế bảng tổng kết hiển thị "Tạm tính" và "Tổng" **bằng nhau** (xem 4.5).

> ⚠️ **Chỗ này dự án làm chưa chuẩn (2 lỗi trong một đoạn):**
> 1. **Deps `[cart, totalPrice]`** (dòng 45): đưa chính `totalPrice` — thứ mà effect này
>    `setTotalPrice` ra — vào danh sách phụ thuộc, tạo nguy cơ chạy lại thừa. May là React
>    tự dừng khi giá trị mới bằng cũ (bail-out). Đúng ra `totalPrice` nên là **giá trị dẫn
>    xuất** (`useMemo` từ `cart`), không cần là state, và deps chỉ cần `[cart]`.
> 2. **`reduce((total, cart) => ...)`**: tham số callback đặt tên `cart` **che mất** biến
>    `cart` bên ngoài (shadowing). Chạy đúng nhưng cực dễ đọc nhầm. Nên đổi tên thành `item`.

### 4.3. Chỉnh số lượng (local): nhập tay, tăng, giảm

`yotea-fe/src/pages/user/cart/CartPage.js:51-99`

```js
const handleUpdateQuantity = (cartId, e) => {
  setDisableBtnUpdate(false);

  const qnt = +e.target.value;
  if (isNaN(qnt)) {
    toast.info("Vui lòng nhập số");
  } else {
    setCartQnt((prev) => {
      return prev?.map((item) =>
        item.id === cartId
          ? {
              id: item.id,
              quantity: qnt,
            }
          : item
      );
    });
  }
};

const increaseQnt = (cartId) => {
  setDisableBtnUpdate(false);

  setCartQnt((prev) => {
    return prev?.map((item) =>
      item.id === cartId
        ? {
            id: item.id,
            quantity: item.quantity + 1,
          }
        : item
    );
  });
};

const decreaseQnt = (cartId) => {
  setDisableBtnUpdate(false);

  setCartQnt((prev) => {
    return prev?.map((item) =>
      item.id === cartId
        ? {
            id: item.id,
            quantity: item.quantity - 1 <= 0 ? 1 : item.quantity - 1,
          }
        : item
    );
  });
};
```

Cả ba hàm cùng một khuôn: bật `setDisableBtnUpdate(false)` để mở khoá nút cập nhật, rồi
`map` qua `cartQnt`, **chỉ đổi đúng dòng có `id === cartId`**, các dòng khác giữ nguyên.

- `handleUpdateQuantity`: nhập tay. `+e.target.value` ép chuỗi thành số; `isNaN` thì báo lỗi.
- `increaseQnt`: `quantity + 1`.
- `decreaseQnt`: `quantity - 1 <= 0 ? 1 : quantity - 1` — **kẹp sàn ở 1**, không cho về 0.

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** nút `-` kẹp sàn ở 1 (không cho 0), nhưng ô nhập
> tay `handleUpdateQuantity` **lại cho gõ 0** (vì `isNaN(0) === false`). Khi bấm "Cập nhật",
> `updateQuantity` gặp `quantity = 0` sẽ **xoá thẳng món** (mục 2.3). Hai hành vi mâu
> thuẫn với nhau. Tệ hơn: gõ `-5` cũng lọt (không phải NaN) → giỏ có số lượng âm và
> `formatCurrency` in ra **tiền âm**. Cách đúng: chặn `qnt <= 0` ngay trong
> `handleUpdateQuantity`.

### 4.4. Ghi số lượng xuống Redux và xoá món

`yotea-fe/src/pages/user/cart/CartPage.js:101-124`

```js
const handleUpdateQnt = () => {
  if (!disableBtnUpdate) {
    dispatch(updateQuantity(cartQnt));
    toast.success("Cập nhật thành công");
    setDisableBtnUpdate(true);
  }
};

const handleRemoveCart = (cartId) => {
  Swal.fire({
    title: "Bạn có chắc chắn muốn xóa sản phẩm này không?",
    text: "Bạn không thể hoàn tác sau khi xóa!",
    icon: "warning",
    showCancelButton: true,
    confirmButtonColor: "#3085d6",
    cancelButtonColor: "#d33",
    confirmButtonText: "Yes, delete it!",
  }).then((result) => {
    if (result.isConfirmed) {
      dispatch(removeItemCart(cartId));
      Swal.fire("Thành công!", "Sản phẩm đã bị xóa.", "success");
    }
  });
};
```

- `handleUpdateQnt`: chỉ chạy khi nút **đang mở khoá** (`!disableBtnUpdate`), rồi
  `dispatch(updateQuantity(cartQnt))` — đẩy **cả mảng số lượng** xuống Redux một lần. Đây
  là lúc số lượng mới thực sự vào giỏ (và redux-persist ghi xuống `localStorage`).
- `handleRemoveCart`: dùng **SweetAlert2** (`Swal.fire`) mở hộp xác nhận đẹp. Nếu người
  dùng bấm đồng ý (`result.isConfirmed`) mới `dispatch(removeItemCart(cartId))` rồi báo
  thành công. `cartId` chính là `item.id` (uuid).

> 📖 **Thuật ngữ:** *SweetAlert2* (import `Swal from "sweetalert2"`) — thư viện hộp thoại
> đẹp thay cho `alert()`/`confirm()` mặc định của trình duyệt. `Swal.fire(...)` trả về một
> **Promise**, nên ta nối `.then((result) => ...)` để biết người dùng bấm nút nào.

### 4.5. JSX: bảng giỏ, ô số lượng, tổng tiền, và trạng thái giỏ rỗng

Phần JSX rất dài và nặng class Tailwind. Ta chỉ trích những phần cốt lõi, lược class bằng chú thích.

**Khung tổng và rẽ nhánh giỏ rỗng** (`CartPage.js:126-131`, `:297-308`):

```jsx
return (
  <>
    <CartNav page="list" />         {/* breadcrumb 3 bước, xem mục 5 */}

    <section /* ... grid 12 cột ... */>
      {cart.length ? (
        <>
          {/* ... bảng giỏ + bảng tổng ... */}
        </>
      ) : (
        // giỏ rỗng
        <section className="text-center col-span-12 py-12">
          <p>Chưa có sản phẩm nào trong giỏ hàng</p>
          <Link to="/thuc-don" /* ... */>
            <button /* ... */>
              <FontAwesomeIcon icon={faLongArrowAltLeft} />
              <span> Tiếp tục mua hàng</span>
            </button>
          </Link>
        </section>
      )}
    </section>
  </>
);
```

`{cart.length ? (...) : (...)}` là mẹo hiển thị có điều kiện: mảng rỗng có `length === 0`
(falsy) → hiện khối "giỏ rỗng"; có hàng → hiện bảng.

**Một dòng sản phẩm trong bảng** (`CartPage.js:163-235`, đã lược class):

```jsx
{cart.map((item, index) => (
  <tr key={index}>
    <td>
      {/* nút xoá dấu × */}
      <button type="button" onClick={() => handleRemoveCart(item.id)}>
        <FontAwesomeIcon icon={faTimes} />
      </button>
    </td>
    <td>
      <Link to={`/san-pham/${item.productSlug}`}>
        <img src={item.productImage} alt="" />
      </Link>
    </td>
    <td>
      <Link to={`/san-pham/${item.productSlug}`}>{item.productName}</Link>
      <div className="text-sm">
        <p>Đá: {item.ice}%</p>
        <p>Đường: {item.sugar}%</p>
      </div>
    </td>

    {/* GIÁ một đơn vị */}
    <td>{formatCurrency(item.productPrice)}</td>

    {/* Ô SỐ LƯỢNG: tìm số lượng local tương ứng trong cartQnt */}
    <td>
      {cartQnt?.map((cartItem, cartIndex) => {
        if (cartItem.id === item.id) {
          return (
            <div key={cartIndex}>
              <button type="button" onClick={() => decreaseQnt(item.id)}>-</button>
              <input
                type="text"
                value={cartItem.quantity}
                onChange={(e) => handleUpdateQuantity(item.id, e)}
              />
              <button type="button" onClick={() => increaseQnt(item.id)}>+</button>
            </div>
          );
        }
      })}
    </td>

    {/* THÀNH TIỀN dòng = giá × số lượng (lấy từ Redux) */}
    <td>{formatCurrency(item.productPrice * item.quantity)}</td>
  </tr>
))}
```

Để ý: ô số lượng render bằng cách **duyệt cả `cartQnt`** rồi `if` lấy dòng khớp `id`.

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** với **mỗi** dòng sản phẩm lại `cartQnt.map` qua
> **toàn bộ** danh sách để tìm một phần tử → độ phức tạp **O(n²)**, và những phần tử không
> khớp trả `undefined` (React bỏ qua nhưng bẩn). Đúng ra dùng
> `cartQnt.find((c) => c.id === item.id)` để lấy đúng một phần tử.

**Bảng tổng và nút thanh toán** (`CartPage.js:265-295`, đã lược class):

```jsx
<table>
  <thead><tr><th colSpan={2}>Cộng giỏ hàng</th></tr></thead>
  <tbody>
    <tr>
      <td>Tạm tính</td>
      <td>{formatCurrency(totalPrice)}</td>
    </tr>
    <tr>
      <td>Tổng</td>
      <td>{formatCurrency(totalPrice)}</td>   {/* CÙNG giá trị với Tạm tính */}
    </tr>
  </tbody>
</table>
<Link to="/checkout">
  <button>Tiến hành thanh toán</button>
</Link>
```

Và nút "Cập nhật giỏ hàng" (`CartPage.js:252-262`) có `disabled={disableBtnUpdate}` — khi
chưa chỉnh gì thì mờ đi, chỉnh rồi mới bấm được, `onClick={handleUpdateQnt}`.

---

## 5. `CartNav.js` — thanh 3 bước

`CartNav` là breadcrumb hiển thị người dùng đang ở bước nào trong luồng: **Cart →
Checkout → Complete**. Nó nhận một prop `page` để tô đậm bước hiện tại.

`yotea-fe/src/components/user/CartNav.js:5-47`

```jsx
const CartNav = ({ page }) => {
  return (
    <section className="container max-w-6xl mx-auto px-3 mt-10">
      <ul className="flex justify-center items-center">
        <li className="text-2xl px-2">
          <Link
            to="/cart"
            className={`${
              page === "list" && "text-black"
            } uppercase text-gray-400 transition ease-linear duration-200 hover:text-black`}
          >
            SHOPPING CART
          </Link>
        </li>
        <li className="text-md text-gray-400 px-2 hidden md:block">
          <FontAwesomeIcon icon={faChevronRight} />
        </li>
        <li className="text-2xl px-2">
          <Link
            to="/checkout"
            className={`${
              page === "checkout" && "text-black"
            } uppercase text-gray-400 transition ease-linear duration-200 hover:text-black`}
          >
            Checkout details
          </Link>
        </li>
        <li className="text-md text-gray-400 px-2 hidden md:block">
          <FontAwesomeIcon icon={faChevronRight} />
        </li>
        <li className="text-2xl px-2">
          <span
            className={`${
              page === "thank-you" && "text-black"
            } uppercase text-gray-400`}
          >
            Order Complete
          </span>
        </li>
      </ul>
    </section>
  );
};
```

Mẹo tô đậm: `` className={`${page === "list" && "text-black"} ...`} ``. Khi `page === "list"`
đúng, biểu thức `&&` trả về chuỗi `"text-black"` (chèn class chữ đen); sai thì trả `false`
(chèn `false` vào chuỗi, coi như không có class). Ba trang gọi `CartNav` với ba giá trị:

| Trang | Gọi | Bước được tô đậm |
|---|---|---|
| `CartPage` | `<CartNav page="list" />` | SHOPPING CART |
| `CheckoutPage` | `<CartNav page="checkout" />` | Checkout details |
| `ThankPage` | `<CartNav page="thank-you" />` | Order Complete |

Chú ý bước cuối "Order Complete" là `<span>` (không phải `<Link>`) — không cho bấm nhảy tới.

---

## 6. `formatCurrency` và lỗ hổng "giá do client quyết định"

### 6.1. Mẹo định dạng tiền Việt

Khắp giỏ hàng, tiền hiển thị qua `formatCurrency`:

`yotea-fe/src/utils/index.js:14-15`

```js
export const formatCurrency = (currency) =>
  currency.toLocaleString("it-IT", { style: "currency", currency: "VND" });
```

`toLocaleString` là hàm sẵn có của mọi con số trong JavaScript, định dạng số theo
**vùng miền (locale)**. Điểm thú vị: tác giả chọn locale **`"it-IT"` (tiếng Ý)**, không
phải `"vi-VN"`. Vì sao?

| Locale | `35000` ra |
|---|---|
| `"vi-VN"` | `35.000 ₫` |
| `"it-IT"` | `35.000 VND` (dùng **dấu chấm** ngăn cách hàng nghìn) |
| `"en-US"` | `₫35,000` (dùng **dấu phẩy**, ký hiệu đứng trước) |

Người Việt quen viết `35.000` (chấm ngăn nghìn), và locale Ý cũng dùng dấu chấm ngăn
nghìn giống hệt — nên `"it-IT"` cho ra định dạng nhìn "thuần Việt" hơn cả `"en-US"`.
Đây là một mẹo nhỏ nhưng đắt: chọn locale theo **kiểu ngăn cách số** mình muốn.

> 💡 **Mẹo:** `formatCurrency` nhận **số**. Nếu lỡ truyền vào `undefined`/`null` (ví dụ giá
> chưa tải xong), `undefined.toLocaleString(...)` sẽ nổ TypeError. Trong giỏ hàng thì an
> toàn vì `productPrice` luôn có sẵn từ lúc thêm vào giỏ.

### 6.2. 🔒 Vì sao "giá ở client" là lỗ hổng thật

Nhớ lại mục 1: giá tiền của từng món (`productPrice`) được **chụp lại (snapshot)** vào
giỏ ngay lúc bấm "Thêm vào giỏ", rồi cất trong `localStorage`. Cả `totalPrice` cũng do
`CartPage` tự tính ở client.

Đến bước thanh toán ([Bài 28](28-thanh-toan.md)), frontend gửi thẳng `totalPrice` và
`productPrice` của từng dòng lên backend. Và backend — trong `controllers/order.js` và
`controllers/orderDetail.js` — **lưu y nguyên những con số đó, KHÔNG tra lại giá thật
trong bảng `products`**.

Ghép hai sự thật lại:

```
localStorage["persist:root"]  →  cart  →  [{ productPrice: 35000, ... }]
                                              │
       kẻ xấu mở DevTools, sửa 35000 → 1000  ▼
                                          [{ productPrice: 1000, ... }]
                                              │  bấm "Đặt hàng"
                                              ▼
   POST /api/orders { totalPrice: 1000 }  →  backend LƯU 1000, không kiểm
```

Chỉ cần sửa một con số trong `localStorage` là mua được ly trà sữa **giá 1.000₫**, và hệ
thống vẫn ghi nhận đơn hợp lệ. Đây là lỗ hổng **thiệt hại tài chính trực tiếp**.

> 🔒 **Ghi chú bảo mật:** nguyên tắc vàng — **không bao giờ tin dữ liệu từ client**. Con
> số giá mà client gửi chỉ nên dùng để *hiển thị*. Khi tạo đơn, backend phải nhận
> `productId` + `quantity`, rồi **tự truy vấn giá thật từ database** và tự tính tổng. Ta
> phân tích và vá đầy đủ ở [Bài 33 — Rà soát bảo mật](33-ra-soat-bao-mat.md) (lỗ hổng #5)
> và [Bài 34 — Refactor](34-refactor-du-an.md).

---

## 7. 🛠️ Tự tay làm

> Mục tiêu: cuối phần này bạn đã vá lỗi gộp topping, có nút "Xoá toàn bộ giỏ" xác nhận,
> và một badge đếm **tổng số món** trên biểu tượng giỏ ở header.

> ⚠️ **Nhắc lại quy tắc giáo trình:** toàn bộ code dưới đây là **bạn tự viết thêm**, dự án
> gốc chưa có. Ta vẫn **không sửa** file gốc trong bài này (trừ bài 34) — hãy chép ra một
> nhánh git riêng hoặc tự tay gõ lại để luyện, tuỳ cách bạn học.

### Bước 1 — Sửa `addCart` để xét cả `toppingId`

Mở `yotea-fe/src/redux/cartSlice.js`, sửa điều kiện gộp trong `addCart` để **hai món khác
topping không bị gộp nhầm**. Đồng thời, để chữa luôn lỗi "hai khoá" ở mục 3, ta sinh `id`
**tất định** từ chính bộ khoá gộp — nhờ vậy "trùng để gộp" và "trùng để xoá/sửa" thành
cùng một khái niệm.

```js
// yotea-fe/src/redux/cartSlice.js  ← đoạn bạn tự sửa
addCart({ cart }, { payload: newProduct }) {
  const exitsProduct = cart.find(
    (item) =>
      item.productId === newProduct.productId &&
      item.ice === newProduct.ice &&
      item.sugar === newProduct.sugar &&
      item.toppingId === newProduct.toppingId   // ← THÊM dòng này
  );

  if (!exitsProduct) {
    cart.push(newProduct);
  } else {
    exitsProduct.quantity += +newProduct.quantity;
  }
},
```

Nếu muốn chữa tận gốc "hai khoá", đặt `id` tất định ngay lúc tạo món ở
`ProductDetailPage.js` (thay cho `uuidv4()`):

```js
// yotea-fe/src/pages/user/.../ProductDetailPage.js  ← bạn tự sửa
const cartData = {
  id: `${product._id}-${+ice}-${+sugar}-${toppingId || "none"}`, // id tất định
  productId: product._id,
  // ... các trường khác giữ nguyên ...
  toppingId,           // nhớ đưa toppingId vào món
  ice: +ice,
  sugar: +sugar,
  quantity,
};
```

Bây giờ hai món cùng `productId + ice + sugar + toppingId` sẽ có **cùng `id`**, nên gộp
và xoá/sửa đều dùng chung một khoá — hết mâu thuẫn.

### Bước 2 — Nút "Xoá toàn bộ giỏ" có xác nhận SweetAlert2

Ta cần một action mới trong slice để dọn sạch giỏ. Có thể tái dùng `finishOrder`, nhưng
để rõ nghĩa, thêm hẳn một reducer `clearCart`:

```js
// yotea-fe/src/redux/cartSlice.js  ← reducer MỚI bạn tự thêm
clearCart(state) {
  state.cart = [];
},
```

Nhớ thêm `clearCart` vào dòng export:

```js
export const {
  addCart,
  removeItemCart,
  updateQuantity,
  finishOrder,
  clearCart,          // ← thêm
} = cartSlice.actions;
```

Trong `CartPage.js`, import thêm `clearCart` rồi viết handler và nút. Học đúng khuôn
`handleRemoveCart` sẵn có:

```jsx
// yotea-fe/src/pages/user/cart/CartPage.js  ← handler MỚI bạn tự thêm
const handleClearCart = () => {
  Swal.fire({
    title: "Xoá toàn bộ giỏ hàng?",
    text: "Tất cả sản phẩm trong giỏ sẽ bị xoá và không thể hoàn tác!",
    icon: "warning",
    showCancelButton: true,
    confirmButtonColor: "#d33",
    cancelButtonColor: "#3085d6",
    confirmButtonText: "Xoá hết",
    cancelButtonText: "Huỷ",
  }).then((result) => {
    if (result.isConfirmed) {
      dispatch(clearCart());
      Swal.fire("Đã xoá!", "Giỏ hàng của bạn đang trống.", "success");
    }
  });
};
```

Đặt nút cạnh nút "Cập nhật giỏ hàng" (khoảng `CartPage.js:262`):

```jsx
<li className="ml-2">
  <button
    type="button"
    onClick={handleClearCart}
    className="uppercase bg-red-500 px-3 h-8 font-semibold text-sm text-white"
  >
    Xoá toàn bộ giỏ
  </button>
</li>
```

### Bước 3 — Badge đếm **tổng số món** trên biểu tượng giỏ ở header

Hiện tại header đang hiển thị `cart.length` — tức **số dòng sản phẩm**, không phải tổng
số ly. Ba dòng, mỗi dòng 5 ly, vẫn hiện "3". Ta sửa để nó cộng tổng `quantity`.

Đây là badge hiện tại, `yotea-fe/src/pages/layouts/WebsiteLayout.js:212-222`:

```jsx
<li
  id="header-cart-label"
  className="uppercase text-base pl-4 text-gray-50 font-light opacity-80 transition ease-linear duration-200 hover:text-white hover:opacity-100"
>
  <Link to="/cart" className="relative">
    <label className="absolute w-4 h-4 bg-green-700 text-xs text-center rounded-full -right-3 -top-1">
      {cart.length}
    </label>
    <FontAwesomeIcon icon={faShoppingCart} />
  </Link>
</li>
```

`cart` ở đây đã có sẵn (`const cart = useSelector(selectCart)` ở đầu `WebsiteLayout`). Ta
tính tổng số món bằng `reduce`, rồi thay vào badge:

```jsx
// yotea-fe/src/pages/layouts/WebsiteLayout.js  ← bạn tự thêm, đặt gần đầu component
const totalItems = cart.reduce((sum, item) => sum + item.quantity, 0);
```

```jsx
{/* sửa nội dung badge */}
<label className="absolute w-4 h-4 bg-green-700 text-xs text-center rounded-full -right-3 -top-1">
  {totalItems}
</label>
```

Nhờ Redux + `useSelector`, mỗi lần giỏ đổi (thêm/xoá/cập nhật) badge tự động vẽ lại — bạn
không phải làm gì thêm.

---

## 8. ✅ Kiểm chứng kết quả

Chạy frontend (đứng tại thư mục `yotea-fe`):

```bash
npm start
```

Mở `http://localhost:3000`, rồi kiểm từng việc:

1. **Giỏ sống ở client:** vào một sản phẩm, chọn đá/đường, bấm "Thêm vào giỏ". Vào
   `/cart` → thấy món. Nhấn **F5** → món **vẫn còn** (redux-persist khôi phục từ
   `localStorage`).
2. **Xem tận mắt localStorage:** mở DevTools (F12) → tab **Application** → **Local Storage**
   → `http://localhost:3000` → khoá `persist:root`. Bên trong có chuỗi JSON `cart` chứa
   mảng món, mỗi món có `id`, `productId`, `productPrice`, `ice`, `sugar`, `quantity`.
3. **Gộp trùng:** thêm đúng một sản phẩm với **cùng** đá/đường hai lần → giỏ vẫn **một
   dòng**, số lượng thành 2. Đổi đá hoặc đường rồi thêm → **hai dòng** riêng.
4. **Tổng tiền:** cột "Tạm tính" của từng dòng = giá × số lượng; ô "Tổng" = tổng tất cả
   dòng, và bằng "Tạm tính" tổng (không có phí ship).
5. **Sau các bước Tự tay làm:**
   - Thêm cùng sản phẩm nhưng **khác topping** → giờ ra **hai dòng** (trước khi vá là một).
   - Bấm "Xoá toàn bộ giỏ" → hiện hộp SweetAlert2, xác nhận → giỏ trống, hiện "Chưa có
     sản phẩm nào trong giỏ hàng".
   - Badge header hiển thị **tổng số ly** (ví dụ 2 dòng mỗi dòng 3 ly = **6**), không còn
     là số dòng.

---

## 9. 🐞 Lỗi thường gặp

| Vấn đề | Nguyên nhân | Cách xử lý |
|---|---|---|
| F5 xong giỏ hàng mất sạch | `cart` chưa nằm trong `persistConfig.whitelist`, hoặc chưa bọc `PersistGate` | Kiểm `redux/store.js` có `whitelist: ["auth", "cart"]` ([Bài 21](21-redux-persist.md)) |
| `Swal is not defined` | Quên `import Swal from "sweetalert2"` | Thêm dòng import ở đầu file |
| Xoá món nhưng không mất | Gọi `removeItemCart` với **giá trị không phải `item.id`** (ví dụ nhầm `productId`) | Truyền đúng `item.id` (uuid); xem lỗi hai khoá ở mục 3 |
| Đổi số lượng mà tiền không đổi | Chỉ mới sửa `cartQnt` (local), chưa bấm "Cập nhật giỏ hàng" | Bấm nút cập nhật để `dispatch(updateQuantity)` |
| Gõ 0 vào ô số lượng thì món biến mất | `updateQuantity` coi `quantity` falsy là "xoá" (`cartSlice.js:30`) | Chặn `qnt <= 0` trong `handleUpdateQuantity` |
| Hai món khác topping bị gộp làm một | `addCart` không xét `toppingId` | Làm Bước 1 phần Tự tay làm |
| `Cannot read properties of undefined (toLocaleString)` | Truyền `undefined` vào `formatCurrency` | Đảm bảo `productPrice`/`totalPrice` là số trước khi format |

---

## 10. 📝 Bài tập

**Bài 1.** Không chạy code, hãy trả lời: giỏ đang có `[{ id: "a", quantity: 2 }, { id: "b",
quantity: 3 }]`. Người dùng gõ 0 vào ô của món `"a"` rồi bấm "Cập nhật giỏ hàng". Redux
`dispatch(updateQuantity(cartQnt))` với `cartQnt = [{ id: "a", quantity: 0 }, { id: "b",
quantity: 3 }]`. Sau khi reducer chạy, giỏ còn gì?

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

`updateQuantity` duyệt mảng: với `{ id: "a", quantity: 0 }`, vì `!0 === true` nên nhánh
`if` chạy → `filter` **bỏ** món `"a"`. Với `{ id: "b", quantity: 3 }`, nhánh `else` tìm
món `"b"` và gán `quantity = 3`. Kết quả giỏ còn: `[{ id: "b", quantity: 3 }]` — món `"a"`
**đã bị xoá** mà người dùng không hề bấm nút xoá nào. Đây chính là lỗi "xoá thầm lặng" ở
mục 2.3.

</details>

**Bài 2.** Viết một **selector** mới trong `cartSlice.js` tên `selectCartTotal` trả về
tổng tiền cả giỏ, để `CartPage` và `CheckoutPage` khỏi phải tự tính lại bằng `reduce` ở
mỗi nơi.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

```js
// yotea-fe/src/redux/cartSlice.js  ← selector MỚI bạn tự thêm
export const selectCartTotal = (state) =>
  state.cart.cart.reduce(
    (total, item) => total + item.productPrice * item.quantity,
    0
  );
```

Dùng trong component:

```js
const totalPrice = useSelector(selectCartTotal);
```

Ưu điểm: logic tính tiền chỉ nằm một chỗ, không lặp lại; và `totalPrice` trở thành giá
trị dẫn xuất từ Redux, không cần `useState` + `useEffect` như `CartPage` hiện tại. (Muốn
tối ưu tránh tính lại thừa, tra thêm `reselect` / `createSelector`.)

</details>

**Bài 3.** (suy ngẫm) Dự án lưu giỏ trong `localStorage`. Nếu sau này muốn giỏ hàng
**đồng bộ giữa các thiết bị** (đăng nhập máy khác vẫn thấy giỏ cũ), phải thay đổi kiến
trúc thế nào? Nêu các thành phần cần thêm ở backend và cách frontend phải đổi.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Cần biến giỏ thành **dữ liệu server-side**:

- **Backend:** thêm `models/cart.js` (mỗi cart gắn `userId`, chứa mảng item), kèm
  `routes/cart.js` + `controllers/cart.js` với CRUD (`GET /cart` của tôi, `POST/PUT/DELETE`
  item), tất cả bọc middleware `requireSignin` + `isAuth` để chỉ chủ giỏ thao tác được.
  **Giá phải do backend tra từ bảng `products`**, không nhận từ client.
- **Frontend:** thay `cartSlice` thuần local bằng luồng gọi API (RTK Query hoặc
  `createAsyncThunk`): thêm/xoá/sửa đều `dispatch` một thunk gọi backend rồi đồng bộ lại
  state. Khi đăng nhập, nạp giỏ từ server. Có thể vẫn giữ `localStorage` cho **khách vãng
  lai** rồi "trộn" vào giỏ server khi họ đăng nhập (merge cart).

Đây cũng là hướng giải quyết luôn lỗ hổng giá ở mục 6, vì giá không còn do client giữ.

</details>

---

## 📌 Tóm tắt

- Giỏ hàng Yotea **sống hoàn toàn ở client**: một mảng trong Redux, được redux-persist
  lưu xuống `localStorage["persist:root"]`. **Không có bảng `cart` trong database.**
- `cartSlice` có 4 reducer: `addCart` (gộp trùng), `removeItemCart` (xoá theo `id`),
  `updateQuantity` (cập nhật cả mảng, `quantity` 0 thì xoá), `finishOrder` (dọn sạch).
- **Lỗi thiết kế cốt lõi:** `addCart` gộp theo `productId + ice + sugar`, còn
  `removeItemCart`/`updateQuantity` lại theo `item.id` (uuid) — **hai khoá khác nhau**.
  Sửa bằng cách dùng một khoá định danh nhất quán (id tất định từ bộ khoá gộp).
- Điều kiện gộp **không xét topping** → hai món khác topping bị gộp sai; vá bằng cách thêm
  `toppingId` vào điều kiện.
- `CartPage` tính tổng bằng `reduce(giá × số lượng)`; "Tạm tính" và "Tổng" bằng nhau (không
  phí ship). Xoá món có xác nhận SweetAlert2.
- `formatCurrency` dùng `toLocaleString("it-IT", ...)` để có định dạng `35.000 VND` kiểu Việt.
- **Lỗ hổng nghiêm trọng:** giá do client giữ trong `localStorage`, backend không kiểm lại
  → sửa `localStorage` là đổi được giá đơn hàng (nối [Bài 33](33-ra-soat-bao-mat.md)).

**Từ khoá tra cứu thêm:** `redux-persist`, `redux toolkit reducer immer`, `array reduce sum`, `toLocaleString currency`, `sweetalert2 confirm`, `two sources of truth react`, `client-side price tampering`

➡️ **Bài tiếp theo:** [28 — Thanh toán: Order + OrderDetail, react-hook-form + yup](28-thanh-toan.md) — biến giỏ hàng thành đơn hàng thật; và bạn sẽ gặp một cái bẫy `forEach` + `await` khét tiếng.
