---
title: "Slate FArguments, SLATE_ATTRIBUTE, 以及 TAttribute"
date: 2022-10-25T22:50:08+08:00
draft: true
toc: true
categories: [Slate]
tags: 
  - Slate
  - FArguments
  - SLATE_ATTRIBUTE
  - TAttribute
---

*UE5版本:5.03*
---

## 举个例子
```cpp
class SMyWidget : public SCompoundWidget
{
    SLATE_BEGIN_ARGS( SMyWidget )
         , _PreferredWidth( 150.0f )
         , _ForegroundColor( FLinearColor::White )
         {}
        SLATE_ATTRIBUTE(float, PreferredWidth)
        SLATE_ATTRIBUTE(FSlateColor, ForegroundColor)
    SLATE_END_ARGS()

	// ... ... 
}
```

## SLATE_BEGIN/END_ARGS
SLATE_BEGIN_ARGS() 和 SLATE_END_ARGS() 的意义在引擎源码中已经写了：让用户可以通过SNew和SAssignNew来构建Slate。但这只是个概述，除此之外的很多细节，我们本文中一一探究。

因为SLATE_END_ARGS()宏仅仅只是一个结束大括号和分号，所以我们将SLATE_BEGIN_ARGS宏进和它一起展开，得到：

```cpp
class SMyWidget : public SCompoundWidget
{
public: 
	struct FArguments : public TSlateBaseNamedArgs<SMyWidget> 
	{ 
		typedef FArguments WidgetArgsType; 

		// FArguments的构造函数
		FORCENOINLINE FArguments()
			, _PreferredWidth( 150.0f )
			, _ForegroundColor( FLinearColor::White )
         	{}
		//SLATE_ATTRIBUTE的部分，暂时先不展开
		SLATE_ATTRIBUTE(float, PreferredWidth)
        SLATE_ATTRIBUTE(FSlateColor, ForegroundColor)
	};
}
```
SLATE_BEGIN_ARGS声明了一个FArguments结构体，并定义了构造函数。

FArguments派生自TSlateBaseNamedArgs。

TSlateBaseNamedArgs 及其父类 FSlateBaseNamedArgs 中包含一组由 *SLATE_PRIVATE_ATTRIBUTE_VARIABLE* 和 *SLATE_PRIVATE_ATTRIBUTE_FUNCTION* 定义的TAttribute以及函数，比如ToolTip, Cursor, Visibility, IsEnabled等，这是每个Slate都具有的共有属性。

## SLATE_ATTRIBUTE宏
SLATE_ATTRIBUTE宏是Slate中用来给一个Slate界面添加属性的，这个属性可以在构建这个Slate的时候传入。

SLAET_ATTRIBUTE宏只能出现在 SLATE_BEGIN_ARGS宏和SLATE_END_ARGS宏之间。

它的语法是: SLATE_ATTRIBUTE(数据类型，属性名称)

它会生成这样一个变量： TAttribute<数据类型> _属性名称 ， 注意属性名称前面添加了一个下划线。

另外它还包含一些可以给自身绑定 *Getter* 方法的函数。



## 展开SLATE_ATTRIBUTE宏
任意带入两个参数，对SLATE_ATTRIBUTE进行宏展开，例如这里带入 FString 和 MyVar。

### 第一次展开
第一次展开SLATE_ATTRIBUTE宏可以得到：
```cpp
#define SLATE_ATTRIBUTE( FString, MyVar ) 
		SLATE_PRIVATE_ATTRIBUTE_VARIABLE( FString, MyVar ); 
		SLATE_PRIVATE_ATTRIBUTE_FUNCTION( FString, MyVar )
```

可以看到分两部分，第一部分是变量，第二部分是函数。

这里用到的两个宏和前面TSlateBaseNamedArgs中使用的是一样的。

### 展开变量部分
```cpp
//#define SLATE_PRIVATE_ATTRIBUTE_VARIABLE( FString, MyVar )
		TAttribute<FString> _MyVar;
```
变量部分比较简单，正如前文所述，生成了一个 TAttribute，并且变量名称前面增加了一个下划线。后文再分析TAttribute。

