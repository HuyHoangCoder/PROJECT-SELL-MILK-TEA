# Bài 19 — Redux Toolkit: slice, action, reducer, selector

> **Phần 3 · Frontend React** — Thời lượng ước tính: **~75 phút**
> ⬅️ Bài trước: [18 — Tầng gọi API với axios](18-tang-api-axios.md) · Bài sau: [20 — `createAsyncThunk` và `extraReducers`](20-async-thunk.md) ➡️
> 🏠 [Mục lục](README.md)

---

## 🎯 Sau bài này bạn sẽ

- Chỉ ra được **prop drilling** là gì và vì sao Yotea sẽ rất khổ nếu không có Redux.
- Đọc hiểu trọn vẹn một slice thật: `cartSlice.js` — 49 dòng, 4 reducer, 1 selector.
- Giải thích được vì sao `cart.push(...)` trong reducer **vẫn đúng** nguyên tắc bất biến (nhờ **Immer**).
- Dùng thành thạo `useSelector` để **đọc** và `useDispatch` để **thay đổi** state.
- Hiểu `combineReducers` ghép 11 slice thành một cây state duy nhất như thế nào.
- **Tự viết** `src/redux/toppingSlice.js`, đăng ký vào `rootReducer` và dùng được trong `ToppingPage`.

## 📋 Cần chuẩn bị

- Đã hoàn thành [Bài 18](18-tang-api-axios.md): bạn đã có `yotea-fe/src/api/topping.js` với đủ `getAll / get / add / update / remove`.
- Đã có `ToppingPage.js` và `ToppingCard.js` từ [Bài 16](16-layout-va-component.md), [Bài 17](17-tailwind-css.md).
- Nắm destructuring, spread/rest, arrow function ([Bài 03](03-kien-thuc-nen.md)).

> **Ở bài trước bạn đã làm** tầng API — nơi *nói chuyện với server*. **Bài này ta làm tiếp**
> tầng state — nơi *cất giữ dữ liệu đã lấy về* để mọi component cùng dùng chung.

---

## 1. Vấn đề Redux sinh ra để giải quyết: prop drilling

Nhìn cái **badge số món trong giỏ** ở góc phải header.

`yotea-fe/src/pages/layouts/WebsiteLayout.js:216-221`

```jsx
                <Link to="/cart" className="relative">
                  <label className="absolute w-4 h-4 bg-green-700 text-xs text-center rounded-full -right-3 -top-1">
                    {cart.length}
                  </label>
                  <FontAwesomeIcon icon={faShoppingCart} />
                </Link>
```

Con số `cart.length` nằm trong `WebsiteLayout` — **đỉnh** cây component. Nhưng nút "Thêm vào
giỏ hàng" lại nằm trong `ProductDetailPage`, tận sâu bên trong `<Outlet />`:

```
WebsiteLayout                     ← badge "cart.length" hiển thị Ở ĐÂY
  └── <Outlet />
        └── ProductDetailPage     ← nút "Thêm vào giỏ hàng" bấm Ở ĐÂY
              └── form đá/đường
```

Không có Redux, bạn buộc phải nâng state lên cha chung rồi **truyền props xuyên qua từng tầng**:

```jsx
// ❌ Cách KHÔNG dùng Redux — prop drilling, đoạn này chỉ để minh hoạ
<WebsiteLayout cart={cart} setCart={setCart}>
  <Outlet context={{ cart, setCart }} />
     <ProductDetailPage cart={cart} setCart={setCart} />
```

> 📖 **Thuật ngữ:** *prop drilling* — "khoan props". Dữ liệu phải xuyên qua hàng loạt
> component trung gian **không hề cần đến nó**, chỉ để tới được component ở đáy.

| Hậu quả | Cụ thể trong Yotea |
|---|---|
| Component trung gian bị "ô nhiễm" | `Outlet`, `ProductContent`, `HomeProducts`… phải nhận `cart` dù chẳng dùng |
| Đổi cấu trúc dữ liệu là sửa cả chục file | Thêm trường vào cart item → sửa mọi chữ ký props |
| Không chia sẻ được giữa các nhánh | `CartPage` và header là **hai nhánh khác nhau** của cây |
| Re-render thừa | Cha đổi state → **mọi** con re-render |

Redux **kéo dữ liệu ra khỏi cây component**, đặt vào một cái kho đứng bên ngoài. Component nào
cần thì tự với tay lấy:

```
            ┌───────────────────────┐
            │      REDUX STORE      │
            │  { cart: {cart: []},  │
            │    auth: {...}, ... } │
            └───────────────────────┘
              ▲                   │
   dispatch(addCart)         useSelector(selectCart)
              │                   ▼
   ProductDetailPage        WebsiteLayout (badge)
```

Hai bên **không hề biết nhau tồn tại**.

---

## 2. Ví von: Redux là một cái kho hàng có nội quy

| Khái niệm Redux | Trong nhà kho | Trong code |
|---|---|---|
| **store** | Cái **kho** — nơi duy nhất chứa hàng | `yotea-fe/src/redux/store.js` |
| **state** | **Hàng** đang nằm trong kho | `{ cart: [...], auth: {...} }` |
| **reducer** | **Thủ kho** — người **DUY NHẤT** được động vào hàng | `addCart`, `logout`… |
| **action** | **Phiếu yêu cầu**: *"làm gì"* + *"với cái gì"* | `{ type: "cart/addCart", payload: {...} }` |
| **dispatch** | Hành động **nộp phiếu** cho thủ kho | `dispatch(addCart(cartData))` |
| **selector** | **Phiếu tra cứu** — chỉ xem, không sửa | `selectCart = (state) => state.cart.cart` |

Nội quy quan trọng nhất: **bạn không được tự tay vào kho bốc hàng.** Muốn gì cũng phải viết
phiếu rồi đưa thủ kho. Nhờ vậy mọi thay đổi đều **có dấu vết**.

### 2.1. Ba nguyên tắc bất di bất dịch

