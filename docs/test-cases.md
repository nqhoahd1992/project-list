# Test Case — Project List App

> Cập nhật: 2026-07-13. Tài liệu nghiệp vụ liên quan: [approval-workflow-plan.md](approval-workflow-plan.md), [sharepoint-schema.md](sharepoint-schema.md).

## 1. Tổng quan

- **Mục đích**: quản lý danh sách dự án (Project_List) của công ty; mọi thay đổi (thêm mới, chỉnh sửa, xóa dự án) đều phải đi qua một **Change Request (CR)** và được phê duyệt qua 2 cấp (Manager → Sponsor) trước khi áp dụng vào danh sách chính thức.
- **Link ứng dụng**: https://apps.powerapps.com/play/e/default-6292eb61-3778-47c8-b769-a982698418cc/a/4c275f4f-228e-4dbd-af2b-fc3835168257?tenantId=6292eb61-3778-47c8-b769-a982698418cc&hint=6312083d-d1ea-406d-bf3c-6cb96c9b5a82&source=sharebutton&sourcetime=1783911983795
- **Nguồn dữ liệu (SharePoint list)**:
  - `Project_List` — https://maxbiocare.sharepoint.com/sites/Powerapps/Lists/Project_List/AllItems.aspx (danh sách dự án chính thức, chỉ được ghi sau khi CR được duyệt đủ 2 cấp)
  - `Project_ChangeRequests` — https://maxbiocare.sharepoint.com/sites/Powerapps/Lists/Project_ChangeRequests/AllItems.aspx (lưu các yêu cầu Create/Update/Delete và trạng thái phê duyệt)
  - `Project_ApprovalLog` — https://maxbiocare.sharepoint.com/sites/Powerapps/Lists/Project_ApprovalLog/AllItems.aspx (nhật ký quyết định của Manager/Sponsor ở từng bước)
  - `Project_User` — https://maxbiocare.sharepoint.com/sites/Powerapps/Lists/Project_User/AllItems.aspx (gán vai trò `Manager`/`Sponsor` cho nhân viên; ai không có dòng active ở đây mặc định là `Requester`)

### 1.1 Vai trò trong hệ thống

| Vai trò | Nguồn | Quyền chính |
|---|---|---|
| Requester (mặc định, không phải giá trị lưu trong SharePoint) | Không có dòng active trong `Project_User` | Xem toàn bộ danh sách dự án; tạo CR (Create/Update/Delete) cho dự án mình đứng tên `RequestedBy` |
| Manager | `Project_User.Role = "Manager"`, `IsActive = true` | Phê duyệt Bước 1 (Manager Approval) cho các CR mà mình là `ProjectManager` |
| Sponsor | `Project_User.Role = "Sponsor"`, `IsActive = true` | Gán Manager cho CR Create (Bước 0); phê duyệt Bước 2 (Sponsor Approval) — bước áp dụng thay đổi vào `Project_List`; bật/tắt "Show Deleted" |

### 1.2 Trạng thái Change Request (`ApprovalStatus`)

```
Create  : Pending Sponsor → (Sponsor gán Manager) → Pending Manager Approval → Pending Sponsor Approval → Approved / Rejected
Update  : Pending Manager Approval → Pending Sponsor Approval → Approved / Rejected   (bỏ qua Pending Sponsor)
Delete  : Pending Manager Approval → Pending Sponsor Approval → Approved / Rejected   (bỏ qua Pending Sponsor)
```

`Approved`/`Rejected` là trạng thái **kết thúc** (terminal), không quay lại được.

## 2. Phạm vi & môi trường test

