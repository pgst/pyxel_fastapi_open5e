<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" class="logo" width="120"/>

# PyxelとFastAPIが通信するD\&D 5eオンラインRPG開発ガイド

本レポートでは、レトロゲームエンジンPyxelをフロントエンド、高速WebフレームワークのFastAPIをバックエンドとして使用し、公開されているダンジョンズアンドドラゴンズ5eのルールを適用したオンラインRPGの開発方法を詳細に解説します。この組み合わせにより、レトロな見た目と現代的なオンライン機能を併せ持つユニークなRPGを作成できます。

## Pyxelの概要とフロントエンド開発

### Pyxelとは

Pyxelは、レトロゲーム開発に特化したPythonのゲームエンジンです。16色のカラーパレット、256×256ピクセルの画面サイズなど、意図的に制限された仕様によって、初心者でも扱いやすく、レトロゲーム開発に最適なツールとなっています[^1_2][^1_8]。また、Windows、Mac、Linuxに加え、Webブラウザ上でも動作するため、プラットフォームの制約なくゲームを配信できます[^1_1][^1_5]。

### Pyxelのインストールと基本構造

Pyxelは以下のコマンドで簡単にインストールできます：

```python
pip install pyxel
```

Pyxelを使用したゲームの基本構造は次のとおりです[^1_2]：

```python
import pyxel

class Game:
    def __init__(self):
        pyxel.init(160, 120, title="D&D Online RPG")
        # ゲームの初期化処理
        pyxel.run(self.update, self.draw)
        
    def update(self):
        # ゲームロジックの更新処理
        pass
        
    def draw(self):
        # 画面描画処理
        pyxel.cls(0)  # 画面をクリア
        # キャラクターやUIの描画

if __name__ == '__main__':
    Game()
```


### Webブラウザでの実行方法

Pyxelで作成したゲームをWebブラウザで実行するには、以下の3つの方法があります[^1_5]：

1. **Pyxel Web Launcherを使用する方法**：GitHubリポジトリを指定して直接実行
2. **HTMLファイルに変換する方法**：`pyxel app2html`コマンドを使用
3. **Pyxelカスタムタグを使用する方法**：HTMLファイル内に特殊なタグを記述

特に2つ目の方法は、以下のコマンドで簡単にHTML化できます[^1_5][^1_11]：

```
pyxel app2html your_app.pyxapp
```


## FastAPIによるバックエンド開発

### FastAPIとWebSocketの概要

FastAPIは高性能なPythonのWebフレームワークで、WebSocketプロトコルをサポートしています。WebSocketは継続的な双方向通信を可能にし、リアルタイムでのデータ交換に最適です[^1_6][^1_16]。

### WebSocketサーバーの実装

FastAPIでWebSocketサーバーを実装する基本的なコードは以下の通りです[^1_16]：

```python
from fastapi import FastAPI, WebSocket

app = FastAPI()

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    while True:
        data = await websocket.receive_text()
        # データの処理
        await websocket.send_text(f"処理結果: {data}")

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="127.0.0.1", port=8000)
```


### 複数クライアントの管理

オンラインRPGでは複数のプレイヤーを管理する必要があります。以下のようにコネクションマネージャを実装することで、複数のクライアントを効率的に管理できます[^1_6]：

```python
class ConnectionManager:
    def __init__(self):
        self.active_connections = []

    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.active_connections.append(websocket)

    def disconnect(self, websocket: WebSocket):
        self.active_connections.remove(websocket)

    async def broadcast(self, message: str):
        for connection in self.active_connections:
            await connection.send_text(message)

manager = ConnectionManager()

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await manager.connect(websocket)
    try:
        while True:
            data = await websocket.receive_text()
            await manager.broadcast(f"{data}")
    except:
        manager.disconnect(websocket)
```


## PyxelとFastAPIの連携

### クライアント側（Pyxel）の実装

PyxelからWebSocketを使用してサーバーと通信するためには、標準ライブラリの`websockets`を利用します。ただし、Pyxelは非同期処理が基本ではないため、スレッドを使用して通信を行います[^1_11]：

