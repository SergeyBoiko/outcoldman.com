---
layout: post
title: "Silverlight basics. Validation. Part 2. IDataErrorInfo & INotifyDataErrorInfo"
date: 2010-11-24 21:18:00
categories: en
tags: [Silverlight, XAML, Validation]
alias: en/blog/show/260
---
<p>At <a href="/en/blog/show/259">previous my blog post</a> I wrote about how to implement validation with DataAnnotations. I this part of my article I will describe interfaces <a href="http://msdn.microsoft.com/en-us/library/system.componentmodel.idataerrorinfo(VS.95).aspx">IDataErrorInfo</a>and <a href="http://msdn.microsoft.com/en-us/library/system.componentmodel.inotifydataerrorinfo(VS.95).aspx">INotifyDataErrorInfo</a>. I recommend you to read <a href="/en/blog/show/259">first part</a> before read this part. Because I use example from previous part.</p>    <h2>Few words about ValidationOnExceptions</h2>  <p>Forgot to tell you, if you want implement validation with exceptions, you may not use DataAnnotations, you can throw your own exceptions from set methods. For example, you can implement check of password confirmation like this:</p>  

```
[Display(Name = "New password confirmation")]
public string NewPasswordConfirmation
{
    get { return _newPasswordConfirmation; }
    set
    {
        _newPasswordConfirmation = value;
        OnPropertyChanged("NewPasswordConfirmation");
        ChangePasswordCommand.RaiseCanExecuteChanged();
        if (string.CompareOrdinal(_newPassword, value) != 0)
            throw new Exception("Password confirmation not equal to password.");
    }
}
```

<p>Looks better than CustomValidationAttribute.</p>

<h2>IDataErrorInfo</h2>

<p>The <a href="http://msdn.microsoft.com/en-us/library/system.componentmodel.idataerrorinfo(VS.95).aspx">IDataErrorInfo</a> interface came with Silverlight 4. If we want implement validation with this interface we should implement it in our ViewModel (one property and one method). Usually developers write <a href="http://johnpapa.net/silverlight/enabling-validation-in-silverlight-4-with-idataerrorinfo/">own class handler</a>, which store validation errors)</p>

```
public class ValidationHandler
{
    private Dictionary<string, string> BrokenRules { get; set; }
 
    public ValidationHandler()
    {
        BrokenRules = new Dictionary<string, string>();
    }
 
    public string this[string property]
    {
        get { return BrokenRules[property]; }
    }
 
    public bool BrokenRuleExists(string property)
    {
        return BrokenRules.ContainsKey(property);
    }
 
    public bool ValidateRule(string property, string message, Func<bool> ruleCheck)
    {
        bool check = ruleCheck();
        if (!check)
        {
            if (BrokenRuleExists(property))
                RemoveBrokenRule(property);
 
            BrokenRules.Add(property, message);
        }
        else
        {
            RemoveBrokenRule(property);
        }
        return check;
    }
 
    public void RemoveBrokenRule(string property)
    {
        if (BrokenRules.ContainsKey(property))
        {
            BrokenRules.Remove(property);
        }
    }
 
    public void Clear()
    {
        BrokenRules.Clear();
    }
}
```

<p>Now let’s rewrite our BindingModel class, we will inherit it from interface IDataErrorInfo and will implement it:</p>

```
public class BindingModel : INotifyPropertyChanged, IDataErrorInfo
{
    private string _newPassword;
    private string _newPasswordConfirmation;
    private readonly ValidationHandler _validationHandler = new ValidationHandler();
 
    #region INotifyPropertyChanged
 
    public event PropertyChangedEventHandler PropertyChanged = delegate { };
 
    private void OnPropertyChanged(string propertyName)
    {
        PropertyChanged(this, new PropertyChangedEventArgs(propertyName));
    }
 
    #endregion
 
    #region IDataErrorInfo
 
    public string this[string columnName]
    {
        get
        {
            if (_validationHandler.BrokenRuleExists(columnName))
            {
                return _validationHandler[columnName];
            }
            return null;
        }
    }
 
    public string Error
    {
        get { return throw new NotImplementedException(); }
    }
 
    #endregion
}
```

