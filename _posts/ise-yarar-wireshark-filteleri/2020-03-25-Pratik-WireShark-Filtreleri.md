---
title: Pratik Wireshark 🦈 Filtreleri 
date: 2020-03-25 12:05 +03:00
tags: [wireshark filters, wireshark filtreleri, wireshark for network analysis, wireshark for infected traffic, wireshark ctf]
description: İşinize en çok yarayacak 20 WireShark filtresini sıraladım. Keyifli okumalar...
image: "img/cover.jpg"
category: Genel
---

Malware analistlerin C&C sunucuları, Network ve Sistem Adminlerin hataları tespit etmekte kullandığı popüler ve alanındaki en iyi paket analiz aracı Wireshark’ı hepimiz biliyoruz. Kimimiz 100.000 paketi tek tek inceliyor, kimimiz ise aradığına FİLTRE adı verilen efsanevi özellik ile direk nokta atışı yapıyor. Tabi biz ilk seçenek için yazmadık bu yazıyı 🙂 Network paket analizi yaparken aradığınızı elinizle koymuş gibi bulmanız için **İşinize Yarayacak 20 Wireshark Filtresini** sizler için derledim.

1) `ip.addr == 192.168.1.1`
Bu filtremiz Source veya Destination’da ilgili IP olan paketleri bizlere sunuyor.

 

2) `ip.addr == 192.168.1.1 && ip.addr == 10.0.0.3`

Bahsi geçen iki IP adresini VE bağlacı ile bağlamış. Yani Source veya Destination’da bu ikisi olmak zorunda. Hangisi Destination, hangisi Source önemli değil.

 

3) `ip.addr == 192.168.0.0/24`

192.168.0.x adresindeki 255 IP adresinden birisi paketlerde var ise, ilgili paketi önümüze getirir.

 

4)` ip.src == 192.168.1.16 && ip.dest == 10.0.0.12`

Source’u 192.168.1.16, Destination’ı 10.0.0.12 olan paketleri filtreler.

 

5) `ip.addr != 192.168.1.16`

İlgili IP adresinin olmadığı tüm paketleri filtreler.

 

6) `tcp.port == 8080`

8080 portuna bağlantı sağlayan paketleri filtreler.

 

7) `tcp.port in {443 80 8443}`

Birden fazla TCP portunun bağlantı yapıldığı paket arıyorsanız, kullanışlı bir filtredir.

 

8) `tcp.flags & 0x02`

TP bayrağı 0x02 oan paketleri filtreler. 0x02, SYN bitine karşılık gelmektedir. Aynı şekilde tcp.flags.syn == 1 filtresi de aynı anlama gelmektedir. Ayrıca bu filtre hem SYN, hemde SYN/ACK paketlerini gösterecektir. Sadece SYN paketlerini göstermek istiyorsanız da tcp.flags.syn == 1 && tcp.flags.ack == 0 filtresini kullanmanız gerekir.

 

9) `tcp contains FLAG`

TCP paketlerinin içerisinde FLAG string’i olan paketi filtreler. Özellikle CTF yarışmalarında işimize yarayacağı aşikar 🙂

 

10)` http.request.uri == “https://fatihsensoy.com/”`

Spesifik bir web sitesine gönderilen veya alınan paketleri filtrelemenize olanak sağlar.

 

11) `http.host matches “microsoft\.(org|com|net|tr)”`

Microsoft adının com, org, net veya tr domainleri ile alakalı giden veya gelen bir paket var ise filtreler.

 

12) `http.request.method == “POST”`

POST isteğinde bulunan tüm paketleri filtreler.

 

13) `http.host == “www.google.com”`

Belirtilen URL’in host olduğu paketleri bize sunar. Dikkat edilmesi gereken bir nokta mevcuttur. URL tam olarak yazılmalıdır. Yani web sitesinin başında www var ise eksik bırakılması halinde doğru filtreleme yapılmayacaktır.

 

14) `http.response.code == 200`

Yapılan Request sonucu dönen Response kodu 200 (yani başarılı bağlantı) olan paketleri filtreler. Aynı şekilde 404 (bulunamadı), 403 (yetkisiz erişim/yasak) gibi tüm HTTP response kodlarını da filtreleyebilirsiniz.

 

15) `http.content.type == “audio/mpeg”`

HTTP paketlerinin içerisinde mpeg türündeki ses dosyaları bulunan paketleri filtreler. Başka içerikleri de kolay bir şekilde filtreleyebilirsiniz. Örneğin “image/jpeg” gibi.

 

16)` udp.port == 69`

İlgili UDP portuna gönderilen veya alınan paketleri filtreler.

 

17) `wlan.ssid == Isyeri_Wifi`

İlgili WIFI’ın SSID’sine yapılan ve alınan paketleri bize sunar.

 

18) `wlan.addr == 00:1c:12:dd:ac:4d`

İlgili WLAN MAC adresine yapılan ve alınan paketleri filtreler.

 

19) `icmp.type == 8 && imcp.type == 0`

ICMP echo 8, echo reply 0 olarak ayarlanan bu filtre bize sadece ICMP Request’lerini filtrelemektedir.

 

20) `icmp`

Bunun gibi tekli kullanımlar da mevcuttur. Fazla ayrıntıya girmeden direk ilgili protokolün olduğu tüm paketleri filtreler. Tahmin edebileceğini üzere http, dns, tcp, udp gibi tekli filtremeler de mevcuttur.

 
Kısa bir şekilde faydalı olabileceğini düşündüğüm Wireshark filtrelerini sizler için derledim. Eğer sizin de “**Şu filtre olmadan bu yazı olmaz!**” dediğiniz, önemli gördüğünüz bir filtre var ise yorumlarda paylaşabilir, yazıyı güncellememe yardımcı olabilirsiniz 🙂