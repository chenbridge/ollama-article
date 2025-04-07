# Ollama是从哪里下载模型的
> 作者：猫叔  
> 日期：2025/4/1  

![deepseek-r1](https://i.imgur.com/3vE09YW.png)

## 前言
本地运行开源大模型，如今最推荐的软件，非 **Ollama** 莫属。 
经过几个月的实际使用体验，我对它的评价是：**简洁高效、使用愉悦**。

一句命令即可运行模型：
```bash
ollama run deepseek-r1
```

实际运行效果如下：
```bash
C:\Users\Bridge>ollama run deepseek-r1
pulling manifest
pulling 96c415656d37... 100% ▕████████████████████▏ 4.7 GB
pulling 369ca498f347... 100% ▕████████████████████▏  387 B
pulling 6e4c38e1172f... 100% ▕████████████████████▏ 1.1 KB
pulling f4d24e9138dd... 100% ▕████████████████████▏  148 B
pulling 40fb844194b2... 100% ▕████████████████████▏  487 B
verifying sha256 digest
writing manifest
success
>>> 你是谁？
<think>

</think>

您好！我是由中国的深度求索（DeepSeek）公司开发的智能助手DeepSeek-R1。如您有任何任何问题，我会尽我所能为您提供帮助。

>>> Send a message (/? for help)
```

看到这些下载进度，我突然好奇：**Ollama是从哪里下载模型的？**  
于是我决定从源码中寻找答案。

## 从源码入手

正所谓“不入虎穴，焉得虎子”。我克隆并使用VS Code打开了 Ollama 的官方仓库：
🔗 https://github.com/ollama/ollama
Ollama 使用 Go 语言编写，我以 `"pulling manifest"` 为关键词搜索，迅速定位到关键函数：
`images.go` 文件中的 `PullModel` 函数：

```go
func PullModel(ctx context.Context, name string, regOpts *registryOptions, fn func(api.ProgressResponse)) error {
    mp := ParseModelPath(name)

    ...

    fn(api.ProgressResponse{Status: "pulling manifest"})

    ...
}
```
可以看出，这就是拉取模型的入口函数。

跟进 `ParseModelPath()`，我们看到模型路径的初始化处理：
```go
func ParseModelPath(name string) ModelPath {
	mp := ModelPath{
		ProtocolScheme: DefaultProtocolScheme,
		Registry:       DefaultRegistry,
		Namespace:      DefaultNamespace,
		Repository:     "",
		Tag:            DefaultTag,
	}
	
	...
}
```
可以看出，`ModelPath`的这些字段使用这些默认值：
```go
const (
    DefaultRegistry       = "registry.ollama.ai"
    DefaultNamespace      = "library"
    DefaultTag            = "latest"
    DefaultProtocolScheme = "https"
)
```

继续跟进，在 `ParseModelPath` 中看到：
```go
manifest, err = pullModelManifest(ctx, mp, regOpts)
```

追踪 `pullModelManifest` 函数，找到如下构造 URL 的逻辑：
```go
requestURL := mp.BaseURL().JoinPath("v2", mp.GetNamespaceRepository(), "manifests", mp.Tag)
```

因此，模型清单的默认请求链接的格式为：
```
https://registry.ollama.ai/v2/library/<MODEL>/manifests/<TAG>
```
- MODEL: 模型ID，例如`deepseek-r1`
- TAG: 标签，一般指参数规模(例如: 7b, 8b)，默认是 `latest`

例如 `deepseek-r1`：
https://registry.ollama.ai/v2/library/deepseek-r1/manifests/latest

## 模型清单
清单结构定义在 `manifest.go`：
```go
type Manifest struct {
    SchemaVersion int
    MediaType     string
    Config        Layer
    Layers        []Layer
}
```

来看一眼实际内容（使用 `jq` 美化 JSON）：
```bash
curl -s https://registry.ollama.ai/v2/library/deepseek-r1/manifests/latest | jq
```

输出如下：
```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
  "config": {
    "mediaType": "application/vnd.docker.container.image.v1+json",
    "digest": "sha256:40fb844194b25e429204e5163fb379ab462978a262b86aadd73d8944445c09fd",
    "size": 487
  },
  "layers": [
    {
      "mediaType": "application/vnd.ollama.image.model",
      "digest": "sha256:96c415656d377afbff962f6cdb2394ab092ccbcbaab4b82525bc4ca800fe8a49",
      "size": 4683073184
    },
    {
      "mediaType": "application/vnd.ollama.image.template",
      "digest": "sha256:369ca498f347f710d068cbb38bf0b8692dd3fa30f30ca2ff755e211c94768150",
      "size": 387
    },
    {
      "mediaType": "application/vnd.ollama.image.license",
      "digest": "sha256:6e4c38e1172f42fdbff13edf9a7a017679fb82b0fde415a3e8b3c31c6ed4a4e4",
      "size": 1065
    },
    {
      "mediaType": "application/vnd.ollama.image.params",
      "digest": "sha256:f4d24e9138dd4603380add165d2b0d970bef471fac194b436ebd50e6147c6588",
      "size": 148
    }
  ]
}
```

## 模型配置
在 `download.go` 中，我们可以看到实际的 blob 文件下载路径是这样生成的：
```go
requestURL = requestURL.JoinPath("v2", opts.mp.GetNamespaceRepository(), "blobs", opts.digest)
```
于是我们知道了下载链接格式：
```
https://registry.ollama.ai/v2/library/<MODEL>/blobs/<SHA256>
```

继续查看 config 文件的内容：
```bash
curl -s -L https://registry.ollama.ai/v2/library/deepseek-r1/blobs/sha256:40fb844194b25e429204e5163fb379ab462978a262b86aadd73d8944445c09fd | jq
```

结果如下：
```json
{
  "model_format": "gguf",
  "model_family": "qwen2",
  "model_families": [
    "qwen2"
  ],
  "model_type": "7.6B",
  "file_type": "Q4_K_M",
  "architecture": "amd64",
  "os": "linux",
  "rootfs": {
    "type": "layers",
    "diff_ids": [
      "sha256:96c415656d377afbff962f6cdb2394ab092ccbcbaab4b82525bc4ca800fe8a49",
      "sha256:369ca498f347f710d068cbb38bf0b8692dd3fa30f30ca2ff755e211c94768150",
      "sha256:6e4c38e1172f42fdbff13edf9a7a017679fb82b0fde415a3e8b3c31c6ed4a4e4",
      "sha256:f4d24e9138dd4603380add165d2b0d970bef471fac194b436ebd50e6147c6588"
    ]
  }
}
```

## 分层数据

### 1. `image.model` 
模型二进制文件，文件很大(4683073184)，下载链接：
https://registry.ollama.ai/v2/library/deepseek-r1/blobs/sha256:96c415656d377afbff962f6cdb2394ab092ccbcbaab4b82525bc4ca800fe8a49

### 2. `image.template` 
用于控制提示词格式的模板，下载链接：
https://registry.ollama.ai/v2/library/deepseek-r1/blobs/sha256:369ca498f347f710d068cbb38bf0b8692dd3fa30f30ca2ff755e211c94768150

实际内容：
```txt
{{- if .System }}{{ .System }}{{ end }}
{{- range $i, $_ := .Messages }}
{{- $last := eq (len (slice $.Messages $i)) 1}}
{{- if eq .Role "user" }}<｜User｜>{{ .Content }}
{{- else if eq .Role "assistant" }}<｜Assistant｜>{{ .Content }}{{- if not $last }}<｜end▁of▁sentence｜>{{- end }}
{{- end }}
{{- if and $last (ne .Role "assistant") }}<｜Assistant｜>{{- end }}
{{- end }}
```

### 3. `image.license` 
模型的许可证文件，下载链接：
https://registry.ollama.ai/v2/library/deepseek-r1/blobs/sha256:6e4c38e1172f42fdbff13edf9a7a017679fb82b0fde415a3e8b3c31c6ed4a4e4

### 4. `image.params` 
模型的推理参数定义，下载链接：
https://registry.ollama.ai/v2/library/deepseek-r1/blobs/sha256:f4d24e9138dd4603380add165d2b0d970bef471fac194b436ebd50e6147c6588

实际内容：
```json
{
  "stop": [
    "<｜begin▁of▁sentence｜>",
    "<｜end▁of▁sentence｜>",
    "<｜User｜>",
    "<｜Assistant｜>"
  ]
}
```

## 总结
本文通过分析 Ollama 的源码，深入了解了其下载模型的机制和具体链接格式：
- 模型清单（manifest）请求地址：
```
https://registry.ollama.ai/v2/library/<MODEL>/manifests/<TAG>
```
- 模型文件（blob）请求地址：
```
https://registry.ollama.ai/v2/library/<MODEL>/blobs/<SHA256>
```
以 `deepseek-r1` 为例，我们完整地解析了其模型的构成、清单结构及每个部分的实际内容。
相信这些信息能帮你更好地理解和使用 `Ollama`，也为日后构建私有模型仓库或调试模型加载过程提供了参考。