<p>The main properties for our BindingModel will look like this:</p>

```
[Display(Name = "New password")]
public string NewPassword
{
    get { return _newPassword; }
    set
    {
        _newPassword = value;
        OnPropertyChanged("NewPassword");
 
        if (_validationHandler.ValidateRule("NewPassword", "New password required", 
                                    () => !string.IsNullOrEmpty(value)))
        {
            _validationHandler.ValidateRule("NewPassword", "Max length of password is 80 symbols.", 
                                    () => value.Length < 80);
        }
 
        ChangePasswordCommand.RaiseCanExecuteChanged();
    }
}
 
[Display(Name = "New password confirmation")]
public string NewPasswordConfirmation
{
    get { return _newPasswordConfirmation; }
    set
    {
        _newPasswordConfirmation = value;
        OnPropertyChanged("NewPasswordConfirmation");
 
        _validationHandler.ValidateRule("NewPasswordConfirmation", "Password confirmation not equal to password.",
                                        () => string.CompareOrdinal(_newPassword, value) == 0);
 
        ChangePasswordCommand.RaiseCanExecuteChanged();
    }
}
```

<p>Each call of VaidationRule checks some condition, and if it not true, then write info about this validation error in errors collection. After binding Silverlight infrastructure will call get method of indexing property this[string columnName] and it will return error information for this property (columnName). We should set property <a href="http://msdn.microsoft.com/en-us/library/system.windows.data.binding.validatesondataerrors(VS.95).aspx">ValidatesOnDataErrors</a> in our binding to true if we want to use this kind of validation. Property Error throw NotImplementedException because Silverlight doesn’t use it. Quotation from <a href="http://msdn.microsoft.com/en-us/library/system.componentmodel.idataerrorinfo(VS.95).aspx">MSDN</a>: “<em>Note that the binding engine never uses the Error property, although you can use it in custom error reporting to display object-level errors</em>.” </p>

<p>In end we should implement again methods for command:</p>

```
public BindingModel()
{
    ChangePasswordCommand = new DelegateCommand(ChangePassword, CanChangePassword);
}
 
public DelegateCommand ChangePasswordCommand { get; private set; }
 
private bool CanChangePassword(object arg)
{
    return !string.IsNullOrEmpty(_newPassword) 
        && string.CompareOrdinal(_newPassword, _newPasswordConfirmation) == 0;
}
 
private void ChangePassword(object obj)
{
    if (ChangePasswordCommand.CanExecute(obj))
    {
        MessageBox.Show("Bingo!");
    }
}
```

<p>We again should use <em>CanChangePassword</em> method, because we need to set button to disabled state when object is invalid. We can’t check valid state of whole object before binding happened. Another problem with this realization: we should write validation rules twice: in property set methods and at CanChangePassword method. But this is problem of this realization; you can solve it with another realization: you can write some class <em>ValidationHandler</em>, which will store not only validation errors, but validation rules, so you can raise validation check in CanChangePassword method. But we still have a problem, we can’t tell to ValidationSummary or some other control at our interface that some validation errors happened or disappeared without binging. </p>

<p>Also you can use DataAnnotation for this approach, but you should write some helper method for that, I will tell you how in next example. </p>

<p>Result of IDataErrorInfo realization (Silverlight sample):</p>
<object data="data:application/x-silverlight-2," type="application/x-silverlight-2" width="100%" height="150px">
		  <param name="source" value="/library/content/03/SLValidation/SilverlightValidation2.xap" />
		  <param name="background" value="white" />
		  <param name="minRuntimeVersion" value="4.0.50826.0" />
		  <param name="autoUpgrade" value="true" />
		  <a href="http://go.microsoft.com/fwlink/?LinkID=149156&amp;v=4.0.50826.0" style="text-decoration:none">
 			  <img src="http://go.microsoft.com/fwlink/?LinkId=161376" alt="Get Microsoft Silverlight" style="border-style:none" />
		  </a>
	    </object>

