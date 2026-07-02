# Finalize_Invoice — Flow Design Plan

Flow mới cần tạo trong Power Automate cho app **invoice-batch-app**, gọi từ
`Src/BatchUploadScreen.pa.yaml` (nút "✅ Done", bên trong `ForAll(colInvoiceParseResults, Finalize_Invoice.Run(...))`).

Vai trò: nhận 1 hóa đơn đã được người dùng review/sửa trong `colInvoiceParseResults`,
copy file đính kèm sang đúng SharePoint Library theo Cost Center, **tạo 1 record chứa
dữ liệu extract trong list `Procurement_InvoiceData`** (list dùng chung với
`procurement-procedure`, KHÔNG ghi metadata lên library nữa), rồi xóa attachment gốc
khỏi `BEUI_InvoiceBatch_Requests`.

Được gọi **tuần tự, 1 lần cho mỗi hóa đơn** trong batch (nếu batch có 10 file thì
chạy 10 lần) — mọi thiết kế bên dưới phải an toàn khi chạy lặp lại nhiều lần trên
cùng 1 batch record.

## A. Việc cần làm trước trong SharePoint (thủ công, trước khi build flow)

**KHÔNG thêm cột nào vào 4 library đích nữa.** `AU_Invoices`, `MY_Invoices`,
`SG_Invoices`, `VN_Invoices` chỉ dùng để chứa file — không cần cột tùy chỉnh.

Thay vào đó, dữ liệu extract được ghi thành **1 record trong list dùng chung
`Procurement_InvoiceData`** (GUID `3a7218e7-e10f-406a-9a8a-d4cbbb2ea82c`, cùng site
`/sites/Powerapps`) — chính list mà flow `Submit_Invoice` bên `procurement-procedure`
đang dùng.

List này **đã có sẵn** các cột: `Title`, `RequestID` (Lookup→Procurement_Requests),
`RequestIDText` (Text), `InvoiceNumber`, `InvoiceDate`, `VendorName`, `BilledTo`,
`Attention`, `Description`, `TotalAmount`, `GSTAmount`, `Currency`, `ParsedAt`,
`InvoiceLink`, `ABN`.

**Cần thêm 4 cột mới** vào `Procurement_InvoiceData` (cột optional, không phá vỡ
`procurement-procedure` vì flow cũ chỉ bỏ trống các cột này):

| Tên cột (thêm mới) | Kiểu | Nguồn dữ liệu |
|---|---|---|
| `CostCenter` | Choice (cùng các giá trị Australia/Malaysia/Singapore/Vietnam như `BEUI_InvoiceBatch_Requests`) | text_7 (CostCenterEdit / ddDefaultCostCenter) |
| `BatchID` | Number | number (gSelectedBatch.ID) |
| `ConfidenceScore` | Number | number_3 (ConfidenceScore) |
| `Origin` | Single line text | Hằng số — tên app ghi record. Flow này set cố định `"invoice-batch-app"`. Dùng để phân biệt record do app nào tạo (procurement flow cũ để trống hoặc set tên riêng). |

