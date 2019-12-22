[asin:B00IR1HYI0:detail]

## 概要
『30日でできる！OS自作入門』で取り扱っているはりぼてOSでは，最大256色使用できる8ビットカラーを使用しています((パレットに16色しか登録していないので4ビットカラーと勘違いするかもしれない（僕も勘違いした）ですが，P83に記載があるように8ビットカラーを使用しています．))．これに対し，最大16,777,216色使用できる24ビットカラーや32ビットカラーが存在します．この記事では，はりぼてOSで24ビットカラーや32ビットカラーを使用するようにコードを書き換えます．また，解像度も大きくします．

### 本の中で述べられている解像度の拡大の方法との違い
使用するVBEモードに対応するモードナンバの選択が，本中のやり方とこの記事でのやり方で異なります．本中では，予めVESAが定義したモードナンバを使用していますが，この記事では，BIOSに使用可能なモードナンバを問い合わせ，その中で最も解像度の高い，かつ24ビットカラーか32ビットカラーでダイレクトカラーのモードナンバを使用します．

VBEのバージョンが2.0未満の時代は，予めVESAが定義したモードナンバに対応するVBEモードを使用していましたが，2.0からは，VESAはモードナンバを新たに定義することを止めました(([規格書](http://www.petesqbsite.com/sections/tutorials/tuts/vbe3.pdf)のPage19にあるNoteを参照．))．バージョン2.0以上でもVESAが既に定義しているモードナンバを使用することも可能ですが，1280x1024以上の解像度が定義されておらず，それ以上の解像度を使用することができません(([規格書](http://www.petesqbsite.com/sections/tutorials/tuts/vbe3.pdf)のPage19にある表を参照))．これより大きい解像度を使用したい場合，BIOSに利用可能なVBEモードを問い合わせる必要があります．この記事ではBIOSに使用可能なモードナンバを問い合わせ，その中で最も解像度の高い，かつ24ビットカラーか32ビットカラーそしてダイレクトカラーのモードナンバを使用します．

### ダイレクトカラー
ダイレクトカラーとは，色をその構成要素に分解し，構成要素毎に値を指定して最終的な色を指定する方式です．例えばRGB，つまり赤緑青の値を指定する方式などが該当します．これに対し，予めパレットに色を登録し，そのパレットの番号を指定する，インデックスカラーというものも存在します．はりぼてOSではこのインデックスカラーを使用しています．今回使用する24ビットカラーや32ビットカラーでは，ダイレクトカラー方式で色を指定します．

### 24ビットカラーと32ビットカラーの違い
まず，24ビットカラーも32ビットカラーも，赤，緑，青の3色の強さをそれぞれ0から255の計256種類の値で指定します．よって256の3乗で16,777,216色を表すことができます．また，赤緑青でそれぞれ8ビット使用することから，計24ビット使用することになります．

32ビットカラーはこの24ビットに加え，追加の8ビットが与えられます．これはアルファチャンネルと言われ，例えば透明度の設定などに使用されますが，全く使用されない場合もあります．何れにせよ，表現できる総色数は変わりません．

### 24ビットカラーや32ビットカラー以外を選択肢から外す理由
単純に24ビットカラーや32ビットカラーが一番多くの種類の色を扱えるから，というのもありますが，他の色深度，例えば16ビットカラーや15ビットカラーだと色を指定するのが少し面倒なんですよね．[Wikipedia](https://ja.wikipedia.org/wiki/%E8%89%B2%E6%B7%B1%E5%BA%A6)によれば，15ビットカラーは赤緑青でそれぞれ5ビットを使用します．これならまだいいのですが，16ビットカラーでは緑だけ6ビットを使用．人間の目は緑に対し敏感というのが理由のようですが，このようにビット数が異なると当然その分コードも複雑になるので，ここでは24ビットカラーか32ビットカラーのみを選択肢として残すことにします．

### 注意
この記事では本の著者が提供しているツールを使用せず，アセンブリ言語のコンパイルにはnasmを使用しています．そのため一部文法が異なる場合があります．そして，書いている本人は本の最後まで実装しきれていません((具体的にはまだ9日目に到達していない．))．故に今回使用しているメモリ領域が後に別の項目で使用される場合もあります．そのような場合，適宜使用するメモリの番地を変更してください．

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

この関数を使うことで，利用可能なVBEモードなどが格納されている情報を取得することができます．

OutputのVBE Return Statusは，`0x004F`なら関数の実行の成功，それ以外なら失敗を表します．

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

ところで，この関数は本中のP278でも使用されてます．そこではVBEの存在確認としてこの関数を呼び出してまずが，以下のコードはそれに若干改変を加えたものです．具体的には，`VBE`に`0x9000`を対応付け，それを`ES`レジスタに代入して利用しています．
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

この関数で得られた情報の中にはVBEのバージョンも含まれています((表中のVbeVersion))．VBEのバージョンが2.0未満である場合，BIOSからモードナンバを得ることができないので，解像度320x200の8ビットカラーを使用します．このコードは同書P279からの引用です．
```asm
MOV AX,[ES:DI+4]
CMP AX,0x0200
JB screen_320
```
### 使用するモードナンバの選択
今回は，24ビットカラーあるいは32ビットカラーで，解像度が最大のモードナンバを使用することにします．

使用可能なモードナンバはメモリ上に，配列のように連続して格納されています．モードナンバの終端は`0xFFFF`という値で終了しています((C言語の文字列が'\0'で終わるような感じ))．今回はこのモードナンバの配列を走査して，順に解像度と色数を確かめていきます．

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
VBEの情報を取得したときと同様，モードナンバに対応するVBEモードの情報を取得すると，情報は`ES:DI`を始点として情報が格納されます．従って，VBEの情報を上書きしないために，VBEの情報の大きさの値を`DI`レジスタに格納しておきます．

#### ループ本体

##### ループ開始位置のラベル付与
ループの開始を示すためにラベルを配置します．
```asm
select_mode:
```

##### 定数定義
モードナンバの配列の先頭のアドレスが，先程載せたVBEの情報が格納されている構造体の`video_modes`変数に格納されています．この変数は構造体の先頭から14バイト離れた位置にあるので，それを定数とします．
```asm
VMODE_PTR EQU 14
```

##### モードナンバの取得
モードナンバの配列からモードナンバを取得します．
```asm
MOV SI,WORD[ES:VMODE_PTR]
MOV FS,WORD[ES:VMODE_PTR+2]
MOV CX,WORD[FS:SI]
```

そして，もしモードナンバの値が`0xFFFF`だった場合，モードナンバの選択を終了します．
```asm
CMP CX,0xFFFF
JE finish_select_mode
```

##### モードナンバに対応するVBEモードの情報取得
[規格書](http://www.petesqbsite.com/sections/tutorials/tuts/vbe3.pdf)のPage30より引用．

>**Function 01h - Return VBE Mode Information**
>
>**Input:**
>AX = 0x4F01 Return VBE Mode Information
>
>CX = Mode number
>
>ES:DI = Pointer to ModeInfoBlock structure
>
>**Output**: AX = VBE Return Status
>
>**Note:** All other registers are preserved.

この関数を利用して，VBEモードの情報を取得します．
```asm
MOV AX,0x4F01
INT 0x10
```
もし失敗した場合，次のモードを候補とします．
```asm
CMP AX,0x004F
JNE next_mode
```
ところで，モードナンバが配列の要素として格納されていながら，実際にその番号を使用してVBEモードの情報を得ようとして失敗する場合は存在するようです．[規格書](http://www.petesqbsite.com/sections/tutorials/tuts/vbe3.pdf)のPage27のNoteより引用．
>It is responsibility of the application to verify the actual availability of any mode returnedby this function through the Return VBE Mode Information (VBE Function 01h) call.Some of the returned modes may not be available due to the actual amount of memoryphysically installed on the display board or due to the capabilities of the attached monitor.

つまり，例えば使用できるメモリの大きさが，ビデオRAMの大きさよりも小さい場合などが該当するようです．

VBEモードの情報は，以下の構造体のような構成となっています．[規格書](http://www.petesqbsite.com/sections/tutorials/tuts/vbe3.pdf)のPage30の説明を表にしました．

##### すべてのバージョンのVBEで格納されている情報
|名前|大きさ|格納されているデータなど|説明|
|----|------|------------------------|----|
|ModeAttributes|dw||mode attributes|
|WinAAttributes|db||window A attributes|
|WinBAttributes|db||window B attributes|
|WinGranularity|dw||window granularity|
|WinSize|dw||window size|
|WinASegment|dw||window A start segment|
|WinBSegment|dw||window B start segment|
|WinFuncPtr|dd||real mode pointer to window function|
|BytesPerScanLine|dw||bytes per scan line|

##### バージョン1.2以降のVBEで格納されている情報
|名前|大きさ|格納されているデータなど|説明|
|----|------|------------------------|----|
|XResolution|dw||horizontal resolution in pixels of characters|
|YResolution|dw||vertical resolution in pixels of characters|
|XCharSize|db||character cell width in pixels|
|YCharSize|db||character cell height in pixels|
|NumberOfPlanes|db||number of memory planes|
|BitsPerPixel|db||bits per pixel|
|NumberOfBanks|db||number of banks|
|MemoryModel|db||memory model type|
|BankSize|db||bank size in KB|
|NumberOfImagePages|db||number of images|
|Reserved|db|1|reserved for page function|

##### ダイレクトカラー情報
|名前|大きさ|格納されているデータなど|説明|
|----|------|------------------------|----|
|RedMaskSize|db||size of direct color red mask in bits|
|RedFieldPosition|db||bit position of lsb of red mask|
|GreenMaskSize|db||size of direct color green mask in bits|
|GreenFieldPosition|db||bit position of lsb of green mask|
|BlueMaskSize|db||size of direct color blue mask in bits|
|BlueFieldPosition|db||bit position of lsb of blue mask|
|RsvdMaskSize|db||size of direct color reserved mask in bits|
|RsvdFieldPosition|db||bit position of lsb of reserved mask|
|DirectColorModeInfo|db||direct color mode attributes|

##### バージョン2.0以降のVBEで格納されている情報
|名前|大きさ|格納されているデータなど|説明|
|----|------|------------------------|----|
|PhysBasePtr|dd||physical address for flat memory frame buffer|
|Reserved|dd|0|Reserved - always set to 0|
|Reserved|dw|0|Reserved - always set to 0|

##### バージョン3.0以降のVBEで格納されている情報
|名前|大きさ|格納されているデータなど|説明|
|----|------|------------------------|----|
|LinBytesPerScanLine|dw||bytes per scan line for linear modes|
|BnkNumberOfImagePages|db||number of images for banked modes|
|LinNumberOfImagePages|db||number of images for linear modes|
|LinRedMaskSize|db||size of direct color red mask (linear modes)|
|LinRedFieldPosition|db||bit position of lsb of red mask (linear modes)|
|LinGreenMaskSize|db||size of direct color green mask (linear modes)|
|LinGreenFieldPosition|db||bit position of lsb of green mask (linear modes)|
|LinBlueMaskSize|db||size of direct color blue mask (linear modes)|
|LinBlueFieldPosition|db||bit position of lsb of blue mask (linear modes)|
|LinRsvdMaskSize|db||size of direct color reserved mask (linear modes)|
|LinRsvdFieldPosition|db||bit position of lsb of reserved mask (linear modes)|
|MaxPixelClook|dd||maximum pixel clock (in Hz) for graphics mode|

##### その他
|名前|大きさ|格納されているデータなど|説明|
|----|------|------------------------|----|
|Reserved|db|189dub?|remainder of ModeInfoBlock|

##### linear framebufferに対応しているかの確認
linear framebufferに対応していると，ビデオRAMのすべてのメモリが一列に並びます．つまり，ディスプレイのどのピクセルもこのビデオRAMのどこかしらに対応しています．linear framebufferに対応していない場合，複数のbankというものに区分けされ，時々bankを切り替える必要があります．

VBEモードがlinear framebufferに対応しているかは，VBEモードの情報の中の`attributes`の第7ビットが1になっているかで確認します．これが1ならlinear framebufferに対応しています．

```asm
MOV AX,WORD[ES:DI]
AND AX,0x80
CMP AX,0x80
JNE next_mode
```

##### packed pixelかどうかの確認
packed pixelというのは，各ピクセルとメモリをバイト単位で結びつけるのではなく，ビット単位で結びつけます．例えば4ビットカラーならば，1ピクセルを1バイト中の4ビットと結びつけ，残りの4ビットを使用しないのではなく，1バイトを2ピクセルと対応付け，前半後半の4ビットをぞれぞれ1ピクセルと対応付けます．つまり隙間を作らないという意味で*packed*ということです．

VBEモードがpacked pixelかどうかは構造体の`memory_model`の値を確認することで判別します．
```asm
MOV AX,WORD[ES:DI+27]
CMP AX,4
JE valid_mode
```
`[ES:DI+27]`の27というのは，`memory_model`が構造体の先頭から27バイト離れているためです．もしこの値，すなわち`AX`の値が4ならば，使用するVBEモードの候補として有効なため，`valid_mode`ラベルに飛ばします．

##### ダイレクトカラーかどうかの確認
ダイレクトカラーかどうかは，`memory_model`の値が6かどうかを確認することで判別します．
```asm
CMP AX,6
JE valid_mode
```
既に`memory_model`の値は`AX`レジスタに格納しているので，ここでは`MOV`の必要はありません．

最後に，もしpacked pixelでもなくダイレクトカラーでもなければ，次のモードを候補とします．
```asm
JMP next_mode
```

ところで，packed pixelかどうか，そしてダイレクトカラーかどうかの確認については，[OSDev Wiki](https://wiki.osdev.org/VESA_Video_Modes)を参考にしました．

##### 解像度の比較
packed pixelかダイレクトカラーだと確認できれば，次は解像度の比較を行います．まずラベルを貼ります．
```asm
valid_mode:
```

解像度の横，縦はそれぞれ構造体の`width`，`height`に格納されています．構造体の先頭からの距離はそれぞれ18バイト，20バイトです．

```asm
MOV AX,WORD[ES:DI+18]
CMP AX,WORD[SCRNX]
JB next_mode

MOV AX,WORD[ES:DI+20]
CMP AX,WORD[SCRNY]
JB next_mode
```

もし横，縦のどちらかでも，それまでの候補のVBEモードよりも小さければ次の候補を検査します．

##### 色数の比較
先程述べたように，24ビットカラーか32ビットカラー以外は選択肢から外します．色数は構造体のbppに格納され，先頭から25バイト離れています．
```asm
CMP BYTE[ES:DI+25],24
JB next_mode
```

##### 候補の更新
見事，今までの選考基準を通過したVBEモードが登場した場合，解像度や色数に関する情報を更新します．

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
モードナンバの配列のポインタを更新し，ループの先頭へ飛びます．
```asm
MOV AX,WORD[ES:VMODE_PTR]
ADD AX,2
MOV WORD[ES:VMODE_PTR],AX

JMP select_mode
```

#### ループ終了後の処理
まずラベルを追加します．
```asm
finish_select_mode:
```

##### 解像度と色数の確認
解像度や色数が初期値のままの場合，使用可能なVBEモードが存在しないので320x200を使用します．
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
まずラベルを貼ります．
```asm
set_vbe_mode:
```

VBEモードを登録します．以下の情報は[規格書](http://www.petesqbsite.com/sections/tutorials/tuts/vbe3.pdf)のPage40からです．
>**Function 02h - Set VBE Mode**
>
>**Input:**
>
>AX = 4F02h Set VBE Mode
>
>BX = Desired Mode to set
>
>ES:DI = Pointer to CRTCInfoBlock structure
>
>**Output:**
>
>AX = VBE Return Status
>
>**Note:** All other registers are preserved.

`BX`レジスタの値は以下の表のとおりです．

|第nビット|値|説明|
|---------|--|----|
|0-8||Mode number|
|9-10||Reserved (must be 0) |
|11|0|Use current default refresh rate|
|11|1|Use user specified CRTC values for refresh rate|
|12-13||Reserved for VBE/AF (must be 0) |
|14|0|Use windowed frame buffer model|
|14|1|Use linear/flat frame buffer model|
|15|0|Clear display memory|
|15|1|Don't clear display memory|

この関数が失敗する場合もあります．その場合，320x200を使用します．

```asm
MOV AX,0x4F02
MOV BX,WORD[VBEMODE]
OR BX,0x4000
INT 0x10

CMP AX,0x004F
JE keystatus
```

表の通り，`BX`レジスタの第14ビット目は，linear framebufferを使用するか否かについてのビットであり，このビットが1だと使用することを表明します．`OR BX,0x4000`というコードはそのためです．

##### 320x200を使用する時の処理
本中のP280のコードを使用していますが，ラベルは`scrn320`から`screen_320`に，`VMODE`は`BPP`に変更しています．
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

### Cファイルの書き換え

#### 基本的なこと
今まではビデオRAMの特定の番地に色の番号を格納していましたが，ダイレクトカラーを使用する時は番号ではなく構成要素の強さを格納します．今回使用する24ビットカラーや32ビットカラーでは赤，緑，青の色の強さを格納します．

例として，本中のP94にある，`boxfill8`関数を改造します．元の関数は以下の通りです．
```c
void boxfill8(unsigned char *vram, int xsize, unsigned char c, int x0, int y0, int x1, int y1)
{
    int x, y;
    for (y = y0; y <= y1; y++){
        for (x = x0; x <= x1; x++)
            vram[y * xsize + x] = c;
    }
    return;
}
```
書き換え後は以下の通りです．
```c
void boxfill8(struct BootInfo* boot_info, unsigned c, int x0, int y0, int x1, int y1)
{
    int x, y;
    int xsize = boot_info->scrnx;
    char* vram = boot_info->vram;
    char vmode = boot_info->vmode;

    for (y = y0; y <= y1; y++) {
        for (x = x0; x <= x1; x++) {
            int idx = (y * xsize + x) * vmode / 8;
            vram[idx] = (c & 0x0000ff);
            vram[idx + 1] = (c & 0x00ff00) >> 8;
            vram[idx + 2] = (c & 0xff0000) >> 16;
        }
    }
}
```
まず，改造前では引数`c`の型が`unsigned char`となっていたのが，改造後は`unsigned`となっているところに気をつけてください．改造前ではこの引数には色のインデックスを代入していましたが，改造後は色のRGBの値を代入します．

次に`for`文中で，ビデオRAMに値を格納しているところに注目してください．まず，格納する番地です．`vmode / 8`を掛けているところに注意してください．これは1つのピクセルを表すのに必要なバイト数です．RGB値はB,G,Rの順番で，分解して格納します．R,G,Bの順番ではないことに注意してください．

このように，24ビットカラーや32ビットカラーを使用する時は，ビデオRAMに値を格納する処理すべてにおいて変更が必要となります．

#### 320x200への対応
関数の失敗や，使用可能なVBEモードが存在しないと言った理由で320x200を使用する場合があります．このような場合，ダイレクトカラーではなくインデックスカラーを使用することになります．従ってRGB値をインデックスに変換するなど，何かしらの処理が必要となります．こちらの説明については長くなってしまうので省略させていただきます．

## トラブルシューティング
上手く行かない時は，QEMUでメモリの特定の番地の値を確認するといいかもしれません．QEMUでのメモリの値の確認はこちらの「[QEMUでメモリの内容を見る](http://niwatolli3.hatenablog.jp/entry/2015/06/01/201341)」を参考にしました．

ここでは，色数などの情報を確認します．QEMUを使用してOSを起動したあと，`Ctrl+Alt+2`でQEMUのターミナルを開いてください．

起動したあと，以下のコマンドを入力してください．

```
xp /10xb 0x0ff2
```
<figure class="figure-image figure-image-fotolife" title="xp /10xb 0x0ff2を打ったあとの画面（拡大しています）">[f:id:tokuchan3515:20191222140304p:plain:alt=xp /10xb 0x0ff2を打ったあとのQEMUのスクリーンショット]<figcaption>xp /10xb 0x0ff2を打ったあとの画面（拡大しています）</figcaption></figure>

はじめの1バイトは色深度で，この場合は`0x18`すなわち24ビットカラーです．1つ飛ばして次の2バイトは解像度の横の長さで，この場合は`0x0a00`すなわち2560ピクセルです．リトルエンディアンなのではじめのバイトが下位，次のバイトが上位であることに注意してください．その次は解像度の縦の長さで，`0x0640`すなわち1600ピクセルです．表示される色がおかしい場合，はじめの1バイトの値が`0x18`や`0x20`ではないかもしれません．

## 終わりの言葉
はりぼてOSを24ビットカラーや32ビットカラーに対応させることができました．また解像度も大きくしたので，そのうち背景画像を設定できるようになるのではと思っています．

[https://github.com/toku-sa-n/ramen:embed:cite]

僕が製作中のOSです．Rustで書かれていますが，もしよければ見ていってください．

## 参考文献

[https://ja.wikipedia.org/wiki/%E8%89%B2%E6%B7%B1%E5%BA%A6:title]

[http://oswiki.osask.jp/?cmd=read&page=VESA&word=vesa:title]

[https://wiki.osdev.org/VESA_Video_Modes:title]

[https://graphicdesign.stackexchange.com/questions/47133/is-32-bit-color-depth-enough:title]

[https://ja.wikipedia.org/wiki/VESA_BIOS_Extensions:title]

[http://oswiki.osask.jp/?%28AT%29memorymap:title]

[https://wiki.osdev.org/Memory_Map_(x86):title]

[http://niwatolli3.hatenablog.jp/entry/2015/06/01/201341:title]
