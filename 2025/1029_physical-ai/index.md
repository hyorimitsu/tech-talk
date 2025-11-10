Physical AI に入門してみた ~ VLA モデルで「見る・理解する・動く」を試す ~
---

生成 AI の進化が止まりません。あらゆる領域で活用が進み、ソフトウェアエンジニアの仕事はこの先どうなるのか？と考える機会が増えてきました。

現状では、複雑な開発タスクにはまだ人間の判断やコントロールが不可欠ですが、モデルの性能だけでなく、周辺エコシステムも急速に発展しており、自分のキャリアや関わり方についても考えさせられます。

そんな中で耳にしたのが、Physical AI という分野です。ロボットが物理世界で知覚・判断・動作することを目指しているものの、汎用 AI という観点ではまだ発展途上とのことで、気になったので遊んでみました。

とはいえ私自身は Physical AI はおろかロボット関連の知識がほぼ皆無なので、あくまで入門レベルの内容になります。また、内容に誤りがある可能性もあるのでその点はご留意ください。


## VLA (Vision-Language-Action) モデルとは

VLA モデルは、視覚入力 (画像) 、言語指令 (テキスト) 、行動出力 (アクション) を統合的に扱う AI モデルで、簡単に言えば「見て、理解して、動く」を一体で学習します。  
代表的なものとしては以下があります。

| モデル名          | 特徴                                                                                                                       |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------- |
| **[RT-2](https://robotics-transformer2.github.io)**          | Web 規模の視覚言語データとロボット軌跡データを組み合わせて学習。ロボットの動作をテキストトークンとして出力する形式を採用。 |
| **[OpenVLA](https://github.com/openvla/openvla)**       | 約 7B パラメータのオープンソース VLA。約 970k のロボット実演データを用いて学習され、研究・教育用途でも扱いやすい。         |
| **[π₀ (pi-zero) ](https://github.com/Physical-Intelligence/openpi)** | VLM を中核に据え、flow-matching による連続アクション生成を組み込んだ VLA。複数のロボット機構を横断して汎用的に動作。       |

今回はこの [π₀ モデル](https://github.com/Physical-Intelligence/openpi)を使って、実際にシミュレーション環境上で試してみました。


## 環境構築

セットアップ手順については以下をご参照ください。

[Quick Start (Local or Google Cloud VM)](https://github.com/hyorimitsu/physical-ai-sandbox?tab=readme-ov-file#quick-start-local-or-google-cloud-vm)

環境構築後は、`pi0` 環境を有効化してすぐに [OpenPI](https://github.com/Physical-Intelligence/openpi) の動作を確認できます。


## OpenPI を動かしてみる

### 動作確認編

[OpenPI](https://github.com/Physical-Intelligence/openpi) は、自然言語によるタスク指示からロボットの動作を生成するためのフレームワークです。  
まずは `pi0_libero` モデルを使用して、「pick and place the cube (キューブを掴んで置く) 」という指示を与えてみます。

```python
from openpi.shared import download
from openpi.policies import policy_config
from openpi.training import config as cfg
import numpy as np

# モデル設定とチェックポイントをロード
conf = cfg.get_config("pi0_libero")
ckpt = download.maybe_download("gs://openpi-assets/checkpoints/pi0_libero")

# 学習済みポリシーの作成
policy = policy_config.create_trained_policy(conf, ckpt)

# ダミー観測を入力 (実際にはカメラ画像・手先画像・状態ベクトルなどが入る)
sample = {
    "observation/image": np.zeros((224, 224, 3), dtype=np.uint8),
    "observation/wrist_image": np.zeros((224, 224, 3), dtype=np.uint8),
    "observation/state": np.zeros((8,), dtype=np.float32),
    "prompt": "pick and place the cube",
}

# 推論 (行動の生成)
out = policy.infer(sample)
print("actions:", out["actions"].shape)
```

出力：

```
actions: (50, 7)
```

これは、7 次元ベクトル (ロボットの姿勢＋グリッパ状態など) で構成された 50 ステップ分のアクション列を表します。  
つまり、AI がこのタスクに対してどう動けばいいかを 50 ステップ分だけ考えた結果です。


## 応用編

応用編として、[OpenPI](https://github.com/Physical-Intelligence/openpi) を [LIBERO](https://github.com/Lifelong-Robot-Learning/LIBERO) 環境で試してみました。  
詳しいコードについては以下をご参照ください。

[Physical AI Sandbox — OpenPI + LIBERO Integration](https://github.com/hyorimitsu/physical-ai-sandbox/tree/main/src)

### 実行結果

以下は、実際に [OpenPI](https://github.com/Physical-Intelligence/openpi) + [LIBERO](https://github.com/Lifelong-Robot-Learning/LIBERO) 環境で実行した結果の動きです。

最初は物体に関心を示す様子すらなかったのですが、いろいろ調整してみた結果、ようやく物体に触れるところまでは進みました。  
それでも私の環境では、残念ながら掴むことまではできず、目標物の手前で空振りしてしまいました。

<div style="text-align:center">
  <video src="https://github.com/user-attachments/assets/b41e7a88-4320-4985-9545-cab90c7c8357" width="80%" controls="true"></video>
</div><br>

観測のずれなどの影響で意図した動きにならないのだろうかと考察しつつ、AI が視覚・言語・行動を統合的に扱って物体にアプローチするプロセスは体験できました。


## まとめ

依存関係などの問題でセットアップに多少ハマったりはしましたが、Hello World レベルであればシミュレーションを実行するところまでは比較的簡単にできることがわかりました。  
一方で、観測のノイズや環境の不確実性などの要因もあり、正確なアクションに落とし込むことの難しさも体感できました。

それでも、視覚・言語・行動を一体で扱うアプローチは非常に面白く、今後どのように発展していくのかが非常に楽しみです。

将来的には、クラウド上のポリシーサーバを通じて学習済みモデルを呼び出し、ロボットをまるで API のように扱える時代が来るかもしれません。


## 参考リンク

- [Physical AI | NVIDIA](https://www.nvidia.com/en-us/glossary/generative-physical-ai)
- [RT-2: Vision-Language-Action Models | GitHub Pages](https://robotics-transformer2.github.io)
- [RT-2: New model translates vision and language into action | Google DeepMind](https://deepmind.google/blog/rt-2-new-model-translates-vision-and-language-into-action)
- [openvla/openvla | GitHub](https://github.com/openvla/openvla)
- [OpenVLA: An Open-Source Vision-Language-Action Model | GitHub Pages](https://openvla.github.io)
- [Physical-Intelligence/openpi | GitHub](https://github.com/Physical-Intelligence/openpi)
- [Physical Intelligence Blog | Physical Intelligence](https://www.physicalintelligence.company/blog)
- [π0 and π0-FAST: Vision-Language-Action Models for General Robot Control | Hugging Face](https://huggingface.co/blog/pi0)
- [Lifelong-Robot-Learning/LIBERO | GitHub](https://github.com/Lifelong-Robot-Learning/LIBERO)
- [LIBERO Project | GitHub Pages](https://libero-project.github.io/main.html)
