# Ejder Yolu — Hata Raporu ve Geliştirme Yol Haritası

Bu belge `index.html` üzerinde yapılan hata denetiminin sonuçlarını, bu turda
uygulanan geliştirmeleri ve sıradaki yükseltme fikirlerini içerir.

---

## 1. Bulunan ve düzeltilen hatalar

### 1.1 Kritik

**Oyun hiçbir zaman kaydedilmiyordu.**
`Store.get/set` yalnızca `window.storage` API'sini deniyordu. Bu API normal bir
tarayıcıda tanımlı değildir, dolayısıyla `save()` her seferinde sessizce hiçbir
şey yapmıyordu. Sayfayı her yenileyen oyuncu sıfırdan başlıyordu; çevrimdışı
kazanç ekranı da hiç görünmüyordu.
*Düzeltme:* `window.storage` varsa o, yoksa `localStorage` kullanılıyor. Ayrıca
sürüm alanı (`v`) ve eski kayıtları yeni alanlara taşıyan `migrate()` eklendi.

**Panel açıkken alt menüye basılamıyordu.**
`.sheet` `z-index:9`, `.nav` ise `z-index:7` idi. Açık panelin alt dolgusu
menünün tamamını kaplıyordu; bir panel açıldıktan sonra başka panele geçmek ya
da menüden kapatmak imkânsızdı (yalnızca ✕ düğmesi çalışıyordu).
*Düzeltme:* `.nav` `z-index:11` yapıldı. Tarayıcı testinde doğrulandı.

**Yetenek öldürücü darbe indirdiğinde oyun hata veriyordu.**
`castSkill()` içinde `hitMob(...)` çağrısı canavarı öldürürse `kill()` `E`'yi
`null` yapıyor, hemen ardından gelen `burst(E.x, ...)` satırı
`TypeError: Cannot read properties of null` fırlatıyordu. Bu, o karede
`render()`, `syncHUD()` ve `save()` adımlarının atlanmasına yol açıyordu.
*Düzeltme:* Konum `hitMob` çağrısından önce yerel değişkene alınıyor.

**Derin bölgelerde savaş kilitlenebiliyordu.**
Hasar `Math.max(1, hasar - E.def)` ile hesaplanıyordu ve `E.def` bölgeyle
üstel (`2.2^z`) büyüyordu. Oyuncunun saldırısı bu hızda büyümediği için vuruş
başına hasar 1'e sabitleniyordu. Metin taşları saldırmadığı için oyuncu ölmüyor
da; savaş sonsuza kadar sürüyordu.
*Düzeltme:* Düz çıkarma yerine oransal azaltma (`mobMit()`, en fazla %30).

### 1.2 Orta

**Şifa yeteneği beklemesini boşa yakıyordu.**
`castSkill()` içindeki `S.skcd[i]=2` satırı, çağıran döngüde hemen
`S.skcd[i]=SKILL_DEF[i].cd` ile eziliyordu. Can doluyken kullanılan şifa 22
saniyelik beklemeyi başlatıyor, oyuncu hasar aldığında yetenek hazır olmuyordu.
*Düzeltme:* `castSkill()` artık `true/false` döndürüyor; kullanılamayan yetenek
yalnızca 1.2 sn sonra tekrar deneniyor. Şifa eşiği %95 → %80 yapıldı.

**Yetenek beklemeleri canavar yokken duruyordu.**
Bekleme azaltma kodu `if(!E) return;` satırının altındaydı.
*Düzeltme:* Bekleme sayacı düşman yokken de işliyor.

**Sekme arka plandayken geçen süre yok sayılıyordu.**
Çevrimdışı kazanç yalnızca sayfa yeniden yüklendiğinde hesaplanıyordu. Oyunu
sekmede bırakıp 3 saat sonra dönen oyuncu hiçbir şey kazanmıyordu.
*Düzeltme:* `visibilitychange` ile geri dönüşte de hesaplanıyor.

**Panel her öldürmede yeniden çiziliyordu.**
`kill()` içindeki `refreshSheet()`, saniyede birkaç kez `innerHTML`'i baştan
kuruyordu. Bu, kaydırma konumunu bozuyor ve parmağın altındaki düğmeyi DOM'dan
kopararak dokunuşu boşa çıkarıyordu.
*Düzeltme:* Yenileme 0.7 sn'de bir; dokunma sırasında ve dokunuştan sonraki
260 ms boyunca erteleniyor. Kaydırma konumu `withScroll()` ile korunuyor.

