# Finalize_Invoice — Flow Design Plan

Flow mới cần tạo trong Power Automate cho app **invoice-batch-app**, gọi từ
`Src/BatchUploadScreen.pa.yaml` (nút "✅ Done", bên trong `ForAll(colInvoiceParseResults, Finalize_Invoice.Run(...))`).

Vai trò: nhận 1 hóa đơn đã được người dùng review/sửa trong `colInvoiceParseResults`,
copy file đính kèm sang đúng SharePoint Library theo Cost Center, ghi metadata lên
chính item đó, rồi xóa attachment gốc khỏi `BEUI_InvoiceBatch_Requests`.

Được gọi **tuần tự, 1 lần cho mỗi hóa đơn** trong batch (nếu batch có 10 file thì
chạy 10 lần) — mọi thiết kế bên dưới phải an toàn khi chạy lặp lại nhiều lần trên
cùng 1 batch record.

## A. Việc cần làm trước trong SharePoint (thủ công, trước khi build flow)

Thêm **cùng một bộ cột metadata** vào cả 4 library đích: `AU_Invoices`, `MY_Invoices`,
`SG_Invoices`, `VN_Invoices` (hiện chưa có cột tùy chỉnh nào).

| Tên cột | Kiểu | Nguồn dữ liệu |
|---|---|---|
| `InvoiceNumber` | Single line text | InvoiceNumberEdit |
| `InvoiceDate` | Single line text | InvoiceDateEdit |
| `VendorName` | Single line text | SupplierNameEdit |
| `BilledTo` | Single line text | BilledToEdit |
| `ABN` | Single line text | ABNEdit |
| `Attention` | Single line text | AttentionEdit |
| `TotalAmount` | Currency/Number | TotalAmountEdit |
| `GSTAmount` | Currency/Number | TaxAmountEdit |
| `Currency` | Single line text | CurrencyEdit |
| `CostCenter` | Choice (giống cột `CostCenter` của `BEUI_InvoiceBatch_Requests`) | CostCenterEdit / gDefaultCostCenter |
| `BatchID` | Number | gSelectedBatch.ID |
| `ConfidenceScore` | Number | ConfidenceScore |
| `ParsedAt` | Date and Time | `utcNow()` (tính trong flow) |

`BatchID` dùng plain Number (không phải Lookup column thật) để đơn giản hóa —
giống cách `RequestIDText` được dùng như bản sao text của lookup trong
`Procurement_InvoiceData` bên `procurement-procedure`.

## B. Trigger input của Finalize_Invoice

Thứ tự **phải khớp đúng** với lời gọi `Finalize_Invoice.Run(...)` trong
`BatchUploadScreen.pa.yaml`. Khi thêm "Ask in PowerApps" trong Power Automate,
thêm đúng thứ tự này và **chỉ định type tường minh** (Text/Number) ngay từ đầu —
rút kinh nghiệm từ lỗi từng gặp: tham số `ABN` bị Studio tự suy luận sai thành
kiểu `Blank` vì không được set type rõ ràng lúc tạo, gây lỗi
`"expected type Blank, found Text"`.

| # | Tên nội bộ | Type | Ý nghĩa |
|---|---|---|---|
| 1 | text | Text | InvoiceURL (attachment URL) |
| 2 | number | Number | BatchID (gSelectedBatch.ID) |
| 3 | text_1 | Text | newFilename (đã gồm extension, trùng tên attachment gốc) |
| 4 | text_2 | Text | InvoiceNumber |
| 5 | text_3 | Text | InvoiceDate |
| 6 | text_4 | Text | SupplierName |
| 7 | text_5 | Text | BilledTo |
| 8 | number_1 | Number | TotalAmount |
| 9 | number_2 | Number | TaxAmount |
| 10 | text_6 | Text | Currency |
| 11 | text_7 | Text | CostCenter |
| 12 | text_8 | Text | Attention |
| 13 | text_9 | Text | ABN |
| 14 | number_3 | Number | ConfidenceScore |

## C. Các bước (actions) trong flow

1. **Compose_File_Path** — giống pattern trong `Parse_Invoice` / `Submit_Invoice`
   (workflow cũ của `procurement-procedure`): chuyển `text` (URL) thành
   server-relative path.

   ```
   @replace(if(contains(triggerBody()?['text'], 'AllItems.aspx'), replace(uriComponentToString(split(split(triggerBody()?['text'], '&id=')[1], '&')[0]), '#', '%23'), replace(triggerBody()?['text'], 'https://maxbiocare.sharepoint.com', '')), '#', '%23')
   ```

