# İç Mimarlık Demo Sitesi (Kütahya) — 6 Senaryo

Tek dosyalık statik demo. Build yok, bağımlılık yok. Palet: **espresso + terrakota**.

**Hepsini bir arada görmek için:** [`senaryolar.html`](senaryolar.html)

> **Durum:** tamam. 3 hero loop + 2×120 kare + 5 galeri önizlemesi üretildi ve yerinde.
> Videolar yine de zorunlu değil: silinirlerse hero CSS fallback'e, scroll
> prosedürel placeholder'a düşer ve site çalışmaya devam eder.

## Senaryolar

3 hero videosu × 2 scroll videosu = 6 kombinasyon. `index.html` URL parametresiyle seçilir:

| | Hero (`?h=`) | Scroll (`?s=`) |
|---|---|---|
| **1 / a** | Altın saat oturma odası, keten kanepe | Giriş holünden salona daire turu |
| **2 / b** | Malzeme masası: keten, meşe, mermer, pirinç | Makro dokudan geniş plana açılış |
| **3** | Akşam mutfağı, taş ada, pirinç armatür | — |

```
index.html?h=1&s=a   ← varsayılan (en geniş kitle)
index.html?h=2&s=b   ← anlatı bütünlüğü en güçlü olan
```

Scroll videosuna göre sahne metinleri de değişir (`COPY` sabiti, `index.html` içinde):
`s=a` → GİRİŞ / AKIŞ / IŞIK · `s=b` → DOKU / DETAY / BÜTÜN

Sitenin sağ altındaki **senaryo anahtarı** ile videolar arasında geçiş yapabilirsin;
scroll konumunu koruduğu için aynı noktada A/B karşılaştırması yapılabilir.

---

## Çalıştırma

`file://` ile açma — `frames/` fetch'i CORS'a takılır. Local server şart:

```bash
python -m http.server 8021
```

`http://localhost:8021/senaryolar.html`

---

## Klasör yapısı

```
index.html          demo site (senaryo parametreli)
senaryolar.html     6 senaryo karşılaştırma galerisi
media/
  hero1..3.mp4/.webm     16 sn dikişsiz boomerang loop, 1600x900, sessiz
  hero1..3-mp.mp4/.webm  DİKEY telefon hero: 828x1794 portre kırpım (0.54–1.24 MB)
  hero1..3-m.mp4/.webm   YATAY telefon hero: 1280x720 (0.52–0.83 MB)
  preview/h1..3.mp4    galeri önizlemeleri (560px, ~150–200 KB)
  preview/sa,sb.mp4    galeri önizlemeleri
frames/a/0001..0120.jpg  scroll-scrub kareleri (1440px, masaüstü)
frames/b/0001..0120.jpg
frames/a-m/, frames/b-m/ MOBİL kare seti (720px, her 3. kare + son kare = 41 kare)
_raw/               ham Higgsfield çıktıları — DAĞITILMAZ (.vercelignore)
_tools/             npm ffmpeg/ffprobe + ekran görüntüleri — DAĞITILMAZ
```

Dağıtılan toplam **38 MB** (media 20 MB + frames 18 MB). Ziyaretçi başına indirilen:
· **masaüstü:** 1 hero webm (0.8–1.3 MB) + 120 kare (7.3–8.0 MB) ≈ **8.2 MB**
· **telefon:** 1 dikey hero (0.54–1.24 MB) + 41 mobil kare (0.82–0.95 MB) = **2.00–2.23 MB** (ölçüldü)

İç mekân görüntüsü (kumaş dokusu, tül, parke deseni) mimarlık versiyonundaki betondan
daha detaylı olduğu için JPEG'ler aynı kalitede daha ağır basıyor — mimar-demo 21 MB'tı.
Hafifletmek gerekirse kare üretiminde `-q:v 4` yerine `-q:v 5` yeter (~%15–20 kazanç).

**Dosya adı tutarlılığı:** kare klasörleri `a`/`b` olduğu için önizleme klipleri de
`sa.mp4`/`sb.mp4` — `s1`/`s2` değil. Karıştırma.

