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

## Event 1: Đăng Ký Hợp Đồng Mới (Vay Tiền)

### Mô Tả Nghiệp Vụ

Trong hệ thống cầm đồ, quan hệ dữ liệu theo 3 cấp:

- **1 Khách hàng** có thể có **nhiều Hợp đồng** (vay nhiều lần)
- **1 Hợp đồng** có thể cầm cố **nhiều Tài sản** (nhiều món đồ)
- **1 Hợp đồng** phát sinh theo thời gian **nhiều lần biến động** trạng thái hoặc số tiền nợ

### Ý Tưởng (Scenario): "Quầy tiếp nhận đồ cầm — 1 phút xong thủ tục"

**Tình huống:** Khách bước vào, mang theo iPhone, đồng hồ muốn cầm lấy 5 triệu. Nhân viên không cần mở nhiều màn hình — chỉ gọi 1 Stored Procedure, truyền vào thông tin khách, danh sách tài sản và 2 deadline. Hệ thống tự làm phần còn lại: tìm hoặc tạo khách hàng, tạo hợp đồng, lưu từng tài sản, in biên lai.

### Luồng Xử Lý Tổng Quát

- **Bước 1.** Kiểm tra đầu vào: `SoTienVay > 0`, `Deadline1 > NgayVay`, `Deadline2 > Deadline1`. Nếu sai → báo lỗi, thoát.
- **Bước 2.** Tra cứu khách hàng theo `SoDienThoai` — nếu chưa có thì INSERT mới vào `KhachHang`.
- **Bước 3.** INSERT vào `HopDong` với `TrangThai = 'Đang vay'`.
- **Bước 4.** INSERT từng tài sản vào `TaiSan` (tối đa 3 món, món 1 bắt buộc).
- **Bước 5.** Trả về bảng tóm tắt hợp đồng vừa tạo, kèm cột `GiaTriDamBaoDu = TongGiaTriTaiSan - SoTienVay`.

```sql
CREATE PROCEDURE [dbo].[sp_TiepNhanHopDong]
    @HoTen          NVARCHAR(150),
    @SoDienThoai    VARCHAR(15),
    @SoCCCD         VARCHAR(20)   = NULL,
    @DiaChi         NVARCHAR(300) = NULL,
    @SoTienVay      MONEY,
    @NgayVay        DATE          = NULL,
    @Deadline1      DATE,
    @Deadline2      DATE,
    @TenTaiSan1     NVARCHAR(200),
    @MoTaTaiSan1    NVARCHAR(500) = NULL,
    @GiaTri1        MONEY,
    @TenTaiSan2     NVARCHAR(200) = NULL,
    @GiaTri2        MONEY         = NULL,
    @TenTaiSan3     NVARCHAR(200) = NULL,
    @GiaTri3        MONEY         = NULL,
    @GhiChu         NVARCHAR(500) = NULL
AS
BEGIN
    SET NOCOUNT ON;

    -- Bước 1: Kiểm tra đầu vào
    IF @SoTienVay <= 0
    BEGIN
        RAISERROR(N'Số tiền vay phải lớn hơn 0!', 16, 1); RETURN;
    END
    IF @NgayVay IS NULL SET @NgayVay = CAST(GETDATE() AS DATE);
    IF @Deadline1 <= @NgayVay
    BEGIN
        RAISERROR(N'Deadline1 phải sau ngày vay!', 16, 1); RETURN;
    END
    IF @Deadline2 <= @Deadline1
    BEGIN
        RAISERROR(N'Deadline2 phải sau Deadline1!', 16, 1); RETURN;
    END

    -- Bước 2: Tìm hoặc tạo khách hàng
    DECLARE @MaKH INT;
    SELECT @MaKH = [MaKhachHang] FROM [KhachHang]
    WHERE [SoDienThoai] = @SoDienThoai;

    IF @MaKH IS NULL
    BEGIN
        INSERT INTO [KhachHang] ([HoTen],[SoDienThoai],[SoCCCD],[DiaChi])
        VALUES (@HoTen, @SoDienThoai, @SoCCCD, @DiaChi);
        SET @MaKH = SCOPE_IDENTITY();
        PRINT N'Đã tạo khách hàng mới: ' + @HoTen;
    END
    ELSE
        PRINT N'Khách hàng đã tồn tại: ' + @HoTen;

    -- Bước 3: Tạo hợp đồng mới
    INSERT INTO [HopDong]
        ([MaKhachHang],[SoTienVay],[NgayVay],[Deadline1],[Deadline2],[TrangThai],[GhiChu])
    VALUES (@MaKH, @SoTienVay, @NgayVay, @Deadline1, @Deadline2, N'Đang vay', @GhiChu);
    DECLARE @MaHD INT = SCOPE_IDENTITY();

    -- Bước 4: Lưu tài sản thế chấp
    INSERT INTO [TaiSan] ([MaHopDong],[TenTaiSan],[MoTa],[GiaTriDinhGia])
    VALUES (@MaHD, @TenTaiSan1, @MoTaTaiSan1, @GiaTri1);

    IF @TenTaiSan2 IS NOT NULL AND @GiaTri2 IS NOT NULL
        INSERT INTO [TaiSan] ([MaHopDong],[TenTaiSan],[GiaTriDinhGia])
        VALUES (@MaHD, @TenTaiSan2, @GiaTri2);

    IF @TenTaiSan3 IS NOT NULL AND @GiaTri3 IS NOT NULL
        INSERT INTO [TaiSan] ([MaHopDong],[TenTaiSan],[GiaTriDinhGia])
        VALUES (@MaHD, @TenTaiSan3, @GiaTri3);

    PRINT N'Tao hop dong thanh cong! Ma HD: ' + CAST(@MaHD AS NVARCHAR);

    -- Bước 5: Trả về biên lai tóm tắt
    SELECT
        hd.[MaHopDong],
        kh.[HoTen]                            AS [TenKhachHang],
        kh.[SoDienThoai],
        hd.[SoTienVay],
        hd.[NgayVay],
        hd.[Deadline1],
        hd.[Deadline2],
        hd.[TrangThai],
        COUNT(ts.[MaTaiSan])                  AS [SoTaiSan],
        SUM(ts.[GiaTriDinhGia])               AS [TongGiaTriTaiSan],
        SUM(ts.[GiaTriDinhGia]) - hd.[SoTienVay] AS [GiaTriDamBaoDu]
    FROM [HopDong] hd
    JOIN [KhachHang] kh ON hd.[MaKhachHang] = kh.[MaKhachHang]
    JOIN [TaiSan] ts    ON hd.[MaHopDong]   = ts.[MaHopDong]
    WHERE hd.[MaHopDong] = @MaHD
    GROUP BY hd.[MaHopDong], kh.[HoTen], kh.[SoDienThoai],
             hd.[SoTienVay], hd.[NgayVay], hd.[Deadline1],
             hd.[Deadline2], hd.[TrangThai];
END;
GO
```
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/2497efc1-3e6e-46bf-8219-a01137d7f290" />


