---
title: C#面试题
description:
slug: C#_interview
date: 2024-12-24
image: 
categories:
    - 面试题
tags:
    - Interview
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

### 1. 字符串统计题

某外企一面编码题目。

Given two strings s and t, return true if t is an anagram of s, and false otherwise.
An Anagram is a word or phrase formed by rearranging the letters of a different word or phrase, typically using all the original letters exactly once.
 
Example 1:
Input: s = "anagram", t = "nagaram"
Output: true
 
Example 2:
Input: s = "rat", t = "car"
Output: false

 ```csharp
internal class Program
{
    private static void Main(string[] args)
    {
        Console.WriteLine(IsAnag("anagram", "nagaram"));
        Console.WriteLine(IsAnag("rat", "car"));
    }

    private static bool IsAnag(string s1, string s2)
    {
        var dict1= GetAnagDict(s1);
        var dict2= GetAnagDict(s2);

        foreach (var entry in dict1)
        {
            bool isMatch  = dict2.ContainsKey(entry.Key) && dict2[entry.Key] == entry.Value;
            if (!isMatch)
            {
                return false;
            }
        }
        return true;
    }

    private static Dictionary<char, int> GetAnagDict(string s1)
    {
        Dictionary<char, int> dict = new();
        for (int i = 0; i < s1.Length; i++)
        {
            if (dict.ContainsKey(s1[i]))
            {
                dict[s1[i]]++;
            }
            else
            {
                dict.Add(s1[i], 1);
            }
        }
        return dict;
    }
}
```

### 2. 杨辉三角问题
题目描述： 杨辉三角是我国古代一个重要的数学成就 。

