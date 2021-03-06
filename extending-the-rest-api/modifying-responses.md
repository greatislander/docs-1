# Modifying Responses

The default endpoints of the WordPress REST API are designed to return data by default that provides for the majority of sites and use cases, but there are often situations where you will need to access or expose additional data in the responses for various object types.

As with the rest of WordPress, the REST API is designed to be highly extensible to fit these needs. This guide details how to add additional data to the responses of default endpoints using the `register_rest_field` and `register_meta` functions. You can use these functions to add fields to any of the object types supported by the REST API. These custom fields can support both get and update operations.


## Important Note about Changing Responses

[alert]
Changing or removing data from core REST API endpoint responses can break plugins or WordPress core behavior, and should be avoided wherever possible.
[/alert]

The API exposes many fields on API responses, including things you might not need, or which might not fit into how your site works. While it's tempting to modify or remove fields from REST API responses, this **will** cause problems with API clients that expect standard responses. This includes mobile clients, third party tools to help you manage your site, or wp-admin itself.

You may only need a small amount of data, but it's important to keep in mind that the API is about exposing an interface to all clients, not just the feature you're working on. Changing responses is dangerous.

Adding fields is not dangerous, so if you need to modify data, it's much better to duplicate the field instead with your modified data. Removing fields is never encouraged; if you need to get back a smaller subset of data, work with contexts instead, and consider making your own context.

If you must remove fields from an existing context you should ensure that the behavior is **opt-in**, for example by providing a custom query parameter to trigger the field removal.

The API cannot prevent you from changing responses, but the code is structured to strongly discourage doing so. Internally, field registration is powered by filters, and these can be used if you absolutely have no other choice.


## Using `register_rest_field` vs `register_meta`

There are two methods which can be used to add data to WordPress REST API responses, `register_rest_field` and `register_meta`.

`register_rest_field` may be used to add arbitrary fields to any REST API response, and can be used to both read and write data using the API. To register a new REST field you must provide your own callback functions to get or set the field's value, as well as manually specify your own schema definition for the field.

`register_meta` is used to whitelist an existing custom meta value for access through the REST API. By setting a meta field's `show_in_rest` parameter to `true`, that field's value will be exposed on a `.meta` key in the endpoint response, and WordPress will handle setting up the callbacks for reading and writing to that meta key. This is much simpler than `register_rest_field`, with one caveat:

[alert]
Meta fields set to `show_in_rest` using `register_meta` are registered for all objects of a given type. If one custom post type shows a meta field, all custom post types will show that meta field.
[/alert]

This is because meta registration is currently handled at the base level of WordPress object (post, user, etc) and cannot be specified per "subtype" (**e.g.** custom post types). [This limitation may be improved](https://core.trac.wordpress.org/ticket/38323) in future versions of WordPress, but if you are trying to add a custom field to a specific API endpoint's response then `register_rest_field` is currently recommended over `register_meta`.


## Adding Custom Fields to API Responses

### Using `register_rest_field`

The [`register_rest_field`](https://developer.wordpress.org/reference/functions/register_rest_field/) function is the most flexible way to add fields to REST API response objects. It accepts three parameters:

1. `$object_type`: The name of the object, as a string, or an array of the names of objects for which the field is being registered. This may be a core type like "post", "terms", "meta", "user" or "comment", but can also be the string name of a custom post type.
2. `$attribute`: The name of the field. This name will be used to define the key in the response object.
3. `$args`: An array with keys that define the callback functions used to retrieve the value of the field ('get_callback'), to update the value of the field ('update_callback'), and to define its schema ('schema').

Each of the keys of the $args array is optional, but if not used, that capability will not be added. This means that you may specify a callback function for reading the value and omit the update callback to make that field read-only if desired.

Fields should be registered at the `rest_api_init` action. Using this action rather than `init` will prevent the field registration from happening during requests to WordPress that do not use the REST API.


### Examples

#### Read and write an extra field in comment responses

```php
<?php
add_action( 'rest_api_init', function () {
    register_rest_field( 'comment', 'karma', array(
        'get_callback' => function( $comment_arr ) {
            $comment_obj = get_comment( $comment_arr['id'] );
            return (int) $comment_obj->comment_karma;
        },
        'update_callback' => function( $karma, $comment_obj ) {
            $ret = wp_update_comment( array(
                'comment_ID'    => $comment_obj->comment_ID,
                'comment_karma' => $karma
            ) );
            if ( false === $ret ) {
                return new WP_Error(
                  'rest_comment_karma_failed',
                  __( 'Failed to update comment karma.' ),
                  array( 'status' => 500 )
                );
            }
            return true;
        },
        'schema' => array(
            'description' => __( 'Comment karma.' ),
            'type'        => 'integer'
        ),
    ) );
} );
```

This example illustrates adding a field called `karma` to the response for posts. It works because the `comment_karma` field exists, but is unused by core. Note that an actual implementation of comment karma would need to use a separate endpoint.

This is a basic example; carefully consider what permissions checks or error handling may be required for your specific field.


### How `register_rest_field` Works

The global variable `$wp_rest_additional_fields` is used internally by the REST API infrastructure to hold the response fields to be added to each object type. The REST API provides `register_rest_field` as a utility function for adding to this global variable. Adding to the global variable directly should be avoided to ensure forward-compatibility.

For each object type &mdash; posts, or users, terms, comments, etc. &mdash; `$wp_rest_additional_fields` contains an array of field definitions containing the callbacks used to retrieve or update the field's value.


## Working with registered meta in the REST API

The [`register_meta`](https://developer.wordpress.org/reference/functions/register_meta/) function simplifies the process of defining a meta field for a particular object type. By setting `'show_in_rest' => true` when registering a new meta key, that key will be accessible through the REST API. Note however that at this time **there is no way to register a meta field for a specific post type**: meta fields registered for the "post" object will appear on **all** custom post types, as well as the default post record. For this reason it is less broadly useful than `register_rest_field`.


### Read and write a post meta field in post responses

```php
<?php
// The object type. For custom post types, this is 'post';
// for custom comment types, this is 'comment'. For user meta,
// this is 'user'.
$object_type = 'post';
$args1 = array( // Validate and sanitize the meta value.
    // Note: currently (4.7) one of 'string', 'boolean', 'integer',
    // 'number' must be used as 'type'. The default is 'string'.
    'type'         => 'string',
    // Shown in the schema for the meta key.
    'description'  => 'A meta key associated with a string meta value.',
    // Return a single value of the type.
    'single'       => true,
    // Show in the WP REST API response. Default: false.
    'show_in_rest' => true,
);
register_meta( $object_type, 'my_meta_key', $args1 );
```

This example shows how to allow reading and writing of a post meta field. This will allow that field to be updated via a POST request to `wp-json/wp/v2/posts/<post-id>` or created along with a post via a POST request to `wp-json/wp/v2/posts/`.

Note that for meta fields registered on custom post types, the post type must have `custom-fields` support. Otherwise the meta fields will not appear in the REST API.
