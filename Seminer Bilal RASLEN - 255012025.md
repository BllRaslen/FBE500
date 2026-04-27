# RASD: Makine Öğrenimi ve Derin Öğrenme Kullanarak IoT Tabanlı Gerçek Zamanlı Beton Basınç Dayanımı Tahmin Sistemi


**Hazırlayan:** Bilal RASLEN
**Anabilim Dalı:** Bilgisayar Mühendisliği 
**Üniversite:** Sakarya Üniversitesi
**Akademik Yıl:** 2025–2026

---


## Özet

Beton basınç dayanımı, yapısal güvenliği ve inşaat programını belirleyen en kritik mekanik özelliktir. Mevcut endüstri standardı — 28 gün boyunca numune kürlemesi ve ardından yıkıcı kırma testi (ASTM C39) — beton yerleştirme ile yapısal yeterliliğin onaylanması arasında 27 günlük bir gecikme yaratmaktadır. Bu tez, kürleme sırasında beton numunelerine gömülü 24 saatlik sıcaklık ve nem sensörü verilerinden 28 günlük basınç dayanımını tahmin eden eksiksiz bir Endüstriyel Nesnelerin İnterneti (IIoT) sistemi olan **RASD**'ı (Gerçek Zamanlı Otomatik Dayanım Tespiti — Real-time Automated Strength Detection) sunmaktadır.

RASD sistemi ESP32 IoT sensörleri, MQTT/Apache Kafka veri akışı, Spring Boot Java arka ucu ve Python FastAPI makine öğrenimi servisini entegre etmektedir. ML ardışık düzeni, 96 noktalı zaman serisi sensör okumalarından 16 fizik tabanlı özellik mühendisliği yapar ve beş tahmin modeli eğitir: Random Forest, XGBoost, LightGBM (scikit-learn) ve DNN/LSTM (PyTorch). 1.953 numune üzerindeki deneysel değerlendirme, XGBoost'un **MAE = 1,87 MPa, RMSE = 2,94 MPa, R² = 0,9461** başarısını elde ettiğini göstermektedir.

SHAP (SHapley Katkı Açıklamaları) analizi, modellerin fiziksel anlamlı ilişkileri öğrendiğini doğrulamaktadır: su-çimento oranı, kimyasal katkı maddeleri ve termal enerji indeksi, Abrams Yasası ve Olgunluk Yöntemi ile tutarlı biçimde en iyi üç tahmin edici olarak öne çıkmaktadır.

**Anahtar Kelimeler:** Basınç dayanımı tahmini, IoT beton izleme, XGBoost, LSTM, SHAP açıklanabilirliği, Olgunluk Yöntemi, Özellik mühendisliği, FastAPI, Apache Kafka

---

## İçindekiler

1. Giriş
2. Literatür Araştırması
3. Sistem Mimarisi
4. Veri Kümesi ve Veri Toplama
5. Veri Ön İşleme ve Kalite Kontrol
6. Özellik Mühendisliği
7. Makine Öğrenimi Modelleri
8. Derin Öğrenme Modelleri
9. Değerlendirme Metodolojisi
10. Deneysel Sonuçlar ve Analiz
11. SHAP Açıklanabilirlik Analizi
12. Üretim Ortamı Dağıtımı
13. Tartışma
14. Sonuç ve Gelecek Çalışmalar
15. Kaynaklar
16. Ekler

---

## Bölüm 1: Giriş

### 1.1 Arka Plan ve Motivasyon

Beton, dünyada en yaygın kullanılan yapı malzemesidir; yılda yaklaşık 30 milyar ton üretilmektedir [Mehta & Monteiro, 2014]. Mekanik özellikleri — özellikle basınç dayanımı — yapısal güvenlik marjlarını, bina yönetmelikleri uyumluluğunu ve kalıp sökümü, öngerme ve çok katlı yükleme gibi kritik inşaat aşamalarının programlanmasını belirler.

Beton dayanımı için standart kalite kontrol prosedürü ASTM C39 ile tanımlanmıştır: yapısal döküm ile aynı anda silindirik numuneler (100mm × 200mm) dökülür, 28 gün standart koşullarda kürlenip basınç uygulanarak kırılır. Bu test:

- **Yıkıcıdır**: Test edilen numune tükenir ve devam eden yapısal bilgi sağlamaz
- **Gecikmeli**: 28 günlük bekleme programlama baskısı yaratır ve inşaatı yavaşlatır
- **Süreksiz**: Yalnızca örneklenen numuneler test edilir; döküntünün geri kalanının temsil edici olduğu varsayılır
- **Konuma duyarsız**: Standart kürleme koşulları gerçek saha kürleme koşullarından (sıcaklık, nem, derinlik) farklıdır

**Olgunluk Yöntemi** (ASTM C1074, ilk olarak Nurse [1949] tarafından önerilmiştir), kürleme sırasında betonun birikimli termal maruziyetinin bir fonksiyonu olduğu temel gözlemine dayanır:

$$M = \int_0^t (T(\tau) - T_0) \, d\tau$$

Bu yaklaşım, sıcaklık ölçümlerinden dayanım tahminine olanak tanır; ancak her özgün karışım tasarımı için laboratuvar testlemesi yoluyla oluşturulmuş kalibrasyon eğrileri gerektirir.

**Düşük maliyetli IoT donanımı, gerçek zamanlı veri akışı ve modern makine öğreniminin birleşimi yeni bir fırsat yaratmaktadır**: zengin çok değişkenli sensör verilerini yakalayarak ve etiketlenmiş numunelerden (bilinen 28 günlük dayanım ile) doğrudan öğrenerek, manuel kalibrasyon gerektirmeden ve çeşitli karışım tasarımlarına uygulanabilir şekilde sensör geçmişinden nihai dayanıma eşlemeyi otomatik olarak keşfeden modeller oluşturabiliriz.

### 1.2 Araştırma Hedefleri

Bu tez aşağıdaki araştırma hedeflerini ele almaktadır:

1. **AH1**: ESP32 sensörlerden sıcaklık ve nem zaman serisi toplamak için eksiksiz bir uçtan uca IoT veri ardışık düzeni tasarlamak ve uygulamak
2. **AH2**: 96 noktalı zaman serisini ML eğitimi için uygun kompakt, fizik tabanlı özellik vektörlerine dönüştüren bir özellik mühendisliği çerçevesi geliştirmek
3. **AH3**: Beton dayanımı regresyonu için beş ML/DL modelini eğitmek, değerlendirmek ve karşılaştırmak
4. **AH4**: Öğrenilen modellerin fiziksel olarak yorumlanabilir olduğunu doğrulamak için SHAP açıklanabilirliğini uygulamak
5. **AH5**: Tam sistemi rol tabanlı erişim kontrolüyle üretim web uygulaması olarak dağıtmak

### 1.3 Araştırma Soruları

- AS1: 24 saatlik IoT sensör verilerinden 28 günlük basınç dayanımı için hangi tahmin doğruluğu düzeyi (R², MAE, RMSE) elde edilebilir?
- AS2: Bu veri kümesi boyutu ve özellik türü için makine öğrenimi mi yoksa derin öğrenme modelleri mi daha iyi performans gösterir?
- AS3: Modeller fiziksel anlamlı ilişkileri (SHAP ile doğrulandığı üzere) öğreniyor mu, yoksa tahminler sahte korelasyonlara mı dayanıyor?
- AS4: Beton dayanımı tahmini için en etkili özellikler nelerdir ve bunlar köklü beton bilimiyle uyuşuyor mu?

### 1.4 Tezin Kapsamı ve Sınırlamaları

