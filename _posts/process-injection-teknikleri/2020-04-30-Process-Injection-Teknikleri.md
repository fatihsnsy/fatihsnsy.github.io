---
title: 💉 Process Injection Teknikleri Ve Detayları
date: 2020-04-30 22:36 +03:00
tags: [apc dll injection, atom bombing injection, process doppelganging, process hollowing, process injection, process injection techniques, process walking, remote dll injection]
description: Bir eğitim niteliğinde olan bu makalemizde sizlere Process Injection Tekniklerini olabildiğince detaylı ve açıklayıcı şekilde anlattım.
image: "/assets/img/process-injection-teknikleri/img/cover.jpg"
---

Hiç legal bir sistem uygulamasının sistem kaynaklarını gereğinden fazla tüketme ve olağan dışı ağ hareketleri gibi alışılmadık davranışlarda bulunduğunu farkettiniz mi? Forumlarda sık sık karşımıza çıkan “svchost.exe virüs müdür?” gibi sorulara işin farklı bir yüzünden en teknik detayları ile cevap veriyorum. Bir eğitim niteliğinde olan bu makalemizde sizlere Process Injection Tekniklerini olabildiğince detaylı ve açıklayıcı şekilde anlattım.

## Process Injection Nedir?

Process Injection (Code Injection diye de nitelendirilir) işleminde temel amaç, zararlı bir uygulamanın veya kodun, legal bir process’in belleğine enjekte edilmesidir. Legal process’in belleğine enjekte edilecek olan nesne, bazen bir executable, bazen bir DLL, bazen de Shellcode olabilir. Enjekte işlemi tamamlandıktan sonra ise legal process bu enjekteyi çalıştırmaya zorlanır. Process Injection saldırgana bir çok avantaj sağlamakla beraber, enjeksiyon yapan saldırgan şu işlemleri yapabilir:

- Legal process’i dosya indirme, yükleme ve keyboard hareketlerini almaya zorlamak gibi işlemler yapabilir.
- API çağrılarını yönlendirebilir, API’lerin parametrelerini ele geçirebilir ve API’lerin export’larını filtreleyebilir.
- Legal process’e enjekte işlemi yaptığı için bazı güvenlik ürünlerini dolaylı yoldan baypass’layabilir.
- Ve çok daha fazlasını yapabilir.
Evet, genellikle saldırganlar (malware geliştiricileri) bu teknikten sık sık faydalanır. Peki sadece saldırganlar mı yararlanır? Tabiki de hayır. Bir çok güvenlik ürünü sistem üzerinde korumayı tam olarak gerçekleştirebilmek adına enjeksiyon tekniklerini kullanmaktadır. Bunu da bir not olarak düşmekte fayda var.

Genel bir örnek ile açıklayacak olursak, enjekte edilecek nesne (exe, DLL,Shellcode vs.) var olan bir process’e veya kendi tarafından başlatılan bir process’e enjekte olabilmesi için öncelikle enjekte olacağı process’i tanımlaması gerekir. Bunun için **enumerate** işlemi yapılabilir. Enumerate işlemi için birkaç Windows API’si bulunmaktadır. Bunlar;

- CreateTool32HelpSnapshot()
- Process32First()
- Process32Next()

**CreateToolhelp32Snapshot()** API’si, o an sistemde çalışan tüm process’lerin bir snapshot’ını alır. Daha sonra **Process32First()** API’si snapshot’ı alınan tüm process’ler arasından ilki hakkında bilgiler alır. **Process32Next()** ile de snapshot’ı alınan diğer tüm process’ler arasında tek tek gezme işlemi yapar ve hepsi hakkında bilgi toplar. Process32First() ve Process32Next() fonksiyonları sayesinde enumerate edilen process’ler hakkında alınabilen bilgilerden bazıları şunlardır;

- Executable’ın adı,
- Process ID’si (PID),
- Child Process’in ID’si,
Ve daha fazla bilgiye erişilebilmektedir.

Process32First() ve Process32Next() API’lerinin birlikte kullanımı **Process Walking** adında bir tekniği de temsil etmektedir. Process Walking, sistem snapshot’ındaki process’leri tek tek gezerek bilgi toplama işlemine verilen addır. Ayrıca **Toolhelp32ReadProcessMemory()** API’si ile de belirli bir process’in belleğini okunabilir.

