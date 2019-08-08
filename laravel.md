# About
Notes regarding laravel forged from 100% pure experience.

# Setup Laravel

``` 
   $ composer create-project laravel/laravel --prefer-dist ./learn_laravel
    cd learn_laravel
    composer install
```

# Laravel specify a different name for `.env` when used docker for development.

Then in case of docker look on: https://stackoverflow.com/questions/56682726/laravel-specify-different-name-from-env-or-make-ingore-env-in-order-not-to

# Database How to
 
On Seeder when following: https://laravel.com/docs/5.8/seeding
Make sure you do:
```
 composer dump-autoload
```

When you develop a new feature and needs somne db migration then do:

```
 php artisan make:migration ^migration_name^ 
```

In case of a new table:

```
 php artisan make:migration ^migration_name^ --create=^table_name^
```

During development change the created file and coniniously nube db with:

```
    php artisan migrate:refresh
```

Then follow the seeding procedure.

> NOTE:
> On development branch you should **NOT** use the artisan's `migrate:refresh` command.

# Tinker Cheatsheet


1. Get all settings (for debugging db connection issues and not only):
   ```
   config()->all();
   ```

2. Getting database-only config:
   ```
    $config=config();
    $config->get('database');
   ```

3. Running factories:
   ```
    use ^model_namespace^
    factory(^name^)->create();
    factory(^name^, ^ammount^)->create();
   ```

# Factories

## Generic workflow for making factories

