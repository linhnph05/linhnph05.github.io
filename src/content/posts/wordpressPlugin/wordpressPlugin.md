---
title: My note for leaning to audit Wordpress plugin
published: 2025-08-26
description: 'Learning to find vulnerabilities in Wordpress plugin'
image: ''
tags: [bug bounty, learning, wordpress, wordpress plugin]
category: 'wordpress'
draft: true 
lang: ''
---

:::note
This note is for my own educational purposes only, it is used for me to quickly remind me how some functions work when auditing plugins, so don't expect it to be well-organized and easy to understand
:::

# Searching:
- https://developer.wordpress.org/reference/
- https://developer.wordpress.org/apis/security/escaping/

# Common functions
- `sanitize_title( string $title, string $fallback_title = '', string $context = 'save' ): string`: Sanitizes a string into a slug, which can be used in URLs or HTML attributes.
- `sanitize_key( string $key ): string`: Keys are used as internal identifiers. Lowercase alphanumeric characters, dashes, and underscores are allowed.
- `balanceTags( $html )` or `force_balance_tags( $html )`: Tries to make sure HTML tags are balanced so that valid XML is output.
- `validate_file()` will validate that an entered file path is a real path (but not whether the file exists).
- `sanitize_html_class( string $classname, string $fallback = '' ): string`: Sanitizes an HTML classname to ensure it only contains valid characters. Strips the string down to A-Z,a-z,0-9,_,-.
- `tag_escape( string $tag_name ): string`: Escapes an HTML tag name.
- `sanitize_text_field( string $str ): string`: Checks for invalid UTF-8, Converts single < characters to entities, Strips all tags,Removes line breaks, tabs, and extra whitespace,Strips percent-encoded characters
- `esc_html()`
- `esc_js()`
- `esc_url()`
- `esc_url_raw()`: Use when storing a URL in the database or in other cases where non-encoded URLs are needed.
- `esc_xml()`
- `esc_attr()`
- `esc_textarea()`
- `wp_kses()` – Use to safely escape for all non-trusted HTML (post text, comment text, etc.). This preserves HTML.
- `wp_kses_post()` – Alternative version of wp_kses()that automatically allows all HTML that is permitted in post content.
- `wp_kses_data()` – Alternative version of wp_kses()that allows only the HTML permitted in post comments.

# Hook

## Action
- https://developer.wordpress.org/reference/functions/add_action/
- https://developer.wordpress.org/apis/hooks/action-reference/


```php
function wporg_callback() {
    // do something
}
add_action( 'init', 'wporg_callback' );
```

**Additional Parameters**

`add_action()` can accept two additional parameters, `int $priority` for the priority given to the callback function, and `int $accepted_args` for the number of arguments that will be passed to the callback function.

```php
add_action('init', 'wporg_callback_run_me_late', 11);
add_action('init', 'wporg_callback_run_me_normal');
add_action('init', 'wporg_callback_run_me_early', 9);
add_action('init', 'wporg_callback_run_me_later', 11);

do_action( 'save_post', $post->ID, $post );
add_action('save_post', 'wporg_custom', 10, 2);
```

## Filter
- https://developer.wordpress.org/apis/hooks/filter-reference/
- https://developer.wordpress.org/reference/functions/add_filter/

> Unlike Actions, filters are meant to work in an isolated manner, and should never have side effects such as affecting global variables and output. Filters expect to have something returned back to them.

```php
function wporg_filter_title( $title ) {
	return 'The ' . $title . ' was filtered';
}
add_filter( 'the_title', 'wporg_filter_title' );
```

## Custom hook
- https://developer.wordpress.org/reference/functions/do_action/
- https://developer.wordpress.org/reference/functions/apply_filters/

```php
// First componennt/plugin
do_action( 'wporg_after_settings_page_html' );

// Second componennt/plugin
add_action( 'wporg_after_settings_page_html', 'myprefix_add_settings' );
```

```php
// First componennt/plugin
function wporg_create_post_type() {
    $post_type_params = [/* ... */];

    register_post_type(
        'post_type_slug',
        apply_filters( 'wporg_post_type_params', $post_type_params )
    );
}

// Second componennt/plugin
function myprefix_change_post_type_params( $post_type_params ) {
	$post_type_params['hierarchical'] = true;
	return $post_type_params;
}
add_filter( 'wporg_post_type_params', 'myprefix_change_post_type_params' );
```

