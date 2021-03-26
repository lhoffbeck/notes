# Summary

When components require that a consumer pass a pluralized version of a string, Polaris currently restricts the consumer to a English-centric pluralization format. This isn't very inclusive and can reduce merchant trust in our UI for non-English locales.

Existing issues:

- [polaris-react #4031](https://github.com/Shopify/polaris-react/issues/4031)
- [polaris-react #1786](https://github.com/Shopify/polaris-react/issues/1786)
- [google-shopping #7292](https://github.com/Shopify/google-shopping/issues/7292)

## Goal
Hopefully this RFC helps build context, but I'm planning to keep this pretty surface-level for now--just wanted to get a gut check on approach and see if this is something that has legs.

## So... what's the issue?

We can look at `IndexTable` as a specific example. The component expects 2 `resourceName` values:

```tsx
// index-provider/context.ts

export interface IndexContextType {
 ...
  resourceName: {
    singular: string;
    plural: string;
  };
  ...
}
```

The consumer then provides this value when using the component:
```tsx
<IndexTable
  ...
  resourceName={{
    singular: 'Cactus',
    plural: 'Cactusses',
  }}
  ...
/>
```


Within `IndexTable`, these values are then used in a few contexts:

**(1) Singular:** refers to 1 of the resource item
```tsx
// IndexTable/.../Checkbox.tsx:42

<PolarisCheckbox
  id={itemId}
  label={i18n.translate('Polaris.IndexTable.selectItem', {
    resourceName: resourceName.singular,
  })}
  labelHidden
  checked={selected}
/>
```

**(2) Specific plural:** associated with an item count _THIS IS POTENTIALLY WRONG FOR NON-ENGLISH LOCALES_
```tsx
// IndexTable.tsx:676

function getPaginatedSelectAllAction() {
  ...

  const actionText =
    selectedItemsCount === SELECT_ALL_ITEMS
      ? i18n.translate('Polaris.IndexTable.undo')
      : i18n.translate('Polaris.IndexTable.selectAllItems', {
          itemsLength: itemCount,
          resourceNamePlural: resourceName.plural.toLocaleLowerCase(),
        });

  return {
    content: actionText,
    onAction: handleSelectAllItemsInStore,
  };
}

```

**(3) Generic plural:** not associated with any item count
```tsx
// IndexTable.tsx:364

...
<div className={styles.LoadingPanel}>
  <div className={styles.LoadingPanelRow}>
    <Spinner size="small" />
    <span className={styles.LoadingPanelText}>
      {i18n.translate(
        'Polaris.IndexTable.resourceLoadingAccessibilityLabel',
        {
          resourceNamePlural: resourceName.plural.toLocaleLowerCase(),
        },
      )}
    </span>
  </div>
</div>
...
```

### The Issue
(stolen from @movermeyer's great [polaris-react issue #4031](https://github.com/Shopify/polaris-react/issues/4031))

> Other languages use different pluralization forms depending on very complicated rules.
  Hard-coding [`singular` and `plural` as the options for `ResourceList`](https://polaris.shopify.com/components/lists-and-tables/resource-list#prop-resourceName:~:text=prominence-,resourceName) hard-codes the English pluralization rules, and means that it is incorrect for any language other than English.

While we only handle 2 pluralization cases, the [Unicode CLDR rules](http://cldr.unicode.org/index/cldr-spec/plural-rules) give us 6 different cases we'd need to account for in order to handle this correctly for all locales. This [Pluralization for JavaScript](https://alistapart.com/article/pluralization-for-javascript/) article is a fantastic resource for further reading, but the relevant information for our case is: 

> CLDR defines up to six different plural forms. Each form is assigned a name: `zero`, `one`, `two`, `few`, `many`, or `other`. Not all locales need every form; remember, English only has two: one and other. The name of each form is based on its corresponding plural rule. Here is a CLDR example for the Polish language—a slightly altered version of our earlier counter rules:
> - If the counter has the integer value of 1, use the plural form `one`.
> - If the counter has a value that ends in 2–4, excluding 12–14, use the plural form `few`.
> - If the counter is not 1 and has a value that ends in either 0 or 1, or the counter ends in 5–9, or the counter ends in 12–14, use the plural form `many`.
> - If the counter has any other value than the above, use the plural form `other`.

Effectively, our copy may be completely wrong in other locales. As an example under our current implementation, in Arabic we may have:

```
- 1 book: كتاب  (👍 singular, we handle this)
- 100 books: ١٠٠ (👍 plural, we handle this)
- 0 books: ٠ كتاب (🛑 we don't handle this)
- 3 books: ٣ كتب  (🛑 we show ١٠٠)
- 11 books: ١١ كتابًا  (🛑 we show ١٠٠)
```

### Issue Scope
Other polaris components that may have this same issue are:

- [IndexProvider](https://github.com/Shopify/polaris-react/blob/main/src/utilities/index-provider/context.ts#L8-L11)
- [DataTable](https://github.com/Shopify/polaris-react/blob/main/src/components/DataTable/DataTable.tsx#L34-L37)
- [ResourceList](https://github.com/Shopify/polaris-react/blob/main/src/components/ResourceList/ResourceList.tsx#L76-L79)
- [FilterCreator](https://github.com/Shopify/polaris-react/blob/main/src/components/ResourceList/components/FilterControl/components/FilterCreator/FilterCreator.tsx#L15-L18)


## Approaches
None of these go too deep, but wanted to suggest a few alternate approaches.

Note that Approach 1 & 2 require (1) that we add a new JS package that handles CLDR logic, and (2) that the user add the current locale to the Polaris scope, probably via `<AppProvider>` (my understanding is that we don't currently keep this state in polaris anywhere). However, I think this might be okay for a few reasons:
- We're beginning to see the need to be able to localize within Polaris beyond the ability to pick `en.json`, `de.json`, etc. Web issues [#24886](https://github.com/Shopify/web/issues/24886) and [#24887](https://github.com/Shopify/web/issues/24887) are examples of how our currently local-less state are hurting us. As we continue to add additional languages and locales, this issue will only become more pronounced. In order to fix this we will need to add some CLDR tooling.
- We can make any of the approaches opt-in instead of breaking changes, so our consumers that need this behavior can use it ASAP but we have a gradual on-ramp for the consumers who aren't affected.

Approach 3 is slightly more painful for consumers to implement, but doesn't require us to make any major changes in the current behavior of Polaris.
- Note that this leaves us in a position where we still need to figure out how to handle plurals in Polaris-sourced translation strings ([#24886](https://github.com/Shopify/web/issues/24886) and [#24887](https://github.com/Shopify/web/issues/24887))


### Approach 1: MessageFormat + client passes a single format string
This approach uses the `MessageFormat` [utility from the OpenJS Foundation](http://messageformat.github.io/messageformat/) to parse a formatstring based on locale and a provided count. Message format uses the [ICU message format standard](http://userguide.icu-project.org/formatparse/messages) to provide pluralization rules in a single format string.

##### Prerequisites
- Include the `@messageformat/core` package 
- Build a utility that takes a format string and a count variable and determines the correct plural based on locale 
- 🛑 Requires us to have a way of programmatically determining the current locale (using `<IndexProvider>` could be a good solution)  

This is how this could work in our IndexTable example:

```tsx
// index-provider/context.ts

export interface IndexContextType {
 ...
  // This format string should use the [ICU message format standard](http://userguide.icu-project.org/formatparse/messages)
  resourceNameFormatString: string;
  ...
}
```

The consumer provides this value when using the component:
```tsx
<IndexTable
  ...
  resourceNameFormatString='{COUNT, plural, one {Cactus} few {Cactii} other {Cactusses}}'
  ...
/>
```

Usage then looks like:

**(1) Singular:** refers to 1 of the resource item
```tsx
// IndexTable/.../Checkbox.tsx:42

<PolarisCheckbox
  ...
  label={pluralize(resourceNameFormatString, 1)}
  ...
/>
```

**(2) Specific plural:** associated with an item count _THIS IS POTENTIALLY WRONG FOR NON-ENGLISH LOCALES_
```tsx
// IndexTable.tsx:676

function getPaginatedSelectAllAction() {
  ...

  const actionText =
    selectedItemsCount === SELECT_ALL_ITEMS
      ? i18n.translate('Polaris.IndexTable.undo')
      : i18n.translate('Polaris.IndexTable.selectAllItems', {
          resourceNamePlural: pluralize(resourceNameFormatString, itemCount),
        });

  ...
}

```

**(3) Generic plural:** not associated with any item count
```tsx
// IndexTable.tsx:364

...
<div className={styles.LoadingPanel}>
  ...
    {i18n.translate(
      'Polaris.IndexTable.resourceLoadingAccessibilityLabel',
      {
        resourceNamePlural: pluralize(resourceNameFormatString, PAGE_SIZE),
      },
    )}
  ...
</div>
...
```

**Pros:**
- This is fairly trivial for the client to implement: they need to provide a locale and 1 format string prop
- This isn't too bad for us to implement either. Adding a new localization package increases the bundle size BUT it may help us future-proof other pluralization issues.

**Cons:**
- Adding a package increases the bundle size
- Client has to provide a locale; means that Polaris can't be locale-agnostic anymore
- Requires the client to provide a properly formatted ICU message format string

### Approach 2: Use Globalize to determine correct plural format
This approach uses the `pluralGenerator` [utility from the GlobalizeJS package](https://github.com/globalizejs/globalize/blob/master/doc/api/plural/plural-generator.md) to programmatically determine which plural form (`zero`, `one`, `two`, `few`, `many`, or `other`) should be used based on a count and the current locale.

##### Prerequisites
- Include the `globalize` and `globalize/plural` packages
- Wrap globalize in a util that takes locale and count and spits out a CLDR plural rule (`one`, `two`, ...) 
- 🛑 Requires us to have a way of programmatically determining the current locale (using `<IndexProvider>` could be a good solution)  

This is how this could work in our IndexTable example:

```tsx
// index-provider/context.ts

export interface IndexContextType {
 ...
    resourceName: {
      one: string;
      other: string;
      zero?: string;
      two?: string;
      few?: string;
      many?: string;
    };
  ...
}
```

The consumer provides this value when using the component:
```tsx
<IndexTable
  ...
  resourceName={{
    one: 'Cactus',
    few: 'Cactii',
    other: 'Cactusses',
  }}
  ...
/>
```

Usage then looks like:

**(1) Singular:** refers to 1 of the resource item
```tsx
// IndexTable/.../Checkbox.tsx:42

<PolarisCheckbox
  ...
    label={i18n.translate('Polaris.IndexTable.selectItem', {
      resourceName: resourceName[getPluralizationType(1)], // (alternately, getPluralization(resourceMap, count))
    })} 
  ...
/>
```

**(2) Specific plural:** associated with an item count _THIS IS POTENTIALLY WRONG FOR NON-ENGLISH LOCALES_
```tsx
// IndexTable.tsx:676

function getPaginatedSelectAllAction() {
  ...

  const actionText =
    selectedItemsCount === SELECT_ALL_ITEMS
      ? i18n.translate('Polaris.IndexTable.undo')
      : i18n.translate('Polaris.IndexTable.selectAllItems', {
          resourceNamePlural: resourceName[getPluralizationType(itemCount)] || resourceName.other,
        });
  ...
}

```

**(3) Generic plural:** not associated with any item count
```tsx
// IndexTable.tsx:364

...
<div className={styles.LoadingPanel}>
  ...
    {i18n.translate(
      'Polaris.IndexTable.resourceLoadingAccessibilityLabel',
      {
        resourceNamePlural: resourceName[getPluralizationType(PAGE_SIZE)] || resourceName.other,
      },
    )}
  ...
</div>
...
```

**Pros:**
- This keeps (roughly) the same API shape as the consumer is used to currently, fairly easy for the consumer to implement

**Cons:**
- Adding a package increases the bundle size
- Client has to provide a locale; means that Polaris can't be locale-agnostic anymore

### Approach 3: Pluralize via injected callback
In this approach, the client passes a callback function that accepts a count variable and returns the correct translation. 

##### Prerequisites (none)

This is how this could work in our IndexTable example:

```tsx
// index-provider/context.ts

export interface IndexContextType {
  ...
  pluralizeResourceName(count: number): string;
  ...
}
```

The consumer provides this value when using the component:
```tsx
<IndexTable
  ...
  pluralizeResourceName={(count: number) => {
    // IF the consumer is rolling translations on the fly:
    if (count === 1) {
      return "Cactus";
    } else if (count === 2) {
      return "Cactii";
    } else {
      return "Cactusses";
    }

    // ELSE IF a robust i18n system is present (like the `react-i18n` package that @shopify/web uses)
    return i18n.translate('cactus', {count});
  }}
  ...
/>
```

Usage then looks like:

**(1) Singular:** refers to 1 of the resource item
```tsx
// IndexTable/.../Checkbox.tsx:42

<PolarisCheckbox
  ...
    label={i18n.translate('Polaris.IndexTable.selectItem', {
      resourceName: pluralizeResourceName(1)
    })} 
  ...
/>
```

**(2) Specific plural:** associated with an item count _THIS IS POTENTIALLY WRONG FOR NON-ENGLISH LOCALES_
```tsx
// IndexTable.tsx:676

function getPaginatedSelectAllAction() {
  ...

  const actionText =
    selectedItemsCount === SELECT_ALL_ITEMS
      ? i18n.translate('Polaris.IndexTable.undo')
      : i18n.translate('Polaris.IndexTable.selectAllItems', {
          resourceNamePlural: pluralizeResourceName(itemCount),
        });

  ...
}

```

**(3) Generic plural:** not associated with any item count
```tsx
// IndexTable.tsx:364

...
<div className={styles.LoadingPanel}>
  ...
    {i18n.translate(
      'Polaris.IndexTable.resourceLoadingAccessibilityLabel',
      {
        resourceNamePlural: pluralizeResourceName(PAGE_SIZE),
      },
    )}
  ...
</div>
...
```

**Pros:**
- Polaris can remain blissfully unaware of locale
- Polaris doesn't have to add any translation/l10n packages
- Very minimal work to implement on the Polaris side

**Cons:**
- Depending on how their translations are set up, this may be more painful for a client to implement
- This doesn't fix the fact that we don't handle pluralization properly within Polaris (see again [#24886](https://github.com/Shopify/web/issues/24886) and [#24887](https://github.com/Shopify/web/issues/24887))


## Conclusion
My hope is that these findings take work off someone else's plate :) My personal feeling is that to fix current issues and as we support more locales/languages and as Polaris grows, we're going to need to make the system locale-aware. Given that, my feeling is that of the 3 approaches, option 1 or 2 may be a better long-term fit.