Process Walking işlemi ile malware, enjekte edilecek process’in var olup olmadığını kontrol edebilir. Var ise enjekte olabilir, yok ise hedef process’i başlatabilir. Daha sonra ise malware nesnesi (DLL, executable, Shellcode vs.) legal process’in belleğine kendisini enjekte eder ve legal process’in enjekte olan zararlıyı çalıştırması için zorlar.

![Process Injection Scheme](/assets/img/process-injection-teknikleri/img/procinj-1.png)

Yukarıdaki görselde bu işlem anlaşılır bir şekilde gösterilmiştir. Malware’ın user space’de çalıştığını da unutmayalım!

Process Injection hakkında bir genelleme yaptık fakat bu ana başlık, farklı farklı teknikleri alt dallarında barındırmaktadır. Şimdi ise Process Injection tekniklerinin detaylarına değinecek ve en çok bilinen, etkili teknikleri açıklayacağız.

## 1.   Remote DLL Injection
Remote DLL Injection metoduna geçmeden önce DLL hakkında kısa bir bilgi vermekte fayda var. Açılımı Dynamic Linking Library olan DLL’ler bir kod/veri kütüphanesidir. Bir çok uygulamanın ortak bir şekilde kullanması için tasarlanmıştır. DLL kullanımı daha fazla performans, daha az bellek kullanımı gibi faydalar sağlamaktadır.

![Process Injection 2](/assets/img/process-injection-teknikleri/img/procinj-2.png)

Yukarıdaki görselde de görüldüğü üzere MZ ve PE headerlarına sahip olmasına rağmen bir executable’ın karakteristiğine sahip olsa da tek başına çalışamamaktadır. Kısa bir şekilde DLL’den de bahsettiğimize göre Remote DLL Injection’a geçiş yapabiliriz.

Remote DLL Injection metodu geçtiğimiz zamanlarda ve günümüzde sıkça kullanılmaktadır. Malware legal bir process’in virtual memory’sine zararlı DLL’in yolunu yazarak ve legal process’de remote thread oluşturarak bu zararlı DLL’in yüklenmesini sağlar.

Öncelikle zararlının yapması gereken işlem Process32First, Process32Next ve CreateToolhelp32Snapshot ile az önce bahsettiğim **Process Walking **işlemlerini yapıp injekte olacağı process’i belirlemektir.

Daha sonra ise OpenProcess API’sini kullanarak tespit ettiği hedef process’in handle’ını alır. Handle’ın tanımını Windows’un MSDN dökümanlarından alıntı yaparak açıklayacak olursak;

> Handle, bir nesneye yapılan başvurudur. Bir process’in bir nesneye (dosya, kayıt defteri, mutex vb.) erişebilmesi bir handle açması gerekir. Örnek vermek gerekirse bir process’in dosyaya yazma işlemi yapmak istediğini düşünelim. Process önce gerekli API’yi(WriteFile) çağırır. Daha sonra  handle’ı WriteFile API’sine ileterek dosyaya yazmak için handle’ı(tanıcı da deniyor) kullanır.

Process’in handle’ını alan malware, daha sonra **VirtualAllocEx** API’si ile bellekte allocate (yer ayırma) işlemi uygular. Daha sonra bellekte ayırdığı lokasyona  **WriteProcessMemory** API’si ile zararlı DLL’in yolunu yazar.

![Process Injection 3](/assets/img/process-injection-teknikleri/img/procinj-3.png)

Daha sonra bellekteki lokasyona yolu yazılan zararlı DLL’in thread’ler tarafından çalıştırılması gerekir. Bunun için de malware, **CreateRemoteThread**, NtCreateThreadEx, RtlCreateUserThread gibi API’leri çağırır. Ve bu API’lerin içine DLL yükleme için kullanılan LoadLibrary API’sini yerleştirir. LoadLibrary API’sinin içerisine ise zararlı DLL’in yerini yerleştirir. Bu işlemlerden sonra Remote DLL Injection’ın pseudo kodu şu şekilde olmaktadır:

`CreateRemoteThread(LoadLibrary(C:\Program Files\zararli.dll))`

CreateRemoteThread API’si artık bir çok güvenlik ürünü tarafından izlenmektedir. Akıllı bir malware geliştiricisi bu API’yi kullanmayacaktır. Aşağıdaki görselde ise bu yöntemi kullanan Rebhip worm’una ait bir statik kod analizini görmektesiniz.

