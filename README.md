# Job管理ツールを調査

候補

-   [Nukesor/pueue](https://github.com/Nukesor/pueue)
-   [jhuckaby/Cronicle](https://github.com/jhuckaby/Cronicle)
-   [Zellij](https://zellij.dev/) - ターミナルマルチプレクサ (cf. screen, tmux)
    -   [zellij-org/zellij](https://github.com/zellij-org/zellij)
-   [dkron-io/dkron](https://github.com/dkron-io/dkron) - 分散 cron
-   [rundeck/rundeck](https://github.com/rundeck/rundeck)

## pueue

pueueがどんなソフトか、AIモードに説明させた。

> カテゴリ：ローカル/パーソナル向け ジョブ管理ツール
>
> * 概要: 主にシングルユーザーのコマンドライン操作を補助するために設計されたツールです。
> * 特徴:
>     * シェル感覚の利用: 時間のかかるコマンド（コンパイル、データ処理、GPU計算など）をキューに追加し、逐次または並列で実行させます。
>     * 非同期実行: ターミナルを閉じてもバックグラウンドで処理が続き、後から状態を確認したり出力を取得したりできます。
>     * 軽量: Rustで書かれており、設定がシンプルで個人の開発環境に向いています。


FreeBSDで試す

```
sudo pkg install pueue
```

portsで入れるとRustからビルドすることになり、時間がかかるのを嫌った。

```
$ pkg info -l pueue
pueue-4.0.1_3:
        /usr/local/bin/pueue
        /usr/local/bin/pueued
        /usr/local/etc/rc.d/pueued
        /usr/local/share/licenses/pueue-4.0.1_3/APACHE20
        /usr/local/share/licenses/pueue-4.0.1_3/LICENSE
        /usr/local/share/licenses/pueue-4.0.1_3/MIT
        /usr/local/share/licenses/pueue-4.0.1_3/catalog.mk
```

構成ファイルは単純。デーモンとクライアントとライセンスファイル。
man ページはない。

<https://github.com/Nukesor/pueue/wiki/Get-started>

1.  `pueued` でデーモンを起動

    `pueued -d` でバックグラウンド化。現在のセッションが終了してもデーモンは生き続ける

2.  `pueue add -- {COMMAND}` でコマンドをキューイング≒タスク化。タスクにはIDが振られる
3.  `pueue status` もしくは `pueue` でタスクの状態を確認
4.  `pueue log {id}` でタスクのログを個別に確認可能
5.  `pueue clean` で終わったタスクを削除
6.  `pueue shutdown` でデーモンを終了

その他

-   `pueue follow` でタスクのログに対して `tail -f` して覗く
-   `pueue parallel {num}` で並列実行数を変更できる。デフォルトは1=直列
-   `pueue group add {name}` でキューを追加
-   `pueue wait` でタスクやキューの終了を待てる
-   stashed task = 投入するがすぐには実行しない
    -   `pueue start`
-   タスク間の依存関係を設定できる

感想:

-   ローカルで1台で使うには便利そう

## cronicle

AIモードによる説明

> カテゴリ：分散型/エンタープライズ向け ジョブスケジューラ
>
> *   概要: 複数のサーバーにまたがるジョブの実行とスケジュール管理を行うためのフル機能のシステムです。
> *   特徴:
>     *    GUIベース: 高機能なWeb UIを備えており、ブラウザからジョブの監視や設定、ログの閲覧が可能です。
>     *    分散実行: マスターサーバーが複数のワーカーサーバー（実行ノード）にジョブを割り振る「分散アーキテクチャ」を採用しています。
>     *    高機能スケジューリング: cron形式の定期実行はもちろん、複雑な依存関係（ジョブAが終わったらBを実行）やエラー通知機能が充実しています。
