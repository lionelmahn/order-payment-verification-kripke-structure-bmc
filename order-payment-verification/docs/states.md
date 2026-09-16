# States — Order Payment Formal Verification

## 1. State Representation

Mỗi state được biểu diễn bằng tuple:

State =
(
    orderState,
    paymentState,
    paymentMethod,
    chargeCount
)

---

## 2. OrderState

OrderState ∈ {

    CREATED,
    PENDING_PAYMENT,
    READY_TO_SHIP,
    SHIPPING,
    COMPLETED,
    CANCELLED

}

---

## 3. PaymentState

PaymentState ∈ {

    UNPAID,
    PROCESSING,
    FAILED,
    PAID

}

PROCESSING và FAILED chỉ áp dụng cho Online Payment.

---

## 4. PaymentMethod

PaymentMethod ∈ {

    ONLINE,
    COD

}

PaymentMethod là immutable trong toàn bộ execution.

---

## 5. ChargeCount

chargeCount được sử dụng để ghi nhận số lần Online Payment SUCCESS
được xử lý.

Đối với Online:

0 = chưa xử lý payment success
1 = đã xử lý đúng một lần
>= 2 = trạng thái lỗi

Đối với COD:

chargeCount luôn bằng 0.

---

# 6. Initial States

Online:

O0 = (CREATED, UNPAID, ONLINE, 0)

COD:

C0 = (CREATED, UNPAID, COD, 0)

Initial state set:

S0 = {O0, C0}

---

# 7. Online States

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

# 8. COD States

C0 = (CREATED, UNPAID, COD, 0)

C1 = (READY_TO_SHIP, UNPAID, COD, 0)

C2 = (SHIPPING, UNPAID, COD, 0)

C3 = (COMPLETED, PAID, COD, 0)

C4 = (CANCELLED, UNPAID, COD, 0)

---

# 9. Complete State Set

S = {

    O0, O1, O2, O3, O4,
    O5, O6, O7, O8,

    C0, C1, C2, C3, C4

}

Total reachable correct states: 14.

---

# 10. Terminal States

The following states are terminal business states:

O6
O7
O8
C3
C4

In the Kripke Structure these states use self-loops.

---

# 11. Important Valid States

The following state is valid:

(SHIPPING, UNPAID, COD, 0)

because COD does not require prepayment.

---

# 12. Invalid State Examples

Online:

(SHIPPING, UNPAID, ONLINE, 0)

(COMPLETED, UNPAID, ONLINE, 0)

(READY_TO_SHIP, FAILED, ONLINE, 0)

(any order state, PAID, ONLINE, chargeCount >= 2)

COD:

(any order state, PROCESSING, COD, 0)

(any order state, FAILED, COD, 0)

(COMPLETED, UNPAID, COD, 0)

These states must not be reachable in the correct model.
They may be introduced intentionally in buggy models for verification
experiments.