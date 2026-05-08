# Van-Gohn
# Bài tập về nhà - Hệ thống quản lý cầm đồ 

## Thông tin cá nhân 

| Thông tin | Nội Dung |
|---|---|
|**Họ và tên**| Hoàng Đình Hiếu |
|**Mã sinh viên**| K235480106025 |
|**Lớp**| K59.KMT.K01|
|**Môn học**| Hệ quản trị cơ sở dữ liệu |
|**Giảng viên**| Đỗ Duy Cốp |
|**Tên Database**| QuanLyCamDo_K235480106025|

---

## Yêu cầu đầu bài 
Xây dựng hệ thống quản lý hợp đồng vay tiền thế chấp tài sản (cầm đồ) với các đặc thù:
- Cơ chế lãi suất **2 giai đoạn**: lãi đơn trước Deadline1, lãi kép sau Deadline1
- Quản lý **danh mục tài sản thế chấp** theo từng hợp đồng
- Xử lý **thanh lý đồ** khi quá hạn Deadline2
- **Audit Log** ghi lại từng lần khách trả tiền

---

## 1. Mô tả bài toán

Hệ thống quản lý các hợp đồng vay tiền thế chấp tài sản (cầm đồ).  
Đặc điểm chính:
- Một khách hàng có thể có nhiều hợp đồng cầm cố
- Một hợp đồng có thể bao gồm nhiều tài sản thế chấp
- Cơ chế tính lãi 2 giai đoạn: **lãi đơn** (trước Deadline1) và **lãi kép** (sau Deadline1)
- Quản lý trạng thái tài sản và xử lý thanh lý khi quá hạn

---

## 2. Sơ đồ ERD


### Các thực thể và mối quan hệ

| Thực thể | Vai trò | Quan hệ |
|---|---|---|
| `KhachHang` | Người đến cầm đồ | 1 Khách Hàng → nhiều Hợp Đồng |
| `HopDong` | Hợp đồng vay tiền | 1 Hợp Đồng → nhiều Tài Sản |
| `TaiSan` | Tài sản thế chấp | N:M với HĐ |
| `HopDong_TaiSan` | Bảng trung gian (N:M) | Lưu giá trị định giá |
| `LichSuGiaoDich` | Audit log mỗi lần trả | 1 Hợp Đồng → nhiều Giao Dịch |
| `NhanVien` | Người thu tiền | 1 Nhân Viên → nhiều Giao  |

---
