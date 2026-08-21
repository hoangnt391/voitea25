# VoiTea 🐘

Source snapshot của hệ thống bán hàng VoiTea.

## Chức năng chính

- Website đặt món cho khách, giỏ hàng, size, topping, voucher và theo dõi đơn.
- POS cho nhân viên Order / Thu ngân.
- Quy trình xác nhận thanh toán và kiểm tra chuyển khoản QR.
- Khu pha chế và bàn giao đơn realtime.
- Dashboard Admin, doanh thu, doanh thu theo nhân viên, menu, danh mục, kho, công thức, voucher, QR bàn và tài khoản ngân hàng.
- Phân quyền Customer / Order / Cashier / Barista / Admin.
- Responsive cho desktop, tablet, POS và mobile.

## Cấu trúc

- `src/App.tsx`: giao diện và luồng nghiệp vụ phía client.
- `src/index.css`: toàn bộ style và responsive.
- `backend/index.ts`: API, auth, order, payment, inventory, recipe, voucher, admin.
- `backend/realtime-subscribers.ts`: quản lý subscription realtime.
- `backend/realtime.ts`: cleanup kết nối realtime.
- `tests/tests.txt`: checklist QA/regression của project.

## Lưu ý môi trường

Project đang sử dụng `@appdeploy/client` ở frontend và `@appdeploy/sdk` ở backend. Hai package/runtime này được cung cấp bởi môi trường AppDeploy khi build/deploy. Source trong repo là snapshot để quản lý phiên bản và tiếp tục phát triển.

## Bảo mật

Không commit mật khẩu, token đăng nhập, API key hoặc secret vào repository. Mật khẩu người dùng trong backend được hash bằng `scrypt` và salt riêng.

## Source snapshot

Snapshot được đồng bộ từ bản VoiTea đang chạy ngày 21/08/2026.