## Other
- https://developer.wordpress.org/reference/functions/remove_action/
- https://developer.wordpress.org/reference/functions/remove_filter/
- https://developer.wordpress.org/reference/functions/current_action/
- https://developer.wordpress.org/reference/functions/current_filter/
- https://developer.wordpress.org/reference/functions/did_action/

# User
> Each user role inherits the previous roles in the hierarchy. For example, the “Administrator”, which is the highest user role on a single site installation, inherits the following roles and their capabilities: “Subscriber”, “Contributor”, “Author” and “Editor”.
- Super Admin
- Administrator
- Editor
- Author
- Contributor
- Subscriber

```php
function wporg_simple_role() {
	add_role(
		'simple_role',
		'Simple Role',
		array(
			'read'         => true,
			'edit_posts'   => true,
			'upload_files' => true,
		),
	);
}

// Add the simple_role.
add_action( 'init', 'wporg_simple_role' );

function wporg_simple_role_remove() {
	remove_role( 'simple_role' );
}

// Remove the simple_role.
add_action( 'init', 'wporg_simple_role_remove' );
```


- When run add_role(), the role and its capabilities are saved to the database. If call add_role() again with different capabilities, nothing will change.**
- After the first call to remove_role() , the Role and it’s Capabilities will be removed from the database. If call remove_role() again with different capabilities, nothing will change**

```php
function wporg_simple_role_caps() {
	// Gets the simple_role role object.
	$role = get_role( 'simple_role' );

	// Add a new capability.
	$role->add_cap( 'edit_others_posts', true );
}

// Add simple_role capabilities, priority must be after the initial role definition.
add_action( 'init', 'wporg_simple_role_caps', 11 );
```

```php
user_can( $user, $capability );
current_user_can( $capability );
current_user_can_for_blog( $blog_id, $capability );
```

## User metadata
- show_user_profile hook: This action hook is fired whenever a user edits it’s own user profile.
- edit_user_profile hook: This action hook is fired whenever a user edits a user profile of somebody else.
```php
/**
 * The field on the editing screens.
 *
 * @param $user WP_User user object
 */
function wporg_usermeta_form_field_birthday( $user ) {
    ?>
    <h3>It's Your Birthday</h3>
    <table class="form-table">
        <tr>
            <th>
                <label for="birthday">Birthday</label>
            </th>
            <td>
                <input type="date"
                       class="regular-text ltr"
                       id="birthday"
                       name="birthday"
                       value="<?= esc_attr( get_user_meta( $user->ID, 'birthday', true ) ) ?>"
                       title="Please use YYYY-MM-DD as the date format."
                       pattern="(19[0-9][0-9]|20[0-9][0-9])-(1[0-2]|0[1-9])-(3[01]|[21][0-9]|0[1-9])"
                       required>
                <p class="description">
                    Please enter your birthday date.
                </p>
            </td>
        </tr>
    </table>
    <?php
}
 
/**
 * The save action.
 *
 * @param $user_id int the ID of the current user.
 *
 * @return bool Meta ID if the key didn't exist, true on successful update, false on failure.
 */
function wporg_usermeta_form_field_birthday_update( $user_id ) {
    // check that the current user have the capability to edit the $user_id
    if ( ! current_user_can( 'edit_user', $user_id ) ) {
        return false;
    }
 
    // create/update user meta for the $user_id
    return update_user_meta(
        $user_id,
        'birthday',
        $_POST['birthday']
    );
}
 
// Add the field to user's own profile editing screen.
add_action(
    'show_user_profile',
    'wporg_usermeta_form_field_birthday'
);
 
// Add the field to user profile editing screen.
add_action(
    'edit_user_profile',
    'wporg_usermeta_form_field_birthday'
);
 
// Add the save action to user's own profile editing screen update.
add_action(
    'personal_options_update',
    'wporg_usermeta_form_field_birthday_update'
);
 
// Add the save action to user profile editing screen update.
add_action(
    'edit_user_profile_update',
    'wporg_usermeta_form_field_birthday_update'
);
```

