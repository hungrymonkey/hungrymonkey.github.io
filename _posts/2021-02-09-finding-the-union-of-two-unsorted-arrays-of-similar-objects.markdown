---
layout: post
title: Finding The Union of Two Unsorted Arrays with Similar Elements by Abusing Python3 Objects v1
date: 2021-2-09 00:00:00 +0300
description: You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. # Add post description (optional)
img: 2021-02-09-python-logo-inkscape.svg # Add image post (optional)
tags: [hackerrank, set, hashmap, python3, object] # add tag
---

# Introduction

Welcome hackerrank coders!

Finding the Union of Two Unsorted Arrays is a coderpad type interview question. The solution uses sets or hashmaps, but reality tends to find new ways to defeat common algorithms by introducing common noise such as complements. This algorithm is useful for validating a new list against an old unsorted list. In this blog post, we overload python3 objects in order to match these complements as interaction passes.

## Problem

Given two unsorted lists of names, write an algorithm to output another list such that the algorithm can also union nicknames.

 
## Implementation

### Data

#### New List

| First Name | Last Name |
|:--|:--|
| Tony | Cohen |
| Miguel | Paublo |
| Jack | Taylor |
| Tony | Schlansky |
| Bob | Doe |
| Katie | Park |

#### Old List

| First Name | Last Name |
|:--|:--|
| Miguel | Paublo |
| Jack | Taylor |
| Antonio | Schlansky |
| Robert | Doe |

#### Translation Table

| First Name | Last Name |
|:--|:--|
|Tony | Antonio |
|Bob | Robert |
|John | Johnathan |

### Satisfying the required methods for `set`

In python3, developers need to overload `__hash__` and `__eq__` in order to satisfy the set container. 

#### `__hash__`

The `hash` function takes one argument and tuples can be passed instead for multiple key elements. 

In an exact match, we can compare all elements together to find a union.

```
def __hash__(self):
		return hash((self.get_first_name(), self.get_last_name()))
```

In a substitution match, we must override has such that the `hash` function hashes the substituted first name instead.

```
def __hash__(self):
		sub = _FIRSTNAME_REGEX_SUB.get(self.first_name, self.first_name) ## get with default key
		return hash((sub, self.get_last_name()))
```

#### `__eq__`


In an exact match, we can compare all elements together to find a union. 

```
def __eq__(self, other):
		if isinstance(other, type(self)):
			return (self.get_first_name() == other.get_first_name()) and (self.get_last_name() == other.get_last_name())
```

In a substitution match, we must override the function in a way that allow us to compare the derived class to the base class. Since we do not need to compare the object to itself, self comparisons can be ignored.

```
if isinstance(other, _base):
	sub = _FIRSTNAME_REGEX_SUB.get(self.first_name, self.first_name) ## get with default key
	return (sub == other.get_first_name()) and (self.get_last_name() == other.get_last_name())
```

#### `__ne__`

The `__ne__` is easy because you can call `__eq__` and reverse the boolean instead.
```
def __ne__(self, other):
		return not self.__eq__(other)
```



### Final Code
```

import re

_FIRSTNAME_REGEX_SUB = dict(
	[
		('Tony', 'Antonio'),
		('Bob', 'Robert'),
		('John', 'Johnathan')
	]
)

class _base:
	def __init__(self, first_name, last_name):
		self.first_name = first_name
		self.last_name = last_name
	def get_first_name(self):
		return self.first_name
	def get_last_name(self):
		return self.last_name
	def __eq__(self, other):
		if isinstance(other, type(self)):
			return (self.get_first_name() == other.get_first_name()) and (self.get_last_name() == other.get_last_name())
	def __ne__(self, other):
		return not self.__eq__(other)
	def __hash__(self):
		return hash((self.get_first_name(), self.get_last_name()))
	def __str__(self):
		return self.first_name + ' ' + self.last_name

class _first_short_hand(_base):
	def __init__(self, o):
		if isinstance(o, _base):
	 		super().__init__(o.get_first_name(), o.get_last_name())
	def __eq__(self, other):
		if isinstance(other, _base):
			return (self.get_long_fname() == other.get_first_name()) and (self.get_last_name() == other.get_last_name())
	def __ne__(self, other):
		return not self.__eq__(other)
	def __hash__(self):
		return hash((self.get_long_fname(), self.get_last_name()))
	def get_long_fname(self):
		sub = _FIRSTNAME_REGEX_SUB.get(self.first_name, self.first_name)
		return sub

if __name__ == "__main__":
	GROUP1 = [
		('Tony', 'Cohen'),
		('Jack', 'Taylor'),
		('Miguel', 'Paublo'),
		('Tony', 'Schlansky'),
		('Bob', 'Doe'),
		('Katie', 'Park')
	]
	GROUP2 = [
		('Miguel', 'Paublo'),
		('Jack', 'Taylor'),
		('Antonio', 'Schlansky'),
		('Robert', 'Doe')
	]
	matched_exact = []
	unmatched_exact = []
	set2 = set(_base(f, l) for (f,l) in GROUP2)
	set1 = set(_base(f, l) for (f,l) in GROUP1)
	## PASS 1
	unmatched_exact = set1 - set2
	matched_exact = set2 & set1
	print("Matched exact: " + str([str(x) for x in matched_exact]))
	print("Unmatch exact: " + str([str(x) for x in unmatched_exact]))

	## PASS 2
	unmatched_short =  set( _first_short_hand(o) for o in unmatched_exact)
	matched_sub = set2 & unmatched_short
	unmatched_sub = unmatched_short - set2 
	print("Matched sub firstname: " + str([str(x) for x in matched_sub]))
	print("Unmatch sub firstname: " + str([str(x) for x in unmatched_sub]))
	pass
```
## Final Output
```
Matched exact: ['Jack Taylor', 'Miguel Paublo']
Unmatch exact: ['Tony Schlansky', 'Katie Park', 'Bob Doe', 'Tony Cohen']
Matched sub firstname: ['Bob Doe', 'Tony Schlansky']
Unmatch sub firstname: ['Katie Park', 'Tony Cohen']
```

## Links

- [Python3 hashable](https://docs.python.org/3/glossary.html#term-hashable)
- [Python3 sets](https://docs.python.org/3/library/stdtypes.html#set)
- [Python3 hash function](https://docs.python.org/3/library/functions.html#hash)
- [Union of two Arrays](https://www.geeksforgeeks.org/find-union-and-intersection-of-two-unsorted-arrays/)

