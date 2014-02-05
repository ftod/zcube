zcube - Counting Trees for Fun and Profit
=========================================

_zcube_ is about counting trees, and aggregating the counts of the _subtrees_ of these trees. The intent is to provide an analytical tool to compute aggregate sums over multiple hierarchical dimensions.

The bulk of the library is written in Java, around hopefully efficient _immutable_ data structures. A thin Clojure layer provides for the public API of the library.

* A **ZDDNumber** represents a linear combination of sets of trees with integer coefficents.
* A **ZDDTree** is a symbolic expression that represents a sets of trees.
* A **ZDD** is a symbolic decision-diagram based representation of a set of trees.

The _ZDD_ type is not exposed by the public Clojure API.     

The rather simple API provides two styles of computing aggregate counts of subtrees :

* An _accumulative style_ whereby :
  * Given a tree and an integer coefficient, occurrences of its subtrees are accumulated into an immutable _ZDDNumber_.
* A _commutative associative_ style whereby :
  * A tree and an integer coefficient yield a _ZDDNumber_
  * Sequences of _ZDDNumber_ may be added.

See [Add ALL The Things][1] for a good introduction to the power of associativity and commutativity.

# Example 1 : About counting subtrees

As an example, consider the following pair of trees, and their respective decompositions into subtrees :

      a      a       a      a         a
     / \  =     +   /   +    \   +   / \
    b   c          b          c     b   c

      a      a       a      a         a
     / \  =     +   /   +    \   +   / \
    b   d          b          d     b   d

We can _symbolically_ sum the above decompositions as follows :

        a        a       a   a       a                           
       / \  =      +    /  +  \  +  / \                          
      b   c            b       c   b   c                         
            +                                                    
        a        a       a                 a       a
       / \  =      +    /                +  \  +  / \
      b   d            b                     d   b   d
     --------------------------------------------------
               2*a     2*a   a       a     a       a
            =      +    /  +  \  +  / \  +  \  +  / \
                       b       c   b   c     d   b   d

Using the _zcube_ Clojure API, this can be written :

```clojure
(ns net.ftod.zcube-test
  ( :use net.ftod.zcube clojure.test )
)

( deftest test-sum-1
  ( is
    ( let [ zn ( sum-subtrees
               [ [ 1 ( cross ( path "a" "b" ) ( path "a" "c" ) ) ] 
               , [ 1 ( cross ( path "a" "b" ) ( path "a" "d" ) ) ]
               ] ) ]
      ( and
        ( =  2 ( ( count-trees ( path "a" )                                ) zn ) )
        ( =  2 ( ( count-trees ( path "a" "b" )                            ) zn ) )
        ( =  1 ( ( count-trees ( path "a" "c" )                            ) zn ) )
        ( =  1 ( ( count-trees ( cross ( path "a" "b" ) ( path "a" "c" ) ) ) zn ) )
        ( =  1 ( ( count-trees ( path "a" "d" )                            ) zn ) )
        ( =  1 ( ( count-trees ( cross ( path "a" "b" ) ( path "a" "d" ) ) ) zn ) )
      ) ) ) )

```
ie we add the subtrees generated by 1 occurrence of each tree, and count the occurrences of the individual trees in the result.

This generalizes easily to multiple occurrences of trees, using again a _multiplicative_ notation to suggest multiple occurrences of trees, as follows :

      5*a      5*a     5*a   5*a     5*a                           
       / \  =      +    /  +    \  +  / \                          
      b   c            b         c   b   c                         
            +                                                      
      3*a      3*a     3*a                    3*       a
       / \  =      +    /                  +    \  +  / \
      b   d            b                         d   b   d
     ------------------------------------------------------
               8*a     8*a   5*a     5*a     3*a     3*a
            =      +    /  +    \  +  / \  +    \  +  / \
                       b         c   b   c       d   b   d

Nothing really new there :


```clojure
(ns net.ftod.zcube-test
  ( :use net.ftod.zcube clojure.test )
)

( deftest test-sum-3 ; Branching trees example
  ( is
    ( let [ zn ( sum-subtrees
               [ [ 5 ( cross ( path "a" "b" ) ( path "a" "c" ) ) ] 
               , [ 3 ( cross ( path "a" "b" ) ( path "a" "d" ) ) ]
               ] ) ]
      ( and
        ( =  8 ( ( count-trees ( path "a" )                                ) zn ) )
        ( =  8 ( ( count-trees ( path "a" "b" )                            ) zn ) )
        ( =  5 ( ( count-trees ( path "a" "c" )                            ) zn ) )
        ( =  5 ( ( count-trees ( cross ( path "a" "b" ) ( path "a" "c" ) ) ) zn ) )
        ( =  3 ( ( count-trees ( path "a" "d" )                            ) zn ) )
        ( =  3 ( ( count-trees ( cross ( path "a" "b" ) ( path "a" "d" ) ) ) zn ) )
      ) ) ) )
```

# Example 2 : Some Analytics

Now, why in the world would you want to do such a thing, decomposing trees into subtrees, and counting their occurrences ?

Well, suppose you want to perform some _analytics_ on a clickstream, where each event in the stream, besides the url, gives you data about the demographics of the user, and the time of click.

You can model such events as trees, for example, using an _informal_ algebraic notation to denote trees :

      male   user on page1 the 1st of january 2014 at 1OH32 ~ www.company.com/page1+gender/male+2014/01/01/10/32
      female user on page2 the 2nd of january 2014 at 11H15 ~ www.company.com/page2+gender/female+2014/01/02/11/15
      female user on page1 the 3rd of january 2014 at 08H15 ~ www.company.com/page1+gender/female+2014/01/03/08/15

