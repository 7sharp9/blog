+++
author = "7sharp9"
categories = ["programming"]
tags = ["FSharp", "iOS", "Xamarin", "Storyboard", "TypeProvider", "Generative", "fsharpX"]
date = "2017-04-11T23:11:55+01:00"
description = "A post detailing the iOS provider presented at my talk at fsharpX"
title = "I want to tell you a storyboard"
type = "post"
+++
So as promised here's a post with more detail on the iOS designer provider that I presented as part of my talk at [fsharpX 2017](https://skillsmatter.com/conferences/8053-f-sharp-exchange-2017) The talk is entitled [Expanding the Horizons of Mobile Development] (https://skillsmatter.com/skillscasts/10042-lightning-talk-session-expanding-the-horizons-of-mobile-development)
<!--more-->

# Background

In iOS, user interfaces can be represented by storyboards, essentially a storyboard is a big ball of xml that is produced by a visual designer like Xcode or Xamarin's Designer.  

The basic premise of designer based user interfaces in .Net is the concept of a code behind file and and a partial class.  For a C# Xamarin.iOS storyboard project something a bit like this will be present as a partial class:

```csharp
namespace Designer
{
  [Register ("DesignerViewController")]
  partial class DesignerViewController
  {
    [Outlet]
    [GeneratedCodeAttribute ("iOS Designer", "1.0")]
    UIKit.UIButton submitButton { get; set; }

    void ReleaseDesignerOutlets ()
    {
      if (submitButton != null) {
        submitButton.Dispose ();
        submitButton = null;
      }
    }
  }
}
```

With the user using another file with the same partial class to consume the controls generated by the designer:
```csharp
namespace Designer
{
  partial class DesignerViewController
  {
      public override void ViewDidLoad ()
      {
        base.ViewDidLoad();
        submitButton.TouchUpInside += (sender, e) => {
            //Do stuff
        }
      }
  }
```

I've always found the idea of partial class a bit of a cop out and never enjoyed having them around, which is why I do not miss them in F#.  It does, however, pose a bit of problem with dealing with feature parity between C# and F# where designer files are generated as partial classes and you have to implement code in the other partial classes.  In F# that whole aspect is entirely missing due to the lack of partial classes, which means anyone with a penchant for visual designers will be disappointed.  _(You can read more about how the iOS designer works in Xamarin.iOS)_ [here](https://developer.xamarin.com/guides/ios/user_interface/designer/introduction/)

# Is there be a better way?

Of course!  We can use the power of F# type providers!  Although not for the feint hearted to create, F# type providers can be quite a boon to development time with their design time access to intellisense and tooltips based of some kind of underlying schema.  

During my time at Xamarin I did research work on lots of different areas of using F#, one of those was using F# type providers with designer tooling for iOS and Android.  A few weeks back [Miguel de Icaza](https://twitter.com/migueldeicaza) kindly agreed to open source the work so that it could be enjoyed by all!

# Introducing the iOS designer provider

Type providers come in two distinct forms, generative and erasing, generally most people use erasing providers where the code is erased at compile time to objects leaving only raw functionality.  This is particularly useful for large schemas where information is generated on demand or the runtime representation is only data and methods.  Generative providers produce real types that can be consumed and augmented in the normal fashion.  Due to the nature of the iOS runtime and the way the Xamarin iOS works, generative type providers are needed. 

## How does it work?

The storyboard files within the current project are processed and any `ViewControllers` that have been assigned a name are processed by the type provider.  Each control within the `ViewController` are also processed if they have been assigned a name.  A property and disposal logic is generated as well as an abstract type for the controller.  

## Example storyboard

Here is a sample storyboard from tvOS: 

![tvOS storyboard](/img/designer-provider/tvOS-storyboard.png)

The `ViewController` is named `ourViewController`.  The controls are equally imaginatively named as follows:

* label1
* entry1
* entry2
* entry3
* entry4
* button1

## Using the type provider

The first step is to instantiate the type provider, this effectively starts the process of finding the storyboards within the project and generating the types in the background:  

```fsharp
type VCContainer = Xamarin.UIProvider
```

The type bound above `VCContainer` can be called anything you wish I named it `VCContainer` because it contains all of the `ViewControllers` found in the storyboards.  

The next step is to create and register a `ViewController` that was defined in one of the storyboards.  Using auto-completion we can _dot into_ the `VCContainer` to find any of the `ViewControllers` within the storyboard.  