---

## Videolar geldiğinde — ffmpeg işlemleri

ffmpeg sistemde kurulu değil, npm ile kurulur:

```bash
mkdir _tools && cd _tools && npm init -y
npm i @ffmpeg-installer/ffmpeg @ffprobe-installer/ffprobe
# yol: node -e "console.log(require('@ffmpeg-installer/ffmpeg').path)"
```

Gelen ffmpeg 2018 sürümüdür — `amix=...:normalize=0` gibi yeni opsiyonlar yok,
ama `reverse`, `delogo`, `concat`, `drawtext` var.
Higgsfield çıktıları tipik olarak: 1920×1080, 24 fps, 8.04 sn, 193 kare, AAC sesli.

### Hero → dikişsiz loop (boomerang)

Düz `concat` yaparsan dönüş noktasında kare tekrar eder ve 1 karelik takılma olur.
Ters klipten ilk ve son kare atılmalı:

```bash
# 1) ileri
ffmpeg -y -i _raw/hero_raw_alt1.mp4 -an -vf "scale=1600:-2" \
  -c:v libx264 -crf 20 -preset veryfast -pix_fmt yuv420p _tools/tmp/f1.mp4
# 2) geri (n=0 ve n=192 atılır → 191 kare)
ffmpeg -y -i _tools/tmp/f1.mp4 -an \
  -vf "reverse,select='between(n\,1\,191)',setpts=N/FRAME_RATE/TB" \
  -c:v libx264 -crf 20 -preset veryfast -pix_fmt yuv420p _tools/tmp/r1.mp4
# 3) birleştir  (list1.txt: file 'f1.mp4' / file 'r1.mp4')
ffmpeg -y -f concat -safe 0 -i _tools/tmp/list1.txt -an \
  -c:v libx264 -crf 25 -preset slow -pix_fmt yuv420p -movflags +faststart media/hero1.mp4
# 4) webm
ffmpeg -y -i media/hero1.mp4 -an -c:v libvpx-vp9 -crf 36 -b:v 0 -row-mt 1 \
  -deadline good -cpu-used 3 media/hero1.webm
```

Sonuç: 384 kare = tam 16.000 sn, 1600×900.

### Scroll → 120 kare

```bash
ffmpeg -y -i _raw/scroll_raw_alt1.mp4 -vf "fps=15,scale=1440:-2" -q:v 4 \
  -frames:v 120 "frames/a/%04d.jpg"
```

- 8 sn × 15 fps = 120 kare → JS'teki `FRAME_COUNT = 120` ile birebir.
- `-frames:v 120` şart, yoksa 121. kare üretilip eşleşme kayar.
- Sıfır-pad 4 hane (`%04d` ↔ `padStart(4,'0')`).
- Video 5 sn geldiyse `fps=24` kullan (5×24=120), `FRAME_COUNT` değişmez.

Watermark çıkarsa CSS ile kapatma, kaynakta sil:
`-vf "delogo=x=1715:y=875:w=175:h=165,fps=15,scale=1440:-2"`

### Galeri önizlemeleri

```bash
ffmpeg -y -i media/hero1.mp4 -an -vf "scale=560:-2" -c:v libx264 -crf 31 \
  -preset slow -pix_fmt yuv420p -movflags +faststart media/preview/h1.mp4
ffmpeg -y -i _raw/scroll_raw_alt1.mp4 -an -vf "scale=560:-2" -c:v libx264 -crf 31 \
  -preset slow -pix_fmt yuv420p -movflags +faststart media/preview/sa.mp4
```

Önizleme yoksa galeri kutusu boş kalmaz — `.pane.empty` yer tutucusu devreye girer.

---

## Ayar noktaları

