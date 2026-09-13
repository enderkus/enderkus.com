---
title: "Zabbix 8 ve ClickHouse: yeni history storage backend'ini kurmak"
---

Zabbix 8.0 ile gelen özelliklerden biri beni epey heyecanlandırdı: **history storage backend olarak ClickHouse desteği**. Konfigürasyon verisi (host'lar, item'lar, trigger'lar, kullanıcılar) hâlâ normal SQL veritabanında tutuluyor, ama yüksek hacimli history verisi, yani her item için birkaç saniyede bir biriken sayısal, metin ve log değerleri, artık doğrudan ClickHouse'a yazılabiliyor.

Bunu Docker üzerinde kurup pratikte nasıl davrandığını görmek için bir akşamımı harcadım, yol boyunca gerçekten ilginç bir hatayla karşılaştım ve sonunda her şeyi çalıştırılabilir bir demo repo haline getirdim. Bu yazı ikisini de anlatıyor.

## Neden ClickHouse

Zabbix, history verisini şimdiye kadar hep PostgreSQL veya MySQL'de tuttu. Bu çalışıyor, ama satır bazlı bir OLTP veritabanına aslında bir OLAP işi yaptırmış oluyorsunuz: sürekli akan zaman damgalı değerleri yazmak ve sonra "bu item için şu iki tarih arasındaki tüm değerleri getir" gibi range sorgularını, potansiyel olarak milyarlarca satır üzerinden yanıtlamak.

ClickHouse tam da bu iş için tasarlanmış, kolon bazlı bir veritabanı:

- **Sıkıştırma.** Her kolon (itemid, timestamp, value) ayrı ayrı depolanıp sıkıştırılıyor. Zaman serisi değerleri birbirine yakın olma eğiliminde olduğu için bu çok iyi sıkışıyor. Satır bazlı bir tabloda aynı veri için gereken disk alanının çok küçük bir kısmıyla aylarca ayrıntılı history tutabiliyorsunuz.
- **Hızlı range ve aggregate sorguları.** Zabbix'in grafikleri ve dashboard'ları zaten tam olarak ClickHouse'un optimize edildiği deseni kullanıyor: itemid'ye göre filtrele, zaman aralığına göre filtrele, aggregate et. Bunun için OLTP tarzı point-lookup index'lere ihtiyaç yok.
- **Postgres'in yükünü azaltıyor.** Yeterince host ve yeterince kısa polling aralığına ulaştığınızda, PostgreSQL tabanlı bir Zabbix kurulumunu ilk zorlayan şey genelde history yazmaları oluyor. Bunları amaca özel bir depoya taşımak, konfigürasyon veritabanının rahat nefes almasını ve history hacminin bağımsız ölçeklenmesini sağlıyor.

Resmi dokümantasyon da trade-off'lar konusunda oldukça açık: Zabbix housekeeper ClickHouse verisini temizlemiyor (retention tamamen ClickHouse'un kendi TTL mekanizmasıyla kontrol ediliyor), trend'ler hâlâ sadece SQL veritabanında hesaplanıp tutuluyor, ve ClickHouse proxy tarafında history backend olarak desteklenmiyor, sadece server'da çalışıyor. Karar vermeden önce bilinmesi gereken şeyler.

## Kurulum: Docker Compose, üç agent, bir hata

Konfigürasyon için Postgres, history için bir ClickHouse container'ı, Zabbix server ve frontend, ve üç agent içeren bir stack kurdum: biri dahili "Zabbix server" host'unu (kendi kendini izleme) temsil ediyor, diğer ikisi ise örnek veri üretmek için kullanılan geçici "test-agent" container'ları.

Stack'i ayağa kaldırdığımda üç host'tan ikisi hemen çalışmaya başladı. Sorun yaşayan ise en beklemediğiniz host'tu: **"Zabbix server"ın kendisi**. Frontend'de sürekli erişilemez görünüyordu, kendi sağlık metrikleri için hiç veri gelmiyordu, oysa iki test agent'ı da sorunsuz raporluyordu.

Loglar durumu net gösterdi:

```
temporarily disabling Zabbix agent checks on host "Zabbix server": interface unavailable
```

Zabbix, dahili "Zabbix server" host'unu agent interface'i `127.0.0.1:10050` olarak sabitlenmiş şekilde gönderiyor. Bu, agent'ın server sürecinin çalıştığı makinede olduğu geleneksel tek makine kurulumları için mantıklı bir varsayılan. Ama bu Docker Compose kurulumunda, o host'un agent'ı kendi container'ında, kendi IP'sinde çalışıyor, `zabbix-server` ile aynı network namespace'ini paylaşmıyor. Yani `zabbix-server` sadakatle kendi loopback interface'ine bakıyor, orada dinleyen bir şey bulamıyor ve host'u erişilemez olarak işaretliyor. İki test agent'ının sorunsuz çalışmasının sebebi, en baştan `127.0.0.1` yerine kendi container'larına işaret eden, DNS tabanlı doğru interface'lerle tanımlanmış olmalarıydı.

Çözüm tek satırlık bir düzeltme: o interface'i `127.0.0.1` yerine agent container'ının compose network'ündeki DNS adına yönlendirmek. Bunu yaptıktan sonra "Zabbix server" tek bir polling döngüsünde diğer ikisiyle birlikte çalışır hale geldi.

## Tekrarlanabilir hale getirmek

Elle veritabanı yaması yapmak, başka birine verebileceğim bir çözüm değildi. Bu yüzden düzeltmeyi, ilk açılışta Zabbix API'siyle konuşan küçük bir init container'a çevirdim: API'nin ayağa kalkmasını bekliyor, giriş yapıyor, "Zabbix server" host'unun interface'inin hâlâ `127.0.0.1`'e mi işaret ettiğini kontrol ediyor ve öyleyse agent container'ına yönlendiriyor. Aynı zamanda iki test-agent host'unu da otomatik olarak kaydediyor, böylece taze bir checkout'ta arayüzde tek bir tıklama bile gerekmiyor. Her iki adım da idempotent, yani her `docker compose up`'ta tekrar çalıştırılabilir.

Her şeyi, Postgres, ClickHouse, şema oluşturma script'i, Zabbix stack'i ve bu provisioning adımını, tek bir Docker Compose dosyasında topladım. Repo'yu klonlayın, `docker compose up -d` çalıştırın, yaklaşık bir dakika bekleyin, elinizde history verisini ClickHouse'a yazan, tamamen izlenen bir Zabbix 8 kurulumu olsun, üstelik ClickHouse verisine doğrudan bakmak için bir Tabix arayüzüyle birlikte.

**Repo:** [github.com/enderkus/zabbix8-clickhouse](https://github.com/enderkus/zabbix8-clickhouse)

README dosyası mimariyi, [Zabbix'in resmi kurulum dokümantasyonunu](https://www.zabbix.com/documentation/8.0/en/manual/appendix/install/clickhouse_setup) yansıtan ClickHouse şemasının tamamını ve her konfigürasyon ayarını anlatıyor. Bu bir öğrenme ve demo kurulumu, production sertleştirme rehberi değil: varsayılan parolalar, TLS yok, tek node'lu ClickHouse. Parçaların nasıl bir araya geldiğini anlamak için bir başlangıç noktası olarak düşünün, production'a yaklaşmadan önce sertleştirin.