![Process Injection 4](/assets/img/process-injection-teknikleri/img/procinj-4.png)

Örnek bir Remote DLL Injection sonrası amacımıza ulaşabiliyoruz:

![Process Injection 5](/assets/img/process-injection-teknikleri/img/procinj-5.png)

## 2. APC DLL Injection

CreateRemoteThread() API’si ile Remote DLL Injection tekniğinin ardından şimdi ise APC DLL Injection tekniğini göreceğiz.

Bu teknik, Remote DLL Injection tekniğine benzer. Fakat ayrım noktası, DLL enjekte işleminde CreateRemoteThread() API’si yerine Windows’un APC(Asynchronous Procedure Call)’sini kullanır. APC’nin kısa bir tanımını yapacak olursak;

> APC, belirli bir thread bağlamında eşzamansız olarak çalışan bir işlevdir. Her thread, hedef thread uyarılabilir bir duruma girdiğinde yürütülecek bir APC sırası içerir.

Yani özetleyecek olursak, APC threadlerin bekleme zamanında daha az bellek kullanması için tasarlanmış bir çalışma birimidir. Bir programın birden fazla thread ile çalıştığını varsayalım. Genellikle thread’ler birbirleri ile eşzamanlı olarak çalışır. Fakat bazı veriler hazır değilse (örneğin program kullanıcıdan girdi veya onay bekliyorsa) thread, stackt’te önemli bir miktarda memory’den yer ayırdığı ve bu bellek onay gelene kadar kullanılamayacağı için thread’i bekleme durumunda memory’de tutmak pek mantıklı olmayacaktır.

Bundan ötürü thread stack’i bellekte daha az yer kaplayan bir nesne olarak oluşturulur. Ve bu nesne, kullanıcı girişlerini alan hizmete aktarılır. Kullanıcıdan yanıt alındığında hizmet bunu nesneye koyar ve nesneyi execute birimine iletir.

Execute hizmeti ise bir veya daha fazla thread ve görev kuyruğundan oluşur. Her çalışan thread bir görev aldığında bunu execute eder. Herhangi bir görev olmadığında ise thread bekler ve böylece bellek kullanılmaz.

Kısa bir şekilde APC’nin de tanımını yaptığımıza göre APC DLL Injection tekniğine geçebiliriz.

Az önce thread’in uyarılabilir(alterable) duruma geçmesinden bahsetmiştik. Bir thread aşağıdaki API’lerden birisini çağırdığında alterable duruma geçebilir:

- SleepEx();
- SignalObjectAndWait();
- MsgWaitForMultipleObjectsEx();
- WairForMultipleObjectsEx();
- WaitForSingleObjectEx();

APC DLL Injection tekniğinde temel amaç; malware’ın hedef process’teki alterable durumda olan veya alterable duruma geçme olasılığı bulunan thread’i tanımlaması ile başlar. Daha sonra zararlı olan custom code’u QueueUserAPC() API’sini kullanarak thread’in APC kuyruğuna yerleştirir. Daha sonra ise thread, kuyruğa alınan bu zararlı custom code’un sırası geldiğinde onu çalıştırır.

### Örnekleyelim

Tekniği anlattık, şimdi ise kısa bir örnek verelim. Zararlı DLL’in legal iexplore.exe uygulamasını APC DLL Injection yöntemi ile enjeksiyonuna göz atacağız.

Bu teknik, Remote DLL Injection’daki 4 adımın aynısını uygular. Yani bir handle açar, hedef process’in memory’sinde yer ayırır, malicious DLL’in yolunu ayrılan belleğe kopyalar ve LoadLibrary() API’sinin adresini belirler. Daha sonra ise hedef thread’in malicious DLL’i yüklemeye zorlanması için şu adımları izler:

1. OpenThread() API’si ile hedef process’in thread’ine bir handle açar. Parametrelerinden birisi ise iexplore.exe process’inin thread’inin ID’sidir.

![Process Injection 6](/assets/img/process-injection-teknikleri/img/procinj-6.png)

OpenThread() API’sinin dönüş değeri iexplore.exe thread’inin handle’ı olmaktadır.

2. Malware process’i, Internet Explorer’ın thread’inin APC kuyruğundaki APC işlevini sıralamak için QueueUserAPC() API’sini çağırır.

