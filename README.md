[![Gem Version](https://badge.fury.io/rb/schema_plus_core.svg)](http://badge.fury.io/rb/schema_plus_core)
[![Build Status](https://github.com/SchemaPlus/schema_plus_core/actions/workflows/prs.yml/badge.svg)](https://github.com/SchemaPlus/schema_plus_core/actions)
[![Coverage Status](https://img.shields.io/coveralls/SchemaPlus/schema_plus_core.svg)](https://coveralls.io/r/SchemaPlus/schema_plus_core)

# SchemaPlus::Core

SchemaPlus::Core creates an internal extension API to ActiveRecord.  The idea is that:

* SchemaPlus::Core does the monkey-patching so clients don't have to know too much about the internal of ActiveRecord.

* SchemaPlus::Core's extension API is consistent across the various connection adapters, so clients don't have to figure out how to extend each connection adapter independently.

* SchemaPlus::Core's extension API intends to remain reasonably stable even as ActiveRecord changes.

By itself, SchemaPlus::Core does not change any behavior or add any external features to ActiveRecord.  It just makes the API available to clients.

SchemaPlus::Core is a client of [schema_monkey](https://github.com/SchemaPlus/schema_monkey), using [modware](https://github.com/ronen/modware) to define middleware callback stacks.


## Compatibility

SchemaPlus::Core is tested on:

<!-- SCHEMA_DEV: MATRIX - begin -->
<!-- These lines are auto-generated by schema_dev based on schema_dev.yml -->
* ruby **2.5** with activerecord **5.2**, using **postgresql:9.6**, **mysql2** or **sqlite3**
* ruby **2.5** with activerecord **6.0**, using **postgresql:9.6**, **mysql2** or **sqlite3**
* ruby **2.7** with activerecord **5.2**, using **postgresql:9.6**, **mysql2** or **sqlite3**
* ruby **2.7** with activerecord **6.0**, using **postgresql:9.6**, **mysql2** or **sqlite3**
* ruby **3.0** with activerecord **6.0**, using **postgresql:9.6**, **mysql2** or **sqlite3**

<!-- SCHEMA_DEV: MATRIX - end -->

<aside class="warning">
As of version 2.0.0, `schema_plus_core` supports only ActiveRecord >= 5.0.
ActiveRecord 4.2.x is supported in version 1.x, maintained in the 1.x
branch.

### Breaking changes in version 2.0 ###

* `SchemaPlus::Core::Schema::Tables` middleware was replaced by `SchemaPlus::Core::DataSources`. Parameters have also changed.
* `SchemaPlus::Core::Dumper::Indexes` middleware was removed; instead `SchemaPlus::Core::Dumper::Table` sets both columns and indexes on env.table.

## Installation

<!-- SCHEMA_DEV: TEMPLATE INSTALLATION - begin -->
<!-- These lines are auto-inserted from a schema_dev template -->
As usual:

```ruby
gem "schema_plus_core"                # in a Gemfile
gem.add_dependency "schema_plus_core" # in a .gemspec
```

<!-- SCHEMA_DEV: TEMPLATE INSTALLATION - end -->


## Usage

The API is in the form of a collection of [modware](https://github.com/ronen/modware) middleware callback stacks.  A client of the API uses [schema_monkey](https://github.com/SchemaPlus/schema_monkey) to insert middleware modules into the stacks.  As per [schema_monkey](https://github.com/SchemaPlus/schema_monkey), the typical module structure looks like:

```ruby
require "schema_plus/core"

module MyClient
  module Middleware
    #
    # Middleware modules to insert in SchemaPlus::Core API stacks
    #
  end
  module ActiveRecord
    #
    # direct ActiveRecord enhancements, should your client need any.
    #
  end
end

SchemaMonkey.register MyClient
```

For example, a client could use the `Migration::Index` stack to automatically make an index unique if any column starts with 'u':

```ruby
require "schema_plus/core"

module AutoUniquify
  module Middleware

    module Migration
      module Index
        def before(env)
          env.options[:unique] = true if env.column_names.grep(/^u/).any?
        end
      end
    end

  end
end

SchemaMonkey.register AutoUniquify
```

Ideally most clients will not need to define direct ActiveRecord enhancements, other than perhaps to create new methods on public classes.  If you have a client that needs more complex monkey-patching, that could be a sign that SchemaPlus::Core's API is missing some useful functionality -- consider submitting a PR to SchemaPlus::Core add it!

## API details

For organizational clarity, the SchemaPlus::Core stacks are grouped into modules based on broad categories.  In the Env field tables below, Initialized

### Schema

Stacks for general operations queries pertaining to the entire database schema:

* `Schema::Define`

  Wrapper around the `ActiveRecord::Schema.define` method loads a dumped schema file (`schema.rb`).

    Env Field    | Description | Initial value
    --- | --- | ---
    `:info` | Schema information hash | *args*
    `:block` | The proc containing the schema definition statements | *args*

  The base implementation calls the block to define the schema.

* `Schema::Indexes`

  Wrapper around the `connection.indexes(table_name)` method.  Env contains:

    Env Field            | Description                                  | Initial value
    -------------------- | -------------------------------------------- | -------------
    `:index_definitions` | The result of the lookup                     | `[]`
    `:connection`        | The current ActiveRecord connection          | *context*
    `:table_name`        | The name of the table to query               | *arg*

  The base implementation appends its results to `env.index_definitions`

* `Schema::DataSources`

  Wrapper around the `connection.data_sources()` method.  Env contains:

    Env Field       | Description                                  | Initialized
    --------------- | -------------------------------------------- | -----------
    `:data_sources` | The result of the lookup                     | `[]`
    `:connection`   | The current ActiveRecord connection          | *context*

  The base implementation appends its results to `env.tables`

### Model

Stacks for class methods on ActiveRecord models.

* `Model::Columns`

  Wrapper around the `Model.columns` query

      Env Field    | Description | Initialized
    --- | --- | ---
    `:columns`     | The resulting Column objects | `[]`
    `:model` | The model Class being queried | *context*

  The base implementation appends its results to `env.columns`

* `Model::ResetColumnInformation`

    Wrapper around the `Model.reset_column_information` method

        Env Field    | Description | Initialized
        --- | --- | ---
        `:model` | The model Class being reset | *context*

    The base implementation performs the reset.

* `Model::Association::Declaration`

    Wrapper around the `Model.has_many`, `Model.has_and_belongs_to_many`, `Model.has_one`, and
    `Model.belongs_to` methods

        Env Field    | Description | Initialized
        --- | --- | ---
        `:model`     | The model Class being defined | *context*
        `:name`      | The name of the association being defined. | *arg*
        `:scope`     | The scope lambda associated with the association | *arg*
        `:options`   | Options associated with the association. | *arg*
        `:extension` | Extensions to the association to be made. | *arg*

    The base implementation creates the association.

### Migration

Stacks for operations that change the schema.  In some cases the operation immediately modifies the database schema, in others the operation defines ActiveRecord objects (e.g., column definitions in a create_table definition) and the actual modification of the database schema will happen some time later.

* `Migration::Column`

  Callback stack for various ways to define or modify a column.

    Env Field    | Description | Initialized
    --- | --- | ---
    `:caller`     | The ActiveRecord instance responsible for performing the action | *context*
    `:operation`  | One of `:add`, `:change`, `:define`, `:record` | *context*
    `:table_name` | The name of the table | *arg*
    `:column_name` | The name of the column | *arg*
    `:type`       | The ActiveRecord column type (`:integer`, `:datetime`, etc.) | *arg*
    `:implements_reference` | This implements a `migration.add_reference`, `t.references` or `t.belongs_to` [See below]| *context*
    `:options`    | The column options | *arg*, default `{}`

  The base implementation performs the column operation.  No value is returned.

  Notes:

  1. The `:operation` field has the following meanings:

      * `:add` - The column will be added immediately (`Migration#add_column`)
      * `:change` - The column will be changed immediately (`Migration#change_column`)
      * `:define` - The column will be added to table definition, which will be emitted later
      * `:record` - The column info will be added to a migration command recorder, for later playback in reverse by `Migration#down`

  2. In the case of a table definition using `t.references` or `t.belongs_to`, the `:type` field will be set to `:reference` and the `:column_name` will include the `"_id"` suffix

  3. ActiveRecord's base implementation handles `migration.add_reference`, `t.references` and `t.belongs_to` by making nested calls to `migration.add_column` or `t.column` to create the resulting column (or two columns, for polymorphic references).  SchemaPlus::Core invokes the `Migration::Column` stack for both the outer `migration.add_reference`, `t.references` or `t.belongs_to` call, as well as for the nested `migration.add_column` or `t.column` call; in the nested call, `env.implements_reference` will be truthy.

  4. Sqlite3 implements `change_column` by a creating a new table. This will result in nested calls to `add_column`, invoking the `Migration::Column` stack for each; SchemaPlus::Core does not currently provide a way to distinguish those calls from explicit top-level calls.

* `Migration::CreateTable`

  Creates a new table

    Env Field    | Description | Initialized
    --- | --- | ---
    `:caller`     | The ActiveRecord instance responsible for creating the table | *context*
    `:table_name` | The name of the table | *arg*
    `:options`    | Create table options | *arg*
    `:block`      | Proc containing table definition statements | *arg*

   The base implementation creates the table, yielding a `table_definition` instance to the block (if a block is given).

* `Migration::DropTable`

  Drops a table from the database

    Env Field    | Description | Initialized
    --- | --- | ---
    `:connection`     | The current ActiveRecord connection | *context*
    `:table_name` | The name of the table | *arg*
    `:options`    | Drop table options | *arg*

   The base implementation drops the table.  No value is returned.

* `Migration::RenameTable`

  Renames a table

    Env Field    | Description | Initialized
    --- | --- | ---
    `:connection`     | The current ActiveRecord connection | *context*
    `:table_name` | The existing name of the table | *arg*
    `:new_name`    | The target name of the table | *arg*

   The base implementation renames the table.  No value is returned.

* `Migration::Index`

  Callback stack for various ways to define an index.

    Env Field    | Description | Initialized
    --- | --- | ---
    `:caller`     | The ActiveRecord instance responsible for performing the action | *context*
    `:operation`  | `:add` or `:define` | *context*
    `:table_name` | The name of the table | *arg*
    `:column_names` | The names of the columns | *arg*
    `:options`    | The index options | *arg*, default `{}`

  The base implementation performs the index creation operation.  No value is returned.

  Notes:

  1. The `:operation` field has the following meanings:

      * `:add` - The index will be added immediately (`Migration#add_index`)
      * `:define` - The index will be added to a table definition, which will be emitted later.

### Sql

Stacks for internal operations that generate SQL.

* `Sql::ColumnOptions`

  Callback stack around generation of the SQL options for a column definition.

    Env Field    | Description | Initialized
    --- | --- | ---
    `:sql` | The resulting SQL | `""`
    `:caller`     | The ActiveRecord::SchemaCreation instance | *context*
    `:connection` | The current ActiveRecord connection | *context* |
    `:column`     | The column definition object | *context* |
    `:options`    | The column definition options | *context* |

  The base implementation appends the options SQL to `env.sql`

* `Sql::IndexComponents`

  Callback stack around generation of the SQL for an index definition.

    Env Field    | Description | Initialized
    --- | --- | ---
    `:sql` | The resulting SQL components, in a struct with fields `:name`, `:type`, `:columns`, `:options`, `:algorithm`, `:using` | *empty struct*
    `:connection` | The current ActiveRecord connection | *context*
    `:table_name` | The name of the table | *context*
    `:column_names` | The names of the columns | *context*
    `:options`    | The index options | *context*

  The base implementation *overwrites* the contents of `env.sql`

  Notes:

  1. SQLite3 ignores the `:type`, `:algoritm`, and `:using` fields of `env.sql`

* `Sql::Table`

  Callback stack around generation of the SQL for a table

    Env Field    | Description | Initialized
    --- | --- | ---
    `:sql` | The resulting SQL components in a struct with fields `:command`, `:name`, `:body`, `:options`, `:quotechar` | *empty struct*
    `:caller`     | The ActiveRecord::SchemaCreation instance | *context*
    `:connection` | The current ActiveRecord connection | *context*
    `:table_definition` | The TableDefinition object for the table | *context*

  The base implementation *overwrites* the contents of `env.sql`

  Notes:

  1. `env.sql.command` contains the index creation command such as `CREATE TABLE` or `CREATE TEMPORARY TABLE`
  2. `env.sql.quotechar` contains the quote character ', ", or \` to wrap `env.sql.name` in.


### Query

Stacks around low-level query execution

* `Query::Exec`

  Callback stack wraps the emission of sql to the underlying dbms gem.

      Env Field    | Description | Initialized
    --- | --- | ---
    `:result` | The result of the database query | *unset*
    `:caller`     | The ActiveRecord::SchemaCreation instance | *context*
    `:sql`        | The SQL string | *context*
    `:binds`      | Values to substitute into the SQL string
    `:query_name` | Label sometimes used by ActiveRecord logging | *arg*


### Dumper

SchemaPlus::Core provides a state object and of callbacks to various phases of the schema dumping process.  The dumping process fleshes out the state object-- nothing is actually written to the dump file until after the state is fleshed out.

#### Schema Dump state

* `Class SchemaPlus::Core::SchemaDump`

  An instance of `SchemaPlus::Core::SchemaDump` gets passed to each of the callback stacks; the dump gets built up by fleshing out its contents.  `SchemaDump` has the following fields and methods:

  * `dump.initial = []` - an array of strings containing statements to start the schema with, such as `enable_extension 'hstore'`
  * `dump.data = OpenStruct.new` - a place for clients to store arbitrary data between phases
  * `dump.tables = {}` - a hash mapping table names to SchemaDump::Table objects
  * `dump.final = []` - an array of strings containing statements to end the schema with.
  * `dump.depends(table_name, [prerequisite_table_names])` - call this method to ensure that the definition of `table_name` won't be output before its prerequisites.

* `Class SchemaPlus::Core::SchemaDump::Table`

  Each table in the dump has its contents in a SchemaDump::Table object, with these fields:

  * `table.name` - the table name
  * `table.pname` - table as actually used in SQL (without any prefixes or suffixes)
  * `table.options` - a string containing the options to `Migration.create_table`
  * `table.columns = []` - an array of SchemaDump::Table::Column objects
  * `table.indexes = []` - an array of SchemaDump::Table::Index objects
  * `table.statements` - a collection of statements to include in the table definition; each is a string that should start with `"t."`
  * `table.trailer` - a collection of migration statements to include immediately outside the table definition.  Each is a string
  * `table.alt` - In some cases, ActiveRecord is unable to dump a table in the form of a migration `create_table` statement; in this case `table.pname` will be nil, and `table.alt` will contain the alternate string to dump instead. (E.g. if the table contains custom types, ActiveRecord will be unable to handle it and will just dump an error message as a comment.)

* `Class SchemaPlus::Core::SchemaDump::Table::Column`

  Each column in a table has its contents in a SchemaDump::Table::Column object, with these fields and methods:

  * `column.name` - the column name
  * `column.type` - the column type (i.e., what comes after `"t."`)
  * `column.options` - a hash containing the options for the column
  * `column.comments` - an array of comment strings for the column

* `Class SchemaPlus::Core::SchemaDump::Table::Index`

  Each index in a table has its contents in a SchemaDump::Table::Index object, with these fields and methods:

  * `index.name` - the index name
  * `index.columns` - the columns that are in the index
  * `index.options` - a hash containing the options for the column

#### Schema Dump Middleware stacks

* `Dumper::Initial`

  Callback stack wraps the creation of initial statements for the dump.

    Env Field     | Description                                     | Initialized
    ---           | ---                                             | ---
    `:initial`    | The initial statements                          | []
    `:dump`       | The SchemaDump object                           | *context*
    `:dumper`     | The current ActiveRecord::SchemaDumper instance | *context*
    `:connection` | The current ActiveRecord connection             | *context*

  The base method appends initial statements to `env.initial`.

* `Dumper::Tables`

  Callback stack wraps the dumping of all tables.

    Env Field     | Description                                     | Initialized
    ---           | ---                                             | ---
    `:dump`       | The SchemaDump object                           | *context*
    `:dumper`     | The current ActiveRecord::SchemaDumper instance | *context*
    `:connection` | The current ActiveRecord connection             | *context*

  The base method iterates through all tables, dumping each.


* `Dumper::Table`

  Callback stack wraps the dumping of each table

    Env Field     | Description                                     | Initialized
    ---           | ---                                             | ---
    `:table`      | A SchemaDump::Table object                      | `table.name` *only*
    `:dump`       | The SchemaDump object                           | *context*
    `:dumper`     | The current ActiveRecord::SchemaDumper instance | *context*
    `:connection` | The current ActiveRecord connection             | *context*

  The base method iterates through all columns and indexes of the table, and *overwrites* the contents of `table`,

  Notes:

  1. When the stack is called, `env.dump.tables[env.table.name]` contains the `env.table` object.
  2. The base method sets *both* `env.table.columns` and `env.tables.indexes`.

## Release Notes

* 2.2.3 Fix dumping complex expression based indexes in AR 5.x
* 2.2.2 Fixed dumping tables in postgresql in AR 5.2 when the PK is not a bigint.
* 2.2.1 Fixed expression index handling in AR5.x.
* 2.2.0 Added AR5.2 support.  Thanks to [@jeremyyap](https://github.com/jeremyyap)
* 2.1.1 Bug fix: Don't lose habtm options.  Thanks to [@iagopiimenta ](https://github.com/iagopiimenta)
* 2.1.0 Added AR5.1 support.  Thanks to [@iagopiimenta ](https://github.com/iagopiimenta)
* 2.0.1 Tighten up AR dependency.  Thanks to [@myabc](https://github.com/myabc).
* 2.0.0 Added AR5 support, removed AR4.2 support.  Thanks to [@boazy](https://github.com/boazy).
* 1.0.2 Missing require
* 1.0.1 Explicit gem dependencies
* 1.0.0 Clean up `SchemaDump::Table::Column` and `SchemaDump::Table::Index` API:  `#options` is now a hash and `#comments` is now an array; no longer have `add_option` and `add_comment` methods.
* 0.6.2 Bug fix: don't choke on INHERITANCE in table definition (#7).  Thanks to [@ADone](https://github.com/ADone).
* 0.6.1 Make sure to require pathname (#5)
* 0.6.0 Added `table.alt` to dumper; Bug fix: Don't crash when AR fails to dump a table. Thanks to [@stenver](https://github.com/stenver) for tracking it down
* 0.5.1 Bug fix: Don't choke on a quoted newline in a `CREATE TABLE` statement ([#3](https://github.com/SchemaPlus/schema_plus_core/pull/3)).  Thanks to [@mikeauclair](https://github.com/mikeauclair)
* 0.5.0 Added `Migration::DropTable`
* 0.4.0 Added `implements_reference` to `Migration::Column` stack env
* 0.3.1 Pass along (undocumented) return values from association declarations ([#2](https://github.com/SchemaPlus/schema_plus_core/pull/2)).  Thanks to [@lowjoel](https://github.com/lowjoel)
* 0.3.0 Added `Model::Association::Declaration` ([#1](https://github.com/SchemaPlus/schema_plus_core/pull/1)).  Thanks to [@lowjoel](https://github.com/lowjoel).
* 0.2.1 Added `Migration::CreateTable` and `Schema::Define`; removed dependency on (defunct) `schema_monkey_rails` gem.  [Oops, this should have been a minor version bump]
* 0.2.0 Added `Migration::DropTable`
* 0.1.0 Initial release

## Development & Testing

Are you interested in contributing to SchemaPlus::Core?  Thanks!  Please follow the standard protocol: fork, feature branch, develop, push, and issue pull request.

Some things to know about to help you develop and test:

<!-- SCHEMA_DEV: TEMPLATE USES SCHEMA_DEV - begin -->
<!-- These lines are auto-inserted from a schema_dev template -->
* **schema_dev**:  SchemaPlus::Core uses [schema_dev](https://github.com/SchemaPlus/schema_dev) to
  facilitate running rspec tests on the matrix of ruby, activerecord, and database
  versions that the gem supports, both locally and on
  [github actions](https://github.com/SchemaPlus/schema_plus_core/actions)

  To to run rspec locally on the full matrix, do:

        $ schema_dev bundle install
        $ schema_dev rspec

  You can also run on just one configuration at a time;  For info, see `schema_dev --help` or the [schema_dev](https://github.com/SchemaPlus/schema_dev) README.

  The matrix of configurations is specified in `schema_dev.yml` in
  the project root.

<!-- SCHEMA_DEV: TEMPLATE USES SCHEMA_DEV - end -->

<!-- SCHEMA_DEV: TEMPLATE USES SCHEMA_MONKEY - begin -->
<!-- These lines are auto-inserted from a schema_dev template -->
* **schema_monkey**: SchemaPlus::Core is implemented as a
  [schema_monkey](https://github.com/SchemaPlus/schema_monkey) client,
  using [schema_monkey](https://github.com/SchemaPlus/schema_monkey)'s
  convention-based protocols for extending ActiveRecord and using middleware stacks.

<!-- SCHEMA_DEV: TEMPLATE USES SCHEMA_MONKEY - end -->
