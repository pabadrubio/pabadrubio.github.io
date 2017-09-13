---
layout: post
title:  "Dremel FSM Construction"
date:   2017-09-04 14:32:02 +0200
categories: jekyll update
---
## The algorithm
The FSM Construction algorithm is used to create the FSM that the Record Assembly algorithm uses. The suggested
algorithm on the Dremel paper has a couple of points which are slightly open to interpretation and some steps which
I could not reproduce or that didn´t provide a correct algorithm in all cases in the way I interpreted them.
For this I felt free to make slight changes to the algorithm in order to make it work.

The algorithm generates a FSM in which the states are the current atomic field to be evaluated and the transitions
are given by the next value for the repetition level. The value for the next repetition level has implicit information
about repeated or absent fields that can be leveraged in the Record Assembly algorithm. For example, gievn the following
schema:
```
message Data
  repeated group A
    value E
    repeated group B
      value D
      repeated value C
```
When the record assembly algorith is currently reading a value on the column C, the next repetition level can be seen to
intuitively show the following:
- When it is 3 it means that the next value comes from a repeated C (level 3) and then the next column to try to read is C
- When it is 2 it means that no more C values where repeated and that a new level 2 record was found, which means we should
  try to read the first value of this record: a D column value
- When it is 1 it means that no group B was repeated, but a new group A was opened. In this case the first value to
  try to read is E.

This example already approached us to one of the ideas behind the FSM construction algorithm. Some next repetition values
in case of nested records will imply for the FSM a transition to a previous field. In this case this is the first field
for the group with the repetition level similar to the NRL.
But of course there is another case to count on: we need to advance to the next field. This is easy.
We will do, but we will check we are not missing repetitions nested.
```
message Data
  repeated group A
    value E
    repeated group B
      value D
      repeated value C
  value F
```
For example in this case, we will only advance when the NRL is 0, as it means that we will get no more C values for this
message.
```
message Data
  repeated group A
    value E
    repeated group B
      value D
      repeated value C
    value F
```
For this case, we will only advance when NRL is 0 or 1, as we need to check repetitions of group A or the end of the
message. In order to know when to advance and when to go back the Dremel paper uses two concepts:
- The barrier, which is nothing but the next atomic field in the schema
- The barrierLevel which is the repetition level of the field and barrier common ancestor. For example in the previous
example the ancestor is A, which has a repetition level of 1. If A was not repeated the barrierLevel would be 0 (In the
  Dremel paper it is not explained as it is, but it is either an error or I am missing something)

With the barrier level we already can calculate the FSM easily, with the following rules:
- For NRL below or equal to the barrierLevel, create a transition to the barrier
- For NRL over the barrierLevel, calculate the group with the repetition level equal to NRL in which C is included and
  get the first field to set the transition.

There is one more situations which need to be dealed with:
- If the atomic field is repeated a NRL with the maximum value should create a transition to itself. (This is not directly
  explained in the Dremel paper and was one of the most confused things to me. I integrated it in the algorithm as I will
  explain later)

To implement this behaviour the following algorithm has been created:

The last part is just taking the NRL under the barrier level and creating a transition. The first part is trying to
create transitions to the first of previous fields. To do it, it goes from the current field back and for each field with
a common repetition level (another mistake in the Dremel paper as I see) over the barrierLevel, we create a transition.
I implemented an optimization, as as soon as we are below the barrier level, we are getting to fields which will never
be reached as the barrier has preference, so we break at the first one.
There is a last thing to deal with. There might be cases in which there is an atomic field that is the first to appear
in two different levels. For example, in this case:
```
message Data
  repeated group A
    repeated group B
      repeated group C
        value D
        repeated value E
  value F
```
Field D is the first to open a new group A, B and C. As the barrier is at A level (1) and with our approach only a transition
at level 3 is made, we need to copy transitions from level 3 to level 2 and below until the barrierLevel is reached.
This is done in a different way as in the paper.

## The implementation
We will implement a basic FSM, just by adding to each of the Field Readers transitions.
