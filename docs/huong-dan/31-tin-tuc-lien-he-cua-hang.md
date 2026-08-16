# Bài 31 — Tin tức, liên hệ, cửa hàng và slider trang chủ

> **Phần 5 · Bài củng cố** — Thời lượng ước tính: **~75 phút**
> ⬅️ Bài trước: [30 — Bình luận, đánh giá sao và yêu thích sản phẩm](30-binh-luan-danh-gia-yeu-thich.md) · Bài sau: [32 — Trang quản trị: CRUD, phân trang, upload ảnh Cloudinary](32-trang-quan-tri.md) ➡️
> 🏠 [Mục lục](README.md)

---

## 🎯 Sau bài này bạn sẽ

- Nhận ra **một khuôn mẫu** (list page → content component + prop tiêm hàm API) lặp lại y hệt giữa Sản phẩm và Tin tức — học một lần, đọc được cả bốn chức năng.
- Đọc trôi bốn khối chức năng còn lại của phía khách hàng: **Tin tức**, **Liên hệ**, **Cửa hàng**, **Slider/Trang chủ**.
- Hiểu form liên hệ chạy bằng `react-hook-form` + `yup` và gửi `POST /api/contact` **không cần đăng nhập**.
- Biết `react-slick` dựng slider ra sao, `NextArrow`/`PrevArrow` gắn vào bằng cách nào.
- Điểm mặt 7 khối của trang chủ, chỉ ra **khối nào lấy dữ liệu thật, khối nào hardcode**, và **đếm được số request** khi mở trang chủ.
- Tự tay thêm khối "Topping bán chạy" vào trang chủ theo khuôn `HomeProducts`.

## 📋 Cần chuẩn bị

- Đã hoàn thành [Bài 30](30-binh-luan-danh-gia-yeu-thich.md), và đặc biệt là [Bài 25 — Danh sách sản phẩm](25-danh-sach-san-pham.md) (bài này liên tục đối chiếu ngược về đó).
- Nhớ lại khuôn "trang tiêm hàm API xuống component con" đã gặp ở [Bài 18](18-tang-api-axios.md).
- Backend + frontend đang chạy; DB đã có ít nhất vài bài tin tức, một cửa hàng, một slider.

> 💡 Đây là **bài củng cố**. Sẽ có ít khái niệm mới, nhiều "à, chỗ này giống hệt chỗ kia".
> Mục tiêu là để bạn tin rằng: khi đã nắm một chức năng, ba chức năng còn lại chỉ là **copy khuôn mẫu**.

---

## 1. Một khuôn mẫu, lặp lại khắp nơi

Ở [Bài 25](25-danh-sach-san-pham.md) bạn đã học cách dự án hiển thị danh sách sản phẩm. Điểm mấu chốt (đã nêu ở phần "sự thật" của giáo trình): **phía khách hàng gần như KHÔNG dùng Redux để lấy dữ liệu**. Thay vào đó, dự án dùng một mẫu **dependency injection** (tiêm phụ thuộc):

```
Trang (pages/user/*)  →  tiêm MỘT HÀM GỌI API xuống làm prop  →  Component con
                                                                  gọi hàm đó trong useEffect
                                                                  giữ kết quả bằng useState
```

> 📖 **Thuật ngữ:** *dependency injection* — thay vì để component con tự quyết gọi API nào,
> ta "tiêm" hàm gọi API từ bên ngoài vào qua prop. Nhờ vậy **một** component con dùng lại được
> cho **nhiều** trang: trang danh sách, trang lọc theo danh mục, trang tìm kiếm.

Bạn đã thấy nó với sản phẩm:

```
ProductPage.js       →  <ProductContent getProducts={getAll} ... />
ProductByCate.js     →  <ProductContent getProducts={getProductByCate} ... />
ProductSearchPage.js →  <ProductContent getProducts={search} ... />
```

Chức năng **Tin tức** dùng **chính xác** khuôn đó, chỉ đổi tên:

```
NewsPage.js       →  <NewsContent getNews={getAll} ... />
NewsByCatePage.js →  <NewsContent getNews={getNewsById} ... />
```

Nắm được điều này, bạn đọc phần Tin tức nhanh gấp đôi. Ta bắt đầu.

---

## 2. Tin tức — soi code (đối chiếu song song với Sản phẩm)

### 2.1. Bảng đối chiếu Sản phẩm ↔ Tin tức (điểm nhấn của cả bài)

Đây là bảng đáng in ra dán lên tường. Cột giữa là những gì bạn đã học ở Bài 25–26; cột phải là bài này.

| Vai trò | Sản phẩm (Bài 25–26) | Tin tức (bài này) |
|---|---|---|
| Trang danh sách | `ProductPage` | `NewsPage` |
| Component nội dung tái dùng | `ProductContent` | `NewsContent` |
| Prop tiêm hàm API | `getProducts` | `getNews` |
| Trang lọc theo danh mục | `ProductByCate` | `NewsByCatePage` |
| Trang chi tiết | `ProductDetailPage` | `NewsDetail` |
| Khối "liên quan" | `ProductRelated` | `NewsRelated` |
| Thanh chọn danh mục | `NavProduct` | `NavNews` |
| Hàm API lấy tất cả | `api/product.js → getAll` | `api/news.js → getAll` |
| Hàm API lọc danh mục | `getProductByCate` | `getNewsById` |
| Hàm API liên quan | `getProductsRelated` | `relatedPost` |
| Route chi tiết (dùng slug) | `/san-pham/:slug` | `/bai-viet/:slug` |
| Quan hệ dữ liệu | `Product.categoryId → Category` | `News.category → CateNews` |
| Số item / trang | `limit = 9` | `limit = 8` |
| Có gọi thêm rating (N+1) | **Có** (`getAvgStar`, `getTotalRating`) | **Không** |