- Test trên **web player** của Power Apps (link ở mục 1), độ phân giải Desktop/Tablet 1366×768.
- Cần tối thiểu 4 tài khoản test đã có `Employee List`:
  1. **Requester** — không có dòng trong `Project_User` (hoặc `IsActive = false`).
  2. **Manager A** — `Project_User.Role = "Manager"`, `IsActive = true`, được gán làm `ProjectManager` của ít nhất 1 dự án/CR test.
  3. **Sponsor A** — `Project_User.Role = "Sponsor"`, `IsActive = true`.
  4. **Manager B / Sponsor B** (khác Manager A/Sponsor A) — dùng để test rằng người *không* phải người phụ trách CR thì không thấy/không thao tác được.
- Cần sẵn ít nhất: 1 dự án Root (`ProjectLevel = 0`) đang active, 1 dự án Root có Child đang active, 1 dự án đã `Deleted`.

## 3. Danh sách Test Case

Chú thích độ ưu tiên: **Cao** (chặn luồng chính/bảo mật), **Trung bình** (nghiệp vụ phụ, validation), **Thấp** (hiển thị, tiện ích).

### 3.1 Nhóm A — Danh sách dự án (AllProjectsScreen)

| ID | Tên test case | Điều kiện tiền đề | Các bước thực hiện | Kết quả mong đợi | Ưu tiên |
|---|---|---|---|---|---|
| A-01 | Xem danh sách dự án mặc định | Đăng nhập bằng bất kỳ tài khoản nào | Mở app | Hiển thị gallery tất cả dự án **chưa bị xóa**, sắp xếp theo ID giảm dần; ô tìm kiếm, bộ lọc trạng thái/cấp độ hiển thị đầy đủ | Cao |
| A-02 | Tìm kiếm theo tên dự án | Có ≥1 dự án tên "ABC" | Gõ "ABC" vào ô tìm kiếm | Chỉ hiển thị dự án có tên hoặc Project ID bắt đầu bằng "ABC" (không phân biệt hoa/thường) | Trung bình |
| A-03 | Lọc theo trạng thái dự án | — | Chọn từng pill: Not Started / In Planning / In Progress / On Hold / Completed / Cancelled | Gallery chỉ hiển thị đúng dự án có `ProjectStatus` tương ứng | Trung bình |
| A-04 | Lọc theo cấp độ (Root/Child) | Có cả dự án Root và Child | Chọn pill "Root", sau đó "Child" | "Root" chỉ hiện `ProjectLevel = 0`; "Child" chỉ hiện `ProjectLevel = 1` | Trung bình |
| A-05 | Nút "Show Deleted" chỉ dành cho Sponsor | Đăng nhập bằng Requester/Manager | Quan sát khu vực filter bar | Không thấy toggle "Show Deleted" | Cao |
| A-06 | Sponsor bật "Show Deleted" | Đăng nhập Sponsor, có ≥1 dự án `Deleted` | Bật toggle "Show Deleted" | Dự án đã xóa hiện trong danh sách (có thể phân biệt bằng trạng thái/màu); tắt toggle thì ẩn lại | Trung bình |
| A-07 | Badge "Update pending"/"Delete pending" | Có 1 dự án đang có CR ở trạng thái Pending Manager/Sponsor Approval | Xem dòng dự án đó trong gallery | Hiển thị badge tương ứng loại CR đang chờ | Trung bình |
| A-08 | Nút Update/Delete chỉ hiện cho chủ dự án | Đăng nhập bằng người **không phải** `RequestedBy` của 1 dự án | Xem dòng dự án đó | Không thấy nút Update/Delete, chỉ thấy nút View | Cao |
| A-09 | Nút Update/Delete ẩn khi đã có CR pending | Đăng nhập bằng `RequestedBy`, dự án đang có CR pending | Xem dòng dự án đó | Nút Update/Delete không hiển thị | Cao |
| A-10 | Nút Update/Delete ẩn với dự án đã Deleted | Sponsor bật Show Deleted, xem 1 dự án Deleted do chính mình tạo | Xem dòng dự án | Không có nút Update/Delete dù là `RequestedBy` | Trung bình |
| A-11 | Chặn xóa Root có Child active | `RequestedBy` của 1 Root project có ≥1 Child chưa xóa | Bấm nút Delete trên dòng Root đó | Hiện `Notify` báo lỗi, **không** điều hướng sang DeleteProjectScreen | Cao |
| A-12 | "PROJ-PENDING-<ID>" cho dự án chưa có ProjectID | Có dự án với `ProjectID` rỗng (giả lập lỗi giữa 2 lần Patch khi Apply) | Xem dòng dự án đó | Hiển thị `PROJ-PENDING-<ID>` thay vì để trống | Thấp |
| A-13 | Nút "Approvals" hiện đúng điều kiện | Đăng nhập Requester chưa từng là Sponsor của CR nào | Xem toolbar | Không hiện nút Approvals (vì `gIsApprover = false` và `gApprovalsPendingCount = 0`) | Trung bình |
| A-14 | Badge số lượng chờ duyệt | Đăng nhập Manager A đang có N CR chờ ở Bước 1 | Xem nút "Approvals" | Hiển thị đúng số N: `Approvals (N)` | Trung bình |
| A-15 | "+ New Project" | Bất kỳ ai | Bấm nút | Điều hướng sang CreateProjectScreen | Thấp |

