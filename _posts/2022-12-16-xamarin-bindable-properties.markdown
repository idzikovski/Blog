---
layout: post
title:  "Bindable Properties in .NET MAUI"
date:   2022-12-16 14:28:17 +0100
categories: xamarin forms bindable properties
---

Have you ever made an attempt at creating a composite component in .NET MAUI or Xamarin Forms? If so then you probably have encountered **Bindable Properties.** I have seen many articles about how to use **Bindable Properties** to expose internal sub-components in your components to the outside world, but I haven't been able to find a good article explaining how they actually work and play an important role in the **Xamarin Forms** framework. In this post we will look at how they work in the background enabling us to practice the MVVM pattern seamlessly.

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