| Ne | Nerede |
|---|---|
| Scroll hızı / uzunluğu | `.scene { height:520vh }` (mobil `400vh`) — büyük = yavaş scrub |
| Kare sayısı | JS `FRAME_COUNT` (ffmpeg fps ile senkron olmalı) |
| Metin sahne zamanları | `band()`/`bell()`: `0.14–0.26`, `0.30–0.56`, `0.58–0.78`, kart `0.80–0.92` |
| Sahne metinleri | JS `COPY` sabiti (scroll videosuna göre iki set) |
| Mobil kare seyreltme | JS `STEP = isMobile ? 3 : 1` + ayrı `frames/<a\|b>-m/` klasörü |
| Telefon tespiti | JS `isPhone` — `innerWidth<768 \|\| min(innerWidth,innerHeight)<600` |
| Mobil hero varyantı | JS `MV` — dikeyde `-mp` (828x1794), yatayda `-m` (1280x720) |
| Mobil hero kırpımı | ffmpeg `crop=W:H:X:Y` — h1 `378:820:1520:260`, h2 `498:1080:320:0`, h3 `434:940:720:140` |
| Mobil varsayılan hero | JS `HERO` — `?h=` yoksa telefonda `'2'`, masaüstünde `'1'` |
| Mobil alt bar | `.mbar` CSS + `<div class="mbar">` HTML; `@media(max-width:860px)` |
| Kare geç yükleme | `preload()` — IntersectionObserver + scroll + hero `loadeddata` + 2.5 sn |
| DPR tavanı | `resize()` içindeki `Math.min(devicePixelRatio, 2)` |
| Palet | `:root` → `--night --terra --frost` |
| Nav hamburger eşiği | `@media(max-width:1200px)` — 6 sekmeli nav ölçülen 1149px ister |

---

## Müşteriye teslim ederken

1. **Senaryoyu sabitle.** `index.html` JS'inde `HERO` / `SCRL` sabitlerini seçilen
   değere sabitle, URL parametresi okumasını kaldır.
2. **Demo anahtarını sil.** `<div class="demo-bar">` bloğu + `.demo-bar` CSS'i +
   `demoBar()` JS fonksiyonu.
3. **Galeriyi sil.** `senaryolar.html` dosyası, header ve footer'daki
   `class="nav-demo"` sekmeleri, kullanılmayan `media/hero*` (`-m` varyantları dahil)
   ve `frames/*` setleri (`-m` klasörleri dahil).
4. **Firma bilgilerini değiştir** (liste `index.html` başındaki yorum bloğunda):
   firma adı + `brand-mono` harfleri, telefon, WhatsApp, e-posta, adres,
   `data-count` istatistikleri, `<title>` ve `<meta name="description">`.
   **Telefon ve WhatsApp iki yerde:** iletişim bölümü *ve* mobil alt bar
   (`.mbar .m-tel` / `.mbar .m-wa`). İkisini birlikte değiştir.
5. **Proje fotoğraflarını koy.** Proje kartlarında `.proj-art` SVG'lerini gerçek
   fotoğrafla değiştir: `<div class="proj-art">…</div>` → `<img src="projeler/1.jpg" alt="…">`
   (stil aynı kalır, `aspect-ratio:4/5`).
6. `about-visual` içindeki izometrik oda SVG'si de bir mekân fotoğrafıyla değişebilir.

**Not:** WhatsApp'tan HTML dosyası göndermek işe yaramaz — `index.html` tek başına
videoları içermez ve telefonda düzgün açılmaz. Her zaman canlı link gönder.

---

## Mobil uyum (Eylül 2026)

Masaüstü görünümü değişmedi; eklemelerin tamamı `@media(max-width:900/860/680px)` içinde
ya da yalnız telefonda çalışan JS dallarında.

### Ne değişti

1. **Mobil sabit alt bar** (`.mbar`) eklendi: **WhatsApp + Ara** yan yana, 860px altında görünür.
   `body`'ye alttan 68px + `env(safe-area-inset-bottom)` padding, `viewport-fit=cover`.
2. **Dokunma hedefleri** telefonda en az 44×44: nav hamburger, marka bağlantısı, menü linkleri,
   menü CTA'sı, tüm `.btn`'ler, iletişim kutusu tel/e-posta linkleri, footer linkleri,
   senaryo anahtarı düğmeleri. Görsel boyutlar aynı, büyüyen şey padding/min-height.
