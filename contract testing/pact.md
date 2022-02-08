**pact nedir?** <br>
pact async mesaj ve http request contract'larını test etmeyi sağlayan bir tool.

projemizde kullanmaya karar verdik çünkü:
- integration testlerimizi isolated ortamlarda yapıyoruz
- e2e testlerimizde de az sayıda user path'lerini test ediyoruz ve bu testler görece yavaş testler.
dolayısıyla uygulamalarımızın doğru bir şekilde etkileşim kurduğundan erken aşamalarda emin olamıyoruz.

pact consumer-driven bir test tool'udur. contract'lar consumer tarafında üretilir, ve provider tarafında test edilir. 
yani contract'larla ilişkisi olmayan provider kodlarında istenildiği gibi değişiklik yapılabilir.

pact'in çalışma mantığını açıklayan güzel bir akış şeması:

<img src="https://github.com/cinaraylin/qa-notes/blob/main/contract%20testing/pact-diagram.png" width="600" title="how pact works" alt="how pact works">


1- consumer tarafında mock bir provider servisi ayağa kaldırıyoruz ve bize dönmesini beklediğimiz response'u bu servise bildiriyoruz.<br>
2- her bir testin sonunda interaction'lar contract dökümanına yazılıyor. <br>
3- Tüm testler bittikten sonra oluşan contract dökümanını pact broker'a yüklüyoruz.<br>
4- provider service pact broker'dan bütün contract'ları alıyor ve her bir interaction'ı kendi tarafında gerçekleştirip sonuçları consumer'ın expectation'larıyla 
   karşılaştırıyor. eğer her request'in response'u consumer'ın beklediği response ile uyuşuyorsa, verification testi geçmiş oluyor.<br>
5- provider service'in etkileşim kurduğu diğer sistemler mocklanıyor ve testler isolated olarak koşulmuş oluyor. 
   (bu aşama 4. aşamayla birlikte gerçekleşiyor aslında)<br>
Son olarak da provider api verification sonucunu pact broker'a iletiyor ve broker'dan da testlerin başarılı olup olmadığını takip edebiliyoruz.

consumer ve provider arasındaki interactionlar/contractların bütününe pact deniyor.
pact bir json dosyasıdır ve içinde şu datalar bulunur:
- consumer adı
- provider adı
- interaction listesi
- pact spesification versiyonu
<br><br>

**consumer tarafında kullanılan komutlar (js)**
- new Pact(options): mock server yaratır. istediğin kadar provider yaratabilirsin
- setup(): mock server'ı başlatır ve hazır olana kadar bekler. bir kere başlatılmalı mock server; bu yüzden beforeAll()'da kullanılmalı
- addInteraction(): expectation eklemek için kullanılır. her server'a veya test'e birden fazla expectation eklenebilir. eklenen expetation validate edilir ve başarılıysa pact'e yazılır.
- verify(): her interaction'ın beklendiği gibi gerçekleştiğini verify eder. her test için bir kere çağrılır
- finalize(): interaction'ları pact dosyasına kaydeder. bir kere; afterAll()'da çağrılır.
<br><br>

**versiyonlama**

versiyonlar can i deploy tarafından kullanılıyor.<br>
3 tane versiyon var pact'te: contract versiyonu, pact verification versiyonu ve consumer versiyonu.<br>
pact'in versiyonuyla biz ilgilenmiyoruz, pact kendisi yönetiyor.<br>
consumer ve provider versiyonlarını biz yönetiyoruz. conflict olmaması için bu versiyonlarda commit numaralarının da kullanılması öneriliyor. <br>
comsumer ve provider versiyonları ile bir matrix oluşturuluyor ve deployment yapılacağı zaman bu matrix kontrol ediliyor.


<img src="https://github.com/cinaraylin/qa-notes/blob/main/contract%20testing/pact-versin-matrix.png" width="600" title="version matrix" alt="version matrix">

- consumer deploy edilecekse; deploy edilecek consumer versiyonu ile  proddaki provider'ın versiyonunun 
- provider deploy edilecekse; deploy edilecek provider'ın versiyonu ile prod'daki consumer'ın versiyonunun 

matrix'deki success değerine bakılıyor. success değeri true ise deployment gerçekleştirilebiliyor.<br>
eğer bir uygulama hem provider hem de consumer ise iki versiyonunun da aynı olması gerekiyor; yoksa can i deploy doğru sonucu üretemez.
<br><br>

**Akılda tutulması gereken birkaç notu sıralayalım.**
- her bir test yalnızca bir interaction'ı test etmeli. eğer bir case için birden fazla interaction gerekiyorsa, her bir interaction ayrı bir test olacak şekilde
  bölünmeli.
  pact dökümantasyonundaki bir örnek:
  “create user 123, then log in” =>  “create user 123” + “log in as user 123 (user 123 exists)”

- provider state kavramı: koşulan testin adı denilebilir. bir interaction'ı verify etmek için provider'ın hangi state'de olması gerektiğini söyler. 
  state sayesinde bir endpointin birden fazla durumu için testler yazılabilir.

- contract testler unit test gibi düşünülmeli ve sadece consumer ve provider arasındaki iletişimi test etmek için kullanılmalı.
  it test veya ui test vs için kullanılırsa redundant bir sürü interaction oluşturmuş oluruz ve bu sadece karışıklık sağlar.

- consumerda random data kullanılmamalı. consumer pact'i publish ettiğinde eğer hiçbir data değişmemişse, son verification sonucu korunur. yani bir önceki koşulan
  test pass olmuşsa, provider testinin tekrar koşmasını beklemeden consumer kodu deploy edilebilir. 
  ama eğer interactionlardaki datalar değişmişse, pact contract değişmiş gibi davranır ve yeni contract'ın provider tarafından verify edilmesi gerekir.

- provider consumer'ın beklediğinden daha fazla field dönebilir, bu conract'ı bozmaz.
