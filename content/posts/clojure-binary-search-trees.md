---
layout: post
title: "Dendrology:"
subtitle: " Binary Search Trees in Clojure"
description: "Creating our own binary search tree in Clojure."
modified: 2015-5-5
tags: [clojure, algorithms, tree, graph, code]
image:
  feature: generative-1.jpg
---
# Clojure Dendrology

This is the first entry in a series geared towards helping people learn Clojure by walking them through the implementation of various tree data structures and algorithms. I'm going to assume at least a passing familiarity with Lisp and the ability to look up information in the [Clojure docs](https://clojure.github.io/clojure/) as this will be more of a high level overview of writing a short, but non-trivial Clojure program. If you're already familiar with Clojure, but are interested in learning about or brushing up on your functional algorithms and data structures, then this is also a good resource for you. I've tried to make the code as readable and idiomatic as possible; if you find anywhere it can be improved towards this end, then please leave a comment or shoot me an at [curtis@partiallyapplied.io](mailto:curtis@partiallyapplied.io).

## What is a Binary Search Tree?

A binary search tree (BST) is a binary tree that conforms to the following properties:
* The left child subtree of a node contains only nodes with values less than the parent node’s value.
* The right child subtree of a node contains only nodes with values greater than the parent node’s value.
* Both the left and right child subtrees must also be binary search trees.

<figure>
    <img src="/images/binary-search-tree.png" alt="" 
    style="display:block; width:50%; height:50%; margin-left:auto; margin-right:auto;">
</figure>

## Data Representation

There are a few different ways we can represent our tree nodes in Clojure, such as records, structs, or just a map. Each option gives us a clojure persistant hash-map under the hood, just with varying levels of abstractions on top. For a small project like this, a regular old map should work just fine.


To represent a BST nodes as a clojure map we are going to need to be able to access a node's value, and it's left and right subtrees. So we just need a map that contains three keys, **:val**, **:left**, and **:right**. So a leaf node (a node with no subtrees) with the value 8 would looke like this:

{% highlight clojure %}
{:val 8 :left nil :right nil}
{% endhighlight %}

However, if we don't have a left or right subtree for our node, we don't actually need our map to have **:left** and **:right** keys in our node. If we try and access key in a map that doesn't contain that key, clojure already returns a nil.

{% highlight clojure %}
user> (:left {:val 8 :left nil :right nil})
nil

user> (:left {:val 8})
nil
{% endhighlight %}

It's usually best practice to never have nil values for map keys in Clojure for this reason. It leads to needless overspecification of your data, and makes it more difficult to visually parse your printed data. When we have a parent node that does have child nodes, it will look like this:

{% highlight clojure %}

;; A parent node with two child leaf nodes
{:val 8
 :left {:val 4}
 :right {:val 12}}

;; A parent node with two child subtrees
{:val 8
 :left {:val 4
        :left {:val 2}}
 :right {:val 12
         :left {:val 10}
         :right {:val 14
                 :right {:val 20}}}}
{% endhighlight %}

## Insertion

Now that we know our data representation, we need a way to insert a new node into an existing BST. We're going to define a function called **bst-insert** that has two parameters, the value we want to insert, and an optional root node of the BST we want to insert it into. When we don't supply an existing BST we want to create a new one with our value in the root node.

When we're inserting into an existing tree we need to find the correct insertion point for the new element. When our value is less than the root, we call **bst-insert** recursively on the left subtree and associate (**assoc**) it's return value as the root's new **:left** subtree. We do the same for the right subtree when our value is greater than the root. Our BST implementation isn't going to allow duplicate values, so when our value is the same as the root's, we just return the root node with no change. 

Our insertion algorithm terminates when it reaches a leaf node, because we have found a open spot for our new value. So when **bst-insert** is passed a root that is a nil, we just return a leaf node that contains our new value instead of making another recursive call.

{% highlight clojure %}
(defn bst-insert
  "Insert val into binary tree at node root, if root is
  not supplied create a new binary tree with one node."
  ([val]
   {:val val})
  ([val root]
   (cond
     (nil? root) {:val val}
     (< val (root :val)) (assoc root :left (bst-insert val (root :left)))
     (> val (root :val)) (assoc root :right (bst-insert val (root :right)))
     (= val (root :val)) root)))
{% endhighlight %}

