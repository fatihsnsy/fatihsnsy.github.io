---
title: TLS Callback ile Eski Bir Anti-Debug Tekniği
date: 2020-08-26 12:00 +03:00
tags: [anti-debug techniques, thread local storage, tls, tls callback, tls callback anti-debug, tls callback code snippet, tls callback ile anti-debug tekniği, tls callback implementation]
description: Zararlı yazılımların uzun yıllardan beridir başvurdukları Anti-Debug tekniklerinden TLS Callback'e farklı bir şekilde göz atıyoruz...
image: "/assets/img/tls-callback-eski-antidebug-teknigi/img/cover.png"
---

Zararlı uygulamalar her ne kadar ofans odaklı olsalar da, ofansif davranışlarını daha uzun ve etkili bir şekilde sürdürmek için defansif yeteneklerini de sıklıkla ortaya koymaktadırlar. Bir kedi-fare oyunu olan bu alanda, zararlı aktörlerin  en büyük hedeflerinden birisi de yazdıkları zararlıların analiz edilmesini olabildiğince engellemeye çalışmaktır. Sizlerle birlikte adını sık sık duyduğumuz, demode olan fakat kedi-fare oyunun başlangıcı sayılabilecek bir tekniği inceleyecek ve gerçeklemesini yapacağız.

Fakat tekniğin detayları ve gerçeklemesine geçmeden önce TLS’in ne olduğunu bilmemiz, bize büyük avantaj kazandıracaktır.

## TLS Nedir?

Açılımı **Thread Local Storage** olan TLS, processin iş parçacıklarının (**thread**) kendilerine özgü verileri bellekte depolayabildiği bir yöntemdir. Çalışma zamanında (**runtime**) iş parçacığına özgün olan veriler TlsAlloc, TlsGetvValue, TlsSetValue gibi Windows API’leri ile desteklenebilmektedir. Ayrıca bilindiği üzere bir processin tüm iş parçacıkları aynı sanal alanı paylaşmaktadır. Bir fonksiyon içerisindeki lokal bir değişken, onu çalıştıran iş parçacığına özgüdür. Fakat fonksiyon haricinde yer alan statik ve global değişkenler ise, processin tüm iş parçacıkları ile paylaşılmaktadır. TLS yapısında oluşturulan değişkenlerde ise durum biraz farklıdır. TLS yapısında iş parçacıklarının, değişkenler için kendilerine özgü olan kopyaları bulunmaktadır.  Yani TLS yapısında, değiştirilen bir değişkenin değeri sadece ilgili olan iş parçacığında değişime uğramaktadır. Diğer iş parçacıklarının kendilerine özgü kopyaları olduğundan dolayı bir değişim olmamaktadır.

TLS yapısı ile, processin global bir index kullanmasıyla, her iş parçacığına özgü veriler sağlaması mümkündür. Aşağıdaki görsel tam olarak TLS’in nasıl çalıştığını anlatmaktadır.

![TLS Callback Cover](/assets/img/tls-callback-eski-antidebug-teknigi/img/cover.png) 

Yukarıdaki görselde iki iş parçacığının TLS kullanımını görmektesiniz. TLS, gdwTlsIndex1 ve gdwTlsIndex2 adında 2 adet global veri oluşturur. TlsGetValue API’si ile iş parçacıklarının ilgili indexlerinin karşılığı olan değer lpvData değişkeninde depolanır.

## TLS Callback İle Debug Tespiti

