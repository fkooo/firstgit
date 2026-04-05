# Tomcat 9 から Tomcat 10 への移行計画（Java 11・Spring Framework維持版）

## 1. 背景と重要な制約
本計画は、ヒアリングによって判明した「Java 11を維持する」「Spring Bootではなく素のSpring Frameworkを使用している」という前提条件に基づき、Tomcat 10への移行を実現するためのアプローチを定めたものです。

### 【前提知識】Java EE / Jakarta EE のバージョンとパッケージ名の関係性
今回の移行において最大の障壁となる「パッケージ名の変更」は、以下の歴史的背景に起因しています。
* **Java EE 8まで**: Oracle社が管轄しており、標準APIは長らく `javax.*` パッケージ（例：`javax.servlet`）で提供されていました。
* **Jakarta EE 8**: 管理がEclipse Foundationに移管され「Jakarta EE」という名称に変わりましたが、内容はJava EE 8と完全互換であり、パッケージ名も `javax.*` のままでした。
* **Jakarta EE 9以降 (重要)**: 権利上の理由から旧来のパッケージ名が利用できなくなり、**すべてのAPIパッケージが `javax.*` から `jakarta.*` へと一斉に変更（破壊的変更）**されました（※一般に「ビッグバン・マイグレーション」と呼ばれています）。
  * 公式参考 (Jakarta EE FAQ - なぜパッケージ名が変わったのか): [Jakarta EE FAQ](https://jakarta.ee/about/faq/)
    * ※上記FAQ内に「_Jakarta EE 9 release introduced the jakarta.* namespace as a replacement for javax.* for Jakarta EE specifications._」という記載があり、Jakarta EE仕様として `javax.*` から `jakarta.*` への直接的な置き換えが行われたことが明記されています。
  * 公式参考 (Jakarta EE 9 リリース仕様): [Jakarta EE 9 Release](https://jakarta.ee/release/9/)
この「Jakarta EE 9（すなわち `jakarta.*` を必須とする仕様）」をベースラインとして作られたのが、今回の移行先である **Tomcat 10** および **Spring Framework 6** です。

### 技術的な課題とジレンマ（なぜソースコードを書き換えないのか）
* **現行システム（Tomcat 9 / Spring 5）の仕様**: まず大前提として、現在のアプリケーション基盤であるTomcat 9およびSpring Framework 5系は、旧来の「Java EE 8 (Jakarta EE 8)」仕様に基づいて構築されており、基幹APIがすべて `javax.*` パッケージであることを前提としています。
  * Tomcat 9 の仕様 (Java EE 8準拠): [Apache Tomcat Versions (Which Version?)](https://tomcat.apache.org/whichversion.html)
  * Spring 5.x の仕様 (Java EE 7/8 (`javax` API) 依存): [Spring Framework Overview (5.3.x Reference Documentation)](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/overview.html#spring-introduction)
* **Tomcat 10の要件**: 内部仕様が「Jakarta EE 9+」に変わったため、デプロイされるアプリケーションは `jakarta.*` パッケージで動作する必要があります。
  * 公式参考: [Apache Tomcat Versions (Which Version?)](https://tomcat.apache.org/whichversion.html)
* **Spring Frameworkの要件**: Spring Frameworkにおいて `jakarta.*` にネイティブ対応しているのは「Spring Framework 6」からです。しかし、**Spring Framework 6 は「Java 17以上」の実行環境が絶対に必要（Baseline）** となります。
  * 公式参考: [A Java 17 and Jakarta EE 9 baseline for Spring Framework 6 (Spring Blog)](https://spring.io/blog/2021/09/02/a-java-17-and-jakarta-ee-9-baseline-for-spring-framework-6)
* **結論**: 「Java 11を維持する」という絶対要件があるため、Spring Framework 6へのアップグレードは不可能です。すなわち、**アプリケーションのソースコード自体を `javax.*` から `jakarta.*` に書き換えることはできません**（Spring 5系でコンパイルエラーや動作エラーとなるため）。

## 2. 移行方針：Tomcat Migration Tool for Jakarta EE の活用
上記の問題を解決するため、ソースコードは現行のまま（Java 11 + Spring Framework 5.x + `javax.*`）維持し、Tomcat本体が提供する公式ツール **[Tomcat Migration Tool for Jakarta EE](https://tomcat.apache.org/download-migration.cgi)** に依存した移行方針を採ります。

具体的には、以下のいずれかのアプローチを選択します。

## > [!IMPORTANT]
## User Review Required
以下のどちらのアプローチを実運用（本番環境やローカル開発環境）に採用するか、ご検討と承認をお願いします。

*   **アプローチA（推奨）: ビルド段階での自動変換 (Build-time Migration)**
    *   **概要**: Maven（またはGradle）のビルドプロセスに「Tomcat Migration Tool」のプラグイン（例：`jakartaee-migration-maven-plugin`）を組み込みます。
    *   **運用**: ビルドの最終段階で生成された `javax` 依存のWARファイルが、自動的に `jakarta` 仕様のWARファイルへ変換され出力されます。これをTomcat 10の `webapps` にデプロイします。
    *   **メリット**: ランタイムのパフォーマンス影響がなく、Tomcat側の設定変更が不要です。
*   **アプローチB: デプロイ段階での動的変換 (Deploy-time Migration)**
    *   **概要**: Tomcat 10 の `server.xml` にレガシー用のデプロイフォルダ（例：`<Host legacyAppBase="webapps-javaee">`）を定義します。
    *   **運用**: 開発チームは今のままビルドされたWARファイルを、この `webapps-javaee` フォルダに配置します。するとTomcat 10が起動時にWARを検知し、メモリ上/裏側で動的に変換を行ってデプロイします。
    *   **メリット**: ビルドプロセスの変更すら不要で、既存のデプロイ先のフォルダ指定を変えるだけで対応が完了します。
    *   **デメリット**: Tomcatの起動時・デプロイ時に変換処理が走るため、起動速度がわずかに低下します。

## 3. 移行フェーズと具体的な作業手順（アプローチAを想定）

### フェーズ 1: 概念実証 (PoC - Proof of Concept)
1.  **ツールの手動実行確認**
    - 現在のパイプラインで作成された既存のWARファイルを用意します。
    - Migration ToolのCLI（コマンドライン）版を用いて、ローカル上でWARファイルを手動変換します。
2.  **Tomcat 10での配備・起動検証**
    - 変換されたWARファイルをローカルのTomcat 10へデプロイし、Spring Framework 5のアプリケーションが正常に起動すること、エラー無くCRUD機能が動作することを確認します。

### フェーズ 2: ツール導入・ビルドプロセス更新
（アプローチAを採用した場合）
1.  **pom.xml の編集**
    - [MODIFY] プロジェクトの `pom.xml` に対して、ビルドフェーズの終端にマイグレーションを実行するプラグイン（Eclipse Transformer / Tomcat Migration Tool）を追加します。
    - **※ソースコード内の `.java` ファイルやアノテーション等に対する影響はゼロ（一切変更しない）です。**

### フェーズ 3: リグレッションテスト
1.  **動作結合テスト**
    - Tomcat 10上での動作において、変換漏れ（JSPやカスタムタグ内に埋もれたパッケージ名など）に起因する `ClassNotFoundException` が発生しないか、主要な画面とバッチ処理等について検証を行います（E2Eを含む手動・自動テスト）。


# Java 21 バージョンアップ時の評価内容

Java バージョンを 21（現在のLTS）にバージョンアップする際に評価すべき重要なポイントについて、公式ドキュメントの情報をもとに整理しました。

## 1. 公式ドキュメント（リファレンスURL）

移行作業の際は、まずは以下の公式ドキュメントおよびリリースノートに目を通し、対象システムに該当する変更がないかを確認してください。

*   **Oracle JDK 21 Migration Guide (公式移行ガイド)**
    [https://docs.oracle.com/en/java/javase/21/migrate/index.html](https://docs.oracle.com/en/java/javase/21/migrate/index.html)
*   **Oracle JDK 21 Release Notes (リリースノート)**
    [https://www.oracle.com/java/technologies/javase/21-relnote-issues.html](https://www.oracle.com/java/technologies/javase/21-relnote-issues.html)
*   **OpenJDK 21 - 対象となるJEP (Java Enhancement Proposal) 一覧**
    [https://openjdk.org/projects/jdk/21/](https://openjdk.org/projects/jdk/21/)

---

## 2. バージョンアップに伴う主要な評価項目

### 2.1. サードパーティライブラリとフレームワークの互換性
Java自体の仕様変更よりも、利用している外部ライブラリ（Spring Boot、Lombok等のバイトコード操作ライブラリ、各種ミドルウェアのドライバ等）がJava 21に対応しているかどうかが最大の障壁となります。
*   **Spring Boot:** Spring Boot 3.2 以降で Java 21 (および仮想スレッド) に正式対応しています。
*   **ビルドツール:** Maven および Gradle の利用バージョンが Java 21 のコンパイルに対応しているか評価およびアップグレードが必要です。
*   **評価方法:** `jdeps` などの依存関係分析ツールを使用し、内部APIなど非互換な仕組みを利用しているライブラリを特定します。
    *(参考: [Migration Guide - Use jdeps to Find Dependencies on JDK Internal APIs](https://docs.oracle.com/en/java/javase/21/migrate/getting-started.html))*

### 2.2. デフォルトの文字エンコーディングの変更（UTF-8 by Default）
Java 18 から、デフォルトの文字セットが環境依存（WindowsならMS932等）から **UTF-8** に変更されました。
*(参考: [JEP 400: UTF-8 by Default](https://openjdk.org/jeps/400) / [Migration Guide - Default Charset Details](https://docs.oracle.com/en/java/javase/21/migrate/significant-changes-java-se-18.html))*
Java 11 や 17 から 21 へ飛ぶ場合に非常に影響を受けやすいポイントです。
*   **評価内容:** ファイル入出力（`FileReader`/`FileWriter` 等）や `String.getBytes()` などの処理で意図せず文字化けが発生しないか検証が必要です。
*   **対応策:** 互換性維持のため、一時的にJVM起動オプションに `-Dfile.encoding=COMPAT` を指定する回避策も検討可能です。

### 2.3. 非推奨・削除されたAPIおよびツールの影響確認
長期間非推奨であったAPIの多くが削除されたり、デフォルトで無効化されたりしています。
*   **Security Manager の非推奨化:** Java 17ですでに非推奨でしたが、関連APIを使用している場合、将来的な対応を含め代替実装の検討が必要です。
    *(参考: [JEP 411: Deprecate the Security Manager for Removal](https://openjdk.org/jeps/411))*
*   **削除されたAPI一覧:** [Migration Guide - Removed APIs, Tools, and Components](https://docs.oracle.com/en/java/javase/21/migrate/removed-tools-and-components.html) を確認し、独自コードでの使用がないか `jdeprscan` ツール（非推奨API使用箇所スキャンツール）でコードベースを検査してください。
    *(参考: [jdeprscan Tool Reference](https://docs.oracle.com/en/java/javase/21/docs/specs/man/jdeprscan.html))*

### 2.4. JVM・GC（ガベージコレクション）のパフォーマンス変化評価
Java 21では ZGCの実装など、GCチューニングに影響を及ぼす機能が追加されています。
*(参考: [JEP 439: Generational ZGC](https://openjdk.org/jeps/439) / [Migration Guide - Garbage Collection Updates](https://docs.oracle.com/en/java/javase/21/migrate/significant-changes-java-se-21.html))*
*   **評価内容:** G1GCなど既存のGCを利用している場合も、内部最適化によりレイテンシやメモリフットプリントが変わるため、本番相当の負荷テストで性能・メモリプロファイルの再評価が必要です。

### 2.5. 新仕様の導入によるアーキテクチャの評価
機能として導入されたものを利用するかどうか（特に仮想スレッド）の評価です。
*   **仮想スレッド (Virtual Threads):** 
    *(参考: [JEP 444: Virtual Threads](https://openjdk.org/jeps/444))*
    高いスループットの並行処理を簡単に実装できる強力な新機能です。これまでのスレッドプールを用いた既存の処理を仮想スレッドベース（`Executors.newVirtualThreadPerTaskExecutor()` など）に移行することで、レイテンシとシステム効率向上が見込めるか評価・検証します。（※既存ライブラリが `ThreadLocal` を多用している場合は枯渇のリスクがあるため、移行には慎重な評価が必要です）

---

## 3. 推奨される検証ステップ

※ 以下のステップは、公式移行ガイドで推奨されているアプローチ（[Getting Started - Identify Potential Issues and Proceed with Migration](https://docs.oracle.com/en/java/javase/21/migrate/getting-started.html)）や機能の変更内容に基づき具体化しています。

1.  **静的コード分析**: `jdeps` および `jdeprscan` コマンドで非互換要素を洗い出す。
2.  **ライブラリアップデート**: 利用中のすべての依存関係を Java 21 サポートの新しいバージョンへと更新する。
3.  **コンパイルと単体テストの実行**: JDK 21 を用いてコンパイルを実行し、既存のUnitテストがパスすることを確認する。
4.  **結合テストの実施（Java 21特有の重点検証項目）**:
    結合テストでは通常の業務シナリオの検証に加え、システム基盤の変更に伴う以下の点に焦点を当ててテストを行います。
    *   **文字エンコーディングの検証 (UTF-8デフォルト化の影響)**:
        *   CSVなどのファイル入出力で、SJISやEUC-JPなどを想定していた処理で文字化けが発生していないか。
        *   マルチバイト文字を含むHTTPリクエスト/レスポンス、データベースへの読み書きが正常に行えるか。
    *   **言語仕様・コアAPIの変更に伴うロジック検証 (重要)**:
        *   **暗黙のソート順・コレクション処理**: `HashMap` 等は仕様上順序不定ですが、JVMのバージョン変更で実際の取得順序が変わるケースが多々あります。明示的なソート処理が漏れていることによるUI等の表示順変更バグが顕在化していないか確認する。
            *(参考: [HashMap API Documentation](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/HashMap.html))*
        *   **Sequenced Collections の影響**: Java 21の JEP 431 で追加された `getFirst()`, `getLast()` メソッド等と、独自実装したメソッド名が衝突していないか。
            *(参考: [JEP 431: Sequenced Collections](https://openjdk.org/jeps/431))*
        *   **Switch文・パターンマッチング**: Pattern Matching の導入で `switch` 構文の網羅性チェックが厳格化された点や、`null` の扱いの挙動（予期せぬフォールスルーやNPEの発生タイミング変化）に問題がないかの境界値テスト。
            *(参考: [JEP 441: Pattern Matching for switch](https://openjdk.org/jeps/441))*
        *   **レコード (Records)**: 既存DTOからRecordへ移行した場合、Jackson等のJSONパーサーやORM (JPA/MyBatis) でのシリアライズ・デシリアライズ挙動に異常がないか。
            *(参考: [JEP 395: Records](https://openjdk.org/jeps/395))*
    *   **外部ライブラリ・ミドルウェアとの連携確認**:
        *   バージョンを更新したサードパーティライブラリ（JDBCドライバ、各種APIクライアントなど）を使用した外部システム連携が正常に行えるか。
    *   **セキュリティおよび通信プロトコルの変更影響検証 (重要)**:
        *   **暗号化・TLSプロトコル**: セキュリティ向上のための古いプロトコルの無効化（TLS 1.0 / 1.1など）やサポートされる暗号スイートの変更により、接続先の古いWeb APIやデータベース等との通信時に `SSLHandshakeException` 等の接続エラーが発生しないか。
            *(参考: [Migration Guide - Security Updates](https://docs.oracle.com/en/java/javase/21/migrate/security-updates.html))*
        *   **公開鍵・トラストストアの検証**: JDKデフォルトのキーストア (`cacerts`) に含まれるルート証明書のアップデートによる影響や、証明書検証の厳格化に伴う認証エラーがないか。
        *   **Security Manager非推奨の影響確認**: 関連API（権限チェック等）を代替手段へ移行した場合の、ファイルI/Oやネットワーク疎通におけるアクセス拒否・許可の挙動テスト。
    *   **非同期・マルチスレッド処理の挙動確認**:
        *   並行処理においてスレッドプールの挙動に異常（デッドロック等）がないか。
        *   （※Virtual Threadsを導入する場合）`ThreadLocal` を使用した処理のリークや不具合、`synchronized` ブロックによる仮想スレッドのピン留め (Pinning) によるスループット低下がないか。
    *   **フレームワークベースの挙動確認**:
        *   Spring Boot等のバージョンアップを伴う場合、AOP（アスペクト）、トランザクション管理、DIコンテナによる依存注入が期待通り動作しているか。
5.  **パフォーマンステスト**: JVMのパフォーマンスパラメータ（GCのアルゴリズムなど）を見直し、長時間の連続稼働や高負荷状態でメモリリークやレイテンシの悪化が生じないか負荷検証を行う。
