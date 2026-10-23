# PLAN – CRS_R003 + CRS_R004: Báo cáo Thăm khách hàng (CST Visiting) & Action after visiting

> **Cập nhật 2026-10-23:** CSV là **một** báo cáo (hai khối cột CRS_R003 và CRS_R004 cạnh nhau), không phải hai báo cáo. Đã triển khai thành một report `CRS_R003_R004` (Visit → CBD → Case). Các mục bên dưới viết theo hướng tách đôi, chỉ còn giá trị tham khảo mapping cột; xem `Report CRS.html` cho bản đúng.

Nguồn yêu cầu: `(CRS_R003 + CRS_R004) Báo cáo Thăm khách hàng(CST Visiting).csv`
Org làm việc: **PARTIAL** (đã kiểm tra bằng describe/query ngày 2026-10-23). QA/PROD làm sau, theo ticket.

## 1. Cách viết (theo mẫu ACC_R001)

Giống ACC_R001, **không viết Apex/LWC** cho báo cáo lưới. Mỗi báo cáo gồm 3 phần metadata:

| Thành phần | ACC_R001 | CRS_R003 | CRS_R004 |
|---|---|---|---|
| Custom Report Type | `D365FO_CustTrans_ACC` | `Visits_with_Customer_Basic_Data` (đã có trong repo) | **tạo mới** `Cases_with_Visit_and_CBD` |
| Report | `ACC_R001` (Summary) | `CRS_R003` | `CRS_R004` |
| Folder | Accounting Reports | CRS Reports (mới) | CRS Reports |

Quy ước lấy từ ACC_R001:
- DeveloperName = mã report (`CRS_R003`, `CRS_R004`); `description` bắt đầu bằng mã + mô tả nghiệp vụ.
- Tên cột tiếng Việt/Anh đặt bằng `displayNameOverride` trong **report type**, không đặt trong report.
- Filter người dùng nhập khi chạy: `isUnlocked=true`, `value` để trống. Filter cố định: `isUnlocked=false`.
- `scope=organization`; hiển thị theo quyền của người chạy.
- Field trong report có dạng `Visit__c$Check_In_Datetime__c`, field qua join `Visit__c.Customer_Basic_Data__r$Total_Vehicles__c`.
- `reportType` ghi kèm hậu tố `__c` (ví dụ `Visits_with_Customer_Basic_Data__c`).

## 2. Hiện trạng đã kiểm tra trên Partial

| Hạng mục | Kết quả |
|---|---|
| Report type `Visits_with_Customer_Basic_Data` | **Có trong repo (commit 7011cf21) nhưng chưa có trên Partial** (retrieve trả "Nothing retrieved") → cần deploy |
| Report CRS_R003 / CRS_R004 | Chưa có (chỉ có 4 report CRS066 trong folder Team MPR) |
| Dữ liệu | 920 Visit, 47 Customer Basic Data (CBD), 17 Case gắn CBD, 7 Case gắn Visit |
| `Case.Due_Date__c` | **Đã có** trên Partial (Date) – yêu cầu "thêm field due date" đã xong ở mức field |
| `Case.Visit__c`, `Case.Customer_Basic_Data__c`, `Case.Case_Result__c`, `Case.Subject1__c` | Đã có |
| Quan hệ Visit → CBD | `CBD.Visit__c` (lookup) → child relationship `Customer_Basic_Data__r` (1 Visit có thể có nhiều CBD → report type dùng **outer join**) |
| Sharing | Visit__c, CBD: Private, `enableReports=false` (giống D365FO_CustTrans__c, báo cáo vẫn chạy được qua report type; người xem chỉ thấy bản ghi mình có quyền) |
| Role CRS | `IVC_CRS_HOD`, `IVC_CRS_HOS`, `IVC_CRS_Staff` |

## 3. Mapping cột – CRS_R003 (CUSTOMER VISITING)

Report dạng **Tabular** (không nhóm, bám sát lưới trong CSV), sắp xếp Ngày thăm giảm dần.

