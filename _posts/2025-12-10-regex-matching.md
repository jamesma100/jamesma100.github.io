---
layout: post
title: "Regular expression matching"
---

This is a writeup of [Leetcode 10. Regular Expression Matching](https://leetcode.com/problems/regular-expression-matching/description/).

My motivation for writing this is twofold.
The first is that most of the submitted solutions on Leetcode try to be too clever and succinct without giving many examples.
Second, this problem is actually quite relevant to my day-to-day work, as I use regular expressions quite often, so I thought it might be worthwhile to slightly understand how they work under the hood.

The problem is as follows: Given an input string `s` and pattern `p`, return true if they match.
`p` is a regex pattern and can include the following:
- `.` matches any single character
- `*` matches zero or more of the preceding element

For example:
`"ab.*"` matches `"abx"`

This is a classic dynamic programming problem where each subresult depends on the previous subresult.
The general idea is that we create a 2d matrix `dp` where `dp[i][j]` represents whether or not substring `s[:i]` (end index is exclusive) matches subpattern `p[:j]`.
Since we have a base case of `("", "")`, our matrix has dimensions `(s.size() + 1, p.size() + 1)`. We then fill in the matrix iteratively, after which the final result is `dp[s.size()][p.size()]`.

## Base case
To start, notice the trivial case where the empty string matches the empty pattern.
```
   "" a *
"" t
a
b
```

To fill in the first row, we need to check whether each subpattern match the empty string. This is true only when the current character is `"*"` and if the previous subpattern also matches.
For example, below we encounter a `"*"`, but the subpattern `"a"` does not match.
```
   "" a *
"" t  f f 
a
b
```

Conversely, in this example, all the subpatterns match.
```
   "" * *
"" t  t t 
a
b
```

To fill in the first column, we can check if each prefix of `s` matches the empty pattern.
This is true only with the empty string, so the rest of the first column should all be false.
```
   "" a *
"" t  f f 
a  f
b  f
```

## Recursive case
At each iteration `dp[i][j]`, we look at string `s[i - 1]` and pattern `p[j - 1]`.
(The `-1` is because the `dp` matrix is one size larger in both dimensions due to the base case `("", "")`).
There are two possibilities.

The first possibility is that we encounter an exact match or a `"."`.
This case should be true if and only if the previous substring also matches the previous subpattern, i.e. `dp[i - 1][j - 1]`.

For example, here we see pattern `"."` matches string `"c"`.
```
p: a b .
       |
       j

s: a b c
       |
       i
```
So we look one step back - pattern `"ab"` also matches string `"ab"`, so `dp[i][j]` is true.
```
p: a b .
     |
     j

s: a b c
     |
     i
```

Here's a counterexample - we get a match, but `dp[i-1][j-1]` does not hold, so `dp[i][j]` is false.
```
p: a b .
       |
       j

s: a x c
       |
       i
```

In code,
```c++
if (s[i - 1] == p[j - 1] || p[j - 1] == '.') {
    dp[i][j] = dp[i - 1][j - 1]
}
```


The second possibility is that we encounter `"*"`.
Since the wildcard `"*"`matches zero or more characters, we can handle three cases separately: no match, exactly one match, or 2+ matches.

First, `"*"` can act as an empty set, i.e. we can choose to _not_ match anything.

Consider the following example:
```
p: a b x *
         |
         j

s: a b
     |
     i
```
In this case we simply move pointer `j` back by 2 to skip over the `"x*"`, i.e. `dp[i][j - 2]`.

```
p: a b x *
     |
     j

s: a b
     |
     i
```
In code,
```c++
if (dp[i][j - 2]) dp[i][j] = true;
```

Next, to match any number of characters, we first have to check if the wildcard actually matches.
This can be done with:

```c++
if (s[i - 1] == p[j - 2] || p[j - 2] == '.') {...}
```

Now we can try to match a single character, e.g. `"x"`.
```
p: a b y x *
           |
           j
s: a b y x
         |
         i
```

To do this, we move `j` back by 2 and `i` back by 1 and look at the cached result there, i.e. whether or not pattern `"aby"` matches string `"aby"` (it does).

```
p: a b y x *
       |
       j

s: a b y x
       |
       i
```
In code:

```c++
if (dp[i - 1][j - 2]) dp[i][j] = true;
```

Lastly, to match 2+ characters, consider the following example:
```
p: a b y x *
           |
           j

s: a b y x x
           |
           i
```
To match the string `"xx"` with the pattern `"x*"`, we keep the same pattern, but check whether the previous substring matches by moving `i` back by 1.

```
p: a b y x *
           |
           j

s: a b y x x
         |
         i
```
We see that pattern `"abyx*"` also matches string `"abyx"`, so `dp[i][j]` is true.
This translates to the following code:

```c++
if (dp[i - 1][j]) dp[i][j] = true;
```

Putting it all together:
```c++
bool isMatch(string s, string p) {
    if (p == "") return (s == "");

    vector<vector<bool>> dp(s.size() + 1, vector<bool>(p.size() + 1));

    // base case: empty string, empty pattern
    dp[0][0] = true;
    // base case: empty pattern, first column
    for (int i = 1; i < s.size() + 1; ++i) dp[i][0] = false;
    // base case: empty string, first row
    for (int j = 2; j < p.size() + 1; ++j) {
        dp[0][j] = p[j - 1] == '*' && dp[0][j - 2];
    }

    for (int i = 1; i < s.size() + 1; ++i) {
        for (int j = 1; j < p.size() + 1; ++j) {
            // case 1: exact match
            if ((s[i - 1] == p[j - 1] || p[j - 1] == '.')) {
                dp[i][j] = dp[i - 1][j - 1];
            // case 2: wildcard (*)
            } else if (p[j - 1] == '*') {
                // case 2a: match zero characters
                if (dp[i][j - 2]) dp[i][j] = true;
                if (s[i - 1] == p[j - 2] || p[j - 2] == '.') {
                    // case 2b: match 1 character
                    if (dp[i - 1][j - 2]) dp[i][j] = true;
                    // case 2c: match 2+ characters
                    if (dp[i - 1][j]) dp[i][j] = true;
                }
            }
        }
    }
    return dp[s.size()][p.size()];
}

```
I realize this implementation is highly redundant and un-optimized, but that is by design.
Anyway, I hope this post didn't leave anyone even more confused :)