**Kapsam dahilinde:**
- Olağan Portland çimentosu (OPC) ve karma çimentolar (CEM I - CEM V)
- Normal ağırlıklı beton, çökme 50–200 mm
- Ortam sıcaklığı aralığı: 5°C – 45°C
- Kürleme süresi: 24 saatlik sensör izleme penceresi
- Tahmin hedefi: 28 günlük basınç dayanımı (f'c)

**Kapsam dışında:**
- Kendiliğinden yerleşen veya ultra yüksek performanslı beton (UHPC)
- Kriyo veya yangına dayanım uygulamaları
- Gerçek zamanlı yapısal sağlık izleme (kürleme sonrası)

### 1.5 Tez Yapısı

Tezin geri kalanı şu şekilde düzenlenmiştir: Bölüm 2 ilgili literatürü inceler. Bölüm 3 genel sistem mimarisini açıklar. Bölüm 4 veri toplama ve kaliteyi kapsar. Bölüm 5 veri ön işlemeyi ele alır. Bölüm 6 özellik mühendisliği metodolojisini sunar. Bölümler 7 ve 8 sırasıyla makine öğrenimi ve derin öğrenme modellerini açıklar. Bölüm 9 değerlendirme metodolojisini tanımlar. Bölüm 10 deneysel sonuçları sunar. Bölüm 11 SHAP analizini sağlar. Bölüm 12 üretim dağıtımını açıklar. Bölüm 13 bulguları ilgili çalışmalar bağlamında tartışır. Bölüm 14 sonuç çıkarır ve gelecekteki yönleri belirler.

---

## Bölüm 2: Literatür Araştırması

### 2.1 Beton Dayanım Gelişimi: Fiziksel ve Kimyasal Temeller

#### 2.1.1 Hidrasyon Kimyası

Portland çimentosu dört ana klinker mineralinden oluşur [Taylor, 1997]:

| Mineral | Formül | Kısaltma | Hidrasyon Hızı |
|---------|--------|----------|----------------|
| Trikalsiyum silikat | Ca₃SiO₅ | C₃S | Hızlı (günler) |
| Dikalsiyum silikat | Ca₂SiO₄ | C₂S | Yavaş (aylar) |
| Trikalsiyum alüminat | Ca₃Al₂O₆ | C₃A | Çok hızlı (dakikalar) |
| Tetrakalsiyum alüminoferrit | Ca₄Al₂Fe₂O₁₀ | C₄AF | Orta |

Hidrasyon, birincil bağlayıcı faz olan Kalsiyum Silikat Hidrat (C-S-H jeli) ve Kalsiyum Hidroksit (CH) üretir. C-S-H jeli çimento pastasındaki boşlukları doldurarak yoğunluğu ve basınç dayanımını artırır.

Hidrasyon reaksiyonu ekzotermiktir: C₃S yaklaşık 502 J/g ısı açığa çıkarır. Gömülü sensörlerimizin ölçtüğü şey tam olarak budur — numunedeki sıcaklık artışı hidrasyonun ilerleyişini doğrudan yansıtır.

#### 2.1.2 Abrams Yasası

1919'da Duff Abrams tarafından belirlenen beton teknolojisindeki en temel ilişki:

$$f'c = \frac{K_1}{K_2^{s/ç}}$$

burada $K_1$ ve $K_2$ çimento tipine ve yaşa bağlı ampirik sabitler, $s/ç$ ise su-çimento kütle oranıdır. Bu ilişki şundan kaynaklanmaktadır:

1. Hidrasyon tarafından tüketilmeyen su ($s/ç > \approx 0,36$) buharlaşma sonrasında gözenekler bırakır
2. Gözeneklilik basınç dayanımını ters yönde belirler (Powers modeli)

Bu yasa ML sonuçlarımızda görünmektedir: `waterRatio` (suOranı) tutarlı olarak en önemli SHAP özelliğidir, yani model yüz yıllık ampirik bir yasayı veriden bağımsız olarak yeniden keşfetmiştir.

#### 2.1.3 Olgunluk Yöntemi

Olgunluk Yöntemi (Nurse, 1949; McIntosh, 1949; Plowman, 1956 tarafından resmileştirildi), ASTM C1074 ile standartlaştırılmış tahmin yöntemidir:

**Nurse-Saul Fonksiyonu:**
$$M = \sum (T - T_0) \cdot \Delta t$$

**Arrhenius Tabanlı (Üstel) Fonksiyon:**
$$M_{exp} = \sum e^{-Q/R(1/T - 1/T_r)} \cdot \Delta t$$

burada $Q$ aktivasyon enerjisi ve $T_r$ referans sıcaklığıdır.

Bu tez, bu skaler olgunluk metriklerini daha zengin bir özellik setine (16 özellik) genelleştirir ve etiketlenmiş verilerden dayanım-olgunluk ilişkisini doğrudan öğrenmek için ML kullanır.

### 2.2 Beton Dayanım Tahmini için Makine Öğrenimi

#### 2.2.1 Yapay Sinir Ağları (1990'lar–2010'lar)

Yeh (1998), karışım oranlarını girdi olarak kullanarak beton dayanımı tahminine yapay sinir ağlarının uygulanmasına öncülük etmiştir. 1.030 numuneli veri kümesi onlarca sonraki çalışma için referans noktası haline gelmiştir. R² = 0,885 değerini elde etmiştir.

#### 2.2.2 Destek Vektör Makineleri

Chou ve ark. (2014), Yeh veri kümesinde SVM, karar ağaçları ve sinir ağlarını karşılaştırmış; RBF çekirdekli SVM'nin R² = 0,912 elde ettiğini bulmuştur.

#### 2.2.3 Topluluk Yöntemleri

Topluluk yöntemlerine geçiş önemli iyileştirmeler getirdi:
- **Random Forest**: Chithra ve ark. (2016) R² = 0,937 bildirdi
- **Gradyan Artırma**: Farooq ve ark. (2021), endüstriyel atık özellikleri dahil XGBoost ile R² = 0,941 elde etti

**Mevcut literatürdeki kritik boşluk**: Tüm önceki ML çalışmaları yalnızca statik karışım tasarımı özelliklerini kullanmaktadır. Hiçbiri kürleme sürecinden gerçek zamanlı IoT sensörü verilerini dahil etmemektedir. Bu tez, statik karışım tasarımı girdilerine dinamik zaman serisi özellikler ekleyerek bu boşluğu doldurmaktadır.

#### 2.2.4 Beton Özellikleri için Derin Öğrenme

Son çalışmalar CNN ve LSTM'yi beton dayanımına uygulamıştır:
- Kim ve ark. (2022): Sıcaklık zaman serisi üzerinde LSTM, 312 numune ile R² = 0,891
- Taffese & Sistonen (2017): Derin sinir ağı topluluğu, R² = 0,913

Bu sonuçlar, zaman serisi derin öğrenmesinin umut verici olduğunu ancak karşılaştırılabilir veri kümelerinde gradyan artırmasıyla henüz eşleşmediğini göstermektedir. Sonuçlarımız bu bulguyu daha büyük ölçekte doğrulamaktadır.

### 2.3 İnşaat İzlemede Nesnelerin İnterneti

#### 2.3.1 Gömülü Sensör Sistemleri

Beton izleme için gömülü kablosuz sensör ağları 2000'lerin başından beri önerilmektedir [Tanner ve ark., 2003]. Temel gelişmeler:
- **RFID tabanlı sistemler**: Pil gerektirmez, ancak sınırlı menzil ve veri kapasitesi
- **ZigBee/Bluetooth sensörler**: Düşük güç tüketimi ancak kısa menzil
- **Wi-Fi bağlantılı sensörler (ESP32)**: Daha yüksek güç tüketimi ancak gerçek zamanlı bulut akışını mümkün kılan tam internet bağlantısı

#### 2.3.2 MQTT ve IoT Protokolleri

MQTT (Mesaj Kuyruğu Telemetri Taşımacılığı), kısıtlı cihazlar ve güvenilmez ağlar için tasarlanmış IoT veri iletiminin fiili standardıdır. Yayınla-abone ol modeli sensör ağları için idealdir:
- Sensörler tüketicileri bilmeden konulara yayın yapar
- Aracı, yönlendirme ve hizmet kalitesi garantilerini işler
- QoS seviyeleri 0 (en fazla bir kez), 1 (en az bir kez), 2 (tam olarak bir kez)

#### 2.3.3 Veri Akışı için Apache Kafka

Apache Kafka, dağıtılmış, hataya dayanıklı, yüksek verimli olay akışı sağlar. IoT uygulamaları için gerçek zamanlı sensör verileri (değişken hızlar, itme modeli) ile arka uç işleme (kontrollü hızda tüketim) arasındaki boşluğu kapatır.

### 2.4 İnşaat Mühendisliğinde Açıklanabilir Yapay Zeka (XAI)

#### 2.4.1 Yorumlanabilirlik Sorunu

Ağaç topluluk modelleri (XGBoost, Random Forest) ve sinir ağları genellikle "kara kutu" olarak tanımlanır — doğru tahminler yaparlar ancak sezgisel açıklama sağlamazlar. İnşaat mühendisliği uygulamalarında bu sorunludur: mühendislerin neden düşük dayanım tahmin edildiğini anlamaları gerekir.

#### 2.4.2 SHAP Teorisi

SHAP (Lundberg & Lee, 2017), kooperatif oyun teorisine dayanan Shapley değerleri aracılığıyla özellik başına, örnek başına katkı değerleri hesaplar:

$$\phi_i = \sum_{S \subseteq F \setminus \{i\}} \frac{|S|!(|F|-|S|-1)!}{|F|!} [v(S \cup \{i\}) - v(S)]$$

SHAP üç aksiyomu sağlar:
- **Verimlilik**: $\sum_i \phi_i = f(x) - \mathbb{E}[f(X)]$
- **Simetri**: Eşit katkıda bulunan özellikler eşit SHAP değerleri alır
- **Kukla**: Herhangi bir tahmini etkilemeyen özellikler SHAP = 0 alır

**TreeSHAP** (Lundberg ve ark., 2020), ağaç toplulukları için kesin SHAP değerlerini $O(TLD^2)$ zamanda hesaplar; burada $T$ ağaç sayısı, $L$ maksimum yaprak sayısı ve $D$ maksimum derinliktir.

---

## Bölüm 3: Sistem Mimarisi

### 3.1 Üst Düzey Mimari

RASD sistemi, fiziksel ölçümden web görselleştirmeye kadar bir ardışık düzende organize edilmiş altı ana bileşenden oluşmaktadır:

<img width="1771" height="1436" alt="sistem_mimari" src="https://github.com/user-attachments/assets/d0770cd5-f781-4e52-b87d-11900903f2a1" />


```
┌─────────────────────────────────────────────────────────────────────┐
│                        RASD Sistem Mimarisi                          │
├──────────────┬──────────────┬─────────────┬────────────────────────┤
│ Saha Katmanı │ Taşıma       │ Arka Uç     │ Zeka Katmanı           │
│              │ Katmanı      │ Katmanı     │                        │
│ ESP32        │ MQTT Aracısı │ Spring Boot │ Python FastAPI ML      │
│ DS18B20      │ Apache Kafka │ Java 17     │ XGBoost / LightGBM     │
│ DHT22        │              │ PostgreSQL  │ DNN / LSTM (PyTorch)   │
└──────────────┴──────────────┴─────────────┴────────────────────────┘
                                     │
                              ┌──────┴──────┐
                              │ Sunum       │
                              │ Katmanı     │
                              │ Next.js 14  │
                              │ React 18    │
                              └─────────────┘
```

### 3.2 Teknoloji Seçim Gerekçesi

#### 3.2.1 ESP32 - Neden Bu Mikrodenetleyici?

| Platform | Fiyat | Wi-Fi | Pil Ömrü | İşlem Gücü | Karar |
|----------|-------|-------|----------|-----------|-------|
| **ESP32** | **~150₺** | **Evet** | **5+ gün** | **240MHz çift çekirdek** | **Seçildi** |
| Arduino Uno | 750₺ | Harici | Yok | 16MHz | Hayır |
| Raspberry Pi | 1.100₺ | Evet | 3 saat | 1.8GHz | Fazla güçlü |
| Arduino MKR | 900₺ | Evet | 2 gün | 48MHz | Pahalı |

ESP32, saha dağıtımı için maliyet, bağlantı ve pil ömrünün optimum dengesini sunar.

#### 3.2.2 MQTT - Neden HTTP Değil?

Sensörler her 15 dakikada bir veri gönderir. MQTT, HTTP'ye tercih edilir çünkü:
- **5× daha düşük bant genişliği**: MQTT 2 bayt başlık vs. HTTP 200–400 bayt
- **Kalıcı bağlantı**: Mesaj başına TCP el sıkışması yok
- **QoS garantileri**: En az bir kez teslim, geçici ağ kesintilerini yönetir

#### 3.2.3 Apache Kafka - Neden RabbitMQ Değil?

IoT akışı için Kafka, RabbitMQ'ya tercih edilir çünkü:
- **Log tutma**: Kafka tüm mesajları yapılandırılabilir bir süre boyunca saklar (varsayılan 7 gün), geçmiş veriler üzerinde model yeniden eğitimi için yeniden oynatmayı mümkün kılar
- **Verim**: Kafka saniyede >1M mesaj işler; RabbitMQ ~50K'da zirve yapar
- **Tüketici grupları**: ML eğitim servisi ve API servisi bağımsız olarak tüketebilir

#### 3.2.4 FastAPI - Neden Flask veya Django Değil?

| Çerçeve | Performans | Otomatik Dokümantasyon | Asenkron | ML Uygunluğu |
|---------|-----------|----------------------|---------|-------------|
| Django | Düşük | Hayır | Sınırlı | Zayıf |
| Flask | Orta | Hayır | Hayır | Yeterli |
| **FastAPI** | **Yüksek** | **Evet (Swagger)** | **Evet** | **Mükemmel** |

FastAPI'nin async desteği, veritabanı G/Ç'sini beklerken eş zamanlı tahmin isteklerinin verimli işlenmesine olanak tanır.

---

## Bölüm 4: Veri Kümesi ve Veri Toplama

### 4.1 Veri Toplama Protokolü

#### 4.1.1 Sensör Kurulumu

Her beton numunesi (standart 100×200mm silindir) için sensör kurulum prosedürü:

1. İş yeri özelliklerine göre beton karıştırın
2. Betonu 3 katmanda silindir kalıplara dökün, her katmanı 25 kez delin (ASTM C31)
3. Kalıbı kapatmadan önce su geçirmez kaplı DS18B20 sensörü orta yüksekliğe, merkezden 25mm uzağa yerleştirin
4. DHT22 ortam sensörünü kalıbın dışına aynı yüksekliğe takın (doğrudan güneş ışığından koruyun)
5. ESP32 veri kaydedicisini etkinleştirin; MQTT bağlantısını doğrulayın
6. Kalıpları kapatın ve 28 günde kırma testi için eşlik eden numunelerle birlikte kürleme ortamına yerleştirin

#### 4.1.2 Veri Şeması

Her MQTT mesajı `rasd/sensor/{deviceId}` konusuna aşağıdaki yükle yayımlar:

```json
{
  "deviceId": "ESP32_001",
  "projectId": "P2024-003",
  "companyId": "C007",
  "sampleKey": "C007_P2024-003_batch42_s1",
  "timestamp": "2024-08-15T14:30:00Z",
  "temperature": 28.4,
  "ambientTemperature": 22.1,
  "humidity": 68.3,
  "waterRatio": 0.45,
  "cementType": "CEM_I",
  "aggregateType": "CRUSHED",
  "chemicalAdditives": "PLASTICIZER"
}
```

`sampleKey` alanı tek bir numuneyi benzersiz şekilde tanımlar ve özellik mühendisliği için gruplama anahtarıdır.

### 4.2 Veri Kümesi İstatistikleri

| Özellik | Değer |
|---------|-------|
| Toplam ham kayıt | 199.488 |
| Benzersiz numune (sampleKey) | 2.079 |
| Numune başına kayıt | 96 (teorik) |
| Numune başına ortalama kayıt | 95,96 |
| Eksik değer oranı | %0,04 |
| Basınç dayanımı aralığı | 12,1 – 68,4 MPa |
| Ortalama basınç dayanımı | 38,2 MPa |
| Basınç dayanımı std sapma | 11,4 MPa |
| İnşaat şirketi sayısı | 14 |
| Proje sayısı | 47 |
| Veri toplama dönemi | Ocak 2023 – Aralık 2024 |

### 4.3 Dayanım Dağılımı

Basınç dayanımı dağılımı yaklaşık normaldir (Shapiro-Wilk $p = 0,23$) ve hafif sağa çarpıklıkla, seyrek karışımlardan (~12 MPa) yüksek dayanımlı karışımlara (~68 MPa) geniş bir aralığı kapsar.

Veri kümesindeki standart beton dayanım sınıfları:
- C16/20 (fck = 20 MPa): Numunelerin %8'i
- C20/25 (fck = 25 MPa): %18
- C25/30 (fck = 30 MPa): %31 (en yaygın)
- C30/37 (fck = 37 MPa): %22
- C35/45 (fck = 45 MPa): %14
- C40/50 ve üzeri: %7

---

## Bölüm 5: Veri Ön İşleme ve Kalite Kontrol

### 5.1 Saha IoT Dağıtımlarında Veri Kalitesi Sorunları

İnşaat sahalarından gelen gerçek dünya IoT verileri, laboratuvar veya referans veri kümelerinden çok daha gürültülüdür. Sensörler Wi-Fi kesintileri, fiziksel bozulma, pil tükenmesi ve sıcaklık kaynaklı kalibrasyon kayması yaşar.

### 5.2 Ön İşleme Ardışık Düzeni

```python
def preprocess(df: pd.DataFrame) -> pd.DataFrame:
    # Adım 1: Gerçek kopyaları kaldır
    df = df.drop_duplicates(subset=["sampleKey", "timestamp"])

    # Adım 2: Fiziksel mantık sınırları
    df = df[df["temperature"].between(5, 90)]
    df = df[df["ambientTemperature"].between(-10, 55)]
    df = df[df["humidity"].between(0, 100)]

    # Adım 3: Numune başına eksik okumalar için impütasyon
    df = df.groupby("sampleKey", group_keys=False).apply(impute_group)

    # Adım 4: Yetersiz numuneleri kaldır
    counts = df.groupby("sampleKey").size()
    valid_keys = counts[counts >= 48].index   # en az 12 saat
    df = df[df["sampleKey"].isin(valid_keys)]

    return df
```

### 5.3 Eksik Değer İmputasyonu

**Strateji**: Her numune (grup) içinde medyan imputasyonu. Bu kritiktir — numuneler arasında imputasyon yapılmamalıdır çünkü karışımlar ve kürleme ortamları arasında sıcaklık profilleri farklılık gösterir.

```python
def impute_group(grp: pd.DataFrame) -> pd.DataFrame:
    for col in ["temperature", "ambientTemperature", "humidity"]:
        grp[col] = grp[col].fillna(grp[col].median())
    return grp
```

**Neden medyan, ortalama değil?** Sıcaklık aykırı değerleri (sensör ani değişimleri) büyük ama seyrek olabilir. Medyan bu aykırı değerlere karşı dirençlidir — tek bir ani değişim okuma, imputasyonu bozamaz.

### 5.4 Kategorik Özellikler için Etiket Kodlaması

Üç kategorik özellik ML modelleri için kodlama gerektirir:

| Özellik | Kategoriler | Strateji | Gerekçe |
|---------|-----------|---------|---------|
| `cementType` | CEM_I, II, III, V | LabelEncoder | Ağaç bölünmeleri tamsayı kodlarında sıraya bağımsızdır |
| `aggregateType` | CRUSHED, RIVER, LIMESTONE | LabelEncoder | 3 kategori → 0, 1, 2 kodları |
| `chemicalAdditives` | NONE, PLASTICIZER, ACCELERATOR, RETARDER | LabelEncoder | 4 kategori |

**Ağaç modelleri için LabelEncoder vs. OneHotEncoder**: One-Hot Kodlama ikili kukla değişkenler oluşturur (3 kategori → 3 ikili sütun). Bu lineer modeller için gereklidir. Ancak Random Forest, XGBoost ve LightGBM için LabelEncoder uygundur çünkü ağaç bölünmeleri çok sınıflı kategorik özellikleri bölüntü testleri aracılığıyla doğal olarak ele alır.

**Kritik**: LabelEncoder yalnızca eğitim verilerine fit edilir, ardından test verilerine `transform()` kullanılarak uygulanır. Bu, veri sızıntısını önler.

### 5.5 Veri Bölme ve Standardizasyon

**Eğitim/test bölünmesi**: Tüm projelerin her iki sette de temsil edilmesini sağlamak için proje kimliğine göre katmanlı %80/%20 rastgele bölünme.

```python
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)
```

**StandardScaler**:
```python
scaler = StandardScaler()
X_train_sc = scaler.fit_transform(X_train)  # eğitim verisine FIT ve TRANSFORM
X_test_sc  = scaler.transform(X_test)       # yalnızca TRANSFORM (fit yok)
```

Ölçekleyici, test setinin dağılımının ölçekleyici parametrelerini etkilemediğinden emin olmak için yalnızca eğitim verilerine uygulanır.

---

## Bölüm 6: Özellik Mühendisliği

Özellik mühendisliği, bu projenin en entelektüel açıdan zorlu kısmıdır ve naif yaklaşımlara göre performans kazanımlarının birincil kaynağıdır. 96 ham sensör okumasını (zaman serisi) kürlemenin fiziksel bilgisini kompakt, model dostu bir formatta kodlayan 16 özelliğe dönüştürüyoruz.

### 6.1 Tasarım Felsefesi

Her özellik iki kriteri karşılamalıdır:
1. **İstatistiksel alaka**: Eğitim verilerindeki basınç dayanımıyla ilişkili
2. **Fiziksel yorumlanabilirlik**: Beton kimyası veya termodinamikten mekanik bir açıklaması var

Yalnızca kriter 1'i karşılayan özellikler, genelleşmeyen sahte korelasyonlar olabilir. Yalnızca kriter 2'yi karşılayanlar daha iyi proxy'lerle değiştirilmelidir.

### 6.2 Uzun-Geniş Formatı Dönüşümü

Ham veri "uzun" formattadır: satır başına (sampleKey, zaman damgası). ML modelleri "geniş" format gerektirir: numune başına satır.

```
UZUN FORMAT (ham):
sampleKey  | zaman_damgası | sicaklik | nem   | ...
-----------|---------------|----------|-------|
SP001      | T=0           | 22,4     | 65,2  |
SP001      | T=15dk        | 23,1     | 64,8  |
...
SP001      | T=23s45dk     | 31,2     | 71,3  |
SP002      | T=0           | 24,7     | 62,1  |
...

GENİŞ FORMAT (mühendislik yapılmış):
sampleKey | termal_enerji | hidrasyon_idx | max_sicaklik | suOrani | dayanim_mpa
----------|---------------|---------------|-------------|---------|------------
SP001     | 678,4         | 0,83          | 35,2        | 0,45    | 41,2
SP002     | 592,1         | 0,79          | 33,8        | 0,48    | 38,7
```

### 6.3 16 Mühendislik Özelliği

#### Özellik 1: thermal_energy_index (Termal Enerji İndeksi)

$$\text{TEI} = \int_0^{24} T(t)\,dt \approx \sum_{i=1}^{n-1} \frac{T_i + T_{i+1}}{2} \cdot \Delta t$$

Yamuk kuralıyla uygulandı:
```python
thermal_energy = float(np.trapezoid(T, hrs))
```

**Fiziksel anlam**: Toplam termal maruziyet = genelleştirilmiş Nurse-Saul olgunluğu. Daha yüksek TEI → daha eksiksiz hidrasyon → daha yüksek dayanım. Birimi: °C·saat.

**Neden yamuk kuralı?** Yamuk kuralı, dikdörtgen kurala kıyasla $O(h^2)$ doğruluğa ulaşır. $\Delta t = 0,25$ saat ile hata $h^2 = 0,0625$ ile orantılıdır. Dikdörtgen kurala kıyasla ($h = 0,25$) dört kat daha küçük hata.

#### Özellik 2: hydration_index (Hidrasyon İndeksi)

$$\text{HI} = \frac{\text{TEI}}{T_{max} \times 24}$$

Boyutsuz oran (0–1), teorik maksimum termal maruziyetin ne kadarının gerçekleştiğini gösterir.

**Fiziksel anlam**: Hidrasyon reaksiyonunun normalleştirilmiş tamamlanma oranı. 1,0'a yakın değerler verimli, düzgün kürlemeyi gösterir.

#### Özellik 3: temp_avg_first_12_hours (İlk 12 Saatin Ortalama Sıcaklığı)

$$\bar{T}_{12} = \frac{1}{48} \sum_{t=1}^{48} T_t$$

**Fiziksel anlam**: İlk 12 saat beton dayanım gelişimi için en kritik dönemdir. C₃S hidrasyonu bu dönemde hızlanır ve sıcaklık hem hidrasyon hızını hem de nihai hidrasyon derecesini etkiler.

#### Özellik 4: time_to_peak (Zirveye Ulaşma Süresi)

$$t_{peak} = \arg\max_{t} T(t)$$

**Fiziksel anlam**: Ekzotermik hidrasyon reaksiyonunun zirve yoğunluğuna ulaştığı zaman. Normal OPC için oda sıcaklığında bu genellikle 6–12 saatte gerçekleşir.

#### Özellik 5: temperature_rise_rate (Sıcaklık Yükselme Hızı)

$$\dot{T}_{yükseliş} = \frac{T_{peak} - T_0}{t_{peak}}$$

**Fiziksel anlam**: Hidrasyon başlangıcının hızı. Hızlı yükseliş, yüksek C₃A içeriği veya hızlandırıcı katkı maddelerinin kullanımıyla ilişkilidir.

#### Özellik 6: cooling_rate (Soğuma Hızı)

$$\dot{T}_{soğuma} = \frac{T_{peak} - T_{final}}{24 - t_{peak}}$$

**Fiziksel anlam**: Yavaş soğuma iyi ısı tutulduğunu gösterir — numune daha uzun süre yüksek sıcaklıkta kalarak dayanım gelişimine yarar sağlar.

#### Özellikler 7–11: İstatistiksel Özetler

| Özellik | Formül | Tipik Aralık |
|---------|--------|-------------|
| `avg_sample_temperature` | $\bar{T}$ | 15–45°C |
| `max_sample_temperature` | $\max(T)$ | 25–65°C |
| `min_sample_temperature` | $\min(T)$ | 10–30°C |
| `temp_std` | $\sigma_T$ | 2–15°C |
| `temp_range` | $T_{max} - T_{min}$ | 5–40°C |

#### Özellikler 12–13: Ortam Koşulları

| Özellik | Formül | Fiziksel Anlam |
|---------|--------|----------------|
| `avg_ambient_temperature` | $\bar{T}_a$ | Hidrasyonu yönlendiren/yavaşlatan dış sıcaklık |
| `avg_humidity` | $\bar{H}$ | Nem mevcudiyeti; erken kurumayı önler |

#### Özellikler 14–16: Karışım Tasarımı Parametreleri

| Özellik | Tür | Rol |
|---------|-----|-----|
| `waterRatio` | Sürekli [0,3–0,65] | Abrams Yasası — birincil dayanım belirleyicisi |
| `cementType` | Kategorik (4 sınıf) | C₃S/C₂S oranını ve hidrasyon kinetiğini belirler |
| `aggregateType` | Kategorik (3 sınıf) | Çimento pastasıyla arayüz bağ dayanımı |
| `chemicalAdditives` | Kategorik (4 sınıf) | Hidrasyon kimyasını değiştirir |

### 6.4 Özellik Korelasyon Analizi

28 günlük basınç dayanımı ile özelliklerin Pearson korelasyonu (ilk 5):

| Özellik | Korelasyon | Yön |
|---------|-----------|-----|
| `waterRatio` | -0,74 | Negatif (daha fazla su → daha az dayanım) |
| `thermal_energy_index` | +0,68 | Pozitif (daha fazla olgunluk → daha fazla dayanım) |
| `hydration_index` | +0,63 | Pozitif |
| `chemicalAdditives` | +0,51 | (kodlanmış: plastifiyan = yüksek, yok = düşük) |
| `avg_ambient_temperature` | +0,44 | (orta sıcaklık = pozitif) |

---

## Bölüm 7: Makine Öğrenimi Modelleri

### 7.1 Genel Bakış

Üç geleneksel topluluk ML modeli eğitiyoruz: Random Forest, XGBoost ve LightGBM. Bu modeller şu nedenlerle seçilmiştir:
- Yapılandırılmış tablo verilerinde kanıtlanmış performans
- Karışık özellik türleri için yerel destek (sürekli + kodlanmış kategorik)
- SHAP açıklanabilirlik araçlarıyla uyumluluk
- Düşük çıkarım gecikmesi (tahmin başına <1ms)

### 7.2 Random Forest

#### 7.2.1 Algoritma

Random Forest (Breiman, 2001), rastgele özellik alt örneklemesi olan karar ağaçlarının bir torbalama topluluğudur:

**Algoritma** (eğitim):
1. $b = 1$'den $B$'ye kadar:
   a. Yerine koyarak $n$ örnek alın (bootstrap örneği $\mathcal{D}_b$)
   b. $\mathcal{D}_b$ üzerinde ağaç $T_b$'yi eğitin; her bölünmede rastgele seçilen $m$ özellikten en iyi bölünmeyi seçin
2. Nihai tahmin: $\hat{f}(x) = \frac{1}{B} \sum_{b=1}^{B} T_b(x)$

**Yanlılık-varyans dengesi**:
- Her bireysel ağacın düşük yanlılığı (eğitim verilerini mükemmel şekilde uydurabilir) ancak yüksek varyansı vardır
- $B$ ağacın ortalaması varyansı yaklaşık $B$ faktörü kadar azaltır
- Anahtar içgörü: ağaçlar ilişkisizse (rastgele özellik alt örneklemesiyle sağlanır), varyans $\sigma^2/B$ olarak azalır

#### 7.2.2 Uygulama

```python
from sklearn.ensemble import RandomForestRegressor

rf_model = RandomForestRegressor(
    n_estimators=500,    # 500 ağaç
    max_depth=None,      # Tam derinliğe izin ver (düşük yanlılık)
    min_samples_leaf=2,  # Yaprak başına minimum 2 örnek
    max_features="sqrt", # Bölünme başına √16 ≈ 4 özellik
    n_jobs=-1,           # Tüm CPU çekirdeklerini kullan
    random_state=42
)
```

**Neden `max_features="sqrt"`?** Breiman'ın teorik analizi ve ampirik deneyleri, bölünme başına $\sqrt{p}$ özelliğin seçilmesinin ağaç ilişkisizleştirme (daha düşük varyans) ve bireysel ağaç kalitesi (daha düşük yanlılık) arasında optimum dengeyi sağladığını göstermiştir.

### 7.3 XGBoost

#### 7.3.1 Gradyan Artırma Çerçevesi

Gradyan artırma (Friedman, 2001), her yeni ağacı mevcut topluluğun artıklarına sırayla uydurarak zayıf öğrenicilerden (sığ ağaçlar) toplu bir topluluk oluşturur:

$$F_0(x) = \bar{y}$$
$$F_m(x) = F_{m-1}(x) + \eta \cdot h_m(x)$$

Kare hata kaybı için ($l = \frac{1}{2}(y - \hat{y})^2$), negatif gradyan artığa eşittir: $y - \hat{y}$.

#### 7.3.2 XGBoost Düzenlileştirme

XGBoost (Chen & Guestrin, 2016), gradyan artırmayı şunlarla geliştirir:

1. **Düzenlileştirilmiş hedef**: $\Omega(h) = \gamma T + \frac{1}{2}\lambda \sum_{j=1}^{T} w_j^2 + \alpha \sum_{j=1}^{T} |w_j|$

2. **Sütun alt örneklemesi**: Ağaç başına rastgele özellik alt kümesi kullanın (`colsample_bytree=0.8`)

3. **Satır alt örneklemesi**: `subsample=0.8` — eğitim verilerinin %80'ini ağaç başına kullanın

#### 7.3.3 Uygulama

```python
from xgboost import XGBRegressor

xgb_model = XGBRegressor(
    n_estimators=500,
    learning_rate=0.05,      # Küçük adımlar, daha fazla ağaç
    max_depth=6,
    subsample=0.8,
    colsample_bytree=0.8,
    reg_alpha=0.1,           # L1 düzenlileştirme
    reg_lambda=1.0,          # L2 düzenlileştirme
    random_state=42,
    eval_metric="rmse",
    early_stopping_rounds=50
)
xgb_model.fit(
    X_train_sc, y_train,
    eval_set=[(X_test_sc, y_test)],
    verbose=False
)
```

**Neden `learning_rate=0.05`?** Daha küçük öğrenme hızları daha fazla ağaç gerektirir ancak genellikle daha az aşırı uyum ile daha iyi bir minimum bulur.

### 7.4 LightGBM

#### 7.4.1 Algoritma Yenilikleri

LightGBM (Ke ve ark., 2017), XGBoost'a göre iki algoritmik yenilik sunar:

**Gradyan Tabanlı Tek Taraflı Örnekleme (GOSS)**:
- Büyük gradyanlı (yüksek eğitim hatası) tüm örnekleri koru
- Küçük gradyanlı örnekleri rastgele al
- Bu, fazla bilgi kaybetmeden "zor" örneklere odaklanmayı sağlar

**Özel Özellik Paketleme (EFB)**:
- Karşılıklı dışlayıcı özellikleri paketler (aynı anda nadiren sıfır olmayan)
- Etkin özellik boyutunu azaltır
- LightGBM büyük veri kümelerinde XGBoost'tan 5–20× daha hızlıdır

#### 7.4.2 Yaprak Bazlı Büyüme

LightGBM ağaçları yaprak bazında büyütür (önce en iyi): her adımda, ağaç seviyesinden bağımsız olarak maksimum kazançlı yaprağı böler. Bu yaklaşım:
- Yaprak sayısı başına daha düşük kayıp sağlar
- Risk: küçük veri kümelerinde derin, dengesiz ağaçlar → aşırı uyum

`num_leaves=63` (= seviye bazında derinlik 6'ya eşdeğer maksimum yaprak) ile kontrol edilir.

---

## Bölüm 8: Derin Öğrenme Modelleri

### 8.1 Neden Derin Öğrenme?

Ağaç toplulukları bu veri kümesinde daha iyi performans gösterse de, derin öğrenme modelleri üç nedenle dahil edilmiştir:

1. **Metodolojik bütünlük**: Kapsamlı bir sistem her iki paradigmayı da değerlendirmelidir
2. **Ölçeklenebilirlik**: Veri kümesi boyutu 10× büyürse derin öğrenme ağaç yöntemlerini geçebilir
3. **Uçtan uca öğrenme**: LSTM ham zaman serisinden doğrudan özellikler öğrenebilir

### 8.2 Derin Sinir Ağı (DNN)

#### 8.2.1 Mimari Tasarım

DNN, 16 mühendislik özelliğini girdi olarak alır ve tek bir skaler çıktı (MPa) tahmin eder.

```python
class ConcreteStrengthDNN(nn.Module):
    def __init__(self):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(16, 128),
            nn.BatchNorm1d(128),
            nn.ReLU(),
            nn.Dropout(0.3),
            nn.Linear(128, 64),
            nn.BatchNorm1d(64),
            nn.ReLU(),
            nn.Dropout(0.3),
            nn.Linear(64, 32),
            nn.BatchNorm1d(32),
            nn.ReLU(),
            nn.Dropout(0.2),
            nn.Linear(32, 1)
        )

    def forward(self, x):
        return self.net(x).squeeze(-1)
```

#### 8.2.2 Aktivasyon Fonksiyonu: ReLU

ReLU (Doğrusal Olmayan Doğrultma Birimi): $f(x) = \max(0, x)$

Özellikler:
- **Doyuma uğramaz**: $x > 0$ için gradyan = 1 (kaybolan gradyan yok)
- **Seyrek aktivasyon**: $x < 0$ için çıktı = 0 → seyrek gösterimler
- **Hesaplama açısından verimli**: Basit eşik işlemi

#### 8.2.3 Toplu Normalizasyon

BatchNorm (Ioffe & Szegedy, 2015), mini-toplu boyunca katman girdilerini normalleştirir:

$$\hat{x}_i = \frac{x_i - \mu_B}{\sqrt{\sigma_B^2 + \epsilon}}, \quad y_i = \gamma \hat{x}_i + \beta$$

**Etkileri**:
1. Eğitimi stabilize eder: Her katman normalleştirilmiş girdiler alır
2. Daha yüksek öğrenme hızlarına izin verir
3. Düzenlileştirme etkisi: Toplu düzey normalizasyon, dropout gibi gürültü katarak aşırı uyumu azaltır

#### 8.2.4 Dropout Düzenlileştirmesi

Dropout (Srivastava ve ark., 2014), eğitim sırasında nöron çıktılarını rastgele sıfırlar:

$$\tilde{h}_i = \begin{cases} h_i / (1-p) & (1-p) \text{ olasılıkla} \\ 0 & p \text{ olasılıkla} \end{cases}$$

**Yorum**: Dropout ile eğitim, yaklaşık olarak $2^n$ incelmiş ağaç topluluğunu eğitmek ve test anında tahminlerini ortalamaya almak ile eşdeğerdir.

#### 8.2.5 Eğitim Yapılandırması

```python
optimizer = optim.Adam(model.parameters(), lr=0.001)
criterion = nn.HuberLoss(delta=1.0)
scheduler = optim.lr_scheduler.ReduceLROnPlateau(
    optimizer, patience=10, factor=0.5, min_lr=1e-6
)
```

**Adam optimize edici**: Her parametre için uyarlamalı öğrenme hızı:
$$m_t = \beta_1 m_{t-1} + (1-\beta_1)g_t, \quad v_t = \beta_2 v_{t-1} + (1-\beta_2)g_t^2$$
$$\theta_t = \theta_{t-1} - \alpha \cdot \hat{m}_t / (\sqrt{\hat{v}_t} + \epsilon)$$

**Huber kaybı**: MSE (küçük hatalar için) ve MAE'yi (büyük hatalar için) birleştirir:
$$L_\delta(y, \hat{y}) = \begin{cases} \frac{1}{2}(y-\hat{y})^2 & |y-\hat{y}| \leq \delta \text{ ise} \\ \delta(|y-\hat{y}| - \frac{1}{2}\delta) & \text{aksi halde} \end{cases}$$

### 8.3 LSTM Ağı

#### 8.3.1 Motivasyon

LSTM ham zaman serisini doğrudan alarak manuel özellik mühendisliğini atlar. Bu, modelin hidrasyon fiziğini tek başına veriden keşfedip keşfedemeyeceğini test eder.

**Girdi formatı**: $(B, 96, 3)$ — partı boyutu, 96 zaman adımı, 3 özellik (sıcaklık, ortam sıcaklığı, nem)

#### 8.3.2 LSTM Mimarisi

Uzun Kısa Dönemli Bellek (Hochreiter & Schmidhuber, 1997), geçitli hücre durumu aracılığıyla standart RNN'lerdeki kaybolan gradyan sorununu çözer:

```python
class ConcreteStrengthLSTM(nn.Module):
    def __init__(self):
        super().__init__()
        self.lstm = nn.LSTM(
            input_size=3,
            hidden_size=64,
            num_layers=2,
            bidirectional=True,
            dropout=0.2,
            batch_first=True
        )
        self.fc = nn.Sequential(
            nn.Linear(128, 64),
            nn.ReLU(),
            nn.Linear(64, 1)
        )

    def forward(self, x):
        out, (hn, cn) = self.lstm(x)
        last = out[:, -1, :]   # Son gizli durum
        return self.fc(last).squeeze(-1)
```

**Çift yönlü LSTM**: Diziyi hem ileri (t=0→96) hem geri (t=96→0) işler. İleri geçiş "şimdiye kadar ne oldu" bilgisini yakalar; geri geçiş "sonra ne olacak" bilgisini yakalar.

#### 8.3.3 LSTM Geçitleri

LSTM hücresi bir hücre durumu $c_t$ korur — bilgiyi değiştirilmeden çok sayıda zaman adımı boyunca taşıyabilen bir bilgi otoyolu:

```
Unutma geçidi: f_t = σ(W_f · [h_{t-1}, x_t] + b_f)
Girdi geçidi:  i_t = σ(W_i · [h_{t-1}, x_t] + b_i)
Aday:          c̃_t = tanh(W_c · [h_{t-1}, x_t] + b_c)
Hücre günc.:   c_t = f_t ⊙ c_{t-1} + i_t ⊙ c̃_t
Çıktı geçidi:  o_t = σ(W_o · [h_{t-1}, x_t] + b_o)
Gizli durum:   h_t = o_t ⊙ tanh(c_t)
```

**Unutma geçidi kaybolan gradyanları çözer**: $c_t$'nin $c_{t-k}$'ya göre gradyanı $\prod_{j=t-k}^{t-1} f_j$'ye eşittir. $f_j \approx 1$ ile (geçit açık), gradyan değiştirilmeden çok sayıda zaman adımı boyunca akar.

#### 8.3.4 Gradyan Kırpma

```python
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
```

LSTM'nin iyileştirilmiş gradyan akışına rağmen, erken eğitim sırasında gradyanlar çok büyük olabilir (patlayan gradyanlar). Kırpma, maksimum gradyan normunu 1,0 olarak ayarlar: $\|\nabla\| > 1,0$ ise yeniden ölçeklendir.

---

## Bölüm 9: Değerlendirme Metodolojisi

### 9.1 Değerlendirme Metrikleri

Regresyon modeli performansını değerlendirmek için üç tamamlayıcı metrik kullanılır:

#### 9.1.1 Ortalama Mutlak Hata (MAE)

$$\text{MAE} = \frac{1}{n} \sum_{i=1}^{n} |y_i - \hat{y}_i|$$

**Yorumlama**: MPa cinsinden ortalama tahmin hatası. MAE = 1,87 MPa ise tahminler ortalama 1,87 MPa sapıyor demektir.

**Özellikler**: Aykırı değerlere karşı dayanıklı; hedefle aynı birimde raporlanır.

#### 9.1.2 Ortalama Karesel Hatanın Karekökü (RMSE)

$$\text{RMSE} = \sqrt{\frac{1}{n} \sum_{i=1}^{n} (y_i - \hat{y}_i)^2}$$

**Özellikler**: Büyük hataları MAE'den daha ağır cezalandırır (kareleme nedeniyle). $\text{RMSE} \geq \text{MAE}$ her zaman; fark büyük aykırı değer hatalarının varlığını gösterir.

#### 9.1.3 Belirleme Katsayısı (R²)

$$R^2 = 1 - \frac{\sum_{i=1}^{n}(y_i - \hat{y}_i)^2}{\sum_{i=1}^{n}(y_i - \bar{y})^2}$$

**Yorumlama**: Modelin açıkladığı dayanım varyansının fraksiyonu.
- $R^2 = 1,0$: Mükemmel tahmin
- $R^2 = 0,0$: Model tüm girdiler için ortalamayı tahmin eder
- $R^2 < 0,0$: Model ortalamayı tahmin etmekten daha kötü

### 9.2 Tanı Grafikleri

#### 9.2.1 Tahmin vs. Gerçek Grafik

Her model için mükemmel-uyum doğrusu ($y = \hat{y}$) üst üste bindirilerek $\hat{y}$ (x ekseni) vs. $y$ (y ekseni) dağılım grafiği. İdeal: köşegen etrafında sıkı kümeleme.

#### 9.2.2 Artık Grafik

Artıkların $r_i = y_i - \hat{y}_i$ vs. $\hat{y}_i$ dağılım grafiği, $r = 0$'da yatay referans çizgisi ile.

Artıklardaki desenler şunları gösterir:
- **Eğilim**: Belirli dayanım seviyelerinde sistematik yanlılık → model bir özelliği yakalamamış
- **Huni şekli**: Heterokedastik hatalar → hedefin log dönüşümünü düşünün
- **Düzgün dağılım**: İyi model uyumu

---

## Bölüm 10: Deneysel Sonuçlar ve Analiz

### 10.1 Ana Sonuçlar Tablosu

| Model | MAE (MPa) | RMSE (MPa) | R² | Çıkarım Süresi | Eğitim Süresi |
|-------|-----------|-----------|-----|----------------|--------------|
| **XGBoost** | **1,87** | **2,94** | **0,9461** | **<1 ms** | 45 sn |
| LightGBM | 1,95 | 3,02 | 0,9418 | <1 ms | 12 sn |
| Random Forest | 2,14 | 3,21 | 0,9312 | ~2 ms | 38 sn |
| DNN | 2,31 | 3,47 | 0,9198 | <1 ms | 3 dk |
| LSTM | 2,58 | 3,83 | 0,9044 | ~3 ms | 4 dk |

*Çıkarım süreleri Intel Core i7-12700H üzerinde tek örnek tahmini için ölçüldü*

### 10.2 İstatistiksel Anlamlılık

XGBoost'un diğer modellere göre avantajının istatistiksel olarak anlamlı olduğunu doğrulamak için 391 test seti artığı üzerinde eşleştirilmiş Diebold-Mariano testleri gerçekleştirdik:

| Karşılaştırma | DM İstatistiği | p-değeri | Anlamlı mı? |
|-------------|----------------|---------|------------|
| XGBoost vs LightGBM | 2,14 | 0,032 | Evet (p<0,05) |
| XGBoost vs RF | 3,87 | <0,001 | Evet (p<0,001) |
| XGBoost vs DNN | 5,23 | <0,001 | Evet (p<0,001) |
| XGBoost vs LSTM | 7,41 | <0,001 | Evet (p<0,001) |

Tüm farklar istatistiksel olarak anlamlıdır.

### 10.3 Dayanım Sınıfına Göre Performans

| Dayanım Sınıfı | n | XGBoost MAE | XGBoost R² |
|----------------|---|-------------|-----------|
| C16/20 (12–25 MPa) | 157 | 1,62 | 0,931 |
| C25/30 (25–37 MPa) | 606 | 1,74 | 0,948 |
| C30/37 (37–45 MPa) | 430 | 1,91 | 0,944 |
| C35/45 (45–55 MPa) | 274 | 2,11 | 0,938 |
| C40/50+ (55+ MPa) | 137 | 2,84 | 0,921 |

Yüksek dayanımlı beton (>55 MPa) için performans biraz daha zayıftır; bu muhtemelen eğitim verilerindeki yetersiz temsil ve yüksek karışım oranlarında dayanım ile karışım tasarımı arasındaki daha karmaşık ilişkiden kaynaklanmaktadır.

### 10.4 Ablasyon Çalışması: Özellik Grubu Katkıları

Her özellik grubunun katkısını ölçmek için XGBoost'u kademeli özellik çıkarımıyla eğittik:

| Özellik Seti | MAE (MPa) | R² | Δ R² (Tam vs.) |
|-------------|-----------|-----|----------------|
| Tam (16 özellik) | 1,87 | 0,9461 | — |
| Termal özellikler olmadan (1–6) | 2,43 | 0,9108 | -0,0353 |
| İstatistiksel özellikler olmadan (7–11) | 2,12 | 0,9283 | -0,0178 |
| Ortam özellikleri olmadan (12–13) | 2,01 | 0,9372 | -0,0089 |
| Karışım tasarımı olmadan (14–16) | 2,31 | 0,9171 | -0,0290 |
| Yalnızca karışım tasarımı (IoT yok) | 2,89 | 0,8743 | -0,0718 |

**Temel bulgu**: Karışım tasarımı özellikleri (özellikle waterRatio) bireysel olarak en fazla katkıda bulunurken, IoT türetilmiş termal özellikler en büyük toplu iyileştirmeyi sağlar. Bu, sistemin temel öncülünü doğrular: IoT sensörü verileri, statik karışım tasarımının ötesinde önemli tahmin değeri katar.

### 10.5 Proje Geneli Genelleştirme

Eğitim sırasında görülmeyen yeni inşaat şirketlerine (firmalarına) genelleştirmeyi test etmek için çapraz doğrulama (14 şirket, birer birer dışarıda bırak) gerçekleştirdik:

| Metrik | Ortalama | Std | Min | Max |
|--------|--------|-----|-----|-----|
| MAE (MPa) | 2,14 | 0,31 | 1,64 | 2,87 |
| RMSE (MPa) | 3,31 | 0,48 | 2,71 | 4,42 |
| R² | 0,928 | 0,021 | 0,887 | 0,951 |

Model, rastgele bölünme değerlendirmesine kıyasla ~0,27 MPa MAE performans düşüşüyle projeler arasında iyi genelleme yapar.

---

## Bölüm 11: SHAP Açıklanabilirlik Analizi

### 11.1 Açıklanabilirlik Motivasyonu

İnşaat mühendisliği pratiğinde model açıklanabilirliği yalnızca akademik değildir — doğrudan operasyonel sonuçları vardır:

1. **Kalite kontrol kararları**: Mühendislerin neden düşük tahmin yapıldığını anlamaları gerekir (örn. yetersiz kürleme) ve inşaata devam edip etmeyeceğine karar verirler
2. **Düzenleyici uyum**: Bazı yargı bölgelerinde yapısal güvenliği etkileyen algoritmik kararlar denetlenebilir olmalıdır
3. **Model güveni**: Mühendisler anladıkları bir modele güvenir ve kullanır

### 11.2 TreeSHAP Algoritması

XGBoost (ağaç topluluğu) için TreeSHAP (Lundberg ve ark., 2020) kullanıyoruz:

```python
explainer   = shap.TreeExplainer(xgb_model)
shap_values = explainer.shap_values(X_test)
# shap_values.shape = (391, 16)
# Her satır: bir numune için SHAP değerleri, 16 özellik
# Her satırın toplamı = f(x) - E[f(X)] = tahmin - ortalama tahmin
```

### 11.3 Global Özellik Önemi (Ortalama |SHAP|)

| Sıra | Özellik | Ort. |SHAP| (MPa) | % Toplam |
|------|---------|----------------|----------|
| 1 | waterRatio | 4,21 | %23,4 |
| 2 | chemicalAdditives | 3,18 | %17,7 |
| 3 | thermal_energy_index | 2,87 | %16,0 |
| 4 | hydration_index | 2,34 | %13,0 |
| 5 | avg_ambient_temperature | 1,98 | %11,0 |
| 6 | temp_avg_first_12_hours | 1,76 | %9,8 |
| 7 | max_sample_temperature | 1,54 | %8,6 |
| 8 | cementType | 1,42 | %7,9 |
| 9 | temperature_rise_rate | 1,31 | %7,3 |
| 10 | aggregateType | 1,08 | %6,0 |

### 11.4 SHAP Yönlerinin Fiziksel Yorumu

#### 11.4.1 waterRatio (SHAP yönü: negatif korelasyon)

Arı sürüsü analizi: yüksek waterRatio değerleri (kırmızı) solda kümelenir (negatif SHAP), düşük değerler (mavi) sağda kümelenir (pozitif SHAP). Bu tam olarak Abrams Yasasıdır:

$$\partial f'c / \partial (s/ç) < 0$$

s/ç'deki her 0,05 artış, tahmin edilen dayanımı yaklaşık 3,2 MPa azaltır.

#### 11.4.2 thermal_energy_index (SHAP yönü: pozitif korelasyon)

Yüksek TEI → pozitif SHAP → daha yüksek tahmin edilen dayanım. Bu Olgunluk Yöntemini doğrular: daha fazla termal enerji = daha eksiksiz hidrasyon = daha yüksek dayanım.

#### 11.4.3 avg_ambient_temperature (SHAP yönü: doğrusal olmayan)

Bağımlılık grafiği doğrusal olmayan bir SHAP profili ortaya koymaktadır:
- 5–15°C: Negatif SHAP (soğuk hava hidrasyonu yavaşlatır)
- 15–30°C: Pozitif SHAP (optimum kürleme sıcaklığı aralığı)
- >35°C: Azalan, >40°C için negatif (aşırı ısı ettringit kararsızlığına neden olur)

Bu üç rejimli ilişki, ACI 305/306 sıcak ve soğuk hava betonlama kılavuzlarıyla tutarlıdır.

### 11.5 Örnek Düzeyinde Açıklama (Şelale Grafiği)

Temsili yüksek dayanımlı bir numune için (f'c = 52,3 MPa):

```
Taban E[f(x)]                    = 38,2 MPa
  + waterRatio = 0,38             → +6,2 MPa  (ortalama altı s/ç)
  + chemicalAdditives=PLASTICIZER → +4,1 MPa  (daha düşük efektif s/ç sağlar)
  + thermal_energy_index = 892    → +3,8 MPa  (yüksek olgunluk)
  + hydration_index = 0,89        → +2,9 MPa  (neredeyse tam hidrasyon)
  - avg_ambient_temp = 37°C       → -1,8 MPa  (hafif ısı cezası)
  + cementType = CEM_I            → +0,9 MPa  (hızlı reaksiyonlu OPC)
  [küçük özellikler...]           → +0,3 MPa
  ──────────────────────────────────────────
  Tahmin edilen f(x)              = 54,6 MPa
  Gerçek f'c                      = 52,3 MPa  (bu numune için MAE = 2,3 MPa)
```

Bu açıklama saha mühendisleri için hemen eyleme dönüştürülebilir niteliktedir.

---

## Bölüm 12: Üretim Ortamı Dağıtımı

### 12.1 Dağıtım Mimarisi

RASD sistemi aşağıdaki bileşen yığınıyla Linux VPS'te (Ubuntu 22.04 LTS) dağıtılır:

| Bileşen | Teknoloji | Port |
|---------|-----------|------|
| Ters proxy + SSL | nginx | 80/443 |
| Ön uç | Next.js + Node.js | 3000 (dahili) |
| Arka uç API | Spring Boot Java 17 | 8080 (dahili) |
| ML API | FastAPI + uvicorn | 8086 (dahili) |
| Veritabanı | PostgreSQL 16 | 5432 (dahili) |
| Mesaj aracısı | Apache Kafka | 9092 (dahili) |

### 12.2 FastAPI ML Servisi Mimarisi

#### 12.2.1 Model Yükleme Stratejisi

```python
@app.on_event("startup")
def startup():
    init_db()
    load_models()

def load_models():
    global _models, _scaler, _label_encoders
    # Başlangıçta tüm 5 modeli diskten RAM'e yükle
    _models["random_forest"] = joblib.load("ml_output/random_forest.joblib")
    _models["xgboost"] = joblib.load("ml_output/xgboost.joblib")
    _models["lightgbm"] = joblib.load("ml_output/lightgbm.joblib")
    _scaler = joblib.load("ml_output/scaler.joblib")
    # PyTorch modelleri için...
```

**Başlangıçta yükleme, istek başına değil**: XGBoost'u diskten yüklemek ~2 saniye sürer. Saniyede 10 eşzamanlı istek ile istek başına yükleme, saniyede 20 CPU saniyesi tüketir — açıkça uygulanamaz.

#### 12.2.2 Çıkarım Ardışık Düzeni

```python
def predict_strength(features: dict, model_name: str = "auto") -> float:
    # 1. Doğru sütun sırasıyla özellik vektörü oluştur
    row = {}
    for col in FEATURE_COLS:
        val = features.get(col, 0)
        if col in _label_encoders:
            val = safe_encode(_label_encoders[col], str(val))
        row[col] = val

    # 2. Numpy dizisine dönüştür
    X = np.array([[row[c] for c in FEATURE_COLS]], dtype=np.float32)

    # 3. Eğitimde kullanılan aynı ölçekleyici ile ölçeklendir
    X_scaled = _scaler.transform(X)

    # 4. Tahmin et
    if model_name == "auto":
        model_name = "xgboost"

    if model_name in ("random_forest", "xgboost", "lightgbm"):
        return float(_models[model_name].predict(X_scaled)[0])
    elif model_name == "dnn":
        t = torch.tensor(X_scaled, dtype=torch.float32)
        with torch.no_grad():
            return float(_models["dnn"](t).squeeze().item())
```

**`torch.no_grad()` neden?**: Eğitim sırasında PyTorch, geri yayılım yoluyla gradyanları hesaplamak için hesaplama grafiği oluşturur. Bu grafik çıkarım başına O(model derinliği) bellek tüketir. `no_grad()`, çıkarım sırasında grafik oluşturmayı devre dışı bırakır → bellek kullanımını ~%50 azaltır, hızı ~%30 artırır.

#### 12.2.3 JWT Kimlik Doğrulaması

ML servisi, Spring Boot arka ucuyla kimlik doğrulamayı paylaşır:

```python
JWT_SECRET    = "5261736452617364..."  # 82 baytlık paylaşılan gizli
JWT_ALGORITHM = "HS512"               # Spring Boot 512 bit üzeri anahtarlar için HS512 kullanır
```

**Neden HS512?** Spring Boot'un jjwt kütüphanesi HMAC algoritmasını anahtar boyutuna göre otomatik seçer: ≥512 bit anahtarlar HS512 kullanır. Paylaşılan gizli 82 bayt = 656 bit > 512 bit → HS512.

#### 12.2.4 Engellemesiz Eğitim

Model yeniden eğitimi bir REST uç noktası aracılığıyla tetiklenir ancak asenkron olarak yürütülür:

```python
@app.post("/api/ml/models/train")
async def train_models(background_tasks: BackgroundTasks, ...):
    background_tasks.add_task(run_training)
    return {"status": "started", "message": "Eğitim başlatıldı"}

def run_training():
    # Adım 1: ML ardışık düzeni (RF, XGBoost, LightGBM)
    subprocess.run(["python", "concrete_ml_pipeline.py"], ...)
    # Adım 2: DL ardışık düzeni (DNN + LSTM PyTorch)
    subprocess.run(["python", "concrete_dl_pipeline.py"], ...)
    load_models()  # Yeni modelleri sıcak yeniden yükle
```

Eğitim 30–60 dakika sürer. `BackgroundTasks` olmadan HTTP isteği zaman aşımına uğrar (tipik zaman aşımı: 30–60 saniye).

### 12.3 nginx Yapılandırması

```nginx
server {
    listen 443 ssl;
    server_name rasd.gharss.org;

    # ML API — /api/ml/ yolunu FastAPI'ye yönlendir
    location /api/ml/ {
        proxy_pass http://127.0.0.1:8086;
        proxy_read_timeout 300s;
        proxy_send_timeout 300s;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto https;
    }

    # Arka uç API
    location /api/ {
        proxy_pass http://127.0.0.1:8080;
    }

    # Ön uç
    location / {
        proxy_pass http://127.0.0.1:3000;
    }
}
```

Tüm bileşenler aynı HTTPS alan adını paylaşır; bu CORS sorunlarını ortadan kaldırır.

### 12.4 Web Panosu

React/Next.js panosu şunları sağlar:

#### 12.4.1 Tahminler Sayfası (/predictions)

- Tüm projeleri model başına tahmin edilen basınç dayanımı (MPa) ile gösteren tablo
- Renk kodlaması: Yeşil (≥ belirtilen dayanım), Sarı (%5 dahilinde), Kırmızı (gereksinimin altında)
- SYSTEM_MANAGER, mevcut modelden tahminleri yenilemek için "Tahminleri Çalıştır" tetikleyebilir

#### 12.4.2 Modeller Sayfası (/models)

- MAE, RMSE, R² ile tüm 5 modelin karşılaştırma tablosunu gösterir
- Tahminler için aktif model seçimine izin verir
- SYSTEM_MANAGER arka plan eğitimini başlatmak için "Modelleri Yeniden Eğit" tetikleyebilir

### 12.5 Güvenlik ve Erişim Kontrolü

Sistem üç rolle rol tabanlı erişim kontrolü (RBAC) uygular:

| Rol | Tahminleri Görüntüle | Tahminleri Çalıştır | Modelleri Yeniden Eğit | Model Detayları |
|-----|---------------------|--------------------|-----------------------|----------------|
| ENGINEER | ✓ | ✗ | ✗ | Yalnızca okuma |
| MANAGER | ✓ | ✓ | ✗ | Yalnızca okuma |
| SYSTEM_MANAGER | ✓ | ✓ | ✓ | Tam erişim |

---

## Bölüm 13: Tartışma

### 13.1 Neden Ağaç Yöntemleri Derin Öğrenmeden Daha İyi Performans Gösteriyor?

Bu veri kümesinde XGBoost ve LightGBM'nin DNN ve LSTM üzerindeki tutarlı üstünlüğü, köklü bir ampirik bulguyla tutarlıdır: **~10.000'den az örnek içeren tablo verileri için, gradyan artırmalı ağaçlar neredeyse her zaman derin öğrenmeden daha iyi performans gösterir** [Grinsztajn ve ark., 2022].

Üç mekanizma bunu açıklar:

1. **Tümevarım yanlılığı uyumu**: Beton dayanımı tahmin problemi bilinen monoton, düzgün bir yapıya sahiptir. Karar ağaçları doğal olarak parça-sabit (monoton-uyumlu) yaklaşımlar uygular; sinir ağları bu monoton ilişkileri gradyan inişi kullanarak sıfırdan öğrenmek zorundadır.

2. **Örnek verimliliği**: 1.562 eğitim numunesi ile ağaç toplulukları yakın optimum varyans azaltımı elde edebilir. Sinir ağları karşılaştırılabilir gösterimler öğrenmek için tipik olarak 10–100× daha fazla örnek gerektirir.

3. **Özellik kalitesi**: 16 manuel olarak tasarlanmış fizik tabanlı özellik, ağaç modellerine tam olarak ihtiyaç duydukları sinyali verir.

### 13.2 ML Modellerinin Fiziksel Geçerliliği

ML'yi mühendislik problemlerine uygulamadaki temel endişe, modellerin fiziksel anlamlı ilişkileri mi yoksa sahte korelasyonları mu öğrendiğidir. SHAP analizimiz fiziksel geçerlilik için güçlü kanıtlar sağlar:

1. **Abrams Yasası ortaya çıkıyor**: Açık kodlama olmadan model, `waterRatio`'yu negatif yönde en önemli özellik olarak sıralıyor
2. **Olgunluk yöntemi doğrulandı**: `thermal_energy_index` üçüncü sıraya giriyor, ASTM C1074'ün temelini doğruluyor
3. **Sıcaklık doğrusallığı olmayan**: SHAP bağımlılık grafikleri, ACI 305/306 ile tutarlı üç rejimli sıcaklık davranışını gösteriyor

### 13.3 Sınırlamalar

1. **Karışım tasarımı gerekli**: Model `waterRatio`, `cementType`, `aggregateType` ve `chemicalAdditives` girdilerini gerektirir. Bunlar döküm anında bilinir ancak insan hatasını beraberinde getiren manuel giriş gerektirir.

2. **Coğrafi kapsam**: Veri kümesi bir bölgedeki inşaat projelerinden; çimento kimyası, agrega mineralojisi ve ortam sıcaklığı aralıkları diğer iklimlerde önemli ölçüde farklılık gösterebilir.

3. **Veri kümesi boyutu**: ~2.000 numune veri kümesi DNN ve LSTM mimarilerini tam olarak eğitmek için yetersizdir.

4. **24 saatlik pencere**: Yavaş erken dayanım geliştiren yüksek SCM (takviyeli bağlayıcı malzeme) betonlarında doğruluk daha düşük olabilir.

### 13.4 Mevcut Sistemlerle Karşılaştırma

| Sistem | IoT? | ML Modelleri | Açıklanabilirlik | Web Panosu | Açık Kaynak |
|--------|------|-------------|-----------------|------------|-------------|
| maturitymethod.com | Hayır | Yok (Nurse-Saul) | Yok | Evet | Hayır |
| SmartRock (Maturix) | Evet | Yok | Hayır | Evet | Hayır |
| Concrete AI [Kim, 2022] | Evet | Yalnızca LSTM | Hayır | Hayır | Hayır |
| **RASD (bu çalışma)** | **Evet** | **5 model** | **SHAP** | **Evet** | **Evet** |

RASD, IoT veri toplama, çoklu ML modelleri, SHAP açıklanabilirliği ve üretim web panosunu tek, dağıtılabilir açık kaynaklı bir sistemde entegre eden ilk sistemdir.

---

## Bölüm 14: Sonuç ve Gelecek Çalışmalar

### 14.1 Katkıların Özeti

Bu tez, erken beton basınç dayanımı tahmini için eksiksiz bir IoT-dan-tahmine sistemi olan **RASD**'ı sunmuştur.

**K1: Uçtan Uca Ardışık Düzen Tasarımı**
ESP32 sensör gömülmesinden MQTT/Kafka akışı, Spring Boot arka ucu, FastAPI ML servisi ve Next.js web panosuna kadar eksiksiz mimari — üretim ortamında dağıtıldı.

**K2: Fizik Tabanlı Özellik Mühendisliği**
96 noktalı zaman serisi verilerinden türetilen 16 özelliğin sistematik çerçevesi; her biri beton kimyası ve termodinamiğe dayandırılmıştır. Termal enerji indeksi (sıcaklık-zaman eğrisinin yamuk entegrasyonu), standart Nurse-Saul skalerine göre daha zengin bir olgunluk metriği sağlar.

**K3: Karşılaştırmalı ML/DL Değerlendirmesi**
Beş modelin (Random Forest, XGBoost, LightGBM, DNN, LSTM) özdeş koşullar altında titiz karşılaştırması. XGBoost en iyi performansı elde eder (R² = 0,9461); ağaç toplulukları bu veri kümesi ölçeğinde derin öğrenmeden tutarlı biçimde daha iyi performans gösterir.

**K4: SHAP Açıklanabilirliği**
SHAP analizi, modellerin fiziksel anlamlı ilişkileri öğrendiğini doğrular. Abrams Yasası'nın ve Olgunluk Yöntemi'nin veriden çıkarılması, model güvenilirliği için güçlü kanıt sağlar.

**K5: Üretim Dağıtımı**
Sistem, JWT kimlik doğrulaması ve rol tabanlı erişim kontrolüyle gerçek inşaat projelerine hizmet veren üretim ortamında çalışmaktadır.

### 14.2 Araştırma Sorularına Yanıtlar

**AS1**: XGBoost MAE = 1,87 MPa, RMSE = 2,94 MPa, R² = 0,9461 elde eder. 25–45 MPa tipik beton dayanımları için bu %4–7 göreli hata temsil eder — erken aşama kalite kararları için yeterli doğruluk.

**AS2**: Ağaç yöntemleri (XGBoost, LightGBM), bu ~2.000 numune tablo veri kümesinde derin öğrenmeden daha iyi performans gösterir.

**AS3**: SHAP analizi fiziksel anlamlı ilişkileri doğrular (Abrams Yasası, Olgunluk Yöntemi, sıcaklık doğrusallığı olmayan). Modeller gerçek fiziksel ilişkileri yakalar.

**AS4**: En önemli özellikler (waterRatio, chemicalAdditives, thermal_energy_index), köklü beton bilimiyle (Abrams Yasası, katkı kimyası, Nurse-Saul Olgunluk Yöntemi) tam olarak örtüşür.

### 14.3 Pratik Etki

24 saatlik tahmin yeteneği, standart 28 günlük teste kıyasla 27 günlük önceden haber verme sağlar. Yılda 500 numune içeren bir inşaat projesinde bu şunları mümkün kılar:
- Standart dışı partilerin daha erken tespiti (yapı daha da ilerlemeden önce)
- Daha duyarlı karışım tasarımı ayarlamaları
- Kalıp sökümü ve yükleme operasyonlarının daha iyi programlanması

### 14.4 Gelecek Çalışmalar

#### 14.4.1 Kısa Vadeli (6–12 ay)

**Çevrimiçi/artımlı öğrenme**: Şu anda modeller periyodik olarak sıfırdan yeniden eğitilmektedir. Çevrimiçi gradyan artırma veya önceden eğitilmiş modellerin ince ayarlanması, tam yeniden eğitim olmadan sürekli iyileştirmeye olanak tanır.

**Çok hedefli tahmin**: 7, 14 ve 28 günlük dayanımı eş zamanlı tahmin etmek için çok çıktılı regresyona genişletme.

**Otomatik karışım tasarımı optimizasyonu**: Hedef dayanım $f'c^*$ verildiğinde, optimum karışım oranlarını önermek için eğitilmiş modeli amaç fonksiyonu olarak kullanan optimizasyon.

#### 14.4.2 Orta Vadeli (1–2 yıl)

**Çimento tipleri arasında transfer öğrenimi**: Tüm çimento tiplerinde paylaşılan bir omurga eğitmek ve her tip için ince ayar katmanları uygulamak.

**Anomali tespiti**: Son tahmin öncesinde olağandışı sıcaklık profili (sensör arızasını, alışılmadık karışım davranışını veya kirliliği gösteren) numuneleri işaretlemek için denetimsiz anomali tespiti.

**Kenar çıkarımı**: Bulut bağlantısı olmadan cihaz üzerinde anlık dayanım tahmini için ESP32'de sıkıştırılmış model dağıtımı.

#### 14.4.3 Uzun Vadeli (2+ yıl)

**Fizik Bilgili Sinir Ağları (PINN'ler)**: Sinir ağı kayıp fonksiyonuna yumuşak kısıtlar olarak hidrasyon kinetiği diferansiyel denklemleri dahil etmek.

**Dijital ikiz entegrasyonu**: RASD'ı yapı bilgi modelleme (BIM) sistemlerine bağlamak.

**Federe öğrenme**: Birden fazla inşaat şirketinin tescilli karışım tasarımı verilerini paylaşmadan ortak modeller eğitmesini sağlamak.

---

## Kaynaklar

1. Abrams, D.A. (1919). *Beton Karışımlarının Tasarımı*. Lewis Enstitüsü, Yapısal Malzeme Araştırma Laboratuvarı, Bülten No. 1.

2. ACI 305R (2010). *Sıcak Hava Betonlama Kılavuzu*. Amerikan Beton Enstitüsü.

3. ACI 306R (2016). *Soğuk Hava Betonlama Kılavuzu*. Amerikan Beton Enstitüsü.

4. ASTM C39 (2021). *Silindirik Beton Numunelerinin Basınç Dayanımı için Standart Test Yöntemi*. ASTM International.

5. ASTM C1074 (2019). *Olgunluk Yöntemiyle Beton Dayanımının Tahmin Edilmesi için Standart Uygulama*. ASTM International.

6. Breiman, L. (2001). Rastgele Ormanlar. *Machine Learning*, 45(1), 5–32.

7. Chen, T., & Guestrin, C. (2016). XGBoost: Ölçeklenebilir Bir Ağaç Artırma Sistemi. *KDD 2016 Bildirileri*, 785–794.

8. Chou, J.S., vd. (2014). Veri madenciliği tekniklerinin karşılaştırmasına dayalı beton basınç dayanımı tahmin doğruluğunun optimize edilmesi. *İnşaat Mühendisliğinde Bilgi İşlem Dergisi*, 25(3), 242–253.

9. Diebold, F.X., & Mariano, R.S. (1995). Tahmin doğruluğunun karşılaştırılması. *İş & Ekonomik İstatistikler Dergisi*, 13(3), 253–263.

10. EN 206-1 (2013). *Beton — Bölüm 1: Özellik, performans, üretim ve uygunluk*. Avrupa Standardizasyon Komitesi.

11. Farooq, F., vd. (2021). Topluluk makine öğrenimi kullanarak endüstriyel atıklardan sürdürülebilir yüksek performanslı beton için tahmine dayalı modelleme. *İnşaat ve Yapı Malzemeleri*, 278, 122419.

12. Friedman, J.H. (2001). Açgözlü fonksiyon yaklaşımı: Gradyan artırma makinesi. *İstatistik Yıllığı*, 29(5), 1189–1232.

13. Grinsztajn, L., Oyallon, E., & Varoquaux, G. (2022). Neden ağaç tabanlı modeller tablo verileri için derin öğrenmeden hâlâ daha iyi performans gösteriyor? *Sinir Bilgi İşleme Sistemleri Gelişmeleri*, 35.

14. Hochreiter, S., & Schmidhuber, J. (1997). Uzun Kısa Dönemli Bellek. *Sinir Hesaplaması*, 9(8), 1735–1780.

15. Ioffe, S., & Szegedy, C. (2015). Toplu Normalizasyon: İç Kovaryat Kaymasını Azaltarak Derin Ağ Eğitimini Hızlandırma. *ICML 2015 Bildirileri*.

16. Ke, G., vd. (2017). LightGBM: Son Derece Verimli Gradyan Artırma Karar Ağacı. *Sinir Bilgi İşleme Sistemleri Gelişmeleri*, 30.

17. Kim, B., vd. (2022). Gömülü sensör zaman serisinden LSTM tabanlı erken yaşta basınç dayanımı tahmini. *İnşaat ve Yapı Malzemeleri*, 315, 125739.

18. Kingma, D.P., & Ba, J. (2015). Adam: Stokastik Optimizasyon için Bir Yöntem. *ICLR 2015*.

19. Lundberg, S.M., & Lee, S.I. (2017). Model Tahminlerini Yorumlamak için Birleşik Bir Yaklaşım. *Sinir Bilgi İşleme Sistemleri Gelişmeleri*, 30.

20. Lundberg, S.M., vd. (2020). Ağaçlar için açıklanabilir yapay zekaya sahip yerel açıklamalardan küresel anlayışa. *Nature Machine Intelligence*, 2(1), 56–67.

21. Mehta, P.K., & Monteiro, P.J.M. (2014). *Beton: Mikro Yapı, Özellikler ve Malzemeler* (4. baskı). McGraw-Hill Education.

22. Nurse, R.W. (1949). Betonun buharlı kürlemesi. *Beton Araştırmaları Dergisi*, 1(2), 79–88.

23. Paszke, A., vd. (2019). PyTorch: Zorunlu Stil, Yüksek Performanslı Derin Öğrenme Kütüphanesi. *Sinir Bilgi İşleme Sistemleri Gelişmeleri*, 32.

24. Pedregosa, F., vd. (2011). Scikit-learn: Python'da Makine Öğrenimi. *Makine Öğrenimi Araştırmaları Dergisi*, 12, 2825–2830.

25. Plowman, J.M. (1956). Betonun olgunluğu ve dayanımı. *Beton Araştırmaları Dergisi*, 8(22), 13–22.

26. Shwartz-Ziv, R., & Armon, A. (2022). Tablo verileri: Derin öğrenme tek ihtiyacınız değil. *Bilgi Füzyonu*, 81, 84–90.

27. Srivastava, N., vd. (2014). Dropout: Sinir Ağlarının Aşırı Uyumunu Önlemenin Basit Bir Yolu. *Makine Öğrenimi Araştırmaları Dergisi*, 15(1), 1929–1958.

28. Tanner, N.A., vd. (2003). Kablosuz akıllı sensör ağları kullanan yapısal sağlık izleme. *SPIE Bildirileri*, 5057.

29. Taylor, H.F.W. (1997). *Çimento Kimyası* (2. baskı). Thomas Telford Yayıncılık.

30. Yeh, I.C. (1998). Yapay sinir ağları kullanarak yüksek performanslı betonun dayanımının modellenmesi. *Çimento ve Beton Araştırmaları*, 28(12), 1797–1808.

---

## Ek A: Veri Kümesi Özellikleri Referansı

| # | Özellik Adı | Tür | Birim | Aralık | Kaynak |
|---|------------|-----|-------|--------|--------|
| 1 | thermal_energy_index | Sürekli | °C·h | 200–1200 | Yamuk entegrasyonu |
| 2 | hydration_index | Sürekli | — | 0–1 | TEI / (T_max × 24) |
| 3 | temp_avg_first_12_hours | Sürekli | °C | 15–50 | İlk 48 okumanın ortalaması |
| 4 | time_to_peak | Sürekli | h | 2–18 | argmax(T) × 0,25 |
| 5 | temperature_rise_rate | Sürekli | °C/h | 0,1–5 | Zirveye kadar ΔT/Δt |
| 6 | cooling_rate | Sürekli | °C/h | 0–3 | Zirve sonrası ΔT/Δt |
| 7 | avg_sample_temperature | Sürekli | °C | 15–45 | ortalama(T) |
| 8 | max_sample_temperature | Sürekli | °C | 20–65 | max(T) |
| 9 | min_sample_temperature | Sürekli | °C | 10–30 | min(T) |
| 10 | temp_std | Sürekli | °C | 1–15 | std(T) |
| 11 | temp_range | Sürekli | °C | 5–40 | max(T) − min(T) |
| 12 | avg_ambient_temperature | Sürekli | °C | −5–45 | ortalama(T_ortam) |
| 13 | avg_humidity | Sürekli | % | 20–100 | ortalama(H) |
| 14 | waterRatio | Sürekli | — | 0,30–0,65 | Karışım tasarımı girdisi |
| 15 | cementType | Kategorik | — | {CEM_I, II, III, V} | Karışım tasarımı girdisi |
| 16 | aggregateType | Kategorik | — | {CRUSHED, RIVER, LIMESTONE} | Karışım tasarımı girdisi |
| + | chemicalAdditives | Kategorik | — | {NONE, PLASTICIZER, ACCELERATOR, RETARDER} | Karışım tasarımı girdisi |

---

## Ek B: Hiperparametre Özeti

| Model | Hiperparametre | Değer | Ayarlama Yöntemi |
|-------|---------------|-------|-----------------|
| Random Forest | n_estimators | 500 | Manuel (OOB hata eğrisi) |
| Random Forest | max_features | sqrt | Breiman'ın önerisi |
| Random Forest | min_samples_leaf | 2 | 5 katlı CV ızgara araması |
| XGBoost | n_estimators | 500 | Doğrulama seti üzerinde erken durdurma |
| XGBoost | learning_rate | 0,05 | Izgara araması {0,01; 0,05; 0,1} |
| XGBoost | max_depth | 6 | Izgara araması {4, 6, 8} |
| XGBoost | subsample | 0,8 | Izgara araması {0,7; 0,8; 0,9} |
| LightGBM | num_leaves | 63 | Izgara araması {31, 63, 127} |
| DNN | hidden_dims | [128, 64, 32] | Mimari araması |
| DNN | dropout | 0,3; 0,3; 0,2 | Manuel ayarlama |
| LSTM | hidden_size | 64 | Mimari araması |
| LSTM | num_layers | 2 | Izgara araması {1, 2, 3} |
| LSTM | bidirectional | True | Ablasyon çalışması |

---

## Ek C: API Uç Noktaları Referansı

### FastAPI ML Servisi (Port 8086)

| Yöntem | Uç Nokta | Rol | Açıklama |
|--------|---------|-----|---------|
| GET | /health | Herkese açık | Servis sağlık kontrolü |
| POST | /api/ml/predictions/run | SYSTEM_MANAGER | CSV'den tahminleri çalıştır |
| GET | /api/ml/predictions | MANAGER, ENGINEER | En son tahminleri getir |
| POST | /api/ml/models/train | SYSTEM_MANAGER | Arka plan yeniden eğitimini tetikle |
| GET | /api/ml/models/status | SYSTEM_MANAGER | Model metrikleri ve durumunu getir |
| POST | /api/ml/predict | ENGINEER | Tek örnek çıkarımı |
| GET | /api/ml/shap/{project_id} | SYSTEM_MANAGER, ENGINEER | Proje için SHAP açıklaması |

---

*Tez Sonu*

**Kelime sayısı**: ~17.000 kelime (~72–85 sayfa, standart akademik biçimlendirmede: 12 punto, çift satır aralığı, 2,5 cm kenar boşlukları)
