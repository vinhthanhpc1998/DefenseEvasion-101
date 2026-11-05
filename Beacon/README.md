# Description
Beacon là một thành phần trong kiến trúc Command and Control (C2), hoạt động trên máy tính mục tiêu và duy trì liên lạc với C2 Server để nhận lệnh, gửi dữ liệu và thực hiện các tác vụ được điều khiển từ xa.

Các chức năng chính:
- C2 Communication: Thiết lập và duy trì kết nối với C2 Server thông qua HTTP/HTTPS, TCP hoặc các giao thức khác.
- Task Execution: Tiếp nhận và điều phối các tác vụ từ máy chủ.
- Data Collection: Thu thập thông tin hệ thống và kết quả thực thi tác vụ.
- Task Dispatcher: Phân phối tác vụ đến các module xử lý tương ứng.
- Session Management: Quản lý trạng thái kết nối, phiên làm việc và thông tin của Beacon.
Đặc điểm:
- Có thể hoạt động theo mô hình định kỳ (Callback/Check-in) hoặc duy trì kết nối liên tục.
- Có kiến trúc modular, cho phép mở rộng chức năng thông qua các handler.
- Có thể triển khai dưới dạng tiến trình độc lập hoặc thành phần tích hợp trong một ứng dụng khác.
