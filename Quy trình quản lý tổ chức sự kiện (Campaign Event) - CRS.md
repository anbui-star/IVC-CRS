# Luồng quản lý tổ chức sự kiện (Campaign Event) - CRS

> Cập nhật: 2026-10-14. Theo cấu hình Partial (đã retrieve 13/10) và dữ liệu thật trên Partial.
> Đại lý (DLR) tổ chức sự kiện **Eco-drive seminar** / **Maintenance training** cho khách hàng, CRS của IVC duyệt. Object **Campaign**, Record Type **Event**. Campaign `Type = Mobile Service` là quy trình khác, **không** đi luồng này.
> Có **2 lần duyệt**: duyệt kế hoạch (Bước 2 → 5, process `CampaignCRSNew`) và duyệt kết quả (Bước 8 → 9, process `CampaignCRS2`).

---

## 1. Toàn bộ luồng

```
Bước 1  DLR tạo Campaign ........................................... Planned

┌─ PHÊ DUYỆT GIAI ĐOẠN 1: duyệt kế hoạch (Bước 2 → Bước 5, process CampaignCRSNew)
│ Bước 2  DLR bấm "Nộp xin Phê Duyệt" (kế hoạch) ................... DLR Review
│ Bước 3  Dealer Manager duyệt ..................................... IVC Review
│ Bước 4  Staff CRS (Quốc HOẶC Trinh, 1 người là xong) duyệt ....... Manager Review
│ Bước 5  Manager CRS (Khôi) duyệt ................................. Approved
└─

Bước 6  DLR tổ chức sự kiện, đổi Status tay ........................ In Progress
Bước 7  DLR nhập kết quả + ĐÍNH KÈM FILE (tab File)

┌─ PHÊ DUYỆT GIAI ĐOẠN 2: duyệt kết quả (Bước 8 → Bước 9, process CampaignCRS2)
│ Bước 8  DLR bấm "Nộp xin Phê Duyệt" (kết quả) .................... Reporting   <- kiểm tra file
│ Bước 9  IVC (Quốc HOẶC Trinh, 1 người là xong) duyệt ............. Completed   <- kiểm tra file
└─
```

- **Phê duyệt giai đoạn 1: Bước 2 → Bước 5** (Bước 2 là DLR submit, Bước 3 → 5 là 3 cấp duyệt).
- **Phê duyệt giai đoạn 2: Bước 8 → Bước 9** (Bước 8 là DLR submit, Bước 9 là IVC duyệt).
- Bước 1, 6, 7 không thuộc phê duyệt (DLR tự làm).
- Từ chối ở **Bước 3, 4, 5 hoặc 9** → Status **Rejected**.
- Chuỗi Status: `Planned` → `DLR Review` → `IVC Review` → `Manager Review` → `Approved` → `In Progress` → `Reporting` → `Completed`.
  (`Manager Review` hiển thị trên giao diện là **IVC Manager Review**.)

---

## 2. Từng bước: ai làm gì

| Bước | Ai | Làm gì | Status sau bước |
|---|---|---|---|
| **1** | DLR CS staff | Tạo Campaign Event. Mã `EVT-xxxxx` và Đại lý tự điền | Planned |
| **2** | DLR CS staff (**chủ record**, chỉ chủ mới submit được) | Bấm "Nộp xin Phê Duyệt" | DLR Review |
| **3** | `DLR Sales manager` (user cố định `sales.manager@dlr.com`) | Approve / Reject | IVC Review |
| **4** | `HỒ THIÊN QUỐC` hoặc `PHẠM MAI TRINH` (ngang hàng, **1 người duyệt là xong**, người kia tự `No Response`) | Approve / Reject | Manager Review |
| **5** | `NGUYỄN VĂN KHÔI` (Manager CRS) | Approve / Reject | **Approved** |
| **6** | DLR | Tổ chức sự kiện, đổi Status sang In Progress (thủ công trên Path, không có tự động hóa) | In Progress |
| **7** | DLR CS staff | Nhập số người, chi phí. Đính kèm file ở tab **File** (Công văn yêu cầu đại lý xuất service SO, Service SO, Invoice) | In Progress |
| **8** | DLR CS staff | Bấm "Nộp xin Phê Duyệt". **Thiếu file thì báo lỗi** | **Reporting** |
| **9** | `HỒ THIÊN QUỐC` hoặc `PHẠM MAI TRINH` (1 người là xong) | Vào **Approval History** → Approve. **Thiếu file thì báo lỗi** | **Completed** |

- Khi sang `Approved` (Bước 5) hoặc `Rejected`, chủ record nhận thông báo chuông.
- Khi sang `Completed` (Bước 9), chủ record nhận email chi phí hỗ trợ (Event Organization Cost, POSM Cost, Supported Cost).
- Lúc đang chờ duyệt, chỉ admin hoặc approver hiện tại sửa được record. Không có Recall.

---

## 3. Kiểm tra file đính kèm (Issue 5770)

Thiếu file = Campaign chưa có file nào ở tab File (chỉ cần ≥ 1 file, không phân biệt loại).

| Chặn ở | Chuyển Status | Thông báo |
|---|---|---|
| **Bước 8** (DLR submit kết quả) | → `Reporting` | *Please attach the official letter (Công văn) requesting the dealer to issue the service SO before submitting to IVC.* |
| **Bước 9** (IVC duyệt kết quả) | `Reporting` → `Completed` | *Cannot approve: the dealer has not attached the official letter (Công văn) requesting the service SO.* |

**Bước 2 → 5 (duyệt kế hoạch) hiện KHÔNG có kiểm tra file.** Người test ticket 5770 cả 3 lần (8/10, 14/10) đều đi Bước 1 → 5, không đính kèm file, Trinh/Quốc duyệt bình thường ở Bước 4 và không thấy lỗi. Ví dụ Campaign `701WC00000vAZmhYAG`.
Gặp user báo "không thấy lỗi" thì **hỏi đang test ở Bước mấy** và đối chiếu `ProcessInstance.ProcessDefinition.DeveloperName`: `CampaignCRSNew` = Bước 2 → 5, `CampaignCRS2` = Bước 8 → 9.

Code: `TriggerCampaignHandler.validateAttachmentRequired` (gọi từ `onBeforeUpdate`).
