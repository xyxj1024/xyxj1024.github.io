---
layout: 		      post
title: 			      "Dictionaries and Spreadsheets in Standard ML"
category:		      "Programming Languages"
tags:			        data-structures functional-programming
permalink:		    /dict-sheets-sml/
---

Standard ML exercises of [CSE 425S "Programming Languages" at Washington University in St. Louis](https://classes.engineering.wustl.edu/cse425s).

<!-- excerpt-end -->

<br />
## Table of Contents
{:.no_toc}
* TOC 
{:toc}
<br />

## Singly Chained Dictionary

A dictionary data structure is for storing certain information associated with keys. A value is uniquely mapped to a key so that given a key we can find or modify a value. The most basic form of dictionary can be implemented as a singly linked list, assuming that the user does not care about the number of keys need to be managed:

```sml
datatype (''k,'v) Record = Empty | Cons of ''k * 'v * (''k, 'v) Record
type (''k, 'v) dictionary = (''k, 'v) Record

fun create() : (''k, 'v) dictionary = Empty

fun get(dict : (''k, 'v) dictionary, key : ''k) : 'v option =
  case dict of
    Empty => NONE
  | Cons (this_key, this_val, next_rec) =>
      if this_key = key
      then SOME this_val
      else get(next_rec, key)

fun put(dict : (''k, 'v) dictionary, key : ''k , value : 'v) : (''k, 'v) dictionary * 'v option =
  let
    val ret = ref NONE
    fun put_helper(d, k, v) =
      case d of
        Empty => Cons (k, v, d)
      | Cons (this_key, this_val, next_rec) =>
          case k = this_key of
            false => 
              Cons (this_key, this_val, put_helper(next_rec, k, v))
          | true => 
              (ret := SOME this_val; Cons (this_key, v, next_rec))
  in
    (put_helper(dict, key, value), !ret)
  end
	
fun remove(dict : (''k, 'v) dictionary, key : ''k) : (''k, 'v) dictionary * 'v option =
  let
    val ret = ref NONE
    fun remove_helper(d, k) =
      case d of
        Empty => Empty
      | Cons (this_key, this_val, next_rec) =>
          case k = this_key of
            false => Cons (this_key, this_val, remove_helper(next_rec, k))
          | true => (ret := SOME this_val; remove_helper(next_rec, k))
  in
    (remove_helper(dict, key), !ret)
  end

fun entries(dict : (''k, 'v) dictionary) : (''k * 'v) list =
  case dict of
    Empty => []
  | Cons (this_key, this_val, next_rec) =>
      (this_key, this_val)::entries(next_rec)
```

## Hashed Dictionary

A more efficient dictionary might organize data into buckets and determine what data go to which bucket with a hash function:
```sml
type ''k hash_function = ''k -> int
type (''k, 'v) bucket = (''k * 'v) list
type (''k, 'v) dictionary = (((''k, 'v) bucket) Vector.vector ref * ''k hash_function)

(* Author: Professor Dennis Cosgrove *)
fun positive_remainder(v : int, n : int) : int = 
  let
    val result = v mod n 
  in 
    if result >= 0 
    then result 
    else result + n
end
```
Here, our dictionary combines a reference to a vector of buckets with a pre-defined hash function. Given an arbitrary key, we first perform hashing upon it and then take the positive remainder of the hashed key divided by the number of buckets as the index into the bucket vector:
```sml
fun find_bucket(dict : (''k, 'v) dictionary, key : ''k) : (''k, 'v) bucket =
  let
    val (ref buckets, hash) = dict
    val index = positive_remainder(hash(key), Vector.length(buckets))
  in
    Vector.sub(buckets, index)
  end
```

To create an empty dictionary:
```sml
fun create(bucket_count_request : int, hash : ''k hash_function) : (''k, 'v) dictionary =
  (ref (Vector.tabulate(bucket_count_request, fn(_) => nil)), hash)
```

The <code>get()</code>, <code>put()</code>, and <code>remove()</code> functions are similar to the singly chained version.

## Spreadsheet

A spreadsheet is a two-dimensional data table with the first row acting as headers. We call each table entry as a "cell". Our implementation is rather simple:
```sml
datatype cell = EMPTY | TEXT of string | INTEGER of int
type sheet = cell list list
```

With a <code>.csv</code> file at hand, we can first convert it into a list of <code>string</code> lists:
```sml
structure Csv = struct
(* fun is_separator(c : char) : bool = c = #"/" *)
    
  fun is_new_line(c : char) : bool = c = #"\n"

  fun is_comma(c : char) : bool = c = #","
    
  fun read_csv(csv:string) : string list list =
    List.map (String.fields is_comma) (String.fields is_new_line csv)
end
```
Then, we can create a spreadsheet by means of parsing each <code>string</code> form the <code>.csv</code> file into a <code>cell</code>:
```sml
fun create_sheet(word_lists : string list list) : sheet =
  let
    (* Check if a char is actually an int *)
    fun is_integer(c : char) : bool =
      c = #"0" orelse c = #"1" orelse c = #"2" orelse c = #"3" orelse c = #"4" orelse
      c = #"5" orelse c = #"6" orelse c = #"7" orelse c = #"8" orelse c = #"9"
          
    (* Convert each string in the string list to a cell *)
    fun create_line(word : string list) : cell list =
      case word of
        [] => []
      | ""::rest => EMPTY::create_line(rest)
      | head::rest =>
          (* Check each char in the char list *)
          if List.all is_integer (String.explode head)
          (* Convert a string into an int *)
          then (INTEGER (valOf (Int.fromString head)))::create_line(rest)
          else (TEXT head)::create_line(rest)
  in
    (* Create a cell list list by mapping create_line to 
       each string list in word_lists *)
    List.map create_line word_lists
  end
```