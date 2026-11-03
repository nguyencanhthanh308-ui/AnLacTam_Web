# Web đặt bàn và menu theo ngày — quán chay An Lạc Tâm

Trạng thái: **bản nháp chờ duyệt** (03/11/2026). Chưa có code.

## 1. Mục đích

Quán chay An Lạc Tâm (Đà Nẵng) cần một web thật để:
- Khách xem **thực đơn của ngày hôm nay** và **đặt bàn** ngay trên điện thoại.
- Chủ quán và nhân viên **cập nhật menu mỗi ngày** và **xử lý đơn đặt bàn** mà không cần biết kỹ thuật.

Lý do: khách thường vào xem hôm nay quán có món gì rồi mới quyết định đến hoặc đặt. Hiện quán chỉ có trang Facebook và thực đơn in, nên menu theo ngày và đơn đặt bàn chưa có chỗ nào gom lại.

## 2. Mục tiêu, không làm, và điều phải tránh

**Mục tiêu**
- Khách mở web trên điện thoại là thấy menu hôm nay, chia theo nhóm món, món hết hàng hiện mờ.
- Khách gửi được đơn đặt bàn trong dưới 1 phút.
- Nhân viên nhận thông báo đơn mới qua Gmail, xác nhận hoặc từ chối trên trang quản lý.
- Chi phí cố định gần như 0đ.

**Không làm (bản đầu)**
- Không thanh toán online, không đặt cọc.
- Không gọi món / đặt đồ mang về.
- Không thông báo Zalo tự động (chỉ chừa chỗ gắn sau).
- Không sơ đồ bàn, không tự tính bàn trống; nhân viên tự quyết khi xác nhận.

**Điều phải tránh (thất bại dù chạy được)**
- Giao diện lệch thương hiệu (xem mục 5): người dùng đã từ chối một bản mẫu sai tone.
- Quy trình cập nhật menu hằng ngày phức tạp hơn việc viết lại menu bằng tay.
- Nhân viên bỏ sót đơn đặt bàn vì không biết có đơn mới.

## 3. Ràng buộc
- Web thật, dùng hằng ngày; ưu tiên miễn phí, **không cần thẻ thanh toán** khi đăng ký.
- Chủ quán và nhân viên dùng nhiều loại thiết bị: web phải chạy tốt trên điện thoại, máy tính bảng và máy tính, không cần cài app.
- Giao diện tiếng Việt.
- Quán có 2 chi nhánh: 117 Phan Đăng Lưu và 41 Lê Quý Đôn (Đà Nẵng). Giờ mở cửa tham khảo 6:30–14:00 và 16:00–20:00; **giờ chính xác từng chi nhánh cần chủ quán xác nhận**.

## 4. Các cách xây đã cân nhắc

| | A. Firebase | B. Supabase + trang tĩnh (**chọn**) | C. Google Sheets + Apps Script |
|---|---|---|---|
| Ưu | Có sẵn đăng nhập, dữ liệu | Có sẵn đăng nhập và phân quyền theo vai trò; dữ liệu dạng bảng hợp với menu và đơn; không cần thẻ | Miễn phí hoàn toàn; gửi Gmail rất dễ |
| Nhược | Muốn tự gửi email phải nâng lên gói trả theo mức dùng (cần thẻ) | Dự án miễn phí tự tạm dừng nếu 1 tuần không ai dùng | Chậm, giao diện quản lý kém, phân quyền yếu, khó thêm Zalo |

Chọn **B** vì khớp yêu cầu đăng nhập 2 mức và gửi Gmail mà không cần thẻ. Nguồn giá và giới hạn gói miễn phí là các trang tổng hợp bên thứ ba; **người làm phải kiểm tra lại trên trang chính thức của Supabase và Cloudflare trước khi bắt đầu**.

## 5. Giao diện và thương hiệu (đã chốt)

Bố cục trang chủ chọn **A**: menu hôm nay nằm trên cùng, nút "Đặt bàn" luôn nổi ở đáy màn hình. Bản mẫu: `docs/mockups/customer-home-A-v2.html`.

Bảng màu đo từ poster và ảnh tường logo do chủ quán gửi, người dùng đã xác nhận đúng tone:

| Vai trò | Màu |
|---|---|
| Nền vàng nghệ | `#F7BF4F` (đậm hơn `#EFA114`) |
| Chữ, logo, nút (nâu đỏ) | `#6A0D05` |
| Nét cọ nhãn nhóm món | `#4E9625` |
| Xanh lá sen (viền ảnh, điểm nhấn) | `#1B6838` |
| Xanh hoa sen (điểm nhấn nhẹ) | `#9CC0DB` |
| Dải họa tiết đáy trang | `#602E2F` |

