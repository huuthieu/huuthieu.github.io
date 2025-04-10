---
layout: post
title: Nén dữ liệu hiệu quả và đánh chỉ mục với roaring bitmap
categories: reading
---


Thế giới hiện đại đã chứng kiến sự bùng nổ về dữ liệu, dẫn đến vấn đề nảy sinh về tài nguyên cơ sở hạ tầng sử dụng để lưu trữ cũng như hiệu suất truy vấn dữ liệu tăng lên đáng kể.
Trong ngữ cảnh đó, các kỹ thuật nén và tối ưu truy vấn trở nên quan trọng và cần thiết. Một số giải pháp tập trung vào việc nén dữ liệu, giúp giảm thiểu lượng data cần để lưu trữ, một số khác tập trung vào việc tối ưu truy vấn thông tin. Roaring Bitmap nổi bật với khả năng cân bằng giữa việc nén và hiệu suất truy vấn.

## Giới thiệu về Bitmap

Trước khi khám phá chi tiết về Roaring Bitmap, chúng ta sẽ cùng tìm hiểu về cơ bản của Bitmap truyền thống.
Bitmap là một cấu trúc dữ liệu đơn giản và phổ biến trong việc biểu diễn và truy vấn tập dữ liệu có giá trị nguyên. Nó biểu diễn các phần tử trong tập dữ liệu bằng các bit 0 và 1, trong đó mỗi bit đại diện cho một phần tử và giá trị bit tương ứng là 1 nếu phần tử đó có mặt trong tập dữ liệu, ngược lại là 0.

| SET NAME |  | TẬP USER |
| --- | --- | --- |
| Sống ở Hà Nội | 1 1 1 1 1 1 1 0 1 1 | {0, 1, 2, 3, 4, 5, 6, 8, 9} |
| Thích xem phim | 0 1 0 1 1 0 0 1 0 0 | {1, 3, 4, 7} |
| Thích nghe nhạc | 0 0 1 1 0 0 0 1 0 0 | {2, 3, 8} |
| Thích xem phim OR Thích nghe nhạc | 0 1 1 1 1 0 0 1 1 0 | {1, 2, 3, 4, 7, 8} |
| Hà Nội AND (Thích phim OR Thích nhạc) | 0 1 1 1 1 0 0 0 1 0 | {1, 3, 4, 7, 8} |

Để query 1 tập user thỏa mãn điều kiện: "Sống ở Hà Nội và (Thích xem phim hoặc Thích nghe nhạc)": ta có thể query nhanh chóng bằng phép toán tập hợp (bitwise operator):

=> Sống ở Hà Nội và Thích xem phim hoặc Thích nghe nhạc = {1, 2, 3, 4, 8}
Mặc dù bitmap truyền thống đơn giản và dễ hiểu, nhưng khi phải sử dụng để lưu trữ chỉ một giá trị lớn như 1 tỷ, đòi hỏi phải có rất nhiều số 0.

```latex
 1000000000000000000000000000000....0000000000000000000000000
 ^
 |
Vị trí 1 tỷ|————————— 999,999,999 bit 0——————————————————————|
```

Trong trường hợp này, sử dụng bitmap truyền thống không hiệu quả vì yêu cầu một lượng lớn không gian lưu trữ. Với việc cần tới khoảng 125 MB để lưu trữ một tập hợp chứa chỉ một số 1 tỷ, dẫn đến việc gây lãng phí không gian và làm giảm hiệu suất.

Hãy cùng tìm hiểu về Roaring Bitmap và cách nó là giải quyết tối ưu cho việc nén dữ liệu, quan trọng hơn nó vẫn giữ cho các biswise operator nhanh chóng.

## Roaring Bitmap là gì?

From roaringbitmap.org:

> Roaring bitmaps are compressed bitmaps. They can be hundreds of times faster.
> 

Roaring Bitmap linh hoạt sử dụng các cấu trúc dữ liệu khác nhau để biểu diễn các tập dữ liệu có tính chất khác nhau. Cụ thể roaring Bitmap chia tập dữ liệu thành các containers nhỏ (2^16 = 65536 bit / 1 container) để lưu trữ giúp tiết kiệm không gian lưu trữ cho các tập dữ liệu có dạng thưa thớt:

- Đối với số nguyên 32 bit ta có tổng 2^16 container (Vì container size = 2^16 nên với 1 roaring bitmap lưu trữ số ngyên 32 bit thì sẽ có tất cả 2^32 / 2^16 = 2^16 containers)
- Để xác định số nguyên X thuộc container nào ta dễ dàng tính như sau:

```latex
X-container = X / 2^16 (16 most significant bits)
```

