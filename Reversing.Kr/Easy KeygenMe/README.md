# 解法

2つの問題ファイルが提供される。
* Easy Keygen.exe
* ReadMe.txt

ReadMe.txtの内容は下記の通り。

```
ReversingKr KeygenMe


Find the Name when the Serial is 5B134977135E7D13

```

pestudioによるEasy Keygen.exeの文字列分析の結果を確認する。気になるのはこの辺の文字列。

```
ascii,5,0x00008030,-,-,-,Wrong
ascii,8,0x00008038,-,-,-,Correct!
ascii,14,0x00008044,-,-,-,Input Serial: 
ascii,6,0x00008054,-,-,-,%s%02X
ascii,12,0x00008060,-,-,-,Input Name: 
```

Easy Keygen.exeを実行してみると名前とシリアル番号の入力を求められる。
入力するシリアル番号はReadMe.txtに記載されたもので良いと思うので、正しいNameを知る必要がありそう。

```
D:\CTF\Reversing.Kr\02.Easy_Keygen\Easy_KeygenMe>"Easy Keygen.exe"
Input Name: unko
Input Serial: 5B134977135E7D13
Wrong
```

IDA Proで静的解析を実施する。

scanf()でNameに該当すると思われる文字列の入力を求められる。入力後、小さいループで入力された文字列をエンコードしている。

"[esp+esi+13Ch+var_130]"に格納されている値が1バイトずつXORキーに使用されていて、XORキーは3バイト毎にリセットされている。上の方の処理を確認すると、この変数には`0x102030`という3バイトの値が格納されていることが分かる。
また、"[esp+ebp+13Ch+var_12C]"の方はXOR対象の文字列(=Name)が格納される。

```
.text:00401077                         loc_401077:                             ; CODE XREF: _main+B4↓j
.text:00401077 83 FE 03                                cmp     esi, 3
.text:0040107A 7C 02                                   jl      short loc_40107E
.text:0040107C 33 F6                                   xor     esi, esi
.text:0040107E
.text:0040107E                         loc_40107E:                             ; CODE XREF: _main+7A↑j
.text:0040107E 0F BE 4C 34 0C                          movsx   ecx, [esp+esi+13Ch+var_130] ★
.text:00401083 0F BE 54 2C 10                          movsx   edx, [esp+ebp+13Ch+var_12C] ★
.text:00401088 33 CA                                   xor     ecx, edx
.text:0040108A 8D 44 24 74                             lea     eax, [esp+13Ch+Buffer]
.text:0040108E 51                                      push    ecx
.text:0040108F 50                                      push    eax
.text:00401090 8D 4C 24 7C                             lea     ecx, [esp+144h+Buffer]
.text:00401094 68 54 80 40 00                          push    offset Format   ; "%s%02X"
.text:00401099 51                                      push    ecx             ; Buffer
.text:0040109A E8 B1 00 00 00                          call    _sprintf
.text:0040109F 83 C4 10                                add     esp, 10h
.text:004010A2 45                                      inc     ebp
.text:004010A3 8D 7C 24 10                             lea     edi, [esp+13Ch+var_12C]
.text:004010A7 83 C9 FF                                or      ecx, 0FFFFFFFFh
.text:004010AA 33 C0                                   xor     eax, eax
.text:004010AC 46                                      inc     esi
.text:004010AD F2 AE                                   repne scasb
.text:004010AF F7 D1                                   not     ecx
.text:004010B1 49                                      dec     ecx
.text:004010B2 3B E9                                   cmp     ebp, ecx
.text:004010B4 7C C1                                   jl      short loc_401077
```

XORでエンコードされたNameと、入力されたSerialが1バイトずつ比較されている。

```
.text:004010DA E8 C3 00 00 00                          call    _scanf
.text:004010DF 83 C4 08                                add     esp, 8
.text:004010E2 8D 74 24 74                             lea     esi, [esp+13Ch+Buffer]
.text:004010E6 8D 44 24 10                             lea     eax, [esp+13Ch+var_12C]
.text:004010EA
.text:004010EA                         loc_4010EA:                             ; CODE XREF: _main+108↓j
.text:004010EA 8A 10                                   mov     dl, [eax]
.text:004010EC 8A CA                                   mov     cl, dl
.text:004010EE 3A 16                                   cmp     dl, [esi]
.text:004010F0 75 1C                                   jnz     short loc_40110E
.text:004010F2 84 C9                                   test    cl, cl
.text:004010F4 74 14                                   jz      short loc_40110A
.text:004010F6 8A 50 01                                mov     dl, [eax+1]
.text:004010F9 8A CA                                   mov     cl, dl
.text:004010FB 3A 56 01                                cmp     dl, [esi+1]
.text:004010FE 75 0E                                   jnz     short loc_40110E
.text:00401100 83 C0 02                                add     eax, 2
.text:00401103 83 C6 02                                add     esi, 2
.text:00401106 84 C9                                   test    cl, cl
.text:00401108 75 E0                                   jnz     short loc_4010EA
.text:0040110A
```

XOR結果がSerialと同一になるNameを入力する必要があると分かったので、CyberChefを使用してXORを行う。その結果、`K3yg3nm3`が期待されている入力値であることが分かった。

> https://gchq.github.io/CyberChef/#recipe=From_Hex('Auto')XOR(%7B'option':'Hex','string':'102030'%7D,'Standard',false)&input=NUIxMzQ5NzcxMzVFN0QxMw

```
D:\CTF\Reversing.Kr\02.Easy_Keygen\Easy_KeygenMe"Easy Keygen.exe"
Input Name: K3yg3nm3
Input Serial: 5B134977135E7D13
Correct!
```

Flag : K3yg3nm3