**1. Một nguồn sự thật duy nhất.** Toàn bộ state nằm trong **một** object, trong **một** store.
Không có chuyện giỏ hàng lưu ở 3 nơi rồi lệch nhau.

**2. State chỉ đọc.** Cách duy nhất để thay đổi là dispatch một action.

```js
state.cart.cart.push(sanPhamMoi);   // ❌ SAI
dispatch(addCart(sanPhamMoi));      // ✅ ĐÚNG
```

**3. Thay đổi chỉ qua hàm thuần.** Reducer nhận `(state cũ, action)` → trả về **state mới**.

| Reducer ĐƯỢC phép | Reducer KHÔNG được phép |
|---|---|
| Tính toán từ `state` và `action` | Gọi API, ghi `localStorage` |
| Trả về state mới | Đọc `Date.now()`, `Math.random()` |

Cùng một `(state, action)` thì **luôn** ra cùng kết quả — đó là lý do Redux DevTools "tua đi
tua lại" được lịch sử state.

> 💡 **Vì sao là "Redux Toolkit" chứ không phải Redux thuần?** Redux thuần bắt bạn viết tay
> hằng số action type, action creator, `switch...case`, tự copy object bằng spread — mỗi tính
> năng nhỏ tốn 3 file. **RTK** gói tất cả vào một hàm: `createSlice`.

---

## 3. Soi code thật: `cartSlice.js`

Đây là slice **duy nhất trong 11 slice của Yotea không có `createAsyncThunk`** (giỏ hàng hoàn
toàn ở client). Vì thế nó là bài mẫu hoàn hảo để học `createSlice`.

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

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 1 | `import { createSlice }` | Hàm "3 trong 1": sinh reducer + action type + action creator |
| 3-5 | `initialState = { cart: [] }` | **Hình dạng ban đầu** của state khi app khởi động |
| 8 | `name: "cart"` | **Tiền tố** cho mọi action type của slice này |
| 9 | `initialState,` | Object shorthand — viết đủ là `initialState: initialState` |
| 10 | `reducers: { ... }` | Nơi khai báo reducer **đồng bộ** (chạy xong ngay lập tức) |
| 11-24 | `addCart(...)` | Thêm sản phẩm vào giỏ, có gộp trùng |
| 25-27 | `removeItemCart(...)` | Xoá một dòng khỏi giỏ |
| 28-39 | `updateQuantity(...)` | Cập nhật số lượng **hàng loạt** |
| 40-42 | `finishOrder(state)` | Dọn sạch giỏ sau khi đặt hàng xong |
| 46 | `export const selectCart` | **Selector** — hàm đọc dữ liệu ra khỏi state |
| 47-48 | `= cartSlice.actions` | Lấy 4 **action creator** mà RTK tự sinh |
| 49 | `export default cartSlice.reducer` | Reducer để lát nữa gắn vào `rootReducer` |

### 3.1. RTK tự sinh action creator như thế nào?

Bạn chỉ viết cái tên `addCart`. RTK tự làm phần còn lại:

```
Bạn viết:   reducers: { addCart(state, action) {...} }
RTK sinh:   • action type    → "cart/addCart"   (name + "/" + tên hàm)
            • action creator → cartSlice.actions.addCart
```

Nên khi gọi `dispatch(addCart(cartData))`, thứ thật sự chạy trong store là một object phẳng:

```js
{
  type: "cart/addCart",
  payload: { id: "a3f1-...", productId: "6650...", quantity: 2, ice: 30, sugar: 50, ... }
}
```

Đây chính là **"tấm phiếu yêu cầu"** ở mục 2: `type` là *"làm gì"*, `payload` là *"với cái gì"*.

> 💡 **Mẹo đọc chữ ký reducer:** tham số thứ hai **luôn là cả action object**. Vì gần như chỉ
> cần `payload`, dự án destructuring luôn tại chỗ: `removeItemCart(state, { payload })`. Còn
> `{ payload: listQuantity }` là **bóc `payload` ra và đổi tên** cho dễ đọc.

---

## 4. Immer: vì sao `cart.push(...)` lại hợp lệ?

Dòng 20 viết `cart.push(newProduct);` — mà `push` là hàm **sửa thẳng mảng gốc**. Nguyên tắc số 2
nói *"state chỉ đọc"* cơ mà?

**Redux Toolkit nhúng sẵn thư viện Immer.** Cơ chế:

```
1. Bạn dispatch một action
2. Immer tạo BẢN NHÁP (draft) — một proxy bọc quanh state thật
3. Reducer của bạn "sửa" thoải mái trên BẢN NHÁP
4. Immer ghi lại từng thay đổi, TỰ TẠO state mới bất biến
5. State cũ vẫn còn nguyên vẹn
```

Nói cách khác: bạn **viết như đang mutate**, Immer **dịch thành immutable**.

| Bạn viết trong reducer | Immer thực sự làm |
|---|---|
| `cart.push(newProduct)` | `return [...cartCu, newProduct]` |
| `exitsProduct.quantity += 1` | Nhân bản item đó với `quantity` mới, giữ nguyên item khác |
| `state.cart = []` | Trả về object mới `{ cart: [] }` |

Vì thế trong `cartSlice` **cả hai kiểu viết** cùng tồn tại mà vẫn đúng:

```js
cart.push(newProduct);                                         // kiểu "mutate"  (dòng 20)
state.cart = state.cart.filter((item) => item.id !== payload); // kiểu "gán mới" (dòng 26)
```

Chú ý cách `addCart` bóc tham số ngay tại chữ ký (dòng 11): `addCart({ cart }, ...)`. Biến
`cart` ở đây **không phải bản sao**, nó là **tham chiếu tới mảng draft**, nên `cart.push(...)`
vẫn được Immer theo dõi bình thường.

