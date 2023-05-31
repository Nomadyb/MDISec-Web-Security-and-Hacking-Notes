<h1 align="center">OAuth Zafiyetleri</h1>

## Lab: Stealing OAuth access tokens via an open redirect
- OAuth servisi veren providerlar validationlar yapıyor. Burası çok karmaşık
    - Burdaki URL parser logic bugs konusu buraya girer.
- Belki sadece domain validation yapıyordur. saçma olur bu           .com.hacker.net
- Open-redirection zafiyeti aramaktayız. Ürünlerde “next product” gibi bir şey varsa ve bu da redirection yapıyorsa VE path parametresini de gidip olduğu gibi Response içine yazıyorsa burda open redirection zafiyeti vardır.
![image](https://github.com/grealyve/MDISec-Web-Security-and-Hacking-Notes/assets/41903311/4640f82a-3c08-4356-8706-5bcf701c66ce)
- Giriş yapma requesti bu olacak ama olmadı çünkü path e kadar validation yapmış adamlar.
![image](https://github.com/grealyve/MDISec-Web-Security-and-Hacking-Notes/assets/41903311/36e51083-3756-4062-9a36-eb902e5f019e)
- /oauth-callback e kadar validation yapmışlar bundan sonrasını kontrol etmiyor. İster yeni parametre ekle path ekle buraya
```python3
https://acOf1f8c1eOae093802e1c3b0075006b.web-security-academy.net/oauth-callback/../../../../post/next?path=http://hacker.com
```
### Bu üst pathe çıkmayı gören server side kısmında(nginx) üst pathe çıkarır ve  “oauth-callback/../../” şu kısmı silerek algılar. Client side’da browser kontrol ediyorsa zaten “oauth-callback/../../” e gitmiş oluruz.
- Doğrudan [hacker.com](http://hacker.com) a gitti yani sunucu tarafına gitmeden Browserdaki bir davranış oluyor.
- Bu istek ile gitti giriş yaptı falan ama olmadı
![image](https://github.com/grealyve/MDISec-Web-Security-and-Hacking-Notes/assets/41903311/fa283eec-af5d-492b-a4fe-3db7976917ed)
- Bu adam tokenı location hash’e set ederek yolluyor redirecti. Bu browserda kalır sunucuya ve loglara gelmez. JS kodu ile tokenı dızlyabiliriz. 
![image](https://github.com/grealyve/MDISec-Web-Security-and-Hacking-Notes/assets/41903311/c53bcfcd-5c75-43f7-b649-7289098dec49)
- iframe ihtiyacı çıktı şimdi. Ama bu iframe içine erişemezsin o yüzden redirectiona ihtiyacımız var. Yapı gereği event message ile data dönüyorsa belki datayı leak edebilirsin
1) MDI kendi web sitesine bir içerik koyuyor, bu içerik iframe ile service provider’a yönlendirme gerçekleştiriyor.
2) Ama bu sayfada bir script var ki sayfadaki location hash bilgisini alıp tekrar kendisine request gönderen ve bu sayede access loguna yazabilme imkanı sağlıyor.
3) MDI web sitesinin linkini kurbana yolluyor ve service provider sayfasını iframe ile açıyor.
4) Provider’da halihazıra giriş yapmış olan kurban, ben seni geri yönlendiriyorum diyor.
5) Sayfadaki open-redirect zaafiyetini kullanarak kendi web sitemize geliyor ve kendi sitemizde(exploit server) adamın tokenını dızlamış oluyoruz.
![image](https://github.com/grealyve/MDISec-Web-Security-and-Hacking-Notes/assets/41903311/0b9ec683-a84a-4d0b-af4b-65f061a241a3)
![image](https://github.com/grealyve/MDISec-Web-Security-and-Hacking-Notes/assets/41903311/03ba0f49-7b5c-44a0-aec4-5abde332abe8)
- Sonra çalıntı token ile gidip giriş yaparken /me isteğinde Bearer ile taşıyor. Gelen response’da API key var.
## Lab: Stealing OAuth access tokens via a proxy page
- Giriş yapma isteğinde yine redirect_uri kısmının en sonunu değiştirebiliyoruz eklemeler yapılır.
- Loginden bir şey çıkmadı gibi ama, bir ürün sitesine gittiğimizde aşağıda yorum falan yapabildiğin için comment-form sitesi geldi böyle.
- Bu sayfa yüklendiğinde POST message alıyor ve data olarak windows.location.href basıyor yani URIdan geliyor .
  - Bu sayfa için iframe açarım ve postmessage listener yazarım
![image](https://github.com/grealyve/MDISec-Web-Security-and-Hacking-Notes/assets/41903311/5be1c6f5-8413-43df-8474-44358101e33b)
- En son tokenı nasıl dönüyor onu anlamak çokomelli. En son iş bittiğinde location ile bir URI a redirection yapıyor
![image](https://github.com/grealyve/MDISec-Web-Security-and-Hacking-Notes/assets/41903311/8295fd37-80b0-4e0d-aa7a-849f02127f6c)
- Şu tokenı çalarsa yerine geçerim diye düşünüyor, bunun cevabı olarak da redirection gelmiş
![image](https://github.com/grealyve/MDISec-Web-Security-and-Hacking-Notes/assets/41903311/ad15d7d0-50ba-4e2f-ba26-dca8154a986a)
- yine sonrasında Bearerdan token ile gelmiş istek ve API key falan dönüyor buda.
- Tüm redirectionların sonucunda yukardaki postMessage olayına getirmek gerekiyor. (postMessage xss poc)
  - Fetch requesti gönderirken location hashden ötürü browserda kalıyor token kayboluyor.
  - Olay yukarıdakiyle aynı ama JS kodları uğraştırdı. Form kısmı??
- comment-form endpointinde data: “window.location.href” event cevabı döndüğünü görünce bu tokendan gelen şeyi alabiliriz diye düşündü.
- Hemen burdan yürüyüp iframe oluşturdu ve üst satıra çıkıp bu iframeler arası iletişim kurmak için
  - Bir sayfayı iframe ile açtığında “postMessage” ile o iframe ile iletişim kurabiliyorsunuz. Ve bu adam comment-form endpointi de kendisini iframe içinde açanlara hrefi gönderiyor. Bu satır olmasa bi bok yapılamazdı.
  - ![image](https://github.com/grealyve/MDISec-Web-Security-and-Hacking-Notes/assets/41903311/b46efbda-b269-495c-abd5-af2a80cf9cb3)
  - Gidip birisine bu sayfayı iframe ile açtırtıyoruz, açan kişi de zaten authentication providerda aktif bir sessionu olduğu için comment-form endpointine gelecek. Ama bu comment-form iframe’e postMessage gönderdiği için eventListener yazdık.
![image](https://github.com/grealyve/MDISec-Web-Security-and-Hacking-Notes/assets/41903311/5c909d3e-5bae-4430-b465-ca4835ac57af)
- Tokenı dızladıktan sonra /me den yine API keyi aldık.
- Şu kodda # dan sonrası sunucuya gitmez 😀
![image](https://github.com/grealyve/MDISec-Web-Security-and-Hacking-Notes/assets/41903311/923a846d-d0cd-482e-b150-c766eb39f705)
