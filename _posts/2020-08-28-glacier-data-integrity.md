---
layout: post
title: Glacier Data Integrity Testing
---

In a past post several years ago, [I described]({{ site.baseurl }}/aws-s3-and-glacier-for-super-cheap-and-easy-backups/) my backup strategy of using AWS Glacier as my offsite backup in favour of running a NAS.

Although AWS specifies an annual data durability of 99.999999999% (9 9's), because I was using the `aws s3 sync` command which determines which files to sync by modified date and file size, I was a bit worried that silent data corruption would result in damaged files over time.

To investigate, I wanted to randomly thaw out some files in and check their integrity vs my locally-available copies.  This was achieved through boto3 in an ipynb which can be downloaded from this [gist](https://gist.github.com/jwfu/45c347991dde7c6c3544aac33cdbcf66).

# Notebook Features

The notebook provides tools to:

- Build a random sample of files from a local location that will be thawed in remote;
- Invoke `S3.Client.restore_object` to thaw the files;
- Download the files once restored;
- Compute SHA1s of both local and remote; and
- Compare the two sets of hashes to check file integrity.

I also included a helper utility to pickle/depickle the file list as it can take up to several days to restore files, based on the retrieval option selected.  Being cheap and not in a hurry, I selected the `Bulk` option with a 48h SLA, although I noted that the files were often available next-day.

# Results

I used the notebook to thaw ~1% of my files (~50 000) objects and found _zero_ examples where the hashes failed to match.  This gave me confidence that silent data corruption is rare enough to not be a worry in my application.

# Other Notes

I came across [this interesting discussion](https://github.com/aws/aws-cli/issues/3415) about the limitations of `aws s3 sync`, particularly when synchronizing from remote to local.  One of the proposed solutions was doing hash comparison between local and remote to determine which files to update.  Unfortuately, as noted in the thread, S3 only stores the MD5 of a file as an `etag` when the file is uploaded in 1 piece.  [@akud](https://www.npmjs.com/package/@akud/aws-s3-sync-by-hash) has built node implementation of this, and I hope there will be better support of this sync method in the future.  [This tag](https://github.com/aws/aws-cli/labels/s3syncstrategy) also tracks issues related to sync strategy, and notes some other features I hope will be incorporated in the future.
