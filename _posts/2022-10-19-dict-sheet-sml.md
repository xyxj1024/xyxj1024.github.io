---
layout: 		      post
title: 			      "Dictionaries and Spreadsheets in Standard ML"
category:		      "Data Structures and Algorithms"
tags:			        functional-programming linked-list cse425-assignment
permalink:		    /blog/dict-sheet-sml
---

Standard ML exercises of [CSE 425S: "Programming Systems and Languages" at Washington University in St. Louis](https://classes.engineering.wustl.edu/cse425s).

<!-- excerpt-end -->

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

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

Below are all the utility methods:
```sml
fun row_count(s : sheet) : int =
  foldl (fn (x,acc) => acc + 1) 0 s

fun column_count(s : sheet) : int =
  case s of 
    [] => 0
  | head::_ => foldl (fn (x,acc) => acc + 1) 0 head

fun row_at(s : sheet, row_index : int) : cell list =
  List.nth(s, row_index)

fun cell_in_row_at_column_index(r : cell list, col_index : int) : cell = 
  List.nth(r, col_index)

fun cell_at(s : sheet, row_index : int, col_index : int) : cell = 
  cell_in_row_at_column_index(row_at(s, row_index), col_index)

fun column_at(s : sheet, col_index : int) : cell list =
  case s of
    [] => []
  | head::rest => List.nth(head, col_index)::column_at(rest, col_index)

fun sum_in_cell_list(cells : cell list) : int =
  case cells of
    [] => 0
  | head::rest => 
      case head of
        INTEGER(x) => x + sum_in_cell_list(rest)
      | _ => sum_in_cell_list(rest)

fun sum_in_row(s : sheet, row_index : int) : int =
  sum_in_cell_list(row_at(s, row_index))

fun sum_in_column(s : sheet, column_index : int) : int =
  sum_in_cell_list(column_at(s, column_index))

fun max_in_cell_list(cells : cell list) : int option =
  let
    fun int_list(cells: cell list) : int list =
      case cells of
        [] => []
      | head::rest => 
          case head of
            INTEGER(x) => x::int_list(rest)
          | _ => int_list(rest)
  in 
    case int_list(cells) of 
      [] => NONE
    | _ => SOME (foldl Int.max 0 (int_list(cells)))
  end

fun max_in_row(s : sheet, row_index : int) : int option =
  max_in_cell_list(row_at(s, row_index))

fun max_in_column(s : sheet, column_index : int) : int option =
  max_in_cell_list(column_at(s, column_index))

fun count_if_in_cell_list(cells : cell list, predicate : (cell -> bool)) : int = 
  foldl (fn (c, acc) => if predicate(c) then acc + 1 else acc) 0 cells

fun count_if_in_row(s : sheet, row_index : int, predicate : (cell -> bool)) : int = 
  count_if_in_cell_list(row_at(s, row_index), predicate)

fun count_if_in_column(s : sheet, col_index : int, predicate : (cell -> bool)) : int = 
  count_if_in_cell_list(column_at(s, col_index), predicate)
```

## Spreadsheet to Dictionaries

Given this spreadsheet:

|---
| **Name** | **Uniform Number** | **Birth Year** | **Games Played** | **Goals** | **Assists**
|:-|:-:|:-:|:-:|:-:|:-:|
| Bobby Orr | 4 | 1948 | 657 | 270 | 645
| Wayne Gretzky | 99 | 1961 | 1487 | 894 | 1963
| Mario Lemieux | 66 | 1965 | 915 | 690 | 1033

The function <code>to_dictionaires_using_headers_as_keys</code> should return a list with 3 single list dictionaries, one for each non-header row. The dictionaries would be filled with entries for each column, using the cell in the header of the column as the key and the cell in the particular row of the column as the value.

```text
[ { TEXT("Name") => TEXT("Bobby Orr"), TEXT("Uniform Number") => INTEGER(4), TEXT("Birth Year") => INTEGER(1948), TEXT("Games Played") => INTEGER(657), TEXT("Goals") => INTEGER(270), TEXT("Assists") => INTEGER(645) }, 
  { TEXT("Name") => TEXT("Wayne Gretzky"), TEXT("Uniform Number") => INTEGER(99), TEXT("Birth Year") => INTEGER(1961), INTEGER("Games Played") => INTEGER(1487), TEXT("Goals") => INTEGER(894), TEXT("Assists") => INTEGER(1963) }, 
  { TEXT("Name") => TEXT("Mario Lemieux"), TEXT("Uniform Number") => INTEGER(66), TEXT("Birth Year") => INTEGER(1965), INTEGER("Games Played") => INTEGER(915),  TEXT("Goals") => INTEGER(690), TEXT("Assists") => INTEGER(1033) } ]
```

```sml
structure SpreadsheetToDictionaries = struct
  open Spreadsheet

  fun spreadsheet_to_dictionaries_using_headers_as_keys(s : sheet) : (cell, cell) SingleChainedDictionary.dictionary list =

    let
      (* returns a cell list, which is the first row of s *)
      val keys = Spreadsheet.row_at(s, 0)

      (* returns a (key, value) pair list list *)
      fun spreadsheet_to_kvs_list(s) =
        let
          fun values l =
            case l of
              [] => []
            | _::rest => rest
        in
          List.map (fn clist => ListPair.zip(keys, clist)) (values s)
        end
      
      (* returns a dictionary given a list of pairs *)
      fun pairs_to_dictionary(ps, dict) =
        case ps of
          [] => dict
        | (k, v)::t =>
            let
              val (new_dict, _) = SingleChainedDictionary.put(dict, k, v)
            in
              pairs_to_dictionary(t, new_dict)
            end
      
      (* returns a dictionary list given a pair list list *)
      fun spreadsheet_to_dict_helper(pslist, dict) =
        List.map (fn ps => pairs_to_dictionary(ps, dict)) pslist
    in
      spreadsheet_to_dict_helper(spreadsheet_to_kvs_list(s), SingleChainedDictionary.create())
    end

end
```