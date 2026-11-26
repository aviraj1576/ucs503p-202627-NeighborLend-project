# Week 3 : Choosing the Item Categories for Image-Based Identification

## Goal

Image-based identification should save the lender from typing the item type.
Before collecting any data, decide which kinds of item the model must
recognise first.

## Relevant Context

NeighborLend is for items that people own but rarely use, and that a
neighbour might need for a day or two: tools, sports and outdoor equipment,
electronics, household appliances.

## Key Observation

An object detector can only recognise the classes it was trained on, and
every class needs its own labelled images. Too many classes at the start
means too few images per class. The first model should cover the items most
likely to be listed, and leave the rest to a later, larger model.

## Solution

Selection criteria for a class:

1. it is commonly lent or rented (a drill, a ladder, a bicycle);
2. it has a recognisable shape in an ordinary phone photo;
3. enough example images can be collected and labelled in the time
   available.

The first model was planned with nine classes:

| Tools | Outdoor and sports | Electronics | Other |
|---|---|---|---|
| drill, hammer, pliers, ladder | bicycle, umbrella | headphones, keyboard | book |

Rules agreed with the frontend for using the model's output:

- the result is only a **suggestion**; the lender can always change it;
- a prediction below a confidence threshold is not used at all;
- an item outside the trained classes falls back to manual entry.

**Because**

A small, reliable model is more useful in the listing form than a large,
unreliable one, since a wrong suggestion has to be corrected by the user.
The class list was extended later (Week 16).
