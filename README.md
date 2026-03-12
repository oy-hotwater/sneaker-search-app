# sneaker-similar-image-search リポジトリ全体の概要
- 「スニーカーの類似画像を検索する画像処理AI」
- のフォルダ群

# **技術仕様書：sneaker_similar_image_search_ver2.ipynb（Drafted by Gemini）**

## **1\. 概要**

本プロジェクトは、Google Colab環境において数万枚規模のスニーカー画像データセットから、特定のクエリ画像に類似した画像を高速かつ高精度に検索するシステムである。従来の線形探索やシングルスレッド処理を排し、最新のGPU並列演算およびベクトルインデックス技術を統合したプロダクション・レディな設計となっている。

## **2\. 技術スタック**

### **2.1 実行環境**

- **Platform:** Google Colab
- **Accelerator:** NVIDIA T4 GPU (or higher)
- **CUDA:** 12.2+ (faiss-gpu-cu12 準拠)

### **2.2 フレームワーク & ライブラリ**

- **Deep Learning:** TensorFlow 2.15+ / Keras
- **Computer Vision:** Pillow, OpenCV (画像検証用)
- **Vector Search:** Meta Faiss-gpu (CUDA 12系最適化版)
- **Data Pipeline:** tf.data API
- **External API:** Kaggle API (データセット自動取得用)

### **2.3 採用モデル**

- **Model:** ResNet50 (Pre-trained on ImageNet)
- **Feature Dimension:** 2,048 dimensions (Global Average Pooling 後の出力)
- **Distance Metric:** Cosine Similarity (L2正規化後の内積計算により実現)

## **3\. システムアーキテクチャ**

### **3.1 データレイヤー (Data Acquisition)**

Kaggle APIを活用し、手動のアップロード作業を排除。Colabのuserdata（シークレット機能）を用いて認証情報を管理することで、ソースコードへのAPIキー直書きを防止。

- **冪等性の担保:** データ存在チェックにより、再実行時の無駄なダウンロードを回避。
- **リソース最適化:** ZIP解凍後の即時削除によるディスク容量の節約。

### **3.2 プロセッシングレイヤー (Robust Pipeline)**

tf.data.Dataset APIを用いた非同期ストリーミング処理。

- **Validation:** 読み込み前に Image.verify() を実行し、破損ファイルによる学習・推論の停止を未然に防止。
- **Parallelization:** num_parallel_calls=AUTOTUNE によりCPUマルチコアで前処理を並列化。
- **Prefetching:** GPUが推論中に、CPUが次のバッチをメモリにロード。

### **3.3 モデルレイヤー (Feature Extraction)**

特徴抽出器としてResNet50を採用。

- **Batch Processing:** model.predict(dataset) により、バッチ単位でGPUへ転送。
- **L2 Normalization:** 抽出した全ベクトルをL2正規化。これにより、以降の検索フェーズで複雑なコサイン類似度計算を単純な内積計算に置換可能にする。

### **3.4 検索レイヤー (Vector Indexing)**

Faissを用いた高速近傍探索。

- **Index Type:** IndexFlatIP (Inner Product)
- **Efficiency:** 全探索 ![][image1] のオーバーヘッドを極限まで抑え、GPUによる超並列演算で検索時間をミリ秒単位に短縮。

## **4\. 主要な最適化のポイント**

| 項目             | 従来の課題                          | 本システムの解決策                   | 効果                                     |
| :--------------- | :---------------------------------- | :----------------------------------- | :--------------------------------------- |
| **I/O 待機時間** | 画像読込中にGPUがアイドル状態になる | tf.data \+ prefetch による非同期処理 | GPU稼働率の最大化                        |
| **推論速度**     | 1枚ずつ処理するオーバーヘッド       | 適切な BATCH_SIZE による一括処理     | 推論時間の劇的な短縮                     |
| **検索計算量**   | データ数に比例して検索が遅くなる    | Faissによるベクトルインデックス構築  | 大規模データでも定数時間に近いレスポンス |
| **セキュリティ** | APIキーの露出リスク                 | Colab Secrets による環境変数管理     | 安全な共有とコラボレーション             |

## **5\. 運用上の注意点**

1. **GPU Runtime:** 本システムはGPUの存在を前提としている。実行前に必ずランタイムが「T4 GPU」以上であることを確認すること。
2. **Memory Management:** BATCH_SIZE はVRAM量に依存する。T4 GPU (16GB) の場合、64〜128が最適。
3. **Kaggle Setup:** Kaggleマイアカウントから発行した KAGGLE_USERNAME および KAGGLE_KEY がColabのシークレットに登録されている必要がある。

## **6\. 今後の拡張性**

- **Approximate Nearest Neighbor (ANN):** データ数が100万件を超える場合、IndexIVFFlat 等の近傍探索アルゴリズムへの切り替えでさらに高速化が可能。
- **Fine-tuning:** 特定のスニーカーブランドやモデルに対する検索精度を向上させるため、Triplet Loss等を用いた追加学習の導入。
