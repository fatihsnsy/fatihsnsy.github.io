---
title: Matrix Ransomware Analizi (Giriş Seviye)
date: 2019-12-30 18:20 +03:00
tags: [malware, malware analysis, matrix ransomware]
description: Matrix Ransomware, dünyada bir çok türevi bulunan bir fidye yazılımıdır. Bizim incelediğimiz türev başka hiçbir yerde incelenmemiştir. “.eman” uzantısında dosyaları kriptolamaktadır.
image: "/assets/img/matrix-ransomware-analizi/img/cover.png"
---

Selamlar herkese. Matrix Ransomware analiz raporumu sizlere paylaşıyorum. Bu benim Malware Analistliği kariyerimde yazdığım ilk raporum ünvanını taşıyor. Diğerlerini de belirli aralıklarla sizlerle paylaşmayı düşünüyorum :)

Matrix Ransomware, dünyada bir çok türevi bulunan bir fidye yazılımıdır. Bizim incelediğimiz türev başka hiçbir yerde incelenmemiştir. “.eman” uzantısında dosyaları kriptolamaktadır.

|  SHA-256 |MD5   |
| :------------: | :------------: | 
| 242713ef2f372f0d39ca8f01bd09c9f99bcfe850e156621c023dd9e0bfb9bd95  | A93BD199D34D21CC9102600C6CE782CF  |

### İlk Bakış

**MatrixRansomware.exe** isimli dosyanın orijinal adı **eman33.exe** olarak internet ortamında dolaşıyor. MatrixRansomware’ın farklı farklı türevleri mevcut ve sürekli gelişim halinde. Ben analizde anlaşılması adına ismini düzenledim. Şimdi ilk olarak bu ransomware’ın yeteneklerini görmek için gelin bir laboratuvar ortamımızda test edelim.

Öncelikle MatrixRansomware.exe isimli dosyamızı kullanıcı yetkilerinde çalıştırıyoruz.

![Matrix Ransomware İlk Bakış](/assets/img/matrix-ransomware-analizi/img/matrix-1.png)

Çalıştırdığımızda masaüstümüzde dosyalar oluştuğunu ve iki adet komut satırı ekranı açıldığını görüyoruz. Laboratuvar ortamımız ağa bağlı.

![Matrix Ransomware İlk Açılış](/assets/img/matrix-ransomware-analizi/img/matrix-2.png)

Görüldüğü üzere öne çıkan komut satırının en üstünde SHARESSCAN ibaresi yer alıyor. Yani şuan paylaşılan klasörleri geziyor. Ağımızdaki diğer bilgisayarlara ulaşmak için IP bloğumuzdan hareket ediyor. Diğer Command Line penceremizde ise bazı bilgilerin yer aldığını görüyoruz.

![Matrix Ransomware İlk Tetikleme](/assets/img/matrix-ransomware-analizi/img/matrix-3.png)

Integrity seviyesini numaralandırdığını görüyoruz ve şuan da 3 seviyesinde. Yani program USER yetkilerinde çalışıyor. 

LDRIVES kısmına baktığımızda ise C: dizinini direk olarak hedef aldığını görüyoruz. Ve dosyalarımızı kriptolarken anlık olarak sistemimizin performansına göre kriptolama hızını gösteren bir ibare de mevcut.

![Matrix Ransomware Kullanıcı Yetkilerinde Oluşturulan Dosyalar](/assets/img/matrix-ransomware-analizi/img/matrix-4.png)

Masaüstüne kullanıcı yetkilerinde iken oluşturduğu dosyalar görselde mevcuttur.

![Matrix Ransomware elog Dosya İçeriği](/assets/img/matrix-ransomware-analizi/img/matrix-5.png)

**elog_0940325A9D85363B.txt** dosyasını açtığımızda ise logları görüyoruz. Ransomware’ımız kullanıcı yetkilerinde olduğu için dosyalara erişememiş ve bir şifreleme yapamamış.

Şimdi ise Admin yetkilerinde malware’imizi çalıştıralım.

