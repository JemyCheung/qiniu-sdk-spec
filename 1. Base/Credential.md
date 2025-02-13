# Credential 认证信息

| 名称       | 类型   | 描述                |
| ---------- | ------ | ------------------- |
| access_key | String | AccessKey，必须存在 |
| secret_key | String | SecretKey，必须存在 |

## 支持接口

### sign()

签发数据

#### 接受参数

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| data | [uint8] | 被签名的数据 |

#### 返回参数

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| signed_data | String | 签名 |

#### 伪代码实现

```
"${access_key}:${urlsafe_base64_encode(hmac_sha1_digest(data, secret_key))}"
```

### sign_with_data()

签发数据（包含数据本身）

#### 接受参数

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| data | [uint8] | 被签名的数据 |

#### 返回参数

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| signed_data | String | 签名（包含数据本身） |

#### 伪代码实现

```
encoded_data = urlsafe_base64_encode(data)
"${sign(encoded_data)}:${encoded_data}"
```

### authorization_v1_for_request()

针对给定的 HTTP 请求信息，生成 Authorization Header

#### 接受参数

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| url | String | URL |
| content_type | String | Content-Type 信息 |
| body | [uint8] | 请求体 |

#### 返回参数

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| authorization | String | Authorization Header |

#### 伪代码实现

```
parsed_url = parse_url(url)
data_to_sign = []
data_to_sign.append(parsed_url.path())
if parsed_url.query() {
	data_to_sign.append('?')
	data_to_sign.append(parsed_url.query())
}
data_to_sign.append('\n')
if content_type && body && content_type == "application/x-www-form-urlencoded" {
	data_to_sign.append(body)
}

"QBox ${sign(data_to_sign)}"
```

### authorization_v2_for_request()

针对给定的 HTTP 请求信息，生成 Authorization Header

#### 接受参数

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| url | String | URL |
| method | String | HTTP 方法 |
| headers | [String:String] | HTTP 头 |
| body | [uint8] | 请求体 |

#### 返回参数

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| authorization | String | Authorization Header |

#### 伪代码实现

```
fn append_data_to_sign_for_x_qiniu_headers(data_to_sign, headers) {
	x_qiniu_headers = headers.filter((key, _) -> { return key.starts_with("X-Qiniu-") }).map((key, value) -> { return "${key}: ${value}" })
	for header_line in x_qiniu_headers.sort() {
		data_to_sign.append(header_line)
		data_to_sign.append('\n')
	}
}

parsed_url = parse_url(url)
data_to_sign = []
data_to_sign.append(method.upper())
data_to_sign.append(' ')
data_to_sign.append(parsed_url.path())
if parsed_url.query() {
	data_to_sign.append('?')
	data_to_sign.append(parsed_url.query())
}
data_to_sign.append("\nHost: ${parsed_url.host()}")
if parsed_url.port() { // 这里的语义是，URL 中是否显式包含端口号
	data_to_sign.append(":${parsed_url.port()}")
}
data_to_sign.append('\n')
content_type = headers.get("content_type")
if content_type {
	data_to_sign.append("\nContent-Type: ${content_type}\n")
	append_data_to_sign_for_x_qiniu_headers(data_to_sign, headers)
	data_to_sign.append('\n')
	if body && (content_type == "application/x-www-form-urlencoded" || content_type == "application/json") {
		data_to_sign.append(body)
	}
} else {
	append_data_to_sign_for_x_qiniu_headers(data_to_sign, headers)
	data_to_sign.append('\n')
}

"Qiniu ${sign(data_to_sign)}"
```

### is_valid_request()

判定给定的 HTTP 请求是否确实来自于七牛的回调

#### 接受参数

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| request | Request，如果没有内建支持此类型，可以用 url，headers，body 三个字段替代 | HTTP 请求 |

#### 返回参数

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| valid | bool | 是否是合法的请求 |

#### 伪代码实现

```
if !request.header("Authorization") {
	return false
}
!request.header("Authorization") == authorization_v1_for_request(request.url(), request.headers(), request.body())
```

### sign_upload_token()

签发上传凭证

#### 接受参数

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| upload_policy | UploadPolicy | 要签发的上传策略 |

#### 返回参数

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| upload_token | String | 上传凭证 |

#### 伪代码实现

```
sign_with_data(dump_json(upload_policy))
```

### sign_download_url_with_deadline()

签发上传凭证

#### 接受参数

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| base_url | String | 要签发的下载地址 |
| deadline | Time | 到期时间 |

#### 返回参数

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| private_url | String | 私有空间的下载地址 |

#### 伪代码实现

```
private_url = copy(base_url)
if private_url.contains('?') {
	private_url.append("&e=${deadline}")
} else {
	private_url.append("&e=${deadline}")
}
token = sign(private_url)
private_url.append("&token=${token}")
```

### sign_download_url_with_lifetime()

签发上传凭证

#### 接受参数

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| base_url | String | 要签发的下载地址 |
| lifetime | Duration（如果没有 Duration 类型，则使用 uint64，单位为秒） | 有效时长 |

#### 返回参数

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| private_url | String | 私有空间的下载地址 |

#### 伪代码实现

```
sign_download_url_with_deadline(base_url, now() + lifetime)
```

