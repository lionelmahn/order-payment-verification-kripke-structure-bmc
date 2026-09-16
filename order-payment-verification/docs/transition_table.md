# Transition Table — Order Payment Formal Verification

## 1. Purpose

Tài liệu này định nghĩa toàn bộ các transition hợp lệ của mô hình
Order Payment Verification.

Transition Table được xây dựng trực tiếp từ:

- scope.md
- business_rules.md
- states.md

Mỗi transition có dạng:

Current State
    -- Event [Guard] / Action -->
Next State

Mỗi transition được xem là atomic.

Các event được xử lý tuần tự.
Correct Model không xét concurrent events hoặc race conditions.

---

# 2. State Representation

Mỗi state có dạng:

State = (
    orderState,
    paymentState,
    paymentMethod,
    chargeCount
)

---

# 3. Online States

O0 = (CREATED, UNPAID, ONLINE, 0)

O1 = (PENDING_PAYMENT, UNPAID, ONLINE, 0)

O2 = (PENDING_PAYMENT, PROCESSING, ONLINE, 0)

O3 = (PENDING_PAYMENT, FAILED, ONLINE, 0)

O4 = (READY_TO_SHIP, PAID, ONLINE, 1)

O5 = (SHIPPING, PAID, ONLINE, 1)

O6 = (COMPLETED, PAID, ONLINE, 1)

O7 = (CANCELLED, UNPAID, ONLINE, 0)

O8 = (CANCELLED, FAILED, ONLINE, 0)

---

# 4. COD States

C0 = (CREATED, UNPAID, COD, 0)

C1 = (READY_TO_SHIP, UNPAID, COD, 0)

C2 = (SHIPPING, UNPAID, COD, 0)

C3 = (COMPLETED, PAID, COD, 0)

C4 = (CANCELLED, UNPAID, COD, 0)

---

# 5. Event Definitions

## CONFIRM_ORDER

Xác nhận một đơn hàng vừa được tạo.

Tùy paymentMethod:

ONLINE:
CREATED → PENDING_PAYMENT

COD:
CREATED → READY_TO_SHIP

---

## CANCEL_ORDER

Hủy đơn hàng khi business rules cho phép.

Online:
- cho phép trước khi payment bắt đầu
- cho phép sau khi payment FAILED
- không cho phép khi PROCESSING
- không cho phép sau PAID

COD:
- cho phép trước khi bắt đầu SHIPPING

---

## START_PAYMENT

Bắt đầu Online Payment lần đầu.

UNPAID → PROCESSING

Chỉ áp dụng cho paymentMethod = ONLINE.

---

## RETRY_PAYMENT

Thử lại Online Payment sau khi lần trước FAILED.

FAILED → PROCESSING

Không giới hạn số lần retry trong mô hình.

---

## PAYMENT_SUCCESS

Payment Gateway thông báo giao dịch Online thành công.

PROCESSING → PAID

Đồng thời:

chargeCount := chargeCount + 1
orderState := READY_TO_SHIP

---

## PAYMENT_FAILURE

Payment Gateway thông báo giao dịch Online thất bại.

PROCESSING → FAILED

---

## PAYMENT_TIMEOUT

Payment Gateway không trả kết quả trong thời gian mong đợi.

PROCESSING → FAILED

PAYMENT_TIMEOUT là event,
không phải PaymentState.

---

## START_SHIPPING

Bắt đầu quá trình giao hàng.

Online:
READY_TO_SHIP + PAID → SHIPPING + PAID

COD:
READY_TO_SHIP + UNPAID → SHIPPING + UNPAID

---

## DELIVERY_SUCCESS

Giao hàng thành công.

Đối với Online:

SHIPPING + PAID
→ COMPLETED + PAID

Đối với COD, event này mang ý nghĩa nguyên tử:

- giao hàng thành công
- khách hàng nhận hàng
- khách hàng thanh toán COD thành công

Do đó:

SHIPPING + UNPAID
→ COMPLETED + PAID

---

