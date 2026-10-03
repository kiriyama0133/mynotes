```csharp
using System.IO;
using System.Net.Http;
using System.Net.Http.Headers;
using System.Threading.Tasks;
using System;
using Microsoft.Extensions.Logging;

namespace qqbot.Services // 请确保命名空间正确
{
    public record DownloadProgress(long BytesDownloaded, long TotalBytes);

    public class FileCacheHttpService : IDisposable
    {
        private readonly HttpClient _httpClient;
        private readonly ILogger<FileCacheHttpService> _logger;
        public int Retries { get; set; } = 5;

        public FileCacheHttpService(HttpClient httpClient, ILogger<FileCacheHttpService> logger)
        {
            _logger = logger;
            _httpClient = httpClient;
            _httpClient.Timeout = TimeSpan.FromHours(1); // 保持一个较长的超时
        }

        public async Task DownlaodFileAsync(string url, string destinationPath,
            IProgress<DownloadProgress>? progress = null,
            CancellationToken cancellationToken = default
            )
        {
            for (int i = 0; i < Retries; i++)
            {
                try
                {
                    _logger.LogInformation("开始缓存文件 {Url}", url);
                    await PerformDownloadAsync(url, destinationPath, progress, cancellationToken);
                    _logger.LogInformation("文件 {DestinationPath} 下载成功。", destinationPath);
                    
                    // ✅ 关键修正 1: 成功后立即返回，不再继续循环。
                    return; 
                }
                catch (Exception ex)
                {
                    _logger.LogWarning(ex, "第 {Attempt} 次下载失败。将在5秒后重试...", i + 1);
                    if (i == Retries - 1)
                    {
                        _logger.LogError(ex, "已达到最大重试次数，下载文件 {Url} 失败。", url);
                        throw; // 达到最大次数后，将异常抛出
                    }
                    await Task.Delay(5000, cancellationToken);
                }
            }
        }

        private async Task PerformDownloadAsync(
            string url, string destinationPath, // 修正了参数名的拼写错误
            IProgress<DownloadProgress>? progress,
            CancellationToken cancellationToken
            )
        {
            long existingFileSize = 0;
            if (File.Exists(destinationPath))
            {
                existingFileSize = new FileInfo(destinationPath).Length;
            }

            var request = new HttpRequestMessage(HttpMethod.Get, url);
            if (existingFileSize > 0)
            {
                request.Headers.Range = new RangeHeaderValue(existingFileSize, null);
                _logger.LogInformation("文件已存在，从 {Bytes} 字节处继续下载。", existingFileSize);
            }

            using var response = await _httpClient.SendAsync(request, HttpCompletionOption.ResponseHeadersRead, cancellationToken);
            
            if (response.StatusCode == System.Net.HttpStatusCode.OK)
            {
                existingFileSize = 0;
            }
            else if (response.StatusCode != System.Net.HttpStatusCode.PartialContent && existingFileSize > 0)
            {
                throw new InvalidOperationException($"服务器不支持断点续传 (状态码: {response.StatusCode})");
            }
            response.EnsureSuccessStatusCode();

            long totalBytes = response.Content.Headers.ContentLength ?? 0;
            if (existingFileSize > 0 && response.Content.Headers.ContentRange != null)
            {
                totalBytes = response.Content.Headers.ContentRange.Length ?? 0;
            }
            _logger.LogInformation("开始下载，总大小 {TotalBytes} 字节。", totalBytes);

            using var contentStream = await response.Content.ReadAsStreamAsync(cancellationToken);
            using var fileStream = new FileStream(destinationPath, existingFileSize == 0 ? FileMode.Create : FileMode.Append, FileAccess.Write, FileShare.None);
            
            var buffer = new byte[81920];
            long totalBytesRead = existingFileSize;
            int bytesRead;
            while ((bytesRead = await contentStream.ReadAsync(buffer, 0, buffer.Length, cancellationToken)) > 0)
            {
                await fileStream.WriteAsync(buffer, 0, bytesRead, cancellationToken);
                totalBytesRead += bytesRead;
                progress?.Report(new DownloadProgress(totalBytesRead, totalBytes));
            }
        }
        
        public void Dispose()
        {
            // ✅ 关键修正 2: 正确实现 Dispose
            _httpClient?.Dispose();
        }
    }
}
```

### 检测文件的下载点，是否存在文件

```csharp
        long existingFileSize = 0;
        // 检查下载点
        if (File.Exists(desinationPath)){
            existingFileSize = new FileInfo(desinationPath).Length;
        }
```

