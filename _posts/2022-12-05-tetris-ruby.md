---
layout: 		      post
title: 			      "Tetris Game in Ruby"
category:		      "Programming Languages"
tags:			        object-oriented-programming game
permalink:		    /tetris-ruby/
last_modified_at: "2022-12-05"
---

Homework 6 of the [programming language course](https://www.coursera.org/learn/programming-languages/) taught by Professor Dan Grossman from University of Washington. Please refer to [this link](https://www.coursera.org/learn/programming-languages-part-c/supplement/8lyk9/homework-6-instructions) for instructions and provided code.

This assignment is about a Tetris game written in Ruby. Ruby is a dynamically-typed, pure object-oriented programming language. The Ruby code in `uw6provided.rb` implements a simple but fully functioning Tetris game. We will be editing `uw6assignment.rb` to make some enhancements. The Ruby code in `uw6graphics.rb` provides a simple graphics library, tailored to Tetris. Run and play the original game with:
```ruby
ruby uw6runner.rb original
```
and the enhanced game with
```ruby
ruby uw6runner.rb enhanced
```

<!-- excerpt-end -->

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## Mac Environment Setup

Intall Homebrew:

```bash
$ /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
```

Install Ruby:

```bash
$ brew install ruby
```

Add Ruby to `.bash_profile`:

```bash
$ export PATH="/usr/local/opt/ruby/bin:$PATH"
```

Don't forget to:

```bash
$ source ~/.bash_profile
```

Install required libraries:

```bash
$ sudo gem install os
$ sudo gem install chunky_png
$ brew install glfw
$ sudo gem install opengl-bindings
$ sudo gem install opener
```

We are ready to go!

## Enhancements

### Make the falling piece rotate 180 degrees

In the enhanced version, the players can press the `<u>` key to make the falling piece rotate 180 degrees.