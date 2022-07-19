---
toc: false
layout: post
description: Ways to write faster operations in pandas
categories: [python, pandas]
image: images/pandas.jpg
title: No more looping for me! Vectorizing in Pandas and Other Fun Tricks
---

Pandas is awesome. I use it nearly everyday for my work and side projects because it makes working with data so easy.

Often I work with small data sets where the rows are < 10k. I get away with using the `.apply` function often to create new features. I’ve read the great article [here](https://engineering.upside.com/a-beginners-guide-to-optimizing-pandas-code-for-speed-c09ef2c6a4d6) about how to move from slow looping to numpy array vectorization, and it sounded exciting. I had moved away from `.iterrows()` a long time ago but I kept using `.apply` thinking it was about as good as it gets. Until reading that article I didn’t really understand that the `.apply` was still looping, applying the function to each row one at a time. Reading it prompted me to start looking for ways I could speed up my work. I quickly ran into to some bumps while working on recreating a dataset in pandas from Excel, and I’ll share what I learned here in hopes of helping someone else with the same problem.

## First issue - how do I vectorize my conditional logic transformations?
I figured trying out some vectorization on a little if/else function would be a good starting place. Here’s the Excel function copied from the first row of data. I needed to replicate this in pandas:

```
=IF(J2="- None -", I2, J2)
```

It’s pretty basic: If the value in column ‘J’ is “- None -” then use the value in column ‘I’, otherwise use what’s in ‘J’.

Here’s the same logic in a quick ‘n dirty function:

```python
def will_fail(col1, col2):
    if col1 == '- None -':
        return col2
    else:
        return col1
```
Here’s how I tried it:

```python
df['test_col2'] = will_fail(df['Current Status'], df['Status at Time of Lead'])
```

Which lovingly returned:

> ValueError: The truth value of a Series is ambiguous. Use a.empty, a.bool(), a.item(), a.any() or a.all().

I felt embarrassed that I couldn’t get a basic function like this to work with the powers of vectorization. So I rewrote it so I could use it in an `.apply` to just to see how long it would take.

```python
def works_but_slow(row):
    if row['Current Status'] == '- None -':
        return row['Status at Time of Lead']
    else:
        return row['Current Status']


%%time
df['test_col'] = df.apply(works_but_slow, axis=1)
```
> CPU times: user 7.93 s, sys: 351 ms, total: 8.29 s
> Wall time: 8.32 s

Applying this basic logic row by row using the `.apply` took more than eight seconds!!! Wow, now I’m really starting to buy into vectorizing my functions, but how?

It wasn’t until I came across [this brave stack overflow post](https://stackoverflow.com/questions/28896769/vectorize-conditional-assignment-in-pandas-dataframe) that I realized I could use `numpy.where` to handle my conditional logic in a fast way. The syntax even read like an Excel IF statement, so it made it all the easier to transfer across.

Here’s a link to the documentation: https://docs.scipy.org/doc/numpy/reference/generated/numpy.where.html

Here’s how I implemented it with `numpy.where` and in a function. Notice the time difference!

```python
%%timeit
## Normalized Status - Vectorized baby!!

df['normalized_status'] = np.where(df['Status at Time of Lead'] == '- None -', 
                                          df['Current Status'], df['Status at Time of Lead'])
```
> 22.5 ms ± 138 µs per loop (mean ± std. dev. of 7 runs, 10 loops each)

This code does the same thing as my previous `.apply` approach but does it over 300x faster! I then started to use this approach with other columns I needed to create. In some cases, to make it look cooler (IMO) I wrapped it in a function and sat back and gloried in my faster function (what am I going to do with all that time saved?) ;)

