## MultiShop - Microservice

### Projede Kullanılacak Mikroservisler
* Katalog
* Yorumlar
* Sepet
* Ödeme
* Google Drive Görseller
* İndirimler
* Kargo
* Kullanıcı İşlemleri
* Admin
* Frontend
* Rapid API

### Mikroservis Nedir?
> Bir uygulamanın küçük servislere ayrılmasıdır. 
> E-ticaret ve bankacılık uygulamalarında sıkça kullanılmaktadır.
>  * Ürün Arama
>  * Ürün Kataloğu
>  * Stok Yönetimi
>  * Alışveriş Sepeti
>   * Ödeme ... örnek mikroservislerdir.

### Mikroservis Neden Kullanılır?
> * Monolitik yapıda teknoloji geliştikçe eklenen katman veya servislerin kontrolü ve bu servislerin projeye entegre edilmesi işlemi karmaşık hale geliyor.
> * Monolitik yapıda örneğin;
>    * Ürün yorumları: Mongo DB
>    * Sepet: Redis
>    * Arama : Elastic Search kullanıldığını düşünelim
>* Monolitik yapıda bu servisleri entegre ettiğimizde tek bir projeye bütün kütüphaneleri eklememiz gerekecekti. Bu da projenin yükünü arttıracak ve yönetimini zor hale getirecektir.
> * Monolitik yapıda tek bir modülde olan hatadan dolayı projenin bütünü deploy edilemeyecek ve hatanın düzeltilmesi beklenecektir.
> * Mikroservis yapısında ise ekipler çoğunlukla birbirini beklemek zorunda değillerdir. Uygulama yatay eksende ölçeklendirilebilir.

## Mongo DB Nedir_?
* Mongo DB NoSQL tabanlı veritabanı programıdır.
* İlişkisel veritabanlarından NoSQL'in en önemli farkı verilerin tablo ve sütunlar ile ilişkili değil de JSON yapıda tutulmasıdır.
* Okuma ve yazma konusunda ilişkisel veritabanlarından çok daha hızlılardır.
* MongoDB'de kayıtlar doküman olarak adlandırılır, ve bu dokümanlar JSON olarak saklanır.

|İlişkisel Veritabanı|| NoSQL Veritabanı  |
|--|--|--|
| Table |-----> |Colection |
| Row|-----> |Document|
| Column|-----> |Field|

* Mongo DB'de migrationlar yoktur.

### MongoDB Entity Oluşturma

#### Category Entity'si
```csharp
using MongoDB.Bson;
using MongoDB.Bson.Serialization.Attributes;

namespace MultiShop.Catalog.Entites
{
    public class Category
    {
        [BsonId]
        [BsonRepresentation(BsonType.ObjectId)]
        public string CategoryId { get; set; }
        public string CategoryName { get; set; }
    }
}
```
#### Product Entity'si
```csharp
using MongoDB.Bson;
using MongoDB.Bson.Serialization.Attributes;

namespace MultiShop.Catalog.Entites
{
    public class Product
    {
        [BsonId]
        [BsonRepresentation(BsonType.ObjectId)]
        public string ProductId { get; set; }
        public string ProductName { get; set; }
        public decimal ProductPrice { get; set; }
        public string ProductImageUrl { get; set; }
        public string ProductDescription { get; set; }
        public string CategoryId { get; set; }

        [BsonIgnore]
        public Category Category { get; set; }
    }
}
```
**[BsonId]
        [BsonRepresentation(BsonType.ObjectId)]**
        Attribute'lerinin kullanılmasının nedeni Primary Key olarak ProductId'yi gösterdiğimizi belli etmektir. MongoDB ObjectId'yi Guid olarak oluşturduğu için ProductId'yi string türünde kullandık.
**[BsonIgnore]**
BsonIgnore Attribute'ünün kullanım nedeni Product entitysi MongoDB belgesine dönüştürülürken Category entity'sinin de dikkate alınıp dönüştürülen Product belgesinde gereksiz yer tutmasını önlemektir.
        


### MongoDB ile Service Oluşturma
#### ProductService
````csharp
using AutoMapper;
using MongoDB.Driver;
using MultiShop.Catalog.Dtos.CategoryDtos;
using MultiShop.Catalog.Dtos.ProductDtos;
using MultiShop.Catalog.Entites;
using MultiShop.Catalog.Settings;

