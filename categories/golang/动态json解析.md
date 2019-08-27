# golang动态解析json数据
golang的静态语言，对于存放json格式当然可以使用map[string]interface{}的形式来存放。
比如如下数据：
```json
{
	"type": "MODBUS",
	"configuration": {
		"transport": {
			"type": "rtu",
			"timeout": 3000,
			"portName": "ttyM0",
			"encoding": "rtu",
			"parity": "none",
			"baudRate": 9600,
			"dataBits": 8,
			"stopBits": 1
		}
	}
}
```
```json
{
	"type": "HTTP",
	"configuration": {
		"converterConfigurations": [{
			"converterId": "123",
			"converters": [{
				"deviceNameJsonExpression": "123",
				"deviceTypeJsonExpression": "213",
				"attributes": [],
				"timeseries": []
			}]
		}]
	}
}
```
但是如果用这种解析真的太肉疼了。
```go
    var conf interface{}
    if err := json.Unmarshal([]byte(input), &conf); err != nil {
        log.Fatal(err)
    }

    var t string = conf.(map[string]interface{})["type"].(string)
    
    switch t{
    case "MODBUS":
        conf.(map[string]interface{})[configuration].(map[string]interface{})["transport"]
    }
    ~~~~~~~~~~
```

### json.RawMessage + interface{} + 类型断言
json.RawMessage是用来延迟解析json数据
首先我们可以把type解析出来，根据type类型，解析configuration成不同类型存放到interface{}的obj中，使用时候根据类型来断言。

```go
type Configuration struct {
	Type string
	Config interface{}
}

type Transport struct {
	``````
}

type ConverterConfigurations struct {
	``````
}


//////// 解析
var raw json.RawMessage
conf := Configuration {
	Config:&raw,
}

if err := json.Unmarshal([]byte(input), &conf); err != nil {
        log.Fatal(err)
}

switch conf.Type {
    case "MODBUS":
        var t Transport
        if err := json.Unmarshal(raw, &t); err != nil {
            log.Fatal(err)
        }
        conf.Config = t
    case "HTTP":
         var c ConverterConfigurations
         if err := json.Unmarshal(raw, &c); err != nil {
                log.Fatal(err)
          }
         conf.Config = c
    }

/////// 使用
    switch conf.Config.(type)
    case Transport:
		//todo
	case ConverterConfigurations:
		//todo
```

如果是type和是一层的话，我们可以定义个Type字段的struct先解析type字段，然后再直接解析具体类型。
```json
{
	"type": "MODBUS",
	"transport": {
		"type": "rtu",
		"timeout": 3000,
		"portName": "ttyM0",
		"encoding": "rtu",
		"parity": "none",
		"baudRate": 9600,
		"dataBits": 8,
		"stopBits": 1
	}
}
```
```go
type DataType struct {
    Type string
}


var tp DataType

if err := json.Unmarshal([]byte(input), &tp); err != nil {
        log.Fatal(err)
}

switch tp.Type {
    case "MODBUS":
        var t Transport
        if err := json.Unmarshal([]byte(input), &t); err != nil {
            log.Fatal(err)
        }
        conf.Config = t
    case "HTTP":
         var c ConverterConfigurations
         if err := json.Unmarshal([]byte(input), &c); err != nil {
                log.Fatal(err)
          }
         conf.Config = c
    }
```