Bunun ilk parametresi malware’ın hedef thread’de yürütülmesini istediği  APC işlevinin işaretçisidir. Yani APC işlevi adresi daha önce belirlenen LoadLibrary() API’sinin kendisidir. İkinci parametresi ise hedef process’in hedef thread’inin handle’ıdır. Üçüncü parametresi ise hedef process’in memory’sinde yer alana zararlı DLL’in tam yolunu içeren adrestir. Thread execute işlemi yaptığında bu adres, LoadLibrary() API’sine parametre olarak iletilmekte ve zararlı DLL execute edilmektedir.

![Process Injection 7](/assets/img/process-injection-teknikleri/img/procinj-7.png)

Görselde de görüldüğü üzere 3. Parametre, iexplore.exe process’inin process memory adresidir.

![Process Injection 8](/assets/img/process-injection-teknikleri/img/procinj-8.png)

Adrese baktığımızda ise zararlı DLL’in tam yolunu görmekteyiz.

## 3. Process Hollowing
Başka bir kod enjeksiyon tekniklerinden birisi olan Process Hollowing, legal bir process’in belleğine zararlı executable’ın enjekte edilmesini amaç edinir.

Process Hollowing tekniği, saldırgana bir çok avantaj sağlar. En önemlisi ise güvenlik ve adli analiz araçları tarafından fark edilmemesini sağlar. Örneğin legal bir process olan iexplore.exe’ye Process Hollowing tekniği olan bir malware üzerinden konuşacak olursak, process’in yolu legal process olan  iexplore.exe’nin yolunu gösterecektir. Ama iexplore.exe’nin belleğinde ise zararlı executable barınmaktadır.

Hollowing’e sözcük bakımından genelde “kancalamak” denmektedir. Process Hollowing tekniğini gerçekleştirecek olan malware, öncelikle legal process’i suspend durumda başlatır.

Suspend durumda başlayan legal process’in executable section’ı belleğe yüklenmiş olur. **PEB** (Process Environment Block) yapısı, memory’e yüklenen legal process’in tam yolunu içerir. PEB’in ImageBaseAddress kısmı ise legal process’in bellekteki executable section’ının hangi adreste olduğunun bilgisini tutar.

Aşağıdaki görselde malware tarafından suspend durumda başlatılan svchost.exe process’ini görüyoruz. Svchost.exe process’i belleğin **0x01000000** adresine yüklenmiş durumda.

Daha sonra malware, PEB.ImageBaseAddress kısmına erişmek için bellekteki PEB yapısının adresini belirler. ImageBaseAddress kısmına eriştiğinde ise legal process’in memory’deki base adresini elde eder.

![Process Injection 9](/assets/img/process-injection-teknikleri/img/procinj-9.png)

PEB’in tespitinden sonra malware, **GetThreadContext**() API’sini çağırır. GetThreadContext() API’si belirtilen thread’in içeriğini alır. Ve iki adet parametre alır. Bunlardan ilki thread’in handle’ıdır. İkinci parametre ise yapının CONTEXT adındaki pointer’ıdır.

Malware ilk parametreye suspend edilen thread’in handle’ını, ikinci parametreye ise CONTEXT yapısının pointer’ını geçer. API çağrısından sonra CONTEXT yapısı, suspend edilen thread’in bağlamı (kaynağı) ile doldurulur.

Bu CONTEXT yapısı artık askıya alınan register durumlarını içerir. Malware daha sonra PEB yapısının işaretçisini içeren CONTEXT._EBX alanını okur. PEB adresi belirlendikten sonra ise ImageBaseAddress kısmını okuduğunu söylemiştik. Bunu yapmasının amacı da legal executable’ın base adresini belirlemekti.

![Process Injection 10](/assets/img/process-injection-teknikleri/img/procinj-10.png)

Yukarıdaki görselde process belleğinin okunma işlemi görülmektedir.

PEB pointer’ını tespit etmek için diğer bir yöntemin NtQueryInformationProcess API’si olduğunu söylemekte de fayda var. Hedef legal process’in base adresini belirleyen malware, legal process’in executable section’ını (çalıştırılabilir kısmını) bellekten ayırır (deallocate eder). Bunu da NtUnMapViewofSection() API’si ile gerçekleştirir.

