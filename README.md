# Low Code Api Builder

**Yeni Nesil, Yüksek Performanslı ve Bellek İçi (In-Memory) Veri Yönetimli İş Akışı (Workflow) Motoru**

Low Code Api Builder, geleneksel süreç yönetimi (BPM) ve otomasyon araçlarının hantallığını ortadan kaldırmak üzere geliştirdiğim, mikroservis odaklı, paralel işlem gücü yüksek bir **Workflow Engine** ve modern bir **Visual Designer** platformudur.



---

## 🚀 Projenin Öne Çıkan Özellikleri ve Mimari Yaklaşımım

Piyasadaki diğer BPM araçlarından (Camunda, n8n, vb.) farklı olarak asıl odaklandığım nokta, veriyi işleme biçimi ve paralel çalışma kapasitesinin maksimum performansta tasarlanması oldu.

### 1. DuckDB Destekli In-Memory State Yönetimi (Sıfır Payload Sızıntısı)
Geleneksel araçlar, node'lar (düğümler) arasında veri aktarımı yaparken devasa JSON dosyalarını HTTP payload'ları veya RAM üzerinden birbirine zincirleme iletir. Bu, büyük verilerde Engine'i ciddi anlamda yorar. 
**Geliştirdiğim Mimari:** Düğümler arası veriyi zincirlemek yerine, arka planda çalışan **DuckDB Memory Service** ve **Variable Service** kurgulayarak veriyi merkezi bir bellek havuzuna (`In-Memory`) yazmayı hedefledim. Düğümler (`TaskNode`, `SqliteNode`, `WebServiceNode`) sadece referansları (Expected Outputs) güncelliyor. Milyon satırlık bir tablo saniyeler içinde bellekte tutulup SQL ile filtrelenerek sonraki adıma aktarılabiliyor.

### 2. Wave-Based Parallel Execution (Gerçek Paralellik)
İş akışı `Parallel Start` ile n-kollu (10, 20 vb.) dallara ayrıldığında süreçlerin birbirini beklemesini istemedim.
**Geliştirdiğim Mimari:** Yönlü Asiklik Grafik (DAG) ve *In-Degree* algoritması sayesinde bağımsız olan tüm görevlerin aynı anda bir "Dalga (Wave)" oluşturarak asenkron (`Task.WhenAll`) tetiklenmesini sağladım. Her düğümü `AddTransient` DI (Dependency Injection) yapısıyla enjekte ettiğim için aynı anda çalışan 10 farklı Web Servis düğümü birbirinin değişkenlerini (state) hiçbir zaman ezmiyor.

### 3. Dinamik Provider ve Bağlantı (Connection) Soyutlaması
Çoğu araçta veritabanı ayarlarının sabit olmasına karşın daha dinamik bir yapı kurmaya çalıştım. 
**Geliştirdiğim Mimari:** `EXEC_WEB_SERVICE` veya `EXEC_SQL` görevlerinde dinamik bir Provider seçildiğinde, Engine çalışma anında (runtime) yetkiyi (`ConnectionString` ve `Auth`) değiştirerek hedefe yöneliyor. Böylece farklı sunuculardaki MSSQL, Oracle, REST ve SOAP servislerini tek bir süreç ağacında izole olarak harmanlayabiliyorum.

### 4. Stop-On-Error ve Kusursuz History Loglama
Bir düğüm hata aldığında sadece .NET Exception fırlatması yeterli değildi. Servisin `HTTP 200 OK` dönüp içerikte Hata Kodu (`SONUCBASARILIKODU`) gönderdiği "gizli" hataları da yakalamam gerekiyordu. Bunun için özel bir response modeli (`WebServiceResponseModel`) tasarladım. Başarısızlık durumunda veritabanı yorulmuyor, süreç anında `Suspended` veya `Failed` durumuna geçerek `BpmHistoryService` üzerinden loglanıyor.

---

## 🏗 Katmanlı Mimari (Architecture)

Low Code Api Builder projesini tamamen modüler yapıda, Clean Architecture prensiplerine sadık kalarak tasarladım.

* **Bpm.WorkFlow.WorkFlowManager:** Uygulamanın kalbi. Süreçlerin (Pipeline) yürütüldüğü, topolojik sıralamanın yapıldığı ve Paralel tetiklemelerin yönetildiği orkestrasyon merkezini burada kurguladım.
* **Bpm.Environment.Service:** Uygulamanın hafızası. Süreç çalışırken geçici olarak oluşturulan tüm değişkenlerin `VariableService` ile, devasa dataların ise `DuckDbMemoryService` ile yönetilmesini sağladım. Süreç bitince havuzu otomatik temizliyorum.
* **Bpm.NodeMiddleware:** Düğümlerin iş mantığı. Yeni bir eylem ekleneceği zaman `IActivityMiddleware` interface'inden türetilip sisteme tanıtılabilmesini (Open-Closed Prensibi) mümkün kıldım.
* **Bpm.Service.WebServiceClient / SqlClient:** Dış dünyaya bağlanan izole servis kütüphaneleri. `IDbAccess` ve `IWebService` aracılığıyla veritabanı ve API etkileşimlerini tamamen soyutladım.
* **Flowlogic Designer (Frontend):** Modern arayüz tasarımı. Sürükle-bırak (React Flow) altyapısı, detaylı konfigürasyon (Properties Panel) kartları ve modern UI elementleri kullanarak kullanıcı dostu bir deneyim hedefledim.

---

## 🛠 Kullanım Senaryoları & Örnekler

### Örnek 1: E-Ticaret Sipariş Orkestrasyonu
1. **Start Node:** Müşteriden gelen sepet bilgisi `initialData` olarak sisteme girer.
2. **Parallel Start:** 
   - Dal 1: `WebServiceNode` ile kargo firmasından (Örn: Aras Kargo SOAP Servisi) takip no alınır.
   - Dal 2: `TaskNode` ile yerel MSSQL veritabanından stok düşümü yapılır.