## Add user
- wp_create_user() allows you to create a new WordPress user. It uses wp_slash() to escape the values. The PHP compact() function to create an array with these values. The wp_insert_user() to perform the insert operation.
```php
// check if the username is taken
$user_id = username_exists( $user_name );

// check that the email address does not belong to a registered user
if ( ! $user_id && email_exists( $user_email ) === false ) {
	// create a random password
	$random_password = wp_generate_password( 12, false );
	// create the user
	$user_id = wp_create_user(
		$user_name,
		$random_password,
		$user_email
	);
}
```

```php
$username  = $_POST['username'];
$password  = $_POST['password'];
$website   = $_POST['website'];
$user_data = [
	'user_login' => $username,
	'user_pass'  => $password,
	'user_url'   => $website,
];

$user_id = wp_insert_user( $user_data );
```

## Update user
```php
$user_id = 1;
$website = 'https://wordpress.org';

$user_id = wp_update_user(
	array(
		'ID'       => $user_id,
		'user_url' => $website,
	)
);
```

# Database
- wp-admin/includes/schema.php
```bash
mysql -u DB_USER -p DB_NAME
# (credentials are in wp-config.php)
DESCRIBE wp_posts;
SHOW CREATE TABLE wp_posts\G
```

# Shortcodes
Built-in Shortcodes:
- [caption] – allows you to wrap captions around content
- [gallery] – allows you to show image galleries
- [audio] – allows you to embed and play audio files
- [video] – allows you to embed and play video files
- [playlist] – allows you to display collection of audio or video files
- [embed] – allows you to wrap embedded items

```php
/**
 * The [wporg] shortcode.
 *
 * Accepts a title and will display a box.
 *
 * @param array  $atts    Shortcode attributes. Default empty.
 * @param string $content Shortcode content. Default null.
 * @param string $tag     Shortcode tag (name). Default empty.
 * @return string Shortcode output.
 */
function wporg_shortcode( $atts = [], $content = null, $tag = '' ) {
	// normalize attribute keys, lowercase
	$atts = array_change_key_case( (array) $atts, CASE_LOWER );

	// override default attributes with user attributes
	$wporg_atts = shortcode_atts(
		array(
			'title' => 'WordPress.org',
		), $atts, $tag
	);

	// start box
	$o = '<div class="wporg-box">';

	// title
	$o .= '<h2>' . esc_html( $wporg_atts['title'] ) . '</h2>';

	// enclosing tags
	if ( ! is_null( $content ) ) {
		// $content here holds everything in between the opening and the closing tags of your shortcode. eg.g [my-shortcode]content[/my-shortcode].
        // Depending on what your shortcode supports, you will parse and append the content to your output in different ways.
		// In this example, we just secure output by executing the_content filter hook on $content.
		$o .= apply_filters( 'the_content', $content );
	}

	// end box
	$o .= '</div>';

	// return output
	return $o;
}

/**
 * Central location to create all shortcodes.
 */
function wporg_shortcodes_init() {
	add_shortcode( 'wporg', 'wporg_shortcode' );
}

add_action( 'init', 'wporg_shortcodes_init' );
```

# Post
> The unique flag allows you to declare whether this key should be unique. A non unique key is something a post can have multiple variations of, like price.
> If you only ever want one price for a post, you should flag it unique and the meta_key will have one value only

- https://developer.wordpress.org/reference/functions/add_post_meta/
- https://developer.wordpress.org/reference/functions/update_post_meta/

## Metabox
```php
function wporg_add_custom_box() {
	$screens = [ 'post', 'wporg_cpt' ];
	foreach ( $screens as $screen ) {
		add_meta_box(
			'wporg_box_id',                 // Unique ID
			'Custom Meta Box Title',      // Box title
			'wporg_custom_box_html',  // Content callback, must be of type callable
			$screen                            // Post type
		);
	}
}
add_action( 'add_meta_boxes', 'wporg_add_custom_box' );

function wporg_custom_box_html( $post ) {
	?>
	<label for="wporg_field">Description for this field</label>
	<select name="wporg_field" id="wporg_field" class="postbox">
		<option value="">Select something...</option>
		<option value="something">Something</option>
		<option value="else">Else</option>
	</select>
	<?php
}

```

## Post type
```php
function wporg_custom_post_type() {
	register_post_type('wporg_product',
		array(
			'labels'      => array(
				'name'          => __('Products', 'textdomain'),
				'singular_name' => __('Product', 'textdomain'),
			),
				'public'      => true,
				'has_archive' => true,
                'rewrite'     => array( 'slug' => 'products' ), // my custom slug
		)
	);
}
add_action('init', 'wporg_custom_post_type');
```

