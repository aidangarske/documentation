# wolfSSL FIPS Ready

FIPS認証の取得が求められるプロジェクトには、wolfSSL FIPS Ready版が役立ちます。wolfSSLソースツリーに含まれるFIPS対応の暗号レイヤーを用いて、FIPS認証を取得する手間を低減できます。FIPS Ready版はFIPS版と同一の構成ですが、認証は取得していません。必要なタイミングでFIPS認証プロセスを実行する準備が整っています。FIPS Ready版にはFIPSコードが組み込まれており、FIPS 140-3に示されるデフォルト・エントリーポイントの実用例やConditional Algorithm Self-tests(CAST)を含んでいます。必要なタイミングで実行環境をテストし、wolfSSLの既存のFIPS 140-3認証に追加、または新たなFIPS140-3認証を申請することですべての手続きが完了します。

FIPS Ready版はオープンソースと商用ライセンスのデュアルライセンスで提供しており、GPLv3ライセンス下で動作検証を行っていただけます。その後商用環境でご利用いただくには、商用ライセンス契約が必要です。

FIPSは複雑なトピックです。このドキュメントをご覧いただいた後、ご質問がありましたらinfo@wolfssl.jpまでお問い合わせください。

## 非FIPS版との違い

wolfCrypt FIPS APIは、FIPS環境内に存在するすべてのアルゴリズム関数のラッパーを提供します。FIPS APIをご利用にならず、非FIPS版にも存在するAPIを用いてコードを記述いただくことも可能です。いずれの場合でも、コンパイル時に自動的にwolfCrypt FIPS APIに置換します。FIPS APIは、実際の関数を呼び出す前に内部ステータスのセルフテストを行います。そして各アルゴリズムの初回実行時にCASTが実行します。このようなタイミングでの実行を避けたい場合は、起動時に予め各テストを実行することも可能です。

wolfCrypt FIPS 140-3 Ready版のコードには、メモリ内の実行可能ファイルの整合性を自動的にチェックする必須の電源投入時自己テスト (POST) が含まれています。これにはアルゴリズムのKnown Answere Tests (KAT) が含まれており、140-2 以降変更しておりません。 POST で使用しないものは、使用が条件付きになりました。実行可能ファイルは、FIPS環境内のコードがメモリ内で隣接するように設計しています。FIPSコードを使用するアプリケーションが起動するか、共有ライブラリがロードされると、ライブラリのデフォルトエントリーポイントを呼び出し、POSTを自動的に実行します。これには、コア内のメモリチェックとPOSTで使用するアルゴリズム(HMAC-SHA256)のためのKATの2つが含まれます。

まずHMAC-SHA256のPOSTを実行し、次いでコア内メモリテストを実行します。メモリ内のコードはHMAC-SHA256でハッシュを取り、一致した場合のみテストを続行します。一致しなければFIPSモジュールはエラー状態となり、整合性が取れるまですべての呼び出しは成功しません。

FIPS環境内の他のアルゴリズムは、初回使用時または開発者が指定したタイミングで既定のデータを用いてテストします。出力は事前に計算された回答と比較します。テスト値はすべてFIPS環境内に存在し、呼び出された際にチェックします。署名と検証などのいくつかのテストではランダム性を含むため、既知のデータを署名し既知のデータで検証します。鍵の生成も同様にテストします。
