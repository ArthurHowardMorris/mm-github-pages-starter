---
title: '2024-07-13'
categories:
  - coding
tags: 
  - estout
  - stata
published: true
---
Easy subsample comparisons with esttab.

It's often the case that we want to display summary statistics for two samples
along with a t-test comparing the two samples. The `ttest` command with
`esttab` allows this, but limits the statistics that you can display.

Here is a minimal example of how to do this while retaining all of the statistics generated by summarize:

## First load some example data:

```Stata 
// load example data
sysuse auto, clear
drop make // this is not numeric, dropping simplifies the example
```

I'm dropping `make` so that I can use `*` to reference all the variables in the dataset.

## Next calculate standard summary stats for both samples:

We'll use `estpost su` with `d` for `detail` and `q` for `quantile` and store each set as a named estimate.

```Stata 
// sum saving quantiles
estimates clear
estpost su * if foreign == 1 , d q
  estimates store FOR
estpost su * if foreign == 0 , d q
  estimates store DOM 
```

## Third, we'll run the T-test:

We pass `foreign` to the `by()` option so that `ttest` computes the sample
split that we did above. The `welch` option relaxes the default assumption that
the two samples have the same variance, this is almost never what we want when
using observational data.

```Stata 
// run the ttest (note welch relaxes the default assumption of equal variances)
estpost ttest * , by(foreign) welch
  estimates store ttests 
```

## Finally, tabulate!

Here things get a little exciting!

`esttab` has several features where you use a series of 1s and 0s to tell it
where to put things. I'll just note that even thought these look similar, they
are not always doing the same thing. So pay attention and read the excellent
docs. Just a note, not a complaint. 

Here is a minimal example of using `esttab` to tabulate counts, means, medians,
and the results of a t-test comparing the two.

```Stata 
// switch delimiter so that the command is easier to read
#delimit ;
esttab FOR DOM ttests ,
	// the cells option allows us to tell esttab what stats we want from each estimate
 	cells((
		// 'pattern' turns the cells on an off for each estimate (i.e. FOR, DOM, and ttest)
		count(pattern(1 1 0 ) fmt(%12.0fc)) 
		mean(pattern(1 1 0 ) fmt(%12.2fc)) 
		sd(pattern(1 1 0 ) fmt(%12.3fc)) 
		p50(pattern(1 1 0 ) fmt(%12.2fc))
		b(star pattern(0 0 1) fmt(%12.2fc))
	)) 
;
#delimit cr
```

Okay, lets take this line by line.

- `#delimit ;` lets us break this long command across several lines so that
it's easier to read.
- `esttab TAB FOR ttests ,` these are the three sets of estimates that we are
passing to `esttab` 
- `cells()` is the option that allows us to specify which statistics to report.
Inside this we are going to pass a list of statistics to report.
- `count()`, `mean()`, `sd()`, `p50()` are all stats generated by `estpost su,
d q` which we have in `FOR` and `DOM`. `ttests` doesn't have any of these, but
does have `b()` so we'll pass all five of them along with arguments that tell
`esttab` which estimates have which statistics.
- the first four stats are in the first two estimates so for each of them we
use `pattern(1 1 0)` to indicate that they should be displayed for `FOR` and
`DOM` but not for `ttests`.
- the final statistic `b()` is only estimated in `ttests` and we indicate this
with `pattern(0 0 1)` we also include `star` so that `esttab` will report the
difference in means (`b()`) along with stars to indicate statistical
significance.
- In all of these cases we are also specifying the format of the numbers with
the `fmt()` option. Display formats in Stata are their own special delight...
but well beyond the scope of this post.