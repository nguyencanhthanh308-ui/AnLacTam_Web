# Web đặt bàn và menu theo ngày An Lạc Tâm — Kế hoạch triển khai

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Web cho quán chay An Lạc Tâm: khách xem menu hôm nay và gửi đơn đặt bàn; chủ quán và nhân viên đăng nhập để bật tắt món, xử lý đơn, nhận email báo đơn mới.

**Architecture:** Trang tĩnh (Vite + TypeScript thuần, hai trang: khách `/`, quản lý `/admin/`) gọi thẳng Supabase. **Mọi luật nghiệp vụ nằm trong Postgres** (hàm SQL + row-level security), nên chỉ có một nguồn sự thật và test được bằng pgTAP; giao diện chỉ hiển thị. Email gửi từ một Edge Function qua giao diện `Notifier` để sau này gắn thêm Zalo.

**Tech Stack:** Vite, TypeScript, Vitest (jsdom), Supabase (Postgres, Auth, Edge Functions Deno, pgTAP qua `supabase test db`), Cloudflare Pages, GitHub Actions (giữ Supabase không bị tạm dừng).

**Spec:** `docs/superpowers/specs/2026-11-03-dat-ban-an-lac-tam-design.md` (mục 14 là các quyết định của người viết, người dùng đã duyệt cả bản).

## Global Constraints

- Giao diện tiếng Việt; mọi "hôm nay" và "ngày" tính theo múi giờ `Asia/Ho_Chi_Minh` (UTC+7), không theo UTC.
- Màu: nền `#F7BF4F` (đậm `#EFA114`), chữ/logo/nút `#6A0D05`, nét cọ nhãn `#4E9625`, xanh lá sen `#1B6838`, xanh hoa sen `#9CC0DB`, dải họa tiết đáy `#602E2F`. Không dùng tông đỏ mận + kem. Font: Anton (logo, giá), Dancing Script (chữ viết tay), Be Vietnam Pro (nội dung). Mẫu: `docs/mockups/customer-home-A-v2.html`.
- Bố cục trang chủ A: menu hôm nay trên cùng, nút "Đặt bàn" cố định ở đáy màn hình.
- Trạng thái đơn: `pending` (chờ xác nhận), `confirmed` (đã xác nhận), `rejected` (từ chối), `cancelled` (huỷ). Số điện thoại bắt buộc. Khách không cần tài khoản, không tự huỷ hay sửa đơn.
- Thông số đặt bàn (bảng `booking_settings`, chủ quán sửa được): bước giờ 30 phút; giờ cuối cách giờ đóng cửa 60 phút; đặt cùng ngày nếu trước giờ đến ít nhất 60 phút; đặt trước tối đa 14 ngày; tối đa 10 người mỗi đơn.
- Trang quản lý: tự làm mới 30 giây; số đơn chờ trên tiêu đề tab; đơn chờ quá 15 phút đổi màu cảnh báo.
- Hai vai trò: `owner` toàn quyền; `staff` xem và xử lý đơn, bật tắt món trong ngày, không sửa danh mục món, giá, cấu hình, tài khoản. Không cho tự đăng ký.
- Menu là một danh mục chung cho cả quán; giờ mở cửa, nhóm món bán và ngày nghỉ cấu hình riêng từng chi nhánh.
- Chi phí cố định 0đ và không cần thẻ; bí mật (mật khẩu Gmail, service key) chỉ ở biến môi trường phía server, không đưa vào code trình duyệt hay git.
- Chi nhánh: 117 Phan Đăng Lưu; 41 Lê Quý Đôn (Đà Nẵng).

## Review Focus

Các tình huống spec ngầm đòi nhưng không nêu rõ, sắp theo khả năng gây hại; mỗi dòng có test ở task ghi trong ngoặc.

1. Khách đặt lúc 00:30 giờ VN (17:30 UTC hôm trước): "hôm nay" phải là ngày VN, không lệch một ngày (Task 3, 4).
2. Giờ rơi vào khoảng nghỉ giữa hai ca (14:00–16:00) hoặc ngày chi nhánh đóng cửa: bị từ chối, không hiện trong danh sách giờ (Task 4).
3. Số điện thoại viết nhiều kiểu (`0905 123 456`, `+84905123456`) và bị gửi lặp: chuẩn hoá về một dạng, quá 3 đơn chờ cùng số thì từ chối, ô ẩn (honeypot) có giá trị thì bỏ qua đơn (Task 4).
4. Gửi email lỗi (Gmail từ chối, hết hạn mật khẩu): đơn **vẫn được lưu**, và trang quản lý hiện cảnh báo "email chưa gửi" cùng nút gửi lại (Task 8, 9).
5. Hai nhân viên cùng bấm xử lý một đơn: người đến sau thấy "đơn đã được xử lý" thay vì ghi đè im lặng (Task 9).

