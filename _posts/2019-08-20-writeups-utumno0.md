---
layout: post
title: "Overthewire - Utumno0 Çözümü"
subtitle: "C Kütüphaneleri mevzuları"
date: 2019-07-20 14:08:13 +0300
background: '/img/posts/bg.png'
---
# Overthewire Çözümleri - Utumno0

## İpucu
- C dinamik kütüphaneleri
- LD_PRELOAD

## Çözüm

Programı çalıştırdığımızda "Read me! :P" çıktısını görüyoruz. Fakat programı **strings** komutuyla açmaya çalıştığımızda izin vermiyor.

```sh
strings: /utumno/utumno0: Permission denied
```

Zaten sonrasında **ls -l** ile programın okuma izni olmadığını
görüyoruz.

```sh
---x--x--- 1 utumno1 utumno0 7188 Nov  2  2018 utumno0
```

Programı okuyamıyoruz. Bu demek oluyor ki **ltrace**, **gdb**, **strings** gibi programların hiçbiri işe yaramayacak.

Programla ilgili şunu biliyoruz ki bir fonksiyon yardımıyla çıktı veriliyor ve bu fonksiyon da yüksek ihtimalle **puts** ya da **printf**'ten bir tanesi. Bu fonksiyonların zaten C kütüphanesinde olduğunu ve elle yazmadığımızı biliyoruz. Yani bir şekilde programın bizim yazdığımız fonksiyonu çağırmasını sağlayabilirsek, programı rahatlıkla okuyabiliriz.

Bunu yapmak için **LD_PRELOAD** çevre değişkenini kullanmamız gerekiyor. C programı çalıştırıldığında eğer **LD_PRELOAD** değişkeninin bir değeri varsa, diğer tüm kütüphanelerden önce **LD_PRELOAD** ile belirtilen kütüphane yüklenir.

Yani biz de kendi **puts** fonksiyonumuzu yazarak programın istediğimiz gibi çalışmasını sağlayabiliriz.

Öncelikle çok basit bir şekilde fonksiyonumuzun çalışıp çalışmadığını test edelim.

```sh
cd `mktemp -d`
vim puts.c
```
```c
#include <stdio.h>

int puts(const char *str) {
  
    printf("Çalıştı");

    return 0;
}
```
```sh
gcc -m32 -fPIC -shared -o puts.so puts.c

LD_PRELOAD=./puts.so /utumno/utumno0
```

Çıktı:
```sh
Çalıştı
```

Fonksiyonumuz çalışıyor. O zaman stack değerlerini okuyarak analizimize başlayalım.

```c
#include <stdio.h>

int puts(const char *str) {

    printf("%x %x %x %x %x %x %x %x %x %x %x %x %x %x %x %x %x %x\n");

    return 0;
}
```

Gördüğünüz gibi kendi **format string** açığımızı oluşturarak **stack**'ten veri okuyacağız. Burada dikkat edilmesi gereken nokta, **printf** fonksiyonu, %x değerlerini her seferinde aynı bellekten okumaya başlayacağı için malesef for döngüsü kullanamıyoruz.

Çıktı:
```
f7fee710 ffffd5f4 f7fcf52c f7fc3dbc 0 ffffd5c8 8048402 80484a5 8048490 0 f7e27286 1 ffffd664 ffffd66c 0 0 0 f7fc2000
```

Bu çıktıyı şu şekilde değerlendirebiliriz. Fonksiyon çağırmalarda return adreslerini stack'e atılır. Kodumuzun 804 ile başlayan adreslere koyulacağını zaten biliyoruz. Böylece inceleyeceğimiz aralığı **8048402-80484a5** olarak belirleyebiliriz. Bir deneyelim:

```c
#include <stdio.h>

int puts(const char *str) {
  
    char *cp;
    int i;

    for (i = 0x8048402; i < 0x80484a5; i++ ) {
        cp = i;
        printf("%c", *cp);
   }
   return 0;
}
```

Çıktı:
```
�����f�f�UWVS�����Û��
                       �l$ ��
                             ����W�������)�����t%1������t$,�t$,U���������9�u���
                                                                                   [^_]Ív��S��������7�[�password: AŞIRI GİZLİ
```

Neyse ki ilk seferde parolayı bulduk. Parolayı çok farklı ve çok daha zor şekilde koda yerleştirebilirlerdi. En azından konsepti öğrenmiş olduk.

## Referanslar

https://medium.com/@meghamohan/everything-you-need-to-know-about-libraries-in-c-e8ad6138cbb4

https://catonmat.net/simple-ld-preload-tutorial