> A custom post type gets its own slug within the site URL structure.
> A post of type wporg_product will use the following URL structure by default: http://example.com/wporg_product/%product_name%.

# Taxonomies
- https://developer.wordpress.org/plugins/taxonomies/working-with-custom-taxonomies/

> A Taxonomy is a fancy word for the classification/grouping of things. Taxonomies can be hierarchical (with parents/children) or flat.
> WordPress stores the Taxonomies in the term_taxonomy database table allowing developers to register Custom Taxonomies along the ones that already exist.
> 
> Taxonomies have Terms which serve as the topic by which you classify/group things. They are stored inside the terms table.
> For example: a Taxonomy named “Art” can have multiple Terms, such as “Modern” and “18th Century”.

# REST API
http://oursite.com/wp-json/
Route: URI
Endpoint: URI + HTTP method
`register_rest_route()` use on `rest_api_init` action hook
`rest_ensure_response()` wraps the data we want to return into a WP_REST_Response, and ensures it will be properly returned.
```php
/**
 * This is our callback function that embeds our phrase in a WP_REST_Response
 */
function prefix_get_endpoint_phrase() {
    // rest_ensure_response() wraps the data we want to return into a WP_REST_Response, and ensures it will be properly returned.
    return rest_ensure_response( 'Hello World, this is the WordPress REST API' );
}

/**
 * This function is where we register our routes for our example endpoint.
 */
function prefix_register_example_routes() {
    // register_rest_route() handles more arguments but we are going to stick to the basics for now.
    register_rest_route( 'hello-world/v1', '/phrase', array(
        // By using this constant we ensure that when the WP_REST_Server changes our readable endpoints will work as intended.
        'methods'  => WP_REST_Server::READABLE,
        // Here we register our callback. The callback is fired when this endpoint is matched by the WP_REST_Server class.
        'callback' => 'prefix_get_endpoint_phrase',
    ) );
}

add_action( 'rest_api_init', 'prefix_register_example_routes' );
```

```php
/**
 * This is our callback function to return our products.
 *
 * @param WP_REST_Request $request This function accepts a rest request to process data.
 */
function prefix_get_products( $request ) {
    // In practice this function would fetch the desired data. Here we are just making stuff up.
    $products = array(
        '1' => 'I am product 1',
        '2' => 'I am product 2',
        '3' => 'I am product 3',
    );

    return rest_ensure_response( $products );
}

/**
 * This is our callback function to return a single product.
 *
 * @param WP_REST_Request $request This function accepts a rest request to process data.
 */
function prefix_get_product( $request ) {
    // In practice this function would fetch the desired data. Here we are just making stuff up.
    $products = array(
        '1' => 'I am product 1',
        '2' => 'I am product 2',
        '3' => 'I am product 3',
    );

    // Here we are grabbing the 'id' path variable from the $request object. WP_REST_Request implements ArrayAccess, which allows us to grab properties as though it is an array.
    $id = (string) $request['id'];

    if ( isset( $products[ $id ] ) ) {
        // Grab the product.
        $product = $products[ $id ];

        // Return the product as a response.
        return rest_ensure_response( $product );
    } else {
        // Return a WP_Error because the request product was not found. In this case we return a 404 because the main resource was not found.
        return new WP_Error( 'rest_product_invalid', esc_html__( 'The product does not exist.', 'my-text-domain' ), array( 'status' => 404 ) );
    }

    // If the code somehow executes to here something bad happened return a 500.
    return new WP_Error( 'rest_api_sad', esc_html__( 'Something went horribly wrong.', 'my-text-domain' ), array( 'status' => 500 ) );
}

/**
 * This function is where we register our routes for our example endpoint.
 */
function prefix_register_product_routes() {
    // Here we are registering our route for a collection of products.
    register_rest_route( 'my-shop/v1', '/products', array(
        // By using this constant we ensure that when the WP_REST_Server changes our readable endpoints will work as intended.
        'methods'  => WP_REST_Server::READABLE,
        // Here we register our callback. The callback is fired when this endpoint is matched by the WP_REST_Server class.
        'callback' => 'prefix_get_products',
    ) );
    // Here we are registering our route for single products. The (?P<id>[\d]+) is our path variable for the ID, which, in this example, can only be some form of positive number.
    register_rest_route( 'my-shop/v1', '/products/(?P<id>[\d]+)', array(
        // By using this constant we ensure that when the WP_REST_Server changes our readable endpoints will work as intended.
        'methods'  => WP_REST_Server::READABLE,
        // Here we register our callback. The callback is fired when this endpoint is matched by the WP_REST_Server class.
        'callback' => 'prefix_get_product',
    ) );
}

add_action( 'rest_api_init', 'prefix_register_product_routes' );
```