---

## Cấu trúc file

```
index.html                          trang khách
admin/index.html                    trang quản lý
src/styles/theme.css                biến màu, font, nét cọ, khung nét đứt
src/lib/supabase.ts                 client Supabase (đọc VITE_SUPABASE_URL, VITE_SUPABASE_ANON_KEY)
src/lib/format.ts                   định dạng giá, thứ, ngày giờ VN
src/customer/{main,menu-view,booking-form}.ts
src/admin/{main,auth,reservations,menu-admin,settings,accounts}.ts
supabase/migrations/000N_*.sql      schema, hàm nghiệp vụ, RLS
supabase/seed.sql                   chi nhánh, nhóm món, menu mẫu "tạm"
supabase/tests/*.sql                pgTAP
supabase/functions/notify-reservation/{index,notifier,email}.ts
.github/workflows/keepalive.yml
docs/staff-guide.md
```

---

### Task 1: Khung dự án và giao diện nền

**Files:** Create `package.json`, `vite.config.ts` (hai điểm vào: `index.html`, `admin/index.html`), `tsconfig.json`, `src/styles/theme.css`, `src/lib/format.ts`, `src/lib/format.test.ts`, `supabase/config.toml` (qua `supabase init`), `.env.example`.

**Interfaces:**
- Produces: `formatPrice(vnd: number): string` ("20000" → `"20K"`, 40000 → `"40K"`, 15500 → `"15.500đ"`); `weekdayLabel(isoDate: string): string` ("2026-11-03" → `"Thứ Ba"`, Chủ nhật → `"Chủ nhật"`); `vnToday(now?: Date): string` trả `YYYY-MM-DD` theo UTC+7. CSS lớp dùng chung: `.brush` (nhãn nét cọ), `.box` (khung nét đứt), `.btn`, `.chip`, `.band`.

- [ ] **Step 1:** Viết `format.test.ts` với các ca trên, thêm `vnToday(new Date("2026-11-02T17:30:00Z"))` phải ra `"2026-11-03"` và `vnToday(new Date("2026-11-02T16:59:00Z"))` ra `"2026-11-02"`.
- [ ] **Step 2:** `npx vitest run` → FAIL (chưa có hàm).
- [ ] **Step 3:** Cài Vite + TypeScript + Vitest + jsdom, cài `@supabase/supabase-js`; cài `format.ts` (tính giờ VN bằng cộng 7 giờ, không dựa vào múi giờ máy); chép biến màu và lớp CSS từ bản mẫu `customer-home-A-v2.html` vào `theme.css`.
- [ ] **Step 4:** `npx vitest run` → PASS; `npm run build` → tạo được `dist/index.html` và `dist/admin/index.html`.
- [ ] **Step 5:** Commit `feat: khung dự án, theme và định dạng`.

### Task 2: Kiểm chứng đường gửi Gmail (spike, quyết định kiến trúc thông báo)

**Files:** Create `supabase/functions/notify-reservation/notifier.ts`, `email.ts`, `index.ts` (bản thử), `docs/notes/gmail-spike.md`.

**Interfaces:**
- Produces: `interface ReservationNotice { id: string; branchName: string; date: string; time: string; partySize: number; name: string; phone: string; note: string | null }`; `interface Notifier { send(n: ReservationNotice): Promise<void> }` (ném lỗi nếu gửi thất bại).

