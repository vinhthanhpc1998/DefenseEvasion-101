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
DLL Hijacking là 1 kĩ thuật liên quan mà tôi đã học được trước đó. DLL Side-loading là 1 trường hợp đặc biệt của DLL hijacking. Vậy nên trước khi tìm hiểu về DLL Side-loading thì tôi đã xem lại kiến thức về DLL hijacking trước. 
## Giả thuyết 
DLL Side-loading là 1 trường hợp của DLL Hijacking, với cùng 1 cơ chế hoạt động là nạp DLL vào 1 chương trình có sẵn trên Windows. Bản chất DLL là thư viện được nạp vào các chương trình, vậy nên trong thư viện sẽ có các hàm được chương trình chính đem ra sử dụng, vậy nên sẽ có khả năng bị Crash nếu DLL bị nạp sai. Có 2 phương án để áp dụng: 1 là fake file DLL, 2 là tìm nơi nạp mà không bị crash. Sau đó tôi đã chọn fake file DLL, vì tìm nơi nạp DLL mà không bị crash với 1 chương trình lớn là rất khó, thậm chí là không tìm được. Về fake DLL, tôi đã đặt ra nhiều giả thuyết và tìm nhiều cách khác nhau, rồi tôi thấy DLL Proxy là khả thi nhất. Ta sẽ tạo 1 file DLL trung gian có chứa mã độc, rồi nạp các hàm của DLL gốc vào file DLL trung gian. Với cách này, khi mã độc được nạp vào chương trình, chương trình sẽ không bị crash và mã độc sẽ được kích hoạt.

## Time Line tìm kiếm thông tin
1. Tìm về DLL Hijacking, để ngẫm lại đúng về bản chất và cách thức thực hiện
2. Tìm kiếm thông tin về Side-loading (Không có quá nhiều thông tin, nhưng biết được có 1 cách gọi khác là DLL Proxy)
3. Tìm kiếm về DLL Proxy, tìm kiếm về cách thực hiện
4. Tường tận về thông tin và thực hành

## Kết luận: 
- Bản chất DLL side-loading là lợi dụng file đã ký để thực thi file chưa được ký, vậy nên cái chúng ta cần là kích hoạt được mã độc bằng file đã kí mà không bị crash chương trình.
- Cách tốt nhất hiện tại là tạo file DLL trung gian giúp load file DLL gốc và kích hoạt file mã độc

# Step to reproduce
Bước 1: Chọn lấy 1 file exe đã được kí sẵn. 
Ở đây tôi chọn GUP.exe, một file thuộc chức năng Update của notepad++ 
<img width="811" height="646" alt="image" src="https://github.com/user-attachments/assets/1973f7bb-5c8b-4d0f-8d3e-cb4fd055c640" />

Bước 2: Phân tích hành vi của file exe đã được kí <br>
Tôi sử dụng Procmon để theo giõi hành động của GUP.exe.
Trong ảnh là các filter được áp dụng
<img width="773" height="685" alt="image" src="https://github.com/user-attachments/assets/4a814d97-6d44-486e-bb6b-af62aaf7b67f" />


Bước 3: Phân tích file DLL được load 
Sau khi áp dụng filter tôi đã tìm ra nơi mà cá thư viện DLL được load, nhưng chương trình đã không thể tìm thấy các DLL này.
<img width="1294" height="733" alt="image" src="https://github.com/user-attachments/assets/8c074718-04dc-4e76-a8e3-3ae14658b8d5" />

Bước 4: Tạo mã độc 
Tôi sử dụng MSFVenom để tạo ra một mẫu mã độc nhỏ
<img width="859" height="193" alt="image" src="https://github.com/user-attachments/assets/cf7e6490-bb85-4ac4-b270-476fc26e1ccc" />


Bước 5: Tạo file DLL Proxy 
Tôi chọn file ncrypt.dll để làm nơi load mã độc
<img width="855" height="220" alt="image" src="https://github.com/user-attachments/assets/f7ce16b0-242b-456a-ad52-2ac3a0f227ca" />

Sau khi chọn được nơi load mã độc rồi, tôi sẽ cần phải tìm ra mẫu file ncrypt.dll để tiến hành custom (Ở đây tôi tiếp dùng dùng procmon để filter và tìm ra nơi mà chương trình đã load được ncrypt.dll gốc)
<img width="1288" height="561" alt="image" src="https://github.com/user-attachments/assets/8ea9cca0-7b61-4f62-90a2-aad1dd6800a5" />

Xem các hàm được export bởi ncrypt.dll bằng công cụ dumpbin
<img width="1054" height="1449" alt="image" src="https://github.com/user-attachments/assets/5532f5bb-d1de-424a-889b-db36a5fddef7" />

Link các hàm từ DLL gốc đến DLL Proxy
<img width="1497" height="1172" alt="image" src="https://github.com/user-attachments/assets/096dc92b-16d4-4f02-8908-05d82ab7e9fd" />

Câu lệnh kích hoạt mã độc
<img width="1186" height="409" alt="image" src="https://github.com/user-attachments/assets/f145979f-db6c-4caa-bba6-8e507b21a537" />

Sau khi hoàn thành file mã nguồn của file DLL proxy, ta hoàn toàn có thể build nó thành DLL qua tính năng biên dịch DLL của C/C++

Đổi tên dll gốc thành tmpru3otb.dll để ngụy trang, đổi tên của DLL trung gian thành ncrypt.dll và đặt mã độc tên là payload.bin ở cạnh file ncrypt.dll
<img width="626" height="92" alt="image" src="https://github.com/user-attachments/assets/39c29a70-3a65-492a-a3a5-67c9516bf5e7" />

Giờ đặt các file này vào trong thư mục load DLL mà bị not found mà chúng ta đã tìm thấy khi phân tích hành vi của GUP.exe 
<img width="797" height="640" alt="image" src="https://github.com/user-attachments/assets/c5efdeca-103c-4919-b812-a2e4cfa06d4c" />


Bước 6: Khởi động chương trình
Khi chương trình khởi chạy, DLL sẽ được load thành công.
<img width="1873" height="759" alt="image" src="https://github.com/user-attachments/assets/56d56df0-c49c-4df1-b275-1690c9bf4b8b" />

Và ở phía máy Attacker liền nhận kết nối thành công
<img width="1097" height="1003" alt="image" src="https://github.com/user-attachments/assets/bb6beb3d-84b8-466d-ad1b-d8da1a454c89" />


# Reference
https://www.ired.team/offensive-security/persistence/dll-proxying-for-persistence </br>
https://attack.mitre.org/techniques/T1574/001 </br>
https://www.r-tec.net/r-tec-blog-dll-sideloading.html </br>