Now that we have a way to insert elements into an existing tree we can use this to create a BST out of a sequence. We'll call this function **build-bst** and it takes a sequence of values and an optional root node. When it's called without a root node, it will create a root node by calling **bst-insert** on the first value in the sequence, then it will casll itself recursively with the rest of the sequence and our new root node. However, if it is passed an empty sequence, it will default to returning nil. 

Similarly, when it is called with a sequence and a root node, it will insert the first value of the sequence into the root, and the recursively call itself with the rest of the sequence and the new root. When it has inserted every value in the sequence, it will end up making that recursive call with an empty sequence, so when we are given an empty sequence, we know that all the values have already been inserted, and we can just return the root.

{% highlight clojure %}
(defn build-bst
  "Recursively builds a binary tree out of the elements in xs"
  ([xs]
   (if (empty? xs)
     nil
     (build-bst (rest xs) (bst-insert (first xs)))))
  ([xs root]
   (if (empty? xs)
     root
     (build-bst (rest xs) (bst-insert (first xs) root)))))
{% endhighlight %}

Now we can call **build-bst** on a vector of integers, and get back a map of our BST.

{% highlight clojure %}
user> (build-bst (shuffle (range 10)))
{:val 6,
 :left {:val 0,
        :right {:val 4,
                :right {:val 5},
                :left {:val 3,
                       :left {:val 2,
                              :left {:val 1}}}}},
 :right {:val 8,
         :right {:val 9},
         :left {:val 7}}}
{% endhighlight %}

## Printing

Before we move on to implementing the other BST operations, it will be usefull to be able to print our tree in a way that's easier to visualize than looking at map in the REPL. To make this easier to implement, we'll print our tree with the root node anchored to the left of the screen instead of the top. Then our right subtree will be above the root, and the left subtree below it. We'll keep track of our depth in the tree as we traverse it, so that we know how many times we nee to indent each node. When a node doesn't have a left or right subtree, we will print a period (**.**) in place of a value.

{% highlight clojure %}
(defn print-bst
  "Prints a binary search tree."
  ([node]
   (do
     (print-bst (node :right) 1)
     (println (node :val))
     (print-bst (node :left) 1)))
  ([node depth]
   (let [padding (clojure.string/join (repeat depth "    "))]
     (if (nil? node)
       (println padding ".")
       (do
         (print-bst (node :right) (inc depth))
         (println padding (node :val))
         (print-bst (node :left) (inc depth)))))))
{% endhighlight %}

Here's what our **print-bst** function looks like when used from the REPL:

{% highlight clojure %}
user> (print-bst (build-bst (shuffle (range 10))))
                 .
           9
                   .
               8
                   .
       7
           .
  6
           .
       5
                       .
                   4
                       .
               3
                   .
           2
                   .
               1
                       .
                   0
                       .
{% endhighlight %}

## Verification

Now lets write a function that allows us to check a binary tree to make sure it adheres to the binary search properties. We're going to call this function **bst?**. We use a question mark in the function name to signal that this function is a predicate (evaluates to either true or false), this is just a Clojure convention that makes programs easier to read. Just like with our previous functions, well be traversing the tree, and at every node checking that the node's value is greater than those in it's left subtree, and less than those in it's right subtree. We use the **cond** macro to run the validation based on whether or not our current node has a left subtree, right subtree, both, or neither. We return a false if we encounter a node that doesn't adhere to the binary search properties, and when we reach a leaf node we return true because there are no more subtrees to validate, and we know that all of it's parent nodes have already been validated.

{% highlight clojure %}
(defn bst? [node]
  "True if the tree satisfies the binary search property."
  (cond
    (and (node :left) (node :right))
      (and (> (node :val) ((node :left) :val))
           (< (node :val) ((node :right) :val))
           (bst? (node :left))
           (bst? (node :right)))
    (node :left)
      (and (> (node :val) ((node :left) :val))
           (bst? (node :left)))
    (node :right)
      (and (< (node :val) ((node :right) :val))
           (bst? (node :right)))
    (node :val) true
    :else false))
{% endhighlight %}

## Search

Now we'll need to implement some search functions so that we can access the values stored in our BST. We may want to be able to tell if our BST contains a certain value, as well as find it's minimum and maximum values. If you've followed along this far, these functions should be fairly self explanatory.