Hai khác biệt đáng nhớ nhất: tên trường quan hệ (**`categoryId`** với sản phẩm, nhưng **`category`** với tin tức), và tin tức **không** phải gánh vòng N+1 gọi rating cho từng phần tử.

### 2.2. `NewsPage` — trang danh sách chỉ vài dòng

`yotea-fe/src/pages/user/NewsPage.js:8-22`

```jsx
const NewsPage = () => {
  const { page } = useParams();

  useEffect(() => {
    updateTitle("Tin tức");
  }, []);

  return (
    <>
      <NavNews slug={""} />

      <NewsContent getNews={getAll} page={Number(page) || 1} url="tin-tuc" />
    </>
  );
};
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 9 | `const { page } = useParams()` | Lấy số trang từ URL `/tin-tuc/page/:page` |
| 17 | `<NavNews slug={""} />` | Thanh chọn danh mục; `slug=""` để ô "Tất cả" sáng lên |
| 19 | `getNews={getAll}` | **Tiêm** hàm `getAll` (từ `api/news`) xuống `NewsContent` |
| 19 | `page={Number(page) || 1}` | Route `/tin-tuc` (không có `:page`) → `page` là `undefined` → về `1` |

### 2.3. `NewsContent` — trái tim, giống hệt `ProductContent`

`yotea-fe/src/components/user/NewsContent.js:8-28`

```jsx
const NewsContent = ({ page, getNews, parameter, url }) => {
  const [emptyNews, setEmptyNews] = useState(false);
  const [news, setNews] = useState();
  const [totalNews, setTotalNews] = useState(1);

  const limit = 8;
  const totalPage = Math.ceil(totalNews / limit);
  page = page < 1 ? 1 : page > totalPage ? totalPage : page;
  const start = (page - 1) * limit > 0 ? (page - 1) * limit : 0;

  useEffect(() => {
    const getDataNews = async () => {
      const { data } = await getNews(0, 0, parameter);
      setEmptyNews(!data.length ? true : false);
      setTotalNews(data.length);

      const { data: newsData } = await getNews(start, limit, parameter);
      setNews(newsData);
    };
    getDataNews();
  }, [page, parameter]);
```

Đây **đúng công thức phân trang của cả dự án** bạn đã mổ ở Bài 25:

- `limit = 8` cố định (sản phẩm là 9).
- `totalPage` = trần của `tổng / limit`.
- Kẹp `page` vào `[1, totalPage]` — lưu ý dòng 15 **gán đè trực tiếp lên prop** `page` (mutate prop — anti-pattern y hệt `ProductContent`).
- Gọi **2 lượt API**: lượt 1 `getNews(0, 0, parameter)` để **đếm tổng** (đây là lý do phải kéo hết bản ghi về), lượt 2 `getNews(start, limit, parameter)` lấy đúng một trang.

> ⚠️ **Chỗ này dự án làm chưa chuẩn (lặp lại từ Bài 25):** tải **toàn bộ** bài viết chỉ để lấy `data.length` rồi lại tải một trang là **lãng phí**. Backend nên trả tổng số qua header (`X-Total-Count`) để chỉ cần **một** request. Ta sẽ bàn hướng sửa ở [Bài 34](34-refactor-du-an.md).

Mỗi bài viết render một thẻ, tiêu đề bấm được về `/bai-viet/${item.slug}` (`NewsContent.js:44-77`):

```jsx
<Link to={`/bai-viet/${item.slug}`} ... >   {/* ← route chi tiết dùng slug */}
```

### 2.4. `api/news.js` — ba hàm khớp ba cách dùng

`yotea-fe/src/api/news.js:7-22`

```js
export const getAll = (start = 0, limit = 0) => {
  let url = `/${DB_NAME}/?_sort=createdAt&_order=desc`;
  if (limit) url += `&_start=${start}&_limit=${limit}`;
  return instance.get(url);
};

export const getNewsById = (start = 0, limit = 0, category) => {
  let url = `/${DB_NAME}/?_sort=createdAt&_order=desc&category=${category}`;
  if (limit) url += `&_start=${start}&_limit=${limit}`;
  return instance.get(url);
};

export const get = (slug) => {
  const url = `/${DB_NAME}/${slug}/?_expand=category`;
  return instance.get(url);
};
```

- `getAll` — lấy tất cả (dùng cho `/tin-tuc`).
- `getNewsById` — lọc theo `category=<id danh mục>` (dùng cho `/tin-tuc/:slug`). Để ý tên tham số là `category` — vì trong `models/news.js` trường tham chiếu tên là `category` chứ không phải `categoryId`.
- `get(slug)` — lấy **một** bài theo slug, kèm `_expand=category` để "nở" id danh mục thành object (nhắc lại `_expand` ở [Bài 09](09-bo-loc-query.md)).

### 2.5. `NewsByCatePage` — đổi slug thành id rồi tiêm xuống

`yotea-fe/src/pages/user/NewsByCatePage.js:9-34`

```jsx
const NewsByCatePage = () => {
  const { slug, page } = useParams();
  const [cateId, setCateId] = useState();

  useEffect(() => {
    const getCateId = async () => {
      const { data } = await get(slug);
      setCateId(data._id);
      updateTitle(`${data.name}`);
    };
    getCateId();
  }, [slug]);

  return (
    <>
      <NavNews slug={slug || ""} />

      <NewsContent
        getNews={getNewsById}
        page={Number(page) || 1}
        url={`tin-tuc/${slug}`}
        parameter={cateId}
      />
    </>
  );
};
```

Luồng: URL cho **slug danh mục** → gọi `get(slug)` của `api/categoryNews` lấy `_id` → truyền `_id` xuống `NewsContent` qua prop `parameter` → `NewsContent` chuyển tiếp vào `getNewsById(start, limit, parameter)`. Giống hệt `ProductByCate`.

> ⚠️ **Bug lặp lại từ `ProductByCate`:** lần render đầu tiên `cateId` là `undefined`. `NewsContent` **không guard** `parameter` nên vẫn bắn `getNewsById(0, 0, undefined)` → query thành `...&category=undefined` → backend lọc rỗng hoặc lỗi. May là ngay sau đó `setCateId` chạy lại effect nên UI vẫn hiện đúng, nhưng có một request thừa/ lỗi lọt vào console. **Cách đúng:** guard `if (!parameter) return;` ở đầu `useEffect`.

### 2.6. `NewsDetail` — trang chi tiết bài viết

`yotea-fe/src/pages/user/NewsDetail.js:19-46`

```jsx
useEffect(() => {
  const getNews = async () => {
    const { data } = await get(slug);
    setNews(data);
    updateTitle(`${data.title}`);
  };
  getNews();
}, [slug]);
```

```jsx
<Link to={`/tin-tuc/${news?.category.slug}`} className="uppercase text-sm">
  {news?.category.name}
</Link>
<h1 className="uppercase font-bold text-xl py-1">{news?.title}</h1>
...
<div className="leading-relaxed text-justify">{news?.content}</div>
```

Trang này còn dựng **hai** khối phụ ở cuối: `NewsRelated` (bài viết liên quan) và `SidebarNews` (chuyên mục + bài viết mới) — `NewsDetail.js:74,78`.

> ⚠️ **Ba lỗi thật trong file này (nêu để bạn biết mà tránh):**
> 1. **Optional-chaining nửa vời:** `news?.category.slug` (dòng 33) chỉ hỏi `news?` ở tầng đầu. Nếu `news` đã có nhưng backend **không** populate `category` thì `.slug` nổ `TypeError`. Đúng phải là `news?.category?.slug`.
> 2. **Nội dung render dưới dạng text thuần** (`{news?.content}`, dòng 46). Admin nhập xuống dòng hay HTML thì mất định dạng / hiện thẻ thô.
> 3. **Link chia sẻ MXH hỏng:** dòng 50/58/66 viết `href="...?u=${window.location.href}/"` bằng **dấu nháy kép**, không phải backtick → chuỗi `${window.location.href}` bị gửi nguyên văn. (Lỗi y hệt ở `ProductDetailPage`.)

### 2.7. `NavNews` và `SidebarNews` — tự gọi API danh mục

`yotea-fe/src/components/user/NavNews.js:7-16`

```jsx
const NavNews = ({ slug }) => {
  const [cateNews, setCateNews] = useState();

  useEffect(() => {
    const getCate = async () => {
      const { data } = await getAll();
      setCateNews(data);
    };
    getCate();
  }, []);
```

`NavNews` tự gọi `getAll()` của `api/categoryNews` để dựng danh sách danh mục. `SidebarNews` (`SidebarNews.js:10-22`) thì gọi **cả hai**: `getAll()` (danh mục) và `getAllNews(0, 10)` (10 bài mới nhất).

### 2.8. `NewsRelated` — và một link chết kinh điển

`yotea-fe/src/components/user/NewsRelated.js:11-17`

```jsx
useEffect(() => {
  const getNewsRelated = async () => {
    const { data } = await relatedPost(id, category, 0, 4);
    setNews(data);
  };
  getNewsRelated();
}, [id]);
```

`relatedPost(id, cateId, 0, 4)` lấy 4 bài **cùng danh mục, khác id** hiện tại (`api/news.js:51-55`, dùng `_id_ne`).

> ⚠️ **Link chết trong `NewsRelated.js:39-40`:**
> ```jsx
> <a href="/#/news/${item.id}" ...>{item.title}</a>
> ```
> Ba cái sai một lúc: (1) dùng nháy kép nên `${item.id}` không nội suy; (2) route thật là `/bai-viet/:slug` chứ không phải `/news/:id`; (3) Mongo dùng `_id` chứ không có `item.id`. Tiêu đề bài liên quan bấm vào **không đi đâu cả**. So sánh: cùng file đó, ảnh ở dòng 26 lại dùng đúng `` to={`/bai-viet/${item.slug}`} `` — chứng tỏ đây là code copy sót từ bản HTML cũ.

### 2.9. Quan hệ News → CateNews (backend)

`yotea-be/src/models/news.js:27-34`

```js
category: {
    type: ObjectId,
    ref: "CateNews"
},
status: {
    type: Number,
    default: 0,
}
```

`yotea-be/src/models/cateNews.js:3-15`

```js
const cateNewsSchema = new Schema({
    name: {
        type: String,
        required: true,
        unique: true
    },
    slug: {
        type: String,
        required: true,
        unique: true,
        lowercase: true,
    },
}, { timestamps: true });
```

Đúng cặp quan hệ như `Product → Category`: một bài viết (`News`) trỏ tới một danh mục tin (`CateNews`) qua trường `category` kiểu `ObjectId` + `ref`. Nhờ `ref` mà `_expand=category` mới nở được.

---

## 3. Liên hệ — form gửi không cần đăng nhập

### 3.1. Schema `yup` và `onSubmit`

`yotea-fe/src/pages/user/ContactPage.js:10-28`

```js
const schema = yup.object().shape({
  name: yup.string().required("Vui lòng nhập họ tên"),
  store: yup.string().required("Vui lòng chọn chi nhánh phản hồi"),
  content: yup.string().required("Vui lòng nhập nội dung phản hồi"),
  confirm: yup
    .bool()
    .oneOf([true], "Vui lòng đồng ý với điều khoản của chúng tôi"),
  email: yup
    .string()
    .required("Vui lòng nhập email")
    .matches(/^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$/, "Email không đúng định dạng"),
  phone: yup
    .string()
    .required("Vui lòng nhập sdt")
    .matches(
      /(84|0[3|5|7|8|9])+([0-9]{8})\b/,
      "Số điện thoại không đúng định dạng"
    ),
});
```

Điểm hay: `confirm` là một **checkbox điều khoản**, ràng buộc `yup.bool().oneOf([true], ...)` — nghĩa là *phải tích vào* mới hợp lệ.

`yotea-fe/src/pages/user/ContactPage.js:40-48`

```js
const onSubmit = async ({ confirm, ...dataInput }) => {
  try {
    await add(dataInput);
    toast.success("Gửi liên hệ thành công");
    reset();
  } catch (error) {
    toast.error("Đã có lỗi xảy ra, vui lòng thử lại");
  }
};
```

Chú ý mẹo `({ confirm, ...dataInput })` — **bóc `confirm` ra rồi vứt đi**, chỉ gửi phần còn lại lên server. Không ai cần lưu "người dùng có tick ô đồng ý hay không" vào DB. (Đây là điều `RegisterPage` **quên** làm với trường `confirm` mật khẩu — xem Bài 23.)

Danh sách cửa hàng cho `<select>` được nạp trong `useEffect` (`ContactPage.js:50-57`) bằng `getAll()` của `api/store` — lại là mẫu "tự fetch, tự giữ state".

### 3.2. `POST /api/contact` — công khai theo chủ ý

`yotea-fe/src/api/contact.js:17-20`

```js
export const add = (contact) => {
  const url = `/${DB_NAME}`;
  return instance.post(url, contact);
};
```

Khác với `add` của product/news/store (đều đính `Authorization: Bearer <token>`), `add` của contact **không** kèm token. Đây là **chủ ý**: khách vãng lai phải gửi được liên hệ. Ở backend, route `POST /contacts` cũng để công khai (xem bảng phân quyền [Bài 33](33-ra-soat-bao-mat.md)).

### 3.3. Model contact — có trường `status`

`yotea-be/src/models/contact.js:20-29`

```js
    store: {
        type: ObjectId,
        ref: "Store",
        required: true,
    },
    status: {
        type: Number,
        default: 0,
    }
}, { timestamps: true });
```

`store` tham chiếu tới `Store` (để admin biết liên hệ này gửi tới chi nhánh nào), còn `status` (mặc định `0`) là cờ để admin đánh dấu "đã xử lý / chưa xử lý" trong trang quản trị liên hệ.

### 3.4. `Iframe.js` và luật CSS ẩn iframe toàn cục

Bản đồ Google Maps của cửa hàng được lưu trong DB dưới dạng **cả đoạn HTML `<iframe>`**, rồi nhồi thẳng vào trang:

`yotea-fe/src/components/user/Iframe.js:1-8`

```jsx
const Iframe = ({ iframe }) => {
  return (
    <div
      className="h-full"
      dangerouslySetInnerHTML={{ __html: iframe || "" }}
    />
  );
};
```

> ⚠️ **Chỗ này dự án làm chưa chuẩn — hai vấn đề:**
>
> **(1) Luật `iframe { display: none }` toàn cục.** `yotea-fe/src/index.css:5-11`:
> ```css
> iframe {
>   display: none;
> }
>
> .store__list-map > div > iframe {
>   @apply w-full h-full rounded-lg;
> }
> ```
> Luật đầu ẩn **MỌI** iframe trên toàn site (chắc để chặn iframe lạ lọt vào nội dung tin tức). Luật sau định "bật lại" cho bản đồ cửa hàng — **nhưng nó chỉ đặt lại width/height/bo góc, KHÔNG đặt lại `display`**. Một phần tử `display: none` thì dù to bao nhiêu cũng không hiện. Kết quả: bản đồ cửa hàng rất dễ **không hiển thị**. Cách đúng là thêm `display: block` (hoặc `@apply block w-full h-full ...`) vào luật `.store__list-map > div > iframe`.
>
> **(2) `dangerouslySetInnerHTML` với dữ liệu từ DB = rủi ro XSS.** Nếu admin (hoặc kẻ chiếm được quyền admin) dán một đoạn `<script>` vào ô "map", nó sẽ chạy trên trình duyệt khách. Đây là lý do cái tên hàm React có chữ *dangerously*. Xem thêm ở [Bài 33](33-ra-soat-bao-mat.md).

---

## 4. Cửa hàng — danh sách + bản đồ

### 4.1. `StorePage` — tự fetch, không đụng Redux

`yotea-fe/src/pages/user/StorePage.js:12-37`

```jsx
useEffect(() => {
  const getStores = async () => {
    const { data } = await getAll();
    setGoogleMap({
      id: data[0]._id,
      map: data[0].map,
    });
    setStores(data);
  };
  getStores();
  updateTitle("Cửa hàng");
}, []);

