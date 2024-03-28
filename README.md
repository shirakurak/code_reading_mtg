# Active Record Migrations の コードを読む

Active Record Migrations は、Ruby on Rails におけるデータベースのスキーマの管理をする機能です。
今回は、そのコードを実際に読むという勉強会を実施していたので、記事化してみます。

## 勉強会の概要

昨年、プロダクトのコードを何人かで読むという勉強会を実施しており、その中でリファクタリングを行ったりしました。
勉強会により、技術の伝承が行わ流だけでなく、似た機能の実装スピードが上がったりなど、案外すぐ役に立つ良い機会となっていました。

本勉強会はその続きという立ち位置で、次はOSSを読むことにしようという流れで始まりました。

勉強会で、気を付けていたこととして

1. 現実的な目標を立てる
2. 準備をしない
3. 3ヶ月以内になんらかのアウトプットを出す

ということがあります。

### 1. 現実的な目標

OSSのコードを読むとなると、いくらでも深く読んでいけてしまうため、現実的（かつライト）な目標をたてて、読んでいこうと決めました。
具体的には、schema_migarationsテーブルにレコードがinsertされる箇所の実装を確認して、そこまでの流れを確認することを目標としました。
モチベーション管理の意味でも良かったです。

### 2. 準備は一切しない

勉強会に向けて、参加し続けることを一番に置き、準備は一切しないことにしました。

### 3. 3ヶ月以内になんらかのアウトプットを出す

締め切り効果みたいな感じで、どうにかアウトプットを生み出そういう力学が働いてよかったです

## 実際に読んでいった内容

ここから追いかけていった内容を記載していきます。
ダラダラとコードの内容があるので、最後のまとめだけ読みたい方はこちら。

### STEP1. ざっくりディレクトリ構成を確認



### STEP2. schema_migationsで検索

何はともあれ、検索することから始めます。
`.`を押すと、ブラウザ上でvscode開くので便利だよ。





db:migrateの実行の流れをまとめる！

https://github.dev/rails/rails/blob/9e01d93547e2082e2e88472748baa0f9ea63c181/activerecord/lib/active_record/migration.rb





残TODO

下の流れを整理する

migration_connectionがなんだかみる

コマンドから、task up : : load config が呼ばれることの根拠







db:migrateの実行の流れ

コマンド実行、スタート！

$ rails db:migrate VERSION=20080906120000

↓

```
bin/rails db:migrate

namespace :db do
  desc ...
  task migrate: :environment do
    puts "Hello"
  end
end
```

```
rakeタスク
rake greet:say_hello

namespace :greet do
  desc "Helloを表示するタスク"
  task say_hello: :environment do
    puts "Hello"
  end
end
```


多分最初これ：　https://github.com/rails/rails/blob/9e01d93547e2082e2e88472748baa0f9ea63c181/activerecord/lib/active_record/railties/databases.rake#L331 



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

DBの設定値などを読み込んでいる？（保留）

activerecord/lib/active_record/tasks/database_tasks.rb:156

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
