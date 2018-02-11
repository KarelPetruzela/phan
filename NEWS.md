Phan NEWS

11 Feb 2018, Phan 0.8.12
------------------------

### Backported from Phan 0.10.4

New Features(Analysis)

+ Support array key types of `int`, `string`, and `mixed` (i.e. `int|string`) in union types such as `array<int,T>` (#824)

  Check that the array key types match when assigning expected param types, return types, property types, etc.
  By default, an array with a key type of `int` can't cast to an array key type of `string`, or vice versa.
  Mixed union types in keys can cast to/from any key type.

  - To allow casting `array<int,T>` to `array<string,T>`, enable `scalar_array_key_cast` in your `.phan/config.php`.

+ Warn when using the wrong type of array keys offsets to fetch from an array (E.g. `string` key for `array<int,T>`) (Issue #1390)
+ Infer array key types of `int`, `string`, or `int|string` in `foreach` over arrays. (#1300)
  (Phan's type system doesn't support inferring key types for `iterable` or `Traversable` right now)
+ Support **parsing** PHPDoc array shapes
  (E.g. a function expecting `['field' => 'a string']` can document this as `@param array{field:string}` $options)
  For now, this is converted to generic arrays (Equivalent to `string[]`).

  `[[0, ...], new stdClass]` would have type `array{0:int[], 1:string}`

  - The field value types can be any union type.
  - Field keys are currently limited to keys matching the regex `[-_.a-zA-Z0-9\x7f-\xff]+`. (Identifiers, numbers, '-', and '.')
    Escape mechanisms such as backslashes (e.g. "\x20" for " ") may be supported in the future.
+ Add `PhanTypeMismatchUnpackKey` and `PhanTypeMismatchUnpackValue` to analyze array unpacking operator (also known as splat) (#1384)

  Emit `PhanTypeMismatchUnpackKey` when passing iterables/arrays with invalid keys to the unpacking operator (i.e. `...`).

  Emit `PhanTypeMismatchUnpackValue` when passing values that aren't iterables or arrays to the unpacking operator.
  (See https://secure.php.net/manual/en/migration56.new-features.php#migration56.new-features.splat)
+ When determining the union type of an array literal,
  base it on the union types of **all** of the values (and all of the keys) instead of just the first 5 array elements.
+ When determining the union type of the possible value types of a array literal,
  combine the generic types into a union type instead of simplifying the types to `array`.
  In practical terms, this means that `[1,2,'a']` is seen as `array<int,int|string>`,
  which Phan represents as `array<int,int>|array<int,string>`.

  In the previous Phan release, the union type of `[1,2,'a']` would be represented as `int[]|string[]`,
  which is equivalent to `array<mixed,int>|array<mixed,string>`

  Another example: `[$strKey => new MyClass(), $strKey2 => $unknown]` will be represented as
  `array<string,MyClass>|array<string,mixed>`.
  (If Phan can't infer a type of a key or value, `mixed` gets added to that key or value.)
+ Improve analysis of try/catch/finally blocks (#1408)
  Analyze `catch` blocks with the inferences about the `try` block.
  Analyze a `finally` block with the combined inferences from the `try` and `catch` blocks.
+ Account for side effects of `&&` and `||` operators in expressions, outside of `if`/`assert` statements. (#1415)
  E.g. `$isValid = ($x instanceof MyClass && $x->isValid())` will now consistently check that isValid() exists on MyClass.
+ Improve analysis of expressions within conditionals, such as `if (!($x instanceof MyClass) || $x->method())`
  or `if (!(cond($x) && othercond($x)))`

  (Phan is now aware of the types of the right hand side of `||` and `&&` in more cases)
+ Add a large number of param and return type signatures for internal functions and methods,
  for params and return types that were previously untyped.
  (Imported from docs.php.net's SVN repo)
+ More precise analysis of the return types of `var_export()`, `print_r()`, and `json_decode()` (#1326, #1327)
+ Improve type narrowing from `iterable` to `\Traversable`/`array` (#1427)
  This change affects `is_array()`/`is_object()` checks and their negations.
+ Fix more edge cases which would cause Phan to fail to infer that properties, constants, or methods are inherited. (PR #1440 for issues #311, #1426, #454)

Plugins
+ Fix bugs in `NonBoolBranchPlugin` and `NonBoolInLogicalArithPlugin` (#1413, #1410)
+ **Make UnionType instances immutable.**
  This will affect plugins that used addType/addUnionType/removeType. withType/withUnionType/withoutType should be used instead.
  To modify the type of elements(properties, method return types, parameters, variables, etc),
  plugin authors should use `Element->setUnionType(plugin_modifier_function(Element->getUnionType()))`.

Language server:
+ Add a CLI option `--language-server-analyze-only-on-save` to prevent the client from sending change notifications. (#1325)
  (Only notify the language server when the user saves a document)
  This significantly reduces CPU usage, but clients won't get notifications about issues immediately.

Bug fixes
+ Warn when attempting to call an instance method on an expression with type string (#1314).
+ Fix a bug in `tool/make_stubs` when generating stubs of global functions.
+ Fix some bugs that occurred when Phan resolved inherited class constants in class elements such as properties. (#537 and #454)
+ Emit an issue when a function/method's parameter defaults refer to an undeclared class constant/global constant.

20 Jan 2018, Phan 0.8.11
------------------------

### Backported from Phan 0.10.3

New Features(CLI, Configs)

+ For `--fallback-parser`: Switch to [tolerant-php-parser](https://github.com/Microsoft/tolerant-php-parser)
  as a dependency of the fallback implementation. (#1125)
  This does a better job of generating PHP AST trees when attempting to parse code with a broader range of syntax errors.
  Keep `PHP-Parser` as a dependency for now for parsing strings.

Maintenance
+ Various performance optimizations, including caching of inferred union types to avoid unnecessary recalculation.
+ Make `phan_client` and the vim snippet in `plugins/vim/phansnippet.vim` more compatible with neovim
+ Upgrade felixfbecker/advanced-json-rpc dependency to ^3.0.0 (#1354)
+ Performance improvements.
  Changed the internal representation of union types to no longer require `spl_object_id` or the polyfill.

Bug Fixes
+ Allow `null` to be passed in where a union type of `mixed` was expected.
+ Don't warn when passing `?T` (PHPDoc or real) where the PHPDoc type was `T|null`. (#609, #1090, #1192, #1337)
  This is useful for expressions used for property assignments, return statements, function calls, etc.
+ Fix a few of Phan's signatures for internal functions and methods.

17 Nov 2017, Phan 0.8.10
------------------------

Phan 0.8.x switched from using AST version 40 (php-ast 0.1.4+) to AST version 50 (requires php-ast 0.1.5+) in this release.

### Backported from Phan 0.10.2

New Features(Analysis)
+ Enable `simplify_ast` by default.
  The new default value should reduce false positives when analyzing conditions of if statements. (#407, #1066)
+ Support less ambiguous `?(T[])` and `(?T)[]` in phpdoc (#1213)
  Note that `(S|T)[]` is **not** supported yet.
+ Support alternate syntax `array<T>` and `array<Key, T>` in phpdoc (PR #1213)
  Note that Phan ignores the provided value of `Key` completely right now (i.e. same as `T[]`); Key types will be supported in Phan 0.10.3.
+ Speed up Phan analysis on small projects, reduce memory usage (Around 0.15 seconds and 15MB)
  This was done by deferring loading the information about internal classes and functions until that information was needed for analysis.
+ Analyze existence and usage of callables passed to (internal and user-defined) function&methods expecting callable. (#1194)
  Analysis will now warn if the referenced function/method of a callable array/string
  (passed to a function/method expecting a callable param) does not exist.

  This change also reduces false positives in dead code detection (Passing in these callable arrays/strings counts as a reference now)
+ Warn if attempting to read/write to an property or constant when the expression is a non-object. (or not a class name, for static elements) (#1268)
+ Split `PhanUnreferencedClosure` out of `PhanUnreferencedFunction`. (Emitted by `--dead-code-detection`)
+ Split `PhanUnreferencedMethod` into `PhanUnreferencedPublicMethod`, `PhanUnreferencedProtectedMethod`, and `PhanUnreferencedPrivateMethod`.
+ Split errors for class constants out of `PhanUnreferencedConst`:
  Add `PhanUnreferencedPublicClassConst`, `PhanUnreferencedProtectedClassConst`, and `PhanUnreferencedPrivateClassConst`.
  `PhanUnreferencedConst` is now exclusively used for global constants.
+ Analyze uses of `compact()` for undefined variables (#1089)
+ Add `PhanParamSuspiciousOrder` to warn about mixing up variable and constant/literal arguments in calls to built in string/regex functions
  (`explode`, `strpos`, `mb_strpos`, `preg_match`, etc.)
+ Preserve the closure's function signature in the inferred return value of `Closure::bind()`. (#869)
+ Support indicating that a reference parameter's input value is unused by writing `@phan-output-reference` on the same line as an `@param` annotation.
  This indicates that Phan should not warn about the passed in type, and should not preserve the passed in type after the call to the function/method.
  (In other words, Phan will analyze a user-defined reference parameter the same way as it would `$matches` in `preg_match($pattern, $string, $matches)`)
  Example usage: `/** @param string $x @phan-output-reference */ function set_x(&$x) { $x = 'result'; }`
+ Make phan infer unreachability from fatal errors such as `trigger_error($message, E_USER_ERROR);` (#1224)
+ Add new issue types for places where an object would be expected:
  `PhanTypeExpectedObjectPropAccess`, `PhanTypeExpectedObjectPropAccessButGotNull`, `PhanTypeExpectedObjectStaticPropAccess`,
  `PhanTypeExpectedObject`, and `PhanTypeExpectedObjectOrClassName
+ Emit more accurate line numbers for phpdoc comments, when warning about phpdoc in doc comments being invalid. (#1294)
  This gives up and uses the element's line number if the phpdoc ends over 10 lines before the start of the element.
+ Work on allowing union types to be part of template types in doc comments,
  as well as types with template syntax.
  (e.g. `array<int|string>` is now equivalent to `int[]|string[]`,
  and `MyClass<T1|T2,T3|T4>` can now be parsed in doc comments)
+ Disambiguate the nullable parameter in output.
  E.g. an array of nullable integers will now be printed in error messages as `(?int)[]`
  A nullable array of integers will continue to be printed in error messages as `?int[]`, and can be specified in PHPDoc as `?(int[])`.

New Features (CLI, Configs)
+ Improve default update rate of `--progress-bar` (Update it every 0.10 seconds)

Bug Fixes
+ Fixes bugs in `PrintfCheckerPlugin`: Alignment goes before width, and objects with __toString() can cast to %s. (#1225)
+ Reduce false positives in analysis of gotos, blocks containing gotos anywhere may do something other than return or throw. (#1222)
+ Fix a crash when a magic method with a return type has the same name as a real method.
+ Allow methods to have weaker PHPdoc types than the overridden method in `PhanParamSignatureMismatch`. (#1253)
  `PhanParamSignatureRealMismatch*` is unaffected, and will continue working the same way in Phan releases analyzing PHP < 7.2.
+ Stop warning about `PhanParamSignatureMismatch`, etc. for private methods. The private methods don't affect each other. (#1250)
+ Properly parse `?self` as a *nullable* instance of the current class in union types (#1264)
+ Stop erroneously warning about inherited constants being unused in subclasses for dead code detection (#1260)
+ For dead code detection, properly track uses of inherited class elements (methods, properties, classes) as uses of the original definition. (#1108)
  Fix the way that uses of private/protected methods from traits were tracked.
  Also, start warning about a subset of issues from interfaces and abstract classes (e.g. unused interface constants)
+ Properly handle `static::class` as a class name in an array callable, or `static::method_name` in a string callable (#1232)
+ Make `@template` tag for [Generic Types](https://github.com/phan/phan/wiki/Generic-Types) case sensitive. (#1243)
+ Fix a bug causing Phan to infer an empty union type (which can cast to any type) for arrays with elements of empty union types. (#1296)

Plugins
+ Make DuplicateArrayKeyPlugin start warning about duplicate values of known global constants and class constants. (#1139)
+ Make DuplicateArrayKeyPlugin start warning about case statements with duplicate values (This resolves constant values the same way as array key checks)
+ Support `'plugins' => ['AlwaysReturnPlugin']` as shorthand for full relative path to a bundled plugin such as AlwaysReturnPlugin.php (#1209)

Maintenance
+ Performance improvements: Phan analysis is 13%-22% faster than 0.10.1, with `simplify_ast` enabled.
+ Used PHP_CodeSniffer to automatically make Phan's source directory adhere closely to PSR-1 and PSR-2, making minor changes to many files.
  (e.g. which line each brace goes on, etc.)
+ Stop tracking references to internal (non user-defined) elements (constants, properties, functions, classes, and methods) during dead code detection.
  (Dead code detection now requires an extra 15MB instead of 17MB for self-analysis)

20 Oct 2017, Phan 0.8.9
-----------------------

### Backported from Phan 0.9.7

New Features(Analysis)
+ Support `@return $this` in phpdoc for methods and magic methods.
  (but not elsewhere. E.g. `@param $this $varName` is not supported, use `@param static $varName`) (#634)
+ Check if functions/methods passed to `array_map` and `array_filter` are compatible with their arguments.
  Recursively analyze the functions/methods passed to `array_map`/`array_filter` if no types were provided. (unless quick mode is being used)

New Features (CLI, Configs)

+ Add Language Server Protocol support (Experimental) (#821)
  Compatibility: Unix, Linux (depends on php `pcntl` extension).
  This has the same analysis capabilities provided by [daemon mode](https://github.com/phan/phan/wiki/Using-Phan-Daemon-Mode#using-phan_client-from-an-editor).
  Supporting a standard protocol should make it easier to write extensions supporting Phan in various IDEs.
  See https://github.com/Microsoft/language-server-protocol/blob/master/README.md
+ Add config (`autoload_internal_extension_signatures`) to allow users to specify PHP extensions (modules) used by the analyzed project,
  along with stubs for Phan to use (instead of ReflectionFunction, etc) if the PHP binary used to run Phan doesn't have those extensions enabled. (#627)
  Add a script (`tool/make_stubs`) to output the contents of stubs to use for `autoload_internal_extension_signatures` (#627).
+ By default, automatically restart Phan without xdebug if xdebug is enabled. (#1161)
  If you wish to analyze a project using xdebug's functions, set `autoload_internal_extension_signatures`
  (e.g. `['xdebug' => 'vendor/phan/phan/.phan/internal_stubs/xdebug.phan_php']`)
  If you wish to use xdebug to debug Phan's analysis itself, set and export the environment variable `PHAN_ALLOW_XDEBUG=1`.
+ Improve analysis of return types of `array_pop`, `array_shift`, `current`, `end`, `next`, `prev`, `reset`, `array_map`, `array_filter`, etc.
  See `ArrayReturnTypeOverridePlugin.php.`
  Phan can analyze callables (for `array_map`/`array_filter`) of `Closure` form, as well as strings/2-part arrays that are inlined.
+ Add `--memory-limit` CLI option (e.g. `--memory-limit 500M`). If this option isn't provided, there is no memory limit. (#1148)

Maintenance
+ Document the `--disable-plugins` CLI flag.

Plugins
+ Add a new plugin capability `ReturnTypeOverrideCapability` which can override the return type of functions and methods on a case by case basis.
  (e.g. based on one or more of the argument types or values) (related to #612, #1181)
+ Add a new plugin capability `AnalyzeFunctionCallCapability` which can add logic to analyze calls to a small subset of functions.
  (e.g. based on one or more of the argument types or values) (#1181)
+ Make line numbers more accurate in `DuplicateArrayKeyPlugin`.
+ Add `PregRegexCheckerPlugin` to check for invalid regexes. (uses `AnalyzeFunctionCallCapability`).
  This plugin is able to resolve literals, global constants, and class constants as regexes.
  See [the corresponding section of .phan/plugins/README.md](.phan/plugins/README.md#pregregexcheckerpluginphp)
+ Add `PrintfCheckerPlugin` to check for invalid format strings or incorrect arguments in printf calls. (uses `AnalyzeFunctionCallCapability`)
  This plugin is able to resolve literals, global constants, and class constants as format strings.
  See [the corresponding section of .phan/plugins/README.md](.phan/plugins/README.md#printfcheckerpluginphp)

Bug Fixes
+ Properly check for undeclared classes in arrays within phpdoc `@param`, `@property`, `@method`, `@var`, and `@return` (etc.) types.
  Also, fix a bug in resolving namespaces of generic arrays that are nested 2 or more array levels deep.
+ Fix uncaught TypeError when magic property has the same name as a property. (#1141)
+ Make AlwaysReturnPlugin warn about functions/methods with real nullable return types failing to return a value.
+ Change the behavior of the `-d` flag, make it change the current working directory to the provided directory.
+ Properly set the real param type and return types of internal functions, in rare cases where that exists.
+ Support analyzing the rare case of namespaced internal global functions (e.g. `\ast\parse_code($code, $version)`)
+ Improve analysis of shorthand ternary operator: Remove false/null from cond_expr in `(cond_expr) ?: (false_expr)` (#1186)

0.8.8 Sep 24, 2017
------------------

Switched package name from etsy/phan to phan/phan.

0.8.7 Sep 24, 2017
------------------

Future releases (0.8.8+, 0.9.6+, 0.10.0+, etc.) will be published to phan/phan instead of etsy/phan on packagist.

After a few more patch releases, Phan 0.8.x will switch from AST version 40 (php-ast 0.1.4+)
to AST version 50 (0.1.5+).

New Features(Analysis)
+ Check types of dimensions when using array access syntax (#406, #1093)
  (E.g. for an `array`, check that the array dimension can cast to `int|string`)

New Features (CLI, Configs)
+ Add option `ignore_undeclared_functions_with_known_signatures` which can be set to `false`
  to always warn about global functions Phan has signatures for
  but are unavailable in the current PHP process (and enabled extensions, and the project being analyzed) (#1080)
  The default was/is to not warn, to reduce false positives.
+ Add CLI flag `--use-fallback-parser` (Experimental).  If this flags is provided, then when Phan analyzes a syntactically invalid file,
  it will try again with a parser which tolerates a few types of errors, and analyze the statements that could be parsed.
  Useful in combination with daemon mode.
+ Add `phpdoc_type_mapping` config setting.
  Projects can override this to make Phan ignore or substitute non-standard phpdoc2 types and common typos (#294)
  (E.g. `'phpdoc_type_mapping' => ['the' => '', 'unknown_type' => '', 'number' => 'int|float']`)

Maintenance
+ Increased minimum `ext-ast` version constraint to 0.1.5, switched to AST version 50.
+ Update links to project from github.com/etsy/phan to github.com/phan/phan.
+ Use the native `spl_object_id` function if it is available for the union type implementation.
  This will make phan 10% faster in PHP 7.2.
  (for PHP 7.1, https://github.com/runkit7/runkit_object_id 1.1.0+ also provides a native implementation of `spl_object_id`)
+ Reduce memory usage by around 5% by tracking only the file and lines associated with variables, instead of a full Context object.


Plugins
+ Increased minimum `ext-ast` version constraint to 0.1.5, switched to AST version 50.
  Third party plugins will need to create a different version, Decls were changed into regular Nodes
+ Implement `AnalyzePropertyCapability` and `FinalizeProcessCapability`.
  Make `UnusedSuppressionPlugin` start using `AnalyzePropertyCapability` and `FinalizeProcessCapability`.
  Fix bug where `UnusedSuppressionPlugin` could run before the suppressed issues would be emitted,
  making it falsely emit that suppressions were unused.

Bug Fixes
+ Fix a few incorrect property names for Phan's signatures of internal classes (#1085)
+ Fix bugs in lookup of relative and non-fully qualified class and function names (#1097)
+ Fix a bug affecting analysis of code when `simplify_ast` is true.
+ Fix uncaught NodeException when analyzing complex variables as references (#1116),
  e.g. `function_expecting_reference($$x)`.

0.8.6 Aug 15, 2017
------------------

### Backported from Phan 0.9.4

New Features (Analysis)
+ Check (the first 5) elements of returned arrays against the declared return union types, individually (Issue #935)
  (E.g. `/** @return int[] */ function foo() {return [2, "x"]; }` will now warn with `PhanTypeMismatchReturn` about returning `string[]`)
+ Check both sides of ternary conditionals against the declared return union types
  (E.g. `function foo($x) : int {return is_string($x) ? $x : 0; }` will now warn with `PhanTypeMismatchReturn`
  about returning a string).
+ Improved analysis of negations of conditions within ternary conditional operators and else/else if statements. (Issue #538)
  Support analysis of negation of the `||` operator. (E.g. `if (!(is_string($x) || is_int($x))) {...}`)
+ Make phan aware of blocks of code which will unconditionally throw or return. (Issue #308, #817, #996, #956)

  Don't infer variable types from blocks of code which unconditionally throw or return.

  Infer the negation of type assertions from if statements that unconditionally throw/return/break/continue.
  (E.g. `if (!is_string($x)) { return false; } functionUsingX($x);`)

  When checking if a variable is defined by all branches of an if statement, ignore branches which inconditionally throw/return/break/continue.
+ To reduce the false positives from analysis of the negation of type assertions,
  normalize nullable/boolean union types after analyzing code branches (E.g. if/else) affecting the types of those variables.
  (e.g. convert "bool|false|null" to "?bool")
+ Add a new plugin file `AlwaysReturnPlugin`. (Issue #996)
  This will add a stricter check that a function with a non-null return type *unconditionally* returns a value (or explicitly throws, or exit()s).
  Currently, Phan just checks if a function *may* return, or unconditionally throws.
+ Add a new plugin file `UnreachableCodePlugin` (in development).
  This will warn about statements that appear to be unreachable
  (statements occurring after unconditional return/break/throw/return/exit statements)

New Features (CLI, Configs)
+ Add config setting `prefer_narrowed_phpdoc_return_type` (See "New Features (CLI, Configs)),
  which will use only the phpdoc return types for inferences, if they're narrowed.
  This config is enabled by default, and requires `check_docblock_signature_return_type_match` to be enabled.

Bug Fixes
+ Work around notice about COMPILER_HALT_OFFSET on windows.
+ Fixes #462 : Fix type inferences for instanceof for checks with dynamic class names are provided.
  Valid class names are either a string or an instance of the class to check against.
  Warn if the class name is definitely invalid.
+ Fix false positives about undefined variables in isset()/empty() (Issue #1039)
  (Fixes bug introduced in Phan 0.9.3)
+ Fix false positive warnings about accessing protected methods from traits (Issue #1033)
  Act as though the class which used a trait is the place where the method was defined,
  so that method visibility checks work properly.
  Additionally, fix false positive warnings about visibility of method aliases from traits.
+ Warn about instantiation of class with inaccessible constructor (Issue #1043)
+ Fix rare uncaught exceptions (Various)
+ Make issues and plugin issues on properties consistently use suppressions from the plugin doc comment.

Changes In Emitted Issues
+ Improve `InvalidVariableIssetPlugin`. Change the names and messages for issue types.
  Emit `PhanPluginUndeclaredVariableInIsset` and `PhanPluginComplexVariableIsset`
  instead of `PhanUndeclaredVariable`.
  Stop erroneously warning about valid property fetches and checks of fields of superglobals.

0.8.5 Jul 11, 2017
------------------

### Backported from Phan 0.9.3

New Features (Analysis)
+ Automatically inherit `@param` and `@return` types from parent methods.
  This is controlled by the boolean config `inherit_phpdoc_types`, which is true by default.
  `analyze_signature_compatibility` must also be set to true (default is true) for this step to be performed.
+ Better analysis of calls to parent::__construct(). (Issue #852)
+ Warn with `PhanAccessOwnConstructor` if directly invoking self::__construct or static::__construct in some cases (partial).
+ Start analyzing the inside of for/while loops using the loop's condition (Issue #859)
  (Inferences may leak to outside of those loops. `do{} while(cond)` is not specially analyzed yet)
+ Improve analysis of types in expressions within compound conditions (Issue #847)
  (E.g. `if (is_array($x) && fn_expecting_array($x)) {...}`)
+ Evaluate the third part of a for loop with the context after the inner body is evaluated (Issue #477)
+ Emit `PhanUndeclaredVariableDim` if adding an array field to an undeclared variable. (Issue #841)
  Better analyze `list($var['field']) = values`
+ Improve accuracy of `PhanTypeMismatchDeclaredReturn` (Move the check to after parse phase is finished)
+ Enable `check_docblock_signature_return_type_match` and `check_docblock_signature_param_type_match` by default.
  Improve performance of those checks.
  Switch to checking individual types (of the union type) of the phpdoc types and emitting issues for each invalid part.
+ Create `PhanTypeMismatchDeclaredParam` (Move the check to after parse phase is finished)
  Also add config setting `prefer_narrowed_phpdoc_param_type` (See "New Features (CLI, Configs))
  This config is enabled by default.

  Also create `PhanTypeMismatchDeclaredParamNullable` when params such as `function foo(string $x = null)`
  are documented as the narrowed forms `@param null $x` or `@param string $x`.
  Those should be changed to either `string|null` or `?string`.
+ Detect undeclared return types at point of declaration, and emit `PhanUndeclaredTypeReturnType` (Issue #835)
+ Create `PhanParamSignaturePHPDocMismatch*` issue types, for mismatches between `@method` and real signature/other `@method` tag.
+ Create `PhanAccessWrongInheritanceCategory*` issue types to warn about classes extending a trait/interface instead of class, etc. (#873)
+ Create `PhanExtendsFinalClass*` issue types to warn about classes extending from final classes.
+ Create `PhanAccessOverridesFinalMethod*` issue types to warn about methods overriding final methods.
+ Create `PhanTypeMagicVoidWithReturn` to warn if `void` methods such as `__construct`, `__set`, etc return a value that would be ignored. (Issue #913)
+ Add check for `PhanTypeMissingReturn` within closures. Properly emit `PhanTypeMissingReturn` in functions/methods containing closures. (Issue #599)
+ Improved checking for `PhanUndeclaredVariable` in array keys and conditional conditions. (Issue #912)
+ Improved warnings and inferences about internal function references for functions such as `sort`, `preg_match` (Issue #871, #958)
  Phan is now aware of many internal functions which normally ignore the original values of references passed in (E.g. `preg_match`)
+ Properly when code attempts to access static/non-static properties as if they were non-static/static. (Issue #936)
+ Create `PhanCommentOverrideOnNonOverrideMethod` and `PhanCommentOverrideOnNonOverrideConstant`. (Issue #926)
  These issue types will be emitted if `@override` is part of doc comment of a method or class constant which doesn't override or implement anything.
  (`@Override` and `@phan-override` can also be used as aliases of `@override`. `@override` is not currently part of any phpdoc standard.)
  NOTE: in Phan 0.8, annotations on class constants can't be read from `php-ast`
+ Add `@phan-closure-scope`, which can be used to annotate closure definitions with the namespaced class it will be bound to (Issue #309, #590, #790)
  (E.g. if the intent was that Closure->bindTo or Closure->bind would be called to bind it to `\MyNS\MyClass` (or an instance of that class),
  then a closure could be declared as `/** @phan-closure-scope \MyNS\MyClass */ function() { $this->somePrivateMyClassMethod(); }`
+ Add `Closure` as a first class type, (Previously, closures were treated as `callable` in some places) (Issue #978)

New Features (CLI, Configs)
+ Create `check_docblock_signature_param_type_match` (similar to `check_docblock_signature_return_type_match`) config setting
  to enable warning if phpdoc types are incompatible with the real types. True(enabled) by default.

  Create `prefer_narrowed_phpdoc_param_type` config setting (True by default, requires `check_docblock_signature_return_type_match` to be enabled).
  When it is true, Phan will analyze each function using the phpdoc param types instead of the provided signature types
  if the possible phpdoc types are narrower and compatible with the signature.
  (E.g. indicate that subclasses are expected over base classes, indicate that non-nullable is expected instead of nullable)
  This affects analysis both inside and outside the method.

  Aside: Phan currently defaults to preferring phpdoc type over real return type, and emits `PhanTypeMismatchDeclaredReturn` if the two are incompatible.
+ Create `enable_class_alias_support` config setting (disabled by default), which enables analyzing basic usage of class_alias. (Issue #586)
  Set it to true to enable it.
  NOTE: this is still experimental.
+ Warn to stderr about running Phan analysis with XDebug (Issue #116)
  The warning can be disabled by the Phan config setting `skip_slow_php_options_warning` to true.
+ Add a config setting 'scalar_implicit_partial' to allow moving away from 'scalar_implicit_cast' (Issue #541)
  This allows users to list out (and gradually remove) permitted scalar type casts.
+ Add `null_casts_as_array` and `array_casts_as_null` settings, which can be used while migrating away from `null_casts_as_any_type`.
  These will be checked if one of the types has a union type of `null`, as well as when checking if a nullable array can cast to a regular array.

Plugins

+ Redesign plugin system to be more efficient. (Issue #600)
  New plugins should extend `\Phan\PluginV2` and implement the interfaces for capabilities they need to have,
  such as `\Phan\PluginV2\AnalyzeClassCapability`.
  In the new plugin system, plugins will only be run when they need to (Phan no longer needs to invoke an empty method body).
  Old subclasses of `\Phan\Plugin\PluginImplementation` will continue to work, but will be less efficient.

Maintenance
+ Reduce memory usage by around 15% by using a more efficient representation of union types (PR #729).
  The optional extension https://github.com/runkit7/runkit_object_id can be installed to boost performance by around 10%.
+ Check method signatures compatibility against all overridden methods (e.g. interfaces with the same methods), not just the first ones (Issue #925)

Bug Fixes
+ Work around known bugs in current releases of two PECL extensions (Issue #888, #889)
+ Fix typo - Change `PhanParamSignatureRealMismatch` to `PhanParamSignatureRealMismatchReturnType`
+ Consistently exit with non-zero exit code if there are multiple processes, and any process failed to return valid results. (Issue #868)
+ Fixes #986 : PhanUndeclaredVariable used to fail to be emitted in some deeply nested expressions, such as `return $undefVar . 'suffix';`
+ Make Phan infer the return types of closures, both for closures invoked inline and closures declared then invoked later (Issue #564)
+ Phan now correctly analyze global functions for mismatches of phpdoc types and real parameter types.
  Previously, it wouldn't emit warnings for global functions, only for methods.
+ Don't add `mixed` to inferred union types of properties which already have non-empty phpdoc types. (Issue #512)
  mixed would just result in Phan failing to emit any types of issues.
+ When `simplify_ast` is true, simplify the ASTs parsed in the parse mode as well.
  Makes analysis consistent when `quick_mode` is false (AST nodes from the parse phase would also be used in the analysis phase)
+ Don't emit PhanTypeNonVarPassByRef on arguments that are function/method calls returning references. (Issue #236)
+ Emit PhanContextNotObject more reliably when not in class scope.

Backwards Incompatible Changes
+ Fix categories of some issue types, renumber error ids for the pylint error formatter to be unique and consistent.

0.8.4 Jun 13, 2017
------------------

### Backported from Phan 0.9.2

New Features (Analysis)
+ Add `PhanParamSignatureRealMismatch*` (e.g. `ParamSignatureRealMismatchTooManyRequiredParameters`),
  which ignores phpdoc types and imitates PHP's inheritance warning/error checks as closely as possible. (Issue #374)
  This has a much lower rate of false positives than `PhanParamSignatureMismatch`, which is based on Liskov Substitution Principle and also accounts for phpdoc types.
  (`PhanParamSignatureMismatch` continues to exist)
+ Create `PhanUndeclaredStaticProperty` (Issue #610)
  This is of higher severity than PhanUndeclaredProperty, because PHP 7 throws an Error.
  Also add `PhanAccessPropertyStaticAsNonStatic`
+ Supports magic instance/static `@method` annotations. (Issue #467)
  This is enabled by default.
+ Change the behavior of non-quick recursion (Affects emitted issues in large projects).
  Improve perfomance of non-quick analysis by checking for redundant analysis steps
  (E.g. calls from two different places passing the same union types for each parameter),
  continuing to recurse when passing by reference.
+ Support for checking for misuses of "@internal" annotations. Phan assumes this means it is internal to a namespace. (Issue #353)
  This checks properties, methods, class constants, and classes.
  (Adds `PhanAccessConstantInternal`, `PhanAccessClassInternal`, `PhanAccessClassConstantInternal`, `PhanAccessPropertyInternal`, `PhanAccessMethodInternal`)
  (The implementation may change)
+ Make conditionals such as `is_string` start applying to the condition in ternary operators (`$a ? $b : $c`)
+ Treat `resource`, `object`, and `mixed` as native types only when they occur in phpdoc.
  Outside of phpdoc (e.g. `$x instanceof resource`), analyze those names as if they were class names.
+ Emit low severity issues if Phan can't extract types from phpdoc,
  the phpdoc `@param` is out of sync with the code,
  or if the phpdoc annotation doesn't apply to an element type (Issue #778)
+ Allow inferring the type of variables from `===` conditionals such as `if ($x === true)`
+ Add issue type for non-abstract classes containing abstract methods from itself or its ancestors
  (`PhanClassContainsAbstractMethod`, `PhanClassContainsAbstractMethodInternal`)
+ Partial support for handling trait adaptations (`as`/`insteadof`) when using traits (Issue #312)
+ Start checking if uses of private/protected class methods *defined in a trait* are visible outside of that class.
  Before, Phan would always assume they were visible, to reduce false positives.
+ If Phan has inferred/been provided generic array types for a variable (e.g. `int[]`),
  then analysis of the code within `if (is_array($x))` will act as though the type is `int[]`.
  The checks `is_object` and `is_scalar` now also preserve known sub-types of the group of types.
  (If Phan isn't aware of any sub-types, it will infer the generic version, e.g. `object`)
+ Start checking if unanalyzable variable accesses such as `$$x` are very likely to be invalid or typos (e.g. $x is an object or array or null)
  Emit `PhanTypeSuspiciousIndirectVariable` if those are seen. (PR #809)
+ Add partial support for inferring the union types of the results of expressions such as `$x ^= 5` (e.g. in `foo($x ^= 5)`) (PR #809)
+ Thoroughly analyze the methods declared within traits,
  using only the information available within the trait. (Issue #800, PR #815)
  If new emitted issues are seen, users can (1) add abstract methods to traits, (2) add `@method` annotations, or (3) add `@suppress` annotations.

New Features (CLI, Configs)
+ (Linux/Unix only) Add Experimental Phan Daemon mode (PR #563 for Issue #22), which allows phan to run in the background, and accept TCP requests to analyze single files.
  (The implementation currently requires the `pcntl` extension, which does not in Windows)
  Server usage: `path/to/phan --daemonize-tcp-port 4846` (In the root directory of the project being analyzed)
  Client usage: `path/to/phan_client --daemonize-tcp-port 4846 -l src/file1.php [ -l src/file2.php ]`
+ Add `--color` CLI flag, with rudimentary unix terminal coloring for the plain text output formatter. (Issue #363)
  Color schemes are customizable with `color_scheme`, in the config file.
+ Add the `exclude_file_regex` config to exclude file paths based on a regular expression (e.g. tests or example files mixed with the codebase) (#635)
  The regular expression is run against the relative path within the project.
+ Add `--dump-parsed-file-list` option to print files which Phan would parse.
+ Add experimental `simplify_ast` config, to simplify the AST into a form which improves Phan's type inference.
  (E.g. handles some variable declarations within `if ()` statements.
   Infers that $x is a string for constructs such as `if (!is_string($x)) {return;} function_using_x($x);`)
  This is slow, and disabled by default.
+ Add `--include-analysis-file-list` option to define files that will be included in static analysis, to the exclusion of others.
+ Start emitting `PhanDeprecatedFunctionInternal` if an internal (to PHP) function/method is deprecated.
  (Phan emits `PhanUndeclaredFunction` if a function/method was removed; Functions deprecated in PHP 5.x were removed in 7.0)

Maintenance
+ Update function signature map to analyze `iterable` and `is_iterable` from php 7.1
  (is_iterable is ignored in phan 0.8)
+ Improve type inferences on functions with nullable default values.
+ Update miscellaneous new functions in php 7.1 standard library (e.g. `getenv`)

Bug Fixes
- Fix PhanTypeMismatchArgument, etc. for uses of `new static()`, static::CONST, etc in a method. (Issue #632)
- Fix uncaught exception when conditional node is a scalar (Issue #613)
- Existence of __get() no longer affects analyzing static properties. (Issue #610)
- Phan can now detect the declaration of constants relative to a `use`d namespace (Issue #509)
- Phan can now detect the declaration of functions relative to a `use`d namespace (Issue #510)
- Fix a bug where the JSON output printer accidentally escaped some output ("<"), causing invalid JSON.
- Fix a bug where a print/echo/method call erroneously marked methods/functions as having a return value. (Issue #811)
- Improve analysis of SimpleXMLElement (Issues #542, #539)
- Fix crash handling trait use aliases which change only the method's visibility (Issue #861)

Backwards Incompatible Changes
- Declarations of user-defined constants are now consistently
  analyzed in a case sensitive way.
  This may affect projects using `define(name, value, case_insensitive = true)`.
  Change the code being analyzed to exactly match the constant name in define())

### Backported from Phan 0.9.1

New Features (Analysis)
+ Conditions in `if(cond(A) && expr(A))` (e.g. `instanceof`, `is_string`, etc) now affect analysis of right hand side of `&&` (PR #540)
+ Add `PhanDeprecatedInterface` and `PhanDeprecatedTrait`, similar to `PhanDeprecatedClass`
+ Supports magic `@property` annotations, with aliases `@property-read` and @property-write`. (Issue #386)
  This is enabled by default.
  Also adds `@phan-forbid-undeclared-magic-properties` annotation,
  which will make Phan warn about undeclared properties if no real property or `@property` annotation exists.

New Features (CLI, Configs)
+ Add `--version` CLI flag
+ Move some rare CLI options from `--help` into `--extended-help`

Maintenance
+ Improved stability of analyzing phpdoc and real nullable types (Issue #567)
+ Fix type signatures Phan has for some internal methods.
+ Improve CLI `--progress-bar` tracking by printing 0% immediately.
+ Add Developer Certificate of Origin

Bug Fixes
+ Fix uncaught issue exception analyzing class constants (Issue #551)
+ Fix group use in ASTs
+ Fix false positives checking if native types can cast to/from nullable native types (Issue #567, #582)
+ Exit with non-zero exit code if an invalid CLI argument is passed to Phan

Backwards Incompatible Changes
+ Change the way that parameter's default values affect type inferences.
  (May now add to the union type or ignore default values. Used to always add the default value types)
  Add `@param` types if you encounter new issues.
  This was done to avoid false positives in cases such as `function foo($maybeArray = false)`
+ Increase minimum `ext-ast` version constraint to 0.1.4

### Backported from Phan 0.9.0

The 0.9.x versions will be tracking syntax from PHP versions 7.1.x and is runnable on PHP 7.1+.
Please use version 0.8.x if you're using a version of PHP < 7.1.

New Features (Analysis)
+ Support php 7.1 class constant visibility
+ Support variadic phpdoc in `@param`, e.g. `@param string ...$args`
  Avoid ambiguity by emitting `PhanTypeMismatchVariadicComment` and `PhanTypeMismatchVariadicParam`.
+ Initial support for php 7.1 nullable types and void, both in phpdoc and real parameters.
+ Initial support for php 7.1 `iterable` type
+ Both conditions from `if(cond(A) && cond(B))` (e.g. `instanceof`, `is_string`, etc.) now affect analysis of the if element's block (PR #540)
+ Apply conditionals such as `is_string` to type guards in ternary operators (Issue #465)
+ Allow certain checks for removing null from Phan's inferred types, reducing false positives (E.g. `if(!is_null($x) && $x->method())`) (#518)
+ Incomplete support for specifying the class scope in which a closure will be used/bound  (#309)
+ Support `@return self` in class context

New Features (CLI, Configs)
+ Introduce `check_docblock_signature_return_type_match` config (slow, disabled by default)
  (Checks if the phpdoc types match up with declared return types)

Maintenance
+ Add Code of Conduct
+ Fix type signatures for some internal methods and internal class properties.

Bug Fixes
+ Allow asserting `object` is a specific object type without warning (Issue #516)
+ Fix bugs in analysis of varargs within a function(Issue #516)
+ Treat null defaults in functions and methods the same way (Issue #508)
  In both, add null defaults to the UnionType only if there's already another type.
  In both, add non-null defaults to the UnionType (Contains `mixed` if there weren't any explicit types)
+ Specially handle phpdoc type aliases such as `boolean` only in phpdoc (Issue #471)
  (Outside of phpdoc, it refers to a class with the name `boolean`)
+ Add some internal classes other than `stdClass` which are allowed to have dynamic, undeclared properties (Issue #433)
+ Fix assertion errors when passing references by reference (Issue #500)

Backwards Incompatible Changes
+ Requires newer `ext-ast` version (Must support version 35).

0.8.3 Jan 26, 2017
------------------

The 0.8.x versions will be tracking syntax from PHP versions 7.0.x and is runnable on PHP 7.0+.
Please use version 0.8.x if you're using a version of PHP < 7.1.
For best results, run version 0.8.x with PHP 7.0 if you are analyzing a codebase which normally runs on php <= 7.0
(If php 7.1 is used, Phan will think that some new classes, methods, and functions exist or have different parameter lists because it gets this info from `Reflection`)

???