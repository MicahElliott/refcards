# Fully Qualified Namespaces

The **`::`** is for a _"Fully-Qualified NameSpace"_, or FQNS as I like to call
it, similar in concept to a
[FQDN](https://en.wikipedia.org/wiki/Fully_qualified_domain_name). It (`::`)
"expands" to either a "required alias" (eg, `::str` is an alias to the dotted
part of `(:require [clojure.string :as str])`), or the current NS it's in (eg,
`::` is an alias to the `(ns myproj.myns ...)` at the top of the current
file). It resolves to having a `/`, as compared to a bare key resolving with
no `/`.

## Comparing shapes of FQNSs

A FQNS (having or resolving to contain a `/`) can take a few shapes, and should be
looked at in comparison to the basic "unqualified" keyword syntax that uses a
single `:` and no `/`. FQNS "shortcut" keyword syntax uses leading colons
(`::`).

### Short-form shortcut (`::k` is "local" and `::aa/k` is "aliased")

These are "expanded" to a full NS as described above. _Think of the `::` as
representing a full long name being squeezed._ They _can_ contain dots
but I feel dashes are clearer. Note that the latter contains a `/`.

### Long-form explicit (`:bb.cc.dd/k`)

These are fully spelled out, explicit with dots and dashes in meaningful ways
to represent namespaces containing the keys. The dots are a low-level
(filesystem) detail, but represent the hierarchy of parent(s)/child, wherein
`aa.bb.cc` is `aa` as grand-parent, `bb` as parent, and `cc` as child. An
"explicit" FQNS has a single `:` but contains a `/` (eg, `:aa/bb`,
`:cc.dd/ee`).

### Bare, basic, unqualified (`:k`)

This is the basic unqualified key you've known about from day-1.

I find it helpful to have your editor make it obvious which shape you're
observing. Here you can see in the screenshot that the `::` is _red_ for the
current NS, the explicit full NS is _blue_, the "aliased" NS is _green_, and the
final "key" is _purple_.

[![Emacs screenshot with colored key syntax][1]][1]

## Full example

Notice in this example (which you can play with in a REPL) how these "resolve"
(see trailing comments). There are more than a dozen cases here that are all a
little different.

```clojure
(ns proj.ns1)
(ns proj.area.ns2)
(ns proj.ns3
  (:require
   [clojure.string   :as str]
   [clojure.data.avl :as d-a]   ; or da or avl or just a; be consistent across code base
   [clojure.data.zip :as d.z]   ; using dots in alias is confusing IMO
   [proj.area.ns2    :as ns2]   ; some fns being used
   [proj.ns1   :as-alias ns3])) ; keying convenience, new in v1.11, avoids circular deps

(def m "A contrived map demonstrating various key shapes"
  {:aa               "good"  ;=> :aa                      ; typical: basic unqualified key
   ::bb              "good"  ;=> :proj.ns3/bb             ; typical: note that / is implicitly added
   ::str             "bad"   ;=> :proj.ns3/str            ; missing key
   :ns3/cc           "weird" ;=> :ns3/cc                  ; missing full ns despite alias
   :proj.area.ns4/dd "ok"    ;=> :proj.area.ns4.dd        ; ns might not exist but ok
   ::ns2/ff          "good"  ;=> :proj.area.ns2/ff        ; common practice
   :proj.area.ns2/gg "ok"    ;=> :proj.area.ns2/gg        ; but this is why we have `:as-alias`
   :proj.ns1/hh      "bad"   ;=> :proj.ns1/hh             ; clearer to just use `::` for cur ns
   :str/ii           "bad"   ;=> :str/ii                  ; an accident: str not expanded
   ::str/jj          "good"  ;=> :clojure.string/jj       ; and typical
   ::kk.ll           "so-so" ;=> :proj.ns3/kk.ll          ; confusing to have dots in actual key
   ::d-a/mm.nn       "so-so" ;=> :clojure.data.json/mm.nn ; again, dots in key
   ::d-a/oo          "good"  ;=> :clojure.data.json/oo    ; typical
   ::d.z/pp          "so-so" ;=> :proj.ns3/pp             ; dots in qualifier diff from ns structure
   :random/qq        "good"  ;=> :random/qq               ; qualified, but not a real ns
   :other.random/rr  "good"  ;=> :other.random/rr         ; qualified, but not a real ns
   })
```

Note that these are all new keys you're "making". The `:jj` key doesn't actually
exist in the `clojure.string` NS, but that doesn't stop you from using it.

## Distilled

One of the most surprising things is that a `/` is added in the expansion in
the local uses (`::`). So these three are the ones to focus on remembering:

```
WRITTEN     RESOLVED
:aa      => :aa (bare)
::bb     => :proj.ns1/bb (qualified)
::ns3/cc => :proj.ns3/cc (qualified, slash explicit)
```

On the left side:

- `aa` and `bb` look almost identical, but resolve very differently
- `bb` and `cc` look quite different, but they're actually resolving very similarly

I somewhat wish there was `::/foo` instead of `::foo` since the implicit `/`
in the latter is what makes the system feel irregular.

It's important to know most of these FQNS cases when you're using libraries
like [malli](https://github.com/metosin/malli) or
[spec](https://clojure.org/about/spec) or
[integrant](https://github.com/weavejester/integrant) or various others, since
they make liberal use of FQNSs. And for any sizeable project, it usually
becomes necessary to qualify some keys to avoid collisions.

## Destructuring

Note the use of `:keys` and selection with `:`, `::`, and neither in
destructuring with these.

```clojure
(let [{:keys [aa]} m]               aa) ; "good" (typical)
(let [{:keys [:aa]} m]              aa) ; "good" (also works with :)
(let [{:keys [::aa]} m]             aa) ; nil
(let [{:keys [::bb]} m]             bb) ; "good"
(let [{:keys [ns2/ff]} m]           ff) ; nil
(let [{:keys [:ns2/ff]} m]          ff) ; nil
(let [{:keys [::ns2/ff]} m]         ff) ; "good"
(let [{:keys [ns3/cc]} m]           cc) ; "weird"
(let [{:keys [:ns3/cc]} m]          cc) ; "weird"
(let [{:keys [other.random/rr]} m]  rr) ; "good"
(let [{:keys [:other.random/rr]} m] rr) ; "good"
```

## Conclusion

This is a tricky area of Clojure syntax to keep straight, yet it is an
essential part of everyday code. It helps reduce confusion if you follow NS
style guides such as these:

- [Clojure Style Guide](https://github.com/bbatsov/clojure-style-guide#namespace-declaration)
- [How to ns](https://stuartsierra.com/2016/clojure-how-to-ns.html)

Note that there may be disagreement here around aliasing conventions (`str` vs
`string`, dots in aliases, etc), but the key is to settle your code base
conventions on _your_ rules and stay consistent.

  [1]: https://i.stack.imgur.com/55fpE.png