*Tạo Stored Procedure `sp_TiepNhanHopDong` thành công*

```sql
-- Khai thác: Khách MỚI mang iPhone + đồng hồ đến cầm 5 triệu
EXEC [dbo].[sp_TiepNhanHopDong]
    @HoTen       = N'Nguyễn Văn An',
    @SoDienThoai = '0912345678',
    @SoCCCD      = '001099012345',
    @DiaChi      = N'123 Trần Phú, Hà Nội',
    @SoTienVay   = 5000000,
    @NgayVay     = '2026-04-01',
    @Deadline1   = '2026-05-01',
    @Deadline2   = '2026-06-01',
    @TenTaiSan1  = N'Điện thoại iPhone 14 Pro',
    @MoTaTaiSan1 = N'Màu đen, 256GB, còn bảo hành',
    @GiaTri1     = 8000000,
    @TenTaiSan2  = N'Đồng hồ Casio G-Shock',
    @GiaTri2     = 2500000;
GO
```
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/a7f3615d-7158-41f5-b414-692855acecb4" />


*Tab Messages: "Đã tạo khách hàng mới: Nguyễn Văn An". Tab Results: bảng tóm tắt với TongGiaTriTaiSan = 10.500.000đ, GiaTriDamBaoDu = 5.500.000đ — tài sản dư đảm bảo so với khoản vay*

```sql
-- Khai thác: Cùng khách hàng đó vay thêm lần 2
EXEC [dbo].[sp_TiepNhanHopDong]
    @HoTen       = N'Nguyễn Văn An',
    @SoDienThoai = '0912345678',   -- SĐT cũ → hệ thống tự nhận ra
    @SoTienVay   = 3000000,
    @NgayVay     = '2026-04-10',
    @Deadline1   = '2026-05-10',
    @Deadline2   = '2026-06-10',
    @TenTaiSan1  = N'Laptop Dell Inspiron 15',
    @GiaTri1     = 7000000;
GO
```
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/b8d2654f-d833-4fd1-a9c5-4a70b457eddc" />

*Tab Messages: "Khách hàng đã tồn tại: Nguyễn Văn An" — SP không tạo hồ sơ trùng, chỉ tạo hợp đồng mới gắn MaKH cũ. Đảm bảo dữ liệu không bị duplicate*

---
## Event 2: Tính Toán Công Nợ Thời Gian Thực

### Ý Tưởng (Scenario): "Bộ máy tính tiền chạy ngầm mỗi giây"

**Tình huống:** Khách hỏi "Hôm nay tôi nợ bao nhiêu?" — nhân viên không cần tính tay, chỉ cần gọi hàm với mã hợp đồng và ngày hôm nay. Hệ thống tự áp công thức lãi đơn hoặc lãi kép tùy vào thời điểm, trừ đi các khoản đã trả, trả về con số chính xác đến từng đồng.

### fn_TinhLaiDon — Hàm hỗ trợ tính lãi đơn