> ⚠️ **Bẫy Immer số 1:** chỉ áp dụng **bên trong reducer của `createSlice`**. Viết
> `cart.push(...)` trong component React thì sai hoàn toàn — React không re-render vì
> reference mảng không đổi.
>
> ⚠️ **Bẫy Immer số 2:** đừng **vừa mutate vừa `return`**. Làm cả hai sẽ ném lỗi
> `Cannot return a new state and modify the draft`.

---

## 5. Mổ kỹ 4 reducer của `cartSlice`

### 5.1. `addCart` — gộp sản phẩm trùng (dòng 11-24)

```js
      const exitsProduct = cart.find(
        (item) =>
          item.productId === newProduct.productId &&
          item.ice === newProduct.ice &&
          item.sugar === newProduct.sugar
      );
```

Khoá gộp là **bộ ba `(productId, ice, sugar)`**, chứ không chỉ `productId`:

| Tình huống | Kết quả trong giỏ |
|---|---|
| Trà sữa A, đá 50%, đường 50% — thêm 2 lần | **1 dòng**, `quantity` cộng dồn |
| Trà sữa A đá **30%**, rồi trà sữa A đá **70%** | **2 dòng riêng biệt** |

Rất hợp nghiệp vụ: cùng món nhưng khác mức đá/đường thì pha khác nhau, phải tách dòng.
Dòng 22 `exitsProduct.quantity += +newProduct.quantity;` — dấu `+` phía trước là **ép kiểu sang
số**, phòng khi `quantity` là chuỗi `"2"` (nếu không, `1 + "2"` ra `"12"`).

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** biến `exitsProduct` **sai chính tả** (đúng là
> `existsProduct`), lặp 3 lần ở dòng 12, 19, 22. Ngoài ra `addCart` **không kiểm tra
> `quantity <= 0`** và không kiểm tra tồn kho.

### 5.2. `removeItemCart` (dòng 25-27)

`payload` ở đây là **`item.id`**, không phải `productId`. Nơi gọi —
`yotea-fe/src/pages/user/cart/CartPage.js:120`:

```js
        dispatch(removeItemCart(cartId));
```

(nằm bên trong hộp xác nhận SweetAlert2, `CartPage.js:109-124`).

### 5.3. `updateQuantity` (dòng 28-39)

Reducer phức tạp nhất, cập nhật **hàng loạt**. `payload` là **một mảng** `[{ id, quantity }, ...]`
dựng ở `yotea-fe/src/pages/user/cart/CartPage.js:26-33`:

```js
      const listIdQnt = cart.map((item) => {
        return {
          id: item.id,
          quantity: item.quantity,
        };
      });

      setCartQnt(listIdQnt);
```

Luồng: gõ số vào ô input → chỉ đổi `useState` cục bộ `cartQnt` → bấm "Cập nhật giỏ hàng"
(`CartPage.js:101-107`) mới `dispatch(updateQuantity(cartQnt))`. Trong reducer, nhánh dòng 30-31
nói: *"số lượng bằng 0 (hoặc rỗng) thì **xoá luôn dòng đó**."*

> ⚠️ **Chỗ này dự án làm chưa chuẩn — 3 lỗi trong một reducer:**
> 1. **Có thể crash trắng trang.** Dòng 33-36 gọi `find` rồi gán thẳng
>    `currentProduct.quantity = ...`. Nếu `find` không tìm thấy (mở 2 tab, tab A đã xoá món đó)
>    thì `currentProduct` là `undefined` → `TypeError: Cannot set properties of undefined`.
>    Cách đúng: `if (currentProduct) currentProduct.quantity = ...`.
> 2. **Số lượng âm lọt lưới.** `CartPage.js:54-55` chỉ chặn `isNaN`. Gõ `-5` thì
>    `isNaN(-5) === false` → lọt vào reducer; `!(-5)` là `false` → **không bị xoá** mà gán
>    `quantity = -5` → tổng tiền ra **số âm**.
> 3. **Xoá ngầm không hỏi.** Xoá bằng nút ✕ thì có hộp xác nhận, còn gõ `0` vào ô số lượng thì
>    món **biến mất luôn** — hai đường xoá, hai trải nghiệm khác nhau.

### 5.4. `finishOrder` (dòng 40-42)

Reducer đơn giản nhất: **không cần `payload`**. Chỉ được dispatch một nơi duy nhất —
`yotea-fe/src/pages/user/cart/CheckoutPage.js:91`, ngay sau khi tạo xong đơn hàng.

---

## 6. ⚠️ Điểm không nhất quán: `item.id` hay `item.productId`?

| Reducer | Khoá dùng để tìm | Dòng |
|---|---|---|
| `addCart` | `productId` + `ice` + `sugar` | 12-17 |
| `removeItemCart` | **`id`** | 26 |
| `updateQuantity` | **`id`** | 31, 34 |

Ba reducer cùng file dùng **hai loại khoá khác nhau**. Vậy `id` từ đâu ra?

**Tìm `uuid` trong toàn bộ `yotea-fe/src` — chỉ đúng 2 dòng, cùng một file:**

```
yotea-fe/src/pages/user/ProductDetailPage.js:20:import { v4 as uuidv4 } from "uuid";
yotea-fe/src/pages/user/ProductDetailPage.js:51:      id: uuidv4(),
```

`yotea-fe/src/pages/user/ProductDetailPage.js:39-62`

```js
  const onSubmit = async ({ ice, sugar }) => {
    // get data product
    const { data: product } = await get(slug);
    const productData = {
      productSlug: product.slug,
      productId: product._id,
      productName: product.name,
      productPrice: product.price,
      productImage: product.image,
    };

    const cartData = {
      id: uuidv4(),
      ...productData,
      quantity,
      ice: +ice,
      sugar: +sugar,
    };

    dispatch(addCart(cartData));
    toast.success(`Thêm ${product.name} vào giỏ hàng thành công`);
    reset();
    setQuantity(1);
  };
```

Vậy một item trong giỏ có **hai loại id hoàn toàn khác nhau**:

