# Scope — Order Payment Formal Verification

## 1. Tên đề tài

**Mô hình hóa và kiểm chứng hình thức quy trình thanh toán đơn hàng
bằng Kripke Structure và Bounded Model Checking**

---

## 2. Mục tiêu

Đề tài xây dựng một mô hình hình thức cho vòng đời của một đơn hàng,
tập trung vào mối quan hệ giữa trạng thái đơn hàng và trạng thái thanh toán.

Mô hình được biểu diễn bằng Kripke Structure và được kiểm chứng bằng
Bounded Model Checking (BMC).

Mục tiêu chính là kiểm tra các thuộc tính an toàn (safety) và tiến triển
(liveness) của quy trình thanh toán, đồng thời tạo các mô hình lỗi có chủ
đích để BMC tìm counterexample.

---

## 3. Đối tượng được kiểm chứng

Mỗi lần kiểm chứng chỉ xét **một đơn hàng duy nhất**.

Hệ thống được mô hình hóa từ thời điểm đơn hàng đã được tạo cho tới khi
đơn hàng đạt một trong hai trạng thái kết thúc:

- `COMPLETED`
- `CANCELLED`

Các hoạt động trước khi tạo đơn hàng và các hoạt động hậu mãi sau khi đơn
hàng hoàn tất không nằm trong phạm vi của mô hình.

---

## 4. Phương thức thanh toán

Mô hình hỗ trợ hai phương thức thanh toán:

### 4.1. Online Payment

Đối với thanh toán Online, đơn hàng phải được thanh toán thành công trước
khi được phép chuyển sang giai đoạn giao hàng.

Luồng tổng quát:

ORDER_CREATED
→ PENDING_PAYMENT
→ PROCESSING
→ PAID / FAILED
→ READY_TO_SHIP
→ SHIPPING
→ COMPLETED

Nếu thanh toán thất bại, người dùng có thể:

- thử thanh toán lại (`RETRY_PAYMENT`), hoặc
- hủy đơn hàng (`CANCEL_ORDER`).

---

### 4.2. Cash on Delivery (COD)

Đối với COD, đơn hàng không cần được thanh toán trước khi giao.

Luồng tổng quát:

ORDER_CREATED
→ READY_TO_SHIP
→ SHIPPING
→ PAYMENT_COLLECTED
→ COMPLETED

Trong giai đoạn `SHIPPING`, một đơn hàng COD có thể có:

paymentState = UNPAID

Điều này được xem là trạng thái hợp lệ.

Đơn hàng COD chỉ được chuyển sang `COMPLETED` sau khi việc giao hàng
thành công và khoản thanh toán COD đã được ghi nhận.

---

## 5. Payment Gateway

Hệ thống có thể tích hợp với một Payment Gateway ở mức triển khai/demo.

Tuy nhiên, trong mô hình kiểm chứng hình thức, Payment Gateway được
trừu tượng hóa thành các sự kiện:

- `PAYMENT_SUCCESS`
- `PAYMENT_FAILURE`
- `PAYMENT_TIMEOUT`

BMC không phụ thuộc trực tiếp vào API hoặc trạng thái nội bộ của Payment
Gateway.

`PAYMENT_TIMEOUT` được xem là một event chứ không phải một payment state.
Khi timeout xảy ra trong trạng thái `PROCESSING`, payment chuyển sang
`FAILED`.

---

## 6. Mô hình sự kiện

Hệ thống giả định các sự kiện được xử lý tuần tự.

Mỗi transition được xem là một bước nguyên tử.

Phiên bản hiện tại không mô hình hóa:

- concurrent events,
- race conditions,
- distributed transactions.

Ví dụ tình huống `PAYMENT_SUCCESS` và `CANCEL_ORDER` xảy ra đồng thời
không nằm trong phạm vi của phiên bản hiện tại.

---

## 7. Payment Retry

Không mô hình hóa giới hạn số lần thử lại thanh toán.

Không sử dụng biến `retryCount`.

Sau một lần thanh toán thất bại:

FAILED
→ RETRY_PAYMENT
→ PROCESSING

có thể được thực hiện nhiều lần trong phạm vi mô hình.

---

## 8. Duplicate Payment Callback

Duplicate payment callback không thuộc luồng hoạt động đúng (correct
model).

Tuy nhiên, callback thanh toán thành công bị xử lý lặp lại sẽ được sử dụng
như một fault scenario trong quá trình kiểm chứng.

Mục tiêu là kiểm tra rằng một giao dịch thanh toán thành công chỉ được ghi
nhận một lần và callback trùng không làm hệ thống xử lý payment thành công
lần thứ hai.

Fault scenario này chủ yếu áp dụng cho Online Payment.

---

## 9. Biến trạng thái chính

Mô hình dự kiến sử dụng các thành phần trạng thái:

- `orderState`
- `paymentState`
- `paymentMethod`
- `chargeCount`

Trong đó:

paymentMethod ∈ {ONLINE, COD}

Phương thức thanh toán được xác định khi đơn hàng được tạo hoặc xác nhận
và không thay đổi trong vòng đời của đơn hàng.

`chargeCount` được sử dụng để kiểm tra việc ghi nhận thanh toán thành công
lặp lại, chủ yếu đối với Online Payment.

---

## 10. Ngoài phạm vi

Phiên bản hiện tại không mô hình hóa:

- Authentication / Authorization
- User management
- Product management
- Shopping cart
- Inventory
- Voucher / Promotion
- Tax calculation
- Partial payment
- Installment payment
- Multiple payment gateways
- Refund
- Return order
- Shipping failure
- Concurrent event processing
- Distributed transaction
- Race condition

Refund có thể được nghiên cứu như một hướng mở rộng trong tương lai.

---

## 11. Trạng thái kết thúc

Hai trạng thái kết thúc của vòng đời đơn hàng là:

### COMPLETED

Đơn hàng đã hoàn tất quy trình giao hàng và nghĩa vụ thanh toán tương ứng.

### CANCELLED

Đơn hàng đã bị hủy và không được phép quay trở lại các giai đoạn thanh
toán hoặc giao hàng trong correct model.

Hai trạng thái này được xem là terminal states trong mô hình nghiệp vụ.

Trong Kripke Structure, chúng có thể sử dụng self-loop để đảm bảo quan hệ
transition là total.

---

## 12. Mục tiêu kiểm chứng

Mô hình sẽ tập trung kiểm chứng các nhóm thuộc tính sau:

1. Tính đúng của luồng Online Payment.
2. Tính đúng của luồng COD.
3. Không giao đơn Online khi chưa thanh toán thành công.
4. Không hoàn thành đơn hàng khi nghĩa vụ thanh toán chưa được đáp ứng.
5. Không xử lý một giao dịch thanh toán thành công nhiều hơn một lần.
6. Không tái kích hoạt một đơn hàng đã bị hủy.
7. Payment đang xử lý phải cuối cùng đạt một kết quả hợp lệ.
8. Các trạng thái kết thúc không được quay lại trạng thái hoạt động.

Các thuộc tính cụ thể sẽ được đặc tả ở giai đoạn `properties.md`.