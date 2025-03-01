# Database - Query Builders

You can read how to work with Database using manually written queries [here](/docs/en/database/access.md).

DBAL component includes a set of query builders used to unify the way of working with different databases and simplify
migration to different DBMS over the lifetime of the application.

## Before we start

To demonstrate query building abilities let's declare sample table in our default database first:

```php
use Cycle\Database\Database;

$schema = $dbal->database()->table('test')->getSchema();

$schema->primary('id');
$schema->datetime('time_created');
$schema->enum('status', ['active', 'disabled'])->defaultValue('active');
$schema->string('name', 64);
$schema->string('email');
$schema->double('balance');
$schema->save();
```

> You can read more about declaring database schemas [here](/docs/en/database/declaration.md).

## Insert Builder

To get an instance of InsertBuilder (responsible for insertions), we have to execute following code:

```php
$insert = $db->insert('test');
```

Now we can add some values to our builder to be inserted into related table:

```php
$insert = $db->insert('test');

$insert->values([
    'time_created' => new \DateTime(),
    'name'         => 'Anton',
    'email'        => 'test@email.com',
    'balance'      => 800.90
]);
```

To run InsertQuery we should only execute method `run()` which will return last inserted id as result:

```php
print_r($db->run());
```

> You can also use fluent syntax: `$database->insert('table')->values(...)->run()`.

### Batch Insert

You add as many values into insert builder as your database can support:

```php
$insert->columns([
    'time_created', 
    'name', 
    'email', 
    'balance'
]);

for ($i = 0; $i < 20; $i++) {
    // we don't need to specify key names in this case
    $insert->values([
        new \DateTime(),
        $this->faker->randomNumber(2),
        $this->faker->email,
        $this->faker->randomFloat(2)
    ]);
}

$insert->run();
```

### Quick Inserts

You can skip InsertQuery creation by talking to your table directly:

```php
$table = $db->table('test');

print_r($table->insertOne([
    'time_created' => new \DateTime(),
    'name'         => 'Anton',
    'email'        => 'test@email.com',
    'balance'      => 800.90
]));
```

> Table class will automatically run a query and return the last inserted id. You can also check the `insertMultiple` 
> method of Table.

## SelectQuery Builder

SelectQuery builder can be retrieved two very similar ways, you can either get it from database or from table instances:

```php
$select = $db->table('test')->select();

// alternative
$select = $db->select()->from('test');

// alternative
$select = $db->test->select();
```

### Select Columns

By default, SelectQuery selects every column (`*`) from its related table. We can always change the set of requested
columns using the `columns` method.

```php
$db->users->select()
    ->columns('name')
    ->fetchAll();
```

You can use your select query as proper iterator or use `run` method which will return instance
of `Cycle\Database\Statement`:

```php
foreach($select->getIterator() as $row) {
    print_r($row);
}
```

To select all values as array use `fetchAll`:

```php
foreach($select->fetchAll() as $row) {
    print_r($row);
}
```

You can always view what SQL is generated by your query by dumping it or via method `sqlStatement`:

```php
print_r(
    $db->users->select()
        ->columns('name')
        ->sqlStatement()
);
```

### Where Statements

Add WHERE conditions to your query using `where`, `andWhere`, `orWhere` methods.

#### Basics

Let's add simple condition on `status` column of our table:

```php
$select = $db->select()
    ->from('test')
    ->columns(['id', 'status', 'name']);

$select->where('status', '=', 'active');

foreach ($select as $row) {
    print_r($row);
}
```

```sql
SELECT `id`,
       `status`,
       `name`
FROM `primary_test`
WHERE `status` = 'active'        
```

> Note that prepared statements used behind the scenes.

You can skip '=' in your conditions:

```php
$select->where('status', 'active');
```

#### Where Operators

Second argument can be used to declare operator:

```php
$select->where('id', '>', 10);
$select->where('status', 'like', 'active');
```

For between and not between conditions you can also use forth argument of where method:

```php
$select->where('id', 'between', 10, 20);
```

Resulted SQL:

```sql
SELECT `id`,
       `status`,
       `name`
FROM `primary_test`
WHERE `id` BETWEEN 10 AND 20  
```

#### Multiple Where Conditions

Chain multiple where conditions using fluent calls:

```php
$select
    ->where('id', 1)
    ->where('status', 'active');
```

Method `andWhere` is an alias for `where`, so we can rewrite such condition to make it more readable:

```php
$select
    ->where('id', 1)
    ->andWhere('status', 'active');
```

Resulted SQL:

```sql
SELECT `id`,
       `status`,
       `name`
FROM `primary_test`
WHERE `id` = 1
  AND `status` = 'active'
```

SelectQuery will generate SQL based respecting your operator order and boolean operators:

```php
$select
    ->where('id', 1)
    ->orWhere('id', 2)
    ->orWhere('status', 'active');
```

```sql
SELECT `id`,
       `status`,
       `name`
FROM `primary_test`
WHERE `id` = 1
   OR `id` = 2
   OR `status` = 'active'
```

#### Complex/Group Where Conditions

Group multiple where conditions using Closure as your first argument:

```php
$select
    ->where('id', 1)
    ->where(
        static function (SelectQuery $select) {
            $select
                ->where('status', 'active')
                ->orWhere('id', 10);
        }
    );
```

```sql
SELECT `id`,
       `status`,
       `name`
FROM `primary_test`
WHERE `id` = 1
  AND (`status` = 'active' OR `id` = 10)
```

Boolean joiners are respected:

```php
$select
    ->where('id', 1)
    ->orWhere(
        static function (SelectQuery $select) {
            $select
                ->where('status', 'active')
                ->andWhere('id', 10);
        }
    );
```

Result:

```sql
SELECT `id`,
       `status`,
       `name`
FROM `primary_test`
WHERE `id` = 1
   OR (`status` = 'active' AND `id` = 10)     
```

> You can nest as many conditions as you want.

#### Simplified/array Where Conditions

