# Deney Sonu Teslimatı

Sistem Programlama ve Veri Yapıları bakış açısıyla veri tabanlarındaki performansı öne çıkaran hususlar nelerdir?

Veritabanı performansı, sistem programlama ve veri yapıları perspektifinden şu noktalara dayanır:
Performans için bellek hiyerarisi optimizasyonu kritiktir. RAM ile disk arasındaki hız farkı nedeniyle sık kullanılan veriler bellekte tutulur. Buffer pool bu işlevi görür; veriler RAM'de olduğunda disk erişimi düşer ve performans artar.
Veri yapısı seçimi belirleyicidir. Index'ler için B+ Tree kullanılır; yüksek düğüm kapasitesi sayesinde yükseklik düşük kalır ve arama hızlıdır. Yaprakların bağlı yapısı sıralı erişimi hızlandırır. Buffer pool'da sayfalara hızlı erişim için hash tabloları kullanılır ve bit mask gibi optimizasyonlarla performans artırılır.
Disk I/O yaklaşımı verimliliği etkiler. Satır bazlı yerine sayfa bazlı erişim tercih edilir; disk bloklarıyla uyumlu olan bu yaklaşım, toplu okuma sayesinde I/O sayısını azaltır. Sistem çağrıları dikkatli seçilir: write hızlıdır ancak crash'te veri kaybı riski taşır; fsync ise senkronizasyon sağlar ancak daha yavaştır. WAL prensibiyle önce log'a yazılır ve group commit ile fsync çağrıları azaltılır. Eşzamanlı erişim için thread-safe I/O çağrıları kullanılır; dosya pointer lock'u gerektirmeyen bu yaklaşım paralellik sağlar.
Sonuç olarak, performans; bellekte tutma, uygun veri yapıları seçimi, blok bazlı I/O ve doğru sistem çağrıları tercihlerinin birleşimiyle elde edilir. 

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
- [ ]  VT' lerde hangi veri yapıları hangi amaçlarla kullanılır?
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
| **Erişim Birimi** | Byte Stream | Sayfa (16KB Bloklar) |
| **Adresleme** | Dosya Yolu + Byte Offset | Tablespace ID + Page No + Offset |
| **Önbellek** | OS Page Cache (Kontrolsüz) | Buffer Pool (Veritabanı optimize) |
| **Veri Yapısı** | Düz Dosya | B+ Tree (Sıralı & Dengeli) |
| **Yazma Güvenliği** | write() (OS Buffer'da bekler) | fsync() (Diske zorlar & WAL) |
| **Arama Hızı** | O(N) (Tüm dosyayı tara) | O(logN) (Ağaçta gezin) |

---

# Video [Linki](https://www.youtube.com/watch?v=Nw1OvCtKPII&t=2635s) 
Ekran kaydı. 2-3 dk. açık kaynak V.T. kodu üzerinde konunun gösterimi. Video kendini tanıtma ile başlamalıdır (Numara, İsim, Soyisim, Teknik İlgi Alanları). 

---

# Açıklama (Ort. 600 kelime)

Bu çalışmada MySQL'in varsayılan depolama motoru olan **InnoDB**'nin kaynak kodları incelenmiştir. Çalışma kapsamında; dosya sistemleri, blok I/O, bellek yönetimi  ve  B+ Tree, Hash Table gibi veri yapıları perspektifinden, sistemin disk erişim maliyetlerini nasıl minimize ettiği ve veri bütünlüğünü nasıl sağladığı incelenmiştir.

### Blok bazlı disk erişimi $\rightarrow$ block_id + offset
Diskler ve depolama birimleri, RAM'e kıyasla erişim süresi açısından oldukça yavaş donanımlardır. Veritabanı sistemleri bu konuyu yönetmek zorundadır. Disk üzerindeki veriler fiziksel adreslerde dursa da, InnoDB bu yönetimi mantıksal bir soyutlama ile yapar.
`storage/innobase/fil/fil0fil.cc` dosyasındaki `fil_addr_t` yapısı incelendiğinde, sistemin disk üzerindeki herhangi bir veriye ulaşmak için **Page ID** (Blok Numarası) ve **Byte Offset** (Blok içi konum) ikilisini kullandığı görülür. Bu yapı, veritabanının disk üzerindeki haritasını oluşturur ve işletim sistemi seviyesindeki dosya yönetimini soyutlayarak veritabanı motorunun donanım bağımsız çalışmasını sağlar.

### VT hangisini kullanır? Satır/ Sayfa okuması
Veritabanı kullanıcıları SQL sorgularında tek bir "Satır" talep etse de, veritabanı motoru diskten her zaman **"Sayfa" (Page)** okur.
InnoDB'de varsayılan sayfa boyutu **16KB**'dır. Bir veri okuma isteği geldiğinde `buf_read_page_low` fonksiyonu çağrılır ve istenen satırı içeren 16KB'lık bloğun tamamı diskten belleğe yüklenir. Disk okuma maliyeti bir bayt için de 16KB için de neredeyse aynıdır. Dolayısıyla sistem, bir satır okunduğunda, muhtemelen o satırın hemen yanındaki diğer kayıtlara da (örneğin aynı siparişin diğer kalemleri) ihtiyaç duyulacağını varsayar. Bu strateji, I/O sayısını azaltarak genel sistem performansını ciddi oranda artırır.

### Veritabanları, Sık kullanılan sayfaları bellekte (RAM) kopyalar mı (caching)?
Evet, InnoDB disk I/O maliyetini düşürmek için bu mekanizmaya **Buffer Pool** adını verir.
Disk erişimi yavaş olduğu için, okunan her sayfa RAM üzerindeki bu yapıya kopyalanır. Bir veri istendiğinde `buf_page_get_gen` fonksiyonu önce belleği kontrol eder. Eğer sayfa bellekteyse, milisaniyeler süren disk işlemine gerek kalmadan veri mikrosaniyeler içinde döndürülür.
Bellek sınırsız olmadığı için, sadece sık kullanılan sayfalar RAM'de tutulur. `buf_page_get_gen` fonksiyonu çağrıldığında önce hash tablosunda arama yapılır. Eğer sayfa bulunursa, veri RAM'den doğrudan okunur. Bulunamazsa, sayfa diskten okunur ve buffer pool'a eklenir. İyi yapılandırılmış sistemlerde aranan dosyaların hash tablosunda bulunma olasılığı yüksektir, bu da disk I/O'sunu neredeyse sıfırlar ve performansı artırır.

### B+ Tree Veri Yapıları VT' lerde nasıl kullanılır?
InnoDB, tabloları ve indeksleri disk üzerinde rastgele değil, sıralı tutmak için **B+ Tree** veri yapısını kullanır.
Bu yapıda veriler, ağacın sadece **Yaprak** düğümlerinde bulunur; **İç** düğümler ise sadece yönlendirme bilgisini tutar. Bu ayrım, bir sayfaya çok sayıda yönlendirici sığmasını sağlar, böylece ağacın derinliği azalır.
Arama işlemi ağacın en tepesindeki kök sayfasından başlar (`btr_cur_search_to_nth_level`). Dallar takip edilerek verinin bulunduğu yaprak sayfaya inilir. Yaprak sayfaya gelindiğinde, sayfa içinde **Binary Search** yapılarak hedef kayıt bulunur. Ayrıca yaprak düğümler birbirine çift yönlü bağlı (Double Linked List) olduğu için, "ID > 100" gibi aralık sorgularında ağaç tekrar gezilmeden yan sayfaya geçilerek sıralı okuma yapılır.

### Log disk (fsync vs write) sistem çağrıları farkı
Veritabanlarının en kritik özelliği olan Veri güvenliği, işletim sistemi seviyesindeki bu iki çağrı arasındaki farka dayanır:

* **write():** Hızlıdır ancak veriyi sadece işletim sisteminin kernel önbelleğine (OS Buffer) yazar. Eğer bu sırada elektrik kesilirse, veri henüz fiziksel diske gitmediği için kaybolur.
* **fsync():** Yavaştır ancak veriyi fiziksel diske yazmaya zorlar.

### WAL (Write Ahead Log) İlkesi

InnoDB, veri kaybını sıfıra indirmek için **Write-Ahead Logging (WAL)** prensibini kullanır. `log0log.cc` dosyasında görüldüğü üzere, Redo Log dosyaları yazılırken **fsync** (`os_file_fsync_posix`) çağrısı kullanılır. Performans artışı için veri sayfaları RAM'de bekletilse bile, işlem "tamamlandı" dendiğinde, log kayıtları `fsync` ile diske mühürlenir. Bu sayede sistem çökse bile, yeniden başlatıldığında bu loglar tekrar oynatılarak veri kurtarılır.

## VT Üzerinde Gösterilen Kaynak Kodları

* **Blok Bazlı Disk Erişimi (Block ID + Offset):** [storage/innobase/include/fil0fil.h](https://github.com/mysql/mysql-server/blob/8.0/storage/innobase/include/fil0fil.h) [storage/innobase/fil/fil0fil.cc](https://github.com/mysql/mysql-server/blob/8.0/storage/innobase/fil/fil0fil.cc)
* **Sayfa Bazlı Okuma (Satır/ Sayfa Okuması):** [storage/innobase/include/univ.i](https://github.com/mysql/mysql-server/blob/8.0/storage/innobase/include/univ.i), [storage/innobase/buf/buf0rea.cc](https://github.com/mysql/mysql-server/blob/8.0/storage/innobase/buf/buf0rea.cc)
* **Buffer Pool Caching (Sık Kullanılan Sayfaları Bellekte Tutma):** [storage/innobase/buf/buf0buf.cc](https://github.com/mysql/mysql-server/blob/8.0/storage/innobase/buf/buf0buf.cc)
* **B+ Tree Veri Yapısı:** [storage/innobase/btr/btr0cur.cc](https://github.com/mysql/mysql-server/blob/8.0/storage/innobase/btr/btr0cur.cc)
* **Log Disk Yönetimi (fsync vs write):** [storage/innobase/os/os0file.cc](https://github.com/mysql/mysql-server/blob/8.0/storage/innobase/os/os0file.cc)
* **WAL İlkesi (Write-Ahead Logging):**  [storage/innobase/mtr/mtr0mtr.cc](https://github.com/mysql/mysql-server/blob/8.0/storage/innobase/mtr/mtr0mtr.cc), [storage/innobase/log/log0log.cc](https://github.com/mysql/mysql-server/blob/8.0/storage/innobase/log/log0log.cc)