Here the `ViewController` we saw in the screen shot above (`ourViewController`) is exposed with a _base_ suffix `ourViewControllerBase` which is also an abstract type.  We don't want to allow the `ViewController` to be instantiated directly as the iOS runtime requires attributes on the exposed type and as we don't yet have [intrinsic type extensions on provided types](https://github.com/fsharp/fslang-suggestions/issues/509) we don't have a great alternative apart from inheritance or augmentation.  

```fsharp
[<Register(VCContainer.ourViewControllerBase.CustomClass)>]
type myViewController(ptr) =
    inherit VCContainer.ourViewControllerBase(ptr)
```

Due to the way Xamarin.iOS works we have to place a `Register` attribute on the type so that the the iOS runtime can instantiate the correct type when the storyboard is constructed.  Another feature of the iOS designer type provider is that the `CustomClass` name is exposed as a convenience so that you don't have to remember or retype the name of the `ViewController` avoiding stringly typed hell.  You can also see that `ourViewControllerBase` is called using the `nativeint` constructor: `inherit VCContainer.ourViewControllerBase(ptr)`, this is how the storyboard infrastructure in iOS creates an instance of your ViewController`.  

When we come to wire up events and consume the controls we can do so very easily via auto completion.  Say we want to wire up `button1` so that when it is clicked we concatenate the contents of the `Entry` controls and place the text into `label1` we could do so like this:  

```fsharp
    override x.ViewDidLoad () =
        base.ViewDidLoad ()
        x.button1.PrimaryActionTriggered.Add(
            fun _ -> x.View.BackgroundColor <- UIColor.Blue
                     x.label1.Text <-
                        [ x.entry1.Text
                          x.entry2.Text
                          x.entry3.Text
                          x.entyr4.Text ]
                        |> String.concat " " )
```
You can see in `ViewDidLoad` we added an event handler to `button1` and consume the relevant UI components.  

## Benefits

The benefits of this approach depend of what angle you look from.  From the C# approach at the start of this post there's the reduced complexity of not having partial types and the designer generated parts being part of the project.  You still get all the benefits of the UI controls being able to be consumed and available in auto completion etc.  Compared to the current F# approach there's less boiler plate because you dont have to manually add properties and `Outlet` attributes to your `ViewController`, with this approach there's also the pitfall of stringly types which are extremly error prone.  The final advantage is that if the storyboard is changed so that a control is deleted, the compilation will fail at design time rather than runtime.  

Type providers provide a welcome safety net and boiler plate reduction for these type of scenarios.  

## Generated Code

For the curious the generated code in `VCContainer` looks like this:  

```csharp
public sealed class VCContainer
{
  public abstract class ourViewControllerBase : UIViewController
  {
    public const string CustomClass = "ourViewController";
	private UITextField __entry1;
	private UITextField __entry2;
	private UITextField __entry3;
	private UITextField __entyr4;
	private UIButton __button1;
	private UILabel __label1;

	[Outlet]
	public UITextField entry1 {
		get {
		  return this.__entry1;
		}
		set {
		  this.__entry1 = value;
		}
	}

    //entry2/3/4 etc
}
```

Here is the disposal logic generated for each control:  

```csharp
public void ReleaseDesignerOutlets () {
    if (this.__entry1 != null) {
        UnboxGeneric<IDisposable>(this.__entry1).Dispose ();
    }
    if (this.__entry2 != null) {
        UnboxGeneric<IDisposable>(this.__entry2).Dispose ();
    }
    if (this.__entry3 != null) {
        UnboxGeneric<IDisposable>(this.__entry3).Dispose ();
    }
    if (this.__entyr4 != null) {
        UnboxGeneric<IDisposable>(this.__entyr4).Dispose ();
    }
    if (this.__button1 != null) {
        UnboxGeneric<IDisposable>(this.__button1).Dispose ();
    }
    if (this.__label1 != null) {
        UnboxGeneric<IDisposable>(this.__label1).Dispose ();
    }
    if (this.__ourViewController != null) {
        UnboxGeneric<IDisposable>(this.__ourViewController).Dispose ();
    }
}
```

# So whats next?

Since [fsharpX](https://skillsmatter.com/conferences/8053-f-sharp-exchange-2017) Ive been super busy catching up with work so I've only managed to get the slides published for the talk and write this blog post, but fairly soon there will be a nuget package available that will allow the designer provider to be used for iOS, tvOS and watchOS.  There is actually an experimental Android UI and fragment provider too, but that needs a little more work before its released on the general public :-)

If anyone is interested in the inner working of the type provider let me know and I'll draft up some notes on that too.  I know type providers are pretty mysterious and advanced concepts to a lot of people, generative providers doubly so.  

As usual I love to get __any feedback__, comments and suggestions...

Until next time!  
