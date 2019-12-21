[asin:B00IR1HYI0:detail]

## 概要
『30日でできる！OS自作入門』で取り扱っているはりぼてOSでは，最大256色使用できる8ビットカラーを使用しています((パレットに16色しか登録していないので4ビットカラーと勘違いするかもしれない（僕も勘違いした）ですが，P83に記載があるように8ビットカラーを使用しています．))．これに対し，最大16,777,216色使用できる24ビットカラーや32ビットカラーが存在します．この記事では，はりぼてOSで24ビットカラーや32ビットカラーを使用するようにコードを書き換えます．また，解像度も大きくします．

### 本の中で述べられている解像度の拡大の方法との違い
使用するVBEモードに対応するモードナンバの選択が，本中のやり方とこの記事でのやり方で異なります．本中では，予めVESAが定義したモードナンバを使用していますが，この記事では，BIOSに使用可能なモードナンバを問い合わせ，その中で最も解像度の高い，かつ24ビットカラーか32ビットカラーでダイレクトカラーのモードナンバを使用します．

VBEのバージョンが2.0未満の時代は，予めVESAが定義したモードナンバに対応するVBEモードを使用していました．しかし2.0からは，VESAはモードナンバを新たに定義することを止め，代わりにBIOSに使用可能なモードナンバを問い合わせる事を推奨しました(([規格書](http://www.petesqbsite.com/sections/tutorials/tuts/vbe3.pdf)のPage19にあるNoteを参照．))．バージョン2.0以上でもVESAが既に定義しているモードナンバを使用することも可能ですが，1280x1024以上の解像度が定義されておらず，それ以上の解像度を使用することができません(([規格書](http://www.petesqbsite.com/sections/tutorials/tuts/vbe3.pdf)のPage19にある表を参照))．よってこの記事ではBIOSに使用可能なモードナンバを問い合わせ，その中で最も解像度の高い，かつ24ビットカラーか32ビットカラーそしてダイレクトカラーのモードナンバを使用します．

### ダイレクトカラー
ダイレクトカラーとは，色をその構成要素に分解し，構成要素毎に値を指定して最終的な色を指定する方式です．例えばRGB，つまり赤緑青の値を指定する方式などが該当します．これに対し，予めパレットに色を登録し，そのパレットの番号を指定する，インデックスカラーという方法も存在します．はりぼてOSではこのインデックスカラーを使用しています．今回使用する24ビットカラーや32ビットカラーでは，ダイレクトカラー方式で色を指定します．

### 24ビットカラーと32ビットカラーの違い
まず，24ビットカラーも32ビットカラーも，赤，緑，青の3色の強さをそれぞれ0から255の計256種類の値で指定します．よって256の3乗で16,777,216色を表すことができます．また，赤緑青でそれぞれ8ビット使用することから，計24ビット使用することになります．

32ビットカラーはこの24ビットに加え，追加の8ビットが与えられます．これはアルファチャンネルと言われ，例えば透明度の設定などに使用されますが，全く使用されない場合もあります．何れにせよ，表現できる総色数は変わりません．

### 24ビットカラーや32ビットカラー以外を選択肢から外す理由
単純に24ビットカラーや32ビットカラーが一番多くの種類の色を扱えるから，というのもありますが，他の色深度，例えば16ビットカラーや15ビットカラーだと色を指定するのが少し面倒なんですよね．[Wikipedia](https://ja.wikipedia.org/wiki/%E8%89%B2%E6%B7%B1%E5%BA%A6)によれば，15ビットカラーは赤緑青でそれぞれ5ビットを使用します．これならまだいいのですが，16ビットカラーでは緑だけ6ビットを使用．人間の目は緑に対し敏感というのが理由のようですが，このようにビット数が異なると当然その分コードも複雑になるので，ここでは24ビットカラーか32ビットカラーのみを選択肢として残すことにします．

### 注意
この記事ではCではなくRustでコードを書いていますが，適宜Cでのの解説を挟んであります．また，本の著者が提供しているツールを使用せず，アセンブリ言語のコンパイルにはnasmを使用しています．そのため一部文法が異なる場合があります．そして，書いている本人は本の最後まで実装しきれていません((具体的にはまだ9日目に到達していない．))．故に今回使用しているメモリ領域が後に別の項目で使用される場合もあります．そのような場合，適宜使用するメモリの番地を変更してください．

## やり方

### asmhead.nasの書き換え
書き換えたものはこちらになります．

<div onclick="obj=document.getElementById('full_asm_code').style; obj.display=(obj.display=='none')?'block':'none';">
<a style="cursor:pointer;">クリックで表示</a>
</div>
<div id="full_asm_code" style="display:none;clear:both;">
```asm
; BOOT_INFO関係
CYLS    EQU     0x0ff0          ; ブートセクタが設定する
LEDS    EQU     0x0ff1

BPP   EQU     0x0ff2          ; 色数に関する情報。何ビットカラーか？
SCRNX   EQU     0x0ff4          ; 解像度のX
SCRNY   EQU     0x0ff6          ; 解像度のY
VRAM    EQU     0x0ff8          ; グラフィックバッファの開始番地
VBEMODE EQU     0x0ffc          ; VBE mode number. word size
VBE_INFO_SIZE EQU 0x0200

VBE     EQU     0x9000

        ORG     0xc200          ; このプログラムがどこに読み込まれるのか
; If VBE doesn't exist, the resolution will be 320x200
        MOV     AX,VBE
        MOV     ES,AX
        MOV     DI,0
        MOV     AX,0x4f00
        INT     0x10
        CMP     AX,0x004f
        JNE     screen_320

; If the version of VBE is less than 2.0, set the resolution as 320x200
        MOV     AX,WORD[ES:DI+4]
        CMP     AX,0x0200
        JB      screen_320

; Loop initialization
        MOV     BYTE[BPP],8
        MOV     WORD[SCRNX],320
        MOV     WORD[SCRNY],200
        MOV     DI,VBE_INFO_SIZE
select_mode:

VMODE_PTR EQU 14
; Get VESA mode number
        MOV     SI,WORD[ES:VMODE_PTR]
        MOV     FS,WORD[ES:VMODE_PTR+2]
        MOV     CX,WORD[FS:SI]

        CMP     CX,0xffff
        JE      finish_select_mode

; Get VESA mode information.
        MOV     AX,0x4f01

        INT     0x10

        CMP     AX,0x004f
        JNE     next_mode

; Check if this graphics mode supports linear frame buffer support.
        MOV     AX,WORD[ES:DI]
        AND     AX,0x80
        CMP     AX,0x80
        JNE     next_mode

; Check if this is a packed pixel
        MOV     AX,WORD[ES:DI+27]
        CMP     AX,4
        JE      valid_mode

; Check if this is a direct color mode
        CMP     AX,6
        JE      valid_mode

        JMP     next_mode

valid_mode:
; Compare dimensions
        MOV     AX,WORD[ES:DI+18]
        CMP     AX,WORD[SCRNX]
        JB      next_mode

        MOV     AX,WORD[ES:DI+20]
        CMP     AX,WORD[SCRNY]
        JB      next_mode

; If bpp is not 24 bit or 32 bit, don't use this.
        CMP     BYTE[ES:DI+25],24
        JB      next_mode

; Set dimension and bits number
        MOV     AX,WORD[ES:DI+18]
        MOV     WORD[SCRNX],AX

        MOV     AX,WORD[ES:DI+20]
        MOV     WORD[SCRNY],AX

        MOV     AL,BYTE[ES:DI+25]
        MOV     BYTE[BPP],AL

        MOV     AX,WORD[ES:DI+40]
        MOV     WORD[VRAM],AX
        MOV     AX,WORD[ES:DI+40+2]
        MOV     WORD[VRAM+2],AX

        MOV     WORD[VBEMODE],CX

next_mode:
        MOV     AX,WORD[ES:VMODE_PTR+2]
        ADD     AX,2
        MOV     WORD[ES:VMODE_PTR+2],AX

        JMP     select_mode

finish_select_mode:
        CMP     WORD[SCRNX],320
        JNE     set_vbe_mode

        CMP     WORD[SCRNY],200
        JNE     set_vbe_mode

        CMP     BYTE[BPP],8
        JNE     set_vbe_mode

        JMP     screen_320

set_vbe_mode:
        MOV     AX,0x4f02
        MOV     BX,WORD[VBEMODE]
        OR      BX,0x4000
        INT     0x10

        CMP     AX,0x004f
        JE      keystatus

screen_320:
        MOV     AL,0x13
        MOV     AH,0x00
        INT     0x10
        MOV     BYTE [BPP],8
        MOV     WORD [SCRNX],320
        MOV     WORD [SCRNY],200

; DO NOT FOLLOW THE INSTRUCTIONS WRITTEN IN BOOK!
; SEE https://qiita.com/tatsumack/items/491e47c1a7f0d48fc762
        MOV     DWORD [VRAM],0xfd000000

keystatus:
```
</div>

以下は説明となります．
#### 定数定義の追加と名称変更
使用するVBEのモードナンバがどこに格納されているかと，VBEの情報の大きさに関する定義を追加します．
```asm
VBEMODE EQU 0x0ffc
VBE_INFO_SIZE EQU 512
```
後にVBEの情報を取得する関数を紹介しますが，VBEの情報は，VBEのバージョンが2.0未満では256バイト，2.0以上では512バイトの大きさです．

また，`VMODE`に関しては，`BPP`((bits per pixel))の方が分かりやすいので，名前を変更しています．これは任意です．
```asm
BPP EQU 0x0ff2
```
### 利用可能な画面モードの取得
[規格書](http://www.petesqbsite.com/sections/tutorials/tuts/vbe3.pdf)のPage25より引用．

>**Function 00h - Return VBE Controller Information**
>
>**Input**:
>
>AX      = 4F00h     Return VBE Controller Information
>
>ES:DI   =           Pointer to buffer in which to place VbeInfoBlock structure (VbeSignature should be set to 'VBE2' when function is called to indicate VBE 3.0 information is desired and the information block is 512 bytes in size.)
>
>**Output**:    AX      =           VBE Return Status
>
>
>**Note**: All other registers are preserved.

この関数を使うことで，利用可能なビデオモードなどが格納されている情報を取得することができます．

情報の構成は，次の構造体のような構成になっています．[規格書](http://www.petesqbsite.com/sections/tutorials/tuts/vbe3.pdf)Page25に記載されているものを表にしました．

|名前|大きさ|格納されているデータなど|説明|
|----|------|------------------------|----|
|VbeSignature|db((バイト))|'VESA'|VBE Signature|
|VbeVersioin|dw((ワード))|0300h|VBE Version|
|OemStringPtr|dd((ダブルワード))||VbeFarPtr to OEM String|
|Capabilities|db|4 dup?((4つの要素が存在する))|Capabilities of graphics controller|
|VideoModePtr|dd||VbeFarPtr to VideoModeList|
|TotalMemory|dw||Number of 64kb memory blocks. Added for VBE 2.0+|
|OemSoftwareRev|dw||VBE implementation Software revision|
|OemVendorNamePtr|dd||VbeFarPtr to Vendor Name String|
|OemProductNamePtr|dd||VbeFarPtr to Product Name String|
|OemProductRevPtr|dd||VbeFarPtr to Product Revision String|
|Reserved|db|222 dub?|Reserved for VBE implementation scratch area|
|OemData|db|256dub?|Data Area for OEM Strings|

ところで，この関数は本中ののP278でも使用されてます．そこではVBEの存在確認としてその関数を呼び出してまずが，以下のコードはそれに若干可変を加えたものです．具体的には，`VBE`に`0x9000`を対応付け，それを`ES`レジスタに代入して利用しています．
```asm
VBE EQU 0x9000

MOV AX,VBE
MOV ES,AX
MOV DI,0
MOV AX,0x4f00
INT 0x10
CMP AX,0x004f
JNE screen_320
```
関数の実行に成功すれば，`AX`レジスタに`0x004F`が代入され，アドレス`VESABIOS`から512バイト((VBEのバージョンが2.0未満ならば216バイト))，VBEに関する情報が格納されます．異なっていれば`screen_320`ラベルに飛ばします．本ではもしVBEが存在しなかった場合，解像度を320x200にするという意味で`JNE screen_320`を書いています．

この関数で得られた情報の中にはVBEのバージョンも含まれています．VBEのバージョンが2.0未満である場合，BIOSからモードナンバを得ることができないので，解像度320x200の4ビットカラーを使用します．このコードは同書P279からの引用です．
```asm
MOV AX,[ES:DI+4]
CMP AX,0x0200
JB screen_320
```
#### 使用するモードナンバの選択
今回は，24ビットカラーあるいは32ビットカラーで，解像度が最大のモードナンバを使用することにします．

使用可能なモードナンバはメモリ上に，配列のように連続して格納されています．モードナンバの終端は`0xFFFF`という値で終了しています．今回はこのモードナンバの列を走査して，順に解像度と色数を確かめていきます．

#### ループの初期化
ループの初期化として，使用可能な色数，解像度の横の長さ，縦の長さを指定します．
```asm
MOV BYTE[BPP],8
MOV WORD[SCRNX],320
MOV WORD[SCRNY],200
```
また，`DI`レジスタに，VBEの情報の大きさ分の値を代入しておきます．
```asm
MOV DI,VBE_INFO_SIZE
```
VBEの情報を取得したときと同様，モードナンバに対応するビデオモードの情報を取得すると，情報は`ES:DI`を始点として情報が格納されます．従って，VBEの情報を上書きしないために，VBEの情報の大きさの値を`DI`レジスタに格納しておきます．

#### ループ本体

##### ループ開始位置のラベル付与
ループの開始を示すためにラベルを配置する．
```asm
select_mode:
```

##### 定数定義
モードナンバの配列の先頭のアドレスが，先程載せたVBEの情報が格納されている構造体の`video_modes`変数に格納されている．この変数は構造体の先頭から14ビット離れた位置にあるので，それを定数とする．
```asm
VMODE_PTR EQU 14
```

##### モードナンバの取得
モードナンバの配列からモードナンバを取得する．
```asm
MOV SI,WORD[ES:VMODE_PTR]
MOV FS,WORD[ES:VMODE_PTR+2]
MOV CX,WORD[FS:SI]
```

そして，もしモードナンバの値が`0xFFFF`だった場合，モードナンバの選択を終わらせる．
```asm
CMP CX,0xFFFF
JE finish_select_mode
```

##### モードナンバに対応するビデオモードの情報取得
[https://wiki.osdev.org/User:Omarrx024/VESA_Tutorial:title]より引用．

>**FUNCTION: Get VESA mode information**
>
>Function code: 0x4F01
>
>Description: This function returns the mode information structure for a specified mode. The mode number should be gotten from the supported modes array.
>
>**Input**: AX = 0x4F01
>
>**Input**: CX = VESA mode number from the video modes array
>
>**Input**: ES:DI = Segment:Offset pointer of where to store the VESA Mode Information Structure shown below.
>
>**Output**: AX = 0x004F on success, other values indicate a BIOS error or a mode-not-supported error.

この関数を利用して，ビデオモードの情報を得る．
```asm
MOV AX,0x4F01
INT 0x10
```
もし失敗した場合，次のモードを候補とする．
```asm
CMP AX,0x004F
JNE next_mode
```
ところで，モードナンバが配列の要素として格納されていながら，実際にその番号を使用してビデオモードの情報を得ようとして失敗する場合は存在するようである．[規格書](http://www.petesqbsite.com/sections/tutorials/tuts/vbe3.pdf)のPage27のNoteより引用．
>It is responsibility of the application to verify the actual availability of any mode returnedby this function through the Return VBE Mode Information (VBE Function 01h) call.Some of the returned modes may not be available due to the actual amount of memoryphysically installed on the display board or due to the capabilities of the attached monitor.

つまり，例えば使用できるメモリの大きさが，ビデオRAMの大きさよりも小さい場合などが該当するようである．

ビデオモードの情報は，以下の構造体のような構成となっている．[https://wiki.osdev.org/User:Omarrx024/VESA_Tutorial:title]より引用．
```c
struct vbe_mode_info_structure {
	uint16 attributes;		// deprecated, only bit 7 should be of interest to you, and it indicates the mode supports a linear frame buffer.
	uint8 window_a;			// deprecated
	uint8 window_b;			// deprecated
	uint16 granularity;		// deprecated; used while calculating bank numbers
	uint16 window_size;
	uint16 segment_a;
	uint16 segment_b;
	uint32 win_func_ptr;		// deprecated; used to switch banks from protected mode without returning to real mode
	uint16 pitch;			// number of bytes per horizontal line
	uint16 width;			// width in pixels
	uint16 height;			// height in pixels
	uint8 w_char;			// unused...
	uint8 y_char;			// ...
	uint8 planes;
	uint8 bpp;			// bits per pixel in this mode
	uint8 banks;			// deprecated; total number of banks in this mode
	uint8 memory_model;
	uint8 bank_size;		// deprecated; size of a bank, almost always 64 KB but may be 16 KB...
	uint8 image_pages;
	uint8 reserved0;

	uint8 red_mask;
	uint8 red_position;
	uint8 green_mask;
	uint8 green_position;
	uint8 blue_mask;
	uint8 blue_position;
	uint8 reserved_mask;
	uint8 reserved_position;
	uint8 direct_color_attributes;

	uint32 framebuffer;		// physical address of the linear frame buffer; write here to draw to the screen
	uint32 off_screen_mem_off;
	uint16 off_screen_mem_size;	// size of memory in the framebuffer but not being displayed on the screen
	uint8 reserved1[206];
} __attribute__ ((packed));
```

##### linear framebufferに対応しているかの確認
linear framebufferに対応していると，ビデオRAMのすべてのメモリが一列に並ぶ．つまり，ディスプレイのどのピクセルもこのビデオRAMのどこかしらに対応している．linear framebufferに対応していない場合，複数のbankというものに区分けされ，時々bankを切り替える必要がある．

ビデオモードがlinear framebufferに対応しているかは，ビデオモードの情報の中の`attributes`の第7ビットが1になっているかで確認する．これが1ならlinear framebufferに対応している．

```asm
MOV AX,WORD[ES:DI]
AND AX,0x80
CMP AX,0x80
JNE next_mode
```

##### packed pixelかどうかの確認
packed pixelというのは，各ピクセルとメモリをバイト単位で結びつけるのではなく，ビット単位で結びつける．例えば4ビットカラーならば，1ピクセルを1バイト中の4ビットと結びつけ，残りの4ビットを使用しないのではなく，1バイトを2ピクセルと対応付け，前半後半の4ビットをぞれぞれ1ピクセルと対応付ける．つまり隙間を作らないという意味で*packed*ということである．

ビデオモードがpacked pixelかどうかは構造体の`memory_model`の値を確認する．
```asm
MOV AX,WORD[ES:DI+27]
CMP AX,4
JE valid_mode
```
27というのは，`memory_model`の構造体の先頭からの距離である．もしこの値が4ならば，使用するビデオモードの候補として有効なため，`valid_mode`ラベルに飛ばす．

##### ダイレクトカラーかどうかの確認
```asm
CMP AX,6
JE valid_mode
```
既に`memory_model`の値は`AX`レジスタに格納しているので，ここでは`MOV`の必要はない．

最後に，もしpacked pixelでもなくダイレクトカラーでもなければ，次のモードを候補とする．
```asm
JMP next_mode
```

##### 解像度の比較
packed pixelかダイレクトカラーだと確認できれば，次は解像度の比較を行う．そのためのラベルを貼る．
```asm
valid_mode:
```

解像度の横，縦はそれぞれ構造体の`width`，`height`に格納されている．構造体の先頭からの距離はそれぞれ18バイト，20バイトである．

```asm
MOV AX,WORD[ES:DI+18]
CMP AX,WORD[SCRNX]
JB next_mode

MOV AX,WORD[ES:DI+20]
CMP AX,WORD[SCRNY]
JB next_mode
```

もし横，縦がどちらかでも，それまでの候補のビデオモードよりも小さければ次の候補に回す．

##### 色数の比較
先程述べたように，24ビットカラーか32ビットカラー以外は選択肢から外する．色数は構造体のbppに格納され，先頭から25バイト離れている．
```asm
CMP BYTE[ES:DI+25],24
JB next_mode
```

##### 候補の更新
見事今までの選考基準を通過したビデオモードが登場した場合，解像度や色数に関する情報を更新する．

```asm
MOV AX,WORD[ES:DI+18]
MOV WORD[SCRNX],AX

MOV AX,WORD[ES:DI+20]
MOV WORD[SCRNY],AX

MOV AL,BYTE[ES:DI+25]
MOV BYTE[BPP],AL

MOV AX,WORD[ES:DI+40]
MOV WORD[VRAM],AX
MOV AX,WORD[ES:DI+40+2]
MOV WORD[VRAM+2],AX

MOV WORD[VBEMODE],CX
```

#### 次のループへの処理
モードナンバの配列のポインタを更新し，ループの先頭へ飛ぶ．
```asm
MOV AX,WORD[ES:VMODE_PTR]
ADD AX,2
MOV WORD[ES:VMODE_PTR],AX

JMP select_mode
```

#### ループ終了後の処理
まずラベルを追加する．
```asm
finish_select_mode:
```

##### 解像度と色数の確認
解像度や色数が初期値のままの場合，使用可能なビデオモードが存在しないので320x200を使用する．
```asm
CMP WORD[SCRNX],320
JNE set_vbe_mode

CMP WORD[SCRNY],200
JNE set_vbe_mode

CMP BYTE[BPP],8
JNE set_vbe_mode

JMP screen_320
```

##### VBEモードの設定
まずラベルを貼る．
```asm
set_vbe_mode:
```

VBEモードを登録する．[https://wiki.osdev.org/User:Omarrx024/VESA_Tutorial:title]より引用．
>**FUNCTION: Set VBE mode**
>
>Function code: 0x4F02
>
>Description: This function sets a VBE mode.
>
>**Input:** AX = 0x4F02
>
>**Input:** BX = Bits 0-13 mode number; bit 14 is the LFB bit: when set, it enables the linear framebuffer, when clear, software must use bank switching. Bit 15 is the DM bit: when set, the BIOS doesn't clear the screen. Bit 15 is usually ignored and should always be cleared.
>
>**Output:** AX = 0x004F on success, other values indicate errors; such as BIOS error, too little video memory, unsupported VBE mode, mode doesn't support linear frame buffer, or any other error.

この関数が失敗する場合もある．その場合，320x200を使用する．

```asm
MOV AX,0x4F02
MOV BX,WORD[VBEMODE]
OR BX,0x4000
INT 0x10

CMP AX,0x004F
JE keystatus
```

VBEのモードナンバの第14ビット目は，linear framebufferを使用するか否かについてのビットであり，このビットが1だと使用することを表明する．`OR BX,0x4000`というコードはこのためにある．

##### 320x200を使用する時の処理
本中のP280よりコードを引用．ただしラベルは`scrn320`から`screen_320`に変更しています．また，`VMODE`は`BPP`に変更しています．
```asm
screen_320:
    MOV AL,0x13
    MOV AH,0x00
    INT 0x10
    MOV BYTE [BPP],8
    MOV WORD [SCRNX],320
    MOV WORD {SCRNY],200
```

本ではこの先，ビデオRAMの番地を0x000a0000に指定していますが，この値にするとQEMU上で何も表示されなくなる場合があります((僕も遭遇した．))．[Qiitaの記事](https://qiita.com/tatsumack/items/491e47c1a7f0d48fc762)によれば，番地を0xfd000000に変更することで表示されるようです．

```asm
MOV DWORD [VRAM],0xfd000000
```
## 参考文献

[https://ja.wikipedia.org/wiki/%E8%89%B2%E6%B7%B1%E5%BA%A6:title]

[http://oswiki.osask.jp/?cmd=read&page=VESA&word=vesa:title]

[https://wiki.osdev.org/VESA_Video_Modes:title]

[https://graphicdesign.stackexchange.com/questions/47133/is-32-bit-color-depth-enough:title]

[https://ja.wikipedia.org/wiki/VESA_BIOS_Extensions:title]

[http://oswiki.osask.jp/?%28AT%29memorymap:title]

[https://wiki.osdev.org/Memory_Map_(x86):title]

[https://qiita.com/MoriokaReimen/items/4853587dcb9eb96fab62:title]

[http://niwatolli3.hatenablog.jp/entry/2015/06/01/201341:title]