- [ ] **Step 1:** Người dùng (không phải agent) tạo tài khoản Gmail của quán, bật xác minh 2 bước, tạo mật khẩu ứng dụng; đặt `GMAIL_USER`, `GMAIL_APP_PASSWORD`, `NOTIFY_TO` làm secret của Supabase (`supabase secrets set`), không đưa vào git.
- [ ] **Step 2:** Viết `email.ts` gửi qua SMTP Gmail từ một Edge Function thử, triển khai lên dự án Supabase miễn phí và gọi thử.
- [ ] **Step 3:** Nếu SMTP bị chặn hoặc lỗi kết nối, chuyển sang Gmail API qua HTTPS (OAuth refresh token) và thử lại. Nếu cả hai đều không chạy, dừng và báo người dùng đề xuất dịch vụ gửi email miễn phí khác trước khi đi tiếp.
- [ ] **Step 4:** Ghi kết quả (đường nào chạy, giới hạn gửi mỗi ngày quan sát được) vào `docs/notes/gmail-spike.md`; email thử phải thực sự đến hộp thư `NOTIFY_TO`.
- [ ] **Step 5:** Commit `chore: kiểm chứng đường gửi Gmail`.

### Task 3: Schema, phân quyền và hàm menu theo ngày

**Files:** Create `supabase/migrations/0001_schema.sql`, `0002_menu.sql`, `supabase/seed.sql`, `supabase/tests/001_rls.sql`, `supabase/tests/002_menu.sql`.

**Interfaces:**
- Produces bảng (tên cột tiếng Anh): `branches(id uuid, name, address, phone)`, `branch_hours(branch_id, weekday smallint 0=CN..6, opens time, closes time)` (nhiều dòng một ngày = nhiều ca), `branch_closed_days(branch_id, day date, reason)`, `menu_groups(id, name, sort)`, `branch_groups(branch_id, group_id, from_time, to_time)`, `menu_items(id, group_id, name, description, price_vnd int, image_url, sort, is_active bool, weekdays smallint[] default '{0,1,2,3,4,5,6}')`, `menu_day_overrides(item_id, day date, available bool)`, `booking_settings(step_minutes, last_before_close_minutes, same_day_lead_minutes, max_days_ahead, max_party)` (đúng 1 dòng), `reservations`, `profiles(user_id, role 'owner'|'staff', display_name, active bool)`.
- Produces: `menu_for_date(p_date date, p_branch uuid default null) returns table(group_id uuid, group_name text, item_id uuid, name text, description text, price_vnd int, image_url text, available bool)` — `available` = (`weekday(p_date)` thuộc `weekdays` và `is_active`), bị **ghi đè** bởi `menu_day_overrides` nếu có dòng cho ngày đó; chỉ trả nhóm có bán tại `p_branch` nếu truyền.
- Produces: `is_staff() returns bool`, `is_owner() returns bool` (dựa trên `profiles` và `auth.uid()`, chỉ tính `active`).

- [ ] **Step 1:** Viết `002_menu.sql` (pgTAP): món có `weekdays='{1,2,3,4,5}'` trả `available=true` cho thứ Ba 2026-11-03 và `false` cho Chủ nhật 2026-11-08; thêm override `available=true` cho Chủ nhật đó thì thành `true`; override `false` cho thứ Ba thì thành `false`; truyền `p_branch` của chi nhánh không bán nhóm Sáng thì không có dòng nhóm Sáng.
- [ ] **Step 2:** Viết `001_rls.sql` (pgTAP): vai `anon` không `select`/`insert` trực tiếp `reservations`, không `update` `menu_items`; `anon` gọi được `menu_for_date`; người dùng `staff` `update` được `menu_day_overrides` nhưng `update menu_items.price_vnd` bị từ chối; `owner` được cả hai; `profiles.active=false` bị từ chối mọi thao tác.
- [ ] **Step 3:** `supabase test db` → FAIL.
- [ ] **Step 4:** Viết các migration (bật RLS mọi bảng; chính sách theo `is_staff()`/`is_owner()`; `menu_for_date` là `security definer` chỉ trả cột công khai); `seed.sql` nhập hai chi nhánh, nhóm món và menu mẫu từ thực đơn in Phan Đăng Lưu (Sáng, Trưa/tối, Hấp luộc, Kho xào, Gỏi, Lẩu, Canh, Chiên, Đặc biệt, kèm giá), mô tả món đánh dấu `(tạm)`; giờ mở cửa mẫu 6:30–14:00 và 16:00–20:00 mọi ngày.
- [ ] **Step 5:** `supabase test db` → PASS; commit `feat: schema, RLS, menu theo ngày`.

### Task 4: Luật đặt bàn trong Postgres

**Files:** Create `supabase/migrations/0003_booking.sql`, `supabase/tests/003_booking.sql`.