### 3.2 Nhóm B — Tạo dự án mới (CreateProjectScreen → CR type Create)

| ID | Tên test case | Điều kiện tiền đề | Các bước thực hiện | Kết quả mong đợi | Ưu tiên |
|---|---|---|---|---|---|
| B-01 | Submit đầy đủ thông tin hợp lệ (Root project) | Đăng nhập bất kỳ | Điền đủ tất cả field bắt buộc, chọn Level = Root, Submit | Tạo CR thành công với `ApprovalStatus = "Pending Sponsor"`, `RequestType = "Create"`; gửi thông báo `SubmittedToSponsor` tới Sponsor đã chọn; điều hướng về AllProjectsScreen | Cao |
| B-02 | Submit dự án Child hợp lệ | Có ≥1 Root project active | Chọn Level = Child, chọn Root Project, điền đủ field, Submit | CR tạo thành công, lưu đúng `RootProjectID` = ProjectID của Root đã chọn | Cao |
| B-03 | Thiếu Project Name | — | Để trống Project Name, Submit | Báo lỗi yêu cầu nhập tên, không tạo CR | Trung bình |
| B-04 | Thiếu Project Type / Department / Priority / Cost Center | — | Bỏ trống lần lượt từng field, Submit | Mỗi trường hợp báo đúng lỗi tương ứng, không tạo CR | Trung bình |
| B-05 | Thiếu Sponsor | — | Không chọn Sponsor, Submit | Báo lỗi yêu cầu chọn Sponsor | Cao |
| B-06 | Danh sách Sponsor chỉ gồm Sponsor active | Có 1 Sponsor với `IsActive = false` | Mở dropdown Sponsor | Sponsor inactive không xuất hiện trong danh sách chọn | Trung bình |
| B-07 | Chọn Level = Child nhưng không chọn Root Project | — | Chọn Child, không chọn Root Project, Submit | Báo lỗi yêu cầu chọn Root Project | Trung bình |
| B-08 | Dropdown Root Project chỉ gồm Root đang active | Có Root project đã `Deleted` | Mở dropdown Root Project | Root đã Deleted không xuất hiện | Trung bình |
| B-09 | Ngày kết thúc bằng ngày bắt đầu | — | `EndDate = StartDate`, điền đủ các field khác, Submit | **Được chấp nhận** (Create chỉ chặn `EndDate < StartDate`) | Trung bình |
| B-10 | Ngày kết thúc trước ngày bắt đầu | — | `EndDate < StartDate`, Submit | Báo lỗi "End Date must be on or after Start Date", không tạo CR | Cao |
| B-11 | Nhập chữ vào Budget Amount | — | Gõ ký tự không phải số vào Budget Amount, Submit | Ghi nhận hành vi thực tế (không có validate số riêng) — kỳ vọng: nên báo lỗi hoặc lưu 0/blank chứ không được lưu sai giá trị số; **đây là test khám phá lỗi tiềm ẩn**, ghi lại kết quả thực tế nếu khác kỳ vọng | Trung bình |
| B-12 | Trường Currency tự sinh theo Market | — | Chọn Market lần lượt AU/MY/SG/VN | Currency tự hiển thị tương ứng AUD/MYR/SGD/VND, không cho sửa tay | Thấp |
| B-13 | Chọn nhiều SKU (Related Product SKU) | — | Chọn 2-3 SKU trong `cmbSKU` | CR lưu đúng danh sách SKU đã chọn (field tùy chọn, có thể để trống) | Thấp |
| B-14 | Không kiểm tra trùng tên dự án (gap nghiệp vụ) | — | Tạo 2 CR Create với cùng Project Name | Cả 2 đều được tạo thành công — xác nhận hệ thống hiện **không** chặn trùng tên; báo lại nếu đây không phải hành vi mong muốn | Trung bình |
| B-15 | Hủy tạo mới | Đang điền dở form | Bấm Cancel | Điều hướng về AllProjectsScreen, **không** có xác nhận mất dữ liệu, dữ liệu đã nhập bị mất | Thấp |
| B-16 | ProjectManager không được chọn ở bước tạo | — | Quan sát form Create | Không có field chọn Project Manager (chỉ chọn Sponsor); Manager sẽ do Sponsor gán sau ở bước duyệt | Trung bình |

