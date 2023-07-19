---
layout: 		      post
title: 			      "Tetris Game in Ruby"
category:		      "Programming Languages"
tags:			        object-oriented-programming game cse425-assignment
permalink:		    /blog/tetris-ruby
last_modified_at: "2022-12-08"
---

Homework 6 of the [programming language course](https://www.coursera.org/learn/programming-languages/) taught by Professor Dan Grossman from University of Washington. Please refer to [this link](https://www.coursera.org/learn/programming-languages-part-c/supplement/8lyk9/homework-6-instructions) for instructions and provided code.

<!-- excerpt-end -->

This assignment is about a Tetris game written in Ruby. Ruby is a dynamically-typed, pure **object-oriented**{: style="color: red"}[^1] programming language. The Ruby code in `uw6provided.rb` implements a simple but fully functioning Tetris game. We will be editing `uw6assignment.rb` to make some enhancements. The Ruby code in `uw6graphics.rb` provides a simple graphics library, tailored to Tetris. Run and play the original game with:
```bash
$ ruby uw6runner.rb original
```
and the enhanced game with
```bash
$ ruby uw6runner.rb enhanced
```

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

We are ready to go! Please note that do not use the OpenGL library directly in any way in this assignment.

## Enhancements

### Make the falling piece rotate 180 degrees

In the enhanced version, players should be allowed to make the falling piece rotate $$180$$ degrees by pressing the `<u>` key. This can be done by adding a `key_bindings` method to the subclass `MyTetris`:

```ruby
def key_bindings
  super
  @root.bind('u', proc { @board.rotate_pi_move })
end
```

The `rotate_pi_move` method is implemented in the subclass `MyBoard`:

```ruby
def rotate_pi_move
  if !game_over? and @game.is_running?
    @current_block.move(0, 0, -2)
  end
  draw
end
```

where we use the `move` method of `Piece` class to control the rotation:

```ruby
# Takes the intended movement in x, y, and rotation and checks to see if the
# movement is possible. If it is possible, makes this movement and returns true.
# Otherwise returns false.
def move (delta_x, delta_y, delta_rotation)
  # Ensures that the rotation will always be a possible formation (as 
  # opposed to nil) by altering the intended rotation so that it stays 
  # within the bounds of the rotation array.
  moved = true
  potential = @all_rotations[(@rotation_index + delta_rotation) % @all_rotations.size]
  # For each individual block in the piece, checks if the intended move
  # will put this block in an occupied space.
  potential.each{|posns| 
    if !(@board.empty_at([posns[0] + delta_x + @base_position[0],
                          posns[1] + delta_y + @base_position[1]]));
      moved = false;  
    end
  }
  if moved
    @base_position[0] += delta_x
    @base_position[1] += delta_y
    @rotation_index = (@rotation_index + delta_rotation) % @all_rotations.size
  end
  moved
end
```

For comparison, a `rotate_clockwise` method looks like this:

```ruby
def rotate_clockwise
  if !game_over? and @game.is_running?
    @current_block.move(0, 0, 1)
  end
  draw
end
```

### Three Additional Pieces

The `rotations` method of `Piece` class figures out all possible rotations of a given piece and returns an array of point arrays (three levels of nested square brackets):

```ruby
def self.rotations (point_array)
  rotate1 = point_array.map {|x,y| [-y,x]}  
  rotate2 = point_array.map {|x,y| [-x,-y]} 
  rotate3 = point_array.map {|x,y| [y,-x]}  
  [point_array, rotate1, rotate2, rotate3]  
  end
```

In addition to the array `All_pieces` holding seven classic pieces and their rotations, these three pieces and their rotations should also be included:

```text
+---+---+
|   |   |
+---+---+---+
|   |   |   |
+---+---+---+

+---+
|   |
+---+---+
|   |   |
+---+---+

+---+---+---+---+---+
|   |   |   |   |   |
+---+---+---+---+---+
```

```ruby
All_My_Pieces = All_Pieces + [rotations([[0,0], [1,0], [-1,0], [0,-1], [-1,-1]]),
                              rotations([[0,0], [1,0], [0,1]]),
                              [[[0,-2], [0,-1], [0,0], [0,1], [0,2]],
                               [[-2,0], [-1,0], [0,0], [1,0], [2,0]]]]
```

Now, we have ten pieces. The initial rotation for each piece is chosen randomly (and uniformly).

### Allow players to cheat

Players should be able to cheat (obtain a cheat piece as follows in the next round) by pressing the `<c>` key:

```text
+---+
|   |
+---+
```

```ruby
# Inside class MyPiece:
Cheat_Piece = [[[0,0]]]

def initialize (point_array, board)
  super
end

## Called by MyBoard's next_piece:
def self.next_piece (board)
  MyPiece.new(All_My_Pieces.sample, board)
end
def self.next_piece_for_cheat (board)
  MyPiece.new(Cheat_Piece, board)
end
```

If the cheating player's score is less than $$100$$, nothing happens; otherwise, cheating costs $$100$$ points. Hitting `<c>` multiple times while a single piece is falling should behave no differently than hitting it once.

We shall add this line of code to our own `key_bindings` method:

```ruby
@root.bind('c', proc { if @board.score >= 100 && !@board.will_cheat
                          @board.will_cheat = true end })
```

The `will_cheat` attribute of `MyBoard` subclass is initialized to `false`. The original `next_piece` method of `Board` class is modified to take into account of the cheating behavior:

```ruby
def next_piece
  if !self.will_cheat
    @current_block = MyPiece.next_piece(self)
  else
    self.will_cheat = false
    @score -= 100
    @current_block = MyPiece.next_piece_for_cheat(self)
  end
  @current_pos = nil
end
```

## Notes

[^1]: Here I would like to cite Robin Milner's thesis that traces the history of object-oriented programming: "In the 1960s there was a great vogue in simulation languages. New ones kept emerging. They all gave you ways of making queues of things (in the process which you wished to simulate), giving objects attributes which would determine how long it took to process them, giving agents attributes to determine what things they could process, tossing coins to make it random, and recording what happened in a histogram... One of them highlighted a new metaphor: the notion of a community of agents all *doing* things to each other, each persisting in time but changing state. This is the notion known to programmers as an *object*, processing its own state and its repertoire of activities, or so-called *methods*; it is now so famous that even non-programmers have heard of it. It originated in the simulation language known as Simula, invented by Ole-Johann Dahl and Kristen Nygaard. *Object-oriented programming* is now a widely accepted metaphor used in applications which have nothing to do with simulation. So the abstract notion of agent or active object, from being a convenient metaphor, is graduating to the status of a concept in computer science." In Dina Goldin, Scott A. Smolka, and Peter Wegner (eds.), *Interactive Computation*, Springer-Verlag Berlin, Heidelberg, 2006, page 3-4.