**Interfaces:**
- Consumes: bảng và `booking_settings` từ Task 3.
- Produces: `available_slots(p_branch uuid, p_date date) returns setof time` — các mốc cách nhau `step_minutes` trong từng ca của ngày đó, bỏ mốc muộn hơn `closes - last_before_close_minutes`, bỏ mốc trước `now() at time zone 'Asia/Ho_Chi_Minh' + same_day_lead_minutes` nếu là hôm nay, rỗng nếu ngày quá `max_days_ahead`, ngày đã qua, hoặc có trong `branch_closed_days`.
- Produces: `create_reservation(p_branch uuid, p_date date, p_time time, p_party int, p_name text, p_phone text, p_note text, p_hp text) returns uuid` — chuẩn hoá điện thoại (bỏ khoảng trắng và dấu chấm, `+84` → `0`), kiểm tra giờ thuộc `available_slots`, `1 ≤ p_party ≤ max_party`, tên không rỗng, điện thoại 10 chữ số bắt đầu bằng `0`; `p_hp` khác rỗng thì trả về một uuid giả **không ghi gì** (giữ bot không biết); từ chối nếu số này đã có từ 3 đơn `pending`. Thành công ghi `reservations` với `status='pending'`.

- [ ] **Step 1:** Viết test pgTAP cho từng luật, đặt giờ hiện tại giả bằng tham số hệ thống hoặc hàm `now()` bị thay (dùng `set local` hoặc bảng cấu hình giờ giả chỉ trong test): giờ `14:30` thứ Ba bị từ chối (khoảng nghỉ); `19:30` bị từ chối còn `19:00` được (đóng 20:00, cách 60 phút); `p_party=11` bị từ chối; ngày thứ 15 bị từ chối; ngày trong `branch_closed_days` rỗng slot; `+84905123456` và `0905 123 456` lưu cùng `0905123456`; đơn thứ 4 cùng số bị từ chối; `p_hp='x'` không tạo dòng nào; lúc `2026-11-02 17:30 UTC`, "hôm nay" là `2026-11-03` và slot trước giờ + 60 phút bị bỏ.
- [ ] **Step 2:** `supabase test db` → FAIL.
- [ ] **Step 3:** Viết `0003_booking.sql` (`security definer`, `set search_path = public`); phép tính "bây giờ" bằng `now() at time zone 'Asia/Ho_Chi_Minh'`, đọc qua một hàm `app_now()` để test thay được.
- [ ] **Step 4:** `supabase test db` → PASS.
- [ ] **Step 5:** Commit `feat: luật đặt bàn trong Postgres`.

### Task 5: Trang khách — Menu hôm nay (bố cục A)

**Files:** Create `index.html`, `src/customer/main.ts`, `src/customer/menu-view.ts`, `src/customer/menu-view.test.ts`, `src/lib/supabase.ts`.

**Interfaces:**
- Consumes: `menu_for_date` (Task 3), `formatPrice`, `weekdayLabel`, `vnToday` (Task 1).
- Produces: `renderMenu(root: HTMLElement, rows: MenuRow[], activeGroupId: string | null): void`, `type MenuRow = { group_id; group_name; item_id; name; description; price_vnd; image_url; available }`.

- [ ] **Step 1:** Viết `menu-view.test.ts` (jsdom): với 3 dòng thuộc 2 nhóm, `renderMenu` tạo đúng 2 nhãn nhóm, món `available=false` có lớp `.sold-out` và chữ "Hôm nay hết món", giá hiện `20K`; bấm chip nhóm lọc chỉ còn nhóm đó.
- [ ] **Step 2:** `npx vitest run` → FAIL.
- [ ] **Step 3:** Cài `menu-view.ts` và `main.ts`: tiêu đề "Thực đơn hôm nay · {weekdayLabel(vnToday())}", chip nhóm, nhãn `.brush` và khung `.box` cho từng nhóm, nút "Đặt bàn" cố định ở đáy (`position: sticky`/`fixed`); ảnh thiếu thì dùng ô tròn giữ chỗ viền `#1B6838`; có trạng thái đang tải và thông báo lỗi tiếng Việt khi mất mạng.
- [ ] **Step 4:** `npx vitest run` → PASS; mở `npm run dev`, so bằng mắt với `docs/mockups/customer-home-A-v2.html` trên màn hình 375px: không cuộn ngang, nút Đặt bàn không che món cuối.
- [ ] **Step 5:** Commit `feat: trang khách hiển thị menu hôm nay`.

### Task 6: Trang khách — Form đặt bàn

