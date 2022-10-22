---
title: "Stream unzip files in S3 with Java"
date: 2022-10-22
tags: ["Java", "S3", "AWS"]
categories: Software Engineering
---

I've been spending a lot of time with AWS S3 recently building data pipelines and have encountered a surprisingly non-trivial challenge of unzipping files in an S3 bucket. 
A few minutes with Google and StackOverflow made it clear many others have faced the same issue.

I'll explain a few options to handle the unzipping as well as the end solution which has led me to build [nejckorasa/s3-stream-unzip](https://github.com/nejckorasa/s3-stream-unzip).

To sum up: 
- there is no support to unzip files in S3 in-line,
- there also is no unzip built-in api available in AWS SDK.


In order to unzip you therefore need to download the files from S3, unzip and upload decompressed files back. 

This solution is simple to implement with the use of [Java AWS SDK](https://aws.amazon.com/sdk-for-java/), and it probably is good enough if you are dealing with smaller files - if files are small enough you can just keep hold of decompressed files in memory and upload them back. 

Alternatively, in case of memory constraints, files can be persisted to disk storage. Great, that works.

Problems arise with larger files. AWS Lambda, for example, [has a 1024MB memory and disk space limit](https://aws.amazon.com/lambda/faqs/). A dedicated EC2 instance will solve the disk space issue, but it requires more maintenance. I'd also argue that storing 500MB+ files to disk is not the most optimal approach. 
That will of course depend on how many files need to be unzipped as well as the run frequency of that operation - it's ok as a one-off but maybe not so if it needs to run daily. In any case, we really can do better.

### Streaming solution

A better approach would be to stream the file from S3, download it in chunks, unzip and upload them back to S3 utilizing multipart upload. That way you completely avoid the need for disk storage and you can minimize the memory footprint by tuning the download and upload chunk sizes.

There are 2 parts of this solution that need to be integrated:

#### 1) Download and uznip

Streaming S3 objects is natively supported by AWS SDK, there is a `getObjectContent()` method that returns the input stream containing the contents of the S3 object.

Java provides [ZipInputStream](https://docs.oracle.com/javase/7/docs/api/java/util/zip/ZipInputStream.html) as an input stream filter for reading files in the ZIP file format. It reads ZIP content entry-by-entry and thus allows custom handling for each entry.

Streaming object content from S3 and feeding that into `ZipInputStream` will give us decompressed chunks of object content we can buffer in memory.

#### 2) Upload unzipped chunks to S3

Uploading files to S3 is a common task and SDK supports several options to choose from, including [multipart upload](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html). 

> What is multipart upload?
> 
> Multipart upload allows you to upload a single object as a set of parts. 
> Each part is a contiguous portion of the object's data. You can upload these object parts independently and in any order. 
> If transmission of any part fails, you can retransmit that part without affecting other parts. 
> 
> After all parts of your object are uploaded, Amazon S3 assembles these parts and creates the object. 
>
> In general, when your object size reaches 100 MB, you should consider using multipart uploads instead of uploading the object in a single operation.

### nejckorasa/s3-stream-unzip

All that is left to do now is to integrate stream download, unzip, and multipart upload. 
I've done all the hard work and built [nejckorasa/s3-stream-unzip](https://github.com/nejckorasa/s3-stream-unzip).

> Java library to manage unzipping of large files and data in AWS S3 without knowing the size beforehand and without keeping it all in memory or writing to disk.

Unzipping is achieved without knowing the size beforehand and without keeping it all in memory or writing to disk. That makes it suitable for large data files - it has been used to unzip files of size 100GB+.

It supports different unzip strategies including an option to split zipped files (suitable for larger files, e.g. csv files). It's lightweight and only requires an AmazonS3 client to run.

It has a simple API:

```java
// initialize AmazonS3 client
AmazonS3 s3CLient = AmazonS3ClientBuilder.standard()
        // customize the client
        .build()

// create UnzipStrategy
var strategy = new NoSplitUnzipStrategy();
var strategy = new SplitTextUnzipStrategy()
        .withHeader(true)
        .withFileBytesLimit(100 * MB);

// or create UnzipStrategy with additional config
var config = new S3MultipartUpload.Config()
        .withThreadCount(5)
        .withQueueSize(5)
        .withAwaitTerminationTimeSeconds(2)
        .withCannedAcl(CannedAccessControlList.BucketOwnerFullControl)
        .withUploadPartBytesLimit(20 * MB)
        .withCustomizeInitiateUploadRequest(request -> {
            // customize request
            return request;
        });

var strategy = new NoSplitUnzipStrategy(config);

// create S3UnzipManager
var um = new S3UnzipManager(s3Client, strategy);
var um = new S3UnzipManager(s3Client, strategy.withContentTypes(List.of("application/zip"));

// unzip options
um.unzipObjects("bucket-name", "input-path", "output-path");
um.unzipObjectsKeyMatching("bucket-name", "input-path", "output-path", ".*\\.zip");
um.unzipObjectsKeyContaining("bucket-name", "input-path", "output-path", "-part-of-object-");
um.unzipObject(s3Object, "output-path");
```

Library is available on [Maven Central](https://search.maven.org/artifact/io.github.nejckorasa/s3-stream-unzip/1.0.1/jar) and on [Github](https://github.com/nejckorasa/s3-stream-unzip).