| # | Cột CSV | Nguồn field | Có trong report type? |
|---|---|---|---|
| – | No | số dòng của report (không cần field) | – |
| 1 | Date | `Visit__c.Check_In_Datetime__c` | Có |
| 2 | DLR | `Visit__c.Dealer_Name__r.Search_Name__c` | Có |
| 3 | IVC PIC | `Visit__c.Owner` | Có |
| 4 | Visit member | `Visit__c.Member__c` (formula Team Member) + `Visit__c.Note__c` | Có |
| 5 | Customer Name | `Visit__c.Vendor__r.Name` ("Khách hàng") | Có |
| 6 | Cat. 01 | `Visit__c.Vendor__r.Customer_Tier__c` | Có |
| 7 | Cat. 02 | `CBD.Customer_Rank__c` | Có |
| 8 | Holding Units | `CBD.Total_Vehicles__c` | Có |
| 9 | Holding Units Detail | `CBD.Total_Isuzu/Hino/HYD/Thaco/Mitsu/TYT/FORD/Other__c` (mỗi field 1 cột) | Có |
| 10 | Business | `CBD.Business__c` | Có |
| 11 | Place | `Visit__c.Province__r.Name` | Có |
| 12 | Inquiry | `CBD.Purchase_Plan_Inquiry__c` | Có |
| 13 | CST Comments | `CBD.Customer_Comments__c` | Có |
| 14 | Other Information, Comments | `CBD.Other_Information__c` | Có |

Filter:
- Filter 1: khoảng ngày `Check_In_Datetime__c` (From–To) → dùng `timeFrameFilter` `INTERVAL_CUSTOM`, người dùng chọn lúc chạy.
- Filter 2: Cat.01 → `Vendor__r.Customer_Tier__c` (unlocked, rỗng).
- Filter 3: Cat.02 → `CBD.Customer_Rank__c` (unlocked, rỗng).

Report type hiện đã đủ cột; **chỉ cần bổ sung** `displayNameOverride` tiếng Việt/Anh theo header CSV (hiện chưa có) và, nếu cần, `Visit_Status__c` để lọc chỉ lấy visit đã Completed.

## 4. Mapping cột – CRS_R004 (ACTION AFTER VISITING)

Base object **Case** (tạo report type mới), report **Tabular**, lọc: `Customer_Basic_Data__c` không rỗng **OR** `Visit__c` không rỗng (filter logic `1 OR 2`; hiện 17 + 7 Case).

| # | Cột CSV | Nguồn field |
|---|---|---|
| 1 | CST's Request | `Case.Subject` (ẩn `Subject1__c` trên layout RT = Case) |
| 2 | To do | `Case.Description` |
| 3 | PIC | `Case.Owner` (có thể là Queue) |
| 4 | Phòng ban xử lý | Department của Owner → **field mới** `Case.Owner_Department__c` (formula `Owner:User.Department`) |
| 5 | Kết quả | `Case.Case_Result__c` |
| 6 | Số báo cáo visit plan liên quan | `Case.Visit__r.Visit_Plan__r.Visit_Plan_Code__c` |
| 7 | Due date xử lý | `Case.Due_Date__c` |
| 8 | Progress/Status | `Case.Status` |
| ngữ cảnh | Customer / Ngày thăm | `Case.Customer_Basic_Data__r.Customer_Name__c`, `Case.Visit__r.Check_In_Datetime__c` |

Filter (đúng CSV): Filter 1 = Status; Filter 2 = PIC (Owner); Filter 3 = Due date. Tất cả `isUnlocked=true`, rỗng.
Lưu ý CSV đánh số filter R004 theo cột (7), (3), (6); dịch ra theo bảng trên (Status, PIC, Due date).

## 5. Các bước thực hiện

**Bước 0 – Retrieve chuẩn bị:** retrieve Case (fields Due_Date__c, Visit__c, Customer_Basic_Data__c, Case_Result__c), Visit__c, CBD, Visit_Plan__c, layout/record type Case từ Partial để chắc local khớp org. (Không đụng PROD.)

**Bước 1 – Report type CRS_R003:** sửa `Visits_with_Customer_Basic_Data` – thêm `displayNameOverride`, đặt tên thân thiện (ví dụ "CRS_R003 Visits with Customer Basic Data"); deploy lên Partial.

**Bước 2 – Field mới `Case.Owner_Department__c`:** formula Text; thêm FLS cho các profile/permission set CRS (xem Bước 6). Nếu Case Owner là Queue thì cột rỗng – ghi nhận trong ghi chú.

