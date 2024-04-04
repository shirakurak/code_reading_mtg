# Ruby on RailsのActiveRecordを読む〜マイグレーションってどんなふうに実装されてるの？〜
\#冒険
\#コードを読む会

ActiveRecordのMigrations は、Ruby on Rails におけるデータベースのスキーマの管理をする機能です。

Ruby on Railsは多機能なフレームワークですが、「何が起きてるんだろう...？🤔」と不安になることがしばしばあります。

OSSを実際に読むことで、信頼性の高い情報源を読めるようになりたいと思い、社内勉強会を実施しました。

## 🧚想定読者
- Railsを使えるだけでなく、Railsの仕組みに興味がある人！
- ActiveRecordの実装に興味があるけど、何から手をつければいいか分からない人！
- 将来的にOSSのコントリビューターになってみたい人！

## 🕵️‍♀️調査内容
### 3つのマイグレーション処理を調べる

```ruby
$ rails db:migrate VERSION=20220808075632
```
を実行すると、

1. schema_migrationテーブルに履歴のない、マイグレーションが実行
1. db/schema.rbのスキーマファイルを更新
1. schema_migrationテーブルにタイムスタンプのレコードを追加

の処理が実行されます。
この流れを理解することがゴールです。


## 🧗‍♀️冒険内容

### STEP1. 全体像の把握
#### ざっくりディレクトリ構成を見てみる


ソースコードのツリー構造を確認します。  

https://github.com/rails/rails  
<img width="2918" alt="スクリーンショット 0006-04-01 9 19 02" src="https://github.com/shirakurak/code_reading_mtg/assets/66200485/d364f275-10ec-4332-abff-4d345bd9b8a9">

ソースコードを見るだけでも、いろんな機能があることがわかります。
他のディレクトリを見るとRailsの興味深い実装があちらこちらあって、浮気しちゃいそうになりますね😇

その気持ちをグッと堪えて、activerecordを見ていきます。

activerecord配下で、マイグレーションしてそうなディレクトリとファイルを発見しました。

