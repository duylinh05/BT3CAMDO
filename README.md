# THIẾT KẾ VÀ CÀI ĐẶT CSDL QUẢN LÝ CẦM ĐỒ

## Mô tả bài toán

Hệ thống cần quản lý các hợp đồng vay tiền thế chấp tài sản. Điểm đặc thù của hệ thống là:

- **Cơ chế tính lãi linh hoạt**: có cả lãi đơn và lãi kép, với hai mốc thời gian khác nhau
- **Quản lý danh mục tài sản thế chấp**: một hợp đồng có thể có nhiều tài sản
- **Xử lý thanh lý đồ khi quá hạn**: tự động chuyển trạng thái hợp đồng và tài sản khi vượt deadline

---

## Các quy tắc nghiệp vụ cần lưu ý

### 1. Mối quan hệ khách hàng - hợp đồng - tài sản

- **Một khách hàng** có thể có **nhiều hợp đồng cầm cố**
- **Một hợp đồng** có thể bao gồm **nhiều tài sản thế chấp** khác nhau

=> Cần thiết kế bảng trung gian để thể hiện quan hệ nhiều-nhiều giữa hợp đồng và tài sản.

### 2. Cơ chế tính lãi suất

**Lãi đơn** (Trước Deadline 1):
- `5.000đ / 1.000.000đ gốc / ngày`
- Tức là `0.5% / ngày`

**Lãi kép** (Từ sau Deadline 1):
- Lãi được tính dựa trên `(Gốc + Lãi đơn đã tích lũy)`
- Từ Deadline 1 đến ngày tính, áp dụng lãi kép trên số tiền gốc cộng lãi đơn

### 3. Trạng thái hợp đồng

Các trạng thái có thể có:

- `DangVay` - hợp đồng đang trong thời hạn vay
- `QuaHan` - hợp đồng đã quá Deadline 1, được xem là nợ xấu
- `DaThanhToan` - khách đã trả hết nợ
- `DaThanhLy` - hợp đồng đã thanh lý tài sản

### 4. Quy tắc trả nợ và chuộc tài sản

- Khách có thể trả nợ **từng phần**
- Chỉ được **lấy lại tài sản** khi:
  - Giá trị các tài sản còn lại **>=** tổng dư nợ hiện tại

=> Đây là điều kiện bảo đảm: tiệm cầm đồ luôn có tài sản đủ giá trị để bảo vệ khoản nợ chưa được thanh toán.

---

## Phân tích thiết kế bảng

Dựa trên các quy tắc nghiệp vụ, hệ thống cần có các bảng sau:

### Bảng 1: Khách hàng (khach_hang)

Lưu thông tin cá nhân của người đi cầm đồ.

| Cột | Kiểu dữ liệu | Mô tả |
|-----|-------------|-------|
| id | INT (IDENTITY) | Khóa chính, tự tăng |
| ho_ten | NVARCHAR(100) | Họ tên khách hàng |
| so_dien_thoai | VARCHAR(20) | Số điện thoại, UNIQUE |
| so_cccd | VARCHAR(20) | Số CCCD, UNIQUE |
| dia_chi | NVARCHAR(255) | Địa chỉ |
| ngay_tao | DATETIME | Ngày tạo hồ sơ, mặc định GETDATE() |

**Ràng buộc**: `so_dien_thoai` và `so_cccd` phải duy nhất để tránh nhập trùng khách hàng.

---

### Bảng 2: Nhân viên (nhan_vien)

Lưu thông tin nhân viên thu tiền hoặc lập hợp đồng.

| Cột | Kiểu dữ liệu | Mô tả |
|-----|-------------|-------|
| id | INT (IDENTITY) | Khóa chính |
| ho_ten | NVARCHAR(100) | Họ tên nhân viên |
| so_dien_thoai | VARCHAR(20) | Số điện thoại |
| chuc_vu | NVARCHAR(50) | Chức vụ |
| ngay_tao | DATETIME | Ngày tạo, mặc định GETDATE() |

---

### Bảng 3: Hợp đồng (hop_dong)

Bảng trung tâm lưu thông tin khoản vay.

| Cột | Kiểu dữ liệu | Mô tả |
|-----|-------------|-------|
| id | INT (IDENTITY) | Khóa chính |
| khach_hang_id | INT | Khóa ngoại → khach_hang.id |
| ngay_lap | DATE | Ngày lập hợp đồng |
| so_tien_goc | DECIMAL(18,2) | Số tiền vay gốc |
| deadline1 | DATE | Mốc 1: hết lãi đơn, bắt đầu lãi kép |
| deadline2 | DATE | Mốc 2: sẵn sàng thanh lý tài sản |
| trang_thai | NVARCHAR(50) | Trạng thái hợp đồng |
| ghi_chu | NVARCHAR(255) | Ghi chú tự do |
| ngay_cap_nhat | DATETIME | Lần cập nhật cuối |

**Ràng buộc**: `khach_hang_id` tham chiếu đến `khach_hang.id`.

---

### Bảng 4: Tài sản (tai_san)

