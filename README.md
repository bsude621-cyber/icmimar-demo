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
  hero1..3.mp4/.webm   16 sn dikişsiz boomerang loop, 1600x900, sessiz
  hero1..3-m.mp4/.webm MOBİL hero: 720px, aynı loop (220–365 KB)
  preview/h1..3.mp4    galeri önizlemeleri (560px, ~150–200 KB)
  preview/sa,sb.mp4    galeri önizlemeleri
frames/a/0001..0120.jpg  scroll-scrub kareleri (1440px, masaüstü)
frames/b/0001..0120.jpg
frames/a-m/, frames/b-m/ MOBİL kare seti (720px, her 3. kare + son kare = 41 kare)
_raw/               ham Higgsfield çıktıları — DAĞITILMAZ (.vercelignore)
_tools/             npm ffmpeg/ffprobe + ekran görüntüleri — DAĞITILMAZ
```

Dağıtılan toplam **30 MB** (media 13 MB + frames 18 MB). Ziyaretçi başına indirilen:
· **masaüstü:** 1 hero webm (0.8–1.3 MB) + 120 kare (7.3–8.0 MB) ≈ **8.2 MB**
· **telefon:** 1 mobil hero (0.22–0.37 MB) + 41 mobil kare (0.82–0.95 MB) ≈ **1.14 MB** (ölçüldü)

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
5. **Mobil medya seti:** telefonda 720px `hero<N>-m.*` ve 41 karelik `frames/<a|b>-m/` iniyor.
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

### Bilinen kalan sorunlar

- **568×320 gibi çok alçak yatay ekranlarda** (iPhone 5 landscape) hero içeriği 320px'e
  sığmıyor; bölüm büyüyor, içerik kırpılmıyor ama CTA için birkaç piksel kaydırmak gerekiyor.
  667×375 ve üstü yatay ekranlarda sığıyor.
- Mobil kare seti 720px; 3x DPR telefonlarda scroll-scrub görüntüsü masaüstü setine göre
  bir tık yumuşak. Bilinçli takas — 1440px set telefonda 7–8 MB ediyordu.
- Masaüstünde nav linkleri (11px) ve `brand-sub` (8.5px) 13px'in altında kalmaya devam ediyor;
  brief masaüstü görünümünün korunmasını istediği için dokunulmadı.

---

*Mekanik kaynağı: `mimar-demo` (mimarlık versiyonu) ← `emlak-video-hero/NASIL-YAPILDI.md`
(scroll-scrub reçetesi) + `insaat-web` (hero + palet).*
