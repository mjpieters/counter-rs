# counter

Counter counts recurrent elements of iterables. It is based on [the Python
implementation](https://docs.python.org/3/library/collections.html#collections.Counter).

The struct [`Counter`](struct.Counter.html) is the entry-point type for this module.

## Examples

### Just count an iterable

```rust
use counter::Counter;
let char_counts = "barefoot".chars().collect::<Counter<_>>();
let counts_counts = char_counts.values().collect::<Counter<_>>();
```

### Update a count

```rust
let mut counts = "aaa".chars().collect::<Counter<_>>();
counts[&'a'] += 1;
counts[&'b'] += 1;
```

```rust
let mut counts = "able babble table babble rabble table able fable scrabble"
    .split_whitespace().collect::<Counter<_>>();
// add or subtract an iterable of the same type
counts += "cain and abel fable table cable".split_whitespace();
// or add or subtract from another Counter of the same type
let other_counts = "scrabble cabbie fable babble"
    .split_whitespace().collect::<Counter<_>>();
let difference = counts - other_counts;
```

Extend a `Counter` with another `Counter`:
```rust
let mut counter = "abbccc".chars().collect::<Counter<_>>();
let another = "bccddd".chars().collect::<Counter<_>>();
counter.extend(&another);
let expect = [('a', 1), ('b', 3), ('c', 5), ('d', 3)].iter()
    .cloned().collect::<HashMap<_, _>>();
assert_eq!(counter.into_map(), expect);
```
### Get items with keys

```rust
let counts = "aaa".chars().collect::<Counter<_>>();
assert_eq!(counts[&'a'], 3);
assert_eq!(counts[&'b'], 0);
```

### Get the most common items

[`most_common_ordered()`] uses the natural ordering of keys which are [`Ord`].

[`most_common_ordered()`]: Counter::most_common_ordered
[`Ord`]: https://doc.rust-lang.org/stable/std/cmp/trait.Ord.html

```rust
let by_common = "eaddbbccc".chars().collect::<Counter<_>>().most_common_ordered();
let expected = vec![('c', 3), ('b', 2), ('d', 2), ('a', 1), ('e', 1)];
assert!(by_common == expected);
```

[`k_most_common_ordered()`] takes an argument `k` of type `usize` and returns the top `k` most
common items.  This is functionally equivalent to calling `most_common_ordered()` and then
truncating the result to length `k`.  However, if `k` is smaller than the length of the counter
then `k_most_common_ordered()` can be more efficient, often much more so.

```rust
let by_common = "eaddbbccc".chars().collect::<Counter<_>>().k_most_common_ordered(2);
let expected = vec![('c', 3), ('b', 2)];
assert!(by_common == expected);
```

[`k_most_common_ordered()`]: Counter::k_most_common_ordered
[`most_common_ordered()`]: Counter::most_common_ordered

### Get the most common items using your own ordering

For example, here we break ties reverse alphabetically.

```rust
let counter = "eaddbbccc".chars().collect::<Counter<_>>();
let by_common = counter.most_common_tiebreaker(|&a, &b| b.cmp(&a));
let expected = vec![('c', 3), ('d', 2), ('b', 2), ('e', 1), ('a', 1)];
assert!(by_common == expected);
```

### Treat it like a `HashMap`

`Counter<T, N>` implements [`Deref`]`<Target=HashMap<T, N>>` and
[`DerefMut`]`<Target=HashMap<T, N>>`, which means that you can perform any operations
on it which are valid for a [`HashMap`].

[`HashMap`]: https://doc.rust-lang.org/std/collections/struct.HashMap.html
[`Deref`]: https://doc.rust-lang.org/stable/std/ops/trait.Deref.html
[`DerefMut`]: https://doc.rust-lang.org/stable/std/ops/trait.DerefMut.html

```rust
let mut counter = "aa-bb-cc".chars().collect::<Counter<_>>();
counter.remove(&'-');
assert!(counter == "aabbcc".chars().collect::<Counter<_>>());
```

Note that `Counter<T, N>` itself implements [`Index`]. `Counter::index` returns a reference to
a [`Zero::zero`] value for missing keys.

[`Index`]: https://doc.rust-lang.org/stable/std/ops/trait.Index.html
[`Zero::zero`]: https://docs.rs/num-traits/latest/num_traits/identities/trait.Zero.html#tymethod.zero

```rust
let counter = "aaa".chars().collect::<Counter<_>>();
assert_eq!(counter[&'b'], 0);
// panics
// assert_eq!((*counter)[&'b'], 0);
```

## Advanced Usage

### Count any iterable which is `Hash + Eq`

You can't use the `most_common*` functions unless `T` is also [`Clone`], but simple counting
works fine on a minimal data type.

[`Clone`]: https://doc.rust-lang.org/stable/std/clone/trait.Clone.html

```rust
#[derive(Debug, Hash, PartialEq, Eq)]
struct Inty {
    i: usize,
}

impl Inty {
    pub fn new(i: usize) -> Inty {
        Inty { i: i }
    }
}

// <https://en.wikipedia.org/wiki/867-5309/Jenny>
let intys = vec![
    Inty::new(8),
    Inty::new(0),
    Inty::new(0),
    Inty::new(8),
    Inty::new(6),
    Inty::new(7),
    Inty::new(5),
    Inty::new(3),
    Inty::new(0),
    Inty::new(9),
];

let inty_counts = intys.iter().collect::<Counter<_>>();
println!("{:?}", inty_counts);
// {Inty { i: 8 }: 2, Inty { i: 0 }: 3, Inty { i: 9 }: 1, Inty { i: 3 }: 1,
//  Inty { i: 7 }: 1, Inty { i: 6 }: 1, Inty { i: 5 }: 1}
assert!(inty_counts.get(&Inty { i: 8 }) == Some(&2));
assert!(inty_counts.get(&Inty { i: 0 }) == Some(&3));
assert!(inty_counts.get(&Inty { i: 6 }) == Some(&1));
```

### Use your own type for the count

Sometimes [`usize`] just isn't enough. If you find yourself overflowing your
machine's native size, you can use your own type. Here, we use an [`i8`], but
you can use most numeric types, including bignums, as necessary.

[`usize`]: https://doc.rust-lang.org/stable/std/primitive.usize.html
[`i8`]: https://doc.rust-lang.org/stable/std/primitive.i8.html

```rust
let counter: Counter<_, i8> = "abbccc".chars().collect();
let expected: HashMap<char, i8> = [('a', 1), ('b', 2), ('c', 3)].iter().cloned().collect();
assert!(counter.into_map() == expected);
```

License: MIT