![Matrix Ransomware Admin Yetkileri](/assets/img/matrix-ransomware-analizi/img/matrix-6.png)

Yine aynı işlemleri yaptı ve komut ekranı kapandı. Kapanırken ise işlemin tamamlandığına dair bir bilgi verdi. Masaüstünde yine birkaç dosya oluşturdu ama bu sefer dosyalarımızı kriptolamıştı ve **#README_EMAN#.rtf **adında bir dosya oluşturmuştu.

![Matrix Ransomware Admin Yetkilerinde Oluşturulan Dosyalar](/assets/img/matrix-ransomware-analizi/img/matrix-7.png)

Dosyayı açtığımızda bize, dosyalarımızı nasıl kurtaracağımıza dair bir yönerge gösteriyordu. Saldırgan ile nasıl iletişime geçileceğinin bilgisi mevcuttu. Hatta güven sağlamak için saldırganlara yollayacağımız 3 adet kriptolanmış veriyi decrypt edip geri vereceklerini söylüyorlardı.

İlk bakışımız bu şekilde sona erdi. Zararlı yazılımın ne yaptığı hakkında genel bir bilgi sahibi olduk. Şimdi ise bir sonraki adımımıza, yani STATİK ANALİZE geçelim.

### Statik Analiz

IDA ile statik analizimizi yapıyoruz. 

![IDA İlk Bakış](/assets/img/matrix-ransomware-analizi/img/matrix-8.png)

Herhangi bir main fonksiyonunun olmadığını görüyoruz. Aslında var ama malware’ı geliştiren kişiler analizi zorlaştırmak için adını değiştirmiş veya malware’i packlemiş olabilirler. Pack detection toolları ile tarama yapıldığında da herhangi bir pack işleminin olmadığını gördük. Malware’ın **3700**’den fazla fonksiyonu mevcut.

![Matrix Ransomware Hex Ve String](/assets/img/matrix-ransomware-analizi/img/matrix-9.png)

Statik analize devam ettiğimizde ise Strings ve Hex Görüntüsü pencerelerine göz attığımızda ise bazı kritik bulgulara ulaşıyoruz. Bunlar indikatörlere ulaşmamızı sağlayacak.

Bulunan Kritik Ve Önemli Bulgular

- Varolan işlemlerin diline bakarak kullanıcının sisteminin dilini öğrenmeye çalışıyor.
- WSAStartup, bind, accept, connect gibi parametreler Soket Programlama’dan tanıdık geliyor. Yani bu malware bir uzak sunucu ile bağlantı kuruyor.
- Listen, accept ve send parametleri de mevcut. Hem veri alıyor, hem de veri gönderimi yapıyor.
- Basic seviyede Proxy-Authorization mekanizması kullanıyor.

Şimdi ise bu malware’ın neleri import ettiğine bakıyoruz.

![Matrix Ransomware Importlar](/assets/img/matrix-ransomware-analizi/img/matrix-10.png)

**Kernel32** ve **wsock32** gibi kritik kütüphaneleri kullanıyor. Wsock32 kütüphanesi import etmesinden de anlayabileceğimiz üzere bir sunucu ile iletişimde olduğu bulgularımızı doğrulamış olduk. Kütüphanelerden ise kullandığı fonksiyonlara göz atalım.

**DeleteFileW** fonksiyonunu dahil etmiş. Yani mesajda bize verdiği **7 günlük** sürenin sonunda iletişime geçilmezse gerçekten de şifrelenmiş dosyaları siliyor.

**CreateMutexW** fonksiyonu ile de Mutex mekanizması kullanıyor. Yani Mutex’i C dilinde yazıp çalıştırırsak, malware “Mutex var ise çalışma” prensibinden dolayı bir daha çalıştırırsak sistemi şifrelemeyecek.

**CryptGenRandom** fonksiyonu ile de bir arabelleği kriptografik baytlar ile dolduruyor. 

### Derinlere İnelim

IDA ile biraz daha ayrıntıya iniyoruz. **FindFirstFileW** fonksiyonu ile belirli bir uzantıdaki dosyaları aradığını anlıyoruz. Biraz daha derin analiz yapıldığında ransomware’in kriptoladığı uzantılar aşağıdaki gibidir:

- .cmd
- .bat
- .vbs
- .ink
- .rtf
- .bmp
- .tmp

32-bit ve 64-bit sistemler için ayrı ayrı işlemler yapıyor.

Disassembly kodlarında** log.txt**’nin de oluşturulduğunu görüyoruz.

**nw.exe** adlı programın –n parametresi ile bir işlem yapıyor ve nw.exe ağ ve Javascript kütüphaneleri içeren bir program olarak biliniyor. Genelde adware yazılımlar için kullanılıyor.

“%d.%d.%d.%d” ile aynı ağda bulunan diğer sistemleri tarayarak IP adreslerini CMD ekranında kullanıcıya gösteriyor.

80, 443, 21 gibi portlar da zararlıya tanımlanmış.

Malware, bulaşma zamanlarını hesaplayarak 7 günlük silme sürecini başlatıyor. Bilgisayarın ve mevcut kullanıcının adını da tanımlama yapmak amacıyla alıyor. flcCipherRSA ibaresinden dosyaların RSA ile kriptolandığını anlıyoruz.

Ayrıca bir URL’e HTTP verisi gönderildiğini tespit ettim. HTTP başlıklarını ayarlıyor. “**set-cookie, Content-Lenght, Content-Type**” gibi standart HTTP Head bilgilerini de yolluyor. Cookie set etmesinden bir kullanıcı tanımlama mekanizması yaptığı ihtimali güçleniyor.

Komuta kontrol sunucusuna (URL’e) yolladığı parametreler ise dikkat çekici:

- /addrecord.php?apikey=
- &compuser=
- &sid=
- &phase=

Ardı ardına birleştirme operatörünü kullanarak (&) bir istek gönderiyor. Burada da her kullanıcıyı tanımlamak için bir mekanizma oluşturduğu ihtimalini kanıtlamış oluyoruz.

Ayrıca user-agent olarak “**Mozilla/4.0 (compatible; Synapse)**” ayarlıyor.

### CFF Explorer

Daha iyi analizle yapmak için CFF Explorer’da da analiz ediyoruz.

![CFF Explorer İlk Bakış](/assets/img/matrix-ransomware-analizi/img/matrix-11.png)

Section Headers’lara baktığımızda standart MZ başlığı yerine **MZP** başlığını görüyoruz. Yani bu ransomware **Pascal** ile yazılmış!

Analizin geçen kısımlarında ise Borland Delphi 3.0 IDE’sini kullandığını da tespit etmiştik.

Import Directory’ye geldiğimizde ise malware’ın etkisini ciddi derecede görebiliyoruz. Import ettiği dll’ler ve fonksiyonlarında kritik seviyede bulunanlar şu şekilde:

- GetSystemInfo
- GetProcAddress
- CreateThread
- CreateMutexW
- GetVolumeInformationW
- GetDiskFreeSpace
- DeleteFileW
- gethostname
- gethostbyname

Yukarıdaki fonksiyonlara baktığımızda;

- sistemin bilgilerini alabildiği,
- bir işlemin adresini alabildiği,
- bir iş parçacığı oluşturabildiği,
- bir Mutex nesnesi oluşturabildiği,
- Dosya sistemi ve hacmine erişebildiği,
- Boş alana erişebildiği,
- Ve en kritiği olan DOSYALARI SİLEBİLDİĞİ gözlemlenmiştir.

![Matrix Ransomware Resource](/assets/img/matrix-ransomware-analizi/img/matrix-12.png)

Dosyanın içindeki kaynaklarda ise iki kısmın olduğu tespit edilmiştir. Strings Tables ve RCData. Strings Tables’da ayların isimleri vs. olduğu gözlemlenmiştir. RCData kısmında ise farklı farklı kaynaklar olduğu görülmektedir.