![Process Injection 11](/assets/img/process-injection-teknikleri/img/procinj-11.png)

Yukarıdaki görselde ilk parametrenin svchost.exe legal process’inin handle’ı, ikinci parametrenin ise legal process’in base adresi olduğunu görüyorsunuz. Bu işlemden sonra legal process’in executable section’ı bellekten ayrılmış, yani unmap edilmiş oluyor. Legal process’in memory’sinden boşaltılan, deallocate edilen kısımda ise RWX izinlerine sahip yeni bir kısım allocate edilir.

Yeni bellek adresi önceki process ile aynı adreste veya farklı bir adreste allocate edilebilir. Yukarıdaki görselde VirtualAllocEx() API’sini memory’de **0x00400000** adresinde ayırma yapması için çağırdığı görülmektedir.

![Process Injection 12](/assets/img/process-injection-teknikleri/img/procinj-12.png)

Yukarıdaki görselde 0x00400000 adresinde allocate edilen bellek alanını görmektesiniz.

Bellekte istediği lokasyondan **RWX** izinlerinde yer ayırma işlemi yapan malware, **WriteProcessMemory** API()’sini kullanarak yürütülebilir dosyayı ve section’larını 0x00400000 adresindeki ayrılan konuma kopyalar. Aşağıdaki görselde bu durum görülmektedir.

![Process Injection 13](/assets/img/process-injection-teknikleri/img/procinj-13.png)

Bu işlemlerden sonra malware, legal process’in PEB.ImageBaseAddress kısmına, artık zararlı içerikle dolu olan 0x00400000 adresini yazar.

![Process Injection 14](/assets/img/process-injection-teknikleri/img/procinj-14.png)

Yukarıdaki görselde de artık legal process’in PEB.ImageBaseAdress kısmında yazılı olan değerin 0x01000000’den artık içinde zararlı executable’ı barındıran 0x00400000 adresinin yazılı olduğunu görüyoruz. Yani kısacası malware, suspend halde olan legal process’in start adresini, bellekte legal process’in executable kısmına enjekte edilen zararlının start adresi ile değiştiriyor.

Bu işlemden sonra artık suspend durumda olan process’in thread’i zararlı kısma işaret etmektedir. Artık saldırganın yapması gereken tek şey suspend edilen thread’in **ResumeThread() **API’si ile suspend durumdan resume durumuna geçmesini sağlayıp enjekte edilen kodu (ya da executable’ı) çalıştırmasını izlemektir.

Ayrıca malware, zararlı executable’ı hedef işleme enjekte etmek için VirtualAllocEx() ve WriteProcessMemory() tekniklerinden kaçınmak amacıyla **NtMapViewSection**() API’sini kullanabilmektedir. Bu API’de Process Hollowing tekniğinde kullanılan API’lerden birisidir.

## 4. Process Doppelgänging

Popüler Code Injection tekniklerinden birisi olan Process Doppelgänging, ilk olarak 2017 yılında BlackHat’te enSilo şirketinde çalışan 2 güvenlik araştırmacısı tarafından açıklandı.

Process Doppelgänging Windows 10 dahil olmak üzere tüm Windows sürümlerinde başarıyla çalışmasından dolayı büyük bir öneme sahiptir. Process Hollowing ile benzerlik gösterse, Process Hollowing’den kesin olarak ayrılan yönleri vardır.

Process Doppelgänging, ilk ortaya çıktığı zamanlarda bir çok AV ürünü tarafından zor tespit edildiği için malware’lar tarafından sıkça kullanılmıştır..

### Ayrım Noktası?

Process Hollowing önce hedef işlemi başlatır, daha sonra unmap işlemini yapar ve zararlı kodu enjekte eder. Process Doppelgänging ise process başlamadan önce image’ın üzerine zararlı kodu yazmaktadır. En önemli ayrım noktaları ise bu tekniktir.

Process Doppelgänging, Windows NTFS işlemlerini kullanmaktadır. TFS dosyasının işlemlerine(oluşturma, silme, değiştirme) dayanan bir tekniktir. İşlemsel NTFS, diğer adıyla TxF, process’leri NTFS dosya sistemine entegre eder. Process Doppelgänging, kötü amaçlı kodu veya yazılımı gizlemek için belirli olan bu özellikleri kullanır. Process’in çalışma esnasında bir dosya oluşturduğumuzu ve içine yazma işlemi yaptığımızı düşünelim. Windows’un yapısı gereği process dosya üzerindeki işlemini bitirmeden veya kapatmadan, dosya diskte görünmeyecektir. İşte Process Doppelgänging’in avantajlarından birisi de bu yöntemdir.

