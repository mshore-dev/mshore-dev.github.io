---
title: "Migrating Mastodon to S3"
date: 2023-01-11T23:29:01-05:00
draft: true
---
## Forewarning
I made some mistakes, and those mistakes cost me fair bit of time. This is *not* a guide! This is my experiences migrating from local file storage to a cloud provider, and the issues and solution I came across along the way. At the end of the post, I have linked a few guides relating to this topic, refer to those for your how-to guides. With that out of the way, let us begin.
## The problem
A few days ago, I set up a Mastodon server. It is not a large server by any means, with me and a few friends being the only registered users. However, the process of federating with a (relatively large amount!) of external servers has generated a lot of cached data - avatars, headers, and toot attachments. In the three days my instance has been running, Mastodon has cached just over 30GB of media. 
```
mike@nonbiri:~$ du -s -h mastodon/mastodon/system/
33G     mastodon/mastodon/system/
mike@nonbiri:~$ df -h /
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1        59G   42G   14G  76% /
```
When initially setting up Mastodon, I severely underestimated the amount of data that would be cached. Surely with only three users, it would be a manageable amount. After all, I had 60GB of storage to play with. Had I not connected my instance to a few relays, this may have been the case. However, adding relays allowed my server to see a much larger portion of the fediverse. This came at the cost of longer Sidekiq queues and - you guessed it! - drastically more media being cached.

## The solution
I could disconnect my instance from relays, but that would defeat the purpose of being connected to the fediverse. Instead, I opted to relieve my VPS of media serving duties and use a cloud service to store and serve media. There are many cloud storage providers to choose from, each with their pros and cons. A popular choice seems to be [iDrive E2](https://www.idrive.com/e2/), however I have decided to go with a more standard option - [Backblaze B2](https://www.backblaze.com/b2/cloud-storage.html). Additionally, I'll be adding a CDN ([Bunny.net](https://bunny.net)) in order to use a custom domain name (media.my-mastodon.tld) and improve loading times for other instances.

## Migrating
There are plenty of wonderful guides on migrating Mastodon to cloud storage[^1][^2][^3] - this will be an abridged version of these with some of my own experience tossed in. The general outline for migrating storage is as follows:
1) Prune current cache to a reasonable size
2) Move existing data to cloud storage
3) Configure Mastodon to use cloud storage
4) Do one final copy to get any new media created after 2, but before 3

For my instance, pruning "old" media is not particularly helpful, as the instance itself has been running for less than 3 days. So for this migration, I will be skipping over this step. Which leads us to...

## Moving existing data
Easy enough, create a bucket on Backblaze, generate an application key for rclone and start transferring. First, it is important to do a "dry run" to ensure the files will end up in the right place. It is much easier to `ls` twice and `cp` once, rather than re-organizing the files afterwords.
```
mike@nonbiri:~/mastodon/mastodon$ rclone copy system/ b2: --dry-run
2023/01/08 19:19:21 NOTICE: media_attachments/thumbnails/109/654/680/682/023/877/original/5df0b7e9ccd26181.png: Skipped copy as --dry-run is set
2023/01/08 19:19:21 NOTICE: media_attachments/thumbnails/109/654/756/060/845/646/original/bd2fc4ad731afd8a.png: Skipped copy as --dry-run is set
2023/01/08 19:19:21 NOTICE: media_attachments/thumbnails/109/644/732/253/007/928/original/4d46805fd057033a.jpg: Skipped copy as --dry-run is set
2023/01/08 19:19:21 NOTICE: media_attachments/thumbnails/109/639/809/444/715/946/original/ab38a20af8bf6518.png: Skipped copy as --dry-run is set
<snip>
```
Initially, I was unsure if I was just supposed to copy the contents of `mastodon/system/` to cloud storage, but after confirming the naming schemes on other Mastodon instances it seems to be correct. Now that we have verified the destination of the data, we can let `rclone` rip.

Here is where we run into our first issue. We already established that `mastodon/system/` was 33GB, but the `cache` folder made up 99% of that. `rclone` ran, copying data for 20 minutes, and only made it through the first ~4000 files, totallying ~1.5GB. All of these files being cached media. Since the instance was brand-new, there was not yet much local media. Instead of attempting to transfer all of the current cache to B2, I opted to leave the cache behind and only copy the `accounts` and `media_attachments` folders. Once again, `ls` twice and `cp` once, so after verifying the command was correct, I started copying the `accounts` folder. 

This resulted in 38 files being copied.

