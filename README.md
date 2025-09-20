
---

# ONO Token (ERC-20)

Sepolia ネットワークにデプロイした ERC-20 トークン。
- **Gnosis Safe (tomato-multisig)** を利用して、transaction作成者・transaction署名者・transaction実行者を分離する仕様

 [Sepolia Etherscan Contract](https://sepolia.etherscan.io/address/0xCE20C9d325168aC1f34d6671293Fd611B23fE0Da#code)

---

##  プロジェクト構成

```text
onion-project/
├─ src/
│   ├─ ONOToken.sol          # ERC-20 本体 (ERC20 + AccessControl)
│   └─ Counter.sol           # Forge初期テンプレ (不要なら削除可)
├─ script/
│   ├─ DeployONOToken.s.sol  # Safeを権限者にしてデプロイするスクリプト
│   └─ Counter.s.sol         # Forge初期テンプレ (不要なら削除可)
├─ lib/                      # forge install したライブラリ
│   ├─ forge-std
│   └─ openzeppelin-contracts
├─ foundry.toml              # Solidityバージョン・最適化設定
├─ .env                      # RPC/鍵/Safeアドレスの設定ファイル
└─ README.md
```

---

##  セットアップ

### 1. Foundry インストール

```bash
curl -L https://foundry.paradigm.xyz | bash
source ~/.bashrc
foundryup
forge --version
```

### 2. ライブラリのインストール

プロジェクトで使用するテスト用ユーティリティと OpenZeppelin のコントラクトを追加します。

```bash
cd onion-project
forge install foundry-rs/forge-std
forge install OpenZeppelin/openzeppelin-contracts@v5.0.2
```

---

### 3. `.env` 設定

```ini
PRIVATE_KEY=0xデプロイ専用アカウントの秘密鍵
RPC_URL=https://sepolia.infura.io/v3/xxxxxxxx

SAFE_ADMIN=0xYourSafeOnSepolia
SAFE_MINTER=0xYourSafeOnSepolia
TREASURY=0xYourSafeOnSepolia

TOKEN_NAME=Onion
TOKEN_SYMBOL=ONO
PREMINT=0
```

- SAFE_ADMIN = 管理者ロール (DEFAULT_ADMIN_ROLE) を持つアドレス
- SAFE_MINTER = ミント権限 (MINTER_ROLE) を持つアドレス
- TREASURY = 発行したトークンや資金を受け取るアドレス

### 4. ビルド

```bash
forge build
```

処理フロー
- 1.foundry.toml の設定を読み込む
　- Solidity のバージョン
　- 最適化オプション（optimizer_runs など）
　- remappings (@openzeppelin/... がどこにあるか)
- 2.src/ 以下の .sol ファイルをコンパイル
　 -依存関係のある lib/ のコードも含めて解析
- 3.結果を out/<ファイル名>.sol/ に出力
　- <コントラクト名>.json を生成
　- この中に ABI・バイトコード・メタ情報すべてが入っている
- 4.lib/ のライブラリも自動で解決して一緒にコンパイル
　- OpenZeppelin や forge-std のコードも参照される

- ＊ABI = スマートコントラクトを外部から正しく操作するためのインターフェース情報（関数一覧と型定義のカタログ）

### 5. デプロイ

```bash
set -a; source .env; set +a

forge script ./script/DeployONOToken.s.sol:DeployONOToken \
  --rpc-url "$RPC_URL" --private-key "$PRIVATE_KEY" --broadcast -vvvv
```

---

##  デプロイ結果 (Sepolia)

* **Contract Address**: `0xCE20C9d325168aC1f34d6671293Fd611B23fE0Da`
* **Chain ID**: 11155111 (Sepolia)
* **Tx Hash**: `0xbb8930c16105f0e0bed4533d99ae52ad005a1a97d4ba28f8b29380f1df53791d`

---

##  Etherscan での確認

### 1. ソースコード検証 (APIキー利用)

1. Etherscan にログインし、API キーを取得する
2. 環境変数を設定

   ```bash
   export ETHERSCAN_API_KEY=xxxxx
   ```
3. ソース検証

   ```bash
   forge verify-contract --watch \
     --chain-id 11155111 \
     0xCE20C9d325168aC1f34d6671293Fd611B23fE0Da \　# デプロイ済みコントラクトのアドレス
     src/ONOToken.sol:ONOToken \　# ソースファイルとコントラクト名
     --verifier etherscan --etherscan-api-key $ETHERSCAN_API_KEY　# Etherscan で検証する
   ```
4. 成功すると、Etherscan に **Read Contract / Write Contract** タブが出現

---

・デプロイ時
　→ ブロックチェーンには「バイトコード（機械語）」だけが記録される

・問題点
　→ バイトコードだけでは、人間には「どんなコントラクトなのか」分からない

・検証（verify）
　→ ソースコードを Etherscan にアップロード
　→ Etherscan が再コンパイルして、ブロックチェーン上のバイトコードと一致するかを照合

・一致すれば公開
　→ 誰でも Etherscan でソースを読める
　→ Read / Write Contract タブから関数を呼べる

---

### 2. Read Contract での確認

* **name()** → `Onion`
* **symbol()** → `ONO`
* **decimals()** → `18`
* **totalSupply()** → `0`

 トークン仕様が正しいことを確認済み

---

### 3.権限確認

管理者権限 (DEFAULT_ADMIN_ROLE) と ミント権限 (MINTER_ROLE) をチェック

結果:

tomato-multisig (Safeアカウント) → 両方の権限を持っている 

tomato (個人ウォレット) → 権限なし

つまり、トークンの管理や発行は Safe にしかできない。
個人のウォレット (EOA) には一切の権限が残っていないので、安全に運用できる。

---

##  初期ミント (Safe 経由)


## 1. ABI の取得（汎用）

Etherscan Testnetから取得（検証済みのとき）

1.コントラクトの Etherscan ページ → Contract タブ → Code → Contract ABI
2.表示された JSON 全体をコピー
3.Safe の Transaction Builder → Enter ABI に貼り付け

※ テストネットでも検証してあれば同様に取得できます。

### 2. Safe ダッシュボードで操作
＊権限を分離させるため、safeの設定を次のとおりにしています。
- **transaction作成者：**　
 - MetamuskアカウントA
- **transaction署名者（2/3マルチシグ）：**　
 - MetamuskアカウントB
 - MetamuskアカウントC
 - MetamuskアカウントD
- **transaction実行者（ブロードキャスト）:**
 - MetamuskアカウントB
 - MetamuskアカウントC
 - MetamuskアカウントD
 - アカウントB･Cが署名を担当すれば、アカウントDが実行を担当。

1. transactionを作成
 - アカウントAが**New Transaction → Contract interaction**
2. Contract address: `0xCE20C9d325168aC1f34d6671293Fd611B23fE0Da`
3. Enter ABI に コピーしたJSONを貼る
4. Transaction information
 - To Addess:  `0xCE20C9d325168aC1f34d6671293Fd611B23fE0Da`
 - Contract Method Selector: mint
 - to(address): 受取先address
 - amount 1000000000000000000 ( 1 ONO = 10^18 wei )
 - *Safe の Transaction Builder で `mint(address,uint256)` を呼び出すときは、  
 - `amount (uint256)` に **wei 単位の整数**を入力してください（小数はエラーになります）。
5. **Add new transaction →　Creat Batch　→　Creat Propose**
6. transactionに1人目が署名
 - アカウントBで**Send Batch → Continue → Sign** 
7. transactionに2人目が署名
 - アカウントCで**Confirm → Continue → Sign** 
9. transactionをブロードキャスト
 - アカウントDで**Execute → Continue → Sign**  

---

##  MetaMask にインポート

1. MetaMask を Sepolia に切り替える
2. 「トークンをインポートImport」→「カスタムトークン」
3. 以下を入力：
 -  network: sepolia
 -  Token Address: `0xCE20C9d325168aC1f34d6671293Fd611B23fE0Da`
 -  Symbol: `ONO`
 -  Decimals: `18`
4. インポートすると残高が表示される
