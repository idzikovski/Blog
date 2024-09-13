---
layout: post
title:  ".NET MAUI: Async UI instantiation"
date:   2024-09-13 00:00:17 +0100
categories: mobile app maui xamarin forms UI async
---

Yes, you read it right, async UI. After years of developing mobile apps, I am fully aware that mixing those two words is a big no no, but bear with me. Recently I found myself in a situation where after migrating a Xamarin.Forms application to .NET MAUI, a page which was made of multiple complex views, was taking a significantly long time to load (see image below). After a bit of testing I was sure that `InitializeComponent` (the parsing of the xaml) was the bottleneck. To address this, I decided to experiment with asynchronous loading for the UI components and I will share my findings with you now.

Before we continue, let's note for the record, that I would strongly advise you to always try to optimize your views by flattening out the hierarchy and reducing the number of nested views where possible, and by reducing the amount of work that is done during the instantiation of your views. However, if you find yourself in a situation where you cannot avoid complex views, or lacking the resources to optimize the views, this little trick might help you out. So let's dive in.

## The Problem: Slow View Initialization

Let's take a look at `VerySlowView`, a simple `ContentView` that introduces a delay during its initialization.

### Example: `VerySlowView`

```csharp
public partial class VerySlowView : ContentView
{
    public static readonly BindableProperty ColorProperty =
          BindableProperty.Create(nameof(Color),
                                  typeof(Color),
                                  typeof(VerySlowView));

    public Color Color
    {
        get => (Color)GetValue(ColorProperty);
        set => SetValue(ColorProperty, value);
    }

    public VerySlowView()
    {
        // Simulating a delay in the view's instantiation
        var delay = Task.Delay(1000);
        delay.Wait();  // Synchronous delay
        InitializeComponent();
    }
}
```

In this example, the `VerySlowView` introduces a 1-second delay upon initialization (`Task.Delay(1000)`), blocking the UI thread and causing the interface to appear unresponsive. When multiple instances of `VerySlowView` are added to a page, like in `SlowUIPage`, this issue compounds.

### Example: `SlowUIPage` with Multiple Slow Views

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             xmlns:asyncuiexample="clr-namespace:AsyncUIExample"
             x:Class="AsyncUIExample.SlowUIPage"
             Title="SlowUIPage">
    <VerticalStackLayout Spacing="10" Margin="5">
        <asyncuiexample:VerySlowView Color="#88c0e3"/>
        <asyncuiexample:VerySlowView Color="#d2e388"/>
        <asyncuiexample:VerySlowView Color="#e38888"/>
    </VerticalStackLayout>
</ContentPage>
```

In `SlowUIPage`, we use three instances of `VerySlowView`. Since the initialization is synchronous, it severely impacts performance, making the page feel sluggish to the user.

<br>
<p align="center">
    <img src="/Blog/assets/images/AsyncUI/before.gif" height="500"/>
</p>
<br>

## The Solution: Asynchronous View Creation

A good approach to mitigate this issue is to instantiate `VerySlowView` objects on a background thread, avoiding blocking the main UI thread. Once the views are ready, they can be assigned to the page on the main thread.

Here’s how we can implement this solution.

### Step 1: Use Bindable Properties for Lazy Loading

First, we define `BindableProperty` for each view in `SlowUIPage`. This allows us to set the views later, once they are fully initialized.

### Solution: `SlowUIPage` with Asynchronous View Creation

```csharp
public partial class SlowUIPage : ContentPage
{
    public static readonly BindableProperty FirstViewProperty =
          BindableProperty.Create(nameof(FirstView),
                                  typeof(View),
                                  typeof(SlowUIPage),
                                  new LoadingView());

    public View FirstView
    {
        get => (View)GetValue(FirstViewProperty);
        set => SetValue(FirstViewProperty, value);
    }

    public static readonly BindableProperty SecondViewProperty =
          BindableProperty.Create(nameof(SecondView),
                                  typeof(View),
                                  typeof(SlowUIPage),
                                  new LoadingView());

    public View SecondView
    {
        get => (View)GetValue(SecondViewProperty);
        set => SetValue(SecondViewProperty, value);
    }

    public static readonly BindableProperty ThirdViewProperty =
          BindableProperty.Create(nameof(ThirdView),
                                  typeof(View),
                                  typeof(SlowUIPage),
                                  new LoadingView());

    public View ThirdView
    {
        get => (View)GetValue(ThirdViewProperty);
        set => SetValue(ThirdViewProperty, value);
    }

    public SlowUIPage()
    {
        InitializeComponent();
    }

    protected override void OnAppearing()
    {
        base.OnAppearing();
        // Offload view creation to a background thread
        Task.Run(CreateViews);
    }

    private void CreateViews()
    {
        // Initialize views on a background thread
        var firstView = new VerySlowView { Color = Color.FromArgb("#88c0e3") };
        var secondView = new VerySlowView { Color = Color.FromArgb("#d2e388") };
        var thirdView = new VerySlowView { Color = Color.FromArgb("#e38888") };

        // Assign views to the bindable properties on the main thread
        MainThread.BeginInvokeOnMainThread(() =>
        {
            FirstView = firstView;
            SecondView = secondView;
            ThirdView = thirdView;
        });
    }
}
```

### Step 2: Offloading View Creation

The key here is the `Task.Run(CreateViews)` method, which executes the view instantiation in the background. By offloading this work, we keep the UI thread responsive.

### Step 3: Updating the UI on the Main Thread

Once the views are created in the background, they need to be assigned to the page's UI elements. Since only the main thread can update the UI, we use `MainThread.BeginInvokeOnMainThread()` to perform the assignment.

### Bonus: `LoadingView` as a Placeholder

The `LoadingView` serves as a placeholder until the `VerySlowView` instances are ready. You can design `LoadingView` to show a spinner or a message to the user, improving the overall experience.

<br>
<p align="center">
    <img src="/Blog/assets/images/AsyncUI/after.gif" height="500"/>
</p>
<br>

## The Benefits

- **Responsiveness**: The UI remains responsive while the views are being initialized.
- **User Experience**: Users see a loading indicator or placeholder instead of a frozen interface.

## Conclusion

UI optimization is very important when developing mobile applications, but when not possible, offloading heavy operations to a background thread can significantly improve the user experience. By using bindable properties and updating the UI on the main thread, we can create responsive and smooth interfaces even when working with complex or time-consuming views.

You can find the example code [here.](https://github.com/ProblemSolvingStories/AsyncUIExample)