| Trường | Nguồn | Vai trò |
|---|---|---|
| `id` | `uuidv4()` sinh ở **frontend** | **Khoá của DÒNG giỏ hàng**, chỉ sống trong trình duyệt, **không bao giờ gửi lên server** |
| `productId` | `product._id` của MongoDB | **Khoá của SẢN PHẨM**, dùng để gộp trùng và gửi lên `POST /api/orderDetail` |

**Về thiết kế, đây KHÔNG hẳn là bug** — một `productId` có thể xuất hiện ở nhiều dòng (khác
đá/đường), nên cần `id` riêng cho từng dòng. Nhưng hệ quả thực tế rất đáng lo:

1. **uuid bị vứt đi khoảng một nửa số lần.** Mỗi lần bấm "Thêm vào giỏ", dòng 51 sinh uuid mới;
   nếu `addCart` phát hiện trùng thì nó chỉ cộng `quantity` (dòng 22) và **uuid vừa sinh bị bỏ ngay**.
2. **uuid chỉ được sinh ở đúng 1 chỗ trong toàn dự án.** Mai bạn thêm nút "Mua nhanh" ở
   `HomeProducts.js` mà quên `id: uuidv4()` → item vào giỏ với `id === undefined`. Khi đó
   `removeItemCart(undefined)` **xoá sạch mọi dòng có `id === undefined`**, và `updateQuantity`
   ghi đè nhầm dòng.
3. **`id` không bao giờ tới backend.** `CheckoutPage.js` chỉ gửi `productId`, `quantity`, `ice`,
   `sugar`… Người mới rất dễ tưởng `id` là id trong database — không phải.
4. **`id` sống rất lâu**, vì giỏ hàng được redux-persist lưu xuống `localStorage` ([Bài 21](21-redux-persist.md)).

> 💡 **Cách chuẩn hơn:** đặt tên là `lineId` cho rõ nghĩa, và sinh nó **ngay trong `addCart`**…
> nhưng khoan — reducer phải **thuần**, mà `uuidv4()` là ngẫu nhiên! Đúng chỗ để làm là
> `prepare` callback của RTK:
> ```js
> addCart: {
>   reducer(state, { payload }) { /* logic gộp */ },
>   prepare(product) {
>     return { payload: { lineId: uuidv4(), ...product } };
>   },
> }
> ```
> `prepare` chạy **trước** reducer nên được phép "không thuần".

---

## 7. `useSelector` và `useDispatch`

### 7.1. `useSelector` — ĐỌC state

`yotea-fe/src/pages/layouts/WebsiteLayout.js:46-53`

```js
  const categories = useSelector(selectCatesProduct);

  const wishlist = useSelector(selectWishlist);
  const isShowWishlist = useSelector(selectShowWishlist);

  const isLogged = useSelector(selectStatusLoggin);
  const { user } = useSelector(selectAuth);
  const cart = useSelector(selectCart);
```

Cách hoạt động: `useSelector` nhận **hàm selector** (nhận `state` toàn cục, trả về **mảnh** bạn
cần). React-Redux đăng ký theo dõi store; mỗi khi store đổi nó chạy lại selector, và **chỉ
re-render** khi giá trị trả về khác lần trước (so sánh `===`). Nhờ vậy `WebsiteLayout` **không**
re-render khi `state.news` hay `state.contact` đổi.

### 7.2. `useDispatch` — THAY ĐỔI state

`yotea-fe/src/pages/user/cart/CartPage.js:101-107`

```js
  const handleUpdateQnt = () => {
    if (!disableBtnUpdate) {
      dispatch(updateQuantity(cartQnt));
      toast.success("Cập nhật thành công");
      setDisableBtnUpdate(true);
    }
  };
```

Ba bước luôn giống nhau, học thuộc là dùng được cả dự án:

```js
import { useDispatch } from "react-redux";
import { updateQuantity } from "../../../redux/cartSlice";

const dispatch = useDispatch();        // 1. lấy hàm dispatch (CartPage.js:17)
dispatch(updateQuantity(cartQnt));     // 2. gọi action creator → 3. nộp phiếu
```

> ⚠️ **Lỗi kinh điển:** viết `dispatch(updateQuantity)` — thiếu ngoặc gọi hàm. Bạn đang nộp
> *cái máy in phiếu* thay vì *tờ phiếu*. Redux ném `Actions must be plain objects`.

### 7.3. Vì sao phải viết selector thay vì `state.cart.cart` khắp nơi?

```js
const cart = useSelector((state) => state.cart.cart);  // ❌ lặp lại ở 3 file
const cart = useSelector(selectCart);                  // ✅ dự án đang dùng
```

| Lý do | Giải thích |
|---|---|
| **Một chỗ sửa duy nhất** | `selectCart` được import ở 3 file (`WebsiteLayout.js:37`, `CartPage.js:12`, `CheckoutPage.js:14`). Đổi hình dạng state chỉ sửa **1 dòng** trong `cartSlice.js` |
| **Che giấu hình dạng state** | Component không cần biết state lồng mấy tầng. `state.cart.cart` (hai chữ `cart`!) là hình dạng xấu, selector che nó đi |
| **Đặt tên có nghĩa** | `selectAuth` đọc là *"lấy thông tin đăng nhập"*, dễ hơn `state.auth.value` |
| **Chống gõ sai âm thầm** | Gõ nhầm `state.card.cart` nổ lúc chạy mà không ai biết vì sao. Dùng selector thì chỉ có 1 nơi để gõ sai |
| **Dọn đường cho memo hoá** | Sau này cần tính toán nặng, chỉ việc bọc `createSelector` tại đúng chỗ đó |

Selector trong Yotea đều là **một dòng**, đặt cuối file slice.

`yotea-fe/src/redux/authSlice.js:57-60`

```js
export const { signin, logout } = authSlice.actions;
export const selectStatusLoggin = (state) => state.auth.isLogged;
export const selectAuth = (state) => state.auth.value;
export default authSlice.reducer;
```

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** `selectStatusLoggin` **sai chính tả** (thừa chữ "g").
> Nó đã dùng ở 3 file nên giờ sửa là phải sửa cả 3 — bài học nhớ đời về việc **đặt tên public
> API cẩn thận ngay từ đầu**.

