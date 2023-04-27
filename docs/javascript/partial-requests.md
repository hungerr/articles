# Http范围请求与断点续传

## Header

范围请求用到几个Header

### Range

`Range`是一个请求首部，告知服务器返回文件的哪一部分。在一个`Range`首部中，可以一次性请求多个部分，服务器会以`multipart`文件的形式将其返回。如果服务器返回的是范围响应，需要使用`206 Partial Content`状态码。假如所请求的范围不合法，那么服务器会返回`416 Range Not Satisfiable`状态码，表示客户端错误。服务器允许忽略`Range`首部，从而返回整个文件，状态码用`200`

```HTTP
Range: <unit>=<range-start>-
Range: <unit>=<range-start>-<range-end>
Range: <unit>=<range-start>-<range-end>, <range-start>-<range-end>
Range: <unit>=<range-start>-<range-end>, <range-start>-<range-end>, <range-start>-<range-end>
```

`range-end`
一个整数，表示在特定单位下，范围的结束值。这个值是可选的，如果不存在，表示此范围一直延伸到文档结束。

示例:
```HTTP
Range: bytes=200-1000, 2000-6576, 19000-
```

### Content-Type

需要返回的MIME类型是`multipart/byteranges`

### Accept-Ranges

服务器使用HTTP响应头`Accept-Ranges`标识自身支持范围请求。字段的具体值用于定义范围请求的单位。

当浏览器发现`Accept-Ranges`头时，可以尝试继续中断了的下载，而不是重新开始。

```HTTP
Accept-Ranges: bytes
Accept-Ranges: none
```

`none`
不支持任何范围请求单位，由于其等同于没有返回此头部，因此很少使用。不过一些浏览器，比如 IE9，会依据该头部去禁用或者移除下载管理器的暂停按钮。

`bytes`
范围请求的单位是 bytes（字节）。

### Content-Range

响应首部`Content-Range`显示的是一个数据片段在整个文件中的位置。
```HTTP
Content-Range: <unit> <range-start>-<range-end>/<size>
Content-Range: <unit> <range-start>-<range-end>/*
Content-Range: <unit> */<size>
Content-Range: bytes 200-1000/67589
```
`size`
整个文件的大小（如果大小未知则用"*"表示）。


## 响应码

### 206 Partial Content

`206 Partial Content`成功状态响应代码表示请求已成功，并且主体包含所请求的数据区间，该数据区间是在请求的`Range`首部指定的。

如果只包含一个数据区间，那么整个响应的 `Content-Type` 首部的值为所请求的文件的类型，同时包含 `Content-Range` 首部。

如果包含多个数据区间，那么整个响应的 `Content-Type` 首部的值为 `multipart/byteranges` ，其中一个片段对应一个数据区间，并提供 `Content-Range` 和 `Content-Type` 描述信息。

单个:
```HTTP
HTTP/1.1 206 Partial Content
Date: Wed, 15 Nov 1995 06:25:24 GMT
Last-Modified: Wed, 15 Nov 1995 04:58:08 GMT
Content-Range: bytes 21010-47021/47022
Content-Length: 26012
Content-Type: image/gif

... 26012 bytes of partial image data ...
```

多个:
```HTTP
HTTP/1.1 206 Partial Content
Date: Wed, 15 Nov 1995 06:25:24 GMT
Last-Modified: Wed, 15 Nov 1995 04:58:08 GMT
Content-Length: 1741
Content-Type: multipart/byteranges; boundary=THIS_STRING_SEPARATES

--THIS_STRING_SEPARATES
Content-Type: application/pdf
Content-Range: bytes 500-999/8000

...the first range...
--THIS_STRING_SEPARATES
Content-Type: application/pdf
Content-Range: bytes 7000-7999/8000

...the second range
--THIS_STRING_SEPARATES--
```

### 416 Range Not Satisfiable

`416 Range Not Satisfiable` 错误状态码意味着服务器无法处理所请求的数据区间。最常见的情况是所请求的数据区间不在文件范围之内，也就是说，`Range` 首部的值，虽然从语法上来说是没问题的，但是从语义上来说却没有意义。

416 响应报文包含一个 `Content-Range` 首部，提示无法满足的数据区间（用星号 `*` 表示），后面紧跟着一个`/`，再后面是当前资源的长度。例如：`Content-Range: */12777`

遇到这一错误状态码时，浏览器一般有两种策略：要么终止操作（例如，一项中断的下载操作被认为是不可恢复的），要么再次请求整个文件。
```HTTP
HTTP/1.1 416 Range Not Satisfiable
Date: Fri, 20 Jan 2012 15:41:54 GMT
Content-Range: bytes */47022
```

## ceph分片上传

ceph分片上传分三步骤:

### CreateMultipartUpload

```HTTP
POST /{Key+}?uploads HTTP/1.1
```

返回一个`UploadId`

```XML
<InitiateMultipartUploadResult>
   <Bucket>string</Bucket>
   <Key>string</Key>
   <UploadId>string</UploadId>
</InitiateMultipartUploadResult>
```

### UploadPart

分片后每个片设置一个PartNumber 用于服务端排序

```HTTP
PUT /Key+?partNumber=PartNumber&uploadId=UploadId HTTP/1.1
```

### CompleteMultipartUpload

告知服务端完成

```HTTP
POST /Key+?uploadId=UploadId HTTP/1.1
```

当对象作为分段上传进行上传时，该对象的 ETag 不是整个对象的 MD5 摘要。Amazon S3 会在上传时计算每个分段的 MD5 摘要。MD5 摘要用于确定最终对象的 ETag。Amazon S3 将 MD5 摘要的字节串联在一起，然后计算这些串联值的 MD5 摘要。创建 ETag 的最后一步是 Amazon S3 在末尾添加一个短横线和分段总数。

例如，考虑使用 ETag 为 C9A5A6878D97B48CC965C1E41859F034-14 的分段上传的对象上传。在本例中，C9A5A6878D97B48CC965C1E41859F034 是串联在一起的所有摘要的 MD5 摘要。-14 表示有 14 个分段与此对象的分段上传相关联。