```php
/**
 * This is our callback function to return our products.
 *
 * @param WP_REST_Request $request This function accepts a rest request to process data.
 */
function prefix_get_products( $request ) {
    // In practice this function would fetch the desired data. Here we are just making stuff up.
    $products = array(
        '1' => 'I am product 1',
        '2' => 'I am product 2',
        '3' => 'I am product 3',
    );

    return rest_ensure_response( $products );
}

/**
 * This is our callback function to return a single product.
 *
 * @param WP_REST_Request $request This function accepts a rest request to process data.
 */
function prefix_create_product( $request ) {
    // In practice this function would create a product. Here we are just making stuff up.
   return rest_ensure_response( 'Product has been created' );
}

/**
 * This function is where we register our routes for our example endpoint.
 */
function prefix_register_product_routes() {
    // Here we are registering our route for a collection of products and creation of products.
    register_rest_route( 'my-shop/v1', '/products', array(
        array(
            // By using this constant we ensure that when the WP_REST_Server changes, our readable endpoints will work as intended.
            'methods'  => WP_REST_Server::READABLE,
            // Here we register our callback. The callback is fired when this endpoint is matched by the WP_REST_Server class.
            'callback' => 'prefix_get_products',
        ),
        array(
            // By using this constant we ensure that when the WP_REST_Server changes, our create endpoints will work as intended.
            'methods'  => WP_REST_Server::CREATABLE,
            // Here we register our callback. The callback is fired when this endpoint is matched by the WP_REST_Server class.
            'callback' => 'prefix_create_product',
        ),
    ) );
}

add_action( 'rest_api_init', 'prefix_register_product_routes' );
```

```php
/**
 * This is our callback function that embeds our resource in a WP_REST_Response
 */
function prefix_get_colors( $request ) {
    // In practice this function would fetch more practical data. Here we are just making stuff up.
    $colors = array(
        'blue',
        'blue',
        'red',
        'red',
        'green',
        'green',
    );

    if ( isset( $request['filter'] ) ) {
       $filtered_colors = array();
       foreach ( $colors as $color ) {
           if ( $request['filter'] === $color ) {
               $filtered_colors[] = $color;
           }
       }
       return rest_ensure_response( $filtered_colors );
    }
    return rest_ensure_response( $colors );
}
/**
 * Validate a request argument based on details registered to the route.
 *
 * @param  mixed            $value   Value of the 'filter' argument.
 * @param  WP_REST_Request  $request The current request object.
 * @param  string           $param   Key of the parameter. In this case it is 'filter'.
 * @return WP_Error|boolean
 */
function prefix_filter_arg_validate_callback( $value, $request, $param ) {
    // If the 'filter' argument is not a string return an error.
    if ( ! is_string( $value ) ) {
        return new WP_Error( 'rest_invalid_param', esc_html__( 'The filter argument must be a string.', 'my-text-domain' ), array( 'status' => 400 ) );
    }

    // Get the registered attributes for this endpoint request.
    $attributes = $request->get_attributes();

    // Grab the filter param schema.
    $args = $attributes['args'][ $param ];

    // If the filter param is not a value in our enum then we should return an error as well.
    if ( ! in_array( $value, $args['enum'], true ) ) {
        return new WP_Error( 'rest_invalid_param', sprintf( __( '%s is not one of %s' ), $param, implode( ', ', $args['enum'] ) ), array( 'status' => 400 ) );
    }
}

/**
 * We can use this function to contain our arguments for the example product endpoint.
 */
function prefix_get_color_arguments() {
    $args = array();
    // Here we are registering the schema for the filter argument.
    $args['filter'] = array(
        // description should be a human readable description of the argument.
        'description' => esc_html__( 'The filter parameter is used to filter the collection of colors', 'my-text-domain' ),
        // type specifies the type of data that the argument should be.
        'type'        => 'string',
        // enum specified what values filter can take on.
        'enum'        => array( 'red', 'green', 'blue' ),
        // Here we register the validation callback for the filter argument.
        'validate_callback' => 'prefix_filter_arg_validate_callback',
    );
    return $args;
}

/**
 * This function is where we register our routes for our example endpoint.
 */
function prefix_register_example_routes() {
    // register_rest_route() handles more arguments but we are going to stick to the basics for now.
    register_rest_route( 'my-colors/v1', '/colors', array(
        // By using this constant we ensure that when the WP_REST_Server changes our readable endpoints will work as intended.
        'methods'  => WP_REST_Server::READABLE,
        // Here we register our callback. The callback is fired when this endpoint is matched by the WP_REST_Server class.
        'callback' => 'prefix_get_colors',
        // Here we register our permissions callback. The callback is fired before the main callback to check if the current user can access the endpoint.
        'args' => prefix_get_color_arguments(),
    ) );
}

add_action( 'rest_api_init', 'prefix_register_example_routes' );
```