---

## 8. `rootReducer` — ghép 11 slice thành một cây state

`yotea-fe/src/redux/rootReducer.js:19-38`

```js
const store = combineReducers({
  cateProduct: cateProductReducer,
  news: newsReducer,
  cateNews: cateNewsReducer,
  contact: contactReducer,
  slider: sliderReducer,
  product: productReducer,
  store: storeReducer,
  user: userReducer,
  wishlist: wishlistReducer,
  auth: authReducer,
  cart: cartReducer,
  [productApi.reducerPath]: productApi.reducer,
  [sliderApi.reducerPath]: sliderApi.reducer,
  [cateProductApi.reducerPath]: cateProductApi.reducer,
  [userApi.reducerPath]: userApi.reducer,
  [newsApi.reducerPath]: newsApi.reducer,
});

export default store;
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 19 | `combineReducers({...})` | Ghép nhiều reducer con thành **một** reducer tổng |
| 20 | `cateProduct: cateProductReducer` | **Key bên trái quyết định state nằm ở đâu** → `state.cateProduct` |
| 30 | `cart: cartReducer` | Vì `cartSlice` có `initialState = { cart: [] }` nên mới ra `state.cart.cart` |
| 31-35 | `[productApi.reducerPath]: ...` | **Computed key** — gắn cache RTK Query vào store ([Bài 22](22-rtk-query.md)) |
| 38 | `export default store` | Xuất reducer tổng cho `store.js` dùng |

Cơ chế rất đơn giản: **mỗi action được gửi tới TẤT CẢ reducer con**. Reducer nào "nhận ra"
action type của mình thì xử lý, còn lại trả về state cũ nguyên vẹn.

### 8.1. State của app trông như thế nào?

| Đường dẫn | Kiểu | Nội dung |
|---|---|---|
| `state.cart.cart` | mảng | Món trong giỏ (`selectCart`) |
| `state.auth.isLogged` / `state.auth.value` | boolean / object | Đã đăng nhập chưa · `{ token, user }` |
| `state.cateProduct.value` | mảng | Danh mục sản phẩm (`selectCatesProduct`) |
| `state.news.value` / `.totalItem` | mảng / số | Tin tức + tổng số |
| `state.cateNews.value` | mảng | Chuyên mục tin |
| `state.contact.value` / `.totalContact` | mảng / số | Liên hệ khách gửi |
| `state.slider.value` | mảng | Banner trang chủ |
| `state.product.value` / `.totalProduct` | mảng / số | **Không ai đọc** — code chết |
| `state.store.value` / `.totalStore` | mảng / số | **Danh sách chi nhánh cửa hàng** |
| `state.user.value` / `.totalUser` | mảng / số | Danh sách tài khoản (admin) |
| `state.wishlist.value` / `.showWishlist` | mảng / boolean | Yêu thích + cờ mở drawer |
| `state.productApi`, `state.sliderApi`… | object | Cache của RTK Query |
| `state._persist` | object | Do redux-persist thêm vào |

> ⚠️ **Ba chữ "store" dễ gây loạn óc:** `store.js` là **file** cấu hình; biến `store` ở
> `rootReducer.js:19` thực chất là **root reducer** (đặt tên sai ngữ nghĩa hoàn toàn); còn
> `state.store` là **danh sách chi nhánh cửa hàng**. Đọc code Yotea luôn phải hỏi "store nào?".

### 8.2. Điểm danh 11 slice của dự án

| # | File slice | Mount key | Một câu tóm tắt |
|---|---|---|---|
| 1 | `authSlice.js` | `auth` | Token + người đang đăng nhập; 2 reducer đồng bộ `signin`/`logout` + 2 thunk sửa hồ sơ. **Quan trọng nhất dự án** |
| 2 | `cartSlice.js` | `cart` | Giỏ hàng client-side, **slice duy nhất không có thunk**; được persist xuống localStorage |
| 3 | `categoryProductSlice.js` | `cateProduct` | CRUD danh mục sản phẩm; ⚠️ `name: "categoryProduct"` **lệch** với mount key |
| 4 | `cateNewsSlice.js` | `cateNews` | CRUD chuyên mục tin — gần như bản sao 1:1 của slice số 3 |
| 5 | `contactSlice.js` | `contact` | Chỉ **đọc + xoá** liên hệ (khách gửi qua form public, admin không thêm được) |
| 6 | `newsSlice.js` | `news` | CRUD tin tức; ⚠️ đặt tên `totalItem` thay vì `totalNews`, lệch quy ước |
| 7 | `productSlice.js` | `product` | ☠️ **100% code chết** — đã bị `productApi` (RTK Query) thay thế, không file nào import |
| 8 | `sliderSlice.js` | `slider` | CRUD banner trang chủ; ⚠️ trùng chức năng với `sliderApi` → hai cache lệch nhau |
| 9 | `storeSlice.js` | `store` | CRUD chi nhánh; ☠️ gần như chết — `StoreList` không được page nào render |
| 10 | `userSlice.js` | `user` | CRUD tài khoản cho admin; ⚠️ sửa user không đồng bộ sang `state.auth` |
| 11 | `wishlistSlice.js` | `wishlist` | Sản phẩm yêu thích + cờ `showWishlist`; 2 reducer đồng bộ + 3 thunk |

Nhận xét quan trọng: **8/11 slice dùng chung một khuôn duy nhất** — `initialState { value: [],
total<X>: 0 }`, thunk `getX/addX/updateX/deleteX`, `reducers: {}` **luôn rỗng**, selector
`selectX`/`selectTotalX`. Quy ước nhất quán này là điểm cộng lớn nhất của dự án, và là lý do
học xong 1 slice thì đọc được cả 11.

### 8.3. ⚠️ Store của dự án dùng API đã bị khai tử

`yotea-fe/src/redux/store.js:30-36`

```js
export const store = createStore(
  persistedReducer,
  applyMiddleware(...middleware)
);
export default persistStore(store);