namespace MultiShop.Catalog.Services.ProductServices
{
    public class ProductService : IProductService
    {
        private readonly IMapper _mapper;
        private readonly IMongoCollection<Product> _productCollection;
        public ProductService(IMapper mapper, IDatabaseSettings _databaseSettings)
        {
            var client = new MongoClient(_databaseSettings.ConnectionString);
            var database = client.GetDatabase(_databaseSettings.DatabaseName);
            _productCollection = database.GetCollection<Product>(_databaseSettings.ProductCollectionName);
            _mapper = mapper;
        }
        public async Task CreateProductAsync(CreateProductDto createProductDto)
        {
            var values = _mapper.Map<Product>(createProductDto);
            await _productCollection.InsertOneAsync(values);
        }
        public async Task DeleteProductAsync(string id)
        {
            await _productCollection.DeleteOneAsync(x => x.ProductId == id);
        }
        public async Task<GetByIdProductDto> GetByIdProductAsync(string id)
        {
            var values = await _productCollection.Find<Product>(x => x.ProductId == id).FirstOrDefaultAsync();
            return _mapper.Map<GetByIdProductDto>(values);
        }
        public async Task<List<ResultProductDto>> GettAllProductAsync()
        {
            var values = await _productCollection.Find(x => true).ToListAsync();
            return _mapper.Map<List<ResultProductDto>>(values);
        }
        public async Task UpdateProductAsync(UpdateProductDto updateProductDto)
        {
            var values = _mapper.Map<Product>(updateProductDto);
            await _productCollection.FindOneAndReplaceAsync(x => x.ProductId == updateProductDto.ProductId, values);
        }
    }
}

````

### MongoDB ve AutoMapper Program.cs Configuration
````csharp
using Microsoft.Extensions.Options;
using MultiShop.Catalog.Services.CategoryServices;
using MultiShop.Catalog.Services.ProductDetailDetailServices;
using MultiShop.Catalog.Services.ProductImageServices;
using MultiShop.Catalog.Services.ProductServices;
using MultiShop.Catalog.Settings;
using System.Reflection;

var builder = WebApplication.CreateBuilder(args);

//yazılan servislerin bildirilmesi
builder.Services.AddScoped<ICategoryService,CategoryService>();
builder.Services.AddScoped<IProductService,ProductService>();  
builder.Services.AddScoped<IProductDetailService,ProductDetailService>();
builder.Services.AddScoped<IProductImageService,ProductImageService>();

//Automapper'ın bildirilmesi
builder.Services.AddAutoMapper(Assembly.GetExecutingAssembly());

// Burada da Settings klasöründe oluşturduğumuz DatabaseSettings sınıfının 
// appsettings.json dosyasındaki DatabaseSettings ile bağlantılı olduğunu 
// ve her IDatabaseSettings istendiğinde appsettings.jsondaki Value'ların kullanılacağının bildirildiği configürasyon

builder.Services.Configure<DatabaseSettings>(builder.Configuration.GetSection("DatabaseSettings"));
builder.Services.AddScoped<IDatabaseSettings>(sp =>
{
    return sp.GetRequiredService<IOptions<DatabaseSettings>>().Value;
});

builder.Services.AddControllers();
// Learn more about configuring Swagger/OpenAPI at https://aka.ms/aspnetcore/swashbuckle
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Configure the HTTP request pipeline.
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

app.UseAuthorization();

app.MapControllers();

app.Run();

````

## Onion Architecture Nedir_?


## Docker Nedir_?