**Files:** Create `src/customer/booking-form.ts`, `src/customer/booking-form.test.ts`; Modify `index.html`, `src/customer/main.ts`.

**Interfaces:**
- Consumes: `available_slots`, `create_reservation` (Task 4).
- Produces: `openBookingForm(root: HTMLElement, branches: Branch[]): void`; `validateForm(v: FormValues): Record<string,string>` (khoá = tên trường, giá trị = lỗi tiếng Việt, rỗng nếu hợp lệ).

- [ ] **Step 1:** Viết test cho `validateForm`: thiếu tên → `"Vui lòng nhập họ tên"`, số điện thoại `0905` → `"Số điện thoại chưa đúng"`, số người 0 hoặc 11 → lỗi, hợp lệ → `{}`.
- [ ] **Step 2:** `npx vitest run` → FAIL.
- [ ] **Step 3:** Cài form: chọn chi nhánh → chọn ngày (14 ngày tới, tính từ `vnToday()`) → danh sách giờ lấy từ `available_slots` (rỗng thì hiện "Chi nhánh không nhận đặt ngày này"), số người, tên, điện thoại, ghi chú, ô ẩn `p_hp`. Thành công hiện "Đơn đang chờ quán xác nhận, quán sẽ gọi hoặc nhắn lại cho bạn" kèm tóm tắt đơn; lỗi từ server (vd quá số đơn chờ) hiện tiếng Việt, không lộ thông báo kỹ thuật.
- [ ] **Step 4:** `npx vitest run` → PASS; thử tay trên dự án Supabase: gửi một đơn thật, kiểm tra dòng `pending` có trong bảng.
- [ ] **Step 5:** Commit `feat: form đặt bàn cho khách`.

### Task 7: Quản lý — đăng nhập và tài khoản

**Files:** Create `admin/index.html`, `src/admin/main.ts`, `src/admin/auth.ts`, `src/admin/accounts.ts`, `supabase/migrations/0004_accounts.sql`, `supabase/functions/invite-staff/index.ts`.

**Interfaces:**
- Produces: `getSession(): Promise<{ userId: string; role: 'owner'|'staff' } | null>`; `requireRole(role: 'owner'|'staff'): Promise<Session>` (chuyển về màn hình đăng nhập nếu chưa đăng nhập hoặc `active=false`).
- Produces: Edge Function `invite-staff` nhận `{email, display_name, role}` từ `owner` và dùng service key phía server để mời; từ chối nếu người gọi không phải `owner`.

- [ ] **Step 1:** Viết test pgTAP thêm vào `001_rls.sql`: `staff` không `insert/update` được `profiles`; `owner` khoá được tài khoản (`active=false`) và tài khoản khoá không gọi được hàm nào.
- [ ] **Step 2:** `supabase test db` → FAIL.
- [ ] **Step 3:** Cài đăng nhập email + mật khẩu, màn hình đăng xuất, và màn "Tài khoản" cho owner (mời, khoá, mở khoá). Tài khoản owner đầu tiên tạo bằng một câu lệnh SQL ghi trong `docs/staff-guide.md`.
- [ ] **Step 4:** `supabase test db` → PASS; thử tay: nhân viên đăng nhập không thấy mục Tài khoản và không gọi được `invite-staff`.
- [ ] **Step 5:** Commit `feat: đăng nhập và quản lý tài khoản`.

### Task 8: Gửi email thông báo đơn mới và nhật ký gửi

**Files:** Create `supabase/migrations/0005_notifications.sql`; Modify `supabase/functions/notify-reservation/index.ts`; Test `supabase/functions/notify-reservation/index.test.ts`.

**Interfaces:**
- Consumes: `Notifier`, `ReservationNotice` (Task 2).
- Produces: bảng `notification_log(reservation_id, channel text, status 'sent'|'failed', error text, attempts int, updated_at)`; Database Webhook khi chèn vào `reservations` gọi `notify-reservation`; hàm `dispatch(notice, notifiers: Notifier[], log: LogWriter)` — chạy từng notifier, **không bao giờ ném lỗi ra ngoài**, ghi `failed` kèm lỗi nếu gửi thất bại.

