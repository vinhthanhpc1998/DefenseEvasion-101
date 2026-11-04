# Description 
Mimikatz là công cụ mã nguồn mở dùng để khai thác và phân tích cơ chế xác thực của Windows, nổi bật với khả năng trích xuất thông tin đăng nhập từ LSASS, SAM và tương tác với Kerberos.

Các vấn đề của Mimikatz:
- Dễ bị Antivirus phát hiện thông qua chữ ký (Signature-based Detection).
- Bị EDR giám sát hành vi truy cập LSASS và Credential Dumping.
- Một số chức năng yêu cầu quyền Administrator hoặc SYSTEM.
- Bị hạn chế bởi các cơ chế bảo vệ như Credential Guard và LSA Protection.
- Khả năng tương thích có thể bị ảnh hưởng bởi các bản cập nhật Windows.
