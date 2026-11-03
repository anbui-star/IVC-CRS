# [6411] Test full luồng Claim chi phí hỗ trợ sự kiện (CRS) – Partial

Tài liệu luồng: `Quy trình Claim chi phí hỗ trợ sự kiện ECO-DRIVE & MANTAINANCE.html` (mục 3, 5, 8).
Môi trường: **Partial** (`isuzuvn@isuzu.com.partial`). Không test trên PROD.

---

## 0. Chuẩn bị (làm 1 lần)

### 0.1 Tài khoản cần dùng (Login As từ admin)

| Vai | Profile | Làm gì |
|---|---|---|
| **DLR CS staff** (chủ Campaign) | `DLR_Sales_CS staff` | Tạo Campaign, nộp kế hoạch, nhập Actual cost, nộp báo cáo, Submit SO |
| **Dealer Manager** của đại lý trên | profile DLR manager | Duyệt kế hoạch (bước 2 của CampaignCRSNew) |
| **Staff CRS** – Hồ Thiên Quốc *hoặc* Phạm Mai Trinh | `IVC_SAL_CRS_CRS_Staff` | Nhập IVC Budget Approve (duyệt 1), sửa Actual cost + Approve (duyệt 2), chuyển SO sang DLR Process |
| **Manager CRS** – Khôi | profile Manager CRS | Duyệt kế hoạch (bước cuối CampaignCRSNew) |
| **Admin** | System Administrator | Cấp quyền, giả lập ERP (mục 0.4 và bước 12) |

Hai người Quốc/Trinh chỉ cần 1 người duyệt (FirstResponse).

### 0.2 Kiểm tra đại lý dùng để test (chạy bằng `sf data query --target-org PARTIAL`)

Đại lý phải có **Account Supplier** (mã nhà cung cấp ERP), nếu không bước tạo PO ERP sẽ lỗi.

```sql
-- Đại lý đã dùng test 6411 trước đây (Id 001WC00000oUW6uYAG)
SELECT Id, Name, Account_Supplier__c, Account_Supplier__r.AccountNumber FROM Account WHERE Id = '001WC00000oUW6uYAG'

-- User DLR CS staff thuộc đại lý đó
SELECT Id, Name, Username, Profile.Name, LanguageLocaleKey FROM User
WHERE IsActive = true AND Profile.Name = 'DLR_Sales_CS staff'
```

> Ghi lại: tên đại lý, username DLR CS staff, username Dealer Manager. Dùng đúng các user này cho cả luồng.

### 0.3 Quyền field (phải xong trước khi test)

Trên Partial, 9 profile `DLR_*`:

| Field | Quyền |
|---|---|
| `Event_Organization_Cost__c`, `POSM_Cost__c` | **Chỉ đọc** |
| `IVC_Supported_Event_Cost__c`, `IVC_Supported_POSM_Cost__c` | **Đọc + Sửa** |

Profile CRS đã sửa được cả 4 field. *(Việc này tôi chưa làm vì phiên `sf` hết hạn – xem cuối tài liệu.)*

### 0.4 File chuẩn bị sẵn trên máy

| File | Dùng ở bước |
|---|---|
| `cong-van-su-kien.pdf` (công văn sự kiện) | Bước 2 (nộp kế hoạch) |
| `chung-tu-chi-phi.pdf` (chứng từ chi phí) | Bước 6 (Report & Claim) |
| `hoa-don-dai-ly.pdf` (hóa đơn của đại lý) | Bước 11 (đính kèm vào SO) |

Tạo 3 file PDF bất kỳ cũng được.

---

## 1. Dữ liệu test

### Kịch bản chính (happy path)

