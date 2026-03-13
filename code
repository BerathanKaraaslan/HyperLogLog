import java.nio.charset.StandardCharsets;

/**
 * HyperLogLog — Cardinality Estimation (Küme Büyüklüğü Tahmini)
 *
 * Teorik standart hata: σ ≈ 1.04 / √m
 *   b=10 → m=1024    → ~%3.25 hata
 *   b=12 → m=4096    → ~%1.63 hata
 *   b=14 → m=16384   → ~%0.81 hata
 *   b=16 → m=65536   → ~%0.41 hata
 *
 * Kaynak: Flajolet et al., "HyperLogLog: the analysis of a near-optimal
 * cardinality estimation algorithm", DMTCS 2007.
 */
class HyperLogLog {

    private static final long TWO_32 = 1L << 32; // 2^32 = 4294967296

    private final int b;         // kova bit sayısı (4–16)
    private final int m;         // kova sayısı = 2^b
    private final byte[] registers; // her kovadaki maksimum rho değeri
    private final double alphaM;    // bias düzeltme sabiti

    // -------------------------------------------------------------------------
    // Kurucu
    // -------------------------------------------------------------------------

    public HyperLogLog(int b) {
        if (b < 4 || b > 16) {
            throw new IllegalArgumentException(
                    "b değeri 4 ile 16 arasında olmalıdır, verildi: " + b);
        }
        this.b = b;
        this.m = 1 << b;                  // 2^b
        this.registers = new byte[m];     // Java byte[] sıfırla başlar
        this.alphaM = computeAlpha(m);
    }

    // -------------------------------------------------------------------------
    // Alpha (bias) sabiti — Flajolet et al. Tablo 1
    // -------------------------------------------------------------------------

    private static double computeAlpha(int m) {
        switch (m) {
            case 16:  return 0.673;
            case 32:  return 0.697;
            case 64:  return 0.709;
            default:  return 0.7213 / (1.0 + 1.079 / m);
        }
    }

    // -------------------------------------------------------------------------
    // Hash fonksiyonu — MurmurHash3 (32-bit finalization karıştırıcısı)
    //
    // FNV-1a gibi basit hash'ler düşük entropi girdilerde (kısa string'ler,
    // ardışık sayılar) bit dağılımını bozabilir. MurmurHash3'ün avalanche
    // mixi, 32-bit çıktı uzayını düzgün doldurur.
    // -------------------------------------------------------------------------

    private static int murmurHash3Mix(int key) {
        // Bayt dizisini int'e dönüştür (Latin-1 yeterli, karakter başına 1 bayt)
        // Gerçek girdiler için UTF-8 baytları üzerinde çalışmak daha doğrudur.
        key ^= (key >>> 16);
        key *= 0x85ebca6b;
        key ^= (key >>> 13);
        key *= 0xc2b2ae35;
        key ^= (key >>> 16);
        return key;
    }

    /**
     * String'i 32-bit hash değerine dönüştürür.
     * İç kısımda FNV-1a ile bayt karıştırması, ardından MurmurHash3 finalizasyonu.
     */
    private int hash(String item) {
        byte[] bytes = item.getBytes(StandardCharsets.UTF_8);
        int h = 0x811c9dc5; // FNV offset basis
        for (byte b : bytes) {
            h ^= (b & 0xFF);
            h *= 0x01000193; // FNV prime
        }
        return murmurHash3Mix(h); // avalanche ile uniform dağılım
    }

    // -------------------------------------------------------------------------
    // rho (ρ): w bitlerinin en-soldan kaçıncı bit'te 1 gördüğü
    //   rho = pozisyon(ilk 1 biti) + 1   (1-tabanlı)
    //   Örnek: w = 00110... → rho = 3
    //   w = 00000...00 (tümü sıfır) → rho = 32 - b + 1  (maksimum)
    // -------------------------------------------------------------------------

