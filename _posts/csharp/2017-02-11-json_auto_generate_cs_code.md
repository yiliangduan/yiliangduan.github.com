---
title: 'Json自动生成C#代码的工具'
date: 2017-02-11 17:03:34
type: "tags"
tags: csharp
comments: false
---


### 需求
由于现在项目里面与服务器通信的数据使用的Json格式，在每次服务器添加协议的时候客户端都要根据服务器返回的数据对照写一遍解析的数据类和解析类，这样重复性的工作比较浪费时间，所以写了一个Json自动生成C#代码的工具，只要把服务器返回的Json数据用这个工具转换一下就可以自动生成对应的C#解析的代码。

<!-- more --> 

### 例子

比如服务器添加了一条数据协议，内容是这样的:

```json
{
    "http_code": 200,
    "data": {
        "code_id": 164872,
        "book_id": 1,
        "free_goods": [
            {
                "price": 10.1,
                "order_id": "12333333",
            }
        ]
    },
    "elapsed": 26
}

```

经过转换工具自动生成:

```cs
using LitJson;

[System.Serializable]
public class Free_goods {

	public double  price{get; set;}
	public string  order_id{get; set;}

} 

[System.Serializable]
public class Data {

	public int code_id {get; set;}
	public int book_id {get; set;}
	public Free_goods[] free_goods{get; set;}

} 

[System.Serializable]
public class GoodsInfo {

	public int http_code {get; set;}
	public Data data{get; set;}
	public int elapsed {get; set;}

} 

public class GoodsInfoParser {

	public GoodsInfo Data {get; private set;}

	public GoodsInfoParser(string jsonStr) {
 		Parse(jsonStr);  
	}

	private void Parse(string jsonStr) {
		Data = JsonMapper.ToObject<GoodsInfo>(jsonStr);  
	}

 }
```

那么我们在使用的时候只需要使用GoodsInfoParser类即可得到我们需要的数据:

```

//jsonData就是上面例子中的json数据
GoodsInfoParser goodsInfo = new GoodsInfoParser(jsonData);

//这里访问price变量
Console.WriteLine(goodsInfo.Data.data.free_goods[0].price);

```

这样不用每次手动去写Free_goods，Data, GoodsInfo这些类了，自动生成也减少了出错的几率。

### 工具代码

>解析的库使用的LitJson

```cs
/*
 * Create by elang
 * 2017-02-11
 */

using System;
using System.IO;
using LitJson;

namespace Json2CSharp
{
	public class GenClassFromJson
	{
		private string outputFilePath;

		private const string propertyTypeDesc = "{get; set;}\n";

		public GenClassFromJson(string jsonStr, string className)
		{
			outputFilePath = "../../" + className + ".cs";

			if (File.Exists(outputFilePath))
			{
				File.Delete(outputFilePath);
			}

			AppendNameSpace("using LitJson;\n\n");

			ParseJson(jsonStr, className);

			AppendParseJsonClass(className);
		}

		private string GetValueType(JsonReader reader)
		{
			return reader.Value != null ? reader.Value.GetType().ToString() : "";
		}

		public void ParseJson(string jsonStr, string rootName)
		{
			JsonReader reader = new JsonReader(jsonStr);

			ParseJsonNode(reader, rootName);
		}

		public void ParseJsonNode(JsonReader reader, string rootName)
		{
			string classDesc = string.Empty;
			string prePropertyValue = string.Empty;

			do
			{
				//Console.WriteLine("{0,14} {1,10} {2,16}", reader.Token, reader.Value, GetValueType(reader));

				if (reader.Token == JsonToken.ObjectStart)
				{
					if (string.IsNullOrEmpty(classDesc))
					{
						classDesc = "\n[System.Serializable]\npublic class " + rootName + " {\n\n";
					}
					else
					{
						string objName = ToHumpFormat(prePropertyValue);

						classDesc += "\tpublic " + objName + " " + prePropertyValue + propertyTypeDesc;

						ParseJsonNode(reader, objName);
					}
				}
				else if (reader.Token == JsonToken.ObjectEnd)
				{
					classDesc += "\n} \n";

					AppendToLocalFile(classDesc);

					return;
				}
				else if (reader.Token == JsonToken.PropertyName)
				{
					prePropertyValue = reader.Value.ToString();
				}
				else if (reader.Token == JsonToken.ArrayStart)
				{
					reader.Read();

					if (reader.Token == JsonToken.ObjectStart)
					{
						string objName = ToHumpFormat(prePropertyValue);
						classDesc += "\tpublic " + objName + "[] " + prePropertyValue + propertyTypeDesc;

						ParseJsonNode(reader, objName);
					}
				}
				else if (reader.Token == JsonToken.ArrayEnd)
				{
					//nothing to do
				}
				else if (reader.Token == JsonToken.Int)
				{
					classDesc += "\tpublic int " + prePropertyValue + " {get; set;}\n";
				}
				else if (reader.Token == JsonToken.Long || 
				         reader.Token == JsonToken.Boolean || 
				         reader.Token == JsonToken.Double || 
				         reader.Token == JsonToken.String
				        )
				{
					string type = GetValueType(reader);

					string[] typeParts = type.Split('.');

					classDesc += "\tpublic " + typeParts[typeParts.Length - 1].ToLower() + "  " + prePropertyValue + propertyTypeDesc;
				}

			} while (reader.Read());

			Console.WriteLine("Json to csharp complete!");
		}

		private void AppendParseJsonClass(string className)
		{
			string parseClassDesc = "\n\npublic class " + className + "Parser {\n\n";

			parseClassDesc += "\tpublic " + className + " Data {get; private set;}\n\n";

			parseClassDesc += "\tpublic " + className + "Parser(string jsonStr) {\n \t\tParse(jsonStr);  \n\t}\n\n";

			parseClassDesc += "\tprivate void Parse(string jsonStr) {\n\t\tData = JsonMapper.ToObject<" + className + ">(jsonStr);  \n\t}\n";

			parseClassDesc += "\n }";

			AppendToLocalFile(parseClassDesc);
		}

		void AppendNameSpace(string namespaceStr)
		{
			AppendToLocalFile(namespaceStr);

			//TODO
		}

		private string ToHumpFormat(string value)
		{
			return value.Substring(0, 1).ToUpper() + value.Substring(1, value.Length - 1);
		}

		private void AppendToLocalFile(string classDesc)
		{
			File.AppendAllText(outputFilePath, classDesc);
		}
	}
}

```