Process Doppelgänging için NTFS işlemlerinde **4 adet basamak** vardır. Bunları şu şekilde sıralayabilir ve açıklayabiliriz:

**Transact**:
Bu aşamada legal process işlenir ve üzerine malicious uygulama yazılır. Bu işlemin alt aşamaları bulunmaktadır:

İlk olarak CreateTransaction() API’si kullanılarak yeni transaction(işlem) oluşturulur.
CreateFileTransacted() API’si ile işlem görmüş bir handle elde edilir. Bu handle gereken tüm dosya işlemleri için kullanılabilir.
Legal dosyanın üzerine WriteFile() API’si ile malicious içerik yazılır.

**Load**:
Bu aşamada ise 1. Aşamada üzerine yazma işlemi yapılarak değiştirilen dosyadan bir memory bölümü oluşturulur. NtCreateSection() API’si ile işlem yapılan dosyadan bir bir bölüm oluşturulur. Bu bölüm, zararlı dosyaya işaret edecektir.

**Rollback**:
Bu aşamada ise yapılan tüm değişiklikler geri alınır. Orijinal dosyayı diskte bırakır. RollbackTransaction() API’si ile bu işlemi gerçekleştirmektedir.

**Execution**:
Bu kısım Process Doppelgänging’in nasıl kaçamaklı, sahte ricatlı (geri çekilmeli) bir teknik olduğunu bize açıklayacaktır. İlk aşamadan itibaren, daha önce açılan bir process’i execute edebilecek, Windows XP’den beridir süregelen eski bir komut bulunmaktadır.

- İlk olarak process ve threadler, **NtCreateProcessEx**() ve **NtCreateThreadEx**() API’leri kullanılarak oluşturulur.
- Process parametreleri **RtlCreateProcessParameters**() API’si ile oluşturulur.
- **VirtualAllocEx**() API’si ve önceden kullanılan parametreler kullanılarak boş alan ayrılır.
- Daha sonra ise **NtResumeThread**() API’si kullanılarak ayrı bir process başlatılır.

Sonuç olarak, dosya içeriği geri alındıktan sonra bile process enjekte edilmiş bir halde başlayabilir. Bu nedenle de bir çok AV ürünü tarafından hiçbir sorun yokmuş gibi gözükecektir.

Örneğin, **mimikatz** normal bir şekilde başlatıldığında AV sistemleri bunu hemen tespit edebildiler. Fakat legal bir process’e, Process Doppelgänging tekniği ile enjekte edilip başladığında AV sistemleri bunu tespit edemedi.

Özet olarak yapmamız gereken şey, zararlı içeriğin tam yolunu vermek iken rollback işleminden sonra PE içeriğine sahip bir bölümü parametre olarak alan Zw/NtCreateProcessEx() API’si vardır. Bu API’yi kullanarak dosyasız bir şekilde injection yapmış gibi oluruz ve işletim sistemi sadece dosya kapandığında değişikleri farkedeceği için bunu algılayamaz.

![Process Injection 15](/assets/img/process-injection-teknikleri/img/procinj-15.png)

Yukarıdaki görselde **NtCreateProcessEx**() API’sinin yapısını görmektesiniz.

Bu teknik tamamen gizli değildir, ve tespiti de snapshot yöntemleri kullanılarak ve karşılaştırmalar ile yapılabilir. Sonuçta Remote bir thread oluşturduğu için AV sistemlerini tetikleyebilir.

2017 yılında ortaya çıkmış bir teknik olarak günümüzdeki AV sistemlerinin bu tekniği tespit edebileceğini söyleyebiliriz.

## 5. Atom Bombing Injection

Process Doppelgänging kod enjeksiyon tekniğinin ardından Atom Bombing Injection tekniğine geliyoruz. Atom Bombing tekniğini, yine adını az önceki Process Doppelgänging tekniğinden de hatırlayacağınız enSilo şirketinde çalışan güvenlik araştırmacıları bulmuştur.

