# Bài 17 — Tailwind CSS trong dự án

> **Phần 3 · Frontend React** — Thời lượng ước tính: **~70 phút**
> ⬅️ Bài trước: [16 — Layout và component tái sử dụng](16-layout-va-component.md) · Bài sau: [18 — Tầng gọi API với axios](18-tang-api-axios.md) ➡️
> 🏠 [Mục lục](README.md)

---

## 🎯 Sau bài này bạn sẽ

- Giải thích được **utility-first** là gì và khác gì cách viết CSS truyền thống.
- Đọc được mọi chuỗi `className` dài ngoằng trong Yotea mà không hoảng.
- Tra cứu nhanh ~50 class hay dùng nhất, biết chúng tương đương CSS nào.
- Dựng được bố cục **responsive** bằng tiền tố `sm:` `md:` `lg:` theo tư duy mobile-first.
- Hiểu **hệ lưới 12 cột** và giá trị tuỳ ý trong ngoặc vuông `bg-[#D9A953]`.
- Tự tay trang trí `ToppingCard` khớp phong cách sẵn có của Yotea.

## 📋 Cần chuẩn bị

- Đã hoàn thành [Bài 16](16-layout-va-component.md): bạn đã có
  `yotea-fe/src/components/user/ToppingCard.js` và `yotea-fe/src/pages/user/ToppingPage.js`.
- Dự án frontend chạy được (`npm start` trong `yotea-fe`, mở `http://localhost:3000`).
- Biết CSS cơ bản: `padding`, `margin`, `display: flex`, `color`, `border`.

> **Ở bài trước bạn đã tách `ToppingCard` thành component tái sử dụng và cắm nó vào `ToppingPage`**
> — nhưng nhìn còn trần trụi, chữ đen trên nền trắng, xếp một cột dọc. **Bài này ta làm tiếp việc
> trang trí:** học Tailwind rồi khoác cho `ToppingCard` bộ áo giống hệt thẻ sản phẩm của Yotea.

---

## 1. Tailwind là gì?

### 1.1. CSS truyền thống vs. utility-first

Cách cũ: đặt tên class, rồi mở **một file khác** viết luật cho tên đó.

```html
<div class="product-card"><h3 class="product-card__name">Trà sữa</h3></div>
```
```css
/* style.css — phải nhảy qua lại giữa 2 file */
.product-card { border: 1px solid #e5e7eb; border-radius: 8px; padding: 12px; }
.product-card__name { font-size: 18px; font-weight: 600; }
```

Ba nỗi khổ kinh điển: **đặt tên mệt hơn viết code**, **nhảy file liên tục**, và **file CSS chỉ
phình ra** (xoá component xong không ai dám xoá CSS của nó vì sợ chỗ khác dùng).

Cách của Tailwind — ghép các viên gạch nhỏ ngay trên thẻ, **không có file CSS nào cả**:

```jsx
<div className="border rounded-lg p-3">
  <h3 className="text-lg font-semibold">Trà sữa</h3>
</div>
```

> 📖 **Thuật ngữ:** *utility class* — class **chỉ làm đúng một việc**, ví dụ `p-3` = `padding: 12px`.
> *Utility-first* = ưu tiên ghép các viên gạch đó trên thẻ, thay vì tự đặt tên class mới.

### 1.2. Ưu và nhược điểm — nói thật lòng

| ✅ Ưu | ❌ Nhược |
|---|---|
| Không phải đặt tên class — hết tranh cãi BEM/SMACSS | JSX rất khó đọc, chuỗi class 300 ký tự là chuyện thường |
| Không nhảy file — sửa ngay tại chỗ đang nhìn | Phải học thuộc bảng tên (`justify-between`, không phải `space-between`) |
| CSS không phình: chỉ sinh class **thực sự được dùng** | Lặp lại kinh khủng — 4 thẻ giống nhau = 4 lần dán cùng chuỗi |
| Nhất quán tự động (`p-3`, `p-4`… hết chuyện chỗ 13px chỗ 14px) | **Class ghép chuỗi động bị xoá mất lúc build** — cạm bẫy số 1 (mục 2.2) |
| Responsive/hover rẻ tiền: thêm `md:` hay `hover:` là xong | Khó grep: tìm "mọi nút cam" phải grep `bg-orange-400` |

> 💡 **Cân bằng:** Tailwind rất hợp dự án nhỏ/vừa, team ít người — đúng như Yotea. Dự án lớn vẫn
> dùng Tailwind nhưng **bọc lại thành component** (`<Button />`, `<Card />`) để khỏi dán 10 lần.

---

## 2. Soi code thật: Tailwind được cấu hình thế nào trong Yotea

### 2.1. File cấu hình

`yotea-fe/tailwind.config.js:1-8`

