# Cách viết thủ tục SQL Server theo phong cách CyberSoft

Áp dụng khi viết mới hoặc sửa bất kỳ thủ tục `CP_*` / hàm `FN_*` nào trên SQL Server.
Trước khi viết, đọc 1–2 thủ tục có sẵn trong DB của dự án để theo đúng cách đặt tên và bố trí
(vd `CP_MobiBEXM_HXMPostToDmHD`, `CP_InvoiceSaveSuport_HDA` nếu có; nếu không có thì hỏi người dùng thủ tục mẫu).

## 1. Không dùng INSERT INTO

Không viết `INSERT INTO Bang (cot...) VALUES/SELECT ...`. Tạo dòng trên bảng tạm rồi cập nhật từng trường:

```sql
SELECT TOP 0 * INTO #X FROM dbo.Bang WITH (NOLOCK) WHERE 1=0
EXECUTE dbo.CP_InsertBlankRow N'#X'          --- 1 dòng giá trị mặc định: số 0, chuỗi '', ngày 19000101
UPDATE #X SET Truong1 = @a,Truong2 = @b       --- cập nhật từng trường
```

Chuyển dữ liệu giữa các bảng (khớp theo tên cột, cột thiếu lấy mặc định), sau đó UPDATE phần còn lại:

```sql
EXECUTE dbo.CP_InsertTable N'#Nguon',N'#Dich',N'1=1'
EXECUTE dbo.CP_InsertTable N'#Nguon',N'BangThat',N'1=1'   --- bảng thật: không ghi dbo.
```

Ngoại lệ duy nhất: `INSERT INTO #T EXECUTE thu_tuc` để hứng kết quả trả về của thủ tục
(`CP_SysXMLToTable`, `CP_SysGetSttRec`, `CP_SysGetnoVoucher`...).

## 2. Không CREATE TABLE #X(...)

Dùng biến khuôn, không khai báo mỗi cột một biến:

```sql
DECLARE @Ten NVARCHAR(200) = N'',@So NUMERIC(18,0) = 0
SELECT TOP(0) @Ten AS MA_KH,@Ten AS DIEN_GIAI,@So AS CHON_KM INTO #XPh
INSERT INTO #XPh EXECUTE dbo.CP_SysXMLToTable N'#XPh',@M_StrXml,N'PH',N''
```

Giá trị có số lẻ (số lượng, %, đơn giá) đọc dạng `@Ten` rồi làm sạch và CAST để không bị làm tròn:

```sql
UPDATE #XCt SET SO_LUONG = CASE WHEN ISNUMERIC(SO_LUONG) = 1 THEN SO_LUONG ELSE N'0' END
... CAST(SO_LUONG AS NUMERIC(20,4)) ...
```

## 3. Kiểu biến

- Chuỗi: luôn `NVARCHAR(200)`
- Số: luôn `NUMERIC(18,0)` hoặc `NUMERIC(20,0)` (không dùng INT, BIGINT, số lẻ khác)
- Ngày: luôn `SMALLDATETIME`

## 4. Không dùng transaction

Không `BEGIN TRAN / COMMIT / ROLLBACK`. Nếu bước sau lỗi thì `DELETE` dữ liệu đã ghi ở bước trước.

## 5. Mẫu trạng thái @Status / _Fn

```sql
DECLARE @Status NVARCHAR(200) = N'N',@Note NVARCHAR(200) = N''
-----
IF @Ma_Kh = N'' BEGIN SET @Note = N'Chưa chọn khách hàng';GOTO _Fn END
...
SET @Status = N'Y'
_Fn:
SELECT @Status AS status,@Note AS note   --- luôn trả bảng status
DROP TABLE IF EXISTS #XPh,#XCt
```

Nếu thủ tục con (vd `..._Luu`) tự trả bảng Status khi lỗi thì để nó trả, sau đó vẫn trả bảng status của mình.

## 6. Trình bày

- Thụt lề bằng tab; danh sách tham số dấu phẩy đầu dòng.
- Dòng phân cách `-----`, chú thích `---`.
- Nhiều lệnh ngắn trên một dòng, cách nhau bằng `;`.
- Đọc bảng luôn `WITH (NOLOCK)`.

## 7. Khung thủ tục mẫu

```sql
ALTER PROCEDURE [dbo].[CP_Ten_Thu_Tuc]
	@M_StrXml NVARCHAR(MAX) = N''
	,@M_Ma_Dvcs NVARCHAR(30) = N''
	,@M_User_Name NVARCHAR(50) = N''
AS
BEGIN
	SET NOCOUNT ON;
	DECLARE @Ten NVARCHAR(200) = N'',@So NUMERIC(18,0) = 0,@Ngay SMALLDATETIME = CONVERT(NVARCHAR(8),GETDATE(),112)
		,@Status NVARCHAR(200) = N'N',@Note NVARCHAR(200) = N''
	-----
	--- Đọc XML
	SELECT TOP(0) @Ten AS MA_KH,@So AS LOAI INTO #XPh
	INSERT INTO #XPh EXECUTE dbo.CP_SysXMLToTable N'#XPh',@M_StrXml,N'PH',N''
	-----
	--- Kiểm tra
	IF NOT EXISTS (SELECT 1 FROM #XPh WHERE MA_KH <> N'') BEGIN SET @Note = N'Chưa chọn khách hàng';GOTO _Fn END
	-----
	--- Ghi dữ liệu
	SELECT TOP 0 * INTO #Ph FROM dbo.Bang WITH (NOLOCK) WHERE 1=0
	EXECUTE dbo.CP_InsertBlankRow N'#Ph'
	UPDATE #Ph SET Ma_Kh = a.MA_KH,Ngay_Ct = @Ngay FROM #XPh AS a
	EXECUTE dbo.CP_InsertTable N'#Ph',N'Bang',N'1=1'
	-----
	SET @Status = N'Y'
_Fn:
	SELECT @Status AS status,@Note AS note
	DROP TABLE IF EXISTS #XPh,#Ph
END
```

## 8. Khi deploy bằng sqlcmd

Luôn có `-I` (QUOTED_IDENTIFIER ON) và `-f 65001`. Thiếu `-I` thì thủ tục đọc XML (`.value()`) lỗi lúc chạy.
Không để file `.sql` trong source dự án nếu người dùng không yêu cầu.
