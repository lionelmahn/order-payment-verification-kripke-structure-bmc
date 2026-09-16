# Business Rules — Order Payment Formal Verification

## 1. Purpose

Tài liệu này định nghĩa các quy tắc nghiệp vụ của mô hình
Order Payment Verification.

Các Business Rules là nguồn đặc tả chính để xây dựng:

- State model
- Transition relation
- Kripke Structure
- Safety properties
- Liveness properties
- Bug scenarios
- Bounded Model Checking experiments

Nếu implementation hoặc formal model mâu thuẫn với Business Rules,
Business Rules được xem là đặc tả cần tuân thủ.

---

# 2. General Rules

## BR-G01 — Initial Order State

Mỗi lần kiểm chứng chỉ xét một đơn hàng.

Khi mô hình bắt đầu:

orderState = CREATED
paymentState = UNPAID
chargeCount = 0

Phương thức thanh toán của đơn hàng là một trong:

paymentMethod ∈ {ONLINE, COD}

---

## BR-G02 — Payment Method Immutability

Phương thức thanh toán được xác định khi đơn hàng được tạo/xác nhận.

Sau khi đã xác định:

ONLINE

hoặc:

COD

paymentMethod không được thay đổi trong suốt vòng đời của đơn hàng.

Không cho phép:

ONLINE → COD

hoặc:

COD → ONLINE

giữa quy trình.

---

## BR-G03 — Sequential Event Processing

Mỗi transition chỉ xử lý một event.

Các event được xử lý tuần tự.

Phiên bản hiện tại không xét:

- concurrent events
- race conditions
- distributed transactions

Mỗi transition được xem là một bước nguyên tử.

---

## BR-G04 — Terminal States

Hai trạng thái kết thúc của order là:

COMPLETED
CANCELLED

Sau khi order đạt một trong hai trạng thái này,
correct model không cho phép quay lại các trạng thái xử lý payment
hoặc shipping.

Trong Kripke Structure có thể sử dụng self-loop:

COMPLETED → COMPLETED
CANCELLED → CANCELLED

để đảm bảo transition relation là total.

---

# 3. Online Payment Rules

## BR-O01 — Online Order Requires Prepayment

Đối với:

paymentMethod = ONLINE

đơn hàng phải được thanh toán thành công trước khi được giao.

Do đó:

ONLINE AND SHIPPING
→ paymentState = PAID

Một trạng thái như:

ONLINE + SHIPPING + UNPAID

là trạng thái không hợp lệ.

---

## BR-O02 — Online Payment Start Condition

Online payment chỉ được bắt đầu khi:

orderState = PENDING_PAYMENT

và:

paymentState = UNPAID

hoặc:

paymentState = FAILED

---

## BR-O03 — Starting Online Payment

Khi event:

START_PAYMENT

hoặc:

RETRY_PAYMENT

được chấp nhận:

paymentState := PROCESSING

---

## BR-O04 — Processing Cannot Be Cancelled

Khi Online Payment đang:

paymentState = PROCESSING

event:

CANCEL_ORDER

không được phép thực hiện.

Hệ thống phải chờ payment đạt một trong hai kết quả:

PAID
FAILED

trước khi quyết định bước tiếp theo.

Điều này tránh việc mô hình cơ sở phải xử lý race condition giữa:

PAYMENT_SUCCESS
và
CANCEL_ORDER.

---

## BR-O05 — Payment Success

Chỉ payment đang ở:

PROCESSING

mới được xử lý event:

PAYMENT_SUCCESS

Sau khi payment thành công:

paymentState := PAID
chargeCount := chargeCount + 1
orderState := READY_TO_SHIP

Trong correct model:

chargeCount = 1

sau lần thanh toán thành công đầu tiên.

---

## BR-O06 — Payment Failure

Nếu payment đang:

PROCESSING

và xảy ra:

PAYMENT_FAILURE

hoặc:

PAYMENT_TIMEOUT

thì:

paymentState := FAILED

order vẫn nằm trong luồng chờ xử lý payment.

PAYMENT_TIMEOUT là event,
không phải một payment state riêng.

---

## BR-O07 — Retry After Failure

Khi:

paymentState = FAILED

người dùng được phép thực hiện:

RETRY_PAYMENT

Khi đó:

FAILED
→ PROCESSING

Không sử dụng retryCount.

Mô hình không giới hạn số lần retry payment.

---

## BR-O08 — Cancel Before Processing

Online order được phép CANCEL khi payment chưa bắt đầu xử lý.

Ví dụ:

PENDING_PAYMENT + UNPAID
→ CANCELLED + UNPAID

---

## BR-O09 — Cancel After Payment Failure

Nếu payment đã:

FAILED

người dùng được phép thực hiện:

CANCEL_ORDER

Khi đó:

PENDING_PAYMENT + FAILED
→ CANCELLED + FAILED

---

## BR-O10 — Cannot Cancel After Successful Payment

Trong phạm vi phiên bản hiện tại,
order Online đã:

paymentState = PAID

không được CANCEL trực tiếp.

Refund không nằm trong phạm vi phiên bản hiện tại.

---

## BR-O11 — No Direct UNPAID to PAID Transition

Online payment không được phép:

UNPAID → PAID

trực tiếp.

Luồng hợp lệ là:

UNPAID
→ PROCESSING
→ PAID

---

## BR-O12 — No Direct FAILED to PAID Transition

Payment không được phép:

FAILED → PAID

trực tiếp.

Luồng hợp lệ phải là:

FAILED
→ RETRY_PAYMENT
→ PROCESSING
→ PAYMENT_SUCCESS
→ PAID

---

## BR-O13 — No Duplicate Successful Payment Processing

Một giao dịch Online thành công chỉ được ghi nhận thành công một lần.

Correct model phải đảm bảo:

chargeCount <= 1

Duplicate PAYMENT_SUCCESS callback không được làm:

chargeCount: 1 → 2

Duplicate callback được nghiên cứu dưới dạng Bug Scenario,
không thuộc luồng hoạt động đúng.

---

# 4. COD Rules

## BR-C01 — COD Does Not Require Prepayment

Đối với:

paymentMethod = COD

order không cần được PAID trước khi giao hàng.

Do đó trạng thái:

COD + SHIPPING + UNPAID

là trạng thái hợp lệ.

---

## BR-C02 — COD Does Not Use Online Payment Processing

COD không sử dụng quy trình Payment Gateway của Online Payment.

Correct model không cho phép COD đi qua:

PROCESSING
FAILED

Không có luồng:

COD
→ START_PAYMENT
→ PROCESSING

---

## BR-C03 — COD Ready To Ship

Sau khi đơn COD được xác nhận,
order có thể chuyển sang:

READY_TO_SHIP

trong khi:

paymentState = UNPAID

---

## BR-C04 — COD Shipping

Một đơn COD ở:

READY_TO_SHIP + UNPAID

có thể chuyển sang:

SHIPPING + UNPAID

Thông qua event:

START_SHIPPING

Việc chưa PAID tại thời điểm SHIPPING là hợp lệ đối với COD.

---

## BR-C05 — COD Delivery Success

Nếu đơn COD đang:

SHIPPING + UNPAID

và:

DELIVERY_SUCCESS

xảy ra đồng thời với việc khách hàng thanh toán thành công,
hệ thống ghi nhận:

paymentState := PAID
orderState := COMPLETED

Luồng:

COD
SHIPPING + UNPAID
→ DELIVERY_SUCCESS
→ COMPLETED + PAID

được xem là luồng thành công.

---

## BR-C06 — COD Delivery Failure

Nếu đơn COD đang:

SHIPPING + UNPAID

và xảy ra một trong các tình huống:

- khách không nhận hàng
- khách từ chối thanh toán
- việc giao hàng không thành công

event:

DELIVERY_FAILED

được ghi nhận.

Sau đó:

orderState := CANCELLED
paymentState := UNPAID

Luồng:

COD
SHIPPING + UNPAID
→ DELIVERY_FAILED
→ CANCELLED + UNPAID

là hợp lệ.

---

## BR-C07 — COD Cancellation Before Shipping

Đơn COD có thể được hủy trước khi bắt đầu shipping.

Ví dụ:

CREATED / READY_TO_SHIP
→ CANCELLED

miễn là order chưa bắt đầu giao.

---

## BR-C08 — No Retry Payment for COD

Các event sau không áp dụng cho COD:

START_PAYMENT
RETRY_PAYMENT
PAYMENT_SUCCESS
PAYMENT_FAILURE
PAYMENT_TIMEOUT

Các event này chỉ thuộc Online Payment flow.

---

# 5. Fulfillment Rules

## BR-F01 — Ready To Ship for Online

Online order chỉ được chuyển sang:

READY_TO_SHIP

khi:

paymentState = PAID

---

## BR-F02 — Ready To Ship for COD

COD order có thể chuyển sang:

READY_TO_SHIP

khi:

paymentState = UNPAID

---

## BR-F03 — Start Shipping

Order chỉ được chuyển sang:

SHIPPING

nếu:

orderState = READY_TO_SHIP

và đáp ứng điều kiện của paymentMethod.

ONLINE:

READY_TO_SHIP + PAID
→ SHIPPING + PAID

COD:

READY_TO_SHIP + UNPAID
→ SHIPPING + UNPAID

---

## BR-F04 — Online Completion

Online order chỉ được chuyển:

SHIPPING → COMPLETED

khi:

paymentState = PAID

Luồng hợp lệ:

ONLINE + SHIPPING + PAID
→ DELIVERY_SUCCESS
→ COMPLETED + PAID

---

## BR-F05 — COD Completion

COD order chỉ được COMPLETED khi:

- delivery thành công
- khách hàng đã thanh toán COD

Kết quả:

COMPLETED + PAID

---

# 6. Safety and Integrity Rules

## BR-S01 — Online Shipping Requires Payment

Đối với Online:

SHIPPING → PAID

---

## BR-S02 — Completed Order Must Be Paid

Bất kể paymentMethod:

COMPLETED → PAID

Không tồn tại correct state:

COMPLETED + UNPAID

---

## BR-S03 — No Duplicate Online Charge

Đối với Online:

chargeCount <= 1

---

## BR-S04 — Cancelled Order Cannot Reactivate

Một order đã:

CANCELLED

không được quay lại:

PENDING_PAYMENT
PROCESSING
READY_TO_SHIP
SHIPPING
COMPLETED

---

## BR-S05 — Completed Order Cannot Reactivate

Một order đã:

COMPLETED

không được quay lại:

PENDING_PAYMENT
PROCESSING
READY_TO_SHIP
SHIPPING
CANCELLED

---

## BR-S06 — Payment Method Cannot Change

Trong mọi execution:

paymentMethod

phải giữ nguyên từ khi model bắt đầu cho tới terminal state.

---

## BR-S07 — Online Processing Cannot Be Cancelled

Nếu:

paymentMethod = ONLINE
AND
paymentState = PROCESSING

thì next state không được là:

CANCELLED

---

## BR-S08 — COD Shipping May Be Unpaid

Không được sử dụng property toàn cục:

SHIPPING → PAID

vì COD cho phép:

COD + SHIPPING + UNPAID

Property "shipping requires paid" chỉ áp dụng cho Online.

---

# 7. Liveness Rules

## BR-L01 — Online Processing Eventually Resolves

Nếu Online Payment đạt:

PROCESSING

thì cuối cùng phải đạt:

PAID

hoặc:

FAILED

Correct model không có execution vô hạn:

PROCESSING
→ PROCESSING
→ PROCESSING
→ ...

---

## BR-L02 — Valid Order Flow Can Reach a Terminal State

Từ các trạng thái hoạt động hợp lệ,
hệ thống phải có đường dẫn tới:

COMPLETED

hoặc:

CANCELLED

Việc formalize property này bằng CTL/LTL sẽ được thực hiện trong
properties.md.

---

# 8. Out of Scope

Business Rules hiện tại không định nghĩa:

- Refund
- Partial payment
- Installment payment
- Authentication
- Authorization
- Inventory
- Voucher
- Tax
- Multiple payment gateways
- Shipping return
- Race conditions
- Concurrent events
- Distributed transactions

Các chức năng này có thể được xem xét trong Future Work.