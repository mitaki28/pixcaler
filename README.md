## Pixel-Art Upscaler(仮)

* Demo： http://mitaki28.info/pixel-art-upscaler/

このコードは[chaienr-pix2pix](https://github.com/pfnet-research/chainer-pix2pix)をベースにして作成されています。

<img src="https://github.com/mitaki28/pixel-art-upscaler/blob/master/image/example.png?raw=true">

(変換元素材: [白螺子屋](http://hi79.web.fc2.com/)様, 学習データ: [カミソリエッジ](https://razor-edge.work/material/fsmchcv/) 様【オリジナルの素材を配布していたのは First Seed Material 様（サイト閉鎖）】）

32x32〜16x16程度のキャラチップを前提としたドット絵の拡大ツールです。

[既存の手法](https://en.wikipedia.org/wiki/Pixel-art_scaling_algorithms)よりもリアルな拡大が可能ですが、画像が歪んだり不自然な拡大がされることも結構あり、まだ実験段階です。

いわゆるディープラーニングと呼ばれる技術を用いて実装されており、[pix2pix](https://arxiv.org/abs/1611.07004) というネットワーク構造をベースにしています。実装は[chainer-pix2pix](https://github.com/pfnet-research/chainer-pix2pix)を改造して制作しました。

[こちら](https://razor-edge.work/material/fsmchcv/)で配布されている First Seed Material 様の素材（高解像度版）約7000枚を用いて学習しています。</p>

### 環境構築
* Python 3.5 が必要です

```
python3 -m venv venv
source venv/bin/activate
pip -r requirements.txt
```

* GPU を利用する場合は手動で cupy を追加インストールしてください
```
pip install cupy
```

### 既存の実装+alpha
いろいろ変えてみましたが、実際のところ、 [chainer-pix2pix](https://github.com/pfnet-research/chainer-pix2pix) そのままで、128x128 にアップスケールして学習してもそこまで結果は変わってないですが、感覚的にこちらのほうが安定しているような感じはしました。（もともとの chainer-pix2pix を使ったときは Generator が Discriminator に振り回されて l1-loss の収束率が悪い感じがしました。）

* 最近、良いと言われている手法をいくつか取り入れています
    * Deconvolution2D を [Nearest-Neighbor ResizeConvolution](https://distill.pub/2016/deconv-checkerboard/) に換装(効果があるかは微妙)
    * Generater/Discriminater の loss 関数を [LSGAN](https://arxiv.org/abs/1611.04076) に変更(効果があるかは微妙)
        * 同時に lam1 倍率を100→10に変更してます（経験上、lsgan に換装すると loss は10分の1ぐらいにスケールされる）
        * 同じ人が作成した [CycleGAN](https://github.com/junyanz/CycleGAN) でも採用されているより安定性の高い loss 関数
* pix2pix ネットワークの encoder, decoder の最上段を kernel size 5x5, stride 1, padding 2 の Convolution2D に換装（効果あり; l1-loss の減少が安定する）
    * もとのネットワークでは画像サイズが128x128以上ないと、画像幅が足りずエラーになります
    * そこで、最上段を5x5のConvolution2D(縮小なし)に換装しました
    * 3x3 ではなく5x5 なのは [既存の手法](https://en.wikipedia.org/wiki/Pixel-art_scaling_algorithms) が5x5のconvolutionをベースとしていたことや、より広い範囲を見たほうが、そのドットのコンテキストを推論しやすいだろうという予想のもとです
* もとの画像を以下の手順で処理して学習しています
    * 点(0, 0) の色を透明色とみなし、該当する色を(0, 0, 0, 0)に変更します
    * 80x80 になるように均等に padding を入れます
    * 画像を平行移動（64x64のcrop）と鏡像反転して、データを水増しします
        * 平行移動はおそらく重要です
            * nearest neighbor 縮小は性質上、1枚の画像に対して、4種類の結果が存在します（縮小時に4x4格子のどの点を取るかの自由度があるため）
                * <img src="https://github.com/mitaki28/pixel-art-upscaler/blob/master/nn-scales.png?raw=true">
            * 画像をランダムに平行移動することによって、この4種類の結果をすべて学習データセットに加えることができ、実質的に画像を4倍に水増しできます
            * さらに、目などの細かいピクセルの模様も nearest neighbor 縮小のいずれかのパターンではきれいに残っていること多いので、学習を繰り返すことで、適切な復元方法にたどり着ける可能性が上がります
    * nearest neighbor 法(PIL.Image.NEAREST_NEIGHBOR によるリサイズ)で縮小し、再度64x64に nearest_neighbor 法で拡大したものを変換元、もとの画像を変換先として学習します

#### その他
* batchsize はマシンスペックに余裕があっても敢えて1にすべきです(効果あり; batchsize=4のときと比較して l1-loss の収束に0.5(lam1=100 のとき)程度の差がありました)
    * ただし、現状、 chainer のバグ？で、 batchsize=1 のとき、一部の BatchNormalization の重みが nan になってしまい、 test モードでの学習ができなくなるようです
        * Web アプリ版のモデルでは、 nan になった重みを無理やり 0 に補正しているため、精度が落ちているように見えます
    * batchsize=1 のときの BatchNormalization は InstanceNormalization と等価になり、性質が変わるとのことです
        * https://github.com/junyanz/pytorch-CycleGAN-and-pix2pix/issues/27

