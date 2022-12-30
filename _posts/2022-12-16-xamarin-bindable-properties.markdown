---
layout: post
title:  "Bindable Properties in .NET MAUI"
date:   2022-12-16 14:28:17 +0100
categories: xamarin forms bindable properties
---

Have you ever made an attempt at creating a composite component in .NET MAUI or Xamarin Forms? If so then you probably have encountered **Bindable Properties.** 

 **Bindable Properties** are the link between the inner components of which your composite component consists and the outside world as shown in the image below. This link is what allows us to use our composite components together with the MVVM pattern. In this article we will have a look under the hood and see how they play a vital role in the Bindable Infrastructure.

![BindableProperties](/Blog/assets/images/BindableProperties/BindableProperties.png)

If you have developing cross platform apps using .NET for a while now, you are probably familiar with the above shown, not so pretty syntax for creating **Bindable Properties.** Probably you know how to use them to create your own reusable components, but you might have wondered how the two components in the code block work together and somehow magically make our bindings work as they do.

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