<p>I think that behavior of this sample the same like in previous part. I want to say that this sample has bug, if user will input <em>Password Confirmation</em> first and then <em>New Password</em>, he will see validation error about password confirmation not equal, because this check happened only on <em>NewPasswordConfirmation</em> binding.&#160; </p>

<h2>INotifyDataErrorInfo</h2>

<p>The <a href="http://msdn.microsoft.com/en-us/library/system.componentmodel.inotifydataerrorinfo(VS.95).aspx">INotifyDataErrorInfo</a>interface came with Silverlight 4 too. The main advantage of this interface you can do synchronous (like in previous samples) and asynchronous validation too. You can wait validation from server, and only after that tell to interface that all ok or some validation error occurred. I like this way of validation more than other. I will use some classes and realization from articles of <a href="http://davybrion.com">Davy Brion</a> about “<a href="http://davybrion.com/blog/2010/08/mvp-in-silverlightwpf-series/">MVP In Silverlight/WPF Series</a>”.</p>

<p>First I got class <em>PropertyValidation</em>, with it we will store validation rule for properties and message which should be showed when validation error occurred. </p>

```
public class PropertyValidation<TBindingModel>
    where TBindingModel : BindingModelBase<TBindingModel>
{
    private Func<TBindingModel, bool> _validationCriteria;
    private string _errorMessage;
    private readonly string _propertyName;
 
    public PropertyValidation(string propertyName)
    {
        _propertyName = propertyName;
    }
 
    public PropertyValidation<TBindingModel> When(Func<TBindingModel, bool> validationCriteria)
    {
        if (_validationCriteria != null)
            throw new InvalidOperationException("You can only set the validation criteria once.");
 
        _validationCriteria = validationCriteria;
        return this;
    }
 
    public PropertyValidation<TBindingModel> Show(string errorMessage)
    {
        if (_errorMessage != null)
            throw new InvalidOperationException("You can only set the message once.");
 
        _errorMessage = errorMessage;
        return this;
    }
 
    public bool IsInvalid(TBindingModel presentationModel)
    {
        if (_validationCriteria == null)
            throw new InvalidOperationException(
                "No criteria have been provided for this validation. (Use the 'When(..)' method.)");
 
        return _validationCriteria(presentationModel);
    }
 
    public string GetErrorMessage()
    {
        if (_errorMessage == null)
            throw new InvalidOperationException(
                "No error message has been set for this validation. (Use the 'Show(..)' method.)");
 
        return _errorMessage;
    }
 
    public string PropertyName
    {
        get { return _propertyName; }
    }
}
```

<p>You will understand how it works when we will start implement validation rules in our example. This class has generic parameter with base class <em>BindingModelBase&lt;T&gt;,</em> from which we will inherit our main <em>BindingModel</em> class.</p>

<p>Let’s implement <em>BindingModelBase</em> class, we will inherit it from <em>INotifyPropertyChanged</em> and <em>INotifyDataErrorInfo</em> interfaces, and will add two fields, one for store validation rules and one for store validation errors:</p>