TLS’in çalışma mantığını az da olsa makalenin konusunu anlayabilecek kadar kavradık. Şimdi ise asıl konumuza geri dönüyoruz. Eski bir gelenek olan (eski diyorum çünkü günümüzde bir çok modern debug tespit yöntemleri geliştirildi.  TLS Callback ile debug tespitinin nasıl olduğuna göz atacağız.

TLS Callback yönteminin popüler olmasının en büyük sebeplerinden birisi, **entrypoint** noktasına gelmeden önce çalışıyor olmasıdır. Yani zararlı aktörler, uygulamalarının ana amacı belli olmadan önce, programın başlangıcında TLS Callback yöntemi ile zararlı uygulamalarının debug edilip edilmediğini anlayabilmektedirler. Eski bir anti-debug tekniği olan TLS Callback, günümüzde kolaylıkla bypass edilebilmektedir.

TLS Callback anti-debug tekniğini daha iyi anlamak adına uygulamasını yapalım.

```cpp
#include <iostream>
#include <windows.h>
#pragma comment(lib, "ntdll.lib")
#pragma section(".CRT$XLB", read) //.CRT$XLB adında yeni bir section oluşturduk. Section adının aynı olması gerekmektedir.
#define NtCurrentProcess() (HANDLE)-1
extern "C" NTSTATUS NTAPI NtQueryInformationProcess(HANDLE hProcess, ULONG InfoClass, PVOID Buffer, ULONG Length, PULONG ReturnLength);
#ifdef _WIN64
#pragma comment(linker, "/INCLUDE:_tls_used")
#define readpeb (PBOOLEAN)__readgsdword(0x60) + 2 //PEB yapısında BeingDebugged'a işaret ediyor. x64 için
#else
#pragma comment(linker, "/INCLUDE:__tls_used")
#define readpeb (PBOOLEAN)__readfsdword(0x30) + 2 //x32 için
#endif
void WINAPI DebuggerDetect(PVOID handle, DWORD Reason, PVOID Reserved) {
    PBOOLEAN BeingDebugged = readpeb;
    HANDLE DebugPort = NULL;
    if (*BeingDebugged) {
        printf("Debugger Detected!!!\n");
    }
    else {
        printf("No debugger detected!\n");
    }
    if (!NtQueryInformationProcess(NtCurrentProcess(), 7, &DebugPort, sizeof(HANDLE), NULL)) { 
        //DebugPort 0 değilse programın debug edildiğinin göstergesidir.
        if (DebugPort) {
            printf("Debugger Detected via QueryInformationProcess!!!\n");
            exit(1);
        }
        else {
            printf("No debugger detected for QueryInformationProcess\n");
        }
    }
}
__declspec(allocate(".CRT$XLB")) PIMAGE_TLS_CALLBACK CallbackAddress[] = { DebuggerDetect, NULL };
int main() {
    printf("Hello, you're now in the entrypoint!");
    return 0;
}
```

Yukarıdaki implementesini yaptığımız TLS Callback ile anti-debug tekniğine bir göz atalım. **NtQueryInformationProcess** API’si ile de bir anti-debug tekniğini eklediğimiz örneğimizde pragma anahtar sözcükleri ile derleyicimize talimatlar vermekteyiz. Ntdll kütüphanesini dahil et, yeni bir section oluştur, linker’a belirtilenleri bağla gibi komutları zaten kodun içerisinde görmektesiniz.  Buradaki en önemli noktalardan birisi **__declspec** anahtar sözcüğü ile tanım yaptığımız satırdır. Pragma anahtar sözcüğünde oluşturduğumuz section için yer ayırma yapıyoruz ve TLS Callback fonksiyonumuzun adresini veriyoruz. Daha sonra yazmış olduğumuz **DebuggerDetect** fonksiyonu çalışıyor ve bu fonksiyondan sonra **main**() fonksiyonumuz çalışıyor.

    ntdll!_PEB
       +0x000 InheritedAddressSpace : UChar
       +0x001 ReadImageFileExecOptions : UChar
       +0x002 BeingDebugged    : UChar
       +0x003 BitField         : UChar
       +0x003 ImageUsesLargePages : Pos 0, 1 Bit
       +0x003 IsProtectedProcess : Pos 1, 1 Bit
       +0x003 IsImageDynamicallyRelocated : Pos 2, 1 Bit
       +0x003 SkipPatchingUser32Forwarders : Pos 3, 1 Bit
       +0x003 IsPackagedProcess : Pos 4, 1 Bit
       +0x003 IsAppContainer   : Pos 5, 1 Bit
       +0x003 IsProtectedProcessLight : Pos 6, 1 Bit
       +0x003 IsLongPathAwareProcess : Pos 7, 1 Bit
       +0x004 Padding0         : [4] UChar
       +0x008 Mutant           : Ptr64 Void
       +0x010 ImageBaseAddress : Ptr64 Void
       +0x018 Ldr              : Ptr64 _PEB_LDR_DATA
       +0x020 ProcessParameters : Ptr64 _RTL_USER_PROCESS_PARAMETERS
       +0x028 SubSystemData    : Ptr64 Void
       +0x030 ProcessHeap      : Ptr64 Void
       +0x038 FastPebLock      : Ptr64 _RTL_CRITICAL_SECTION
       +0x040 AtlThunkSListPtr : Ptr64 _SLIST_HEADER
       +0x048 IFEOKey          : Ptr64 Void
       +0x050 CrossProcessFlags : Uint4B
       +0x050 ProcessInJob     : Pos 0, 1 Bit
       +0x050 ProcessInitializing : Pos 1, 1 Bit
       +0x050 ProcessUsingVEH  : Pos 2, 1 Bit
       +0x050 ProcessUsingVCH  : Pos 3, 1 Bit
       +0x050 ProcessUsingFTH  : Pos 4, 1 Bit
       +0x050 ProcessPreviouslyThrottled : Pos 5, 1 Bit
       +0x050 ProcessCurrentlyThrottled : Pos 6, 1 Bit
       +0x050 ReservedBits0    : Pos 7, 25 Bits
       +0x054 Padding1         : [4] UChar
       +0x058 KernelCallbackTable : Ptr64 Void
       +0x058 UserSharedInfoPtr : Ptr64 Void
       +0x060 SystemReserved   : Uint4B
       +0x064 AtlThunkSListPtr32 : Uint4B
       +0x068 ApiSetMap        : Ptr64 Void
       +0x070 TlsExpansionCounter : Uint4B
       +0x074 Padding2         : [4] UChar
       +0x078 TlsBitmap        : Ptr64 Void
       +0x080 TlsBitmapBits    : [2] Uint4B
       +0x088 ReadOnlySharedMemoryBase : Ptr64 Void
       +0x090 SharedData       : Ptr64 Void
       +0x098 ReadOnlyStaticServerData : Ptr64 Ptr64 Void
       +0x0a0 AnsiCodePageData : Ptr64 Void
       +0x0a8 OemCodePageData  : Ptr64 Void
       +0x0b0 UnicodeCaseTableData : Ptr64 Void
       +0x0b8 NumberOfProcessors : Uint4B
       +0x0bc NtGlobalFlag     : Uint4B
       +0x0c0 CriticalSectionTimeout : _LARGE_INTEGER
       +0x0c8 HeapSegmentReserve : Uint8B
       +0x0d0 HeapSegmentCommit : Uint8B
       +0x0d8 HeapDeCommitTotalFreeThreshold : Uint8B
       +0x0e0 HeapDeCommitFreeBlockThreshold : Uint8B
       +0x0e8 NumberOfHeaps    : Uint4B
       +0x0ec MaximumNumberOfHeaps : Uint4B
       +0x0f0 ProcessHeaps     : Ptr64 Ptr64 Void
       +0x0f8 GdiSharedHandleTable : Ptr64 Void
       +0x100 ProcessStarterHelper : Ptr64 Void
       +0x108 GdiDCAttributeList : Uint4B
       +0x10c Padding3         : [4] UChar
       +0x110 LoaderLock       : Ptr64 _RTL_CRITICAL_SECTION
       +0x118 OSMajorVersion   : Uint4B
       +0x11c OSMinorVersion   : Uint4B
       +0x120 OSBuildNumber    : Uint2B
       +0x122 OSCSDVersion     : Uint2B
       +0x124 OSPlatformId     : Uint4B
       +0x128 ImageSubsystem   : Uint4B
       +0x12c ImageSubsystemMajorVersion : Uint4B
       +0x130 ImageSubsystemMinorVersion : Uint4B
       +0x134 Padding4         : [4] UChar
       +0x138 ActiveProcessAffinityMask : Uint8B
       +0x140 GdiHandleBuffer  : [60] Uint4B
       +0x230 PostProcessInitRoutine : Ptr64     void 
       +0x238 TlsExpansionBitmap : Ptr64 Void
       +0x240 TlsExpansionBitmapBits : [32] Uint4B
       +0x2c0 SessionId        : Uint4B
       +0x2c4 Padding5         : [4] UChar
       +0x2c8 AppCompatFlags   : _ULARGE_INTEGER
       +0x2d0 AppCompatFlagsUser : _ULARGE_INTEGER
       +0x2d8 pShimData        : Ptr64 Void
       +0x2e0 AppCompatInfo    : Ptr64 Void
       +0x2e8 CSDVersion       : _UNICODE_STRING
       +0x2f8 ActivationContextData : Ptr64 _ACTIVATION_CONTEXT_DATA
       +0x300 ProcessAssemblyStorageMap : Ptr64 _ASSEMBLY_STORAGE_MAP
       +0x308 SystemDefaultActivationContextData : Ptr64 _ACTIVATION_CONTEXT_DATA
       +0x310 SystemAssemblyStorageMap : Ptr64 _ASSEMBLY_STORAGE_MAP
       +0x318 MinimumStackCommit : Uint8B
       +0x320 FlsCallback      : Ptr64 _FLS_CALLBACK_INFO
       +0x328 FlsListHead      : _LIST_ENTRY
       +0x338 FlsBitmap        : Ptr64 Void
       +0x340 FlsBitmapBits    : [4] Uint4B
       +0x350 FlsHighIndex     : Uint4B
       +0x358 WerRegistrationData : Ptr64 Void
       +0x360 WerShipAssertPtr : Ptr64 Void
       +0x368 pUnused          : Ptr64 Void
       +0x370 pImageHeaderHash : Ptr64 Void
       +0x378 TracingFlags     : Uint4B
       +0x378 HeapTracingEnabled : Pos 0, 1 Bit
       +0x378 CritSecTracingEnabled : Pos 1, 1 Bit
       +0x378 LibLoaderTracingEnabled : Pos 2, 1 Bit
       +0x378 SpareTracingBits : Pos 3, 29 Bits
       +0x37c Padding6         : [4] UChar
       +0x380 CsrServerReadOnlySharedMemoryBase : Uint8B
       +0x388 TppWorkerpListLock : Uint8B
       +0x390 TppWorkerpList   : _LIST_ENTRY
       +0x3a0 WaitOnAddressHashTable : [128] Ptr64 Void
       +0x7a0 TelemetryCoverageHeader : Ptr64 Void
       +0x7a8 CloudFileFlags   : Uint4B
       +0x7ac CloudFileDiagFlags : Uint4B
       +0x7b0 PlaceholderCompatibilityMode : Char
       +0x7b1 PlaceholderCompatibilityModeReserved : [7] Char
   
x86 ve x64 mimari uyumlu olan implementemizde, öncelikle mimariye göre PEB yapısına erişmemiz gerekiyor. PEB yapısına eriştikten sonra yukarıdaki tabloda da görüldüğü üzere yapının 3. sıradaki değişkeni olan BeingDebugged’a erişim sağlamak adına 2. indexi belirtiyoruz.Eğer BeingDebugged 0’dan farklı bir değere sahip ise bu, programın debug edildiği anlamına gelmektedir. Ayrıca yukarıdaki kod parçasında mimariye göre PEB’in konumunun farklılık gösterdiği göz ardı edilmemelidir.

PEB ile anti-debug tekniğinin ardından küçük bir ekleme daha yaparak anti-debug uygulamamızı biraz daha geliştiriyoruz. Windows API’larının sunduğu nimetlerden faydanalanarak, uygulamanını debug edilip edilmediğini anlayabilmekteyiz. Örnekte görüldüğü üzere **NtQueryInformationProcess** API’si ile benzer bir anti-debug tekniği kullanmış olduk. İlgili API, az önce bahsettiğimiz gibi kendisi mimariye göre PEB’in başlangıç noktasını bulur ve bizimle neredeyse aynı işlemleri yapar. **ProcessInformationClass** parametresine geçtiğimiz 7 değeri ile **DebugPort** değişkenine atama yapmaktayız. Eğer DebugPort değişkeni 0’dan farklı bir değere sahip ise programın debug edildiği anlaşılmaktadır.

![TLS Callback Result](/assets/img/tls-callback-eski-antidebug-teknigi/img/tls-callback-1.png) 

Sizlere bu makalemde TLS’in genel olarak tanımı ve yapısı, TLS Callback ile debug tespiti ve PEB yapısına değinmiş olduk. Bir sonraki makalelerimde farklı anti-debug ve anti-reverse teknikleri ile karşınızda olacağım. Sağlıcakla kalın…


## Kaynakça

[1]  https://en.wikipedia.org/wiki/Thread-local_storage

[2] https://docs.microsoft.com/tr-tr/cpp/parallel/thread-local-storage-tls

[3] https://docs.microsoft.com/en-us/windows/win32/procthread/thread-local-storage

[4] http://rinseandrepeatanalysis.blogspot.com/p/peb-structure.html
