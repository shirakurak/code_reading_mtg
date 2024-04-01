# Railsのマイグレーションについて、ソースコードを理解してみる。
\#冒険
\#コードを読む会

Active Record Migrations は、Ruby on Rails におけるデータベースのスキーマの管理をする機能です。

Ruby on Railsは多機能なフレームワークですが、コマンドやメソッドを使用する際、「何が起きてるんだろう...？🤔」と不安になることがしばしばあります。

ソースコードを実際に読むことで、信頼性の高い情報源を読めるようになりたいと思い、社内勉強会を実施しました。

## 勉強会のルール🕹️

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

## 調べること🕵️‍♀️

```ruby
$ rails db:migrate VERSION=20220808075632
```
を実行すると、

1. schema_migrationテーブルに履歴のない、マイグレーションが実行
1. db/schema.rbのスキーマファイルを更新
1. schema_migrationテーブルにタイムスタンプのレコードを追加

がおきます。
この流れをコードから理解することがゴールです。


## 実際に読んでいった内容📚

ここから追いかけていった内容を記載していきます。
ダラダラとコードの内容があるので、最後のまとめだけ読みたい方はこちら。

### STEP1. ざっくりディレクトリ構成を把握する

https://github.com/rails/rails
ソースコードのツリー構造を確認します。
<img width="2918" alt="スクリーンショット 0006-04-01 9 19 02" src="https://github.com/shirakurak/code_reading_mtg/assets/66200485/d364f275-10ec-4332-abff-4d345bd9b8a9">

ソースコードを見るだけでも、いろんな機能があることがわかります。
他のディレクトリを見るとRailsのよく使う機能がいくつもあって、浮気しちゃいそうになりますね😇

その気持ちをグッと堪えて、activerecordを見ていきます。

activerecord配下で、マイグレーションしてそうなディレクトリとファイルを発見しました。

<img width="777" alt="スクリーンショット 0006-04-01 17 16 43" src="https://github.com/shirakurak/code_reading_mtg/assets/66200485/cfa4e646-6f59-418a-99f5-09803acbf3b3">

### STEP2.ファイルの中身を読んでみる
このステップでは、結局理解できなかったので、読み飛ばしてもらっても大丈夫です！

activerecord/lib/active_record/migration/ディレクトリ配下のファイルを見てみます。

- command_recorder.rb
  - マイグレーション中に行われたコマンドを記録し、それらを逆転させられるようにするファイル？
- compatibility.rb
  - Railsのバージョンが違う場合でもマイグレーションできるようにするファイル？
- default_strategy.rb
  -  マイグレーション実行のためのデフォルトのファイル？
- execution_strategy.rb
  - 異なるマイグレーションを実行するときに使うファイル？
- join_table.rb
  - joinテーブルの作成や削除をサポートするヘルパーメソッドを提供するファイル？
- pending_migration_connection.rb
  - 未実行のマイグレーションが存在するかどうかをチェックするためにDBに接続するためのファイル？

この読み方では流れを理解できないことがわかりました🙅‍♀️


### STEP3. 「schema_migations」で検索してみる

何はともあれ、検索することから始めます。
ソースコードのページで`.`を押すと、ブラウザ上でVSCode開くことができるので、

そこで、検索をしてみます。

「schema_migations」で検索。

<img width="1290" alt="スクリーンショット 0006-04-01 17 45 19" src="https://github.com/shirakurak/code_reading_mtg/assets/66200485/54f66632-863f-4005-a8e2-de4c6a2a91f1">


すると、いくつかファイルがヒットしたのですが、ヒットしたファイルの中で、`activerecord/lib/active_record/migration.rb`

のファイルには、MigrationErrorクラスやMigrationContextクラス、Migratorクラスなどがあったので、

クラスとそのクラスに定義されているメソッドを読んでいきました。

<img width="746" alt="スクリーンショット 0006-04-01 17 54 18" src="https://github.com/shirakurak/code_reading_mtg/assets/66200485/ef42a6d8-db43-4325-ac17-f15ff95549be">