```
public abstract class BindingModelBase<TBindingModel> : INotifyPropertyChanged, INotifyDataErrorInfo
    where TBindingModel : BindingModelBase<TBindingModel>
{
    private readonly List<PropertyValidation<TBindingModel>> _validations = new List<PropertyValidation<TBindingModel>>();
    private Dictionary<string, List<string>> _errorMessages = new Dictionary<string, List<string>>();
 
    #region INotifyDataErrorInfo
 
    public IEnumerable GetErrors(string propertyName)
    {
        if (_errorMessages.ContainsKey(propertyName)) 
            return _errorMessages[propertyName];
 
        return new string[0];
    }
 
    public bool HasErrors
    {
        get { return _errorMessages.Count > 0; }
    }
 
    public event EventHandler<DataErrorsChangedEventArgs> ErrorsChanged = delegate { };
 
    private void OnErrorsChanged(string propertyName)
    {
        ErrorsChanged(this, new DataErrorsChangedEventArgs(propertyName));
    }
 
    #endregion
 
    #region INotifyPropertyChanged
 
    public event PropertyChangedEventHandler PropertyChanged = delegate { };
 
    protected void OnPropertyChanged(string propertyName)
    {
        PropertyChanged(this, new PropertyChangedEventArgs(propertyName));
    }
 
    #endregion
}
```

<p>Also I want to add one helper method additional to OnPropertyChanged, it will be method which has Expression in parameters and will let us to use it like this:</p>

```
public string NewPassword
{
    get { return _newPassword; }
    set { _newPassword = value; OnCurrentPropertyChanged(); }
}
```

<p>This method is very useful. Implementation of this method:</p>

```
public string NewPassword
{
    get { return _newPassword; }
    set { _newPassword = value; OnPropertyChanged(() => NewPassword); }
}
```

<p>Implementation:</p>

```
protected void OnPropertyChanged(Expression<Func<object>> expression)
{
    OnPropertyChanged(GetPropertyName(expression));
}
 
private static string GetPropertyName(Expression<Func<object>> expression)
{
    if (expression == null)
        throw new ArgumentNullException("expression");
 
    MemberExpression memberExpression;
 
    if (expression.Body is UnaryExpression)
        memberExpression = ((UnaryExpression)expression.Body).Operand as MemberExpression;
    else
        memberExpression = expression.Body as MemberExpression;
    
    if (memberExpression == null)
        throw new ArgumentException("The expression is not a member access expression", "expression");
 
    var property = memberExpression.Member as PropertyInfo;
    if (property == null)
        throw new ArgumentException("The member access expression does not access a property", "expression");
 
    var getMethod = property.GetGetMethod(true);
    if (getMethod.IsStatic)
        throw new ArgumentException("The referenced property is a static property", "expression");
 
    return memberExpression.Member.Name;
}
```

<p>Method get property name from expression. Next, let’s add some methods which will perform validation:</p>

```
public void ValidateProperty(Expression<Func<object>> expression)
{
    ValidateProperty(GetPropertyName(expression));
}
 
private void ValidateProperty(string propertyName)
{
    _errorMessages.Remove(propertyName);
 
    _validations.Where(v => v.PropertyName == propertyName).ToList().ForEach(PerformValidation);
    OnErrorsChanged(propertyName);
    OnPropertyChanged(() => HasErrors);
}
 
private void PerformValidation(PropertyValidation<TBindingModel> validation)
{
    if (validation.IsInvalid((TBindingModel) this))
    {
        AddErrorMessageForProperty(validation.PropertyName, validation.GetErrorMessage());
    }
}
 
private void AddErrorMessageForProperty(string propertyName, string errorMessage)
{
    if (_errorMessages.ContainsKey(propertyName))
    {
        _errorMessages[propertyName].Add(errorMessage);
    }
    else
    {
        _errorMessages.Add(propertyName, new List<string> {errorMessage});
    }
}
```

<p>Method <em>ValidateProperty</em> delete information about all validation errors which was happened with current property, then check each rule for current property, and if some rule is false it will add error to errors collection. Also we can raise validation check for each property auto with PropertyChanged event:</p>

```
protected BindingModelBase()
{
    PropertyChanged += (s, e) => { if (e.PropertyName != "HasErrors") ValidateProperty(e.PropertyName); };
}
```

<p>For easy add validation rules to our collection we will add next method:</p>

```
protected PropertyValidation<TBindingModel> AddValidationFor(Expression<Func<object>> expression)
{
    var validation = new PropertyValidation<TBindingModel>(GetPropertyName(expression));
    _validations.Add(validation);
    return validation;
}
```

