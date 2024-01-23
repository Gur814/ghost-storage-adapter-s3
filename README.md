# Ghost storage adapter S3

THIS FORK IS A WORK-IN-PROGRESS. PLEASE DON'T USE IT YET.

An AWS S3 storage adapter for Ghost 5.x. Forked from [ghost-storage-adapter-s3](https://github.com/Gur814/ghost-storage-adapter-s3). Without this original work by @colinmeinke and others, this would not have been possible.

## Changes from the base repo

1) Your bucket no longer has to be public. This adapter uses an IAM user to serve files from S3 using your server as a proxy. A web browser will see your domain for files in S3 using this adapter, instead of S3 links.
2) Fixed a crash when trying to upload videos to S3. Ghost was calling a function that didn't exist when the original S3 adapter was written.
3) Image paths in your ghost's database use relative paths, instead of S3 URLs, just like the native Ghost file adapters.
    - Original S3 adapter behavior would save an image's location like this: `https://bucket-name.s3.us-west-2.amazonaws.com/2024/01/my-image.jpg`
        - Since it saves the full S3 URL to your database, you'll need to manually update every single post if you ever switch your blog away from S3 hosting.
    - This S3 adapter's behavior: `/content/images/2024/01/my-image.png`
        - Since this fork uses the same relative paths that Ghost does, you can just copy your folders from S3 to your new file host and switch the adapter without touching your database or posts. To a web browser, it will look as if your images never moved at all.
    - *Caveat*: Most other adapters I've seen will save the full paths as well.
4) Ghost added a way to specify different adapters for different file types: `images`, `media`, and `files`. This adapter is split into 3 parts: one for each type. You can mix and match with other adapters, including the default local file system adapters built into Ghost. For example, if you only want to host images on S3, but keep files hosted locally on your server, you can just install and configure the `s3-images` adapter from this repo.


## AWS Configuration

You should create a new S3 bucket for your blog with a specific IAM role. If you are hosting multiple blogs, I recommend setting up a bucket for each. You can get away with hosting them all in the same bucket, but it's much cleaner to use separate buckets in the event that you need to decouple the blogs in the future.

*Caveat*: I have not configured a CDN for my blog so I am unsure of the appropriate steps. I left the original author's CDN instructions below in case they're helpful. Please submit a PR if you know how to do that so we can update the steps.

### S3 Setup

#### Bucket Creation

- Create a new bucket
- Region and name are whatever you prefer.
- Keep ACLs disabled.
- Keep public access blocked.
- I chose SSE-S3 encryption with Bucket Key enabled. It should work with other settings.

#### IAM

You will now create an IAM user that will access your bucket's image/file data from your Ghost blog. Go to the IAM console.

##### Policy

- Click `Policies` in the sidebar and create a new policy.
- Click on `JSON` and paste in the following object. **Don't forget to change the bucket name!** Set `ghost-bucket` in this JSON object to whatever your bucket name is. There are two places to put your bucket name.

```JSON
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::ghost-bucket"
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:PutObjectVersionAcl",
                "s3:DeleteObject",
                "s3:PutObjectAcl"
            ],
            "Resource": "arn:aws:s3:::ghost-bucket/*"
        }
    ]
}
```

- Click `Next` and give your policy a name.
- Finish creating the policy

##### User

- Click `Users` in the IAM sidebar and create a new one.
- They do not need access to the AWS management console, so leave those settings off.
- On Step 2, click `Attach policies directly` and search for the policy you created in the previous section. Select it, then finish creating your user.
- Click on your new user and select `Security Credentials`.
- Scroll down to `Access Keys` and click `Create access key`.
- For `Use case`, select `Other` then `Next`.
- Save the `Access key` and `Secret access key` in a safe place. We'll use them when configuring the adapters

Now that your bucket, IAM User, IAM Policy, and access keys are created, you can install the adapter. You'll need to know your `bucket name`, `bucket region`, `IAM User Access key`, `IAM User Secret access key`, and bucket server-side encryption choice. If you chose `SSE-S3`, this maps to `AES256` in the config.

https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html
Search for `ServerSideEncryption` for more info.

## Installation

I recommend making a backup of your site before proceeding. It is unlikely that you'll break anything, but it's good practice before messing with server files.

I do not currently have this hosted on NPM. Sorry. If you'd like to help me with that, please reach out! I've never done it before.

Basically, these adapters are installed in `/var/www/<your-site>/content/adapters/storage/`. Ghost will look in this directory for a Javascript file or a directory that matches the adapter name provided in your Ghost config file. If it finds a directory, it looks for `index.js` at the root.

```shell
cd /var/www/<your-site>
# TODO
mkdir -p ./content/adapters/storage
# cp -r ./node_modules/ghost-storage-adapter-s3 ./content/adapters/storage/s3
```

