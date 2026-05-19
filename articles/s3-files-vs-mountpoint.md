---
title: "S3 Files と Mountpoint の本質的な違いを整理してみた。同じS3バケットに同じPOSIX命令を送ったら結果が違った話"
emoji: "🗂️"
type: "tech"
topics: ["aws", "s3", "s3files", "mountpoint", "linux"]
published: true

---
こんにちは、ukkiyです。

「S3をフォルダみたいに使いたい」、そう思ったことはありませんか?
Mountpoint for S3 で「できる！」と思ったら制約だらけで諦めた方も多いはず。私も一度諦めました(笑)。

2026年4月にリリースされた **Amazon S3 Files** で、その悩みがかなり解消されました。
ただ、ドキュメントを読んでいて気になったのは「**同じS3バケットを Mountpoint と S3 Files の両方からマウントしたら、同じ操作なのに結果がどう違うのか**」という点です。

この記事では、実際に同じS3バケットを両方の方法でマウントして、同じ操作を試して挙動を比較します。
さらに「**なぜ違いが生まれるのか**」を、POSIX 規格とファイルシステム実装の観点から掘り下げます。

冒頭でひとつ結論を言っておくと、**Mountpoint は制約が多いのではなく、S3 の制約をそのまま正直に見せていただけ** だった、というのが個人的に一番面白かった発見です。

## はじめに

Amazon S3 Files は「S3バケットをファイルシステムとしてマウントできる」という機能です。

2023年にリリースされた Mountpoint for S3 でもマウントはできていました。ただし、既存ファイルの編集もファイル名の変更もできない、読み取り特化の「なんちゃってフォルダ」でしたが。

では何が違うのか。ドキュメントを読むだけでなく、実際に手を動かして、同じS3バケットを両方の方法でマウントして、同じ操作を試して比べてみました。

## そもそもの疑問:同じPOSIX命令なのに結果が違うのはなぜか

ハンズオンの前に、仕組みを理解しておくと結果の意味がわかりやすくなります。

### POSIX とは

POSIX(ポジックス)とは、Linux や macOS などの OS 間で「ファイル操作の命令を統一しよう」という規格です。

POSIX で定められている主な操作は次のようなものです。

- 基本的なファイル操作(作成・読み取り・書き込み・削除)
- ディレクトリ操作(作成・一覧・削除)
- パーミッション・所有者の管理
- 高度な操作(複数プロセスの排他制御、ファイルをメモリにマッピングなど)

「ファイルシステム」と呼ばれるものは、原則としてこの POSIX に従って動くことが期待されています。

### なぜ Mountpoint はフル POSIX 準拠にしなかったのか

これは制約ではなく、**意図的な設計判断**です。AWS 公式ブログにはこう書かれています。

> S3 のオブジェクト API で効率よく実装できない操作はサポートしない

たとえば `rename()` を例にすると違いが明確になります。

```
ローカルの rename:
  ファイル名のメタデータを書き換えるだけ → 一瞬で完了

Mountpoint で S3 の rename:
  新しいキーにオブジェクトをコピー
  → 古いキーを削除
  → S3 への 2 回の API コール、しかもアトミックでない
  → 10万ファイルなら 10万回 × 2 = 20万回の API コール
```

これを「`rename()`」という顔をして提供すると、「一瞬で終わるはず」という期待を裏切ることになります。だから実装しない、という判断です。

### S3 Files はなぜフル POSIX セマンティクスを実現できたのか

答えは、**S3 の上に直接実装しようとしなかったから**です。

```
Mountpoint : S3 API の上で POSIX をエミュレート
           → S3 の制約がそのまま制約になる

S3 Files   : EFS(本物のファイルシステム)をキャッシュ層として使う
           → EFS で rename を実行 → あとで S3 に同期
           → POSIX の制約は EFS が引き受ける
```

EFS はもともとフル POSIX 準拠のファイルシステムなので、その上に乗っかることで自然にフル POSIX セマンティクスが実現できています。S3 Files は EFS の技術を内部実装として借りているわけです。

### 一言でまとめると

- **POSIX** : ファイル操作の命令セットの規格
- **Mountpoint** : 基本操作だけ POSIX 対応、S3 で実装困難な操作は意図的にスキップ
- **S3 Files** : EFS を挟むことで POSIX の全操作に対応

アプリが送る命令は同じ。違いは**ファイルシステムの実装**にあります。

