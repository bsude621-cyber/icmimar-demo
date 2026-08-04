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
  preview/h1..3.mp4    galeri önizlemeleri (560px, ~150–200 KB)
  preview/sa,sb.mp4    galeri önizlemeleri
frames/a/0001..0120.jpg  scroll-scrub kareleri (1440px)
frames/b/0001..0120.jpg
_raw/               ham Higgsfield çıktıları — DAĞITILMAZ (.vercelignore)
_tools/             npm ffmpeg/ffprobe — DAĞITILMAZ
```

Dağıtılan toplam **26 MB** (media 11 MB + frames 16 MB). Ziyaretçi başına indirilen:
1 hero webm (0.8–1.3 MB) + 120 kare (7.3–8.0 MB); mobilde `STEP=2` ile kareler yarıya iner.

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
| Mobil kare seyreltme | JS `STEP = isMobile ? 2 : 1` |
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
   `class="nav-demo"` sekmeleri, kullanılmayan `media/hero*` ve `frames/*` setleri.
4. **Firma bilgilerini değiştir** (liste `index.html` başındaki yorum bloğunda):
   firma adı + `brand-mono` harfleri, telefon, WhatsApp, e-posta, adres,
   `data-count` istatistikleri, `<title>` ve `<meta name="description">`.
5. **Proje fotoğraflarını koy.** Proje kartlarında `.proj-art` SVG'lerini gerçek
   fotoğrafla değiştir: `<div class="proj-art">…</div>` → `<img src="projeler/1.jpg" alt="…">`
   (stil aynı kalır, `aspect-ratio:4/5`).
6. `about-visual` içindeki izometrik oda SVG'si de bir mekân fotoğrafıyla değişebilir.

**Not:** WhatsApp'tan HTML dosyası göndermek işe yaramaz — `index.html` tek başına
videoları içermez ve telefonda düzgün açılmaz. Her zaman canlı link gönder.

---

*Mekanik kaynağı: `mimar-demo` (mimarlık versiyonu) ← `emlak-video-hero/NASIL-YAPILDI.md`
(scroll-scrub reçetesi) + `insaat-web` (hero + palet).*