const handleClickStore = async (id) => {
  const { data } = await get(id);
  setGoogleMap({
    id: data._id,
    map: data.map,
  });
};

const handleSearchStore = async (e) => {
  const keyword = e.target.value;
  const { data } = await search(keyword);
  setStores(data);
};
```

- Vào trang: `getAll()` lấy tất cả cửa hàng, chọn cửa hàng đầu tiên làm bản đồ mặc định.
- Bấm một cửa hàng: `handleClickStore(id)` → `get(id)` → đổi `googleMap` → `<Iframe>` render bản đồ mới.
- Gõ ô tìm: `handleSearchStore` gọi `search(keyword)`.

Bản đồ hiển thị ở cuối JSX (`StorePage.js:96`): `<Iframe iframe={googleMap?.map || ""} />`.

Lưu ý sư phạm: `StorePage` **không dùng** `storeSlice` — nó gọi thẳng `api/store`. Đúng như "sự thật" đã nói: **phía khách hàng gần như không dùng Redux để lấy dữ liệu**. `storeSlice` chỉ phục vụ trang **admin** quản lý cửa hàng.

> ⚠️ **Bug thật:** `data[0]._id` (dòng 16) **không guard**. Nếu DB chưa có cửa hàng nào → `data[0]` là `undefined` → `.­_id` nổ `TypeError` → trắng trang. Ngoài ra `handleSearchStore` gọi API **mỗi lần gõ phím** (không debounce), và nút bên phải ô tìm kiếm (`StorePage.js:57-62`) là `type="button"` **không có `onClick`**, lại còn dùng icon mũi tên xuống thay vì kính lúp.

### 4.2. `api/store.js` — thêm hàm `search` dùng `_like`

`yotea-fe/src/api/store.js:6-20`

```js
export const getAll = (start = 0, limit = 0) => {
  let url = `/${DB_NAME}/?_sort=createdAt&_order=desc`;
  if (limit) url += `&_start=${start}&_limit=${limit}`;
  return instance.get(url);
};

