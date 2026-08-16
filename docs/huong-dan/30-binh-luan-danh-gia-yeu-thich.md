# Bài 30 — Bình luận, đánh giá sao và yêu thích sản phẩm

> **Phần 5 · Các luồng nghiệp vụ phía khách hàng** — Thời lượng ước tính: **~90 phút**
> ⬅️ Bài trước: [29 — Tài khoản của tôi: sửa thông tin, đổi mật khẩu, lịch sử đơn](29-tai-khoan-cua-toi.md) · Bài sau: [31 — Tin tức, liên hệ, cửa hàng và slider trang chủ](31-tin-tuc-lien-he-cua-hang.md) ➡️
> 🏠 [Mục lục](README.md)

---

## 🎯 Sau bài này bạn sẽ

- Hiểu được **ba chức năng tương tác** của Yotea — bình luận, đánh giá sao, yêu thích — đều dựa trên **bảng nối** giữa `User` và `Product`.
- Đọc được luồng dữ liệu của từng chức năng: từ form → API → controller → model → hiển thị.
- Giải thích được vì sao frontend phải **tự ghép (join) thủ công** hai mảng comment và rating.
- Chỉ ra được **điểm số trung bình được tính ở đâu** (frontend, không phải backend) và vì sao cách đó gây lỗi **N+1 request**.
- Nhận diện **cơ chế kép** của "yêu thích": vừa lưu bảng `FavoritesProduct`, vừa tăng bộ đếm `favorites` trên `Product` — dẫn tới **hai nguồn sự thật** dễ lệch.
- Tự tay thêm nút "xoá bình luận của chính mình" và ràng buộc "mỗi người chỉ đánh giá một lần / một sản phẩm" ở **cả frontend lẫn backend**.

## 📋 Cần chuẩn bị

- Đã hoàn thành [Bài 29](29-tai-khoan-cua-toi.md), đặc biệt nắm chắc mẫu "trang truyền hàm API xuống component con" và cách đọc `selectAuth`.
- Nhớ lại [Bài 10 — Quan hệ & populate](10-quan-he-va-populate.md) (vì cả ba model đều dùng `ObjectId` + `ref`) và [Bài 25 — Danh sách sản phẩm](25-danh-sach-san-pham.md) (nơi phát sinh lỗi N+1 sẽ nói ở bài này).
- Backend và frontend đang chạy được; có sẵn 1 tài khoản khách thường để đăng nhập.

> 💡 Ba chức năng này rất giống nhau về cấu trúc (đều là bảng nối 2 khoá ngoại), nên khi
> hiểu một cái bạn gần như hiểu cả ba. Ta sẽ đi lần lượt và chỉ dừng lâu ở chỗ khác biệt.

---

## 1. Bức tranh tổng: ba bảng nối giống nhau

Cả ba chức năng đều lưu **mối quan hệ giữa một người dùng và một sản phẩm**. Về bản chất
dữ liệu chúng là **bảng nối n–n** (many-to-many) giữa `User` và `Product`, chỉ khác nhau
ở phần "nội dung kèm theo":

| Chức năng | Collection | Nội dung kèm theo | Bắt buộc đăng nhập? |
|---|---|---|---|
| Bình luận | `comments` | `content` (chuỗi văn bản) | ✅ (POST cần `requireSignin`) |
| Đánh giá sao | `ratings` | `ratingNumber` (số sao) | ✅ |
| Yêu thích | `favoritesproducts` | *(không có gì thêm)* | ✅ |

Sơ đồ quan hệ (trích từ [Bài 10](10-quan-he-va-populate.md)):

```
             +-----------+                 +-----------+
             |   User    |                 |  Product  |
             +-----+-----+                 +-----+-----+
                   |  userId (ObjectId, ref)     | productId (ObjectId, ref)
      +------------+------------+-----------------+------------+
      |            |            |                              |
      v            v            v                              |
 +---------+  +---------+  +------------------+                |
 | Comment |  | Rating  |  | FavoritesProduct |<---------------+
 +---------+  +---------+  +------------------+
   content     ratingNumber      (chỉ 2 khoá)
```

Điểm mấu chốt cần nhớ ngay từ đầu: **`Comment` và `Rating` là hai bảng tách rời**, nhưng
trên màn hình chi tiết sản phẩm chúng lại được hiển thị gộp thành **một "bình luận có
kèm số sao"**. Vì backend không gộp, **frontend phải tự ghép hai mảng lại** — đây là chỗ
dễ vỡ nhất của bài, ta sẽ mổ ở mục 3.

---

## 2. Chức năng BÌNH LUẬN

### 2.1. Sơ đồ luồng

```
[Người dùng đã đăng nhập]
        │  gõ nội dung + chọn số sao trong <CommentProduct>
        ▼
  onSubmit()  ──(1)──►  checkUserRating(userId, productId)   (đã có rating chưa?)
        │                     │
        │              chưa ──► AddRating(...)   /  đã có ──► update(...)
        │
        └──(2)──►  addComment({ productId, userId, content })
                          │  POST /api/comments/:userId  (requireSignin + isAuth)
                          ▼
                  controllers/comment.js  →  new Comment(req.body).save()
                          │
                          ▼
                  onReRender(prev => !prev)   →  <CommentList> tải lại danh sách
```

Chú ý: **một lần bấm "Gửi đi" ghi vào HAI collection** — một bản ghi `Comment` và một bản
ghi `Rating` (hoặc cập nhật rating cũ). Đó là lý do form đặt trong cùng một component
`CommentProduct`.

### 2.2. Model `Comment`

`yotea-be/src/models/comment.js:1-20`

```js
import { Schema, model, ObjectId } from "mongoose";

const commentSchema = new Schema({
    userId: {
        type: ObjectId,
        ref: "User"
    },
    content: {
        type: String,
        required: true
    },
    productId: {
        type: ObjectId,
        ref: "Product"
    }
}, { timestamps: true });

commentSchema.index({'$**': 'text'});

export default model("Comment", commentSchema);
```

**Đọc từng trường:**

| Trường | Kiểu | Ghi chú |
|---|---|---|
| `userId` | `ObjectId` ref `User` | **Không** `required` — về lý thuyết lưu được bình luận "mồ côi" |
| `content` | `String`, `required` | Nội dung bình luận — trường bắt buộc duy nhất |
| `productId` | `ObjectId` ref `Product` | **Không** `required` |
| `createdAt`/`updatedAt` | `Date` | Sinh tự động nhờ `{ timestamps: true }` |

