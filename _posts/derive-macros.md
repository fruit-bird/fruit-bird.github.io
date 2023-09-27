---
title: Just What are Derive Macros?
date: 2023-09-27 9:30:00 -495
categories: [rust]
tags: [rust,macros]
---

This procedural macro type is what allows for the `#[derive(Debug, Copy)]` syntax to implement the `Debug` and `Copy` traits on structs, enums, and unions

To make a derive macro, we need to create a function annotated with the `#[proc_macro_derive(Trait)]` ^[Only functions that have this attribute can be exported from the crate] attribute. This function is what will be called with we annotate a type with `#[derive(Trait)]`

## Example: Deriving the `IntoMap` Trait
We want to create a derive macro to simplify the implementation of the `IntoMap` trait. Our macro will only be valid when derived from a struct with named fields

This trait has one member function `into_map`, that "serializes" the struct and maps it to a `BTreeMap`. We define it as follows:
```rust
pub trait IntoMap {
    fn into_map(&self) -> BTreeMap<String, String>;
}
```

We start by declaring our function with the predefined derive signature: `TokenStream` as input, `TokenStream` as output
```rust
#[proc_macro_derive(IntoMap)]
pub fn into_map_derive(input: TokenStream) -> TokenStream {}
```

We then use `parse_macro_input!` from the [`syn` crate](https://crates.io/crates/syn) to parse the `input` into a `DeriveInput`, and extract the struct's name (identifier)

```rust
let parsed_input = parse_macro_input!(input as DeriveInput);
let struct_name = parsed_input.ident;
```

We perform (deeply) nested pattern matching to validate our logical choice to only parse structs with named fields. We invalidate everything else by panicking

```rust
match parsed_input.data {
	Data::Struct(DataStruct {
		fields: Fields::Named(FieldsNamed { ref named, .. }),
		..
	}) => todo!(),
	_ => panic!("#[derive(IntoMap)] works in structs w/ named fields"),
};
```

`named` is of type `Punctuated<Field, Comma>`, which implements the `Iterator` trait. We now want to return an iterator over `TokenStream`, where each iteration contains the generated code for inserting each field in `named` into a `BTreeMap`

The variables `map` and `self` do not exist within the current context, which might seem like we are making a non-hygienic macro. That is not the case, as we are just creating a `TokenStream` for now. These tokens will be passed to a context which does have `map` and `self`

We obtain our lazy iterator of type `impl Iterator<Item = TokenStream>`

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
		fn into_map(&self) -> BTreeMap<String, String> {
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
pub fn into_map_derive(input: TokenStream) -> TokenStream {}
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
pub fn into_map_derive(input: TokenStream) -> TokenStream {}
```

The way to do it is slightly more complex, but it only requires changing the predicate inside our `all` call. If the field's attribute is `intomap`[^path], we parse its arguments as an identifier, and match on the result such that `all` returns true only if all of the attribute arguments do not match `ignore`.

 [^path]: `.path()` returns the path that identifies the interpretation of an attribute, which is `test` in `#[test]`, `derive` in `#[derive(Copy)]`, and `path` in `#[path = "sys/windows.rs"]`

```rust
field.attrs.iter().all(|attr| {
	if attr.path().is_ident(ATTR_INTOMAP) {
		match attr.parse_args::<Ident>() {
			Ok(ident) => ident != IDENT_IGNORE,
			Err(_) => true,
		}
	} else {
		true
	}
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
fn field_rename(field: &Field) -> Option<Ident> {}
```

We then iterate over the field's attributes, and use a `find_map` to return the first `Some` instance of the passed closure. Our closure consists of validating that the attribute's path [^path] is `intomap`, parsing its arguments as an assignment expression because our syntax is expression-like.  Any diverging path returns a `None`

```rust
field.attrs.iter().find_map(|attr| {
	if attr.path().is_ident(ATTR_INTOMAP) {
		attr.parse_args::<ExprAssign>().ok().and_then(|expr| {
			todo!()
		})
	} else {
		None
	}
})
```

Finally, we match the left and right sides of the expression, validating that the left side is `Ident(rename)`, and converting the string literal on the right side to an identifier.

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