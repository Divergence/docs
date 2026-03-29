### [⤺ Back to Table of Contents](/README.md#divergence-framework-documentation)

# ORM
Divergence uses a typical ActiveRecord pattern for its models.

## Model Architecture

### **If you do not want any default fields extend** `Divergence\Models\ActiveRecord`
### **If you would like to use default fields extend** `Divergence\Models\Model`

| Type | Field | Description |
| --- | --- | --- |
| `integer` | `ID` | The primary key. |
| `enum` | `Class` | Fully qualified PHP namespaced class. |
| `timestamp` | `Created` | Time when the object is created in the database. |
| `integer` | `CreatorID` | Reserved for use with your authentication system. |

`Divergence\Models\Model` automatically also pulls in `Divergence\Models\Getters`, so you do not have to do it yourself.

### Important Functionality
| Trait | Description |
| --- | --- |
| `Divergence\Models\Getters` | Suite of methods to pull records from the database. |
| `Divergence\Models\Relations` | Lets you build relationships between models. |
| `Divergence\Models\Versioning` | Automatically tracks history of models. |

Classes that use ActiveRecord may optionally use traits to enable relationship features and versioning features respectively.

#### Object Oriented Architecture
When using array mapping, ActiveRecord merges `public static $fields` and `public static $relationships` at runtime, giving priority to the child.

A child class may choose to unset a relationship or field simply by setting the config to `null`. A child class may also use a different type for the same database field name.

Overrides must use the key for the field configuration.

If you are using the older array-mapping style, you will usually define the common class configurables shown in this section. With attribute-based mapping, the framework can infer more of the model structure directly from your class.

#### Subclassing
```php
public static $rootClass = __CLASS__;
public static $defaultClass = __CLASS__;
public static $subClasses = [__CLASS__];
```

In the event that you have subclasses you can define them here. By default just use the above configuration. You'll want to override `$rootClass` and `$defaultClass` if necessary for yourself.

#### Table Name & Nouns
```php
public static $tableName = 'table';
public static $singularNoun = 'table';
public static $pluralNoun = 'tables';
```

Table name is for the database table.

Singular noun and plural noun are mostly used by `RecordsRequestHandler` to load the right template in HTML mode, so think of those as resource and template names.

#### Field Mapping

Divergence supports field mapping using PHP attributes as well as the older static-array style.

For example:

```php
#[Column(type: "integer", primary:true, autoincrement:true, unsigned:true)]
private $ID;
```

For field mapping by array, ActiveRecord also merges each `static::$fields` for every class from child to parent. Any defined `$fields` are usable as `$Model->$fieldName`.

```php
public static $fields = [
    'Tag',
    'Slug',
];
```

#### About Default Field Configs
By default, if you just have a string that will be treated as the name of the field for the model. In practice that means a PHP string and a short text column at the schema layer if the framework has to auto-create the table.

```php
protected $title;
```

#### Automatically Create Tables
If you try to use a model and the database responds with a table-not-found error, the framework can attempt to build the SQL, create the table, and rerun the original operation.

You can disable this behavior by setting:

```php
public static $autoCreateTables = false;
```

Note the current property name is `autoCreateTables`.

## Making a Basic Model
Here's an example of a minimum model:

```php
<?php
namespace yourApp\Models;

use Divergence\Models\Mapping\Column;

class Tag extends \Divergence\Models\Model
{
    // support subclassing
    public static $rootClass = __CLASS__;
    public static $defaultClass = __CLASS__;
    public static $subClasses = [__CLASS__];

    // ActiveRecord configuration
    public static $tableName = 'tags';
    public static $singularNoun = 'tag';
    public static $pluralNoun = 'tags';

    #[Column(type: 'string', required: true, notnull: true)]
    private $Tag;

    #[Column(type: 'string', blankisnull: true, notnull: false)]
    private $Slug;
}
```

We get these fields from `\Divergence\Models\Model` as defaults:

```php
#[Column(type: "integer", primary:true, autoincrement:true, unsigned:true)]
private $ID;

#[Column(type: "enum", notnull:true, values:[])]
private $Class;

#[Column(type: "timestamp", default:'CURRENT_TIMESTAMP')]
private $Created;

#[Column(type: "integer", notnull:false)]
private $CreatorID;
```