export const search = (keyword) => {
  const url = `/${DB_NAME}/?name_like=${keyword}&_sort=createdAt&_order=desc`;
  return instance.get(url);
};

export const get = (id) => {
  const url = `/${DB_NAME}/${id}`;
  return instance.get(url);
};
```

`search` dùng bộ lọc `name_like=<từ khoá>` — chính là cú pháp `*_like` bạn học ở [Bài 09](09-bo-loc-query.md). Backend biến nó thành truy vấn `RegExp` không phân biệt hoa thường.

### 4.3. Model store và `storeSlice`

`yotea-be/src/models/store.js:3-33` khai báo cửa hàng gồm `name`, `image`, `address`, `phone`, `timeStart`, `timeEnd`, `map` — tất cả `required`. Trường `map` chính là đoạn HTML iframe bản đồ.

`yotea-fe/src/redux/storeSlice.js:9-19` (dùng ở admin):

```js
export const getStores = createAsyncThunk(
  "store/getStores",
  async ({ start, limit }) => {
    const { data } = await getAll();
    const totalStore = data.length;

    const { data: storeList } = await getAll(start, limit);

    return { totalStore, storeList };
  }
);
```

Đúng khuôn thunk quen thuộc: gọi `getAll()` **hai lần** — lần một đếm tổng, lần hai lấy một trang. `extraReducers` chỉ xử lý case `fulfilled` cho `getStores/addStore/updateStore/deleteStore` (`storeSlice.js:41-64`) — không có `pending`/`rejected`, đúng thói quen của mọi slice trong dự án.

---

## 5. Slider và trang chủ

### 5.1. `HomePage` — chỉ là bảy khối xếp chồng

`yotea-fe/src/pages/user/HomePage.js:16-26`

```jsx
return (
  <>
    <HomeBanner />
    <HomeCategory />
    <HomeWhy />
    <HomeProducts />
    <HomeNews />
    <HomeFeedback />
    <HomeShow />
  </>
);
```

`HomePage` **không** có state, **không** gọi API. Mỗi khối con tự lo dữ liệu của mình. Ta điểm danh từng khối:

| # | Khối | Lấy dữ liệu từ đâu | Số request |
|---|---|---|---|
| 1 | `HomeBanner` | `useGetSlidersQuery` (RTK Query) → `GET /slider` | 1 |
| 2 | `HomeCategory` | `useGetCatesProductQuery` (RTK Query) → `GET /category` | 1 |
| 3 | `HomeWhy` | **HARDCODE 100%** (ảnh + chữ nhúng cứng) | 0 |
| 4 | `HomeProducts` | `getAll(0, 8)` + `getAvgStar` cho **từng** sản phẩm | 1 + 8 = 9 |
| 5 | `HomeNews` | `getAll(0, 4)` của `api/news` | 1 |
| 6 | `HomeFeedback` | **HARDCODE 100%** (3 lời khen nhúng cứng) | 0 |
| 7 | `HomeShow` | **HARDCODE 100%** (8 ảnh Instagram nhúng cứng) | 0 |

**Tổng số request khi mở trang chủ ≈ 12** (1 slider + 1 category + 9 products/rating + 1 news). Con số này phần lớn đến từ khối `HomeProducts` với vòng N+1 gọi rating.

### 5.2. `HomeBanner` + `react-slick` + arrow tuỳ biến

`yotea-fe/src/components/user/home/HomeBanner.js:6-14`

```jsx
const HomeBanner = () => {
    const { data: sliders } = useGetSlidersQuery("");

    const settings = {
        autoplay: true,
        infinite: true,
        nextArrow: <NextArrow onClick={() => {}} />,
        prevArrow: <PrevArrow onClick={() => {}} />
    }
```

> 📖 **Thuật ngữ:** *react-slick* — thư viện làm carousel/slider cho React. Bạn bọc các slide trong `<Slider {...settings}>`, truyền một object `settings` để bật autoplay, lặp vô hạn, và **thay mũi tên mặc định** bằng component của mình qua `nextArrow`/`prevArrow`.

Hai component mũi tên rất ngắn, `yotea-fe/src/components/admin/NextArrow.js:4-13`:

```jsx
const NextArrow = ({ onClick }) => {
  return (
    <button
      className="... right-6 group-hover:right-4 ..."   /* ← đã lược class Tailwind */
      onClick={onClick}
    >
      <FontAwesomeIcon icon={faChevronRight} />
    </button>
  );
};
```

`PrevArrow` y hệt, chỉ đổi `faChevronLeft` và nằm bên trái. Chúng đặt trong thư mục `components/admin/` nhưng được dùng lại ở cả 3 slider phía khách (`HomeBanner`, `HomeFeedback`, `HomeShow`).

> 💡 `onClick={() => {}}` truyền vào NextArrow/PrevArrow là **hàm rỗng**. react-slick sẽ tự **ghi đè** `onClick` thật khi nó clone phần tử mũi tên vào bên trong slider, nên hàm rỗng này chỉ là chỗ giữ chỗ — mũi tên vẫn bấm chuyển slide được.

### 5.3. `sliderSlice` và `api/slider` (RTK Query)

`yotea-fe/src/api/slider.js:44-58`

```js
export const sliderApi = createApi({
  reducerPath: "sliderApi",
  baseQuery: fetchBaseQuery({
    baseUrl: "http://localhost:8080/api",
  }),
  tagTypes: ["Slider"],
  endpoints: (builder) => ({
    getSliders: builder.query({
      query: () => `${DB_NAME}/?_sort=createdAt&_order=desc`,
      providesTags: ["Slider"],
    }),
  }),
});

export const { useGetSlidersQuery } = sliderApi;
```

Đây là một trong **số ít** chỗ phía khách hàng dùng **RTK Query** (nhắc lại [Bài 22](22-rtk-query.md)): `HomeBanner` gọi `useGetSlidersQuery`, `HomeCategory` gọi `useGetCatesProductQuery`. Còn `sliderSlice.js` (createAsyncThunk kiểu cũ, `sliderSlice.js:8-11`) chủ yếu phục vụ trang **admin** quản lý slider.

> 💡 **Vì sao slider dùng RTK Query mà tin tức/sản phẩm phía khách lại không?** Không có lý do kỹ thuật nhất quán — dự án là bài tập lớn, tác giả trộn nhiều phong cách. Bạn cứ ghi nhận: **cùng một việc "lấy danh sách" mà dự án có tới ba cách làm** (axios trực tiếp, `createAsyncThunk`, RTK Query). Bài 34 sẽ bàn cách thống nhất.

### 5.4. Ba khối hardcode

`yotea-fe/src/components/user/home/HomeWhy.js:3` — cả ảnh nền lẫn 3 icon đều là URL Cloudinary nhúng cứng:

```jsx
<section style={{backgroundImage: 'url(https://res.cloudinary.com/levantuan/image/upload/v1642595093/fpoly/asm-js/bg_why_vzvhn6.jpg)'}} ... >
```

`yotea-fe/src/components/user/home/HomeFeedback.js:57-59` — ba "khách hàng" là chữ viết cứng:

```jsx
<p className="font-semibold text-gray-300 text-xl italic">
  Tường Vy
</p>
```

`HomeShow.js:24` — 8 ảnh Instagram cũng là URL Cloudinary nhúng cứng.

> ⚠️ **Hai điều cần nói thẳng:**
> 1. **Ba khối `HomeWhy` / `HomeFeedback` / `HomeShow` là nội dung tĩnh 100%.** Admin **không** sửa được qua trang quản trị — muốn đổi phải vào sửa code. Trên một website thật, những khối này phải lấy từ DB.
> 2. **Toàn bộ ảnh trỏ về Cloudinary `levantuan`** — đây là tài khoản của **tác giả gốc bài giảng (FPoly)**, không phải của dự án. Thậm chí "khách hàng" tên **Tường Vy** trùng đúng chuỗi bí mật JWT `"TuongVy"` hardcode ở backend (xem [Bài 33](33-ra-soat-bao-mat.md)) — dấu vết rõ ràng của việc copy nguyên si từ template gốc. Nếu tài khoản Cloudinary kia bị xoá, **mọi ảnh này biến mất**.

### 5.5. Một link chết trong header

Trong lúc soi trang chủ, để ý header dùng chung ở `WebsiteLayout`. `yotea-fe/src/pages/layouts/WebsiteLayout.js:298`:

```jsx
<Link to="/gio-hang" className="relative">   {/* badge giỏ hàng bản mobile */}
```

> ⚠️ **Link chết:** route giỏ hàng thật là `/cart` (`App.js:145`), **không** có route `/gio-hang`. Vì `App.js` chưa khai báo `errorElement`, bấm vào badge giỏ hàng trên **giao diện mobile** sẽ ra **trang trắng**. (Badge giỏ hàng bản desktop ở dòng 216 thì trỏ đúng `/cart`.)

---

## 6. 🛠️ Tự tay làm — thêm khối "Topping bán chạy" vào trang chủ

> Mục tiêu: cuối phần này trang chủ có thêm một slider "Topping bán chạy", chạy theo **đúng khuôn** `HomeProducts` + `react-slick`. Toàn bộ code dưới đây là **code bạn tự viết thêm** (dự án chưa có), dựng trên chức năng Topping bạn đã xây từ Bài 04–32.

### Bước 0 — Nhắc lại `api/topping.js` bạn đã viết

Từ mạch thực hành xuyên suốt, bạn đã có `yotea-fe/src/api/topping.js` theo đúng khuôn `api/product.js`. Nếu chưa, tạo nó (đây là **file bạn tự viết**):

```js
// yotea-fe/src/api/topping.js  ← file bạn tự tạo từ các bài trước
import instance from "./instance";

const DB_NAME = "toppings";

export const getAll = (start = 0, limit = 0) => {
  let url = `/${DB_NAME}/?_sort=createdAt&_order=desc`;
  if (limit) url += `&_start=${start}&_limit=${limit}`;
  return instance.get(url);
};
```

### Bước 1 — Tạo component `home/HomeTopping.js`

Tạo file MỚI `yotea-fe/src/components/user/home/HomeTopping.js`. Ta lấy khuôn `HomeProducts` (fetch một ít bản ghi trong `useEffect`) và khuôn `HomeShow` (bọc trong `<Slider>` của react-slick):

```jsx
// yotea-fe/src/components/user/home/HomeTopping.js  ← file bạn tự tạo
import { useEffect, useState } from "react";
import Slider from "react-slick";
import { getAll } from "../../../api/topping";
import { formatCurrency } from "../../../utils";
import NextArrow from "../../admin/NextArrow";
import PrevArrow from "../../admin/PrevArrow";

const HomeTopping = () => {
  const [toppings, setToppings] = useState();

  useEffect(() => {
    const getToppings = async () => {
      const { data } = await getAll(0, 8);   // lấy 8 topping mới nhất
      setToppings(data);
    };
    getToppings();
  }, []);

  const settings = {
    autoplay: true,
    infinite: true,
    slidesToShow: 4,
    slidesToScroll: 1,
    nextArrow: <NextArrow onClick={() => {}} />,
    prevArrow: <PrevArrow onClick={() => {}} />,
  };

  return (
    <section className="container max-w-6xl mx-auto py-9 px-3">
      <div className="text-center">
        <h2 className="uppercase text-[#D9A953] text-2xl font-semibold">
          TOPPING BÁN CHẠY
        </h2>
        <p>Thêm chút topping cho ly trà sữa của bạn thêm trọn vị.</p>
      </div>

      <div className="mt-5 group">
        <Slider {...settings}>
          {toppings?.map((item, index) => (
            <div className="px-2" key={index}>
              <div
                style={{ backgroundImage: `url(${item.image})` }}
                className="bg-cover bg-center pt-[100%] rounded-xl"
              />
              <div className="text-center py-2">
                <p className="font-semibold text-lg">{item.name}</p>
                <p className="text-sm">{formatCurrency(item.price)}</p>
              </div>
            </div>
          ))}
        </Slider>
      </div>
    </section>
  );
};

export default HomeTopping;
```

**Vì sao viết vậy:**

- `getAll(0, 8)` — dùng đúng chữ ký hàm quen thuộc `(start, limit)`, y như `HomeProducts.js:24` gọi `getAll(0, 8)`.
- `settings` copy từ `HomeShow.js:6-13` (có `slidesToShow`) để nhiều topping hiện cùng lúc.
- Bọc slider trong `<div className="group">` để hai mũi tên `NextArrow`/`PrevArrow` (vốn dùng class `group-hover:visible`) hiện ra khi rê chuột.
- `formatCurrency(item.price)` — tái dùng helper ở `utils/index.js` (Bài 03).

### Bước 2 — Ghép vào `HomePage`

Mở `yotea-fe/src/pages/user/HomePage.js`, thêm import và chèn khối vào cây JSX (ví dụ ngay sau `HomeProducts`):

```jsx
// thêm dòng import cùng nhóm các HomeXxx khác
import HomeTopping from "../../components/user/home/HomeTopping";

// ... trong phần return, chèn giữa HomeProducts và HomeNews:
<HomeProducts />
<HomeTopping />   {/* ← khối mới của bạn */}
<HomeNews />
```

---

## 7. ✅ Kiểm chứng kết quả

**1) Chạy dự án** (đứng ở `yotea-fe`):

```bash
npm start
```

**2) Bốn chức năng cũ** — mở lần lượt và quan sát:

- `http://localhost:3000/#/tin-tuc` → lưới bài viết, phân trang chạy được, bấm một bài → sang `/#/bai-viet/<slug>`.
- `http://localhost:3000/#/lien-he` → điền form, **chưa tick ô đồng ý** thì bấm gửi phải thấy lỗi *"Vui lòng đồng ý với điều khoản..."*; điền đủ + tick → toast *"Gửi liên hệ thành công"*.
- `http://localhost:3000/#/cua-hang` → danh sách cửa hàng bên trái; bấm một cửa hàng → bản đồ bên phải đổi theo (nếu bản đồ không hiện, nhớ lại lỗi CSS `iframe { display: none }` ở mục 3.4).
- `http://localhost:3000/` → banner tự trượt, các khối xếp đúng thứ tự.

**3) Khối Topping mới** — ở trang chủ, kéo tới khối *"TOPPING BÁN CHẠY"*: phải thấy slider topping tự chạy, rê chuột hiện mũi tên trái/phải.

