## 使用s3上传大文件报错
bitpoke/mysql-operator里使用的使用,因为是通过stream的方式传递给rclone的
```txt
I1029 19:00:01.129508 1 deleg.go
ost"="ec-core-5744-mysql-1.mysql.domi
2025/10/29 19:00:01 NoTIcE: S3 bucket cloud-native-uat path backups: Streaming uploads using chunk size 5Mi will have maximum file size of 48.828Gi
2025/10/30 06:38:55 ERR0R:ec-core-5744-auto-2025-10-29t19-00-00.xbackup-gz.tmp:Post request rcat error:multipart upload: failed to finalise:failed to complete multipart upload "s3dynamic-d4b68a55-43f3-4c619462-4c827c030ad9":operation error S3:CompleteMultipartUpload,https response error StatusCode:400,RequestID: 1983785158339678208,HostID:2,api error InvalidPart: 0ne or more of the specified parts couldnot be found. The part might not have been uploaded,or the specified entity tag might not have matchedthe part's entity tag, or the part size is not right.
2025/10/30 06:38:55 NoTIcE:Failed to rcat with 2 errors:last error was:multipart upload: failed to finalise:failed to complete multipart upload "s3dynamic-d4b68a55-43f3-4c61-9462-4c827c030ad9":operation error S3:CompleteMultipartUpload,https response error StatusCode: 400, RequestID: 1983785158339678208,HostID: 2, api error InvalidPart: One or more of the specified parts could not be found. The part might nothave been uploaded,or the specified entity tag might not have matched the part's entity tag,or the part size is not right.
E1030 06:38:55.916579
1 deleg-go:144] sidecar "msg"="take backup command failed"error"="exit stat
us 1"
```
rclone调用代码的地方:
```go
// func OpenChunkWriter
if size == -1 {
    warnStreamUpload.Do(func() {
        fs.Logf(f, "Streaming uploads using chunk size %v will have maximum file size of %v",
            f.opt.ChunkSize, fs.SizeSuffix(int64(chunkSize)*int64(uploadParts)))
    })
} else {
    chunkSize = chunksize.Calculator(src, size, uploadParts, chunkSize)
}
```
另一个错误出现调用的地方
```go
// func (w *s3ChunkWriter) Close(ctx context.Context) (err error)

```
### 解决
在mysql的cr里添加如下选项: spec.rcloneExtraArgs: '--s3-chunk-size=100Mi'
文档: 
- docs/content/s3.md:1444
- https://rclone.org/s3/#s3-chunk-size
- https://rclone.org/s3/#multipart-uploads-1