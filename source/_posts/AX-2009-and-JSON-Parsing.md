title: AX 2009 and JSON Parsing
date: 2015-05-13 07:00:00
tags:
 - X++
 - JSON
 - C#
 - CLR Interop
 - .NET
categories:
 - AX 2009
 - Data
---
As our implementation of AX begins to mature and we move into a "wish list" phase, there is more of a demand to integrate other software systems to be accessed via AX. Along the way we've worked with various public APIs that request and respond using different languages. For reading responses, XML tends to be the easiest since there are native classes you can use to load XML and all you need are the respective XPaths to the nodes you want. If you have access to a WCF WSDL file, AIF is your friend (admittedly none of the APIs we have hit fall into this category so I can't speak to them much). However, one common format that is a bit more of a challenge is JavaScript Object Notation - JSON. AX 2009 does not offer any kind of native support in parsing JSON and AX 2012 only offers it via the Retail module.

If you work with .NET, you are probably familiar with JSON.NET - the free open source library that helps serialize and deserialize JSON objects and XML into .NET objects with minimal effort. Since there is no native functionality in AX, we opted to use this library to do all the heavy lifting of deserialization. However, since JSON.NET was designed for use with C# and takes heavy advantage of concepts not available in X++ such as generics, we will have to do some wrapping to get everything to work correctly. Additionally, since AX does not easily support "objects" in the traditional C#/.NET sense, we also need a way to traverse the JSON object - not unlike how we would deal with XML - instead of mapping the JSON to a local class.

With that in mind, here's what we've come up with as a base model/wrapper class:

```axapta JsonReader
class JsonReader
{
    Newtonsoft.Json.Linq.JObject    jObject;
}

public void loadJson(str _json)
{
    ;

    jObject = Newtonsoft.Json.Linq.JObject::Parse(_json);
}

public static JsonReader parseJson(str _json)
{
    JsonReader reader = new JsonReader();
    ;

    reader.loadJson(_json);
    return reader;
}

private anytype traversePath(str                               path,
                             Newtonsoft.Json.Linq.JContainer   obj = jObject)
{
    List                            pathElements;
    ListEnumerator                  le;
    Newtonsoft.Json.Linq.JValue     value;
    Newtonsoft.Json.Linq.JToken     token;
    Newtonsoft.Json.Linq.JTokenType thisType,
                                    nestedType;
    Newtonsoft.Json.Linq.JObject    newObject;
    Newtonsoft.Json.Linq.JArray     newArray;
    str                             current,
                                    thisTypeString,
                                    nestedTypeString;
 
    #define.JObject("Newtonsoft.Json.Linq.JObject")
    #define.JArray ("Newtonsoft.Json.Linq.JArray")
 
    ;
 
    pathElements = strSplit(path, @".\/");
 
    le = pathElements.getEnumerator();
 
    if (le.moveNext())
    {
        current = le.current();
 
        thisType = obj.GetType();
        thisTypeString = thisType.ToString();
 
        switch (thisTypeString)
        {
            case #JObject:
                token = obj.get_Item(current);
                break;
            case #JArray:
                token = obj.get_Item(str2int(current) - 1);
                break;
            default:
                return null;
        }
 
        if (token)
        {
            nestedType = token.GetType();
            nestedTypeString = nestedType.ToString();
 
            if (nestedTypeString != #JObject && nestedTypeString != #JArray)
            {
                switch (thisTypeString)
                {
                    case #JArray:
                        return obj.get_Item(str2int(current) - 1);
                    case #JObject:
                        return obj.get_Item(current);
                    default:
                        return null;
                }
            }
             
            switch (nestedTypeString)
            {
                case #JObject:
                    newObject = Newtonsoft.Json.Linq.JObject::FromObject(token);
                    return this.traversePath(strDel(path, 1, strLen(current) + 1), newObject);
                case #JArray:
                    newArray = Newtonsoft.Json.Linq.JArray::FromObject(token);
                    return this.traversePath(strDel(path, 1, strLen(current) + 1), newArray);
                default:
                    return null;
            }
        }
        else
        {
            return null;
        }
    }
    else
    {
        return null;
    }
}
```