- [ ] **Step 1:** Viết test Deno cho `dispatch` với `Notifier` giả: gửi thành công ghi `sent`; notifier ném lỗi ghi `failed` và các notifier khác vẫn chạy; hai notifier (giả lập "email" và "zalo") đều được gọi.
- [ ] **Step 2:** `deno test` → FAIL.
- [ ] **Step 3:** Cài `dispatch`, nối `EmailNotifier` (đường đã chọn ở Task 2), cấu hình Database Webhook, và hàm `retry_notification(p_reservation uuid)` cho nhân viên gọi để gửi lại.
- [ ] **Step 4:** `deno test` → PASS; tạo một đơn thật ở Task 6 và kiểm tra email đến hộp thư `NOTIFY_TO` với đủ chi nhánh, ngày giờ, số người, tên, điện thoại, ghi chú; đặt sai mật khẩu ứng dụng, tạo đơn mới, đơn vẫn lưu và `notification_log` ghi `failed`.
- [ ] **Step 5:** Commit `feat: gửi email thông báo đơn mới`.

### Task 9: Quản lý — danh sách đơn đặt bàn

**Files:** Create `src/admin/reservations.ts`, `src/admin/reservations.test.ts`, `supabase/migrations/0006_handle_reservation.sql`, `supabase/tests/004_handle.sql`.

**Interfaces:**
- Produces: `handle_reservation(p_id uuid, p_new_status text, p_staff_note text) returns void` — chỉ chuyển từ `pending` sang `confirmed`/`rejected`, hoặc `confirmed` sang `cancelled`; nếu đơn không còn ở trạng thái mong đợi ném lỗi `already_handled`; ghi `handled_by`, `handled_at`.
- Produces: `isOverdue(createdAt: string, now: Date): boolean` (true khi chờ quá 15 phút); `pendingTitle(count: number): string` (`"(3) Đơn đặt bàn"`, 0 → `"Đơn đặt bàn"`).

- [ ] **Step 1:** Viết pgTAP: hai lần liên tiếp `handle_reservation` cùng một đơn, lần hai ném `already_handled`; `anon` gọi bị từ chối; viết test Vitest cho `isOverdue` (14 phút → false, 16 phút → true) và `pendingTitle`.
- [ ] **Step 2:** `supabase test db` và `npx vitest run` → FAIL.
- [ ] **Step 3:** Cài danh sách đơn (lọc ngày, chi nhánh, trạng thái; đơn chờ nổi bật; nút gọi `tel:`; ô ghi chú nội bộ), tự làm mới 30 giây, đổi `document.title`, hiện cảnh báo "email chưa gửi" với nút "Gửi lại" khi `notification_log.status='failed'`; khi gặp `already_handled` hiện "Đơn đã được người khác xử lý" rồi tải lại danh sách.
- [ ] **Step 4:** Cả hai bộ test → PASS; thử tay bằng hai trình duyệt, hai tài khoản cùng bấm xác nhận một đơn.
- [ ] **Step 5:** Commit `feat: danh sách và xử lý đơn đặt bàn`.

### Task 10: Quản lý — menu hôm nay và danh mục món

**Files:** Create `src/admin/menu-admin.ts`, `src/admin/menu-admin.test.ts`.

**Interfaces:**
- Produces: `setDayOverride(itemId: string, day: string, available: boolean | null): Promise<void>` (null xoá override); `saveItem(item: ItemInput): Promise<void>` (owner); `toggleLabel(available: boolean, rule: boolean): string` mô tả "Bật theo ngày" / "Tắt hôm nay (ghi đè)" để nhân viên hiểu công tắc đang ghi đè quy tắc tuần.

- [ ] **Step 1:** Viết test cho `toggleLabel` ba trường hợp (theo quy tắc, bật ghi đè, tắt ghi đè) và test pgTAP trong `001_rls.sql` rằng `staff` chỉ ghi được `menu_day_overrides`.
- [ ] **Step 2:** Chạy cả hai bộ test → FAIL.
- [ ] **Step 3:** Cài màn "Menu hôm nay" (mọi nhân viên: công tắc theo ngày đang xem) và màn "Danh mục món" (chỉ owner: thêm/sửa/ẩn món, giá, nhóm, thứ tự, ảnh qua Supabase Storage, quy tắc thứ trong tuần dạng 7 ô tick).
- [ ] **Step 4:** Cả hai bộ test → PASS; thử tay: tắt một món rồi tải trang khách, món hiện mờ "Hôm nay hết món".
- [ ] **Step 5:** Commit `feat: quản lý menu hôm nay và danh mục món`.

### Task 11: Quản lý — cài đặt chi nhánh và thông số đặt bàn

