<!--
Sync Impact Report
- Version change: template (unratified) -> 1.0.0
- Added principles:
  - I. Code đơn giản và giải thích được
  - II. Modular monolith và phân lớp rõ ràng
  - III. Mọi tính năng phải có kiểm thử
  - IV. Học trước khi triển khai
  - V. Phát triển tăng dần theo MVP
- Added sections:
  - Ràng buộc kỹ thuật và an toàn dữ liệu
  - Quy trình phát triển và cổng chất lượng
- Removed sections: none
- Follow-up TODOs: none
-->

# FinSight AI Constitution

## Core Principles

### I. Code đơn giản và giải thích được

Code MUST ưu tiên cách triển khai dễ đọc, tên gọi rõ nghĩa và luồng xử lý trực tiếp. Mỗi abstraction,
dependency hoặc kỹ thuật phức tạp MUST có lý do cụ thể gắn với yêu cầu hiện tại. Người thực hiện MUST
có thể giải thích input, output, business rule, lỗi có thể xảy ra và cách kiểm thử của phần code mình
thêm vào. Không chấp nhận code chỉ vì AI sinh ra nhưng người thực hiện không hiểu.

### II. Modular monolith và phân lớp rõ ràng

Ứng dụng MUST được xây dựng dưới dạng modular monolith. Dependency MUST đi theo hướng
`API -> Service -> Repository -> PostgreSQL`. Route chỉ xử lý HTTP, dependency injection và chuyển
đổi request/response; service sở hữu business rule và transaction boundary; repository sở hữu truy
vấn dữ liệu. Pydantic schema và SQLAlchemy model MUST được tách riêng. Chỉ tách microservice khi có
bằng chứng về nhu cầu deploy, scale hoặc cô lập lỗi độc lập.

### III. Mọi tính năng phải có kiểm thử

Mỗi tính năng hoặc sửa lỗi MUST có automated test ở cấp phù hợp. Business rule thuần MUST có unit
test; API contract, database query, transaction và migration MUST có integration test khi liên quan.
Happy path, validation, authorization và lỗi quan trọng MUST được kiểm tra. Một tính năng chưa hoàn
thành nếu test liên quan chưa chạy thành công.

### IV. Học trước khi triển khai

Trước khi bắt đầu code một tính năng, kế hoạch MUST liệt kê ngắn gọn kiến thức nền cần hiểu và giải
thích chúng bằng ví dụ nhỏ khi đó là khái niệm mới. Việc học MUST theo nhu cầu của task hiện tại,
không yêu cầu học toàn bộ framework trước. Sau khi triển khai, người thực hiện MUST có thể mô tả lại
luồng `request -> route -> service -> repository -> database -> response` của tính năng.

### V. Phát triển tăng dần theo MVP

Mỗi thay đổi MUST tạo ra một lát cắt nhỏ, chạy được và kiểm thử được. Chỉ triển khai phạm vi đã có
acceptance criteria; các ý tưởng OCR, AI, Redis và worker MUST được giữ ở roadmap cho đến khi nền
tảng backend tương ứng hoàn thành. Không thêm abstraction hoặc hạ tầng chỉ để dự phòng cho nhu cầu
chưa được chứng minh. Ưu tiên giải pháp nhỏ nhất đáp ứng đúng yêu cầu hiện tại.

## Ràng buộc kỹ thuật và an toàn dữ liệu

- Backend chính MUST dùng Python, FastAPI và PostgreSQL.
- Cấu hình và secret MUST đến từ biến môi trường; secret và dữ liệu tài chính thật MUST NOT được
  commit vào Git.
- Mọi input từ client MUST được validate; mọi truy vấn dữ liệu người dùng MUST kiểm tra ownership.
- Password, token, raw OCR text và thông tin ngân hàng MUST NOT xuất hiện trong log.
- Dữ liệu tiền tệ MUST dùng kiểu số chính xác phù hợp, không dùng binary floating point.
- Dependency mới MUST được ghi rõ mục đích và chỉ thêm khi thư viện chuẩn hoặc dependency hiện có
  không đáp ứng hợp lý.
- Migration MUST đi kèm mọi thay đổi schema database và phải có khả năng áp dụng trên database mới.

## Quy trình phát triển và cổng chất lượng

Mỗi feature MUST đi qua các bước sau:

1. Viết specification gồm user story, phạm vi, acceptance criteria và trường hợp lỗi.
2. Làm rõ điểm mơ hồ trước khi chọn giải pháp kỹ thuật.
3. Viết plan gồm kiến trúc, API contract, data model, test strategy và kiến thức cần học.
4. Chia thành task nhỏ theo thứ tự dependency; chỉ thực hiện một phạm vi dễ kiểm tra mỗi lần.
5. Triển khai theo đúng layer, viết test liên quan và chạy test trước khi đánh dấu hoàn thành.
6. Chạy formatter/linter, cập nhật tài liệu bị ảnh hưởng và đối chiếu code với specification.

Definition of Done cho một feature gồm: acceptance criteria được đáp ứng; validation và authorization
được xử lý; business logic không nằm trong route; migration tồn tại nếu schema thay đổi; test liên
quan chạy thành công; tài liệu API hoặc README được cập nhật khi hành vi công khai thay đổi.

## Governance

Constitution này là nguồn quy tắc cao nhất cho các specification, plan, task và thay đổi code của
FinSight AI. Mọi review MUST kiểm tra sự tuân thủ; ngoại lệ MUST được ghi trong plan cùng lý do và
phương án loại bỏ hoặc đánh giá lại.

Mọi sửa đổi constitution MUST được mô tả, cập nhật Sync Impact Report và tăng version theo Semantic
Versioning: MAJOR cho thay đổi không tương thích hoặc loại bỏ nguyên tắc; MINOR cho nguyên tắc hay
phần quản trị mới hoặc mở rộng đáng kể; PATCH cho diễn đạt rõ hơn mà không đổi ý nghĩa. Ngày sửa đổi
MUST được cập nhật khi nội dung quản trị thay đổi.

Khi có xung đột, constitution này ưu tiên hơn tài liệu kế hoạch cũ. Việc tuân thủ MUST được kiểm tra
trước khi implement và khi chạy bước converge của mỗi feature.

**Version**: 1.0.0 | **Ratified**: 2026-10-13 | **Last Amended**: 2026-10-13