setupListeners(store.dispatch);
```

Dự án dùng `createStore` + `applyMiddleware` — API **Redux đời cũ, đã deprecated từ Redux v5** —
trong khi đã cài sẵn `@reduxjs/toolkit`. Hậu quả bạn đụng ngay khi debug:

| Mất gì | Vì sao đau |
|---|---|
| **Redux DevTools** | Không xem được cây state, không xem lịch sử action, không "tua ngược". Debug Redux mà thiếu DevTools như sửa xe trong bóng tối |
| `serializableCheck` | Không ai cảnh báo khi lỡ nhét `Date`, `Map`, class instance vào state |
| `immutableCheck` | Lỡ mutate state **ngoài** reducer thì im lặng luôn |
| `actionCreatorCheck` | Không cảnh báo khi `dispatch(actionCreator)` thiếu ngoặc |
| Thunk sẵn có | Phải cài thêm `redux-thunk` và tự thêm vào mảng middleware (`store.js:5,22`) |

Cách viết đúng chỉ tốn chừng này dòng:

```js
// ĐÂY LÀ CODE ĐÚNG — dự án CHƯA có, ta sẽ sửa thật ở Bài 34
export const store = configureStore({
  reducer: persistedReducer,
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({ serializableCheck: false }).concat(
      productApi.middleware,
      sliderApi.middleware,
      cateProductApi.middleware,
      userApi.middleware,
      newsApi.middleware
    ),
});
```

`configureStore` **tự bật DevTools ở chế độ development**, **tự nạp `redux-thunk`**, và **tự bật
3 lớp kiểm tra**. Ta sẽ refactor ở [Bài 34](34-refactor-du-an.md); bài này **giữ nguyên** để bạn
vẫn chạy được dự án gốc.

---

## 9. 🛠️ Tự tay làm — viết `toppingSlice.js`

> Mục tiêu: cuối phần này bạn có một slice Topping hoàn chỉnh (state + reducer đồng bộ + selector),
> đã gắn vào store, và `ToppingPage` đọc dữ liệu **từ Redux** thay vì `useState` cục bộ.

**Ở bài trước bạn đã viết** `src/api/topping.js` để gọi API. **Bài này ta làm tiếp**: đưa dữ liệu
đó vào Redux để mọi component dùng chung.

> 📌 Toàn bộ code phần này là **code bạn tự viết thêm — dự án gốc chưa có**.

### Bước 1 — Tạo file `src/redux/toppingSlice.js`

```js
// yotea-fe/src/redux/toppingSlice.js  ← file MỚI, bạn tự tạo
import { createSlice } from "@reduxjs/toolkit";

const initialState = {
  value: [],
};

const toppingSlice = createSlice({
  name: "topping",
  initialState,
  reducers: {
    setToppings(state, { payload }) {
      state.value = payload;
    },
  },
});

export const selectToppings = (state) => state.topping.value;
export const { setToppings } = toppingSlice.actions;
export default toppingSlice.reducer;
```

Đối chiếu với `cartSlice.js` bạn vừa mổ xẻ:

| Phần | Ở `cartSlice` | Ở `toppingSlice` của bạn |
|---|---|---|
| `name` | `"cart"` | `"topping"` → action type là `"topping/setToppings"` |
| `initialState` | `{ cart: [] }` | `{ value: [] }` — **theo đúng quy ước 8 slice khác** |
| reducer | 4 cái | 1 cái: `setToppings` |
| selector | `selectCart` | `selectToppings` |

> 💡 **Vì sao `{ value: [] }` chứ không phải `{ topping: [] }`?** Vì `cartSlice` dùng
> `{ cart: [] }` nên mới đẻ ra `state.cart.cart` xấu xí. 8 slice còn lại đều dùng `value` — hãy
> theo số đông. State của bạn sẽ là `state.topping.value`, đọc rõ nghĩa.

### Bước 2 — Đăng ký vào `rootReducer.js`

Mở `yotea-fe/src/redux/rootReducer.js`, thêm **2 dòng** (đây là **ngoại lệ duy nhất** bạn được
sửa file dự án trong mạch thực hành — không đăng ký thì slice vô dụng):

```js
// yotea-fe/src/redux/rootReducer.js — dòng bạn TỰ THÊM
import toppingReducer from "./toppingSlice";   // đặt cạnh các import slice khác

const store = combineReducers({
  cateProduct: cateProductReducer,
  // ... giữ nguyên toàn bộ các dòng cũ ...
  cart: cartReducer,
  topping: toppingReducer,          // ← DÒNG MỚI: quyết định state nằm ở state.topping
  [productApi.reducerPath]: productApi.reducer,
  // ... giữ nguyên phần còn lại ...
});
```

> ⚠️ **Nhớ kỹ:** key `topping` ở đây **phải khớp** với `state.topping.value` trong selector. Sai
> một chữ là `selectToppings` trả `undefined` mà **không có lỗi nào hiện ra** — đúng cái bẫy ở mục 7.3.

### Bước 3 — Dùng trong `ToppingPage.js`

Mở `yotea-fe/src/pages/user/ToppingPage.js` (tạo ở Bài 16), bỏ `useState`, chuyển sang Redux:

```jsx
// yotea-fe/src/pages/user/ToppingPage.js — bạn TỰ SỬA
import { useEffect } from "react";
import { useDispatch, useSelector } from "react-redux";
import { getAll } from "../../api/topping";
import { selectToppings, setToppings } from "../../redux/toppingSlice";
import ToppingCard from "../../components/user/ToppingCard";
import { updateTitle } from "../../utils";