Alternatively you can use [MongoDB style](https://docs.mongodb.org/manual/reference/operator/query/) to build your where
conditions:

```php
$select->where([
    'id'     => 1,
    'status' => 'active'
]);
```

Such code is identical to two where method calls and generates such sql:

```sql
SELECT `id`,
       `status`,
       `name`
FROM `primary_test`
WHERE (`id` = 1 AND `status` = 'active')
```

You can also specify custom comparison operators using nested arrays:

```php
$select->where([
    'id'     => ['in' => new Parameter([1, 2, 3])],
    'status' => ['like' => 'active']
]);
```

> Attention, you have to wrap all array arguments using Parameter class. Scalar arguments will be wrapped automatically.

Resulted SQL:

```sql
SELECT `id`,
       `status`,
       `name`
FROM `primary_test`
WHERE (`id` IN (1, 2, 3) AND `status` LIKE 'active')
```

Multiple conditions per field are supported:

```php
$select->where([
    'id' => [
        'in' => [1, 2, 3],
        '<'  => 100
    ]
]);
```

Use `@or` and `@and` groups to create where groups:

```php
$select
    ->where(
        static function (SelectQuery $select) {
            $select
                ->where('id', 'between', 10, 100)
                ->andWhere('name', 'Anton');
        }
    )
    ->orWhere('status', 'disabled');
```

Using short syntax:

```php
$select->where([
    '@or' => [
        [
            'id'   => ['between' => [10, 100]],
            'name' => 'Anton'
        ],
        ['status' => 'disabled']
    ]
]);
```

In both cases resulted SQL will look like:

```sql
SELECT `id`,
       `status`,
       `name`
FROM `primary_test`
WHERE ((`id` BETWEEN 10 AND 100 AND `name` = 'Anton') OR `status` = 'disabled')
```

You can experiment with both ways to declare where conditions and pick the one you like more.

#### Parameters

Spiral mocks all given values using `Parameter` class internally, in some cases (array) you might need to
pass `Parameter`
directly. You can alter the parameter value at any moment, but before the query `run` method:

```php
use Cycle\Database\Injection\Parameter;
// ...

$select = $db->select()
    ->from('test')
    ->columns(['id', 'status', 'name']);

$select->where('id', $id = new Parameter(null));

//Bind new parameter value
$id->setValue(15);

foreach ($select as $row) {
    print_r($row);
}
```

> You can also pass requested PDO parameter type as second argument: `new Parameter(1, PDO::PARAM_INT)`.
> Internally, every value passed into the `where` method is going to be wrapped using the Parameter class.

You can implement ParameterInterface if you want to declare your parameter wrappers with custom logic.

#### SQL Fragments and Expressions

QueryBuilders allow you to replace some of where statements with custom SQL code or expression.
Use `Cycle\Database\Injections\Fragment`
and `Cycle\Database\Injections\Expression` for such purposes.

Use fragment to include SQL code into your query bypassing escaping:

```php
use Cycle\Database\Injection\Fragment;

//255
$select->where('id', '=', new Fragment("DAYOFYEAR('2015-09-12')"));
```

```sql
SELECT `id`,
       `status`,
       `name`
FROM `primary_test`
WHERE `id` = DAYOFYEAR('2015-09-12')
```

If you wish to compare complex value to user parameter, replace where the column with the expression:

```php
use Cycle\Database\Injection\Expression;

$select->where(
    new Expression("DAYOFYEAR(concat('2015-09-', id))"), 
    '=',
    255
);
```

```sql
SELECT *
FROM `x_users`
WHERE DAYOFYEAR(concat('2015-09-', `id`)) = 255
```

> Note that all column identifiers in Expressions will be quoted.

Join multiple columns same way:

```php
$select->where(new \Cycle\Database\Injection\Expression("CONCAT(id, '-', status)"), '1-active');
```

```sql
SELECT `id`,
       `status`,
       `name`
FROM `primary_test`
WHERE CONCAT(`id`, '-', `status`) = '1-active'
```

Expressions are handy when your Database has a non-empty prefix:

```php
$select->where(
    new \Cycle\Database\Injection\Expression("CONCAT(test.id, '-', test.status)"), 
    '1-active'
);
```

```sql
SELECT `id`,
       `status`,
       `name`
FROM `primary_test`
WHERE CONCAT(`primary_test`.`id`, '-', `primary_test`.`status`) = '1-active'
```

You can also use expressions and fragments as column values in the insert and update statements.

> Please keep client data as far from Expressions and Fragments as possible.

### Table and Column aliases

QueryBuilders support user defined table and column aliases:

```php
$select = $db->select()
    ->from('test as abc')
    ->columns([
        'id',
        'status',
        'name'
    ]);

$select->where('abc.id', '>', 10);

foreach ($select as $row) {
    print_r($row);
}
```

```sql
SELECT `id`,
       `status`,
       `name`
FROM `primary_test` as `abc`
WHERE `abc`.`id` > 10
```

Columns:

```php
$select = $db->select()
    ->from('test')
    ->columns([
        'id',
        'status as st',
        'name',
        "CONCAT(test.name, ' ', test.status) as name_and_status"
    ]);

foreach ($select as $row) {
    print_r($row);
}
```

SQL:

```sql
SELECT `id`,
       `status`                                                    as `st`,
       `name`,
       CONCAT(`primary_test`.`name`, ' ', `primary_test`.`status`) as `name_and_status`
FROM `primary_test`
```

#### Sub/Nested Queries

Every spiral QueryBuilder is as instance of `FragmentInterface`, this makes you able to create complex nested queries
when you need them:

```php
$select = $db->select()
    ->from('test')
    ->columns(['id', 'status', 'name']);

$select->where(
    'id',
    'IN',
    $database->select('id')
        ->from('test')
        ->where('id', 'BETWEEN', 10, 100)
);

foreach ($select as $row) {
    print_r($row);
}
```

```sql
SELECT `id`,
       `status`,
       `name`
FROM `primary_test`
WHERE `id` IN (SELECT `id`
               FROM `primary_test`
               WHERE `id` BETWEEN 10 AND 100)  
```

You can compare nested query return value in where statements:

```php
$select->where(
    $db->select('COUNT(*)')
        ->from('test')
        ->where('id', 'BETWEEN', 10, 100), 
    '>',
    1
);
```

```sql
SELECT `id`,
       `status`,
       `name`
FROM `primary_test`
WHERE (SELECT COUNT(*)
       FROM `primary_test`
       WHERE `id` BETWEEN 10 AND 100) > 1   
```

You can exchange column identifiers between parent and nested query using `Expression` class:

```php
$select = $db->select()
    ->from('test')
    ->columns(['id', 'status', 'name']);

$select->where(
    $db->select('name')
        ->from('users')
        ->where('id', '=', new \Cycle\Database\Injection\Expression('test.id'))
        ->where('id', '!=', 100),
    'Anton'
);
```

```sql
SELECT `id`,
       `status`,
       `name`
FROM `primary_test`
WHERE (SELECT `name`
       FROM `primary_users`
       WHERE `id` = `primary_test`.`id`
         AND `id` != 100) = 'Anton'
```

> Nested queries will only work when the nested query belongs to the same database as a primary builder.

### Having

Use methods `having`, `orHaving`, and `andHaving` methods to define HAVING conditions. The syntax is identical to the
WHERE statement.

> Yep, it was quick.

### Joins

You can join any desired table to your query using `leftJoin`, `join`, `rightJoin`, `fullJoin` and `innerJoin` methods:

```php
$select = $db->table('test')
    ->select(['test.*', 'u.name as u']);

$select->leftJoin('users', 'u')
    ->on('users.id', 'test.id');
```

```sql
 SELECT `x_test`.*,
        `u`.`name` AS `u`
 FROM `x_test`
          LEFT JOIN `x_users` AS `u`
                    ON `x_users`.`id` = `x_test`.`id`
```

Method `on` works exactly as `where` except provided values treated as identifier and not as user value. Chain `on`
, `andOn` and `orOn` methods to create more complex joins:

```php
$select->leftJoin('users')
    ->on('users.id', 'test.id')
    ->orOn('users.id', 'test.balance');
```

Array based where conditions are also supported:

```php
$select->leftJoin('users', 'u')->on([
    '@or' => [
        ['u.id' => 'test.id'],
        ['u.id' => 'test.balance']
    ]
]);
```

Generated SQL:

```sql
SELECT `primary_test`.*,
       `primary_users`.`name` as `user_name`
FROM `primary_test`
         LEFT JOIN `primary_users`
                   ON (`primary_users`.`id` = `primary_test`.`id` OR `primary_users`.`id` = `primary_test`.`balance`)    
```

#### On Where statement

To include user value into ON statement, use methods `onWhere`, `orOnWhere` and `andOnWhere`:

```php
$select
    ->innerJoin('users')
    ->on(['users.id' => 'test.id'])
    ->onWhere('users.name', 'Anton');
```

```sql
SELECT `primary_test`.*,
       `primary_users`.`name` as `user_name`
FROM `primary_test`
         INNER JOIN `primary_users`
                    ON `primary_users`.`id` = `primary_test`.`id` AND `primary_users`.`name` = 'Anton'
```

#### Advanced Join statements

You may also specify more advanced join statements. To get started, pass a closure or array as the fourth argument to
the join method.

**Example 1**

```php
$select->join('LEFT', 'photos', 'pht', ['pht.user_id', 'users.id']);
```

```sql
SELECT *
FROM `users`
         LEFT JOIN `photos` AS `pht`
                   ON `pht`.`user_id` = `users`.`id`
```

**Example 2**

You may use grouped statements.

```php
$select->join('LEFT', 'photos', 'pht', [
    '@or' => [
        ['pht.user_id' => 'users.id'],
        ['users.is_admin' => new Parameter(true)]
    ],
]);
```

```sql
SELECT *
FROM `users`
         LEFT JOIN `photos` AS `pht`
                   ON (
                               `pht`.`user_id` = `users`.`id`
                           OR
                               `users`.`is_admin` = true
                       )
```

**Example 3**

You may use grouped statements with sub grouped statements.

```php
$select->join('LEFT', 'photos', 'pht', [
    [
        '@or' => [
            [
                'pht.user_id' => 'users.id',
                'users.is_admin' => new Parameter(true),
            ],
            [
                '@or' => [
                    ['pht.user_id' => 'users.parent_id'],
                    ['users.is_admin' => new Parameter(false)],
                ],
            ],
        ],
    ],
]);
```

```sql
SELECT *
FROM `users`
         LEFT JOIN `photos` AS `pht`
                   ON (
                           (`pht`.`user_id` = `users`.`id` AND `users`.`is_admin` = true)
                           OR
                           (`pht`.`user_id` = `users`.`parent_id` OR `users`.`is_admin` = false)
                       )
```

**Example 4**

You may combine simple statements with grouped statements.

```php
$select->join('LEFT', 'photos', 'pht', [
    'pht.user_id' => 'admins.id',
    'users.is_admin' => 'pht.is_admin',
    '@or' => [
        [
            'users.name' => new Parameter('Anton'),
            'users.is_admin' => 'pht.is_admin',
        ],
        [
            'users.status' => new Parameter('disabled'),
        ],
    ],
]);
```

```sql
SELECT *
FROM `users`
         LEFT JOIN `photos` AS `pht`
                   ON (
                               `pht`.`user_id` = `users`.`id`
                           AND
                               `users`.`is_admin` = true
                           AND (
                                       (`users`.`name` = "Anton" AND `users`.`is_admin` = `pht`.`is_admin`)
                                       OR
                                       `users`.`status` = "disabled"
                                   )
                       )
```

**Example 5**

You may use closure as a fourth argument.

```php
$select->join('LEFT', 'photos', 'pht', static function (
    \Cycle\Database\Query\SelectQuery $select,
    string $boolean, 
    callable $wrapper
): void {
    $select
        ->on('photos.user_id', 'users.id')
        ->onWhere('photos.type', 'avatar');
});
```

```sql
SELECT *
FROM `users`
         LEFT JOIN `photos` AS `pht`
                   ON (
                               `photos`.`user_id` = `users`.`id`
                           AND
                               `photos`.`type` = "avatar"
                       )
```

**Example 6**

You may use closure statement inside array.

```php
$select->join('LEFT', 'photos', 'pht', [
    'pht.user_id' => 'users.id',
    'users.is_admin' => 'pht.is_admin',
    static function (
        \Cycle\Database\Query\SelectQuery $select, 
        string $boolean, 
        callable $wrapper
    ): void {
        $select
            ->on('photos.user_id', 'users.id')
            ->onWhere('photos.type', 'avatar');
    }
]);
```

```sql
SELECT *
FROM `users`
         LEFT JOIN `photos` AS `pht`
                   ON (
                               `pht`.`user_id` = `users`.`id`
                           AND
                               `users`.`is_admin` = `pht`.`is_admin`
                           AND
                               (`photos`.`user_id` = `users`.`id` AND `photos`.`type` = "avatar")
                       )
```

#### Aliases

Second parameter in join methods are dedicated to table alias, feel free to use it in `on` and `where` statements of
your query:

```php
$select = $db->table('test')
    ->select(['test.*', 'uu.name as user_name'])
    ->innerJoin('users', 'uu')
    ->onWhere('uu.name', 'Anton');
```

Alternatively:

```php
$select = $db->table('test')
    ->select(['test.*', 'uu.name as user_name'])
    ->innerJoin('users as uu')
    ->onWhere('uu.name', 'Anton');
```

```sql
SELECT `primary_test`.*,
       `uu`.`name` as `user_name`
FROM `primary_test`
         INNER JOIN `primary_users` as `uu`
                    ON `uu`.`id` = `primary_test`.`id` AND `uu`.`name` = 'Anton'       
```

### OrderBy

User `orderBy` to specify sort direction:

```php
//We have a join, so table name is mandatory
$select
    ->orderBy('test.id', SelectQuery::SORT_DESC);
```

Multiple `orderBy` calls are allowed:

```php
$select
    ->orderBy(
        'test.name', SelectQuery::SORT_DESC
    )->orderBy(
        'test.id', SelectQuery::SORT_ASC
    );
```

Alternatively:

```php
$select
    ->orderBy([
        'test.name' => SelectQuery::SORT_DESC,
        'test.id'   => SelectQuery::SORT_ASC
    ]); 
```

Both ways will produce such SQL:

```sql
SELECT `primary_test`.*,
       `uu`.`name` as `user_name`
FROM `primary_test`
         INNER JOIN `primary_users` as `uu`
                    ON `uu`.`id` = `primary_test`.`id` AND `uu`.`name` = 'Anton'
ORDER BY `primary_test`.`name` DESC, `primary_test`.`id` ASC
```

> You can also use Fragments instead of sorting identifiers (by default identifiers are treated as column name or expression).

### GroupBy and Distinct

If you wish to select unique results from your selection use method `distinct` (always use `distinct` while loading
HAS_MANY/MANY_TO_MANY relations in ORM).

```php
$select->distinct();
```

Result grouping is available using `groupBy` method:

```php
$select = $db->table('test')
    ->select(['status', 'count(*) as count'])
    ->groupBy('status');
```

As you might expect produced SQL looks like:

```sql
SELECT `status`,
       count(*) as `count`
FROM `primary_test`
GROUP BY `status`
```

### Aggregations and Count

Since you can manipulate with selected columns including COUNT and other aggregation functions into your query might
look like:

```php
$select = $db->table('test')->select(['COUNT(*)']);
```

Though, in many cases you want to get query count or summarize results without column manipulations, use `count`, `avg`
, `sum`, `max` and `min` methods to do that:

```php
$select = $db->table('test')
    ->select(['id', 'name', 'status']);

print_r($select->count());
print_r($select->sum('balance'));
```

```sql
SELECT COUNT(*)
FROM `primary_test`;

SELECT SUM(`balance`)
FROM `primary_test`;
```

### Pagination

You can paginate your query using methods `limit` and `offset`:

```php
$select = $db->table('test')
    ->select(['id', 'name', 'status'])
    ->limit(10)
    ->offset(1);

foreach ($select as $row) {
    print_r($row);
}
```

## UpdateQuery Builder

Use "update" method of your table or database instance to get access to UpdateQuery builder, call `run` method of such
query to execute result:

```php
$update = $db->table('test')
    ->update(['name' => 'Abc'])
    ->where('id', '<', 10)
    ->run();
```

```sql
UPDATE `primary_test`
SET `name` = 'Abc'
WHERE `id` < 10
```

You can use `Expression` and `Fragment` instances in your update values:

```php
$update = $db->table('test')
    ->update([
        'name' => new \Cycle\Database\Injection\Expression('UPPER(test.name)')
    ])
    ->where('id', '<', 10)
    ->run();
```

```sql
UPDATE `primary_test`
SET `name` = UPPER(`primary_test`.`name`)
WHERE `id` < 10
```

Nested queries are also supported:

```php
$update = $db->table('test')
    ->update([
        'name' => $database
            ->table('users')
            ->select('name')
            ->where('id', 1)
    ])
    ->where('id', '<', 10)->run();
```

```sql
UPDATE `primary_test`
SET `name` = (SELECT `name`
              FROM `primary_users`
              WHERE `id` = 1)
WHERE `id` < 10 
```

> Where methods work identically as in SelectQuery.

## DeleteQuery Builders

Delete queries are represent by DeleteQuery accessible via "delete" method:

```php
$db->table('test')
    ->delete()
    ->where('id', '<', 1000)
    ->run();
```

You can also specify where statement in Table `delete` method in a form of where array:

```php
$db->table('test')
    ->delete([
        'id' => ['>' => 1000]
    ])
    ->run();
```

## Complex Expressions

You can use Expression object to create complex, driver-specific, SQL injections with included parameters.

```php
$db->table('test')
    ->select()
    ->where(new \Cycle\Database\Injection\Expression('SUM(column) = ?', $value))
    ->run();
```
