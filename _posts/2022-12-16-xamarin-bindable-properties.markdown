---
layout: post
title:  "Bindable Properties in .NET MAUI"
date:   2022-12-16 14:28:17 +0100
categories: xamarin forms bindable properties
---

Have you ever made an attempt at creating a composite component in .NET MAUI or Xamarin Forms? If so then you probably have encountered **Bindable Properties.** 

 **Bindable Properties** are the link between the inner components of which your composite component consists and the outside world as shown in the image below. This link is what allows us to use our composite components together with the MVVM pattern. In this article we will have a look under the hood and see how they play a vital role in the Bindable Infrastructure.

![BindableProperties](/Blog/assets/images/BindableProperties/BindableProperties.png)

If you have developing cross platform apps using .NET for a while now, you are probably familiar with the code shown bellow which is used for creating **Bindable Properties.** Probably you know how to use them to create your own reusable components, but you might have wondered how the two components in the code block work together and somehow magically make our bindings work as they do.

```csharp
public static readonly BindableProperty MessageProperty =
            BindableProperty.Create(nameof(Message),
                                    typeof(string),
                                    typeof(ToastMessage),
                                    default);

public string Message
{
    get => (string)GetValue(MessageProperty);
    set => SetValue(MessageProperty, value);
}
```

If you look at it, it looks a bit like a normal C# property, where `Message` is the public facing property and `MessageProperty` is the private backing field. And that's exactly what it is!! `MessageProperty` is in essence a backing field even though it name suggests otherwise. But not just a regular backing field but a backing field on steroids, enhanced with the ability to support handling of things like value changes, validation, default values etc.

Another confusing thing would be the fact that `MessageProperty` is static but somehow as a so called backing field it should be able to hold different values for all of the instances of our component throughout the entire app. And we will see how that works here.

The contents of the [BindableProperty](https://github.com/dotnet/maui/blob/main/src/Controls/src/Core/BindableProperty.cs){:target="\_blank"} class don't reveal much about how these "backing fields" are connected to their respective properties. In fact all the `BindableProperty.Create` method does is create an `BindableProperty` instance by calling an internal constructor that in turn only sets the passed values.

The key to the link between `Message` and `MessageProperty` lies in the [BindableObject](https://github.com/dotnet/maui/blob/main/src/Controls/src/Core/BindableObject.cs){:target="\_blank"} class. This class is used as a base class for all the components that are to support Data Binding. The `VisualElement` class ultimately derives from `BindableObject` which means all of the things we can see on the screen (including our component) do too.

The glue that bonds "backing field" with property here are the `GetValue` and `SetValues` methods which are defined in the `BindableObject` class. And the actual value of the property is kept in a helper class called `BindablePropertyContext` which is nested inside `BindableObject`. You can see it in the code snippet bellow taken from the actual source code. Note the **Value** property inside it. This is the actual container that keeps the value for a specific **Bindable Property**.

```csharp
...

readonly Dictionary<BindableProperty, BindablePropertyContext> _properties = new Dictionary<BindableProperty, BindablePropertyContext>(4);

...

internal class BindablePropertyContext
{
    public BindableContextAttributes Attributes;
    public BindingBase Binding;
    public Queue<SetValueArgs> DelayedSetters;
    public BindableProperty Property;
    public object Value;

    public bool StyleValueSet;
    public object StyleValue;
}

...
```

An instance of this class is created for each **Bindable Property** in your component the first time the `SetValue` method is called on it, and then it is stored in a `Dictionary` field called **_properties**. Note that the type for the key specified in the declaration is `BindableProperty`. This means that the static bindable property is used as a key to look up the `BindablePropertyContext` object which is tied to a specific instance of your component. 

And there you have it. That is the mystery of the **Bindable Properties** uncovered. Hope you enjoyed this read and found it insightful.