3. **Yazı boyutları** telefonda en az 13px, gövde 16px: `hero-kick`, `kick`, `proj-meta`,
   `proj-loc`, `stat .lbl`, `tile-lbl`, `scrub-specs .l`, `scrub-card .loc`, `scrub-hint`,
   `foot-links`, `foot-copy`, senaryo anahtarı ve izometrik SVG etiketi. Büyük harf +
   harf aralıklı etiketlerde punto büyürken `letter-spacing` kısaltıldı ki satır taşmasın.
   `brand-sub` (8.5px) dikey modda navda gizlendi — 34 karakter 13px'te dar nava sığmıyor;
   footer'da ve yatay modda 13px olarak görünüyor.
4. **iOS video kuralı:** hero artık `<source>` listesi kullanmıyor. Format `canPlayType` ile
   seçiliyor (Safari/iOS → her zaman mp4), **tek `src`** veriliyor, `error` olayında diğer
   formata geçiliyor, otomatik oynatma engellenirse ilk dokunuş/tıklamada başlatılıyor.
   `saveData` açıksa video hiç inmiyor, CSS fallback görünüyor.
5. **Mobil medya seti:** telefonda `hero<N>-mp.*` (dikey) / `hero<N>-m.*` (yatay) ve
   41 karelik `frames/<a|b>-m/` iniyor.
   Kareler geç yükleniyor (IntersectionObserver + ilk kaydırma + hero `loadeddata`, en geç
   2.5 sn) — ilk ekran hero ile bant genişliği yarışmıyor.
6. **Yatay mod** (≤900×≤480): nav bandı inceldi, hero metni nav ile alt bar arasına ortalandı,
   CTA'lar yan yana kaldı, scroll-scrub kartı sıkıştırılıp sola alındı (sağ altta senaryo
   çipi var), scrub ipucu gizlendi.
7. **Senaryo anahtarı** telefonda katlanır: kapalıyken tek bir "Senaryo" çipi (44px),
   açıkken tam panel. Alt barın 8px üstünde duruyor, çakışma ölçüldü (yok). Menü açıkken gizli.
8. **Mobil menü** erişilebilirlik: `aria-expanded` + `aria-label` güncelleniyor, Esc kapatıyor
   ve odağı hamburgere geri veriyor, Tab odağı panelde dönüyor, açıkken `body` scroll kilitli,
   1200px üstüne genişlerken panel otomatik kapanıyor. Panelin üst kenarı artık sabit px değil,
   JS ile ölçülen `--navh`.
9. **Güvenli alan:** `--pad` telefonda `max(20px, env(safe-area-inset-left/right))`,
   alt bar ve senaryo çipi `env(safe-area-inset-bottom)` kadar yukarıda.
10. **`senaryolar.html`** de mobil uyumlu: 860px altında tüm yazılar ≥13px, dokunma hedefleri
    ≥44px; 320px'te marka ve "Siteye dön" alt alta geçiyor (yan yana 6px taşıyordu).

### Ölçümler

Chromium, `python -m http.server`, önbellek boş. Her genişlikte 4 durum ölçüldü
(üst, menü açık, scroll-scrub sonu, footer).

| Genişlik | Yatay taşma | <44px dokunma hedefi | <13px yazı | Konsol hatası | Ziyaretçi başına |
|---|---|---|---|---|---|
| 320×700 | 0 | 0 | 0 | 0 | 1.14 MB |
| 360×780 | 0 | 0 | 0 | 0 | 1.14 MB |
| 375×812 | 0 | 0 | 0 | 0 | 1.14 MB |
| 390×844 | 0 | 0 | 0 | 0 | 1.14 MB |
| 414×896 | 0 | 0 | 0 | 0 | 1.14 MB |
| 812×375 (yatay) | 0 | 0 | 0 | 0 | 1.14 MB |
| 1440×900 (masaüstü) | 0 | değişmedi | değişmedi | 0 | 8.19 MB |

Ziyaretçi başına inen 1.14 MB = doküman 80 KB + mobil hero 248 KB + 41 kare 835 KB.
Hedef 2.5 MB idi; öncesi (masaüstü seti telefona iniyordu) ~8.2 MB'tı.