## Create, Update, and Delete
Divergence ActiveRecord is simple and makes use of normal PHP object patterns whenever possible.

### Creating
Example without defaults:

```php
$Tag = new Tag();
echo $Tag->Name; // prints null
$Tag->Name = 'Divergence';
echo $Tag->Name; // prints Divergence
```

Example with record instantiation via constructor:

```php
$Tag = new Tag([
    'Name' => 'Divergence',
]);
echo $Tag->Name; // prints Divergence
```

Example with record instantiation via `create()`:

```php
$Tag = Tag::create([
    'Name' => 'Divergence',
]);
echo $Tag->Name; // prints Divergence
```

Example with record instantiation via `create()` and save directly to the database:

```php
$Tag = Tag::create([
    'Name' => 'Divergence',
], true);
echo $Tag->Name; // prints Divergence
echo $Tag->ID; // prints ID assigned by the database auto increment
```

Another save example:

```php
$Tag = new Tag();
$Tag->Name = 'Divergence';
echo $Tag->ID; // prints null
$Tag->save();
echo $Tag->ID; // prints ID assigned by the database auto increment
```

### Update
```php
$Tag = Tag::getByID(1);
echo $Tag->ID; // prints 1
$Tag->Name = 'Divergence';
$Tag->save();
```

Get by field:

```php
$Tag = Tag::getByField('ID', 1);
echo $Tag->ID; // prints 1
$Tag->Name = 'Divergence';
$Tag->save();
```

### Delete
```php
$Tag = Tag::getByID(1);
echo $Tag->ID; // prints 1
$Tag->destroy(); // record still exists in the variable
```

or statically:

```php
Tag::delete(1); // returns true if affected rows > 0
```

## Getter Layer and Factory Runtime

The public model API still looks like classic Divergence:

- `getByID`
- `getByField`
- `getByHandle`
- `getByWhere`
- `getByQuery`
- `getAll`
- `getAllByField`
- `getAllByWhere`
- `getAllByQuery`
- `getUniqueHandle`

But the current implementation is more modular than older docs implied.

Today:

- `Divergence\Models\Getters` is a thin forwarding trait
- static getter calls route into `Divergence\Models\Factory`
- `Factory` registers dedicated getter classes such as `GetByID`, `GetByField`, `GetAllByWhere`, and `GetUniqueHandle`
- `Factory` also coordinates model metadata, instantiation, connection resolution, and storage caching

That means the external API is stable, but the query/runtime plumbing behind it has been decomposed into smaller pieces.

## Versioning
Your **model** must be defined with `use Versioning` in its definition.

```php
<?php
namespace Divergence\Tests\MockSite\Models;

use Divergence\Models\Model;
use Divergence\Models\Versioning;

class Tag extends Model
{
    use Versioning;
}
```

#### Configurables
You **must** provide these settings to use versioning.

```php
public static $historyTable = 'test_history';
public static $createRevisionOnDestroy = true;
public static $createRevisionOnSave = true;
```

If you did not create your tables yet, a versioned model can have its history table automatically created by the same missing-table path the main model uses.

#### Trait `\Divergence\Models\Versioning` provides these fields.

##### Definition
```php
#[Column(type: "integer", unsigned:true, notnull:false)]
private $RevisionID;
```

#### Trait `\Divergence\Models\Versioning` provides these methods.
| Method | Purpose |
| --- | --- |
| `getRevisionsByID` | Returns an array of versions of a model by ID and `$options`. |
| `getRevisions` | Returns an array of versions of a model by `$options`. |

#### Trait `\Divergence\Models\Versioning` provides these relationships.
| Relationship | Type | Purpose |
| --- | --- | --- |
| `History` | `history` | Pulls old versions of this model |

##### Definition
```php
'History' => [
    'type' => 'history',
    'order' => ['RevisionID' => 'DESC'],
],
```

##### Example
```php
$Model = Tag::getByID(1);
$Model->History; // array of revisions where ID == 1 ordered by RevisionID
```

## Relationships
Your model **must** be defined with `use Relations` in its definition.

