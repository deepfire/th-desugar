`th-desugar` release notes
==========================

Version next
------------
* Fix a bug in which `toposortTyVarsOf` would error at runtime if given types
  containing `forall`s as arguments.
* Fix a bug in which `fvDType` would return incorrect results if given a type
  containing quantified constraints.
* Add more functions which compute free variables.

Version 1.9
-----------
* Suppose GHC 8.6.

* Add support for `DerivingVia`. Correspondingly, there is now a
  `DDerivStrategy` data type.

* Add support for `QuantifiedConstraints`. Correspondingly, there is now a
  `DForallPr` constructor in `DPred` to represent quantified constraint types.

* Remove the `DStarT` constructor of `DType` in favor of `DConT ''Type`.
  Two utility functions have been added to `Language.Haskell.TH.Desugar` to
  ease this transition:

  * `isTypeKindName`: returns `True` if the argument `Name` is that
    of `Type` or `★` (or `*`, to support older GHCs).
  * `typeKindName`: the name of `Type` (on GHC 8.0 or later) or `*` (on older
    GHCs).

* `th-desugar` now desugars all data types to GADT syntax. The most significant
  API-facing changes resulting from this new design are:

  * The `DDataD`, `DDataFamilyD`, and `DDataFamInstD` constructors of `DDec`
    now have `Maybe DKind` fields that either have `Just` an explicit return
    kind (e.g., the `k -> Type -> Type` in `data Foo :: k -> Type -> Type`)
    or `Nothing` (if lacking an explicit return kind).
  * The `DCon` constructor previously had a field of type `Maybe DType`, since
    there was a possibility it could be a GADT (with an explicit return type)
    or non-GADT (without an explicit return type) constructor. Since all data
    types are desugared to GADTs now, this field has been changed to be simply
    a `DType`.
  * The type signature of `dsCon` was previously:

    ```haskell
    dsCon :: DsMonad q => Con -> q [DCon]
    ```

    However, desugaring constructors now needs more information than before,
    since GADT constructors have richer type signatures. Accordingly, the type
    of `dsCon` is now:

    ```haskell
    dsCon :: DsMonad q
          => [DTyVarBndr] -- ^ The universally quantified type variables
                          --   (used if desugaring a non-GADT constructor)
          -> DType        -- ^ The original data declaration's type
                          --   (used if desugaring a non-GADT constructor).
          -> Con -> q [DCon]
    ```

    The `instance Desugar [Con] [DCon]` has also been removed, as the previous
    implementation of `desugar` (`concatMapM dsCon`) no longer has enough
    information to work.

  Some other utility functions have also been added as part of this change:

  * A `conExistentialTvbs` function has been introduced to determine the
    existentially quantified type variables of a `DCon`. Note that this
    function is not 100% accurate—refer to the documentation for
    `conExistentialTvbs` for more information.

  * A `mkExtraDKindBinders` function has been introduced to turn a data type's
    return kind into explicit, fresh type variable binders.

  * A `toposortTyVarsOf` function, which finds the free variables of a list of
    `DType`s and returns them in a well scoped list that has been sorted in
    reverse topological order.

* `th-desugar` now desugars partial pattern matches in `do`-notation and
  list/monad comprehensions to the appropriate invocation of `fail`.
  (Previously, these were incorrectly desugared into `case` expressions with
  incomplete patterns.)

* Add a `mkDLamEFromDPats` function for constructing a `DLamE` expression using
  a list of `DPat` arguments and a `DExp` body.

* Add an `unravel` function for decomposing a function type into its `forall`'d
  type variables, its context, its argument types, and its result type.

* Export a `substTyVarBndrs` function from `Language.Haskell.TH.Desugar.Subst`,
  which substitutes over type variable binders in a capture-avoiding fashion.

* `getDataD`, `dataConNameToDataName`, and `dataConNameToCon` from
  `Language.Haskell.TH.Desugar.Reify` now look up local declarations. As a
  result, the contexts in their type signatures have been strengthened from
  `Quasi` to `DsMonad`.

* Export a `dTyVarBndrToDType` function which converts a `DTyVarBndr` to a
  `DType`, which preserves its kind.

* Previously, `th-desugar` would silently accept illegal uses of record
  construction with fields that did not belong to the constructor, such as
  `Identity { notAField = "wat" }`. This is now an error.

Version 1.8
-----------
* Support GHC 8.4.

* `substTy` now properly substitutes into kind signatures.

* Expose `fvDType`, which computes the free variables of a `DType`.

* Incorporate a `DDeclaredInfix` field into `DNormalC` to indicate if it is
  a constructor that was declared infix.

* Implement `lookupValueNameWithLocals`, `lookupTypeNameWithLocals`,
  `mkDataNameWithLocals`, and `mkTypeNameWithLocals`, counterparts to
  `lookupValueName`, `lookupTypeName`, `mkDataName`, and `mkTypeName` which
  have access to local Template Haskell declarations.

* Implement `reifyNameSpace` to determine a `Name`'s `NameSpace`.

* Export `reifyFixityWithLocals` from `Language.Haskell.TH.Desugar`.

* Export `matchTy` (among other goodies) from new module `Language.Haskell.TH.Subst`.
  This function matches a type template against a target.

