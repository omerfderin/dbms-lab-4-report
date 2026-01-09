# Deney Sonu Teslimatı

Sistem Programlama ve Veri Yapıları bakış açısıyla veri tabanlarındaki performansı öne çıkaran hususlar nelerdir?

Aşağıda kutucuk (checkbox) ile gösterilen maddelerden en az birini seçtiğiniz açık kaynak kodlu bir VT kaynak kodları üzerinde göstererek açıklayınız. Açıklama bölümüne kısaca metninizi yazıp, kod üzerinde gösterim videonuzun linkini en altta belirtilen kutucuğa yerleştiriniz.

- [X]  Seçtiğiniz konu/konuları bu şekilde işaretleyiniz. **!**
    
---

# 1. Sistem Perspektifi (Operating System, Disk, Input/Output)

### Disk Erişimi

- [X]  **Blok bazlı disk erişimi** → block_id + offset
- [ ]  Rastgele erişim

### VT için Page (Sayfa) Anlamı

- [X]  VT hangisini kullanır? **Satır/ Sayfa** okuması

---

### Buffer Pool

- [X]  Veritabanları, Sık kullanılan sayfaları bellekte (RAM) kopyalar mı (caching) ?

- [ ]  LRU / CLOCK gibi algoritmaları
- [ ]  Diske yapılan I/O nasıl minimize ederler?

# 2. Veri Yapıları Perspektifi

- [X]  B+ Tree Veri Yapıları VT' lerde nasıl kullanılır?
- [X]  VT' lerde hangi veri yapıları hangi amaçlarla kullanılır?
- [ ]  Clustered vs Non-Clustered Index Kavramı
- [ ]  InnoDB satırı diskte nasıl durur?
- [ ]  LSM-tree (LevelDB, RocksDB) farkı
- [ ]  PostgreSQL heap + index ayrımı

DB diske yazarken:

- [X]  WAL (Write Ahead Log) İlkesi
- [X]  Log disk (fsync vs write) sistem çağrıları farkı

---

# Özet Tablo