**Dokunma hasarında sınır yoktu.**
`pointerdown` her olayda `ST.atk*0.6` hasar veriyordu; hızlı dokunan oyuncu
otomatik savaşın onlarca katı hasar üretebiliyordu.
*Düzeltme:* En az 110 ms ara + seri (combo) ödülü (0.55× → 1.00×).

**Ekran döndürüldüğünde canavar ekran dışında kalıyordu.**
`resize()` yalnızca tuvali güncelliyordu; `E.x` ve `E.sc` eski değerlerde
kalıyordu.
*Düzeltme:* `resize()` yaşayan düşmanı yeniden konumlandırıp ölçekliyor.

### 1.3 Küçük

| Sorun | Düzeltme |
|---|---|
| `drawSkillBar()` içinde `parseInt(getComputedStyle(...--safe-b))` her zaman `NaN` döndürüyordu | Menü konumu `getBoundingClientRect()` ile ölçülüyor |
| Yükselişten sonra HUD'daki isim güncellenmiyordu | `syncIdentity()` eklendi |
| `zoneChip` tıklanabilir görünüyor ama hiçbir şey yapmıyordu | Haritayı açıyor |
| Ses bağlamı jest dışında açılmaya çalışılıyordu | İlk dokunuşta `unlockAudio()` |
| Yükselişte `totalKills` sıfırlanıyordu ("toplam" olmasına rağmen) | Ömür boyu sayaçlar (`S.lt`) ayrıldı |
| `gainExp` koruma sayacı 400'de tecrübeyi sessizce yutuyordu | 5000'e çıkarıldı, `NaN` koruması eklendi |
| `mobStats` içinde kullanılmayan `zi` değişkeni ve etkisiz `(1+zone*0.02)` çarpanı | Temizlendi |
| `spawn()` içinde gereksiz ikinci `E.name` ataması | Kaldırıldı |
| Bozuk/eksik kayıt oyunu kırabiliyordu | `migrate()` tüm alanları doğruluyor |

---

## 2. Bu turda eklenen geliştirmeler

**Görevler paneli (yeni sekme).** 14 kademeli başarım: öldürme, metin, şef,
seviye, demirci ve yükseliş hedefleri. Her biri yang / yükseltme taşı / kitap /
durum puanı veriyor. Ödüller elle alınıyor, ilerleme çubuğuyla gösteriliyor.
Başarımlar ve ömür boyu istatistikler ruh yükselişinde sıfırlanmıyor.

**Şef çağırma.** Eskiden her bölge şefi yalnızca bir kez öldürülebiliyordu ve
kitap kaynağı 7 bölge × 2 = 14 ile sınırlıydı; bu, yetenek ağacını erkenden
tıkıyordu. Artık devrilen şef 🔷 `15 + bölge×10` karşılığında yeniden
çağrılabiliyor (tekrar ödülü +1 📕).

**Sıyrılma (dodge) istatistiği.** Çeviklik artık kritik ve hıza ek olarak
saldırı savuşturma sağlıyor (Ninja +%6 doğuştan). Durum panelinde gösteriliyor.

**Alt menü bildirim işaretleri.** Yükseltilebilir eşya, öğrenilebilir yetenek,
alınmayı bekleyen ödül veya dağıtılmamış durum puanı varsa ilgili sekmede
yanıp sönen nokta beliriyor.

**Ayarlar.** Ses ve titreşim ayrı ayrı açılıp kapanabiliyor (eskiden tek
anahtardı), otomatik bölge geçişi, düşük efekt modu (zayıf telefonlar için
parçacıkları ~%65 azaltır) ve iki aşamalı onaylı kayıt sıfırlama eklendi.

**Genişletilmiş durum paneli.** Saldırı hızı, sıyrılma, can yenileme ve
saniyelik öldürme hızı eklendi. Görevler panelinde ömür boyu istatistikler
(canavar, metin, şef, ölüm, en yüksek seviye, oynama süresi) var.

---

## 3. Sıradaki yükseltme fikirleri