```php
/**
 * This is our callback function that embeds our resource in a WP_REST_Response.
 *
 * The parameter is already sanitized by this point so we can use it without any worries.
 */
function prefix_get_item( $request ) {
    if ( isset( $request['data'] ) ) {
        return rest_ensure_response( $request['data'] );
    }

    return new WP_Error( 'rest_invalid', esc_html__( 'The data parameter is required.', 'my-text-domain' ), array( 'status' => 400 ) );
}

/**
 * Validate a request argument based on details registered to the route.
 *
 * @param  mixed            $value   Value of the 'filter' argument.
 * @param  WP_REST_Request  $request The current request object.
 * @param  string           $param   Key of the parameter. In this case it is 'filter'.
 * @return WP_Error|boolean
 */
function prefix_data_arg_validate_callback( $value, $request, $param ) {
    // If the 'data' argument is not a string return an error.
    if ( ! is_string( $value ) ) {
        return new WP_Error( 'rest_invalid_param', esc_html__( 'The filter argument must be a string.', 'my-text-domain' ), array( 'status' => 400 ) );
    }
}

/**
 * Sanitize a request argument based on details registered to the route.
 *
 * @param  mixed            $value   Value of the 'filter' argument.
 * @param  WP_REST_Request  $request The current request object.
 * @param  string           $param   Key of the parameter. In this case it is 'filter'.
 * @return WP_Error|boolean
 */
function prefix_data_arg_sanitize_callback( $value, $request, $param ) {
    // It is as simple as returning the sanitized value.
    return sanitize_text_field( $value );
}

/**
 * We can use this function to contain our arguments for the example product endpoint.
 */
function prefix_get_data_arguments() {
    $args = array();
    // Here we are registering the schema for the filter argument.
    $args['data'] = array(
        // description should be a human readable description of the argument.
        'description' => esc_html__( 'The data parameter is used to be sanitized and returned in the response.', 'my-text-domain' ),
        // type specifies the type of data that the argument should be.
        'type'        => 'string',
        // Set the argument to be required for the endpoint.
        'required'    => true,
        // We are registering a basic validation callback for the data argument.
        'validate_callback' => 'prefix_data_arg_validate_callback',
        // Here we register the validation callback for the filter argument.
        'sanitize_callback' => 'prefix_data_arg_sanitize_callback',
    );
    return $args;
}

/**
 * This function is where we register our routes for our example endpoint.
 */
function prefix_register_example_routes() {
    // register_rest_route() handles more arguments but we are going to stick to the basics for now.
    register_rest_route( 'my-plugin/v1', '/sanitized-data', array(
        // By using this constant we ensure that when the WP_REST_Server changes our readable endpoints will work as intended.
        'methods'  => WP_REST_Server::READABLE,
        // Here we register our callback. The callback is fired when this endpoint is matched by the WP_REST_Server class.
        'callback' => 'prefix_get_item',
        // Here we register our permissions callback. The callback is fired before the main callback to check if the current user can access the endpoint.
        'args' => prefix_get_data_arguments(),
    ) );
}

add_action( 'rest_api_init', 'prefix_register_example_routes' );
```