**Bước 3 – Report type CRS_R004:** tạo `Cases_with_Visit_and_CBD` (base Case, đưa vào các field ở mục 4 qua lookup `Visit__r`, `Visit__r.Visit_Plan__r`, `Customer_Basic_Data__r`).

**Bước 4 – Folder `CRS Reports`:** `ReportFolder` chia sẻ View cho `IVC_CRS_HOD`, `IVC_CRS_HOS`, `IVC_CRS_Staff` (hỏi thêm: có chia cho FLS/SPV/SAL không).

**Bước 5 – Hai report:** viết `CRS_R003.report-meta.xml`, `CRS_R004.report-meta.xml` (Tabular, scope organization, filter như mục 3–4); deploy cùng report type + folder trong **một lần** (`package.xml` đúng API version của org).

**Bước 6 – Phần làm tay (Setup UI, không deploy layout):**
- Thêm `Due_Date__c` (và `Owner_Department__c` nếu muốn) vào Case Page Layout.
- Ẩn `Subject1__c` trên layout/record type = Case.
- Kiểm tra FLS `Case.Due_Date__c` cho profile CRS (describe bằng user admin không thấy field, nghĩa là FLS chưa cấp cho user đó).

**Bước 7 – Kiểm thử (Playwright trên Partial, có ảnh minh chứng đánh số):**
1. Chạy CRS_R003 với khoảng ngày có dữ liệu → đối chiếu 3 visit mẫu với bản ghi Visit/CBD.
2. Lọc Cat.01/Cat.02.
3. Chạy CRS_R004 → đối chiếu 17 + 7 Case; kiểm tra cột Visit Plan Code và Department.
4. Chạy bằng user role `IVC_CRS_Staff` để xác nhận quyền xem (Visit/CBD Private).

**Bước 8 – Tài liệu & bàn giao:** mô tả cách xem báo cáo (link, cách chọn filter) bằng ngôn ngữ nghiệp vụ; QA/PROD triển khai sau theo ticket (PROD vá theo file PROD, không đẩy file Partial).

## 6. Rủi ro / điểm cần BA xác nhận

1. **Customer Name từ Lead hoặc Account** (ghi chú trong CSV): `Visit__c` hiện chỉ có `Vendor__c` (Account) và `Customer_Name__c` (Contact), **không có Lead**. Bản v1 dùng Account; muốn thăm cả khách chưa là khách hàng IVC thì phải thêm lookup Lead – ngoài phạm vi report, cần quyết định riêng.
2. **Cat.01 / Cat.02:** CSV ghi "FLS / SPV" cho Cat.01 và "Big 4, Reefer, Silver" cho Cat.02, nhưng picklist thực tế là `Customer_Tier__c` (FLS Diamond…, SPV Gold…) và `Customer_Rank__c` (Key Account, FLS-CV…, Rear Body…). Giá trị "Big 4", "Reefer" không có trong picklist – cần BA xác nhận có thêm giá trị không.
3. **Holding Units Detail** trong CSV là text mô tả từng xe ("5u QKR…"), còn CBD chỉ có số lượng theo hãng → báo cáo hiển thị số theo hãng, không có chi tiết model.
4. **Visit → nhiều CBD:** 1 Visit có thể có nhiều CBD → report hiện nhiều dòng cho cùng Visit. Cần xác nhận thực tế 1 Visit có tối đa 1 CBD không (có thể kiểm tra bằng query trước khi làm).
5. **Case chưa có dữ liệu đủ:** chỉ 7 Case có Visit; cột Visit Plan Code sẽ trống ở phần còn lại.
6. **Quyền xem:** Visit/CBD Private + `enableReports=false` – người không sở hữu bản ghi sẽ chỉ thấy phần được chia sẻ; CRS_HOD/HOS có thấy toàn bộ hay không phụ thuộc role hierarchy → kiểm tra ở Bước 7.4.
7. **Export:** nếu CRS cần xuất Excel đúng định dạng lưới trong CSV (hai khối R003/R004 cạnh nhau) thì report chỉ xuất được 2 file riêng – cần xác nhận chấp nhận.
