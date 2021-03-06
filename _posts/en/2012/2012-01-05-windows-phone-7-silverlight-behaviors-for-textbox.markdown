---
layout: post
title: "Windows Phone 7 Silverlight: Behaviors for TextBox"
date: 2012-01-05 05:21:00
categories: en
tags: [Silverlight, XAML, Expression Blend, Binding, Windows Phone 7, Behavior, Blend, Interactivity]
alias: en/blog/show/308
---
<p>I’ve submitted my first application for Windows Phone! It is very simple app. Marketplace has a good amount of similar apps: <a href="http://www.windowsphone.com/en-US/apps/f5cf9254-ab40-44be-a042-01a76091076c">Planning Poker</a>. Anyway, I’ve wrote couple own behaviors for TextBox and want to share this code with you.</p>  <p>If you are Silverlight/WPF developer and have already used Blend than you should know what are Behaviors and Interactions. If not or if you forget what it is I will remind you. You can find these libraries in directory “c:\Program Files (x86)\Microsoft SDKs\Expression\Blend\” (in case if you have Windows x86 than you should remove from path (x86)). And of course these libraries exist only in case if Blend has been installed. In this directory you can find packs of libraries for WPF/Silverlight and WindowsPhone. If you don’t know what are Behaviors and Interaction I suggest you to read <a href="http://msdn.microsoft.com/en-us/library/dd440765.aspx">Expression Blend SDK for Windows Phone</a> on MSDN. In few words: this is approach to extend default control’s features, more than! this approach allows you to use MVVM pattern. </p>  <p>Let’s extend some features of default TextBox control.</p>    <h2>FocusOnLoadedBehavior</h2>  <p>If you will try to open phone contact’s item to edit on your device you’ll see that device will set focus on first TextBox. This is very good user experience: if you open something for edit than most likely you will put focus on TextBox to edit data in this TextBox. Otherwise it is good question what if this page will have more than one TextBox, should we place focus on first of them or not? Windows Phone does it, it places focus on first TextBox if you will open edit page of contact’s name. </p>  <p>I found that it will be good to do the same in some of my pages, so I wrote next behavior:</p>  

```
/// <summary>
/// UI Behavior to set focus to <see cref="AssociatedObject"/> on loaded event.
/// </summary>
public class FocusOnLoadedBehavior : Behavior<Control>
{
    public static readonly DependencyProperty SetFocusOnLoadedProperty =
        DependencyProperty.Register("SetFocusOnLoaded", typeof(bool), typeof(FocusOnLoadedBehavior), new PropertyMetadata(true));
 
    /// <summary>
    /// Set focus to <see cref="AssociatedObject"/> on loaded event. By default <value>true</value>.
    /// </summary>
    public bool SetFocusOnLoaded
    {
        get { return (bool)GetValue(SetFocusOnLoadedProperty); }
        set { SetValue(SetFocusOnLoadedProperty, value); }
    }
 
    protected override void OnAttached()
    {
        base.OnAttached();
 
        AssociatedObject.Loaded += AssociatedObject_Loaded;
    }
 
    protected override void OnDetaching()
    {
        base.OnDetaching();
 
        AssociatedObject.Loaded -= AssociatedObject_Loaded;
    }
 
    void AssociatedObject_Loaded(object sender, RoutedEventArgs e)
    {
        AssociatedObject.Loaded -= AssociatedObject_Loaded;
 
        if (SetFocusOnLoaded)
        {
            AssociatedObject.Focus();
        }
    }
}
```

<p>It is easy to use it. Just place it for one of your TextBox like I will show below and this behavior will set focus on Loaded event (in most cases it will be the moment when user opens page with this control):</p>

```
<TextBox MaxLength="255" Text="{Binding Path=DisplayName, Mode=TwoWay}" 
     xmlns:interactivity="clr-namespace:System.Windows.Interactivity;assembly=System.Windows.Interactivity"
     xmlns:oBehaviors="clr-namespace:outcold.Phone.Behaviors;assembly=outcold.Phone">
    <interactivity:Interaction.Behaviors>
        <oBehaviors:FocusOnLoadedBehavior />
    </interactivity:Interaction.Behaviors>
</TextBox>
```

<p>I’ve put namespace definitions “interactivity” and “oBehavior” to TextBox attributes just for example, you can move it on top level, for example you can add it to your page attributes.</p>

<p>This behavior has property SetFocusOnLoad. If you will need to manage when you need set this focus and when not – you can use this property. For example it can be the case when you have two TextBoxes and you need to decide which of them should get focus on page load.</p>

<h2>TextBoxSelectOnFocusBehavior</h2>

<p>This is second must have behavior. If user (or you) set focus to TextBox than in different cases user can expect different default position of cursor. For example if it is TextBox for edit some word than most likely user don’t want to add some letters to this word, in most cases he will delete this word to write new one (for example it can be dictionary), so it is good UX to select whole word on focus set event. To do so you can use next behavior:</p>

