---
title: "zabterm: Zabbix, terminalinde"
---

Gün içinde Zabbix'te bakılan şeylerin çoğu tek bir soruya çıkıyor: şu an ne bozuk? Frontend bu soruyu iyi cevaplıyor ama bir tarayıcı sekmesinde, bir girişin arkasında, işin geri kalanının döndüğü terminalden birkaç tık uzakta duruyor. Ben de [zabterm](https://github.com/enderkus/zabterm)'i yaptım: açıldığı anda bu soruyu cevaplayan, sonra da fareye uzanmadan detaya inmenizi sağlayan, Zabbix için bir terminal arayüzü.

<figure>
<a href="{{ '/assets/images/zabterm-dashboard.png' | relative_url }}"><img src="{{ '/assets/images/zabterm-dashboard.png' | relative_url }}" alt="Host ve problem kutucukları, önem derecesi çubuğu, filo CPU grafiği ve host tablosu olan zabterm dashboard'u" width="1800" height="1131"></a>
<figcaption>Dashboard, birkaç olayın sürdüğü yaklaşık yirmi hostluk bir demo filoya bağlıyken.</figcaption>
</figure>

## Kısa bir tur

Dashboard birkaç saniyede okunacak şekilde tasarlandı. En üstte büyük rakamlı kutucuklar kaç host olduğunu, kaçına erişilebildiğini, kaç problemin aktif olduğunu ve filonun CPU ile bellekte ne kadar yoğun olduğunu gösteriyor, her birinin altında da en yoğun host yazıyor. Altında tek bir çubuk aktif problemleri önem derecesine göre bölüyor, böylece tek satır okumadan günün nasıl gittiği görünüyor. Ortada en yoğun hostların son 30 dakikalık CPU ya da bellek grafiği ve en son problemlerin akışı yan yana duruyor, en altta da her hostun son CPU değerlerini küçük bir sparkline ile gösteren bir tablo var.

Hosts ekranı aynı listenin daha geniş hali: her host için erişilebilirlik, gruplar, arayüz, CPU ve bellek göstergeleri, disk, load, uptime ve önem derecesine göre problem sayacı. `/` ile filtreleyebiliyor, problemlere, CPU'ya ya da belleğe göre sıralayabiliyorsunuz. Alttaki panel de seçiminizi takip edip o hostun son CPU ve bellek geçmişini gösteriyor.

<figure>
<a href="{{ '/assets/images/zabterm-hosts.png' | relative_url }}"><img src="{{ '/assets/images/zabterm-hosts.png' | relative_url }}" alt="Host tablosu ve seçili host için CPU ve bellek grafikleri olan zabterm hosts ekranı" width="1800" height="1131"></a>
<figcaption>Problemlere göre sıralanmış hostlar. Erişilemeyen host kırmızı, son bilinen değerleri soluk görünüyor.</figcaption>
</figure>

Bir host'ta enter'a bastığınızda o hostun sayfası açılıyor: braille karakterleriyle çizilmiş CPU, bellek, ağ ve load grafikleri, zaman aralığı da `[` ve `]` ile 15 dakikadan 7 güne kadar değişiyor. Yan panel dosya sistemi kullanımını ve hostun aktif problemlerini listeliyor, ikinci sekme ise hosttaki her item'ı son değeriyle birlikte gösteriyor, ada ya da key'e göre aranabiliyor.

Problems ekranı şu an tetiklenmiş her şeyi en kritikten başlayarak, yaşı, hostu ve tag'leriyle listeliyor, seçili olan için de bir detay paneli gösteriyor. `a` tuşuyla bir mesaj yazıp acknowledge edebiliyor, trigger manuel kapatmaya izin veriyorsa aynı adımda kapatabiliyorsunuz.

<figure>
<a href="{{ '/assets/images/zabterm-problems.png' | relative_url }}"><img src="{{ '/assets/images/zabterm-problems.png' | relative_url }}" alt="Önem derecesi etiketleri, problem listesi ve detay paneli olan zabterm problems ekranı" width="1800" height="1131"></a>
<figcaption>En kritikten başlayarak problemler, altta seçili olanın detayları.</figcaption>
</figure>