## Setting up Bunny.net
To set up Bunny CDN, I followed [their official guide for using Bunny CDN with Backblaze B2](https://support.bunny.net/hc/en-us/articles/360018649972-How-to-speed-up-your-BackBlaze-B2-file-delivery-with-BunnyCDN). Afterwords, I added a custom domain by creating a CNAME record at my domain registrar, as shown in [their guide for setting up a custom subdomain](https://support.bunny.net/hc/en-us/articles/207790279-How-to-set-up-a-custom-CDN-hostname).

## Configuring Mastodon

After copying the important files over and setting up Bunny, it was time to configure Mastodon to use B2 and Bunny. As outlined in these wonderful guides[^1][^2][^3], it should be as simple as changing a few environment variables and restarting Mastodon. Since I deployed Mastodon using Docker Compose, I added the following lines to the the web, streaming and sidekiq containers:
```yaml
environment:
        S3_ENABLED: "true"
        S3_BUCKET: "${MASTODON_S3_BUCKET}"
        S3_HOSTNAME: "media.${MASTODON_DOMAIN}"
        AWS_ACCESS_KEY_ID: "${MASTODON_S3_KEY_ID}"
        AWS_SECRET_ACCESS_KEY: "${MASTODON_S3_SECRET_KEY}"
        S3_PROTOCOL: "https"
        S3_ENDPOINT: "${MASTODON_S3_ENDPOINT}"
  ```
(The secret values are stored in a `.env` file)

In theory, this should grant us a working Mastodon instance, serving files on cloud storage, behind a CDN. With a great deal of trepidation, I restarted the Mastodon containers...

Surprisingly enough, it *almost* worked.

I made a small error while copy the files (or configuring Mastodon, not entirely sure) but Mastodon was trying to serve files from a sub-directory on B2, which did not have the uploaded files. After moving the files into the correct place:
```
mike@nonbiri:~$ rclone copy b2:nonbiri-social/accounts/ b2:nonbiri-social/nonbiri-social/accounts/
mike@nonbiri:~$ rclone copy b2:nonbiri-social/media_attachments/ b2:nonbiri-social/nonbiri-social/media_attachments/
```
Local user media was back! New media was also stored correctly! However, old content from other instances would not load. In all honesty, I had anticipated running into this problem, but hoped nevertheless hoped I would get lucky. Fortunately, `tootctl` has a function to clear the local media cache:
```
mike@nonbiri:~/mastodon$ ./tootctl.sh media remove --days 0
22653/22653 |===============================================================================| Time: 00:13:52
Removed 22653 media attachments (approx. 24.3 GB)
```
This would, unfortunately, mean Mastodon would have to re-download (and in this case, re-upload) remote media files. Removing the 22.6K media files took around 12 minutes.

Alas, this did not bring back old remote media either. In the end, I was unable to force Mastodon to immediately re-download the missing media. In an attempt to restore the missing avatars and headers, I set the media retention times to one day, and let the instance run as normal for a few days. Sadly, this did not bring back the avatars/headers either.

After further research, I came across [this issue on mastodon/mastodon GitHub](https://github.com/mastodon/mastodon/issues/9657) which suggests that "media" refers only to content attached to posts, not avatars and headers. This issue recommends ``tootctl account refresh --all`` as the equivalent for re-downloading the data I am missing. While this seems like it would solve my issue, I had a few concerns about the time and bandwidth costs of fetching the information of the 20K accounts my instance knows of.
```
mike@nonbiri:~/mastodon$ ./tootctl.sh account refresh --all --dry-run
20223/20234 |============================================================================== |  ETA: ??:??:??
Refreshed 20234 accounts (DRY RUN)
```
Reluctantly, I opened a new `tmux` session and let it run. The estimated time for completion started at just over three hours. It was doing a bit more work than I needed it to, but it appears to be the only solution to fix *all* of the missing avatars/headers in one go. After about 4 hours, all of the accounts were finally refreshed:
```
mike@nonbiri:~/mastodon$ ./tootctl.sh accounts refresh --all
<snip>
Refreshed 20590 accounts
```
Just like that, all of the missing avatars and headers were back!

## Conclusion
Overall, this migration went... Yeah, it went. Had I been more diligent, perhaps I could have avoided these issues. But I am glad to have run into them now, rather than down the road. In the end, everything worked fine and it was a great learning experience.

## Future Goals
My main reason for setting up a Mastodon instance is to learn more about systems administration and the fediverse. In the future, I would like to deploy a scaled-up, public Mastodon instance. To work towards that goal, I decided to start on a smaller scale, and iron out some kinks ahead of time. 

With that said, the next subject I would like to investigate is a proper backup solution.

Some additional topics I would like to investigate further down the road:
* Adding more Sidekiq workers to aid in scaling
* Self-hosting [Minio](https://github.com/minio/minio) to serve media
* Mastodon forks: [glitch-soc](https://github.com/glitch-soc/mastodon) and [Hometown](https://github.com/hometown-fork/hometown)

### Other resources
[^1]: https://stanislas.blog/2018/05/moving-mastodon-media-files-to-wasabi-object-storage/
[^2]: https://github.com/cybrespace/cybrespace-meta/blob/master/s3.md
[^3]: https://maciej.lasyk.info/2023/Jan/08/migrating-mastodon-storage-to-s3-compatible/