```sql
CREATE FUNCTION [dbo].[fn_TinhLaiDon]
(
    @SoTienGoc  MONEY,
    @SoNgay     INT
)
RETURNS MONEY
AS
BEGIN
    -- Lãi suất: 5.000đ / 1.000.000đ / ngày = 0.5%/ngày
    RETURN @SoTienGoc * 0.005 * @SoNgay;
END;
GO
```
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/4cc2bf93-4af8-417a-abc8-516f21c5629f" />


### fn_CalcMoneyTransaction — Tính nợ 1 hợp đồng đến ngày cụ thể

**Luồng xử lý tổng quát:**

- **Bước 1.** Lấy thông tin hợp đồng: `SoTienVay`, `NgayVay`, `Deadline1`
- **Bước 2.** So sánh `TargetDate` với `Deadline1`
- **Bước 3.** Nếu `TargetDate ≤ Deadline1` → tính lãi đơn: `Gốc × 0.005 × SoNgay`
- **Bước 4.** Nếu `TargetDate > Deadline1` → tính nợ tại Deadline1, sau đó áp lãi kép: `GốcMới × (1.005)^N`
- **Bước 5.** Trừ đi tổng các khoản đã trả trong `LichSuThanhToan`
- **Bước 6.** Trả về dư nợ thực tế (tối thiểu = 0)

```sql
CREATE FUNCTION [dbo].[fn_CalcMoneyTransaction]
(
    @MaHopDong  INT,
    @TargetDate DATE
)
RETURNS MONEY
AS
BEGIN
    DECLARE @SoTienVay  MONEY;
    DECLARE @NgayVay    DATE;
    DECLARE @Deadline1  DATE;
    DECLARE @TongNo     MONEY = 0;

    SELECT
        @SoTienVay = [SoTienVay],
        @NgayVay   = [NgayVay],
        @Deadline1 = [Deadline1]
    FROM [HopDong]
    WHERE [MaHopDong] = @MaHopDong;

    IF @SoTienVay IS NULL RETURN 0;

    -- Giai đoạn 1: Lãi đơn (TargetDate còn trong hạn)
    IF @TargetDate <= @Deadline1
    BEGIN
        DECLARE @SoNgayDon INT = DATEDIFF(DAY, @NgayVay, @TargetDate);
        SET @TongNo = @SoTienVay + dbo.[fn_TinhLaiDon](@SoTienVay, @SoNgayDon);
    END
    -- Giai đoạn 2: Lãi kép (TargetDate đã vượt Deadline1)
    ELSE
    BEGIN
        -- Tính nợ tại Deadline1
        DECLARE @SoNgayDenD1 INT = DATEDIFF(DAY, @NgayVay, @Deadline1);
        DECLARE @NoDiD1 MONEY =
            @SoTienVay + dbo.[fn_TinhLaiDon](@SoTienVay, @SoNgayDenD1);

        -- Áp lãi kép từ Deadline1 → TargetDate
        -- Dùng hàm POWER() để tính lũy thừa: GốcMới × (1.005)^N
        DECLARE @SoNgayKep INT = DATEDIFF(DAY, @Deadline1, @TargetDate);
        SET @TongNo = @NoDiD1 * POWER(CAST(1.005 AS FLOAT), @SoNgayKep);
    END

    -- Trừ các khoản đã trả trong LichSuThanhToan
    DECLARE @DaTra MONEY = 0;
    SELECT @DaTra = ISNULL(SUM([SoTienTra]), 0)
    FROM [LichSuThanhToan]
    WHERE [MaHopDong] = @MaHopDong
      AND CAST([NgayTra] AS DATE) <= @TargetDate;

    SET @TongNo = @TongNo - @DaTra;
    IF @TongNo < 0 SET @TongNo = 0;

    RETURN CAST(@TongNo AS MONEY);
END;
GO
```
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/18dce849-36e3-4947-90dd-9308f7e8e97f" />

*Tạo Function `fn_CalcMoneyTransaction` — dùng `POWER()` cho lãi kép, tự động trừ khoản đã trả từ LichSuThanhToan*

```sql
-- Khai thác: Tính nợ hợp đồng số 1 đến hôm nay
SELECT
    dbo.[fn_CalcMoneyTransaction](1, CAST(GETDATE() AS DATE))
    AS [TongNoHopDong1_HomNay];

-- Khai thác: Tính nợ hợp đồng số 2 đến ngày cụ thể
SELECT
    dbo.[fn_CalcMoneyTransaction](2, '2026-06-15')
    AS [TongNoHopDong2_Den15Thang6];
```

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/64ba9ed9-6fac-4fe6-a24d-22e53aa8e0b0" />

*Hàm trả về số tiền nợ chính xác — đã áp đúng giai đoạn lãi đơn hoặc lãi kép tùy ngày, đã trừ các khoản từng trả*

---

### fn_CalcMoneyContract — Tính tổng nợ của 1 khách hàng