**4) Đếm request** — mở DevTools tab **Network**, lọc `Fetch/XHR`, tải lại trang chủ. Bạn phải thấy khoảng **12 request** (chưa tính khối Topping mới): 1 slider, 1 category, 1 products, 8 rating, 1 news. Khối Topping thêm **1** request nữa (`GET /api/toppings`).

Ví dụ JSON khi gọi thử `GET http://localhost:8080/api/store` bằng trình duyệt:

```json
[
  {
    "_id": "6650...",
    "name": "Yotea Cầu Giấy",
    "address": "...",
    "phone": "0987...",
    "timeStart": "08:00",
    "timeEnd": "22:00",
    "map": "<iframe src=\"https://www.google.com/maps/embed?...\"></iframe>"
  }
]
```

---

## 8. 🐞 Lỗi thường gặp

| Thông báo / hiện tượng | Nguyên nhân | Cách xử lý |
|---|---|---|
| Trang cửa hàng trắng xoá | DB chưa có cửa hàng nào → `data[0]._id` nổ `TypeError` (`StorePage.js:16`) | Thêm ít nhất 1 cửa hàng qua trang admin, hoặc guard `data[0] &&` |
| Bản đồ cửa hàng không hiện | Luật `iframe { display: none }` toàn cục ẩn cả bản đồ (`index.css:5`) | Thêm `display: block` cho `.store__list-map > div > iframe` |
| Tin theo danh mục lần đầu trống | `cateId` còn `undefined`, `NewsContent` gọi API với `category=undefined` | Guard `if (!parameter) return;` trong `useEffect` |
| Bấm tiêu đề "bài viết liên quan" không đi đâu | `NewsRelated.js:40` dùng nháy kép + route sai `/news/${item.id}` | Sửa thành `` <Link to={`/bai-viet/${item.slug}`}> `` |
| Slider Topping không chạy / lỗi `Slider is not defined` | Chưa `import Slider from "react-slick"` hoặc thiếu CSS của slick | Kiểm tra import; đảm bảo CSS slick đã nhúng ở `index.js` như các slider khác |
| `Cannot read properties of undefined (reading 'slug')` ở NewsDetail | `news.category` chưa populate, mà code viết `news?.category.slug` | Dùng optional-chaining đủ tầng: `news?.category?.slug` |

