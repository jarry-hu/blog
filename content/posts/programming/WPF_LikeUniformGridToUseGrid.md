
title: "WPF 像UnifromGrid 一样去使用 Grid"
description: "WPF 像UnifromGrid 一样去使用 Grid，不用去设置每个控件的Row 与Column"
date: 2025-04-22
tags: ["programming", "C#", "WPF"]
---
```C#
    public class GridHelper
    {
        #region 1. Rows (行定义)
        public static readonly DependencyProperty RowsProperty =
            DependencyProperty.RegisterAttached("Rows", typeof(string), typeof(GridHelper),
                new PropertyMetadata(string.Empty, OnLayoutChanged));

        public static string GetRows(DependencyObject obj) => (string)obj.GetValue(RowsProperty);
        public static void SetRows(DependencyObject obj, string value) => obj.SetValue(RowsProperty, value);
        #endregion

        #region 2. Columns (列定义)
        public static readonly DependencyProperty ColumnsProperty =
            DependencyProperty.RegisterAttached("Columns", typeof(string), typeof(GridHelper),
                new PropertyMetadata(string.Empty, OnLayoutChanged));

        public static string GetColumns(DependencyObject obj) => (string)obj.GetValue(ColumnsProperty);
        public static void SetColumns(DependencyObject obj, string value) => obj.SetValue(ColumnsProperty, value);
        #endregion

        #region 3. AutoLayout (开关：是否自动填充)
        public static readonly DependencyProperty AutoLayoutProperty =
            DependencyProperty.RegisterAttached("AutoLayout", typeof(bool), typeof(GridHelper),
                new PropertyMetadata(false, OnLayoutChanged));

        public static bool GetAutoLayout(DependencyObject obj) => (bool)obj.GetValue(AutoLayoutProperty);
        public static void SetAutoLayout(DependencyObject obj, bool value) => obj.SetValue(AutoLayoutProperty, value);
        #endregion

        // 核心变更处理
        private static void OnLayoutChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
        {
            if (d is Grid grid)
            {
                // 确保 Grid 加载完成后再计算（因为要等 Children 集合填充完毕）
                if (grid.IsLoaded)
                {
                    RefreshGrid(grid);
                }
                else
                {
                    // 避免重复订阅
                    grid.Loaded -= Grid_Loaded;
                    grid.Loaded += Grid_Loaded;
                }
            }
        }

        private static void Grid_Loaded(object sender, RoutedEventArgs e)
        {
            var grid = sender as Grid;
            grid.Loaded -= Grid_Loaded; // 用完即弃
            RefreshGrid(grid);
        }

        // ★★★ 核心逻辑：解析定义 + 自动分配位置 ★★★
        private static void RefreshGrid(Grid grid)
        {
            // 1. 解析 Rows
            var rowsStr = GetRows(grid);
            if (!string.IsNullOrEmpty(rowsStr))
            {
                grid.RowDefinitions.Clear();
                foreach (var def in ParseDefinitions(rowsStr))
                    grid.RowDefinitions.Add(new RowDefinition { Height = def });
            }

            // 2. 解析 Columns
            var colsStr = GetColumns(grid);
            if (!string.IsNullOrEmpty(colsStr))
            {
                grid.ColumnDefinitions.Clear();
                foreach (var def in ParseDefinitions(colsStr))
                    grid.ColumnDefinitions.Add(new ColumnDefinition { Width = def });
            }

            // 3. 自动矩阵布局 (Matrix Layout)
            bool isAutoLayout = GetAutoLayout(grid);
            if (isAutoLayout)
            {
                var colCount = grid.ColumnDefinitions.Count;
                // 防止除以0错误
                if (colCount == 0) colCount = 1;

                int childIndex = 0;
                foreach (UIElement child in grid.Children)
                {
                    // 算法：
                    // 行号 = 索引 / 列总数
                    // 列号 = 索引 % 列总数
                    int row = childIndex / colCount;
                    int col = childIndex % colCount;

                    // 如果行不够了，自动补齐行定义（可选功能，防止越界）
                    if (row >= grid.RowDefinitions.Count)
                    {
                        grid.RowDefinitions.Add(new RowDefinition { Height = GridLength.Auto });
                    }

                    Grid.SetRow(child, row);
                    Grid.SetColumn(child, col);

                    childIndex++;
                }
            }
        }

        // 解析器（Auto, *, 100）
        private static GridLength[] ParseDefinitions(string definitions)
        {
            var parts = definitions.Split(',');
            var lengths = new GridLength[parts.Length];

            for (int i = 0; i < parts.Length; i++)
            {
                var part = parts[i].Trim().ToLower();
                if (part == "auto")
                    lengths[i] = GridLength.Auto;
                else if (part.EndsWith("*"))
                {
                    double val = 1;
                    if (part.Length > 1) double.TryParse(part.Replace("*", ""), out val);
                    lengths[i] = new GridLength(val, GridUnitType.Star);
                }
                else
                {
                    if (double.TryParse(part, out double val)) lengths[i] = new GridLength(val);
                    else lengths[i] = GridLength.Auto;
                }
            }
            return lengths;
        }
    }
```