## FUSE と NFS の違い
![](https://static.zenn.studio/user-upload/8c367e0ebab9-20260519.png)

もう一つ、両者を分ける重要な技術要素が「どのプロトコルで OS と通信するか」です。

### Mountpoint(FUSE):カーネルの外で処理

FUSE(Filesystem in Userspace)は、カーネルの外(ユーザー空間)でデーモンが翻訳係として動く仕組みです。アプリの POSIX 命令をユーザー空間のデーモンが受け取り、S3 API に変換します。

S3 はファイルの部分書き込みや rename をサポートしないため、これらの操作は実装できません。

### S3 Files(NFS):カーネルの中で処理

NFS はカーネルに組み込まれたプロトコルなので、フルのファイルシステム機能が使えます。内部で EFS をキャッシュ層として使い、データの実体は S3 に置きます。


## ハンズオン:両方をマウントして比べてみる

### 構成

```
S3バケット(mount-s3-demo-202604)
       ↓
  同じバケットを2つの方法でマウント
       ↓
EC2 ─┬─ Mountpoint  → /mnt/mountpoint-s3
     └─ S3 Files    → /mnt/s3files
```

![](https://static.zenn.studio/user-upload/ace6cd4e66c0-20260519.png)


### Part 1:EC2 のセットアップと S3 バケット作成

EC2 インスタンスを作成し、SSH 接続します。

```bash
# キーペアのパーミッション変更(必須。これをしないとSSH接続できない)
chmod 400 ~/Downloads/s3files-key.pem

# SSH接続(パブリックIPはEC2コンソールで確認)
ssh -i ~/Downloads/s3files-key.pem ec2-user@<パブリックIP>
```

`[ec2-user@ip-172-31-xx-xxx ~]$` のプロンプトが表示されれば接続成功です。

マウント用に S3 バケット `mount-s3-demo-202604` を作成しておきます。

### Part 2:Mountpoint for S3 のセットアップと実験

#### インストール

```bash
# ダウンロード & インストール
wget https://s3.amazonaws.com/mountpoint-s3-release/latest/x86_64/mount-s3.rpm
sudo yum install -y ./mount-s3.rpm

# バージョン確認
mount-s3 --version
```

#### FUSE の設定

デフォルトでは `ec2-user` が FUSE マウントにアクセスできなかったので、設定ファイルを変更します。

```bash
sudo nano /etc/fuse.conf
```

`user_allow_other` の行の `#` を外して有効化します。

```diff
- #user_allow_other
+ user_allow_other
```

#### マウント実行

```bash
# マウント先フォルダ作成(空フォルダでなければならない)
sudo mkdir /mnt/mountpoint-s3

# マウント(--allow-other と --uid 1000 が必要)
sudo mount-s3 mount-s3-demo-202604 /mnt/mountpoint-s3 --allow-other --uid 1000

# 確認(空のバケットなので何も表示されなければOK)
ls /mnt/mountpoint-s3
```

オプションの意味:

- `--allow-other`:マウントしたユーザー以外もアクセス可能にする
- `--uid 1000`:ファイルのオーナーを `ec2-user`(uid=1000)に設定する

#### 書き込みテスト

```bash
# ファイル作成
echo "Mountpointのテスト" > /mnt/mountpoint-s3/mp_test.txt

# 確認
cat /mnt/mountpoint-s3/mp_test.txt
# → Mountpointのテスト

# S3への反映確認
aws s3 ls s3://mount-s3-demo-202604
# → mp_test.txt が表示される
```

S3 コンソールを見に行くと、即時で作成されていました。

![](https://static.zenn.studio/user-upload/e974948619e4-20260519.png)

#### Mountpoint の制約を確認

```bash
# 既存ファイルの上書き
echo "書き換えてみる" > /mnt/mountpoint-s3/mp_test.txt
# → -bash: /mnt/mountpoint-s3/mp_test.txt: Operation not permitted ❌

# ファイル名変更
mv /mnt/mountpoint-s3/mp_test.txt /mnt/mountpoint-s3/mp_renamed.txt
# → mv: cannot move '...': Function not implemented ❌

# 追記
echo "追記してみる" >> /mnt/mountpoint-s3/mp_test.txt
# → -bash: /mnt/mountpoint-s3/mp_test.txt: Operation not permitted ❌
```

3つとも失敗しました。エラーコードの意味は以下の通りです。

| エラーコード | 意味 |
|---|---|
| `EPERM`(Operation not permitted) | その操作自体をサポートしていない。権限の問題ではない |
| `ENOSYS`(Function not implemented) | そのシステムコール自体が未実装 |

`EPERM` は「権限がない」ではなく「**その操作はそもそもできない**」という意味です。Mountpoint が意図的に既存ファイルの上書きをサポートしていないため返ってきます。

`ENOSYS` は Mountpoint が `rename()` システムコールを実装していないため返ってきます。

### Part 3:S3 Files のセットアップと実験

#### S3 ファイルシステムの作成(コンソール)

AWS コンソール → S3 → 左メニュー「ファイルシステム」→「ファイルシステムの作成」を選びます。

バケット名に先ほど作成したバケット名(`mount-s3-demo-202604`)を入力して作成します。

:::message
**バージョニングが必須です**

S3 Files を使うには S3 バケットのバージョニングが有効化されている必要があります。
コンソールからファイルシステムを作成すると自動で有効化されますが、既存バケットに後から適用する場合は旧バージョンが保持されることによるストレージコスト増と、既存のライフサイクルルールとの整合性を事前に確認してください。
:::

作成後、ファイルシステム ID が発行されます。

```
fs-011a011a1234f12345
```

マウントターゲットが自動作成されます。「利用可能」になるまで2〜3分待ちました。

#### セキュリティグループの設定

S3 Files は NFS プロトコル(TCP 2049 番)を使います。EC2 のセキュリティグループにインバウンドルールを追加します。

| タイプ | プロトコル | ポート | ソース |
|---|---|---|---|
| NFS | TCP | 2049 | 0.0.0.0/0 |

:::message alert
本番環境では `0.0.0.0/0` は使わないでください。すべての IP アドレスからのアクセスを許可する設定です。今回はハンズオン用に簡略化しています。
:::

#### EFS ドライバーのインストール

S3 Files は EFS のドライバーを使います。

```bash
# インストール
sudo yum install -y amazon-efs-utils

# バージョン確認(コマンド名に注意)
mount.efs --version
```

#### マウント実行

```bash
# マウント先フォルダ作成
sudo mkdir /mnt/s3files

# マウント実行
sudo mount -t s3files fs-011a011a1234f12345 /mnt/s3files

# マウント確認
mount | grep s3files
# → 127.0.0.1:/ on /mnt/s3files type nfs4 (rw,vers=4.2,...)
```

書き込みで `Permission denied` が出る場合は、ファイルの所有者が `root` になっているので変更します。

```bash
# 所有者確認
ls -la /mnt/s3files
# → root root になっている

# 所有者変更
sudo chown ec2-user:ec2-user /mnt/s3files
sudo chown ec2-user:ec2-user /mnt/s3files/mp_test.txt

# 確認
ls -la /mnt/s3files
# → ec2-user ec2-user になっていればOK
```

同じ S3 バケットをマウントしているので、Mountpoint で作ったファイルが S3 Files 側からも見えます。

```bash
ls /mnt/s3files
# → mp_test.txt
```

#### Mountpoint でできなかった3つを試す

```bash
# 1. 既存ファイルの上書き
echo "書き換えてみる" > /mnt/s3files/mp_test.txt
cat /mnt/s3files/mp_test.txt
# → 書き換えてみる  ✅ 成功！

# 2. ファイル名変更(mv)
mv /mnt/s3files/mp_test.txt /mnt/s3files/sf_renamed.txt
ls /mnt/s3files
# → sf_renamed.txt  ✅ 成功！

# 3. 追記
echo "追記してみる" >> /mnt/s3files/sf_renamed.txt
cat /mnt/s3files/sf_renamed.txt
# → 書き換えてみる
# → 追記してみる  ✅ 成功！
```

3つとも成功しました。S3 コンソールを見に行くとちゃんと反映されています。

#### S3 への同期確認

実際のログを見ると、リネームしたあとの S3 側への反映に時間がかかっていることが分かりました。

```
$ mv /mnt/s3files/mp_test.txt /mnt/s3files/sf_renamed.txt

$ ls /mnt/s3files
sf_renamed.txt

$ aws s3 ls s3://mount-s3-demo-202604
2026-04-09 04:44:31         23 mp_test.txt     ← まだ反映されていない

(数十秒後)

$ aws s3 ls s3://mount-s3-demo-202604
2026-04-09 04:46:26         41 sf_renamed.txt  ← 反映された
```

反映に時間がかかるのは意図的な設計です。

### 「60秒のコミット遅延」の本当の意味

60秒という数字を「遅い」と感じるかもしれませんが、これは単なる制約ではなく**コスト最適化の仕組み**でもあります。

ログファイルへの追記のように短時間に何度も書き込みが発生するケースでは、60秒のウィンドウ内での変更が集約され、S3 へは1回の PUT として反映されます。書き込みのたびに S3 のバージョンが作られるのではなく、まとめて1つのオブジェクトになるため、S3 のリクエストコストとバージョニングによるストレージコストが抑えられます。

ただし、ジョブAがファイルを書き込み、ジョブBが S3 API 経由で即座にそのオブジェクトを取得するような構成では、**最大60秒古いデータを見る可能性があります**。ジョブ間の受け渡しもファイルシステム経由で行うか、S3 のイベント通知で後続を起動する設計にする必要があります。

## 比較結果まとめ

### 操作比較

| 操作 | Mountpoint | S3 Files |
|---|---|---|
| ファイル作成 | ✅ | ✅ |
| ファイル読み取り | ✅ | ✅ |
| 既存ファイルの上書き | ❌ `EPERM` | ✅ |
| ファイル名変更(`mv`) | ❌ `ENOSYS` | ✅ |
| 追記(`>>`) | ❌ `EPERM` | ✅ |
| S3 への自動同期 | ✅(即時) | ✅(遅延あり) |

### 仕組みの比較

| 項目 | Mountpoint | S3 Files |
|---|---|---|
| リリース | 2023年8月 | 2026年4月 |
| プロトコル | FUSE(ユーザー空間) | NFS v4.2(カーネル内) |
| レイテンシ | S3 と同等 | アクティブデータ約1ms |
| 同時接続 | 各自独立 | 最大25,000接続 |
| ファイルロック | ❌ | ✅ |
| 追加コスト | なし | あり |
| Linux 専用 | ✅ | ✅ |

### S3 Files の性能仕様

| 指標 | 値 |
|---|---|
| 1クライアントあたり最大リードスループット | 3 GiB/s |
| ファイルシステムあたり合計リードスループット | テラバイト/秒級 |
| ファイルシステムあたり最大リードIOPS | 250,000 |
| ファイルシステムあたり最大ライトIOPS | 50,000 |
| S3 → FS の反映速度 | 最大2,400オブジェクト/秒 |
| FS → S3 のコミット間隔 | 約60秒 |

EBS の gp3 ボリュームのデフォルトスループットが 125 MiB/s であるのに対し、S3 Files は 1 クライアントあたり 3 GiB/s を出せます。しかもボリュームのプロビジョニング不要で、使った分だけの課金です。

## S3 Files の設計背景:なぜ EFS をキャッシュ層に選んだのか

S3 Files の設計背景を書いた AWS VP Andy Warfield 氏のブログが非常に興味深いので紹介します。

開発当初、チームは「**EFS3**」という名前でファイルとオブジェクトを完全に統合することを目指していました。

しかし何ヶ月も議論を重ねた結果、どんな設計にしてもファイルかオブジェクトのどちらかが何かを犠牲にする「**不快な妥協の連続**」にしかならないという結論に至り、一度行き詰まりました。

クリスマス休暇後、チームは方針を転換します。
「境界を消す」のではなく、「**境界そのものを設計の核心にする**」という逆転の発想です。

そこから生まれたのが **stage & commit** という考え方です。Git から借りた言葉で:

- **stage**:変更をまず EFS キャッシュ層に蓄積する
- **commit**:約60秒ごとにまとめて S3 にプッシュする

ファイルとオブジェクトの境界を明示することで、どちらの特性も犠牲にせず共存できるようになりました。

「境界を消そうとして失敗し、境界を設計の核心にした瞬間にすべてがうまくいった」というのは、設計の教訓としても面白いエピソードです。

### rename の注意点

`mv` 自体は動きます。ただし大量のファイルが入ったディレクトリをまるごとリネームするときは注意が必要です。

S3 には rename という操作がそもそも存在しないため、内部では「新しいキーにコピーして、古いキーを削除する」という処理に変換されます。ファイルシステム上では rename は一瞬で終わりますが、S3 への反映はその後にバックグラウンドで走ります。

大量のファイルを含むディレクトリを rename する場合、S3 側の反映に数分かかることがあり、その間 S3 API からは古いキーしか見えない状態が続きます。日常的なファイル操作では気にする必要はありませんが、デプロイスクリプトなどで大規模なディレクトリ移動を行う場合は注意してください。

## 使い分けの指針

```
「既存ファイルを書き換えたい」
または「複数インスタンスが同時に書き込みたい」
または「mv や flock を使いたい」

        ↓ Yes が1つでもある
     → S3 Files

上記すべてが No(読み取り + 新規書き込みのみ)
        → Mountpoint(無料・シンプル)
```

### S3 Files が特に効果的なユースケース

- ML の前処理パイプライン(S3 → EBS のステージング層を廃止できる)
- Lambda で大きな参照データを扱う関数(`/tmp` の容量制約から解放される)
- ファイルシステム前提で書かれたレガシーアプリのクラウド移行
- AI エージェントが複数並走して共有データを読み書きする場面

## まとめ

同じ S3 バケットに対して同じ POSIX 命令を送っているのに、Mountpoint と S3 Files で結果が全然違う。その理由はファイルシステムの実装の違いにあります。

- **Mountpoint**:FUSE を使いユーザー空間で S3 API に翻訳。S3 がサポートしない操作(rename・部分書き込み)は実装できない
- **S3 Files**:NFS をカーネル内で処理し、EFS をキャッシュ層として使う。フルのファイル機能が使える

「読んで新規書き出しだけ」なら Mountpoint で十分。「編集・共有・同時書き込み」が必要なら S3 Files、という使い分けが基本です。

冒頭で「Mountpoint で諦めた」と書きましたが、S3 Files を触ってみてその理由がよくわかりました。Mountpoint は制約が多いのではなく、**S3 の制約をそのまま正直に見せていただけ** だったんですね。S3 Files はその制約を EFS で引き受けることで解決した、という設計の話が個人的には一番面白かったです。

---

## おまけ:ハマったポイント集

実際に試してぶつかったエラーと対処法です。同じ環境(WSL → EC2)で試す方の参考になれば。

| エラー | 原因 | 対処法 |
|---|---|---|
| `Failed to create FUSE session` | `sudo` なしで実行 | `sudo mount-s3` で実行 |
| `Permission denied`(`ls` 時) | `--allow-other` なし | オプションを追加して再マウント |
| `sed -i` で `/etc/fuse.conf` が変わらない | `#user_allow_other` の前にスペースがある | `sudo nano /etc/fuse.conf` で直接編集 |
| `Permission denied`(書き込み時1) | `user_allow_other` が無効 | `/etc/fuse.conf` を編集して有効化 |
| `Permission denied`(書き込み時2) | `--uid` 未指定 | `--uid 1000` を追加(`ec2-user` の uid に合わせる) |
| `mount point does not exist` | マウント先フォルダ未作成 | `sudo mkdir /mnt/s3files` |
| `timeout`(S3 Files) | マウントターゲットの SG に NFS 未設定 | TCP 2049 を追加 |
| `Permission denied`(S3 Files 書き込み時) | ファイル所有者が `root` | `sudo chown ec2-user:ec2-user` |

### `--allow-other` だけでは足りない理由

`--allow-other` は「マウントしたユーザー以外もアクセスできる」というオプションですが、これだけではファイルの所有者が `root` のままです。`--uid 1000` を追加することで「ファイルのオーナーを `ec2-user`(uid=1000)として扱う」ようになり、書き込みができるようになります。

```bash
# NG:--allow-other だけだと書き込みで Permission denied
sudo mount-s3 my-bucket /mnt/mountpoint-s3 --allow-other

# OK:--uid も必要
sudo mount-s3 my-bucket /mnt/mountpoint-s3 --allow-other --uid 1000
```

自分の uid は `id` コマンドで確認できます。

## 後片付け

EC2 は起動している間課金されます。忘れずに削除してください。

```bash
# S3 バケットの中身を削除
aws s3 rm s3://mount-s3-demo-202604 --recursive

# アンマウント
sudo umount /mnt/s3files
sudo umount /mnt/mountpoint-s3

# EC2 からログアウト
exit
```

AWS コンソールで:

- EC2 インスタンスを「終了」
- S3 Files のファイルシステムを削除
- S3 バケットを削除

最後まで読んでいただきありがとうございました。