Lưu thông tin tài sản được thế chấp. Trong ví dụ, tài sản là quạt.

| Cột | Kiểu dữ liệu | Mô tả |
|-----|-------------|-------|
| id | INT (IDENTITY) | Khóa chính |
| ten_tai_san | NVARCHAR(100) | Tên tài sản |
| loai_tai_san | NVARCHAR(50) | Loại (quạt cây, quạt trần,...) |
| gia_tri_dinh_gia | DECIMAL(18,2) | Giá trị định giá ban đầu |
| mo_ta | NVARCHAR(255) | Mô tả chi tiết |
| trang_thai | NVARCHAR(50) | Trạng thái tài sản |
| ngay_nhap | DATETIME | Ngày nhập, mặc định GETDATE() |

**Trạng thái tài sản**:
- `DangCamCo` - đang được cầm cố
- `DaTraKhach` - đã trả lại khách hàng
- `SanSangThanhLy` - sẵn sàng thanh lý (hợp đồng quá hạn và vượt deadline2)
- `DaBanThanhLy` - đã bán thanh lý

---

### Bảng 5: Chi tiết hợp đồng - tài sản (chi_tiet_hop_dong)

**Bảng trung gian** thể hiện quan hệ nhiều-nhiều giữa hợp đồng và tài sản.

| Cột | Kiểu dữ liệu | Mô tả |
|-----|-------------|-------|
| id | INT (IDENTITY) | Khóa chính |
| hop_dong_id | INT | Khóa ngoại → hop_dong.id |
| tai_san_id | INT | Khóa ngoại → tai_san.id |
| gia_tri_cam_co | DECIMAL(18,2) | Giá trị tài sản dùng để đảm bảo khoản vay |
| da_tra_tai_san | BIT (0/1) | Đã trả tài sản chưa |
| ngay_tra_tai_san | DATE | Ngày trả tài sản (nếu có) |

**Lưu ý**: `gia_tri_cam_co` có thể khác với `gia_tri_dinh_gia` trong bảng `tai_san`, vì giá trị định giá là giá thị trường, còn giá cầm cố là giá tiệm chấp nhận.

---

### Bảng 6: Lịch sử thanh toán (lich_su_thanh_toan)

**Bảng audit log** - lưu mọi lần khách trả tiền.

| Cột | Kiểu dữ liệu | Mô tả |
|-----|-------------|-------|
| id | INT (IDENTITY) | Khóa chính |
| hop_dong_id | INT | Khóa ngoại → hop_dong.id |
| ngay_thanh_toan | DATETIME | Ngày giờ thanh toán |
| so_tien_tra | DECIMAL(18,2) | Số tiền trả trong lần này |
| nhan_vien_id | INT | Khóa ngoại → nhan_vien.id |
| ghi_chu | NVARCHAR(255) | Ghi chú (ví dụ: "Trả lãi đợt 1") |

**Tại sao cần bảng này?**
- Không được lưu một cột `so_no_con_lai` trong `hop_dong` và ghi đè
- Phải lưu từng giao dịch thanh toán để có thể:
  - Truy vết lịch sử dòng tiền
  - Tính lại công nợ tại bất kỳ thời điểm nào
  - Kiểm tra ai đã thu tiền
  - Đối soát sổ sách

---

## Sơ đồ ERD (mô tả bằng text)

khach_hang (1) -------- (n) hop_dong (n) -------- (n) chi_tiet_hop_dong (1) -------- (n) tai_san
|
|
+-------- (n) lich_su_thanh_toan (1) -------- (1) nhan_vien



**Giải thích quan hệ:**

- `khach_hang` 1-n `hop_dong`: một khách có nhiều hợp đồng
- `hop_dong` n-n `tai_san`: một hợp đồng có nhiều tài sản, một tài sản có thể thuộc nhiều hợp đồng (ở các thời điểm khác nhau) → dùng bảng `chi_tiet_hop_dong`
- `hop_dong` 1-n `lich_su_thanh_toan`: một hợp đồng có nhiều lần thanh toán
- `nhan_vien` 1-n `lich_su_thanh_toan`: một nhân viên có thể thu nhiều lần

---

## Đảm bảo chuẩn hóa 3NF

Thiết kế trên đã đáp ứng chuẩn 3NF:

1. **Không có thuộc tính phụ thuộc một phần khóa chính**: tất cả bảng đều có khóa chính đơn giản (INT identity), không có khóa phức hợp.
2. **Không có thuộc tính phụ thuộc transitive**: tất cả thuộc tính không khóa đều phụ thuộc trực tiếp vào khóa chính.
   - Ví dụ: trong `hop_dong`, `trang_thai` phụ thuộc vào `id`, không phụ thuộc vào `khach_hang_id`.
3. **Không có lặp dữ liệu**: thông tin khách hàng chỉ lưu ở `khach_hang`, không lặp ở `hop_dong`.

---

## Tạo toàn bộ cấu trúc bảng

