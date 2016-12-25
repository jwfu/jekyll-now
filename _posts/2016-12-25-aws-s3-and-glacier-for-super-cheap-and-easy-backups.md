---
layout: post
title: AWS S3 and Glacier for Super Cheap and Easy Backups
---

Since I started seriously using computers and taking pictures, I'm lucky to say I have not had an incident of (noticed) data loss by doing regular backups to external memory.

As I'm currently London-based with stable abode, a little while ago I was looking into building a NAS to live in the closet. When investigating options for operating systems, I was really surprised to see that with RAID arrays, there is the possibility of [silent data corruption](http://arstechnica.com/information-technology/2014/01/bitrot-and-atomic-cows-inside-next-gen-filesystems/), which made me worry about long term storage. This got me looking into all of the options for data integrity checking, which pretty much summarised that modern copy-on-write file systems are required to ensure that the bit rot is detected. All this still requires the management of the cluster itself with all of the possible issues with hardware that presents.

Enter AWS.

Amazon has been offering [Glacier](https://aws.amazon.com/glacier/) as their cheapest tier of offline data storage. The [cost structure](https://aws.amazon.com/glacier/pricing/) makes it free to upload data in, with very low monthly storage costs, with the limitation of requiring several hours to retrieve data at relatively high cost. In terms of data integrity:

> Amazon Glacier is designed to provide average annual durability of 99.999999999% for an archive. The service redundantly stores data in multiple facilities and on multiple devices within each facility. To increase durability, Amazon Glacier synchronously stores your data across multiple facilities before returning SUCCESS on uploading archives. Glacier performs regular, systematic data integrity checks and is built to be automatically self-healing.

I'll describe my workflow for getting data into Glacier, as well as my preferred settings to do so. This is mostly using high-level AWS CLI commands, but I also created an `.ipynb` which uses boto3 to do some low-level monitoring and management of multi-part uploads.

# Set up Lifecycle Rules
S3 allows you to set up lifecycle rules for your files that define when a file transition to infrequent access (cheaper than normal S3 but more expensive than Glacier), Glacier, and be deleted for both the current and old versions.  It is also possible here to set up when to discard multipart uploads.  My goals are:
+ Send the files to Glacier as soon as possible (1 day).
+ Give all multipart uploads a chance to finish before killing it (7 days).

This gives me the following Rules when I run [`get_bucket_lifecycle()`](http://boto3.readthedocs.io/en/latest/reference/services/s3.html#S3.Client.get_bucket_lifecycle) in Boto3:
~~~py
'Rules': [{
        'AbortIncompleteMultipartUpload': {
            'DaysAfterInitiation': 7
        },
        'ID': 'auto glacier',
        'Prefix': '',
        'Status': 'Enabled',
        'Transition': {
            'Days': 1,
            'StorageClass': 'GLACIER'
        }
    }]
~~~
# AWS S3 Sync

`aws s3 sync` is the main command I used, which worked great for my requirements:

- One-way syncho so files which exist in the Glacier archive but are not present in on my machine will not get deleted. This means the archive will have all the files of the latest sync, as well as all previous syncs. Two-way synchro is possible as well using the `--delete` flag.
- Determines which files are new or revised automatically.
- Does multipart uploads of large files, allowing for concurrent uploads of multiple parts, and resume download for flaky connections. This turned out to work in a strange way (more on this next).

# Monitoring all of this using Boto3
Some of the strange behaviour I saw was:
+ Files getting stuck, as evidenced by successful upload of preceding and succeeding files it a directory
+ Multipart files making the terminal hang with no visibility as to whether the upload is proceeding

To investigate these, I used Boto3 in Jupyter Notebook.  First, a tool to determine all of the uploads underway:

~~~py
# view all incomplete multipart uploads.  really useful for cleaning up when
# high level s3 commands are used which abstract away the visibiity of any partial uploads.
client = boto3.client('s3')

lst = client.list_multipart_uploads(Bucket = bucketname)
pd.DataFrame(lst['Uploads']) #for pretty printing
~~~

And a tool to use this data to delete suck multipart uploads:

~~~py
# utility to clear partial uploads given the output of list_multipart_uploads
# and the index items to clear.

def clearupload(num, lst):
    lst['Uploads']

    filename = lst['Uploads'][num]['Key']
    fileid = lst['Uploads'][num]['UploadId']

    client.abort_multipart_upload(
    Bucket = bucketname,
    Key=filename,
    UploadId=fileid,
    RequestPayer='requester')
~~~

# Conclusions
Using this method, I have approx 150 GB in storage at a cost of $0.62 per month.  Compared to building a NAS at several hundred dollars, I have found using AWS Glacier to be cost-effective.

If anyone has any feedback about my process (especially ways to make it better!) do get in touch.
