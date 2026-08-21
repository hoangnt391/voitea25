# Kiến trúc source VoiTea

Tài liệu này đóng vai trò comment cấp project để người tiếp quản code có thể hiểu nhanh từng module mà không phải đọc toàn bộ file lớn trước.

## 1. Frontend — `src/App.tsx`

### App shell & phân quyền
- `App`: giữ trạng thái màn hình hiện tại, user đăng nhập, token, menu, topping và giỏ hàng.
- `Mode`: customer / orders / checkout / pos / staffQueue / bar / admin.
- `Role`: customer / order / cashier / barista / staff / admin.
- `go()`: kiểm tra quyền trước khi chuyển workspace.
- `login()` / `logout()`: đồng bộ session token với `localStorage`.

### Giỏ hàng
- `sizesOf()`: lấy danh sách size từ dữ liệu Admin; fallback M/L chỉ dùng cho dữ liệu cũ.
- `priceOf()`: lấy đúng giá theo size.
- `toppingSum()` và `lineTotal()`: tính topping + giá size theo số lượng.
- `cartConfigKey()`: tách riêng các dòng cùng món nhưng khác size/topping.
- `newRequestId()`: idempotency key để chống bấm đặt đơn hai lần.

### Khách hàng
- `Customer`: menu, tìm món, danh mục, giỏ hàng nổi.
- `ProductCard`: chọn size/topping ngay trên card.
- `Cart`: checkout, voucher, khách hàng, QR bàn và phương thức thanh toán.
- Bàn được mở bằng `?table=...`; khi có mã bàn hợp lệ chỉ cho phép QR/chuyển khoản.
- `OrderTracker`: theo dõi realtime trạng thái đơn và lịch sử hóa đơn.

### Thanh toán QR
1. Khách tạo đơn QR.
2. Khách chuyển khoản rồi bấm `Đã quét QR • Theo dõi đơn`.
3. Backend ghi `paymentCheckStatus=checking` và báo Order/Thu ngân.
4. Nhân viên kiểm tra tiền thực tế trong tài khoản.
5. Nếu chưa thấy tiền, bấm `Chưa nhận được tiền`; khách nhận realtime notification và có nút báo kiểm tra lại.
6. Chỉ khi nhân viên bấm `Đã nhận tiền • Duyệt đơn` thì đơn mới chuyển sang `waiting` cho pha chế.

### POS
- `Pos`: nhân viên Order/Thu ngân/Admin lên đơn cho khách.
- Đơn POS ghi `actor` là tài khoản trực tiếp lên đơn.
- Nhân viên thường chỉ được xác nhận thanh toán cho đơn POS do chính mình tạo; Admin được override.
- Hóa đơn mới sticky trên desktop/tablet và trở về normal-flow trên mobile.

### Duyệt đơn & bàn giao
- `StaffQueue`: danh sách FIFO chờ xác nhận thanh toán và chờ bàn giao.
- Hàng chờ được cập nhật realtime, không cần refresh thủ công.

### Pha chế
- `Bar`: waiting → making → done → ready → delivered.
- `claimedBy` được ghi khi nhân viên bấm `Nhận đơn / Bắt đầu pha` lần đầu, dùng để đối soát doanh thu/công việc pha chế.

### Admin
- `Admin`: dashboard và CRUD toàn hệ thống.
- Module: Tổng quan, Đơn hàng, Menu, Danh mục, Công thức nền, Công thức pha chế, Kho, Khách hàng, Nhân viên, Voucher, QR bàn, Ngân hàng, Cài đặt.
- Báo cáo nhân viên chỉ tính hai loại công việc: tài khoản trực tiếp lên đơn POS và tài khoản trực tiếp nhận pha.
- Trạng thái đóng/mở cửa không khóa quyền xem báo cáo Admin.

## 2. Backend — `backend/index.ts`