**Tình huống:** Khách có 3 hợp đồng đang còn nợ. Kế toán cần biết tổng dư nợ toàn bộ các hợp đồng của khách đó đến ngày hôm nay.

**Luồng xử lý:**
- Dùng CURSOR duyệt qua tất cả hợp đồng chưa kết thúc của khách
- Gọi `fn_CalcMoneyTransaction` cho từng hợp đồng
- Cộng dồn vào tổng

```sql
CREATE FUNCTION [dbo].[fn_CalcMoneyContract]
(
    @MaKhachHang    INT,
    @TargetDate     DATE
)
RETURNS MONEY
AS
BEGIN
    DECLARE @TongNo MONEY = 0;
    DECLARE @MaHD   INT;

    -- Dùng CURSOR để duyệt từng hợp đồng chưa kết thúc
    DECLARE cur_HD CURSOR FOR
        SELECT [MaHopDong]
        FROM [HopDong]
        WHERE [MaKhachHang] = @MaKhachHang
          AND [TrangThai] NOT IN (N'Đã thanh toán', N'Đã thanh lý');

    OPEN cur_HD;
    FETCH NEXT FROM cur_HD INTO @MaHD;

    WHILE @@FETCH_STATUS = 0
    BEGIN
        SET @TongNo = @TongNo
            + dbo.[fn_CalcMoneyTransaction](@MaHD, @TargetDate);
        FETCH NEXT FROM cur_HD INTO @MaHD;
    END;

    CLOSE cur_HD;
    DEALLOCATE cur_HD;

    RETURN @TongNo;
END;
GO
```
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/a654d801-67a6-4ceb-aef7-570ceceb4126" />

*`fn_CalcMoneyContract` dùng CURSOR để gọi lại `fn_CalcMoneyTransaction` cho từng hợp đồng — đảm bảo logic tính lãi nhất quán*

```sql
-- Khai thác: Tổng nợ tất cả hợp đồng của khách hàng số 1 đến hôm nay
SELECT
    kh.[HoTen],
    dbo.[fn_CalcMoneyContract](1, CAST(GETDATE() AS DATE))
    AS [TongNoCuaKhachHang1]
FROM [KhachHang] kh WHERE kh.[MaKhachHang] = 1;
```
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/fd7972a3-fdce-436b-b5af-e113e17fe1f1" />

*Kết quả tổng hợp dư nợ tất cả hợp đồng đang hoạt động của 1 khách — hữu ích khi khách có nhiều lần vay*

---
## Event 3: Xử Lý Trả Nợ và Hoàn Trả Tài Sản

### Ý Tưởng (Scenario): "Quầy thu tiền — mỗi xu đều được ghi lại, không bao giờ mất vết"

**Tình huống:** Khách đến trả 2 triệu cho khoản nợ 5 triệu. Nhân viên cần: tính chính xác dư nợ hiện tại (có lãi kép), ghi log khoản trả, cập nhật trạng thái hợp đồng, và đưa ra gợi ý tài sản nào có thể trả lại cho khách ngay hôm nay.

### Luồng Xử Lý Tổng Quát

- **Bước 1.** Kiểm tra trạng thái hợp đồng — nếu `'Đã thanh lý'` thì thông báo và thoát
- **Bước 2.** Tính dư nợ thực tế hôm nay bằng `fn_CalcMoneyTransaction`
- **Bước 3.** Nếu `SoTienTra > DuNo` thì chỉ thu đúng phần dư nợ
- **Bước 4.** INSERT vào `LichSuThanhToan` (audit log — không ghi đè)
- **Bước 5a.** Nếu trả đủ → cập nhật `HopDong` thành `'Đã thanh toán'`, tất cả `TaiSan` thành `'Đã trả khách'`
- **Bước 5b.** Nếu chưa đủ → cập nhật `HopDong` thành `'Đang trả góp'`
- **Bước 6.** Xuất danh sách gợi ý tài sản có thể trả lại: điều kiện `GiaTriTaiSanConLai ≥ DuNoConLai`