```sql
/* =========================================================
   TẠO CƠ SỞ DỮ LIỆU VÀ CÁC BẢNG
   ========================================================= */

USE master;
GO

-- Xóa database cũ nếu tồn tại
IF EXISTS (SELECT name FROM sys.databases WHERE name = 'quan_ly_cam_do')
    DROP DATABASE quan_ly_cam_do;
GO

-- Tạo database mới
CREATE DATABASE quan_ly_cam_do;
GO

USE quan_ly_cam_do;
GO

/* ---------------------------------------------------------
   BẢNG KHÁCH HÀNG
   --------------------------------------------------------- */
CREATE TABLE khach_hang (
    id INT IDENTITY(1,1) PRIMARY KEY,
    ho_ten NVARCHAR(100) NOT NULL,
    so_dien_thoai VARCHAR(20) NOT NULL UNIQUE,
    so_cccd VARCHAR(20) NOT NULL UNIQUE,
    dia_chi NVARCHAR(255) NULL,
    ngay_tao DATETIME NOT NULL DEFAULT GETDATE()
);
GO

/* ---------------------------------------------------------
   BẢNG NHÂN VIÊN
   --------------------------------------------------------- */
CREATE TABLE nhan_vien (
    id INT IDENTITY(1,1) PRIMARY KEY,
    ho_ten NVARCHAR(100) NOT NULL,
    so_dien_thoai VARCHAR(20) NULL,
    chuc_vu NVARCHAR(50) NULL,
    ngay_tao DATETIME NOT NULL DEFAULT GETDATE()
);
GO

/* ---------------------------------------------------------
   BẢNG HỢP ĐỒNG
   ---------------------------------------------------------
   Bảng trung tâm lưu thông tin khoản vay.
   Mỗi hợp đồng thuộc về một khách hàng và có 2 mốc thời gian.
   --------------------------------------------------------- */
CREATE TABLE hop_dong (
    id INT IDENTITY(1,1) PRIMARY KEY,
    khach_hang_id INT NOT NULL,
    ngay_lap DATE NOT NULL DEFAULT GETDATE(),
    so_tien_goc DECIMAL(18,2) NOT NULL,
    deadline1 DATE NOT NULL,  -- Mốc chuyển từ lãi đơn sang lãi kép
    deadline2 DATE NOT NULL,  -- Mốc xét điều kiện thanh lý tài sản
    trang_thai NVARCHAR(50) NOT NULL DEFAULT N'DangVay',
    ghi_chu NVARCHAR(255) NULL,
    ngay_cap_nhat DATETIME NOT NULL DEFAULT GETDATE(),
    CONSTRAINT fk_hop_dong_khach_hang
        FOREIGN KEY (khach_hang_id) REFERENCES khach_hang(id)
        ON DELETE NO ACTION
        ON UPDATE CASCADE
);
GO

/* ---------------------------------------------------------
   BẢNG TÀI SẢN
   ---------------------------------------------------------
   Lưu thông tin tài sản thế chấp.
   Trong ví dụ, tài sản là quạt (quạt cây, quạt trần, quạt điều hòa,...).
   --------------------------------------------------------- */
CREATE TABLE tai_san (
    id INT IDENTITY(1,1) PRIMARY KEY,
    ten_tai_san NVARCHAR(100) NOT NULL,
    loai_tai_san NVARCHAR(50) NOT NULL,
    gia_tri_dinh_gia DECIMAL(18,2) NOT NULL,
    mo_ta NVARCHAR(255) NULL,
    trang_thai NVARCHAR(50) NOT NULL DEFAULT N'DangCamCo',
    ngay_nhap DATETIME NOT NULL DEFAULT GETDATE()
);
GO

/* ---------------------------------------------------------
   BẢNG CHI TIẾT HỢP ĐỒNG - TÀI SẢN
   ---------------------------------------------------------
   Bảng liên kết nhiều-nhiều giữa hop_dong và tai_san.
   Mỗi dòng cho biết một tài sản cụ thể nằm trong một hợp đồng cụ thể.
   --------------------------------------------------------- */
CREATE TABLE chi_tiet_hop_dong (
    id INT IDENTITY(1,1) PRIMARY KEY,
    hop_dong_id INT NOT NULL,
    tai_san_id INT NOT NULL,
    gia_tri_cam_co DECIMAL(18,2) NOT NULL,
    da_tra_tai_san BIT NOT NULL DEFAULT 0,
    ngay_tra_tai_san DATE NULL,
    CONSTRAINT fk_chi_tiet_hop_dong_hop_dong
        FOREIGN KEY (hop_dong_id) REFERENCES hop_dong(id)
        ON DELETE CASCADE,
    CONSTRAINT fk_chi_tiet_hop_dong_tai_san
        FOREIGN KEY (tai_san_id) REFERENCES tai_san(id)
        ON DELETE NO ACTION
);
GO

/* ---------------------------------------------------------
   BẢNG LỊCH SỬ THANH TOÁN (AUDIT LOG)
   ---------------------------------------------------------
   Lưu mọi lần khách hàng trả tiền.
   Không được ghi đè số nợ còn lại, mà phải thêm bản ghi mới.
   Từ đây có thể tính tổng đã trả và công nợ tại bất kỳ thời điểm nào.
   --------------------------------------------------------- */
CREATE TABLE lich_su_thanh_toan (
    id INT IDENTITY(1,1) PRIMARY KEY,
    hop_dong_id INT NOT NULL,
    ngay_thanh_toan DATETIME NOT NULL DEFAULT GETDATE(),
    so_tien_tra DECIMAL(18,2) NOT NULL,
    nhan_vien_id INT NOT NULL,
    ghi_chu NVARCHAR(255) NULL,
    CONSTRAINT fk_lich_su_thanh_toan_hop_dong
        FOREIGN KEY (hop_dong_id) REFERENCES hop_dong(id)
        ON DELETE NO ACTION,
    CONSTRAINT fk_lich_su_thanh_toan_nhan_vien
        FOREIGN KEY (nhan_vien_id) REFERENCES nhan_vien(id)
        ON DELETE NO ACTION
);
GO
Chèn dữ liệu mẫu (chủ đề quạt)

/* =========================================================
   DỮ LIỆU MẪU - CHỦ ĐỀ QUẠT
   ========================================================= */

-- Khách hàng mẫu
INSERT INTO khach_hang (ho_ten, so_dien_thoai, so_cccd, dia_chi)
VALUES
(N'Nguyễn Văn An', '0901000001', '001001000001', N'Hà Nội'),
(N'Trần Thị Bình', '0901000002', '001001000002', N'Hải Phòng'),
(N'Lê Văn Cường', '0901000003', '001001000003', N'Đà Nẵng');
GO

-- Nhân viên mẫu
INSERT INTO nhan_vien (ho_ten, so_dien_thoai, chuc_vu)
VALUES
(N'Phạm Thu Hà', '0912000001', N'Thu ngân'),
(N'Hoàng Minh Đức', '0912000002', N'Quản lý');
GO

-- Hợp đồng mẫu
INSERT INTO hop_dong (khach_hang_id, ngay_lap, so_tien_goc, deadline1, deadline2, trang_thai, ghi_chu)
VALUES
(1, '2026-05-01', 3000000, '2026-05-10', '2026-05-20', N'DangVay', N'Cầm quạt cây'),
(2, '2026-05-03', 4500000, '2026-05-12', '2026-05-22', N'DangVay', N'Cầm quạt điều hòa'),
(3, '2026-05-05', 2500000, '2026-05-14', '2026-05-24', N'DangVay', N'Cầm quạt hộp');
GO

-- Tài sản quạt mẫu
INSERT INTO tai_san (ten_tai_san, loai_tai_san, gia_tri_dinh_gia, mo_ta, trang_thai)
VALUES
(N'Quạt cây Panasonic', N'Quạt cây', 5000000, N'Quạt 5 cánh, màu đen', N'DangCamCo'),
(N'Quạt điều hòa Sunhouse', N'Quạt điều hòa', 6500000, N'Có bình nước 20 lít', N'DangCamCo'),
(N'Quạt hộp Senko', N'Quạt hộp', 3500000, N'Quạt hộp mini', N'DangCamCo'),
(N'Quạt treo tường', N'Quạt treo tường', 4000000, N'Điều khiển từ xa', N'DangCamCo');
GO

-- Liên kết hợp đồng với tài sản
INSERT INTO chi_tiet_hop_dong (hop_dong_id, tai_san_id, gia_tri_cam_co, da_tra_tai_san, ngay_tra_tai_san)
VALUES
(1, 1, 5000000, 0, NULL),
(2, 2, 6500000, 0, NULL),
(3, 3, 3500000, 0, NULL),
(3, 4, 4000000, 0, NULL);
GO

-- Lịch sử thanh toán mẫu
INSERT INTO lich_su_thanh_toan (hop_dong_id, ngay_thanh_toan, so_tien_tra, nhan_vien_id, ghi_chu)
VALUES
(1, '2026-05-06 09:00:00', 500000, 1, N'Khách trả đợt 1')

## Event 1: Đăng ký hợp đồng mới (Vay tiền)

Nghiệp vụ này dùng khi cửa hàng tiếp nhận một hợp đồng cầm đồ mới.  
Khi đó hệ thống phải thực hiện đồng thời nhiều việc:

- lưu thông tin khách hàng nếu khách chưa tồn tại
- tạo hợp đồng vay mới
- lưu danh sách tài sản khách đem cầm
- gắn tài sản vào hợp đồng
- thiết lập `deadline1` và `deadline2`

Trong bài này, để procedure có thể tiếp nhận **danh sách tài sản**, ta dùng một **table type**.  
Cách làm này tốt hơn việc truyền từng tài sản bằng nhiều biến rời rạc, vì một hợp đồng có thể có nhiều tài sản.

### Bước 1: Tạo kiểu bảng để nhận danh sách tài sản

```sql
-- Kiểu bảng dùng để truyền danh sách tài sản vào procedure tạo hợp đồng
-- Mỗi dòng tương ứng một tài sản khách đem cầm
CREATE TYPE danh_sach_tai_san AS TABLE (
    ten_tai_san NVARCHAR(100),
    loai_tai_san NVARCHAR(50),
    gia_tri_dinh_gia DECIMAL(18,2),
    mo_ta NVARCHAR(255)
);
GO
Bước 2: Procedure tiếp nhận hợp đồng mới
Procedure dưới đây thực hiện đầy đủ quy trình:

tìm khách theo CCCD
nếu chưa có thì thêm mới
tạo hợp đồng
thêm từng tài sản trong danh sách
liên kết tài sản với hợp đồng

-- Procedure đăng ký hợp đồng mới
-- Nhận vào thông tin khách hàng, số tiền gốc, 2 mốc deadline
-- và danh sách tài sản thế chấp
CREATE OR ALTER PROCEDURE sp_dang_ky_hop_dong_moi
    @ho_ten NVARCHAR(100),
    @so_dien_thoai VARCHAR(20),
    @so_cccd VARCHAR(20),
    @dia_chi NVARCHAR(255),
    @so_tien_goc DECIMAL(18,2),
    @deadline1 DATE,
    @deadline2 DATE,
    @ghi_chu NVARCHAR(255),
    @ds_tai_san danh_sach_tai_san READONLY
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @khach_hang_id INT;
    DECLARE @hop_dong_id INT;

    -- Tìm khách hàng theo CCCD để tránh tạo trùng
    SELECT @khach_hang_id = id
    FROM khach_hang
    WHERE so_cccd = @so_cccd;

    -- Nếu khách chưa tồn tại thì thêm mới
    IF @khach_hang_id IS NULL
    BEGIN
        INSERT INTO khach_hang (ho_ten, so_dien_thoai, so_cccd, dia_chi)
        VALUES (@ho_ten, @so_dien_thoai, @so_cccd, @dia_chi);

        SET @khach_hang_id = SCOPE_IDENTITY();
    END

    -- Tạo hợp đồng vay mới
    INSERT INTO hop_dong (
        khach_hang_id,
        ngay_lap,
        so_tien_goc,
        deadline1,
        deadline2,
        trang_thai,
        ghi_chu,
        ngay_cap_nhat
    )
    VALUES (
        @khach_hang_id,
        CAST(GETDATE() AS DATE),
        @so_tien_goc,
        @deadline1,
        @deadline2,
        N'DangVay',
        @ghi_chu,
        GETDATE()
    );

    SET @hop_dong_id = SCOPE_IDENTITY();

    -- Thêm toàn bộ tài sản vào bảng tai_san
    -- Sau đó gắn vào hợp đồng thông qua bảng chi_tiet_hop_dong
    INSERT INTO tai_san (
        ten_tai_san,
        loai_tai_san,
        gia_tri_dinh_gia,
        mo_ta,
        trang_thai,
        ngay_nhap
    )
    OUTPUT
        @hop_dong_id,
        inserted.id,
        inserted.gia_tri_dinh_gia,
        0,
        NULL
    INTO chi_tiet_hop_dong (
        hop_dong_id,
        tai_san_id,
        gia_tri_cam_co,
        da_tra_tai_san,
        ngay_tra_tai_san
    )
    SELECT
        ten_tai_san,
        loai_tai_san,
        gia_tri_dinh_gia,
        mo_ta,
        N'DangCamCo',
        GETDATE()
    FROM @ds_tai_san;

    -- Trả kết quả để dễ kiểm tra
    SELECT
        @khach_hang_id AS khach_hang_id,
        @hop_dong_id AS hop_dong_id;