### 3.3 Nhóm C — Xem chi tiết dự án (ViewProjectScreen)

| ID | Tên test case | Điều kiện tiền đề | Các bước thực hiện | Kết quả mong đợi | Ưu tiên |
|---|---|---|---|---|---|
| C-01 | Xem đầy đủ thông tin dự án | Bất kỳ ai, bấm View trên 1 dự án | Mở màn hình | Hiển thị đủ: Type, Department, Priority, Sponsor, Manager, RequestedBy, Status, Start/End date, Budget, ActualCost, CapEx/OpEx, Market, Channel, Strategic Objective, Cost Center, Related SKU, Project Documents, Description | Trung bình |
| C-02 | Field trống hiển thị "—" | Dự án có `ActualCost`/`RequestedBy`/`RelatedSKU` trống | Xem chi tiết | Các field trống hiển thị "—" thay vì để trắng | Thấp |
| C-03 | Deliverables chỉ hiện với người liên quan | Đăng nhập bằng người **không phải** RequestedBy/Sponsor/Manager của dự án | Xem chi tiết dự án đó | Không hiển thị dòng Deliverables, các thông tin khác vẫn hiển thị đầy đủ | Cao |
| C-04 | Deliverables hiện với RequestedBy/Sponsor/Manager | Đăng nhập lần lượt bằng 3 vai trò này của cùng 1 dự án | Xem chi tiết | Cả 3 đều thấy dòng Deliverables | Cao |
| C-05 | Đóng màn hình | — | Bấm Close | Quay về AllProjectsScreen | Thấp |

### 3.4 Nhóm D — Yêu cầu chỉnh sửa (UpdateProjectScreen → CR type Update)