```sql
CREATE PROCEDURE [dbo].[sp_XuLyTraNo]
    @MaHopDong      INT,
    @SoTienTra      MONEY,
    @NhanVienThu    NVARCHAR(150)
AS
BEGIN
    SET NOCOUNT ON;

    -- Lấy thông tin hợp đồng
    DECLARE @TrangThaiHD NVARCHAR(30);
    SELECT @TrangThaiHD = [TrangThai]
    FROM [HopDong] WHERE [MaHopDong] = @MaHopDong;

    IF @TrangThaiHD IS NULL
    BEGIN
        RAISERROR(N'Hợp đồng không tồn tại!', 16, 1); RETURN;
    END

    -- Bước 1: Tài sản đã thanh lý → từ chối thu tiền
    IF @TrangThaiHD = N'Đã thanh lý'
    BEGIN
        PRINT N'[TU CHOI] Hop dong da bi thanh ly tai san.';
        PRINT N'Khong thu tien, khong tra do.';
        RETURN;
    END

    IF @TrangThaiHD = N'Đã thanh toán'
    BEGIN
        PRINT N'Hop dong nay da duoc thanh toan day du.';
        RETURN;
    END

    -- Bước 2: Tính dư nợ hiện tại
    DECLARE @NgayHomNay DATE = CAST(GETDATE() AS DATE);
    DECLARE @DuNo MONEY =
        dbo.[fn_CalcMoneyTransaction](@MaHopDong, @NgayHomNay);

    PRINT N'Du no hien tai: ' + FORMAT(@DuNo, 'N0') + N'd';
    PRINT N'So tien KH tra: ' + FORMAT(@SoTienTra, 'N0') + N'd';

    -- Bước 3: Chỉ thu tối đa bằng dư nợ
    IF @SoTienTra > @DuNo
    BEGIN
        PRINT N'[DIEU CHINH] Chi thu: ' + FORMAT(@DuNo, 'N0') + N'd';
        SET @SoTienTra = @DuNo;
    END

    DECLARE @DuNoSau MONEY = @DuNo - @SoTienTra;

    -- Bước 4: Ghi vào LichSuThanhToan (audit log)
    INSERT INTO [LichSuThanhToan]
        ([MaHopDong],[NgayTra],[SoTienTra],[NhanVienThu],[DuNoTruocKhi],[DuNoSauKhi])
    VALUES
        (@MaHopDong, GETDATE(), @SoTienTra, @NhanVienThu, @DuNo, @DuNoSau);

    -- Bước 5: Cập nhật trạng thái
    IF @DuNoSau = 0
    BEGIN
        -- Trả đủ: đóng hợp đồng và trả hết đồ
        UPDATE [HopDong] SET [TrangThai] = N'Đã thanh toán'
        WHERE [MaHopDong] = @MaHopDong;

        UPDATE [TaiSan] SET [TrangThai] = N'Đã trả khách'
        WHERE [MaHopDong] = @MaHopDong AND [TrangThai] = N'Đang cầm cố';

        PRINT N'[THANH CONG] Thanh toan du! Toan bo tai san da tra lai cho khach.';
    END
    ELSE
    BEGIN
        -- Trả một phần
        UPDATE [HopDong] SET [TrangThai] = N'Đang trả góp'
        WHERE [MaHopDong] = @MaHopDong;

        PRINT N'[GHI NHAN] Da tra mot phan. Du no con lai: '
            + FORMAT(@DuNoSau, 'N0') + N'd';

        -- Bước 6: Gợi ý tài sản có thể trả lại cho khách
        -- Điều kiện: Tổng giá trị tài sản CÒN LẠI sau khi trả bớt
        --            vẫn >= Dư nợ còn lại
        SELECT
            ts.[MaTaiSan],
            ts.[TenTaiSan],
            FORMAT(ts.[GiaTriDinhGia], 'N0') AS [GiaTriDinhGia],
            FORMAT(
                (SELECT ISNULL(SUM(ts2.[GiaTriDinhGia]),0)
                 FROM [TaiSan] ts2
                 WHERE ts2.[MaHopDong] = @MaHopDong
                   AND ts2.[TrangThai] = N'Đang cầm cố'
                   AND ts2.[MaTaiSan] <> ts.[MaTaiSan])
            , 'N0') AS [GiaTriConLaiNeuTraMon],
            CASE
                WHEN (SELECT ISNULL(SUM(ts2.[GiaTriDinhGia]),0)
                      FROM [TaiSan] ts2
                      WHERE ts2.[MaHopDong] = @MaHopDong
                        AND ts2.[TrangThai] = N'Đang cầm cố'
                        AND ts2.[MaTaiSan] <> ts.[MaTaiSan])
                     >= @DuNoSau
                THEN N'Co the tra lai cho khach'
                ELSE N'Chua du dieu kien'
            END AS [GopY]
        FROM [TaiSan] ts
        WHERE ts.[MaHopDong] = @MaHopDong
          AND ts.[TrangThai] = N'Đang cầm cố'
        ORDER BY ts.[GiaTriDinhGia] DESC;
    END
END;
GO
```
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/5e30265b-3e07-4adf-86a1-793d28890cbe" />

*Tạo Stored Procedure `sp_XuLyTraNo` thành công*

```sql
-- Khai thác: Khách trả 2 triệu cho hợp đồng số 1
EXEC [dbo].[sp_XuLyTraNo]
    @MaHopDong   = 1,
    @SoTienTra   = 2000000,
    @NhanVienThu = N'Hoàng Đình Hiếu';
GO
```

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/6a668427-c3bd-4564-9b7e-307e167f586a" />


*Tab Messages: Dư nợ trước/sau khi trả. Tab Results: Danh sách gợi ý tài sản có thể hoàn trả — cột GopY = "Co the tra lai" nghĩa là giá trị tài sản còn lại vẫn đủ bảo đảm cho khoản dư nợ*