Phong cách: nhãn nét cọ xanh với chữ viết tay vàng nhạt, khung nét đứt bo tròn quanh từng nhóm món, vân giấy mờ trên nền vàng, dải họa tiết nâu ở đáy trang. Font trong bản mẫu: Anton (logo, giá), Dancing Script (chữ viết tay), Be Vietnam Pro (nội dung); **người làm có thể thay font gần tương đương** nhưng phải giữ cảm giác này. Ảnh món trong bản thật là ảnh cắt nền đặt trên lá chuối như poster; **cần chủ quán cung cấp ảnh**, trong lúc chờ dùng ô tròn giữ chỗ.

Không dùng tông đỏ mận + kem (suy từ ảnh khách chụp trên Foody, đã bị bác).

## 6. Phần khách

- **Trang chủ:** tên quán, giờ mở cửa; "Thực đơn hôm nay · Thứ X"; chip chọn nhóm (Sáng, Trưa/tối, Gỏi, Lẩu...); danh sách món theo nhóm (tên, mô tả ngắn, giá, ảnh); món tắt hôm nay hiện mờ kèm "Hôm nay hết món"; nút "Đặt bàn" cố định ở đáy.
- **Form đặt bàn:** chi nhánh, ngày, giờ, số người, họ tên, **số điện thoại (bắt buộc)**, ghi chú. Chỉ cho chọn ngày giờ trong giờ mở cửa của chi nhánh đó. Sau khi gửi hiện thông báo "Đơn đang chờ quán xác nhận, quán sẽ gọi hoặc nhắn lại cho bạn".
- Khách **không cần đăng ký tài khoản**.

## 7. Phần quản lý (chủ quán và nhân viên)

- **Đăng nhập** (email + mật khẩu qua Supabase). *Đề xuất của người viết, chưa chốt:* 2 vai trò — chủ quán toàn quyền; nhân viên xem và xử lý đơn, bật tắt món trong ngày, không sửa danh mục món và giá.
- **Đơn đặt bàn:** danh sách có lọc theo ngày, chi nhánh, trạng thái; trạng thái `chờ xác nhận` / `đã xác nhận` / `từ chối` / `huỷ`; đơn chờ xác nhận nổi bật; nút gọi số điện thoại khách nhanh. Có ô ghi chú nội bộ.
- **Menu hôm nay:** danh sách toàn bộ món, mỗi món có công tắc bật/tắt cho ngày đang xem.
- **Danh mục món (chủ quán):** thêm, sửa, ẩn món: tên, mô tả, giá, ảnh, nhóm, thứ tự, **quy tắc theo thứ trong tuần** (vd "Cơm dĩa" chỉ bật thứ 2–thứ 6; mặc định mọi ngày).

## 8. Menu theo ngày — quy tắc

- Mỗi món có quy tắc lặp theo thứ trong tuần (mặc định đủ 7 ngày).
- Trạng thái hiển thị của một món vào một ngày = quy tắc tuần, **bị ghi đè** bởi lựa chọn bật/tắt riêng của ngày đó (nếu có).
- Khách xem "hôm nay" theo giờ Việt Nam (UTC+7). Món dành cho buổi nào (sáng, trưa/tối) thể hiện bằng nhóm món, không tự ẩn theo giờ.
- **Menu là một danh mục chung cho cả quán** (người dùng chốt: hai chi nhánh bán cùng một menu). Công tắc bật/tắt món trong ngày áp dụng cho cả quán.
- Hai chi nhánh chỉ khác nhau ở **nhóm món/buổi bán trong ngày** (người dùng nói "khác nhóm bán trong ngày"; *cách hiểu của người viết, cần người dùng xác nhận*): mỗi chi nhánh có cấu hình riêng gồm nhóm món nào đang bán và khung giờ nào. Khách chọn chi nhánh khi đặt bàn thì thấy đúng nhóm món của chi nhánh đó; trang chủ mặc định hiện menu chung.
- Ngày nghỉ (lễ, sửa chữa) tính **riêng từng chi nhánh**: một cờ "nghỉ" theo ngày và chi nhánh. Ngày nghỉ thì form đặt bàn chặn ngày đó cho chi nhánh đó.

## 9. Thông báo đơn mới

- Khi có đơn mới, hệ thống gửi **email qua Gmail** cho quán (người nhận cấu hình được). Dùng tài khoản Gmail bật xác minh 2 bước và "mật khẩu ứng dụng", đặt trong biến môi trường phía server, **tuyệt đối không đưa vào code phía trình duyệt**. Gmail có giới hạn số thư mỗi ngày nhưng đủ cho quán nhỏ.
- Trang quản lý cũng hiện số đơn đang chờ xác nhận.
- **Chừa sẵn "cổng thông báo":** phần gửi thông báo được tách thành một giao diện riêng (nhận thông tin đơn, gửi qua 1 kênh) để sau này thêm Zalo mà không sửa chỗ khác. Hiện chỉ cài kênh email.
- Zalo: gửi tự động qua API **có phí** (nguồn tổng hợp ghi khoảng 165–200đ/tin; gói OA cần API có thể vài triệu đồng/năm; thông tin chưa kiểm chứng từ nguồn chính thức). Không làm ở bản đầu.

## 10. Bảo mật và dữ liệu

