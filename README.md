# WordPress Search Diagnostics

A practical troubleshooting guide for diagnosing and resolving WordPress search functionality problems.

This repository covers search query behavior, missing results, custom post types, search columns, query filters, search relevance, pagination, and common search configuration issues.

## Table of Contents

- [Overview](#overview)
- [Common Search Problems](#common-search-problems)
- [Requirements](#requirements)
- [1. Understand WordPress Search](#1-understand-wordpress-search)
- [2. Test a Basic Search Query](#2-test-a-basic-search-query)
- [3. Diagnose Missing Search Results](#3-diagnose-missing-search-results)
- [4. Check Searchable Post Types](#4-check-searchable-post-types)
- [5. Inspect Search Columns](#5-inspect-search-columns)
- [6. Diagnose Search Filters](#6-diagnose-search-filters)
- [7. Check Search Exclusion Rules](#7-check-search-exclusion-rules)
- [8. Troubleshoot Search Pagination](#8-troubleshoot-search-pagination)
- [9. Diagnose Search Relevance](#9-diagnose-search-relevance)
- [10. Inspect Search Query Variables](#10-inspect-search-query-variables)
- [11. Troubleshoot Custom Field Search](#11-troubleshoot-custom-field-search)
- [12. Check Search Template Loading](#12-check-search-template-loading)
- [13. Debug Search SQL](#13-debug-search-sql)
- [Useful Diagnostic Examples](#useful-diagnostic-examples)
- [Troubleshooting Checklist](#troubleshooting-checklist)
- [Best Practices](#best-practices)
- [Contributing](#contributing)
- [License](#license)

## Overview

WordPress provides a built-in search system powered by `WP_Query`.

When a visitor searches for a keyword, WordPress processes the search request, builds a database query, retrieves matching content, and displays the results through the active theme.

Search problems can occur because of:

- Incorrect search query configuration
- Unsupported or excluded post types
- Search filters modifying query behavior
- Missing searchable content
- Custom field search limitations
- Incorrect search templates
- Pagination configuration problems
- Plugin or theme conflicts
- Custom SQL modifications

This repository provides practical diagnostic techniques for identifying and troubleshooting these problems.

## Common Search Problems

| Problem | Possible Cause |
|---|---|
| Search returns no results | Query configuration or missing content |
| Pages are missing | Post type restrictions |
| Custom post types are missing | Search configuration |
| Search ignores certain fields | Search column limitations |
| Results are incorrect | Query filters |
| Search pagination fails | Incorrect query settings |
| Custom fields are not searchable | Default search limitations |
| Search template does not load | Theme template hierarchy |
| Results appear in unexpected order | Search ordering modifications |

## Requirements

- WordPress installation
- Basic PHP knowledge
- Access to the active theme or custom plugin
- WP-CLI (recommended)
- Access to WordPress debugging logs

Always test diagnostic changes on a staging environment before applying them to a production website.

## 1. Understand WordPress Search

WordPress uses the `WP_Query` class to retrieve search results.

A search request commonly uses the `s` query variable.

Example:

```php
$query = new WP_Query( array(
    's' => 'wordpress',
) );
```

The default search looks through these post fields:

- Post title
- Post excerpt
- Post content

The default search behavior does not automatically include arbitrary custom fields.

For official query documentation:

https://developer.wordpress.org/reference/classes/wp_query/

## 2. Test a Basic Search Query

Create a simple diagnostic query to verify that WordPress can retrieve matching content.

```php
$search_query = new WP_Query( array(
    's'              => 'wordpress',
    'post_type'      => 'any',
    'post_status'    => 'publish',
    'posts_per_page' => 10,
) );

echo 'Found results: ' . $search_query->found_posts;

if ( $search_query->have_posts() ) {
    while ( $search_query->have_posts() ) {
        $search_query->the_post();

        echo '<p>' . esc_html( get_the_title() ) . '</p>';
    }
}

wp_reset_postdata();
```

This example checks whether published content can be retrieved for the specified keyword.

Use a keyword that you know exists in your website content.

## 3. Diagnose Missing Search Results

If search returns no results, check the following:

1. Confirm that the keyword exists in published content.
2. Verify the post status.
3. Check the selected post types.
4. Review search-related filters.
5. Check whether plugins modify the search query.
6. Verify that the active theme displays the retrieved results.

Test with a known keyword:

```php
$query = new WP_Query( array(
    's'           => 'known-keyword',
    'post_status' => 'publish',
) );

echo 'Matching posts: ' . $query->found_posts;
```

If the query returns results but the website displays none, investigate the theme's search template and result loop.

## 4. Check Searchable Post Types

WordPress search behavior can be modified to include or exclude specific post types.

Inspect registered post types:

```php
$post_types = get_post_types(
    array(),
    'objects'
);

foreach ( $post_types as $post_type ) {
    echo $post_type->name . ' | ';
    echo $post_type->public ? 'Public' : 'Not Public';
    echo PHP_EOL;
}
```

To include selected post types in the main frontend search:

```php
add_action( 'pre_get_posts', function ( $query ) {

    if (
        ! is_admin() &&
        $query->is_main_query() &&
        $query->is_search()
    ) {
        $query->set(
            'post_type',
            array( 'post', 'page', 'product' )
        );
    }

} );
```

Replace `product` with an actual registered post type on your website.

This example changes only the main frontend search query.

Always verify that the selected post types are registered and that the desired content is publicly accessible.

## 5. Inspect Search Columns

WordPress 6.2 and later supports the `search_columns` argument in `WP_Query`.

Supported columns include:

- `post_title`
- `post_excerpt`
- `post_content`

Example:

```php
$query = new WP_Query( array(
    's' => 'wordpress',

    'search_columns' => array(
        'post_title',
        'post_content',
    ),
) );
```

This example searches post titles and content while excluding excerpts.

You can also use the `post_search_columns` filter:

```php
add_filter(
    'post_search_columns',
    function ( $columns, $search, $query ) {

        return array(
            'post_title',
            'post_content',
        );

    },
    10,
    3
);
```

Use this filter carefully because it can affect multiple search queries.

## 6. Diagnose Search Filters

Plugins and themes can modify search behavior through WordPress hooks.

Important search-related hooks include:

| Hook | Purpose |
|---|---|
| `pre_get_posts` | Modify query variables |
| `post_search_columns` | Modify searchable columns |
| `posts_search` | Modify search SQL |
| `posts_search_orderby` | Modify search ordering |
| `posts_clauses` | Modify SQL query clauses |

Search-related filters may unintentionally change the main query or custom queries.

To identify conflicts:

1. Review custom code in the active theme.
2. Search plugin code for search-related hooks.
3. Temporarily deactivate suspected plugins on staging.
4. Test with a default WordPress theme.
5. Compare search results before and after changes.

Avoid removing filters from production without understanding their purpose.

## 7. Check Search Exclusion Rules

WordPress supports excluding search terms by prefixing them with a hyphen.

Example:

```text
wordpress -plugin
```

This searches for content containing `wordpress` while excluding content matching `plugin`.

The default exclusion prefix can be changed through:

```php
add_filter(
    'wp_query_search_exclusion_prefix',
    function ( $prefix ) {
        return '-';
    }
);
```

If expected results are missing, inspect the search input and any custom exclusion logic.

## 8. Troubleshoot Search Pagination

Search pagination can fail when query settings are modified incorrectly.

Check the main search query:

```php
add_action( 'pre_get_posts', function ( $query ) {

    if (
        ! is_admin() &&
        $query->is_main_query() &&
        $query->is_search()
    ) {
        $query->set( 'posts_per_page', 10 );
    }

} );
```

Important checks:

- Confirm that `posts_per_page` has the expected value.
- Verify that the theme uses the correct pagination functions.
- Check whether custom `offset` values interfere with pagination.
- Confirm that the search query is not replaced by another query.
- Verify that the current page number is preserved in search URLs.

For custom query pagination, use the correct query object and its pagination properties.

## 9. Diagnose Search Relevance

WordPress search results may not match the order visitors expect.

Investigate:

- Search keyword matching
- Post title matching
- Search-related ordering filters
- Custom query modifications
- Plugin-based relevance systems

Inspect the search ordering filter:

```php
add_filter(
    'posts_search_orderby',
    function ( $search_orderby, $query ) {

        if ( $query->is_search() ) {
            // Inspect or customize search ordering here.
        }

        return $search_orderby;
    },
    10,
    2
);
```

Do not assume that the default search provides advanced relevance scoring.

If a website requires weighted ranking, typo tolerance, synonym support, or more advanced search behavior, consider a dedicated search solution.

## 10. Inspect Search Query Variables

Use `WP_Query` to inspect the variables used by a search request.

Example:

```php
add_action( 'wp', function () {

    if ( ! is_search() ) {
        return;
    }

    global $wp_query;

    error_log(
        'Search Query Variables: ' .
        print_r(
            $wp_query->query_vars,
            true
        )
    );

} );
```

Check:

- Search keyword
- Post type
- Post status
- Search columns
- Posts per page
- Current page
- Order and ordering rules

Avoid logging sensitive user input unnecessarily.

## 11. Troubleshoot Custom Field Search

The default WordPress search does not automatically search arbitrary post metadata.

For a specific custom field, use a dedicated query with `meta_query`.

Example:

```php
$query = new WP_Query( array(
    'post_type' => 'post',

    'meta_query' => array(
        array(
            'key'     => 'product_code',
            'value'   => 'ABC123',
            'compare' => 'LIKE',
        ),
    ),
) );
```

This example searches a specific metadata field.

It is not a complete replacement for combining normal keyword search with custom field search.

For advanced custom field search, carefully design the query and evaluate its performance on larger websites.

## 12. Check Search Template Loading

WordPress themes use the template hierarchy to display search results.

The primary template for search results is:

```text
search.php
```

If it does not exist, WordPress falls back to other applicable templates in the hierarchy.

Check the active theme for:

- `search.php`
- `index.php`
- Search result loops
- Pagination markup
- Search form markup

A working search query does not guarantee that the theme displays its results correctly.

If WordPress retrieves matching posts but the page appears empty, inspect the template and its loop.

Official reference:

https://developer.wordpress.org/themes/basics/template-hierarchy/

## 13. Debug Search SQL

WordPress provides query-related debugging information that can help identify unexpected search behavior.

For development environments, enable debugging in `wp-config.php`:

```php
define( 'WP_DEBUG', true );
define( 'WP_DEBUG_LOG', true );
define( 'WP_DEBUG_DISPLAY', false );
```

Inspect the last SQL query generated by a `WP_Query` instance:

```php
$query = new WP_Query( array(
    's' => 'wordpress',
) );

error_log( $query->request );
```

The generated SQL can help identify:

- Unexpected WHERE conditions
- Incorrect post type restrictions
- Search filter modifications
- Unexpected ordering
- Incorrect query limits

Do not display raw SQL or debugging information publicly on production websites.

## Useful Diagnostic Examples

### Check Whether the Current Request Is a Search

```php
if ( is_search() ) {
    error_log( 'Search request detected.' );
}
```

### Check the Search Keyword

```php
$keyword = get_search_query();

error_log(
    'Search keyword: ' . $keyword
);
```

### Check the Main Search Query

```php
add_action( 'pre_get_posts', function ( $query ) {

    if (
        ! is_admin() &&
        $query->is_main_query() &&
        $query->is_search()
    ) {
        error_log(
            'Main search query detected.'
        );

        error_log(
            print_r(
                $query->query_vars,
                true
            )
        );
    }

} );
```

### Check Whether a Search Returned Results

```php
$query = new WP_Query( array(
    's' => 'wordpress',
) );

if ( $query->have_posts() ) {
    echo 'Search results found.';
} else {
    echo 'No search results found.';
}

wp_reset_postdata();
```

## Troubleshooting Checklist

Use this checklist when diagnosing WordPress search problems.

- [ ] Confirm that the search keyword exists.
- [ ] Check the search query variables.
- [ ] Verify published post status.
- [ ] Check searchable post types.
- [ ] Inspect search columns.
- [ ] Review `pre_get_posts` modifications.
- [ ] Check `post_search_columns` filters.
- [ ] Inspect `posts_search` filters.
- [ ] Check custom search exclusion rules.
- [ ] Verify pagination settings.
- [ ] Review search ordering modifications.
- [ ] Check custom field search requirements.
- [ ] Inspect the active search template.
- [ ] Review plugin and theme conflicts.
- [ ] Enable debugging on a staging environment.
- [ ] Inspect generated SQL when necessary.

## Best Practices

- Use `WP_Query` for custom search queries.
- Use `pre_get_posts` when modifying the main query.
- Always check `is_main_query()` when targeting the main query.
- Avoid modifying unrelated admin queries.
- Use supported search columns.
- Keep custom search filters narrowly scoped.
- Test pagination after changing query variables.
- Avoid unnecessary SQL modifications.
- Do not use `query_posts()` to replace the main query.
- Test search functionality after plugin or theme updates.
- Use a dedicated search engine when advanced search capabilities are required.

## Contributing

Contributions are welcome.

You can help improve this repository by:

- Reporting search-related bugs.
- Adding diagnostic examples.
- Improving troubleshooting instructions.
- Documenting compatibility issues.
- Sharing reproducible search problems.

When reporting an issue, include:

- WordPress version
- PHP version
- Active theme
- Relevant plugins
- Search keyword example
- Expected behavior
- Actual behavior
- Relevant error messages

Do not include passwords, API keys, or private user information.

## License

This project is licensed under the MIT License.

You are free to use, modify, and distribute this material with appropriate attribution.

---

**WordPress Search Diagnostics**

A practical reference for troubleshooting WordPress search queries and search result problems.
