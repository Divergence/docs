### [⤺ Back to Table of Contents](README.md#divergence-framework-documentation)

# Math

Once your records are in a collection, you should be able to do something useful with them. Add up receipts. Find the slowest requests. See what an ordinary transaction looks like and which ones are way outside that range.

These methods work on both plain collections and ORM record collections. Pass a selector that returns the field or calculated value you want to measure.

## Start With Some Numbers

```php
use Divergence\Data\Collections\Factory\Factory;

$requests = (new Factory())->create(array_map(
    fn ($latency) => (object) ['Latency' => $latency],
    [10, 20, 30, 40]
));

$latency = fn ($request) => $request->Latency;

echo $requests->sum($latency); // 100
echo $requests->median($latency); // 25
echo $requests->percentile($latency, 95); // 40
```

Numeric selectors must return finite PHP integers or floats. A numeric string is still a string. `null`, booleans, `NAN`, and infinity are rejected with `InvalidArgumentException`, not quietly treated as zero. Convert values deliberately in your selector if your input needs it.

The operations don't reorder the original collection. Results are calculated in memory, and several methods allocate arrays or sort a copy. This isn't a streaming statistics engine or a substitute for a database aggregate over millions of rows.

## Method Reference

| Method | Result | Empty collection |
| --- | --- | --- |
| `sum($selector)` | Total, as an integer or float | `0` |
| `median($selector)` | Middle value; mean of the middle pair for an even count | `null` |
| `percentile($selector, $percentile)` | Nearest-rank percentile, from 0 through 100 | `null` |
| `quantile($selector, $quantile)` | Nearest-rank quantile, from 0 through 1 | `null` |
| `variance($selector)` | Population variance | `null` |
| `stddev($selector)` | Population standard deviation | `null` |
| `histogram($selector, $bucketCount = 10)` | List of `min`, `max`, `count` buckets | `[]` |
| `mode($selector)` | List of all tied modes | `[]` |
| `frequency($selector)` | List of `value`, `count` pairs | `[]` |
| `countBy($selector)` | Alias for `frequency()` | `[]` |
| `covariance($firstSelector, $secondSelector)` | Population covariance | `null` |
| `correlation($firstSelector, $secondSelector)` | Pearson correlation | `null` |
| `topK($selector, $count)` | Highest-ranked records, descending | `[]` |
| `bottomK($selector, $count)` | Lowest-ranked records, ascending | `[]` |
| `movingAverage($selector, $windowSize)` | Means of complete windows | `[]` |
| `rolling($windowSize, $callback)` | Callback result for each complete window | `[]` |
| `zScore($selector)` | One population z-score per record | `[]` |
| `outliers($selector, $threshold = 3)` | Records whose absolute z-score meets the threshold | `[]` |

Invalid method arguments are still invalid on an empty collection. Bucket and window sizes must be positive integers. Ranking counts cannot be negative. Percentiles, quantiles, and outlier thresholds must be finite and in range.

## Percentiles Aren't Always Medians

`quantile()` uses nearest rank: sort the values and select rank `ceil(q * count)`, with the endpoints clamped to the first and last values. `percentile()` is the same calculation with a 0–100 input. Products within floating-point epsilon of an integer are treated as that boundary.

For `[10, 20, 30, 40]`:

```php
echo $requests->median($latency); // 25
echo $requests->quantile($latency, 0.5); // 20
echo $requests->percentile($latency, 50); // 20
echo $requests->percentile($latency, 0); // 10
echo $requests->percentile($latency, 100); // 40
```

That difference is intentional. `median()` averages the middle two values. Nearest-rank p50 selects a value from the dataset. If you're comparing results to another statistics package, check which percentile definition it uses.

## Variance, Standard Deviation, and Relationships

Variance and covariance divide by the number of records, not `count - 1`. These are population statistics, not sample estimators.

```php
echo $requests->variance($latency); // 125
echo $requests->stddev($latency); // approximately 11.1803398875

$twiceLatency = fn ($request) => $request->Latency * 2;
echo $requests->covariance($latency, $twiceLatency); // 250
echo $requests->correlation($latency, $twiceLatency); // approximately 1
```

The two relationship selectors are evaluated on the same records. A constant series has zero variance and standard deviation. Correlation is `null` when either series has zero variance, including a single-record dataset; there isn't a meaningful correlation to report.

## Histograms

```php
$buckets = $requests->histogram($latency, 3);
// [
//     ['min' => 10.0, 'max' => 20.0, 'count' => 1],
//     ['min' => 20.0, 'max' => 30.0, 'count' => 1],
//     ['min' => 30.0, 'max' => 40.0, 'count' => 2],
// ]
```

Buckets include their minimum and exclude their maximum, except the last bucket includes both. A constant distribution returns one bucket with equal minimum and maximum. Empty buckets are retained for non-constant data.

Bounds are floats. If the range overflows or the requested buckets are too narrow for floating-point resolution, the method throws `InvalidArgumentException`.

## Frequency and Mode

These selectors can return non-numeric values:

```php
$events = (new Factory())->create(array_map(
    fn ($status) => (object) ['Status' => $status],
    ['paid', 'pending', 'paid', 'pending', 'refunded']
));

$counts = $events->countBy(fn ($event) => $event->Status);
// [
//     ['value' => 'paid', 'count' => 2],
//     ['value' => 'pending', 'count' => 2],
//     ['value' => 'refunded', 'count' => 1],
// ]

$modes = $events->mode(fn ($event) => $event->Status);
// ['paid', 'pending']
```

Results keep first-seen order. Tied modes are all returned. If every value appears once, every value is a mode.

The result is a list of pairs, not an array keyed by the selected value. That matters: `1`, `'1'`, `true`, `false`, `null`, and `''` remain separate categories. Composite values are grouped by their serialized representation, not object identity. Prefer a scalar category such as a status or jurisdiction code; closures and resources aren't useful grouping keys.

## Rankings and Outliers

```php
$slowest = $requests->topK($latency, 2); // records with 40, then 30
$fastest = $requests->bottomK($latency, 2); // records with 10, then 20
$scores = $requests->zScore($latency);
$unusual = $requests->outliers($latency, 1); // records with 10 and 40
```

Rankings return the original records, not their selected numbers. Ties preserve input order. A count of zero returns `[]`; a count larger than the collection returns every record in ranked order.

`zScore()` uses `(value - mean) / populationStddev`. A constant dataset returns zeros. `outliers()` uses `abs(zScore) >= threshold`, so a threshold of zero includes every record. The threshold cannot be negative. A high score is a statistical signal, not proof of fraud or a bad transaction.

## Moving Averages and Rolling Windows

```php
$averages = $requests->movingAverage($latency, 3);
// [20, 30]

$ranges = $requests->rolling(3, function (array $window) {
    $values = array_map(fn ($request) => $request->Latency, $window);
    return max($values) - min($values);
});
// [20, 20]
```

A window moves forward one record at a time. Only complete windows are returned. For `n` records and a window of `w`, that is `max(0, n - w + 1)` results. A window larger than the collection returns `[]`.

Order is whatever order the collection already has. These methods don't look for timestamps or sort by date. Put records in the right order first. The rolling callback receives an array of the original records; don't mutate them unless you mean to.

## Accounting With Integer Cents

If you need exact cents, store integer cents. A `decimal` model field maps to a PHP float; the SQL column's scale does not turn PHP arithmetic into decimal arithmetic. `sum()` uses PHP's `array_sum()`, so adding floats still has the usual floating-point limitations.

Here are two receipts using an illustrative tax rate of 8.875%. This example rounds tax once per receipt, half away from zero, including refunds. The rate is fixture data, not a tax-rate lookup.

```php
class TaxJurisdiction
{
    public string $Name;
    public int $RateNumerator;
    public int $RateDenominator;

    public function __construct(string $name, int $numerator, int $denominator)
    {
        $this->Name = $name;
        $this->RateNumerator = $numerator;
        $this->RateDenominator = $denominator;
    }

    public function taxCents(int $subtotalCents): int
    {
        $numerator = abs($subtotalCents) * $this->RateNumerator;
        $tax = intdiv($numerator, $this->RateDenominator);

        if (($numerator % $this->RateDenominator) * 2 >= $this->RateDenominator) {
            $tax++;
        }

        return $subtotalCents < 0 ? -$tax : $tax;
    }
}

class Receipt
{
    public int $SubtotalCents;
    public TaxJurisdiction $Jurisdiction;

    public function __construct(int $subtotalCents, TaxJurisdiction $jurisdiction)
    {
        $this->SubtotalCents = $subtotalCents;
        $this->Jurisdiction = $jurisdiction;
    }

    public function totalCents(): int
    {
        return $this->SubtotalCents + $this->Jurisdiction->taxCents($this->SubtotalCents);
    }
}

$newYork = new TaxJurisdiction('New York', 8875, 100000);
$receipts = (new Factory())->create([
    new Receipt(10000, $newYork),
    new Receipt(5000, $newYork),
]);

echo $receipts->sum(fn ($receipt) => $receipt->SubtotalCents); // 15000
echo $receipts->sum(fn ($receipt) => $receipt->Jurisdiction->taxCents($receipt->SubtotalCents)); // 1332
echo $receipts->sum(fn ($receipt) => $receipt->totalCents()); // 16332
```

Rounding each receipt and rounding one combined subtotal can produce different totals. Pick the rule your application needs and test it. Also keep amounts and intermediate products within PHP's integer range; integer overflow promotes arithmetic to floats. This small example isn't an arbitrary-precision money library.

The framework's accounting tests use a deterministic 200-receipt ledger with multiple jurisdictions, refunds, and boundary cases. The Math tests cover empty inputs, ties, constant distributions, invalid selectors, percentile boundaries, and numeric extremes. See [Developing](developing.md#unit-testing) for running them.

## Under the Hood

The collection methods forward to classes in `Divergence\Data\Collections\Math`:

| Class | Operations |
| --- | --- |
| `Aggregates` | Sum, median, percentile, quantile |
| `Deviations` | Variance, standard deviation, z-scores, outliers |
| `Distributions` | Histogram, frequency, countBy, mode |
| `Relationships` | Covariance, correlation |
| `Rankings` | topK, bottomK |
| `Windows` | Moving averages, rolling callbacks |

You normally don't need to call these directly. If you do, pass the collection as the first argument, followed by the same arguments as the collection method.
