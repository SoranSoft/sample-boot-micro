sample-boot-micro
---

### はじめに

[Spring Cloud Netflix](https://cloud.spring.io/spring-cloud-netflix/) の仕組みを利用したマイクロサービス実装サンプルです。

[Spring Boot](http://projects.spring.io/spring-boot/) / [Spring Security](http://projects.spring.io/spring-security/) / [Hibernate ORM](http://hibernate.org/orm/) を元にしたマルチプロジェクト下での DDD サンプルとしても利用できます。  

> モノリシックなアプローチについては [sample-boot-hibernate](https://github.com/jkazama/sample-boot-hibernate) を参考にしてください。  

フレームワークとしては作っていないため、 Spring Boot を利用したマルチプロジェクトを立ち上げる際の元テンプレートとして利用される事を期待しています。

考え方の骨子については以前発表した資料 ( [Spring Bootを用いたドメイン駆動設計](http://www.slideshare.net/jkazama/jsug-20141127) ) を参照してください。

UI 側の実装サンプルについては [sample-ui-vue](https://github.com/jkazama/sample-ui-vue) / [sample-ui-react](https://github.com/jkazama/sample-ui-react) などを利用してください。

*※ JavaDoc に記載をしていますが現状のものは参考実装レベルです。製品水準のコードが含まれているわけではありません。*

#### レイヤリングの考え方

オーソドックスな三層モデルですが、横断的な解釈としてインフラ層を考えています。

| レイヤ          | 特徴                                                        |
| -------------- | ----------------------------------------------------------- |
| UI             | ユースケース処理を公開 ( 必要に応じてリモーティングや外部サイトを連携 ) |
| アプリケーション | ユースケース処理を集約 ( 外部リソースアクセスも含む )                 |
| ドメイン        | 純粋なドメイン処理 ( 外部リソースに依存しない )                      |
| インフラ        | DI コンテナや ORM 、各種ライブラリ、メッセージリソースの提供          |

UI 層の公開実装は通常 JSP や Thymeleaf を用いて行いますが、本サンプルでは異なる種類のクライアント利用を想定して RESTfulAPI での API 提供のみをおこないます。 ( 利用クライアントは別途用意する必要があります )

#### Spring Boot の利用方針

Spring Boot は様々な利用方法が可能ですが、本サンプルでは以下のポリシーで利用します。

- 設定ファイルは yml を用いる。 Bean 定義に xml 等の拡張ファイルは用いない。
- インフラ層のコンポーネントは @Bean で、 それ以外のコンポーネントは @Component 等でベタに登録していく。
    - `ApplicationConfig` / `ApplicationDbConfig` / `ApplicationSecurityConfig`
- 例外処理は終端 ( RestErrorAdvice / RestErrorCotroller ) で定義。 whitelabel 機能は無効化。
- JPA 実装として Hibernate に特化。
- Spring Security の認証方式はベーシック認証でなく、昔からよくある HttpSession で。
- 基礎的なユーティリティで Spring がサポートしていないのは簡易な実装を用意。

#### Java コーディング方針

Java8 以上を前提としていますが、従来の Java で推奨される記法と異なっている観点も多いです。  
以下は保守性を意識した上で簡潔さを重視した方針となっています。

- Lombok を積極的に利用して冗長さを排除。
- 名称も既存クラスと重複しても良いのでなるべく簡潔に。
- インターフェースの濫用をしない。
- ドメインの一部となる DTO などは内部クラスで表現。
- Java8 で追加された概念/記法は積極的に利用。

#### プロジェクト構成

Gradle のマルチプロジェクト構成をとっています。

```
gradle                                … Gradle 実行バイナリ
micro-app                             … アプリケーションプロセス ( ドメイン API )
  libs                                … 商用ライブラリ等
  src                                 … アプリケーションコード
micro-asset                           … アプリケーションプロセス ( 資産ドメイン API )
micro-core                            … 共通ライブラリプロジェクト
micro-registry                        … レジストリプロセス ( Eureka Server )
micro-web                             … Web フロントプロセス ( UI向け公開 API )
- build.gradle                        … Gradle プロジェクト定義
- settings.gradle                     … Gradle プロジェクト設定
```

- `micro-app` / `micro-asset` / `micro-web` は `micro-core` に依存しています。
- UI からの要求が `micro-web -> micro-app` / `micro-web -> micro-asset` で処理されるシンプルな構成です。
- 本サンプルにおいて `micro-web` は外部公開する前提のプロセス、それ以外は ( 信頼できる ) 内部に閉じたプロセスとして考えます。
- プロセス間で共有する DB として、テスト環境では `micro-registry` にメモリDB ( H2 ) を立ち上げています。
    - そのため最初に `micro-registry` の起動が必要となります。
- `micro-app` から特定ドメインをマイクロサービス化する例については `micro-asset` を参照してください。
    - 本サンプルでは実装をシンプルに留めるため、 Dto ( Entity) や Facade を `micro-web` 間で使い回しています。
    - 完全に疎なマイクロサービスにする場合は、上記クラスもサブプロジェクト単位で独自に定義していく必要があります。  
      ( RemoteInvocation でなく RestTemplate なのでリモーティング箇所のクラス共有は前提とならない )
    - リリースバージョンが異なったり、実装規模が大きくなる時はリポジトリ分離を検討してください。

> ドメイン要求が肥大化してマイクロサービスを分離していく必要がある時は micro-app プロジェクトをドメイン単位のプロジェクトへと分解していきます。

#### パッケージ構成

パッケージ/リソース構成については以下を参照してください。

```
main
  java
    sample
      api                             … プロセス間で利用される API
      context                         … インフラ層
      controller                      … UI 層
      model                           … ドメイン層
      usecase                         … アプリケーション層
      util                            … 汎用ユーティリティ
      - [プロセス名].java                … 実行可能な起動クラス
  resources
    - application.yml                 … 設定ファイル
    - application-[profile].yml       … プロセス/環境固有設定ファイル
    - ehcache.xml                     … Spring Cache 設定ファイル
    - logback.xml                     … ロギング設定ファイル
    - messages-validation.properties  … 例外メッセージリソース
    - messages.properties             … メッセージリソース
```

### サンプルユースケース

サンプルユースケースとしては以下のようなシンプルな流れを想定します。

- **口座残高 100 万円を持つ顧客**が出金依頼 ( 発生 T, 受渡 T + 3 ) をする。
- **システム**が営業日を進める。
- **システム**が出金依頼を確定する。(確定させるまでは依頼取消行為を許容)
- **システム**が受渡日を迎えた入出金キャッシュフローを口座残高へ反映する。

### 動作確認

本サンプルは [Gradle](https://gradle.org/) を利用しているので、 IDE やコンソールで手間なく動作確認を行うことができます。

*※ライブラリダウンロードなどが自動で行われるため、インターネット接続が可能な端末で実行してください。*

#### サーバ起動 （ Eclipse ）

開発IDEである[Eclipse](https://eclipse.org/)で本サンプルを利用するには、事前に以下の手順を行っておく必要があります。

- JDK8 以上のインストール
- [Lombok](http://projectlombok.org/download.html) のパッチ当て ( .jar を実行してインストーラの指示通りに実行 )

> 以降は Gradle Plugin [ Buildship ] の利用を前提としているため、 Eclipse Mars 以降を推奨します。

次の手順で本サンプルをプロジェクト化してください。  

1. パッケージエクスプローラから 「 右クリック -> Import -> Project 」 で *Gradle Project* を選択して *Next* を押下
1. *Project root directory* にダウンロードした *sample-boot-micro* ディレクトリを指定して *Next* を押下
1. *Import Options* で *Next* を押下
1. *Gradle project structure* でプロジェクトを確認後に *Finish* を押下 ( 依存ライブラリダウンロードがここで行われます )

> Pivotal 版を利用する際ははじめに Build Model ボタンを押下する事を忘れないでください

次の手順で本サンプルを実行してください。

1. *MicroRegistry.java* に対し 「 右クリック -> Run As -> Java Application 」
1. *Console* タブに 「 Started Application 」 という文字列が出力されればポート 8761 で起動が完了
1. ブラウザを立ち上げて 「 http://localhost:8761/ 」 で状態を確認
1. *MicroApp.java* に対し 「 右クリック -> Run As -> Java Application 」
1. *Console* タブに 「 Started Application 」 という文字列が出力されればポート 8090 で起動が完了
1. ブラウザを立ち上げて 「 http://localhost:8090/ 」 で状態を確認 ( Spring Admin )
1. *MicroAsset.java* に対し 「 右クリック -> Run As -> Java Application 」
1. *Console* タブに 「 Started Application 」 という文字列が出力されればポート 8100 で起動が完了
1. ブラウザを立ち上げて 「 http://localhost:8100/api/management/health 」 で状態を確認
1. *MicroWeb.java* に対し 「 右クリック -> Run As -> Java Application 」
1. *Console* タブに 「 Started Application 」 という文字列が出力されればポート 8080 で起動が完了
1. ブラウザを立ち上げて 「 http://localhost:8080/api/management/health 」 で状態を確認

> STS (Spring Tool Suite) のプラグインを利用すると上記 main クラスを GUI の Boot Dashboard 経由で簡単に実行できます。

#### サーバ起動 （ コンソール ）

Windows / Mac のコンソールから実行するには Gradle のコンソールコマンドで行います。  

*※事前に JDK8 以上のインストールが必要です。*

1. ダウンロードした *sample-boot-micro* ディレクトリ直下へコンソールで移動
1. 「 gradlew :micro-registry:bootRun 」 を実行
1. 別コンソールで 「 gradlew :micro-app:bootRun 」 を実行
1. 別コンソールで 「 gradlew :micro-asset:bootRun 」 を実行
1. 別コンソールで 「 gradlew :micro-web:bootRun 」 を実行
1. ブラウザを立ち上げて 「 http://localhost:8080/api/management/health 」 で状態を確認

#### クライアント検証

Eclipse またはコンソールでサーバを立ち上げた後、 test パッケージ配下にある `SampleClient.java` の各検証メソッドをユニットテストで実行してください。

##### 顧客向けユースケース

| URL                              | 処理                 | 実行引数 |
| -------------------------------- | ------------------- | ------------- |
| `/api/asset/cio/withdraw`        | 振込出金依頼          | [`accountId`: sample, `currency`: JPY, `absAmount`: 依頼金額] |
| `/api/asset/cio/unprocessedOut/` | 振込出金依頼未処理検索 | -       |

*※振込出金依頼はPOST、それ以外はGET*

##### 社内向けユースケース

| URL                     | 処理             | 実行引数                                           |
| ----------------------- | --------------- | ------------------------------------------------- |
| `/api/admin/asset/cio/` | 振込入出金依頼検索 | [`updFromDay`: yyyy-MM-dd, `updToDay`: yyyy-MM-dd]|

*※GET*

##### バッチ向けユースケース

| URL                                     | 処理                                          | 実行引数 |
| --------------------------------------- | --------------------------------------------- | ------ |
| `/api/system/job/daily/processDay`      | 営業日を進める(単純日回しのみ)                    | -      |
| `/api/system/job/daily/closingCashOut`  | 当営業日の出金依頼を締める                        | -      |
| `/api/system/job/daily/realizeCashflow` | 入出金キャッシュフローを実現する(受渡日に残高へ反映) | -      |

*※POST*

### 配布用jarの作成

Spring Boot では Executable Jar ( ライブラリや静的リソースなども内包する jar ) を作成する事で単一の配布ファイルでアプリケーションを実行することができます。

1. コンソールから 「 gradlew build 」 を実行
1. サブプロジェクトの `build/libs` 直下に jar が出力されるので Java8 以降の実行環境へ配布
1. 実行環境でコンソールから 「 java -jar xxx.jar 」 を実行して起動
    - サブプロジェクト同士の依存が発生する jar は xxx-exec.jar に読み換えてください

> 実行引数に 「 --spring.profiles.active=[プロファイル名]」 を追加する事で application.yml の設定値を変更できます。

### 依存ライブラリ

| ライブラリ               | バージョン | 用途/追加理由 |
| ----------------------- | -------- | ------------- |
| `spring-cloud`          | Camden.+ | Spring Cloud 基盤 |
| `spring-boot-starter-*` | 1.5.+    | Spring Boot 基盤 (actuator/security/aop/cache/data-jpa/web) |
| `ehcache`               | 3.1.+    | JCache 実装 |
| `HikariCP`              | 2.5.+    | コネクションプーリング実装の組み立て用途 |
| `jackson-datatype-*`    | 2.8.+    | JSON 変換時の Java8 / Hibernate 対応 |
| `commons-*`             | -        | 汎用ユーティリティライブラリ |
| `icu4j-*`               | 58.+   | 文字変換ライブラリ |

> 実際の詳細な定義は `build.gradle` を参照してください

### 補足解説（インフラ層）

インフラ層の簡単な解説です。

※細かい概要は実際にコードを読むか、 「 `gradlew javadoc` 」 を実行して 「 `[subproject]/build/docs` 」 に出力されるドキュメントを参照してください

#### DB / トランザクション

`sample.context.orm` 直下。ドメイン実装をより Entity に寄せるための ORM サポート実装です。 Repository ( スキーマ単位で定義 ) にドメイン処理を記載しないアプローチのため、 Spring Boot が提供する JpaRepository は利用していません。  
トランザクション定義はトラブルの種となるのでアプリケーション層でのみ許し、なるべく狭く限定した形で付与しています。単純な readOnly な処理のみ `@Transactional([Bean名称])` を利用してメソッド単位の対応を取ります。

スキーマは標準のビジネスロジック用途 ( `DefaultRepository` ) とシステム用途 ( `SystemRepository` ) の2種類を想定しています。 Entity 実装ではスキーマに依存させず、引数に渡す側 ( 主にアプリケーション層 ) で判断させます。

> Spring Data JPA が提供する JpaRepository を利用することも可能です。 ( 標準で DefaultRepository が管理しているスキーマへ接続します )

#### 認証/認可

`sample.context.security` 直下。顧客 ( ROLE_USER ) / 社員 ( ROLE_ADMIN ) の 2 パターンを想定しています。それぞれのユーザ情報 ( UserDetails ) 提供手段は `sample.usecase.SecurityService` において定義しています。

認証 / 認可の機能を有効にするには `application-web.yml` の `extension.security.auth.enabled` に `true` を設定してください ( 標準ではテスト用途にfalse ) 。顧客 / 社員それぞれ同一 VM での相乗りは考えていません。社員専用モードで起動する時は起動時のプロファイル切り替え等で `extension.security.auth.admin` を `true` に設定してください。

> 本サンプルでは外部公開フロントである micro-web のみを設定対象として考えています。

#### 利用者監査

`sample.context.audit` 直下。 「いつ」 「誰が」 「何をしたか」 の情報を顧客 /システムそれぞれの視点で取得します。アプリケーション層での利用を想定しています。ログインした `Actor` の種別 ( User / System ) によって書き出し先と情報を切り替えています。運用時に行動証跡を取る際に利用可能です。

#### 例外

汎用概念としてフィールド単位にスタックした例外を持つ `ValidationException` を提供します。  
例外は末端の UI 層でまとめて処理します。具体的にはアプリケーション層、ドメイン層では用途別の実行時例外をそのまま上位に投げるだけとし、例外捕捉は `sample.context.rest` 直下のコンポーネントにおいて AOP を用いた暗黙的差し込みを行います。

#### 日付/日時

`sample.context.Timestamper` を経由して Java8 で追加された `time` ライブラリを利用します。休日等を考慮した営業日算出はドメイン概念が含まれるので `sample.model.BusinessDayHandler` で別途定義しています。

#### キャッシング

`AccountService` 等で Spring が提供する @Cacheable を利用しています。 UI 層かアプリケーション層のどちらかに統一した方が良いですが、本サンプルではアプリケーション層だけ付与しています。  
Hibernate の 2nd / Query キャッシュは Entity 内で必要になる以外、利用しないことを推奨します。

#### リモーティング

Eureka + Ribbon を利用して、シンプルに RestTemplate を用いています。  
プロセス間で連携する API 定義を Facade という形で切り出し、投げ側 ( Invoker ) と受け側 ( Exporter ) に分けてそれぞれ実装しています。  

#### テスト

パターンとしては通常の Spring コンテナを用いる 2 パターン ( WebMock テスト / コンテナテスト ) と、 Hibernate だけに閉じた実行時間に優れたテスト ( Entity のみが対象 ) の合計 3 パターンで考えます。 （ それぞれ基底クラスは `WebTestSupport` / `UnitTestSupport` / `EntityTestSupport` ）  

### License

本サンプルのライセンスはコード含めて全て *MIT License* です。  
Spring Boot を用いたプロジェクト立ち上げ時のベース実装サンプルとして気軽にご利用ください。
