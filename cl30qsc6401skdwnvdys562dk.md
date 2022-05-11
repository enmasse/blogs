## A use case for enumerators and ranges in C#

I've struggled to find a use case for the possibility to write ```
.GetEnumerator()
``` as extension methods in C# 9, and I think I found one. ```
Range
```s in C# is only for producing slices of arrays and not for defining a collection as in many other languages. But with a simple extension method, it's possible.


```
public static class RangeExtensions
{
	public static IEnumerator<int> GetEnumerator(this Range range) =>
		Enumerable.Range(range.Start.Value, range.End.Value - range.Start.Value + 1)
		.GetEnumerator();
}
``` 

Now it's possible to iterate over a ```
Range
``` with a ```
foreach
``` like this.


```
foreach(var indx in 1..11)
{
	Console.WriteLine(indx);
}
``` 

That will produce a result like this.


```
1
2
3
4
5
6
7
8
9
10
11
``` 

I hope you'll find some use for it. :)