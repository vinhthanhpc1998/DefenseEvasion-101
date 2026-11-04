# Description
DLL Side-Loading là kỹ thuật lợi dụng một ứng dụng hợp pháp để nạp một DLL độc hại thay vì DLL chính thống mà ứng dụng mong đợi. Kẻ tấn công thường đặt DLL độc hại cùng thư mục với file EXE hợp pháp để ứng dụng ưu tiên nạp DLL này khi khởi chạy.

## Quy trình cơ bản:
1. Lựa chọn một ứng dụng hợp pháp có cơ chế nạp DLL phù hợp.
2. Tạo hoặc chuẩn bị DLL độc hại có tên trùng với DLL mà ứng dụng yêu cầu.
3. Đặt DLL cạnh file EXE hợp pháp.
4. Khởi chạy EXE để kích hoạt việc nạp và thực thi mã trong DLL.
## Mục đích thường gặp:
- Defense Evasion.
- Malware Execution.
- Persistence.
- Ngụy trang hoạt động độc hại dưới tiến trình hợp pháp.
  
# Troubleshoot

# Step to reproduce

# Reference