3. **If Node:** Stok kontrolü başarısızsa akış `False` dalına gidip "Siparişi İptal Et" Web servisini tetikler.
4. **End Node:** Tüm süreç verileri toplanıp `ResponseNode` aracılığıyla API sonucu olarak saniyeler içinde döner.

### Örnek 2: DuckDB ile Milyon Satırlık Veri Analizi
1. `SqliteNode` ile dışarıdan 1 Milyon satırlık satış verisi çekilip bellekte `MyTempSalesTable` olarak saklanır.
2. `SetVariableNode` (Expression modu) kullanılarak bu tablo üzerinde filtremeler yapılır (`SELECT SUM(Tutar) FROM MyTempSalesTable WHERE Kategori='Teknoloji'`).
3. Sonuç, `VariableService` aracılığıyla hızlıca bir SMS servisine iletilir. Sunucu diskine tek bir byte bile yazılmadan tüm işlemi doğrudan RAM üzerinde tamamlamış oluyorum.

---

## ⚙️ Teknik Kurulum (Developer Guide)

### Gereksinimler
- .NET 9.0 SDK
- Node.js & npm (Designer UI için)
- MS SQL Server (veya desteklenen başka bir Provider, Log/History için)

### Backend (Engine) Başlatma
```bash
cd Bpm.Api.Management
dotnet build
dotnet run
```
*API ayağa kalktığında Swagger üzerinden `/api/Execute/start/{workflowId}` endpointi ile süreçleri tetikleyebilirsiniz.*

### Frontend (Designer) Başlatma
```bash
cd flowlogic-designer
npm install
npm run dev
```
*Uygulama localhost üzerinde çalışır. Sürükle bırak ile Node'ları bağlayabilir, sağ panelden dinamik değişkenleri (Örn: `{{TaskNode_1.MusteriID}}`) enjekte edebilirsiniz.*

---

## 🔮 Vizyon ve Gelecek
Low Code Api Builder'ı sadece bir araç değil, işletmelerin karmaşık iş akışlarını yazılımcı bağımlılığını en aza indirerek çizebileceği, test edebileceği ve ölçekleyebileceği bir altyapı olarak geliştirmeye devam ediyorum. Modern mikroservis orkestrasyonu ve Event-Sourcing altyapılarına tam uyumlu bir platform inşa etmeye çalışıyorum. 

---

## 📋 Yol Haritası (Roadmap)

### ✅ Bitenler (Neler Yaptım)
- [x] **React Flow Tabanlı Görsel Tasarımcı**: Node tabanlı, sürükle-bırak UI arayüzünü kurguladım.
- [x] **DAG Tabanlı Paralel Asenkron Motor**: `Task.WhenAll` kullanarak Wave-based paralel işleme mimarisini inşa ettim.
- [x] **Gelişmiş Düğüm Yapıları**: `Parallel Start`, `Parallel End`, `Condition` (N-kollu If) ve `Response` node'larını tamamladım.
- [x] **VariableService & In-Memory State**: Süreçler arası JSON/payload taşımasını ortadan kaldıran, bellek tabanlı değişken yönetimini devreye aldım.
- [x] **DuckDB Entegrasyonu**: Büyük veri setlerini bellek içi (in-memory) SQL tablolarıyla işleme yeteneği kazandırdım.
- [x] **Dinamik Servis Entegrasyonları**: `TaskNode` (MSSQL), `SqliteNode` (DuckDB), `WebServiceNode` (SOAP/REST) altyapılarını bağladım.
- [x] **Dinamik Connection Routing**: Her düğümde çalışma anında farklı veritabanı veya servis sağlayıcısı seçilebilmesini sağladım.
- [x] **Akıllı StopOnError Mekanizması**: Web servis cevaplarını okuyarak HTTP 200 gelse dahi içindeki kodlara göre hataları yakalayan ResponseModel mimarisini yazdım.
- [x] **Kapsamlı Loglama**: `BpmHistoryService` ile Pipeline ve Job bazlı veritabanı history / loglama sistemini kurdum.

### 🚧 Yapılacak Olanlar (Neler Yapmaya Çalışıyorum)
- [ ] **Modern UI/UX İyileştirmeleri**: Nodeların görsel tasarımlarını (Dify/Didit esintili) daha yenilikçi ve spesifik custom tasarımlarla güncellemeyi planlıyorum.
- [ ] **Sol Panel (Toolbox) Optimizasyonu**: Designer tarafında node menüsünü daha profesyonel ve kategorize edilmiş bir yapıya geçirmeyi hedefliyorum.
- [ ] **Loop (Döngü) Düğümü**: Koleksiyonlar (liste) üzerinde satır satır çalışan Iterator node'unu sisteme dahil etmeye çalışacağım.
- [ ] **Geciktirme (Wait/Timer) Düğümü**: Akışı belirli bir süre veya tarihe kadar bekletebilme özelliğini kodlayacağım.
- [ ] **Retry (Yeniden Dene) & Catch**: Hata alınan node'larda X kere tekrar deneme ve hata durumunda özel alt-akışlara yönlendirme mantığını kurgulamaya çalışıyorum.
- [ ] **Simülasyon (Dry-Run)**: Workflow'u canlıya almadan önce veritabanına yazmadan UI üzerinden simüle etme (Debug) ekranını arayüze eklemeyi planlıyorum.
- [ ] **UI Grid / Form Düğümleri**: Kullanıcıdan ekran vasıtasıyla onay (User Task) bekleyen stateful akış mimarisine altyapı hazırlamaya çalışıyorum.

> **"Kodu değil, işi tasarla."**
