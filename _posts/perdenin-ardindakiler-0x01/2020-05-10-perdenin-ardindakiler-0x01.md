---
title: Perdenin Ardındakiler 0x01 - Android Zararlısını Işığa Kavuşturmak
date: 2020-05-10 22:59 +03:00
tags: [android malware, android malware analysis, android malware deobfuscation, droiddream, droidream malware analysis, malware deobfuscation]
description: Perdenin Ardındakiler serisinin 0x01 adresinde Android zararlısını ışığa kavuşturuyoruz.
image: "/assets/img/perdenin-ardindakiler-0x01/img/cover.jpg"
---

Selamlar dostlar...

Zararlı yazılım analiz raporları her gün, her hafta ve her ay karşımıza çıkmakta. Raporların içeriğinde de sık sık “**Teknik analizler sonucunda gizlenmiş IP adresi karşımıza çıkmakta**” veya  “**Decryption işlemi sonucunda zararlı URL bulmaktayız**” gibi ifadeleri görüyoruz. E tamam decryption işlemi yaptın da ulaştın obfuscate edilmiş URL’e ama nasıl yaptın? Bu sorunu baz aldım ve piyasada decryption işleminin nasıl yapıldığına dair çok fazla içerik bulunmadığını gördüm. Bu eksiklikten yola çıkarak analizini yapıp deobfuscate ettiğim zararlı yazılımları, nasıl deobfuscate ettiğime dair bilgilendiri, eğitici ve ileri seviye teknik bir seri başlatıyorum. Serinin adı da “**Perdenin Ardındakiler!**”

~~(Evet isim aklıma geldikten sonra araştırma yaptım, böyle bir müzik grubu olduğunu öğrenmiş oldum, olayın müzikle pek alakası yok 🙂 )~~

Perdenin Ardındakiler serisinin ilk içeriği olan 0x01’de sizlere çook uzun süre ortaya çıkmış olan Android zararlısı **DroidDream**’in içerisinde yer alan obfuscate edilmiş, yani plaintext olarak okuyamadığımız ve belirli bir decrypt rutininden sonra dinamik olarak ortaya çıkan komuta kontrol sunucusunun statik olarak nasıl deobfuscate edilip bizlere ışıklar içinde görüneceğini anlatacağım. Serinin ilk içeriği olması sebebiyle biraz basitten başlayıp işin mantığını kavramaya çalışacağız. 

### Giriş
DroidDream zararlısı içerisinde birkaç farklı yapı barındırmakta. Android’in bir zafiyetini kullanarak sistemi exploit etmekte ve uzaktan ilgili binary’ye bağlantı sağlandığında ise bağlanan saldırgana ROOT yetkisi vermektedir. Bu seride zararlılara çok fazla değinmeden ön izlenim geçeceğim çünkü konumuz Malware Analiz Raporu değil, o malware içerisindeki gizlenmiş indikatörleri ortaya çıkarmak. 

Öncelikle bir dex decompiler’a ihtiyacımız var. Bu da Desktop Zararlı Analizleri sırasındaki IDA’nın yerini alacak olan **Jadx**’ten başka bir tool olamaz tabi! Aslında IDA ile de decompile edebilirdik sonuçta dalvik bytecodelarını da decompile edebiliyor ama Android zararlıları için pek de amaca uygun değil.  

Şimdi ne yapacağız? Jadx ile direk APK dosyamızı açacağız. Tebrikler, kısmen doğru cevap verdiniz. Tam doğru olanı ise Jadx’i komut satırı üzerinden `--show-bad-code` parametresi ile başlatmak. Sebebi ise Jadx’in decompile işlemi yaparken bazı metodları tam olarak ayırştırıp Java kodu olarak sunamamasından dolayı bu parametre ile başlatıyoruz ve diyoruz ki “**Sen decompile edebildiğin kadarını göster bize**”. Ve karşımıza yine de güzel bir Java metodu çıkmış oluyor. Bu işlemi yapmaz isek karşılaşacağımı tablo aşağıdaki gibi olmaktadır.

![pa01-1](/assets/img/perdenin-ardindakiler-0x01/img/pa01-1.png)

### Decrypt Rutinin Tespiti