Now, computing the subtrees generated by these, and summing, you get the following terms :

      2*(www.company.com+2014/01) ~ 2 clicks on the domain www.company.com in January 2014
      2*(www.company.com+2014+gender/female) ~ 2 clicks on the domain www.company.com in January 2014 by female users
      
ie computing the subtree decomposition is tantamount to performing multidimensional aggregate sums.

This translates as follows using the zcube API :

```clojure
(ns net.ftod.zcube-test
  ( :use net.ftod.zcube clojure.test )
)

( deftest test-analytics ; Analytics example
  ( is
    ( let [ zn ( sum-subtrees
                 [ [ 1 ( cross
                         ( path "www.company.com" "page1" )
                         ( path "gender" "male" )
                         ( path "2014" "01" "01" "10" "32" ) ) ]
                 , [ 1 ( cross
                         ( path "www.company.com" "page2" )
                         ( path "gender" "female" )
                         ( path "2014" "01" "02" "11" "35" ) ) ]
                 , [ 1 ( cross
                         ( path "www.company.com" "page1" )
                         ( path "gender" "female" )
                         ( path "2014" "01" "03" "08" "15" ) ) ]
                 ]
               ) ]
      ( and
        ( = 3 ( ( count-trees ( path "www.company.com" )                                     ) zn ) )
        ( = 2 ( ( count-trees ( path "www.company.com" "page1" )                             ) zn ) )
        ( = 3 ( ( count-trees ( path "2014" "01" )                                           ) zn ) )
        ( = 2 ( ( count-trees ( path "gender" "female" )                                     ) zn ) )
        ( = 2 ( ( count-trees ( cross ( path "gender" "female" ) ( path "2014" "01" ) )      ) zn ) )
        ( = 1 ( ( count-trees ( cross ( path "gender" "female" ) ( path "2014" "01" "02" ) ) ) zn ) )
      ) ) ) )
```

# The Tree API

The _tree API_ is the part of the API that handles the construction of ...trees.

In fact, the term _tree_ is a bit misleading, as the API rather provides for the construction of sets of trees :

|Name|Description|
|----|-----------|
|top|The _singleton_ set containing only the _empty_ tree.|
|bot|The _empty_ set of trees.|
|path|Build a _singleton_ set containing a _path_, ie a linear tree.|
|prefix|Build a set of trees by prepending a segment to all trees in a set.|
|cross|Build the _cross product_ of set of trees, by taking the union of trees in each sets.|
|sum|Build the _cross product_ of set of trees, by taking the union of trees in each sets.|

In the previous sections, I have glossed over this distinction between trees and sets of trees so as not to confuse the reader. Intuitively, a singleton set of trees can be identified with the only tree it contains.  

The usual pattern for constructing trees is as follows, combining _paths_ with the _cross_ operator :

```clojure
( cross
  ( path "www.company.com" "page1" )
  ( path "gender" "male" )
  ( path "2014" "01" "01" "10" "32" )
)
```

## The Tree Algebra

A few algebraic identities hold :

    path(a,b,c,...) = prefix(a,prefix(b,prefix(c,... top)))
    sum(a,bot) = a
    sum(a,b) = sum(b,a)
    sum(a,sum(b,c)) = sum(sum(a,b),c)
    cross(a,top) = a
    cross(a,b) = cross(b,a)
    cross(a,cross(b,c)) = cross(cross(a,b),c)
    cross(sum(a,b),c) = sum(cross(a,c),cross(b,c))
    prefix(x,cross(a,b,c,...)) = cross(prefix(x,a),prefix(x,b),prefix(x,c),...)
    prefix(x,sum(a,b,c,...)) = sum(prefix(x,a),prefix(x,b),prefix(x,c),...)

## Algebraic Tricks

**TODO** Represent overlapping hierarchical dimensions with the Tree Algebra.
    
# The Associative/Commutative API

## Basic API 

|Expression      |Description                                                  |
|----------------|-------------------------------------------------------------|
|nil             |The _ZDDNumber_ zero.                                        |
|( subtrees l t )|Linear combination of l times the subtrees of the the tree t.|
|( add z1 z2 )   |Sum of _ZDDNumbers_ z1, z2.                                  |
|( sub z1 z2 )   |Difference of _ZDDNumbers_ z1, z2.                           |

_add_ is _associative_ and _commutative_, and thus lends itself well to the concurrent execution of aggregation operations.   

## Filtering

**TODO**

# The Accumulative API

## Basic API

## Filtering

**TODO**

# The Implementation

The data structures are immutable variants of _ZDD_ (zero-suppressed binary decision diagrams) and numerical representations based on on ZDD, taken from the work of pr. _Shin-Ichi Minato_. _ZDD_ offer a compressed representation of sets of sets as found in combinatorial problems, that usually suffer from exponential size explosion.

# Future Work

* Use multiple hash functions to reduce collisions probability. Again, [Add ALL The Things][1] explains the idea neatly.   [Bloom Filters][4] are based on that technique, too.
* More operations...
  * Max and min.
  * Multiplication and division.

# Resources

* [Add ALL the Things: Abstract Algebra Meets Analytics][1]
* [VSOP Calculator based on Zero-Suppressed Binary Decision Diagrams][2]
* [Fun with ZDDs: Notes from Knuth’s 14th Annual Christmas Tree Lecture][3]
* [Bloom Filters][4]

[1]: http://www.infoq.com/presentations/abstract-algebra-analytics
[2]: https://github.com/ftod/zcube/blob/master/papers/VSOP%20(Valued-Sum-Of-Products)%20Calculator%20Based%20on%20Zero-Suppressed%20BDDs.pdf?raw=true
[3]: http://ashutoshmehra.net/blog/2008/12/notes-on-zdds/
[4]: http://en.wikipedia.org/wiki/Bloom_filters