Kontrast: alt bar WhatsApp düğmesi `#25d366` üzerine `--night` metin **9.9:1**,
Ara düğmesi `--frost` metin **14.9:1** — ikisi de AA'nın üstünde.

`senaryolar.html` aynı genişliklerde ayrıca ölçüldü: 320–414 ve 812×375'te yatay taşma 0,
13px altı yazı 0, 44px altı hedef 0, konsol temiz.

Ekran görüntüleri: `_tools/tmp/mobil/` (375 hero / hizmetler / scrub / iletişim / menü /
senaryo paneli / senaryolar.html, 812×375 yatay, 1440 hero).

### 2. tur — gerçek telefon geri bildirimi (Eylül 2026)

Mert siteyi iPhone'da açtı: *hero bulanık* ve *ilk ekranda yazı/düğme o kadar çok yer
kaplıyor ki videoda ne olduğu anlaşılmıyor.* İkisi de düzeltildi.

**A) Hero çözünürlüğü ve kadrajı.** Asıl sorun piksel sayısı değil **en-boy oranıydı.**
Dikey telefonda hero tam ekran ve `object-fit:cover`; 16:9 bir dosya 1170x2532 cihaz
pikseline kaplatılırken **6.24 kat** büyütülüyordu (720x406 dosya). Yalnızca genişliği
1280'e çıkarmak bunu 3.52'ye indirirdi — masaüstü dosyasının telefondaki 2.81'inden
hâlâ kötü. Bu yüzden telefona **dikey kırpım** üretildi:

| | dosya | `videoWidth/(clientWidth·DPR)` | **gerçek büyütme** (`cover`) |
|---|---|---|---|
| önce | 720x406 | 0.62 | **6.24x** |
| masaüstü dosyası telefonda | 1600x900 | 1.37 | 2.81x |
| 1280x720 (sadece genişlik artsaydı) | 1280x720 | 1.09 | 3.52x |
| **şimdi (375x812)** | 828x1794 | 0.74 | **1.36x** |
| **şimdi (390x844)** | 828x1794 | 0.71 | **1.41x** |
| **şimdi (414x896)** | 828x1794 | 0.67 | **1.50x** |
| şimdi (812x375 yatay) | 1280x720 | 0.53 | 1.90x |

İstenen `videoWidth/(clientWidth·DPR)` oranı dikey kırpımda 1.0'ın altında kalıyor, çünkü
o oran videonun **genişliğe** göre ölçeklendiğini varsayar; `cover` ile dikey kapta ölçek
**yükseklikten** belirleniyor. Karar verici sayı son sütun: 6.24 → 1.36. 16:9 kalıp bir
dosyayla 1.0 oranını tutturmak (1170x658) gerçek büyütmeyi 3.85'e çıkarırdı, yani daha kötü.

**Kadraj ölçümü.** Her hero videosunun 4 karesi gri tona çevrilip sütun başına gradyan
(detay) yoğunluğu ölçüldü; 498 px'lik kayan pencereyle en iyi ofset arandı:

| | 10 banda göre detay | seçilen x0 (merkez) | skor | tepe | orta kırpım |
|---|---|---|---|---|---|
| hero1 salon | soldan sağa artıyor (0.8 → 3.5) | **1330** (%82) | 3.24 | 3.35 | 2.20 (**+%47**) |
| hero2 malzeme | düz (2.4–3.2), masa her yerde dolu | **480** (%38) | 2.90 | 3.03 | 2.69 (+%8) |
| hero3 mutfak | düz (1.3–1.7) | **1080** (%69) | 1.43 | 1.64 | 1.60 |

hero3'te tepe skor mutfak dolabının çekmece çizgilerinden geliyor; kadraj pirinç bataryayı
ve mermer adayı merkeze alacak şekilde seçildi — ölçüm yol gösterici, karar kompozisyon.
`object-position` kullanılmadı: kırpım kaynakta yapıldığı için hem kadraj hem çözünürlük
aynı anda düzeldi.

