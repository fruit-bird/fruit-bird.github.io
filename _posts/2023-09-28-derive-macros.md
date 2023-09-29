---
title: Just How do Derive Macros Work?
date: 2023-09-28 13:11:00 +0100
pin: true
categories: [Rust]
tags: [rust, macros]
---

This procedural macro type is what allows for the `#[derive(Debug, Copy)]` syntax to implement the `Debug` and `Copy` traits on structs, enums, and unions

To make a derive macro, we need to create a function annotated with the `#[proc_macro_derive(Trait)]` [^proc-macro] attribute. This function is what will be called with we annotate a type with `#[derive(Trait)]`

[^proc-macro]: Only functions that have this attribute can be exported from the crate

## Example: Deriving the `IntoMap` Trait
We want to create a derive macro to simplify the implementation of the `IntoMap` trait. Our macro will only be valid when derived from a struct with named fields

This trait has one member function `into_map`, that "serializes" the struct and maps it to a `BTreeMap`. We define it as follows:
```rust
use std::collections::BTreeMap;

pub trait IntoMap {
    fn as_map(&self) -> BTreeMap<String, String>;
}
```

We start by declaring our function with the predefined derive signature: `TokenStream` as input, `TokenStream` as output
```rust
#[proc_macro_derive(IntoMap)]
pub fn intomap_derive(input: TokenStream) -> TokenStream {
    todo!()
}
```