const ToppingPage = () => {
  const dispatch = useDispatch();
  const toppings = useSelector(selectToppings);   // ĐỌC từ store

  useEffect(() => {
    updateTitle("Topping");

    const fetchToppings = async () => {
      const { data } = await getAll();
      dispatch(setToppings(data));                // GHI vào store
    };
    fetchToppings();
  }, [dispatch]);

  return (
    <section className="container max-w-6xl mx-auto px-3 mb-8">
      <h1 className="text-2xl font-semibold my-6">Danh sách topping</h1>

      <div className="grid grid-cols-12 gap-4">
        {toppings.map((topping) => (
          <ToppingCard key={topping._id} topping={topping} />
        ))}
      </div>
    </section>
  );
};

export default ToppingPage;
```

**Đọc lại luồng cho chắc:**

```
ToppingPage mount
   → useEffect chạy
      → getAll()  gọi GET /api/topping          (tầng API — Bài 18)
      → dispatch(setToppings(data))             (nộp phiếu)
         → reducer setToppings gán state.value = data
            → store đổi → useSelector trả về mảng mới
               → ToppingPage re-render, vẽ ra các ToppingCard
```

> 💡 **Vì sao dependency là `[dispatch]` chứ không phải `[]`?** `dispatch` được React-Redux đảm
> bảo **ổn định** (không đổi giữa các lần render) nên effect vẫn chỉ chạy 1 lần — nhưng ESLint
> `react-hooks/exhaustive-deps` sẽ không càu nhàu nữa.

---

## 10. ✅ Kiểm chứng kết quả

```bash
# terminal 1 — đứng tại thư mục yotea-be
npm start

# terminal 2 — đứng tại thư mục yotea-fe
npm start
```

Mở `http://localhost:3000/topping` — danh sách topping phải hiển thị **y như trước**, nhưng lần
này dữ liệu đi qua Redux.

**Bước 1 — in state ra console.** Vì dự án không có Redux DevTools (mục 8.3), ta kiểm chứng thô
sơ. Thêm tạm vào `ToppingPage.js`:

```jsx
  const toppings = useSelector(selectToppings);
  console.log("state.topping.value =", toppings);   // ← dòng TẠM để kiểm chứng
```

Mở F12 → tab **Console**, phải thấy **đúng 2 lần in**:

```
state.topping.value = []                       ← lần render đầu, state rỗng
state.topping.value = (3) [{…}, {…}, {…}]      ← sau khi dispatch(setToppings)
```

Bung một phần tử ra phải thấy đúng dữ liệu backend trả về:

```json
{
  "_id": "6650a1f2c4e8b91234abcd77",
  "name": "Trân châu đen",
  "price": 5000,
  "slug": "tran-chau-den",
  "createdAt": "2026-08-15T09:12:00.000Z"
}
```

**Bước 2 — chứng minh "kho dùng chung".** Thêm tạm vào **`WebsiteLayout.js`** (file hoàn toàn
khác, không nhận props gì từ `ToppingPage`):

```jsx
  const toppings = useSelector(selectToppings);       // nhớ import selectToppings
  console.log("Layout thấy", toppings.length, "topping");
```

Vào `/topping` rồi xem Console: `WebsiteLayout` **cũng in ra đúng số topping**, dù chẳng ai
truyền gì cho nó. **Đó chính xác là điều Redux giải quyết** — hết prop drilling.

**Bước 3 — soi cả cây state.** Thêm tạm `window.store = store;` vào `yotea-fe/src/index.js`, rồi
gõ trong Console:

```js
store.getState().topping
// → { value: Array(3) }

Object.keys(store.getState())
// → ['cateProduct','news','cateNews','contact','slider','product','store',
//    'user','wishlist','auth','cart','topping','productApi',...,'_persist']
```

Thấy `topping` nằm cạnh 11 slice kia nghĩa là bạn đã đăng ký thành công.
**Kiểm chứng xong nhớ xoá hết `console.log` và dòng `window.store` đi nhé.**

---

## 11. 🐞 Lỗi thường gặp

| Thông báo lỗi | Nguyên nhân | Cách sửa |
|---|---|---|
| `Cannot read properties of undefined (reading 'value')` | Selector viết `state.topping.value` nhưng chưa đăng ký `topping:` trong `rootReducer` | Thêm `topping: toppingReducer` vào `combineReducers` |
| `toppings.map is not a function` | `payload` truyền vào là cả response axios, không phải mảng | `dispatch(setToppings(data))` — nhớ bóc `data` ra trước |
| `Actions must be plain objects` | `dispatch(setToppings)` **thiếu ngoặc gọi hàm** | Viết `dispatch(setToppings(data))` |
| `could not find react-redux context value` | Component nằm **ngoài** `<Provider store={store}>` | Kiểm tra `yotea-fe/src/index.js` |
| `Cannot return a new state and modify the draft` | Reducer vừa `state.value = ...` vừa `return {...}` | Chọn **một** kiểu |
| Component không re-render dù state đã đổi | Mutate state **ngoài** reducer (Immer chỉ chạy trong reducer) | Luôn đi qua `dispatch` |
| `useSelector` chạy vô hạn, app đơ | Selector **tạo mảng/object mới mỗi lần**, ví dụ `useSelector(s => s.topping.value.filter(...))` | Trả về giá trị nguyên gốc rồi lọc trong component, hoặc dùng `createSelector` |
| Không thấy Redux DevTools | Dự án dùng `createStore` thay vì `configureStore` | Xem mục 8.3 — sửa ở [Bài 34](34-refactor-du-an.md) |

---

## 12. 📝 Bài tập

**Bài 1.** `addCart` so sánh bằng **`productId`**, còn `removeItemCart` lọc bằng **`item.id`**.
(a) `item.id` sinh ra ở đâu, bằng hàm gì? (b) Điều gì xảy ra nếu ai đó thêm nút "Mua nhanh" ở
trang chủ mà quên sinh `id`?

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

(a) Sinh bằng `uuidv4()` tại **đúng một chỗ duy nhất trong toàn dự án**:

```
yotea-fe/src/pages/user/ProductDetailPage.js:20:import { v4 as uuidv4 } from "uuid";
yotea-fe/src/pages/user/ProductDetailPage.js:51:      id: uuidv4(),
```