```php
<?php
namespace Divergence\Tests\MockSite\Models;

use Divergence\Models\Model;
use Divergence\Models\Relations;

class Tag extends Model
{
    use Relations;
}
```

#### Configurables
For array mapping you **must** provide relationship configurations in the static variable `$relationships`.

```php
public static $relationships = [
    /*
        'RelationshipName' => [
            ... config ...
        ]
    */
];
```

Otherwise you can define it with attributes like so:

```php
#[Relation(
    type:'one-one',
    class:Tag::class,
    local: 'ThreadID',
    foreign: 'ID',
)]
protected $Tag;
```

#### Keep in Mind
- Relationships should not have the same name.
- The second will override the first.
- Children classes can override parent classes by setting the class configuration to `null`.
- Relationship configs will be stacked with priority given to the child class.
- Relationships are callable by their key name from `$this->$relationshipKey`, but model field names take priority.

## Relationships Reference
Internally, the relationship resolver supports:

- `one-one`
- `one-many`
- `many-many`
- `context-parent`
- `context-children`
- `history`

### `one-one`
Use `one-one` when the current record points to exactly one related record.

```php
#[Relation(
    type:'one-one',
    class:User::class,
    local: 'AuthorID',
    foreign: 'ID',
)]
protected $Author;
```

Defaults:

- omitting `type` behaves like `one-one`
- omitting `local` defaults it to `<RelationshipName>ID`
- omitting `foreign` defaults it to `ID`

### `one-many`
Use `one-many` when the current record owns a collection of related records.

```php
#[Relation(
    type:'one-many',
    class:Thread::class,
    local: 'ID',
    foreign: 'CategoryID'
)]
protected $Threads;
```

You can also add `conditions` and `order`.

### `many-many`
Use `many-many` when two models are connected through a join model or join table.

```php
#[Relation(
    type:'many-many',
    class:Tag::class,
    linkClass:PostTags::class,
    linkLocal: 'PostID',
    linkForeign: 'TagID',
    local: 'ID',
    foreign: 'ID'
)]
protected $Tags;
```

### `context-parent`
Use `context-parent` when a record stores a polymorphic parent reference through `ContextClass` and `ContextID`.

```php
#[Relation(
    type:'context-parent',
    local: 'ContextID',
    classField: 'ContextClass'
)]
protected $Context;
```

### `context-children`
Use `context-children` for the inverse of `context-parent`.

```php
#[Relation(
    type:'context-children',
    class:Media::class,
    local: 'ID',
    contextClass: Post::class
)]
protected $Media;
```

### `history`
Use `history` with versioned models to expose prior revisions.

```php
'History' => [
    'type' => 'history',
    'order' => ['RevisionID' => 'DESC'],
],
```

## Examples
#### One-One
```php
#[Relation(
    type:'one-one',
    class:Tag::class,
    local: 'ThreadID',
    foreign: 'ID',
)]
protected $Tag;

#[Relation(
    type:'one-one',
    class:Post::class,
    local: 'PostID',
    foreign: 'ID',
)]
protected $Post;
```

#### One-Many
Feel free to create multiple relationship configurations with different conditions and orders.

```php
#[Relation(
    type:'one-many',
    class:Thread::class,
    local: 'ID',
    foreign: 'CategoryID'
)]
protected $Threads;

#[Relation(
    type:'one-many',
    class:Thread::class,
    local: 'ID',
    foreign: 'CategoryID',
    conditions: [
        'Created > DATE_SUB(NOW(), INTERVAL 1 HOUR)',
    ],
    order: ['Title' => 'ASC']
)]
protected $RecentThreads;
```

## Supported Field Types

| Field Type | Typical Use |
| --- | --- |
| `int` | Integer values |
| `integer` | Integer values |
| `uint` | Unsigned integer values |
| `string` | Short text |
| `clob` | Long text |
| `float` | Approximate decimal values |
| `decimal` | Exact fixed-point values |
| `enum` | Controlled one-of-many values |
| `boolean` | True/false flags |
| `password` | Hashed secret material |
| `timestamp` | Date + time values |
| `date` | Date-only values |
| `serialized` | Serialized structured data |
| `set` | Multi-value controlled sets |
| `list` | Ordered delimited lists |
| `binary` | Raw binary blobs such as session IP storage |