<p>Now we can implement <em>BindingModel</em> class, which we will use in our last example. If we want to implement validation with <em>INotifyDataErrorInfo</em> interface we should set to true property <a href="http://msdn.microsoft.com/en-us/library/system.windows.data.binding.validatesonnotifydataerrors(VS.95).aspx">ValidatesOnNotifyDataErrors</a>in bindings.</p>

<p><em>BindingModel</em> implementation:</p>

```
public class BindingModel : BindingModelBase<BindingModel>
{
    private string _newPassword;
    private string _newPasswordConfirmation;
 
    public DelegateCommand ChangePasswordCommand { get; private set; }
 
    public BindingModel()
    {
        ChangePasswordCommand = new DelegateCommand(ChangePassword);
 
        AddValidationFor(() => NewPassword)
            .When(x => string.IsNullOrEmpty(x._newPassword))
            .Show("New password required field.");
 
        AddValidationFor(() => NewPassword)
            .When(x => !string.IsNullOrEmpty(x._newPassword) && x._newPassword.Length > 80)
            .Show("New password must be a string with maximum length of 80.");
 
        AddValidationFor(() => NewPasswordConfirmation)
            .When(x => !string.IsNullOrEmpty(x._newPassword) && string.CompareOrdinal(x._newPassword, x._newPasswordConfirmation) != 0)
            .Show("Password confirmation not equal to password.");
    }
 
    [Display(Name = "New password")]
    public string NewPassword
    {
        get { return _newPassword; }
        set
        {
            _newPassword = value;
            OnPropertyChanged(() => NewPassword);
        }
    }
    
    [Display(Name = "New password confirmation")]
    public string NewPasswordConfirmation
    {
        get { return _newPasswordConfirmation; }
        set
        {
            _newPasswordConfirmation = value;
            OnPropertyChanged(() => NewPasswordConfirmation);
        }
    }
 
    private void ChangePassword(object obj)
    {
        throw new NotImplementedException();
    }
}
```

<p>In ctor we describe all three validation rules for properties. Look’s very good (Thanks Davy Brion!). I told you that I don’t like to set disabled button, so from now it will be always enabled. For realization command’s method <em>ChangePassword</em> I need some method which will check all validation rules of current object, it will be <em>ValidateAll</em> method, which I will implement in <em>BindingModelBase</em> class:</p>

```
public void ValidateAll()
{
    var propertyNamesWithValidationErrors = _errorMessages.Keys;
 
    _errorMessages = new Dictionary<string, List<string>>();
 
    _validations.ForEach(PerformValidation);
 
    var propertyNamesThatMightHaveChangedValidation =
        _errorMessages.Keys.Union(propertyNamesWithValidationErrors).ToList();
 
    propertyNamesThatMightHaveChangedValidation.ForEach(OnErrorsChanged);
 
    OnPropertyChanged(() => HasErrors);
}
```

<p>This method deletes all validation errors in collection. Then check validation rules, write errors if rule is false, and then raise <em>OnErrorsChanged</em> event for each property which has changed validation state.</p>

<p>Implementation of method <em>ChangePassword</em>:</p>

```
private void ChangePassword(object obj)
{
    ValidateAll();
 
    if (!HasErrors)
    {
        MessageBox.Show("Bingo!");
    }
}
```

<p>Result (Silverlight application):</p>
<object data="data:application/x-silverlight-2," type="application/x-silverlight-2" width="100%" height="150px">
		  <param name="source" value="/library/content/03/SLValidation/SilverlightValidation3.xap" />
		  <param name="background" value="white" />
		  <param name="minRuntimeVersion" value="4.0.50826.0" />
		  <param name="autoUpgrade" value="true" />
		  <a href="http://go.microsoft.com/fwlink/?LinkID=149156&amp;v=4.0.50826.0" style="text-decoration:none">
 			  <img src="http://go.microsoft.com/fwlink/?LinkId=161376" alt="Get Microsoft Silverlight" style="border-style:none" />
		  </a>
	    </object>

