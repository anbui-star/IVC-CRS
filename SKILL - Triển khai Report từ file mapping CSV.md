---
name: report-from-csv-mapping
description: Triển khai Salesforce Report từ file CSV yêu cầu của BA (hàng "O1 mapping field") trên org Partial, kèm trang HTML mapping field để giải thích với BA. Dùng khi user đưa file "(<mã report>) <Tên báo cáo>.csv" và yêu cầu hiện thực report.
---

# Skill: Triển khai Report từ file mapping CSV (O1)

Rút ra từ CRS_R001 + CRS_R002 (Khảo sát khách hàng). Mẫu đầu ra:
- Report: `TeamCRS/CRS_R001_R002` trên Partial.
- Trang HTML: `(CRS_R001 + CRS_R002) Mapping field báo cáo Khảo sát khách hàng (Customer Survey).html`. Dùng file này làm mẫu CSS và cấu trúc.

## Nguyên tắc bắt buộc (không bỏ qua)

1. **Org:** chỉ làm trên **PARTIAL**, không đụng PROD. Mọi lệnh `sf` phải ghi rõ `--target-org PARTIAL`, vì org mặc định của repo là PROD. Chạy `sf` bằng **PowerShell**, không dùng Bash.
2. **Quy trình:** retrieve các file liên quan từ Partial → `git commit -m "Retrieve code"` → push. Chỉ commit file của task này. Sau đó liệt kê các bước và **chờ user duyệt** rồi mới sửa hoặc deploy.
3. **Trước mỗi lần deploy:** kiểm tra lại LastModifiedDate của flow, permset và report trên org, xác nhận không ai sửa từ lúc retrieve. Bắt buộc dry-run trước.
4. **Không chạy lệnh xoá** (metadata hay data). Đưa user các bước xoá tay trong Setup.
5. **Thêm field vào Page Layout / Record Page:** hướng dẫn làm tay, không deploy layout.
6. **Playwright:** chỉ dùng khi bắt buộc, ví dụ dữ liệu chỉ tạo được qua giao diện guest như trả lời khảo sát. **Phải nói lý do và hỏi trước.**
7. **Sửa dữ liệu test hoặc backfill:** chỉ điền ô trống, không ghi đè. Làm bước chỉ đọc để đối chiếu trước, đưa bảng cho user duyệt rồi mới ghi.
8. **File HTML:** user có thể đã tự chỉnh. Chỉ sửa đúng phần được yêu cầu, **không revert**, không viết lại toàn bộ file.

## Bước 1 – Đọc CSV

- Đọc bằng python `csv` vì có ô xuống dòng. Lưu ý encoding `utf-8-sig`. Console Windows in tiếng Việt bị lỗi cp1252, nên ghi kết quả ra file trong scratchpad rồi `cat`.
- Cấu trúc thường gặp:
  - Hàng `Tên report`
  - Các hàng `Filter`
  - Hàng tiêu đề cột
  - Hàng mô tả
  - Hàng **`O1 mapping field`**
  - `Data sample`
- Tiêu đề kiểu **"A + B"** (ví dụ CRS_R001 + CRS_R002) là **MỘT** report, không tách thành hai.
- Cột không có mapping trong O1 thì không tự đoán. Hỏi user (ví dụ cột "Số điện thoại 2" cuối cùng bị bỏ).
- O1 thường viết tắt hoặc sai tên field ("Case.completedate", "Visit. Plandate"). Tra field thật trên org và ghi rõ trong Ghi chú.

## Bước 2 – Chọn object gốc (quan trọng nhất)

Bài học: O1 trộn nhiều nguồn, ví dụ phần lớn cột `SurveyQuestionResponse.*` nhưng cột Đại lý lại ghi `Survey Result.*`. Lần đầu mình chọn sai gốc, phải làm lại. **Khi O1 trộn object, hỏi user báo cáo đi từ object nào trước khi build.**

Cách kiểm tra quan hệ trên org (lookup và related list):

```powershell
# rel_scan.ps1 – sửa danh sách $objs / $interest theo report
$objs = 'Survey_Result__c','SurveyInvitation','Visit__c','Case'
$interest = $objs
foreach ($o in $objs) {
  $d = (sf sobject describe --sobject $o --target-org PARTIAL --json | ConvertFrom-Json).result
  "==== $o"
  "  LOOKUP: " + (($d.fields | ? { $_.type -eq 'reference' } | % { "$($_.name)->$($_.referenceTo -join '/')" }) -join ', ')
  "  CHILD:  " + (($d.childRelationships | ? { $_.childSObject -in $interest -and $_.relationshipName } | % { "$($_.childSObject).$($_.field) [$($_.relationshipName)]" }) -join ', ')
}
```