2. **Switch_Cost_Center** — dùng action **Switch** (không dùng nested-If như flow
   `Submit_Invoice` cũ — flow cũ có bug âm thầm rơi hóa đơn "Unknown Region" vào
   `MY_Invoices` mặc định thay vì báo lỗi):
   - `On`: `@triggerBody()?['text_7']`
   - Case `Australia` → Compose `Target_Library` = `"AU_Invoices"`
   - Case `Malaysia` → `"MY_Invoices"`
   - Case `Singapore` → `"SG_Invoices"`
   - Case `Vietnam` → `"VN_Invoices"`
   - **Default** → action **Terminate** (Failed, message
     `"Unknown Cost Center: " & triggerBody()?['text_7']`) — chặn đứng thay vì
     âm thầm lưu sai chỗ, vì đây là tài liệu tài chính.

3. **Get_Attachment_Content** — OpenApiConnection `HttpRequest` GET, đọc file gốc
   từ `outputs('Compose_File_Path')` (giống hệt action cùng tên trong
   `Submit_Invoice`).

4. **Create_Invoice_File** — SharePoint `CreateFile`:
   - `folderPath`: `@outputs('Target_Library')` (dynamic theo Cost Center — đã
     được chứng minh hoạt động trong `Submit_Invoice` cũ với `InvoiceRegion`)
   - `name`: `@{triggerBody()?['text_1']}` (đã có extension sẵn — **không** nối
     thêm extension lần nữa)
   - `body`: `@body('Get_Attachment_Content')`
   - `overwrite`: `true`

5. **Get_New_File_Item_Id** — HttpRequest GET tới:
   ```
   _api/web/GetFileByServerRelativePath(decodedurl='/sites/Powerapps/@{outputs('Target_Library')}/@{triggerBody()?['text_1']}')/ListItemAllFields?$select=Id
   ```

6. **Update_Invoice_Metadata** — HttpRequest POST (MERGE) tới:
   ```
   _api/web/lists/getbytitle('@{outputs('Target_Library')}')/items(@{body('Get_New_File_Item_Id')?['Id']})
   ```
   Header: `X-HTTP-Method: MERGE`, `IF-MATCH: *`,
   `Content-Type: application/json;odata=nometadata`.
   Body JSON set toàn bộ 13 cột ở mục A, map trực tiếp từ `triggerBody()`, cộng
   `ParsedAt: @{utcNow()}`.

7. **Delete_Source_Attachment** — SharePoint `DeleteAttachment`:
   - `table` = `BEUI_InvoiceBatch_Requests`
   - `id` = `triggerBody()?['number']`
   - `fileIdentifier` = `triggerBody()?['text_1']`

   Chỉ xóa **đúng 1 attachment vừa xử lý**, không xóa toàn bộ attachments của
   record — vì flow này chạy tuần tự nhiều lần trong cùng 1 `ForAll` ở
   `BatchUploadScreen.pa.yaml`; xóa toàn bộ ngay từ lần chạy đầu sẽ làm mất file
   của các hóa đơn chưa kịp xử lý trong cùng batch. Sau khi `ForAll` chạy hết,
   kết quả cuối cùng vẫn là toàn bộ attachments đã được xóa — nhưng an toàn theo
   từng bước.

8. **Response** (tùy chọn) — trả về:
   ```
   newInvoiceLink = concat('https://maxbiocare.sharepoint.com/sites/Powerapps/', outputs('Target_Library'), '/', triggerBody()?['text_1'])
   ```
   Dự phòng nếu sau này app cần hiển thị link — hiện tại
   `BatchUploadScreen.pa.yaml` không capture giá trị trả về nên bước này không
   bắt buộc.

## D. Không cần sửa gì ở `BatchUploadScreen.pa.yaml`

Lời gọi `Finalize_Invoice.Run(...)` ở dòng ~560-575 đã đúng thứ tự 14 tham số —
flow build xong trong Power Automate là gọi được ngay, không cần đổi code
Power Apps.

## Tham chiếu

- Flow cũ tham khảo: `Workflows/Parse_Invoice-*.json`, `Workflows/Submit_Invoice-*.json`
  (từ `procurement-procedure`, dùng cùng SharePoint site và cùng 3 library
  AU/SG/MY_Invoices — `VN_Invoices` là library mới thêm riêng cho app này).
- Lời gọi flow trong app: `Src/BatchUploadScreen.pa.yaml`.
