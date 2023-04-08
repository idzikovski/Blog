---
layout: post
title:  ".NET MAUI: Display Custom Snackbar from ViewModels"
date:   2023-03-02 10:00:17 +0100
categories: mobile app maui xamarin forms snackbar toast
---

SnackBar is a common UI element that displays a short message, usually at the bottom of the screen, to provide feedback or notify users of an event. It is an essential UI component in any app that provides feedback to users. MAUI provides a built in [snackbar](https://learn.microsoft.com/en-us/dotnet/communitytoolkit/maui/alerts/snackbar?tabs=android){:target="\_blank"} that you could use and customize it's layout an position to a certain degree. In this post I will show you how to create a snackbar with the ability to completely customize it's layout and position, combined with the [builder pattern](https://endjin.com/blog/2019/06/design-patterns-in-csharp-the-builder-pattern#:~:text=The%20builder%20pattern%20is%20used,constructor%20would%20not%20be%20adequate){:target="\_blank"} making it easy to display your message form any ViewModel.

<br>
<p align="center">
    <img src="/Blog/assets/images/MauiSnackbar/demo.gif" height="500"/>
</p>
<br>

To achieve this we will be using the [Mopups](https://github.com/LuckyDucko/Mopups){:target="\_blank"} library which is a .NET MAUI alternative of the famous "Rg.Plugins.Popups" library for Xamarin, used to create pop ups. Creating the custom SnackBar involves three elements: The XAML part of a `PopupPage` which will hold the design and layout of the Snackbar, the code-behind of the same page that defines the SnackBar's behavior, and the ISnackBarBuilder interface, which configures and displays the SnackBar from the ViewModel.

### Creating the design

You will need to create a new `PopupPage`, which derives from `ContentPage` and is made up of two parts: the XAML file, which we will use to define the look of our SnackBar, and the code behind which is where the behavior will be defined, but we will come to that later. You can see an example of the root element of the page in the code snippet below. What is important to note is setting the `BackgroundInputTransparent` property to true so that the user can interact with the UI while the message is showing, and second, to set the `CloseWhenBackgroundIsClicked` to false so that the message doesn't get dismissed when he does.

```xml
<mopups:PopupPage  xmlns:mopups="clr-namespace:Mopups.Pages;assembly=Mopups"
                   xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
                   xmlns:views="clr-namespace:MAUISnackBar.Views"
                   xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
                   x:Class="MAUISnackBar.Views.SnackBar"
                   BackgroundInputTransparent="True"
                   CloseWhenBackgroundIsClicked="False"
                   Title="SnackBar">

...
```

You can see the design I went for in the gif above, and also the corresponding XAML in the code below. It's a simple design consisting of a `Frame` with round corners, a `Label` to show the message, and another `Label` used as a close button using the &#x2715; unicode character. But when it comes to designing your SnackBar, you you have the complete freedom to tailor it according to your needs.

```xml
...

<Frame x:Name="Pancake"
       Margin="20"
       VerticalOptions="End"
       x:FieldModifier="internal"
       BackgroundColor="{Binding ToastBackgroundColor, Source={RelativeSource AncestorType={x:Type views:SnackBar}}}"
       CornerRadius="12"
       Padding="20">
    <Grid ColumnDefinitions="*, Auto"
          RowDefinitions="Auto">
        <Label Grid.Row="0"
               Grid.Column="0"
               Text="{Binding Message, Source={RelativeSource AncestorType={x:Type views:SnackBar}}}"
               TextColor="{Binding TextColor, Source={RelativeSource AncestorType={x:Type views:SnackBar}}}"
               FontSize="15"
               VerticalOptions="Center"/>
        <Label Grid.Row="0"
               Grid.Column="1"
               TextColor="{Binding TextColor, Source={RelativeSource AncestorType={x:Type views:SnackBar}}}"
               FontSize="25"
               Text="&#x2715;">
            <Label.GestureRecognizers>
                <TapGestureRecognizer Tapped="CloseButtonTapped"/>
            </Label.GestureRecognizers>
        </Label>
    </Grid>
</Frame>

...
```

### Describing the behavior

The first thing we would need to do in the code-behind is create bindable properties to expose the things we want to be able to customize from the outside. If you're unfamiliar with bindable properties you could check my previous [blog post](/Blog/mobile/app/maui/xamarin/forms/bindable/properties/2023/01/03/BINDABLE-PROPERTIES-MAUI.html) where I explain them in more detail.

An example for the `Message` bindable property can be seen below. We will use this to easily change the text that we want our SnackBar to display. In a similar fashion, we need to create properties for the `Duration`, to specify the time after which the message will disappear on it's own, as well as the `SnackBarBackgroundColor` and the `TextColor` so that we can also easily modify those.

Note how we use the [RelativeSource](https://learn.microsoft.com/en-us/dotnet/maui/fundamentals/data-binding/relative-bindings?view=net-maui-7.0#bind-to-an-ancestor){:target="\_blank"} markup extension combined with it's `AncestorType` property to connect the inner target properties with the outwards facing bindable properties.

```csharp
public static readonly BindableProperty MessageProperty = 
                            BindableProperty.Create(nameof(Message),
                                                    typeof(string),
                                                    typeof(SnackBar),
                                                    default);

public string Message
{
    get => (string)GetValue(MessageProperty);
    set => SetValue(MessageProperty, value);
}
```

We also need a timer instance to countdown the duration of the message and dismiss it when it has elapsed if the user hasn't explicitly closed it already, and the `SnackBarDismissed` event that will be used to notify the builder that the SnackBar was dismissed.

We make use of the `OnAppearing` method to initialize and start the timer and the `SnackBarDismissed` event is invoked either when the timer elapses or the users taps on the close button as shown in the code block below.

```csharp
...

public event EventHandler SnackBarDismissed;
private Timer _timer;

...

protected override void OnAppearing()
{
    base.OnAppearing();

    _timer = new Timer();
    _timer.Interval = Duration;
    _timer.AutoReset = false;
    _timer.Elapsed += CloseButtonTapped;
    _timer.Start();
}

internal void CloseButtonTapped(object sender, EventArgs e)
{
    SnackBarDismissed?.Invoke(this, EventArgs.Empty);
}
    
...
```

### The SnackBar Builder

With the design and behavior of the SnackBar in place, all that is left is to create the builder which will allow us to set the text, duration and colors for the SnackBar, and ultimately show it from our ViewModel.

It consists of the interface `ISnackBarBuilder`, which will allow us to easily inject it wherever we need it, and the `SnackBarBuilder` which implements it. The interface is a contract with the following methods:

```csharp
public interface ISnackBarBuilder
{
    ISnackBarBuilder Create();

    ISnackBarBuilder WithMessage(string message);

    ISnackBarBuilder WithDuration(double duration);

    ISnackBarBuilder WithBackgroundColor(Color backgroundColor);

    ISnackBarBuilder WithTextColor(Color textColor);

    void Show();
}
```

Note how the methods used to create and build the message all return an instance of `ISnackBarBuilder` so that they can be chained together in a fluent manner. The `Create` and `WithMessage` methods are always needed to initialize the SnackBar, but the duration and colors have default values and so the corresponding methods are optional. And at the end we have the `Show` method that is self explanatory.

```csharp
...

private SnackBar _snackbar;

public ISnackBarBuilder Create()
{
    _snackbar = new SnackBar();
    return this;
}

public ISnackBarBuilder WithMessage(string message)
{
    _snackbar.Message = message;
    return this;
}

...

public void Show()
{
    _snackbar.SnackBarDismissed += MessageDismissed;
    MopupService.Instance.PushAsync(_snackbar);
}

internal void MessageDismissed(object sender, EventArgs e)
{
    var snackbar = (SnackBar)sender;

    snackbar.SnackBarDismissed -= MessageDismissed;
    MopupService.Instance.RemovePageAsync(snackbar);
}

...
```

The implementation contains a private field of the SnackBar page we previously created. In the `Create` method this field is initialized to a new instance and then the `ISnackBarBuilder` instance is returned in order to be able to chain the builder methods together. Similarly, the `WithMessage` method sets the message of the SnackBar and again returns the instance, and so do the rest of the builder methods.

Finally, in the `Show` method, a subscription is made to the `SnackBarDismissed` event and the SnackBar is shown using the `MopupService`. Analogous to this, in the handler method of the `SnackBarDismissed` event, we unsubscribe from the event and remove the SnackBar from the navigation stack.

### Usage

To use the SnackBar, simply inject the `ISnackBarBuilder` into your ViewModel, initialize it and show it like shown below.

```csharp
_snackBarBuilder.Create()
                .WithMessage("Snackbar from VM")
                .WithTextColor(Color.FromRgb(200, 200, 200))
                .WithBackgroundColor(Color.FromRgba(20, 20, 20, 200))
                .WithDuration(5000)
                .Show();
```

As I mentioned before, if your defaults are correctly defined you can reduce the above statement to only set the message and show the SnackBar and only use the full customization for exceptions.

I hope that you enjoyed this guide and found it helpful. If you have any questions or issues don't hesitate to [contact me.](mailto:ivan.dzikovski@gmail.com)