Quy tắc chọn:
- **Report type là một đường thẳng:** cha → con → cháu (tối đa 4 cấp), kèm field lấy lên qua lookup (tối đa 5 cấp). **Không lấy được 2 nhánh con song song** của cùng một cha.
- **Joined report** không ra được "1 dòng / bản ghi" như CSV, và không có row-level formula.
- **Liên kết đa hình** (ví dụ `SurveySubject`) không dùng được trong report type.
- **Object không customizable** (`SurveyQuestionResponse`, `SurveyResponse`) không thêm field được. Kiểm tra bằng `EntityDefinition.IsCustomizable`.
- **Ưu tiên object có đúng 1 bản ghi cho mỗi dòng nghiệp vụ** và đã có sẵn bản sao các giá trị cần (flow/code đã chép sang). Ví dụ `Survey_Result__c`.
- **Nếu thiếu đường nối** (ví dụ Visit/Case không biết thuộc khảo sát nào): thêm **lookup trên object gốc** và sửa flow gán khi tạo. Không đặt lookup ngược ở object con, vì report type không đi xuống 2 nhánh được.

## Bước 3 – Khảo sát org Partial

- Field của object:
  - `sf sobject describe --sobject X --target-org PARTIAL --json`.
  - Query `FieldDefinition` **không hỗ trợ OR** ("Disjunctions not supported"), nên lọc bằng describe.
- Picklist: lấy giá trị thực tế (ví dụ Case Reason chỉ có giá trị mẫu của Salesforce, phải báo BA).
- Dữ liệu: đếm số bản ghi, xem vài dòng mẫu, kiểm tra cột nào đang trống.
- **Long Text Area** (ví dụ `ResponseValue`) không lọc/so sánh số trong report được. Dùng field số tương ứng.
- Nguồn dữ liệu (để ghi Ghi chú cho BA):
  - Flow: `sf data query --use-tooling-api --query "SELECT VersionNumber, Status, CreatedDate FROM Flow WHERE Definition.DeveloperName='X'"`. Đọc `Metadata` từng version để biết version nào bắt đầu ghi field nào.
  - Object/field: `CustomObject` / `CustomField` (Tooling) → `CreatedDate`.
  - Code tạo bản ghi: grep class local, rồi đối chiếu `ApexClass.Body` trên Partial vì local có thể cũ.
- Đổi giờ UTC sang giờ VN (+7) khi ghi vào tài liệu.

## Bước 4 – Retrieve → commit → push

```powershell
sf project retrieve start --target-org PARTIAL --metadata "CustomObject:<Obj>" "Flow:<Flow>" "PermissionSet:<Permset>" "Report:<Folder>/<Report>" "ReportType:<RT>" --ignore-conflicts
git add <chỉ file của task>; git commit -m "Retrieve code"; git push
```

`git status` thường có file của người khác (app, flexipage…), **không commit chúng**.

## Bước 5 – Trình bày kế hoạch và chờ duyệt

Nêu ngắn gọn:
- Kết quả khảo sát (thiếu liên kết gì, field nào không tồn tại).
- Các bước đánh số: field mới, sửa flow, report type, report, FLS, dry-run/deploy, test, tài liệu.
- Điểm cần chốt.

Chỉ hỏi những gì đổi được kết quả. Không hỏi lại điều user đã chốt.

## Bước 6 – Viết metadata

**Field mới:**
- Lookup có `relationshipName` không trùng child relationship hiện có của object đích.
- Formula Text dùng `ISPICKVAL`, `Case__r.IsClosed`…, trả `""` khi không có giá trị.
- Ghi `description` bắt đầu bằng mã report.

**Sửa flow:** dùng python patch file XML. Không dùng Flow Builder.
- Thêm `inputAssignments` vào bước tạo record với giá trị `<elementReference>Create_X</elementReference>`. Nếu bước tạo không chạy thì giá trị là null, không lỗi.
- Đặt phần tử XML đúng thứ tự loại: `recordCreates` → `recordLookups` → `recordUpdates` → `start`.
- So diff với bản gốc để chắc chỉ thêm đúng phần cần.

**Permission set:** thêm `fieldPermissions` (field formula/lookup chỉ đọc; field đại lý nhập thì cho sửa). `description` tối đa 255 ký tự.

**Report type:** sinh bằng script python từ một danh sách `(field path, nhãn tiếng Việt, checkedByDefault)`:
- Lookup chuẩn dùng **tên quan hệ**: `Response.CompletionDateTime`, `Invitation.Type__c`.
- Lookup custom dùng **tên field**: `Dealer__c.Search_Name__c`, `Visit__c.Planned_Date__c`.
- `displayNameOverride` = tiêu đề cột trong CSV. Nhờ vậy report không cần đặt tên cột.
- Chia `sections` theo khối nghiệp vụ.