## DELIVERY_FAILED

Chỉ được sử dụng trong COD flow của phiên bản hiện tại.

Event đại diện cho một trong các tình huống:

- khách không nhận hàng
- khách từ chối thanh toán
- giao hàng không thành công

Kết quả:

SHIPPING + UNPAID
→ CANCELLED + UNPAID

---

## STAY

Self-loop kỹ thuật cho terminal states trong Kripke Structure.

Không đại diện cho một hành động nghiệp vụ mới.

---

# 6. Online Payment Transition Table

| ID | From | Event | Guard | Action | To |
|----|------|-------|-------|--------|----|
| TO-01 | O0 | CONFIRM_ORDER | method = ONLINE | order := PENDING_PAYMENT | O1 |
| TO-02 | O0 | CANCEL_ORDER | payment = UNPAID | order := CANCELLED | O7 |
| TO-03 | O1 | START_PAYMENT | payment = UNPAID | payment := PROCESSING | O2 |
| TO-04 | O1 | CANCEL_ORDER | payment = UNPAID | order := CANCELLED | O7 |
| TO-05 | O2 | PAYMENT_SUCCESS | payment = PROCESSING | payment := PAID; chargeCount := 1; order := READY_TO_SHIP | O4 |
| TO-06 | O2 | PAYMENT_FAILURE | payment = PROCESSING | payment := FAILED | O3 |
| TO-07 | O2 | PAYMENT_TIMEOUT | payment = PROCESSING | payment := FAILED | O3 |
| TO-08 | O3 | RETRY_PAYMENT | payment = FAILED | payment := PROCESSING | O2 |
| TO-09 | O3 | CANCEL_ORDER | payment = FAILED | order := CANCELLED | O8 |
| TO-10 | O4 | START_SHIPPING | payment = PAID | order := SHIPPING | O5 |
| TO-11 | O5 | DELIVERY_SUCCESS | payment = PAID | order := COMPLETED | O6 |
| TO-12 | O6 | STAY | terminal | no change | O6 |
| TO-13 | O7 | STAY | terminal | no change | O7 |
| TO-14 | O8 | STAY | terminal | no change | O8 |

---

# 7. COD Transition Table

| ID | From | Event | Guard | Action | To |
|----|------|-------|-------|--------|----|
| TC-01 | C0 | CONFIRM_ORDER | method = COD | order := READY_TO_SHIP | C1 |
| TC-02 | C0 | CANCEL_ORDER | payment = UNPAID | order := CANCELLED | C4 |
| TC-03 | C1 | START_SHIPPING | payment = UNPAID | order := SHIPPING | C2 |
| TC-04 | C1 | CANCEL_ORDER | payment = UNPAID | order := CANCELLED | C4 |
| TC-05 | C2 | DELIVERY_SUCCESS | payment = UNPAID | payment := PAID; order := COMPLETED | C3 |
| TC-06 | C2 | DELIVERY_FAILED | payment = UNPAID | order := CANCELLED; payment remains UNPAID | C4 |
| TC-07 | C3 | STAY | terminal | no change | C3 |
| TC-08 | C4 | STAY | terminal | no change | C4 |

---

# 8. Correct Transition Relation

Ignoring event labels, the Kripke transition relation R is:

R_online = {

    (O0, O1),
    (O0, O7),

    (O1, O2),
    (O1, O7),

    (O2, O4),
    (O2, O3),

    (O3, O2),
    (O3, O8),

    (O4, O5),

    (O5, O6),

    (O6, O6),
    (O7, O7),
    (O8, O8)

}

R_cod = {

    (C0, C1),
    (C0, C4),

    (C1, C2),
    (C1, C4),

    (C2, C3),
    (C2, C4),

    (C3, C3),
    (C4, C4)

}

Complete relation:

R = R_online ∪ R_cod

Note:

PAYMENT_FAILURE và PAYMENT_TIMEOUT đều tạo cùng cạnh:

(O2, O3)

Trong Kripke Structure,
hai event khác nhau không nhất thiết tạo hai cặp R khác nhau.