>* Docker, uygulamalarınızı hızla derlemenize, test etmenize ve dağıtmanıza imkan tanıyan bir yazılım platformudur.
>* Docker, yazılımları kitaplıklar, sistem araçları, kod ve çalışma zamanı dahil olmak üzere yazılımların çalışması için gerekli her şeyi içeren [container](https://aws.amazon.com/tr/containers/) adlı standartlaştırılmış birimler halinde paketler.
>* Docker'ı kullanarak her ortama hızla uygulama dağıtıp uygulamaları ölçeklendirebilir ve kodunuzun çalışacağından emin olabilirsiniz.
>* Bir [sanal makinenin](https://aws.amazon.com/tr/ec2/) sunucu donanımını sanallaştırmasına (doğrudan yönetme gereksinimini ortadan kaldırma) benzer şekilde container'lar da bir sunucunun işletim sistemini sanallaştırır.
>* Docker her sunucuya yüklenir ve container'ları oluşturmak, başlatmak veya durdurmak için kullanabileceğiniz basit komutlar sağlar.
## Docker Volumes Nedir_?
[Kaynak: # Docker Serisi — Docker Volumes - Ahmet Cokgungordu](https://acokgungordu.medium.com/docker-serisi-docker-volumes-1c509f043f98) 
* Docker Volumes, Docker Container’larındaki verileri saklamamız veya Container’lar arasında veri paylaşmamız gerektiğinde çok kullanışlıdır.

* Docker Volumes çok önemli bir kavramdır. Çünkü Docker Container silindiğinde tüm dosya sistemi de yok edilir. Bu gibi durumlarda verileri bir şekilde saklamak istiyorsak, Docker Volumes kullanmamız gerekiyor.

**Docker Volume Oluşturma ve Kaldırma**

Aşağıdaki komutu kullarak bir Docker Volume oluşturabiliriz.
````
docker volume create volume_name
````
Tüm Volume’ları listelemek için aşağıdaki komutu kullanıyoruz.
````
docker volume ls
````
Bir Docker Volume silmek istersek, aşağıdaki komut ile yaparız.
````
docker volume rm volume_name
````
**Bir Container’a Docker Volume Ekleme**

“-v” veya “--volume” flagi bir Docker Volume eklemek için de kullanılır. Ancak, dizin eşlerken ana bilgisayardaki dizin pathini veriyorduk. Burada Volume adını yazıyoruz.
````
docker run -d -it --name devtest -v volume_name:/var/www nginx:latest
````
**_Not:_**  Varolmayan bir Volume belirtirsek, öncelikle Volume’u oluşturacaktır.

**Dockerfile ile Volume Ekleme**

Aşağıda bir Dockerfile içinde Volume tanımı örneği vardır.
````
FROM alpine  
VOLUME ["/data"]  
ENTRYPOINT ["/bin/sh"] 
````
Bu Dockerfile’a göre “/data” dizini ana makinedeki Volume name ile yaratılan bir dizine eşlenir. Container’a inspect komutu ile bakarak “source” alanında hangi path ile eşlendiği görülebilir.
````
[  
    {  
        "Destination": "/data",  
        "Driver": "local",  
        "Mode": "",  
        "Name": "f9fb8ttb78494939453fc7a09198da3c20406f15272121722632a4fab54w40qc",  
        "Propagation": "",  
        "RW": true,  
        "Source": "/var/lib/docker/volumes/f9fb8ttb78494939453fc7a09198da3c20406f15272121722632a4fab54w40qc/_data",  
        "Type": "volume"  
    }  
]
````
**Docker-Compose ile Volume Kullanımı**

Aşağıdaki örnekte görüleceği gibi bir yml dosyası oluştururuz.
````
version: '2'services:  
  webserver:  
    build: .  
    ports:  
     - "9000:80"  
    volumes:  
     - .:/usr/share/nginx/html
````
Yukarıdaki örnekte YML dosyasının bulunduğu dizini Container için “/usr/share/nginx/html” dizinine eşler.

**_volumes:  
-/var/lib/mysql_**

Yalnız Container’daki dizini tanımlar, ana makinede anonim Volume oluşur.

**_volumes:  
-/opt/data:/var/lib/mysql_**

Ana makinedeki ve Container’daki dizini birlikte tanımlar

**_volumes:  
-./cache:/tmp/cache_**

YML dosyasının bulunduğu dizine göre tanımlar

**_volumes:  
-html-volume:/usr/share/nginx/html_**

İsimlendirilmiş bir volume üzerinden tanımlar

Eğer isimlendirilmiş bir Volume üzerinden bir tanımlama var ise Volume konfigurasyonunda ek olarak yapılmalıdır.
````
version: '2'services:  
  webserver:  
    build: .  
    ports:  
     - "8080:80"  
    volumes:  
     - html-volume:/usr/share/nginx/html
     - volumes: 
      html-volume:
````

**Read-Only Mode**

Bir Docker Volume birden fazla Container’a eklenebilir. Bunun yanı sıra bazı Container’ların bu Volume’a yazmasını, bazılarının yazmamasını isteyebiliriz. Bunu yapmak için Volume bağlarken read-only kullanılacak Container’da “ro” opsiyonunu aşağıdaki şekilde kullanırız.
````
docker run -d -it --name devtest -v volume_name:/var/www:ro nginx:latest
````
Bu komut hem path verirken hem de Docker Volume bağlarken kullanılabilir.

_Volumes ile çalışırken Container silerken Volume’ları da mutlaka yönetin. Yoksa sistemde gereksiz yere sismelere neden olabilir._

**Docker Images Komutu**
* Docker Image'ları listelemek için:
````
docker images
````

## Portainer Nedir_? 

[Kaynak: # Portainer Nedir? | Docker ile Portainer Kurulumu - Dogukan Turan](https://medium.com/devopsturkiye/docker-ile-portainer-kurulumu-ve-portainera-h%C4%B1zl%C4%B1-bak%C4%B1%C5%9F-2fdcf2b31deb) 

* Portainer, Docker veya Docker Swarm Cluster’ımızı yönetmemizi sağlayan bir management UI’dır.
* Docker’ı terminalden kullanmak sizlere işkence gibi geliyorsa, docker’ı bir arayüz ile yönetmek/kullanmak istiyorsanız portainer tamda bu anda yardımınıza koşan tatlımı tatlı bir yardımcı.
## Portainer Kurulumu? 

Volume Oluşturma:
````
docker volume create portainer_data
````
 Portainer İçin Gerekli Kod:
````
docker run -d -p 8000:8000 -p 9000:9000 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce
````
* Sonrasında tarayıcıda **localhost:9000** adresinde portainer arayüzü bizi karşılıyor.

Username: admin
Password: 123456789aA*

Portainer'da Oluşturulan DB: OrderDb
SA Password: 123456aA*
host: 1440

Portainer'da Oluşturulan DB: IdentityDb
SA Password: 123456aA*
host: 1433