| Mục | Giá trị |
|---|---|
| Campaign Name | `TEST 6411 – Eco-Drive 02/11` |
| Record Type | Event |
| Type | Event |
| Start / End Date | hôm nay / hôm nay + 5 ngày |
| Vehicle Type | `CV` |
| Duyệt 1 – Event Organization Cost | **10.000.000** |
| Duyệt 1 – POSM Cost | **5.000.000** |
| Actual cost DLR nhập – IVC Supported Event Cost | **10.000.000** |
| Actual cost DLR nhập – IVC Supported POSM Cost | **8.000.000** |
| Actual cost Trinh/Quốc chỉnh khi duyệt 2 – Event | **10.000.000** (giữ) |
| Actual cost Trinh/Quốc chỉnh khi duyệt 2 – POSM | **5.000.000** (hạ về đúng ngân sách) |
| **Số tiền SO Claim mong đợi** | **15.000.000** |
| Item trên SO line | `CRS_CAM_CVV` (vì Vehicle Type = CV) |

### Kịch bản phụ (làm sau khi happy path xong)

| Mã | Mục đích | Dữ liệu |
|---|---|---|
| **N1** | Nộp thiếu Actual cost bị chặn | Campaign thứ 2, để trống IVC Supported POSM Cost |
| **N2** | Nộp thiếu file bị chặn | Campaign thứ 2, đủ số nhưng chưa đính kèm file |
| **N3** | Reject duyệt 2 | Campaign thứ 3, Reject kèm comment |
| **N4** | Dòng xe LCV | Campaign thứ 4, Vehicle Type = `LCV` → Item `CRS_CAM_LCV` |
| **N5** | Submit SO thiếu hóa đơn bị chặn | Dùng SO ở bước 10 trước khi đính kèm hóa đơn |
| **N8** | Staff CRS Approve bước 1 khi thiếu IVC Budget Approve | Xem bước 4, phần "Kiểm tra bắt buộc" |
| **N6** | Thông báo lỗi song ngữ | Cùng lỗi N1, so sánh DLR (tiếng Việt) và CRS/Admin (tiếng Anh) |

---

## 2. Các bước test (happy path)

> Mỗi bước có **kết quả mong đợi** để anh/chị tick ✅ / ❌. Chụp màn hình mỗi bước nếu cần làm evidence.

### Giai đoạn 1 – Duyệt kế hoạch (CampaignCRSNew)

**Bước 1 – DLR tạo Campaign** *(Login As DLR CS staff, app DLR)*
1. Campaigns → New → Record Type **Event**.
2. Nhập Name, Type, Start/End Date theo bảng dữ liệu, chọn Vehicle Type = `CV`.
3. Save.

Mong đợi:
- ☐ Dialog New hiện **đủ các field** (Name, ngày, Dealer…), không chỉ mỗi section "Actual cost".
- ☐ Section **IVC Budget Approve** (Event Organization Cost, POSM Cost) **chỉ đọc**, DLR không nhập được.
- ☐ Status = `Planned`.

**Bước 2 – DLR nộp kế hoạch**
1. Mở Campaign → tab **Report & Claim** → đính kèm `cong-van-su-kien.pdf`.
2. Bấm nút gửi duyệt (Submit for Approval, process `CampaignCRSNew`).

Mong đợi:
- ☐ Status chuyển `DLR Review` → `IVC Review` (theo từng bước process).

**Bước 3 – Dealer Manager duyệt** *(Login As Dealer Manager)*
- Approve.
- ☐ Status = `Manager Review` sau khi qua bước Staff CRS (bước 4).

**Bước 4 – Staff CRS nhập IVC Budget Approve và duyệt** *(Login As Quốc hoặc Trinh)*
1. Mở Campaign (từ chuông / Items to Approve).
2. Section **IVC Budget Approve**: nhập Event Organization Cost = `10.000.000`, POSM Cost = `5.000.000`. Save.
3. Approve.

**Kiểm tra bắt buộc (làm trước khi nhập):** bấm Approve khi **chưa** nhập Event Organization Cost / POSM Cost (để trống 1 hoặc cả 2).
- ☐ Báo lỗi "Please fill in Event Organization Cost & POSM Cost (IVC Budget Approve) before approving." (Vietnamese user: tiếng Việt), Campaign vẫn `IVC Review`, vẫn ở bước Staff CRS.
- ☐ Nhập `0` vẫn được chấp nhận.

