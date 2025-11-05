## 上传到s3的例子
配置文件
```txt
# vi ~/.config/rclone/rclone.conf
[minio]
type = s3
provider = Minio
access_key_id = minioadmin
secret_access_key = minioadmin
endpoint = http://10.6.178.178:9000
# 用于控制创建 bucket 时的默认访问权限（Access Control List）
# 它只在 rclone 创建 bucket 时生效（例如你第一次上传时目标 bucket 不存在，rclone 自动帮你创建它），
# 不会影响已经存在的 bucket 或上传的对象。
bucket_acl = private
# 控制上传的文件是否公开等
#acl = private
upload_cutoff = 10Mi
```
展示配置文件
```txt
rclone config show minio
[minio]
type = s3
provider = Minio
access_key_id = minioadmin
secret_access_key = minioadmin
endpoint = http://10.6.178.178:9000
bucket_acl = private
upload_cutoff = 10Mi
```
### 上传
错误的命令, 导致复制到了本地的minio目录下
```bash
➜ rclone copy /tmp/1.mp4 minio/bucket1105 --progress
Transferred:      336.372 MiB / 336.372 MiB, 100%, 0 B/s, ETA -
Transferred:            1 / 1, 100%
Elapsed time:         0.7s
```
正确的: rclone copy /tmp/1.mp4 minio:/bucket1105 --progress, 主要是实例名称(minio)和桶名称(/bucket1105)之间需要一个冒号  
如果桶bucket1105不存在, 会创建这个桶  
```bash
➜ rclone copy /tmp/1.mp4 minio:/bucket1105 --progress
Transferred:      336.372 MiB / 336.372 MiB, 100%, 10.879 MiB/s, ETA 0s
Transferred:            1 / 1, 100%
Elapsed time:        31.1s
```
### 用mc命令查看文件
- mc alias list, 我们的别名为`pool2`
- 查看桶 mc ls pool2/
- 查看bucket1105这个桶里的文件 mc ls pool2/bucket1105
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

## rclone的upload_cutoff参数

## 上传大文件消耗的内存
源代码里的MANUAL.html里的
```txt
Multipart uploads will use --transfers * --s3-upload-concurrency * --s3-chunk-size extra memory. Single part uploads to not use extra memory.
```
实际的消耗,执行命令: `rclone copy /tmp/1.up minio:/bucket1105 --s3-chunk-size 100Mi -vv`
```bash
➜ ps -ef|grep rclone
root     1425991   19262 31 16:24 pts/1    00:00:39 rclone copy /tmp/1.up minio:/bucket1105 --s3-chunk-size 100Mi -vv
root     1641336 2039031  0 16:27 pts/3    00:00:00 grep --color=auto rclone

cat /proc/1425991/status | grep -E "VmRSS|VmSize"
VmSize:  1295960 kB
VmRSS:     61440 kB
```

### s3-chunk-size
默认5Mi, 因为s3的一个文件的最大分块是10000个,如果单个文件超过48Gi, 就需要修改s3-chunk-size, 比如改为100Mi

### s3-upload-concurrency
源代码里的MANUAL.html里的
```txt
--s3-upload-concurrency int                           Concurrency for multipart uploads and copies (default 4)
```

### transfers
源代码里的MANUAL.html里的
控制同时多少个文件上传
```txt
--transfers int
The number of file transfers to run in parallel. It can sometimes be useful to set this to a smaller number if the remote is giving a lot of timeouts or bigger if you have lots of bandwidth and a fast remote.
The default is to run 4 file transfers in parallel.
Look at --multi-thread-streams if you would like to control single file transfers.
```
### AI关于各个参数的区别
https://chatgpt.com/share/690afff7-9d88-800d-89ba-97a47f3ae701