### 展开函数部分
```cpp
#define SLATE_PRIVATE_ATTRIBUTE_FUNCTION( FString, MyVar ) 
		WidgetArgsType& MyVar( TAttribute< FString > InAttribute ) 
		{ 
			MyVar = MoveTemp(InAttribute); 
			return static_cast<WidgetArgsType*>(this)->Me(); 
		} 
		
		/* Bind attribute with delegate to a global function
		 * NOTE: We use a template here to avoid 'typename' issues when hosting attributes inside templated classes */ 
		template< typename... VarTypes >
		WidgetArgsType& MyVar_Static( typename TAttribute< FString >::FGetter::template FStaticDelegate<VarTypes...>::FFuncPtr InFunc, VarTypes... Vars )
		{ 
			_MyVar = TAttribute< FString >::Create( TAttribute< FString >::FGetter::CreateStatic( InFunc, Vars... ) ); 
			return static_cast<WidgetArgsType*>(this)->Me(); 
		} 

		/* Bind attribute with delegate to a lambda
		 * technically this works for any functor types, but lambdas are the primary use case */ 
		WidgetArgsType& MyVar_Lambda(TFunction< FString(void) >&& InFunctor) 
		{ 
			_MyVar = TAttribute< FString >::Create(Forward<TFunction< FString(void) >>(InFunctor)); 
			return static_cast<WidgetArgsType*>(this)->Me(); 
		} 
		
		/* Bind attribute with delegate to a raw C++ class method */ 
		template< class UserClass, typename... VarTypes >	
		WidgetArgsType& MyVar_Raw( UserClass* InUserObject, typename TAttribute< FString >::FGetter::template TConstMethodPtr< UserClass, VarTypes... > InFunc, VarTypes... Vars )	
		{ 
			_MyVar = TAttribute< FString >::Create( TAttribute< FString >::FGetter::CreateRaw( InUserObject, InFunc, Vars... ) ); 
			return static_cast<WidgetArgsType*>(this)->Me(); 
		} 
		
		/* Bind attribute with delegate to a shared pointer-based class method.  Slate mostly uses shared pointers so we use an overload for this type of binding. */ 
		template< class UserClass, typename... VarTypes >	
		WidgetArgsType& MyVar( TSharedRef< UserClass > InUserObjectRef, typename TAttribute< FString >::FGetter::template TConstMethodPtr< UserClass, VarTypes... > InFunc, VarTypes... Vars )	
		{ 
			_MyVar = TAttribute< FString >::Create( TAttribute< FString >::FGetter::CreateSP( InUserObjectRef, InFunc, Vars... ) ); 
			return static_cast<WidgetArgsType*>(this)->Me(); 
		} 
		
		/* Bind attribute with delegate to a shared pointer-based class method.  Slate mostly uses shared pointers so we use an overload for this type of binding. */ 
		template< class UserClass, typename... VarTypes >	
		WidgetArgsType& MyVar( UserClass* InUserObject, typename TAttribute< FString >::FGetter::template TConstMethodPtr< UserClass, VarTypes... > InFunc, VarTypes... Vars )	
		{ 
			_MyVar = TAttribute< FString >::Create( TAttribute< FString >::FGetter::CreateSP( InUserObject, InFunc, Vars... ) ); 
			return static_cast<WidgetArgsType*>(this)->Me(); 
		} 

		/* Bind attribute with delegate to a UObject-based class method */ 
		template< class UserClass, typename... VarTypes >	
		WidgetArgsType& MyVar_UObject( UserClass* InUserObject, typename TAttribute< FString >::FGetter::template TConstMethodPtr< UserClass, VarTypes... > InFunc, VarTypes... Vars )	
		{ 
			_MyVar = TAttribute< FString >::Create( TAttribute< FString >::FGetter::CreateUObject( InUserObject, InFunc, Vars... ) ); 
			return static_cast<WidgetArgsType*>(this)->Me(); 
		} 
```

这里是一组函数，这组函数可以接收可调用对象(Lambda, 原生C++类方法，基于共享指针的类方法，UObject的类方法)，将其传给TAttribute的相应的类静态函数Create(其实就是把这个函数设为Getter), 生成TAttribute, 并赋值给_MyVar。

## TAttribute
查看TAttribute源码(Attribute.h)可以发现，它是一个模板类，接受一个ObjectType模板参数。

### 构造方法
无参构造函数构造一个空的对象， 没有任何东西被初始化。

可以直接用存储的数据类型的一个数据构造，初始化了数值，但是没有绑定Getter。

也可以传入函数，初始化Getter函数，但是数值并未初始化。

### 核心数据成员
它最核心的数据成员就是：
```cpp
mutable ObjectType Value;
```

### 静态方法
该类有一些静态函数Create，他们接受各种类型的可调用对象(Lambda, 原生C++类方法，基于共享指针的类方法，UObject的类方法) 作为 *Getter* 函数，并返回一个TAttribute。有些类似于工厂函数。

### 成员方法

Set() 和 Get() 方法用于设置和获取数值

IsSet() 用于返回数值是否被设置

Bind(), BindStatic(), BindRaw(), BindUObject(), BindUFunction() 等用于绑定 *Getter* 函数

> P.S. 注意Bind函数族只是绑定 Getter，并没有绑定 Setter，要设置值只能手动调用Set()