## Bir şey değişince haber veriyor

zabterm her sorgulamayı bir öncekiyle karşılaştırıyor. Bir problem tetiklendiğinde ya da çözüldüğünde, bir host erişilemez olduğunda ya da geri geldiğinde köşede küçük bir bildirim gösteriyor ve bir masaüstü bildirimi gönderiyor: Linux'ta `notify-send`, macOS'ta Bildirim Merkezi üzerinden. Minimum önem derecesi ayarı gürültüyü azaltıyor, açtığınızda zaten bozuk olan şeyler için değil, sadece sonrasında değişenler için bildirim geliyor.

## Omarchy düşünülerek yapıldı

[Omarchy](https://omarchy.org)'de yerel bir uygulama gibi hissettirmesini istedim. Tema `auto` olarak ayarlıyken zabterm aktif Omarchy temasının renklerini okuyor ve tema değiştirdiğiniz an, yeniden başlatmaya gerek kalmadan kendini yeniden boyuyor. Kurulum scripti onu kendi ikonuyla uygulama menüsüne ekliyor, bildirimleri de masaüstündeki diğer her şey gibi mako'ya düşüyor. Omarchy dışında Tokyo Night, Catppuccin, Gruvbox ve Nord dahil sekiz hazır tema ile geliyor.

## Tek bir küçük config dosyası

Her şey `~/.config/zabterm/config.toml` dosyasında duruyor, `zabterm init` de sizin için açıklamalı bir tane yazıyor. İçinde bir ya da birden fazla profil tutulabiliyor, böylece production, staging ve bir test ortamı yan yana durabiliyor, `zabterm -p staging` ile birini seçiyorsunuz. Token'ların dosyada düz metin olarak durması da gerekmiyor: bir profil token'ı bir ortam değişkeninden ya da onu yazdıran herhangi bir komuttan okuyabiliyor.

```toml
[profiles.prod]
url = "https://zabbix.example.com"
token_cmd = "op read op://ops/zabbix-prod/token"
```

## Perde arkası

zabterm Rust ile, [ratatui](https://ratatui.rs) kullanılarak yazıldı. API istemcisi bir arka plan görevine ait ve Zabbix JSON-RPC API'sini o sorguluyor, arayüz ise sadece son anlık görüntüyü okuyor. Bu sayede API yavaşken ya da erişilemezken bile ekran donmuyor. Her sorgulama, filo ne kadar büyük olursa olsun birkaç çağrıdan ibaret, özet ekranı da standart Linux agent şablonunun item key'lerini okuyor. Zabbix'e geri yazdığı tek şey bir acknowledge.

Geliştirmeyi [önceki yazımdaki]({% post_url 2026-09-13-zabbix-8-clickhouse-veri-deposu %}) Zabbix 8 ve ClickHouse ortamına karşı yaptım, Zabbix 7.0 ve sonrasıyla da çalışması gerekiyor. Linux ve macOS'ta çalışıyor.

## Deneyin

Omarchy'de, herhangi bir Linux'ta ya da macOS'ta:

```sh
curl -fsSL https://raw.githubusercontent.com/enderkus/zabterm/main/install.sh | sh
```

Ya da Homebrew ile:

```sh
brew install enderkus/tap/zabterm
```

Sonra `zabterm init` çalıştırın, Zabbix adresinizi ve bir API token ekleyin, bağlantıyı `zabterm check` ile kontrol edin ve `zabterm` ile başlatın. [Dokümantasyon](https://enderkus.github.io/zabterm/) ayarları, tüm tuşları ve API ile nasıl konuştuğunu anlatıyor. Kod MIT lisansıyla [GitHub'da](https://github.com/enderkus/zabterm), issue'lara, fikirlere ve pull request'lere açığım.