{% highlight clojure %}
(defn bst-contains? [node val]
  "True if the binary tree at node contains a node
   with the value val, false otherwise."
  (cond
    (not node) false
    (< val (node :val)) (bst-contains? (node :left) val)
    (> val (node :val)) (bst-contains? (node :right) val)
    :else true))

(defn bst-find-min [node]
  "returns the smallest value in the
   binary tree."
  (cond
   (not node) nil
   (node :left) (bst-find-min (node :left))
   :else (node :val)))

(defn bst-find-max [node]
  "returns the largest value from the
   binary tree."
  (cond
   (not node) nil
   (node :right) (bst-find-max (node :right))
   :else (node :val)))
{% endhighlight %}

## Deletion

Now for the trickiest operation on a BST, deletion. We not only have to remove the right node from the tree, but we also have to ensure that it maintains the binary search properties. This means we'll have to move some nodes around in the tree. First we're going to simplify the problem down to just deleting the minimum node in the BST by writing a **bst-delete-min** function. Once we've done that, we can use our new function to help us solve the more general case. Before we even get to that though, we'll need another helper function **leaf?**, that takes a node and returns true if it's a leaf node and false otherwise.

{% highlight clojure %}
(defn leaf? [node]
  "True if node is a leaf node."
  (and (nil? (:left node))
       (nil? (:right node))))
{% endhighlight %}

Our **bst-delete-min** function is going to return both the value that it deleted, and the new BST with that value deleted. It will become clear why we do this when we get to implementing the general **bst-delete** function. 

For deleting the minimum value, we only care about the left subtree because that's where all of the values smaller than the root are. We can divide how we handle the left subtree into three cases:
1. There is no left subtree.
2. The left subtree is a leaf node.
3. The left subtree is a another root node.

The first case is the easiest to deal with because then we know that our root node's value is the minimum, so we can just return the right subtree unmodified. The second case is also fairly easy. If the left subtree is a leaf node, we know that it is our minimum value. Therefore, we can just return our root node with its left subtree removed.

We handle the last case by recursively calling **bst-delete-min** on the left subtree and then returning our root node with it's left subtree replaced by the new subtree that has the value deleted. The recursive call will continue to be made until we reach on of our first two cases that we already know how to handle.

{% highlight clojure %}
(defn bst-delete-min [node]
  "Delete the minimum value in a binary tree. Returns a vector with the
   value that was deleted, and the new tree without that value."
  (cond
    (nil? (node :left))
      [(node :val) (node :right)]
    (leaf? (node :left))
      [((node :left) :val) {:val (node :val) :right (node :right)}]
    :else
      (let [[min left] (bst-delete-min (node :left))]
        [min {:left left :val (node :val) :right (node :right)}])))
{% endhighlight %}

For the general deletiong function we once again have three cases we have to deal with:
1. The node we want to delete is in the left subtree.
2. The node we want to delete is in the right subtree.
3. The node we want to delete is the current node.

The first two cases are handled by recursively calling **bst-delete** on the appropriate subtree, and then replacing that subtree in our current node with the new subtree with the value deleted.

In the last case, if either our left or right subtree is a leaf node we can delete the current value by just returning the opposite subtree. Otherwise, we delete the current node by replacing it's value with the minimum value of it's right subtree, and then delete the minimum value from the right subtree. We use our **bst-delete-min** to both get the minimun of the right subtree, and a new right subtree with that value deleted. By replacing our current nodes value with the minimum of the right subtree, we ensure that the binary search properties are maintained.

{% highlight clojure %}
(defn bst-delete [node val]
  "Delete a value from a binary tree."
  (cond
    (< val (node :val))
    (assoc node :left (bst-delete (node :left) val))
    (> val (node :val))
    (assoc node :right (bst-delete (node :right) val))
    :else
    (cond
      (leaf? (node :right)) (node :left)
      (leaf? (node :left)) (node :right)
      :else (let [[min right] (bst-delete-min (node :right))]
              {:left (node :left) :val min :right right}))))
{% endhighlight %}

You may have noticed that we could have also implemented deletion by using a **bst-delete-max** function and using it to replace the current node's value with the maximum of the left subtree.
