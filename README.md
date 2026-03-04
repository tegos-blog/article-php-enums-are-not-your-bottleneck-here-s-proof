# PHP Enums Are Not Your Bottleneck (Here's Proof)

![PHP Enums Are Not Your Bottleneck (Here's Proof)](assets/poster.jpg)

When exporting 50,000+ rows in Laravel, enum resolution often becomes a suspect in performance discussions.

This article benchmarks PHP 8.1 enums in a real-world export scenario and measures their actual impact on execution time, memory usage, and garbage collection.

## What's inside:

* How PHP enums are implemented (singleton cases)
* Whether `tryFrom()` creates new objects
* Benchmark results from a 50,000-row export
* `tryFrom()` vs pre-cached array lookup comparison
* Memory and GC profiling analysis
* Identification of the real performance bottlenecks

## 📎 Read Full

[PHP Enums Are Not Your Bottleneck (Here's Proof)](https://dev.to/tegos/php-enums-are-not-your-bottleneck-heres-proof-1887)