```
public enum TextBoxFocusSelectType
{
    None = 0, 
    SelectAll = 1,
    SetCursorToTheEnd = 2
}
 
/// <summary>
/// UI Behavior to set how to select text in TextBox on Focused.
/// </summary>
public class TextBoxSelectOnFocusBehavior : Behavior<TextBox>
{
    public TextBoxSelectOnFocusBehavior()
    {
        Type = TextBoxFocusSelectType.None;
    }
 
    /// <summary>
    /// Set behavior of how to select text in <see cref="AssociatedObject"/>.
    /// By default <value>TextBoxFocusSelectType.None</value>.
    /// </summary>
    public TextBoxFocusSelectType Type
    {
        get;
        set;
    }
 
    protected override void OnAttached()
    {
        base.OnAttached();
 
        AssociatedObject.GotFocus += AssociatedObject_GotFocus;
    }
 
    protected override void OnDetaching()
    {
        base.OnDetaching();
 
        AssociatedObject.GotFocus -= AssociatedObject_GotFocus;
    }
 
    void AssociatedObject_GotFocus(object sender, RoutedEventArgs e)
    {
        if (Type == TextBoxFocusSelectType.SelectAll)
        {
            AssociatedObject.SelectAll();
        }
        else if (Type == TextBoxFocusSelectType.SetCursorToTheEnd)
        {
            AssociatedObject.Select(AssociatedObject.Text.Length, 0);
        }
    }
}
```

<p>To use it just add it to TextBox:</p>

```
<TextBox MaxLength="255" Text="{Binding Path=DisplayName, Mode=TwoWay}" 
     xmlns:interactivity="clr-namespace:System.Windows.Interactivity;assembly=System.Windows.Interactivity"
     xmlns:oBehaviors="clr-namespace:outcold.Phone.Behaviors;assembly=outcold.Phone">
    <interactivity:Interaction.Behaviors>
        <oBehaviors:FocusOnLoadedBehavior />
        <oBehaviors:TextBoxSelectOnFocusBehavior Type="SetCursorToTheEnd" />
    </interactivity:Interaction.Behaviors>
</TextBox>
```

<h2>TextBoxInputRegexFilterBehavior</h2>

<p>The last behavior for today TextBoxInputRegexFilterBehavior, which allows you to filter input text. Actually this behavior can add some unobviousness behavior to your application: user sees some button, but when he presses it he doesn’t see any changes. So you should use it very careful. One place where I used it: I need to get integer values from user, but TextBox has minimum InputScope=”Number” which has digits and dot (or comma, it depends on your device language). I saw in one of the apps that one developer created his own keyboard to get rid of this problem. I didn’t like it, and of course I don’t want to use validators for Windows Phone applications, we have very small screen for validators. So I’ve wrote next behavior:</p>

```
/// <summary>
/// UI behavior for <see cref="TextBox"/> to filter input text with special RegularExpression
/// </summary>
public class TextBoxInputRegexFilterBehavior : Behavior<TextBox>
{
    private Regex _regex;
 
    private string _originalText;
    private int _originalSelectionStart;
    private int _originalSelectionLength;
 
    public string RegularExpression 
    {
        get { return _regex.ToString(); } 
        set 
        {
            if (string.IsNullOrEmpty(value))
            {
                _regex = null;
            }
            else
            {
                _regex = new Regex(value);
            }
        } 
    }
 
    protected override void OnAttached()
    {
        base.OnAttached();
 
        AssociatedObject.TextInputStart += AssociatedObject_TextInputStart;
        AssociatedObject.TextChanged += AssociatedObject_TextChanged;
    }
 
    protected override void OnDetaching()
    {
        base.OnDetaching();
 
        AssociatedObject.TextInputStart -= AssociatedObject_TextInputStart;
        AssociatedObject.TextChanged -= AssociatedObject_TextChanged;
    }
 
    void AssociatedObject_TextInputStart(object sender, TextCompositionEventArgs e)
    {
        if (_regex != null && e.Text != null && !(e.Text.Length == 1 && Char.IsControl(e.Text[0])))
        {
            if (!_regex.IsMatch(e.Text))
            {
                _originalText = AssociatedObject.Text;
                _originalSelectionStart = AssociatedObject.SelectionStart;
                _originalSelectionLength = AssociatedObject.SelectionLength;
            }
        }
    }
 
    public void AssociatedObject_TextChanged(object sender, TextChangedEventArgs e)
    {
        if (_originalText != null)
        {
            string text = _originalText;
            _originalText = null;
            AssociatedObject.Text = text;
            AssociatedObject.Select(_originalSelectionStart, _originalSelectionLength);
        }
    }
}
```

<p>To use it:</p>

```
<TextBox Text="{Binding Path=Value, Mode=TwoWay}" MaxLength="3" Width="100" InputScope="Number" 
        xmlns:interactivity="clr-namespace:System.Windows.Interactivity;assembly=System.Windows.Interactivity"
        xmlns:oBehaviors="clr-namespace:outcold.Phone.Behaviors;assembly=outcold.Phone">
    <interactivity:Interaction.Behaviors>
        <oBehaviors:TextBoxSelectOnFocusBehavior Type="SelectAll" />
        <oBehaviors:TextBoxInputRegexFilterBehavior RegularExpression="[0-9]+" />
    </interactivity:Interaction.Behaviors>
</TextBox>
```

<p>In next topics I will show you some other behaviors which I use for my Windows Phone applications. And don’t forget to read section <a href="http://msdn.microsoft.com/en-us/library/dd440765.aspx">Expression Blend SDK for Windows Phone</a> on MSDN!</p>
