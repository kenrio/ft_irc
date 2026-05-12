# C++で実装したIRC（Internet Relay Chat）サーバー

開発リポジトリ: https://github.com/funa-yuy/42_irc

![IRCサーバーデモ](/ft-irc-1.gif "IRCサーバーデモ")

---

既存のIRCクライアントで接続してチャットができるIRCサーバー

複数クライアントの同時接続、チャンネル機能、プライベートメッセージなどのIRCコマンドに対応。`poll()` を用いたノンブロッキングI/Oで多重化処理を実現し、RFC 2812（IRCプロトコル）に準拠した実装を行った。

設計面では、Commandパターンとファクトリメソッドを採用。コマンド処理とネットワークI/Oの責務を分離し、仕様への対応と拡張性を両立する構造を意識した。

機能：

* 複数クライアントの同時接続
* チャンネル管理（参加・退出・モード設定・招待）
* プライベートメッセージ
* IRCコマンド対応（USER, PASS, NICK, PRIVMSG, JOIN, KICK, INVITE, MODE, TOPIC, PING, PONG, CAP, QUIT）
* 認証フロー（PASS / NICK / USER）

担当範囲：

* Serverクラス：ノンブロッキングI/O、接続管理、イベントループの実装
* Databaseクラス：クライアント・チャンネル状態の一元管理
* Clientクラス：接続状態管理、バッファ処理、認証フロー
* IRCコマンド：USER, PASS, NICK, PRIVMSG, PING, PONG, CAP, INVITE, MODE, QUITの実装

## 実行環境

* macOS / Linux
* C++98対応コンパイラ

外部ライブラリへの依存なし

## 実行方法

ビルド：
```
$ make
```
サーバー起動：
```
$ ./ircserv <port> <password>
```

クライアントからの接続例（IRCクライアント `nc` を使用）：
```
$ nc localhost <port>
PASS <password>
NICK <nickname>
USER <username> 0 * :<realname>
```

動作検証は`nc`およびIRCクライアント`Irssi`を用いて行った

## 苦労した点

- ノンブロッキングI/Oの実装

	複数クライアントを単一プロセスで扱うため、`poll()` を用いたノンブロッキングI/O多重化を実装した。実装にあたっては、ブロッキング処理を一切行わない設計が必要となり、以下の点で苦労した：

	* ソケットを `O_NONBLOCK` に設定したうえで、`recv()`/`send()` の戻り値に応じて`EAGAIN`/`EWOULDBLOCK`/`EINTR`を判定し、適切に処理する制御
		* `src/Server.cpp` の `Server::Server`, `acceptNewClient()`：ソケットへの`O_NONBLOCK`設定（`fcntl(F_SETFL, O_NONBLOCK)`）
		* `src/Server.cpp` の `readFromSocket()`, `sendAllNonBlocking()`：`recv()`/`send()`の戻り値とerrnoの判定
	* TCPストリームの境界（メッセージが分割または結合されて届く）に対応するため、クライアントごとに送受信バッファを保持し、行末の改行（`\n`）を見つけて1メッセージずつ切り出す処理（`\r`は後段でトリム）
		* `includes/Client.hpp` ：`std::string _buffer` → クライアントごとの受信バッファ
		* `src/Server.cpp` の `extractClientBufferLine()`：バッファからの行単位の切り出しと`\r\n`のトリム

	これらにより、複数クライアントが同時接続している状況でも、特定クライアントの処理が他クライアントをブロックしない構造を実現した。

## 工夫・力をいれた点

* Commandパターンによる拡張性と保守性の両立

	コマンド数の増加に伴う条件分岐の肥大化を避けるため、Commandパターンとファクトリメソッドを採用。各IRCコマンドを共通インターフェースを持つ独立したクラスとして実装し、コマンド追加時の修正範囲を限定する構造とした。新しいコマンドの追加が、既存コードに影響を与えずに行える。
	* `includes/Command/Command.hpp`：抽象基底クラス（純粋仮想関数`execute`）
	* `includes/Command/*.hpp`, `src/Command/*.cpp`：13個のコマンド派生クラス
	* `src/Server.cpp` の `Server::Server`, `createCommandObj`：`_cmd_map`による文字列→ファクトリ関数の登録とディスパッチ

* コマンド処理とネットワークI/Oの責務分離

	コマンド処理とネットワーク処理の責務が混在することを避けるため、コマンド実行結果を共通の構造体で返す設計を採用。「送信内容・送信対象・切断有無」をひとまとまりのデータとして扱うことで、コマンドロジックは「何をすべきか」を返すことに集中し、実際の送信処理はネットワーク層が担う形に分離した。
	* `includes/irc.hpp` の `t_response` 構造体：コマンド実行結果のデータ構造（`reply`/`target_fds`/`sould_send`/`sould_disconnect`）
	* `src/Command/*.cpp`の各`execute`メソッド：`std::vector<t_response>`の生成のみ
	* `src/Server.cpp`の`sendResponses()`/`executeCmdLine()`：`t_response`を受け取り、実際の`send()`呼び出しと切断判定を担当

* RFC 2812に準拠した仕様対応

	RFCの読み込みに加え、IRCクライアント（Irssi）の挙動を検証し、実際の動作を確認した上で実装を進めた。エッジケースの多いコマンド仕様に対し、RFC記述と実装の動作を突き合わせながら対応した。
	* `src/Parser.cpp` の `exec()`：512バイト上限、トレーリングパラメータ抽出、引数15個上限などRFC準拠のパース
	* `src/Command/*.cpp` の各`isValidCmd`/`is_validCmd`メソッド：RFC2812数値レスポンス(461/403/442/482/441等)	に基づくエラー応答

* チーム開発のプロセス設計

	PRテンプレートとGitHub ActionsによるCIを導入し、レビュー観点の統一と品質の担保を行なった。並行開発においても大きな手戻りなく開発を進められる体制を構築した。

## 参考にしたソースファイル

* RFC 2812 - Internet Relay Chat (https://datatracker.ietf.org/doc/html/rfc2812)
	* 参考箇所: IRCコマンドの仕様、メッセージフォーマット、レスポンスコード、パーシング

* man poll(2), man socket(2), man recv(2), man send(2)
	* 参考箇所: ソケットAPI、ノンブロッキングI/O、poll() の使い方