1. Create a file in `./database/factories`.
2. Fill it as seen in https://laravel.com/docs/5.7/database-testing#writing-factories.
3. Create new autoloader via `composer dump-autoload -o`.
4. Test in tinker as seen [cheatsheet](#tinker-cheatsheet).
5. Use it into your test.

## Creating a foreign key in a factory without duplicate entries.

As seen in this stackoverflow [question](https://stackoverflow.com/q/56749976/4706711) in a factory you can call an another factory for the relationship model.

Foe example we have a `Rover` that is related with a `Grid`. The foreighn key is the `grid_id` and you can use the following factory:

```php
$factory->define(Rover::class, function(Faker $faker) {
    $command = generateRandomCommand(rand(0));
    $commandLength = strlen($command);
    $commandPos = rand(0,$commandLength);
    $lastExecutedCommand = substr($command,$commandPos,$commandPos);

    //This will create a second entry in the database.
    $randomGrid = factory(Grid::class)->create();

    return [
        'grid_id' => $randomGrid->id, //no need for the value() method, they are all attributes
        'grid_pos_x' => rand(0,$randomGrid->width), 
        'grid_pos_y' => rand(0,$randomGrid->height),
        'rotation' => RoverConstants::ORIENTATION_EAST,
        'command' => $command,
        'last_commandPos' => $commandPos,
        'last_command' => $lastExecutedCommand,
    ];
});
```

But this in case of an attribute overide `grid_id` as this code shows:

```php
$existingGrid = factory(Grid::class)->create();
factory(Rover::class)->create([
    'grid_id' => $existingGrid->id,
    'grid_pos_x' => rand(0, $existingGrid->width),
    'grid_pos_y' => rand(0, $existingGrid->height),
]);
```

Would create 2 `Grids` instead of one resulting junk data to the testing database. Therefore providing a `Closure` ad foreign key creation would do the job:

```
$factory->define(Rover::class, function(Faker $faker) {
    $command = generateRandomCommand(rand(0));
    $commandLength = strlen($command);
    $commandPos = rand(0,$commandLength);
    $lastExecutedCommand = substr($command,$commandPos,$commandPos);

    return [
        'grid_id' => function() {
            return factory(Grid::class)->create()->id;
        }
        'grid_pos_x' => '', //But then i got nothing for this.
        'grid_pos_y' => '', //But then i also got nothing for this.
        'rotation' => RoverConstants::ORIENTATION_EAST,
        'command' => $command,
        'last_commandPos' => $commandPos,
        'last_command' => $lastExecutedCommand,
    ];
});
```

Or even better have an external function to do that:

```php
if(!function_exists('getGridIdIfNotExists')){

    function getGridIdIfNotExists(){
       return factory(Grid::class)->create()->id;
    }

}

$factory->define(Rover::class, function(Faker $faker) {
    $command = generateRandomCommand(rand(0));
    $commandLength = strlen($command);
    $commandPos = rand(0,$commandLength);
    $lastExecutedCommand = substr($command,$commandPos,$commandPos);

    return [
        'grid_id' => function() {
            return getGridIdIfNotExists();
        }
        'grid_pos_x' => function(){
            $max_width=App\User::find($rover['grid_id'])->width;
            return rand(0,$max_width);
        },
        'grid_pos_y' => function(){
            $max_height=App\User::find($rover['grid_id'])->height;
            return rand(0,$max_height);
        },
        'rotation' => RoverConstants::ORIENTATION_EAST,
        'command' => $command,
        'last_commandPos' => $commandPos,
        'last_command' => $lastExecutedCommand,
    ];
});

```

So by changing the code in `getGridIdIfNotExists` you can specify your approach of fetching a `Grid`, for example geting a random grid.

# Migrations

## Running and generating migrations on specific folder.

Assuming that you have created a folder named `./migrations/tests`. In that folder you can:

- You create a migraction with the command:
  ```
  php artisan make:migration --path './migrations/tests' ^migration_name^ --table=^table_name^
  ```

- On development you can use the command to rerun the tests:
  ```
  php artisan migration:refresh --path './migrations/tests'
  ```

- And you can run the migration as:
  ```
  php artisan migrate --path './migrations/tests'
  ```

## Make test-only database migrations only for setting up a testing database

Sometimes you need to test an existing schema and the original project has no migration tests whatsoever only a database schema manually created. In that case you need to make migrations in a seperate database replicating the database schema (only the nessesary fields) same as the production one. Also you need to ensure that theese migrations will run only in testing environment as well. 

In order to do that follow theese steps:

1. Make a migration:
  ```
  php artisan make:migration ^migration_name^ --table=^table_name^
  ```
2. On this use the `App` Facade in order to check whether in production. In your case use the following approach of this dummy facade:
  ```
    use Illuminate\Support\Facades\Schema;
    use Illuminate\Database\Schema\Blueprint;
    use Illuminate\Database\Migrations\Migration;

    class CreateZostirasTable extends Migration
    {
        /**
         * Run the migrations.
         *
         * @return void
         */
        public function up()
        {
            if(App::environment() == 'testing'){
               //Create or alter table here
            }
        }

        /**
         * Reverse the migrations.
         *
         * @return void
         */
        public function down()
        {
            if(App::environment() == 'testing'){
              // Nuke it here
            }
        }
    }
  ```

## Specify database connection on migrations:

Instead of using the:

```
Schema::table
```

Use the:

```
Schema::connection('etable_website')->create(
```

# Laravel and docker

## For development

On `docker-compose.yml` **DO NOT SET** enviromental variables with the **SAME** name as the one used in laravel.

Instead prefix them with a unique prefix in order to seperate them with the ones uses in laravel. A good idea is for all the enviromental variables offered vrom docker to be prefixed with **DOCKER_** prefix. Then in `.env` using the ${VARIABLE_NAME} syntax load them. For example:

We have the following `docker-compose.yml`:

```
version: '3.1'
services:
  develop:
    image: ddesyllas/php-dev:dev-n-build
    volumes:
      - ".:/var/www/html"
      - "./.laravel.env:/var/www/html/.env"
    links:
      - memcache
    environment:
      DOCKER_DB_CONNECTION: pgsql
      DOCKER_DB_HOST : postgresql
      DOCKER_DB_PORT : 5432
      DOCKER_DB_DATABASE: ${DOCKER_POSTGRES_DB}
      DOCKER_DB_USERNAME: ${DOCKER_POSTGRES_USER}
      DOCKER_DB_PASSWORD: ${DOCKER_POSTGRES_PASSWORD}

  nginx:
    image: nginx:alpine
    ports:
      - 7880:7880
    links:
      - "develop:develop"
    volumes:
      - ".:/var/www/html"
      - "./docker/nginx.conf:/etc/nginx/nginx.conf:ro"


  postgresql:
    image: postgres:alpine
    volumes:
      - './docker/misc_volumes/postgresql:/var/lib/postgresql/data'
    environment:
      POSTGRES_USER: ${DOCKER_POSTGRES_USER}
      POSTGRES_DB: ${DOCKER_POSTGRES_DB}
      POSTGRES_PASSWORD: ${DOCKER_POSTGRES_PASSWORD}

  memcache:
    image: memcached:alpine

```

Then the `.env` sill be:

```
APP_NAME=Laravel
APP_ENV=local
APP_KEY=base64:VJU6sZLgo9v13QftelmITraHNrx96mVWeCO/xeCyZWU=
APP_DEBUG=true
APP_URL=http://localhost

LOG_CHANNEL=stack

DB_CONNECTION=${DOCKER_DB_CONNECTION}
DB_HOST=${DOCKER_DB_HOST}
DB_PORT=${DOCKER_DB_PORT}
DB_DATABASE=${DOCKER_DB_DATABASE}
DB_USERNAME=${DOCKER_DB_USERNAME}
DB_PASSWORD=${DOCKER_DB_PASSWORD}

```

# Testing with Laravel

## Testing a Model relationship:

Let suppose we have theese models:

The `Grid` model:

```php
namespace App\Model;

use Illuminate\Database\Eloquent\Model;
use App\Model\Rover;

class Grid extends Model
{
    /**
     * Table Name
     */
    protected $table='grid';

    public function rovers()
    {
        return $this->hasMany(Rover::class);
    }
}
```

The `Rover` model:

```php
namespace App\Model;

use Illuminate\Database\Eloquent\Model;
use App\Model\Grid;

class Rover extends Model
{

    /**
     * Table Name
     */
    protected $table='rover';

    public function grid()
    {
        return $this->belongsTo(Grid::class);
    }

    public function setGridPosXValue($value)
    {
        $Grid = $this->grid()->first();
        $width = $Grid->width;
        if($value < 0 || $value > $width){
            throw new \InvalidArgumentException("X is out of grid bounds");
        }

        $this->attributes['x']=$value;
    }

    public function setGridPosYValue($value)
    {
        $Grid = $this->grid()->first();
        $height = $Grid->height;

        if($value < 0 || $value > $height){
            throw new \InvalidArgumentException("Y is out of grid bounds");
        }

        $this->attributes['y']=$value;
    }
}
```

When the relationships are being accessed via function instead of member variable then it needs to be mocked because it access the db. In our case we use the methods seen in [documentation](https://laravel.com/docs/5.8/eloquent-collection), especially the `belongsTo()` and the `hasMany()` that return an Eloquent [collection](https://laravel.com/docs/5.8/eloquent-collections). In that case mock the method that call the helper function `collect` using a partial mock as seen in the following situations:

* https://stackoverflow.com/a/56734842/4706711
