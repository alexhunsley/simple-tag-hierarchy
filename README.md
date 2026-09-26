# Simple Tag Hierarchy (STH)

Useful for taxonomies and subsumption search.

STH supports multiple parents and reusable groups of tags.

For the examples (and the reference implementation *watch this space*) I'm using a simple ASCII tree format for the STH.

## Example

```
ceramics
-pottery
--wheel
--handbuild
-glazing
--dipping
--brushing

textiles
-weaving
--tapestry
--rug
-dyeing
--natural
--synthetic
```

(All top level tags -- `ceramics`, `textiles` -- are children of an unseen root node.)

## Splitting up trees

A deep tree can be made more manageable by splitting it up. This is done by writing the parent of a node after it in brackets, e.g. `design(pottery)`:

```
ceramics
-bioceramics
-pottery

design(pottery)
-colour
-pattern
```

This is the same as:

```
ceramics
-bioceramics
-pottery
--design
---colour
---pattern
```

We can also give a tag more than one parent: `tag(parent1,parent2)`. This attaches the tag and its subtree under each listed parent. Example:

```
ceramics
-pottery

textiles
-weaving

design(pottery,weaving)
-colour
-pattern
```

This is basically the same as this tag tree:

```
# this is not a valid spec
ceramics
-pottery
--design
---colour
---pattern

textiles
-weaving
--design
---colour
---pattern
```

(The above is actually invalid as a spec because you're not allowed to give the same tag >0 children in more than one place. **Use the bracketed parent technique to do this.**)

## Ghost tags

A **ghost tag** is one prefixed with `*`: it's a reference for the spec that doesn't translate into an actual tag outside of any spec parsing.

If earlier we'd made `design` into a ghost tag by writing `*design`:

```
ceramics
-pottery

textiles
-weaving

*design(pottery,weaving)
-colour
-pattern
```

then the effective tree would be:

```
ceramics
-pottery
--colour    # colour is direct child of pottery (no 'design')
--pattern   # pattern is direct child of pottery (no 'design')

textiles
-weaving
--colour    # colour is direct child of weaving (no 'design')
--pattern   # pattern is direct child of weaving (no 'design')
```

## Use of the tree

Any system using STH probably wants to offer a few kinds of query:

* `isDescendent(tagA, tagB, strict: Bool = true)` -- is `tagB` a descendent of `tagA`? (If `strict == false`, then `tagA == tagB` is considered a descendent)
* `isAncestor(tagA, tagB, strict: Bool = true)` -- is `tagB` an ancestor of `tagA`? (If `strict == false`, then `tagA == tagB` is considered an ancestor)
* `childTagsOf(tag: a, strict: Bool = true)` -- the set of tags (no dupes!) that are a descendent of `a`. (If `strict == false`, then `a` is in the returned set)

The implementation choices -- pre-compiling the tree / memoization / walking the STH upon query -- depends on things like:

* use cases
* data size
* frozen STH versus dynamic / real-time updating

## Validation of STH

A spec should be checked when it loads for:

- cycles
- tags defined in more than one place
- references to undeclared parents
- a transparent group name used as a tag on an item, or listed as a parent

## Why not use YAML?

YAML can express something very close to STH.

Anchors (`&name`) and aliases (`*name`) allows us to define a subtree once and reuse it:

```yaml
ceramics:
  pottery:
    design: &design
      colour:
      pattern:
textiles:
  weaving:
    design: *design
```

A ghost tag can be written with a merge key (<<), which splices the children in without adding a design level:

```yaml
_ghosts:
  design: &design
    colour:
    pattern:

ceramics:
  pottery:
    <<: *design
textiles:
  weaving:
    <<: *design
```

But if I'm understanding YAML correctly, there are some shortcomings:

* it's noisier: colons on every line and empty `colour:` leaves are harder on human eye
* references go the other way. In STH the child declares its parents `design(pottery,weaving)`, whereas In YAML each parent has to contain an alias to the child; so the parent list is spread across the file instead of sitting on one line
* an anchor must appear before any alias that uses it, so the first parent in the file "owns" the definition
* anchor scope is limited to current YAML doc
* ghosts are convention; `_ghosts`: is a key, so it becomes a node unless your loader knows to drop it. And merge keys (`<<`) aren't part of YAML 1.2 (but some parsers still support them)

Of course YAML file could reference the parent tags by a `parents: ` key (convention). But again, somewhat unwieldy.

## Related / reference

- **SKOS / ISO 25964** (thesauri): broader/narrower relations and polyhierarchy. See e.g. `rdflib` for a SKOS python lib
- **Org-mode:** group tags etc. Quite similar to STH, but unwieldy
- **Graphviz DOT:** multi-parent edge syntax
- **YAML:** (see anchors and aliases; like ghost tags) 
- **MeSH:** explode search
