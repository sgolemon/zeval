# EXPERIMENTAL PROJECT - NOT TESTED OR NECESSARILY SUITABLE FOR USE (YET)

# ZEval

Reusable wrapper header for writing PHP extensions that will build/run against PHP5 and PHP7 APIs.

The chief goal of this portability header is to deal with the change in how the Zend Engine passes around the zval.  In PHP5, zvals tend to be passed/stored at one level of indirection (e.g. `zval*`).  By contrast, in PHP7, zvals tend to be passed as immediates (`zval` with no pointers).  This means that temporaries no longer need to be allocated (but they still need to be dtor'd properly), accesses need to use (or not use) the `_P` suffix, and initialization macros need to (not) take the value by reference.

At the core of this portability layer is the `zeval`, or "Zend Engine Value", which is defined as `zval` for PHP7, and `zval*` for PHP5.  The ZEVAL family of macros can then be used to normalize this type to a fixed level of indirection, or the `zeval` can be directly referenced for APIs which also shifted their level of indirection between versions.

## Boolean types

Since PHP5 uses a general `IS_BOOL` type while PHP7 uses `IS_TRUE` and `IS_FALSE`, the `ZE_TYPE` macro for PHP5 decomposes to `IS_TRUE`/`IS_FALSE` defined by this wrapper for ZE2. 

## String duplication flag

In PHP5, operations such as `ZVAL_STRING` and `add_assoc_string` allow you to pre-allocate the string (using `emalloc`) then attach it to the variable or array specifying `0` for the dup parameter.  In PHP7, basic C strings are always duplicated into a `zend_string` before being attached.  Rather than try to replicate the `zend_string` structure (and confuse users into thinking it can be used with zpp), zeval takes the position of assuming you always want to duplicate the string in the macro/function.  For more complex logic, extensions should use custom ifdefs.

To make calling these common functions easier, the `ZEVAL_DUP_CC` macro is included and defined as `, 1` for PHP5, or blank for PHP7.  This way, argument layout matches for both major versions.

## Optional arguments

In PHP5, an optional argument is often initialized as `zval *arg = NULL;` because zpp will not overwrite the NULL on not receiving an arg.  PHP7 has us use an immediate `zval`, so we solve this problem by initializing it to the type `IS_UNDEF` and testing for whether or not it's been materialized as a concrete type.  To make this version agnostic, zeval introduces two macros: `ZEVAL_UNINIT(v)` for either setting the pointer to `NULL` or the type to `IS_UNDEF` as appropriate, and `ZEVAL_ISDEF(v)` for testing those respective conditions.

# Examples

## Receiving arguments

```
PHP_FUNCTION(foo) {
  zeval arg;
  zeval arr;

  ZEVAL_UNINIT(arr);

  if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "z|a", &arg, &arr) == FAILIRE) { return; }

  if (ZE_TYPE(arg) == IS_STRING) {
    php_printf("arg == %s\n", ZE_STRVAL(arg));
  } else if (ZE_TYPE(arg) == IS_TRUE) {
    php_printf("arg is true\n");
  }

  if (ZEVAL_ISDEF(arr)) {
    zeval val;

    if (zeval_symtable_find(ZE_ARRVAL(arr), "foo", sizeof("foo"), &val)) {
      // Use val
    }
  }
}
```

## Creating Locals and calling Zend APIs

In this example, some of the APIs remain `zval*` in both versions of the Zend Engine, so we use `P_ZEVAL(v)` to normalize `zeval` into `zval*` during the call.  `zval_ptr_dtor()`, however, expects a `zval**` for PHP5, and a `zval*` for PHP7, so we simply take the `zeval` by reference without normalization.

```
{
  zeval arr;
  MAKE_STD_ZEVAL(arr); // Does nothing in PHP7

  array_init(P_ZEVAL(arr));
  add_next_index_long(P_ZEVAL(arr), 42);
  add_next_index_string(P_ZEVAL(arr), "foo" ZEVAL_DUP_CC);

  zval_ptr_dtor(&arr);
}
```