**Lưu ý về `RequestID` / `BatchID`:** cột `RequestID` là Lookup trỏ tới
`Procurement_Requests` — app này KHÔNG có record trong đó (BatchID thuộc list khác
`BEUI_InvoiceBatch_Requests`). Vì vậy flow **không set `item/RequestID/Id`** (set sai
sẽ tạo lookup hỏng / lỗi runtime). Liên kết ngược về batch dùng:
- cột `BatchID` (Number) mới thêm, và
- `RequestIDText` (Text, đã có sẵn) = BatchID ở dạng text, giữ đúng vai trò "text join
  key" như bên `procurement-procedure`.

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
| 3 | text_1 | Text | newFilename (đã gồm extension) — app compose theo naming convention của `Parse_Invoice`: `{yyyyMMdd} - {Supplier} - {BilledTo} - {InvoiceNumber} - {Attention} - {Description} - {Currency}{Total} - GST {Currency}{Tax}.{ext}`, đã sanitize ký tự cấm `\ / : * ? " < > # %`. **KHÁC tên attachment gốc** |
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
| 15 | text_10 | Text | Description (User nhập, required ở chế độ Review) |

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
   - Trước Switch: **Initialize variable** `TargetLibrary` (Type: String, value
     để trống). KHÔNG đặt Compose trong từng nhánh — action nằm trong nhánh
     không khớp sẽ bị **Skipped**, và mọi tham chiếu `outputs('...')` tới action
     Skipped ở các bước sau Switch sẽ lỗi runtime. Biến thì sống xuyên suốt
     flow, nhánh nào chạy cũng gán được.
   - `On`: `@triggerBody()?['text_7']`
   - Case `Australia` → **Set variable** `TargetLibrary` = `AU_Invoices`
   - Case `Malaysia` → Set variable `TargetLibrary` = `MY_Invoices`
   - Case `Singapore` → Set variable `TargetLibrary` = `SG_Invoices`
   - Case `Vietnam` → Set variable `TargetLibrary` = `VN_Invoices`
   - **Default** → action **Terminate** (Failed, message
     `"Unknown Cost Center: " & triggerBody()?['text_7']`) — chặn đứng thay vì
     âm thầm lưu sai chỗ, vì đây là tài liệu tài chính.

3. **Get_Attachment_Content** — OpenApiConnection `HttpRequest` GET, đọc file gốc
   từ `outputs('Compose_File_Path')` (giống hệt action cùng tên trong
   `Submit_Invoice`).

4. **Create_Invoice_File** — SharePoint `CreateFile`:
   - `folderPath`: `@variables('TargetLibrary')` (dynamic theo Cost Center — đã
     được chứng minh hoạt động trong `Submit_Invoice` cũ với `InvoiceRegion`)
   - `name`: `@{triggerBody()?['text_1']}` (đã có extension sẵn — **không** nối
     thêm extension lần nữa)
   - `body`: `@body('Get_Attachment_Content')`
   - `overwrite`: `true` — bắt buộc, là nền tảng của cơ chế retry: bấm Done lại
     sau lỗi giữa chừng sẽ tạo lại file cùng tên, phải ghi đè được thay vì fail.
   - **Allow chunking: OFF** (Settings → Content transfer) — lỗi đã gặp thật:
     khi bật chunking, connector chuyển sang bộ API upload theo mảnh
     (`StartUpload/ContinueUpload/FinishUpload`) vốn **không tôn trọng cờ
     `overwrite`**, nên vẫn fail `"A file with the name ... already exists"`
     dù code view có `"overwrite": true`. Chunking chỉ cần cho file >100MB —
     hóa đơn PDF vài chục KB không bao giờ cần. KHÔNG bật lại.
   - Lưu ý hệ quả: dữ liệu extract KHÔNG còn ghi lên library nữa nên file chỉ có
     **1 version** (chỉ CreateFile). Metadata đi vào record của
     `Procurement_InvoiceData` ở bước 5.