---

# 9. Valid Online Execution Examples

## 9.1 Successful Online Payment

O0
-- CONFIRM_ORDER -->
O1
-- START_PAYMENT -->
O2
-- PAYMENT_SUCCESS -->
O4
-- START_SHIPPING -->
O5
-- DELIVERY_SUCCESS -->
O6

Tương ứng:

CREATED / UNPAID
→
PENDING_PAYMENT / UNPAID
→
PENDING_PAYMENT / PROCESSING
→
READY_TO_SHIP / PAID
→
SHIPPING / PAID
→
COMPLETED / PAID

---

## 9.2 Failed Payment Then Retry Successfully

O0
→ O1
→ O2
→ O3
→ O2
→ O4
→ O5
→ O6

Events:

CONFIRM_ORDER
→ START_PAYMENT
→ PAYMENT_FAILURE
→ RETRY_PAYMENT
→ PAYMENT_SUCCESS
→ START_SHIPPING
→ DELIVERY_SUCCESS

---

## 9.3 Failed Payment Then Cancel

O0
→ O1
→ O2
→ O3
→ O8

Events:

CONFIRM_ORDER
→ START_PAYMENT
→ PAYMENT_FAILURE
→ CANCEL_ORDER

---

## 9.4 Cancel Before Payment

O0
→ O1
→ O7

hoặc:

O0
→ O7

tùy thời điểm người dùng hủy đơn.

---

# 10. Valid COD Execution Examples

## 10.1 Successful COD

C0
-- CONFIRM_ORDER -->
C1
-- START_SHIPPING -->
C2
-- DELIVERY_SUCCESS -->
C3

Tương ứng:

CREATED / UNPAID
→
READY_TO_SHIP / UNPAID
→
SHIPPING / UNPAID
→
COMPLETED / PAID

Điểm quan trọng:

SHIPPING + UNPAID

là hợp lệ đối với COD.

---

## 10.2 COD Delivery Failed

C0
→ C1
→ C2
→ C4

Events:

CONFIRM_ORDER
→ START_SHIPPING
→ DELIVERY_FAILED

Kết quả:

CANCELLED + UNPAID

---

## 10.3 COD Cancel Before Shipping

C0
→ C1
→ C4

hoặc:

C0
→ C4

---

# 11. Forbidden Online Transitions

Các transition sau KHÔNG tồn tại trong Correct Model.

## FT-O01 — Ship Without Payment

O1
→
(SHIPPING, UNPAID, ONLINE, 0)

Forbidden.

Lý do:

Online order phải PAID trước khi SHIPPING.

---

## FT-O02 — Cancel While Processing

O2
→ O7/O8

Forbidden.

Không cho phép CANCEL_ORDER khi:

paymentState = PROCESSING

---

## FT-O03 — Direct UNPAID to PAID

O1
→ O4

Forbidden.

Payment phải đi qua:

UNPAID
→ PROCESSING
→ PAID

---

## FT-O04 — Direct FAILED to PAID

O3
→ O4

Forbidden.

Payment phải retry:

FAILED
→ PROCESSING
→ PAID

---

## FT-O05 — Pay Again After Success

O4
→ O2

Forbidden.

Một giao dịch đã thành công không được quay lại PROCESSING.

---

## FT-O06 — Reactivate Cancelled Order

O7/O8
→ O1/O2/O4/O5

Forbidden.

CANCELLED là terminal business state.

---

## FT-O07 — Reactivate Completed Order

O6
→ O1/O2/O4/O5

Forbidden.

COMPLETED là terminal business state.

---

# 12. Forbidden COD Transitions

## FT-C01 — COD Online Payment Processing

C0/C1/C2
→
COD + PROCESSING

Forbidden.

COD không sử dụng Payment Gateway Online.

---

## FT-C02 — COD Failed Payment State

Không có correct COD state:

paymentState = FAILED

---

## FT-C03 — COD Completed Without Payment

