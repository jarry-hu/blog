---
title: "WPF 像UnifromGrid 一样去使用 Grid"
description: "WPF 像UnifromGrid 一样去使用 Grid，不用去设置每个控件的Row 与Column"
date: 2025-12-18
tags: ["programming", "C#", "WPF"]
---
```C#
    
    public class GridHelper
    {
        #region 1. Rows (行定义字符串)
        public static readonly DependencyProperty RowsProperty =
            DependencyProperty.RegisterAttached("Rows", typeof(string), typeof(GridHelper),
                new PropertyMetadata(string.Empty, OnLayoutChanged));

        public static string GetRows(DependencyObject obj) => (string)obj.GetValue(RowsProperty);
        public static void SetRows(DependencyObject obj, string value) => obj.SetValue(RowsProperty, value);
        #endregion

        #region 2. Columns (列定义字符串)
        public static readonly DependencyProperty ColumnsProperty =
            DependencyProperty.RegisterAttached("Columns", typeof(string), typeof(GridHelper),
                new PropertyMetadata(string.Empty, OnLayoutChanged));

        public static string GetColumns(DependencyObject obj) => (string)obj.GetValue(ColumnsProperty);
        public static void SetColumns(DependencyObject obj, string value) => obj.SetValue(ColumnsProperty, value);
        #endregion

        #region 3. AutoLayout (自动布局开关)
        public static readonly DependencyProperty AutoLayoutProperty =
            DependencyProperty.RegisterAttached("AutoLayout", typeof(bool), typeof(GridHelper),
                new PropertyMetadata(false, OnLayoutChanged));

        public static bool GetAutoLayout(DependencyObject obj) => (bool)obj.GetValue(AutoLayoutProperty);
        public static void SetAutoLayout(DependencyObject obj, bool value) => obj.SetValue(AutoLayoutProperty, value);
        #endregion

        #region 4. DefaultRowHeight (默认自动生成行的行高)
        public static readonly DependencyProperty DefaultRowHeightProperty =
            DependencyProperty.RegisterAttached("DefaultRowHeight", typeof(string), typeof(GridHelper),
                new PropertyMetadata("Auto", OnLayoutChanged));

        public static string GetDefaultRowHeight(DependencyObject obj) => (string)obj.GetValue(DefaultRowHeightProperty);
        public static void SetDefaultRowHeight(DependencyObject obj, string value) => obj.SetValue(DefaultRowHeightProperty, value);
        #endregion

        // 统一的属性变更回调
        private static void OnLayoutChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
        {
            if (d is Grid grid)
            {
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
            grid.Loaded -= Grid_Loaded;
            RefreshGrid(grid);
        }

        // ★★★ 核心布局逻辑 ★★★
        private static void RefreshGrid(Grid grid)
        {
            // 1. 解析 Columns
            var colsStr = GetColumns(grid);
            if (!string.IsNullOrEmpty(colsStr))
            {
                grid.ColumnDefinitions.Clear();
                foreach (var def in ParseDefinitions(colsStr))
                    grid.ColumnDefinitions.Add(new ColumnDefinition { Width = def });
            }

            // 2. 解析 Rows
            var rowsStr = GetRows(grid);
            grid.RowDefinitions.Clear();
            if (!string.IsNullOrEmpty(rowsStr))
            {
                foreach (var def in ParseDefinitions(rowsStr))
                    grid.RowDefinitions.Add(new RowDefinition { Height = def });
            }

            // 3. 自动布局 (支持 Span 的贪吃蛇算法)
            if (GetAutoLayout(grid))
            {
                int colCount = grid.ColumnDefinitions.Count;
                if (colCount == 0) colCount = 1;

                // 默认行高解析
                var defaultHeightStr = GetDefaultRowHeight(grid);
                var defaultHeightLength = ParseSingleLength(defaultHeightStr);

                // ★★★ 核心变化：使用 HashSet 记录已占用的坐标 (Row, Col) ★★★
                var occupiedCells = new HashSet<(int, int)>();

                // 搜索游标（从 0,0 开始找空位）
                int searchRow = 0;
                int searchCol = 0;

                foreach (UIElement child in grid.Children)
                {
                    // A. 获取显式设置的位置 (插队逻辑)
                    var explicitRow = child.ReadLocalValue(Grid.RowProperty);
                    var explicitCol = child.ReadLocalValue(Grid.ColumnProperty);

                    // 获取控件的跨度 (Span)
                    int rowSpan = Grid.GetRowSpan(child);
                    int colSpan = Grid.GetColumnSpan(child);

                    int targetRow, targetCol;

                    // B. 确定放置位置
                    if (explicitRow != DependencyProperty.UnsetValue || explicitCol != DependencyProperty.UnsetValue)
                    {
                        // 如果用户显式指定了位置，就用用户的
                        targetRow = (explicitRow != DependencyProperty.UnsetValue) ? (int)explicitRow : 0;
                        targetCol = (explicitCol != DependencyProperty.UnsetValue) ? (int)explicitCol : 0;

                        // 同时更新搜索游标，让后续控件接在这个后面
                        // (这里简单处理：重置到当前行开头或下一行，实际情况会自动被 while 循环修正)
                        searchRow = targetRow;
                        searchCol = targetCol;
                    }
                    else
                    {
                        // ★★★ 查找下一个空位 ★★★
                        while (true)
                        {
                            // 检查当前格子是否被占用
                            if (!occupiedCells.Contains((searchRow, searchCol)))
                            {
                                // 找到了空位！
                                targetRow = searchRow;
                                targetCol = searchCol;
                                break;
                            }

                            // 移动到下一格
                            searchCol++;
                            if (searchCol >= colCount)
                            {
                                searchCol = 0;
                                searchRow++;
                            }
                        }
                    }

                    // C. 动态补齐行定义
                    // 注意：要考虑 RowSpan，如果跨了 2 行，可能需要补 2 行
                    int rowsNeeded = targetRow + rowSpan;
                    while (rowsNeeded > grid.RowDefinitions.Count)
                    {
                        grid.RowDefinitions.Add(new RowDefinition { Height = defaultHeightLength });
                    }

                    // D. 应用位置
                    Grid.SetRow(child, targetRow);
                    Grid.SetColumn(child, targetCol);

                    // E. ★★★ 标记占用区域 (占座) ★★★
                    for (int r = 0; r < rowSpan; r++)
                    {
                        for (int c = 0; c < colSpan; c++)
                        {
                            // 标记 (targetRow + r, targetCol + c) 为已占用
                            occupiedCells.Add((targetRow + r, targetCol + c));
                        }
                    }
                }
            }
        }

        // 解析逗号分隔字符串 "Auto, *, 50"
        private static GridLength[] ParseDefinitions(string definitions)
        {
            var parts = definitions.Split(',');
            var lengths = new GridLength[parts.Length];
            for (int i = 0; i < parts.Length; i++)
            {
                lengths[i] = ParseSingleLength(parts[i]);
            }
            return lengths;
        }

        // 解析单个长度字符串
        private static GridLength ParseSingleLength(string part)
        {
            part = part.Trim().ToLower();
            if (part == "auto")
                return GridLength.Auto;
            else if (part.EndsWith("*"))
            {
                double val = 1;
                if (part.Length > 1) double.TryParse(part.Replace("*", ""), out val);
                return new GridLength(val, GridUnitType.Star);
            }
            else
            {
                if (double.TryParse(part, out double val)) return new GridLength(val);
                return GridLength.Auto;
            }
        }
    }
```