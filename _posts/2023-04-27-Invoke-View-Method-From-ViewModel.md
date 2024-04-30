---
layout: post
title:  "(Don't) Invoke View methods from your ViewModel in .NET MAUI"
date:   2024-04-26 10:00:17 +0100
categories: mobile app maui xamarin forms mvvm
---

When developing a .NET MAUI application using the MVVM pattern, in 90% of the time you won't need to and probably should not invoke View methods from your ViewModels. There are exceptions however, for example when you are doing something related to graphics and you want to send a message to your View to re-draw itself. For this purpose I will show you a simple way to achieve this using Bindable properties.

The basic idea is to use a Bindable property of type ICommand. But instead of using ICommand as a way to react to user input in our ViewModel, the direction would be reversed using the `OneWayToSource` binding mode. The first step would be to create the Bindable property, note how we set the default binding mode to `OneWayToSource`

```csharp
public static readonly BindableProperty MessageProperty =
            BindableProperty.Create(nameof(Message),
                                    typeof(ICommand),
                                    typeof(MyCustomView),
                                    defaultBindingMode: BindingMode.OneWayToSource);

public ICommand UpdateDrawingCommand
{
    get => (ICommand)GetValue(MessageProperty);
    set => SetValue(MessageProperty, value);
}
```