END;
GO
Ví dụ gọi Procedure

-- Chuẩn bị danh sách tài sản mẫu
DECLARE @ds danh_sach_tai_san;

INSERT INTO @ds (ten_tai_san, loai_tai_san, gia_tri_dinh_gia, mo_ta)
VALUES
(N'Quat cay Panasonic', N'Quat cay', 5000000, N'Quat 5 canh, mau den'),
(N'Quat hop Senko', N'Quat hop', 3200000, N'Quat hop mini de ban');

-- Gọi procedure tạo hợp đồng mới
EXEC sp_dang_ky_hop_dong_moi
    @ho_ten = N'Pham Van Loc',
    @so_dien_thoai = '0901999999',
    @so_cccd = '001001009999',
    @dia_chi = N'Can Tho',
    @so_tien_goc = 4000000,
    @deadline1 = '2026-05-20',
    @deadline2 = '2026-05-30',
    @ghi_chu = N'Cam 2 chiec quat',
    @ds_tai_san = @ds;
GO
Event 2: Tính toán công nợ thời gian thực
Phần này gồm 2 hàm:

fn_CalcMoneyTransaction(TransactionID, TargetDate)
fn_CalcMoneyContract(ContractID, TargetDate)
Trong thiết kế hiện tại:

TransactionID được hiểu là một dòng trong bảng chi_tiet_hop_dong
ContractID là một hợp đồng trong bảng hop_dong
Ý tưởng tính lãi
Ta sử dụng đúng quy tắc đề bài:

trước deadline1: tính lãi đơn
sau deadline1: tính lãi kép
Mức lãi:

5.000đ / 1.000.000đ / ngày
tương đương 0.005 mỗi ngày
Công thức
Nếu so_tien_goc = P, lai_suat = r, so_ngay = n

Lãi đơn: P * r * n
Lãi kép sau deadline1:
trước hết tính lãi đơn đến deadline1
sau đó lấy (P + lãi đơn) làm cơ sở
rồi áp dụng POWER(1 + r, số_ngày_sau_deadline1)
Function 1: fn_CalcMoneyTransaction
Hàm này tính số tiền phải trả cho một dòng tài sản trong hợp đồng đến ngày cần tính.

Do hợp đồng vay gốc là chung, để phân bổ cho từng transaction, ta chia số tiền gốc theo tỷ lệ giá trị tài sản cầm cố trong tổng giá trị cầm cố của hợp đồng.

Cách làm này hợp lý vì:

một hợp đồng có nhiều tài sản
mỗi transaction là một phần của hợp đồng
cần một cách phân chia gốc để tính riêng từng transaction

-- Hàm tính số tiền phải trả của một transaction (một dòng tài sản trong hợp đồng)
-- đến ngày TargetDate
CREATE OR ALTER FUNCTION fn_CalcMoneyTransaction
(
    @TransactionID INT,
    @TargetDate DATE
)
RETURNS DECIMAL(18,2)
AS
BEGIN
    DECLARE @hop_dong_id INT;
    DECLARE @gia_tri_cam_co DECIMAL(18,2);
    DECLARE @tong_gia_tri_cam_co DECIMAL(18,2);
    DECLARE @so_tien_goc_hop_dong DECIMAL(18,2);
    DECLARE @so_tien_goc_phan_bo DECIMAL(18,2);
    DECLARE @ngay_lap DATE;
    DECLARE @deadline1 DATE;
    DECLARE @lai_suat_ngay DECIMAL(18,6) = 0.005;
    DECLARE @so_ngay_lai_don INT = 0;
    DECLARE @so_ngay_lai_kep INT = 0;
    DECLARE @tien_lai_don DECIMAL(18,2) = 0;
    DECLARE @tong_tien DECIMAL(18,2) = 0;

    -- Lấy thông tin transaction và hợp đồng liên quan
    SELECT
        @hop_dong_id = ct.hop_dong_id,
        @gia_tri_cam_co = ct.gia_tri_cam_co,
        @so_tien_goc_hop_dong = hd.so_tien_goc,
        @ngay_lap = hd.ngay_lap,
        @deadline1 = hd.deadline1
    FROM chi_tiet_hop_dong ct
    INNER JOIN hop_dong hd ON ct.hop_dong_id = hd.id
    WHERE ct.id = @TransactionID;

    -- Nếu transaction không tồn tại
    IF @hop_dong_id IS NULL
        RETURN 0;

    -- Tính tổng giá trị cầm cố của cả hợp đồng
    SELECT @tong_gia_tri_cam_co = SUM(gia_tri_cam_co)
    FROM chi_tiet_hop_dong
    WHERE hop_dong_id = @hop_dong_id;

    IF @tong_gia_tri_cam_co IS NULL OR @tong_gia_tri_cam_co = 0
        RETURN 0;

    -- Phân bổ số tiền gốc của hợp đồng theo tỷ lệ giá trị cầm cố
    SET @so_tien_goc_phan_bo = @so_tien_goc_hop_dong * (@gia_tri_cam_co / @tong_gia_tri_cam_co);

    -- Nếu ngày cần tính trước ngày lập hợp đồng
    IF @TargetDate < @ngay_lap
        RETURN 0;

    -- Trường hợp chỉ có lãi đơn
    IF @TargetDate <= @deadline1
    BEGIN
        SET @so_ngay_lai_don = DATEDIFF(DAY, @ngay_lap, @TargetDate);
        SET @tien_lai_don = @so_tien_goc_phan_bo * @lai_suat_ngay * @so_ngay_lai_don;
        SET @tong_tien = @so_tien_goc_phan_bo + @tien_lai_don;
        RETURN @tong_tien;
    END

    -- Trường hợp có lãi kép
    SET @so_ngay_lai_don = DATEDIFF(DAY, @ngay_lap, @deadline1);
    SET @tien_lai_don = @so_tien_goc_phan_bo * @lai_suat_ngay * @so_ngay_lai_don;
    SET @so_ngay_lai_kep = DATEDIFF(DAY, @deadline1, @TargetDate);

    SET @tong_tien = (@so_tien_goc_phan_bo + @tien_lai_don) * POWER(1 + @lai_suat_ngay, @so_ngay_lai_kep);

    RETURN @tong_tien;
END;
GO
Ví dụ dùng fn_CalcMoneyTransaction

SELECT dbo.fn_CalcMoneyTransaction(1, '2026-05-09') AS so_tien_transaction_1;
GO
Function 2: fn_CalcMoneyContract
Hàm này tính tổng số tiền khách phải trả cho cả hợp đồng đến ngày TargetDate.

Cách tính:

lấy số tiền gốc của hợp đồng
tính lãi đơn trước deadline1
nếu vượt deadline1 thì chuyển sang lãi kép
trừ toàn bộ số tiền khách đã thanh toán từ bảng lich_su_thanh_toan

-- Hàm tính tổng số tiền phải trả của cả hợp đồng
-- đến ngày TargetDate
CREATE OR ALTER FUNCTION fn_CalcMoneyContract
(
    @ContractID INT,
    @TargetDate DATE
)
RETURNS DECIMAL(18,2)
AS
BEGIN
    DECLARE @so_tien_goc DECIMAL(18,2);
    DECLARE @ngay_lap DATE;
    DECLARE @deadline1 DATE;
    DECLARE @lai_suat_ngay DECIMAL(18,6) = 0.005;
    DECLARE @so_ngay_lai_don INT = 0;
    DECLARE @so_ngay_lai_kep INT = 0;
    DECLARE @tien_lai_don DECIMAL(18,2) = 0;
    DECLARE @tong_da_tra DECIMAL(18,2) = 0;
    DECLARE @tong_tien DECIMAL(18,2) = 0;

    -- Lấy thông tin hợp đồng
    SELECT
        @so_tien_goc = so_tien_goc,
        @ngay_lap = ngay_lap,
        @deadline1 = deadline1
    FROM hop_dong
    WHERE id = @ContractID;

    -- Nếu hợp đồng không tồn tại
    IF @so_tien_goc IS NULL
        RETURN 0;

    -- Tính tổng số tiền khách đã trả
    SELECT @tong_da_tra = ISNULL(SUM(so_tien_tra), 0)
    FROM lich_su_thanh_toan
    WHERE hop_dong_id = @ContractID
      AND CAST(ngay_thanh_toan AS DATE) <= @TargetDate;

    -- Nếu ngày cần tính trước ngày lập
    IF @TargetDate < @ngay_lap
        RETURN 0;

    -- Trường hợp chỉ tính lãi đơn
    IF @TargetDate <= @deadline1
    BEGIN
        SET @so_ngay_lai_don = DATEDIFF(DAY, @ngay_lap, @TargetDate);
        SET @tien_lai_don = @so_tien_goc * @lai_suat_ngay * @so_ngay_lai_don;
        SET @tong_tien = @so_tien_goc + @tien_lai_don - @tong_da_tra;

        IF @tong_tien < 0
            SET @tong_tien = 0;

        RETURN @tong_tien;
    END

    -- Trường hợp có lãi kép
    SET @so_ngay_lai_don = DATEDIFF(DAY, @ngay_lap, @deadline1);
    SET @tien_lai_don = @so_tien_goc * @lai_suat_ngay * @so_ngay_lai_don;
    SET @so_ngay_lai_kep = DATEDIFF(DAY, @deadline1, @TargetDate);

    SET @tong_tien = (@so_tien_goc + @tien_lai_don) * POWER(1 + @lai_suat_ngay, @so_ngay_lai_kep) - @tong_da_tra;

    IF @tong_tien < 0
        SET @tong_tien = 0;

    RETURN @tong_tien;
END;
GO
Ví dụ dùng fn_CalcMoneyContract

SELECT dbo.fn_CalcMoneyContract(1, '2026-05-09') AS tong_no_hop_dong_1;
GO

SELECT dbo.fn_CalcMoneyContract(1, GETDATE()) AS tong_no_hien_tai;
GO
Event 3: Xử lý trả nợ và hoàn trả tài sản
Đây là nghiệp vụ phức tạp hơn vì phải xử lý đồng thời:

kiểm tra trạng thái hợp đồng
chặn thanh toán nếu hợp đồng đã thanh lý
ghi nhận tiền khách vừa trả
tính lại dư nợ
nếu trả hết thì trả toàn bộ tài sản
nếu chưa trả hết thì cập nhật trạng thái và gợi ý tài sản có thể hoàn lại
Quy tắc cốt lõi
Khách chỉ được nhận lại tài sản nếu:

giá trị các tài sản còn lại >= dư nợ còn lại
Điều này có nghĩa:

không thể tùy ý trả lại tài sản
phải bảo đảm cửa hàng vẫn giữ đủ giá trị tài sản thế chấp cho phần nợ chưa thanh toán
Procedure xử lý trả nợ từng phần
Procedure dưới đây sẽ:

kiểm tra hợp đồng có tồn tại không
nếu hợp đồng đã thanh lý thì dừng
ghi lịch sử thanh toán
tính lại dư nợ bằng fn_CalcMoneyContract
nếu dư nợ = 0:
cập nhật hợp đồng thành DaThanhToan
trả toàn bộ tài sản
nếu còn nợ:
cập nhật hợp đồng thành DangTraGop
xuất ra danh sách tài sản có thể trả cho khách

