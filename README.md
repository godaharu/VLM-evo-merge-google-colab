# VLM-evo-merge-google-colab

**動機**

[**進化的アルゴリズムによる基盤モデルの構築 (sakana.ai)**](https://sakana.ai/evolutionary-model-merge-jp/)

Sakana AIが既存の基盤モデル同士をマージすることによって、日本語の基盤モデルでの最先端の性能を達成しています。進化的モデルマージという手法を使い、言語モデル同士や画像言語モデルと言語モデルを組み合わせて、さらに性能の良い新たな基盤モデルの作成を行っています。この手法の利点として、通常の基盤モデルの学習方法と比べ、計算リソースが少なく済む点が挙げられます。そのため、個人レベルでも新たな基盤モデルを作ることができる手法として期待されています。

[GitHub \- arcee-ai/mergekit: Tools for merging pretrained large language models.](https://github.com/arcee-ai/mergekit)

Sakana AIからはマージを行うためのコードは提供されていませんが、Mergekitというツールには、Sakana AIの研究にインスパイアされたmergekit-evolveという言語モデルの進化的モデルマージを簡単に実装できる機能が搭載されています。この機能を使えば、configファイルにhugging faceのモデルパスを指定するだけで言語モデルの進化的モデルマージを行うことができます。ただし、日本語性能の良い言語モデルを作る場合、日本語のベンチマークを使用する必要があり、タスクの設定を手動で記述しなければなりませんが、既に  
・[Mergekit-Evolve登場！進化的アルゴリズムで手元のLLMを最強進化させよう！ – soy-software (soysoftware.sakura.ne.jp)](https://soysoftware.sakura.ne.jp/archives/3872)  
・[Google Colab で mergekit-evolve による 進化的モデルマージ を試す｜npaka](https://note.com/npaka/n/n42129c043026)  
こちらの記事の方々が実装し、マージに成功しています。

また、進化的モデルマージの論文では、言語モデル同士のマージだけでなく、画像言語モデルと日本語言語モデルのマージも行っており、これにより日本語の画像QAのベンチマークで最先端の性能を達成しています。しかし、mergekit-evolveでは基本的に言語モデル同士のマージをサポートしていますが、画像言語モデルと言語モデルのマージは2024年6月15日現在は公式にはサポートしていません。そこで、今回はmergekitのコードに追加を行い、画像言語モデルと言語モデルのマージを行うことができる環境を作ることを試みました。

**手法**

公式にはサポートされていないものの、[こちら](https://github.com/EleutherAI/lm-evaluation-harness/pull/1832/files)の投稿を見ると開発途中？なのか、画像言語モデルであるLlavaのサポートを含んだコードが挙げられています。このコードと先程挙げたmeregekit-evolveによる言語モデルの記事、そして画像言語モデルと言語モデルをChatVectorと呼ばれる手法でかけあわせている  
[https://zenn.dev/toshi\_456/articles/0166a6eaa81c7b](https://zenn.dev/toshi_456/articles/0166a6eaa81c7b)  
の記事をベースにmergekitで画像言語モデルLlavaと日本語言語モデルのマージを行いました。

環境はGoogle Colab A100GPUを使用しました。  
全体のコードは[Llava\_EvoMerge.ipynb](https://colab.research.google.com/github/godaharu/VLM-evo-merge-google-colab/blob/main/Llava_EvoMerge.ipynb)に記載し、デフォルトからの変更点のうち重要なものを下記に載せます。

・LlaVa用の評価タスクの登録  
mergekit-evolveではモデルの評価のために、[lm-evaluation-harness](https://github.com/EleutherAI/lm-evaluation-harness)というライブラリが使用されています。そのため、デフォルトで用意されていないベンチマークでモデルを評価するにはlm-evaluation-harnessに手動でタスクを登録する必要があります。今回は日本語の言語モデル評価のタスクを登録している[こちら](https://soysoftware.sakura.ne.jp/archives/3872)や[こちら](https://note.com/npaka/n/n42129c043026)の記事や公式の[new\_task\_guide](https://github.com/EleutherAI/lm-evaluation-harness/blob/main/docs/new_task_guide.md)を参考に、日本語の画像QAのタスクの登録を行いました。  
使用するデータセットには[JA-VLM-Bench-In-the-Wild](https://huggingface.co/datasets/SakanaAI/JA-VLM-Bench-In-the-Wild)を用いています。

・MergekitのLlaVa対応  
llavaモデル（llava-hf/llava-1.5-7b-hf）を読み込むために、mergekitで指定されているllamaモデルのアーキテクチャを記述したjsonファイルの変更が必要でした。具体的にはmergekit/mergekit/\_data/architectures/llama.jsonの”architectures”にLlavaLlamaForCausalLMを追加しました。これによりmergekit内でllavaの言語モデル部分を読み込むことができると考えています。llavaの言語モデル部分だけを読み込むことで、mergekitを使用して他の言語モデルとのマージを行うことができます。

・Mergekitで作成した言語モデルを用いてlm-evaluation-harnessで画像QAの推論を行う  
上述の対応では、mergekitで作成されたマージモデルが言語モデルとして保持されています。これを画像言語モデルとして使用するために、マージされた言語モデルとしてのパラメータを画像言語モデルにコピーします。具体的にはllavaモデル（llava-hf/llava-1.5-7b-hf）を読み込み、そのモデルの言語部分にマージしたモデルの重みをコピーしていきます。これにより、登録した画像ＱＡのタスクでマージしたモデルの評価を行うことができます。

**結果**

![図1: 進化計算の各反復に応じたbest_score](https://github.com/user-attachments/assets/3a19f35a-56e5-4f7d-9bf2-b6fd7d0e1564)  
図１．進化計算の各反復に応じたbest_score

![図2: 画像VQAの例](https://github.com/user-attachments/assets/4f28faf7-35e2-4c40-b33b-c795dade2a27)  
図２．画像VQAの例

マージ手法はlinearを使用し、画像言語モデルliuhaotian/llava-v1.5-7bと英語モデルmeta-llama/Llama-2-7b-hfと日本語言語モデルelyza/ELYZA-japanese-Llama-2-7bの進化的モデルマージを行いました。Google Colab A100GPUで12時間程度実行しました。best scoreは48.966でした。  
上記の図１は進化計算において、各反復毎にモデルのbest\_scoreが改善されていく様子です。結果から今回記述したタスクやマージの設定が機能し、モデルの性能が上がっていることが確認できます。

続いてscoreの値について見ていきます。  
[こちら](https://zenn.dev/toshi_456/articles/0166a6eaa81c7b)の記事によると、既存の画像言語モデルでのJA-VLM-Bench-In-the-Wildのscoreは１位がSakana AIの51.25で2位が[llava-jp-1.3b-v1.0-620k](https://huggingface.co/toshi456/llava-jp-1.3b-v1.0-620k)の44.58なので、今回の48.966設定でかなり性能の良いモデルができていることが分かります。ただし、使用するデータセットと評価するデータセットが同じなので過学習をしていることに十分注意してください。その状況でもSakana AIの結果を上回ることができないのは、計算時間や進化計算の設定が影響している可能性を考えています。

**まとめ**

画像言語モデルLlavaの進化的マージををmergekitを使用して実行しました。結果から個人レベルの計算資源でもかなり強力な画像言語モデルを作ることができると確認しました。  
次はこれを使って、様々なマージ設定やベンチマークを試す、llama3でマージを行う、別のタスクの特化型モデルを作る　等をしてみたいと思います。
