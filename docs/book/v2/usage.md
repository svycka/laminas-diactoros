# Usage

Usage will differ based on whether you are writing an HTTP client, or a server-side application.

For HTTP client purposes, you will create and populate a `Request` instance, and the client should
return a `Response` instance.

For server-side applications, you will create a `ServerRequest` instance, and populate and return a
`Response` instance.

## HTTP Clients

A client will _send_ a request, and _return_ a response. As a developer, you will _create_ and
_populate_ the request, and then _introspect_ the response.  Both requests and responses are
immutable; if you make changes &mdash; e.g., by calling setter methods &mdash; you must capture the return
value, as it is a new instance.

```php
// Create a request
$request = (new Laminas\Diactoros\Request())
    ->withUri(new Laminas\Diactoros\Uri('http://example.com'))
    ->withMethod('PATCH')
    ->withAddedHeader('Authorization', 'Bearer ' . $token)
    ->withAddedHeader('Content-Type', 'application/json');

// OR:
$request = new Laminas\Diactoros\Request(
    'http://example.com',
    'PATCH',
    'php://memory',
    [
        'Authorization' => 'Bearer ' . $token,
        'Content-Type'  => 'application/json',
    ]
);

// If you want to set a non-origin-form request target, set the
// request-target explicitly:
$request = $request->withRequestTarget((string) $uri);       // absolute-form
$request = $request->withRequestTarget($uri->getAuthority()); // authority-form
$request = $request->withRequestTarget('*');                 // asterisk-form

// Once you have the instance:
$request->getBody()->write(json_encode($data));
$response = $client->send($request);

printf("Response status: %d (%s)\n", $response->getStatusCode(), $response->getReasonPhrase());
printf("Headers:\n");
foreach ($response->getHeaders() as $header => $values) {
    printf("    %s: %s\n", $header, implode(', ', $values));
}
printf("Message:\n%s\n", $response->getBody());
```

(Note: `laminas-diactoros` does NOT ship with a client implementation; the above is just an
illustration of a possible implementation.)

## Server-Side Applications

Server-side applications will need to marshal the incoming request based on superglobals, and will
then populate and send a response.

### Marshaling an incoming Request

PHP contains a plethora of information about the incoming request, and keeps that information in a
variety of locations. `Laminas\Diactoros\ServerRequestFactory::fromGlobals()` can simplify marshaling
that information into a request instance.

You can call the factory method with or without the following arguments, in the following order:

- `$server`, typically `$_SERVER`
- `$query`, typically `$_GET`
- `$body`, typically `$_POST`
- `$cookies`, typically `$_COOKIE`
- `$files`, typically `$_FILES`

The method will then return a `Laminas\Diactoros\ServerRequest` instance. If any argument is omitted,
the associated superglobal will be used.

```php
$request = Laminas\Diactoros\ServerRequestFactory::fromGlobals(
    $_SERVER,
    $_GET,
    $_POST,
    $_COOKIE,
    $_FILES
);
```

When no cookie array is supplied, `fromGlobals` will first try to parse the supplied `cookie` header
before falling back to the `$_COOKIE` superglobal. This is done because PHP has some legacy handling
for request parameters which were then registered as global variables. Due to this, cookies with a period
in the name were renamed with underlines. By getting the cookies directly from the cookie header, you have
access to the original cookies in the way you set them in your application and they are send by the user
agent.

### Manipulating the Response

Use the response object to add headers and provide content for the response.  Writing to the body
does not create a state change in the response, so it can be done without capturing the return
value. Manipulating headers does, however.

```php
$response = new Laminas\Diactoros\Response();

// Write to the response body:
$response->getBody()->write("some content\n");

// Multiple calls to write() append:
$response->getBody()->write("more content\n"); // now "some content\nmore content\n"

// Add headers
// Note: headers do not need to be added before data is written to the body!
$response = $response
    ->withHeader('Content-Type', 'text/plain')
    ->withAddedHeader('X-Show-Something', 'something');
```