C2
→
(COMPLETED, UNPAID, COD, 0)

Forbidden.

COD chỉ COMPLETED khi giao và thu tiền thành công.

---

## FT-C04 — COD Reactivation After Cancellation

C4
→ C1/C2/C3

Forbidden.

CANCELLED là terminal.

---

# 13. Bug Injection Transitions

Các transition dưới đây không thuộc Correct Model.
Chúng chỉ được thêm vào các buggy model để BMC tìm counterexample.

---

## BUG-01 — Online Shipping Without Payment

Tạo state lỗi:

B1 = (
    SHIPPING,
    UNPAID,
    ONLINE,
    0
)

Thêm transition:

O1
-- BUG_SHIP_WITHOUT_PAYMENT -->
B1

Expected violation:

ONLINE AND SHIPPING AND NOT PAID

Property liên quan:

P1 — Online Shipping Requires Payment

---

## BUG-02 — Duplicate Online Payment Processing

Sau payment thành công:

O4 = (
    READY_TO_SHIP,
    PAID,
    ONLINE,
    1
)

cố tình cho phép duplicate callback.

Tạo state lỗi:

B2 = (
    READY_TO_SHIP,
    PAID,
    ONLINE,
    2
)

Transition:

O4
-- DUPLICATE_PAYMENT_SUCCESS -->
B2

Expected violation:

chargeCount > 1

Property liên quan:

P2 — No Duplicate Online Charge

---

## BUG-03 — Reactivate Cancelled Online Order

Thêm transition:

O8
-- INVALID_RETRY -->
O2

Tương đương:

CANCELLED + FAILED
→
PENDING_PAYMENT + PROCESSING

Expected violation:

CANCELLED state is not terminal.

Property liên quan:

P3 — Cancelled Order Cannot Reactivate

---

## BUG-04 — Processing Never Resolves

Trong correct model:

O2 có successor:

O3
O4

Buggy model thêm:

O2
-- WAIT_FOREVER -->
O2

Điều này cho phép execution:

O2
→ O2
→ O2
→ O2
→ ...

Expected violation:

PROCESSING eventually reaches PAID or FAILED.

Property liên quan:

P4/P5 — Processing Eventually Resolves

Lưu ý:
Việc kiểm chứng liveness bằng BMC cần xử lý loop/lasso semantics
phù hợp với toolkit được sử dụng.

---

## BUG-05 — COD Completed Without Payment

Tùy chọn.

Tạo state lỗi:

B5 = (
    COMPLETED,
    UNPAID,
    COD,
    0
)

Thêm:

C2
-- INVALID_DELIVERY_COMPLETION -->
B5

Expected violation:

COMPLETED implies PAID.

Property liên quan:

Completed Order Must Be Paid.

---

# 14. Transition Invariants

Correct Model phải đảm bảo:

1. paymentMethod không bao giờ thay đổi.

2. Online:
   SHIPPING ⇒ PAID.

3. COD:
   SHIPPING + UNPAID là hợp lệ.

4. COMPLETED ⇒ PAID đối với cả Online và COD.

5. Online chargeCount <= 1.

6. COD chargeCount = 0.

7. PROCESSING chỉ tồn tại đối với ONLINE.

8. FAILED chỉ tồn tại đối với ONLINE.

9. CANCELLED không có successor hoạt động ngoài self-loop.

10. COMPLETED không có successor hoạt động ngoài self-loop.

---

# 15. Consistency Requirements

Trước khi chuyển sang code, Transition Table phải thỏa mãn:

- Mọi transition có source state tồn tại trong states.md.
- Mọi transition có target state tồn tại trong states.md.
- Mọi non-terminal state có ít nhất một successor.
- Mọi terminal state có self-loop.
- Không có transition ONLINE → COD hoặc COD → ONLINE.
- Không có transition COD → PROCESSING.
- Không có transition ONLINE/UNPAID → SHIPPING.
- Không có transition CANCELLED → active state trong Correct Model.
- Không có transition COMPLETED → active state trong Correct Model.