## ORM Typing Explanation
This section explains what each field type means in practice, how the framework stores it, and what you should expect when reading and writing values.

### `int`
Whole-number numeric field.

### `integer`
Also a whole-number field. In practice `int` and `integer` are the same family here.

### `uint`
Unsigned integer. Should never be negative.

### `string`
Short text, usually the right fit for names, titles, slugs, and handles.

### `clob`
Long-form text for bodies, descriptions, and content.

### `float`
Approximate decimal values. Fine for measurements where small rounding drift is acceptable.

### `decimal`
Exact fixed-point decimal values. Use for money or values where precision matters.

### `enum`
Restricts a field to one of a predefined set of values.

### `boolean`
True/false flag field.

### `password`
Intended for hashed secrets rather than plaintext input.

### `timestamp`
Time values with time-of-day precision.

### `date`
Calendar dates without time-of-day precision.

### `serialized`
Stores structured PHP data serialized into text.

### `set`
Stores multiple values from a controlled list.

### `list`
Stores an ordered delimited list of values.

### `binary`
Stores raw binary data. The session model uses this for `LastIP`.

## Canary Model - An Example Utilizing Every Field Type
The mock test app ships a `Canary` model that exists specifically as an example of field mapping coverage.

```php
<?php
namespace App\Models;

use Divergence\Models\Versioning;
use Divergence\Models\Mapping\Column;

class Canary extends \Divergence\Models\Model
{
    use Versioning;

    public static $tableName = 'canaries';
    public static $historyTable = 'canaries_history';
    public static $createRevisionOnDestroy = true;
    public static $createRevisionOnSave = true;

    #[Column(type: 'int', default:7)]
    protected $ContextID;

    #[Column(type: 'enum', values: [Tag::class], default: Tag::class)]
    protected $ContextClass;

    #[Column(type: 'clob', notnull:true)]
    protected $DNA;

    #[Column(type: 'string', required: true, notnull:true)]
    protected $Name;

    #[Column(type: 'string', blankisnull: true, notnull:false)]
    protected $Handle;

    #[Column(type: 'boolean', default: true)]
    protected $isAlive;

    #[Column(type: 'password')]
    protected $DNAHash;

    #[Column(type: 'timestamp', notnull: false)]
    protected $StatusCheckedLast;

    #[Column(type: 'serialized')]
    protected $SerializedData;

    #[Column(type: 'set', values: ["red", "blue", "green"])]
    protected $Colors;

    #[Column(type: 'list', delimiter: '|')]
    protected $EyeColors;

    #[Column(type: 'float')]
    protected $Height;

    #[Column(type: 'int', notnull: false)]
    protected $LongestFlightTime;

    #[Column(type: 'uint')]
    protected $HighestRecordedAltitude;

    #[Column(type: 'integer', notnull: true)]
    protected $ObservationCount;

    #[Column(type: 'date')]
    protected $DateOfBirth;

    #[Column(type: 'decimal', notnull: false, precision: 5, scale: 2)]
    protected $Weight;
}
```

## Validation
Validation is available to you through a static config in your model. The config is an array of validator configs. Whenever possible Divergence validators use built in PHP validation helpers.

Validators are evaluated in the order in which they appear. `save()` calls `validate()` before persistence.

#### A snippet from ActiveRecord's save path.
```php
if (!$this->validate($deep)) {
    throw new Exception('Cannot save invalid record');
}
```

##### Deep is true by default. It will validate loaded relationships as well.

Set validators in your model:

```php
public static $validators = [
    [
        'field' => 'Name',
        'required' => true,
        'errorMessage' => 'Name is required.',
    ],
];
```

### Examples
```php
[
    'field' => 'Name',
    'minlength' => 2,
    'required' => true,
    'errorMessage' => 'Name is required.',
]
```

```php
[
    'field' => 'Name',
    'maxlength' => 5,
    'required' => true,
    'errorMessage' => 'Name is too big. Max 5 characters.',
]
```

```php
[
    'field' => 'ID',
    'required' => true,
    'validator' => 'number',
    'max' => PHP_INT_MAX,
    'min' => 1,
    'errorMessage' => 'ID must be between 1 and PHP_INT_MAX ('.PHP_INT_MAX.')',
]
```