| ID | Tên test case | Điều kiện tiền đề | Các bước thực hiện | Kết quả mong đợi | Ưu tiên |
|---|---|---|---|---|---|
| D-01 | Submit yêu cầu update hợp lệ | Đăng nhập RequestedBy của 1 dự án, không có CR pending | Sửa Description/Deliverables/End Date, nhập Reason, Submit | Tạo CR `RequestType = "Update"`, `ApprovalStatus = "Pending Manager Approval"` (bỏ qua Pending Sponsor); gửi `SubmittedToManager` tới `ProjectManager` hiện tại của dự án | Cao |
| D-02 | Chỉ sửa 1 trong 3 field | — | Chỉ đổi Description, giữ nguyên Deliverables/End Date, nhập Reason, Submit | CR vẫn tạo thành công với đúng field đã đổi | Trung bình |
| D-03 | Không đổi gì | — | Không sửa Description/Deliverables/EndDate, chỉ nhập Reason, Submit | Hiện cảnh báo "No changes to submit", **không** tạo CR | Cao |
| D-04 | Thiếu Update Reason | Có sửa ít nhất 1 field | Để trống Reason, Submit | Báo lỗi yêu cầu nhập lý do, không tạo CR | Cao |
| D-05 | New End Date bằng Start Date | — | Đặt `NewEndDate = StartDate`, Submit | Bị từ chối — Update yêu cầu **strictly greater than** Start Date (khác Create cho phép bằng) | Cao |
| D-06 | New End Date sau Start Date 1 ngày | — | `NewEndDate = StartDate + 1`, Submit | CR tạo thành công | Trung bình |
| D-07 | Nút Submit bị khóa khi đã có CR pending | Dự án đang có CR pending | Mở lại UpdateProjectScreen (qua URL/nav trực tiếp nếu có thể) | Nút Submit ở trạng thái Disabled | Cao |
| D-08 | Giữ nguyên End Date (không nhập) | — | Không chọn ngày mới ở `dpNewEndDate` (placeholder "Keep current date"), chỉ đổi Description, Submit | CR lưu End Date = ngày hiện tại của dự án (không đổi) | Trung bình |

### 3.5 Nhóm E — Yêu cầu xóa (DeleteProjectScreen → CR type Delete)

| ID | Tên test case | Điều kiện tiền đề | Các bước thực hiện | Kết quả mong đợi | Ưu tiên |
|---|---|---|---|---|---|
| E-01 | Submit yêu cầu xóa hợp lệ | RequestedBy của 1 dự án không có Child active, không có CR pending | Tick checkbox xác nhận, nhập lý do, Submit | Tạo CR `RequestType = "Delete"`, `ApprovalStatus = "Pending Manager Approval"`; gửi `SubmittedToManager` | Cao |
| E-02 | Chưa tick checkbox xác nhận | — | Không tick checkbox, nhập lý do, Submit | Báo lỗi yêu cầu tick xác nhận, không tạo CR | Cao |
| E-03 | Thiếu lý do xóa | Đã tick checkbox | Để trống Delete Reason, Submit | Báo lỗi yêu cầu nhập lý do | Cao |
| E-04 | Nút Submit khóa khi có Child active | Root project có Child chưa xóa | Mở DeleteProjectScreen của Root đó | Banner cảnh báo hiện ra, nút Submit ở trạng thái Disabled | Cao |
| E-05 | Nút Submit khóa khi đã có CR pending | Dự án đang có CR pending khác | Mở DeleteProjectScreen | Banner cảnh báo CR pending, nút Submit Disabled | Cao |
| E-06 | Sau khi xóa root hết Child, xóa được | Root từng bị chặn xóa vì có Child, nay tất cả Child đã bị xóa | Thử lại xóa Root | Không còn bị chặn, submit thành công | Trung bình |

### 3.6 Nhóm F — Phê duyệt (ApprovalsScreen)