すると、MigrationContextクラスには、upメソッドやdownメソッドなどが定義されており、Migratorクラスのインスタンスメソッドであるmigrateを実行していたので、

間違いなく、ここでup、downのマイグレーションを行っていることがわかりました。

ここからは、とにかくメソッドを辿って辿って読んでいくと、流れを掴むことができました！👏

### STEP4. マイグレーションされる流れを整理する
とにかく辿って読む、、辿って読む、、を繰り返していたのですが、

```ruby
rails db:migrate
```
実行後の流れをなんか理解できた気がする！けど、本当に理解できているのか？🤔

となったので、最初のコマンド実行から、schema_migrationテーブルに日付がインサートされる流れを整理してみました。


#### db:migrateの実行の流れをまとめる！

```ruby
rails db:migrate
```

が実行されると、

activerecord/lib/active_record/railties/databases.rakeファイルの
namespaceで定義されたdb:migrateのRakeタスクが実行されます。

migrateだけでなく、status、rollback、versionなど、見たことがあるコマンドも拝見されます。

確認のため、versionのタスクについては、putsで出力している箇所を確認してみました。
```ruby
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
➜  git:(main) ✗ rails db:version
Running via Spring preloader in process 26
Current version: 20220808075632
```
ちゃんと書かれていることが確認できました！

ちなみに、:load_configは、config/database.ymlファイルのデータベース設定を読み込んで、

タスク実行でデータベースを接続する準備をしているようです。

それでは、migrateを辿ります。
```ruby
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

データベースが複数ある場合で分岐されていますが、
```ruby
ActiveRecord::Tasks::DatabaseTasks.migrate(version)
```
ここでマイグレーションしていることには違いなさそうなので、

activerecord/lib/active_record/tasks/database_tasks.rb
を確認します。

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

そして、


https://github.dev/rails/rails/blob/9e01d93547e2082e2e88472748baa0f9ea63c181/activerecord/lib/active_record/railties/databases.rake#L181

desc 'Run the "up" for a given migration VERSION.'
task up: :load_config do
  ActiveRecord::Tasks::DatabaseTasks.raise_for_multi_db(command: "db:migrate:up")

  raise "VERSION is required" if !ENV["VERSION"] || ENV["VERSION"].empty?

  ActiveRecord::Tasks::DatabaseTasks.check_target_version

  ActiveRecord::Tasks::DatabaseTasks.migration_connection.migration_context.run(
    :up,
    ActiveRecord::Tasks::DatabaseTasks.target_version
  )
  db_namespace["_dump"].invoke
end



load_config

https://github.com/rails/rails/blob/9e01d93547e2082e2e88472748baa0f9ea63c181/activerecord/lib/active_record/railties/databases.rake#L11-L28 



参考：  

📝メモ

    def self.valid_version_format?(version_string) # :nodoc:
      [
        MigrationFilenameRegexp,
        /\A\d(_?\d)*\z/ # integer with optional underscores
      ].any? { |pattern| pattern.match?(version_string) }
    end

versionにも _入れて良いぽい！という発見



次回： activerecord/lib/active_record/railties/databases.rake:189

おそらくupが実際に実行されているところを見る！

      ActiveRecord::Tasks::DatabaseTasks.migration_connection_pool.migration_context.run(
        :up,
        ActiveRecord::Tasks::DatabaseTasks.target_version
      )



ActiveRecord::Tasks::DatabaseTasks.check_target_version

それは、これ：　https://github.dev/rails/rails/blob/9e01d93547e2082e2e88472748baa0f9ea63c181/activerecord/lib/active_record/tasks/database_tasks.rb#L290

      def check_target_version
        if target_version && !Migration.valid_version_format?(ENV["VERSION"])
          raise "Invalid format of target version: `VERSION=#{ENV['VERSION']}`"
        end
      end



      ActiveRecord::Tasks::DatabaseTasks.migration_connection.migration_context.run(
        :up,
        ActiveRecord::Tasks::DatabaseTasks.target_version
      )

 定義は以下：

      def migration_connection # :nodoc:
        migration_class.connection
      end