![image](https://github.com/shirakurak/code_reading_mtg/assets/66200485/a495fd7d-ad00-4203-a6ba-b6e2797e0c4b)


### STEP2.ファイルの中身を確認
<img width="777" alt="スクリーンショット 0006-04-01 17 16 43" src="https://github.com/shirakurak/code_reading_mtg/assets/66200485/cfa4e646-6f59-418a-99f5-09803acbf3b3">

activerecord/lib/active_recordディレクトリ配下のファイルを見てみます。

#### migration/を見てみる
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

#### migration.rbを見てみる
- Error系のクラス
- Migrationクラス
- MigrationContextクラス
- Migratorクラス

どうやらこれは怪しそうだ。。👀


![image](https://github.com/shirakurak/code_reading_mtg/assets/66200485/d4e0288a-982f-4140-a4b1-518c9a844a20)



### STEP3. キーワード検索

何はともあれ、検索することから始めます。
リポジトリ内で`.`を押すと、ブラウザ上でVSCode開くことができるので、

そこで、「schema_migations」を検索してみます。

「schema_migations」で検索。

<img width="1290" alt="スクリーンショット 0006-04-01 17 45 19" src="https://github.com/shirakurak/code_reading_mtg/assets/66200485/54f66632-863f-4005-a8e2-de4c6a2a91f1">


すると、いくつかファイルがヒットしたのですが、ヒットしたファイルの中で、`activerecord/lib/active_record/migration.rb`

のファイルには、MigrationErrorクラスやMigrationContextクラス、Migratorクラスなどがあったので、

クラスとそのクラスに定義されているメソッドを読んでいきました。

<img width="746" alt="スクリーンショット 0006-04-01 17 54 18" src="https://github.com/shirakurak/code_reading_mtg/assets/66200485/ef42a6d8-db43-4325-ac17-f15ff95549be">

すると、MigrationContextクラスには、upメソッドやdownメソッドなどが定義されており、Migratorクラスのインスタンスメソッドであるmigrateを実行していたので、

間違いなく、ここでup、downのマイグレーションを行っていることがわかりました。

ここからは、とにかくメソッドを辿って辿って読んでいくと、流れを掴むことができました！👏

![image](https://github.com/shirakurak/code_reading_mtg/assets/66200485/ea1d5bc2-0b20-465a-8420-248d85576fcc)


### STEP4. Rakeタスクの実行
とにかく辿って読む、、辿って読む、、を繰り返していたのですが、

```ruby
$ rails db:migrate VERSION=20220808075632
```
実行後の流れをなんか理解できた気がする！けど、本当に理解できているのか？🤔

となったので、最初のコマンド実行から、schema_migrationテーブルに日付がインサートされる流れを整理してみました。

#### db:migrateの実行の流れをまとめる！

```ruby
$ rails db:migrate VERSION=20220808075632
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
$ rails db:version
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


データベースが複数ある場合、ない場合で分岐されているようで、、
```ruby
ActiveRecord::Tasks::DatabaseTasks.migrate(version)
```
ここでマイグレーションしていることには違いなさそうです。

versionsはソートされていることも確認できますね。

activerecord/lib/active_record/tasks/database_tasks.rb
を確認します。
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

そして、
```ruby
migration_connection_pool.migration_context.migrate(target_version) do |migration|
...
```
によって、データベースに接続したあと、migration_contextメソッドが呼び出され、
MigrationContextクラスをインスタンス化し、migrateメソッドを呼び出します。

![image](https://github.com/shirakurak/code_reading_mtg/assets/66200485/1abb443c-0f45-4254-940d-01dee2e1caa1)



### STEP5.マイグレーション処理の特定

MigrationContextクラス
```activerecord/lib/active_record/migration.rb
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
migrateメソッドでは、target_versionによる分岐が行われており、

activerecord/lib/active_record/tasks/database_tasks.rbファイルのtarget_versionメソッドで渡されているENV["VERSION"]、

つまり、コマンドでVERSION指定した日付を使っていることがわかります。

```
$ rails db:migrate VERSION=20220808075632
```

その後、target_versionによって分岐され、
同じクラス内のupメソッドやdownメソッドが実行されると、

MigrationContextクラス
```activerecord/lib/active_record/migration.rb
    def up(target_version = nil, &block) # :nodoc:
      selected_migrations = if block_given?
        migrations.select(&block)
      else
        migrations
      end

      Migrator.new(:up, selected_migrations, schema_migration, internal_metadata, target_version).migrate
    end
```
&blockによって、マイグレーションするファイルを決定すると、Migratorをインスタンス化して、

migrateメソッドを実行します。

Migratorクラス
```activerecord/lib/active_record/migration.rb
    def migrate
      if use_advisory_lock?
        with_advisory_lock { migrate_without_lock }
      else
        migrate_without_lock
      end
    end
```
マイグレーションの実行中にアドバイザリーロックを使用するかどうかで分岐しています。
アドバイザリーロックとは、他のプロセスが同時にマイグレーションを実行することを防ぐことができます。

アドバイザリーロックがない場合は、
```ruby
      def migrate_without_lock
        if invalid_target?
          raise UnknownMigrationVersionError.new(@target_version)
        end

        record_environment
        runnable.each(&method(:execute_migration_in_transaction))
      end
```
migrate_without_lockが実行され、execute_migration_in_transactionメソッドによって、
upメソッドやdownメソッドなどによって、テーブルが更新されて、
schema_migrationsにインサートされていきます。


Migratorクラス
```activerecord/lib/active_record/migration.rb
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

この後は、
①のmigration_connection_pool.schema_cache.clear!によって、
変更されたスキーマが正確に反映されます。
これで、

1. schema_migrationテーブルに履歴のない、マイグレーションが実行
1. db/schema.rbのスキーマファイルを更新
1. schema_migrationテーブルにタイムスタンプのレコードを追加

の3つの処理が実行される流れを追うことができました👏

![image](https://github.com/shirakurak/code_reading_mtg/assets/66200485/a251ffd3-00c9-4c34-ad74-81073962f8a9)


## 🕹️勉強会のルール

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

## まとめ
実際にActiveRcordの中身を読んでみて、**「OSSも意外に読める！」** ということが分かりました。

これまで、「OSSはなんか凄そう...」「きっと魔法のようなコードが書かれているんだろう...」と思っていました。

しかし、実際に読んでみると、普通にプロダクト開発しているコードと同じように、難しいところもあれば、わかりやすいところもありました。

加えて、コード変更した履歴が1ヶ月前にバージョンも更新されている行などもあり、人間味を感じられて面白かったです。


**つまり、魔法は使ってなかったのです。**

OSSだからといって、特別視する必要はないと思えたことが一番の収穫だと思っています。

今後は、もっと積極的にOSSの世界に関わっていくことで、技術的な成長だけでなく、世界中の開発者とのつながりも深めていけることを願っています。

OSSの旅は、これからが本当の始まりです🧗‍♂️













--- 
📝メモ


activerecord/lib/active_record/connection_adapters/abstract/connection_pool.rb
にある、

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
