
これは、東プレのカスタムキーボードを作るための不完全なガイドであり、具体的には
PCB、プレート、ファームウェアを紹介しています。これらのプロジェクトを通して得られた情報
の蓄積です。
[カスタム東プレボードのデザイン](https://deskthority.net/workshop-f7/designing-a-custom-topre-board-t11734.html),
[分割HHKB - Realforce/TypeHeaven mod](https://deskthority.net/workshop-f7/another-custom-split-hand-topre-board-need-your-input-t14769.html).

不明な点や追加すべき点があれば教えてください。

始まりのきっかけを作ってくれた [hasu's research](https://github.com/tmk/tmk_keyboard/tree/master/keyboard/hhkb/doc)
に感謝します。

**目次**

1. [回路図](#circuitry)
    1. [基本回路図](#basic-schematic)
    2. [ドレインピン](#drain-pin)
    3. [実用上の注意点](#practical-considerations)
        1. [寄生容量](#parasitic-capacitance)
        2. [その他の注意事項](#other-notes)
    4. [KiCad ファイル](#kicad-files)
2. [ハードウェア (ケース/プレート)](#hardware-caseplate)
    1. [穴のサイズ](#hole-sizes)
    2. [Realforceに関する注意事項](#warning-about-realforce)
3. [ファームウェア](#firmware)
    1. [基本的な読み取り方法](#basic-read-procedure)
    2. [標準化](#normalisation)
    3. [キャリブレーション](#calibration)
        1. [概要](#overview)
        2. [キャリブレーションの手順](#calibration-procedure)
        3. [警告](#a-warning)
    4. [深さの処理](#handling-the-depths)
        1. [Digital conversion](#digital-conversion)
        2. [Analog](#analog)
    5. [Example](#example)


# 回路図

## 基本回路図

私がキーの押し下げを検知するために使用している方法は、とてもシンプルです。
私が行ったテストでは、ファームウェアでキャリブレーションを行って測定値を正規化することで、
うまく機能しました。

東プレのキーボードのマトリックスクロスポイントは、基本的に
「ストロボ」ラインと「リード」ラインをつなぐ可変コンデンサーです。
ストロボラインとリードラインはPCB上に直接電極を形成し、
ドームの下にある円錐形のスプリングがそれらを結合して、0〜6pF（大体）の範囲の
可変コンデンサーを作り出しています。
ストロボラインは、マイコンのデジタル出力端子からのデジタル信号です。
読み取り線は、以下の回路図で扱われています。

![Schematic](images/schematic.png "基本回路図")

各読み取りラインは，個別の22k抵抗でグランドにプルされ、アナログマルチプレクサに供給されます。
マルチプレクサで読み取りラインを選択した後、マイクロコントローラが列をストローブすると、
選択された読み取りラインに小さな電圧パルスが現れ、パルスが大きいほどキーの押し下げが
大きくなります。

選択されたリードラインは、コンデンサC1に接続されており、これによりリードラインは単純なRC減衰回路のように動作します。
その値は、次の式を与えて選択することができます。

```
                            センシングしているキーの静電容量
ピーク出力電圧 = 入力電圧 * -----------------------------------
                                  全行のキャパシタンス
```

ここでは，入力電圧は `Vdd` です。全列の静電容量（対地）は、C1とその列のすべてのキーの
静電容量で構成されています。そのため、C1を（キーの静電容量に比べて）大きな値にすることは、
行の他のキーが押されても読み値が大きく変化しないようにするために重要であることは明らかです。

しかし、単にC1を大きくするだけでは、ピーク出力電圧が下がり、結果的にノイズレベルが
高くなってしまいます。私は1nFが適当だと思います。

ドレイン端子は無視して、リードラインは電流制限抵抗を経て、非反転アンプに入ります。
このアンプの目的は、きれいな信号を0〜3.3Vの範囲に戻すことです。ゲインは「1 + R2 / R4」で
与えられ、ここでは約200となっています。

また、ストロボラインがグランドに戻ったときに発生する負の電圧からマイクロコントローラーを
保護する役割もあります。アンプの出力は、マイクロコントローラのADC端子に接続します。

## ドレインピン

選択されたリードラインがRC回路を形成している場合、グランドにリラックスするまでの時間は、
単純に`5 * RC時定数`に支配されることがわかります。
この時定数は単に`R * C`であり、この例ではリラックスタイムの合計は
`5 * 22k Ohm * 1 nF ~ 100 us`となります。
次のキーを読み出す前にマトリックスがリラックスしてグランドになるのを待たなければならないことを
考慮すると、これはキーボードのキー1つにつき100秒かかることになり、10キー以上のキーボードの
場合、ポーリングレートは1000Hz以下になります。この問題を解決するには、電流制限用の1k抵抗R1を
介して、読み取りラインをマイクロコントローラのピンに接続します。
このピンは、ストローブと読み取りプロセスの間はフローティングにしておきますが、
読み取り値をADCに取り込んだ後（約5秒かかります）は接地して、並列抵抗の公式に従って抵抗Rを
小さくします。

```
  1     1       1
 --- = ---- + -----
  R     R1     22k
```

which gives `R ~ 950 Ohms` for our chosen values. Recalculating the relax time
now gives `5 * 950 Ohms * 1 nF ~ 5 us`. This would allow 1000 Hz polling for
even 100 key keyboards.

## Practical considerations

## Parasitic capacitance

When routing any analog lines (and to some extent the digital strobe lines)
care must be taken to reduce parasitic capacitance due to the low signal level.
The lines must be surrounded by a ground pour and have a ground plane on
surrounding layers. Crossing tracks should ideally occur with a ground plane
between them, but this requires a 4 layer PCB. 2 layers is perfectly fine, as
long as you ensure traces cross at right angles and do so as little as
possible. Basically just don't run the strobe lines (or other digital lines)
close to the analog lines where you can avoid it! Study the Topre PCBs to see
the careful routing of the matrix.

## Other notes

Any unused inputs of the multiplexer should be grounded to prevent additional
sources of noise: this goes for any unused op amp pins too. If they are not
connected, ground them.

I found it important to use a very fast amplifier, opting for the OPA350A.
Cheaper options proved to be too slow, turning the voltage spike into more of a
voltage mound, making reading unpredictable.

## KiCad files

See `kicad` folder, it contains an example switch footprint and schematic
library file.

![Topre footprint](images/topre-pad.png)

# Hardware (case/plate)

The stackup is fairly simple, the spacer size is determined by the housing
dimensions:

```
1.2 mm thick steel plate
3/16 inch (length) spacers
PCB
```

The PCB must be securely fastened to the plate - the force from keypresses (and
the force of pressing the keycaps onto the sliders) is transferred to the PCB,
not the plate. Making sure there are enough fastening points is also important
to keep the housings firmly sitting on the PCB and holding the domes down, or
you may end up with a "squishing" sound when keys are at the bottom of the
stroke. Put some fasteners in the central sections of the keyboard to keep it
rigid - copy Topre if in doubt.

## Hole sizes

The sizes shown in this section are for Topre keyboards - not clones! Clones
seem to be 14 x 14 mm square for 1U keys. They also require more consideration
of fastener positions since they don't have the semicircular cutouts for bolts
to pass through.

Single unit wide key housings require a hole of size 14.6 mm x 14 mm, they are
wider horizontally. The corners can be chamfered 1 mm along each edge, as Topre
does.

Double unit key housings (backspace, shift keys, ANSI enter key) should be 32.4
mm x 14 mm. I made mine 32.6 mm but they are slightly too large. The bottom
edge can have chamfers in the corners, like Topre does: around 3 mm along the
bottom edge with a 30 degree angle.

![Example DXF Screenshot](images/dxf-example.png "Example DXF Screenshot")

See example-split-hhkb.dxf for example sizes (The double unit keys are too wide
though, as I mention above).

## Warning about Realforce

Realforce keyboards (I don't know if this is true of all of them, but certainly
the ANSI 55g 87u that I have) have a strange bottom left control key. The stem
is rotated by 90 degrees, and so a housing which can hold this keycap must be
rotated as well. Since they are not square this is a problem! If you want to
support moving the control key around (swapping with alt, for example) both
should be made 14.6 x 14.6 mm. The PCB will hold the housing when the keyboard
is built.

# Firmware

You will need to modify a firmware quite substantially to get your keyboard
working. Your initial choice depends on your chosen microcontroller and
preferences in firmware. You need access to an ADC and some EEPROM memory in
order to store calibration values: at least 2 bytes per key, if the stored
values take 1 byte each (uint8_t). Using something like a Teensy is perfectly
fine, the initial work I did was using a Teensy 3.1/3.2.

## Basic read procedure

Global variables
```
relaxTime (5 * R * C, include drain pin if using)
state[num reads, num strobes] a structure containing:
    depth   (current depth of key)
    pressed (whether the key is currently "pressed", see digital conversion)
```

Matrix scan
```
for each read:
    select read line on multiplexer
    delay for relaxTime
    for each strobe:
        value <- strobeRead(strobe)
        depth <- normalise(strobe, read, value)
        state->depth[read, strobe] <- depth
```

Read value on ADC
```
strobeRead(strobe):
    static lastTime
    wait until lastTime + relaxTime < current time
    float drain pin
    disable interrupts (could throw timing off)
    set strobe high
    value <- read on ADC
    set strobe low
    enable interrupts
    ground drain pin
    lastTime <- current time
    return value
```

The values which come out of the strobeRead function are assumed to be 8 bit
unsigned integers, that is, 0 to 255. This directly correlates to the voltage
of the read line, and also the capacitance of the key. Thanks to Topre's method
of using a conical spring, the capacitance of the key is linear with key
depression. So the output of strobeRead can be taken as the key depth!

Of course it isn't quite so simple, because each key will have an offset value
from ground and a different peak value when the key is fully depressed. As
such, we need to be able to rescale each key to find the true depth. For
example, if a key is reading on average 34 when unpressed and 200 when pressed,
we must rescale this into the range [0, 255]. These values should be stored in
EEPROM for each key, and can be determined by the [calibration
procedure](#calibration).

## Normalisation

See [this wikipedia article](https://en.wikipedia.org/wiki/Feature_scaling#Rescaling).
In the example procedure, we are rescaling to the range [0, 255].

Example procedure for uint8_t values, with no floating point operations
```
uint8_t normalise(strobe, read, uint8_t value):
    (calLow, calHigh) <- calibrationValues(strobe, read)
    // clamp to minimum and maximum values
    if (value < calLow)
        value <- calLow
    else if (value > calHigh)
        value <- calHigh

    uint16_t numerator   <- 0xFF * (value - calLow)
    uint8_t  denominator <- calHigh - calLow

    return (uint8_t) (numerator / denominator);
```

## キャリブレーション

### 概要

1つのキーに対するADCからの繰り返しの読み取りは、次のようになります。

![ADC Plot](images/adcplot.png "ADC Plot")

これは、ADCの測定値が作動深さの上で交差し、後にリリース深さの下で交差することで、
完全なキーのプレスとリリースを示している。
キーが押されていないときの平均値と、キーが押されたときの平均値を求めることで、
値を簡単に正規化することができます。
プロットに記されているHighMax、LowMax、LowMinの値は簡単に測定でき
（[calibration procedure](#calibration-procedure)を参照）、
必要な平均値を算出することができます。我々は、ノイズを

```
noise = lowMax - lowMin
```

となり、S/N比は

```
       highMax - lowMax
SNR = ------------------
             noise
```

これは、キーがどの程度優れているかを読み取るのに便利です。
一般的には、少なくともSNR>10を求めていますが、上記のデザインでは20〜30の間に
収まることが多いようです。

lowAvgとhighAvgの値は単なる

```
          lowMax + lowMin
lowAvg = -----------------
                 2
```
```
                     noise
highAvg = highMax - -------
                       2
```

これらは正規化ルーチンのためにEEPROMに保存する必要があります。

### キャリブレーションの手順 

まず、lowMaxとlowMinの値を決定します。これは簡単です。
マトリックスをスキャンしている間、キーボードに手を触れずにしばらく放置します
（数秒で十分です）。
各キーについて、最高と最低の測定値を記録し、これがそれぞれlowMaxとlowMinになります。

highMaxを見つけるには、ユーザーが各キーを順番に押している間にマトリックスをスキャンして、
最高の測定値を記録します。

基本的な校正ルーチンとしてはこれで十分であり、安定した測定を行うためにはこれだけで
十分であることがわかりました。
これを改善するには、より高度な信号処理方法による継続的な再校正を導入するとよいでしょう。

### 警告

メインのキー検出ルーチンからのある種の逃げ道があると便利です
（例えば、キープレスをホストに送るのを中断するデジタルボタンなど）。
マトリックスを1000Hzでスキャンしているときに、何かを間違えてしまった場合、
例えばキャリブレーションの値が間違っていて、正気の入力値をランダムに0や255に
正規化してしまった場合、誤って毎秒数百回のキープレスをホストに送信してしまう可能性があります。
また、シリアル接続を介して（例えば`screen`を介して）中断する方法しかない場合、
ひどい目に遭うでしょう。

## 深さの処理

各キーの正規化された深さを収集する方法がわかったので、
それを使って何かをしなければなりません。

### デジタル変換

正規化されたキーの深さをデジタル値に変換する最も基本的な方法は、
キーが目的の作動ポイントよりも深いかどうかをチェックすることです。
これではあまりうまくいかず、ノイズの多い測定値のおかげでホストにキープレスのスパムを
与えてしまいます。
キーの深さの測定にヒステリシスを導入する必要があり、深さと同様にキープレスの状態を
保存することでこれを行います。

1つのキーに対する簡単な手順は以下の通りです。

```
if (not pressed and depth > actuation depth):
    pressed <- true
    send key press signal
else if (pressed and depth < release depth):
    pressed <- false
    send key release signal
```

アクチュエーションの深さは、真ん中あたりを選ぶとよいでしょう。
あまりにも0や255に近づけすぎると、ノイズのある測定値のためにプレスを逃したり、
誤ったプレスをしたりする危険性がありますのでご注意ください。
キャリブレーションの手順でノイズレベルを決定し、それに応じてバッファ値を設定することができます
（安全のために測定されたノイズ値の数倍の値を使用してください）。
予備的には、0～255の範囲で30程度のバッファを設定するとうまくいくと思います。
リリースの深さは、作動の深さからこのバッファ値を引いたものになります。

### アナログ 

理論的には、深度を軸に直接パイプで送ることができます。
コントローラの軸でも、アナログマウスのキーのようなもっと面白いものでも構いません。
しかし、ノイズの多い測定値の場合、デッドゾーンを設ける必要があります。

スプリットキーボードの場合、キープレスだけをデジタルで送るのではなく、
接続を通じて深度情報を共有することが重要です。
これは、マスターがスレーブのアナログコマンドを処理できるようにするためです。

## 例

[Kiibohdの私のフォーク](https://github.com/tomsmalley/controller)、
特にScan/SplitHHKBを参照してください。
Warning: これは完全なものではなく、基本的には壊れていて、しばらくするとクラッシュします。
Topreの部分は問題なく動作していますが、問題なのは私がハックしたインターコネクトの
ソリューションです。