- Khách chỉ **gửi** đơn và **đọc** menu hôm nay; không đọc được đơn của người khác.
- Chỉ người đã đăng nhập mới xem và sửa đơn, menu; áp dụng phân quyền ở tầng cơ sở dữ liệu (row-level security), không chỉ ẩn nút trên giao diện.
- Có chống spam đơn ảo (giới hạn tần suất, trường ẩn hoặc tương đương) — *cách làm để người làm chọn*.
- Dữ liệu cá nhân của khách chỉ là tên, số điện thoại, ghi chú. Nên có dòng ghi chú ngắn nói quán dùng số điện thoại để xác nhận đặt bàn.
- Supabase miễn phí tự tạm dừng sau 1 tuần không hoạt động: cần một tác vụ định kỳ gọi nhẹ để giữ hoạt động, hoặc ghi rõ cách bật lại trong hướng dẫn.

## 11. Chi phí dự kiến
- Hosting trang web, cơ sở dữ liệu, đăng nhập: gói miễn phí.
- Gmail: miễn phí.
- Tên miền riêng (tuỳ chọn): khoảng vài trăm nghìn đồng/năm; nếu không mua thì dùng địa chỉ miễn phí do nhà cung cấp hosting cấp.

## 12. Để người làm quyết định
- Công nghệ phía trang web (HTML/JS thuần hay một framework nhẹ), miễn là chạy tĩnh và tải nhanh trên điện thoại.
- Chi tiết bảng dữ liệu, cách chống spam, cách giữ Supabase không bị tạm dừng.
- Cách xử lý ngày quán nghỉ, mức chi tiết của quy tắc tuần.
- Cách gửi Gmail từ phía server (hàm chạy trên Supabase hay dịch vụ tương đương).
- Hướng dẫn sử dụng ngắn cho chủ quán và nhân viên (ảnh chụp màn hình các thao tác chính).

## 13. Còn mở
- Giờ mở cửa chính xác của từng chi nhánh.
- Số người tối đa mỗi đơn, có giới hạn đặt trước bao nhiêu ngày không.
- Chủ quán cung cấp ảnh món và logo bản gốc.
- Ai tạo và giữ tài khoản Gmail dùng để gửi thông báo.

## 14. Quyết định của người viết (chưa được người dùng xác nhận từng điều)

Các điều dưới đây do người viết tự quyết sau khi người đọc độc lập chỉ ra chỗ phải đoán; người dùng cần xem lại.

- **Hosting:** Cloudflare Pages (trang tĩnh) kết hợp Supabase. Nếu cần một lớp server nhỏ thì dùng Cloudflare Workers. Biến môi trường bí mật đặt ở phía server, không ở trình duyệt.
- **Gửi Gmail:** người gửi là một tài khoản Gmail của quán. Người làm cần kiểm tra xem môi trường Supabase miễn phí có cho gửi qua SMTP không (nghi ngờ cổng SMTP bị chặn). Nếu bị chặn thì dùng Gmail API qua HTTPS (OAuth) hoặc cầu nối khác, nhưng người gửi vẫn là Gmail của quán. Nếu không dùng được Gmail thì chọn dịch vụ gửi email có gói miễn phí và báo lại người dùng trước khi đổi.
- **Không bỏ sót đơn:** trang quản lý tự làm mới khoảng 30 giây, có số đơn chờ trên tiêu đề tab, và đơn chờ quá 15 phút đổi sang màu cảnh báo. Không bắt buộc âm báo ở bản đầu.
- **Thông số đặt bàn (lưu trong bảng cấu hình, chủ quán sửa được trên một màn hình cài đặt đơn giản):** bước giờ 30 phút; giờ đặt cuối cùng cách giờ đóng cửa 1 giờ; đặt cùng ngày được nếu trước giờ đến ít nhất 1 giờ; đặt trước tối đa 14 ngày; tối đa 10 người mỗi đơn (nhiều hơn thì gọi quán). Giờ mở cửa mỗi chi nhánh nằm trong cùng bảng cấu hình, có thể nhiều ca trong ngày.
- **Tài khoản nhân viên:** không cho tự đăng ký. Chủ quán tạo hoặc mời nhân viên bằng email; chủ quán có một màn hình quản lý tài khoản đơn giản (thêm, khoá). Hai vai trò như mục 7.
- **Khách không tự huỷ hoặc sửa đơn:** muốn đổi thì gọi quán. Không cần mã đơn riêng cho khách. Nhân viên đặt trạng thái huỷ.
- **Thông báo cho khách:** không gửi email hay SMS tự động cho khách ở bản đầu; quán gọi hoặc nhắn lại bằng tay như đã chốt. Form không cần ô email của khách.
- **Dữ liệu ban đầu:** người làm nhập menu mẫu dựa trên thực đơn in của chi nhánh Phan Đăng Lưu (nhóm Sáng, Trưa/tối, Hấp luộc, Kho xào, Gỏi, Lẩu, Canh, Chiên, Đặc biệt) kèm giá, đánh dấu là "tạm". Chủ quán kiểm tra và sửa trước khi mở cho khách.
