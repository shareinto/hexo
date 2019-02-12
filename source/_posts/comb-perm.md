title: 排列组合算法
date: 2016-10-31 20:22:56
tags: 
  - algorithm
---

组合
```bash
public static void Comb(int[] array, int offset, int n, int[] result)
{
    if (n == 0)
    {
        Print(result);
        return;
    }
    for (int i = offset; i < array.Length; i++)
    {
        result[result.Length - n] = array[i];
        Comb(array, i + 1, n - 1, result);
    }
}
```
排列
```bash
public static void Perm(int[] array, int offset,int n)
{
    if (offset == n)
    {
        Print(array, n);
        return;
    }
    for (int i = offset; i < array.Length; i++)
    {
        if (i != offset)
        {
            Swap(ref array[i], ref array[offset]);
        }
        Perm(array, offset + 1, n);
        if (i != offset)
        {
            Swap(ref array[i], ref array[offset]);
        }
    }
}
```

