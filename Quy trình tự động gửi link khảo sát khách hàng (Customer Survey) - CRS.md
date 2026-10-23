# Hướng Dẫn Kiểm Thử (Test Guide / UM) - Tự Động Gửi & Xử Lý Khảo Sát Khách Hàng (CRS Survey)

> **Mã Test Case:** `E2E-02-068` | **Ticket DevOps:** [Issue 5781 [CRS] Test Case ID E2E-02-068_Auto send link cho Customer](https://isuzu-it.visualstudio.com/DMS%20%E2%80%93%20OMN1%20-%20IVC/_workitems/edit/5781)  
> **Phân hệ:** CRS | **Đối tượng áp dụng:** QA Tester / Staff CRS / BA / Consultant  
> **Phiên bản:** Cập nhật 2026-10-22 (Đã rà soát chi tiết codebase Apex & Object DMS)  
> **Mục tiêu:** Before Go-live (Riêng tính năng quét DO hàng ngày theo ngày nhận xe: Sau Go-live)

---

## 1. Toàn Bộ Luồng Tổng Quan

```
[ NGHIỆP VỤ PHÁT SINH TẠI ĐẠI LÝ ]
  ├─ Nhánh 1 (Bán xe): Sales Order Item (DLR to CST) có Actual_Delivery_Date__c = TODAY
  │  └─ Batch Batch_SendSurveyByReceivedDate quét daily 1 lần/ngày
  └─ Nhánh 2 (Dịch vụ): Vehicle_Check_in__c xác nhận Check-out Completed
          │
          ▼
[ BƯỚC 1: TỰ ĐỘNG GỬI SURVEY ]
  ├─ Hệ thống kiểm tra chống gửi trùng (loại bỏ record đã có SurveyInvitation)
  ├─ Tạo SurveyInvitation (Type: 'Sale' hoặc 'Service', OptionsAllowGuestUserResponse = true)
  ├─ Tạo SurveySubject liên kết SO Item hoặc Check-in
  └─ Gửi Email tự động (Template: Email_Survey) đến Contact_Using__c.Email của xe
          │
          ▼
[ BƯỚC 2: KHÁCH HÀNG THỰC HIỆN KHẢO SÁT ]
  └─ Khách mở link Community (SurveyCRS) -> Thực hiện đánh giá & Submit câu trả lời
          │
          ▼
[ BƯỚC 3: XỬ LÝ PHẢN HỒI (Survey Response) ]
  ├─ Ghi nhận điểm số & câu trả lời vào Survey_Result__c
  └─ Phân loại điểm đánh giá (Rating / NPS):
        ├─ TC_04: Đánh giá > 3 sao (4 - 5 sao) ───> Hoàn tất quy trình (Lưu báo cáo CSAT)
        ├─ TC_05: Đánh giá = 3 sao ───────────────> Tự động tạo [ Case ] (Origin: Survey) -> Staff CRS xử lý
        └─ TC_06: Đánh giá < 3 sao (1 - 2 sao) ───> Tự động tạo [ Visit ] (Visit Survey) -> Staff CRS thăm khách
```

---

## 2. Chuẩn Bị Dữ Liệu Trước Khi Test (Prerequisites)

Để test thành công trên môi trường Sandbox/Partial, tester bắt buộc phải chuẩn bị các điều kiện sau:

1. **Cấu hình Sandbox Deliverability (Bắt buộc để nhận được Email):**
   * Vào **Setup** $\rightarrow$ Tìm **Deliverability**.
   * Đảm bảo trường **Access level** được chọn là **All email** (nếu đang là *No access* thì Salesforce sẽ chặn toàn bộ email gửi ra ngoài).
2. **Email người nhận (Tester Email):**
   * Đã cấu hình email nhận khảo sát: **`an.bui@omn1solution.com`**
3. **Bộ Dữ Liệu Test Đã Khởi Tạo Sẵn Trên Môi Trường PARTIAL:**
   * 🚗 **Xe (Vehicle):** [VINTESTCRS2026001 (Biển số: 51A-999.99)](https://ivcl--partial.sandbox.lightning.force.com/lightning/r/Vehicle__c/a25WC000005NpADYA0/view) (`Id: a25WC000005NpADYA0`)
   * 👤 **Người nhận xe (Contact Using):** [Bùi An (Tester Survey)](https://ivcl--partial.sandbox.lightning.force.com/lightning/r/Contact/003WC000158JjiYAE/view) (`Id: 003WC000158JjiYAE` - Email: `an.bui@omn1solution.com`)
   * 🏢 **Khách hàng sở hữu (Current Owner):** [Khách Hàng Test Survey CRS](https://ivcl--partial.sandbox.lightning.force.com/lightning/r/Account/001WC00000w0vQ4YAI/view) (`Id: 001WC00000w0vQ4YAI`)
   * 🏬 **Đại lý liên quan (Dealer):** [Công ty Cơ Khí Ô tô Sài Gòn (SAMCO)](https://ivcl--partial.sandbox.lightning.force.com/lightning/r/Account/001Vu00002h45RCIAY/view) (`Id: 001Vu00002h45RCIAY`)
4. **Community & Survey Template:**
   * Community Network `SurveyCRS` phải đang ở trạng thái **Active/Published**.
   * Survey Template `ĐÁNH GIÁ TRẢI NGHIỆM MUA XE` (hoặc fallback `Mail - Đánh giá mức độ hài lòng`) phải được kích hoạt.
   * Email Template có DeveloperName là `Email_Survey` phải tồn tại trên hệ thống.

---

## 3. Từng Bước Thực Hiện Chi Tiết Từng Test Case (Ai làm gì)

### TC_01: Bán xe (Vehicle Sales) – Tự động gửi link khảo sát mua xe

| Bước | Ai | Làm gì | Status sau bước |
|:---:|---|---|---|
| **1** | DLR Sales staff / Admin | Tạo Sales Order (Record Type: `Vehicle DLR to CST`), thêm `Sales_Order_Item__c`, gán thông tin Xe (`Vehicle__c`). | `Draft` / `Open` |
| **2** | DLR Sales staff | Bàn giao xe cho khách hàng, cập nhật ngày nhận xe thực tế: trường `Actual_Delivery_Date__c = TODAY` [Ship out date] trên SO Item. | `Completed` |
| **3** | Hệ thống (hoặc Tester) | **Cách 1 (Nhanh nhất):** Vào trang [*Run Test*](https://ivcl--partial.sandbox.lightning.force.com/lightning/n/Run_Test) $\rightarrow$ Bấm nút **"Run Survey by Received Date (Daily DO Scan)"**.<br>**Cách 2:** Mở Developer Console chạy: `Database.executeBatch(new Batch_SendSurveyByReceivedDate());` | Tự động tạo `SurveyInvitation` (Type: `Sale`) & `SurveySubject` |
| **4** | Hệ thống | Gửi email tự động (Template `Email_Survey`) đến `Contact_Using__c.Email` của xe. | `Sent` (Email đã gửi) |
| **5** | Khách hàng (Tester đóng vai) | Mở hộp thư email, nhận email tiêu đề *"Xin ý kiến đánh giá mua xe tại [Tên đại lý]"*, bấm vào nút link khảo sát. | Truy cập thành công trang khảo sát (HTTP 200, **không bị 404**) |

* Khi Batch chạy xong (Bước 3), bản ghi `SurveyInvitation` được tạo với `OptionsAllowGuestUserResponse = true`, liên kết SO Item và Xe.
* Email chỉ gửi thành công nếu `Contact_Using__c` có email hợp lệ và Deliverability trên Sandbox là *All email*.
* Kiểm tra nhanh: `SELECT Id, InvitationLink FROM SurveyInvitation WHERE Sales_Order_Item__c = '<Id_Item>'`

---

### TC_02: Dịch vụ xưởng (Service) – Tự động gửi link khảo sát dịch vụ

| Bước | Ai | Làm gì | Status sau bước |
|:---:|---|---|---|
| **1** | DLR Service Adviser | Tạo phiếu tiếp nhận xe `Vehicle_Check_in__c`, chọn xe `VIN__c`, nhập yêu cầu sửa chữa/bảo dưỡng. | `Check In Completed` |
| **2** | DLR Technician / Adviser | Tiến hành bảo dưỡng/sửa chữa xe cho khách theo quy trình xưởng. | `In Progress` |
| **3** | DLR Service Adviser | Hoàn tất dịch vụ, bàn giao xe cho khách và xác nhận Check-out trên `Vehicle_Check_in__c`. | `Check Out Completed` |
| **4** | Hệ thống (Trigger) | `TriggerVehicleCheckinHandler.sendSurveyCheckOut` tự động bắt sự kiện, tạo `SurveyInvitation` (Type: `Service`) và `SurveySubject`. | `Survey Created` |
| **5** | Hệ thống | Gửi email tự động đến `Contact_Using__c.Email` với tiêu đề *"Xin ý kiến đánh giá dịch vụ tại [Tên đại lý]"*. | `Sent` |
| **6** | Khách hàng (Tester đóng vai) | Mở email, bấm vào link khảo sát. | Mở thành công form khảo sát dịch vụ |

* DLR Service Adviser phải được setup thuộc một Dealer hợp lệ, nếu thiếu sẽ báo lỗi chặn check-out.
* Khách hàng nhận email chứa link mã hóa an toàn qua Community `SurveyCRS`.

---

### TC_03: Cơ chế Chống gửi trùng (Anti-spam / Idempotency)

| Bước | Ai | Làm gì | Status sau bước |
|:---:|---|---|---|
| **1** | Tester | Xác định 1 SO Item (hoặc Check-in) đã từng gửi survey thành công ở TC_01 / TC_02 (đã có `SurveyInvitation`). | Đã có sẵn 1 bản ghi `SurveyInvitation` |
| **2** | Tester | Cố ý kích hoạt lại Batch quét bằng cách bấm nút **"Run Survey by Received Date (Daily DO Scan)"** trên trang [*Run Test*](https://ivcl--partial.sandbox.lightning.force.com/lightning/n/Run_Test) (hoặc qua Developer Console). | Batch thực thi xử lý |
| **3** | Hệ thống | Kiểm tra database: SO Item này đã tồn tại `SurveyInvitation` với `Type__c = 'Sale'` $\rightarrow$ **Bỏ qua (Bypass)**. | Không tạo thêm bản ghi |
| **4** | Tester | Kiểm tra hòm thư email và kiểm tra số lượng `SurveyInvitation` trong hệ thống. | Số lượng `SurveyInvitation` vẫn là **1**, không có email trùng lặp gửi đi |

* Logic chống gửi trùng được quản lý chặt chẽ trong `TriggerSalesOrderItemHandler.sendSurveyForItems`: lọc loại trừ danh sách đã tồn tại `Type__c = 'Sale'`.

---

### TC_04: Khảo sát Hài lòng (Đánh giá 4 - 5 sao)

| Bước | Ai | Làm gì | Status sau bước |
|:---:|---|---|---|
| **1** | Khách hàng | Mở link khảo sát từ email trên trình duyệt. | Giao diện khảo sát sẵn sàng |
| **2** | Khách hàng | Chọn điểm đánh giá **4 sao** hoặc **5 sao** tại câu hỏi mức độ hài lòng, nhập nhận xét tích cực (nếu có). | Đã chọn điểm |
| **3** | Khách hàng | Bấm nút **Submit / Hoàn tất**. | Màn hình thông báo cảm ơn hoàn tất khảo sát |
| **4** | Hệ thống | Trigger bắt sự kiện, ghi nhận bản ghi `Survey_Result__c` với `Score__c = 4` hoặc `5`. | `Survey_Result__c` tạo mới |
| **5** | Hệ thống | Kiểm tra điểm số $> 3$ sao $\rightarrow$ Không tạo Case, không tạo Visit. | Kết thúc quy trình khảo sát |
| **6** | Staff CRS | Vào tab **Survey Results** xem báo cáo ghi nhận điểm đánh giá tích cực của khách. | Đã ghi nhận báo cáo |

* Điểm số $\ge 4$ sao được xếp vào nhóm khách hàng hài lòng, hệ thống lưu trữ dữ liệu phục vụ báo cáo CSAT (CRS_R001 / CRS_R002).

---

### TC_05: Khảo sát Trung bình (= 3 sao) – Tự động tạo Case CSKH

| Bước | Ai | Làm gì | Status sau bước |
|:---:|---|---|---|
| **1** | Khách hàng | Mở link khảo sát từ email. | Form khảo sát hiển thị |
| **2** | Khách hàng | Chọn điểm đánh giá đúng **3 sao** (Mức độ chưa thực sự hài lòng). | Đã chọn 3 sao |
| **3** | Khách hàng | Bấm nút **Submit / Hoàn tất**. | Hoàn tất làm khảo sát |
| **4** | Hệ thống | Ghi nhận bản ghi `Survey_Result__c` với `Score__c = 3`. | Đã lưu điểm |
| **5** | Hệ thống | Phát hiện `Score == 3` $\rightarrow$ Tự động tạo 1 bản ghi **Case**: Record Type: `Case`, Status: `New`, Origin: `Survey`, map đúng Khách hàng, Xe, Đại lý. | Bản ghi **Case** được tạo mới |
| **6** | Staff CRS | Vào danh sách **Cases** (List View: *Survey Cases*), mở Case vừa sinh ra để tiếp nhận xử lý. | `In Progress` (Đang giải quyết) |
| **7** | Staff CRS | Gọi điện cho khách hàng hỏi thăm, làm rõ lý do chỉ đánh giá 3 sao, phối hợp Đại lý xử lý và đóng Case. | `Closed` |

* Khi phát sinh Case, Staff CRS là người trực tiếp tiếp nhận và xử lý khiếu nại/chăm sóc khách hàng.

---

### TC_06: Khảo sát Kém (< 3 sao: 1 hoặc 2 sao) – Tự động tạo Kế hoạch thăm khách hàng (Visit)

| Bước | Ai | Làm gì | Status sau bước |
|:---:|---|---|---|
| **1** | Khách hàng | Mở link khảo sát từ email. | Form khảo sát hiển thị |
| **2** | Khách hàng | Chọn điểm đánh giá **1 sao** hoặc **2 sao** (Bất mãn / Không hài lòng dịch vụ/xe). | Đã chọn 1 - 2 sao |
| **3** | Khách hàng | Bấm nút **Submit / Hoàn tất**. | Hoàn tất gửi phản hồi |
| **4** | Hệ thống | Ghi nhận bản ghi `Survey_Result__c` với `Score__c = 1` hoặc `2`. | Đã lưu điểm |
| **5** | Hệ thống | Phát hiện `Score < 3` $\rightarrow$ Tự động tạo 1 bản ghi **Visit__c** với Tên: `Visit Survey`, map Customer, Dealer, VIN. | Bản ghi **Visit** được tạo mới |
| **6** | Staff CRS | Vào danh sách **Visits** (báo cáo Customer Visiting - CRS_R003/R004), kiểm tra bản ghi `Visit Survey`. | `Planned` |
| **7** | Staff CRS | Lên kế hoạch và cùng Đại lý trực tiếp đến thăm/hỗ trợ khách hàng tại chỗ, sau đó cập nhật kết quả thăm vào Visit. | `Completed` |

* Điểm $< 3$ sao được phân loại là cảnh báo nghiêm trọng, chuyển thẳng sang luồng thăm khách hàng trực tiếp (`Visit__c`) thay vì chỉ xử lý Case từ xa.

---

## 4. Bảng Câu Lệnh SOQL Dành Cho Tester Kiểm Tra Dữ Liệu (Query Verification)

Tester mở **Developer Console** $\rightarrow$ tab **Query Editor**, dán và chạy các câu lệnh dưới đây để kiểm tra dữ liệu ngầm mà không cần tìm kiếm thủ công:

### 1. Kiểm tra Lịch sử gửi Survey (`SurveyInvitation`)
```sql
SELECT Id, Name, InvitationLink, ResponseStatus, Sales_Order_Item__c, 
       Vehicle_Check_in__c, Vehicle__c, Type__c, CreatedDate 
FROM SurveyInvitation 
ORDER BY CreatedDate DESC 
LIMIT 5
```

### 2. Kiểm tra Kết quả khảo sát đã lưu (`Survey_Result__c`)
```sql
SELECT Id, Name, Question_Name__c, Score__c, Type__c, 
       Sales_Order_Item__c, Vehicle_Check_in__c, Dealer__c, Customer__c, Vehicle__c, CreatedDate 
FROM Survey_Result__c 
ORDER BY CreatedDate DESC 
LIMIT 5
```

### 3. Kiểm tra Case tự động tạo do đánh giá 3 sao
```sql
SELECT Id, CaseNumber, Origin, Status, RecordTypeId, Dealer__c, 
       Customer_Name__c, VIN__c, Vehicle_Check_in__c, Survey__c, CreatedDate 
FROM Case 
WHERE Origin = 'Survey' 
ORDER BY CreatedDate DESC 
LIMIT 5
```

### 4. Kiểm tra Visit tự động tạo do đánh giá < 3 sao
```sql
SELECT Id, Name, Vendor__c, Customer_Name__c, Dealer_Name__c, 
       VIN__c, Vehicle_Check_in__c, Survey__c, CreatedDate 
FROM Visit__c 
WHERE Name = 'Visit Survey' 
ORDER BY CreatedDate DESC 
LIMIT 5
```

---

## 5. Những Điểm Cần Lưu Ý & Khắc Phục Lỗi Khi Test (Troubleshooting)

| STT | Triệu chứng lỗi | Nguyên nhân thực tế từ Code/Hệ thống | Cách xử lý / Ghi chú cho Dev & QA |
|:---:|---|---|---|
| **1** | Mở link báo lỗi **404** | Sai domain Community hoặc Community `SurveyCRS` chưa được Active/Publish. | Vào *Digital Experiences* $\rightarrow$ kiểm tra site `SurveyCRS` đã Publish chưa. |
| **2** | Không nhận được Email | 1. Contact thiếu Email.<br>2. Deliverability của Sandbox đang là *No Access*.<br>3. Thiếu Email Template `Email_Survey`. | 1. Điền email vào Contact xe.<br>2. Bật Deliverability lên *All email*.<br>3. Kiểm tra DeveloperName của template phải chính xác là `Email_Survey`. |
| **3** | Submit xong nhưng không có bản ghi `Survey_Result__c` và không tạo `Case` / `Visit__c` | **Nguyên nhân cốt lõi trong code:**<br>Tại [SurveyQuestionResponseHandler.cls](file:///c:/Users/DELL/Videos/IVC_Project/force-app/main/default/classes/SurveyQuestionResponseHandler.cls#L8-L10), hai dòng xử lý sau đang bị comment:<br>`//createCaseOrVisit(listNew, newMap);`<br>`//updateScore(listNew);`<br>Ngoài ra cần kiểm tra quyền Create của Guest User trên Community. | Báo Dev mở comment và kiểm tra quyền Guest User để luồng kích hoạt bình thường. |
| **4** | Đối với Bán xe (Sales), Case/Visit thiếu link SO Item | Trong logic `createCaseOrVisit`, code hiện chỉ map `Vehicle_Check_in__c`. | Cần thống nhất với BA/Dev bổ sung trường hoặc ghi nhận nguồn từ Sales Order Item. |