- Tất cả các số nguyên thuộc 1 container được lưu trữ cùng nhau trên memory.
- Roaring bitmap sử dụng 1 sorted array để lưu trữ các index tới từng containers. Arrays size dynamic tăng khi các containers mới được thêm vào bitmap.

Roaring Bitmap được thiết kế để thực hiện các phép toán tập hợp (bitwise operator) nhanh chóng và hiệu quả. Các phép toán như hợp nhất (union), giao (intersection) và hiệu (difference) trên các tập dữ liệu có thể được thực hiện trong thời gian ngắn đáng kể so với các cấu trúc dữ liệu truyền thống. Điều này giúp tăng cường hiệu suất truy vấn và giảm thiểu thời gian xử lý, đặc biệt là khi xử lý các tập dữ liệu lớn và phức tạp.

## Các loại containers trong Roaring Bitmap

Số lượng phần tử (cardinality) trong mỗi container sẽ quyết định loại container:

### 1. Array container: cardinality < 4,096 integers

Array container là loại container sử dụng một mảng số nguyên để lưu trữ các phần tử của tập dữ liệu dưới dạng được mảng sắp xếp và đóng gói (sorted packed arrays). Đóng gói ở đây nghĩa là với mỗi số nguyên integer 32 bit nó chỉ cần 16 bit để lưu trữ (do mỗi container chỉ có tối đa 2^16 phần tử). Ví dụ ta có tập {65537, 65538, 65539, 65540, ..., 66534, 66535, 66536} có thể biểu diễn dưới dạng đóng gói tương ứng như sau:

Array container thích hợp cho các tập dữ liệu có dạng thưa thớt, vì chỉ lưu trữ các phần tử thực sự có mặt trong tập dữ liệu. Tuy nhiên, khi tập dữ liệu có mật độ cao, việc lưu trữ mảng số nguyên sẽ tốn không gian hơn so với bitmap container.

### 2. Bitmap container

Bitmap container là một loại container dùng để biểu diễn một tập hợp dữ liệu có mật độ cao, tức là số lượng phần tử trong tập dữ liệu lớn. Nó sử dụng một dãy bit (bitmap) để đánh dấu sự có mặt của các phần tử trong tập dữ liệu. Trong trường hợp này nó không khác gì 1 bitmap truyền thống (với 2^16 bit)

Ví dụ: Giả sử chúng ta có tập dữ liệu {1, 3, 5, 7, 9}. Bit set container sẽ biểu diễn tập dữ liệu này thành dãy bit: 0101010101.

Bitmap container container thích hợp cho các tập dữ liệu có mật độ cao, vì nó tiết kiệm không gian lưu trữ và cho phép thực hiện các phép toán các phép toán tập hợp (bitwise operator) nhanh chóng. Tuy nhiên, khi tập dữ liệu có mật độ thấp, việc lưu trữ nhiều bit 0 sẽ làm lãng phí không gian.

### 3. Run container

Run container là loại container sử dụng để biểu diễn các khoảng giá trị liên tiếp trong tập dữ liệu. Run container giúp tiết kiệm không gian lưu trữ cho các tập dữ liệu có các khoảng giá trị liên tiếp. Thay vì lưu trữ từng phần tử riêng lẻ, nó sẽ ghi nhận các khoảng giá trị liên tiếp và lưu trữ dưới dạng các giá trị bắt đầu của các khoảng run và độ dài của đoạn run.

Ví dụ: Nếu ta có tập dữ liệu {1, 2, 3, 5, 6, 7}, Run container sẽ lưu trữ thành các run: (1, 3), (5, 3).

Nhờ sử dụng các loại containers khác nhau, Roaring Bitmap có thể linh hoạt và hiệu quả trong việc biểu diễn và truy vấn các tập dữ liệu có tính chất đa dạng. Sự lựa chọn linh hoạt các loại container sẽ giúp tối ưu không gian lưu trữ và cải thiện hiệu suất truy vấn của Roaring Bitmap khi làm việc với các tập dữ liệu lớn và phức tạp.

## Kết luận

Roaring Bitmap là một kỹ thuật nén mạnh mẽ trong việc lưu trữ và truy vấn dữ liệu lớn. Với khả năng tối ưu không gian lưu trữ, tích hợp dễ dàng và các loại containers linh hoạt, Roaring Bitmap đã trở thành một công cụ hữu ích trong việc xử lý dữ liệu lớn và các ứng dụng phức tạp.

Roaring Bitmap cũng cung cấp các thư viện và công cụ hỗ trợ tích hợp dễ dàng vào nhiều ngôn ngữ lập trình phổ biến. Điều này giúp cho việc sử dụng Roaring Bitmap trở nên tiện lợi và linh hoạt trong nhiều trường hợp.