**Report (XML):**
- Cột viết dạng `<Base>$<path>`, ví dụ `Survey_Result__c$Dealer__c.Search_Name__c`.
- Row-level formula đặt trong `customDetailFormulas` (`developerName` CDF1, `dataType` Text). Công thức tham chiếu field dạng `Survey_Result__c.Score__c`, cột trong report là `CDF1`.
- Bộ lọc cho người dùng nhập khi chạy:
  - Text/picklist: `isUnlocked=true`, `operator=contains`, `value` rỗng. `equals` với giá trị rỗng sẽ thành "is empty" và làm mất hết dòng.
  - Số: `lessOrEqual` giá trị lớn nhất (ví dụ 5).
- Bộ lọc thời gian dùng `timeFrameFilter`, ví dụ `INTERVAL_THISMONTH`, `INTERVAL_CURFY`.
- `format` Tabular, `scope` organization, sort giảm dần theo ngày.
- **Tên report ≤ 40 ký tự**, description ≤ 255 ký tự.
- Đổi `reportType` của report đã có bằng metadata vẫn được (link report giữ nguyên).
- Sau khi retrieve, giá trị picklist trong filter có thể bị lưu thành mã (ví dụ "Rating" thành "4"). Đó là bình thường.

## Bước 7 – Deploy (metadata-dir, API 67.0)

```powershell
$S = "<scratchpad>\mdapi"
sf project convert source --output-dir $S --metadata "CustomField:Obj.F1__c" "Flow:X" "PermissionSet:P" "ReportType:RT" "Report:Folder/R"
(Get-Content "$S\package.xml" -Raw) -replace '<version>[\d.]+</version>','<version>67.0</version>' | Set-Content -NoNewline -Encoding ascii "$S\package.xml"
sf project deploy start --metadata-dir $S --target-org PARTIAL --dry-run --test-level NoTestRun --wait 20
sf project deploy start --metadata-dir $S --target-org PARTIAL --test-level NoTestRun --wait 20
```

- Chỉ đưa vào **đúng field của task**. Local có thể có field chưa tồn tại trên Partial (ví dụ `Survey_Result__c.Email_Survey__c`).
- Báo deploy ID cho user.

## Bước 8 – Kiểm tra report qua API

