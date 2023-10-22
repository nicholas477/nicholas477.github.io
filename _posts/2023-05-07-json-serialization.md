---
layout: post
title: Unreal UObject Json Serialzation
description: Man this should be an engine feature
importance: 0
date:   2023-05-07 16:40:16
---

I was working on an analytics collector in Unreal and I wasn't very happy with the system they have in place now, which uses function calls to record events. Each event is either a key/value pair or a key/dict pair, and you have to log each pair manually using function calls. A few lines of json

{% highlight json linenos %}
{
	"sessionId" : "e26380784eff322fd6cb7fb2d145d404-2023.06.24-15.32.43",
	"userId" : "e26380784eff322fd6cb7fb2d145d404",
	"events" : [
		{
			"eventName" : "Build"
,			"attributes" : [
			{
				"name" : "Version",
				"value" : "++UE5+Release-5.2-CL-25360045"
			}
			,
			{
				"name" : "PlatformName",
				"value" : "Windows"
			}
			,
			{
				"name" : "PlatformUserName",
				"value" : "nicholas477"
			}
			,
			{
				"name" : "Config",
				"value" : "Development"
			}
			]
		}
,
		{
			"eventName" : "UserSettings"
,			"attributes" : [
			{
				"name" : "OverallScalability",
				"value" : "Epic"
			}
			,
			{
				"name" : "DesktopResolution",
				"value" : "X=2560 Y=1440"
			}
			]
		}
	]
}
{% endhighlight %}

turns into this spaghetti nightmare:

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/json-serialization/event-recording.png" title="event recording" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

It's an odd design choice for an engine that has serialization built in. So I decided to use that built in serialization and write my own analytics system that serializes objects to json and sends it wherever. But I noticed that while Unreal has support for UStruct serialization to json, it does not have support for UObjects. So I wrote a plugin for that.

Since the property to json serialization is already included in the engine, the object serialization code itself is pretty short, about 100 lines total.

{% highlight c++ linenos %}
#include "JsonObjectConverter.h"
#include "UObject/UnrealType.h"

template<typename T>
static TArray<TSharedPtr<FJsonValue>> SerializePropertyAsJsonArray(const T* Data, FArrayProperty* Property, TSet<const UObject*>& TraversedObjects)
{
	const uint8* PropData = Property->ContainerPtrToValuePtr<uint8>(Data);
	FScriptArrayHelper Helper(Property, PropData);
	TArray<TSharedPtr<FJsonValue>> ValueArray;

	for (int32 i = 0, n = Helper.Num(); i < n; ++i)
	{
		const uint8* InnerPropData = Helper.GetRawPtr(i);
		if (FArrayProperty* ArrayProperty = CastField<FArrayProperty>(Property->Inner)) // Array
		{
			TArray<TSharedPtr<FJsonValue>> InnerArray = SerializePropertyAsJsonArray(InnerPropData, ArrayProperty, TraversedObjects);
			ValueArray.Emplace(new FJsonValueArray(InnerArray));
		}
		else if (FStructProperty* StructProperty = CastField<FStructProperty>(Property->Inner)) // Struct
		{
			TSharedPtr<FJsonObject> StructObject = MakeShareable(new FJsonObject);
			const uint8* StructPropData = StructProperty->ContainerPtrToValuePtr<uint8>(InnerPropData);
			for (TFieldIterator<FProperty> PropertyItr(StructProperty->Struct); PropertyItr; ++PropertyItr)
			{
				SerializePropertyAsJsonObjectField((void*)StructPropData, StructObject, *PropertyItr, TraversedObjects);
			}
			ValueArray.Emplace(new FJsonValueObject(StructObject));
		}
		else if (FObjectProperty* ObjectProperty = CastField<FObjectProperty>(Property->Inner)) // Object
		{
			const UObject* SubObject = ObjectProperty->GetObjectPropertyValue_InContainer(InnerPropData);
			if (SubObject->IsValidLowLevel() && !TraversedObjects.Contains(SubObject))
			{
				TraversedObjects.Add(SubObject);
				TSharedPtr<FJsonObject> JsonSubObject = MakeShared<FJsonObject>();
				for (TFieldIterator<FProperty> PropertyItr(SubObject->GetClass()); PropertyItr; ++PropertyItr)
				{
					SerializePropertyAsJsonObjectField(SubObject, JsonSubObject, *PropertyItr, TraversedObjects);
				}
				ValueArray.Emplace(new FJsonValueObject(JsonSubObject));
			}
		}
		else
		{
			TSharedPtr<FJsonValue> JsonValue;
			const uint8* InnerInnerPropData = Property->Inner->ContainerPtrToValuePtr<uint8>(InnerPropData);
			ValueArray.Emplace(FJsonObjectConverter::UPropertyToJsonValue(Property->Inner, InnerInnerPropData));
		}
	}
	return ValueArray;
}