Giriş kısmında zararlı APK’mızı show-bad-code modunda JADX’te decompile etmiştik. Şimdi ise neyi deobfuscate edeceğimizi bulmamız gerekiyor. Bunun için en iyi yol uygulamanın çalışma hiyerarşisine göre trace etmek. Yani main sınıfından başlayıp “ne nereye gidiyor, hangi parametreleri yolluyor?” sorusunu kendinize sorup çözüm aramaya çalışırsanız karşınızda bir anda aşağıdaki görselde olduğu gibi Xor veya matematiksel işlemlerin yoğun olduğu bir kod parçası belirecektir.  

![pa01-2](/assets/img/perdenin-ardindakiler-0x01/img/pa01-2.png)

Crypt metodunu incelediğimizde buffer adında byte tipinde bir diziyi parametre olmakta almakta. Daha sonra yaptığı işlem ise buffer dizisinin boyu kadar i değişkenini bir bir artırıp dizinin i. indisindeki değer il KEYVALUE dizisinin pos indisindeki değeri xor işlemine tabi tutmak. İç içe for döngüsü kullanılabilirdi fakat saldırgan pos değerini **pos++** işlemi ile artırmayı tercih etmiş. Hemen aşağısında bulunan if döngüsü bu konu için bizi alakadar etmemektedir. 

**KEYVALUE** adındaki byte dizisi nerede diye sorarsanız saldırgan bunu her yerde tanımlamış olabilir. Ama bizim örneğimizde sınıfın en üstünde tanımlanmış durumda ve içerisinde barındırdığı değerler de ilgi çekici. 

![pa01-3](/assets/img/perdenin-ardindakiler-0x01/img/pa01-3.png)

Bir string dizisini **getBytes**() metodu ile baytlara çevirmiş ve sonucunda da byte tipinde KEYVALUE adındaki diziye tanımlamış durumda. 

### Obfuscate Edilmiş Verinin Tespiti

Decrypt rutinini tespit etmiştik. Ve içerisine bir bayt dizisi almakta idi. Yani çözeceği veri bir byte dizisinin içinde olmalı veyahut başka veri tipinde dizi olup, byte veri tipine zorlanmış olmalı. Uygulamayı yine çalıştırma hiyerarşisine göre analiz ettiğimizde karşımıza tam da aradığımız şekilde bir bayt dizisi çıkmakta. 

![pa01-4](/assets/img/perdenin-ardindakiler-0x01/img/pa01-4.png)

**bArr** adındaki byte dizisini bulduk ve obfuscate edilmiş veri olduğundan şüphelendik diyelim. Peki bundan sonra ne yapacağız? Cevap basit. bArr dizisini trace etmek, yani izini sürmek. Kapsamı bArr dizisi ile daralttığımız için işimiz gerçekten de kolay oluyor. 

### Trace Trace Trace!

Saldırgan 45 boylu bArr adındaki byte veri tipindeki diziye görselden de gördüğünüz üzere atamalar yapmış bulunmakta. Atamaların sonuna geldiğimizde ise **u = bArr; **şeklinde bir referans ataması olduğunu görüyoruz. Java’da referans atamasının karşılığını C’de pointer geçme olarak aklınızda tutabilirsiniz. Saldırgan burada izini kaybettirmeye çalışmış. Artık trace etmemiz gereken değişken udeğişkeni oluyor. Ve biraz daha analize devam ettiğimizde ise onCreate() metodunun üst kısımlarında aradığımızı görebilmekteyiz. 

![pa01-5](/assets/img/perdenin-ardindakiler-0x01/img/pa01-5.png)

Gördüğünüz üzere obfuscate edildiğinden şüphe ettiğimiz verilerin tutulup referansının u değişkenine atandığı dizinin bir kopyasını alıp byte veri tipinde c adında başka bir diziye atamakta. Bunun hemen altında ise decrypt rutini olan crypt metoduna c’yi parametre olarak geçmekte. Bunun sonucunda ise artık obfuscate edildiğinden şüphe duyduğumuz veri çözümlenmekte. 

Dikkat ederseniz şu ana kadar hep “obfuscate edildiğinden şüphe duyduğumuz veri” cümlesini kullandım. Çünkü henüz emin değiliz ve bunun gibi bir çok dizi olabilirdi. Saldırgan analistin işini zorlaştırmak için ne taklalar atıyor bir bilseniz! 

### Emin Olma Ve C# Zamanı

Şimdiye kadar yaptığımız şey aslında tüm uygulamayı trace ettikten sonra decrypt rutini ve akabinde obfuscate edilen verinin ne yollarla decrypt rutinine yollandığının algoritmasını kafamızda çözümlemekti. Şimdi ise sıra teorilerimizi kanıtlamaya ve biraz da kod yazmaya geldi. 

