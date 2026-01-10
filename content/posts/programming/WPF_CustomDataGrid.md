---
title: "WPF DataGrid自定义"
description: "DataGrid 添加序号与CheckBox"
date: 2025-04-22
tags: ["programming", "C#", "WPF"]
---

DataGrid 添加序号与CheckBox
```cs

    public static class DataGridHelper
    {
        // 1. 定义附加属性 IsAllSelected
        public static readonly DependencyProperty IsAllSelectedProperty =
            DependencyProperty.RegisterAttached(
                "IsAllSelected",
                typeof(bool?),
                typeof(DataGridHelper),
                new FrameworkPropertyMetadata(false, FrameworkPropertyMetadataOptions.BindsTwoWayByDefault, OnIsAllSelectedChanged));

        public static bool? GetIsAllSelected(DependencyObject element) => (bool?)element.GetValue(IsAllSelectedProperty);
        public static void SetIsAllSelected(DependencyObject element, bool? value) => element.SetValue(IsAllSelectedProperty, value);

        private static void OnIsAllSelectedChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
        {
            if (d is DataGrid dg && e.NewValue is bool all)
            {
                if (!dg.IsLoaded)
                {
                    RoutedEventHandler loadedHandler = null;
                    loadedHandler = (s, args) =>
                    {
                        dg.Loaded -= loadedHandler;
                        ApplySelection(dg, all);
                    };
                    dg.Loaded += loadedHandler;
                }
                else
                {
                    ApplySelection(dg, all);
                }
            }
        }

        private static void ApplySelection(DataGrid dg, bool all)
        {
            dg.Dispatcher.BeginInvoke(new Action(() =>
            {
                if (all)
                {
                    if (dg.SelectedItems.Count != dg.Items.Count) dg.SelectAll();
                }
                else
                {
                    if (dg.SelectedItems.Count > 0) dg.UnselectAll();
                }
            }));
        }

        // 2. 定义附加属性 EnableSelectionSync
        public static readonly DependencyProperty EnableSelectionSyncProperty =
            DependencyProperty.RegisterAttached(
                "EnableSelectionSync",
                typeof(bool),
                typeof(DataGridHelper),
                new PropertyMetadata(false, OnEnableSelectionSyncChanged));

        public static bool GetEnableSelectionSync(DependencyObject element) => (bool)element.GetValue(EnableSelectionSyncProperty);
        public static void SetEnableSelectionSync(DependencyObject element, bool value) => element.SetValue(EnableSelectionSyncProperty, value);

        // 3. 定义 SelectedCommand
        public static readonly DependencyProperty SelectedCommandProperty =
            DependencyProperty.RegisterAttached(
                "SelectedCommand",
                typeof(System.Windows.Input.ICommand),
                typeof(DataGridHelper),
                new PropertyMetadata(null));

        public static System.Windows.Input.ICommand GetSelectedCommand(DependencyObject element) => (System.Windows.Input.ICommand)element.GetValue(SelectedCommandProperty);
        public static void SetSelectedCommand(DependencyObject element, System.Windows.Input.ICommand value) => element.SetValue(SelectedCommandProperty, value);

        // 4. 定义 ShowRowNumber 附加属性
        public static readonly DependencyProperty ShowRowNumberProperty =
            DependencyProperty.RegisterAttached(
                "ShowRowNumber",
                typeof(bool),
                typeof(DataGridHelper),
                new PropertyMetadata(false));

        public static bool GetShowRowNumber(DependencyObject element) => (bool)element.GetValue(ShowRowNumberProperty);
        public static void SetShowRowNumber(DependencyObject element, bool value) => element.SetValue(ShowRowNumberProperty, value);

        private static void OnEnableSelectionSyncChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
        {
            if (d is DataGrid dg)
            {
                if ((bool)e.NewValue) dg.SelectionChanged += DataGrid_SelectionChanged;
                else dg.SelectionChanged -= DataGrid_SelectionChanged;
            }
        }

        private static void DataGrid_SelectionChanged(object sender, SelectionChangedEventArgs e)
        {
            if (sender is DataGrid dg)
            {
                // 1. 同步 IsAllSelected 状态
                bool? isAll = dg.SelectedItems.Count == dg.Items.Count;
                if (dg.SelectedItems.Count > 0 && dg.SelectedItems.Count < dg.Items.Count) isAll = null; 
                if (GetIsAllSelected(dg) != isAll)
                {
                    SetIsAllSelected(dg, isAll);
                }

                // 2. 执行 SelectedCommand
                var command = GetSelectedCommand(dg);
                if (command != null && command.CanExecute(dg.SelectedItems))
                {
                    command.Execute(dg.SelectedItems);
                }
            }
        }
        // 5. 定义 EnableAutoFiller 附加属性
        // 当设置为 True 时，会自动在 DataGrid 末尾添加一个 * 宽度的空白列，使其占满剩余空间
        public static readonly DependencyProperty EnableAutoFillerProperty =
            DependencyProperty.RegisterAttached(
                "EnableAutoFiller",
                typeof(bool),
                typeof(DataGridHelper),
                new PropertyMetadata(false, OnEnableAutoFillerChanged));

        public static bool GetEnableAutoFiller(DependencyObject element) => (bool)element.GetValue(EnableAutoFillerProperty);
        public static void SetEnableAutoFiller(DependencyObject element, bool value) => element.SetValue(EnableAutoFillerProperty, value);

        private static void OnEnableAutoFillerChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
        {
            if (d is DataGrid dg && (bool)e.NewValue)
            {
                dg.Loaded += DataGrid_Loaded_Filler;
            }
        }

        private static void DataGrid_Loaded_Filler(object sender, RoutedEventArgs e)
        {
            if (sender is DataGrid dg)
            {
                dg.Loaded -= DataGrid_Loaded_Filler;

                // 如果已经有 * 宽度的列，则不需要添加占位列
                if (dg.Columns.Any(c => c.Width.IsStar)) return;

                // 添加一个隐藏内容的占位列，宽度为 *，占满剩余所有空间
                var filler = new DataGridTemplateColumn
                {
                    Width = new DataGridLength(1, DataGridLengthUnitType.Star),
                    IsReadOnly = true,
                    Header = null,
                    CellStyle = new Style(typeof(DataGridCell))
                    {
                        Setters = {
                            new Setter(DataGridCell.BackgroundProperty, System.Windows.Media.Brushes.Transparent),
                            new Setter(DataGridCell.BorderThicknessProperty, new Thickness(0)),
                            new Setter(DataGridCell.FocusableProperty, false)
                        }
                    }
                };
                dg.Columns.Add(filler);
            }
        }
    }
```

