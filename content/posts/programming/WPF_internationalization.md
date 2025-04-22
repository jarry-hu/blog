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

[ ] ScreenToGif 使用WPF Resource的方式来实现，个人感觉不太优雅，毕竟国际化不只有在View层会实现，在ViewModel也会涉及文字的转换，而Resource个人理解的View的东西；另一方面国际化不单单涉及文字，还有 **时间、货币、数字**
[x] PowerToys使用resw方式来实现，resw 相当于 resx 的进阶版，在UWP、MAUI中可以直接通过在xaml中x:UID="资源名"的方式直接可以从resw文件中找到字体资源。

- 我选择使用resx的方式来实现，还有个原因是resx有很多开源工具辅助编辑，像[ResXResourceManager](https://github.com/dotnet/ResXResourceManager)
（vs插件），它最神奇的是已经对接了很多AI工具; WPF中目前只支持resx的方式，但是我们可以自定义一个MarkupExtension，来实现类x:UID的实现;
``` C#
    ├── MainWindow.xaml
├── Resources/
│   ├── Strings.resx              // 默认语言（如英文）
│   ├── Strings.zh-Hans.resx      // 简体中文
│   └── Strings.fr.resx           // 法语
├── TranslationSource.cs
├── LocExtension.cs
└── LocalizationManager.cs

    
    public class TranslationSource : INotifyPropertyChanged
    {
        public static TranslationSource Instance { get; } = new TranslationSource();

        private readonly ResourceManager _resourceManager =
            new ResourceManager("MyWpfApp.Resources.Strings", Assembly.GetExecutingAssembly());

        public string this[string key] => _resourceManager.GetString(key, CultureInfo.CurrentUICulture);

        public event PropertyChangedEventHandler PropertyChanged;

        public void RaisePropertyChanged()
        {
            PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(string.Empty));
        }
    }

    public class LocExtension : MarkupExtension
    {
        public string Key { get; set; }

        public LocExtension(string key)
        {
            Key = key;
        }

        public override object ProvideValue(IServiceProvider serviceProvider)
        {
            var binding = new Binding($"[{Key}]")
            {
                Source = TranslationSource.Instance,
                Mode = BindingMode.OneWay
            };
            return binding.ProvideValue(serviceProvider);
        }
    }

    public static class LocalizationManager
    {
        public static void SetCulture(string culture)
        {
            var ci = new CultureInfo(culture);
            Thread.CurrentThread.CurrentCulture = ci;
            Thread.CurrentThread.CurrentUICulture = ci;

            TranslationSource.Instance.RaisePropertyChanged();
        }
    }
    // 这样我们在xaml 中就可以直接这样调用
    TextBlock Text="{local:Loc HelloWorld}" />
    // 对比UWP中的实现，代码量也差不多
    TextBlock x:UID="HelloWorld" />
```