Mong đợi sau khi nhập đủ:
- ☐ Sửa được 2 field trên (do là người duyệt hiện tại).
- ☐ Chuyển sang Manager CRS.

**Bước 5 – Manager CRS (Khôi) duyệt** *(Login As Khôi)*
- Approve.
- ☐ Campaign `Approved`.

Rồi **DLR đổi Status tay sang `In Progress`** *(Login As DLR CS staff, bấm Path → Mark Status)*.
- ☐ Status = `In Progress`.

### Giai đoạn 2 – Báo cáo chi phí và duyệt (CampaignCRS2)

**Bước 6 – DLR nhập Actual cost** *(Login As DLR CS staff)*
1. Tab **Report & Claim** → section **Actual cost**: IVC Supported Event Cost = `10.000.000`, IVC Supported POSM Cost = `8.000.000`.
2. Đính kèm `chung-tu-chi-phi.pdf`.
3. Save.

Mong đợi:
- ☐ Section Actual cost nằm **phía trên** danh sách file, DLR nhập được.
- ☐ Section **IVC Budget Approve** (bên trái) hiện 10tr / 5tr nhưng DLR **không sửa được**.

**Bước 7 – (Kịch bản N1/N2 làm ở đây trên Campaign phụ)** – xem mục 3.

**Bước 8 – DLR nộp báo cáo**
- Bấm **"Nộp xin Phê Duyệt"**.

Mong đợi:
- ☐ Status = `Report Review`.
- ☐ Campaign chờ Quốc/Trinh duyệt.

**Bước 9 – Staff CRS chỉnh Actual cost và Approve** *(Login As Trinh hoặc Quốc)*
1. Mở Campaign → Actual cost: sửa IVC Supported POSM Cost từ `8.000.000` thành `5.000.000`.
2. Approve, ghi comment: `Hạ POSM về đúng ngân sách IVC duyệt`.

Mong đợi:
- ☐ Sửa được Actual cost.
- ☐ Status = `Report Approval`.
- ☐ Tab **SO Claim** có **1 Sales Order** mới, Status `New`, Record Type `CRS Claim`.
- ☐ SO có **1 dòng**: Item `CRS_CAM_CVV`, Quantity 1, Claim Amount **15.000.000**.
- ☐ Trên SO **không có** nút "Submit for Approval"; trường ERP Default Dimension không hiện.

Kiểm tra nhanh bằng SOQL:
```sql
SELECT Id, Name, Status__c, Account__r.Name, Campaign__c,
  (SELECT ItemId__c, Quantity__c, Actual_Amount__c, Claim_Amount__c FROM Sales_Order_Item__r)
FROM Sales_Order__c WHERE Campaign__r.Name = 'TEST 6411 – Eco-Drive 02/11'
```

### Giai đoạn 3 – Sales Order Claim

**Bước 10 – Staff CRS chuyển SO sang DLR Process** *(Login As Trinh/Quốc)*
1. Mở SO → đổi Status `New` → `DLR Process`.

Mong đợi:
- ☐ SO `DLR Process`.
- ☐ DLR nhận **chuông** (icon 🔔) và **email** đề nghị đính kèm hóa đơn + bấm Submit SO (kèm link SO).

**Bước 11 – DLR đính kèm hóa đơn và Submit SO** *(Login As DLR CS staff)*
1. Mở SO (từ thông báo chuông) → nút **Submit SO** hiện (chỉ khi `DLR Process`).
2. **N5:** bấm Submit SO khi chưa có hóa đơn → ☐ báo lỗi tiếng Việt, SO vẫn `DLR Process`.
3. Đính kèm `hoa-don-dai-ly.pdf` vào SO.
4. Bấm **Submit SO**.