```powershell
$out = sf api request rest "/services/data/v67.0/analytics/reports/<ReportId>?includeDetails=true" --target-org PARTIAL 2>$null
$r = ($out -join "`n") | ConvertFrom-Json
$labels = $r.reportMetadata.detailColumns | % { $r.reportExtendedMetadata.detailColumnInfo.$_.label }
"Columns: " + ($labels -join ' | ')
$r.factMap.'T!T'.rows | % { ($_.dataCells | % { $_.label }) -join ' | ' }
```

Đối chiếu số cột với CSV và số dòng với dữ liệu. Kiểm tra cột formula ra đúng quy tắc.

## Bước 9 – Dữ liệu test cho "1 dòng đầy đủ"

- Ưu tiên cập nhật bản ghi test mình đã tạo, chỉ điền ô trống.
- Nếu đổi bản ghi dùng chung (ví dụ Vehicle) thì báo user.
- Giải thích các cột theo nhánh nghiệp vụ. Ví dụ 1–2 sao chỉ có Visit, 3 sao chỉ có Case, nên không dòng nào đủ cả hai khối. Không ép dữ liệu sai quy tắc.
- Gửi link bản ghi gốc (`/lightning/r/<Object>/<Id>/view`) và link report.

## Bước 10 – Backfill dữ liệu cũ (khi vừa thêm lookup và sửa flow)

1. **Chỉ đọc:**
   - Liệt kê bản ghi gốc còn thiếu lookup và các ứng viên.
   - Ghép theo khoá chắc chắn. Ví dụ bản ghi do **cùng một lần chạy flow** tạo thì cách nhau ≤ 1 giây, và có cùng khách hàng, cùng xe.
2. Đưa bảng ghép (bản ghi, giờ VN, bản ghi đích) và danh sách không ghép được kèm lý do. **Chờ duyệt.**
3. Script ghi:
   - **Dừng nếu ô đích đã có giá trị.**
   - Cập nhật từng bản ghi bằng `sf data update record --json`.
   - Query lại để xác nhận.
4. Chạy lại report. Ghi vào tài liệu: đã backfill ngày nào, cách ghép, QA/PROD phải chạy lại.

## Bước 11 – Trang HTML mapping (cho BA)

Tên file: `(<mã report> + <mã report>) Mapping field báo cáo <Tên VN> (<English>).html`. Copy nguyên `<head>`/CSS từ file mẫu. Thêm vào menu `docs/CRS/index.html` (mảng `DOCS`, `file` được URL-encode).

**Cấu trúc:**
1. **Bảng meta:** Nguồn yêu cầu (tên CSV + "theo hàng O1"), Báo cáo (link), Report type (đường đi), Folder (link).
2. **"1. Quan hệ giữa các object":**
   - Sơ đồ `<pre><code>` cha/con/lookup, đánh dấu lookup mới kèm ngày.
   - Các gạch đầu dòng: object nào không nối được với object nào; vì sao không đi từ 2 object; kết luận đi từ object nào.
3. **"2. Mapping cột":**
   - Danh sách "Luồng dữ liệu" (`<ol>`): bản ghi nguồn được tạo thế nào (job/code nào, nút nào, giao diện nào), flow nào tạo gì, mốc ngày trên Partial.
   - Bảng 3 cột: `STT | Field trên Report | Field lấy từ (Object.Field / cách tính)`. Ô thứ 3 gồm:
     - `O1 mapping (CSV):` nguyên văn CSV
     - `Triển khai:` đường dẫn field thật
     - `Ghi chú:` nhãn trạng thái + `<ul class="note">`
4. **"3. Bộ lọc của báo cáo":** cùng định dạng, `STT | Bộ lọc trên Report | Field lấy từ`.

**Quy tắc viết Ghi chú** (mục đích: user dùng để giải thích với BA):
- Dạng **list**. Gạch đầu tiên là **"Khớp O1."** hoặc **"Khác O1: dùng `X` thay cho `Y`."**
- Với "Khác O1", phải có:
  - **Lý do không làm khớp được**, đã kiểm tra trên org (ví dụ không có lookup, field không tồn tại, Long Text không lọc số được).
  - **Nguồn dữ liệu cụ thể:** flow/class nào ghi field này; người dùng bấm gì trên giao diện nào thì bản ghi được tạo; có từ phiên bản/ngày nào trên Partial.
  - **Bằng chứng** nếu có (ví dụ CreatedDate trùng CompletionDateTime tới từng giây).
- Cột trùng lý do thì ghi "Lý do: như cột N". Khi bỏ hoặc đánh số lại cột, sửa các tham chiếu "cột N".
- Nếu BA vẫn muốn đúng đường dẫn O1, ghi phương án và cái giá (thêm lookup + sửa flow, dữ liệu cũ trống nếu không backfill).
- **Không đưa vào HTML:** việc chưa làm, kết quả test, thành phần đã deploy, các bước xoá tay. Những thứ này báo trong chat (user đã bỏ các mục đó).

**Kỹ thuật sửa HTML:**
- Sửa bằng `Edit` hoặc python script trong scratchpad, **sao lưu file trước**.
- Thay thế hàng loạt phải giới hạn đúng bảng/hàng. Lỗi đã gặp: thay "⏳" toàn file làm hỏng dòng chú thích ký hiệu.
- Không dùng heredoc chứa tiếng Việt trong `bash -c` (lỗi quote). Ghi nội dung bằng Write tool rồi ghép.

## Bước 12 – Báo cáo cuối và việc làm tay

Báo ngắn trong chat:
- Link report, deploy ID, số dòng/cột đã kiểm tra.
- Field mới theo từng object (API name, nhãn, kiểu, dùng cho cột nào).
- Việc user làm tay:
  - Thêm field vào trang/layout.
  - Xoá thành phần không dùng, **theo đúng thứ tự**: field formula trước, rồi lookup nó dùng.
  - **Phiên bản Flow cũ (Obsolete) vẫn chặn xoá field.** Phải xoá đúng version còn tham chiếu (kiểm tra bằng `MetadataComponentDependency`), không xoá version Active.
- Điểm cần BA chốt, và QA/PROD chưa triển khai.
- Code chưa commit, trừ khi user yêu cầu.

## Lỗi/mẹo đã gặp

| Hiện tượng | Cách xử lý |
|---|---|
| `sf` trong Bash báo "'C:\Program' is not recognized" | Chạy bằng PowerShell |
| PowerShell in `NativeCommandError` dù lệnh thành công | Là output stderr bình thường. Dùng `--json` để đọc `status` |
| `FieldDefinition` query lỗi "Disjunctions not supported" | Dùng `sf sobject describe` |
| Tạo `SurveySubject` báo field không ghi được | Không cần cho flow. Flow đọc dữ liệu từ Invitation |
| Filter để trống làm report rỗng | Dùng `contains` thay cho `equals` |
| Field không xoá được | `MetadataComponentDependency` (Tooling): thường do phiên bản Flow cũ hoặc formula tham chiếu |
| Bulk replace trong HTML làm hỏng phần khác | Giới hạn regex theo bảng/hàng, sao lưu trước |
