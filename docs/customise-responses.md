# How to customise responses (HTTP codes, headers, content types)

Claudia automatically configures your API for the most common use case, the way javascript developers expect. The HTTP response code 500 will be used for all runtime errors, and 200 for successful operations. The `application/json` content type is default for both successes and failures. 

Of course, you can easily change all those parameters, add your own custom headers and respond with different codes. There are several ways to configure the response:

* [Static configuration](#static-configuration), when all responses on a route should always use the same code and headers
* [Use an ApiResponse object](#apiresponse-object), when you want to decide at runtime which HTTP response code/headers to use
* [Define Custom API Gateway responses](#api-gateway-responses), when you want to configure API Gateway errors (such as 'method not found')

In addition, Claudia API builder has a few nice shortcuts for managing Cross-Origin Resource Sharing (CORS). Check out the [Configuring CORS](cors.md) page for more information.

## Static configuration

You can provide static configuration to a handler, by setting the third argument of the method. All the keys are optional, and the structure is:

* `error`: a number or a key-value map. If a number is specified, it will be used as the HTTP response code. If a key-value map is specified, it should have the following keys:
  * `code`: HTTP response code
  * `contentType`: the content type of the response
  * `headers`: a key-value map of hard-coded header values, or an array enumerating custom header names. See [Custom headers](#custom-headers) below for more information
* `success`: a number or a key-value map. If a number is specified, it will be used as the HTTP response code. If a key-value map is specified, it should have the following keys:
  * `code`: HTTP response code
  * `contentType`: the content type of the response
  * `headers`: a key-value map of hard-coded header values, or an array enumerating custom header names. See [Custom headers](#custom-headers) below for more information
* `apiKeyRequired`: boolean, determines if a valid API key is required to call this method. See [Requiring Api Keys](#requiring-api-keys) below for more information

For example:

```javascript
api.get('/greet', function (request) {
	return request.queryString.name + ' is ' + superb();
}, {
  success: { contentType: 'text/plain' }, 
  error: {code: 403}
});

```

These special rules apply to content types and codes:

  * When the error content type is `text/plain` or `text/html`, only the error message is sent back in the body, not the entire error structure.
  * When the error content type is `application/json`, the entire error structure is sent back with the response.
  * When the response type is `application/json`, the response is JSON-encoded. So if you just send back a string, it will have quotes around it.
  * When the response type is `text/plain`, `text/xml`, `text/html` or `application/xml`, the response is sent back without JSON encoding (so no extra quotes). 
  * In case of 3xx response codes for success, the response goes into the `Location` header, so you can easily create HTTP redirects.

To see these options in action, see the  [Serving HTML Example project](https://github.com/claudiajs/example-projects/tree/master/web-serving-html).

You can also configure header values in the configuration (useful for ending sessions in case of errors, redirecting to a well-known location after log-outs etc),  use the `success.headers` and `error.headers` keys. To do this, list headers as key-value pairs. For example: 

  ```javascript
  api.get('/hard-coded-headers', function () {
  	return 'OK';
  }, {success: {headers: {'X-Version': '101', 'Content-Type': 'text/plain'}}});
  ```

## ApiResponse object

To decide at runtime which HTTP response code/headers to use, instead of a string or JSON object, return or throw an instance of `ApiBuilder.ApiResponse`. This will allow you to dynamically set headers and the response code. 

```javascript
new ApiResponse(body, headers, httpCode)
```

* `body`: string &ndash; the body of the response
* `header`: object &ndash; key-value map of header names to header values, all strings
* `httpCode`: numeric response code. Defaults to 200 for successful responses and 500 for errors.

Here's an example:
```javascript
api.get('/programmatic-headers', function () {
  return new ApiBuilder.ApiResponse('OK', {'X-Version': '202', 'Content-Type': 'text/plain'}, 204);
});
```

You could throw:
```javascript
api.get('/error', function () {
  throw new ApiBuilder.ApiResponse('<error>NOT OK</error>', {'Content-Type': 'text/xml'}, 500);
});
```

Or resolve/reject:
```javascript
api.get('/heartbeat', () =>
  gateway.heartbeat()
    .then(() => new ApiBuilder.ApiResponse({healthy:true}, 200))
    .catch(() => new ApiBuilder.ApiResponse({healthy:false}, 503))
});
```

To see custom headers in action, see the [Custom Headers Example Project](https://github.com/claudiajs/example-projects/blob/master/web-api-custom-headers/web.js).

## API Gateway Responses

_since claudia-api-builder 3.0.0, claudia 3.0.0_

API Gateway supports customising error responses generated by the gateway itself, without any interaction with your Lambda. This is useful if you want to provide additional headers with an error response, or if you want to change the default behaviour for unmatched paths, for example.

To define a custom API Response, use the `setGatewayResponse` method. The syntax is:

```javascript
api.setGatewayResponse(responseType, responseConfig)
```

* `responseType`: `string` &ndash; one of the supported [API Gateway Response Types](http://docs.aws.amazon.com/apigateway/api-reference/resource/gateway-response/).
* `config`: `object` &ndash; the API Gateway Response Configuration, containing optionally `statusCode`, `responseParameters` and `responseTemplates`. Check the [API Gateway Response Types](http://docs.aws.amazon.com/apigateway/api-reference/resource/gateway-response/) for more information on those parameters.


Here's an example:

```javascript
api.setGatewayResponse('DEFAULT_4XX', {
  responseParameters: {
    'gatewayresponse.header.x-response-claudia': '\'yes\'',
    'gatewayresponse.header.x-name': 'method.request.header.name',
    'gatewayresponse.header.Access-Control-Allow-Origin': '\'a.b.c\'',
    'gatewayresponse.header.Content-Type': '\'application/json\''
  },
  statusCode: 411,
  responseTemplates: {
    'application/json': '{"custom": true, "message":$context.error.messageString}'
  }
});
```

In addition to the standard parameters supported by API Gateway directly, Claudia API Builder also provides a shortcut for setting headers. Use the key `headers`, and a map `string` to `string` of header names to values.


```javascript
api.setGatewayResponse('DEFAULT_4XX', {
  statusCode: 411,
  headers: {
    'x-response-claudia': 'yes',
    'Content-Type': 'application/json',
    'Access-Control-Allow-Origin': 'a.b.c'
    }
  }
);

```

