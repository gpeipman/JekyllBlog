---
layout: post
title:  "gzip and cache control on Azure Blob Storage"
date:   2019-05-13 08:00:00
categories: ['Azure']
permalink: /azure/azure-blob-storage-gzip-cache-control/
---

Hosting WordPress site on Azure virtual machines means keeping away the loads we can serve from cheaper services. Same time we still want to keep site optimized so pages load fast. This blog post shows how to use gzip compression and cache control headers on Azure blob storage.

I keep my blog content on Azure blob storage. Content is served to visitors using Azure Akamai CDN offering. I made this move in hope that my dear readers who live in Asia, South-America and other exotic places (exotic for me) have now shorter download times in my blog.

## File hosting models

I want to support two scenarios. static files are stored on Azure blob storage and visitors get files either directly from blob storage or through CDN service.

![alt text]({{ site.baseurl }}/images/2019/05/request-cdn-blob-storage.png.webp "Logo Title Text 1") 

If CDN doesn’t prove to be effective for visitors from exotic countries then I can remove it and server content directly from blob storage without any need to modify files metadata on blob storage.

## Converting files to compressed blobs

During optimization I found out that JavaScript and CSS files are not always compressed by CDN. Any solution at blob storage would be nice but there’s no compression options available. After some experimenting I found a way how to serve compressed JavaScript and CSS straight from blob storage.

1. Using 7zip or your favorite archiving tool compress scripts and styles as gzip files.
2. Rename files so they have .js and .css extensions.
3. Upload files to blob storage using Azure Storage Explorer or azCopy.
4. Update files settings on blob storage:
	* ContentType: text/css; charset=utf8 or application/javascript; charset=utf8
	* ContentEncoding: gzip
	* CacheControl: max-age=864000, public, must-revalidate

I’m using Azure Storage Explorer to set properties of blobs. It is free tool by Microsoft.

![alt text]({{ site.baseurl }}/images/2019/05/azure-storage-explorer-css-properties.png.webp "Logo Title Text 1") 

Here is the example of response headers returned to browser when opening CSS file from blob storage.

![alt text]({{ site.baseurl }}/images/2019/05/azure-blob-storage-css-headers.png.webp "Logo Title Text 1") 

It also works with CDN as CDN sends to browsers same headers it got from source storage.

> NB! Akamai CDN offering on Azure supports content compression. There are still some cases when compression by CDN is not applied. One case is when file is served directly from source storage (new file for CDN and it’s not deployed to all end-points yet). If files are compressed on source storage then CDN respects this.

## Wrapping up

Azure blob storage with CDN is great for moving load away from small virtual machines that host WordPress site. Although blob storage doesn’t support content compression on service level we can still create compressed files and set response headers so browsers understand that content is compressed. This way we can compress JavaScript and CSS files by example. Additionally we can set cache control header to make browsers use their own cache when file is once downloaded.