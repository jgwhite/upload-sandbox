# Upload Sandbox

Thoughts on implementing direct upload to S3 in an Ember app.

## File Proxy Mixins

**Problem:** We want to extend the interface of native javascript `File` objects.

**Solution:** Use Ember’s `ObjectProxy` and some specialised mixins.

### Example: ObjectURLMixin

Provides a computed `url` property by means of `URL.createObjectURL`.

```js
var ObjectURLMixin = Ember.Mixin.create({
  url: function() {
    var file = this.get('content');
    if (file) {
      return window.URL.createObjectURL(file);
    }
  }.property('content').readOnly()
});

var FileProxy = Ember.ObjectProxy.extend(ObjectURLMixin);

// Assuming file has been received from a file input or via drag/drop
var fileProxy = FileProxy.create({ content: file });
fileProxy.get('url'); // => 'blob:...'
```

### Example: S3UploadMixin

When the content of the proxy changes to a new file (and/or on `init`), it would:

1. Fetch a signed aws authorization from the API
2. Post the file to the desired bucket/object
3. As upload is in progress, update a `progress` object
4. On completion, update the `url` property with the newly available S3 url

This allows us to pass an object around the system that automatically gains a
`url` property at some later point in time. Bindings take care of the rest.

```js
var S3UploadMixin = Ember.Mixin.create({
  state: null,
  progress: null,
  url: null,

  upload: function() {
    // TODO Fetch authorization
    // TODO Upload file
    // TODO Update progress
    // TODO Set url property
    // TODO Return a promise?
  }.observes('content').on('init')
});

var FileProxy = Ember.ObjectProxy.extend(S3UploadMixin);

// Assuming file has been received from a file input or via drag/drop
var fileProxy = FileProxy.create({ content: file });
fileProxy.get('state'); // => 'authorizing'
// Some time later...
fileProxy.get('state'); // => 'uploading'
fileProxy.get('progress'); // => 0.2
// Some time later...
fileProxy.get('url'); // => 'http://s3.amazonaws.com/bucket/file.png'
```

The above is bindings/observer friendly, but a promise-driven interface may
also be desirable.

```js
fileProxy.upload().then(function(url) {
  // ...
}, function(error) {
  // ...
});
```

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