ActiveRecord::Tasks::DatabaseTasks.migration_connection.migration_contextは、以下？

https://github.dev/rails/rails/blob/9e01d93547e2082e2e88472748baa0f9ea63c181/activerecord/lib/active_record/connection_adapters/abstract_adapter.rb#L249

      def migration_context # :nodoc:
        MigrationContext.new(migrations_paths, schema_migration, internal_metadata)
      end



https://github.com/rails/rails/blob/9e01d93547e2082e2e88472748baa0f9ea63c181/activerecord/lib/active_record/migration.rb#L1221 

   def run(direction, target_version) # :nodoc:
      Migrator.new(direction, migrations, schema_migration, internal_metadata, target_version).run
    end



https://github.com/rails/rails/blob/9e01d93547e2082e2e88472748baa0f9ea63c181/activerecord/lib/active_record/migration.rb#L1471 



ここを読んでいく

↓

ActiveRecord::Migration.copy(destination, railties,
                                    on_skip: on_skip, on_copy: on_copy)

https://github.com/rails/rails/blob/9e01d93547e2082e2e88472748baa0f9ea63c181/activerecord/lib/active_record/railties/databases.rake#L637 

activerecord/lib/active_record/railties/databases.rake

task：コマンドに存在しそう？（ db:migrate:prepare など）

namespace：存在しなさそう？

参考：https://docs.google.com/spreadsheets/d/1xDpbBz5ww9_OUMcfLElCvNcKiElpR2IEu3lygbe2Jog/edit#gid=0 



↓

copy

↓

MigrationContext.migrate

https://github.com/rails/rails/blob/1f6cef4ca546b3a9f7aa12c0f10c7d1d1cfbab5a/activerecord/lib/active_record/migration.rb#L1230 

↓

MigrationContext.up または down

https://github.com/rails/rails/blob/9e01d93547e2082e2e88472748baa0f9ea63c181/activerecord/lib/active_record/migration.rb#L1293 

↓

Migrator.migrate

https://github.com/rails/rails/blob/9e01d93547e2082e2e88472748baa0f9ea63c181/activerecord/lib/active_record/migration.rb#L1479 

↓

ロックする

Migrator.migrate_without_lock
または
Migrator.run_without_lock

↓

ここでmigration実行！

sheme_migrationsの更新もここでしてる

Migrator.execute_migration_in_transaction

https://github.com/rails/rails/blob/1f6cef4ca546b3a9f7aa12c0f10c7d1d1cfbab5a/activerecord/lib/active_record/migration.rb#L1525 



migration.rbの全体構成

エラー系

MigrationError < ActiveRecordError

IrreversibleMigration < MigrationError

DuplicateMigrationVersionError < MigrationError

DuplicateMigrationNameError < MigrationError

UnknownMigrationVersionError < MigrationError

IllegalMigrationNameError < MigrationError

InvalidMigrationTimestampError < MigrationError

PendingMigrationError < MigrationError

ConcurrentMigrationError < MigrationError

NoEnvironmentInSchemaError < MigrationError

ProtectedEnvironmentError < ActiveRecordError

EnvironmentMismatchError < ActiveRecordError

EnvironmentStorageError < ActiveRecordError

Migration
MigrationProxy
MigrationContext

migrate

up/down

Migrator

migrate

run

migrate_without_lock（private）

run_without_lock（private）

execute_migration_in_transaction（private）

record_version_state_after_migrating（private）

---

## まとめ

最後に良かったこととして、OSSでも別に読めるな、ということがちゃんとわかったことです。
OSSはなんか凄そうとか、よくわからない実装だとか、あるいは逆に実はめちゃわかりやすいのでは、とかそういうふうに特別視する必要ないということです。
普通にプロダクト開発しているコードと同じように、難しいところもあれば、わかりやすいところもあるしって感じで、人の書いたコードだなと思いました。