5. **Create_InvoiceData_Item** — SharePoint **PostItem** (Create item), KHÔNG dùng
   HttpRequest MERGE nữa. Đây là action giống hệt `Procurement_InvoiceData` trong
   `Submit_Invoice`, chỉ đổi lại phần map tham số cho đúng thứ tự trigger của flow này.
   - `dataset`: `https://maxbiocare.sharepoint.com/sites/Powerapps`
   - `table`: `3a7218e7-e10f-406a-9a8a-d4cbbb2ea82c` (GUID của `Procurement_InvoiceData`)
   - Map các trường (dùng thứ tự trigger của **Finalize_Invoice** ở mục B — **KHÁC**
     thứ tự của `Submit_Invoice`, không được copy nguyên si):

   | item/... | Giá trị | Ghi chú |
   |---|---|---|
   | `Title` | `@{triggerBody()?['text_1']}` | đã gồm extension — **không** nối thêm `.ext` như `Submit_Invoice` cũ |
   | `InvoiceNumber` | `@triggerBody()?['text_2']` | |
   | `InvoiceDate` | `@if(empty(triggerBody()?['text_3']), null, formatDateTime(if(startsWith(triggerBody()?['text_3'], '/'), substring(triggerBody()?['text_3'], 1, sub(length(triggerBody()?['text_3']), 1)), triggerBody()?['text_3']), 'yyyy-MM-dd'))` | cột kiểu **Date**, phải chuẩn hoá — xem cảnh báo dưới bảng |
   | `VendorName` | `@triggerBody()?['text_4']` | SupplierName |
   | `BilledTo` | `@triggerBody()?['text_5']` | |
   | `TotalAmount` | `@triggerBody()?['number_1']` | |
   | `GSTAmount` | `@triggerBody()?['number_2']` | TaxAmount |
   | `Currency` | `@triggerBody()?['text_6']` | |
   | `Attention` | `@triggerBody()?['text_8']` | |
   | `ABN` | `@triggerBody()?['text_9']` | |
   | `Description` | `@triggerBody()?['text_10']` | |
   | `CostCenter/Value` | `@triggerBody()?['text_7']` | cột Choice → set qua `item/CostCenter/Value` |
   | `BatchID` | `@triggerBody()?['number']` | cột Number mới |
   | `ConfidenceScore` | `@triggerBody()?['number_3']` | cột Number mới |
   | `RequestIDText` | `@{triggerBody()?['number']}` | BatchID dạng text (join key) |
   | `Origin` | `invoice-batch-app` | hằng số cố định |
   | `InvoiceLink` | `@concat('https://maxbiocare.sharepoint.com/sites/Powerapps/', variables('TargetLibrary'), '/', triggerBody()?['text_1'])` | |
   | `ParsedAt` | `@utcNow()` | |

   - **KHÔNG set `item/RequestID/Id`** — lookup này trỏ tới `Procurement_Requests`,
     BatchID không phải id hợp lệ ở đó (xem mục A).
   - **⚠ `InvoiceDate` là cột Date (lỗi đã gặp thật):** map thô `@triggerBody()?['text_3']`
     sẽ fail `Input parameter 'item/InvoiceDate' is required to be of type 'String/date'`
     khi AI trả date lệch định dạng (vd `"/2026-06-12"` có `/` thừa đầu chuỗi). Phải bọc
     bằng biểu thức chuẩn hoá ở bảng trên: bỏ `/` thừa rồi `formatDateTime(..., 'yyyy-MM-dd')`,
     và trả `null` khi trống để không làm hỏng cả action. `ParsedAt` (cũng Date/Time) dùng
     `@utcNow()` nên luôn hợp lệ, không cần xử lý.
   - **Ưu điểm so với MERGE cũ:** PostItem truyền giá trị qua tham số connector, KHÔNG
     ghép JSON thủ công → **không còn bẫy thiếu dấu nháy kép** gây 400
     `InvalidClientQueryException` như phương án MERGE. Bỏ luôn được bước
     `Get_New_File_Item_Id` (không cần Id của file trên library nữa).
   - **Vẫn cần bật Formula-level error management** (Settings → Updates) trong app thì
     `IfError` quanh `Finalize_Invoice.Run(...)` ở `btnDone` mới bắt được lỗi flow —
     nếu tắt, app sẽ coi run lỗi là thành công và đánh dấu `IsFinalized` sai.

   **⚠ Bẫy idempotency mới (khác hẳn MERGE):** MERGE cũ ghi đè cùng 1 item library nên
   chạy lại nhiều lần vẫn ra 1 kết quả. PostItem thì **mỗi lần chạy tạo 1 record MỚI** →
   nếu 1 hóa đơn được finalize lại (ví dụ file CreateFile xong nhưng Delete lỗi, user bấm
   Done lại) sẽ sinh **record trùng** trong `Procurement_InvoiceData`. Hai cách xử lý:
   - **(Khuyến nghị, dựa vào app)** App đã `Patch(... {IsFinalized: true})` sau khi Run
     thành công. Chỉ cần `ForAll` ở `btnDone` lọc `IsFinalized = false` (hoặc kiểm tra cờ
     trước khi gọi) là hóa đơn đã xong sẽ không chạy lại → không trùng. Đây là tuyến
     phòng thủ chính, gần như đủ dùng.
   - **(Chốt chặn trong flow, tùy chọn)** Thêm trước PostItem 1 action **Get items** trên
     `Procurement_InvoiceData` với filter `BatchID eq @{triggerBody()?['number']} and
     Title eq '@{triggerBody()?['text_1']}'`, rồi bọc PostItem trong **Condition**
     `length(...) is equal to 0`. Chỉ tạo record khi chưa tồn tại. Dùng nếu muốn an toàn
     tuyệt đối kể cả khi cờ `IsFinalized` bị lệch.