Atom Bombing tekniği Windows’un tüm sürümlerinde çalışmaktadır. Bir bug veya zafiyet değildir, aksine Windows’un doğası gereği ortaya çıkan bir tekniktir. Bundan dolayı da herhangi bir yama söz konusu değildir.

Ortaya çıkmasından sonra artık AV ürünleri tarafından tespiti de yapılabilmektedir. Bu teknik ise 2018 yılında ortaya çıkmıştır.

Atom Bombing, adını Windows’un atom tablolarından almaktadır. Atom tabloları, process’ler arasında paylaşılan sistem belleğini kullanarak veri paylaşımı/değişimi gibi işlemleri gerçekleştirir. Atom tablolarını Windows şu şekilde tanımlamaktadır:

> “Atom tablosu, strings’leri ve karşılık gelen tanımlayıcıları saklayan sistem tanımlı bir tablodur. Bir uygulama, bir string’i atom tablosuna yerleştirir ve o string’e erişmek için 16 bitlik bir tam sayı alır. Atom tablosuna yerleştirilen bu string’e atom adı verilir.”

Bu açıklamadan yola çıkarak bu tekniğin arkasında yatan planı da az çok kurguluyor gibiyiz. Malware process’i legal bir string yerine zararlı kodu atom olarak oluşturuyor ve hedef olan legal process’in bu oluşturulan zararlı atom’u yükleyerek çalıştırmasını sağlıyor.

![Process Injection 16](/assets/img/process-injection-teknikleri/img/procinj-16.jpg)

Yukarıdaki görselde Atom Bombing tekniğinin çalışma yapısı gösterilmiştir. Atom Bombing tekniğinin nasıl gerçekleştiğini anlamak için teknik olarak açıklayalım:

Malware (zararlı process), GlobalAddAtom() ile zararlı kodu string biçiminde atom tablosuna yerleştirir. Atom tablosu sistemde çalışan her process tarafından erişilebilir durumdadır.

**APC**(Asynchronous Procedure Call) kullanarak, **GlobalGetAtomName**() ile zararlı kodu atom tablosundan legal process’in bellek alanına kopyalar. APC kullanıldığı için bu teknik alterable durumda olan herhangi bir process’in thread’i tarafından yapılabilir.

Daha sonrasında ise sistemi yeni bir executable bellek allocate etmeye zorlar. Bellek allocate işleminden sonra zararlı kodu ayrılan bellek alanına kopyalar ve çalıştırır.

Yukarıdaki adımlarda da teknik detayını açıkladık. Bu tekniğin hakkında önemli olan bazı kısımlar mevcuttur.

Atom Bombing, bir Privilege Escalation(yetki yükseltme) sağlamaz. AV ürünleri tarafından beyaz listede olan process’leri hedef alarak bypass yapar.

Process’in özel verilerine erişmeye imkan sağlar. Yani belirli bir process’i hedef aldığınızda, (Web Browser gibi), web içeriğini değiştirebilir, tarayıcıdaki parolalara erişebilirsiniz. Ekran görüntüsü alabilirsiniz. Bu tamamen hangi process’i hedef aldığınıza bağlı olmak ile birlikte saldırgana çok geniş imkanlar sunmaktadır.

Gerçek bir örnek vermek gerekirse Dridex bankacılık trojanı, bu teknik ortaya çıktıktan sonra Atom Bombing’i kullanmaya başlamıştır. Bilin bakalım neden 🙂

Atom Bombing tekniğinin APC’yi kullandığını tekrar hatırlayalım. APC hakkında çok az bilgi mevcut. Ama örneklendirmek gerekirse normal işlevinde ilerleyen başka bir process’in thread’ini zararlı kodu çalıştırmak için Malware’lar tarafından kullanılmaktadır.


Sizlere en çok ses getiren, ilginç ve işe yarar Process Injection tekniklerini anlattım. Artık svchost.exe’nin legal bir sistem uygulaması olduğunu ve zararlı yazılımların da uğrak yeri olduğunu biliyoruz 🙂

Başka bir teknik makalede görüşmek üzere…

## Yararlanılan Kaynaklar

[1] https://www.elastic.co/blog/ten-process-injection-techniques-technical-survey-common-and-trending-process

[2] https://www.deepinstinct.com/2019/09/15/malware-evasion-techniques-part-1-process-injection-and-manipulation

[3] https://kaganisildak.com/2019/02/10/process-doppelganging/
