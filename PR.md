## Title
fix(binder): report duplicate declaration error for block-scoped variables with expando assignments in JS files

## Summary

- In JavaScript files, a duplicate `const` declaration was silently accepted when an expando property assignment appeared between the two declarations (e.g., `const x = {}; x.foo = 1; const x = 1` produced no errors)
- The root cause was in `declareSymbol` in the binder: the condition that allows assignment declarations to merge with variables was too permissive — it checked only that the existing symbol had the `Assignment` flag, without verifying that the existing symbol wasn't already a `Variable` itself
- Added `!(symbol.flags & SymbolFlags.Variable)` guard so the merge exemption only applies when the existing symbol is a pure assignment (not already a variable declaration)

## Details

When `x.foo = 1` is processed in a JS file, the binder recognizes `x` as an expando symbol (initialized with `{}`) and adds `SymbolFlags.Module | SymbolFlags.Assignment` to the existing `x` symbol via `addDeclarationToSymbol`. When a subsequent `const x = 1` is bound, the duplicate declaration check at `binder.ts:807` would skip the error because the condition `!(includes & Variable && symbol.flags & Assignment)` was satisfied — the `Assignment` flag from the expando masked the conflict.

The fix adds a single guard: `!(symbol.flags & SymbolFlags.Variable)`, ensuring that when the existing symbol is already a Variable (i.e., `const`/`let`/`var`), the `Assignment` flag does not suppress the duplicate declaration error.

## Test plan

- [ ] New test `typeFromPropertyAssignment41` verifies that `const x = {}; x.foo = 1; const x = 1` in a JS file reports TS2451
- [ ] Existing test `typeFromPropertyAssignment31` (expando function + namespace merging) still passes
- [ ] Existing test `codeFixMissingTypeAnnotationOnExports43-expando-functions-5` still passes