**Files:** Create `src/admin/settings.ts`, `supabase/tests/005_settings.sql`.

**Interfaces:**
- Produces: `saveBookingSettings(s: BookingSettings): Promise<void>`, `saveBranchHours(branchId: string, rows: HourRow[]): Promise<void>`, `closeBranchDay(branchId: string, day: string, reason: string): Promise<void>`, `saveBranchGroups(branchId: string, groups: { groupId: string; from?: string; to?: string }[]): Promise<void>` — tất cả chỉ `owner`.

- [ ] **Step 1:** Viết pgTAP: đổi `max_party` thành 6 thì `create_reservation` với 7 người bị từ chối; thêm ngày nghỉ cho một chi nhánh thì `available_slots` rỗng chỉ cho chi nhánh đó; `staff` không sửa được bốn bảng này.
- [ ] **Step 2:** `supabase test db` → FAIL.
- [ ] **Step 3:** Cài màn "Cài đặt" (chỉ owner) cho thông số, giờ mở cửa nhiều ca, ngày nghỉ, nhóm món bán của từng chi nhánh.
- [ ] **Step 4:** `supabase test db` → PASS; thử tay: đóng cửa một chi nhánh ngày mai rồi mở form khách, ngày đó không còn giờ để chọn.
- [ ] **Step 5:** Commit `feat: cài đặt chi nhánh và thông số đặt bàn`.

### Task 12: Triển khai, giữ Supabase hoạt động và tài liệu bàn giao

**Files:** Create `.github/workflows/keepalive.yml`, `supabase/migrations/0007_ping.sql`, `docs/staff-guide.md`.

**Interfaces:**
- Produces: hàm `ping() returns text` (`anon` gọi được, trả `'ok'`); workflow chạy mỗi 3 ngày gọi `ping()` bằng `curl` tới URL Supabase lấy từ secret của GitHub.

- [ ] **Step 1:** Viết test pgTAP `ping()` trả `'ok'` cho `anon`.
- [ ] **Step 2:** `supabase test db` → FAIL, cài hàm, chạy lại → PASS.
- [ ] **Step 3:** Thêm workflow `keepalive.yml` (cron `0 3 */3 * *`, cho phép chạy tay); người dùng thêm secret `SUPABASE_URL` và `SUPABASE_ANON_KEY` vào GitHub; chạy tay một lần và kiểm tra thành công.
- [ ] **Step 4:** Kết nối repo với Cloudflare Pages (lệnh build `npm run build`, thư mục ra `dist`, hai biến `VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY`); kiểm tra trang khách và `/admin/` chạy trên địa chỉ công khai và đăng nhập được.
- [ ] **Step 5:** Viết `docs/staff-guide.md` tiếng Việt, có ảnh chụp màn hình: đăng nhập, xử lý đơn, bật tắt món hôm nay, sửa menu, ngày nghỉ, cách bật lại dự án Supabase nếu bị tạm dừng, cách đổi mật khẩu ứng dụng Gmail; commit `docs: hướng dẫn cho quán và triển khai`.

---

## Tự kiểm tra so với spec

- Mục 6 (khách): Task 5, 6. Mục 7 (quản lý): Task 7, 9, 10, 11. Mục 8 (menu theo ngày, chi nhánh, ngày nghỉ): Task 3, 4, 10, 11. Mục 9 (thông báo, cổng Zalo): Task 2, 8. Mục 10 (bảo mật, chống spam, giữ hoạt động): Task 3, 4, 12. Mục 11 (chi phí): Task 12. Mục 14 (quyết định người viết): Task 2, 4, 7, 9, 11, seed ở Task 3.
- Bốn điều còn mở ở mục 13 (giờ chính xác từng chi nhánh, giới hạn đặt trước, ảnh/logo gốc, người giữ tài khoản Gmail) không chặn code: giờ mẫu và thông số mặc định nằm trong cấu hình, ảnh thiếu dùng ô giữ chỗ, Gmail là bước thủ công của người dùng ở Task 2.
- Rủi ro lớn nhất là Task 2; nếu cả SMTP và Gmail API đều không chạy trên gói miễn phí thì dừng và hỏi lại trước khi làm Task 8.
- Zalo: Task 8 chỉ cài kênh email; thêm Zalo sau là thêm một `Notifier` thứ hai vào danh sách `dispatch`, không đổi code khác.