<p>I like this realization. It much more flexible and it can use all advantages of previous variants. What about DataAnnotation? If you like to describe validation rules with data annotation attributes I can give you one more help method for that. This method will get all rules from attributes and convert them to <em>PropertyValidation</em>:</p>

```
protected PropertyValidation<TBindingModel> AddValidationFor(Expression<Func<object>> expression)
{
    return AddValidationFor(GetPropertyName(expression));
}
 
protected PropertyValidation<TBindingModel> AddValidationFor(string propertyName)
{
    var validation = new PropertyValidation<TBindingModel>(propertyName);
    _validations.Add(validation);
 
    return validation;
}
 
protected void AddAllAttributeValidators()
{
    PropertyInfo[] propertyInfos = GetType().GetProperties(BindingFlags.Public | BindingFlags.Instance);
 
    foreach (PropertyInfo propertyInfo in propertyInfos)
    {
        Attribute[] custom = Attribute.GetCustomAttributes(propertyInfo, typeof(ValidationAttribute), true);
        foreach (var attribute in custom)
        {
            var property = propertyInfo;
            var validationAttribute = attribute as ValidationAttribute;
 
            if (validationAttribute == null)
                throw new NotSupportedException("validationAttribute variable should be inherited from ValidationAttribute type");
 
            string name = property.Name;
 
            var displayAttribute = Attribute.GetCustomAttributes(propertyInfo, typeof(DisplayAttribute)).FirstOrDefault() as DisplayAttribute;
            if (displayAttribute != null)
            {
                name = displayAttribute.GetName();
            }
 
            var message = validationAttribute.FormatErrorMessage(name);
 
            AddValidationFor(propertyInfo.Name)
                .When(x =>
                {
                    var value = property.GetGetMethod().Invoke(this, new object[] { });
                    var result = validationAttribute.GetValidationResult(value,
                                                            new ValidationContext(this, null, null) { MemberName = property.Name });
                    return result != ValidationResult.Success;
                })
                .Show(message);
 
        }
    }
}
```

<p>And the last <em>BindingModel</em> variant:</p>
 

```
public class BindingModel : BindingModelBase<BindingModel>
{
    private string _newPassword;
    private string _newPasswordConfirmation;
 
    public DelegateCommand ChangePasswordCommand { get; private set; }
 
    public BindingModel()
    {
        ChangePasswordCommand = new DelegateCommand(ChangePassword);
 
        AddAllAttributeValidators();
 
        AddValidationFor(() => NewPasswordConfirmation)
            .When(x => !string.IsNullOrEmpty(x._newPassword) && string.CompareOrdinal(x._newPassword, x._newPasswordConfirmation) != 0)
            .Show("Password confirmation not equal to password.");
    }
 
    [Display(Name = "New password")]
    [Required]
    [StringLength(80, ErrorMessage = "New password must be a string with maximum length of 80.")]
    public string NewPassword
    {
        get { return _newPassword; }
        set
        {
            _newPassword = value;
            OnPropertyChanged(() => NewPassword);
        }
    }
 
    [Display(Name = "New password confirmation")]
    public string NewPasswordConfirmation
    {
        get { return _newPasswordConfirmation; }
        set
        {
            _newPasswordConfirmation = value;
            OnPropertyChanged(() => NewPasswordConfirmation);
        }
    }
 
    private void ChangePassword(object obj)
    {
        ValidateAll();
 
        if (!HasErrors)
        {
            MessageBox.Show("Bingo!");
        }
    }
}
```

<p>Source code of these samples you can download from my <a href="https://www.assembla.com/code/outcoldman_p/subversion/nodes/BlogProjects/SilverlightValidation">assembla.com</a> repository.</p>
