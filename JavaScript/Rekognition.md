The following is an example of how JavaScript SDK users can implement exponential backoff/retry above and beyond the SDK’s built-in retry mechanism:

```js
// jshint esversion:6
// jshint node: true
 
const aws = require('aws-sdk');
const rekognition = new aws.Rekognition();
 
// OLD method that will fail if API is rate limited
function SearchFaces(params) {
    return rekognition.searchFacesByImage(params).promise();
}
 
const backoff_promise = require('backoff-promise');
const backoff = new backoff_promise();
 
// NEW utility method to test for rate limiting and cause retry
var shouldBackoffRetry = function(err) {
    if (err.code !== 'ProvisionedThroughputExceededException') {
        throw err;
    }
};
 
// NEW method that implements exponential backoff/retry
function SearchFaces(params) {
    return backoff.run(function() {
        return rekognition.searchFacesByImage(params).promise();
    }, shouldBackoffRetry);
}
```