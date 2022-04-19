---
layout: post
title: Merge Sort Algorithm
categories: [Algorithm, JS]
---

I found out about [Merge Sort Algorithm][Merge Sort Algorithm].

More below the cut.

---

Thanks to book [High Performance JavaScript][Amazon Link] I figured out with interesting **algorithm**.

The code below:

```js
function merge(left, right) {
  let result = [];

  while (left.length > 0 && right.length > 0) {
    left[0] < right[0] ? result.push(left.shift()) : result.push(right.shift());
  }

  return result.concat(left).concat(right);
}

function mergeSort(items) {
  if (items.length === 1) {
    return items;
  }

  let middle = Math.floor(items.length / 2);
  let left = items.slice(0, middle);
  let right = items.slice(middle);

  return merge(mergeSort(left), mergeSort(right));
}
```

It is almost the code from the book, nevertheless I edited a couple lines.

So, yes it is the simple as looks like. However the example above uses **[Recursion][Recursion]**.

In spite of that recursion is a normal thing for developer it has a couple problems:

- Recursion may be too deep and takes a long time for execution.
- Don't forget about [call stack][Call Stack].
- Function call is expensive therefore recursion may be more expensive than simple loop.

Hence the book porposes to us use the algorithm without recursion, like here:

```js
function merge(left, right) {
  let result = [];

  while (left.length > 0 && right.length > 0) {
    left[0] < right[0] ? result.push(left.shift()) : result.push(right.shift());
  }

  return result.concat(left).concat(right);
}

function mergeSort(items) {
  if (items.length === 1) {
    return items;
  }

  let work = items.map(item => [item]);
  work.push([]); // if we have odd count of elements

  for (let limit = items.length; limit > 1; limit = Math.floor((limit + 1) / 2)) {
    let j = 0;
    for (let k = 0; k < limit; ++j, k += 2) {
      work[j] = merge(work[k], work[k + 1]);
    }

    work[j] = []; // if we have odd count of elements
  }

  return work[0];
}
```

Yes, I edited a couple lines here as well.






[Merge Sort Algorithm]: https://en.wikipedia.org/wiki/Merge_sort
[Amazon Link]: https://www.amazon.com/High-Performance-JavaScript-Application-Interfaces/dp/059680279X
[Recursion]: https://en.wikipedia.org/wiki/Recursion_(computer_science)
[Call Stack]: https://en.wikipedia.org/wiki/Call_stack