| ID | Tên test case | Điều kiện tiền đề | Các bước thực hiện | Kết quả mong đợi | Ưu tiên |
|---|---|---|---|---|---|
| F-01 | Tab "To Approve" hiện đúng CR theo vai trò | Manager A có CR đang `Pending Manager Approval` với `ProjectManager` = Manager A | Đăng nhập Manager A, mở tab "To Approve" | Chỉ thấy đúng những CR mình cần duyệt, không thấy CR của Manager khác | Cao |
| F-02 | Manager khác không thấy/không duyệt được CR không thuộc mình | CR đang chờ Manager A duyệt | Đăng nhập Manager B, mở "To Approve" | Không thấy CR đó trong danh sách; nếu cố truy cập trực tiếp thì action bar (Approve/Reject) bị ẩn | Cao |
| F-03 | Tab "Assign Manager" chỉ dành cho Sponsor đúng CR | CR Create đang `Pending Sponsor` với `ProjectSponsor` = Sponsor A | Đăng nhập Sponsor A | Thấy CR trong tab "Assign Manager", chọn Manager rồi bấm "Confirm" | `ProjectManager` được set, `ApprovalStatus → "Pending Manager Approval"`, gửi `SubmittedToManager`; **không** tạo dòng `Project_ApprovalLog` cho bước này | Cao |
| F-04 | Sponsor B không thấy CR không thuộc mình ở tab Assign Manager | Cùng CR trên | Đăng nhập Sponsor B | Không thấy CR đó ở tab Assign Manager | Cao |
| F-05 | Manager duyệt (Approve) Bước 1 | CR `Pending Manager Approval`, đăng nhập đúng Manager | Bấm Approve | Ghi `Project_ApprovalLog` (StepNumber=1, Action=Approved); `ApprovalStatus → "Pending Sponsor Approval"`; gửi `ManagerApprovedToSponsor` tới Sponsor | Cao |
| F-06 | Manager từ chối (Reject) Bước 1 không nhập remark | CR `Pending Manager Approval` | Bấm Reject, để trống remark, Confirm | Báo lỗi "A remark is required to reject", CR không đổi trạng thái | Cao |
| F-07 | Manager từ chối Bước 1 có remark | — | Bấm Reject, nhập remark, Confirm | Ghi log (StepNumber=1, Rejected, Remark); `ApprovalStatus → "Rejected"` (kết thúc); gửi `FinalRejected` cho Requester | Cao |
| F-08 | Sponsor duyệt (Approve & Apply) Bước 2 — Create | CR Create `Pending Sponsor Approval` | Bấm "Approve & Apply" | Tạo dòng mới trong `Project_List` với đầy đủ field đề xuất + sinh `ProjectID` đúng định dạng `PROJ-<Market>-<DeptCode>-<Year>-<seq3>`; ghi log (StepNumber=2, Approved); CR → `"Approved"`; gửi `FinalApproved` (Requester) + `FinalApprovedManagerCopy` (Manager) | Cao |
| F-09 | Sponsor duyệt Bước 2 — Update | CR Update `Pending Sponsor Approval` | Bấm "Approve & Apply" | Chỉ cập nhật `ProjectDescription`, `Deliverables`, `EndDate` trên dòng `Project_List` gốc, các field khác giữ nguyên; log + notify như trên | Cao |
| F-10 | Sponsor duyệt Bước 2 — Delete | CR Delete `Pending Sponsor Approval` | Bấm "Approve & Apply" | `ProjectStatus` của dự án gốc chuyển thành `"Deleted"` (soft-delete, không xóa hẳn dòng dữ liệu); log + notify như trên | Cao |
| F-11 | Sponsor từ chối Bước 2 không remark | CR `Pending Sponsor Approval` | Bấm Reject, để trống remark | Báo lỗi bắt buộc remark | Cao |
| F-12 | Sponsor từ chối Bước 2 có remark | — | Bấm Reject, nhập remark, Confirm | Log (StepNumber=2, Rejected, Remark); CR → `"Rejected"`; gửi `FinalRejected` (Requester) **và** `FinalRejectedManagerCopy` (Manager) — khác Bước 1 chỉ gửi Requester | Cao |
| F-13 | Apply thất bại ở Bước 2 vẫn giữ trạng thái chờ duyệt | Giả lập lỗi ghi (vd. field Choice không khớp) khi Approve & Apply | Bấm Approve & Apply | Hiện thông báo lỗi; CR **giữ nguyên** `"Pending Sponsor Approval"` (không bị đẩy thành Approved), có thể duyệt lại | Cao |
| F-14 | Tab "Mine" | Requester đã tạo ≥2 CR | Mở tab "My Requests" | Hiển thị đúng tất cả CR do chính mình gửi (mọi trạng thái) | Trung bình |
| F-15 | Tab "History" chỉ dành cho Manager/Sponsor | Đăng nhập Requester thường | Quan sát các tab | Không thấy tab "History" | Trung bình |
| F-16 | Tab "History" hiển thị đúng phạm vi | Manager A có CR đã Approved/Rejected liên quan đến dự án mình phụ trách | Đăng nhập Manager A, mở "History" | Chỉ thấy CR đã kết thúc (Approved/Rejected) liên quan tới các dự án mình là Manager, không thấy toàn bộ hệ thống | Trung bình |
| F-17 | Diff view khi xem CR Update | CR Update đang chờ duyệt | Sponsor/Manager mở chi tiết CR | Hiển thị bảng so sánh giá trị hiện tại vs. đề xuất (Description/Deliverables/EndDate), trường thay đổi được tô màu nổi bật | Thấp |
| F-18 | Cảnh báo "target no longer exists" | CR Update/Delete nhưng dự án gốc đã bị xóa/không còn tồn tại trước khi duyệt | Mở chi tiết CR đó | Hiện cảnh báo dự án gốc không còn tồn tại | Thấp |
| F-19 | Badge số lượng ở từng tab khớp với dữ liệu thực tế | Manager A có N CR chờ Bước 1, Sponsor A có M CR chờ Assign Manager | Quan sát số đếm trên tab | Số hiển thị khớp chính xác với số CR thực sự hiển thị khi mở tab | Trung bình |