```php
[
    'field' => 'Float',
    'required' => true,
    'validator' => 'number',
    'max' => 0.759,
    'min' => 0.128,
    'errorMessage' => 'Float must be between 0.128 and 0.759',
]
```

Email validation:

```php
[
    'field' => 'Email',
    'required' => true,
    'validator' => 'email',
]
```

Custom validation:

```php
[
    'field' => 'Email',
    'required' => true,
    'validator' => [
        Validate::class,
        'email',
    ],
]
```

## Event Binding
Every ActiveRecord save will call `$class::$beforeSave` and `$class::$afterSave` if they are set to PHP callables.

If you set `ActiveRecord::$beforeSave` you can hook into every save for every model on the entire site.

Both `$beforeSave` and `$afterSave` get passed an instance of the object being saved as the only parameter.

Events are not overridden by child classes. An event will fire for every parent of a child class.

#### The two relevant snippets from ActiveRecord's event-definition path.
```php
if (is_callable($class::$beforeSave)) {
    if (!empty($class::$beforeSave)) {
        if (!in_array($class::$beforeSave, static::$_classBeforeSave)) {
            static::$_classBeforeSave[] = $class::$beforeSave;
        }
    }
}
```

```php
if (is_callable($class::$afterSave)) {
    if (!empty($class::$afterSave)) {
        if (!in_array($class::$afterSave, static::$_classAfterSave)) {
            static::$_classAfterSave[] = $class::$afterSave;
        }
    }
}
```

Also note that the current save flow routes through handler classes:

- `beforeSaveHandler`
- `afterSaveHandler`
- `saveHandler`
- `destroyHandler`
- `deleteHandler`

So the framework's event and persistence lifecycle is more modular than older docs implied, even though the conceptual hooks are the same.

## Advanced Techniques
Here are a few examples of how to use ActiveRecord but still do custom things with your model.

### Dynamic Fields
This is a case where you'll want to extend `getValue($field)`.

```php
public function getValue($field)
{
    switch ($field) {
        case 'HeightCM':
            return static::inchesToCM($this->Height);

        case 'calculateTax':
            return $this->calculateTaxTotal();

        default:
            return parent::getValue($field);
    }
}

public static function inchesToCM($value)
{
    return $value * 2.54;
}

public function calculateTaxTotal()
{
    $taxTotal = 0;
    if ($state = $this->getStateTaxRate()) {
        $taxTotal += ($state * $this->Price);
    }
    if ($local = $this->getLocalTaxRate()) {
        $taxTotal += ($local * $this->Price);
    }
    return $taxTotal;
}
```

### Get Models By Custom Join
In this example we let the table names come right from the class. We also make sure our query only gives us the one model we actually want to instantiate from the data.

#### Standalone Example

```php
if (App::$App->is_loggedin()) {
    $where = "`Status` IN ('Draft','Published')";
} else {
    $where = "`Status` IN ('Published')";
}

$BlogPosts = BlogPost::getAllByQuery(
    "SELECT `bp`.* FROM `%s` `bp`
    INNER JOIN %s as `t` ON `t`.`BlogPostID`=`bp`.`ID`
    WHERE `t`.`TagID`='%s' AND $where",
    [
        BlogPost::$tableName,
        PostTags::$tableName,
        $Tag->ID,
    ]
);
```

#### Same thing as a Dynamic Field
```php
public function getValue($field)
{
    switch ($field) {
        case 'getAllByTag':
            return static::getAllByTag($_REQUEST['tag']);
        default:
            return parent::getValue($field);
    }
}

public static function getAllByTag($slug)
{
    if ($Tag = Tag::getByField('Slug', $slug)) {
        if (App::$App->is_loggedin()) {
            $where = "`Status` IN ('Draft','Published')";
        } else {
            $where = "`Status` IN ('Published')";
        }

        return static::getAllByQuery(
            "SELECT `bp`.* FROM `%s` `bp`
            INNER JOIN %s as `t` ON `t`.`BlogPostID`=`bp`.`ID`
            WHERE `t`.`TagID`='%s' AND $where",
            [
                static::$tableName,
                PostTags::$tableName,
                $Tag->ID,
            ]
        );
    }
}
```