Yapacağımız ilk şey tanımlamalar olmakta. İlk olarak Xor anahtarlarını tanımlıyoruz. Daha sonra ise obfuscate edilmiş bArr dizisini ve tüm içeriklerini tanımlıyoruz.

```csharp
    string KEYVALUE = "6^)(9-p35a%3#4S!4S0)$Yt%^&5(j.g^&o(*0)$Yv!#O@6GpG@=+3j.&6^)(0-=1";
                byte[] anahtar = Encoding.UTF8.GetBytes(KEYVALUE); //Stringi byte olarak alıyoruz.
                byte[] bArr = new byte[45];
                bArr[0] = 94;
                bArr[1] = 42;
                bArr[2] = 93;
                bArr[3] = 88;
                bArr[4] = 3;
                bArr[5] = 2;
                bArr[6] = 95;
                bArr[7] = 2;
                bArr[8] = 13;
                bArr[9] = 85;
                bArr[10] = 11;
                bArr[11] = 2;
                bArr[12] = 19;
                bArr[13] = 1;
                bArr[14] = 125;
                bArr[15] = 19;
                bArr[17] = 102;
                bArr[18] = 30;
                bArr[19] = 24;
                bArr[20] = 19;
                bArr[21] = 99;
                bArr[22] = 76;
                bArr[23] = 21;
                bArr[24] = 102;
                bArr[25] = 22;
                bArr[26] = 26;
                bArr[27] = 111;
                bArr[28] = 39;
                bArr[29] = 125;
                bArr[30] = 2;
                bArr[31] = 44;
                bArr[32] = 80;
                bArr[33] = 10;
                bArr[34] = 90;
                bArr[35] = 5;
                bArr[36] = 119;
                bArr[37] = 100;
                bArr[38] = 119;
                bArr[39] = 60;
                bArr[40] = 4;
                bArr[41] = 87;
                bArr[42] = 79;
                bArr[43] = 42;
                bArr[44] = 52;
```
Dikkat edilmesi gereken noktalardan birisi ise saldırgan Java’daki getBytes() metodunu default biçiminde kullanmış. Yani UTF8’den baytlara çevirmiş. Bizde C#’da bunu belirterek yapıyoruz. Java’daki gibi direk bir kullanımı C#’da bulunmuyor.


Decrypt adındaki metodumuzu yazmaya başlıyoruz. 

```csharp
  public void decrypt(byte[] dizi, byte[] anahtar)
            {
                int i;
                for(i = 0; i < dizi.Length; i++)
                {
                    dizi[i] = (byte)(dizi[i] ^ anahtar[i]);
                }
                string urlPlain = Encoding.UTF8.GetString(dizi); 
                Console.Write("URL: " + urlPlain)
            }
```

Kod parçasında görüldüğü üzere pos değişkenini hiç hesaba katmadık. Yaptığımız tek şey dizi adındaki dizi ve anahtar dizisini de karşılık gelen i. indisteki değer ile xor işlemine tabi tutmak oldu.  Dizinin içerisinde bulunan değer bayt türünde olduğundan dolayı baytları UTF8 biçimindeki stringlere, C#’ın GetString() metodu sayesinde dönüştürüp, ekrana çıktılama yaptık ve karşımıza gelen sonuç, yüz güldüren cinstendi 🙂

![pa01-6](/assets/img/perdenin-ardindakiler-0x01/img/pa01-6.png)

Kodun tam halini buradan görebilirsiniz. Github Gist üzerinden de indirebilirsiniz. 

Perdenin Ardındakiler serisinin 0x01 bölümünde sizlere Android zararlısı Droid Dream’e decryption kodu yazarak tün indikatörleri ışığa kavuşturmayı anlatttım. Herhangi bir sorunuz, öneriniz olursa yorumlardan bana bildirebilirsiniz. Bu arada pratik yapmak için de DroidDream zararlısının hashini buraya bırakıyorum. “Hep sayfanın en üstüne koyarlar sen neden en alta hash koyuyorsun” diye merak ettiyseniz de konuyu tam olarak dinleyip, daha sonra pratik yapmanızın daha doğru olduğunu düşünüyorum şeklinde bir cevap verebilirim 🙂 

Serinin bir sonraki bölümünde görüşmek üzere, sağlıcakla kalın…

`DroidDream MD5 HASH: ecad34c72d2388aafec0a1352bff2dd9 `