```python
import pyxel
import asyncio
import websockets
import threading
import json

# WebSocketクライアント
class WebSocketClient:
    def __init__(self, uri):
        self.uri = uri
        self.messages = []
        self.connected = False
        threading.Thread(target=self.start_client).start()

    def start_client(self):
        asyncio.run(self.connect())

    async def connect(self):
        async with websockets.connect(self.uri) as websocket:
            self.connected = True
            while True:
                try:
                    message = await websocket.recv()
                    self.messages.append(json.loads(message))
                except:
                    self.connected = False
                    break

    def send_message(self, message):
        asyncio.run(self.send(message))

    async def send(self, message):
        async with websockets.connect(self.uri) as websocket:
            await websocket.send(json.dumps(message))

# ゲームクラス
class Game:
    def __init__(self):
        pyxel.init(160, 120, title="D&D Online RPG")
        self.client = WebSocketClient("ws://localhost:8000/ws")
        self.player_x = 80
        self.player_y = 60
        pyxel.run(self.update, self.draw)
    
    def update(self):
        # プレイヤー移動
        if pyxel.btn(pyxel.KEY_LEFT):
            self.player_x -= 2
        # 他のキー入力処理
        
        # サーバーに位置情報を送信
        self.client.send_message({
            "type": "position", 
            "x": self.player_x, 
            "y": self.player_y
        })
        
    def draw(self):
        pyxel.cls(0)
        # 自プレイヤーの描画
        pyxel.circ(self.player_x, self.player_y, 8, 11)
        
        # 他プレイヤーの描画（サーバーからのデータに基づく）
        for msg in self.client.messages:
            if msg["type"] == "player":
                pyxel.circ(msg["x"], msg["y"], 8, 9)

if __name__ == '__main__':
    Game()
```


### サーバー側（FastAPI）の実装

サーバー側では、各クライアントからの情報を受け取り、他のクライアントに配信します[^1_16]：

```python
from fastapi import FastAPI, WebSocket
import json

app = FastAPI()
active_players = {}

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    player_id = id(websocket)
    active_players[player_id] = {"x": 80, "y": 60}
    
    try:
        while True:
            data = await websocket.receive_text()
            message = json.loads(data)
            
            if message["type"] == "position":
                # プレイヤーの位置情報を更新
                active_players[player_id]["x"] = message["x"]
                active_players[player_id]["y"] = message["y"]
                
            # 全プレイヤーの情報をブロードキャスト
            for pid, pos in active_players.items():
                if pid != player_id:  # 自分以外のプレイヤー情報を送信
                    await websocket.send_json({
                        "type": "player",
                        "id": pid,
                        "x": pos["x"],
                        "y": pos["y"]
                    })
    except:
        del active_players[player_id]
```


## D\&D 5eルールの実装

### SRD 5.1の利用

D\&D 5eのルールはSRD 5.1として公開されており、現在はCreative Commons CC-BY-4.0ライセンスで利用可能です[^1_17]。これにより、商用・非商用を問わず、適切なクレジット表記をすればルールを利用できます。

### オープンなAPIの活用

Open5eのようなオープンなAPIを利用することで、モンスター、呪文、アイテムなどのデータを取得できます[^1_9]。例えば、次のようにAPIを呼び出すことができます：

```python
import requests

def get_monsters(cr=None):
    url = "https://api.open5e.com/monsters/"
    params = {}
    if cr:
        params["cr"] = cr
    response = requests.get(url, params=params)
    return response.json()["results"]

def get_spell_by_name(name):
    url = f"https://api.open5e.com/spells/?search={name}"
    response = requests.get(url)
    return response.json()["results"]
```


### キャラクター作成とゲームロジック

D\&D 5eのキャラクター作成とゲームロジックは、以下のクラス設計を参考にできます：

```python
class Character:
    def __init__(self, name, race, character_class, level=1):
        self.name = name
        self.race = race
        self.character_class = character_class
        self.level = level
        self.hp = 0
        self.abilities = {
            "str": 10, "dex": 10, "con": 10,
            "int": 10, "wis": 10, "cha": 10
        }
        self.init_hp()
        
    def init_hp(self):
        # クラスに応じてHPを設定
        if self.character_class == "barbarian":
            base_hp = 12
        elif self.character_class == "wizard":
            base_hp = 6
        # 他のクラス...
        else:
            base_hp = 8
        
        # レベル1の場合の最大HP
        self.hp = base_hp + self.get_ability_modifier("con")
        
    def get_ability_modifier(self, ability):
        return (self.abilities[ability] - 10) // 2
        
    def roll_attack(self, target, weapon):
        import random
        # D20で攻撃ロール
        roll = random.randint(1, 20)
        # 攻撃修正値を追加（筋力/敏捷に基づく）
        if weapon["type"] == "melee":
            modifier = self.get_ability_modifier("str")
        else:
            modifier = self.get_ability_modifier("dex")
        
        # 習熟ボーナスを追加
        proficiency = 2  # レベル1-4の習熟ボーナス
        
        total = roll + modifier + proficiency
        
        # 目標のACと比較
        if total >= target.armor_class:
            # ダメージ計算
            damage_roll = random.randint(1, weapon["damage_die"])
            damage = damage_roll + modifier
            target.hp -= damage
            return True, damage
        else:
            return False, 0
```


## システム統合と実装手順

### 開発ステップ

オンラインD\&D RPGを開発するための一般的な手順は以下の通りです：