### Auth
- Password được băm bằng `scrypt` + salt riêng.
- `session()` kiểm tra token còn hạn.
- `ensureSuperAdmin()` bảo vệ Admin đầu tiên khỏi bị hạ quyền.

### Catalog
- `ensureMenuCatalog()`: seed menu/danh mục một lần cho dữ liệu ban đầu.
- `canonicalOrderItems()`: backend không tin giá từ client; luôn đọc lại sản phẩm/size/topping chuẩn trước khi tính đơn.

### Kho & công thức
- `ensureRecipeCatalog()`: seed nguyên liệu/công thức nền/công thức món ban đầu.
- `stockRequirements()`: gom nhu cầu thành phẩm nền + topping.
- `checkOrderStock()`: chặn đơn nếu thiếu hàng.
- `consumeOrderStock()` / `restoreOrderStock()`: trừ hoặc hoàn kho theo lifecycle đơn.

### Order lifecycle
- Tạo đơn: `payment_pending`.
- Xác nhận trả tiền: `waiting`.
- Pha chế: `making`.
- Pha xong: `done`.
- Chuyển bàn giao: `ready`.
- Giao khách: `delivered`.
- Chỉ cho hủy trước khi đã bắt đầu pha.

### Payment ownership
- Đơn `source=pos` có `actor`.
- Order/Cashier thường chỉ thao tác payment trên đơn do đúng `actor` tạo.
- Admin có toàn quyền override.
- Đơn web không có nhân viên lên đơn nên Order/Cashier/Admin được kiểm tra thanh toán theo quyền workspace.

### Voucher
- `voucherLive()` kiểm tra active, ngày hiệu lực và usage limit.
- `voucher()` tính percent/fixed discount và max discount.
- Voucher đã dùng được hoàn lượt nếu đơn bị hủy đúng flow.

### Doanh thu nhân viên
- Người lên đơn: lấy từ `order.actor` với `source=pos`, chỉ đơn đã thanh toán.
- Người nhận pha: lấy từ `order.claimedBy`, được ghi tại thời điểm nhận đơn pha chế.
- Xác nhận thanh toán, bấm pha xong và giao khách không tạo thêm doanh thu nhân viên.

## 3. Realtime

### `backend/realtime-subscribers.ts`
- Lưu mapping `entity_type + entity_id + connection_id`.
- Staff subscription yêu cầu session hợp lệ.
- Customer-order subscription yêu cầu `trackToken` riêng của đơn để tránh xem nhầm đơn người khác.
- `notifySubscribers()` gửi entity update tới tất cả websocket connection đang subscribe.

### `backend/realtime.ts`
- Khi websocket disconnect, xóa subscription cũ để tránh registry rác.

## 4. CSS — `src/index.css`
- Một stylesheet duy nhất được giữ nguyên theo snapshot đang chạy.
- Breakpoint mobile chính: `max-width: 780px`.
- Tablet/POS: `781px–1180px`.
- Desktop rộng: `>=1181px`.
- Các bảng Admin dài chỉ scroll trong container; không đẩy toàn bộ viewport rộng ra.
- Footer chỉ nằm trong `Customer`, nên Checkout/Order tracking không bị kéo xuống footer.

## 5. Tests

`tests/tests.txt` là checklist regression hiện tại. Trước khi sửa các flow quan trọng nên chạy lại các nhóm:
- Double submit / idempotency.
- Size + topping + voucher.
- Inventory guard.
- QR bàn chỉ chuyển khoản.
- Payment owner của POS.
- Realtime `not_received` cho khách.
- Pha chế/bàn giao.
- Dashboard doanh thu và doanh thu nhân viên.
- Responsive mobile/tablet/POS.

## 6. Lưu ý môi trường

Source snapshot này dùng runtime AppDeploy (`@appdeploy/client` và `@appdeploy/sdk`). Không commit token, mật khẩu thật, API key hoặc secret vào repo. Dữ liệu database/storage production không nằm trong Git source.