-- Procedure xử lý trả nợ từng phần và gợi ý hoàn trả tài sản
CREATE OR ALTER PROCEDURE sp_xu_ly_tra_no
    @hop_dong_id INT,
    @so_tien_tra DECIMAL(18,2),
    @nhan_vien_id INT,
    @ghi_chu NVARCHAR(255) = NULL
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @trang_thai_hop_dong NVARCHAR(50);
    DECLARE @tong_no_truoc_khi_tra DECIMAL(18,2);
    DECLARE @du_no_con_lai DECIMAL(18,2);

    -- Kiểm tra hợp đồng
    SELECT @trang_thai_hop_dong = trang_thai
    FROM hop_dong
    WHERE id = @hop_dong_id;

    IF @trang_thai_hop_dong IS NULL
    BEGIN
        RAISERROR(N'Hop dong khong ton tai.', 16, 1);
        RETURN;
    END

    -- Nếu đã thanh lý thì không nhận tiền
    IF @trang_thai_hop_dong = N'DaThanhLy'
    BEGIN
        RAISERROR(N'Hop dong da thanh ly, khong thu tien va khong tra tai san.', 16, 1);
        RETURN;
    END

    -- Tính tổng nợ trước khi trả
    SET @tong_no_truoc_khi_tra = dbo.fn_CalcMoneyContract(@hop_dong_id, CAST(GETDATE() AS DATE));

    -- Ghi lịch sử thanh toán
    INSERT INTO lich_su_thanh_toan (
        hop_dong_id,
        ngay_thanh_toan,
        so_tien_tra,
        nhan_vien_id,
        ghi_chu
    )
    VALUES (
        @hop_dong_id,
        GETDATE(),
        @so_tien_tra,
        @nhan_vien_id,
        @ghi_chu
    );

    -- Tính lại dư nợ sau khi trả
    SET @du_no_con_lai = dbo.fn_CalcMoneyContract(@hop_dong_id, CAST(GETDATE() AS DATE));

    -- Nếu đã trả hết nợ
    IF @du_no_con_lai <= 0
    BEGIN
        UPDATE hop_dong
        SET
            trang_thai = N'DaThanhToan',
            ngay_cap_nhat = GETDATE()
        WHERE id = @hop_dong_id;

        -- Đánh dấu đã trả toàn bộ tài sản cho khách
        UPDATE chi_tiet_hop_dong
        SET
            da_tra_tai_san = 1,
            ngay_tra_tai_san = CAST(GETDATE() AS DATE)
        WHERE hop_dong_id = @hop_dong_id
          AND da_tra_tai_san = 0;

        -- Cập nhật trạng thái tài sản
        UPDATE ts
        SET ts.trang_thai = N'DaTraKhach'
        FROM tai_san ts
        INNER JOIN chi_tiet_hop_dong ct ON ts.id = ct.tai_san_id
        WHERE ct.hop_dong_id = @hop_dong_id;

        SELECT
            N'Khach da thanh toan het no. Da tra toan bo tai san.' AS thong_bao,
            @tong_no_truoc_khi_tra AS tong_no_truoc_khi_tra,
            0 AS du_no_con_lai;

        RETURN;
    END

    -- Nếu chưa trả hết thì chuyển sang trạng thái đang trả góp
    UPDATE hop_dong
    SET
        trang_thai = N'DangTraGop',
        ngay_cap_nhat = GETDATE()
    WHERE id = @hop_dong_id;

    -- Trả thông tin tổng quan sau khi thanh toán
    SELECT
        N'Khach da tra mot phan. Hop dong chuyen sang DangTraGop.' AS thong_bao,
        @tong_no_truoc_khi_tra AS tong_no_truoc_khi_tra,
        @du_no_con_lai AS du_no_con_lai;

    -- Gợi ý danh sách tài sản có thể trả
    -- Điều kiện: sau khi trả một tài sản, tổng giá trị tài sản còn lại vẫn >= dư nợ còn lại
    ;WITH ds_tai_san_chua_tra AS
    (
        SELECT
            ct.id AS chi_tiet_id,
            ct.tai_san_id,
            ts.ten_tai_san,
            ct.gia_tri_cam_co
        FROM chi_tiet_hop_dong ct
        INNER JOIN tai_san ts ON ct.tai_san_id = ts.id
        WHERE ct.hop_dong_id = @hop_dong_id
          AND ct.da_tra_tai_san = 0
    ),
    tong_gia_tri_con_giu AS
    (
        SELECT SUM(gia_tri_cam_co) AS tong_gia_tri
        FROM ds_tai_san_chua_tra
    )
    SELECT
        d.chi_tiet_id,
        d.tai_san_id,
        d.ten_tai_san,
        d.gia_tri_cam_co,
        (t.tong_gia_tri - d.gia_tri_cam_co) AS gia_tri_con_lai_sau_khi_tra,
        @du_no_con_lai AS du_no_con_lai
    FROM ds_tai_san_chua_tra d
    CROSS JOIN tong_gia_tri_con_giu t
    WHERE (t.tong_gia_tri - d.gia_tri_cam_co) >= @du_no_con_lai;
END;
GO
Ví dụ gọi Procedure xử lý trả nợ

EXEC sp_xu_ly_tra_no
    @hop_dong_id = 1,
    @so_tien_tra = 700000,
    @nhan_vien_id = 1,
    @ghi_chu = N'Khach tra them dot tiep theo';
GO