> ⚠️ **Chỗ này dự án làm chưa chuẩn — không có trường `status` để duyệt bình luận.**
> Model `Comment` **chỉ có 3 trường**, hoàn toàn **không có trường `status`** (kiểu "chờ
> duyệt / đã duyệt"). Nghĩa là mọi bình luận **hiển thị ngay lập tức**, không qua kiểm
> duyệt. Ở trang admin (`AdminCommentList`, xem mục 2.5) người quản trị **chỉ có thể xoá**
> chứ không "ẩn/duyệt" được. Với một trang bán hàng thật, đây là lỗ hổng nội dung: khách
> có thể spam quảng cáo, chửi bới, và admin chỉ dọn được sau khi nó đã hiện. Cách làm
> đúng: thêm trường `status` (0 = chờ duyệt, 1 = hiển thị), mặc định `0`, và chỉ trả về
> các bình luận `status: 1` cho phía khách.

### 2.3. Component `CommentProduct` — form gửi bình luận (phải đăng nhập)

`yotea-fe/src/components/user/CommentProduct.js:18-56` (phần logic; đã lược phần JSX form)

```jsx
const CommentProduct = ({ productId, onReRender, productData }) => {
  const { user } = useSelector(selectAuth);
  const [emptyCmt, setEmptyCmt] = useState(false);
  const {
    register,
    handleSubmit,
    formState: { errors },
    reset,
  } = useForm({ resolver: yupResolver(schema) });

  const onSubmit = async ({ star: ratingNumber, content }) => {
    try {
      // check user rating
      const { data } = await checkUserRating(user._id, productId);
      if (!data.length) {
        AddRating({
          userId: user._id,
          productId,
          ratingNumber,
        });
      } else {
        update({
          _id: data[0]["_id"],
          ratingNumber,
        });
      }

      addComment({
        productId,
        userId: user._id,
        content,
      })
        .then(() => toast.success("Bình luận thành công"))
        .then(() => reset())
        .then(() => onReRender((prev) => !prev));
    } catch (error) {
      toast.error("Có lỗi xảy ra, vui lòng thử lại");
    }
  };
```

**Đọc từng phần:**

| Dòng | Ý nghĩa |
|---|---|
| 19 | Lấy `user` từ Redux (`selectAuth`). Component này **giả định đã đăng nhập** — `user._id` được dùng ở nhiều nơi bên dưới. |
| 21-26 | `react-hook-form` + `yup` để kiểm tra form (nhắc lại [Bài 28](28-thanh-toan.md)). `schema` bắt buộc chọn sao và nhập nội dung. |
| 28 | Bóc tách + **đổi tên**: trường `star` của form được đổi thành `ratingNumber`. |
| 31-43 | **Chống đánh giá trùng:** gọi `checkUserRating(user._id, productId)` trước. Nếu người này **chưa** đánh giá (`!data.length`) thì `AddRating` mới; nếu **đã** đánh giá rồi thì `update` bản ghi cũ. (Đây chính là ràng buộc "một người một sao / một sản phẩm" — nhưng **chỉ ở phía frontend**, xem cảnh báo ở mục 3.4.) |
| 45-49 | Gửi bình luận: `addComment({ productId, userId, content })`. |
| 50-52 | Chuỗi `.then()`: báo thành công → `reset()` form → gọi `onReRender` để `CommentList` tải lại. |

> ⚠️ **`userId` do client tự khai.** `addComment` gửi lên `userId: user._id` **trong body**,
> và controller lưu thẳng `new Comment(req.body)` (xem mục 2.4). Backend **không** lấy id
> người viết từ token. Kẻ xấu đổi `userId` thành id người khác là **mạo danh bình luận**
> được. Cách đúng: backend gán `userId = req.auth._id` (lấy từ token đã ký), bỏ qua
> `userId` client gửi.

Về ràng buộc phải đăng nhập: `CommentProduct` **luôn dùng `user._id`**. Nếu `user` là
`null` (khách vãng lai) thì `user._id` ném `TypeError`. Trong thực tế component này chỉ
được render sau khi trang chi tiết đã xác định người dùng, nhưng đây là điểm mong manh —
form không tự chặn khách chưa đăng nhập bằng một thông báo tử tế.

### 2.4. Controller `create` — lưu thẳng `req.body`

`yotea-be/src/controllers/comment.js:3-13`

```js
export const create = async (req, res) => {
    try {
        const comment = await new Comment(req.body).save();
        res.json(comment);
    } catch (error) {
        res.status(400).json({
            message: "Thêm bình luận thất bại",
            error
        });
    }
};
```

Đúng như đã cảnh báo: `new Comment(req.body)` nhét nguyên dữ liệu client gửi lên. Route
bảo vệ nó là `yotea-be/src/routes/comment.js:8-12`:

```js
router.post("/comments/:userId", requireSignin, isAuth, create);
router.get("/comments/:id", read);
router.get("/comments", list);
router.put("/comments/:id/:userId", requireSignin, isAuth, update);
router.delete("/comments/:id/:userId", requireSignin, isAuth, remove);
```

`GET` công khai (ai cũng đọc bình luận), còn `POST/PUT/DELETE` cần `requireSignin, isAuth`
— **nhưng không có `isAdmin` và không kiểm chủ sở hữu** (ta gom vào phần Tổng kết ở mục 5).

### 2.5. Hiển thị & duyệt ở admin — `AdminCommentList`

`yotea-fe/src/components/admin/AdminCommentList.js:22-42`

```jsx
  const handleRemoveCmt = async (id) => {
    Swal.fire({
      title: "Bạn có chắc chắn muốn xóa không?",
      text: "Bạn không thể hoàn tác sau khi xóa!",
      icon: "warning",
      showCancelButton: true,
      confirmButtonColor: "#3085d6",
      cancelButtonColor: "#d33",
      confirmButtonText: "Yes, delete it!",
    }).then((result) => {
      if (result.isConfirmed) {
        remove(id)
          .then(() => {
            Swal.fire("Thành công!", "Đã xóa thành công.", "success");
          })
          .then(() =>
            setComments((prev) => prev?.filter((item) => item._id !== id))
          );
      }
    });
  };
```

Nhìn vào đây bạn **suy ra được thiết kế duyệt bình luận của dự án**: bảng admin chỉ có
đúng một hành động **"Delete"** (`AdminCommentList.js:117-122`), không có nút "Duyệt" hay
cột "Trạng thái". Kết hợp với việc model `Comment` không có trường `status`, kết luận:
**Yotea không có quy trình kiểm duyệt bình luận — chỉ có xoá sau khi đã hiển thị.** Sau
khi `remove(id)` xong, code cập nhật danh sách ngay tại client bằng `filter` (bỏ đúng
phần tử vừa xoá) mà không cần gọi lại API — một kỹ thuật "optimistic update" đơn giản.

---

## 3. Chức năng ĐÁNH GIÁ SAO

### 3.1. Sơ đồ luồng (đọc)

```
<ProductContent> (trang danh sách)          <CommentList> (trang chi tiết)
        │                                            │
        │ với MỖI sản phẩm:                          │ getRating(productId)  → mảng rating
        │   getAvgStar(productId)  → số sao TB       │ getCmt(productId)     → mảng comment
        │   getTotalRating(productId) → số lượt      │        │
        ▼                                            ▼        ▼
   renderStar(ratingNumber)                    ghép (join) comment + rating theo userId
                                                     │
                                                     ▼
                                               renderStar(item.rating)
```

### 3.2. Model `Rating` và tầng API

`yotea-be/src/models/rating.js:1-20`

```js
import { Schema, model, ObjectId } from "mongoose";

const ratingSchema = new Schema({
    userId: {
        type: ObjectId,
        ref: "User"
    },
    ratingNumber: {
        type: Number,
        required: true
    },
    productId: {
        type: ObjectId,
        ref: "Product"
    }
}, { timestamps: true });

ratingSchema.index({'$**': 'text'});

export default model("Rating", ratingSchema);
```

Giống hệt `Comment`, chỉ thay `content` bằng `ratingNumber` (số). Route
(`yotea-be/src/routes/rating.js:8-12`) cũng y hệt comment: `GET` công khai, `POST/PUT/DELETE`
chỉ `requireSignin, isAuth`.

> ⚠️ **`ratingNumber` không có `min`/`max`.** Schema chỉ khai `type: Number, required: true`.
> Client gửi `999` hay `-5` vẫn lưu tuốt. Kết hợp việc điểm trung bình được tính ở
> frontend (mục 3.3), một bản ghi rác đủ làm sai lệch số sao trung bình. Cách đúng: thêm
> `min: 1, max: 5` ở schema.

Bây giờ là phần thú vị nhất — **điểm trung bình được tính ở đâu?**

`yotea-fe/src/api/rating.js:34-47`

```js
export const getAvgStar = async (productId) => {
  const url = `/${DB_NAME}/?productId=${productId}`;
  const { data } = await instance.get(url);
  const totalRating = data.reduce(
    (total, item) => total + item.ratingNumber,
    0
  );
  return Math.ceil(totalRating / data.length) || 0;
};

export const getTotalRating = (productId) => {
  const url = `/${DB_NAME}/?productId=${productId}`;
  return instance.get(url);
};
```

**Khẳng định từ code:** điểm trung bình được tính **hoàn toàn ở frontend**. Backend chỉ
trả về **danh sách thô** các bản ghi rating (qua bộ lọc `list()` chung); `getAvgStar`
tải cả mảng đó về rồi tự `reduce` cộng dồn `ratingNumber`, chia cho `data.length` và làm
tròn lên bằng `Math.ceil`. Backend **không hề có** endpoint kiểu `/ratings/average`.

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 35 | `url = .../?productId=...` | Lấy **mọi** rating của một sản phẩm |
| 37-40 | `data.reduce(...)` | Cộng dồn tất cả `ratingNumber` |
| 41 | `Math.ceil(totalRating / data.length) \|\| 0` | Trung bình, làm tròn **lên**; nếu chưa có rating (`data.length = 0` → `NaN`) thì `\|\| 0` cho về 0 |

> 💡 **Để ý `Math.ceil`:** trung bình 3.1 sao sẽ bị làm tròn **lên thành 4 sao**. Đây là
> lựa chọn "làm đẹp" điểm số, nhưng gây hiểu nhầm. Trung bình thực tế nên hiển thị số
> thập phân (ví dụ "4.2/5") thay vì làm tròn lên.

### 3.3. Hiển thị sao bằng FontAwesome

Cả `ProductContent` (danh sách) và `CommentList` (chi tiết) đều có **cùng một hàm**
`renderStar`. Trích từ `yotea-fe/src/components/user/CommentList.js:78-97`:

```jsx
  // render star
  const renderStar = (ratingNumber) => {
    const ratingArr = [];
    for (let i = 0; i < ratingNumber; i++) {
      ratingArr.push(
        <div className="text-yellow-400" key={i}>
          <FontAwesomeIcon icon={faStar} />
        </div>
      );
    }

    for (let i = 0; i < 5 - ratingNumber; i++) {
      ratingArr.push(
        <div className="text-gray-300" key={i + 5}>
          <FontAwesomeIcon icon={faStar} />
        </div>
      );
    }

    return ratingArr;
  };
```

Ý tưởng đơn giản: vẽ `ratingNumber` ngôi sao **vàng** (`text-yellow-400`), rồi vẽ nốt
`5 - ratingNumber` ngôi sao **xám** (`text-gray-300`) cho đủ 5 sao. Icon lấy từ
`@fortawesome/free-solid-svg-icons` (`faStar`). Vì `ratingNumber` đã bị `Math.ceil` làm
tròn thành số nguyên, vòng lặp luôn cho ra số sao nguyên — không có "nửa sao".

### 3.4. Frontend tự ghép comment + rating — `CommentList`

Đây là đoạn khó nhất của bài. Vì `Comment` và `Rating` là hai collection tách rời,
`CommentList` phải **tải cả hai rồi ghép theo `userId`**:

`yotea-fe/src/components/user/CommentList.js:28-56`

```jsx
  useEffect(() => {
    const getComments = async () => {
      const { data } = await getCmt(productId);
      setTotalCmt(data.length);

      const { data: dataComment } = await getCmt(productId, start, limit);
      const { data: dataRating } = await getRating(productId);

      const commentJoinRating = dataComment.map((cmt) => {
        const rating = dataRating.find(
          (item) => item.userId === cmt.userId._id
        );

        return {
          cmtId: cmt._id,
          ratingId: rating._id,
          userId: cmt.userId._id,
          userAvatar: cmt.userId.avatar,
          content: cmt.content,
          createdAt: cmt.createdAt,
          fullName: cmt.userId.fullName,
          rating: rating.ratingNumber,
        };
      });

      setComments(commentJoinRating);
    };
    productId && getComments();
  }, [productId, reRender, productDetailRerender, page]);
```

| Dòng | Ý nghĩa |
|---|---|
| 30-31 | Gọi `getCmt(productId)` **không phân trang** chỉ để **đếm tổng** (`totalCmt`) — lại là kiểu "gọi 2 lần" lãng phí như [Bài 03](03-kien-thuc-nen.md) đã bàn |
| 33 | Gọi lại `getCmt` **có `start/limit`** để lấy đúng một trang bình luận; nhờ `_expand=userId` (`api/comment.js:16`) nên `cmt.userId` đã là **object user đầy đủ** (có `avatar`, `fullName`) |
| 34 | Lấy **toàn bộ** rating của sản phẩm |
| 36-51 | **Ghép thủ công:** với mỗi bình luận, `find` bản rating có `item.userId === cmt.userId._id`, rồi gom thành một object phẳng |

> ⚠️ **`rating.ratingNumber` có thể làm sập cả danh sách.** Dòng 37-39 tìm rating khớp
> `userId`; nếu **không tìm thấy** (người này đã bình luận nhưng chưa từng đánh giá sao,
> hoặc rating bị xoá), `rating` là `undefined`, và dòng `ratingId: rating._id` sẽ ném
> `TypeError: Cannot read properties of undefined`. Vì `CommentProduct` luôn tạo comment
> **và** rating cùng lúc nên thực tế hiếm khi lệch — nhưng chỉ cần admin xoá một trong
> hai (mà không xoá cái kia) là toàn bộ danh sách bình luận **trắng trang**. Cách an toàn:
> `rating?.ratingNumber ?? 0` và bỏ `ratingId` khi không có rating.

Ngoài ra để ý sự trộn kiểu dữ liệu: dòng 37-38 so sánh `item.userId === cmt.userId._id`
— `item.userId` (bên rating) là **chuỗi id** (không `_expand`), còn `cmt.userId` (bên
comment) đã `_expand` thành **object** nên phải lấy `._id`. Viết lệch một chút là ghép
trượt.

### 3.5. ⚠️ Lỗi N+1 khi hiển thị danh sách sản phẩm

Đây là hệ quả trực tiếp của việc "tính trung bình ở frontend". Nhớ lại [Bài 25](25-danh-sach-san-pham.md):
trang danh sách render qua `ProductContent`. Xem vòng lặp tải dữ liệu:

`yotea-fe/src/components/user/ProductContent.js:31-54`

```jsx
  useEffect(() => {
    const getData = async () => {
      const { data } = await getProducts(
        start,
        limit,
        filter.sort,
        filter.order,
        parameter
      );

      const listProduct = [];
      for await (let product of data) {
        const ratingNumber = await getAvgStar(product._id);
        const { data: totalRating } = await getTotalRating(product._id);
        listProduct.push({
          ...product,
          ratingNumber,
          totalRating: totalRating.length,
        });
      }

      setProducts(listProduct);
    };
    getData();
```

> ⚠️ **Chỗ này dự án làm chưa chuẩn — lỗi N+1 request.** Sau khi lấy danh sách sản phẩm
> (1 request), vòng lặp `for await` gọi **thêm 2 request cho MỖI sản phẩm**: `getAvgStar`
> và `getTotalRating`. Trang hiển thị 9 sản phẩm (`limit = 9`, dòng 26) → **1 + 9×2 = 19
> request** chỉ để vẽ vài ngôi sao. Tệ hơn, `for await` chạy **tuần tự** (chờ từng sản
> phẩm xong mới sang sản phẩm kế), nên trang càng chậm. Đây đúng là "bài toán N+1" kinh
> điển: 1 truy vấn gốc kéo theo N truy vấn con.
>
> **Cách làm đúng:** backend trả kèm điểm trung bình và số lượt đánh giá **ngay trong
> danh sách sản phẩm** (dùng `aggregate` + `$lookup` gom rating theo `productId`, tính
> `$avg` một lần cho tất cả). Khi đó frontend chỉ cần **1 request**. Nếu buộc phải gọi ở
> frontend, tối thiểu hãy chạy song song bằng `Promise.all(data.map(...))` thay cho
> `for await` tuần tự.

### 3.6. Có kiểm trùng đánh giá không?

**Có — nhưng chỉ ở frontend.** Như đã thấy ở mục 2.3, `CommentProduct.onSubmit` gọi
`checkUserRating(user._id, productId)` trước; nếu đã có thì `update` thay vì tạo mới.
API tương ứng:

`yotea-fe/src/api/rating.js:6-9`

```js
export const checkUserRating = (userId, productId) => {
  const url = `/${DB_NAME}/?userId=${userId}&productId=${productId}`;
  return instance.get(url);
};
```

Nó chỉ là một truy vấn lọc theo `userId` + `productId`. **Backend không có ràng buộc nào**
— model `Rating` **không có unique index `{ userId, productId }`**. Nghĩa là nếu ai đó gọi
thẳng `POST /api/ratings/:userId` nhiều lần (bỏ qua bước kiểm ở frontend), họ tạo được
**nhiều rating trùng** cho cùng một sản phẩm, làm sai điểm trung bình. Ta sẽ vá cả hai
tầng ở phần Tự tay làm.

---

## 4. Chức năng YÊU THÍCH (wishlist)

### 4.1. Sơ đồ luồng — cơ chế KÉP

```
[bấm nút trái tim trên <ProductContent>]
        │
        ▼
handleFavorites(productId, slug)
        │
        ├─(0)─ chưa đăng nhập? → toast "Vui lòng đăng nhập"
        │
        └─(1)─ checkUserHeart(userId, productId)   (đã thích chưa?)
                   │
             đã có ─► toast "đã tồn tại"  (dừng)
                   │
             chưa ─┤
                   ├─(2a)─ getProduct(slug) → product.favorites++ → clientUpdate(product)
                   │            (TĂNG BỘ ĐẾM trên Product bằng số client tự tính)
                   │
                   └─(2b)─ dispatch(addWishlist({ userId, productId }))
                                (LƯU bản ghi FavoritesProduct qua Redux thunk)
```

Chú ý ngay: **một cú bấm tim ghi vào HAI nơi** — bộ đếm `favorites` trên document
`Product`, **và** một bản ghi mới trong collection `FavoritesProduct`. Đây là "cơ chế
kép" mà đề bài nhắc tới.

### 4.2. Model `FavoritesProduct` và bộ đếm `favorites`

`yotea-be/src/models/favoritesProduct.js:1-16`

```js
import { Schema, model, ObjectId } from "mongoose";

const schema = new Schema({
    userId: {
        type: ObjectId,
        ref: "User"
    },
    productId: {
        type: ObjectId,
        ref: "Product"
    }
}, { timestamps: true });

schema.index({'$**': 'text'});

export default model("FavoritesProduct", schema);
```

Chỉ đúng hai khoá ngoại — bảng nối thuần tuý. (Biến schema đặt tên trần trụi là `schema`,
khác mọi file còn lại — một điểm không nhất quán nhỏ.)

Song song, **trên chính document `Product` có một bộ đếm `favorites`** (trích
`yotea-be/src/models/product.js`, phần lõi):

```js
    favorites: {
        type: Number,
        default: 0,
    },
```

→ Vậy "số lượt yêu thích" của một sản phẩm được lưu ở **hai chỗ**: (a) đếm số bản ghi
trong `FavoritesProduct` có `productId` đó, và (b) con số `favorites` trên `Product`.

### 4.3. Tầng API `favorites` — `checkUserHeart`

`yotea-fe/src/api/favorites.js:1-32`

```js
import { isAuthenticate } from "../utils/localStorage";
import instance from "./instance";

const DB_NAME = "favoritesProduct";

export const add = (data, { token, user } = isAuthenticate()) => {
  const url = `/${DB_NAME}/${user._id}`;
  return instance.post(url, data, {
    headers: {
      Authorization: `Bearer ${token}`,
    },
  });
};

export const getAll = (userId) => {
  const url = `/${DB_NAME}/?userId=${userId}&_expand=userId&_expand=productId&_sort=createdAt&_order=desc`;
  return instance.get(url);
};

export const remove = (id, { token, user } = isAuthenticate()) => {
  const url = `/${DB_NAME}/${id}/${user._id}`;
  return instance.delete(url, {
    headers: {
      Authorization: `Bearer ${token}`,
    },
  });
};

export const checkUserHeart = (userId, productId) => {
  const url = `/${DB_NAME}/?userId=${userId}&productId=${productId}`;
  return instance.get(url);
};
```

- `getAll` dùng **`_expand=productId`** để "nở" `productId` thành object sản phẩm đầy đủ —
  nhờ vậy trang wishlist hiển thị được tên/ảnh/giá sản phẩm đã thích.
- `checkUserHeart` giống hệt `checkUserRating`: lọc theo `userId` + `productId` để biết
  "đã thích chưa". Lại là **kiểm trùng ở frontend**, backend không có unique index.

### 4.4. `wishlistSlice` — Redux quản lý danh sách yêu thích

Đây là **một trong đúng ba slice** mà phía khách hàng thật sự dùng Redux (cùng `authSlice`
và `cartSlice`). Toàn văn:

`yotea-fe/src/redux/wishlistSlice.js:9-59`

```js
export const getWishlist = createAsyncThunk(
  "wishlist/getWishlist",
  async (userId) => {
    const { data } = await getAll(userId);
    return data;
  }
);

export const addWishlist = createAsyncThunk(
  "wishlist/addWishlist",
  async ({ userId, productId }) => {
    return add({ userId, productId }).then(async () => {
      const { data } = await getAll(userId);
      return data;
    });
  }
);

export const deleteWishlist = createAsyncThunk(
  "wishlist/deleteWishlist",
  async (id) => {
    return remove(id).then(() => id);
  }
);

const wishlistSlice = createSlice({
  name: "wishlist",
  initialState,
  reducers: {
    clearWishlist(state) {
      state.value = [];
    },
    showWishlist(state, { payload }) {
      state.showWishlist = payload;
    },
  },
  extraReducers: (builder) => {
    builder.addCase(getWishlist.fulfilled, (state, { payload }) => {
      state.value = payload;
    });

    builder.addCase(addWishlist.fulfilled, (state, { payload }) => {
      state.value = payload;
      state.showWishlist = true;
    });

    builder.addCase(deleteWishlist.fulfilled, (state, { payload }) => {
      state.value = state.value.filter((item) => item._id !== payload);
    });
  },
});
```

| Thunk | Làm gì |
|---|---|
| `getWishlist(userId)` | Tải toàn bộ wishlist của người dùng |
| `addWishlist({userId, productId})` | `add(...)` xong thì **tải lại toàn bộ** danh sách và trả về (nên `payload` là cả mảng mới) |
| `deleteWishlist(id)` | `remove(...)` xong trả về `id` để reducer `filter` bỏ khỏi state |

> 💡 Slice này chỉ xử lý case `.fulfilled` (đúng như ghi chú chung của dự án — không có
> `pending`/`rejected`). `addWishlist` chọn cách "ghi xong tải lại toàn bộ" cho chắc, đổi
> lại tốn thêm một request.

### 4.5. ⚠️ Hai nguồn sự thật + client tự gửi số đếm

Nhìn lại `handleFavorites` (mục 4.1), `yotea-fe/src/components/user/ProductContent.js:82-94`:

```jsx
      if (!data.length) {
        // cập nhật số lượng yêu thích
        const { data: product } = await getProduct(slug);
        product.favorites++;

        clientUpdate(product);

        dispatch(
          addWishlist({
            userId: user._id,
            productId,
          })
        );
```

Và endpoint mà `clientUpdate` gọi tới, `yotea-be/src/controllers/product.js:278-296`:

```js
export const clientUpdate = async (req, res) => {
  const filter = { _id: req.params.id };
  const { view, favorites } = req.body;
  const options = { new: true };

  try {
    const product = await Product.findOneAndUpdate(
      filter,
      { view, favorites },
      options
    ).exec();
    res.json(product);
  } catch (error) {
    res.status(400).json({
      message: "Cập nhật sản phẩm thất bại",
      error,
    });
  }
};
```

> ⚠️ **Chỗ này dự án làm chưa chuẩn — hai vấn đề chồng nhau.**
>
> **(1) Hai nguồn sự thật (dual source of truth).** Số lượt thích được lưu ở cả bộ đếm
> `Product.favorites` **lẫn** số bản ghi trong `FavoritesProduct`. Chúng được cập nhật
> bằng **hai request riêng biệt, không có giao dịch (transaction)** — `clientUpdate` tăng
> đếm ở (2a), `addWishlist` lưu bản ghi ở (2b). Nếu một trong hai lỗi (đứt mạng giữa
> chừng), hai con số **lệch nhau vĩnh viễn**. Ngoài ra khi **bỏ thích** (`deleteWishlist`),
> code **không hề giảm** `Product.favorites` — nên bộ đếm chỉ có tăng, không có giảm, càng
> ngày càng sai.
>
> **(2) Client tự tính rồi gửi số đếm.** Frontend `getProduct(slug)` → `product.favorites++`
> → gửi **con số tuyệt đối** lên. `clientUpdate` **ghi đè** bằng đúng số client gửi (không
> dùng `$inc`), và route `PATCH /api/products/userUpdate/:id` **không có middleware auth**.
> Hệ quả: bất kỳ ai cũng gửi `{ "favorites": 999999 }` để bơm khống, hoặc hai người bấm
> tim cùng lúc thì cả hai đọc cùng giá trị cũ → chỉ cộng lên 1 thay vì 2 (**race
> condition**).
>
> **Cách làm đúng:** chọn **một** nguồn sự thật. Đơn giản nhất là bỏ bộ đếm, "số lượt
> thích" = `count` bản ghi `FavoritesProduct`. Nếu muốn giữ bộ đếm để truy vấn nhanh thì
> để **backend** tự tăng/giảm bằng `$inc` mỗi khi thêm/xoá wishlist, client **không gửi**
> con số nào.

---

## 5. ⚠️ Tổng kết các điểm chưa chuẩn (nối [Bài 33](33-ra-soat-bao-mat.md))

Gom lại để bạn thấy bức tranh chung của cả ba chức năng:

| Vấn đề | Bằng chứng | Hệ quả |
|---|---|---|
| Route `POST/PUT/DELETE` chỉ có `requireSignin, isAuth`, **thiếu `isAdmin`** | `routes/comment.js:11-12`, `rating.js:11-12`, `favoritesProduct.js:11-12` | Không phân biệt admin/khách ở các thao tác sửa/xoá |
| **Không kiểm chủ sở hữu** khi sửa/xoá | `controllers/comment.js:122-134` chỉ lọc theo `_id` của bình luận | Khách A xoá/sửa được bình luận/đánh giá của khách B (chỉ cần đoán `_id`) — xem [Bài 33](33-ra-soat-bao-mat.md) lỗ hổng #7 |
| **`userId` do client khai**, không lấy từ token | `new Comment(req.body)`, `AddRating({ userId: user._id })` | Mạo danh người khác để bình luận / đánh giá |
| **Không kiểm trùng ở backend** (thiếu unique `{userId, productId}`) | `models/rating.js`, `models/favoritesProduct.js` không có compound index | Spam rating/wishlist trùng, làm sai thống kê |
| **Không có `status` duyệt bình luận** | `models/comment.js` chỉ 3 trường; `AdminCommentList` chỉ có nút xoá | Nội dung xấu hiển thị ngay, admin dọn sau |
| **Điểm TB tính ở FE → N+1** | `ProductContent.js:42-49` vòng `for await` | Chậm, tốn request |
| **Hai nguồn sự thật + client gửi số đếm** | `Product.favorites` vs `FavoritesProduct`; `clientUpdate` ghi đè | Số lệch, bơm khống, race condition |

`isAuth` (`middlewares/checkAuth.js:9-19`) chỉ đảm bảo **"`:userId` trên URL đúng là bạn"**
— nó **không** kiểm bình luận/đánh giá bạn định xoá có phải của chính bạn không. Đó là
lý do "kiểm chủ sở hữu" phải làm thêm trong controller.

---

## 6. 🛠️ Tự tay làm

> Mục tiêu: cuối phần này bạn sẽ (A) có nút "Xoá" bình luận **chỉ hiện với chủ nhân**, xoá
> xong danh sách tự cập nhật; và (B) chặn được việc **một người đánh giá nhiều lần** cùng
> một sản phẩm ở **cả frontend lẫn backend**.

### Bài A — Nút "xoá bình luận của chính mình"

Dự án đã có sẵn một phần logic này trong `CommentList` (`CommentList.js:122`), nhưng ta sẽ
**tự viết lại cho hiểu** và vá luôn lỗ hổng "không kiểm chủ sở hữu" ở backend.

#### Bước 1 — Nút chỉ hiện với chủ (frontend)

Mở `yotea-fe/src/components/user/CommentList.js`, trong phần render mỗi bình luận, bọc nút
xoá bằng điều kiện so `item.userId` với `user._id`:

```jsx
// yotea-fe/src/components/user/CommentList.js  ← đoạn bạn tự chỉnh trong <ul> mỗi bình luận
{user && item.userId === user._id && (
  <li
    onClick={() => handleRemoveCmt(item.cmtId)}
    className="btn-remove transition hover:text-black cursor-pointer"
  >
    Xóa
  </li>
)}
```

Vì `item.userId` đã được chuẩn hoá thành **chuỗi id** ở bước ghép (`CommentList.js:44`:
`userId: cmt.userId._id`), phép so `=== user._id` chạy đúng. (Bản gốc của dự án còn cho
admin xoá bằng `|| user.role` — bạn có thể giữ hoặc bỏ tuỳ ý; ở đây ta chỉ cho **chủ**
xoá để bám sát đề.)

#### Bước 2 — Gọi API xoá + cập nhật danh sách

`handleRemoveCmt` đã có sẵn (`CommentList.js:58-75`) — nó dùng `Swal` hỏi xác nhận rồi gọi
`deleteCmt(cmtId)` và `setRerender(!reRender)` để `useEffect` chạy lại, tải danh sách mới.
Nếu muốn cập nhật **ngay tại client** (không gọi lại API) như `AdminCommentList` làm, đổi
thành:

```jsx
// đoạn bạn tự sửa trong handleRemoveCmt, thay cho setRerender
deleteCmt(cmtId).then(() => {
  setComments((prev) => prev?.filter((item) => item.cmtId !== cmtId));
  Swal.fire("Thành công!", "Bình luận đã bị xóa.", "success");
});
```

#### Bước 3 — Vá kiểm chủ sở hữu ở backend (quan trọng nhất)

Ẩn nút ở frontend **không bảo vệ được gì** — kẻ xấu vẫn gọi thẳng
`DELETE /api/comments/:id/:userId`. Sửa controller để chỉ chủ nhân mới xoá được:

```js
// yotea-be/src/controllers/comment.js  ← phiên bản remove BẠN TỰ VIẾT THÊM (dự án chưa có)
export const remove = async (req, res) => {
  try {
    const comment = await Comment.findById(req.params.id).exec();
    if (!comment) return res.status(404).json({ message: "Không tìm thấy bình luận" });

    // req.auth._id lấy từ token đã ký — nguồn đáng tin, KHÔNG tin :userId trên URL
    if (comment.userId.toString() !== req.auth._id) {
      return res.status(403).json({ message: "Bạn không có quyền xoá bình luận này" });
    }

    await comment.deleteOne();
    res.json(comment);
  } catch (error) {
    res.status(400).json({ message: "Xóa bình luận không thành công", error });
  }
};
```

> 🔒 **Ghi chú bảo mật:** mấu chốt là so `comment.userId` với **`req.auth._id`** (giải mã
> từ token), **không** với `:userId` trên URL. Nhớ trả `return` sau mỗi `res` để không
> chạy tiếp — đúng bài học ở [Bài 33](33-ra-soat-bao-mat.md).

### Bài B — Mỗi người chỉ đánh giá một lần / một sản phẩm

#### Bước 1 — Frontend đã chặn sẵn

Như mục 3.6, `CommentProduct.onSubmit` đã gọi `checkUserRating` trước khi `AddRating`. Đó
là lớp chặn phía frontend. Nhưng nó **chỉ chống nhầm lẫn của người dùng bình thường**, không
chống được ai gọi thẳng API.

#### Bước 2 — Ràng buộc cứng ở backend bằng unique index

Thêm compound unique index vào model:

```js
// yotea-be/src/models/rating.js  ← dòng BẠN TỰ THÊM, đặt ngay trước dòng export
ratingSchema.index({ userId: 1, productId: 1 }, { unique: true });
```

Từ nay MongoDB **từ chối** (lỗi `E11000 duplicate key`) mọi rating thứ hai của cùng một
`userId` cho cùng một `productId` — kể cả khi request đi vòng qua frontend.

#### Bước 3 — Controller xử lý va chạm cho êm

Bắt lỗi trùng và trả thông báo tử tế thay vì để lộ lỗi Mongo:

```js
// yotea-be/src/controllers/rating.js  ← create phiên bản BẠN TỰ VIẾT THÊM
export const create = async (req, res) => {
  try {
    const rating = await new Rating({
      userId: req.auth._id,            // lấy từ token, không tin client
      productId: req.body.productId,
      ratingNumber: req.body.ratingNumber,
    }).save();
    res.json(rating);
  } catch (error) {
    if (error.code === 11000) {
      return res.status(409).json({ message: "Bạn đã đánh giá sản phẩm này rồi" });
    }
    res.status(400).json({ message: "Thêm đánh giá thất bại", error });
  }
};
```

> 💡 Với dữ liệu **đã có sẵn** những rating trùng từ trước, tạo unique index sẽ báo lỗi.
> Bạn cần dọn các bản ghi trùng trong MongoDB trước, rồi mới build index.

---

## 7. ✅ Kiểm chứng kết quả

**Với chức năng bình luận + đánh giá:**

1. Đăng nhập bằng tài khoản khách, mở một trang chi tiết sản phẩm (`/san-pham/:slug`).
2. Chọn số sao, nhập nội dung, bấm **Gửi đi** → thấy toast "Bình luận thành công", bình
   luận mới hiện lên đầu danh sách kèm đúng số sao.
3. Mở Postman kiểm tra hai collection cùng có bản ghi mới:

```
GET http://localhost:8080/api/comments/?productId=<id-sp>&_expand=userId
GET http://localhost:8080/api/ratings/?productId=<id-sp>
```

`comments` phải có `content` bạn vừa gõ; `ratings` phải có `ratingNumber` bạn vừa chọn.

4. Gửi bình luận **lần hai** cho cùng sản phẩm → có thêm một comment mới, nhưng số bản ghi
   `ratings` **không tăng** (bản rating cũ bị `update`, không tạo mới). Đó là bằng chứng
   ràng buộc "một rating / sản phẩm" đang chạy (ở frontend).

**Với chức năng yêu thích:**

5. Bấm nút trái tim trên một sản phẩm ở trang danh sách → toast "Đã thêm...". Kiểm tra:

```
GET http://localhost:8080/api/favoritesProduct/?userId=<id-cua-ban>&_expand=productId
GET http://localhost:8080/api/products/<slug-sp>
```

Phải thấy một bản ghi `FavoritesProduct` mới, **và** `favorites` của sản phẩm tăng 1.

6. Bấm tim **lần nữa** cùng sản phẩm → toast "Sản phẩm đã tồn tại..." (nhờ `checkUserHeart`).

**Sau khi làm phần Tự tay làm:** thử `DELETE /api/comments/<id-cmt-của-người-khác>/<id-của-bạn>`
bằng Postman với token của bạn → phải nhận **403** thay vì xoá thành công.

---

## 8. 🐞 Lỗi thường gặp

| Thông báo / hiện tượng | Nguyên nhân | Cách xử lý |
|---|---|---|
| Danh sách bình luận trắng trang, console báo `Cannot read properties of undefined (reading '_id')` | `CommentList.js:43` — có comment nhưng không tìm được rating khớp `userId` | Dùng `rating?.ratingNumber ?? 0`, bỏ `ratingId` khi thiếu (xem mục 3.4) |
| Bấm gửi bình luận báo `Cannot read properties of null (reading '_id')` | Chưa đăng nhập nên `user` là `null`, `user._id` nổ | Chỉ render `CommentProduct` khi đã đăng nhập, hoặc chặn sớm bằng `if (!user)` |
| `401 Unauthorized` khi gửi bình luận | Token hết hạn (JWT sống 3h) hoặc `:userId` không khớp token | Đăng nhập lại |
| Trang danh sách sản phẩm tải rất chậm | Lỗi N+1: mỗi sản phẩm 2 request sao (mục 3.5) | Chuyển sang `Promise.all`, hoặc để backend trả sẵn điểm TB |
| Số `favorites` trên sản phẩm to bất thường / không giảm khi bỏ thích | Client gửi số tuyệt đối + bỏ thích không giảm đếm (mục 4.5) | Để backend `$inc`/`$inc: -1`; hoặc bỏ đếm, dùng `count` |
| Tạo unique index báo `E11000` ngay khi khởi động | Đã có rating trùng trong DB | Xoá bản ghi trùng trước rồi mới build index |

---

## 9. 📝 Bài tập

**Bài 1.** `getAvgStar` (`api/rating.js:34-42`) trả về `Math.ceil(totalRating / data.length) || 0`.
Giải thích chính xác chuyện gì xảy ra khi một sản phẩm **chưa có đánh giá nào**, và vì
sao `|| 0` là cần thiết.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Khi chưa có rating, `data` là mảng rỗng: `data.reduce(..., 0)` trả `0`, còn `data.length`
bằng `0`, nên `0 / 0 = NaN`. `Math.ceil(NaN)` vẫn là `NaN`. Nếu trả `NaN` về, hàm
`renderStar(NaN)` sẽ có vòng lặp `for (i = 0; i < NaN; ...)` — điều kiện `0 < NaN` là
`false` nên **không vẽ sao vàng nào**, còn vòng thứ hai `i < 5 - NaN` cũng là `NaN` → không
vẽ nốt sao xám. Kết quả là **không có ngôi sao nào** (giao diện vỡ). Nhờ `|| 0`, `NaN`
(giá trị falsy) bị thay bằng `0`, nên `renderStar(0)` vẽ đúng **5 sao xám** — trạng thái
"chưa ai đánh giá" nhìn hợp lý.

</details>

**Bài 2.** Viết lại vòng lặp N+1 trong `ProductContent.js:41-50` để **giảm thời gian tải**
mà **không** đổi backend. (Chỉ cần chạy song song, không cần bỏ N+1.)

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

```jsx
// đoạn bạn tự viết thay cho vòng for await
const listProduct = await Promise.all(
  data.map(async (product) => {
    const ratingNumber = await getAvgStar(product._id);
    const { data: totalRating } = await getTotalRating(product._id);
    return { ...product, ratingNumber, totalRating: totalRating.length };
  })
);
setProducts(listProduct);
```

`for await` chạy **tuần tự** (sản phẩm thứ 2 chờ sản phẩm thứ 1 xong), tổng thời gian ≈
tổng của mọi request. `Promise.all(data.map(...))` bắn **tất cả cùng lúc**, tổng thời gian
≈ request chậm nhất. Vẫn còn 18 request (chưa hết N+1) nhưng nhanh hơn nhiều. Muốn diệt
hẳn N+1 thì phải sửa backend như đã nói ở mục 3.5.

</details>

**Bài 3.** (khó) Chức năng "bỏ thích" hiện **không giảm** `Product.favorites`. Hãy mô tả
cách sửa để bộ đếm luôn đúng, và giải thích vì sao cách "để backend `$inc`" tốt hơn cách
"frontend đọc số cũ rồi trừ 1 rồi gửi lên".

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Cách đúng: khi thêm wishlist, backend chạy `Product.findByIdAndUpdate(productId, { $inc: { favorites: 1 } })`;
khi xoá wishlist, chạy `{ $inc: { favorites: -1 } }`. Client **không gửi** con số nào, chỉ
gửi `productId`.

Vì sao `$inc` ở backend tốt hơn "frontend đọc-trừ-gửi":
- **Nguyên tử (atomic):** `$inc` cộng/trừ ngay trong một thao tác ghi của MongoDB, hai
  người thao tác cùng lúc vẫn ra đúng kết quả. Cách "đọc số cũ ở client rồi gửi số mới"
  bị **race condition**: hai người cùng đọc `favorites = 10`, cùng gửi `11`, kết quả chỉ
  là `11` thay vì `12`.
- **Không tin client:** client không quyết định được con số, nên không thể bơm khống
  `favorites: 999999`.

Lý tưởng nhất là gói cả "ghi bản ghi FavoritesProduct" và "`$inc` bộ đếm" trong **một
transaction** để không bao giờ lệch — hoặc bỏ hẳn bộ đếm và dùng `count`.

</details>

---

## 📌 Tóm tắt

- Bình luận, đánh giá, yêu thích đều là **bảng nối** `User` ↔ `Product`; `Comment` thêm
  `content`, `Rating` thêm `ratingNumber`, `FavoritesProduct` không thêm gì.
- `Comment` và `Rating` là **hai collection tách rời** → `CommentList` phải **tự ghép** hai
  mảng theo `userId` (`CommentList.js:36-51`), rất dễ vỡ nếu một bên thiếu.
- **Điểm trung bình tính ở frontend** (`getAvgStar` dùng `reduce` + `Math.ceil`), backend
  chỉ trả danh sách thô → gây **N+1 request** ở trang danh sách (2 request/sản phẩm).
- **Kiểm trùng** (đánh giá / yêu thích) **chỉ ở frontend** qua `checkUserRating` /
  `checkUserHeart`; backend **thiếu unique index `{userId, productId}`**.
- Yêu thích dùng **cơ chế kép**: lưu `FavoritesProduct` **và** tăng bộ đếm `Product.favorites`
  bằng con số **client tự tính** qua `clientUpdate` → **hai nguồn sự thật, dễ lệch, bơm
  khống được, race condition**.
- Route comment/rating/favorites chỉ `requireSignin, isAuth` — **thiếu `isAdmin`, không
  kiểm chủ sở hữu, không có `status` duyệt bình luận** (nối [Bài 33](33-ra-soat-bao-mat.md)).

**Từ khoá tra cứu thêm:** `many-to-many join table`, `N+1 query problem`, `MongoDB $inc atomic counter`, `compound unique index mongoose`, `denormalized counter`, `IDOR ownership check`

➡️ **Bài tiếp theo:** [31 — Tin tức, liên hệ, cửa hàng và slider trang chủ](31-tin-tuc-lien-he-cua-hang.md) — rời nhóm sản phẩm để xem các trang nội dung tĩnh và banner trang chủ hoạt động ra sao.
