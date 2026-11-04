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

Bước 2: Phân tích hành vi của file exe đã được kí 
Tôi sử dụng Procmon để theo giõi hành động của GUP.exe.

Bước 3: Phân tích file DLL được load 
sử dụng Procmon và áp dụng các filter và tìm ra nơi mà DLL được load

Bước 4: Tạo file DLL Proxy 

Bước 5: Tạo mã độc 

Bước 6: Khởi động chương trình

# Reference
https://www.ired.team/offensive-security/persistence/dll-proxying-for-persistence </br>
https://attack.mitre.org/techniques/T1574/001 </br>
https://www.r-tec.net/r-tec-blog-dll-sideloading.html </br>
