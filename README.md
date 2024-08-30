
在数据绑定过程中，我们经常会使用`StringFormat`对要显示的数据进行格式化，以便获得更为直观的展示效果，但在某些情况下格式化操作并未生效，例如 `Button`的 `Content`属性以及`ToolTip`属性绑定数据进行`StringFormat`时是无效的。首先回顾一下`StringFormat`的基本用法。


## `StringFormat`的用法


`StringFormat`是 `BindingBase`的属性，指定如果绑定值显示为字符串，应如何设置该绑定的格式。因此，`BindingBase` 的三个子类：`Binding`、`MultiBinding`、`PriorityBinding`都可以对绑定数据进行格式化。


### Binding


`Binding` 是最常用的绑定方式，使用`StringFormat`遵循[.Net格式字符串标准](https://github.com)即可。例如：



```


```

或者



```


```

其中`{0}`表示第一个数值，如果 `StringFormat` 属性的值是以花括号开头，前边需要有一对花括号 `{}` 进行转义，也就是第一个例子中的 `{}{0:C}`，否则不需要，如第二个示例一样。
如果设置 `Converter` 和 `StringFormat`属性，则首先将转换器应用于数据值，然后`StringFormat` 应用该值。


### MultiBinding


`Binding` 绑定时，格式化只能指定一个参数，`MultiBinding` 绑定时则可指定多个参数。例如：



```

    
        
            
            
        
    


```

这个例子中 `MultiBinding` 是由多个子 `Binding` 组成，`StringFormat` 仅在设置 `MultiBinding` 时适用，子 `Binding` 中虽然也可以设置 `StringFormat`，但是会被忽略。


### PriorityBinding


相比于前两种绑定，`PriorityBinding` 使用的频率没那么高，它的主要作用是按照一定优先级顺序设置绑定列表， 如果最高优先级绑定在处理时成功返回值，则无需处理列表中的其他绑定。 如果计算优先级最高的绑定需要很长时间，那么将会使用成功返回值的次高优先级，直到优先级较高的绑定成功返回值。`PriorityBinding` 和其包含的绑定列表中的子 `Binding` 也都可以设置 `StringFormat` 属性。例如：



```

    
        
            
            
            
        
    


```

与 `MultiBinding` 不同的是，`PriorityBinding` 的子 `Binding`中的 `StringFormat`是会生效的，其规则是优先使用子 `Binding` 设置的格式，其次才使用`PriorityBinding` 设置的格式。


## Content属性格式化失效的原因


`Button` 的 `Content` 属性可以用字符串赋值并显示在按钮上，但是使用 `StringFormat` 格式化并不会生效。原本我以为是涉及到类型转换器，在类型转换过程中处理掉了，但这只是猜测，通过源码发现并不是这样的。在 `BindingExpressionBase` 中有这样一段代码：



```
internal virtual bool AttachOverride(DependencyObject target, DependencyProperty dp)
{
	_targetElement = new WeakReference(target);
	_targetProperty = dp;
	DataBindEngine currentDataBindEngine = DataBindEngine.CurrentDataBindEngine;
	if (currentDataBindEngine == null || currentDataBindEngine.IsShutDown)
	{
		return false;
	}
	_engine = currentDataBindEngine;
	DetermineEffectiveStringFormat();
	DetermineEffectiveTargetNullValue();
	DetermineEffectiveUpdateBehavior();
	DetermineEffectiveValidatesOnNotifyDataErrors();
	if (dp == TextBox.TextProperty && IsReflective && !IsInBindingExpressionCollection && target is TextBoxBase textBoxBase)
	{
		textBoxBase.PreviewTextInput += OnPreviewTextInput;
	}
	if (TraceData.IsExtendedTraceEnabled(this, TraceDataLevel.Attach))
	{
		TraceData.TraceAndNotifyWithNoParameters(TraceEventType.Warning, TraceData.AttachExpression(TraceData.Identify(this), target.GetType().FullName, dp.Name, AvTrace.GetHashCodeHelper(target)), this);
	}
	return true;
}

```

其中第11行调用了一个名为 `DetermineEffectiveStringFormat` 的方法，顾名思义就是检测有效的 `StringFormat`。接下来看看里边的逻辑：



```
internal void DetermineEffectiveStringFormat()
{
	Type type = TargetProperty.PropertyType;
	if (type != typeof(string))
	{
		return;
	}
	string stringFormat = ParentBindingBase.StringFormat;
	for (BindingExpressionBase parentBindingExpressionBase = ParentBindingExpressionBase; parentBindingExpressionBase != null; parentBindingExpressionBase = parentBindingExpressionBase.ParentBindingExpressionBase)
	{
		if (parentBindingExpressionBase is MultiBindingExpression)
		{
			type = typeof(object);
			break;
		}
		if (stringFormat == null && parentBindingExpressionBase is PriorityBindingExpression)
		{
			stringFormat = parentBindingExpressionBase.ParentBindingBase.StringFormat;
		}
	}
	if (type == typeof(string) && !string.IsNullOrEmpty(stringFormat))
	{
		SetValue(Feature.EffectiveStringFormat, Helper.GetEffectiveStringFormat(stringFormat), null);
	}
}

```

这段代码的作用就是检测有效的 `StringFormat`，并通过 `SetValue` 方法保存起来，从第4\~7行代码可以看到，一开始就会检测目标属性的类型是不是 `String` 类型，不是的话直接返回，绑定表达式中的 `StringFormat` 也就不会保存了。在后续的 `BindingExpression` 类计算绑定表达式值时获取到 `StringFormat` 为 `null`，也就不会进行格式化了。
[![image](https://img2024.cnblogs.com/blog/3056716/202408/3056716-20240830130432496-639077746.png)](https://github.com):[飞数机场](https://ze16.com)


`Button` 的 `Content` 属性虽然可以用字符串赋值，但它其实的 `Object` 类型。因此，在检测有效的 `StringFormat` 表达式时直接过滤了。`ToolTip`也同样是 `Object` 类型。
[![image](https://img2024.cnblogs.com/blog/3056716/202408/3056716-20240830130446906-1084937100.png)](https://github.com)


## 解决方法


对于 `Content` 这种 `Object` 类型的属性绑定字符串并且需要格式化时，可以采用以下三种方式解决：


1. 最通用的方法就是自定义 `ValueConverter`，在 `ValueConverter` 中对字符串进行格式化；
2. 绑定到其他可进行 `StringFormat` 的属性上，比如 `TextBlock` 的 `Text` 属性进行格式化，`ToolTip` 绑定到 `Text` 上；
3. 既然是 `Object` 类型，那也可把 `TextBlock` 作为 `Content`的值。



```

    
        
    


```

## 小结


数据绑定时出现StringFormat失效的主要分为两种情况。一是没有遵循绑定时StringFormat使用的约束，二是绑定的目标属性不是 `String` 类型。