```js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: ["./src/**/*.{js,jsx,ts,tsx}"],
  theme: {
    extend: {},
  },
  plugins: [],
};
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 1 | `/** @type {...} */` | Chú thích kiểu cho editor → gõ `content:` là có gợi ý |
| 2 | `module.exports = {` | CommonJS — file chạy bằng Node lúc build, không phải ES module |
| 3 | `content: ["./src/**/*.{js,jsx,ts,tsx}"]` | **Quan trọng nhất.** Danh sách file Tailwind phải "đọc" để biết bạn dùng class nào |
| 4-6 | `theme: { extend: {} }` | Nơi mở rộng bảng màu/khoảng cách/font — ở đây **để trống** |
| 7 | `plugins: []` | Không cài plugin nào (`forms`, `typography`, `line-clamp` đều không có) |

Toàn bộ cấu hình chỉ **8 dòng** mà Tailwind vẫn chạy, vì `react-scripts@5`
(`yotea-fe/package.json:22`) **tự phát hiện** file này rồi tự thêm plugin PostCSS — dự án **không
có** `postcss.config.js`, cũng **không có** `autoprefixer`. Đây là "magic" riêng của CRA 5.

`yotea-fe/package.json:61-63`

```json
  "devDependencies": {
    "tailwindcss": "^3.4.3"
  }
```

Tailwind chỉ là **devDependency** — nó là công cụ build sinh ra file CSS, không đi kèm sản phẩm cuối.

### 2.2. `content` quan trọng đến mức nào?

Tailwind hoạt động kiểu **quét chuỗi**: mở từng file khớp `content`, dùng regex tìm mọi đoạn text
trông giống tên class, rồi **chỉ sinh CSS cho những cái tìm thấy**.

```
content: ["./src/**/*.js"] ──► Tailwind mở TẤT CẢ file khớp, quét text thô
                               thấy "p-3", "flex", "bg-orange-400", "lg:col-span-8"…
                               └─► sinh đúng chừng đó luật CSS  →  bundle chỉ vài chục KB
```

> ⚠️ **Cạm bẫy số 1 — class ghép chuỗi động SẼ BỊ XOÁ MẤT.** Tailwind quét text thô, nó **không
> chạy JavaScript**. Đoạn này trông đúng nhưng không có màu:
>
> ```jsx
> const color = "red";
> <div className={`bg-${color}-500`}>…</div>   // ❌ file không hề chứa chuỗi "bg-red-500"
> ```
>
> Cách đúng là **viết đủ tên class nguyên văn ở đâu đó trong file**:
>
> ```jsx
> const COLORS = { red: "bg-red-500", green: "bg-green-500" };   // ✅
> <div className={COLORS[color]}>…</div>
> ```

Dự án có sẵn một lỗi gần giống — `yotea-fe/src/pages/admin/category/CategoryListPage.js:99`:

```jsx
                      <tr key={index} className="cate__list-item-${cate.id}">
```

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** tác giả định dùng template literal nhưng gõ **dấu nháy kép**
> thay vì backtick, nên class thật trong DOM đúng là chuỗi literal `cate__list-item-${cate.id}`.
> May là nó không phải class Tailwind và không CSS nào dùng nên chưa ai phát hiện — bug "im lặng"
> điển hình.

### 2.3. Ba dòng `@tailwind` trong `index.css`

`yotea-fe/src/index.css:1-3`

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

| Directive | Chèn cái gì vào |
|---|---|
| `@tailwind base` | **Preflight** — reset CSS: bỏ margin mặc định của `body`/`h1`, `img` thành `display:block`… |
| `@tailwind components` | Chỗ dành cho class component (kể cả class bạn viết bằng `@apply`) |
| `@tailwind utilities` | Toàn bộ utility: `p-3`, `flex`, `text-lg`… — đặt cuối để **đè** được các luật trên |

File này nạp một lần duy nhất ở `yotea-fe/src/index.js:3` (`import "./index.css";`) → có hiệu lực toàn site.

> 💡 **Preflight giải thích một hiện tượng lạ:** viết `<h1>Tiêu đề</h1>` trong Yotea ra chữ **bé như
> chữ thường**. Đó là do `@tailwind base` đã xoá style mặc định của trình duyệt — muốn to phải tự
> thêm `text-2xl font-semibold`, đúng như `HomeProducts.js:91` đang làm.

---

## 3. Bảng tra cứu class hay dùng

Quy tắc chung: **số trong tên class × 4 = số pixel** (`p-3` → 12px, `mt-10` → 40px).
Ngoại lệ: `p-0.5` = 2px, `p-px` = 1px.

### 3.1. Khoảng cách và kích thước

| Class | CSS tương đương | | Class | CSS tương đương |
|---|---|---|---|---|
| `p-3` | `padding: 12px` | | `w-full` | `width: 100%` |
| `px-2` | `padding-left/right: 8px` | | `w-8` / `w-10` | `width: 32px / 40px` |
| `py-3` | `padding-top/bottom: 12px` | | `h-10` / `h-full` | `height: 40px` / `100%` |
| `pt-3` `pb-1.5` `pl-1` `pr-6` | padding một phía (top/bottom/left/right) | | `max-w-6xl` | `max-width: 72rem` (1152px) — khung nội dung của Yotea |
| `mt-10` / `mb-9` | `margin-top: 40px` / `margin-bottom: 36px` | | `min-h-40` | `min-height: 10rem` (160px) |
| `mx-auto` | `margin-left/right: auto` — căn giữa khối | | `min-w-[150px]` | `min-width: 150px` (giá trị tuỳ ý, mục 5) |
| `gap-5` / `gap-x-4` | `gap: 20px` / `column-gap: 16px` (khoảng cách trong flex/grid) | | `h-40` | `height: 10rem` (160px) |

### 3.2. Flex và Grid

| Class | CSS tương đương | | Class | CSS tương đương |
|---|---|---|---|---|
| `flex` | `display: flex` | | `grid` | `display: grid` |
| `items-center` | `align-items: center` | | `grid-cols-12` | `grid-template-columns: repeat(12, minmax(0,1fr))` |
| `justify-between` | `justify-content: space-between` | | `grid-cols-2` | Chia 2 cột bằng nhau |
| `justify-center` / `justify-end` | `justify-content: center` / `flex-end` | | `col-span-6` | `grid-column: span 6 / span 6` — chiếm 6 cột |
| `flex-col` | `flex-direction: column` | | `lg:col-span-8` | Chỉ từ ≥1024px mới chiếm 8 cột |
| `relative` / `absolute` | `position: relative` / `absolute` | | `divide-y` | Kẻ đường ngang **giữa** các con |

### 3.3. Chữ và màu

| Class | CSS tương đương | | Class | CSS tương đương |
|---|---|---|---|---|
| `text-xs`/`sm`/`base`/`lg`/`2xl` | `font-size: 12/14/16/18/24px` | | `text-gray-500` | `color: #6b7280` |
| `font-light` / `font-semibold` | `font-weight: 300` / `600` | | `text-gray-400` / `text-gray-50` | Xám nhạt hơn / gần trắng |
| `uppercase` | `text-transform: uppercase` | | `text-white` / `text-black` / `text-red-500` | Trắng / đen / đỏ báo lỗi |
| `text-center`/`right`/`justify` | `text-align: …` | | `bg-orange-400` | `background-color: #fb923c` — nút chính của Yotea |
| `leading-tight` / `leading-5` | `line-height: 1.25` / `20px` | | `bg-white` / `bg-gray-50` | Nền trắng / xám rất nhạt |
| `select-none` | `user-select: none` — không cho bôi đen chữ nút | | `border-[#D9A953]` | Viền **màu tuỳ ý** = màu thương hiệu Yotea |

### 3.4. Viền, bo góc, hiệu ứng và trạng thái

| Class | CSS tương đương | | Class | CSS tương đương |
|---|---|---|---|---|
| `border` / `border-2` | `border-width: 1px` / `2px` | | `shadow` / `shadow-md` / `shadow-lg` | `box-shadow` nhẹ → nặng |
| `border-t` / `border-l` | Chỉ viền trên / trái | | `shadow-none` | Xoá bóng |
| `border-b-2` | Viền **dưới** dày 2px | | `transition` | Bật hiệu ứng chuyển mượt |
| `border-gray-500` | Đổi màu viền | | `duration-300` / `ease-linear` | 300ms / timing-function tuyến tính |
| `rounded` / `rounded-md` / `rounded-lg` | `border-radius: 4px / 6px / 8px` | | `opacity-0` / `opacity-95` | `opacity: 0` / `0.95` |
| `rounded-full` | `border-radius: 9999px` — tròn hoàn toàn | | `hover:` / `focus:` | Áp dụng khi rê chuột / khi ô được chọn |
| `overflow-hidden` | Cắt phần con tràn ra ngoài | | `group` + `group-hover:` | Rê chuột vào **cha** → **con** đổi style |

### 3.5. Ví dụ thật của `group` / `group-hover`

`yotea-fe/src/components/user/home/HomeProducts.js:100-118` (trích phần cốt lõi, class dài đã giữ nguyên văn):

```jsx
      <div className="grid grid-cols-2 md:grid-cols-4 gap-3 mt-5">
        {products?.map((item, index) => (
          <div className="group" key={index}>
            <div className="relative bg-[#f7f7f7] overflow-hidden">
              <Link
                to={`/san-pham/${item.slug}`}
                style={{ backgroundImage: `url(${item.image})` }}
                className="bg-cover pt-[100%] bg-center block"
              />
              <button className="absolute w-full bottom-0 h-9 bg-[#D9A953] ... translate-y-full group-hover:translate-y-0">
                Xem nhanh
              </button>
              <button
                onClick={() => handleFavorites(item._id, item.slug)}
                className="btn-heart absolute top-3 right-3 w-8 h-8 rounded-full border-2 ... opacity-0 group-hover:opacity-100"
              >
                <FontAwesomeIcon icon={faHeart} />
              </button>
            </div>
```

*(Hai chuỗi class ở dòng 109 và 114 đã được rút gọn bằng `...` cho dễ đọc; phần bị lược là các
class màu và `transition ease-linear duration-300` / `hover:*`.)*

**Đọc từng dòng:**

| Dòng | Class then chốt | Ý nghĩa |
|---|---|---|
| 100 | `grid grid-cols-2 md:grid-cols-4 gap-3` | 2 cột trên mobile, 4 cột từ 768px, cách nhau 12px |
| 102 | `group` | "Tôi là cha — con tôi được phép phản ứng khi rê chuột vào tôi" |
| 103 | `relative … overflow-hidden` | Làm mốc định vị cho con `absolute`; cắt phần con tràn ra |
| 107 | `bg-cover pt-[100%] bg-center block` | Mẹo tạo **ô vuông**: `pt-[100%]` = padding-top bằng 100% chiều rộng |
| 109 | `translate-y-full group-hover:translate-y-0` | Nút "Xem nhanh" ẩn dưới đáy, rê vào thẻ thì trượt lên |
| 114 | `opacity-0 group-hover:opacity-100` | Nút trái tim vô hình, rê vào thẻ mới hiện |

Toàn bộ hiệu ứng này **không cần một dòng JavaScript nào**.

---

## 4. Responsive: `sm:` `md:` `lg:` và tư duy mobile-first

| Tiền tố | Áp dụng khi màn hình rộng **từ** | Thiết bị điển hình |
|---|---|---|
| *(không có)* | 0px — **mọi màn hình** | Điện thoại |
| `sm:` | 640px | Điện thoại nằm ngang |
| `md:` | 768px | Máy tính bảng |
| `lg:` | 1024px | Laptop |
| `xl:` | 1280px | Màn hình lớn |

**Mobile-first** nghĩa là class **không có tiền tố** là mặc định cho **điện thoại**; các tiền tố
chỉ **ghi đè khi màn hình TO HƠN**. Không bao giờ có chuyện `md:` áp dụng cho màn nhỏ hơn 768px.

```
        0px          768px        1024px         →
grid-cols-1  ────────►│             │        điện thoại: 1 cột
        │  md:grid-cols-2 ─────────►│        tablet:    2 cột
        │             │  lg:grid-cols-4 ────► laptop:   4 cột
```

### 4.1. Ví dụ thật: `col-span-12 lg:col-span-8`

`yotea-fe/src/pages/user/cart/CheckoutPage.js:119-130`

```jsx
          <form
            action=""
            onSubmit={handleSubmit(onSubmit)}
            method="POST"
            className="container max-w-6xl mx-auto px-3 mt-10 mb-9 grid grid-cols-12 gap-5"
          >
            <div className="col-span-12 lg:col-span-8 border-t-2 pt-3">
              <div className="flex items-center justify-between mb-2">
                <h3 className="uppercase text-gray-500 font-semibold text-lg">
                  Thông tin thanh toán
                </h3>
              </div>
```

**Đọc từng dòng:**

| Dòng | Class | Ý nghĩa |
|---|---|---|
| 123 | `container max-w-6xl mx-auto px-3` | Khung nội dung rộng tối đa 1152px, tự căn giữa, chừa 12px hai mép |
| 123 | `mt-10 mb-9` | Cách trên 40px, cách dưới 36px |
| 123 | `grid grid-cols-12 gap-5` | Chia **12 cột**, các ô cách nhau 20px |
| 125 | `col-span-12` | **Trên điện thoại:** chiếm 12/12 cột → full chiều ngang |
| 125 | `lg:col-span-8` | **Từ 1024px:** chỉ chiếm 8/12 ≈ 66% chiều ngang |
| 125 | `border-t-2 pt-3` | Kẻ vạch trên dày 2px, đẩy nội dung xuống 12px |
| 126 | `flex items-center justify-between` | Xếp ngang, căn giữa theo chiều dọc, đẩy hai đầu ra hai mép |

### 4.2. Ẩn/hiện theo màn hình

`yotea-fe/src/pages/layouts/WebsiteLayout.js:110`

```jsx
        <div className="bg-[#D9A953] hidden md:block">
```

Đọc là: *"thanh vàng trên cùng — **ẩn** trên điện thoại, **hiện lại** từ 768px."* Cặp `hidden` +
`md:block` là cách chuẩn để làm menu desktop / menu mobile khác nhau.

Các lưới responsive khác rải khắp dự án:

| File:dòng | Class | Nghĩa |
|---|---|---|
| `HomeProducts.js:100` | `grid grid-cols-2 md:grid-cols-4 gap-3` | 2 → 4 cột |
| `NewsContent.js:41` | `grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4` | 1 → 2 → 4 cột |
| `AboutPage.js:21` | `grid grid-cols-1 md:grid-cols-2 gap-5 mt-2` | 1 → 2 cột |
| `StorePage.js:95` | `col-span-12 md:col-span-7 min-h-[450px]` | Bản đồ chiếm 7/12 từ tablet |

---

## 5. Giá trị tuỳ ý trong ngoặc vuông `[...]`

Khi bảng có sẵn không có giá trị bạn cần, viết thẳng nó trong ngoặc vuông:

```
bg-[#D9A953] → background-color:#D9A953    pt-[100%]     → padding-top:100%
min-h-[450px] → min-height:450px           max-h-[70vh]  → max-height:70vh
```

**Vì sao phải thay dấu cách bằng `_`?** Vì tên class **không được chứa dấu cách** — có dấu cách là
trình duyệt hiểu thành *nhiều class khác nhau*. Tailwind quy ước: **gõ `_`, nó tự dịch ngược thành
dấu cách** khi sinh CSS.

Ví dụ thật — `yotea-fe/src/pages/auth/LoginPage.js:102-108`:

```jsx
          <input
            type="password"
            {...register("password")}
            id="form__login-password"
            className="shadow-[inset_0_1px_2px_rgba(0,0,0,0.1)] hover:shadow-none focus:shadow-[0_0_5px_#ccc] w-full border px-2 h-10 text-sm outline-none"
            placeholder="Mật khẩu"
          />
```

Dịch từng mảnh chuỗi class dòng 106:

| Class | CSS thật sinh ra | Hiệu ứng nhìn thấy |
|---|---|---|
| `shadow-[inset_0_1px_2px_rgba(0,0,0,0.1)]` | `box-shadow: inset 0 1px 2px rgba(0,0,0,0.1)` | Bóng **chìm vào trong** → ô trông lõm xuống |
| `hover:shadow-none` | `:hover { box-shadow: none }` | Rê chuột vào thì hết lõm |
| `focus:shadow-[0_0_5px_#ccc]` | `:focus { box-shadow: 0 0 5px #ccc }` | Bấm vào ô thì có quầng sáng xám |
| `w-full border px-2 h-10 text-sm outline-none` | Rộng 100%, viền 1px, đệm ngang 8px, cao 40px, chữ 14px, bỏ viền focus mặc định | Ô nhập chuẩn của dự án |

Để ý `rgba(0,0,0,0.1)` — bên trong ngoặc `()` **vẫn giữ dấu phẩy bình thường**, chỉ chỗ đáng lẽ là
*dấu cách* mới đổi thành `_`. Chuỗi này lặp nguyên xi ở **6 ô input của `RegisterPage.js` (dòng 82,
101, 117, 133, 152, 171)**, 2 ô của `LoginPage.js`, và toàn bộ form `CheckoutPage.js`.

Còn nút bấm — `yotea-fe/src/pages/auth/LoginPage.js:124-126`:

```jsx
        <button className="select-none mt-4 px-3 py-2 bg-orange-400 font-semibold uppercase text-white text-sm transition ease-linear duration-300 hover:shadow-[inset_0_0_100px_rgba(0,0,0,0.2)]">
          Đăng nhập
        </button>
```

`hover:shadow-[inset_0_0_100px_rgba(0,0,0,0.2)]` là mẹo hay: thay vì đổi `background-color` khi
hover, tác giả **phủ một lớp bóng đen mờ 20% toả rộng 100px vào trong** → nút tự tối đi mà không
cần biết màu nền là gì, dùng lại được cho nút cam, vàng hay xanh.

### 5.1. `#D9A953` — màu thương hiệu bị "rải" khắp nơi

`yotea-fe/src/index.css:56-58`

```css
.filter__btn-view.active {
  @apply text-[#D9A953];
}
```

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** màu thương hiệu `#D9A953` xuất hiện **97 lần** trong
> `yotea-fe/src` (đếm bằng `grep -ro "D9A953" src | wc -l`) dưới dạng giá trị tuỳ ý `[#D9A953]`,
> trong khi `theme.extend` (`tailwind.config.js:4-6`) **để trống**. Muốn đổi tông màu thương hiệu
> phải find-replace 97 chỗ, sót một chỗ là lệch màu.
>
> **Cách đúng** — khai một lần trong config *(đoạn này bạn tự viết thêm để tham khảo, dự án CHƯA có)*:
>
> ```js
> theme: { extend: { colors: { brand: "#D9A953" } } },
> ```
>
> Rồi dùng `bg-brand`, `text-brand`, `border-brand` ở mọi nơi. Ta sẽ thật sự làm việc này ở
> [Bài 34](34-refactor-du-an.md).

---

## 6. Hệ lưới 12 cột — vì sao là 12?

Vì **12 chia hết cho 2, 3, 4, 6** — chia đôi (6+6), chia ba (4+4+4), chia tư (3+3+3+3) hay chia
2/3–1/3 (8+4) đều ra số nguyên. Bootstrap, Foundation và Tailwind đều theo con số này.

Cột phải của trang thanh toán — `yotea-fe/src/pages/user/cart/CheckoutPage.js:244-247`:

```jsx
            <div className="col-span-12 lg:col-span-4 border-l p-4 border-2 border-[#D9A953] min-h-40">
              <h3 className="uppercase text-gray-500 font-semibold mb-3 text-lg">
                Đơn hàng của bạn
              </h3>
```

`8 + 4 = 12` — vừa khít một hàng. Và bên trong cột trái còn có **lưới lồng trong lưới** —
`yotea-fe/src/pages/user/cart/CheckoutPage.js:132-137`:

```jsx
              <div className="grid grid-cols-12 gap-x-4">
                <div className="col-span-6 mb-3">
                  <label
                    htmlFor="cart__checkout-form-name"
                    className="font-semibold mb-1 block"
                  >
                    Họ và tên *
```

Sơ đồ toàn trang:

```
container max-w-6xl · grid-cols-12 · gap-5
┌──────────────────────────────────────────────┬───────────────────┐
│ col-span-12 lg:col-span-8  (Thông tin TT)    │ col-span-12       │
│  ┌── grid-cols-12 gap-x-4  (lưới CON) ────┐  │ lg:col-span-4     │
│  │ col-span-6      │ col-span-6           │  │ (Đơn hàng của bạn)│
│  │ Họ và tên       │ Điện thoại           │  │ border-2          │
│  ├─────────────────┴──────────────────────┤  │ border-[#D9A953]  │
│  │ col-span-12  ·  Email                  │  │ min-h-40          │
│  └────────────────────────────────────────┘  │                   │
└──────────────────────────────────────────────┴───────────────────┘

  ĐIỆN THOẠI (<1024px): 2 khối xếp CHỒNG, mỗi khối chiếm trọn 12/12
```

**Điểm mấu chốt:** lưới con `grid-cols-12` ở dòng 132 **đếm lại từ đầu** trong phạm vi cột cha, chứ
không kế thừa 12 cột của cha. Nên `col-span-6` ở dòng 133 = **nửa cột trái**, không phải nửa trang.

> 💡 **Quy tắc vàng:** tổng `col-span-*` của các con trong một hàng phải **bằng đúng số cột của
> cha**. Thừa một cột là ô cuối bị đẩy xuống hàng dưới — lỗi bố cục kinh điển mà DevTools rất khó
> chỉ ra.

---

## 7. Mẹo thực chiến

**7.1. Cài extension "Tailwind CSS IntelliSense"** (Tailwind Labs) trong VS Code. Nó cho bạn: gợi ý
tên class khi gõ trong `className`; **xem CSS thật** khi rê chuột (rê vào `p-3` hiện
`padding: 0.75rem /* 12px */`); ô màu nhỏ cạnh `bg-orange-400` và `text-[#D9A953]`; cảnh báo class
trùng nhau. Extension đọc chính `yotea-fe/tailwind.config.js` → nhớ **mở thư mục `yotea-fe`** làm
workspace, đừng mở thư mục gốc repo, nếu không nó sẽ không tìm thấy config.

**7.2. Dùng DevTools để dò class.** Chuột phải → **Inspect**. Tab **Styles**: mỗi utility là **một
khối luật riêng**, bạn thấy đúng class nào tạo ra `padding: 12px`. Muốn thử nhanh, bấm vào ô
`class="…"` trong tab Elements, **gõ thêm class rồi Enter** — trang đổi ngay, không cần sửa code.
Class bị **gạch ngang** là class đang thua về độ ưu tiên.

> 💡 Thứ tự viết class trong `className` **không** quyết định class nào thắng. `p-3 p-4` thì `p-4`
> thắng không phải vì nó đứng sau, mà vì trong file CSS sinh ra `p-4` được xếp sau. Đừng dựa vào
> thứ tự — hãy xoá class thừa.

**7.3. Gom class lặp lại — ba cách.**

*Cách 1 — biến chuỗi trong file JS* (đơn giản nhất). Chuỗi vẫn nằm **nguyên văn** trong file nên
Tailwind quét thấy:

```js
// đoạn này bạn tự viết thêm — mẫu tham khảo, dự án CHƯA làm
const inputClass =
  "shadow-[inset_0_1px_2px_rgba(0,0,0,0.1)] hover:shadow-none focus:shadow-[0_0_5px_#ccc] w-full border px-2 h-10 text-sm outline-none";

<input className={inputClass} … />
```

*Cách 2 — `@apply` trong file CSS*, đúng như dự án đang làm ở `yotea-fe/src/index.css:9-11`:

```css
.store__list-map > div > iframe {
  @apply w-full h-full rounded-lg;
}
```

`@apply` = "nhét nội dung CSS của các utility này vào đây". Rất hợp khi bạn **không sửa được HTML**
— ví dụ iframe bản đồ do backend trả về dạng chuỗi rồi nhét vào bằng `dangerouslySetInnerHTML`
(`yotea-fe/src/components/user/Iframe.js:1-8`), nên không thể gắn class Tailwind trực tiếp.

*Cách 3 — tách component React.* Đây mới là cách "đúng React" nhất, và cũng là việc bạn sắp làm với
`ToppingCard`: viết chuỗi class **một lần**, dùng lại 100 lần.

### 7.4. Hai chỗ khó chịu của dự án — biết trước để đỡ mất thời gian

> ⚠️ **Chỗ này dự án làm chưa chuẩn (1): chuỗi class dài kinh khủng, dán lặp lại.** Chuỗi dài nhất
> dự án là **375 ký tự** ở `yotea-fe/src/pages/layouts/MyAccountLayout.js:64`, và nó bị dán y hệt
> **4 lần** — dòng 41, 49, 57, 64:
>
> `yotea-fe/src/pages/layouts/MyAccountLayout.js:39-44`
>
> ```jsx
>               <NavLink
>                 to="/my-account/"
>                 className="py-2 uppercase font-semibold text-sm text-gray-400 block transition ease-linear duration-200 hover:text-gray-700 relative hover:after:opacity-100 after:transition after:opacity-0 after:content-[''] after:absolute after:right-0 after:w-[3px] after:h-full after:bg-blue-500 after:top-1/2 after:-translate-y-1/2"
>               >
>                 Thông tin tài khoản
>               </NavLink>
> ```
>
> Đổi `bg-blue-500` sang màu thương hiệu là phải sửa **4 chỗ**, quên một chỗ là lệch. Cách đúng:
> tách thành component `<MyAccountNavItem to="…">…</MyAccountNavItem>`, hoặc ít nhất đặt chuỗi vào
> một `const` ở đầu file (mục 7.3).

> ⚠️ **Chỗ này dự án làm chưa chuẩn (2): luật `iframe { display: none }` toàn cục.**
>
> `yotea-fe/src/index.css:5-11`
>
> ```css
> iframe {
>   display: none;
> }
>
> .store__list-map > div > iframe {
>   @apply w-full h-full rounded-lg;
> }
> ```
>
> Dòng 5-7 **ẩn TOÀN BỘ iframe trên cả website**: Google Maps, video YouTube nhúng, plugin
> Facebook — tất cả biến mất. Dòng 9-11 là bản "vá": bật lại **riêng** iframe bản đồ trong
> `.store__list-map` (`yotea-fe/src/pages/user/StorePage.js:95`). Hậu quả: mai mốt bạn nhúng một
> video vào trang chủ, nó **không hiện** và bạn sẽ dò DevTools cả buổi — thủ phạm nằm ở file CSS
> toàn cục cách đó 10 thư mục. Cách đúng: viết `.some-wrapper iframe { display: none }` nhắm đúng
> chỗ cần ẩn, đừng đánh tổng lực vào tên thẻ.

---

## 8. 🛠️ Tự tay làm — trang trí `ToppingCard`

> Mục tiêu: cuối phần này, `http://localhost:3000/topping` hiển thị lưới thẻ topping **đẹp như thẻ
> sản phẩm của Yotea** — thẻ có viền, ảnh bo góc, hover đổi bóng, giá màu cam, và tự động
> **1 cột mobile → 2 cột tablet → 4 cột desktop**.

> ⚠️ Toàn bộ code dưới đây là **code bạn tự viết**. Dự án gốc **không có** `ToppingCard.js` hay
> `ToppingPage.js` — chúng do bạn tạo ra ở [Bài 16](16-layout-va-component.md).

### Bước 1 — Chốt "phong cách Yotea" (lấy chuẩn từ `HomeProducts.js:119-133`)

| Thành phần | Class Yotea đang dùng |
|---|---|
| Nền ảnh | `bg-[#f7f7f7]` — xám rất nhạt, tránh ảnh nền trắng bị "chìm" |
| Tên sản phẩm | `block font-semibold text-lg` |
| Nhãn phụ | `uppercase text-xs text-gray-400` |
| Giá | `text-sm pt-1` |
| Nút bấm | `bg-orange-400 … uppercase text-white text-sm transition ease-linear duration-300 hover:shadow-[inset_0_0_100px_rgba(0,0,0,0.2)]` |
| Khung nội dung | `container max-w-6xl mx-auto px-3` |

### Bước 2 — Viết lại phần `return` của `ToppingCard.js`

```jsx
// yotea-fe/src/components/user/ToppingCard.js  ← file BẠN tự viết, dự án gốc không có
import { Link } from "react-router-dom";
import { formatCurrency } from "../../utils";

const ToppingCard = ({ topping }) => {
  return (
    <div className="group border rounded-lg p-3 bg-white transition ease-linear duration-300 hover:shadow-lg hover:border-[#D9A953]">
      {/* KHỐI ẢNH */}
      <Link
        to={`/topping/${topping.slug}`}
        className="block overflow-hidden rounded-md bg-[#f7f7f7]"
      >
        <img
          src={topping.image}
          alt={topping.name}
          className="w-full h-40 object-cover transition duration-300 group-hover:scale-105"
        />
      </Link>

      {/* KHỐI CHỮ */}
      <div className="text-center pt-3">
        <p className="uppercase text-xs text-gray-400">Topping</p>

        <Link
          to={`/topping/${topping.slug}`}
          className="block font-semibold text-lg limit-line-2 transition duration-200 hover:text-[#D9A953]"
        >
          {topping.name}
        </Link>

        <div className="text-orange-500 font-semibold text-base pt-1">
          {formatCurrency(topping.price || 0)}
        </div>

        <button className="select-none mt-3 w-full py-2 bg-orange-400 font-semibold uppercase text-white text-sm transition ease-linear duration-300 hover:shadow-[inset_0_0_100px_rgba(0,0,0,0.2)]">
          Thêm vào ly
        </button>
      </div>
    </div>
  );
};

export default ToppingCard;
```

**Giải thích từng nhóm class:**

| Nhóm | Class | Vì sao dùng |
|---|---|---|
| **Khung thẻ** | `border rounded-lg p-3 bg-white` | Viền 1px, bo góc 8px, đệm 12px, nền trắng |
| | `group` | Cho phép ảnh phóng to khi rê chuột vào **cả thẻ**, không chỉ vào ảnh |
| | `transition ease-linear duration-300` | Hiệu ứng mượt 300ms — thiếu là bóng "nhảy" cụp một cái |
| | `hover:shadow-lg hover:border-[#D9A953]` | Rê chuột: nổi bóng + viền chuyển màu thương hiệu |
| **Khối ảnh** | `block overflow-hidden rounded-md` | `overflow-hidden` để ảnh phóng to bị **cắt gọn trong góc bo** |
| | `bg-[#f7f7f7]` | Nền xám nhạt — giống hệt `HomeProducts.js:103` |
| | `w-full h-40 object-cover` | Full ngang, cao cố định 160px, **`object-cover` chống méo ảnh** |
| | `group-hover:scale-105` | Rê chuột vào thẻ → ảnh phóng 105% |
| **Khối chữ** | `text-center pt-3` | Căn giữa, cách ảnh 12px |
| | `uppercase text-xs text-gray-400` | Nhãn phụ nhỏ, xám — sao chép đúng `HomeProducts.js:120` |
| | `font-semibold text-lg limit-line-2` | Tên đậm 18px; `limit-line-2` là **class riêng của dự án** (`index.css:29-35`) cắt tên dài còn 2 dòng kèm dấu `…` |
| | `text-orange-500 font-semibold` | **Giá màu cam**, đúng yêu cầu |
| **Nút** | `w-full py-2 bg-orange-400 … hover:shadow-[inset_0_0_100px_...]` | Copy đúng công thức nút của `LoginPage.js:124` |
| | `select-none` | Không cho bôi đen chữ trên nút — chi tiết nhỏ nhưng chuyên nghiệp |

> 💡 **Vì sao `formatCurrency(topping.price || 0)`?** Vì `formatCurrency`
> (`yotea-fe/src/utils/index.js:14-15`) gọi thẳng `currency.toLocaleString(...)` mà không kiểm tra
> `undefined`. Nếu API trả topping thiếu `price`, cả trang trắng với lỗi
> `Cannot read properties of undefined`. Dấu `|| 0` là tấm lưới an toàn.

### Bước 3 — Sửa lưới trong `ToppingPage.js`

```jsx
// yotea-fe/src/pages/user/ToppingPage.js  ← file BẠN tự viết
    <section className="container max-w-6xl mx-auto px-3 py-9">
      <div className="text-center mb-6">
        <h2 className="uppercase text-[#D9A953] text-2xl font-semibold">
          TOPPING YOTEA
        </h2>
        <p className="text-gray-500 text-sm mt-1">
          Thêm chút trân châu, thạch, pudding… cho ly trà sữa của bạn.
        </p>
      </div>

      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-5">
        {toppings?.map((item) => (
          <ToppingCard key={item._id} topping={item} />
        ))}
      </div>
    </section>
```

Dòng quan trọng nhất:

```
grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-5
     └─ mobile ┘ └─ ≥768px ──┘ └─ ≥1024px ──┘ └ cách nhau 20px
```

Đúng yêu cầu **1 cột mobile · 2 cột tablet · 4 cột desktop**. Để ý `grid-cols-1` **không có tiền
tố** — mặc định cho màn nhỏ nhất, đúng tinh thần mobile-first.

---

## 9. ✅ Kiểm chứng kết quả

```bash
# đứng tại thư mục yotea-fe
npm start
```

Mở `http://localhost:3000/topping` và kiểm tra đủ **5 điểm**:

| # | Việc cần làm | Kết quả phải thấy |
|---|---|---|
| 1 | Nhìn tổng thể trên laptop | **4 thẻ một hàng**, mỗi thẻ có viền mảnh và góc bo tròn |
| 2 | Rê chuột vào một thẻ | Thẻ **nổi bóng lên**, viền chuyển vàng `#D9A953`, ảnh **phóng nhẹ** trong khung bo góc |
| 3 | Nhìn dòng giá | Chữ **màu cam**, định dạng kiểu `10.000 ₫` |
| 4 | Kéo hẹp cửa sổ trình duyệt | Qua mốc 1024px → còn **2 cột**; qua mốc 768px → còn **1 cột** |
| 5 | F12 → chọn thẻ `<img>` | Tab Styles hiện `object-fit: cover` (từ `object-cover`) và `height: 10rem` (từ `h-40`) |

Muốn kiểm tra breakpoint chính xác: F12 → biểu tượng điện thoại (Toggle device toolbar) → chọn lần
lượt **iPhone SE (375px)** → 1 cột, **iPad (768px)** → 2 cột, **Responsive 1280px** → 4 cột.

---

## 10. 🐞 Lỗi thường gặp

| Hiện tượng | Nguyên nhân | Cách sửa |
|---|---|---|
| Gõ class Tailwind nhưng **không có gì đổi** | Class ghép động `` className={`bg-${x}-500`} `` — Tailwind quét text không thấy | Viết đủ tên class trong object/map (mục 2.2) |
| Sửa `tailwind.config.js` mà giao diện y nguyên | CRA chỉ đọc config **lúc khởi động** | Ctrl+C dừng `npm start` rồi chạy lại |
| Class trong file mới tinh không ăn | File nằm **ngoài** `./src/**`, không khớp `content` (`tailwind.config.js:3`) | Đặt file trong `src/`, hoặc bổ sung đường dẫn vào `content` |
| `bg-[#D9A953 ]` không có tác dụng | Có **dấu cách** bên trong ngoặc vuông | Dùng `_` thay dấu cách: `shadow-[0_0_5px_#ccc]` |
| `<h1>` hiện ra bé tí, không đậm | `@tailwind base` (Preflight) đã reset style mặc định | Tự thêm `text-2xl font-semibold` |
| Nhúng iframe/video mà **không hiện** | Luật `iframe { display: none }` ở `index.css:5-7` | Bọc trong wrapper rồi bật lại bằng `@apply` (mục 7.4) |
| `hover:` không hoạt động trên điện thoại | Điện thoại không có con trỏ chuột | Đừng giấu thông tin **quan trọng** sau `hover:` |
| Ảnh bị **méo/kéo giãn** | Chỉ đặt `w-full h-40`, thiếu `object-cover` | Thêm `object-cover` |
| Ảnh phóng to **tràn ra khỏi góc bo** | Thẻ cha thiếu `overflow-hidden` | Thêm `overflow-hidden` vào cha |
| DOM có class lạ tên `false` | `` className={`${active && "active"} …`} `` — khi `active` là `false`, template literal ép thành chuỗi `"false"` (lỗi thật ở `Loading.js:8`) | Viết `${active ? "active" : ""}` |

---

## 11. 📝 Bài tập

**Bài 1.** Dịch chuỗi class sau (nguyên văn từ `yotea-fe/src/pages/auth/LoginPage.js:124`) sang CSS
thuần, ghi rõ luật nào nằm trong `:hover`:

```
select-none mt-4 px-3 py-2 bg-orange-400 font-semibold uppercase text-white text-sm transition ease-linear duration-300 hover:shadow-[inset_0_0_100px_rgba(0,0,0,0.2)]
```

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

```css
button {
  user-select: none;                   /* select-none */
  margin-top: 16px;                    /* mt-4 */
  padding: 8px 12px;                   /* py-2 px-3 */
  background-color: #fb923c;           /* bg-orange-400 */
  font-weight: 600;                    /* font-semibold */
  text-transform: uppercase;           /* uppercase */
  color: #ffffff;                      /* text-white */
  font-size: 14px;                     /* text-sm */
  transition-property: color, background-color, border-color, ...;  /* transition */
  transition-timing-function: linear;  /* ease-linear */
  transition-duration: 300ms;          /* duration-300 */
}

button:hover {
  box-shadow: inset 0 0 100px rgba(0, 0, 0, 0.2);   /* hover:shadow-[...] */
}
```

Chỉ **một** luật nằm trong `:hover` — đúng class có tiền tố `hover:`. 12 class còn lại là style mặc
định. Đây là điểm quan trọng: **tiền tố `hover:` `focus:` `md:` chỉ ảnh hưởng đúng class nó dính
vào**, không lan sang class bên cạnh.

</details>

**Bài 2.** Thêm vào `ToppingCard` một **badge** ở góc trên bên phải khối ảnh, ghi chữ `MỚI`, nền
vàng `#D9A953`, chữ trắng, bo tròn, và **chỉ hiện khi** `topping.isNew` là `true`.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Gợi ý: cha cần `relative`, con dùng `absolute` + `top-2 right-2`.

```jsx
// đoạn này bạn tự viết thêm, trong ToppingCard.js
      <Link
        to={`/topping/${topping.slug}`}
        className="relative block overflow-hidden rounded-md bg-[#f7f7f7]"
      >
        {topping.isNew && (
          <span className="absolute top-2 right-2 z-10 px-2 py-0.5 rounded-full bg-[#D9A953] text-white text-xs font-semibold uppercase">
            Mới
          </span>
        )}
        <img
          src={topping.image}
          alt={topping.name}
          className="w-full h-40 object-cover transition duration-300 group-hover:scale-105"
        />
      </Link>
```

Ba điểm phải nhớ:

1. **`relative` ở cha** — thiếu nó, `absolute` sẽ neo vào `<body>` và badge bay lên góc màn hình.
2. **`z-10`** — để badge nằm trên ảnh khi ảnh phóng to.
3. **`{điều_kiện && (<JSX/>)}`** — cú pháp hiển thị có điều kiện đã học ở
   [Bài 03 mục 1.10](03-kien-thuc-nen.md).

</details>

**Bài 3.** Bạn muốn nút "Thêm vào ly" đổi màu theo trạng thái còn/hết hàng, nên viết:

```jsx
const mau = topping.inStock ? "green" : "gray";
<button className={`bg-${mau}-500 text-white px-3 py-2`}>Thêm vào ly</button>
```

Chạy lên thì nút **không có màu nền**. Giải thích vì sao và sửa lại cho đúng.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

**Nguyên nhân:** Tailwind quét file bằng **regex trên text thô**, nó không chạy JavaScript. Trong
file chỉ tồn tại chuỗi `` `bg-${mau}-500` `` — không có chuỗi `bg-green-500` hay `bg-gray-500` nào
cả. Vì thế hai luật CSS đó **không bao giờ được sinh ra**, `className` cuối cùng trỏ tới class
không tồn tại.

**Cách sửa 1 — viết đủ hai chuỗi (khuyên dùng):**

```jsx
// đoạn này bạn tự viết thêm
<button
  className={`${
    topping.inStock ? "bg-green-500" : "bg-gray-500"
  } text-white px-3 py-2`}
>
  Thêm vào ly
</button>
```

**Cách sửa 2 — object tra cứu, gọn khi có nhiều trạng thái:**

```jsx
const MAU_NUT = { con: "bg-green-500", het: "bg-gray-500" };

<button className={`${MAU_NUT[topping.inStock ? "con" : "het"]} text-white px-3 py-2`}>
  Thêm vào ly
</button>
```

Cả hai cách đều đảm bảo chuỗi `bg-green-500` và `bg-gray-500` **xuất hiện nguyên văn** trong file
`.js` nằm trong `./src/**` → Tailwind quét thấy → sinh CSS.

**Cách 3 (chỉ khi bất khả kháng):** khai `safelist` trong `tailwind.config.js`. Cách này khiến CSS
phình ra và dễ quên cập nhật — coi nó là phương án cuối.

</details>

---

## 📌 Tóm tắt

- **Tailwind = utility-first**: ghép class nhỏ ngay trên thẻ thay vì đặt tên class rồi viết file
  `.css`. Đổi lại: JSX khó đọc và class lặp lại nhiều.
- `tailwind.config.js:3` khai `content: ["./src/**/*.{js,jsx,ts,tsx}"]` — Tailwind **chỉ sinh CSS
  cho class nó quét thấy nguyên văn**, nên **class ghép chuỗi động sẽ bị xoá mất**.
- Ba dòng `@tailwind base/components/utilities` (`index.css:1-3`) là cửa ngõ của Tailwind vào dự án;
  `base` reset luôn style mặc định của `<h1>`, `<img>`…
- Quy tắc số: **số × 4 = pixel** (`p-3` = 12px, `mt-10` = 40px).
- **Mobile-first**: class không tiền tố là mặc định cho điện thoại; `sm:` `md:` `lg:` chỉ ghi đè
  **khi màn hình to hơn**. `col-span-12 lg:col-span-8` = full ngang trên mobile, 8/12 từ 1024px.
- **Lưới 12 cột** chia hết cho 2/3/4/6 → mọi tỉ lệ đều tròn; lưới lồng nhau **đếm lại từ đầu**.
- **Giá trị tuỳ ý** `[...]`: dùng `_` thay dấu cách (`shadow-[inset_0_1px_2px_rgba(0,0,0,0.1)]`).
- Dự án rải màu `#D9A953` **97 lần** thay vì khai vào `theme.extend.colors` — sẽ sửa ở
  [Bài 34](34-refactor-du-an.md). Hai bẫy khác: chuỗi class 375 ký tự dán 4 lần ở
  `MyAccountLayout.js`, và luật `iframe { display: none }` toàn cục ở `index.css:5-7`.

**Từ khoá tra cứu thêm:** `tailwind utility-first`, `tailwind content configuration`,
`tailwind arbitrary values`, `tailwind responsive design`, `tailwind group-hover`,
`tailwind apply directive`, `tailwind safelist`, `Tailwind CSS IntelliSense`

➡️ **Bài tiếp theo:** [18 — Tầng gọi API với axios](18-tang-api-axios.md) — giao diện đã đẹp, nhưng
dữ liệu topping vẫn là mảng cứng. Bài sau bạn sẽ tự viết `src/api/topping.js` để kéo dữ liệu thật
từ API Topping bạn đã dựng ở phần backend.
