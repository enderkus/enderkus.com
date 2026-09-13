---
title: "Zabbix'i büyük ölçekte işletmek: ClickHouse history storage neden önemli"
---

Birkaç yıl önce, yaklaşık 7.000 host'u tek bir Zabbix ile izlediğimiz bir şirkette çalıştım. Orada izleme bir yan proje değildi, tüm mühendislik ekiplerinin bir şeylerin alevlenip alevlenmediğini anlamak için güvendiği platformdu. Kubernetes üzerinde çalıştırıyorduk, özellikle izleme katmanının kendisi kendini iyileştirebilsin diye; çünkü Zabbix çökerse, tam da görünürlüğe en çok ihtiyaç duyulan anda diğer bütün ekipler kör kalırdı. Orada kesinti bir rahatsızlık değildi, her şeyi etrafında tasarladığımız tek başarısızlık senaryosuydu.

O platformu işletmenin zor kısmı hiçbir zaman host sayısı olmadı. Beni gerçekten uykusuz bırakan şey veriydi: ayda yaklaşık 2,5 TB yeni history satırı ve bunları silmeye yetişemeyen bir housekeeper süreci.

## Geride kalmanın gerçek bedeli

Pratikte şöyle görünüyordu. Kağıt üzerinde son derece makul bir retention penceremiz vardı. Ama disk kullanımı ay be ay o pencerenin işaret ettiği seviyenin çok üzerine çıkmaya devam ediyordu, çünkü housekeeper'ın silme işlemleri insert'lere hiçbir zaman tam olarak yetişemiyordu. Bu, bir izleme sisteminde gerçekten rahatsız edici bir konum: diskinin bitmesini asla göze alamayacağınız o veritabanı, sizin ayarlayabileceğiniz hiçbir parametre olmadan, sessizce dolmaya devam ediyor.

İşe yarayan tek gerçek çözüm kabaydı: anlık alan kazanmak için history tablolarını periyodik olarak silip yeniden oluşturmak. Sonunda bunu yaklaşık altı ayda bir yapar hale geldik. Pratikte bu, tüm ekiplerin alerting için bağımlı olduğu bir production sistemde bakım penceresi planlamak ve housekeeper'ın kendiliğinden geri kazanması gereken disk alanını satın almak için, kapasite planlaması için işimize yarayabilecek aylarca birikmiş history verisini çöpe atmayı kabul etmek anlamına geliyordu. İşe yarıyordu, ama bu bir takvime bağladığımız bir operasyonel vergiydi, gerçek bir çözüm değil; housekeeper'ın o hacimde artık uygulanabilir bir retention mekanizması olmadığının sessiz bir itirafıydı.

Zabbix 8.0, alternatif bir history storage backend olarak ClickHouse desteği ekliyor ve bu tam olarak o soruna hedef alıyor. Host sayısına değil, sorgu hızına değil, doğrudan retention mekanizmasının kendisine.

## Darboğaz gerçekte nerede

Zabbix'in history tabloları (sayısal, metin, log değerleri, her item için her poll'da bir satır), tüm konfigürasyon verinizle birlikte SQL veritabanınızda, Postgres veya MySQL'de tutulur. Birkaç host'ta bu hiç sorun değildir. Her 30-60 saniyede bir poll edilen binlerce host'ta ise history tabloları o veritabanındaki açık ara en büyük ve en meşgul yapı haline gelir ve durmadan büyür.

Housekeeper'ın görevi, ayarladığınız retention penceresinden eski satırları silmektir. Kulağa basit gelen bu iş, satır bazlı bir ilişkisel veritabanında hızla pahalılaşır:

- Silme işlemi satır satır (veya batch halinde) bir `DELETE`'tir; index'leri gezip güncellemek, WAL/binlog üretmek zorundadır; Postgres'te sadece `VACUUM` ile geri kazanılan dead tuple'lar bırakır, MySQL'de InnoDB'de de benzer şekilde fragmentasyon birikir.
- Bu silme işi, sürekli akan insert'lerin ihtiyaç duyduğu aynı disk I/O'su ve lock'lar için rekabet eder.
- Gelen veri hızı yeterince yüksekse, housekeeper'ın silme throughput'u yeni gelen veriye karşı yarışı basitçe kaybeder. Biraz geride kalmaz, kalıcı olarak geride kalır ve veri hacmi büyüdükçe bu fark sadece açılır.

## ClickHouse bunu gerçekten nasıl çözüyor, sadece dolanmıyor

ClickHouse kolon bazlı bir veritabanı ve sadece sıkıştırma bile yardımcı oluyor: zaman serisi değerleri kolon kolon depolandığında çok iyi sıkışıyor, yani aynı ayda 2,5 TB'lık ham history, satır bazlı bir depoda kaplayacağı alanın çok küçük bir kısmını kaplıyor. Ama bizim gerçek sorunumuzu çözecek olan kısım sıkıştırma değildi. Retention'dı.

Zabbix'in ClickHouse şemasının kullandığı `MergeTree` tabloları zamana göre partition'lanır ve expiry, bu partition'lamayla bağlantılı bir `TTL` clause'u ile yönetilir. Veri süresi dolduğunda, ClickHouse expired satırları tarayıp tek tek silmez. İçindeki her satırın TTL'i geçtiğinde tüm partition dosyasını bir bütün olarak düşürür. Bir partition'ı düşürmek neredeyse bir dosya sistemi işlemidir: o partition'da bin satır olması ile yüz milyon satır olması arasında maliyet açısından neredeyse fark yoktur ve bir `DELETE`'in yaptığı gibi yazma yolu ile rekabet etmez.

Asıl çözüm bu. Mesele "ClickHouse daha fazla veri tutabiliyor" değil, "ClickHouse'un retention mekanizması, gelen veri hacmi büyüdükçe bozulmuyor", ki bu tam olarak bizim ilişkisel housekeeper'ımızın hiçbir zaman sahip olmadığı özellik. Yılda iki kez bakım penceresi yok, disk alanına oynamak yok, sadece yeni history'e yer açmak için eski history'yi çöpe atmak yok.

Belirtmekte fayda var: Zabbix'in kendi housekeeper'ı ClickHouse verisini hiç yönetmiyor (bu dokümante edilmiş bir davranış, bug değil). Oradaki retention tamamen ClickHouse'un kendi `TTL`/partition-drop mekanizmasına, yani gerçekten tasarlandığı işe bırakılmış durumda. Trade-off ise trend'lerin hâlâ sadece SQL veritabanında hesaplanıp tutulması ve ClickHouse'un proxy tarafında history backend olarak desteklenmemesi, sadece server'da çalışması; trend verisinin ham history'e kıyasla çok daha küçük, önceden aggregate edilmiş bir veri seti olduğu düşünüldüğünde ikisi de kabul edilebilir sınırlamalar.

## Pratikte denemek

7.000 host'luk bir ortam kurmadan mekaniği görmek isterseniz, Zabbix 8'in ClickHouse history provider'ını uçtan uca bağlayan, şema dahil, küçük bir Docker Compose referansı hazırladım; history verisinin ClickHouse'a düştüğünü izleyip doğrudan inceleyebilirsiniz.

**Repo:** [github.com/enderkus/zabbix8-clickhouse](https://github.com/enderkus/zabbix8-clickhouse)

Bu bir öğrenme kurulumu, production deployment rehberi değil; ama gerçek bir kurulumu migrate etmeye değip değmeyeceğine karar vermeden önce ClickHouse tarafını görmenin hızlı bir yolu.
