---
title: Sharing Private S3 Object With Presigned URLs
date: 2020-06-21T12:00:00.000
author: Mike Wzorek
summary: I came across a problem at work the other day where I needed to be able to show a preview of, and allow our users to download, a variety of different types of content from a web app, i.e. videos, images, open office docs, pdfs, etc. that are stored in a private S3 bucket. The solution that I found, was to generate a pre-signed URL.  A pre-signed URL allows you to grant temporary access to users who don’t have permission to directly run AWS operations in your account. A pre-signed URL is signed with your AWS credentials and can be used by any user.
tags:
  - aws
  - s3
  - nodejs
  - typescript
---

I came across a problem at work the other day where I needed to be able to show a preview of, and allow our users to download, a variety of different types of content from a web app, i.e. video, image, Open Office files, PDF, etc. that are stored in a private S3 bucket

The solution that I found was to generate a pre-signed URL.  A pre-signed URL allows you to grant temporary access to users who don’t have permission to directly run AWS operations in your account. A pre-signed URL is signed with your AWS credentials and can be used by any user.

Our API is built in Node.js so I took advantage of the AWS SDK to get the pre-signed URL.

First install the AWS SDK.
`yarn install aws-sdk` or `npm install aws-sdk`.

Next initialize the S3 client in the class constructor.

```typescript
import AWS, { S3 } from 'aws-sdk';

export class S3Service {
  s3: S3;
  constructor() {
    super();
    AWS.config.update({
      region: 'us-east-1', // the region your s3 bucket is in.
      accessKeyId: process.env.AWS_ACCESS_KEY_ID, // I'm using environment variables to store my secrets
      secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY,
    });
    this.s3 = new S3();
  }
// ...
}
```

There is a few things going on in the implementation, some of which are beyond returning a signed url, but I figured it would be useful to include them.  I’ll list them out in order of operations and include the full code snippet towards the end:

1. Based on the S3 Object `key`  passed into the function I’m grabbing the file type and extension using a lightweight MIME type module, [GitHub - broofa/mime: Mime types for JavaScript](https://github.com/broofa/mime)

```typescript
import mime from 'mime'
// ...
const type = mime.getType(key); // key = '/my/folder/path/object.pdf'
const extension = mime.getExtension(type || '');
// ...
```

2. Set up the parameters to get the signed url.  You will notice a bit of conditional logic in determining the `ResponseContentType`.  What’s going on there is that if the `contentType` variable I instantiated in the first step is an image type, i.e.  jpeg, png, etc., I want to the pre-signed url header to include the content-type header equal to `’binary/octet-stream'`.  Returning the image pre-signed URLs like this allows us to show a preview of the image in our web app, but also make them download-able.

```typescript
const getSignedUrlParams = {
   Bucket: s3Bucket, // 'name-of-your-s3-bucket'
   Key: key, // key = '/my/folder/path/object.pdf'
   Expires: 3600, // time in seconds (3600 = 1 hour).  Default is 15 minutes.
   ResponseContentType: isImageContentType(contentType as string) ? 'binary/octet-stream' : contentType,
};
```

3. Lastly, I call the `getSignedUrlPromise` function and return the string URL response.  I’m using the promise based function because I am making this request in an asynchronous context.

```typescript
const signedUrl = await this.s3.getSignedUrlPromise(‘getObject’, getSignedUrlParams);
return {
    signedUrl,
    contentType: type || ‘’,
    extension: extension || ‘’,
};
```

Bringing it all together, here’s the full implementation:

```typescript
import AWS, { S3 } from 'aws-sdk';
import mime from 'mime';

interface S3SignedUrlResponse {
  signedUrl: string;
  contentType: string;
  extension: string;
}

export class S3Service {
  s3: S3;
  constructor() {
    super();
    AWS.config.update({
      region: 'us-east-1',
      accessKeyId: process.env.AWS_ACCESS_KEY_ID,
      secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY,
    });
    this.s3 = new S3();
  }

  async getS3ObjectSignedUrl(s3Bucket: string, key: string): Promise<S3SignedUrlResponse> {
    try {
      const contentType = mime.getType(key); // key = '/my/folder/path/object.pdf'
      const extension = mime.getExtension(type || '');
      const getSignedUrlParams = {
        Bucket: s3Bucket, // 'name-of-your-s3-bucket'
        Key: key,
        Expires: 3600, // time in seconds (3600 = 1 hour)
        ResponseContentType: isImageContentType(contentType as string) ? 'binary/octet-stream' : contentType,
      };
      const signedUrl = await this.s3.getSignedUrlPromise('getObject', getSignedUrlParams);
      return {
        signedUrl,
        contentType: contentType || '',
        extension: extension || '',
      };
    } catch (error) {
      throw new Error('Error getting a signedUrl');
    }
  }
}
```

As part of my posts I’m going to make it a point to include some content beyond code.  I imagine this will likely take the form of a song I’m enjoying especially while writing code or an insightful article which may or may not be software related.  

I hope you got something out this article.  Feel free to provide any feedback by contacting me via email or social media.

#### Song of the Post

Lane 8 - Keep On
[Spotify](https://open.spotify.com/track/5xMMxb79haEK4MxUzyOr4E?si=GkEuUyNuTmm50wHBZVjo0Q)
[YouTube](https://www.youtube.com/watch?v=6FQ9iS2Ch40)

#### Random Article

As someone who's built enterprise applications in both React and Angular, I do love my frameworks and think they offer a lot of value.  That said, with any choice in technology its important to put thought into why you use or need them for your particular use case.

[Frameworkless Movement](https://www.frameworklessmovement.org/)


#### References

[AWS Docs - Share Object with Presigned URL](https://docs.aws.amazon.com/AmazonS3/latest/dev/ShareObjectPreSignedURL.html)
[Class: AWS.S3 — AWS SDK for JavaScript](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html#getSignedUrl-property)