(b) Item vào giỏ với `id === undefined`, kéo theo:

- **Xoá sai hàng loạt:** `removeItemCart(undefined)` chạy `filter((item) => item.id !== undefined)`
  → **mọi dòng có `id === undefined` bị xoá cùng lúc**, dù người dùng chỉ bấm ✕ trên **một** dòng.
- **Sửa nhầm số lượng:** `updateQuantity` dùng `find((item) => item.id === cartItem.id)` → luôn
  tìm thấy **dòng `undefined` đầu tiên**, nên gõ số cho dòng 3 lại sửa nhầm dòng 1.
- Nghịch lý: `addCart` **vẫn chạy đúng** vì nó gộp theo `(productId, ice, sugar)` — nên bug ẩn
  mình rất lâu, chỉ lộ khi người dùng bấm xoá.

**Cách phòng:** sinh `id` trong `prepare` callback của chính `addCart` (mục 6), để không nơi gọi
nào có thể "quên".

</details>

**Bài 2.** Viết thêm cho `toppingSlice.js` hai reducer đồng bộ: `addTopping` (thêm 1 topping vào
cuối mảng) theo **kiểu mutate**, và `removeTopping` (xoá theo `_id`) theo **kiểu gán mới**.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

```js
// yotea-fe/src/redux/toppingSlice.js — code bạn tự viết thêm
  reducers: {
    setToppings(state, { payload }) {
      state.value = payload;
    },

    // kiểu MUTATE — giống cart.push(newProduct) ở cartSlice.js:20
    addTopping(state, { payload }) {
      state.value.push(payload);
    },

    // kiểu GÁN MỚI — giống cartSlice.js:26
    removeTopping(state, { payload }) {
      state.value = state.value.filter((item) => item._id !== payload);
    },
  },
```

Nhớ export: `export const { setToppings, addTopping, removeTopping } = toppingSlice.actions;`

Cả hai kiểu đều đúng nhờ Immer. Kiểu `state.value = [...state.value, payload]` cũng chạy — và đó
chính là cách **8 slice CRUD của dự án** đang viết
(`yotea-fe/src/redux/categoryProductSlice.js:52-54`):

```js
    builder.addCase(addCate.fulfilled, (state, { payload }) => {
      state.value = [...state.value, payload];
    });
```

</details>

**Bài 3.** Nếu bạn đổi dòng `cart: cartReducer` trong `rootReducer.js:30` thành
`gioHang: cartReducer` thì **những gì sẽ hỏng**, và cần sửa ở đâu?

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

**Hỏng 2 thứ:**

1. **Selector `selectCart` vỡ.** `cartSlice.js:46` viết `state.cart.cart` → giờ `state.cart` là
   `undefined` → `Cannot read properties of undefined (reading 'cart')`. Cả 3 nơi dùng
   (`WebsiteLayout.js:53`, `CartPage.js:18`, `CheckoutPage.js:36`) đều trắng trang.
2. **redux-persist mất giỏ hàng.** `store.js:16` khai `whitelist: ["auth", "cart"]` — key `cart`
   không còn tồn tại nên giỏ hàng **không được lưu nữa**, F5 là mất sạch.

**Cần sửa 2 chỗ:** `cartSlice.js:46` → `state.gioHang.cart`, và `store.js:16` →
`whitelist: ["auth", "gioHang"]`.

**Bài học:** mount key trong `combineReducers` là một **hợp đồng ngầm** với selector và với
redux-persist. Đây là loại lỗi **TypeScript bắt được ngay**, còn JavaScript thuần chỉ nổ lúc chạy.
Cũng vì thế mà `categoryProductSlice.js:40` khai `name: "categoryProduct"` nhưng mount ở key
`cateProduct` là một quả bom hẹn giờ — may là selector đang viết đúng.

</details>

---

## 📌 Tóm tắt

- Redux sinh ra để diệt **prop drilling** — ví dụ sống là badge giỏ hàng ở `WebsiteLayout` nhưng
  dữ liệu lại được thêm từ `ProductDetailPage`, hai nhánh xa nhau trên cây component.
- Ví von: **store = kho**, **reducer = thủ kho** (người duy nhất được sửa hàng), **action = phiếu
  yêu cầu**, **dispatch = nộp phiếu**, **selector = phiếu tra cứu**.
- Ba nguyên tắc: **một nguồn sự thật duy nhất** · **state chỉ đọc** · **thay đổi chỉ qua hàm thuần**.
- `createSlice({ name, initialState, reducers })` **tự sinh** action type (`"cart/addCart"`),
  action creator và reducer — thay cho 3 file của Redux thuần.
- **Immer** cho phép viết `cart.push(...)` trong reducer mà vẫn bất biến — nhưng **chỉ bên trong
  reducer của `createSlice`**.
- `useSelector(selector)` để **đọc**, `useDispatch()` + `dispatch(action())` để **ghi**. Luôn viết
  selector thay vì rải `state.cart.cart` khắp nơi.
- `combineReducers` ghép 11 slice; **key bên trái quyết định state nằm ở đâu** — sai key là
  selector trả `undefined` trong im lặng.
- ⚠️ Dự án dùng `createStore` (đã deprecated) thay cho `configureStore` ⇒ **mất Redux DevTools**
  và toàn bộ lớp kiểm tra khi phát triển. Sẽ sửa ở [Bài 34](34-refactor-du-an.md).

**Từ khoá tra cứu thêm:** `redux toolkit createSlice`, `immer produce draft`, `useSelector useDispatch`,
`combineReducers`, `redux three principles`, `reselect createSelector`, `configureStore vs createStore`

➡️ **Bài tiếp theo:** [20 — `createAsyncThunk` và `extraReducers`](20-async-thunk.md) — slice của
bạn hiện vẫn phải gọi API thủ công trong `useEffect` rồi mới `dispatch`. Bài sau ta gói cả việc
gọi API **vào bên trong Redux**, kèm luôn ba trạng thái `pending / fulfilled / rejected`.