1. **環境セットップ**：Pyxel、FastAPI、必要なパッケージをインストール
2. **基本的なゲームループの作成**：Pyxelを使用して基本的なゲーム画面を表示
3. **ローカルでのキャラクター操作**：キャラクターの移動など基本操作を実装
4. **D\&D 5eルールの基本実装**：キャラクター作成、戦闘システムなどの実装
5. **WebSocket通信の実装**：FastAPIでWebSocketサーバーを構築
6. **オンライン機能の実装**：複数プレイヤー、チャット機能など
7. **UIの改良**：ゲームのインターフェースをわかりやすく改善
8. **テストとバグ修正**：動作テスト、バグ修正
9. **デプロイ**：サーバーのデプロイとクライアントの配布

### フォルダ構成の例

```
project/
├── client/
│   ├── main.py        # Pyxelゲームメイン
│   ├── ui.py          # UI関連コード
│   ├── network.py     # WebSocket通信
│   ├── characters.py  # D&Dキャラクター関連
│   ├── combat.py      # 戦闘システム
│   └── assets/        # 画像・音声素材
├── server/
│   ├── main.py        # FastAPIメインサーバー
│   ├── game_state.py  # ゲーム状態管理
│   ├── dnd_rules.py   # D&Dルールロジック
│   └── db.py          # データベース連携
└── shared/
    ├── constants.py   # 共通定数
    └── models.py      # 共通データモデル
```


## 課題と解決策

### 既知の課題と対応策

1. **Pyxelの制限**：
    - 16色に限定された色彩表現
    - 解決策：色の制限を活かしたユニークなビジュアルスタイルの採用
2. **WebSocketのレイテンシ**：
    - オンラインゲームでの遅延発生
    - 解決策：状態予測や補間技術の実装
3. **複雑なD\&Dルール**：
    - 全ルールの実装は膨大な作業
    - 解決策：最初は基本ルールのみ実装し、徐々に拡張
4. **スケーラビリティ**：
    - 多数のプレイヤーへの対応
    - 解決策：シャーディングやロードバランシング技術の導入

### PyxelをWebブラウザで動作させる際の注意点

Pyxelは、Web版として以下の方法で実行できますが、いくつかの制限があります[^1_5]：

- Pyxel Web Launcherを使用する場合、追加パッケージは限られたものしか使用できない
- カスタムタグを使用する場合、ローカルでのファイル読み込みにはサーバーが必要
- モバイルデバイスでの操作感は最適化が必要

これらの制限に対処するため、バーチャルゲームパッドの活用やUI要素の大きさ調整が重要です[^1_5][^1_11]。

## 結論

Pyxelをフロントエンド、FastAPIをバックエンドとした組み合わせは、レトロな見た目と現代的な機能を兼ね備えたオンラインD\&D RPGを開発するための強力な選択肢です。SRD 5.1のCreative Commonsライセンス化により、D\&D 5eのルールを自由に利用できるようになりました。

この組み合わせにより、懐かしさと新しさを融合させたユニークなゲーム体験を提供できます。開発にはPythonの知識が必要ですが、比較的シンプルなAPIと豊富なリソースにより、中級レベルのプログラマーでも実装可能です。

オープンソースツールとオープンなゲームルールを活用することで、コミュニティ主導のゲーム開発が可能になり、D\&Dファンに新たな遊び方を提供することができるでしょう。

<div style="text-align: center">⁂</div>

[^1_1]: https://github.com/kitao/pyxel

[^1_2]: https://gomafree-tech.com/?p=3588

[^1_3]: https://docs.rs/pyxel

[^1_4]: https://github.com/kitao/pyxel/wiki/Pyxel-User-Examples

[^1_5]: https://github.com/kitao/pyxel/blob/main/docs/pyxel-web-ja.md

[^1_6]: https://fastapi.tiangolo.com/advanced/websockets/

[^1_7]: https://media.wizards.com/2016/downloads/DND/SRD-OGL_V5.1.pdf

[^1_8]: https://www.youtube.com/watch?v=gXpe9HZ3Au8

[^1_9]: https://open5e.com/api-docs

[^1_10]: https://www.reddit.com/r/Python/comments/ybf948/pyxel_a_retro_game_engine_for_python_reaches/

[^1_11]: https://qiita.com/Prosamo/items/6e2f7147e38aca3db5ca

[^1_12]: https://fastapi.tiangolo.com/ja/advanced/websockets/

[^1_13]: https://www.dndbeyond.com/srd

[^1_14]: https://github.com/open5e/open5e-api

[^1_15]: https://pypi.org/project/pyxel/1.2.8/

[^1_16]: https://apidog.com/jp/blog/using-websocket-with-fastapi/

[^1_17]: https://www.reddit.com/r/UnearthedArcana/comments/10onf8x/ogl_10a_remains_and_the_srd_51_is_now_creative/

