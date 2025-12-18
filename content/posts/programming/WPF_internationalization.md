---
title: "WPF 国际化"
description: "C# 的国际化还是老老实实用resx"
date: 2025-04-22
tags: ["programming", "C#", "WPF"]
---

C#国际化官方推荐使用resx  

参考了了WPF的两个开源项目:
1. [ScreenToGif](https://github.com/NickeManarin/ScreenToGif) 
2. [PowerToys](https://github.com/microsoft/PowerToys)

* [ ] ScreenToGif 使用WPF Resource的方式来实现，个人感觉不太优雅，毕竟国际化不只有在View层会实现，在ViewModel也会涉及文字的转换，而Resource个人理解的View的东西；另一方面国际化不单单涉及文字，还有 **时间、货币、数字**
* [x] PowerToys使用resw方式来实现，resw 相当于 resx 的进阶版，在UWP、MAUI中可以直接通过在xaml中x:UID="资源名"的方式直接可以从resw文件中找到字体资源。

- 我选择使用resx的方式来实现，还有个原因是resx有很多开源工具辅助编辑，像[ResXResourceManager](https://github.com/dotnet/ResXResourceManager)
（vs插件），它最神奇的是已经对接了很多AI工具; WPF中目前只支持resx的方式，但是我们可以自定义一个MarkupExtension，来实现类x:UID的实现;
``` C#
    ├── MainWindow.xaml
├── Resources/                    // 手动创建Strings.resx 其他resx文件由ResXResourceManager生成
│   ├── Strings.resx              // 默认语言（如英文）
│   ├── Strings.zh-Hans.resx      // 简体中文
│   └── Strings.fr.resx           // 法语
├── TranslationSource.cs
├── LocExtension.cs
└── LocalizationManager.cs

    

    /// <summary>
    /// 语言管理
    /// </summary>
    public class TranslationManager : INotifyPropertyChanged
    {
        // 单例模式
        public static TranslationManager Instance { get; } = new TranslationManager();

        private ResourceManager? _resourceManager = new ResourceManager("MVI3000.Client.Core.Resources.Strings", Assembly.GetExecutingAssembly());
        private CultureInfo? _currentCulture;

        public CultureInfo? CurrentCulture
        {
            get => _currentCulture;
            set
            {
                if (_currentCulture != value)
                {
                    _currentCulture = value;
                    // 切换语言时，通知所有监听者
                    OnPropertyChanged(nameof(CurrentCulture));
                }
            }
        }

        // 设置你的资源文件 (Resx)
        public ResourceManager? ResourceManager
        {
            get => _resourceManager;
            set => _resourceManager = value;
        }

        public string GetString(string key)
        {
            if (string.IsNullOrEmpty(key) || _resourceManager == null) return key;
            return _resourceManager.GetString(key, _currentCulture) ?? key;
        }

        public event PropertyChangedEventHandler? PropertyChanged;
        protected void OnPropertyChanged(string name) =>
            PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(name));
    }

    public class TranslateConverter : IMultiValueConverter
    {
        public object Convert(object[] values, Type targetType, object parameter, CultureInfo culture)
        {
            // values[0]: 实际的 Key (字符串)
            // values[1]: 语言管理器实例 (我们不使用它，只是为了监听它的变动)

            if (values.Length > 0 && values[0] is string key)
            {
                return TranslationManager.Instance.GetString(key);
            }
            return string.Empty;
        }

        public object[] ConvertBack(object value, Type[] targetTypes, object parameter, CultureInfo culture)
        {
            throw new NotImplementedException();
        }
    }

    /// <summary>
    /// 语言本地化扩展
    /// </summary>
    [ContentProperty(nameof(Key))]
    public class LocExtension : MarkupExtension
    {
        public object Key { get; set; } = string.Empty;
        public string? StringFormat { get; set; }

        public LocExtension()
        {

        }

        public LocExtension(object key)
        {
            Key = key;
        }

        public override object ProvideValue(IServiceProvider serviceProvider)
        {
            // 1. 基础判空
            if (serviceProvider == null) return this;
            var pvt = serviceProvider.GetService(typeof(IProvideValueTarget)) as IProvideValueTarget;
            if (pvt == null || pvt.TargetObject == null) return this;

            // 2. ★★★ 增强版设计模式检查 ★★★
            if (DesignerProperties.GetIsInDesignMode(new DependencyObject()))
            {
                // 如果目标是 DependencyProperty
                if (pvt.TargetProperty is DependencyProperty dp)
                {
                    // 如果目标是 String 或者 Object (比如 Content)，都可以安全地返回字符串
                    if (dp.PropertyType == typeof(string) || dp.PropertyType == typeof(object))
                    {
                        return $"[{Key}]";
                    }

                    // 如果是其他类型 (比如 Visibility, Brush)，
                    // 千万别返回字符串，否则设计器报“类型转换错误”
                    // 直接返回 UnsetValue，让它用默认值，别报错就行
                    return DependencyProperty.UnsetValue;
                }

                // 如果是在 Setter 或 Template 里，直接返回 this 混过去
                return this;
            }

            // 3. 运行时逻辑
            try
            {
                // ★★★ 核心修复：检查单例是否存活 ★★★
                // 在设计器某些极端情况下(或者单元测试中)，即使 IsInDesignMode 没拦截住，
                // Instance 也可能是 null。如果这里不防守，下面 Source=null 就会炸。
                if (TranslationManager.Instance == null)
                {
                    return Key?.ToString() ?? "";
                }

                var multiBinding = new MultiBinding
                {
                    Converter = new TranslateConverter(),
                    Mode = BindingMode.OneWay,
                    StringFormat = StringFormat,
                };

                if (Key is BindingBase bindingKey)
                {
                    multiBinding.Bindings.Add(bindingKey);
                }
                else
                {
                    multiBinding.Bindings.Add(new Binding { Source = Key });
                }

                var langBinding = new Binding(nameof(TranslationManager.CurrentCulture))
                {
                    Source = TranslationManager.Instance,
                };
                multiBinding.Bindings.Add(langBinding);

                return multiBinding.ProvideValue(serviceProvider);
            }
            catch
            {
                return DependencyProperty.UnsetValue;
            }
        }
    }
    // 这样我们在xaml 中就可以直接这样调用
    <TextBlock Text="{local:Loc HelloWorld, StringFormat={}{0}:}" />
```