### 3.7 Nhóm G — Phân quyền & bảo mật (cross-cutting)

| ID | Tên test case | Điều kiện tiền đề | Các bước thực hiện | Kết quả mong đợi | Ưu tiên |
|---|---|---|---|---|---|
| G-01 | Requester không thấy action bar phê duyệt | Đăng nhập Requester, mở 1 CR bất kỳ qua tab "Mine" | Xem chi tiết CR | Không có nút Approve/Reject/Assign Manager | Cao |
| G-02 | Đổi Manager/Sponsor của dự án giữa lúc CR đang chờ duyệt | CR đang `Pending Manager Approval`, sau đó đổi `ProjectManager` của dự án (qua SharePoint) | Manager cũ mở lại CR | Xác nhận hành vi thực tế: quyền duyệt bám theo `ProjectManager` **tại thời điểm kiểm tra** (không snapshot) — kiểm tra 3-4 vị trí (gallery filter, action bar, badge count) có nhất quán với nhau không | Trung bình |
| G-03 | Tài khoản không có trong `Employee List` | Đăng nhập bằng tài khoản lạ | Mở app | Hiện thông báo "account not found", không cho thao tác | Cao |
| G-04 | Chỉnh sửa CR trực tiếp qua SharePoint UI (rủi ro đã biết) | Có quyền edit `Project_ChangeRequests` ngoài app | Sửa trực tiếp trạng thái CR trên SharePoint | Ghi nhận đây là rủi ro đã biết (governance ngoài app), không phải lỗi app — dùng để đối chiếu khi audit | Thấp |

### 3.8 Nhóm H — Thông báo (Project_Notify flow)