6. **Delete_Source_Attachment** — HttpRequest POST (X-HTTP-Method: DELETE),
   KHÔNG dùng action connector "Delete attachment" (lỗi đã gặp thật: connector
   resolve File Identifier dạng tên trần thành đường dẫn từ gốc site
   `/sites/Powerapps/<tên file>` → "file does not exist"). Gọi REST trực tiếp:
   ```
   _api/web/lists/getbytitle('BEUI_InvoiceBatch_Requests')/items(@{triggerBody()?['number']})/AttachmentFiles/getByFileName('<tên gốc>')
   ```
   Header: `Accept: application/json;odata=nometadata`, `X-HTTP-Method: DELETE`.
   Tên gốc suy ra từ URL — **KHÔNG dùng `text_1`** (`text_1` là tên mới theo
   naming convention, không trùng tên gốc), nhân đôi dấu nháy đơn nếu có:
   ```
   replace(uriComponentToString(last(split(outputs('Compose_File_Path'), '/'))), '''', '''''')
   ```

   Chỉ xóa **đúng 1 attachment vừa xử lý**, không xóa toàn bộ attachments của
   record — vì flow này chạy tuần tự nhiều lần trong cùng 1 `ForAll` ở
   `BatchUploadScreen.pa.yaml`; xóa toàn bộ ngay từ lần chạy đầu sẽ làm mất file
   của các hóa đơn chưa kịp xử lý trong cùng batch. Sau khi `ForAll` chạy hết,
   kết quả cuối cùng vẫn là toàn bộ attachments đã được xóa — nhưng an toàn theo
   từng bước.

7. **Response** (tùy chọn) — trả về:
   ```
   newInvoiceLink = concat('https://maxbiocare.sharepoint.com/sites/Powerapps/', variables('TargetLibrary'), '/', triggerBody()?['text_1'])
   ```
   Dự phòng nếu sau này app cần hiển thị link — hiện tại
   `BatchUploadScreen.pa.yaml` không capture giá trị trả về nên bước này không
   bắt buộc.

## D. Trạng thái phía `BatchUploadScreen.pa.yaml`

Lời gọi `Finalize_Invoice.Run(...)` trong `btnDone.OnSelect` đã đúng thứ tự 14
tham số. Tham số 3 (`newFilename`) được app compose từ các trường đã review
(`InvoiceDateEdit`, `SupplierNameEdit`, ...) theo đúng naming convention ở mục B
và đã sanitize ký tự cấm — khớp 1:1 với label "Invoice Name Preview" trong
`rowInvoiceDetail`. Nếu compose ra chuỗi rỗng (mọi trường trống), app fallback
về `FileNameBase` (tên gốc bỏ extension).

## Tham chiếu

- Flow cũ tham khảo: `Workflows/Parse_Invoice-*.json`, `Workflows/Submit_Invoice-*.json`
  (từ `procurement-procedure`, dùng cùng SharePoint site và cùng 3 library
  AU/SG/MY_Invoices — `VN_Invoices` là library mới thêm riêng cho app này).
- Lời gọi flow trong app: `Src/BatchUploadScreen.pa.yaml`.