---

## 9. 📝 Bài tập

**Bài 1.** Dựa vào bảng đối chiếu ở mục 2.1, hãy liệt kê **ba khác biệt** giữa `ProductContent` và `NewsContent`, và giải thích vì sao tin tức **không** cần vòng gọi rating N+1.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Ba khác biệt: (1) `limit = 9` (sản phẩm) so với `limit = 8` (tin tức); (2) prop tiêm hàm tên `getProducts` so với `getNews`; (3) `ProductContent` có thêm vòng `for await` gọi `getAvgStar` + `getTotalRating` cho từng sản phẩm, `NewsContent` thì không.

Tin tức không cần rating vì **bài viết không có tính năng chấm sao**. Rating (đánh giá sao) chỉ gắn với sản phẩm — nó là quan hệ `Rating → Product`, không có `Rating → News`. Vì thế `NewsContent` chỉ tốn đúng **2 request/trang** (đếm tổng + lấy trang), còn `ProductContent` tốn `2 + 2×9 = 20 request/trang`.

</details>

**Bài 2.** Form liên hệ dùng `onSubmit = async ({ confirm, ...dataInput }) => { await add(dataInput); }`. Nếu ai đó đổi thành `await add({ confirm, ...dataInput })` (gửi cả `confirm`) thì có sao không? Model `contact` sẽ làm gì với trường `confirm`?

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Gửi kèm `confirm` **không gây lỗi**, nhưng thừa. `models/contact.js` chỉ khai báo `name`, `content`, `email`, `phone`, `store`, `status`. Mongoose chạy ở **strict mode** mặc định nên **âm thầm loại bỏ** trường `confirm` không có trong schema — y hệt trường hợp `CheckoutPage` gửi `sizeId`/`toppingId` mà `orderDetail` bỏ qua (xem Bài 28).

