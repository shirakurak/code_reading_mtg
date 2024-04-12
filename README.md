以下は、社内勉強会で実施した内容を、社外に共有するために作成している文章です。

社内勉強会のメンバー

- [tomohiko9090](https://github.com/tomohiko9090)
- [lmi-yumin](https://github.com/lmi-yumin)
- [shirakurak](https://github.com/shirakurak)

---

# Ruby on RailsのActiveRecordを読む〜マイグレーションってどんなふうに実装されてるの？〜

\#冒険
\#コードを読む会

こんにちは！ 私（達）は Ruby on Rails でプロダクト開発をしているエンジニアです。
日々、Ruby on Rails を使用していると、（その多機能さゆえ）「内部で何が起きてるんだろう...？🤔」と不安になることがよくありました。
その不安を解消しつつ、信頼性の高い情報源を読めるようになりたい！と思い、OSSを読む社内勉強会を実施することになりました。

何を読んでいくか考えた末、Ruby on Rails の中でもデータベースのスキーマを管理することができる Migrations を対象としました。
いろいろ難しく、完全に理解したとはとても言えないですが、どのようにOSSを冒険していったかを共有できたらと思います。

## 🧚 想定読者

- Railsを使えるだけでなく、Railsの仕組みに興味がある人！
- ActiveRecordの実装に興味があるけど、何から手をつければいいか分からない人！
- 将来的にOSSのコントリビューターになってみたい人！

## 🕵️‍♀️ 調査内容

### 調査対象とするマイグレーション処理

マイグレーションにも複数の機能がありますが、今回は、対象を絞り、以下のコマンドの流れを（ふんわりとでも）理解することを目標としました。

```sh
rails db:migrate VERSION=20220808075632
```

本コマンドは、特定のバージョン番号に対応するマイグレーションをターゲットとし、データベースマイグレーションを実行します。マイグレーションファイルに記載されたデータベースのテーブルを作成、変更、削除することになりますが、他にも

- db/schema.rb のスキーマファイルを更新
- schema_migration テーブルにタイムスタンプのレコードを追加

などの処理が行われます。これらを含め、処理を見ていこうと考えました。

## 🧗‍♀️ 冒険内容

### STEP1. 全体像の把握

![image](https://github.com/shirakurak/code_reading_mtg/assets/66200485/a495fd7d-ad00-4203-a6ba-b6e2797e0c4b)

#### ざっくりディレクトリ構成を見てみる

[こちら](https://github.com/rails/rails)から確認します。

<img width="800" alt="スクリーンショット 0006-04-01 9 19 02" src="https://github.com/shirakurak/code_reading_mtg/assets/66200485/d364f275-10ec-4332-abff-4d345bd9b8a9">

- Action View
- Active Model
- Active Support

など、Rails の主要な機能（に対応するディレクトリ）の名前が並んでいますね。
他のディレクトリを見るとRailsの興味深い実装があちらこちらあって、浮気しちゃいそうになります😇

その気持ちをグッと堪えて、activerecord ディレクトリを見ていきます。

`activerecord` 配下で、マイグレーションしてそうなディレクトリとファイルを発見しました。

<img width="777" alt="スクリーンショット 0006-04-01 17 16 43" src="https://github.com/shirakurak/code_reading_mtg/assets/66200485/cfa4e646-6f59-418a-99f5-09803acbf3b3">

### STEP2.ファイルの中身を確認

![image](https://github.com/shirakurak/code_reading_mtg/assets/66200485/d4e0288a-982f-4140-a4b1-518c9a844a20)

#### migration/を見てみる

まずどこから読めばいいかさっぱりだったので、それっぽいディレクト名を探して読んでいこうと考えました。 `activerecord/lib/active_record` ディレクトリ配下に `migration` というディレクトリを発見しました。ここ読めばいいんじゃね？と短絡的に考え、ざっと中身を見てみます。

- `command_recorder.rb`
  - マイグレーション中に行われたコマンドを記録し、それらを逆転させられる（マイグレーションのrevert）ようにする処理が書かれている？
- `compatibility.rb`
  - Railsのバージョンが違う場合でもマイグレーションできるようにする（後方互換性？）処理が書かれている？
- `default_strategy.rb`
  - マイグレーション実行のための抽象クラスの役割を持っている？
- `execution_strategy.rb`
  - 異なるマイグレーションを実行するときの処理が書かれている？
- `join_table.rb`
  - joinテーブル（中間テーブル）の名前のための処理が書かれている？
- `pending_migration_connection.rb`
  - 一時的なデータベース接続プールを作成するための機能を提供するための処理が書かれている？

うーん、マイグレーションに関係するどこか処理ではありそうですが、本来の目的はここを読むだけでは達成できなそうです🙅‍♂️

#### `migration.rb` を見てみる

他のディレクトリは何かあるかなぁとみていくと、 `migration.rb` というファイルを見つけました。中身はざっとこういう構成です：

- Error系のクラス
- Migrationクラス
- MigrationContextクラス
- Migratorクラス

どうやらこれは怪しそうだ。。👀

### STEP3. キーワード検索

![image](https://github.com/shirakurak/code_reading_mtg/assets/66200485/ea1d5bc2-0b20-465a-8420-248d85576fcc)

「主要なワードで全検索してみたらいんじゃない？」という話になった（最初からそうした方が早かったやん...と思いつつ）ので、検索してみることにしました。

リポジトリ内で`.`を押すと、ブラウザ上でVSCode開くことができるので、そこで、「schema_migations」を検索してみます。

<img width="1290" alt="スクリーンショット 0006-04-01 17 45 19" src="https://github.com/shirakurak/code_reading_mtg/assets/66200485/54f66632-863f-4005-a8e2-de4c6a2a91f1">

すると、いくつかファイルがヒットし、ヒットしたファイルの中には、

怪しそうだった`activerecord/lib/active_record/migration.rb`のファイルが！👏

クラスとそのクラスに定義されているメソッドを読むことにしました。

<img width="450" alt="スクリーンショット 0006-04-05 3 01 42" src="https://github.com/shirakurak/code_reading_mtg/assets/66200485/fec5da43-87bb-47bc-976c-6498a8bdacce">

各クラスを確認すると、

upメソッドやdownメソッドなどが定義されていたり、migrateメソッドを実行したりしていたMigrationContextクラスにマイグレーション系の処理を行っていそうということが分かりました。

その後、メソッドを検索して辿って読んでいくと、

`activerecord/lib/active_record/railties/databases.rake` のファイルで、実行したコマンドに関するタスクが実行されていることが分かりました👏

### STEP4. Rakeタスクの実行

![image](https://github.com/shirakurak/code_reading_mtg/assets/66200485/1abb443c-0f45-4254-940d-01dee2e1caa1)

コマンド実行から、schema_migrationテーブルに日付がインサートされるまでの流れをみていきます。

#### db:migrateを実行してからの流れを確認する

```ruby
$ rails db:migrate VERSION=20220808075632
```

が実行されると、`activerecord/lib/active_record/railties/databases.rake`ファイル中で定義されているdb:migrateのRakeタスクが走ることが分かりました。

migrateだけでなく、status、rollback、versionなど、見たことがあるコマンドたちも見つかりました🌝

確認のため、versionのタスクについてputsの文字列が出力されるかみてみました。

activerecord/lib/active_record/railties/databases.rake

```activerecord/lib/active_record/railties/databases.rake
  desc "Retrieve the current schema version number"
  task version: :load_config do
    ActiveRecord::Tasks::DatabaseTasks.with_temporary_pool_for_each(env: Rails.env) do |pool|
      puts "\ndatabase: #{pool.db_config.database}\n"
      puts "Current version: #{pool.migration_context.current_version}"
      puts
    end
  end
```

```
$ rails db:version
Running via Spring preloader in process 26
Current version: 20220808075632
```

ちゃんと書かれていることが確認できました！🙌

ちなみに、:load_configは、`config/database.yml`ファイルのデータベース設定を読み込んで、データベースを接続するための準備をしています。

それでは、本題のdb:migrateを辿ります。

activerecord/lib/active_record/railties/databases.rake

```activerecord/lib/active_record/railties/databases.rake
  desc "Migrate the database (options: VERSION=x, VERBOSE=false, SCOPE=blog)."
  task migrate: :load_config do
    db_configs = ActiveRecord::Base.configurations.configs_for(env_name: ActiveRecord::Tasks::DatabaseTasks.env)

    if db_configs.size == 1
      ActiveRecord::Tasks::DatabaseTasks.migrate
    else
      mapped_versions = ActiveRecord::Tasks::DatabaseTasks.db_configs_with_versions

      mapped_versions.sort.each do |version, db_configs|
        db_configs.each do |db_config|
          ActiveRecord::Tasks::DatabaseTasks.with_temporary_connection(db_config) do
            ActiveRecord::Tasks::DatabaseTasks.migrate(version)
          end
        end
      end
    end

    db_namespace["_dump"].invoke
  end
```

データベースが複数ある場合、ない場合で分岐されているようです。

```sh
ActiveRecord::Tasks::DatabaseTasks.migrate(version)
```
ここでmigrateメソッドを実行しています！

この時、versionsはソートされていることも確認できますね。

では、DatabaseTasksクラスが定義されている所を見ていきましょう。

activerecord/lib/active_record/tasks/database_tasks.rb ー①

```activerecord/lib/active_record/tasks/database_tasks.rb
      def migrate(version = nil)
        scope = ENV["SCOPE"]
        verbose_was, Migration.verbose = Migration.verbose, verbose?

        check_target_version

        migration_connection_pool.migration_context.migrate(target_version) do |migration|
          if version.blank?
            scope.blank? || scope == migration.scope
          else
            migration.version == version
          end
        end.tap do |migrations_ran|
          Migration.write("No migrations ran. (using #{scope} scope)") if scope.present? && migrations_ran.empty?
        end

        migration_connection_pool.schema_cache.clear!
      ensure
        Migration.verbose = verbose_was
      end
```

```ruby
migration_connection_pool.migration_context.migrate(target_version) do |migration|
...
```

によって、データベースに接続したあと、migration_contextメソッドが呼び出され、MigrationContextクラスのmigrateメソッドが呼び出されることが分かりました。

### STEP5.マイグレーション処理の特定

![image](https://github.com/shirakurak/code_reading_mtg/assets/66200485/a251ffd3-00c9-4c34-ad74-81073962f8a9)

MigrationContextクラスを辿っていきます🫡

activerecord/lib/active_record/migration.rb

```activerecord/lib/active_record/migration.rb
Class MigrationContext
  ・・・
    def migrate(target_version = nil, &block)
      case
      when target_version.nil?
        up(target_version, &block)
      when current_version == 0 && target_version == 0
        []
      when current_version > target_version
        down(target_version, &block)
      else
        up(target_version, &block)
      end
    end
```

migrateメソッドでは、target_versionによる分岐が行われており、activerecord/lib/active_record/tasks/database_tasks.rbファイルのtarget_versionメソッドで渡されているENV["VERSION"]、

つまり、コマンドで指定した日付（VERSION=20220808075632）を使っているようです。

その後、target_versionによって分岐され、同じクラス内のupメソッドやdownメソッドが実行されると、

activerecord/lib/active_record/migration.rb

```activerecord/lib/active_record/migration.rb
Class MigrationContext
  ・・・
    def up(target_version = nil, &block) # :nodoc:
      selected_migrations = if block_given?
        migrations.select(&block)
      else
        migrations
      end

      Migrator.new(:up, selected_migrations, schema_migration, internal_metadata, target_version).migrate
    end
```

&blockによって、マイグレーションするファイルを決定します。

その後、Migratorをインスタンス化して、migrateメソッドを実行します。

activerecord/lib/active_record/migration.rb

```activerecord/lib/active_record/migration.rb
class Migrator
  ・・・
    def migrate
      if use_advisory_lock?
        with_advisory_lock { migrate_without_lock }
      else
        migrate_without_lock
      end
    end
```

migrateメソッドでは、マイグレーションの実行中にアドバイザリーロックを使用するかどうかで分岐しています。
アドバイザリーロックとは、他のプロセスが同時にマイグレーションを実行することを防ぐロックのことです。

今回は、マイグレーションの処理を見つけられればよいで、アドバイザリーロックがない場合のmigrate_without_lockメソッドを見ていきます。

```ruby
class Migrator
  ・・・
    private
      def migrate_without_lock
        if invalid_target?
          raise UnknownMigrationVersionError.new(@target_version)
        end

        record_environment
        runnable.each(&method(:execute_migration_in_transaction))
      end
```

migrate_without_lockが実行されると、execute_migration_in_transactionメソッドが最後に実行されていることがわかったので、

最後にこのメソッドを追います！🚴‍♂️

activerecord/lib/active_record/migration.rb

```activerecord/lib/active_record/migration.rb
Class Migrator
  ・・・
    private
      def execute_migration_in_transaction(migration)
          return if down? && !migrated.include?(migration.version.to_i)
          return if up?   &&  migrated.include?(migration.version.to_i)
  
          Base.logger.info "Migrating to #{migration.name} (#{migration.version})" if Base.logger
  
          ddl_transaction(migration) do
            migration.migrate(@direction)
            record_version_state_after_migrating(migration.version)
          end
        rescue => e
          msg = +"An error has occurred, "
          msg << "this and " if use_transaction?(migration)
          msg << "all later migrations canceled:\n\n#{e}"
          raise StandardError, msg, e.backtrace
        end
```

upメソッドやdownメソッドなどによって、テーブルが更新され、schema_migrationsにインサートされているようです。

この後は、①のmigration_connection_pool.schema_cache.clear!によって、変更されたスキーマが正確に反映されます。

かなり端折ったところもありますが、

1. schema_migrationテーブルに履歴のない、マイグレーションが実行
1. db/schema.rbのスキーマファイルを更新
1. schema_migrationテーブルにタイムスタンプのレコードを追加

の3つの処理が実行される流れを追うことができました💪

## まとめ
実際にActiveRcordの中身を読んでみて、**「OSSも意外に読める！」** ということが分かりました。

これまで、「OSSはなんか凄そう...」「きっと魔法のようなコードが書かれているんだろう...」と思っていました。

しかし、実際に読んでみると、普通にプロダクト開発しているコードと同じように、難しいところもあれば、わかりやすいところもありました。

加えて、コード変更した履歴が1ヶ月前にバージョンも更新されている行などもあり、人間味を感じられて面白かったです。

**つまり、魔法は使ってなかったのです。**

OSSだからといって、特別視する必要はないと思えたことが一番の収穫だと思っています。

今後は、もっと積極的にOSSの世界に関わっていくことで、技術的な成長だけでなく、世界中の開発者とのつながりも深めていけることを願っています。

OSSの旅は、これからが本当の始まりです🚀

## 追記: 社内勉強会のルール

昨年、社内のバックエンドエンジニア数人で、プロダクトのコードを読むという勉強会を実施しており、その中で最終アウトプットとして、リファクタリングを行いました。

実際やってみると、技術の伝承が行われるだけでなく、似た機能の実装スピードが上がったりなど、案外すぐ役に立つ良い機会となっていました。

本勉強会はその続きという立ち位置で、次はOSSを読むことにしようという流れで始まりました。

勉強会で、気を付けていたこととして

1. 現実的な目標を立てる
2. 準備は一切しない
3. 3ヶ月以内になんらかのアウトプットを出す

ということがあります。

### 1. 現実的な目標

OSSのコードを読むとなると、いくらでも深く読んでいけてしまうため、現実的（かつライト）な目標をたてて、読んでいこうと決めました。
具体的には、schema_migarationsテーブルにレコードがインサートされるまでの流れを確認することを目標としました。
ある程度流れがわかっていたので、「なんとなく頑張れば理解できそう」というレベルに設定したのは、
**モチベーション管理**の意味でもとても良かったなと思います。

### 2. 準備は一切しない

勉強会をやるとなると、参加前の準備が大変で、
気づくと参加する人がいなくて、崩壊することがしばしば...
そこで僕たちの勉強会では、**参加し続けることを一番に置き**、準備は一切しないことにしました。

### 3. 3ヶ月以内になんらかのアウトプットを出す

**締切がないと人間必死になれません。。**
集中して、最後は達成感を味わいたい！ということから、定めたのですが、
勉強会最後は、「間に合わせるにはどうやって呼んでいくか？」という発想ができたり、
「どうにかアウトプットを生み出そう」いう力学が働いたりして良かったです。
