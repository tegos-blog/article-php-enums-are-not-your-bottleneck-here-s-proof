When you're building a large export - say, 50_000 order items, you start looking at every part of the code and wondering: *what's slowing this down?*

Here we have some order item export simplified code:

```php
final class OrderItemExport
{
    public function store(): void
    {
        $rows = $this->query()->toBase()->cursor();

        foreach ($rows as $row) {
            $this->writer->addRow($this->map($row));
        }

        $this->writer->close();
    }

    private function map(object $row): array
    {
        $status = OrderItemStatusEnum::tryFrom($row->status_id);
        $currency = CurrencyEnum::tryFrom($row->currency_id);

        return [
            $row->order_id,
            $status?->label(),
            $currency?->code(),
        ];
    }
}
```

Enums are an easy target. They look like objects, they have methods, and you call them on every row. So the question came up naturally: do PHP enums create overhead in a loop like this?

```php
$orderItemStatus = OrderItemStatusEnum::tryFrom($row->status_id);
$statusLabel = $orderItemStatus?->label();
```

Let's find out.

## Enums in PHP are singletons

PHP 8.1 enums are objects - but each case is implemented as a singleton. Each enum case is instantiated once and reused for the lifetime of the request. That means:

```php
$a = OrderItemStatusEnum::tryFrom(7);
$b = OrderItemStatusEnum::tryFrom(7);

$a === $b; // true
```

Same instance, every time. No new allocations, no garbage to collect.

## The optimization experiment

To be sure, I refactored the export to use pre-cached arrays instead of calling `tryFrom()` on every row:

```php
// Constructor
$this->statusLabels = OrderItemStatusEnum::labels();
$this->currencyNames = CurrencyEnum::names();

// Per row only array lookup
$this->statusLabels[$rowOrderItem->status_id] ?? null,
```

Also moved JSON localization to a raw SQL expression, so PHP doesn't decode JSON on every row:

```php
->selectRaw("order_items.name->>'$.$locale' as localized_name")
```

Then I ran both versions against 50_000 rows with proper memory and GC profiling.

## The results

| Metric            | Before  | After   |
|-------------------|---------|---------|
| Time elapsed      | 4281 ms | 4265 ms |
| Rows/sec          | 11,679  | 11,723  |
| Memory used       | 8.5 MB  | 8.5 MB  |
| GC runs           | 0       | 0       |
| Objects collected | 0       | 0       |

Nothing changed. Literally.
That's a 0.37% difference is just statistically irrelevant in real-world workloads.

## What does this tell us?

GC never triggered because enum cases are persistent singletons and do not create cyclic references or short-lived heap allocations. Memory stayed flat because singletons don't accumulate. The 16ms diffrence is just noise. The real bottleneck in large exports is **writing, I/O, and data formatting**, not enum resolution.

## Practical things

- Use `tryFrom()` when you need clean, readable code. It's safe.
- Pre-cache labels as arrays if you access them thousands of times, not for memory, but for micro-call overhead.
- Don't profile enums through `gc_status()` - GC won't see them. Use Xdebug, Blackfire, php-meminfo for real memory/time analysis.
- Enums are a feature for **safety and readability**, not a performance risk.

The optimization was real. The improvement... wasn't. And that's actually a good result: it means your enums are fine.

## Author's Note

Thanks for sticking around!
Find me on [dev.to](https://dev.to/tegos), [linkedin](https://www.linkedin.com/in/ivan-mykhavko/), or you can check out my work on [github](https://github.com/tegos).

**Notes from real-world Laravel.**