Dù không lỗi, gửi `confirm` vẫn là **rò rỉ dữ liệu không cần thiết** qua mạng. Cách bóc `confirm` ra rồi vứt (`{ confirm, ...dataInput }`) là chuẩn hơn — và đó chính là điều `RegisterPage` quên làm với `confirm` mật khẩu.

</details>

**Bài 3.** (khó hơn) Hãy sửa khối Topping ở phần "Tự tay làm" để **chỉ hiện khi có ít nhất 1 topping** (tránh render một slider rỗng lúc mới tải trang), đồng thời mỗi topping bấm vào dẫn tới trang chi tiết topping `/topping/:slug` mà bạn đã làm ở các bài trước.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Thêm điều kiện `&&` ngoài `<section>` và bọc mỗi ảnh trong `<Link>` (code bạn tự viết):

```jsx
// đầu return
if (!toppings?.length) return null;   // chưa có dữ liệu thì không vẽ gì

// ... trong .map, thay <div ảnh> bằng:
<Link
  to={`/topping/${item.slug}`}
  style={{ backgroundImage: `url(${item.image})` }}
  className="block bg-cover bg-center pt-[100%] rounded-xl"
/>
```

`return null` là cách "không render gì cả" trong React. Nhờ vậy, trước khi `getAll` trả về, khối Topping biến mất thay vì hiện slider trống. Nhớ `import { Link } from "react-router-dom";`. Route `/topping/:slug` phải khớp với route bạn đã khai trong `App.js` ở các bài trước.

