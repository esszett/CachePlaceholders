# Cacheable Placeholders for ProcessWire

This is a [ProcessWire](https://processwire.com/) module that enables you to include dynamic content in cached page output. This works by including special placeholders inside your source code that are replaced with dynamic content for every request. This works with both the template cache and the `$cache` API. The module comes with some built-in placeholders for CSRF (or other random) tokens and retreiving values from superglobals like the current session. You can also define your own placeholders with a callback that generates the dynamic output. The module can be used in automatic mode, which automatically replaces placeholders during every request (even when the response comes from the template cache) or manually by using the `replaceTokens` method on specific pieces of cached data.

## Table of contents

- [Motivation and breakdown](#motivation-and-breakdown)
    - [Why not use Hanna Code or any existing tag parser?](#why-not-use-hanna-code-or-any-existing-tag-parser)
    - [Does this support Form Builder's CSRF protection?](#does-this-support-form-builders-csrf-protection)
- [Usage & examples](#usage--examples)
    - [Manual usage](#manual-usage)
    - [Token data and parameters](#token-data-and-parameters)
    - [Example #1: Random number with limits](#example-1-random-number-with-limits)
    - [Example #2: Personal greeting](#example-2-personal-greeting)
- [Built-in tokens](#built-in-tokens)
    - [superglobal: Accessing session, request or cookie data](#superglobal-accessing-session-request-or-cookie-data)
    - [csrf: CSRF-Input for forms](#csrf-csrf-input-for-forms)
    - [random_hex: Cryptographically secure hexdecimal tokens](#random_hex-cryptographically-secure-hexdecimal-tokens)
- [Changing the token format](#changing-the-token-format)

## Motivation and breakdown

Caching is a huge factor for page speed. Caching the output means most of your template code doesn't need to run for every request, which decreases server load and increases response times. Unfortunately, caching becomes difficult if we need dynamic content in our pages, i.e. content that depends on some variable that is not stored inside a page field, like the current user, session or time. For example, let's say you want to have a personal greeting on your site that includes the name of the current user which is stored in the session. If you want to do this server-side (using JavaScript comes with it's own set of problems), you run into a problem with the cache. If we show Bob a page that was cached during a page visit by Alice, Bob will see `Hello Alice!` instead of `Hello Bob!`. This means that you can't really use the [template cache](https://processwire.com/docs/start/structure/templates/#other-page-settings-managed-by-templates) anymore, because it caches page output wholesale. You can still use the [$cache API](https://processwire.com/api/ref/wire-cache/) to cache individual sections of your page output, you just have to be careful not to cache the section that includes the greeting. Now this problem gets worse if the greeting (or any piece of dynamic content) is not placed in a predictable place in the templates, but instead used as a [Hanna Code shortcode](https://modules.processwire.com/modules/process-hanna-code/) in a textarea. At this point, the greeting may appear anywhere on your page, so you can't really cache much anymore. This necessitates a much more granular caching approach, which will be more complicated and less efficient. Another solution is to vary your cache keys by whatever is the source of variance in the template – for example, the current user ID. But that results in far fewer cache hits, because a cached response created for Alice can't be served to Bob.

This module aims to solve that by introducing placeholders into your cached content that get replaced dynamically on each request. This way, only the code that actually needs to run during every request does so, while everything else can be cached. I'll refer to those placeholders as *cache tokens* troughout the documentation. This module is intended as a developer tool with a focus on registering your own cache tokens. For this purpose, the module provides the following:

- A hookable method you can use to add your own cache tokens and callbacks, as well as a parameter parser for multiple positional or named parameters (including multi-value parameters).
- A method to perform token replacements manually, as well as an automatic mode that replaces tokens in the page output for every request.
- A couple of built-in tokens for common tasks.
- Customization options for the token format & delimiters to make sure the format is compatible with other tag / shortcode parsers.

### Why not use Hanna Code or any existing tag parser?

This module has some overlap with the functionality of existing plugins such as [Hanna Code](https://modules.processwire.com/modules/process-hanna-code/) and [Variable Context Tag Parser](https://modules.processwire.com/modules/textformatter-tag-parser/). Hanna Code in particular could conceivably be used for the same purpose. However, there are multiple reasons to have this functionality as a separate module:

- The modules mentioned above are typically run as textformatters, so their respespective shortcodes are replaced *before* the rendered page is cached. This means they can't generate dynamic content when used this way, because their output will be cached as well. Of course, you can use Hanna Code manually by hooking after `Page::render`, but then you can't cache it's output at all.
- This module can by used in conjunction with Hanna Code. You can even generate cache placeholders *inside* Hanna Code shortcodes. This way, Hanna Code can do the heavy lifting in terms of argument parsing etc. and only generate cache placeholders for the parts that actually need to be dynamic.
- Because this module needs to run during every request, it's important that it has a small footprint so it doesn't degrade your site's performance. This is why it has a very limited feature set as opposed to the more feature-rich Hanna Code, so it just needs to perform a single regular expression and call the appropriate callback functions.
- This module is intended as a developer tool, NOT as a shortcode parser. This is why the tag format is very limited (for example, you can't use whitespace or diacritics inside token parameters). Cache tokens should usually be written in template files, not in textarea fields. If you need to have dynamic placeholders inside WYSIWYG editors (like CK Editor fields), I recommend you use Hanna Code shortcodes that generate cache tokens as described above.

### Does this support Form Builder's CSRF protection?

Unfortunately, the commercial [Form Builder](https://processwire.com/store/form-builder/) module currently does not include any hooks that would allow this. If you have an idea how to make this work, talk to me!

## Usage & examples

This is a step-by-step guide to getting started with a custom placeholder token.

First, you need to register your custom token(s) with the module (unless you want to use one of the built-in tokens, see below). This is done by hooking `CachePlaceholders::getTokens`. A token definition consists of an alphanumeric name (hyphens and underscores are allowed) and a callback function. The callback should return the dynamic value that you want to output in place of the token.

```php
// site/ready.php
wire()->addHookAfter('CachePlaceholders::getTokens', function (HookEvent $e) {
    $tokens = $e->return;
    // add a new "random_number" token
    // make sure not to overwrite the array or the built-in tokens won't work anymore!
    $tokens['random_number'] = [
        // the token definition is an array with a required "callback" key
        // the callback needs to accept a single array argument
        'callback' => function (array $tokenData) {
            // the callback function returns the replacement value for this token
            return rand();
        }
    ];
    $e->return = $tokens;
});
```

This registers a `random_number` token that just returns a new random integer every time using [rand](https://www.php.net/manual/en/function.rand.php). Read on below to find out how to use the `$tokenData` array to make the token more flexible. Now go to the module configuration (*Modules -> Configure -> CachePlaceholders*), which should list your `random_number` token in the *Token list* field. This will also alert you if there are any problems with your token definition.

Now add the token somewhere in one of your PHP templates: `{{{random_number}}}` and reload the page. You should see a random number in place of the token. Now activate the template cache for the current template and open a new private window (so you see the page as a guest user). Reload the page a couple of times and check if the value is changing for every request. If it does, the module is working correctly and replacing the token dynamically inside the cached page.

### Manual usage

While the module is operating in automatic mode, it uses a hook after `Page::render` to replace tokens even in cached pages. You can also use the module manually to replace tokens in any text you pass it. This is useful if you're using a more granular caching approach, like caching individual sections of your page with the [$cache API](https://processwire.com/api/ref/wire-cache/). In this case, you may want to deactivate the automatic mode through the settings, so your output doesn't get parsed twice (though technically this won't hurt anyone).

Here's an example using the module's `replaceTokens` method to dynamically replace tokens in a cached piece of HTML code:

```php
$cachedOutput = wire('cache')->get('my-content-cache', 3600, function () {
    // normally you will include a template file and render some more content here
    return '<p>My random number: {{{random_number}}}</p>';
})
$CachePlaceholders = wire('modules')->get('CachePlaceholders');
echo $CachePlaceholders->replaceTokens($cachedOutput);
// <p>My random number: 905271593</p>
```

### Token data and parameters

Tokens can optionally include any number of parameters. Those get parsed and passed to your token callback. Here's an example including all forms those parameters can take.

```text
{{{token_name::foo::bar::a_key=a_value::a_multivalue_field=value1|value2}}}
```

- `{{{` and `}}}` open and close the token, respectively. There may not be any whitespace in between.
- `token_name` is the name of your token (`random_number` in the example above).
- The double colons `::` separate the token name from the parameters as well as the individual parameters from each other.
- Parameters including an equals sign `=` are named parameters, parameters without one are positional.
- You can use multivalue parameters (represented as an array) by separating individual values with a pipe `|` (both in named and positional parameter values).

The callback function for your token receives an array containing the following information on the token it replaces. For the example token above the function would receive the following data:

```php
$tokens['random_number'] = [
    'callback' => function (array $tokenData) {
        // name of the token
        print_r($tokenData['name']);
        // -> token_name

        // parameters (parsed as an array)
        print_r($tokenData['params']);
        // -> Array
        // (
        //     [0] => foo
        //     [1] => bar
        //     [a_key] => a_value
        //     [a_multivalue_field] => Array
        //         (
        //             [0] => value1
        //             [1] => value2
        //         )
        // )

        // the raw parameter string, if any
        print_r($tokenData['raw_params']);
        // -> foo::bar::a_key=a_value::a_multivalue_field=value1|value2

        // the complete original token as it appeared in the text, in case it's needed
        print_r($tokenData['original']);
        // -> {{{token_name::foo::bar::a_key=a_value::a_multivalue_field=value1|value2}}}
    }
];
```

Note the way positional and named parameters are parsed, as well as the array that the piped multi-value parameter produces.

### Example #1: Random number with limits

You may want to allow a minimum and maximum as parameters for the `random_number` token above. You can decide if you want to use positional or named parameters. In this case, since it would be nice to be able to omit either one, I will use named parameters. This requires just a small adjustment in the callback:

```php
// site/ready.php
wire()->addHookAfter('CachePlaceholders::getTokens', function (HookEvent $e) {
    $tokens = $e->return;
    $tokens['random_number'] = [
        'callback' => function (array $tokenData) {
            $min = (int) ($tokenData['params']['min'] ?? 0);
            $max = (int) ($tokenData['params']['max'] ?? getrandmax());
            return rand($min, $max);
        }
    ];
    $e->return = $tokens;
});
```

Note that you need to cast to integer because all parameters are parsed as strings. Now you can produce random numbers in a custom range:

```php
$text = 'A very large number: {{{random_number::min=2147483642}}} <br>' .
    'A very small number: {{{random_number::max=5}}} <br>' .
    'A number between 5 and 15: {{{random_number::min=5::max=15}}}';

$CachePlaceholders = wire('modules')->get('CachePlaceholders');
echo $CachePlaceholders->replaceTokens($text);
// A very large number: 2147483645
// A very small number: 3
// A number between 5 and 15: 7
```

### Example #2: Personal greeting

As another more practical example, let's get back to the problem from the introduction: Putting a personal greeting on your site. For demonstration purposes, let's say we want to accept a single parameter containing the name of the HTML element used to wrap the greeting.

```php
// site/ready.php
wire()->addHookAfter('CachePlaceholders::getTokens', function (HookEvent $e) {
    $tokens = $e->return;
    $tokens['greeting'] = [
        'callback' => function (array $tokenData) {
            // we'll use positional arguments this time
            $element = $tokenData['params'][0] ?? 'h1';
            $user = wire('user');
            $username = !$user->isGuest() ? ucfirst($user->name) : 'honoured guest';
            return sprintf('<%1$s>Hello %2$s!</%1$s>', $element, $username);
        }
    ];
    $e->return = $tokens;
});
```

You can use this one with or without the parameter (note the fallback to `h1` in the callback, make sure to use the [null coalescing operator](https://www.php.net/manual/en/migration70.new-features.php#migration70.new-features.null-coalesce-op) so you don't get warnings).

```php
$CachePlaceholders = wire('modules')->get('CachePlaceholders');

echo $CachePlaceholders->replaceTokens('{{{greeting}}}');
// Output for user "admin":
// -> <h1>Welcome Admin!</h1>
// Ouput for the guest user:
// -> <h1>Welcome honoured guest!</h1>

echo $CachePlaceholders->replaceTokens('{{{greeting::mark}}}');
// -> <mark>Welcome Admin!</mark>
// -> <mark>Welcome honoured guest!</mark>
```

## Built-in tokens

The module comes with a couple of built-in tokens that you can use directly, adapt to your needs or modify through hooks.

### superglobal: Accessing session, request or cookie data

- Token name: `superglobal`
- Hook: `CachePlaceholders::tokenSuperglobal`

Use this token if you want to dynamically output data from one of PHP's [superglobals](https://www.php.net/manual/en/language.variables.superglobals.php). The most common use-case for this is accessing a string stored in the session or a cookie. It requires two parameters: The name of the superglobal you want to access (in lowercase, without the `$_` prefix) and the key of the value you want to display. You can use it like this:

```php
$CachePlaceholders = wire('modules')->get('CachePlaceholders');

// set a value for testing purposes
$_SESSION['foobar'] = 'Some dynamic text';

// the token parameters are the superglobal to get values from (session) and the array key to retrieve (foobar)
$text = '{{{superglobal::session::foobar}}}';
echo $CachePlaceholders->replaceTokens($text);
// -> Some dynamic text

// you can also get nested values, use a multi-value paramter for the array path
$_SESSION['path']['to']['nested']['data'] = 'Nested session data';
$text = '{{{superglobal::session::path|to|nested|data}}}';
echo $CachePlaceholders->replaceTokens($text);
// -> Nested session data
```

This is the list of superglobals you can access this way: `session` | `request` | `get` | `post` | `cookie` | `server` | `env`

This token is also intended as a starting point for you to copy and adjust to your own needs. You can use the public helper method `CachePlaceholders::getDataFromArrayByPath` for retrieving data from nested arrays.

### csrf: CSRF-Input for forms

- Token name: `csrf`
- Hook: `CachePlaceholders::tokenCSRF`

This token is a thin wrapper around [SessionCSRF::renderInput](https://processwire.com/api/ref/session-c-s-r-f/render-input/). You can specify the optional ID as the first parameter. Usage:

```php
$CachePlaceholders = wire('modules')->get('CachePlaceholders');

$text = '{{{csrf}}}';
echo $CachePlaceholders->replaceTokens($text);
// <input type="hidden" name="TOKEN1683095992X1589268804" value=".w8pIclLB/6MkxSxZ7Kq/VmHPv1DNyzs" class="_post_token">

$text = '{{{csrf::foo_bar}}}';
echo $CachePlaceholders->replaceTokens($text);
// <input type="hidden" name="TOKEN1277485165X1589292868" value="G.bW4MFIi6aYbu/mv0E2oX.HVD2MuvO." class="_post_token">
```

You can use this placeholder inside custom forms and validate it using [SessionCSRF::hasValidToken](https://processwire.com/api/ref/session-c-s-r-f/has-valid-token/).

### random_hex: Cryptographically secure hexdecimal tokens

- Token name: `random_hex`
- Hook: `CachePlaceholders::tokenRandomHex`

This token generates a random hexadecimal string of the specified length (default: 16). This is just a wrapper around `bin2hex(random_bytes($length))`. Because of the way those functions work, it can only generate even-length strings. Usage:

```php
$CachePlaceholders = wire('modules')->get('CachePlaceholders');

$text = '{{{random_hex}}}';
echo $CachePlaceholders->replaceTokens($text);
// bb485fa44b8da5ad

$text = '{{{random_hex::6}}}';
echo $CachePlaceholders->replaceTokens($text);
// 793cf9
```

Since [random_bytes is cryptographically secure](https://www.php.net/manual/en/function.random-bytes.php), you can use this for nonce tokens or similar. You can hook after the token method to save the returned token to the session.

## Changing the token format

You can change all the delimiters used by this module, in case the curly brackets clash with some other shortcode format or template system you are using. Go to the module configuration and change any delimiters you need under `Token formats & delimiters`. Those settings affect both your own as well as the built-in tokens, so make sure to adjust any tokens you're using to the new format. Note that the module generates a regular expression using the delimiters you provide, which is somewhat unpredictable. So if you're trying to break the module with weird delimiter settings, you probably can.