```sql
-- Kiểm tra LichSuThanhToan sau khi trả
SELECT
    lst.[MaLichSu],
    kh.[HoTen],
    lst.[MaHopDong],
    FORMAT(lst.[SoTienTra],    'N0') AS [SoTienTra],
    FORMAT(lst.[DuNoTruocKhi], 'N0') AS [DuNoTruocKhi],
    FORMAT(lst.[DuNoSauKhi],   'N0') AS [DuNoSauKhi],
    lst.[NhanVienThu]
FROM [LichSuThanhToan] lst
JOIN [HopDong] hd   ON lst.[MaHopDong]  = hd.[MaHopDong]
JOIN [KhachHang] kh ON hd.[MaKhachHang] = kh.[MaKhachHang]
ORDER BY lst.[NgayTra] DESC;
```

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/3054fcff-81f8-4216-b424-993c27b385a4" />

*Bảng LichSuThanhToan ghi lại đầy đủ: ai trả, bao nhiêu, dư nợ trước và sau — không bao giờ ghi đè, không mất dấu vết dòng tiền*

---
## Event 4: Truy Vấn Danh Sách Nợ Xấu

### Ý Tưởng (Scenario): "Bảng cảnh báo đỏ mỗi sáng của ban giám đốc"

**Tình huống:** Mỗi buổi sáng, giám đốc cần xem danh sách các hợp đồng đã vượt Deadline1 mà chưa thanh toán — kèm dự báo số tiền nợ sẽ là bao nhiêu sau 1 tháng nữa nếu không can thiệp. Cột dự báo giúp đánh giá mức độ rủi ro tăng của từng khách do lãi kép.

```sql
CREATE FUNCTION [dbo].[fn_DanhSachNoXau]()
RETURNS TABLE
AS
RETURN
(
    SELECT
        kh.[HoTen]                              AS [TenKhachHang],
        kh.[SoDienThoai],
        hd.[MaHopDong],
        hd.[SoTienVay]                          AS [SoTienVayGoc],
        hd.[NgayVay],
        hd.[Deadline1],
        DATEDIFF(DAY, hd.[Deadline1], GETDATE()) AS [SoNgayQuaHan],
        dbo.[fn_CalcMoneyTransaction](
            hd.[MaHopDong],
            CAST(GETDATE() AS DATE)
        )                                       AS [TongTienPhaiTraHomNay],
        dbo.[fn_CalcMoneyTransaction](
            hd.[MaHopDong],
            CAST(DATEADD(MONTH, 1, GETDATE()) AS DATE)
        )                                       AS [TongTienPhaiTraSau1Thang],
        hd.[TrangThai]
    FROM [HopDong] hd
    JOIN [KhachHang] kh ON hd.[MaKhachHang] = kh.[MaKhachHang]
    WHERE hd.[Deadline1] < CAST(GETDATE() AS DATE)
      AND hd.[TrangThai] NOT IN (N'Đã thanh toán', N'Đã thanh lý')
);
GO

-- Khai thác
SELECT * FROM dbo.[fn_DanhSachNoXau]()
ORDER BY [SoNgayQuaHan] DESC;
```
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/78aab3ab-948f-45ac-bd12-37dd1f453084" />

*Danh sách nợ xấu sắp xếp theo số ngày quá hạn giảm dần. Cột TongTienPhaiTraSau1Thang cho thấy mức tăng đáng kể do lãi kép — khách nào càng để lâu, gánh nặng tăng càng nhanh*

---
## Event 5: Quản Lý Thanh Lý Tài Sản (Trigger)

### Ý Tưởng (Scenario): "Hệ thống cảnh báo và tự động hóa — không cần nhân viên nhớ"

**Tình huống:** Tiệm cầm đồ có hàng trăm hợp đồng. Không thể hàng ngày ngồi check từng hợp đồng xem cái nào quá hạn. Ba Trigger dưới đây tự động vận hành vòng đời của hợp đồng và tài sản — không cần can thiệp thủ công.

### Trigger 1: trg_CapNhatQuaHan

Khi hợp đồng đang `'Đang vay'` mà đã vượt Deadline1 → tự động chuyển sang `'Quá hạn (nợ xấu)'`.

```sql
CREATE TRIGGER [trg_CapNhatQuaHan]
ON [HopDong]
AFTER INSERT, UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    UPDATE [HopDong]
    SET [TrangThai] = N'Quá hạn (nợ xấu)'
    WHERE [TrangThai] = N'Đang vay'
      AND [Deadline1] < CAST(GETDATE() AS DATE)
      AND [MaHopDong] IN (SELECT [MaHopDong] FROM [inserted]);

    IF @@ROWCOUNT > 0
        PRINT N'[Trigger 1]: Da cap nhat sang "Qua han (no xau)"';
END;
GO
```
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/eb3cf53c-fc6c-4162-acad-2d3f2c9de7de" />

*Trigger 1 kích hoạt sau mỗi INSERT/UPDATE trên HopDong — nếu Deadline1 đã qua mà vẫn "Đang vay" thì tự động chuyển trạng thái*

