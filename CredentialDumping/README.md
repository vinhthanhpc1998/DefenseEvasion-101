# Description 
Mimikatz là công cụ mã nguồn mở dùng để khai thác và phân tích cơ chế xác thực của Windows, nổi bật với khả năng trích xuất thông tin đăng nhập từ LSASS, SAM và tương tác với Kerberos.

Các vấn đề của Mimikatz:
- Dễ bị Antivirus phát hiện thông qua chữ ký (Signature-based Detection).
- Bị EDR giám sát hành vi truy cập LSASS và Credential Dumping.
- Một số chức năng yêu cầu quyền Administrator hoặc SYSTEM.
- Bị hạn chế bởi các cơ chế bảo vệ như Credential Guard và LSA Protection.
- Khả năng tương thích có thể bị ảnh hưởng bởi các bản cập nhật Windows.

# Troubleshoot
Dump Credential của người dùng trên máy, thường sẽ dump từ lsass.exe và có rất nhiều công cụ phổ biến và cách làm khác nhau, chưa chắc đã bắt buộc phải cần Minikaiz.
## Giả thuyết
- Mimikaiz là một công cụ đã quá phổ biến, vậy nên chắc chắn sẽ có Signature trên các hệ thống AV, vậy nên nếu tải thẳng Mimikatz lên máy nạn nhân thì Mimikaiz sẽ bị bắt.
- Nếu sử dụng công cụ khác có được không?
  + Lsass.exe khi chạy là 1 process đặc trưng, ít có ai tác động đến nó, nếu tác động vào nó và đặc biệt là dump process đó ra thì đó sẽ là hành động lạ và AV/EDR sẽ bắt ngay.
  + AV chỉ bắt file data đã được dump ở dưới ổ đĩa.

## Time Line 
- Khám phá công cụ Mimikatz (Tôi nghiên cứu thêm về các tính năng của Mimikatz, những tính năng nằm ngoài việc dump process lsass.exe).
- Thử dump bằng các cách khác (Tôi dump bằng task manager và procdump, nhưng vẫn bị AV bắt).
- Quan sát hành động của AV (AV nhận file dmp được dump ra từ process chạy lsass.exe là Malware).
- Viết 1 công cụ dump khác.
- Dump được dữ liệu thì liền transfer dữ liệu đó về máy Attacker để phân tích.

## Kết luận 
- Dữ liệu được dump ra sẽ chứa credential của người dùng, nên AV khi đọc được phần này (Signature) sẽ liền xóa đi luôn.
- Trong trường hợp file dmp bị mã hóa trước khi rơi xuống ổ đĩa thì sao.

# Step to reproduce 


# Reference