[^1_18]: https://qiita.com/automation2025/items/91e6a29264289653fa5e

[^1_19]: https://sourceforge.net/projects/pyxel.mirror/

[^1_20]: https://www.youtube.com/watch?v=0g0L5iGKv9g

[^1_21]: https://www.educationalinsights.com/codewith-pyxel-help

[^1_22]: https://make-muda.net/2017/10/5645/

[^1_23]: https://www.linkedin.com/pulse/end-to-end-real-time-gaming-time-management-system-using-tuấn-dương-wopxc

[^1_24]: https://apidog.com/jp/blog/python-websocket-tutorial/

[^1_25]: https://gihyo.jp/book/2025/978-4-297-14657-3

[^1_26]: https://zenn.dev/kun432/scraps/91f2dfa6daa79a

[^1_27]: https://clogique.fr/nsi/premiere/fil_rouge/documentation-pyxel-2023.pdf

[^1_28]: https://community.silverbullet.md/t/query-open5e-api-in-silverbullet-with-space-script/183/2

[^1_29]: https://library.wiremock.org/catalog/api/d/dnd5eapi.co/dnd5eapi-co/

[^1_30]: https://github.com/OhkuboSGMS/pyxel-tutorial

[^1_31]: https://note.com/koide_mizu1433/n/na465e518f4c4

[^1_32]: https://note.com/koide_mizu1433/n/nbefc660a8ebf

[^1_33]: https://qiita.com/munepi0713/items/8acf9ce353948fc1baf1

[^1_34]: https://github.com/kitao/pyxel/issues/412

[^1_35]: https://github.com/kitao/pyxel/wiki/How-To-Use-Pyxel-Web/3c7ccc624e95584ecc1c9696628cafca91bff7df

[^1_36]: http://ateliereclair.blog.fc2.com/blog-entry-721.html

[^1_37]: https://kitao.github.io/pyxel/wasm/launcher/

[^1_38]: https://qiita.com/rwatanab1999/items/d5c0bb876f0b44cac2f0

[^1_39]: https://note.com/koide_mizu1433/n/nb86259d5c963

[^1_40]: https://qiita.com/kyukkyu81/items/3d186ba5b78f2de83cfe

[^1_41]: https://zenn.dev/slm/articles/1c9cb4cd0af808

[^1_42]: https://store.steampowered.com/app/652940/Pyxel_Knight/?l=japanese\&cc=br

[^1_43]: https://itch.io/games/made-with-pyxel-edit/tag-multiplayer

[^1_44]: https://local.codewithpyxel.com

[^1_45]: https://www.pyxelstudio.net

[^1_46]: https://qiita.com/Katsuya_Ogata/items/1f321f6d9fc14fcd4607

[^1_47]: https://zenn.dev/knot/articles/dbce3e3a2f3f3a

[^1_48]: https://qiita.com/ryutarom128/items/2016c1567208ff38d461

[^1_49]: https://github.com/zhiyuan8/FastAPI-websocket-tutorial

[^1_50]: https://note.com/koide_mizu1433/n/n44193a4008d9

[^1_51]: https://pixel-networks.com

[^1_52]: https://zenn.dev/kazuhito/articles/bacdd4d2a1bc4e

[^1_53]: https://note.com/koide_mizu1433/n/nbd846c361854

[^1_54]: https://note.com/213414/n/nd9981ad5bb19

[^1_55]: https://note.com/pipechair/n/nab7338534b48

[^1_56]: https://www.youtube.com/watch?v=xt_DhZ622zw

[^1_57]: https://screenrant.com/dnd-2024-srd-52-creative-commons-license-explainer/

[^1_58]: https://en.wikipedia.org/wiki/Open_Game_License

[^1_59]: https://www.reddit.com/r/dndnext/comments/10muldi/wizards_backs_down_on_ogl_10a_deauthorization/?tl=ja

[^1_60]: https://www.reddit.com/r/dndnext/comments/1clpz4i/dd_system_reference_document_52_will_release_in/

[^1_61]: https://media.wizards.com/2016/downloads/SRD-OGL_V1.1.pdf

[^1_62]: https://qiita.com/hann-solo/items/d5093fd30a89f42f08c4

[^1_63]: https://open5e.com

[^1_64]: https://github.com/open5e/open5e-api/releases

[^1_65]: https://stackoverflow.com/questions/70282857/error-while-setting-data-from-web-api-as-a-json-object-w-system-err-org-json-js

[^1_66]: https://github.com/brianclogan/dnd-5e-api-php

[^1_67]: https://www.reddit.com/r/DnD/comments/11mswgh/open5e_is_a_lightweight_open_source_visualization/

[^1_68]: https://github.com/5e-bits/5e-database

