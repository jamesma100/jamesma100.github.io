---
layout: post
title: "Strategies for autocomplete"
---

Autocomplete works by automatically suggesting words as you type.
For instance, you may type the word "app" and get a list of suggestions, such as "app store", "apple", and "application", sorted by search frequency.
This means that upon entering a prefix, any algorithm we choose would need to  efficiently fetch the top results and return it to the user.
And upon searching a full word, it would need to update that word's frequency, or add it to our data structure if it doesn't exist.

Note that it is more important to optimize the former than the latter, as we have a latency requirement for autocomplete, but not for maintaining our data structure - the latter can be done asynchronously after the user is done with their search, while the former, displaying a list of suggestions, has to be almost instantaneous!


#### First pass
The first solution is to maintain a map of all searched words and their frequencies.
When a word `w0` is searched, loop through all the words, collect those with prefix `w0` into a list, sort the list by frequency, and return it.
Then, increment `w0`'s frequency by 1.
If `N` is the total number of words ever searched, `L` is the average length of a word, and `X` is a user-specified constant representing the number of results to return, this would take `O(N)` for looping through all the words, `O(L)` for checking for prefix match, and `O(XlogX)` for sorting the resultant list.
Ignoring hash collisions, incrementing the frequency is constant.
This would give us `O(N*L + XlogX)` per letter entered!

#### Prefix trees
Alternatively, we could represent our search space with a prefix tree.
A prefix tree consists of a node per letter in a word, and a full word is formed by following the appropriate nodes from the root.

For example, a prefix tree consisting of "beetle" would look like:

`root -> b -> be -> bee -> beet -> beetl -> beetle`

So upon entering some prefix, an autocomplete operation consists of two parts: 1. finding the prefix node, then 2. traversing the rest of the subtree to find all words that share the common prefix.
Our total runtime is now `O(L + F^L)`, `F` being the average branching factor of a node, and `L` is once again the average length of a word.
Since `F` is at most 26, the number of letters in the English alphabet, we now have `O(L + 26^L)` - a vast improvement!

Notice that the more characters we type in, the faster the autocomplete returns, as we spend more time on finding the root node of the subtree (part 1), and less on traversing the subtree (part 2), the latter of which dominates the total time.
This is because the longer our prefix, the smaller the subtree of matching characters becomes, and thus the less time it takes to traverse it.

An implementation could look something like below.
Note that some functions, such as for adding and deleting nodes, are commented out as they're not relevant to returning the results but only for constructing the tree, but you can find the full code [here](https://github.com/jamesma100/triehugger/blob/main/triehugger.py).
Once again, here we only care about the time it takes for the autocomplete to return our results, ignoring the cost of insertion, deletion, pruning the tree, etc. as the user is not blocked on those operations.

```
class AutocompleteWithTree(object):
    def __init__(self, val="", parent=None, store=True):
        self.val = val
        self.children = {} # maps string value to node object
        self.parent = parent
        self.store = store

    def add(self, val):
        pass

    def delete(self, val):
        pass

    def find_root(self, word, idx=0):
        prefix = word[:idx + 1]
        for child in self.children.values():
            if child.val == prefix:
                if prefix == word:
                    return child
                return child.find_root(word, idx + 1)
        return None

    def complete_helper(self, word, suggestions=[]):
        if not self.children:
            return [self.val] if self.store else []
        child_suggestions = []
        for child in self.children.values():
            child_suggestions.extend(child.complete_helper(word, []))
        if self.store:
            suggestions.append(self.val)
        suggestions.extend(child_suggestions)
        return suggestions

    def complete(self, word):
        root = self.find_root(word, 0)
        if not root:
            return []
        return root.complete_helper(word, [])
```

Ideally for each prefix, we keep all words beginning with the prefix sorted by frequency in the tree such that we can simply return the first `X` elements we traverse, instead of searching through the whole subtree.
This would make our runtime just `O(L + X)`.
Lucene actually does this with its `FSTCompletionBuilder` class by sorting edge weights and traversing the highest-weight nodes first, allowing it to terminate as soon as the number of results requested is reached.
This is a bit trickier to implement, but you can read more about the algorithm [here](https://lucene.apache.org/core/7_1_0/suggest/org/apache/lucene/search/suggest/fst/FSTCompletionBuilder.html).

#### Sorted sets
The other solution is based on an excellent [blog post](http://oldblog.antirez.com/post/autocomplete-with-redis.html) that descibes using sorted sets to store words based on frequency, by the creator of Redis.
The idea is that we have a sorted set that sorts words based on their score, and if two words have identical scores, they are ordered lexicographically.
So autocompleting based on some prefix is a matter of 1. finding the position of that prefix in the set and 2. returning every word after it, as they will necessarily contain the given prefix.

```
class AutocompleteRedis:
    def __init__(self, source, name=":compl", port=6379):
        self.r = redis.Redis(host="localhost", port=port, decode_responses=True)
        self.name = name
        self.source = source
    def create_db(self):
        pass
    def complete(self, s, limit=10):
        pos = self.r.zrank(":compl", s)
        if not pos:
            pos = self.r.zrank(":compl", s + "*") # * denotes stored word
        if pos:
            count = 0  # tracks number of results to return
            matched = False
            results = self.r.zrange(":compl", pos, -1)
            words = []
            for word in results:
                if not word.startswith(s):
                    # no longer have common prefix, so end loop
                    if matched:
                        break
                    continue
                if word.endswith("*"):
                    words.append(name)
                    matched = True
                    count += 1
                    if count == limit:
                        break
        return words
```
To order based on frequency, the author proposes using a sorted set for every prefix, and using it to store every searched word beginning with that prefix.
When a full word is searched, we look for that word in the every sorted set representing each prefix, and update its score, thus promoting it towards the front of the set.
Our autocomplete operation now consists of searching through every sorted prefix set (there are `L` total), finding the word in it (`logN`) and returning the rest of the set, which adds up to `O(L*logN)` - bad!

Since we are now storing words redundantly (a word of size `S` is stored `S` times, once in each prefix set), the author suggests limiting the size of each set, e.g. 300, and evicting the item with the lowest score if the set gets too large.
This actually makes our runtime just `O(L)`, though we are kind of cheating since we are regularly pruning our data.
You can imagine that if a few searches make up majority of total searches (a realistic scenario), new search terms never have a change to accumulate in our set because they would always be the first to get evicted.

Also note that though we focused on big O and are thus dropping constants and coefficients, it can be a totally impractical metric.
A runtime of `O(300,000,000,000L)` is linear time with respect to the average word length `L`, but certainly not fast!
