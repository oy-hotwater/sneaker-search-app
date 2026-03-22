## sneaker_similar_image_search_ver2
#### 以前のsneaker_similar_image searchの問題点: 
- CPUを使っており、非効率
- CPUだからか、画像をfor文で一枚ずつ見ており、時間がかかる
#### 改善策:
- GPUを使用。Google Colabの「編集」→「ノートブックの設定」から、「TP4 GPU」を選択
#### トラブルシューティング:
- `!pip install -q faiss-gpu`では、Google Colabではエラーが出る。
- 下記のように書き換えることで解決。詳しくは**faiss-gpuの仕様**参照
```
!apt install -q libomp-dev
!pip install -q faiss-gpu-cu12  # CUDA 12系の場合
```

#### faiss-gpuの仕様
- 通常の `pip install faiss-gpu` だけでは依存ライブラリの不足でエラーになることがあるため、以下のコマンドを組み合わせて実行するのが一般的らしい
```
# 依存ライブラリのインストールとセットで行う
!apt install libomp-dev
!pip install faiss-gpu
```

- また、ColabのCUDA環境（現在は主にCUDA 12系）に合わせる場合、以下のパッケージを指定するとより安定する
```
!pip install faiss-gpu-cu12  # CUDA 12系の場合
```