> ## A Small Intermission
> In order to leverage Rust's elegant error handling, and because of the fixed derive function signature, we typically export a macro's implementation to another function that returns a `syn::Result<TokenStream2>`, where `TokenStream2` is an aliased `proc_macro2::TokenStream`
> 
> ```rust
> mod expand {
>     pub fn intomap_impl(parsed_input: DeriveInput) -> Result<TokenStream2> {
>         todo!()
>     }
> }
> ```
> We use `parse_macro_input!` from the [syn crate](https://crates.io/crates/syn) to parse the `input` into a `DeriveInput`, and extract the struct's name (identifier) 
> 
> We will then call this in the main derive function, and convert any `Err` into a compile error. The `.into()` is to convert from `TokenStream2` into `TokenStream`
>
> ```rust
> #[proc_macro_derive(IntoMap)]
> pub fn intomap_derive(input: TokenStream) -> TokenStream {
>     let parsed_input = parse_macro_input!(input as DeriveInput);
>     expand::intomap_impl(parsed_input)
>         .unwrap_or_else(Error::into_compile_error)
>         .into()
> }
> ```
{: .prompt-tip }

Now, back in `expand::intomap_impl`, We extract the struct's name, which we will need later. We then perform nested pattern matching to parse only structs with named fields. We invalidate everything else by returning an error


```rust
let struct_name = parsed_input.ident;
match parsed_input.data {
    Data::Struct(DataStruct {
        fields: Fields::Named(FieldsNamed { ref named, .. }),
        ..
    }) => todo!(),
    _ => {
        return Err(Error::new(
            Span::call_site(),
            "#[derive(IntoMap)] is only valid for structs with named fields",
        ))
    }
};
```

`named` is of type `Punctuated<Field, Comma>`, which implements the `Iterator` trait. We now want to return an iterator over `TokenStream`, where each iteration contains the generated code for inserting each field in `named` into a `BTreeMap`

The variables `map` and `self` do not exist within the current context, which might seem like we are making a non-hygienic macro. That is not the case, as we are just creating a `TokenStream` for now. These tokens will be passed to a context which does have `map` and `self`

We obtain our lazy iterator of tokens `impl Iterator<Item = TokenStream>`

```rust
let insert_tokens = named.iter().map(|field| {
    let field_name = field.ident.clone().unwrap();
    quote! {
        map.insert(
            stringify!(#field_name).to_string(),
            self.#field_name.to_string()
        );
    }
}
```

Finally, we create the `TokenStream` that our function will be returning. We first bring the necessary types into scope, then implement `IntoMap` for our struct, whose name (identifier) we extracted earlier

Inside `into_map`, we find ourselves in the previously mentioned context that has both `map` and `self`. We then consume the iterator to add our tokens that perform the insertions using a syntax similar to that of declarative macro repetitions, courtesy of `quote!`

```rust
let tokens = quote! {
    use std::collections::BTreeMap;
    use into_map::IntoMap;

    impl IntoMap for #struct_name {
        fn as_map(&self) -> BTreeMap<String, String> {
            let mut map = BTreeMap::new();
            #(#insert_tokens)*
            map
        }
    }
};
```

### An Attribute to Ignore Fields
We want to extend the capability of our macro by adding the option to ignore some struct fields when converting it to a `BTreeMap`. We will do this by tagging the fields with the `#[intomap_ignore]` attribute. To do so, we first have to define the attribute in the function definition

```rust
const ATTR_IGNORE: &str = "intomap_ignore";

#[proc_macro_derive(IntoMap, attributes(intomap_ignore))]
pub fn intomap_derive(input: TokenStream) -> TokenStream {
    todo!()
}
```

We then will modify how we obtain `insert_tokens` so that it skips the fields tagged with our new attribute. We will replace our `map` over the struct fields with a `filter_map`, and only return `Some(TokenStream)` if the field is not tagged with `#[intomap_ignore]`

We do that by iterating over the attributes of each field, validating that `all` of its attributes are not `"intomap_ignore"`, then returning the insert tokens wrapped in a `Some` if the condition is valid

```rust
let insert_tokens = named.iter().filter_map(|field| {
    let field_name = field.ident.clone().unwrap();
    field
        .attrs
        .iter()
        .all(|attr| !attr.path().is_ident(ATTR_IGNORE))
        .then_some(quote! {
            map.insert(
                stringify!(#field_name).to_string(),
                self.#field_name.to_string()
            );
        })
})
```

### A Better Ignore
We previously used `#[intomap_ignore]` as `#[ignore]` already exists (for ignoring a test) and will cause conflict if we give the same name to our attribute. But `#[intomap_ignore]` is ugly and if we want to add other attributes, we'd have to follow this style of `#[intomap_...]`

A more elegant attribute would look like `#[intomap(ignore)]`. This way, we can pseudo-namespace all of our attributes

```rust
const ATTR_INTOMAP: &str = "intomap";
const IDENT_IGNORE: &str = "ignore";

#[proc_macro_derive(IntoMap, attributes(intomap))]
pub fn intomap_derive(input: TokenStream) -> TokenStream {}
```

The way to do it is slightly more complex, but it only requires changing the predicate inside our `all` call. If the field's attribute is `intomap`[^path], we parse its arguments as an identifier, and match on the result such that `all` returns true only if all of the attribute arguments do not match `ignore`.

[^path]: `.path()` returns the path that identifies the interpretation of an attribute, which is `test` in `#[test]`, `derive` in `#[derive(Copy)]`, and `path` in `#[path = "sys/windows.rs"]`

```rust
field
    .attrs
    .iter()
    .filter(|attr| attr.path().is_ident(ATTR_INTOMAP))
    .all(|attr| match attr.parse_args::<Ident>() {
        Ok(ident) => ident != IDENT_IGNORE,
        Err(_) => true,
    })
    .then_some(quote! {
        map.insert(
            stringify!(#field_name).to_string(),
            self.#field_name.to_string()
        );
    })
```

This way, we have a prettier attribute that allows for later consistency with our library. We can even add later attributes that will make more sense with this structure like `#[intomap(rename = "new_name")]`. Let's do that next

### An Attribute to Rename Fields

All that we will need to change is the key name when inserting into the map

```rust
let field_rename = todo!();
quote! {
    map.insert(
        stringify!(#field_rename).to_string(),
        self.#field_name.to_string()
    );
})
```

Now to parse our rename syntax. Let's export the functionality to a new function. Our derive function is starting to get long. We want to take a field as input, go through its attributes, and return `Ident(new_name)` if there is a rename attribute

```rust
fn field_rename(field: &Field) -> Option<Ident> {
    todo!()
}
```

We then iterate over the field's attributes, filter out the attributes whose paths [^path] aren't `intomap`, and use a `find_map` to return the first instance of a renaming. We then parse its arguments as an assignment expression because our rename syntax is expression-like

```rust
field
    .attrs
    .iter()
    .filter(|attr| attr.path().is_ident(ATTR_INTOMAP))
    .find_map(|attr| {
        attr.parse_args::<ExprAssign>().ok().and_then(|expr| {
            todo!()
        })
    })
```

Finally, we match the left and right sides of the expression, validating that the left side is the `rename` identifier, and converting the string literal on the right side to an identifier.

```rust
match (*expr.left, *expr.right) {
    (
        Expr::Path(ExprPath { path, .. }),
        Expr::Lit(ExprLit { lit: Lit::Str(s), .. }),
    ) if path.is_ident(IDENT_RENAME) => {
        Some(Ident::new(s.value().as_str(), s.span()))
    }
    _ => None,
}
```

Now to use this function, we call it, and default to the original field name in case of a `None`

```rust
let field_rename = field_rename(&field).unwrap_or(field_name.clone());
```

We're done! Derive macros are one Rust's greatest features, and understanding them lifts the veil of complexity. No more magic, or rather, you are now a magician. All of the code is available in [this repo](https://github.com/fruit-bird/intomap)
