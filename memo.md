
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