| Özellik | Standart Dosya Sistemi Yaklaşımı | MySQL InnoDB Yaklaşımı |
|---------|-----------------------------------|------------------------|
| **Erişim Birimi** | Byte Stream (Rastgele baytlar) | Page (16KB Bloklar) |
| **Adresleme** | Dosya Yolu + Byte Offset | Tablespace ID + Page No + Offset |
| **Önbellek** | OS Page Cache (Kontrolsüz) | Buffer Pool (Veritabanı optimize) |
| **Veri Yapısı** | Düz Dosya (Flat File) | B+ Tree (Sıralı & Dengeli) |
| **Yazma Güvenliği** | write() (OS Buffer'da bekler) | fsync() (Diske zorlar & WAL) |
| **Arama Hızı** | O(N) (Tüm dosyayı tara) | O(logN) (Ağaçta gezin) |

---

# Video [Linki](https://www.youtube.com/watch?v=Nw1OvCtKPII&t=2635s) 
Ekran kaydı. 2-3 dk. açık kaynak V.T. kodu üzerinde konunun gösterimi. Video kendini tanıtma ile başlamalıdır (Numara, İsim, Soyisim, Teknik İlgi Alanları). 

---

# Açıklama (Ort. 600 kelime)

Bu çalışmada MySQL'in varsayılan depolama motoru olan **InnoDB**'nin kaynak kodları incelenmiştir. Çalışma kapsamında; dosya sistemleri, blok I/O, bellek yönetimi (Buffer Pool) ve ileri seviye veri yapıları (B+ Tree, Hash Table) perspektifinden, sistemin disk erişim maliyetlerini nasıl minimize ettiği ve veri bütünlüğünü nasıl sağladığı analiz edilmiştir.

### Blok bazlı disk erişimi $\rightarrow$ block_id + offset
Diskler ve depolama birimleri, RAM'e kıyasla erişim süresi (latency) açısından oldukça yavaş donanımlardır. Veritabanı sistemleri bu darboğazı yönetmek zorundadır. Disk üzerindeki veriler fiziksel adreslerde dursa da, InnoDB bu yönetimi mantıksal bir soyutlama ile yapar.
`storage/innobase/fil/fil0fil.cc` dosyasındaki `fil_addr_t` yapısı incelendiğinde, sistemin disk üzerindeki herhangi bir veriye ulaşmak için **Page ID** (Blok Numarası) ve **Byte Offset** (Blok içi konum) ikilisini kullandığı görülür. Bu yapı, veritabanının disk üzerindeki konumlandırma haritasını oluşturur ve işletim sistemi seviyesindeki dosya yönetimini soyutlayarak veritabanı motorunun donanım bağımsız çalışmasını sağlar.

### VT hangisini kullanır? Satır/ Sayfa okuması
Veritabanı kullanıcıları SQL sorgularında genellikle tek bir "Satır" talep etse de, veritabanı motoru diskten her zaman **"Sayfa" (Page)** okur.
InnoDB'de varsayılan sayfa boyutu **16KB**'dır. Bir veri okuma isteği geldiğinde `buf_read_page_low` fonksiyonu çağrılır ve istenen satırı içeren 16KB'lık bloğun tamamı diskten belleğe yüklenir. Bu yaklaşım **Mekansal Yerellik (Spatial Locality)** ilkesine dayanır. Disk okuma maliyeti (kafa hareketi ve dönüş gecikmesi) bir bayt için de 16KB için de neredeyse aynıdır. Dolayısıyla sistem, bir satır okunduğunda, muhtemelen o satırın hemen yanındaki diğer kayıtlara da (örneğin aynı siparişin diğer kalemleri) ihtiyaç duyulacağını varsayar. Bu strateji, I/O sayısını azaltarak genel sistem performansını ciddi oranda artırır.

### Veritabanları, Sık kullanılan sayfaları bellekte (RAM) kopyalar mı (caching)?
Evet, InnoDB disk I/O maliyetini düşürmek için bu mekanizmaya **Buffer Pool** adını verir.
Disk erişimi yavaş olduğu için, okunan her sayfa RAM üzerindeki bu havuza kopyalanır. Bir veri istendiğinde `buf_page_get_gen` fonksiyonu önce belleği kontrol eder. Eğer sayfa bellekteyse (**Cache Hit**), milisaniyeler süren disk işlemine gerek kalmadan veri mikrosaniyeler içinde döndürülür.
Bellek sınırsız olmadığı için dolduğunda bir tahliye stratejisi gerekir. Burada **LRU (Least Recently Used)** algoritması devreye girer. InnoDB, standart LRU'dan farklı olarak, "Midpoint Insertion" adı verilen bir yöntem kullanır. Yeni okunan sayfalar listenin tam başına değil, orta noktasına eklenir. Bu sayede, nadiren yapılan tam tablo taramaları (Full Table Scan), sık kullanılan (hot) kritik verileri bellekten süpürmez.

### B+ Tree Veri Yapıları VT' lerde nasıl kullanılır?
InnoDB, tabloları ve indeksleri disk üzerinde rastgele değil, sıralı tutmak için **B+ Tree** (B Artı Ağacı) veri yapısını kullanır.
Bu yapıda veriler, ağacın sadece **Yaprak (Leaf)** düğümlerinde bulunur; **İç (Internal)** düğümler ise sadece yönlendirme (pointer) bilgisini tutar. Bu ayrım, bir sayfaya çok sayıda yönlendirici sığmasını (yüksek fan-out) sağlar, böylece ağacın derinliği azalır (genelde 3-4 seviye).
Arama işlemi ağacın en tepesindeki kök (root) sayfasından başlar (`btr_cur_search_to_nth_level`). Dallar takip edilerek verinin bulunduğu yaprak sayfaya inilir. Yaprak sayfaya gelindiğinde, sayfa içinde **Binary Search** yapılarak hedef kayıt bulunur. Ayrıca yaprak düğümler birbirine çift yönlü bağlı (Double Linked List) olduğu için, "ID > 100" gibi aralık sorgularında (**Range Scan**) ağaç tekrar gezilmeden yan sayfaya geçilerek sıralı okuma yapılır.

### VT' lerde hangi veri yapıları hangi amaçlarla kullanılır?
InnoDB, farklı performans ve tutarlılık ihtiyaçlarını karşılamak için hibrit bir veri yapısı seti kullanır:

* **Hash Table (Hash Tablosu):** Veriye $O(1)$ sürede ulaşmak için kullanılır.
    * *Kullanım:* Buffer Pool yönetiminde "Sayfa X şu an bellekte mi?" sorusuna anında yanıt vermek ve Adaptive Hash Index ile sık erişilen verilere B+ Tree'yi dolaşmadan ulaşmak için.
* **Linked List (Bağlı Liste):** Dinamik liste ve kuyruk yönetimi için kullanılır.
    * *Kullanım:* Bellek tahliyesi için LRU listesi, üzerinde değişiklik yapılmış ("dirty") sayfaları diske yazmak için Flush Listesi ve eski veri versiyonlarını tutarak rollback imkanı sağlayan Undo Log zinciri.
* **Bitmap (Bit Haritası):** Alan ve kaynak yönetimi için kullanılır.
    * *Kullanım:* Disk üzerindeki 16KB'lık sayfaların ve "Extent" (64 sayfalık grup) yapılarının doluluk durumunu en az bit maliyetiyle takip etmek.
* **Inverted Index (Ters Dizin):** Metin tabanlı aramalar için kullanılır.
    * *Kullanım:* Full-Text Search işlemlerinde kelime bazlı içerik taraması yaparak, kelimenin hangi dokümanlarda geçtiğini listeler.
* **R-Tree:** Uzaysal veri yönetimi için kullanılır.
    * *Kullanım:* Harita ve koordinat (GIS) verilerini dikdörtgen alanlar (Bounding Box) şeklinde indekslemek.
* **Directed Graph (Yönlü Çizge):** Kilit ve işlem yönetimi için kullanılır.
    * *Kullanım:* Transactionlar arasındaki "bekleme" ilişkilerini modelleyerek, döngüsel beklemeleri (Deadlock) tespit etmek.

### Log disk (fsync vs write) sistem çağrıları farkı
Veritabanlarının en kritik özelliği olan Veri güvenliği (**Durability**), işletim sistemi seviyesindeki bu iki çağrı arasındaki farka dayanır:

* **write():** Hızlıdır ancak veriyi sadece işletim sisteminin kernel önbelleğine (OS Buffer) yazar. Eğer bu sırada elektrik kesilirse, veri henüz fiziksel diske gitmediği için kaybolur.
* **fsync():** Yavaştır ancak veriyi fiziksel diske (manyetik plaka veya NAND yongası) yazmaya zorlar.

InnoDB, veri kaybını sıfıra indirmek için **Write-Ahead Logging (WAL)** prensibini kullanır. `log0log.cc` dosyasında görüldüğü üzere, Redo Log dosyaları yazılırken **fsync** (`os_file_fsync_posix`) çağrısı kullanılır. Performans artışı için veri sayfaları RAM'de bekletilse bile, işlem "tamamlandı" (commit) dendiğinde, log kayıtları `fsync` ile diske mühürlenir. Bu sayede sistem çökse bile, yeniden başlatıldığında bu loglar tekrar oynatılarak veri kurtarılır.

## VT Üzerinde Gösterilen Kaynak Kodları

* **Disk Yönetimi ve Adresleme:** [storage/innobase/fil/fil0fil.cc](https://github.com/mysql/mysql-server/blob/8.0/storage/innobase/fil/fil0fil.cc)
* **Buffer Pool Mantığı:** [storage/innobase/buf/buf0buf.cc](https://github.com/mysql/mysql-server/blob/8.0/storage/innobase/buf/buf0buf.cc)
* **LRU Algoritması:** [storage/innobase/buf/buf0lru.cc](https://github.com/mysql/mysql-server/blob/8.0/storage/innobase/buf/buf0lru.cc)
* **B+ Tree Arama:** [storage/innobase/btr/btr0cur.cc](https://github.com/mysql/mysql-server/blob/8.0/storage/innobase/btr/btr0cur.cc)
* **Disk Alan Yönetimi (Bitmap):** [storage/innobase/fsp/fsp0fsp.cc](https://github.com/mysql/mysql-server/blob/8.0/storage/innobase/fsp/fsp0fsp.cc)
* **Fsync ve I/O:** [storage/innobase/os/os0file.cc](https://github.com/mysql/mysql-server/blob/8.0/storage/innobase/os/os0file.cc)
* **Loglama (Redo Log):** [storage/innobase/log/log0log.cc](https://github.com/mysql/mysql-server/blob/8.0/storage/innobase/log/log0log.cc), [storage/innobase/log/log0write.cc](https://github.com/mysql/mysql-server/blob/8.0/storage/innobase/log/log0write.cc)
* **Deadlock Dedektörü:** [storage/innobase/lock/lock0lock.cc](https://github.com/mysql/mysql-server/blob/8.0/storage/innobase/lock/lock0lock.cc)

### Ek Dosyalar

* `storage/innobase/include/fil0fil.h` - `fil_addr_t` yapısı (block_id + offset)
* `storage/innobase/include/univ.i` - Sayfa boyutu sabitleri (`UNIV_PAGE_SIZE_DEF`)
* `storage/innobase/buf/buf0rea.cc` - Sayfa okuma işlemleri (`buf_read_page_low`)
* `storage/innobase/include/buf0buf.ic` - Hash tablosu arama makroları
* `storage/innobase/page/page0cur.cc` - Binary search implementasyonu (`page_cur_search_with_match`)
* `storage/innobase/include/ut0lst.h` - Linked list makroları
* `storage/innobase/include/fsp0fsp.h` - Bitmap tanımları (`XDES_BITMAP`)