Mong đợi:
- ☐ SO = `Open`.
- ☐ Sau vài giây: field **PurchId** (số PO ERP, dạng `PSPO00xxxxx`) được điền vào SO.
- ☐ Campaign vẫn `Report Approval`.

> Lỗi ERP thường gặp: Account đại lý thiếu **Account Supplier**; ERP chưa có item `CRS_CAM_CVV`/`CRS_CAM_LCV` (đã tạo trên UAT).

**Bước 12 – Giả lập ERP thanh toán đủ (VendTrans Remaining = 0)** *(Admin chạy Anonymous Apex)*

Trên thực tế bước này do kế toán IVC làm trên ERP, về DMS qua đồng bộ master data (trễ theo lịch). Để test nhanh, giả lập bằng script dưới (thay `<SO_ID>`):

```apex
Id soId = '<SO_ID>';
D365FO_VendTrans__c vt = new D365FO_VendTrans__c(
    Sales_Order__c = soId,
    AmountCur__c = '-15000000'
);
insert vt;
vt.Settlement__c = '-15000000';
update vt;
System.debug([SELECT Status__c FROM Sales_Order__c WHERE Id = :soId]);
System.debug([SELECT Status FROM Campaign WHERE Id IN (SELECT Campaign__c FROM Sales_Order__c WHERE Id = :soId)]);
```

Mong đợi:
- ☐ SO = `Completed`.
- ☐ Campaign = `Completed`.
- ☐ Thử thanh toán thiếu (Settlement `-5000000`) → SO vẫn `Open`.

> Script này tạo dữ liệu VendTrans giả trên Partial. Sau test tôi sẽ hướng dẫn anh/chị tự xóa tay (tôi không chạy lệnh xóa).

---

## 3. Kịch bản phụ

**N1 – Thiếu Actual cost** *(Login As DLR CS staff, Campaign phụ ở `In Progress`)*
Để trống IVC Supported POSM Cost, đính kèm file, bấm Nộp xin Phê Duyệt.
- ☐ Báo lỗi (DLR: tiếng Việt – "Vui lòng nhập IVC Supported Event Cost và IVC Supported POSM Cost…"), Campaign vẫn `In Progress`.
- ☐ Không đòi Event Organization Cost / POSM Cost khi nộp.

**N2 – Thiếu file**
Nhập đủ 2 Actual cost, **chưa** đính kèm file, bấm nộp.
- ☐ Báo lỗi yêu cầu file đính kèm, Campaign vẫn `In Progress`.

**N3 – Reject duyệt 2**
Nộp đủ, rồi Quốc/Trinh bấm **Reject** kèm comment `Chứng từ chưa rõ`.
- ☐ Campaign về `In Progress`, **không** sinh SO.
- ☐ DLR sửa và nộp lại được.

**N4 – LCV**
Vehicle Type = `LCV`, chạy đến bước 9.
- ☐ SO line Item = `CRS_CAM_LCV`.

**N6 – Song ngữ**
Gây lỗi N1 bằng Login As DLR (ngôn ngữ Vietnamese) rồi bằng Admin/CRS user (English).
- ☐ DLR thấy tiếng Việt, user khác thấy tiếng Anh.

**N7 – Chỉ 1 SO mỗi Campaign**
Sau bước 9, thử đưa Campaign lùi về `Report Review` rồi `Report Approval` lại.
- ☐ Không sinh SO thứ 2.

---

## 4. Việc tôi chưa làm được

1. **Cấp quyền field (mục 0.3)** và kiểm tra giao diện New/Edit/View bằng Playwright: phiên đăng nhập `sf` của Partial hết hạn ("expired access/refresh token"). Cần chạy `sf org login web --alias PARTIAL`, sau đó tôi làm tiếp.
2. **Chưa tạo sẵn dữ liệu trong org** (đại lý, user, Campaign): vì cùng lý do trên, tôi chưa kiểm tra được đại lý nào có Account Supplier. Mục 0.2 là các câu query để xác nhận; sau khi đăng nhập lại tôi sẽ chạy và điền tên cụ thể vào bảng.
