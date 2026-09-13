---
title: "Zabbix'i büyük ölçekte işletmek: ClickHouse history storage neden önemli"
---

Birkaç yıl önce, yaklaşık 7.000 host'u tek bir Zabbix ile izlediğimiz bir kurulumda çalıştım. Kubernetes üzerindeydik, çünkü izleme kritik bir altyapıydı ve kesinti kabul edilemezdi. Host sayısı hiçbir zaman asıl zorluk olmadı. Asıl zorluk veriydi: ayda yaklaşık 2,5 TB yeni history satırı ve bunları silmeye yetişemeyen bir housekeeper süreci.

Zabbix 8.0, alternatif bir history storage backend olarak ClickHouse desteği ekliyor ve bu tam olarak o soruna hedef alıyor. Bu yazıda büyük ölçekte geleneksel kurulumda gerçekte neyin kırıldığını ve ClickHouse'un bunu neden çözdüğünü anlatıyorum.

## Darboğaz gerçekte nerede

Zabbix'in history tabloları (sayısal, metin, log değerleri, her item için her poll'da bir satır), tüm konfigürasyon verinizle birlikte SQL veritabanınızda, Postgres veya MySQL'de tutulur. Birkaç host'ta bu hiç sorun değildir. Her 30-60 saniyede bir poll edilen binlerce host'ta ise history tabloları o veritabanındaki açık ara en büyük ve en meşgul yapı haline gelir ve durmadan büyür.

Housekeeper'ın görevi, ayarladığınız retention penceresinden eski satırları silmektir. Kulağa basit gelen bu iş, satır bazlı bir ilişkisel veritabanında hızla pahalılaşır:

- Silme işlemi satır satır (veya batch halinde) bir `DELETE`'tir; index'leri gezip güncellemek, WAL/binlog üretmek zorundadır; Postgres'te sadece `VACUUM` ile geri kazanılan dead tuple'lar bırakır, MySQL'de InnoDB'de de benzer şekilde fragmentasyon birikir.
- Bu silme işi, sürekli akan insert'lerin ihtiyaç duyduğu aynı disk I/O'su ve lock'lar için rekabet eder.
- Gelen veri hızı yeterince yüksekse, housekeeper'ın silme throughput'u yeni gelen veriye karşı yarışı basitçe kaybeder. Biraz geride kalmaz, kalıcı olarak geride kalır ve veri hacmi büyüdükçe bu fark sadece açılır.

Pratikteki belirti tam olarak bizim karşılaştığımız şeydi: kağıt üzerinde makul bir retention penceresi ayarlanmıştı, ama disk kullanımı buna rağmen artmaya devam ediyordu, çünkü silme işlemi asla tam olarak yetişemiyordu. Elimizdeki tek gerçek çözüm, anlık alan kazanmak için history tablolarını periyodik olarak silip yeniden oluşturmaktı; bunu sonunda yaklaşık altı ayda bir yapar hale geldik. İşe yarıyordu, ama bu bir bakım penceresi operasyonuydu, gerçek bir çözüm değil. Housekeeper'ın o hacimde uygulanabilir bir retention mekanizması olmadığının itirafıydı aslında.

## ClickHouse bunu gerçekten nasıl çözüyor, sadece dolanmıyor

ClickHouse kolon bazlı bir veritabanı ve sadece sıkıştırma bile yardımcı oluyor: zaman serisi değerleri kolon kolon depolandığında çok iyi sıkışıyor, yani aynı ayda 2,5 TB'lık ham history, satır bazlı bir depoda kaplayacağı alanın çok küçük bir kısmını kaplıyor. Ama housekeeper sorununu çözen kısım sıkıştırma değil. Retention.

Zabbix'in ClickHouse şemasının kullandığı `MergeTree` tabloları zamana göre partition'lanır ve expiry, bu partition'lamayla bağlantılı bir `TTL` clause'u ile yönetilir. Veri süresi dolduğunda, ClickHouse expired satırları tarayıp tek tek silmez. İçindeki her satırın TTL'i geçtiğinde tüm partition dosyasını bir bütün olarak düşürür. Bir partition'ı düşürmek neredeyse bir dosya sistemi işlemidir: o partition'da bin satır olması ile yüz milyon satır olması arasında maliyet açısından neredeyse fark yoktur ve bir `DELETE`'in yaptığı gibi yazma yolu ile rekabet etmez.

Asıl çözüm bu. Mesele "ClickHouse daha fazla veri tutabiliyor" değil, "ClickHouse'un retention mekanizması, gelen veri hacmi büyüdükçe bozulmuyor", ki bu tam olarak ilişkisel bir housekeeper'ın büyük ölçekte sahip olmadığı özellik.

Belirtmekte fayda var: Zabbix'in kendi housekeeper'ı ClickHouse verisini hiç yönetmiyor (bu dokümante edilmiş bir davranış, bug değil). Oradaki retention tamamen ClickHouse'un kendi `TTL`/partition-drop mekanizmasına, yani gerçekten tasarlandığı işe bırakılmış durumda. Trade-off ise trend'lerin hâlâ sadece SQL veritabanında hesaplanıp tutulması ve ClickHouse'un proxy tarafında history backend olarak desteklenmemesi, sadece server'da çalışması; trend verisinin ham history'e kıyasla çok daha küçük, önceden aggregate edilmiş bir veri seti olduğu düşünüldüğünde ikisi de kabul edilebilir sınırlamalar.

## Pratikte denemek

7.000 host'luk bir ortam kurmadan mekaniği görmek isterseniz, Zabbix 8'in ClickHouse history provider'ını uçtan uca bağlayan, şema dahil, küçük bir Docker Compose referansı hazırladım; history verisinin ClickHouse'a düştüğünü izleyip doğrudan inceleyebilirsiniz.

**Repo:** [github.com/enderkus/zabbix8-clickhouse](https://github.com/enderkus/zabbix8-clickhouse)

Bu bir öğrenme kurulumu, production deployment rehberi değil; ama gerçek bir kurulumu migrate etmeye değip değmeyeceğine karar vermeden önce ClickHouse tarafını görmenin hızlı bir yolu.