Dikey hero 828x1794 (crf 30 / vp9 crf 38), yatay hero 1280x720. Bütçe kare setinden
kısılmadan tutturuldu — mobil toplam **1.87–2.18 MB** (en ağır kombinasyon h=2&s=b, iOS mp4).

**B) İlk ekranda video görünürlüğü.** Metin bloğu (etiket + başlık + alt metin + CTA'lar)
küçültüldü. Küçülen yalnızca **başlık ve boşluklar**; gövde 16px, etiket 13px, düğme ≥44px
kaldı.

| | önce | sonra |
|---|---|---|
| H1 (375px) | 42px | 30px (`clamp(28px,8vw,34px)`) |
| etiket satırı | 2 satır (`.22em`) | 1 satır (`.08em`) |
| başlık alt boşluğu | 30px | 16px |
| alt metin satır yüksekliği / boşluk | 1.68 / 40px | 1.5 / 18px |
| hero alt dolgusu | `12vh + 40px` | `68px + safe + 64px` |
| **metin bloğu / ekran (375x812)** | **%53** | **%41** |
| **metin bloğu / ekran (390x844)** | **%52** | **%39.7** |
| metin bloğu alanı / ekran alanı (390) | — | %35.7 |
| üstte kesintisiz video (390x844) | — | 377px = **%44.6** |

Perde (`hero-scrim`) mobilde yeniden kuruldu: masaüstündeki soldan gelen koyu katman
kaldırıldı, üst %13–33 neredeyse şeffaf, metnin başladığı %43'ten sonra sertçe koyulaşıyor.
Kontrast tahmin edilmedi — metin gizlenmiş kareden arka planın **en parlak %5'i** ölçülüp
kompozit metin rengiyle WCAG oranı hesaplandı (güneş vuran duvarın üstü, en kötü durum):

| | 375x812 | 390x844 | gerek |
|---|---|---|---|
| etiket (13px, terra-bright) | **5.19** | **5.46** | 4.5 |
| H1 (30px, frost) | **9.94** | **10.17** | 3.0 |
| alt metin (16px, frost-dim) | **6.26** | **6.35** | 4.5 |

İlk perde denemesi (üstte tamamen şeffaf) ölçümde etiket için **1.16** verdi — o yüzden
%43'ten sonraki koyuluk 0.74'e çekildi. Bu, "video görünsün" hedefini bozmuyor: koyulaşma
metnin zaten kapattığı bandın altında başlıyor.

### 3. tur — "videoda ne olduğu anlaşılmıyor" (Eylül 2026)

Netlik düzeldikten sonra ilk ekranın üst yarısı hâlâ düz duvar/perdeydi. Sorun kadraj
değil **kamera geometrisiydi:** h=1 ve h=3 göz hizasından çekilmiş; mobilya karenin alt
yarısında, üstte duvar/perde/pencere var. Tek bir statik kırpımla mobilyayı üst yarıya
taşımak, karenin dörtte birinden küçük bir alana inmek demekti (büyütme 4.5x).

**Ölçüm.** Metin bloğunun üstünde kalan bölge (ekranın ilk %43'ü) için iki puan hesaplandı:
*detay* (tam çözünürlükte gradyan — perde dokusu da sayar) ve **yapı** (bölge 40px'e
küçültülüp gradyan — ince doku bastırılır, nesne siluetleri kalır). Yapı puanı "burada
tanınır bir şey var mı" sorusunun vekili:

| hero | mevcut kırpımda yapı | en iyi tam-yükseklik | yorum |
|---|---|---|---|
| h1 salon | 11.8 | 12.0 | üstte yalnız perde |
| **h2 malzeme** | **18.2** | **18.8** | tepeden çekim — kare her yerde dolu |
| h3 mutfak | 4.7 | 7.8 | üstte tavan + pencere |

**Karar: mobil varsayılan hero = h2 (malzeme masası).** Tepeden çekildiği için dikey
kırpımda kadraj sorunu yok; üst yarıda keten, pirinç, traverten ve ceviz örnekleri
görünüyor. Üstelik **netlikten hiç ödün verilmedi** — tam yükseklik kırpım (498x1080,
kaynak→ekran 2.35x) korundu. Masaüstü varsayılanı h=1 olarak kaldı, `?h=` her zaman kazanır,
senaryo anahtarı çalışmaya devam ediyor.

Diğer iki hero da (anahtarla seçildiğinde) yeniden kırpıldı — burada netlikten biraz
verildi, çünkü mobilyayı yukarı almanın başka yolu yok:

| hero | kırpım | yapı puanı | kaynak→ekran | üst yarıda görünen |
|---|---|---|---|---|
| h1 | 378x820+1520+260 | 12.5 | 3.10x | kanepe sırtı + yastıklar |
| **h2 (varsayılan)** | **498x1080+320+0** | **18.8** | **2.35x** | keten, pirinç, traverten, ceviz |
| h3 | 434x940+720+140 | 7.8 | 2.70x | pencere + tezgâh + pirinç batarya |

Teslim edilen dosya her üçünde de 828x1794, yani **dosya→ekran büyütme 1.36–1.50x**
(piksellenme yok); değişen, kaynaktan gelen yumuşaklık.

**İlk ekran payı yeniden ölçüldü.** İki CTA yan yana alındı (harf aralığı .06em, uzun olan
iki satıra sarıyor, yükseklik ≥44px, punto 13px):

| | 1. tur sonrası | 2. tur sonrası | **3. tur** |
|---|---|---|---|
| metin bloğu / ekran (375x812) | %53 | %41 | **%35.1** |
| metin bloğu / ekran (390x844) | %52 | %39.7 | **%34.1** |
| metin alanı / ekran alanı (390) | — | %35.7 | **%30.6** |
| üstte kesintisiz video (375) | — | %42.7 | **%48.6** |
| üstte kesintisiz video (390) | — | %44.6 | **%50.3** |

Perde de buna göre kaydırıldı: şeffaf bant %13–40'a genişledi, koyulaşma metnin başladığı
%49'dan sonra. Kontrast yeni (çok daha parlak) kadrajla yeniden ölçüldü — arka planın en
parlak %5'i üzerinde:

| | 375x812 | 390x844 | gerek |
|---|---|---|---|
| etiket (13px) | **6.30** | **6.53** | 4.5 |
| H1 (30px) | **11.79** | **11.93** | 3.0 |
| alt metin (16px) | **6.80** | **6.83** | 4.5 |

Bütçe: varsayılan mobil ziyaret **2.10 MB**, en ağır kombinasyon (iOS mp4, `?s=b`) **2.23 MB**.

### 4. tur — "yazıları ve butonları daha da küçült" (Eylül 2026)

Malzeme masası hero'su işe yaradı; Mert bu kez metin bloğunun kendisini küçültmek istedi.
Hedef: metin bloğu ilk ekranın **en fazla %25'i**, video **en az %55'i**.

| | 3. tur | **4. tur** | hedef |
|---|---|---|---|
| H1 (375px) | 30px | **24,75px** | 24–26 |
| H1 (390px) | 31,2px | **25,7px** | 24–26 |
| alt metin | 16px / 1,5 / 4 satır | **15px / 1,45 / 2 satır** | 15px, ~1,4 |
| etiket (kicker) | 13px | **13px** | 12–13 |
| düğme yüksekliği | 62px | **48px** | 48–52 |
| düğme puntosu | 13px | **13px** | 12–13 |
| **metin bloğu / ekran (375×812)** | %35,1 | **%24,3** | ≤%25 |
| **metin bloğu / ekran (390×844)** | %34,1 | **%23,7** | ≤%25 |
| **üstte kesintisiz video (375)** | %48,6 | **%59,4** | ≥%55 |
| **üstte kesintisiz video (390)** | %50,3 | **%60,7** | ≥%55 |

Puntoyu daha da düşürmek yerine **iki yerde metin kısaltıldı** (koordinatörün talimatı):

1. **Alt metnin 2. cümlesi telefonda gizli** (`<span class="more">`): "Konsept, moodboard,
   3D görsel, malzeme seçimi ve uygulama — hepsi aynı ekipte." 4 satır ilk ekranın %10'unu
   yiyordu ve aynı bilgi Hizmetler bölümünde zaten var. Masaüstünde tam metin duruyor.
2. **İkinci CTA telefonda "Ön Görüşme"** (`<span class="l-short">`). "Ücretsiz Ön Görüşme"
   yan yana düzende iki satıra sarıyor ve düğmeyi 62px yapıyordu. Masaüstünde tam etiket.

Dokunma hedefleri korundu: düğme görsel yüksekliği 48px (alt sınır 44px), etiketler ≥13px,
gövde 15px. Tek istisna **320px** genişlikte hero etiketi 12px — 13px'te iki satıra sarıyor;
12px izin verilen alt sınır ve yalnızca bu tek öğe için geçerli, sitedeki diğer her yazı
≥13px kaldı.

Perdenin şeffaf bandı metin aşağı indiği için %45'e genişletildi, koyulaşma %52,5'ten
sonra. Kontrast yeni puntolarla yeniden ölçüldü (arka planın en parlak %5'i):

| | 375×812 | 390×844 | gerek |
|---|---|---|---|
| etiket (13px) | **6,81** | **6,91** | 4,5 |
| H1 (24,75/25,7px) | **12,48** | **12,48** | 4,5 |
| alt metin (15px) | **6,90** | **6,93** | 4,5 |

Birincil düğme (koyu metin / terra zemin) **6,06**, ikincil düğme frost metin koyu perde
üzerinde 10'un üzerinde.

**Hero CTA çifti korundu.** Diyetisyen ve Duştaş'ta hero CTA'ları alt bara taşındı çünkü
oralarda ekranda iki WhatsApp düğmesi vardı. Burada hero "Projeleri Gör" + "Ön Görüşme",
alt bar "WhatsApp" + "Ara" — işlev tekrarı yok, dört farklı eylem. Kaldırmak yerine
küçültüldüler.

Dar ekranlar: 320px'te düğmeler yine yan yana (içerik genişliğine göre esnek), metin bloğu
%27,8 / video %53,3. 360px'te %25,1 / %57,9.

### Bilinen kalan sorunlar

- **568×320 gibi çok alçak yatay ekranlarda** (iPhone 5 landscape) hero içeriği 320px'e
  sığmıyor; bölüm büyüyor, içerik kırpılmıyor ama CTA için birkaç piksel kaydırmak gerekiyor.
  667×375 ve üstü yatay ekranlarda sığıyor.
- Mobil kare seti 720px; 3x DPR telefonlarda scroll-scrub görüntüsü masaüstü setine göre
  bir tık yumuşak. Bilinçli takas — 1440px set telefonda 7–8 MB ediyordu.
- h1 ve h3 dikey kırpımları kaynak karenin %20–23'ünü gösteriyor; mobilyayı üst yarıya
  almak için daraltıldılar, bu yüzden h2'den bir tık yumuşaklar. Mobil varsayılan h2
  olduğu için ziyaretçilerin çoğu en keskin olanı görüyor.
- h3'te (mutfak) üst üçte bir hâlâ gece mavisi pencere; pirinç batarya ~%40'tan itibaren
  giriyor. Tam yükseklik kırpımda yapı puanı 4.7'den 7.8'e çıktı ama h2 seviyesinde değil.
- Yatay modda (812x375) metin bloğu ekran yüksekliğinin %54'ü; 375px yükseklikte bundan
  kısmak punto kurallarını bozardı. Metin 56vw'lik bir sütuna alındığı için **alan** payı
  %33.8 ve sağ tarafta video net görünüyor. %45 hedefi dikey ekranlar için tutuldu.
- Masaüstünde nav linkleri (11px) ve `brand-sub` (8.5px) 13px'in altında kalmaya devam ediyor;
  brief masaüstü görünümünün korunmasını istediği için dokunulmadı.

---

*Mekanik kaynağı: `mimar-demo` (mimarlık versiyonu) ← `emlak-video-hero/NASIL-YAPILDI.md`
(scroll-scrub reçetesi) + `insaat-web` (hero + palet).*