## Configuration

In `/var/www/<your-site>/config.production.json`.

This is a little messy since there are 3 different adapters. I haven't figured out a way to share config details between adapters. If you are only hosting a single blog, environment variables may be a better choice. But this is also pretty easy.

```json
"storage": {
  "active": "s3",
  "media": "s3-media",
  "images": "s3-images",
  "files": "s3-files",
  "s3-*": {
    "accessKeyId": "YOUR_ACCESS_KEY_ID",
    "secretAccessKey": "YOUR_SECRET_ACCESS_KEY",
    "region": "YOUR_REGION_SLUG",
    "bucket": "YOUR_BUCKET_NAME",
    "assetHost": "YOUR_OPTIONAL_CDN_URL (See note 1 below)",
    "signatureVersion": "REGION_SIGNATURE_VERSION (See note 5 below)",
    "pathPrefix": "YOUR_OPTIONAL_BUCKET_SUBDIRECTORY",
    "endpoint": "YOUR_OPTIONAL_ENDPOINT_URL (only needed for 3rd party S3 providers)",
    "serverSideEncryption": "YOUR_OPTIONAL_SSE (See note 2 below)",
    "forcePathStyle": true,
    "acl": "YOUR_OPTIONAL_ACL (See note 4 below)",
  }
}
```
Note 1: Be sure to include "//" or the appropriate protocol within your assetHost string/variable to ensure that your site's domain is not prepended to the CDN URL.

Note 2: if your s3 bucket enforces SSE use serverSideEncryption with the [appropriate supported](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html#putObject-property) value.

Note 3: if your s3 providers requires path style you can enable it with `forcePathStyle`

Note 4: if you use CloudFront the object ACL does not need to be set to "public-read"

Note 5: [Support for AWS4-HMAC-SHA256](https://github.com/colinmeinke/ghost-storage-adapter-s3/issues/43)

### Via environment variables

```
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_DEFAULT_REGION
GHOST_STORAGE_ADAPTER_S3_PATH_BUCKET
GHOST_STORAGE_ADAPTER_S3_ASSET_HOST  // optional
GHOST_STORAGE_ADAPTER_S3_PATH_PREFIX // optional
GHOST_STORAGE_ADAPTER_S3_ENDPOINT // optional
GHOST_STORAGE_ADAPTER_S3_SSE // optional
GHOST_STORAGE_ADAPTER_S3_FORCE_PATH_STYLE // optional
GHOST_STORAGE_ADAPTER_S3_ACL // optional
```

## OLD

### IAM (Old)

What this policy does is allow the user access to see the contents of the bucket (the first statement), and then manipulate the objects stored in the bucket (the second statement).

Finally, create the user and copy the **Access key** and **Secret access key**, these are what you'll use in your configuration.

At this point you could be done, but, optionally, you could put Amazon's CloudFront CDN in front of the bucket to speed things up.

### CloudFront

**WARNING**: These steps are copied from the base repo and are untested with this fork. If you figure out Cloudfront with the updated configurations of this fork, let me know so we can update this section. I've left the original author's instructions here for reference.

CloudFront is a CDN that replicates objects in servers around the world so your blog's visitors will get your assets faster by using the server closest to them. It uses your S3 bucket as the "source of truth" that it populates its servers with.

Got to CloudFront in AWS and choose to **Create a Distribution**. On the next screen you'll want to leave everything the same, except change the following:
- Origin Domain Name: **Set this to the Endpoint url listed in the Static website hosting panel in the S3 bucket configuration**
- Viewer Protocol Policy: **Redirect HTTP to HTTPS**
- Compress Objects Automatically: **Yes**

Then create the distribution.

Next you'll want to configure your domain name to point a subdomain at CloudFront so you can serve static content through the CDN. Click on the distribution you just created and go the General tab. In Alternate Domain Names, add a subdomain from your url to be the CDN. For instance, if your domain is *yourdomain.com*, do something like *cdn.yourdomain.com*.

Next, you'll want to enable SSL. If you're already using Amazon's Route53 DNS service, you may already have an SSL certificate for your domain with a wildcard, if not, choose to create one for your subdomain. If you're using Route53 you can have them automatically add the proper entries to your DNS records for validation and have the certificate generated. If not, go through the alternate route.

Next, configure the DNS entry for the subdomain for CloudFront. Go to your DNS configuration and add an A record for **cdn** (or whatever subdomain your chose), and then set it up as an alias that points at your CloudFront distribution URL. If you're using Route53 it will actually provide you with distribution as an option.

Finally, in your configuration, use the subdomain for the CloudFront distribution as your setting for *assetHost*.

## License

[ISC](./LICENSE.md)
