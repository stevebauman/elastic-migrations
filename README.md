# Elastic Migrations

[![Latest Stable Version](https://poser.pugx.org/babenkoivan/elastic-migrations/v/stable)](https://packagist.org/packages/babenkoivan/elastic-migrations)
[![Total Downloads](https://poser.pugx.org/babenkoivan/elastic-migrations/downloads)](https://packagist.org/packages/babenkoivan/elastic-migrations)
[![License](https://poser.pugx.org/babenkoivan/elastic-migrations/license)](https://packagist.org/packages/babenkoivan/elastic-migrations)
[![Tests](https://github.com/babenkoivan/elastic-migrations/workflows/Tests/badge.svg)](https://github.com/babenkoivan/elastic-migrations/actions?query=workflow%3ATests)
[![Code style](https://github.com/babenkoivan/elastic-migrations/workflows/Code%20style/badge.svg)](https://github.com/babenkoivan/elastic-migrations/actions?query=workflow%3A%22Code+style%22)
[![Static analysis](https://github.com/babenkoivan/elastic-migrations/workflows/Static%20analysis/badge.svg)](https://github.com/babenkoivan/elastic-migrations/actions?query=workflow%3A%22Static+analysis%22)
[![Donate PayPal](https://img.shields.io/badge/donate-paypal-blue)](https://paypal.me/babenkoi)

---

<p align="center">
    🎅🏻 Ho ho ho! Christmas is coming! Please consider supporting and starring the project! ⭐️
</p>

<p align="center">
    <a href="https://ko-fi.com/ivanbabenko" target="_blank"><img src="support.png" alt="Support the project!" height="80"></a>
</p>

---

Elastic Migrations for Laravel allow you to easily modify and share indices schema across the application's environments.

## Contents

* [Compatibility](#compatibility)
* [Installation](#installation) 
* [Configuration](#configuration)
* [Writing Migrations](#writing-migrations)
* [Running Migrations](#running-migrations)
* [Reverting Migrations](#reverting-migrations)
* [Starting Over](#starting-over)
* [Migration Status](#migration-status)
* [Zero Downtime Migration](#zero-downtime-migration)
* [Troubleshooting](#migration-status)

## Compatibility

The current version of Elastic Migrations has been tested with the following configuration:

* PHP 7.3-8.0
* Elasticsearch 7.x
* Laravel 6.x-8.x

## Installation

The library can be installed via Composer:

```bash
composer require babenkoivan/elastic-migrations
```

If you want to use Elastic Migrations with [Lumen framework](https://lumen.laravel.com/) check [this guide](https://github.com/babenkoivan/elastic-migrations/wiki/Lumen-Installation).

## Configuration

Elastic Migrations uses [babenkoivan/elastic-client](https://github.com/babenkoivan/elastic-client) as a dependency. 
If you want to change the default client settings (and I'm pretty sure you do), then you need to create the configuration file first:

```bash
php artisan vendor:publish --provider="ElasticClient\ServiceProvider"
```

You can change Elasticsearch host and other client settings in the `config/elastic.client.php` file. Please refer to 
[babenkoivan/elastic-client](https://github.com/babenkoivan/elastic-client) for more details.

You can also publish Elastic Migrations settings:

```bash
php artisan vendor:publish --provider="ElasticMigrations\ServiceProvider"
```

This will create the `config/elastic.migrations.php` file, which allows you to configure the following options:

* `table` - the migration table name
* `connection` - the database connection
* `storage_directory` - the migrations directory
* `index_name_prefix` - the indices prefix
* `alias_name_prefix` - the aliases prefix

Finally, don't forget to run Laravel database migrations to create Elastic Migrations table:

```bash
php artisan migrate
```

## Writing Migrations

You can effortlessly create a new migration file using an Artisan console command:

```bash
php artisan elastic:make:migration create_my_index
```

This command creates a migration class in the `elastic/migrations` directory. 

Every migration includes two methods: `up` and `down`. `up` is used to alternate the index schema and `down` is used to revert that action.

You can use `ElasticMigrations\Facades\Index` facade to perform basic operations over Elasticsearch indices:

#### Create Index

You can create an index with the default settings: 

```php
Index::create('my-index');
``` 

You can use a modifier to configure mapping and settings:

```php
Index::create('my-index', function (Mapping $mapping, Settings $settings) {
    // to add a new field to the mapping use method name as a field type (in Camel Case), 
    // first argument as a field name and optional second argument for additional field parameters  
    $mapping->text('title', ['boost' => 2]);
    $mapping->float('price');

    // you can define a dynamic template as follows
    $mapping->dynamicTemplate('my_template_name', [
        'match_mapping_type' => 'long',
        'mapping' => [
            'type' => 'integer',
        ],
    ]);
    
    // you can also change the index settings and the analysis configuration
    $settings->index([
         'number_of_replicas' => 2,
         'refresh_interval' => -1
    ]);
    
    $settings->analysis([
        'analyzer' => [
            'title' => [
                'type' => 'custom',
                'tokenizer' => 'whitespace'    
            ]
        ]
    ]);
});
```

There is also the `createRaw` method in your disposal:

```php
$mapping = [
    'properties' => [
        'title' => [
            'type' => 'text'
        ]
    ]
];

$settings = [
    'number_of_replicas' => 2
];

Index::createRaw('my-index', $mapping, $settings);
```

Finally, it is possible to create an index only if it doesn't exist:

```php
// you can use a modifier as shown above
Index::createIfNotExists('my-index', $modifier);
// or you can use raw mapping and settings 
Index::createIfNotExistsRaw('my-index', $mapping, $settings);
```

#### Update Mapping

You can use a modifier to adjust the mapping:

```php
Index::putMapping('my-index', function (Mapping $mapping) {
    $mapping->text('title', ['boost' => 2]);
    $mapping->float('price');
});
```

Alternatively, you can use the `putMappingRaw` method as follows:

```php
Index::putMappingRaw('my-index', [
    'properties' => [
        'title' => [
            'type' => 'text',
            'boost' => 2
        ],
        'price' => [
            'price' => 'float'
        ]      
    ]   
]);
```

#### Update Settings

You can use a modifier to change an index configuration:

```php
Index::putSettings('my-index', function (Settings $settings) {
    $settings->index([
         'number_of_replicas' => 2,
         'refresh_interval' => -1
    ]);
});
``` 

The same result can be achieved with the `putSettingsRaw` method:

```php
Index::putSettingsRaw('my-index', [
    'index' => [
        'number_of_replicas' => 2,
        'refresh_interval' => -1
    ]
]); 
```

It is possible to update analysis settings only on closed indices. The `pushSettings` method closes the index, 
updates the configuration and opens the index again:

```php
Index::pushSettings('my-index', function (Settings $settings) {
    $settings->analysis([
        'analyzer' => [
            'title' => [
                'type' => 'custom',
                'tokenizer' => 'whitespace'
            ]
        ]
    ]);
});
```

The same can be done with the `pushSettingsRaw` method:

```php
Index::pushSettingsRaw('my-index', [
    'analysis' => [
        'analyzer' => [
            'title' => [
                'type' => 'custom',
                'tokenizer' => 'whitespace'
            ]
        ]
    ]
]); 
```

#### Drop Index

You can unconditionally delete the index:

```php
Index::drop('my-index');
```

or delete it only if it exists:

```php
Index::dropIfExists('my-index');
```

#### Create Alias

You can create an alias with optional filter query:

```php
Index::putAlias('my-index', 'my-alias', ['term' => ['user_id' => 1]]);
```

#### Delete Alias

You can delete an alias by its name:

```php
Index::deleteAlias('my-index', 'my-alias');
```

#### More

Finally, you are free to inject `Elasticsearch\Client` in the migration constructor and execute any supported by client actions.

## Running Migrations

You can either run all migrations:

```bash
php artisan elastic:migrate
```

or run a specific one:

```bash
php artisan elastic:migrate 2018_12_01_081000_create_my_index
```

Use the `--force` option if you want to execute migrations on production environment:

```bash
php artisan elastic:migrate --force
```

## Reverting Migrations

You can either revert the last executed migrations:

```bash
php artisan elastic:migrate:rollback 
```

or rollback a specific one:

```bash
php artisan elastic:migrate:rollback 2018_12_01_081000_create_my_index
```

Use the `elastic:migrate:reset` command if you want to revert all previously migrated files:

```bash
php artisan elastic:migrate:reset 
```

## Starting Over

Sometimes you just want to start over, rollback all the changes and apply them again:

```bash
php artisan elastic:migrate:refresh
```

Alternatively you can also drop all existing indices and rerun the migrations:

```bash
php artisan elastic:migrate:fresh
```

## Migration Status

You can always check which files have been already migrated and what can be reverted by the `elastic:migrate:rollback` command (the last batch):

```bash
php artisan elastic:migrate:status
```

## Zero Downtime Migration

Changing an index mapping with zero downtime is not a trivial process and might vary from one project to another.
Elastic Migrations library doesn't include such feature out of the box, but you can implement it in your project by [following this guide](https://github.com/babenkoivan/elastic-migrations/wiki/Changing-Mapping-with-Zero-Downtime).

## Troubleshooting

If you see one of the messages below, follow the instructions:

* `Migration table is not yet created` - run the `php artisan migrate` command
* `Migration directory is not yet created` - create a migration file using the `elastic:make:migration` command or 
create `migrations` directory manually
  
In case one of the commands doesn't work as expected, try to publish configuration:

```bash
php artisan vendor:publish --provider="ElasticMigrations\ServiceProvider"
```