### 3.0 Tamamlananlar

Aşağıdakiler ayrı PR'larla eklendi:

- ✅ **Otomatik demirci** (#2) — 1.5 sn'de bir en ucuz karşılanabilir parçayı
  yükseltir.
- ✅ **Savaş sayacı** (#2) — son 10 saniyenin gerçek DPS'i, en yüksek DPS,
  dakikada öldürme, ölçülen kritik oranı.
- ✅ **Çevrimdışı süre yükseltmesi** (#2) — ruh taşı başına +30 dk, 12→24 saat.
- ✅ **Günlük görevler ve seri** (#3) — her gün üç hedef, seri ödülleri %70'e
  kadar artırır.
- ✅ **Envanter ve eşya düşürme** (#4) — beş nadirlik, yedi nitelik türü,
  otomatik kuşanma, toplu satış.

### 3.1 Sıradaki — yüksek etki

1. **Set bonusu.** Aynı nadirlikten 3/6 parça kuşanınca eşik bonusu. Envanter
   altyapısı hazır olduğu için ucuz; eşya toplamaya yön verir.
2. **Eşya basma / birleştirme.** İki eşyayı birleştirip nitelik aktarma ya da
   yükseltme taşıyla eşya niteliği yeniden atma. Çantanın uzun vadeli anlamı
   olur.
3. **Yetenek elle kullanım seçeneği.** Otomatik kullanımın yanında yetenek
   simgesine dokunarak erken tetikleme.

### 3.2 Orta maliyet

4. **Evcil hayvan / ejder yoldaşı.** Kendi seviyesi ve beslenme kaynağı olan,
   pasif hasar veya yang bonusu sağlayan bir yoldaş. Metin2 ruhuna çok uygun.
5. **Ruh ağacı.** Ruh taşları düz çarpan yerine harcanabilir puan olsun:
   çevrimdışı verim, yükseltme başarı şansı, eşya düşme şansı gibi dallar.
   Yükselişi bir seçim haline getirir.
6. **Zindan / dalga modu.** Süreli, artan zorlukta dalgalar; kitap ve nadir
   malzeme veren ayrı bir kaynak.
7. **Elementler ve zayıflıklar.** Sınıf ve canavar tipleri arasında taş-kâğıt-
   makas ilişkisi; bölge seçimine taktik katar.

### 3.3 Uzun vadeli

8. **PWA / çevrimdışı çalışma.** `manifest.json` + service worker ile ana
   ekrana eklenebilir, internetsiz açılabilir hale getirme. Şu anda yazı
   tipleri Google Fonts'tan çekiliyor; bunları gömmek ilk adım olur.
9. **Bulut kaydı.** Kayıt dizesi dışa/içe aktarma (kopyalanabilir metin) en
   ucuz çözüm; sonrasında hesap tabanlı senkron.
10. **Sıralama tablosu.** En yüksek seviye / en çok yükseliş.
11. **Sınıf başına özgün yetenek görselleri.** Şu anda dört sınıf aynı dört
    efekti paylaşıyor; yalnızca isimler farklı.
12. **Ses tasarımı.** Osilatör bipleri yerine kısa örneklenmiş sesler.

---

## 4. Test notları

Değişiklikler Chromium'da (Playwright) 390×844 ve 844×390 çözünürlüklerde
uçtan uca doğrulandı:

- sınıf seçimi → savaş → seviye atlama → ölüm → yeniden doğuş
- altı panelin tamamının açılması, panel arası geçiş, kaydırma korunması
- ekipman yükseltme, yetenek öğrenme, görev ödülü alma (14/14)
- şef çağırma ve tekrar ödülü, otomatik bölge geçişi
- kayıt → sayfa yenileme → kaydın geri yüklenmesi
- sekme dönüşünde çevrimdışı kazanç
- ekran döndürme, düşük efekt modu, ses/titreşim anahtarları
- 6 saniyelik yoğun savaş yükü: parçacık/hasar yazısı sızıntısı yok

Konsolda hata yok. (Yalnızca çevrimdışı sanal ortamda Google Fonts'a
erişilemediği için kaynak yükleme uyarısı görülüyor; oyun sistem yazı tipine
düşerek normal çalışıyor.)
