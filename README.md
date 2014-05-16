# File Proxy

An example of implementing direct upload to S3 in an Ember app.

## Architecture

- Ember app with FileProxy object
- Rack app with AWS credentials

## FileProxy

A class that wraps javascript file objects and extends their interface.

For example, we might have `ObjectURLFileProxy`, whose job is to wrap a file
and provide a `url` property by means of the native `URL.createObjectURL`
method.

```js
var ObjectURLFileProxy = Ember.ObjectProxy.extend({
  url: function() {
    var file = this.get('content');
    if (file) {
      return window.URL.createObjectURL(file);
    }
  }.property('content').readOnly()
});

// Assuming file has been received from a file input or via drag/drop
var fileProxy = ObjectURLFileProxy.create({ content: file });
fileProxy.get('url'); // => 'blob:...'
```

This concept leads naturally to an S3UploadFileProxy. Given a file, its job is:

1. Fetch a signed aws authorization from the API
2. Post the file to the desired bucket/object
3. As upload is in progress, update a `progress` object
4. On completion, update the `url` property with the newly available S3 url

This allows us to pass an object around the system that automatically gains a
`url` property at some later point in time. Bindings take care of the rest.

```js
var S3UploadFileProxy = Ember.ObjectProxy.extend({
  url: null,

  uploadFile: function() {
    // Fetch authorization
    // Upload file
    // Update progress
    // Set url property
  }.on('init')
});

// Assuming file has been received from a file input or via drag/drop
var fileProxy = S3UploadFileProxy.create({ content: file });
fileProxy.get('state'); // => 'authorizing'
// Some time later...
fileProxy.get('state'); // => 'uploading'
fileProxy.get('progress'); // => 0.2
// Some time later...
fileProxy.get('url'); // => 'http://s3.amazonaws.com/bucket/file.png'
```

The interface above is bindings/observer friendly, but we may also want a
promise-style interface. TBD

## API App

The API needs to provide an authentication mechanism and the ability to
generate upload authorizations.

For example, assuming there were a simple database of email/username
credentials to check against:

```
POST /authorizations
{ "email": "hello@example.com", "password": "secret" }
201 Created
{ token: "ABC123" }

POST /upload-authorizations
Authorization: token ABC123
{ "amz-operation": "object.post" }
201 Created
{ "authorization": "long auth header" }
```