    private int rho(int w) {
        // numberOfLeadingZeros(0) = 32, dolayısıyla w=0 için rho=33.
        // Ancak anlamlı maksimum değer (32 - b + 1) olduğundan klampleriz.
        int leading = Integer.numberOfLeadingZeros(w);
        int rhoVal  = leading + 1;
        int maxRho  = 32 - b + 1;
        return Math.min(rhoVal, maxRho);
    }

    // -------------------------------------------------------------------------
    // add — Eleman ekle
    // -------------------------------------------------------------------------

    public void add(String item) {
        int hashVal = hash(item);

        // Üst b bit → kova indeksi (unsigned right shift garantisi için >>> kullan)
        // & (m-1) ile [0, m) aralığına klampla (negatif hash'lere karşı güvenli)
        int bucketIndex = (hashVal >>> (32 - b)) & (m - 1);

        // Alt (32-b) bit → rho hesabı için w
        // Sol kaydırma ile üst b biti temizle
        int w = hashVal << b;

        int rhoVal = rho(w);

        // Kova kaydını sadece büyük değerle güncelle
        // byte & 0xFF: Java'da byte işaretli (-128..127), 0xFF ile unsigned okuma
        int current = registers[bucketIndex] & 0xFF;
        if (rhoVal > current) {
            registers[bucketIndex] = (byte) rhoVal;
        }
    }

    // -------------------------------------------------------------------------
    // count — Kardinalite tahmini
    //   1. Ham tahmin: E = αm · m² · (Σ 2^(-register[i]))⁻¹
    //   2. Küçük aralık düzeltmesi (LinearCounting): E ≤ 5/2 · m ve boş kova var
    //   3. Büyük aralık düzeltmesi: E > 1/30 · 2^32
    // -------------------------------------------------------------------------

    public long count() {
        // Harmonik ortalama paydası: Z = Σ 2^(-register[i])
        double z = 0.0;
        for (int i = 0; i < m; i++) {
            z += Math.pow(2.0, -(registers[i] & 0xFF)); // unsigned okuma!
        }

        // Ham tahmin
        double e = alphaM * (double) m * (double) m / z;

        // --- Küçük aralık düzeltmesi (LinearCounting) ---
        if (e <= 2.5 * m) {
            int v = 0; // boş kova sayısı
            for (int i = 0; i < m; i++) {
                if (registers[i] == 0) v++;
            }
            if (v > 0) {
                // LinearCounting: m * ln(m / v)
                return (long) (m * Math.log((double) m / v));
            }
            // Boş kova yoksa ham tahmin kullan
            return (long) e;
        }

        // --- Büyük aralık düzeltmesi ---
        if (e > TWO_32 / 30.0) {
            // -2^32 * ln(1 - E/2^32)
            return (long) (-TWO_32 * Math.log(1.0 - e / TWO_32));
        }

        // --- Orta aralık: ham tahmin doğrudan kullanılır ---
        return (long) e;
    }

    // -------------------------------------------------------------------------
    // merge — İki HLL yapısını birleştir (veri kaybı olmadan)
    //   Her kova için maksimum register değeri alınır.
    //   Özellik: merge(A, B).count() ≈ count(A ∪ B)
    // -------------------------------------------------------------------------

    public HyperLogLog merge(HyperLogLog other) {
        if (this.b != other.b) {
            throw new IllegalArgumentException(
                    "Birleştirilecek HLL yapıları aynı b değerine sahip olmalıdır. " +
                            "this.b=" + this.b + ", other.b=" + other.b);
        }
        HyperLogLog merged = new HyperLogLog(this.b);
        for (int i = 0; i < m; i++) {
            // Her iki kaynakta da unsigned okuma
            int a = this.registers[i]  & 0xFF;
            int o = other.registers[i] & 0xFF;
            merged.registers[i] = (byte) Math.max(a, o);
        }
        return merged;
    }

