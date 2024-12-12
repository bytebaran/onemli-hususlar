

---

# İçerik:
##  - [1-data normalizasyonu](#1-data-normalizasyonu)
 - [1.1- verinin en sağlam şekilde tutulması](#11--verinin-en-sağlam-şekilde-tutulması)
 - [1.2- verinin gerektiği yerde ile business'a uygun şekilde cachelenip sunulması](#12-verinin-gerektiği-yerde-ile-businessa-uygun-şekilde-cachelenip-sunulması)

##  - [2- event driven sistemler:](#2-event-driven-sistemler)
 - [2.1-sistemlerin parçalanabilmesi gereği](#21-sistemlerin-parçalanabilmesi-gereği)
 - [2.2-idempotent event listenerlar](#22-idempotent-event-listenerlar)
 - [2.3-execution order sorunlarının yönetimi](#23-execution-order-sorunlarının-yönetimi)
 - [2.4-eventin invokelanmasının akışı bozması sorunlarının yönetimi](#24-eventin-invokelanmasının-akışı-bozması-sorunlarının-yönetimi)
 - [2.5-herhangi bir olayı yeni bir sistem ekleyerek implemente edebilme](#25-herhangi-bir-olayı-yeni-bir-sistem-ekleyerek-implemente-edebilme)


## - [3-tek ownership:](#3-tek-ownership)
- [3.1-data normalizasyonu üzerinden bir açıklama](#31-data-normalizasyonu-üzerinden-bir-açıklama)
- [3.2-weak reference](#32-weak-reference)
- [3.3-strong reference (ownership)](#33-strong-reference-ownership)
- [3.3.1-IndexedComponent ve Context](#331-indexedcomponent-ve-context)
- [3.3.2-Data Modelleri](#332-data-modelleri)
- [3.3.3-İçinde her işi çözen, data tutan managerlar](#333-içinde-her-işi-çözen-data-tutan-managerlar)

---




# 1-Data normalizasyonu

"Data normalizasyonu" diyerek aslında sql veri tabanlarındaki normalizasyon kavramını ifade etmek istiyorum.
Burada çok benzer birşey amaçlıyoruz.

## 1.1- Verinin en sağlam şekilde tutulması:

>Context içinde bir bilgiye ulaşmak eğer mümkün ise aynı şeyi ifade eden yeni bir bilgi tanımlanmamalıdır.
>Eğer tanımlanması gerekiyorsa da bu yeni bilgi her zaman orjinalini yansıtmalı ve orjinalinin değişimi halinde anında güncellenmelidir.
>
>Örnek:
>```cs
>public class Board : MonoBehaviour {
>    private Dictionary<Vector2Int, BoardObject> m_BoardObjects;
>    public ReadOnlyDictionary<Vector2Int, BoardObject> BoardObjects => m_BoardObjects.AsReadOnly();
>    
>    public void Add(BoardObject boardObject){
>        m_BoardObjects.Add(boardObject.Cell, boardObject);
>    }
>    
>    public void Remove(Vector2Int cell){
>        m_BoardObjects.Remove(cell);
>    }
>    
>    public BoardObject Get(Vector2Int cell){
>        return m_BoardObjects[cell];
>    }
>}
>
>public class BoardObject : MonoBehaviour {
>    public Vector2Int Cell;
>}
>```
>
>Bu örnekte hem Board'da BoardObject'lerin yerini tutuyoruz hem de BoardObject'te kendi cell'ini tutuyoruz.
>
>Buradaki sorun böyle yanlış birşeye izin veriyor olmamız:
>```cs
>var boardObject = board.Get(new Vector2Int(0,0));
>boardObject.Cell = new Vector2Int(1, 0);
>```
>
>
>
>Board'daki dictionary güncellenmeden BoardObject'in cell'ini değiştirebiliyoruz.
>
>
>Çözüm ise:
>
>A-Get'i sunmamak, ([içinde-her-işi-çözen-data-tutan-managerlar](#333-içinde-her-işi-çözen-data-tutan-managerlar)'a bakabiliriz) (get'i neden sunmalıyız: [sistemlerin-parçalanabilmesi-gereği](#21-sistemlerin-parçalanabilmesi-gereği))
>fazla büyük managerlara neden olabilir
>
>
>B-Get'te objeyi klonlayıp klonunu sunmak veya gette immutable (yani readonly) obje sunmak. ([data-modelleri](#332-data-modelleri)'ne bakabiliriz)
>
>
>C-Boarddaki Add ve Remove'u kaldırmak ve Board'un board objectleri reaktif indexleyip sunması:
>
>    ```cs
>    public class Board : MonoBehaviour {
>        private Dictionary<Vector2Int, BoardObject> m_BoardObjects;
>        public ReadOnlyDictionary<Vector2Int, BoardObject> BoardObjects => m_BoardObjects.AsReadOnly();
>        
>        private void OnEnable(){
>            EventBus<BoardObjectCellChanged>.Subscribe(OnBoardObjectCellChanged);
>            RefreshAll();
>        }
>        private void OnDisable(){
>            EventBus<BoardObjectCellChanged>.Unsubscribe(OnBoardObjectCellChanged);
>        }
>    
>        public BoardObject Get(Vector2Int cell){
>            return m_BoardObjects[cell];
>        }
>        
>        public void OnBoardObjectCellChanged(BoardObjectCellChanged e){
>            OnBoardObjectCellChanged(e.BoardObject);
>        }
>        
>        public void RefreshOne(BoardObject boardObject){
>            if(boardObject.Cell == null){
>               m_BoardObjects.Remove(boardObject.Cell);
>            }
>            else{
>               m_BoardObjects[boardObject.Cell.Value] = boardObject;
>            }
>        }
>        public void RefreshAll(){
>           // bütün board objectleri RefreshOne ile refreshle
>        }
>       
>    }
>    
>    public class BoardObject : MonoBehaviour {
>        private Vector2Int? m_Cell;
>        public Vector2Int? Cell{
>            get => m_Cell;
>            set{
>                m_Cell = value;
>                EventBus<BoardObjectCellChanged>.Invoke(new BoardObjectCellChanged(this));
>            }
>        }
>        private void OnDestroy(){
>            Cell = null;
>        }
>    }
>    ```
>    
>    Bu çözümde BoardObject'in cell'i değiştiğinde Board'a haber veriyoruz ve Board da bu değişikliği kendi içinde güncelliyor.
>    İki yapının senkronizasyonu garanti edilmiş oluyor.





## 1.2-Verinin gerektiği yerde ile business'a uygun şekilde cachelenip sunulması

>Örnek:
>
>```cs
>public interface IItem{
>    public int Id { get; }
>    public ItemType Type { get; }
>} 
>public struct Sword : IItem{
>    public int Id { get; set; }
>    public ItemType Type => ItemType.Sword;
>    public float Durability { get; set; }
>}
>public struct Arrows : IItem{
>    public int Id { get; set; }
>    public ItemType Type => ItemType.Arrows;
>    public int Quantity { get; set; }
>}
>```
>
>Bir developer datasını herhangi bir sebepten ötürü bu şekilde tanımlamaya karar veriyor:
>
>```cs
>List<IItem> items;
>```
>
>Ama ben herhangi bir nedenden dolayı bu şekilde görmek istiyorum diyelim:
>```cs
>List<Sword> swords;
>List<Arrows> arrows;
>```
>
>Böyle bir durumda datanın tutuluş şeklini tartışmak yerine datayı fazladan bir yoldan daha sunmak da mümkün:
>```cs
>public class ItemsByType{
>    private List<Sword> m_Swords;
>    private List<Arrows> m_Arrows;
>
>    public ReadOnlyList<Sword> Swords => m_Swords.AsReadOnly();
>    public ReadOnlyList<Arrows> Arrows => m_Arrows.AsReadOnly();
>    
>    private OnEnable(){
>        EventBus<ItemChanged>.Subscribe(OnItemChanged);
>        // RefreshAll(); 3.3.1'de bahsedeceğim yine
>    }
>    private OnDisable(){
>        EventBus<ItemChanged>.Unsubscribe(OnItemChanged);
>    }
>    
>    public void OnItemChanged(ItemChanged e){
>        RefreshOne(e.Id);
>    }
>    
>    public void RefreshOne(int id){
>        //get the item by id 
>        //check if it exists 
>        //update the m_Swords and m_Arrows for this item, remove or add
>    }
>}
>```
>
>Bu şekilde herkes mutlu olabiliyor.
>
>
>Bunun iyi bir örneği mesela:
>Bizim son market oyunundaki Conveyor, ConveyorNextItems, AllConveyorItemsAspect.
>
>Sort booster AllConveyorItemsAspect üzerinden itemleri sortluyor mesela.
>```
>Conveyor:
>    Conveyor üzerindeki item gameobjectleri sunan yapı
>ConveyorNextItems: 
>    Gameobjesi olmayan sıradaki itemleri tutan yapı
>AllConveyorItemsAspect:
>    Conveyor + ConveyorNextItems'taki itemleri birbirlerinden ayırt etmeyen bir arayüzle sunan yapı,
>    Herhangi bir indexteki itemı sorgulayıp değiştirebiliyoruz.
>    Indexe göre ya Conveyora ya da ConveyorNextItems'a gidiyor. 
>```


---




# 2-Event driven sistemler:

"Sistem" diye adlandırdığım classları bu şekilde tanımlamak istiyorum:
Aplikasyonun datasındaki belli bir koşul sağlandığında belli bir aksiyonu gerçekleştiren classlar.

Koşulu kontrol etmenin nerede yapılacağı ise şunlar gibi olabilir:
- Update'te sürekli kontrol edilmesi

- Belli eventlerde kontrol edilmesi (reaktif sistem oluyor o zaman)

```
Reaktif Sistem:
Event(ler) -> Kondisyon kontrolü -> Aksiyon
```

Örnek:


```
Market oyunundaki SortItemsInBoxSystem:
    Event: 
      Item state değişimi
    Kondisyon:
      Item'ın içinde bulunduğu kutudaki itemların dizilimlerinin arada boşluklu olması (bu boşluk istenmiyor)
      &&
      Boxun içindeki itemların tween halinde olmaması
      &&
      Item'ın içinde bulunduğu kutunun tween halinde olmaması
    Aksiyon: 
      Item'ları sırala (Idempotent)


market'teki ConveyorItemMovementSystem:
    Event: 
        Update
    Kondisyon: 
        yok
    Aksiyon: 
        Itemları'ı conveyor üzerinde ilerlet (Idempotent değil)
    
  
fourx'teki BuildingUnlockModel:
    Event: 
        Runner level değişimi, HQ Building değişimi
    Kondisyon: 
        Şu anki Runner level'da ve Hq leveldeki açılmış olması gereken binalar unlock olmuş mu
    Aksiyon: 
        Unlock olmamış binaları unlock et (Idempotent)

    
fourx'teki BuildingCellIndexModel:
    Event: 
        Building data değişimi
    Kondisyon: 
        yok
    Aksiyon: 
        Building cell -> Building index mapping'deki ilgili buildingi güncelle (Idempotent)
```


## 2.1-Sistemlerin parçalanabilmesi gereği:
>Eğer managerları/sistemleri parçalamak için iyi yöntemlerimiz olmazsa parçalaması zor büyük managerlar oluşur. (nedenini burada açıklıyorum: [İçinde her işi çözen, data tutan managerlar](#333-içinde-her-işi-çözen-data-tutan-managerlar))
> 
>Yukardaki örnek sistemlere bakarsanız sistemlerin amaçlarının net olduğunu görebilirsiniz.
> 
>Bir iş yapılması için bilinmesi gereken projeye özel kod miktarı oldukça azdır.
> 
>İşlerin bu şekilde küçük parçalabilmesi ve net ifade edilebilmesi developer'ın işini kolaylaştırır, kodun okunabilirliğini arttırır.


## 2.2-Idempotent event listenerlar:
>Idempotent: Birden fazla tekrarlamada sonucu değişmeyen aksiyonlar.
> 
>Sistemlerin aksiyonlarını iyi bir neden yoksa Idempotent implement etmeyi öneriyorum.
> 
>Idempotent listenerların birden fazla tekrarlanması sorun olmadığı için sistemlerin tetiklendiği eventleri istediğimiz kadar çoğaltabiliyoruz, hatta update'e bağlayabiliyoruz. Fazla refresh'lemenin performans dışında bir zararı olmuyor.
>
>    Idempotent event listenarların en yaygın örneği index modelları dediğim modellerdir, genel kalıp budur:
>    ```cs
>    public class ABCIndexModel : MonoBehaviour // (monobehaviour olmak zorunda değil)
>    {
>        /*
>        ...
>        cache fieldlar
>        ...
>        */
>        
>       /*
>       ...
>       sorgu yapmak için public metodlar, GetABCAtCell, GetABCByType gibi
>       ...
>       */
>       
> 
>        private void OnEnable(){
>            EventBus<ABCChanged>.Subscribe(OnABCChanged,listenOrder: -100);
>            RefreshAll();
>        }
>        
>        private void OnDisable(){
>            EventBus<ABCChanged>.Unsubscribe(OnABCChanged);
>        }
>        
>        private void OnABCChanged(ABCChanged e){
>            RefreshOne(e.ABC);
>        }
>        
>        private void RefreshOne(ABC abc){
>            // abc ile ilgili cache fieldlarda güncellenecek veriyi güncelle
>        }
>        
>        private void RefreshAll(){
>            // bütün abc'leri RefreshOne ile refreshle
>        }
>    }
>    ```




## 2.3-Execution order sorunlarının yönetimi

>```
>market oyunundan bir örnek:    
>    Box Data:
>        IsOverlapped: bool
>        
>    MainAreaIndex(main areada'ki boxların grid placementlarını indexleyen yapı):
>        Event: box state değişimi
>        Condition: yok
>        Action: ilgili box'un yerini kendi dictionarinde güncelle
>        
>    BoardObjectOverlappedSystem:
>        Event: box state değişimi
>        Condition: yok
>        Action: MainAreaIndex'e bak ve değişen boxun etkilediği alandaki kutuların IsOverlapped'larını güncelle
>            
>    Buradaki riskli durum şu:
>        MainAreaIndex box'ları dinliyor
>        BoardObjectOverlappedSystem hem box'ları dinliyor hem BoardObjectOverlappedSystem'a erişiyor
>```
>``` 
>Sorun:
>    Eğer BoardObjectOverlappedSystem box'un eventine ilk subscribe olursa MainAreaIndex daha kendini güncelleyemeden MainAreaIndex'e sorgu atmış olur ve yanlış cevabı alır.
>```    
>```
>Çözüm:
>    A-BoardObjectOverlappedSystem boxun eventini değil MainAreaIndex'in eventini dinlesin
>        
>    B-Explicit event listen orderla subscribe olma. (markette bunu uyguladım)
>        Kullandığımız event bus böyle birşey destekliyor:
>            MainAreaIndex:
>                EventBus<BoxStateChanged>.Subscribe(listenOrder: -100)
>            BoardObjectOverlappedSystem:
>                EventBus<BoxStateChanged>.Subscribe(listenOrder: -95)
>        
>        Bunu yapınca BoardObjectOverlappedSystem'ın her zaman MainAreaIndex'in en güncel halini göreceğini garantilemiş oluyoruz.
>            
>        Not: Indexleme yapan modellerinin her zaman negatif listen order kullanmasını tercih edelim. 
>        Çünkü bu modelleri kullanan yerler datanın en güncel halini görmek isteyecektir.
>```


## 2.4-Eventin invokelanmasının akışı bozması sorunlarının yönetimi
>```
> Örnek bir sorun:
>     Diyelim yine bloklu bir oyun var
>     Bir de BlockSpawner var
>     BlockSpawner önü boş oldukça Block spawn etsin.
>     
> BlockSpawnerSystem:
>     Event: CellChanged
>     Condition: önü boş olan BlokSpawner var mı?
>     Action: önü boş olan BlockSpawner'ın önüne Block spawn et
> 
>             
> Bir de oyunda save - load edebildiğimizi düşünelim.
>      
> Senaryo:
>     1- Boardda bir tane BlockSpawner var, önünde de bir tane Block var.
>     2- Save edelim
>     3- Clearlayalım
>     4- Load edelim
>         Load edilme sırasında olacak olan olası birşey şu:
>             -Önce BoxSpawner save datasından oluşturulur
>                 -CellChanged eventi tetiklenir
>                 -BlockSpawnerSystem tetiklenir
>                 -BlockSpawnerSystem, BlockSpawner'ın önünde Block göremez, kondisyon sağlanır ve aksiyonu tetikler. 
>                  Aslında oluşmaması gereken bir box oluşur.
>             -Sonra Box save datasından oluşturulur ama bunu boarda koyduğumuz anda collision olur çünkü o cellde zaten bir blok var.
>     5- Save ettiğimiz ana doğru bir şekilde dönememiş olduk, şuan buglu bir statedeyiz.
>             
> Korkunç bir durum gibi geliyor, işleri eventlere ayırıp parçalayıp görememekten korkmamızın sebebi böyle olaylar.
> Ama çözümü aslında basit. Böyle şeyler için EventBus'ta alternatif bir listener var.
>         
> Buradaki temel hata BlockSpawnerSystem'in başka sistemlerin akışını bölerek girip iş yapmaya çalışması.
> Business yapan, state'leri değiştiren sistemlerin derin event dinleme stacklerinde iş yapmaması gerekir.
> Böyle bir sistemin herkesin işi bittikten sonra sağlıklı bir anda devreye girip kondisyonu kontrol edip aksiyonunu çalıştırması gerekir.
> Bunu için EventBus.SubscribeDeferred var.
> EventBus.SubscribeDeferred:
>     Normal subscribe gibi ama listenerı direk invokelamıyor, bufferlanıyor.
>     Unity Updateinde tetiklenen bir ApplyDeferred var. Bufferlanan eventler burada tetikleniyor.
> 
>             
>         
> Bu senaryodaki hataya neden olan kullanım:
>    BlockSpawnerSystem.OnEnable içinde:
>        EventBus<CellChanged>.Subscribe(OnCellChanged);
> 
> Doğrusu:
>    BlockSpawnerSystem.OnEnable içinde:
>        EventBus<CellChanged>.SubscribeDeferred(OnCellChanged);
> 
> 
> Sadece buna dikkat etmemiz bunun gibi hatalardan bizi koruyor.
> 
> Özet:
>     Business yapan, data setleyen sistemler EventBus.SubscribeDeferred yapmalı, akışı bölüp girmemeli.
>     Index sunan yapılar ise değişime anında tepki vermeli ver herkesten önce çalışmalı. EventBus.Subscribe(listenOrder:-100) gibi birşey yapılabilir.
>     Index sunan yapıların birbirlerinden beslendiği durumlar varsa listenOrder'ları özel ayarlanmalı.
>```




## 2.5-Herhangi bir olayı yeni bir sistem ekleyerek implemente edebilme:
>Sistemlerin dinledikleri eventler ve ilgilendikleri kondisyonları basit tutuyoruz.
> 
>Herhangi bir koşulu ilgili eventleriyle birlikte kontrol etmek çok zor olmayacaktır.
> 
>Yeni featureları aynı koşula kadar inmiş ama implemente edeceğimiz olayla alakası olmayan başka bir sistem bulup ona yedirmeye çalışmayalım.
> 
>Bir listener ve bir check daha eklemenin performansa etkini tartışmayalım burada. 
















# 3-Tek ownership:

Bir değişkenin aynı anda birden fazla sahibi olmamalı.


## 3.1-Data normalizasyonu üzerinden bir açıklama:
>Ownership bir ağaç gibi düşünülebilir. fourx'ten örnek verelim:
>```
>GameContext
>|
>|_BuildingsModel
>|  |_Buildings
>|    |_Barracks
>|    | |_Training
>|    |   |_Soldiers
>|    |_Hospital
>|      |_Healing
>|        |_Soldiers    
>|_ResourcesModel
>|  |_BasicResources
>|  |_Soldiers
>|_WorldSquadsModel
>|_Squads
>|_Heroes
>|_Soldiers
>```
>
>Bu ağaçtaki bütün ilişkiler ownership ilişkileri
>
>Ownership olmayan bir ilişki örneği için BuildingCellIndexModel'i verebilirim:
>```
>    BuildingCellIndexModel:
>        Dictionary<Vector2Int, Building> buildings;
>    OnEnable:
>        subscribe to EventBus<BuildingChanged>, update buildings dictionary
>```
>
>Bu modelde Dictionary'deki Building referansları bir ownership değildir.
> 
>Building burada aslında sadece id olarak kullanılmıştır.
> 
>(Rust'ta burada obje referansı kullanmak mümkün olmazdı ve id kullanmak zorunda kalırdık. C#'ta böyle kolay olduğu için obje referansı kullanıyoruz ve dikkat ediyoruz.)
>





## 3.2-Weak reference
>Ownership olmayan bütün referanslara weak referans diyebiliriz.
>Weak referanslara mümkün olduğunca güvenmemek ve kontrol etmek gerekir.
>Objenin lifecycleını yöneten yer objenin ownerıdır. Owner'ı istediği gibi objeyi yok edebilir.
>Obje poollanmış bir objeyse ownerı poola koyup geri çıkarabilir.
>Bunun gibi durumları weak refarans kullanacak taraf takip etmelidir.
>

>### Weak referance örnekleri:
>>### PooledObjRef:
>>Bir obje poola geri döndüğünde otomatik invalide olan bir weak referans.
>>```cs
>>public struct PooledObjRef<T>{
>>    public T? Get(){...}
>>}
>>```
>>Get null dönerse obje poola en az bir kere dönmüş demektir.
>
>>### Closure allocationla gelen referanslar:
>> Closure'dan gelen referansların check edilerek kullanılması
>> ```
>>  Örnek:
>>      Pooldan bir obje çekelim
>>      Objeye tween ile bir hareket yaptıralım
>>      Tween bittiğinde objenin üzerinde birşey yapalım.
>>      Bu obje sistemler tarafından ortak yönetilen bir obje olsun. (objenin ownerı ana context yani)
>>  
>>  Sorun:    
>>      Biz bu tweeni başlatırken tweene aslında objenin referansını vermiş oluyoruz ve objeyi o tweenin yönetimine bırakmış oluyoruz.
>>      Obje aslında context'e ait olduğu için ve sistemler context üzerinde sürekli işlem yaptığı için objenin tween esnasında bizden bağımsız başına herşey gelebilir.
>>      Closure içinde kaydettiğimiz değişkenler güvenilir değildir.
>>  
>>  Çözüm A:
>>      Objenin tweeni bittiğinde işlemi yapmadan önce objenin hala valid olup olmadığını kontrol etmemiz gerekir.
>>  Çözüm B:
>>      Objeyi tweene sokmadan önce objenin contextten çıkarırız ve objenin tek referansı tweende kalmış olur. 
>>      Bu durumda tweendeki objenin referansına ownership diyebiliriz ve tween objeye ne isterse yapabilir, hiç bir sistem de karışmayacaktır.
>> ```
> 
>>###    Delayli çağrılan callbackler ve coroutinelere de benzer örnekler bulunabilir.



## 3.3-Strong reference (ownership)

Data yönetimi yaptığımız, değişkene gerçekten sahip olduğumuz yerler.
Buraya erişimi olan sistemler değişkenin lifecycleını yönetebilir, ne isterse yapabilir.

Kullandığımız ownership yönetim biçimleri:

#### 3.3.1-IndexedComponent ve Context:
>ECS'e benzeyen monobehaviour componentları indeksleyen yapı.
> 
>Yaptığım şey aslında unity'deki FindObjectOfType<>'ın optimize bir halini oluşturmak.
> 
>Bir MonoBehaviour'a IIndexedComponent interface + CoreEntity componenti eklediğimiz anda ilgili MonoBehaviour bütün sistemlere context.Components.GetAll<T> üzerinden erişilir oluyor.
> 
>Objenin sahibi context oluyor. Ve context'e erişimi olan herkes bu objeyi yönetebiliyor.
> 
> Market oyununda sadece bunu kullandık

#### 3.3.2-Data Modelleri:
>düz struct data:
> 
>>fourx'teki bütün data modelleri örnek verilebilir.
>>Data objeleri immutable tanımlanıyor ve bütün mutasyonlar ana Model üzerindeki public Set'ten yapılıyor.
>>Bu şekilde bütün değişimleri takip edip gerekli eventleri atabiliyoruz.
> 
>gameobject ile data:
>>Bir data modelinde gameobjectler private bir şekilde yönetilir.
>>Dışarıya ilgili gameobject readonly bir şekilde sunulur.
>>Bütün set işlemleri model üzerinden yapılır.
>>tercih ettiğim bir yöntem değil ama duruma göre makul olabilir.


#### 3.3.3-İçinde her işi çözen, data tutan managerlar:

>Örnek:
>
>```
>PhotonConnectionManager:
>    ConnectRoom
>    DisconnectRoom
>    OnPacketReceived
>    OnPlayerJoined
>    OnPlayerLeft
>    etc etc
>```
>
>Tamamen kendini ilgilen içinde state tutan, rahat bir public arayüz sunan bir yapı.
>Bunun public arayüzü dışında içindeki verilere erişmemizin hiç bir avantajı yok.
>Standart bir OOP örneği.
> 
>
>Ama bunu game logic yürüten managerlar için uygulamaya çalışırsak çok büyük ve bölünemeyen bir God Object oluşmasına neden olabiliriz.
>God object managerlar bence bizim şirkette yaygın oluşan bir durum ve bu şekilde meydana geliyor.
>
>Manager komple oyunun managerı olsun diyorsak da o manager içinde bir parçalama yapılması lazım.
>İçerde yine bahsettiğim sistemlerin kurulması ve işlerin parça parça yönetilmesi lazım.
>
>
>MissionManager, TutorialManager felan olsa belki kendi state'ini içinde private yönetebilir ama yine de çıkmaza girmesi olası geliyor bana.
>
>Managerların birbirleri üzerinden ilgili private datalara erişimleri olmadığı için çok spesifik, birbirleri için tanımlanan özel methodlar çağırmaya başlaması bence çok kötü birşey.
>Bunu gördüğümüz zaman datayı ayırıp public yapmalıyız bence. Datayı behaviourdan ayırmak spagetti kodu direk çözüyor dependency ilişkilerini ağaç haline getiriyor.