```json
{
  "methods": {
    "GET": true
  },
  "accept_json": false,
  "accept_raw": false,
  "show_in_index": true,
  "args": {
    "context": {
      "description": "Scope under which the request is made; determines fields present in response.",
      "type": "string",
      "sanitize_callback": "sanitize_key",
      "validate_callback": "rest_validate_request_arg",
      "enum": [
        "view",
        "embed",
        "edit"
      ],
      "default": "view"
    },
    "page": {
      "description": "Current page of the collection.",
      "type": "integer",
      "default": 1,
      "sanitize_callback": "absint",
      "validate_callback": "rest_validate_request_arg",
      "minimum": 1
    },
    "per_page": {
      "description": "Maximum number of items to be returned in result set.",
      "type": "integer",
      "default": 10,
      "minimum": 1,
      "maximum": 100,
      "sanitize_callback": "absint",
      "validate_callback": "rest_validate_request_arg"
    },
    "search": {
      "description": "Limit results to those matching a string.",
      "type": "string",
      "sanitize_callback": "sanitize_text_field",
      "validate_callback": "rest_validate_request_arg"
    },
    "after": {
      "description": "Limit response to resources published after a given ISO8601 compliant date.",
      "type": "string",
      "format": "date-time",
      "validate_callback": "rest_validate_request_arg"
    },
    "author": {
      "description": "Limit result set to posts assigned to specific authors.",
      "type": "array",
      "default": [],
      "sanitize_callback": "wp_parse_id_list",
      "validate_callback": "rest_validate_request_arg"
    },
    "author_exclude": {
      "description": "Ensure result set excludes posts assigned to specific authors.",
      "type": "array",
      "default": [],
      "sanitize_callback": "wp_parse_id_list",
      "validate_callback": "rest_validate_request_arg"
    },
    "before": {
      "description": "Limit response to resources published before a given ISO8601 compliant date.",
      "type": "string",
      "format": "date-time",
      "validate_callback": "rest_validate_request_arg"
    },
    "exclude": {
      "description": "Ensure result set excludes specific ids.",
      "type": "array",
      "default": [],
      "sanitize_callback": "wp_parse_id_list"
    },
    "include": {
      "description": "Limit result set to specific ids.",
      "type": "array",
      "default": [],
      "sanitize_callback": "wp_parse_id_list"
    },
    "offset": {
      "description": "Offset the result set by a specific number of items.",
      "type": "integer",
      "sanitize_callback": "absint",
      "validate_callback": "rest_validate_request_arg"
    },
    "order": {
      "description": "Order sort attribute ascending or descending.",
      "type": "string",
      "default": "desc",
      "enum": [
        "asc",
        "desc"
      ],
      "validate_callback": "rest_validate_request_arg"
    },
    "orderby": {
      "description": "Sort collection by object attribute.",
      "type": "string",
      "default": "date",
      "enum": [
        "date",
        "relevance",
        "id",
        "include",
        "title",
        "slug"
      ],
      "validate_callback": "rest_validate_request_arg"
    },
    "slug": {
      "description": "Limit result set to posts with a specific slug.",
      "type": "string",
      "validate_callback": "rest_validate_request_arg"
    },
    "status": {
      "default": "publish",
      "description": "Limit result set to posts assigned a specific status; can be comma-delimited list of status types.",
      "enum": [
        "publish",
        "future",
        "draft",
        "pending",
        "private",
        "trash",
        "auto-draft",
        "inherit",
        "any"
      ],
      "sanitize_callback": "sanitize_key",
      "type": "string",
      "validate_callback": [
        {},
        "validate_user_can_query_private_statuses"
      ]
    },
    "filter": {
      "description": "Use WP Query arguments to modify the response; private query vars require appropriate authorization."
    },
    "categories": {
      "description": "Limit result set to all items that have the specified term assigned in the categories taxonomy.",
      "type": "array",
      "sanitize_callback": "wp_parse_id_list",
      "default": []
    },
    "tags": {
      "description": "Limit result set to all items that have the specified term assigned in the tags taxonomy.",
      "type": "array",
      "sanitize_callback": "wp_parse_id_list",
      "default": []
    }
  },
  "callback": [
    {},
    "get_items"
  ],
  "permission_callback": [
    {},
    "get_items_permissions_check"
  ]
}
```