![](https://s3.bmp.ovh/imgs/2024/12/25/f721899905bb6f3e.png)

如上图，杨辉三角是一个满足以下条件的几何排列：

1、每个数等于它上方两数之和。
2、每行数字左右对称，由1开始逐渐变大。
第 n 行的数字有 n 项。

请编写一个程序，按题目要求输出杨辉三角中第 n 行第 m 个数字。

输入

第一行，两个数字 n 和 m ，表示需要输出的数字在杨辉三角上的位置，行列均从 1 开始，（1<=n,m<=10000），以空格分隔。

输出

仅包含一个整数，即杨辉三角中第 n 行第 m 列处的数字。

输入示例
7 5

输出示例
15

```csharp
using YangHuiTriangleDemo;

/// <summary>
/// 题目5：杨辉三角
/// </summary>
internal class Program
{
    private static void Main(string[] args)
    {
        string tips = """
            Please input 2 integers to get the result of YangHui triangle.
            The 2 integers represents the row number and column number seperately.
            For example, if you input 3 2, the result will be 2.
            The range of row number is 1 to 10000, and the range of column number is 1 to 10000.
            Press Ctrl+C to exit.
            """;
        Console.WriteLine(tips);
        Console.WriteLine();
        YangHuiTriangle yangHuiTriangle = new YangHuiTriangle();
        while (true)
        {
            Console.WriteLine("Please input 2 integers:");
            string input = Console.ReadLine();
            if (input == null)
            {
                break;
            }
         
            (bool isOk,int row, int column) = ParseRowColumn(input);
            if (!isOk)
            {
                Console.WriteLine("Only 2 integers is allowed and they are seperated with one blank space.");
                continue;
            }
            if (row>10000|| column>10000)
            {
                Console.WriteLine("The range of row number is 1 to 10000, and the range of column number is 1 to 10000.");
                continue;
            }
            int result = yangHuiTriangle.GetNumber(row, column);
            Console.WriteLine(result);
        }

    }

    private static (bool isOk,int row, int column) ParseRowColumn(string input)
    {
        string[] inputArray = input.Split(' ');
        if (inputArray.Length!=2)
        {
            return (false,default,default);
        }
        bool isRowOk = int.TryParse(inputArray[0],out int row);
        bool isColumnOk = int.TryParse(inputArray[1],out int column);
        return (isRowOk && isColumnOk,row,column);
    }
}

namespace YangHuiTriangleDemo;

public class YangHuiTriangle
{
    // 当前内存中杨辉三角数组的最大行数
    private int _maxNumRows;
    private int[,] _arr;

    public int GetNumber(int row, int col)
    {
        GenerateYanghuiTriangle(row);
        return _arr[row - 1, col - 1];
    }

    /// <summary>
    /// 生成杨辉三角
    /// </summary>
    /// <param name="numRows"></param>
    private void GenerateYanghuiTriangle(int numRows)
    {
        // 如果当前内存中已经存储了足够的杨辉三角行数，直接返回
        if (numRows <= _maxNumRows)
        {
            return;
        }

        // 创建二维数组来存储杨辉三角形
        int[,] arr = new int[numRows, numRows];

        // 扩容数组，补充还没生成的杨辉三角部分
        if (_maxNumRows > 0)
        {
            CopyJaggedArray(_arr, arr);
        }

        // 按照杨辉三角形的规律填充数组元素
        for (int i = _maxNumRows; i < numRows; i++)
        {
            for (int j = 0; j <= i; j++)
            {
                if (j == 0 || j == i)
                {
                    arr[i, j] = 1;
                }
                else
                {
                    arr[i, j] = arr[i - 1, j - 1] + arr[i - 1, j];
                }
            }
        }

        _maxNumRows = numRows;
        _arr = arr;
    }

    /// <summary>
    /// 拷贝交错数组
    /// </summary>
    /// <param name="oldArr"></param>
    /// <param name="newArr"></param>
    private void CopyJaggedArray(int[,] oldArr, int[,] newArr)
    {
        for (int i = 0; i < oldArr.GetLength(0); i++)
        {
            for (int j = 0; j < oldArr.GetLength(1); j++)
            {
                newArr[i, j] = oldArr[i, j];
            }
        }
    }
}


```

### 3. 数据可视化

编程语言：不限

题目描述：有句话是这么说的：“文不如表，表不如图”。形象地描述了图表在传达信息时，给接收者带来的截然不同的效率和体验。因此，在计算机计算能力、数据规模和决策需求都不断提升的当下，数据可视化的应用也越来越普遍。

数据可视化的范围很广，涉及到数据的获取、加工、建模、图形学，人机交互等很多概念和领域，想更快上手，获得更好的体验，使用DragonFly BI这样的专业工具和服务是更明智的选择。

今天，我们通过一个简化的命题，来亲手实现简单的数据可视化。编写一个程序，对于给定的一组数据和要求，输出一个以字符组成的柱状图。

输入

第一行，一个整数 N（1 <=n<=20），表示这组数据的条目数。
第二行，两个字符串，用于表示数据展示在柱状图上的排序方式。第一个字符串是“Name” 或者 “Value”，表示排序的依据是数据条目的名称亦或数值；第二个字符串是 “ASC” 或者 “DESC”，表示升序或降序。
随后的 N 行，每行包含一个字符串 S 和一个数字 V，以空格分隔，表示一条数据。S 即数据条目的名称，仅包含小写字母，V 即对应的数值，是一个整数，(0<=V<=1,000,000)< /P>

输出

图表外框转角符号：

“┌”（\u250c）
“┐”（\u2510）
“└”（\u2514）
“┘”（\u2518）
图表中的横、竖线：

“─”（\u2500）
“│”（\u2502）
图表中的各种交叉线：

“├”（\u251c）
“┤”（\u2524）
“┬”（\u252c）
“┴”（\u2534）
“┼”（\u253c）
用来拼柱子的字符：

“█”（\u2588）
图表中的空格：

“ ”（\u0020）
图表中名称区域的宽度，由这组数据中名称的最大长度决定，所有名称向右对齐， 图表中柱的最大长度为 20，每个柱的长度由该柱对应数据和这组数据中最大值（此值一定大于 0）的比值与 20 相乘获得，不足一格的部分舍去。

输入示例

3
Value DESC
apple 5
pen 3
pineapple 10

输出示例

![](https://s3.bmp.ovh/imgs/2024/12/26/05e908cd4234101f.png)

``` c#

namespace DataVisualizationDemo;

/// <summary>
/// 题目2：数据可视化
/// </summary>
internal class Program
{
    static void Main(string[] args)
    {
        DataProcessor processor = new();
        while (true)
        {
            Console.WriteLine("Please input your data:");

            IList<Product> products = processor.Parse();

            products=processor.ProcessData(products);

            processor.PrintData(products);
        }
    }

}

namespace DataVisualizationDemo;

public class Product
{
    public string Name { get; set; }
    public int Value { get; set; }
}


using System.Text;

namespace DataVisualizationDemo;

internal class DataProcessor
{
    private string orderField;
    private bool isAsc;

    public IList<Product> Parse()
    {
        if (!int.TryParse(Console.ReadLine(), out int lineCount))
        {
            return new List<Product>();
        }
        string sortDesc = Console.ReadLine();
        string[] arr = sortDesc.Split(" ");
        orderField = arr[0];
        isAsc = arr[1].Equals("ASC", StringComparison.CurrentCultureIgnoreCase);

        IList<Product> products = new List<Product>();
        for (int i = 0; i < lineCount; i++)
        {
            string line = Console.ReadLine();
            string[] lineArray = line.Split(" ");
            if (lineArray.Length != 2)
            {
                continue;
            }
            string name = lineArray[0];
            int value = int.Parse(lineArray[1]);
            products.Add(new Product { Name = name, Value = value });
        }
        return products;
    }

    public IList<Product> ProcessData(IList<Product> products)
    {
        if ("name".Equals(orderField, StringComparison.CurrentCultureIgnoreCase))
        {
            products = isAsc ? products.OrderBy(p => p.Name).ToList() :
            products.OrderByDescending(p => p.Name).ToList();
        }
        else
        {
            products = isAsc ? products.OrderBy(p => p.Value).ToList() :
            products.OrderByDescending(p => p.Value).ToList();
        }
        return products;
    }

    public void PrintData(IList<Product> products)
    {
        int maxNameLength = products.Max(p => p.Name.Length);
        int maxValueLength = products.Max(p => p.Value);
        int maxValueThreshold = Console.WindowWidth - maxNameLength - 4;

        PrintHeader(maxNameLength, maxValueLength, maxValueThreshold);

        StringBuilder sb = new();
        for (int i = 0; i < products.Count; i++)
        {
            PrintDataLine(products[i], maxNameLength, maxValueLength, sb, maxValueThreshold);
            if (i == products.Count - 1)
            {
                PrintFooter(maxNameLength, maxValueLength, maxValueThreshold);
                break;
            }
            Console.Write("├");
            PrintNameRowSpliter(maxNameLength);

            Console.Write("┼");
            PrintValueRowSpliter(maxValueLength, maxValueThreshold);

            Console.WriteLine("┤");
        }
    }

    private void PrintDataLine(Product product, int maxNameLength, int maxValueLength, StringBuilder sb, int maxValueThreshold)
    {
        int actualWidth = maxValueLength < maxValueThreshold ? maxValueLength : maxValueThreshold;

        List<string> formatedValue = GenerateFormatedValue(sb, product.Value, maxValueThreshold);

        for (int i = 0; i < formatedValue.Count; i++)
        {
            if (i == 0)
            {
                Console.WriteLine($"│{product.Name.PadLeft(maxNameLength)}│{formatedValue[i].PadRight(actualWidth)}│");
                continue;
            }
            else
            {
                Console.WriteLine($"│{string.Empty.PadLeft(maxNameLength)}│{formatedValue[i].PadRight(actualWidth)}│");
            }
        }
    }

    /// <summary>
    /// 生成格式化的值字符串
    /// </summary>
    /// <param name="sb"></param>
    /// <param name="productValue"></param>
    /// <param name="maxValueThreshold">值的一行最大的字符数</param>
    /// <returns></returns>
    private List<string> GenerateFormatedValue(StringBuilder sb, int productValue, int maxValueThreshold)
    {
        sb.Clear();
        // 当value很大时，需要换行，所以需要计算行数
        int valueLineCount = (int)Math.Ceiling(productValue * 1.0d / maxValueThreshold);

        List<string> result = new List<string>(valueLineCount);
        int actualWidth = productValue < maxValueThreshold ? productValue : maxValueThreshold;
        for (int i = 0; i < productValue; i++)
        {
            sb.Append("█");
            if (sb.Length == maxValueThreshold)
            {
                result.Add(sb.ToString());
                sb.Clear();
            }
        }
        if (sb.Length > 0)
        {
            // 拼上最后一行
            result.Add(sb.ToString());
        }
        return result;
    }

    private static void PrintHeader(int maxNameLength, int maxValueLength, int maxValueThreshold)
    {
        Console.Write("┌");
        PrintNameRowSpliter(maxNameLength);

        Console.Write("┬");
        PrintValueRowSpliter(maxValueLength, maxValueThreshold);

        Console.WriteLine("┐");
    }

    private static void PrintFooter(int maxNameLength, int maxValueLength, int maxValueThreshold)
    {
        Console.Write("└");
        PrintNameRowSpliter(maxNameLength);
        Console.Write("┴");
        PrintValueRowSpliter(maxValueLength, maxValueThreshold);
        Console.WriteLine("┘");
    }

    private static void PrintValueRowSpliter(int maxValueLength, int maxValueThreshold)
    {
        int actualWidth = maxValueLength < maxValueThreshold ? maxValueLength : maxValueThreshold;

        for (int i = 0; i < actualWidth; i++)
        {
            Console.Write("─");
        }
    }

    private static void PrintNameRowSpliter(int maxNameLength)
    {
        for (int i = 0; i < maxNameLength; i++)
        {
            Console.Write("─");
        }
    }
}

```