template<typename T>
static void SerializePropertyAsJsonObjectField(const T* Data, TSharedPtr<FJsonObject> OuterObject, FProperty* Property, TSet<const UObject*>& TraversedObjects)
{
	if (Property->GetName() == "UberGraphFrame"
		|| Property->HasAnyPropertyFlags(CPF_Transient))
	{
		// Don't include "UberGraphFrame" or any transient properties
		return;
	}

	if (FArrayProperty* ArrayProperty = CastField<FArrayProperty>(Property)) // Array
	{
		TArray<TSharedPtr<FJsonValue>> Values = SerializePropertyAsJsonArray(Data, ArrayProperty, TraversedObjects);
		OuterObject->SetArrayField(Property->GetAuthoredName(), Values);
	}
	else if (FStructProperty* StructProperty = CastField<FStructProperty>(Property)) // Struct
	{
		TSharedRef<FJsonObject> StructObject = MakeShareable(new FJsonObject);
		const uint8* PropData = Property->ContainerPtrToValuePtr<uint8>(Data);
		for (TFieldIterator<FProperty> PropertyItr(StructProperty->Struct); PropertyItr; ++PropertyItr)
		{
			SerializePropertyAsJsonObjectField((void*)PropData, StructObject, *PropertyItr, TraversedObjects);
		}
		OuterObject->SetObjectField(Property->GetAuthoredName(), StructObject.ToSharedPtr());
	}
	else if (FObjectProperty* ObjectProperty = CastField<FObjectProperty>(Property)) // Object
	{
		const UObject* SubObject = ObjectProperty->GetObjectPropertyValue_InContainer(Data);
		if (SubObject->IsValidLowLevel() && !TraversedObjects.Contains(SubObject))
		{
			TraversedObjects.Add(SubObject);
			TSharedPtr<FJsonObject> JsonSubObject = MakeShared<FJsonObject>();
			for (TFieldIterator<FProperty> PropertyItr(SubObject->GetClass()); PropertyItr; ++PropertyItr)
			{
				SerializePropertyAsJsonObjectField(SubObject, JsonSubObject, *PropertyItr, TraversedObjects);
			}
			OuterObject->SetObjectField(Property->GetAuthoredName(), JsonSubObject);
		}
	}
	else
	{
		TSharedPtr<FJsonValue> JsonValue;
		const uint8* PropData = Property->ContainerPtrToValuePtr<uint8>(Data);
		OuterObject->SetField(Property->GetAuthoredName(), FJsonObjectConverter::UPropertyToJsonValue(Property, PropData));
	}
}

TSharedPtr<FJsonObject> FJsonSerializationModule::SerializeUObjectToJson(const UObject* Object)
{
	TSet<const UObject*> TraversedObjects;
	TraversedObjects.Add(Object);

	TSharedPtr<FJsonObject> JsonObject = MakeShared<FJsonObject>();

	for (TFieldIterator<FProperty> PropertyItr(Object->GetClass()); PropertyItr; ++PropertyItr)
	{
		SerializePropertyAsJsonObjectField(Object, JsonObject, *PropertyItr, TraversedObjects);
	}

	return JsonObject;
}
{% endhighlight %}

I thought this would be incredibly useful for anyone else who wants to avoid the hassle of the event logging system, so I put the code up on github as a plugin. Grab it below.

# Download Links

- [Plugin Link](https://github.com/nicholas477/JsonSerialization/)