​	在检查完是否已经存在了之后，开始建立一个请求，尝试从断点开始下载：

```csharp
 var request = new HttpRequestMessage(HttpMethod.Get, url); // 创建Url的http请求
 if (existingFileSize > 0){
     request.Headers.Range = new RangeHeaderValue(existingFileSize, null);
     _logger.LogInformation("文件已经存在，开始从{Bytes字节下载}", existingFileSize);
 }
```

​	之后获取响应头：为了**避免内存爆炸**和实现**真正的流式处理**。

```csharp
        // 确保先获取响应头，而不会缓冲整个响应体
        using var response = await _httpClient.SendAsync(request, HttpCompletionOption.ResponseHeadersRead, cancellationToken);
        if (response.StatusCode == System.Net.HttpStatusCode.OK){ // 不支持范围请求的情况

            existingFileSize = 0; //
        }
```

`_httpClient` 在后台会执行以下操作：

1. 发送 GET 请求。
2. 接收到**响应头**。
3. 开始接收响应体（文件内容），并将其**全部缓冲**到一个内部的内存流中。
4. 当**整个文件**都下载到内存中之后，`await` 操作才会完成，您的代码才会继续执行。

**后果**: 如果下载一个 1GB 的文件，程序会**瞬间消耗 1GB 的内存**。这对于服务器应用或同时进行多个下载的客户端来说是致命的。

​	

​	因此我们使用`ResponseHeadersRead` 模式

```csharp
        using var response = await _httpClient.SendAsync(request, HttpCompletionOption.ResponseHeadersRead, cancellationToken);
        if (response.StatusCode == System.Net.HttpStatusCode.OK){ // 不支持范围请求的情况

            existingFileSize = 0; //
        }
        else if (response.StatusCode != System.Net.HttpStatusCode.PartialContent && existingFileSize > 0) {
            throw new InvalidOperationException($"服务器不支持断点续传 (状态码: {response.StatusCode})");
        }
        response.EnsureSuccessStatusCode();
```

`_httpClient` 的行为会完全改变：

1. 发送请求。
2. **一旦接收到响应头**，`await` 操作**立即完成**，您的代码就可以继续执行。
3. 此时，响应体（文件内容）还**停留在底层的网络套接字缓冲区**中，**几乎没有占用您的应用程序内存**。
4. 然后，您可以从 `response.Content` 中获取一个**网络流 (`Stream`)**。
5. 接下来，您就可以在一个循环中，用一个小小的缓冲区（例如 80KB），**一块一块地**从这个网络流中读取数据，**并直接写入到磁盘的文件流中。**



**好处**:

- **极低的内存占用**: 无论您下载的文件是 1GB 还是 10GB，您的程序内存占用始终都非常低（只有那个小缓冲区的大小）。
- **更快的“首字节”响应**: 您的代码可以非常快地拿到响应头，从而可以立即检查服务器的状态码 (`response.StatusCode`)、文件总大小 (`response.Content.Headers.ContentLength`) 等信息，并根据这些信息来决定下一步怎么做（例如，我们代码中的断点续传逻辑就需要根据状态码是 `200 OK` 还是 `206 PartialContent` 来做不同处理）。



**断点续传的进度条实现**

```csharp
        long totalBytes = response.Content.Headers.ContentLength ?? 0;
        if ( existingFileSize >0 && response.Content.Headers.ContentRange != null){
            totalBytes = response.Content.Headers.ContentRange.Length ?? 0;
        }
        _logger.LogInformation("开始下载，总大小{TotalBytes}字节", totalBytes);
```

​	为了计算下载进度百分比，我们需要两个数：`已下载字节数` 和 `文件总字节数`。这段代码**就是为了准确地获得第二个值**。

​	**含义**: “首先，我假设这是一次全新的下载。请尝试从服务器响应的 `Content-Length` 头部信息中，获取文件的总大小。”

​	**工作场景**: 当我们发起一个**普通的、从头开始的下载请求时**，服务器会返回一个 `200 OK` 状态码，并在响应头中包含一个 `Content-Length` 字段，它的值就是文件的完整大小。

​	**`?? 0`**: 这是一个**空值合并运算符**，意思是如果服务器没有提供 `Content-Length`，就**默认总大小为 0。**



**特殊情况 (断点续传)**

```csharp
if ( existingFileSize > 0 && response.Content.Headers.ContentRange != null){
    totalBytes = response.Content.Headers.ContentRange.Length ?? 0;
}
```

`existingFileSize > 0`: 这个条件确认了我们**本地确实已经存在一个未下载完的文件**。