</details>

---

## 📌 Tóm tắt

- Phía khách hàng lặp lại **một khuôn mẫu**: trang list **tiêm hàm gọi API** xuống một component nội dung tái dùng (`getProducts`/`getNews`), component con tự `useEffect` + `useState`. Tin tức là bản sao của Sản phẩm — chỉ khác `limit` (8 vs 9), tên trường quan hệ (`category` vs `categoryId`) và không có vòng rating.
- **Liên hệ**: form `react-hook-form` + `yup`, `POST /api/contact` **không cần token** (khách vãng lai gửi được); mẹo `{ confirm, ...dataInput }` để bỏ trường điều khoản; model `contact` có `status` để admin đánh dấu xử lý.
- **Cửa hàng**: `StorePage` gọi thẳng `api/store` (không dùng `storeSlice`); bản đồ lưu dạng HTML iframe render qua `Iframe.js` (`dangerouslySetInnerHTML`).
- ⚠️ Luật `iframe { display: none }` toàn cục trong `index.css` ẩn **mọi** iframe; luật scoped cho bản đồ cửa hàng quên đặt lại `display` → bản đồ dễ không hiện.
- **Trang chủ** gồm 7 khối; 4 khối lấy dữ liệu thật (Banner, Category, Products, News), 3 khối **hardcode 100%** (Why, Feedback, Show) với ảnh trỏ Cloudinary của **tác giả gốc**. Mở trang chủ tốn ~**12 request** (phần lớn do N+1 rating ở `HomeProducts`).
- **Slider** dựng bằng `react-slick`; mũi tên tuỳ biến `NextArrow`/`PrevArrow`; `HomeBanner`/`HomeCategory` là số ít chỗ phía khách dùng **RTK Query**.
- ⚠️ Link chết `/gio-hang` trong `WebsiteLayout` (route thật là `/cart`).

**Từ khoá tra cứu thêm:** `dependency injection react`, `react-slick settings`, `dangerouslySetInnerHTML XSS`, `mongoose strict mode`, `RTK Query createApi`

➡️ **Bài tiếp theo:** [32 — Trang quản trị: CRUD, phân trang, upload ảnh Cloudinary](32-trang-quan-tri.md) — lật sang phía admin, nơi tất cả dữ liệu bạn vừa xem được tạo ra, sửa và xoá.