```xaml

    <BooleanToVisibilityConverter x:Key="BooleanToVisibilityConverter" />
    <cvs:RowIndexConverter x:Key="RowIndexConverter" />
    <!--  重写左上角的 SelectAll 按钮样式  -->
    <Style x:Key="{ComponentResourceKey ResourceId=DataGridSelectAllButtonStyle, TypeInTargetAssembly={x:Type DataGrid}}" TargetType="Button">
        <Setter Property="Template">
            <Setter.Value>
                <ControlTemplate TargetType="Button">
                    <Grid>
                        <!--  背景和边框，保持和 Header 一致的风格  -->
                        <Border Background="Transparent" />

                        <StackPanel
                            HorizontalAlignment="Center"
                            VerticalAlignment="Center"
                            Orientation="Horizontal">
                            <!--  全选 CheckBox  -->
                            <CheckBox
                                Margin="0,0,8,0"
                                IsChecked="{Binding Path=(att:DataGridHelper.IsAllSelected), RelativeSource={RelativeSource AncestorType=DataGrid}, Mode=TwoWay}"
                                Visibility="{Binding Path=(att:DataGridHelper.EnableSelectionSync), RelativeSource={RelativeSource AncestorType=DataGrid}, Converter={StaticResource BooleanToVisibilityConverter}}" />

                            <!--  编号标题 #  -->
                            <TextBlock
                                x:Name="NumberHeader"
                                VerticalAlignment="Center"
                                FontWeight="Bold"
                                Text="#"
                                Visibility="{Binding Path=(att:DataGridHelper.ShowRowNumber), RelativeSource={RelativeSource AncestorType=DataGrid}, Converter={StaticResource BooleanToVisibilityConverter}}" />
                        </StackPanel>
                    </Grid>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
    </Style>

    <Style x:Key="MVIDataGridColumnHeaderStyle" TargetType="{x:Type DataGridColumnHeader}">
        <Setter Property="VerticalContentAlignment" Value="Center" />
        <Setter Property="HorizontalAlignment" Value="Left" />
        <Setter Property="HorizontalContentAlignment" Value="Left" />
        <Setter Property="FontWeight" Value="Bold" />
        <Setter Property="Foreground" Value="{DynamicResource PrimaryTextBrush}" />
        <Setter Property="Padding" Value="4,0" />
        <Setter Property="Template">
            <Setter.Value>
                <ControlTemplate TargetType="DataGridColumnHeader">
                    <hc:SimplePanel>
                        <Border
                            Padding="{TemplateBinding Padding}"
                            Background="Transparent"
                            BorderBrush="{TemplateBinding BorderBrush}"
                            BorderThickness="{TemplateBinding BorderThickness}">
                            <Grid HorizontalAlignment="{TemplateBinding HorizontalContentAlignment}" VerticalAlignment="{TemplateBinding VerticalContentAlignment}">
                                <Grid.ColumnDefinitions>
                                    <ColumnDefinition />
                                    <ColumnDefinition Width="Auto" />
                                </Grid.ColumnDefinitions>
                                <ContentPresenter
                                    VerticalAlignment="Center"
                                    RecognizesAccessKey="True"
                                    SnapsToDevicePixels="{TemplateBinding SnapsToDevicePixels}" />
                                <ToggleButton
                                    x:Name="ToggleButtonSortDirection"
                                    Grid.Column="1"
                                    Width="20"
                                    Height="{x:Static system:Double.NaN}"
                                    Padding="4,0"
                                    VerticalAlignment="Center"
                                    hc:IconSwitchElement.Geometry="{StaticResource DownGeometry}"
                                    hc:IconSwitchElement.GeometrySelected="{StaticResource UpGeometry}"
                                    Foreground="{DynamicResource PrimaryBrush}"
                                    IsEnabled="False"
                                    Opacity="1"
                                    Style="{StaticResource ToggleButtonIconTransparent}" />
                            </Grid>
                        </Border>
                        <Thumb
                            x:Name="PART_LeftHeaderGripper"
                            HorizontalAlignment="Left"
                            Style="{StaticResource ColumnHeaderGripperStyle}" />
                        <Thumb
                            x:Name="PART_RightHeaderGripper"
                            HorizontalAlignment="Right"
                            Style="{StaticResource ColumnHeaderGripperStyle}" />
                    </hc:SimplePanel>
                    <ControlTemplate.Triggers>
                        <Trigger Property="SortDirection" Value="{x:Null}">
                            <Setter TargetName="ToggleButtonSortDirection" Property="Visibility" Value="Collapsed" />
                        </Trigger>
                        <Trigger Property="SortDirection" Value="Ascending">
                            <Setter TargetName="ToggleButtonSortDirection" Property="IsChecked" Value="True" />
                        </Trigger>
                        <Trigger Property="SortDirection" Value="Descending">
                            <Setter TargetName="ToggleButtonSortDirection" Property="IsChecked" Value="False" />
                        </Trigger>
                    </ControlTemplate.Triggers>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
        <Style.Triggers>
            <Trigger Property="SortDirection" Value="Ascending">
                <Setter Property="Foreground" Value="{DynamicResource PrimaryBrush}" />
            </Trigger>
            <Trigger Property="SortDirection" Value="Descending">
                <Setter Property="Foreground" Value="{DynamicResource PrimaryBrush}" />
                <Setter Property="Foreground" Value="{DynamicResource PrimaryBrush}" />
            </Trigger>
        </Style.Triggers>
    </Style>

    <Style
        x:Key="MVIDataGridColumnHeaderCenterStyle"
        BasedOn="{StaticResource MVIDataGridColumnHeaderStyle}"
        TargetType="{x:Type DataGridColumnHeader}">
        <Setter Property="HorizontalContentAlignment" Value="Center" />
        <Setter Property="HorizontalAlignment" Value="Center" />
    </Style>

    <Style
        x:Key="MVIDataGridColumnHeaderRightStyle"
        BasedOn="{StaticResource MVIDataGridColumnHeaderStyle}"
        TargetType="{x:Type DataGridColumnHeader}">
        <Setter Property="HorizontalContentAlignment" Value="Right" />
        <Setter Property="HorizontalAlignment" Value="Right" />
    </Style>

    <Style x:Key="MVIDataGridRowHeaderStyle" TargetType="DataGridRowHeader">
        <Setter Property="Padding" Value="8,0" />
        <Setter Property="Background" Value="Transparent" />
        <Setter Property="Template">
            <Setter.Value>
                <ControlTemplate TargetType="DataGridRowHeader">
                    <Border
                        Padding="{TemplateBinding Padding}"
                        Background="{TemplateBinding Background}"
                        BorderBrush="{TemplateBinding BorderBrush}"
                        BorderThickness="{TemplateBinding BorderThickness}">
                        <hc:SimplePanel HorizontalAlignment="Center">
                            <StackPanel
                                HorizontalAlignment="Center"
                                VerticalAlignment="Center"
                                Orientation="Horizontal">
                                <!--  1. 默认内容（用于正常显示，比如序号）  -->
                                <ContentPresenter
                                    x:Name="DefaultContent"
                                    VerticalAlignment="Center"
                                    SnapsToDevicePixels="{TemplateBinding SnapsToDevicePixels}" />

                                <!--  2. 勾选框  -->
                                <CheckBox
                                    x:Name="SelectionCheckBox"
                                    Margin="0,0,8,0"
                                    VerticalAlignment="Center"
                                    IsChecked="{Binding IsSelected, RelativeSource={RelativeSource AncestorType=DataGridRow}, Mode=TwoWay, UpdateSourceTrigger=PropertyChanged}"
                                    Visibility="{Binding Path=(att:DataGridHelper.EnableSelectionSync), RelativeSource={RelativeSource AncestorType=DataGrid}, Converter={StaticResource BooleanToVisibilityConverter}}" />

                                <!--  3. 编号  -->
                                <TextBlock
                                    x:Name="RowNumber"
                                    VerticalAlignment="Center"
                                    Visibility="{Binding Path=(att:DataGridHelper.ShowRowNumber), RelativeSource={RelativeSource AncestorType=DataGrid}, Converter={StaticResource BooleanToVisibilityConverter}}">
                                    <TextBlock.Text>
                                        <MultiBinding Converter="{StaticResource RowIndexConverter}">
                                            <Binding />
                                            <Binding RelativeSource="{RelativeSource AncestorType=DataGrid}" />
                                        </MultiBinding>
                                    </TextBlock.Text>
                                </TextBlock>
                            </StackPanel>

                            <Thumb
                                x:Name="PART_TopHeaderGripper"
                                VerticalAlignment="Top"
                                Style="{StaticResource RowHeaderGripperStyle}" />
                            <Thumb
                                x:Name="PART_BottomHeaderGripper"
                                VerticalAlignment="Bottom"
                                Style="{StaticResource RowHeaderGripperStyle}" />
                        </hc:SimplePanel>
                    </Border>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
    </Style>

    <Style x:Key="MVIDataGridRowStyle" TargetType="DataGridRow">
        <Setter Property="BorderBrush" Value="{Binding BorderBrush, RelativeSource={RelativeSource AncestorType=DataGrid}}" />
        <Setter Property="BorderThickness" Value="0,0,0,1" />
        <Setter Property="SnapsToDevicePixels" Value="true" />
        <Setter Property="Validation.ErrorTemplate" Value="{x:Null}" />
        <Setter Property="ValidationErrorTemplate">
            <Setter.Value>
                <ControlTemplate>
                    <TextBlock
                        Margin="2,0,0,0"
                        VerticalAlignment="Center"
                        Foreground="Red"
                        Text="!" />
                </ControlTemplate>
            </Setter.Value>
        </Setter>
        <Setter Property="Template">
            <Setter.Value>
                <ControlTemplate TargetType="{x:Type DataGridRow}">
                    <Border
                        x:Name="DGR_Border"
                        Background="{TemplateBinding Background}"
                        BorderBrush="{TemplateBinding BorderBrush}"
                        BorderThickness="{TemplateBinding BorderThickness}"
                        SnapsToDevicePixels="True">
                        <SelectiveScrollingGrid>
                            <Grid.ColumnDefinitions>
                                <ColumnDefinition Width="Auto" />
                                <ColumnDefinition Width="*" />
                            </Grid.ColumnDefinitions>

                            <Grid.RowDefinitions>
                                <RowDefinition Height="*" />
                                <RowDefinition Height="Auto" />
                            </Grid.RowDefinitions>

                            <DataGridCellsPresenter
                                Grid.Column="1"
                                ItemsPanel="{TemplateBinding ItemsPanel}"
                                SnapsToDevicePixels="{TemplateBinding SnapsToDevicePixels}" />

                            <DataGridDetailsPresenter
                                Grid.Row="1"
                                Grid.Column="1"
                                SelectiveScrollingGrid.SelectiveScrollingOrientation="{Binding RelativeSource={RelativeSource AncestorType={x:Type DataGrid}}, Path=AreRowDetailsFrozen, Converter={x:Static DataGrid.RowDetailsScrollingConverter}, ConverterParameter={x:Static SelectiveScrollingOrientation.Vertical}}"
                                Visibility="{TemplateBinding DetailsVisibility}" />

                            <DataGridRowHeader
                                Grid.RowSpan="2"
                                SelectiveScrollingGrid.SelectiveScrollingOrientation="Vertical"
                                Visibility="{Binding RelativeSource={RelativeSource AncestorType={x:Type DataGrid}}, Path=HeadersVisibility, Converter={x:Static DataGrid.HeadersVisibilityConverter}, ConverterParameter={x:Static DataGridHeadersVisibility.Row}}" />
                        </SelectiveScrollingGrid>
                    </Border>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
        <Style.Triggers>
            <Trigger Property="IsNewItem" Value="True">
                <Setter Property="Margin" Value="{Binding RelativeSource={RelativeSource AncestorType={x:Type DataGrid}}, Path=NewItemMargin}" />
            </Trigger>

            <Trigger Property="IsSelected" Value="True">
                <Setter Property="Background" Value="{DynamicResource MVI.Brush.Table.Selected.Background}" />
            </Trigger>

            <MultiTrigger>
                <MultiTrigger.Conditions>
                    <Condition Property="IsMouseOver" Value="True" />
                    <Condition Property="IsSelected" Value="False" />
                </MultiTrigger.Conditions>
                <Setter Property="Background" Value="{DynamicResource MVI.Brush.Table.MouseOver.Background}" />
            </MultiTrigger>
        </Style.Triggers>
    </Style>

    <Style x:Key="MVIDataGridCellStyle" TargetType="DataGridCell">
        <Setter Property="FocusVisualStyle" Value="{x:Null}" />

        <Setter Property="BorderThickness" Value="0" />

        <Setter Property="Template">
            <Setter.Value>
                <ControlTemplate TargetType="DataGridCell">
                    <Border
                        Background="{TemplateBinding Background}"
                        BorderBrush="{TemplateBinding BorderBrush}"
                        BorderThickness="{TemplateBinding BorderThickness}"
                        SnapsToDevicePixels="True">
                        <ContentPresenter VerticalAlignment="Center" SnapsToDevicePixels="{TemplateBinding SnapsToDevicePixels}" />
                    </Border>
                </ControlTemplate>
            </Setter.Value>
        </Setter>

        <Style.Triggers>
            <Trigger Property="IsSelected" Value="True">
                <Setter Property="BorderBrush" Value="Transparent" />
                <Setter Property="BorderThickness" Value="0" />
                <Setter Property="Background" Value="Transparent" />
                <Setter Property="Foreground" Value="{DynamicResource MVI.Brush.Black}" />

            </Trigger>
        </Style.Triggers>
    </Style>

    <Style x:Key="MVIDataGridStyle" TargetType="DataGrid">
        <Setter Property="HeadersVisibility" Value="Column" />
        <Setter Property="AlternationCount" Value="2" />
        <Setter Property="BorderBrush" Value="{DynamicResource MVI.Brush.Header.Border}" />
        <Setter Property="HorizontalGridLinesBrush" Value="{DynamicResource MVI.Brush.Table.GridLine}" />
        <Setter Property="VerticalContentAlignment" Value="Center" />
        <Setter Property="GridLinesVisibility" Value="None" />
        <Setter Property="ColumnHeaderStyle" Value="{StaticResource MVIDataGridColumnHeaderStyle}" />
        <Setter Property="RowHeaderStyle" Value="{StaticResource MVIDataGridRowHeaderStyle}" />
        <Setter Property="CellStyle" Value="{StaticResource MVIDataGridCellStyle}" />
        <Setter Property="RowStyle" Value="{StaticResource MVIDataGridRowStyle}" />
        <Setter Property="RowBackground" Value="{DynamicResource MVI.Brush.Table.Background}" />
        <Setter Property="MinRowHeight" Value="32" />
        <Setter Property="AlternatingRowBackground" Value="{DynamicResource MVI.Brush.Table.AlternatingBackground}" />
        <Setter Property="BorderThickness" Value="1" />
        <Setter Property="hc:BorderElement.CornerRadius" Value="8" />
        <Setter Property="Template">
            <Setter.Value>
                <ControlTemplate TargetType="{x:Type DataGrid}">
                    <Border
                        Padding="{TemplateBinding Padding}"
                        Background="{TemplateBinding Background}"
                        BorderBrush="{TemplateBinding BorderBrush}"
                        BorderThickness="{TemplateBinding BorderThickness}"
                        CornerRadius="{TemplateBinding hc:BorderElement.CornerRadius}"
                        SnapsToDevicePixels="True">
                        <ScrollViewer Name="DG_ScrollViewer" Focusable="false">
                            <ScrollViewer.Template>
                                <ControlTemplate TargetType="{x:Type ScrollViewer}">
                                    <Grid>
                                        <Grid.RowDefinitions>
                                            <RowDefinition Height="40" />
                                            <RowDefinition Height="*" />
                                            <RowDefinition Height="Auto" />
                                        </Grid.RowDefinitions>

                                        <Grid.ColumnDefinitions>
                                            <ColumnDefinition Width="Auto" />
                                            <ColumnDefinition Width="*" />
                                            <ColumnDefinition Width="Auto" />
                                        </Grid.ColumnDefinitions>

                                        <!--  Left Column Header Corner  -->
                                        <Button
                                            Width="{Binding RelativeSource={RelativeSource AncestorType={x:Type DataGrid}}, Path=CellsPanelHorizontalOffset}"
                                            Command="{x:Static DataGrid.SelectAllCommand}"
                                            Focusable="false"
                                            Style="{DynamicResource {ComponentResourceKey TypeInTargetAssembly={x:Type DataGrid},
                                                                                          ResourceId=DataGridSelectAllButtonStyle}}"
                                            Visibility="{Binding RelativeSource={RelativeSource AncestorType={x:Type DataGrid}}, Path=HeadersVisibility, Converter={x:Static DataGrid.HeadersVisibilityConverter}, ConverterParameter={x:Static DataGridHeadersVisibility.All}}" />
                                        <!--  Column Headers  -->
                                        <DataGridColumnHeadersPresenter
                                            Name="PART_ColumnHeadersPresenter"
                                            Grid.Column="1"
                                            Visibility="{Binding RelativeSource={RelativeSource AncestorType={x:Type DataGrid}}, Path=HeadersVisibility, Converter={x:Static DataGrid.HeadersVisibilityConverter}, ConverterParameter={x:Static DataGridHeadersVisibility.Column}}" />

                                        <!--  DataGrid content  -->
                                        <ScrollContentPresenter
                                            x:Name="PART_ScrollContentPresenter"
                                            Grid.Row="1"
                                            Grid.ColumnSpan="2"
                                            CanContentScroll="{TemplateBinding CanContentScroll}" />

                                        <ScrollBar
                                            Name="PART_VerticalScrollBar"
                                            Grid.Row="1"
                                            Grid.Column="2"
                                            Maximum="{TemplateBinding ScrollableHeight}"
                                            Orientation="Vertical"
                                            ViewportSize="{TemplateBinding ViewportHeight}"
                                            Visibility="{TemplateBinding ComputedVerticalScrollBarVisibility}"
                                            Value="{Binding Path=VerticalOffset, RelativeSource={RelativeSource TemplatedParent}, Mode=OneWay}" />

                                        <Grid Grid.Row="2" Grid.Column="1">
                                            <Grid.ColumnDefinitions>
                                                <ColumnDefinition Width="{Binding RelativeSource={RelativeSource AncestorType={x:Type DataGrid}}, Path=NonFrozenColumnsViewportHorizontalOffset}" />
                                                <ColumnDefinition Width="*" />
                                            </Grid.ColumnDefinitions>
                                            <ScrollBar
                                                Name="PART_HorizontalScrollBar"
                                                Grid.Column="1"
                                                Maximum="{TemplateBinding ScrollableWidth}"
                                                Orientation="Horizontal"
                                                ViewportSize="{TemplateBinding ViewportWidth}"
                                                Visibility="{TemplateBinding ComputedHorizontalScrollBarVisibility}"
                                                Value="{Binding Path=HorizontalOffset, RelativeSource={RelativeSource TemplatedParent}, Mode=OneWay}" />

                                        </Grid>
                                    </Grid>
                                </ControlTemplate>
                            </ScrollViewer.Template>
                            <ItemsPresenter SnapsToDevicePixels="{TemplateBinding SnapsToDevicePixels}" />
                        </ScrollViewer>
                    </Border>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
        <Style.Triggers>
            <MultiTrigger>
                <MultiTrigger.Conditions>
                    <Condition Property="IsGrouping" Value="true" />
                    <Condition Property="VirtualizingPanel.IsVirtualizingWhenGrouping" Value="false" />
                </MultiTrigger.Conditions>
                <Setter Property="ScrollViewer.CanContentScroll" Value="false" />
            </MultiTrigger>
        </Style.Triggers>
    </Style>

    <Style BasedOn="{StaticResource MVIDataGridStyle}" TargetType="DataGrid">
        <Setter Property="AutoGenerateColumns" Value="False" />
        <Setter Property="DataGridHelper.EnableAutoFiller" Value="True" />
        <Setter Property="DataGridHelper.EnableSelectionSync" Value="False" />
        <Setter Property="DataGridHelper.ShowRowNumber" Value="False" />

        <Setter Property="IsReadOnly" Value="True" />
        <Style.Triggers>
            <Trigger Property="DataGridHelper.EnableSelectionSync" Value="True">
                <Setter Property="HeadersVisibility" Value="All" />
            </Trigger>
            <Trigger Property="DataGridHelper.ShowRowNumber" Value="True">
                <Setter Property="HeadersVisibility" Value="All" />
            </Trigger>
        </Style.Triggers>
    </Style>
```

``` cs
    public class RowIndexConverter : IMultiValueConverter
    {
        public object Convert(object[] values, Type targetType, object parameter, CultureInfo culture)
        {
            if (values == null || values.Length < 2) return "";

            var item = values[0];
            var dg = values[1] as DataGrid;

            // DependencyProperty.UnsetValue is important in WPF Bindings to avoid errors during initial pass
            if (item != null && item != DependencyProperty.UnsetValue && dg != null)
            {
                try
                {
                    int index = dg.Items.IndexOf(item);
                    if (index >= 0)
                    {
                        return (index + 1).ToString();
                    }
                }
                catch
                {
                    // Ignore index errors during transitional states
                }
            }
            return "";
        }

        public object[] ConvertBack(object value, Type[] targetTypes, object parameter, CultureInfo culture)
        {
            return null;
        }
    }
```