Version 1.7
-----------
* Support for TH's support for `TypeApplications`, thanks to @RyanGlScott.

* Support for unboxed sums, thanks to @RyanGlScott.

* Support for `COMPLETE` pragmas.

* `getRecordSelectors` now requires a list of `DCon`s as an argument. This
  makes it easier to return correct record selector bindings in the event that
  a record selector appears in multiple constructors. (See
  [goldfirere/singletons#180](https://github.com/goldfirere/singletons/issues/180)
  for an example of where the old behavior of `getRecordSelectors` went wrong.)

* Better type family expansion (expanding an open type family with variables works now).

Version 1.6
-----------
* Work with GHC 8, with thanks to @christiaanb for getting this change going.
  This means that several core datatypes have changed: partcularly, we now have
  `DTypeFamilyHead` and fixities are now reified separately from other things.

* `DKind` is merged with `DType`.

* `Generic` instances for everything.

Version 1.5.5
-------------

* Fix issue #34. This means that desugaring (twice) is idempotent over
expressions, after the second time. That is, if you desugar an expression,
sweeten it, desugar again, sweeten again, and then desugar a third time, you
get the same result as when you desugared the second time. (The extra
round-trip is necessary there to make the output smaller in certain common
cases.)

Version 1.5.4.1
---------------
* Fix issue #32, concerning reification of classes with default methods.

Version 1.5.4
-------------
* Added `expandUnsoundly`

Version 1.5.3
-------------
* More `DsMonad` instances, thanks to David Fox.

Version 1.5.2
-------------
* Sweeten kinds more, too.

Version 1.5.1
-------------
* Thanks to David Fox (@ddssff), sweetening now tries to use more of TH's `Type`
constructors.

* Also thanks to David Fox, depend usefully on the th-orphans package.

Version 1.5
-----------
* There is now a facility to register a list of `Dec` that internal reification
  should use when necessary. This avoids the user needing to break up their
  definition across different top-level splices. See `withLocalDeclarations`.
  This has a side effect of changing the `Quasi` typeclass constraint on many
  functions to be the new `DsMonad` constraint. Happily, there are `DsMonad`
  instances for `Q` and `IO`, the two normal inhabitants of `Quasi`.

* "Match flattening" is implemented! The functions `scExp` and `scLetDec` remove
  any nested pattern matches.

* More is now exported from `Language.Haskell.TH.Desugar` for ease of use.

* `expand` can now expand closed type families! It still requires that the
  type to expand contain no type variables.

* Support for standalone-deriving and default signatures in GHC 7.10.
  This means that there are now two new constructors for `DDec`.

* Support for `static` expressions, which are new in GHC 7.10.

Version 1.4.2
-------------
* `expand` functions now consider open type families, as long as the type
   to be expanded has no free variables.

Version 1.4.1
-------------
* Added `Language.Haskell.TH.Desugar.Lift`, which provides `Lift` instances
for all of the th-desugar types, as well as several Template Haskell types.

* Added `applyDExp` and `applyDType` as convenience functions.

Version 1.4.0
-------------
* All `Dec`s can now be desugared, to the new `DDec` type.

* Sweetening `Dec`s that do not exist in GHC 7.6.3- works on a "best effort" basis:
closed type families are sweetened to open ones, and role annotations are dropped.

* `Info`s can now be desugared. Desugaring takes into account GHC bug #8884, which
meant that reifying poly-kinded type families in GHC 7.6.3- was subtly wrong.

* There is a new function `flattenDValD` which takes a binding like
  `let (a,b) = foo` and breaks it apart into separate assignments for `a` and `b`.

* There is a new `Desugar` class with methods `desugar` and `sweeten`. See
the documentation in `Language.Haskell.TH.Desugar`.

* Variable names that are distinct in desugared code are now guaranteed to
have distinct answers to `nameBase`.

* Added a new function `getRecordSelectors` that extracts types and definitions
of record selectors from a datatype definition.

Version 1.3.1
-------------
* Update cabal file to include testing files in sdist.

Version 1.3.0
-------------
* Update to work with `type Pred = Type` in GHC 7.9. This changed the
`DPred` type for all GHC versions, though.

Version 1.2.0
-------------
* Generalized interface to allow any member of the `Qausi` class, instead of
  just `Q`.

Version 1.1.1
-------------
* Made compatible with HEAD after change in role annotation syntax.

Version 1.1
-----------
* Added module `Language.Haskell.TH.Desugar.Expand`, which allows for expansion
  of type synonyms in desugared types.
* Added `Show`, `Typeable`, and `Data` instances to desugared types.
* Fixed bug where an as-pattern in a `let` statement was scoped incorrectly.
* Changed signature of `dsPat` to be more specific to as-patterns; this allowed
  for fixing the `let` scoping bug.
* Created new functions `dsPatOverExp` and `dsPatsOverExp` to allow for easy
  desugaring of patterns.
* Changed signature of `dsLetDec` to return a list of `DLetDec`s.
* Added `dsLetDecs` for convenience. Now, instead
  of using `mapM dsLetDec`, you should use `dsLetDecs`.

Version 1.0
-----------

* Initial release