The first couple methods are fairly standard and easy to understand. A load function parses the JSON string and saves it into the class for use elsewhere, and a static method is available to make it easier to create a new class. The traversePath method is actually the most important part of the entire class. You can pass in an object notation path to the node you want to get to - for example, "Message.More.Text" - and the function will return the value of that node. If the node is not found, null is returned. 

In short the process operates:

- Separate the path based on common object notation delimiters ("." "/" and "\") into a list of elements (eg, `"Message.More.Text"` is separated to `<"Message", "More", "Text">`)
- Attempt to find a property in the JSON object given (by default, this is the object loaded into the class) with a name equal to the first path element
    - If the object being checked is an array type, use the one-based index of the array
    - If no property is found, return null 
- Get the type of the value attached to that property 
    - If it is not an object or array, return the value 
    - If the value is another object or array: 
        - Trim the first path element from the original path, leaving the remaining node(s) (eg. `"Message.More.Text"` is adjusted to `"More.Text"`) 
        - Create a new JSON object based on the value of the property (eg. `{"Object":{"Property_1":15, "Property_2":"Stuff"}}` becomes `{"Property_1":15, "Property_2":"Stuff"}`) 
        - Pass the new path and new object recursively back into the function and return the result

By taking advantage of recursion, we can dig as deep into the object as we need until the requested node isn't found or there is a value to return.

Finally, because the CLR object hasn't been marshaled to the X++ data type yet (which generally only happens on assignment), we need to convert the result to the specific value type using some additional methods:

```axapta JsonReader
public real getRealNode(str path)
{
    return this.traversePath(path);
}

public int getIntNode(str path)
{
    return this.traversePath(path);
}

public str getStringNode(str path)
{
    return System.Convert::ToString(this.traversePath(path));
}

public boolean isFound(str _path)
{
    return this.traversePath(_path) != null;
}
```

You'll notice the string function is a little different because marshaling from `System.String` to str doesn't actually work because it is a class, not the base data type. We run it through the `System.Convert` method first to ensure we are getting the base data type. For any other data type you can directly use the result of the `traversePath` method. Keep in mind you can't use the result directly in a strfmt statement (you will get a `ClrObject static method invocation error`), so you will need to first assign them to local variables. 

If you need to loop through an array of object results, you can do something like this:

```axapta
    JsonReader  reader;
    str         json,
                strResult;
    int         i,
                intResult;
    ;

    json = "{\"object\":[{\"Property\":\"Alpha\",\"Value\":1},{\"Property\":\"Beta\",\"Value\":2},{\"Property\":\"Gamma\",\"Value\":3}]}";

    reader = JsonReader::parseJson(json);

    for (i = 1; reader.isFound(strfmt("object.%1.Property", i)); i++)
    {
        strResult = reader.getStringNode(strfmt("object.%1.Property", i));
        intResult = reader.getIntNode(strfmt("object.%1.Value", i));
        info(strfmt("%1 = %2", strResult, intResult));
    }
```
 You will need to make sure you check for a property that is always present in the object - if you have a JSON result that has dynamic objects that hide empty/null values this won't work correctly. You also can't directly check that an array index exists if all it contains are objects (eg. checking for `"object.1"` in the object `"{"object":[{"Property":"Alpha","Value":1},{"Property":"Beta","Value":2}]}"` will return false, even though it is a valid index). If the array is just values however, it will work correctly (eg. checking for `"object.1"` in the object `"{\"object\":[\"Alpha\",\"Beta\",\"Gamma\"]}"` will property return true).

As with most projects, I'm sure this will continue to evolve, and this is by no means a perfect solution. However, it does handle most of the common situations you'll run into with JSON, and provides a reasonable framework to build from.

Questions, comments, missed situations? Shoot me a comment below!