```sql
-- Demo: Tạo hợp đồng với Deadline1 trong quá khứ để trigger kích hoạt
EXEC [dbo].[sp_TiepNhanHopDong]
    @HoTen       = N'Trần Thị Bình',
    @SoDienThoai = '0987654321',
    @SoTienVay   = 10000000,
    @NgayVay     = '2026-03-01',
    @Deadline1   = '2026-03-31',  -- Đã quá hạn
    @Deadline2   = '2026-04-30',
    @TenTaiSan1  = N'Laptop Dell Inspiron 15',
    @GiaTri1     = 12000000;

-- Kiểm tra: TrangThai đã tự động chuyển chưa?
SELECT [MaHopDong], [TrangThai], [Deadline1]
FROM [HopDong] WHERE [MaKhachHang] = 2;
```
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/3f97e99d-9b3a-42d1-98ac-ff193c471693" />

*Ngay sau khi INSERT, Trigger 1 kích hoạt và TrangThai tự động đổi sang "Quá hạn (nợ xấu)" trong cùng transaction — không cần chạy lệnh UPDATE riêng*

---

### Trigger 2: trg_CapNhatSanSangThanhLy

Khi hợp đồng chuyển sang `'Quá hạn (nợ xấu)'` **và** đã vượt Deadline2 → tự động chuyển tài sản sang `'Sẵn sàng thanh lý'`.

```sql
CREATE TRIGGER [trg_CapNhatSanSangThanhLy]
ON [HopDong]
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    IF UPDATE([TrangThai])
    BEGIN
        UPDATE [TaiSan]
        SET [TrangThai] = N'Sẵn sàng thanh lý'
        WHERE [MaHopDong] IN (
            SELECT i.[MaHopDong]
            FROM [inserted] i
            WHERE i.[TrangThai] = N'Quá hạn (nợ xấu)'
              AND i.[Deadline2] < CAST(GETDATE() AS DATE)
        )
        AND [TrangThai] = N'Đang cầm cố';

        IF @@ROWCOUNT > 0
            PRINT N'[Trigger 2]: Da chuyen tai san sang "San sang thanh ly"';
    END
END;
GO
```
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/0bb2405f-6eb6-42bd-a166-df2f624b3801" />

*Trigger 2 lắng nghe sự kiện UPDATE trên HopDong — khi TrangThai vừa đổi sang "Quá hạn" mà Deadline2 đã qua, tất cả tài sản đang cầm cố được chuyển sang "Sẵn sàng thanh lý"*

---

### Trigger 3: trg_CapNhatDaBanThanhLy

Khi nhân viên cập nhật hợp đồng thành `'Đã thanh lý'` → tự động chuyển toàn bộ tài sản còn lại sang `'Đã bán thanh lý'`.

```sql
CREATE TRIGGER [trg_CapNhatDaBanThanhLy]
ON [HopDong]
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    IF UPDATE([TrangThai])
    BEGIN
        UPDATE [TaiSan]
        SET [TrangThai] = N'Đã bán thanh lý'
        WHERE [MaHopDong] IN (
            SELECT i.[MaHopDong]
            FROM [inserted] i
            WHERE i.[TrangThai] = N'Đã thanh lý'
        )
        AND [TrangThai] IN (N'Đang cầm cố', N'Sẵn sàng thanh lý');

        IF @@ROWCOUNT > 0
            PRINT N'[Trigger 3]: Da chuyen tai san sang "Da ban thanh ly"';
    END
END;
GO
```
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/0682d334-5c37-448a-ad4e-049173d7f076" />

*Ba Trigger hoạt động phối hợp tạo thành vòng đời tự động hoàn chỉnh của hợp đồng và tài sản*

```sql
-- Demo chuỗi Trigger: Cập nhật hợp đồng sang "Đã thanh lý"
UPDATE [HopDong]
SET [TrangThai] = N'Đã thanh lý'
WHERE [MaHopDong] = 4;

-- Kiểm tra: Tài sản đã tự động đổi sang "Đã bán thanh lý" chưa?
SELECT [MaTaiSan], [TenTaiSan], [TrangThai]
FROM [TaiSan]
WHERE [MaHopDong] = 4;
```

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/8f7362bc-7260-4e46-a304-e90c4f8407ca" />

*Tab Messages hiển thị "[Trigger 3]: Da chuyen tai san sang Đã bán thanh lý". Bảng TaiSan tự động cập nhật — không cần nhân viên nhớ chạy lệnh UPDATE riêng cho từng tài sản*

---

## Sự Kiện Bổ Sung 1: Gia Hạn Hợp Đồng

### Ý Tưởng (Scenario): "Khách muốn né lãi kép — trả lãi để reset đồng hồ"

**Tình huống:** Khách gần đến Deadline1 nhưng chưa đủ tiền trả gốc. Họ có thể trả toàn bộ tiền lãi tích lũy để "làm mới" hợp đồng với 2 Deadline mới — tránh bị tính lãi kép, đồng hồ lãi bắt đầu lại từ đầu.

