# HyperLogLog — Olasılıksal Kardinalite Tahmini

Java ile sıfırdan yazılmış **HyperLogLog (HLL)** implementasyonu. Büyük veri setlerinde tekil eleman sayısını düşük bellek kullanımıyla tahmin eder.

---

## İçindekiler

- [Nedir?](#nedir)
- [Algoritma Nasıl Çalışır?](#algoritma-nasıl-çalışır)
- [Teorik Hata Analizi](#teorik-hata-analizi)
- [Kurulum ve Çalıştırma](#kurulum-ve-çalıştırma)
- [Kullanım](#kullanım)
- [API Referansı](#api-referansı)
- [Test Çıktısı](#test-çıktısı)


---

## Nedir?

HyperLogLog, bir veri setindeki **tekil eleman sayısını** (cardinality) saymak için kullanılan olasılıksal bir veri yapısıdır. Tam sayım yerine istatistiksel tahmin yapar; bu sayede milyarlarca elemanı yalnızca birkaç kilobayt bellekle işleyebilir.

| Yöntem | 10⁹ eleman için bellek | Hata |
|---|---|---|
| HashSet (tam sayım) | ~8 GB | %0 |
| HyperLogLog (b=14) | ~16 KB | ~%0.81 |

**Tipik kullanım alanları:** günlük aktif kullanıcı sayımı, ağ trafiğinde tekil IP sayısı, arama motorlarında tekil sorgu sayısı.

---

## Algoritma Nasıl Çalışır?

### 1. Hash Fonksiyonu
Her eleman FNV-1a ile hash'lenir, ardından MurmurHash3 finalizasyon mixi ile 32-bit uniform bir değere dönüştürülür. Uniform dağılım, tahminin doğruluğu için kritiktir.

### 2. Bucketing (Kovalama)
Hash değerinin üst `b` biti kova indeksini (`bucketIndex`) belirler. `m = 2^b` adet kova kullanılır.

```
hashVal = 10110011 00101010 ...
          |--b bit--|------ w -------|
           bucketIndex     rho hesabı
```

### 3. Rho (ρ) — Ardışık Sıfır Sayısı
Kalan bitlerin (`w`) en baştaki 1-bitinin konumu `ρ` olarak tanımlanır (1-tabanlı). Bu değer her kovadaki **maksimum** ile karşılaştırılarak register güncellenir.

```
w = 00110...  →  ρ = 3   (2 sıfır + 1 bit = 3. konumda ilk 1)
```

### 4. Kardinalite Tahmini
Harmonik ortalama formülü ile ham tahmin hesaplanır:

```
E = αm · m² / Σ(2^(-register[i]))
```

Ardından iki düzeltme uygulanır:

- **Küçük aralık** (`E ≤ 2.5m`): Boş kova varsa LinearCounting kullanılır → `m · ln(m / v)`
- **Büyük aralık** (`E > 2³²/30`): Büyük değer düzeltmesi → `-2³² · ln(1 - E/2³²)`

### 5. Merge (Birleştirme)
İki HLL yapısı veri kaybı olmadan birleştirilebilir. Her kova için maksimum register değeri alınır:

```
merged.register[i] = max(A.register[i], B.register[i])
```

Bu özellik sayesinde dağıtık sistemlerde paralel hesaplanan HLL'ler birleştirilerek küme birleşiminin kardinalitesi tahmin edilebilir.

---

## Teorik Hata Analizi

Standart göreceli hata formülü (Flajolet et al., 2007):

```
σ_relative ≈ 1.04 / √m
```

`m` dört katına çıktığında hata yarıya iner:

| b | m (kova sayısı) | Teorik hata | Bellek |
|---|---|---|---|
| 4 | 16 | %26.00 | 16 B |
| 6 | 64 | %13.00 | 64 B |
| 8 | 256 | %6.50 | 256 B |
| 10 | 1.024 | %3.25 | 1 KB |
| 12 | 4.096 | %1.63 | 4 KB |
| 14 | 16.384 | %0.81 | 16 KB |
| 16 | 65.536 | %0.41 | 64 KB |

Çoğu üretim uygulaması için **b=12** veya **b=14** iyi bir denge noktasıdır.

---

## Kurulum ve Çalıştırma

Gereksinim: **Java 8+**

```bash
# Derleme
javac HyperLogLog.java

# Çalıştırma
java HyperLogLog
```

---

## Kullanım

```java
// 1. HLL oluştur (b=14 → m=16384, ~%0.81 hata)
HyperLogLog hll = new HyperLogLog(14);

// 2. Eleman ekle
hll.add("kullanici_1");
hll.add("kullanici_2");
hll.add("kullanici_1"); // tekrar — sayılmaz

// 3. Tahmini kardinaliteyi al
long tahmin = hll.count(); // ≈ 2

// 4. İki HLL'i birleştir (A ∪ B)
HyperLogLog hllA = new HyperLogLog(12);
HyperLogLog hllB = new HyperLogLog(12);
// ... elemanlari ekle ...
HyperLogLog birlesik = hllA.merge(hllB);
long birlesikTahmin = birlesik.count();
```

---

## API Referansı

### `HyperLogLog(int b)`
Yeni bir HLL yapısı oluşturur.
- `b`: Kova bit sayısı. `[4, 16]` aralığında olmalıdır. Kova sayısı `m = 2^b`'dir.

### `void add(String item)`
Veri yapısına bir eleman ekler. Aynı eleman birden fazla eklenirse register değişmez.

### `long count()`
Tahmini tekil eleman sayısını döndürür. Küçük ve büyük aralık düzeltmeleri otomatik uygulanır.

### `HyperLogLog merge(HyperLogLog other)`
İki HLL yapısını birleştirerek yeni bir HLL döndürür. Her iki yapının `b` değeri eşit olmalıdır.

### `double theoreticalError()`
Bu yapı için teorik göreceli standart hatayı `[0, 1]` aralığında döndürür (`1.04 / √m`).

---

## Test Çıktısı

```
=== HyperLogLog Test ===

-- Test 1: Temel ekleme --
Gerçek tekil eleman : 4
HLL tahmini         : 4

-- Test 2: Büyük veri seti (N=100.000) --
Gerçek sayı  : 100.000
HLL tahmini  : 99.412
Gerçek hata  : 0.59%
Teorik hata  : 0.81%

-- Test 3: merge() — A ∪ B tahmini --
Gerçek |A ∪ B|    : 75.000
|A| tahmini        : 50.218
|B| tahmini        : 49.876
|A ∪ B| tahmini    : 75.104

-- Test 4: b artınca hata azalır --
b     m        Teorik σ (%)     Gerçek hata (%)
--------------------------------------------------
4     16       26.00            31.82
6     64       13.00            14.55
8     256      6.50             5.28
10    1024     3.25             2.91
12    4096     1.63             1.24
14    16384    0.81             0.67
16    65536    0.41             0.38
```

---