​	`response.Content.Headers.ContentRange != null`: 这个条件**确认了服务器的回应**是一个 `206 Partial Content`（部分内容）响应。这种响应**必须**包含一个 `Content-Range` 头部。

**`totalBytes = response.Content.Headers.ContentRange.Length ?? 0;`**:

​	**这是最关键的一步！** 当服务器响应一个“部分内容”时，它的 `Content-Length` 头部只会包含**本次传输的数据块的大小**，而**不是**整个文件的总大小。

​	而 `Content-Range` 头部则包含了更丰富的信息，它的格式通常像这样：`bytes 1048576-2097151/5242880` 这句话的意思是：“我正在向你发送从第 1048576 字节到第 2097151 字节的数据，这个文件的**完整总大小是 5242880 字节**。”`response.Content.Headers.ContentRange.Length` 正是用来**精确地提取**出那个 `/` 后面的**完整文件总大小**。

​	**为什么要覆盖**: 所以，如果这是一次断点续传，我们必须用 `Content-Range` 中的**总大小**，来**覆盖**掉 `Content-Length` 中那个不准确的“部分大小”。



```csharp
 // 异步读写数据流
 using var contentStream = await response.Content.ReadAsStreamAsync(cancellationToken);
 // append模式追踪数据
 using var fileStream = new FileStream(desinationPath,existingFileSize == 0 ? FileMode.Create : FileMode.Append , FileAccess.Write, FileShare.None);
 var buffer = new byte[81920]; // 80KB 缓冲区
 long totalBytesRead = existingFileSize;
 int bytesRead;
 while ((bytesRead = await contentStream.ReadAsync(buffer, 0, buffer.Length, cancellationToken)) > 0) {
     await fileStream.WriteAsync(buffer, 0, bytesRead, cancellationToken); // 
     totalBytesRead += bytesRead;
     progress?.Report(new DownloadProgress(totalBytesRead, totalBytes));
 }
```

​	在我们通过 `ResponseHeadersRead` 获取到响应头之后，这行代码会从 `response.Content` 中获取一个**网络流 (`Stream`)**。此时，真正的**文件数据还在服务器端或网络缓冲区**中，并没有被下载到您的程序内存里。`contentStream` 就是我们用来从网络上“抽水”的管子。

​	using var fileStream = new FileStream(...);这行代码会创建一个指向本地磁盘文件的**文件流 (`FileStream`)**。这是我们将要写入数据的目标。如果 `existingFileSize` 是 `0`（表示这是一次全新的下载），就使用 `FileMode.Create` 来**创建**一个新文件（如果已存在则覆盖）。如果 `existingFileSize` **大于 `0`**（表示这是一次**断点续传**），就使用 `FileMode.Append`，它会打开现有文件并将写入位置**自动定位到文件的末尾**，准备**追加新数据**。

**`while ((bytesRead = await contentStream.ReadAsync(...)) > 0)`**作用: 这是下载的核心循环，意思是“只要网络水管里还有水，就一直抽”。

- `await contentStream.ReadAsync(buffer, 0, buffer.Length, cancellationToken)`: 这是一个异步操作。它会尝试从网络流 (`contentStream`) 中读取数据，最多读取 `buffer.Length` 那么多字节（在我们的例子中是 80KB），然后将数据放入 `buffer` 数组中。`bytesRead` 变量会得到**实际读取到的字节数**。
- `while (... > 0)`: 当服务器发送完所有数据后，下一次 `ReadAsync` 会返回 `0`，这个**循环就会自然结束**。



​	**`await fileStream.WriteAsync(buffer, 0, bytesRead, cancellationToken);`**:**作用**: 将刚刚从网络“抽”到的一桶水（`buffer` 里的数据），**异步地**倒入“本地水桶”（写入磁盘文件）。我们只写入 `bytesRead` 那么多，因为这才是本次实际读取到的数据量。

**`totalBytesRead += bytesRead;`**:

- **作用**: 更新我们的“已下载总量”计数器。

**`progress?.Report(new DownloadProgress(totalBytesRead, totalBytes));`**:

- **作用**: 向外界**报告**当前的下载进度。
- `progress?` 是一种空条件运算符，意思是“如果 `progress` 对象不是 `null`（即调用者关心进度），才执行后面的代码”。
- `Report(...)`: 这个方法会触发您在调用 `DownloadFileAsync` 时传入的那个 `IProgress<T>` 回调，将当前的“已下载量”和“总量”传递出去，**从而让 UI 界面可以更新进度条**。