| ID | Tên test case | Điều kiện tiền đề | Các bước thực hiện | Kết quả mong đợi | Ưu tiên |
|---|---|---|---|---|---|
| H-01 | Thông báo đầy đủ theo từng bước của luồng happy-path | Thực hiện trọn vẹn 1 CR Create từ Submit → Approved | Theo dõi hộp thư của Requester/Sponsor/Manager ở từng bước | Nhận đủ 5 email/Teams card: `SubmittedToSponsor`, `SubmittedToManager`, `ManagerApprovedToSponsor`, `FinalApproved`, `FinalApprovedManagerCopy` | Cao |
| H-02 | `FinalApprovedManagerCopy`/`FinalRejectedManagerCopy` có thể chưa hoạt động | — | Kiểm tra 2 loại thông báo này trên môi trường thật | Nếu flow Power Automate chưa được cập nhật case tương ứng, các lệnh gọi này sẽ rơi vào nhánh `Default → Terminate(Failed)` — ghi nhận là defect đã biết cần theo dõi | Trung bình |
| H-03 | Fallback email khi người nhận không xác định được | Giả lập `ProjectManager`/`ProjectSponsor` có Email trống trong `Employee List` | Thực hiện bước gửi thông báo tương ứng | Email được gửi tới hộp thư fallback `app.admin@maxbiocare.com` thay vì lỗi/không gửi | Thấp |

### 3.9 Nhóm I — Trường hợp biên & toàn vẹn dữ liệu

| ID | Tên test case | Điều kiện tiền đề | Các bước thực hiện | Kết quả mong đợi | Ưu tiên |
|---|---|---|---|---|---|
| I-01 | Không cho tạo CR Update/Delete thứ 2 khi đã có 1 CR đang chờ | Dự án đang có CR `Pending Manager/Sponsor Approval` | RequestedBy cố mở lại Update hoặc Delete cho dự án đó | Nút Update/Delete trên gallery đã ẩn từ trước; nếu vào được màn hình thì nút Submit bị Disabled | Cao |
| I-02 | Hai CR Create trùng tên dự án | — | Submit 2 CR Create với `txtProjectName` giống hệt nhau | Cả 2 được tạo — xác nhận hệ thống hiện không có ràng buộc unique tên/mã dự án | Trung bình |
| I-03 | ProjectID sinh đúng định dạng | Sponsor Approve & Apply 1 CR Create | Kiểm tra `ProjectID` của dòng mới trong `Project_List` | Đúng định dạng `PROJ-<Market>-<DeptCode>-<Year>-<seq3>`, không trùng với ProjectID đã có | Trung bình |
| I-04 | Xóa mềm không làm mất dữ liệu | CR Delete được Approve | Kiểm tra dòng dự án trong `Project_List` sau khi duyệt | Dòng dữ liệu vẫn còn, chỉ đổi `ProjectStatus = "Deleted"`, không bị xóa vật lý | Cao |
| I-05 | Dự án Deleted bị loại khỏi các danh sách chọn liên quan | Có dự án Root đã Deleted | Mở CreateProjectScreen, chọn Level = Child | Root đã Deleted không xuất hiện trong `cmbRootProject` | Trung bình |
| I-06 | `RequestedBy` trống trên dữ liệu cũ (legacy) | Có dòng `Project_List` với `RequestedBy` trống (dữ liệu nhập tay/di trú) | Đăng nhập bất kỳ ai, kể cả người tạo gốc | Nút Update/Delete không hiện với bất kỳ ai trên dòng đó | Thấp |
| I-07 | Giới hạn đếm CountRows khi số lượng CR lớn | Môi trường có > 500 CR (giới hạn non-delegable mặc định) | Kiểm tra badge số lượng chờ duyệt trong điều kiện dữ liệu lớn | Ghi nhận nếu số đếm không chính xác vượt ngưỡng delegation — đây là giới hạn đã biết của SharePoint connector | Thấp |

## 4. Ma trận truy vết (Traceability) theo màn hình

| Màn hình | Test case liên quan |
|---|---|
| AllProjectsScreen | A-01 → A-15, G-01, I-01, I-05 |
| CreateProjectScreen | B-01 → B-16, I-02, I-03 |
| ViewProjectScreen | C-01 → C-05 |
| UpdateProjectScreen | D-01 → D-08, I-01 |
| DeleteProjectScreen | E-01 → E-06, I-01, I-04 |
| ApprovalsScreen | F-01 → F-19, G-02, H-01 → H-03 |
