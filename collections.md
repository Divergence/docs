### [⤺ Back to Table of Contents](README.md#divergence-framework-documentation)

# Collections

A Collection keeps a set of objects in memory. You can iterate it, index fields, find records, and do math over the results without going back to the database.

There are two versions. `Divergence\Data\Collections\Collection` works with ordinary PHP objects. `Divergence\Models\Collections\RecordCollection` adds model validation, persistence, and index updates for ActiveRecord models. You don't need the ORM to use collections.

## Making a Collection

Use the factory for a plain collection. It sets up the collection's handlers.

```php
use Divergence\Data\Collections\Factory\Factory;

$receipts = (new Factory())->create([
    (object) ['ID' => 1, 'City' => 'New York', 'TotalCents' => 10888],
    (object) ['ID' => 2, 'City' => 'Portland', 'TotalCents' => 2500],
    (object) ['ID' => 3, 'City' => 'New York', 'TotalCents' => 5444],
], ['ID', 'City', 'TotalCents']);

echo count($receipts); // 3
echo $receipts[0]->TotalCents; // 10888

foreach ($receipts as $receipt) {
    echo $receipt->City;
}
```

Pass objects, not arrays of field values. The collection's record keys use object identity for non-ORM objects. Adding the same object twice does not create a second entry; two separate objects with the same fields are still two records.

`toArray()` returns the collection's records. It does not turn each object into an array or clone it.

## Indexing and Finding

The second factory argument is the list of fields to index. You can add an index later:

```php
$receipts->createIndexByField('City');

$receipt = $receipts->getByField('ID', 2);
$newYork = $receipts->getAllByField('City', 'New York');

echo $receipt->TotalCents; // 2500
echo count($newYork); // 2
echo $newYork->sum(fn ($receipt) => $receipt->TotalCents); // 16332
```

`getByField()` returns one object or `null`. `getAllByField()` returns another collection, including when nothing matches. These two methods expect the field to already be indexed. An unindexed field produces no matches; it does not fall back to scanning the records.

Criteria queries can build missing indexes for you:

```php
use Divergence\Models\Expr\Criteria;
use Divergence\Models\Expr\CriteriaType;

$largeReceipts = $receipts->getAllByCriteria([
    new Criteria('City', 'New York'),
    new Criteria('TotalCents', 10000, CriteriaType::GreaterThanOrEqual),
]);

echo count($largeReceipts); // 1
```

Despite the namespace, these expression classes work with plain collections too. An array of criteria means AND. `getAllByCriteria()` returns an **array of objects**, not a collection. `getByCriteria()` returns one object or `null`. An empty criteria group returns no matches; use `toArray()` to get everything.

### Groups

Use `CriteriaGroup` when you need OR or nested conditions:

```php
use Divergence\Models\Expr\Conjunction;
use Divergence\Models\Expr\CriteriaGroup;

$matches = $receipts->getAllByCriteria(new CriteriaGroup([
    new Criteria('City', 'Portland'),
    new CriteriaGroup([
        new Criteria('City', 'New York'),
        new Criteria('TotalCents', 10000, CriteriaType::GreaterThan),
    ]),
], Conjunction::GroupOr));
```

`GroupNotAnd` negates the intersection. `GroupNotOr` negates the union. Both operate against the records in this collection, not a database table.

### Operators

| CriteriaType | What it does in a collection |
| --- | --- |
| `Equal`, `NotEqual` | Match or exclude a value |
| `GreaterThan`, `GreaterThanOrEqual`, `LessThan`, `LessThanOrEqual` | Compare against an ordered value |
| `In`, `NotIn` | Match or exclude values from an array |
| `Like`, `NotLike` | Case-insensitive pattern match; `%` matches any number of characters, `_` matches one |
| `Nulled`, `NotNulled` | Match or exclude `null` |
| `Exists`, `NotExists` | Aliases for non-null and null checks, not SQL subqueries |
| `FieldEqual`, `FieldNotEqual`, `FieldGreaterThan`, `FieldGreaterThanOrEqual`, `FieldLessThan`, `FieldLessThanOrEqual` | Compare two fields on each record; pass the other field name as the value |

`Raw` is for SQL expressions. It is not an in-memory query language.

Use values that match the field's type. In particular, null, booleans, strings, and integers are not interchangeable equality keys. Model field mapping can normalize values before they reach the index. Don't assume every comparison has your database's coercion or collation rules.

Equality queries use the field's cardinality lookup. Range queries cache ordering and search the range boundaries. LIKE still has to inspect the indexed values. Building an index also costs time and memory; pre-index fields you query repeatedly.

## Adding, Removing, and Changing Records

```php
$receipt = (object) ['ID' => 4, 'City' => 'Portland', 'TotalCents' => 900];
$receipts->add($receipt);
$receipts->remove($receipt);
```

`addMany()` and `removeMany()` accept groups of records. Removing a record from a collection does not delete it from the database.

For a plain object, remove it before changing an indexed value, then add it again:

```php
$receipt = $receipts->getByField('ID', 2);
$receipts->remove($receipt);
$receipt->City = 'New York';
$receipts->add($receipt);
```

The collection cannot intercept arbitrary writes to a normal PHP object. The remove/add also moves that object to the end of the collection. Don't modify `Index`, `Indexes`, or `HashKeyIndex` directly to manage records; those structures need to agree.

## ORM Collections

You can construct a `RecordCollection` explicitly:

```php
use Divergence\Models\Collections\RecordCollection;
use yourApp\Models\Tag;

$tags = new RecordCollection(Tag::getAll(), ['ID', 'Tag'], Tag::class);
```

Or opt a model into collection results from its bulk getters:

```php
use Divergence\Models\Mapping\InMemoryIndexing;

#[InMemoryIndexing(indexes: ['Tag'])]
class Tag extends \Divergence\Models\Model
{
    public static $tableName = 'tags';

    #[\Divergence\Models\Mapping\Column]
    private string $Tag;
}
```

Now `Tag::getAll()` returns a `RecordCollection`. Without the attribute, bulk model getters normally return arrays. Raw-record getters still return database rows. Code accepting either result should use `foreach` and `count()`, or deliberately convert a collection with `toArray()`.

Model collections track mapped field changes through model events. Persisted records are identified by primary key; unsaved records use object identity until they are saved. Keep a record collection scoped to the intended model class.

```php
$tags = Tag::getAll();
$tag = $tags->getByField('Tag', 'Divergence');

if ($tag) {
    $tag->Tag = 'Divergence Framework';
    $tags->save();
}
```

`save()` saves dirty and unsaved models. `saveWithTransaction()` does that inside a database transaction. On a caught exception, it rolls the database back, restores the collection's model state and indexes, and rethrows the exception. It does not undo external effects from your event handlers, such as sending mail or writing a file.

Neither method means that removing an object from the collection deletes its row. Call the model's `destroy()` when that is what you want.

## Math

All collections expose the same [Math methods](math.md): totals, percentiles, distributions, rankings, rolling windows, and more. The calculations use records already in memory. They don't issue aggregate SQL queries.
