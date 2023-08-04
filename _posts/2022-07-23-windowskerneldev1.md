---
layout: post
title: Windows Kernel Development 1 [TR]
date: 2022-07-23 11:58:20 +0300
description: Zararlı yazılımlara hükmetme yolunda önemli bir adım... Windows üzerinde kernel geliştiriyoruz
tags: [Kernel, Windows, TR]
---

“Never memorize something that you can look up.”
-- Albert Einstein


Bu serinin ilk yazısının, devamında yayınlayacağım projelerin temel taşı olacak Windows Kernel hakkında 101 eğitimi niteliğinde bir yazı olmasını hedefliyorum.

$$E=mc^2$$

## Kernel Nedir?
Tabii önce kendisinin ne olduğunu söylemek icap eder... Kernel (işletim sistemi çekirdeği), işletim sistemi üzerinde her şey üzerinde denetimi bulunan merkezi bir bileşendir.
Kısaca: uygulamalar ve donanımlar arasındaki köprü görevini görür.

## Neden Kernel Geliştiriyoruz?
Bir zararlı yazılım, Virtual Machine ("Sanal Makine" kendisi bundan sonra VM olarak çağırılacaktır) üzerinde çalıştırılıp çalıştırılmadığını bilmek ister. Üstelik günümüzde zararlı yazılım geliştiricilerinin sıkça tercih ettiği packer ve karmaşıklaştırıcılar (bkz. Themida, VMProtect) antivm özelliğini kendilerinde bulunduruyorlar yani geliştiricinin ayriyeten yazmasına gerek kalmıyor.
![github.com/Back-X](https://camo.githubusercontent.com/c69f002bb0b7e7fb319f2ac6fd2b7a8e6b84abe7268391023d51989dfa6f7c82/68747470733a2f2f7777772e6879627269642d616e616c797369732e636f6d2f66696c652d696e6c696e652f3630363663616662383961626139316132313135633134622f73637265656e73686f742f73637265656e5f312e706e67)
Bu yazının asıl gündemi başlıktan da anlaşılacağı üzere kernel geliştirme hakkında temel, öz bilgiler. Buradan da SSDT Hook kısmına yürüyüp; VM içerisinde kullanabileceğimiz, zararlının AntiVM sırasında kullandığı sorguları otomatik atlatan bir kernel...

## Kernel ile Hello World
Öncelikle Visual Studio 22, WDK ve SDK kurulumlarını geçiyorum bunları da yapmadıysanız sadece Windows Docs açmanız yeterli kurulum esnasında yaşanabilecek her türlü problemin çözümü internette var ("Spectre Mitigated, KMDF Projesi Visual Studio'ya gelmiyor..."). 
Tamam, şimdi başlayalım. Visual Studio üzerinde "KMDF Driver Empty" projesini açtıktan sonra "Source Files" klasörü altında kernel.c adında bir dosya açıyorum ve başlıyorum içine yazmaya...

### Temel Fonksiyonlar

User-Mode üzerinde yazdığımız herhangi basit bir uygulamanın ana yapısı ile karşılaştırarak gidersek anlaşılmayacak bir durum kalmayacak sanıyorum bu yüzden böyle yapacağım.
Kernel üzerinde ilk olarak tanımamız gereken 2 fonksiyonumuz var.

1. Fonksiyonumuz DriverEntry. Bu fonksiyon user-mode üzerinde yazdığımız Main fonksiyonu ile birebir. Kernel dosyasının EntryPoint'i yani giriş noktası. Kernel yüklendiği zaman önce bu fonksiyon çalışacak.

{% highlight cpp %}
NTSTATUS DriverEntry(_In_ PDRIVER_OBJECT DriverObject, _In_ PUNICODE_STRING RegistryPath)
{ 
    UNREFERENCED_PARAMETER(DriverObject);
    UNREFERENCED_PARAMETER(RegistryPath);

    DriverObject->DriverUnload = SampleUnload; //Ornek Unload kullanimi
    DbgPrint(("Sample driver initialized successfully\n"));
    return STATUS_SUCCESS;
}
{% endhighlight %}

2. opsiyonel fakat bu yazıda tanımlamasını yapacağız örnek kullanımı şu şekilde:

{% highlight cpp %}
void SampleUnload(_In_ PDRIVER_OBJECT DriverObject) {
    UNREFERENCED_PARAMETER(DriverObject);
    DbgPrint(("Sample driver Unload called\n"));
}
{% endhighlight %}

- Fonksiyon NTSTATUS ile tanımlanmış yani bittiğinde bu türden bir veri döndürecek o halde ilk açıklamamız gereken yerden başlayalım `NTSTATUS`, adı üstünde durum hakkında durum hakkında bilgi. [WDK GitHub](https://github.com/tpn/winsdk-10/blob/master/Include/10.0.14393.0/shared/ntstatus.h) hesabından ntstatus.h dosyasını inceleyecek olursanız onlarca durum kodunun tanımlanmış olduğunu göreceğiz işte bunlardan bazıları:
![Screenshot_1](https://user-images.githubusercontent.com/65495573/180623025-c404f8e5-8d08-4786-9ca5-9babc3053427.png)

- Fonksiyona parametre olarak verdiğimiz `_In_` türünde olan veriler Source (Code) Annotation Language (SAL)'ın bir parçasıdır. statik analiz ve kodu inceleyen insanlar için kullanışlı meta veriler sunar.
- DriverEntry fonksiyonuna iki argüman kabul edildiğini gördük ve bunlardan birincisi diğer fonksiyonda da gördüğümüz DriverObject. Bu çekirdek sürücümüz için önemli çünkü aslında onun bir görüntüsünü temsil ediyor. I/O yöneticisi bir driver'ın DriverEntry rutinini çağırdığında, driver'ın sürücü nesnesinin (yani DriverObject) adresini sağlar. DriverObject, bir kernel driver'ın standart rutinlerinin çoğunun EntryPoint yani giriş noktalarını depolamakla mükelleftir. Üstelik Unload'da yaptığımız gibi standart rutinlerin doldurulmasından da yazdığımız sürücü sorumlu tutuluyor. Aşağıdaki döküm MSDN sayfasından alıntı ve bu dökümde neleri tanımlayabileceğimizi açıkça görebiliyoruz.

{% highlight cpp %}
  typedef struct _DRIVER_OBJECT {
  CSHORT             Type;
  CSHORT             Size;
  PDEVICE_OBJECT     DeviceObject;
  ULONG              Flags;
  PVOID              DriverStart;
  ULONG              DriverSize;
  PVOID              DriverSection;
  PDRIVER_EXTENSION  DriverExtension;
  UNICODE_STRING     DriverName;
  PUNICODE_STRING    HardwareDatabase;
  PFAST_IO_DISPATCH  FastIoDispatch;
  PDRIVER_INITIALIZE DriverInit;
  PDRIVER_STARTIO    DriverStartIo;
  PDRIVER_UNLOAD     DriverUnload;
  PDRIVER_DISPATCH   MajorFunction[IRP_MJ_MAXIMUM_FUNCTION + 1];
} DRIVER_OBJECT, *PDRIVER_OBJECT;
{% endhighlight %}

![MSDN](https://docs.microsoft.com/en-us/windows-hardware/drivers/kernel/images/24drvobj.png)

- UNREFERENCED_PARAMETER() fonksiyonu compiler derleme sırasında hata çıkartmasın diye koyuluyor. Çünkü içerde argümanı kullanmayınca uyarı çıkartıyor bak dikkat et bunu kullanmadın diye. ("the following warning is treated as an error")

Tüm kodu şimdi derleyelim ve sanal makinede yüklemeye hazırlayalım. Sanal makine olarak Windows10 kullanacağım VMWare üzerinde...

{% highlight cpp %}
#include <ntddk.h>

void DriverUnload(_In_ PDRIVER_OBJECT DriverObject) {

	UNREFERENCED_PARAMETER(DriverObject);

	DbgPrint("Driver ayrildi\n");
}

NTSTATUS DriverEntry(_In_ PDRIVER_OBJECT DriverObject, _In_ PUNICODE_STRING RegistryPath) {
	
	UNREFERENCED_PARAMETER(RegistryPath);

	DriverObject->DriverUnload = DriverUnload;

	DbgPrint("Driver yuklendi!\n");

	return STATUS_SUCCESS;
}
{% endhighlight %}

Kodu derleyip `repos\kernel1\x64\` yolunda bulunan .sys uzantılı dosyayı, OSR Loader programını ve DebugView (Debug çıktısını görmek için) programlarını sanal makineme atıyorum.

![Screenshot_2](https://user-images.githubusercontent.com/65495573/180648106-c8c8e2a9-ca1d-41e8-a24b-0517b5607fd7.png)

Şimdi esasında biz cmd üzerinde komutlar ile de driver'ı sisteme kaydedip çalıştırabiliriz fakat daha kolay olması açısından OSR Loader kullanıyoruz. Browse butonu ile .sys uzantılı derlenen dosyayı seçip "Register Service" butonuna basıyoruz. Sonra da çalıştırıp çıktıları almak için Start Service butonuna basıyoruz.

![Screenshot_3](https://user-images.githubusercontent.com/65495573/180648185-f0406ba2-03cb-4a42-b332-2a14ef40a231.png)

Bu şekilde bir hata gelecek. Komut satırını yönetici modda açıp `bcdedit /set testsigning on` komutunu girip bilgisayarı yeniden başlatmamız gerekli bu işlemi başarılı şekilde yaptığımızda: 

![Screenshot_4](https://user-images.githubusercontent.com/65495573/180648228-8f333833-ab25-4b22-9ce4-b62c8d5366c0.png)

Bilgisayar açılınca da:

![Screenshot_5](https://user-images.githubusercontent.com/65495573/180648240-a62a9ae8-7a61-42e8-aec0-84a8b5cee919.png)

Tekrar Start Service butonuna tıklayalım ve hatta Unload fonksiyonunun da çıktısını almak için Stop Service tuşuna da tıklayalım.

![Screenshot_6](https://user-images.githubusercontent.com/65495573/180648300-341ab372-e1af-4cb9-a6d5-adff4db80f63.png)

evet, işlem başarılı. Bitmiştir...

## Driver Reverse Edelim

### DIE Sonuçları

Hayatımda hiç driver görmemiş olsam önce Detect It Easy'e atmak aklıma gelirdi bu dosya türü ne ile eşleşiyor diye bakalım:

`Linker: Microsoft Linker(14.32**)[Driver64,signed]`
![Screenshot_7](https://user-images.githubusercontent.com/65495573/180656096-18e33196-7656-4f2f-8f16-2586719275fb.png)

Tabii strings bakmadan olmaz. Yazdığımız stringler karşılıyor bizi

![Screenshot_8](https://user-images.githubusercontent.com/65495573/180656193-c2064f09-3391-4f84-850c-cf1657cbc0d0.png)

ntoskrnl.exe içerisinden import ettiği fonksiyonlar:
![Screenshot_9](https://user-images.githubusercontent.com/65495573/180656228-2f57c8b6-e01c-4265-8435-d82c1b3a0ca8.png)

### IDA'ya Gönderelim

Dosyayı IDA ile açalım bakalım içerde neler dönüyor :)
![Screenshot_10](https://user-images.githubusercontent.com/65495573/180656829-43247515-3edf-4001-85a7-b9364e3b2de3.png)

Fonksiyon listesi bu şekilde. Burada DriverEntry'i ve ntoskrnl.exe dosyasından import edilen DbgPrint fonksiyonunu görebiliyoruz ki zaten pembe ile highlight edilmesinin sebebi `External Symbol` sayılması. Fakat bir sorun var Unload fonksiyonu nerede :D şimdi DriverEntry fonksiyonuna girip bakalım neler olmuş...

![Screenshot_11](https://user-images.githubusercontent.com/65495573/180656927-3b6a4ccd-3a19-49af-a25e-b53ff53ae7ae.png)

Assembly komutlarını okuyamayanlar için bir de decompiled versiyonunu paylaşalım.

{% highlight cpp %}
NTSTATUS __stdcall DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath)
{
  const UNICODE_STRING *v2; // rbx
  PDRIVER_OBJECT v3; // rdi

  v2 = RegistryPath;
  v3 = DriverObject;
  sub_140005000();
  return sub_14000111C((__int64)v3, v2);
}
{% endhighlight %}

Şimdi burada ilk başta argüman ataması yapılmış bu genel olarak çoğu dosyayı reverse ederken karşıma çıkan bir şey oluyor. Fakat burada fark ettiyseniz birkaç değişiklik var. Biz DriverEntry içerisinde hiçbir fonksiyon çağırmamıştık üstelik return ettiği fonksiyon argümanları almış ve devam etmiş bu da neyin nesi oluyor? DriverObject yapısındaki DriverUnload eventine de atama yapmıştık hatırlarsanız o da yok burada :)) 

Garip ve çağırılmış bir fonksiyona bir bakalım neymiş bu
{% highlight cpp %}
__int64 sub_140005000()
{
  __int64 result; // rax

  if ( !_security_cookie || _security_cookie == 47936899621426i64 )
    __fastfail(6u);
  result = ~_security_cookie;
  qword_140003038 = ~_security_cookie;
  return result;
}
{% endhighlight %}

[/GS (Buffer Security Check)](https://docs.microsoft.com/en-us/cpp/build/reference/gs-buffer-security-check?view=msvc-170) arabellek taşmasını engellemek adına yapılmış bir koruyucu fonksiyon. Derleyici tarafından konulmuş yani bizimle bir alakası yok.

return edilen fonksiyona bakalım.

![Screenshot_12](https://user-images.githubusercontent.com/65495573/180657712-25317d42-3919-464c-8e1a-94731c16596c.png)

{% highlight cpp %}
__int64 __fastcall sub_14000111C(__int64 a1, const UNICODE_STRING *a2)
{
  __int64 v2; // rdi
  __int64 result; // rax
  __int64 v4; // rdx
  __int64 v5; // rcx
  signed int v6; // ebx
  __int64 v7; // r8
  __int64 v8; // r9
  unsigned int v9; // eax
  __int64 v10; // rax

  v2 = a1;
  if ( !a1 )
    return sub_140001000(0i64);
  qword_1400032B8 = a1;
  *(_DWORD *)&DestinationString.Length = 34078720;
  DestinationString.Buffer = (PWSTR)&unk_140003080;
  RtlCopyUnicodeString(&DestinationString, a2);
  result = WdfVersionBind(v2, &DestinationString, &unk_140003000, &qword_1400032B0);
  if ( (int)result >= 0 )
  {
    qword_140003288 = *(_QWORD *)(qword_1400032A8 + 1608);
    v6 = sub_140001384((__int64)&unk_140003000);
    if ( v6 < 0 )
      goto LABEL_8;
    v6 = sub_1400012C0();
    if ( v6 < 0 )
      goto LABEL_8;
    v9 = sub_140001000(v2);
    v6 = v9;
    if ( (v9 & 0x80000000) != 0 )
    {
      DbgPrintEx(77i64, 0i64, "DriverEntry failed 0x%x for driver %wZ\n", v9);
      sub_140001060();
LABEL_8:
      sub_1400010BC(v5, v4, v7, v8);
      return (unsigned int)v6;
    }
    if ( *(_BYTE *)(qword_1400032B0 + 48) )
    {
      v10 = qword_1400032C0;
      if ( *(_QWORD *)(v2 + 104) )
        v10 = *(_QWORD *)(v2 + 104);
      qword_1400032C0 = v10;
      *(_QWORD *)(v2 + 104) = sub_140001290;
    }
    else if ( *(_DWORD *)(qword_1400032B0 + 8) & 2 )
    {
      qword_140003288 = (__int64)sub_140001280;
    }
    result = 0i64;
  }
  return result;
}
{% endhighlight %}

dikkat edilmesi gereken yer en başı:
{% highlight cpp %}
v2 = a1;
if ( !a1 )
  return sub_140001000(0i64);
{% endhighlight %}

`0i64 = a1` 

{% highlight cpp %}
__int64 __fastcall sub_140001000(__int64 a1)
{
  *(_QWORD *)(a1 + 0x68) = sub_140001040;
  DbgPrint("Driver yuklendi!\n");
  return 0i64;
}
{% endhighlight %}

evet, içerisine girdiğimiz zaman da bizim yazdığımız DriverEntry fonksiyonunu görüyoruz `*(a1 + 0x68)` yapısı _DRIVER_OBJECT yapısının içindeki `PDRIVER_UNLOAD DriverUnload;` satırını işaret ediyor ve atama yapılıyor. Tahmin edersiniz ki sub_140001040 fonksiyonu da Unload rutinimiz.
