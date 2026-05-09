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

## 3. Thiết kế cơ sở dữ liệu 

### Tạo Database
```sql
CREATE DATABASE [QuanLyCamDo_K235480106025];
GO
USE [QuanLyCamDo_K235480106025];
GO
```
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/d48fb57b-6018-4204-8e84-284903237c49" />
*Tạo database `QuanLyCamDo_K235480106025` thành công*

---
### Bảng `KhachHang`
- `MaKhachHang` là **PK**, tự tăng `IDENTITY(1,1)`
- `SoDienThoai` có ràng buộc `UNIQUE` — dùng để tra cứu khách cũ/mới
```sql
CREATE TABLE [KhachHang] (
    [MaKhachHang]   INT             NOT NULL IDENTITY(1,1),
    [HoTen]         NVARCHAR(150)   NOT NULL,
    [SoDienThoai]   VARCHAR(15)     NOT NULL,
    [SoCCCD]        VARCHAR(20)     NULL,
    [DiaChi]        NVARCHAR(300)   NULL,
    [NgayTao]       DATETIME        NOT NULL DEFAULT GETDATE(),
    CONSTRAINT [PK_KhachHang]           PRIMARY KEY ([MaKhachHang]),
    CONSTRAINT [UQ_KhachHang_SDT]       UNIQUE ([SoDienThoai])
);
GO
```
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/5f7d63ba-98c1-482e-a183-68722f808c1e" />
*Bảng KhachHang với UNIQUE constraint trên SoDienThoai — đảm bảo không tạo hồ sơ trùng*

---
### Bảng `[HopDong]`

- `MaHopDong` là **PK**, `MaKhachHang` là **FK** → KhachHang
- `Deadline1` và `Deadline2` có ràng buộc `CHECK`: Deadline2 > Deadline1 > NgayVay
- `TrangThai` có `CHECK` chỉ nhận 5 giá trị hợp lệ

```sql
CREATE TABLE [HopDong] (
    [MaHopDong]     INT             NOT NULL IDENTITY(1,1),
    [MaKhachHang]   INT             NOT NULL,
    [SoTienVay]     MONEY           NOT NULL,
    [NgayVay]       DATE            NOT NULL DEFAULT CAST(GETDATE() AS DATE),
    [Deadline1]     DATE            NOT NULL,
    [Deadline2]     DATE            NOT NULL,
    [TrangThai]     NVARCHAR(30)    NOT NULL DEFAULT N'Đang vay',
    [GhiChu]        NVARCHAR(500)   NULL,
    CONSTRAINT [PK_HopDong]           PRIMARY KEY ([MaHopDong]),
    CONSTRAINT [FK_HopDong_KhachHang] FOREIGN KEY ([MaKhachHang])
        REFERENCES [KhachHang]([MaKhachHang]),
    CONSTRAINT [CK_HopDong_SoTienVay] CHECK ([SoTienVay] > 0),
    CONSTRAINT [CK_HopDong_Deadline]  CHECK (
        [Deadline2] > [Deadline1] AND [Deadline1] > [NgayVay]
    ),
    CONSTRAINT [CK_HopDong_TrangThai] CHECK ([TrangThai] IN (
        N'Đang vay', N'Quá hạn (nợ xấu)', N'Đang trả góp',
        N'Đã thanh toán', N'Đã thanh lý'
    ))
);
```

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/a20f6209-5515-4ffc-bcc0-ac4c53365747" />

*Bảng HopDong với 2 cột Deadline và CHECK constraint đảm bảo thứ tự thời gian hợp lệ*

---
### Bảng `[TaiSan]`

- `MaTaiSan` là **PK**, `MaHopDong` là **FK** → HopDong
- `TrangThai` theo dõi từng tài sản riêng lẻ: `'Đang cầm cố'` | `'Đã trả khách'` | `'Sẵn sàng thanh lý'` | `'Đã bán thanh lý'`

```sql
CREATE TABLE [TaiSan] (
    [MaTaiSan]      INT             NOT NULL IDENTITY(1,1),
    [MaHopDong]     INT             NOT NULL,
    [TenTaiSan]     NVARCHAR(200)   NOT NULL,
    [MoTa]          NVARCHAR(500)   NULL,
    [GiaTriDinhGia] MONEY           NOT NULL,
    [TrangThai]     NVARCHAR(30)    NOT NULL DEFAULT N'Đang cầm cố',
    CONSTRAINT [PK_TaiSan]           PRIMARY KEY ([MaTaiSan]),
    CONSTRAINT [FK_TaiSan_HopDong]   FOREIGN KEY ([MaHopDong])
        REFERENCES [HopDong]([MaHopDong]),
    CONSTRAINT [CK_TaiSan_GiaTri]    CHECK ([GiaTriDinhGia] > 0),
    CONSTRAINT [CK_TaiSan_TrangThai] CHECK ([TrangThai] IN (
        N'Đang cầm cố', N'Đã trả khách',
        N'Sẵn sàng thanh lý', N'Đã bán thanh lý'
    ))
);
```

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/cb88b3b2-cd87-4daf-8519-25145ea36156" />

*Bảng TaiSan theo dõi trạng thái riêng từng tài sản — quan trọng cho nghiệp vụ gợi ý hoàn trả*

---

### Bảng `[LichSuThanhToan]` — Audit Log

Bảng này **chỉ INSERT, không bao giờ UPDATE hay DELETE**. Mỗi lần khách trả tiền là 1 dòng mới — giữ nguyên dấu vết dòng tiền.

```sql
CREATE TABLE [LichSuThanhToan] (
    [MaLichSu]      INT             NOT NULL IDENTITY(1,1),
    [MaHopDong]     INT             NOT NULL,
    [NgayTra]       DATETIME        NOT NULL DEFAULT GETDATE(),
    [SoTienTra]     MONEY           NOT NULL,
    [NhanVienThu]   NVARCHAR(150)   NOT NULL,
    [DuNoTruocKhi]  MONEY           NOT NULL,
    [DuNoSauKhi]    MONEY           NOT NULL,
    [GhiChu]        NVARCHAR(500)   NULL,
    CONSTRAINT [PK_LichSuThanhToan]  PRIMARY KEY ([MaLichSu]),
    CONSTRAINT [FK_LichSu_HopDong]   FOREIGN KEY ([MaHopDong])
        REFERENCES [HopDong]([MaHopDong]),
    CONSTRAINT [CK_LichSu_SoTienTra] CHECK ([SoTienTra] > 0)
);
```

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/fafd4d37-f669-460f-bd32-9e3a64c85335" />

*Bảng LichSuThanhToan lưu DuNoTruocKhi và DuNoSauKhi — có thể reconstruct toàn bộ lịch sử dòng tiền bất kỳ lúc nào*

---