```sql
CREATE PROCEDURE [dbo].[sp_GiaHanHopDong]
    @MaHopDong      INT,
    @Deadline1Moi   DATE,
    @Deadline2Moi   DATE,
    @NhanVienThu    NVARCHAR(150)
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @TrangThaiHD NVARCHAR(30);
    DECLARE @SoTienVay   MONEY;

    SELECT @TrangThaiHD = [TrangThai], @SoTienVay = [SoTienVay]
    FROM [HopDong] WHERE [MaHopDong] = @MaHopDong;

    IF @TrangThaiHD IN (N'Đã thanh toán', N'Đã thanh lý')
    BEGIN
        RAISERROR(N'Hop dong nay da ket thuc, khong the gia han!', 16, 1);
        RETURN;
    END

    DECLARE @NgayHomNay DATE = CAST(GETDATE() AS DATE);

    -- Tính tiền lãi phải trả để gia hạn = TổngNợ - Gốc
    DECLARE @TongNo      MONEY = dbo.[fn_CalcMoneyTransaction](@MaHopDong, @NgayHomNay);
    DECLARE @TienLaiPhaiTra MONEY = @TongNo - @SoTienVay;
    IF @TienLaiPhaiTra < 0 SET @TienLaiPhaiTra = 0;

    -- Ghi log khoản trả lãi
    IF @TienLaiPhaiTra > 0
        INSERT INTO [LichSuThanhToan]
            ([MaHopDong],[NgayTra],[SoTienTra],[NhanVienThu],
             [DuNoTruocKhi],[DuNoSauKhi],[GhiChu])
        VALUES
            (@MaHopDong, GETDATE(), @TienLaiPhaiTra, @NhanVienThu,
             @TongNo, @SoTienVay, N'Tra lai de gia han hop dong');

    -- Cập nhật Deadline mới và reset trạng thái
    UPDATE [HopDong]
    SET [Deadline1] = @Deadline1Moi,
        [Deadline2] = @Deadline2Moi,
        [TrangThai] = N'Đang vay'
    WHERE [MaHopDong] = @MaHopDong;

    PRINT N'Gia han thanh cong!';
    PRINT N'Tien lai da thu: ' + FORMAT(@TienLaiPhaiTra, 'N0') + N'd';
    PRINT N'Deadline1 moi  : ' + CONVERT(NVARCHAR, @Deadline1Moi, 103);
    PRINT N'Deadline2 moi  : ' + CONVERT(NVARCHAR, @Deadline2Moi, 103);
END;
GO

-- Khai thác: Gia hạn hợp đồng số 1
EXEC [dbo].[sp_GiaHanHopDong]
    @MaHopDong    = 1,
    @Deadline1Moi = '2026-06-01',
    @Deadline2Moi = '2026-07-01',
    @NhanVienThu  = N'Hoàng Đình Hiếu';
GO
```
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/87a7d810-188c-4e9e-b756-c45345f12849" />

*SP in ra số tiền lãi đã thu và 2 deadline mới. Hợp đồng reset về "Đang vay" — từ đây lại tính lãi đơn từ đầu. Khoản trả lãi được ghi vào LichSuThanhToan với GhiChu = "Tra lai de gia han hop dong"*

---

## Sự Kiện Bổ Sung 2: Lịch Sử Hợp Đồng (Audit Log)

### Tại Sao Cần Audit Log?

Nếu chỉ lưu một con số "tổng nợ hiện tại" vào bảng HopDong thì khi khách trả từng phần, con số đó bị ghi đè — mất hoàn toàn dấu vết dòng tiền. Không thể trả lời: "Ngày 15/4 khách đã trả bao nhiêu? Ai là người thu tiền? Lúc đó dư nợ là bao nhiêu?"

Bảng `LichSuThanhToan` giải quyết vấn đề này bằng cách **chỉ INSERT, không bao giờ UPDATE hay DELETE**.

```sql
-- Xem toàn bộ lịch sử thanh toán của 1 hợp đồng
SELECT
    lst.[MaLichSu],
    lst.[NgayTra],
    FORMAT(lst.[SoTienTra],    'N0') AS [SoTienTra],
    FORMAT(lst.[DuNoTruocKhi], 'N0') AS [DuNoTruocKhi],
    FORMAT(lst.[DuNoSauKhi],   'N0') AS [DuNoSauKhi],
    lst.[NhanVienThu],
    ISNULL(lst.[GhiChu], N'Thanh toán thông thường') AS [GhiChu]
FROM [LichSuThanhToan] lst
WHERE lst.[MaHopDong] = 1
ORDER BY lst.[NgayTra] ASC;
```
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/6ba7fbdb-819d-4c93-8de8-a285ca8837c3" />

*Mỗi dòng là 1 lần trả tiền — có thể reconstruct toàn bộ lịch sử dòng tiền của hợp đồng bất kỳ lúc nào. DuNoTruocKhi và DuNoSauKhi của dòng N phải khớp với DuNoSauKhi của dòng N-1 — đảm bảo tính liên tục và chính xác*

---