    // -------------------------------------------------------------------------
    // Yardımcı: teorik standart hatayı döndür
    // -------------------------------------------------------------------------

    public double theoreticalError() {
        return 1.04 / Math.sqrt(m);
    }

    // -------------------------------------------------------------------------
    // toString
    // -------------------------------------------------------------------------

    @Override
    public String toString() {
        return String.format(
                "HyperLogLog{b=%d, m=%d, tahminiKardinality=%d, teorikHata=%.2f%%}",
                b, m, count(), theoreticalError() * 100);
    }

    // -------------------------------------------------------------------------
    // main — Test ve demonstrasyon
    // -------------------------------------------------------------------------

    public static void main(String[] args) {

        System.out.println("=== HyperLogLog Test ===\n");

        // ---- Test 1: Temel ekleme ve sayım ----
        System.out.println("-- Test 1: Temel ekleme --");
        HyperLogLog hll = new HyperLogLog(10); // m=1024, ~%3.25 hata
        String[] meyveler = {"elma", "armut", "muz", "elma", "kivi", "elma", "armut"};
        for (String m : meyveler) hll.add(m);
        System.out.println("Gerçek tekil eleman : 4");
        System.out.println("HLL tahmini         : " + hll.count());
        System.out.println(hll);

        // ---- Test 2: Büyük veri seti ----
        System.out.println("\n-- Test 2: Büyük veri seti (N=100_000) --");
        HyperLogLog hll2 = new HyperLogLog(14); // m=16384, ~%0.81 hata
        int N = 100_000;
        for (int i = 0; i < N; i++) {
            hll2.add("kullanici_" + i);
        }
        long tahmin2 = hll2.count();
        double hata2 = Math.abs(tahmin2 - N) * 100.0 / N;
        System.out.printf("Gerçek sayı  : %,d%n", N);
        System.out.printf("HLL tahmini  : %,d%n", tahmin2);
        System.out.printf("Gerçek hata  : %.2f%%%n", hata2);
        System.out.printf("Teorik hata  : %.2f%%%n", hll2.theoreticalError() * 100);

        // ---- Test 3: merge (birleştirme) ----
        System.out.println("\n-- Test 3: merge() -- A ∪ B tahmini --");
        HyperLogLog hllA = new HyperLogLog(12);
        HyperLogLog hllB = new HyperLogLog(12);

        // A: kullanici_0 .. kullanici_49999
        for (int i = 0;      i < 50_000; i++) hllA.add("kullanici_" + i);
        // B: kullanici_25000 .. kullanici_74999  (25_000 ortak eleman)
        for (int i = 25_000; i < 75_000; i++) hllB.add("kullanici_" + i);

        HyperLogLog hllMerged = hllA.merge(hllB);
        int gercekBirlesen = 75_000; // 0..74999 = 75_000 tekil

        System.out.println("Gerçek |A ∪ B|    : " + gercekBirlesen);
        System.out.println("|A| tahmini        : " + hllA.count());
        System.out.println("|B| tahmini        : " + hllB.count());
        System.out.println("|A ∪ B| tahmini    : " + hllMerged.count());

        // ---- Test 4: m artırımı → hata azalması ----
        System.out.println("\n-- Test 4: b (kova bit sayısı) artınca hata azalır --");
        System.out.printf("%-5s %-8s %-16s %-12s%n", "b", "m", "Teorik σ (%)", "Gerçek hata (%)");
        System.out.println("-".repeat(50));
        int nTest = 50_000;
        for (int bVal = 4; bVal <= 16; bVal += 2) {
            HyperLogLog h = new HyperLogLog(bVal);
            for (int i = 0; i < nTest; i++) h.add("item_" + i);
            long est = h.count();
            double errPct = Math.abs(est - nTest) * 100.0 / nTest;
            System.out.printf("%-5d %-8d %-16.2f %-12.2f%n",
                    bVal, 1 << bVal, h.theoreticalError() * 100, errPct);
        }
    }
}