Fakat erişilmeye çalışıldığında bozuk bir ASCII çıktısı bizi karşılamaktadır. Buradan yapılan çıkarımla **RCData** kaynağının içinde çok önemli bilgilerin olduğu ve okunamaması için de özel bir şifreleme türü ile şifrelendiği ortaya çıkmaktadır.

### Dinamik Analize Geçelim

Statik analizde bir çok önemli veriyi ele geçirmeyi başardık. Şimdi ise bize kritik bilgileri verecek olan, bir Malware Analizinin olmazsa olmazı Dinamik Analize geçiyoruz. Şimdi bulduğumuz kritik fonksiyonlara breakpoint koyuyoruz ve analiz etmeye başlıyoruz.

![Matrix Ransomware x64dbg](/assets/img/matrix-ransomware-analizi/img/matrix-13.png)

Hepsine breakpoint koyduk. GetVolumeInformation fonksiyonuna geldiğinde registerlarda “C:\” ifadesini görüyoruz. Yani direk olarak C dizinini hedef almış. Ama genel olarak Memory Map’e de baktığımız zaman bellekte de analizi zorlaştırmak için bir şifreleme yaptığı ortaya çıkıyor.

DeleteFileW fonksiyonuna gelince ise `"C:\Program Files\Windows Mail\wabmig.exe"` dosyasına ulaştığını görüyoruz. Windows Mail’i de hedef alıyor. 

![Matrix Ransomware Command Line Çıktıları](/assets/img/matrix-ransomware-analizi/img/matrix-14.png)

Bu arada masaüstünde iki tane .exe ve bir adet .bat dosyası oluşturduğunu gözlemiyoruz. “.bat” dosyasına baktığımızda ise bazı komutların yer aldığını görüyoruz. Username bilgisine ulaştığını, ve sahipliğini üzerine aldığını, ve başka dizinlere atladığını görüyoruz.

`FOR /F "UseBackQ Tokens=3,6 delims=: " %%I IN ("umsBf6bj.exe -accepteula %FN% -nobanner") DO (umsBf6bj.exe -accepteula -c %%J -y -p %%I -nobanner)`

Yukarıda kod diziminde ise masaüstünde oluşturduğu .exe dosyasını çalıştırıp parameterler giriyor. EULA’yı da kabul ettiriyor her seferinde. Sysinternals Handle Viewer programının kullanımı otomatikleştirmiş.

**umsBf6bj.exe** dosyasının UPX ile packlendiğini de ortaya çıkarttık. Şimdi unpack yapıp analiz edelim.

![Matrix Ransomware Gizli Dosya](/assets/img/matrix-ransomware-analizi/img/matrix-15.png)

Görüldüğü üzere Multiple koruma yöntemi uygulanmış. Unpack işlemimizi başarıyla gerçekleştirdik. Artık tüm fonksiyonları görebiliriz.

umsBf6bj.exe uygulaması, tarihleri alam ve ayarlama, kayıt defteri verilerini okuma, oluşturma ve silme işlemlerini, dosya oluşturma, silme, kriptolama işlemlerini, pointerları encrypt ve decryptleme işlemlerini, loglama işlemlerini yerine getiriyor.

**NWcxwNQL.exe** dosyayı ise MatrixRansomware.exe’nin kopyası. Fakat aynı ağ üzerinde bulunan diğer sistemleri de tarıyor. Eğer paylaşılan dosya bulursa onu da kriptoluyor.

umsBf6bj.exe dosyasının Resource’larına baktığımızda Sysinternals Handle Viewer programını görüyoruz. Dosyalara erişebilmek ve sahibini görüntülemek için Windows’un kendi uygulaması kullanılmış.

Wireshark ile detaylı ağ analizi yaptığımızda ise **eman.mygoodsday[.]org** komuta kontrol sunucusu bulunmuştur. Fakat artık böyle bir komuta kontrol sunucusunun mevcut olmadığı tespit edilmiştir.

### Sonsöz

MatrixRansomware’ı olabildiğince analiz etmeye çalıştım. Resource’larda ve memory’de şifreleme kullanıldığı için bazı önemli verilere ulaşamadım. Bu analiz raporu ilerleyen günlerde güncellenecektir.

